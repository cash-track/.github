# .github

Reusable GitHub Actions workflows shared across the `cash-track` org.

## Workflows

### `build.yml` — build and push a Docker image

Reusable build + push. Logs in to Docker Hub, derives image tags from the calling git
ref (so a push to `v1.2.9` produces `cashtrack/<name>:1.2.9` and `cashtrack/<name>:sha-abc1234`,
with `:latest`), builds with Buildx + GHA cache, pushes, and emits SLSA
provenance attestation.

### `deploy.yml` — deploy a service to the prod droplet

Reusable deploy-only. Joins the tailnet as `tag:ci` and runs
`/opt/cashtrack/bin/deploy-service <service> <tag>` over Tailscale SSH. The
on-droplet script edits `VERSION_<SERVICE>` in `/opt/cashtrack/.env`, then
`docker compose pull <service> && docker compose up -d --no-deps <service>`.

No build, no Docker Hub credentials, no rebuild on rollback.

### `ansible.yml` — run a playbook against the prod droplet

Reusable wrapper that checks out `cash-track/infra`, installs Ansible + the required Galaxy
collections, installs the 1Password CLI, joins the tailnet as `tag:ci`, and runs
`ansible-playbook` with optional `--tags` / `--limit` / `--check` filters. Triggered from a
caller workflow in `cash-track/infra` on push to `ansible/**`, `compose/**`, `terraform/**`, or
on `workflow_dispatch`.

### `quality-go.yml` — Go build, vet, lint, test

Quality workflow: Go 1.26, golangci-lint v2.11.2, `CGO_ENABLED=1`, race detector on.

### `quality-php.yml` — PHP static analysis, coding standards, PHPUnit

Runs Psalm, PHP_CodeSniffer, and PHPUnit with MySQL + Redis service
containers. Defaults to PHP 8.4 and the API's known extension set.

### `quality-node.yml` — Node lint and test

Lints with the caller-defined `npm run lint`; tests run when a
non-empty `test-command` is supplied.

## Using these workflows from outside the `cash-track` org

The workflows live in a public repo, so any GitHub repo (public or private, any owner) can call them with the same `uses: cash-track/.github/.github/workflows/<name>.yml@<ref>` syntax. The differences from in-org callers:

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

## Local checks

`actionlint` covers all six workflows:

```bash
actionlint .github/workflows/*.yml
```
