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

Reusable build+push. Logs in to Docker Hub, builds with Buildx + GHA cache, pushes the image at
the supplied verbatim tag (no semver normalisation), optionally pushes `:latest`, and generates
build provenance attestation.

Inputs:

| Input | Required | Default | Description |
|---|---|---|---|
| `image` | yes | — | Docker repo (e.g. `cashtrack/api`) |
| `tag` | yes | — | Image tag — used verbatim, typically `${{ github.ref_name }}` |
| `context` | no | `.` | Docker build context |
| `dockerfile` | no | `Dockerfile` | Dockerfile path relative to `context` |
| `push_latest` | no | `true` | Also push the `:latest` tag |
| `build_args` | no | `""` | Newline-separated `KEY=VALUE` build args |
| `attest` | no | `true` | Push SLSA provenance attestation to the registry |

Required secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`.

Output: `digest` — image digest of the pushed manifest, available to downstream jobs.

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

`cash-track/api/.github/workflows/release.yml` — auto-fires on tag, chains build → deploy:

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
      tag:   ${{ github.ref_name }}
      build_args: |
        GIT_COMMIT=${{ github.sha }}
        GIT_TAG=${{ github.ref_name }}
    secrets: inherit

  deploy:
    needs: build
    uses: cash-track/.github/.github/workflows/deploy.yml@main
    with:
      service: api
      tag:     ${{ github.ref_name }}
      run_migrations: true
    secrets: inherit
```

`cash-track/api/.github/workflows/build.yml` — manual rebuild without deploy:

```yaml
name: build
on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Image tag to build (e.g. v1.2.9 or 1.2.9-rc1)"
        required: true
        type: string
jobs:
  build:
    uses: cash-track/.github/.github/workflows/build.yml@main
    with:
      image: cashtrack/api
      tag:   ${{ inputs.tag }}
      build_args: |
        GIT_COMMIT=${{ github.sha }}
        GIT_TAG=${{ inputs.tag }}
    secrets: inherit
```

`cash-track/api/.github/workflows/deploy.yml` — manual redeploy / rollback:

```yaml
name: deploy
on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Tag to deploy (must already exist on Docker Hub)"
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
