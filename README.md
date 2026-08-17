# go-ci

Shared, reusable GitHub Actions workflows for Go projects. Callers keep a
thin wrapper workflow that delegates to the workflows here via
`workflow_call`, instead of carrying their own copy of the CI logic.

## Why

Copy-pasted CI across many repos drifts over time: action versions fall out
of sync, one repo is missing a dependabot ecosystem, another still has a
deprecated step syntax, tag patterns become inconsistent, publishing
strategies fork. Centralizing the logic here means there is one place to
fix that, and dependabot can keep every caller current by bumping a single
`uses:` ref per repo.

## Versioning

Tag releases here as `v1`, `v2`, ... Callers should pin to a major tag
(`@v1`), not `@main`, so a breaking change in this repo doesn't roll out to
every caller at once. Dependabot's `github-actions` ecosystem in each
caller repo will pick up new tags automatically.

## Workflows

- `test.yml` — Go build + test matrix (ubuntu/macos/windows) plus a
  `security` job (`go vet`, race detector, `govulncheck`, `gosec`).
- `lint.yml` — `golangci-lint`.
- `release.yml` — GoReleaser on tag push.
- `docker-publish.yml` — builds once, pushes to GHCR (and optionally Docker
  Hub), multi-arch, with SBOM/provenance attestation and cosign signing.
  Validates (build, no push) on PRs; publishes on push to `main` or a
  `v*.*.*` tag. The two registries are handled differently:
  - **GHCR** (`ghcr.io/<owner>/<repo>`) is the default target for every
    project. Its ref is derived automatically from `github.repository` —
    no input needed, and it always matches the repo it's built from.
  - **Docker Hub** is opt-in, for legacy projects that already publish
    there — new projects shouldn't need it. Set the `dockerhub-image`
    input (e.g. `czerwonk/your-project`) and pass the `DOCKERHUB_USERNAME`
    / `DOCKERHUB_TOKEN` secrets to enable it; leave `dockerhub-image`
    unset and it's skipped entirely, GHCR-only.
- `helm-release.yml` — packages a chart and publishes it to the `gh-pages`
  branch as a traditional Helm repo.

## Example: wiring up a caller repo

The caller repo keeps its own trigger conditions (`on:`) — those can't live
in the reusable workflow — and just delegates the job body.

`your-project/.github/workflows/test.yml`:

```yaml
name: Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    uses: czerwonk/go-ci/.github/workflows/test.yml@v1
```

`your-project/.github/workflows/lint.yml`:

```yaml
name: Lint

on:
  push:
  pull_request:

jobs:
  lint:
    uses: czerwonk/go-ci/.github/workflows/lint.yml@v1
```

`your-project/.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    tags: ["v*.*.*"]

jobs:
  release:
    uses: czerwonk/go-ci/.github/workflows/release.yml@v1
```

`your-project/.github/workflows/docker.yml`:

```yaml
name: Docker

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]

jobs:
  docker:
    uses: czerwonk/go-ci/.github/workflows/docker-publish.yml@v1
```

That publishes to GHCR only — the right default for new projects. A legacy
project that still needs Docker Hub adds the input and secrets:

```yaml
jobs:
  docker:
    uses: czerwonk/go-ci/.github/workflows/docker-publish.yml@v1
    with:
      dockerhub-image: czerwonk/your-project
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKER_HUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
```

`your-project/.github/workflows/helm.yml` (only if the project ships a Helm
chart):

```yaml
name: Release Helm Chart

on:
  push:
    branches: [main]
    paths: ["charts/**"]
  workflow_dispatch:

jobs:
  helm:
    uses: czerwonk/go-ci/.github/workflows/helm-release.yml@v1
    with:
      chart-path: charts/your-chart
      pages-url: https://your-org.github.io/your-project
```
