# ewws-workflows

Reusable GitHub Actions workflows. Each one is exposed via `workflow_call`, so a
consumer repository calls it with a single `uses:` entry.

This is the `v1` line — the stable ref. `@main` carries the rolling tip and any
workflow added after v1.

## Workflows

| Workflow | Purpose | Inputs |
| --- | --- | --- |
| `test-pr-title.yml` | Conventional-Commits pull-request-title gate. | `types` |
| `release-please.yml` | Cuts release pull requests and tags from Conventional Commits. Outputs `released` and `tag-name`. | `release-type`, `package-name`, `config-file`, `manifest-file` |
| `build-and-push.yml` | Multi-arch (amd64 + arm64) Docker build, pushed to `ghcr.io`. | `image-name` (required), `dockerfile`, `context`, `platforms`, `push` |
| `go-lint.yml` | `golangci-lint` v2 for Go modules. | `go-version`, `golangci-lint-version`, `working-directory` |

## Calling them

A reusable workflow can only downgrade the permissions it is called with, never
raise them, so the caller declares everything the job needs.

### `test-pr-title.yml`

```yaml
permissions:
  pull-requests: read

jobs:
  test-pr-title:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/test-pr-title.yml@v1
```

### `go-lint.yml`

```yaml
permissions:
  contents: read

jobs:
  go-lint:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/go-lint.yml@v1
    with:
      go-version: "1.25.x"
```

### `release-please.yml`

```yaml
permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/release-please.yml@v1
```

### `build-and-push.yml`

```yaml
permissions:
  contents: read
  packages: write

jobs:
  build-and-push:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/build-and-push.yml@v1
    with:
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}
    secrets: inherit
```
