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

## Local checks

`actionlint` covers all six workflows:

```bash
actionlint .github/workflows/*.yml
```
