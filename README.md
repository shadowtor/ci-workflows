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
- Every build receives `GIT_SHA` (the commit being built; the rollback sha on a rollback). `GIT_BRANCH` is passed **only
  when `variant` equals the branch name**, i.e. when the image tag is already branch-specific; otherwise it is empty,
  because a commit built on one branch and promoted to another is retagged, not rebuilt, and a baked branch would go
  stale. Declare `ARG GIT_SHA` / `ARG GIT_BRANCH` and matching `ENV` lines in the **final** Dockerfile stage (after the
  heavy layers, so the cache is not busted) to read them at runtime. Coolify cannot supply these for image apps: it
  sets `SOURCE_COMMIT=HEAD` and no `COOLIFY_BRANCH`. Treat an empty, `unknown` or `HEAD` value as absent.
- Prune deletes only `sha-*`-only versions (newest 10 kept).
- Docs-only pushes skip build and deploy: if every file changed since the previous
  push matches `skip-paths` (default `**/*.md`, `docs/**`, `.planning/**`, `.claude/**`),
  nothing runs. Rollbacks, manual runs and new branches always deploy. Set
  `skip-paths: ""` to always deploy, or pass your own list.

### Caller example

Pin the caller to a **full 40-character commit sha** of this repo, not `@main` or a tag.
This workflow runs with `packages: write` and the caller's secrets (`secrets: inherit` in
some callers), so a moving ref would let any change here run with those secrets with no
review in the caller. Add a `github-actions` entry to the caller's `.github/dependabot.yml`
so sha bumps arrive as reviewable PRs, and read this repo's diff before merging one.
Changes here land through a PR, wait for the Codex review comment, then merge to `main`;
callers pick them up by merging their Dependabot PR.

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
    uses: shadowtor/ci-workflows/.github/workflows/image-deploy.yml@<40-char-sha> # note which change this is
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
