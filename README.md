# cash-track/.github

Reusable GitHub Actions workflows shared across the `cash-track` org. Each workflow lives under
`.github/workflows/` and is consumed by other repos via `uses:`.

## Release pipeline shape

The release pipeline is split into two reusables — `build.yml` and `deploy.yml` — to mirror
the existing per-repo structure (`build.yml`, `deploy.yml`, `release.yml`). Splitting matters
because:

- **Tag push** auto-builds, pushes, then deploys via a chained `release.yml` in the service repo.
- **Rebuild without redeploy** (e.g. cache invalidation, base-image refresh) calls `build.yml`
  directly — no SSH, no droplet contact.
- **Redeploy without rebuild** (rollback) calls `deploy.yml` directly with the older tag —
  pulls the existing image off Docker Hub, no rebuild, deterministic.

A combined "ship" workflow would force a rebuild for every rollback, breaking that guarantee.

## Workflows

### `build.yml` — build and push a Docker image

Reusable build + push. Logs in to Docker Hub, derives image tags via
[`docker/metadata-action@v5`](https://github.com/docker/metadata-action) from the calling git
ref (so a push to `v1.2.9` produces `cashtrack/<name>:1.2.9` and `cashtrack/<name>:sha-abc1234`,
with `:latest` controlled by `flavor`), builds with Buildx + GHA cache, pushes, and emits SLSA
provenance attestation.

Inputs:

| Input | Required | Default | Description |
|---|---|---|---|
| `image` | yes | — | Docker repo (e.g. `cashtrack/api`) |
| `context` | no | `.` | Docker build context |
| `dockerfile` | no | `Dockerfile` | Dockerfile path relative to `context` |
| `tag_rules` | no | `type=sha` + `type=semver,pattern={{version}}` | docker/metadata-action `tags` rules. Override only when a repo needs an extra rule (e.g. `type=raw,value=stable,enable={{is_default_branch}}`). |
| `flavor` | no | `latest=auto` | docker/metadata-action `flavor` rules. `latest=auto` pushes `:latest` only on the highest semver tag; set `latest=false` to disable, `latest=true` to always push. |
| `build_args` | no | `GIT_COMMIT` + `GIT_TAG` | Newline-separated `KEY=VALUE` build args. Default stamps the image with `GIT_COMMIT=${{ github.sha }}` and `GIT_TAG=${{ github.ref_name }}`. Override only when extra args are needed — re-specify the defaults in the override if you still want them. |
| `attest` | no | `true` | Push SLSA provenance attestation to the registry |

Required secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`.

Outputs:
- `digest` — image digest of the pushed manifest.
- `version` — primary version derived by metadata-action (e.g. `1.2.9` for git tag `v1.2.9`). **This is the value `release.yml` should pass to `deploy.yml` so the deploy tag matches what was actually pushed.**
- `tags` — newline-separated full `image:tag` refs that were pushed.

### `deploy.yml` — deploy a service to the prod droplet

Reusable deploy-only. Joins the tailnet as `tag:ci` and runs
`/opt/cashtrack/bin/deploy-service <service> <tag> <run_migrations>` over Tailscale SSH. The
on-droplet script edits `VERSION_<SERVICE>` in `/opt/cashtrack/.env` (service name upper-cased,
`-` → `_`, so `mysql-backup` resolves to `VERSION_MYSQL_BACKUP`), then
`docker compose pull <service> && docker compose up -d --no-deps <service>`. Restart-style
recreation is acceptable for stateless services and tolerated for `mysql`/`redis` (brief
downtime on image bump). `mysql-backup` is a cron container so restart is trivial.

No build, no Docker Hub credentials, no rebuild on rollback.

Inputs:

| Input | Required | Default | Description |
|---|---|---|---|
| `service` | yes | — | Compose service name |
| `tag` | yes | — | Image tag to deploy, verbatim |
| `run_migrations` | no | `false` | Run `php app.php migrate -s -n` before recreate (api only) |
| `droplet_host` | no | `cashtrack-prod-0` | Tailscale hostname |
| `droplet_user` | no | `ops` | SSH user on droplet |

Required secrets: `TS_OAUTH_CLIENT_ID`, `TS_OAUTH_SECRET`.

### Caller examples

`cash-track/api/.github/workflows/release.yml` — auto-fires on tag, chains build → deploy.
Note that `deploy` consumes `needs.build.outputs.version` (the metadata-action-stripped tag),
not `github.ref_name`, so the deploy tag matches what `build` actually pushed:

```yaml
name: release
on:
  push:
    tags: ['v*.*.*']
jobs:
  build:
    uses: cash-track/.github/.github/workflows/build.yml@main
    with:
      image: cashtrack/api
    secrets: inherit

  deploy:
    needs: build
    uses: cash-track/.github/.github/workflows/deploy.yml@main
    with:
      service: api
      tag:     ${{ needs.build.outputs.version }}
      run_migrations: true
    secrets: inherit
```

`cash-track/api/.github/workflows/build.yml` — manual rebuild without deploy. Operator dispatches
with `gh workflow run build.yml --ref v1.2.9` (or any branch / SHA); metadata-action derives the
tag from the checked-out ref:

```yaml
name: build
on:
  workflow_dispatch:
jobs:
  build:
    uses: cash-track/.github/.github/workflows/build.yml@main
    with:
      image: cashtrack/api
    secrets: inherit
```

`cash-track/api/.github/workflows/deploy.yml` — manual redeploy / rollback. The `tag` input is
the bare version pushed to Docker Hub (`1.2.8`, not `v1.2.8`):

```yaml
name: deploy
on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Image tag on Docker Hub (e.g. 1.2.8 — no leading v)"
        required: true
        type: string
      run_migrations:
        type: boolean
        default: false
jobs:
  deploy:
    uses: cash-track/.github/.github/workflows/deploy.yml@main
    with:
      service: api
      tag:     ${{ inputs.tag }}
      run_migrations: ${{ inputs.run_migrations }}
    secrets: inherit
```

The infra image repos (`cash-track/mysql`, `cash-track/redis`, `cash-track/mysql-backup`) follow
the exact same shape — only `service` and `image` change, and `run_migrations` is always `false`.

### `ansible-apply.yml` — run a playbook against the prod droplet

Reusable wrapper that checks out `cash-track/infra`, installs Ansible + the required Galaxy
collections, installs the 1Password CLI, joins the tailnet as `tag:ci`, and runs
`ansible-playbook` with optional `--tags` / `--limit` / `--check` filters. Triggered from a
caller workflow in `cash-track/infra` on push to `ansible/**`, `compose/**`, `terraform/**`, or
on `workflow_dispatch`.

Inputs (all optional):

| Input | Default | Description |
|---|---|---|
| `ref` | `main` | `cash-track/infra` ref to check out |
| `playbook` | `site.yml` | Path relative to `infra/ansible/` |
| `tags` | `""` | Forwarded as `--tags` |
| `limit` | `""` | Forwarded as `--limit` |
| `check_mode` | `false` | When `true`, runs `--check --diff` |

Required secrets:

`OP_SERVICE_ACCOUNT_TOKEN`, `TS_OAUTH_CLIENT_ID`, `TS_OAUTH_SECRET`. `SPACES_TFSTATE_ID` and
`SPACES_TFSTATE_KEY` are passed through when present.

Caller example (`cash-track/infra/.github/workflows/ansible-apply.yml`):

```yaml
name: Ansible apply
on:
  push:
    branches: [main]
    paths:
      - 'ansible/**'
      - 'compose/**'
      - 'terraform/**'
  workflow_dispatch:
    inputs:
      tags:
        type: string
      check_mode:
        type: boolean
        default: false
jobs:
  apply:
    uses: cash-track/.github/.github/workflows/ansible-apply.yml@main
    with:
      tags:       ${{ inputs.tags || '' }}
      check_mode: ${{ inputs.check_mode || false }}
    secrets: inherit
```

### `quality-go.yml` — Go build, vet, lint, test

Used by `gateway`. Defaults match the existing self-managed quality workflow: Go 1.26,
golangci-lint v2.11.2, `CGO_ENABLED=1`, race detector on.

Inputs (all optional): `go-version`, `runs-on`, `golangci-lint-version`, `test-packages`,
`cgo-enabled`, `upload-coverage`.

Optional secrets: `CODECOV_TOKEN`.

Caller example (`cash-track/gateway/.github/workflows/quality.yml`):

```yaml
name: quality
on:
  push:
    branches: [main]
  pull_request:
jobs:
  quality:
    uses: cash-track/.github/.github/workflows/quality-go.yml@main
    with:
      test-packages: $(go list ./... | grep /gateway/ | grep -v /mocks)
    secrets: inherit
```

### `quality-php.yml` — PHP static analysis, coding standards, PHPUnit

Used by `api`. Runs Psalm, PHP_CodeSniffer, and PHPUnit with MySQL + Redis service
containers. Defaults to PHP 8.4 and the API's known extension set.

Inputs (all optional): `php-version`, `runs-on`, `php-extensions`, `mysql-image`, `redis-image`,
`run-tests`, `upload-coverage`.

Optional secrets: `CODECOV_TOKEN`.

Caller example (`cash-track/api/.github/workflows/quality.yml`):

```yaml
name: quality
on:
  push:
    branches: [master]
  pull_request:
jobs:
  quality:
    uses: cash-track/.github/.github/workflows/quality-php.yml@main
    secrets: inherit
```

### `quality-node.yml` — Node lint and test

Used by `frontend` and `website`. Lints with the caller-defined `npm run lint`; tests run when a
non-empty `test-command` is supplied.

Inputs (all optional): `node-version`, `runs-on`, `install-command`, `lint-command`,
`test-command`, `working-directory`.

Caller example (`cash-track/frontend/.github/workflows/quality.yml`):

```yaml
name: quality
on:
  push:
    branches: [master]
  pull_request:
jobs:
  quality:
    uses: cash-track/.github/.github/workflows/quality-node.yml@main
    with:
      test-command: npm run test:unit -- --run
```

## Using these workflows from outside the `cash-track` org

The workflows live in a public repo, so any GitHub repo (public or private, any owner) can call them with the same `uses: cash-track/.github/.github/workflows/<name>.yml@<ref>` syntax. The differences from in-org callers:

- **Pin to an immutable ref.** You don't control changes here. Use a commit SHA (`@<40-char-sha>`) or a release tag — never `@main`. Dependabot's `package-ecosystem: github-actions` keeps the pin current.
- **`secrets: inherit` does not cross organizations.** Pass each required secret explicitly by name, and create them in your repo (Settings → Secrets and variables → Actions) first:
  ```yaml
  secrets:
    DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
    DOCKERHUB_TOKEN:    ${{ secrets.DOCKERHUB_TOKEN }}
  ```
- **Caller `GITHUB_TOKEN` permissions cap the reusable.** `build.yml` declares `id-token: write` and `attestations: write` on its own job, but the effective token is the intersection of caller workflow-level and called job-level permissions. If your repo's default token is read-only, declare them at the top of the caller workflow:
  ```yaml
  permissions:
    contents: read
    id-token: write
    attestations: write
  ```
  Set `attest: false` to skip provenance entirely if you don't want to grant `id-token: write`.
- **No org-policy gating.** `cash-track/.github` is public, so no "Allow actions from selected organizations" allowlist entry is needed in the caller's org settings — public reusable workflows are always callable.

### What's actually portable

| Workflow | Portable? | Notes |
|---|---|---|
| `build.yml` | yes | Set `image:` to your own Docker Hub repo. Bring your own `DOCKERHUB_*` secrets. |
| `quality-php.yml` | yes | Defaults match the cash-track API; override `php-version`, `php-extensions`, `mysql-image`, `redis-image` for other apps. |
| `quality-go.yml` | yes | Override `go-version`, `golangci-lint-version`, `test-packages` to match the project. |
| `quality-node.yml` | yes | Set `lint-command`, `test-command`, `working-directory` to match. |
| `deploy.yml` | yes, with droplet provisioning | Targets the cash-track shared droplet via Tailscale SSH. Callable from any repo whose service is already provisioned in `cash-track/infra` (env template, compose entry, version pin). See "Deploying an external Laravel service to the shared droplet" below. |
| `ansible-apply.yml` | no | Checks out `cash-track/infra` and joins the cash-track tailnet. Fork and adapt. |

### Minimal external caller (build + push on tag)

```yaml
name: build
on:
  push:
    tags: ['v*.*.*']
permissions:
  contents: read
  id-token: write
  attestations: write
jobs:
  build:
    uses: cash-track/.github/.github/workflows/build.yml@<pinned-sha-or-tag>
    with:
      image: yourorg/yourapp
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN:    ${{ secrets.DOCKERHUB_TOKEN }}
```

### Minimal external caller (Node quality on PR)

```yaml
name: quality
on:
  pull_request:
jobs:
  quality:
    uses: cash-track/.github/.github/workflows/quality-node.yml@<pinned-sha-or-tag>
    with:
      working-directory: ./web
      test-command: npm test -- --run
```

### Deploying an external Laravel service to the shared droplet

External Laravel apps that already run as containers on the cash-track droplet — `crashers-bot` and `home-exporter` are the existing precedents — can chain `build.yml` → `deploy.yml` exactly like in-org services. The droplet bumps the image tag, pulls, and recreates the container. No kubectl, no DOKS.

This is *not* a generic Laravel-deploy reusable. It's specifically for services that share the cash-track droplet. If you need to deploy a Laravel app to your own infra, fork `deploy.yml` — the workflow's value is in the on-droplet script (`/opt/cashtrack/bin/deploy-service`) plus the Tailscale ACL grants, not in the workflow YAML.

#### Prerequisites (one-time, in `cash-track/infra`)

Before the caller workflow can deploy anything, the service must already exist on the droplet. That's an infra-repo PR, not a caller-repo change:

1. Add the service to `versions:` in `ansible/group_vars/all/main.yml` with an initial tag.
2. Add the bare service name to `secret_files:` in the same file.
3. Render `ansible/roles/compose-render/templates/<service>.env.tpl` with the env vars and `op://` references for secrets (see `crashers-bot.env.tpl` as a template).
4. Add a service block to one of the compose files under `infra/compose/` (e.g. `compose.app.yml` for HTTP services, `compose.telegram.yml` for bots) — image must read `${VERSION_<SERVICE_UPPER>}` from `.env`.
5. Add the service name to `compose_services:` if it should run on the droplet today.
6. Push to `main` → `ansible-apply.yml` provisions the droplet.

The Tailscale ACL already grants `tag:ci` SSH to the droplet for any service that uses `deploy.yml`, so no ACL change is needed.

#### Caller workflow (Laravel example, chained build → deploy)

This is the shape `crashers-bot` (and any other external Laravel app) should adopt:

```yaml
name: release
on:
  push:
    tags: ['v*.*.*']

permissions:
  contents: read
  id-token: write
  attestations: write

jobs:
  build:
    uses: cash-track/.github/.github/workflows/build.yml@<pinned-sha>
    with:
      image: vovanms/crashers_bot_api   # your Docker Hub repo
      flavor: latest=false              # don't push :latest from a non-org repo
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN:    ${{ secrets.DOCKERHUB_TOKEN }}

  deploy:
    needs: build
    uses: cash-track/.github/.github/workflows/deploy.yml@<pinned-sha>
    with:
      service: crashers-bot                       # matches versions: key (with `-`, not `_`)
      tag:     ${{ needs.build.outputs.version }} # metadata-action-stripped semver, matches the pushed tag
    secrets:
      TS_OAUTH_CLIENT_ID: ${{ secrets.TS_OAUTH_CLIENT_ID }}
      TS_OAUTH_SECRET:    ${{ secrets.TS_OAUTH_SECRET }}
```

Add a separate `workflow_dispatch`-only `deploy.yml` in the caller repo for rollbacks — same shape, with a `tag` input passed straight through to the reusable.

#### Secrets to add to the caller repo

Four secrets, all owned by the cash-track operator. Get them from 1Password and copy into the caller repo (Settings → Secrets and variables → Actions):

| Secret | Source | Used by |
|---|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub account that has push rights to your image repo | `build.yml` |
| `DOCKERHUB_TOKEN` | Docker Hub access token (not password) | `build.yml` |
| `TS_OAUTH_CLIENT_ID` | Tailscale OAuth client with `auth_keys` + `devices:write` scopes, tag scope `tag:ci` | `deploy.yml` |
| `TS_OAUTH_SECRET` | Same OAuth client's secret | `deploy.yml` |

The Tailscale OAuth client is shared across all services that deploy via `deploy.yml`. You don't create a new one per repo — re-use the cash-track org's client. The bot repo just gets a copy of the same `TS_OAUTH_*` pair.

#### Laravel migrations

The on-droplet `deploy-service` script gates its migrate step on `SERVICE == "api"` (it runs `php app.php migrate` — Spiral, not Artisan). For any external Laravel service:

- **`run_migrations: true` is a no-op.** Don't pass it expecting Artisan to run.
- **Run migrations at container boot instead.** Make the Laravel image's entrypoint run `php artisan migrate --force` before starting RoadRunner / the queue worker. This is the cleanest path — every `deploy.yml` invocation is just an image bump + restart, and migrations happen as a side-effect of the new container starting. crashers-bot already follows this pattern.
- **If boot-time migration is unacceptable** (e.g. a long migration that would block container readiness), petition `cash-track/infra` to extend `deploy-service` with a per-service migration command map. That's an infra-repo change — don't try to bolt it onto the caller workflow with extra SSH steps.

#### Post-deploy hooks (webhooks, queue restart, cache clear)

The script has no post-deploy hook system. Two options, in order of preference:

1. **Make the entrypoint idempotent.** `php artisan config:cache`, `queue:restart`, telegram `webhook:set` — anything that can run safely every container start belongs in the entrypoint. Restart-style deploys (which is what `deploy.yml` does) trigger this naturally.
2. **Don't bolt extra SSH steps onto the caller.** Re-implementing the Tailscale + SSH dance to fire one post-deploy command duplicates the reusable's auth shape and leaks the OAuth secret into ad-hoc shell. If the hook genuinely cannot live in the entrypoint, extend `deploy-service` upstream.

## Local checks

`actionlint` covers all six workflows:

```bash
actionlint .github/workflows/*.yml
```
