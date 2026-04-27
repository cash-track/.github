# cash-track/.github

Reusable GitHub Actions workflows shared across the `cash-track` org. Each workflow lives under
`.github/workflows/` and is consumed by other repos via `uses:`.

## Workflows

### `ship-service.yml` — build, push, and deploy a service

Tag-driven release pipeline used by every cashtrack-owned service repo: `api`, `gateway`,
`frontend`, `website`, plus the infra images `mysql`, `redis`, `mysql-backup`. Builds the
Docker image, pushes it to Docker Hub, joins the tailnet as `tag:ci`, and runs the on-droplet
`/opt/cashtrack/bin/deploy-service` script over Tailscale SSH (no PAT, no SSH key in CI).

The deploy script edits `VERSION_<SERVICE>` in `/opt/cashtrack/.env` (the service name is
upper-cased and `-` → `_`, so `mysql-backup` resolves to `VERSION_MYSQL_BACKUP`), then
`docker compose pull <service> && docker compose up -d --no-deps <service>`. Restart-style
recreation is acceptable for stateless services and tolerated for `mysql`/`redis` (brief
downtime on image bump). `mysql-backup` is a cron container so restart is trivial.

Inputs:

| Input | Required | Default | Description |
|---|---|---|---|
| `service` | yes | — | Compose service name (`api`, `gateway`, …) — used for the `VERSION_<SERVICE>` env var on the droplet |
| `image` | yes | — | Docker Hub repo (e.g. `cashtrack/api`) |
| `tag` | yes | — | Image tag — typically `${{ github.ref_name }}` |
| `context` | no | `.` | Docker build context |
| `dockerfile` | no | `Dockerfile` | Dockerfile path relative to `context` |
| `run_migrations` | no | `false` | When `true` and `service == "api"`, runs `php app.php migrate -s -n` before `up -d` |

Required secrets (use `secrets: inherit` in the caller):

`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `TS_OAUTH_CLIENT_ID`, `TS_OAUTH_SECRET`.

Caller example (`cash-track/api/.github/workflows/release.yml`):

```yaml
name: Release
on:
  push:
    tags: ['v*.*.*']
jobs:
  ship:
    uses: cash-track/.github/.github/workflows/ship-service.yml@main
    with:
      service: api
      image:   cashtrack/api
      tag:     ${{ github.ref_name }}
      run_migrations: true
    secrets: inherit
```

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
`SPACES_TFSTATE_KEY` are passed through when present (used by terraform-touching tasks).

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

`actionlint` covers all five workflows:

```bash
actionlint .github/workflows/*.yml
```
