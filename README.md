# ci-workflows

Shared GitHub Actions workflows for shadowtor / Goated-Geese projects.

## `image-deploy.yml`

Builds a Docker image in GitHub Actions, pushes it to GHCR, and asks Coolify to
deploy it, so the hosting server only pulls images and never builds. This is the
default deploy path for every project (see the `project-stack` skill, "Hosting").

- Tags: `sha-<commit>` (immutable) plus a moving tag (branch name by default).
- Coolify runs a **Docker Image** app tracking the moving tag; the deploy call uses a
  **deploy-only** Coolify token stored as a branch-restricted environment secret.
- Already-built commits are retagged with `crane`, never rebuilt.
- Prune deletes only `sha-*`-only versions (newest 10 kept).
- Docs-only pushes skip build and deploy: if every file changed since the previous
  push matches `skip-paths` (default `**/*.md`, `docs/**`, `.planning/**`, `.claude/**`),
  nothing runs. Rollbacks, manual runs and new branches always deploy. Set
  `skip-paths: ""` to always deploy, or pass your own list.

### Caller example

Call it from `@main`: `main` is the stable line, so fixes reach every repo with no
version bumps. Changes land on a branch and merge to `main` only once tested.

```yaml
name: build-deploy
on:
  push: { branches: [test, main] }
  workflow_dispatch:
    inputs:
      sha: { description: Commit to roll back to, required: true }
permissions:
  contents: read
  packages: write
jobs:
  verify:
    if: github.event_name != 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: 22, cache: npm }
      - run: npm ci && npm run build --if-present
  deploy:
    needs: verify
    if: ${{ !cancelled() && (needs.verify.result == 'success' || needs.verify.result == 'skipped') }}
    uses: shadowtor/ci-workflows/.github/workflows/image-deploy.yml@main
    with:
      image: ghcr.io/owner/repo
      environment: ${{ github.ref_name == 'main' && 'production' || 'test' }}
      coolify-app-uuid: ${{ github.ref_name == 'main' && vars.COOLIFY_UUID_MAIN || vars.COOLIFY_UUID_TEST }}
      sha: ${{ inputs.sha }}
    secrets: inherit
```

Per repo: create environments `test` (branch `test`) and `production` (branch `main`)
with secret `COOLIFY_DEPLOY_TOKEN` (Coolify token with **deploy** only), and repo
variables for the Coolify app uuids.
