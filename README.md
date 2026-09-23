# ewws-workflows

Reusable GitHub Actions workflows. Each one is exposed via `workflow_call`, so a
consumer repository calls it with a single `uses:` entry.

`@v1` is the stable ref; `@main` is the rolling tip.

## Workflows

| Workflow | Purpose | Inputs |
| --- | --- | --- |
| `test-pr-title.yml` | Conventional-Commits pull-request-title gate. | `types` |
| `release-please.yml` | Cuts release pull requests and tags from Conventional Commits. Outputs `released` and `tag-name`. | `release-type`, `package-name`, `config-file`, `manifest-file` |
| `build-and-push.yml` | Multi-arch (amd64 + arm64) Docker build, pushed to `ghcr.io`. | `image-name` (required), `dockerfile`, `context`, `platforms`, `push` |
| `go-lint.yml` | `golangci-lint` v2 for Go modules. | `go-version`, `golangci-lint-version`, `working-directory` |
| `claude-code-review.yml` | One Claude review per non-draft pull request, triggered by a human or an `allowed-bots` App; comments only. Guidelines are read from the base branch, out of the pull request's reach. | `runner`, `timeout-minutes`, `guidelines`, `extra-instructions`, `ci-workflow`, `allowed-bots`, `model` |

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

### `claude-code-review.yml`

```yaml
on:
  pull_request:
    types: [opened, ready_for_review, reopened]

permissions:
  contents: read
  pull-requests: write
  issues: read
  actions: read # the action reads the caller's own workflow run
  id-token: write # the action exchanges the OIDC token for its GitHub App token

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/claude-code-review.yml@main
    with:
      guidelines: AGENTS.md and CLAUDE.md
      ci-workflow: tests.yaml
      extra-instructions: |
        Errors are wrapped with context and never dropped.
    secrets:
      anthropic_token: ${{ secrets.ANTHROPIC_API_KEY }}
```

The job runs on `ubuntu-latest` unless `runner` passes a different `runs-on`
value as a JSON array. Skipped: drafts, `release-please--*` and `dependabot/*`
branches, events triggered by a bot unless `allowed-bots` names some, and
pull requests from a fork — GitHub withholds the organisation secret from a
fork, so the review could only fail there.

**The review reads its guidelines from the base branch.** The job checks the
base revision out a second time, into `.review-base/`, and the session is told
to take `guidelines` from there. Otherwise a pull request could edit the file
that instructs the reviewer about to read it — the hole the action's own
workflow-file guard closes for the workflow, left open for everything the
workflow points at.

The review runs on `claude-opus-5-5` unless `model` names another; an empty
`model` leaves the choice to Claude Code's default.

A review session that ends without publishing anything fails the job instead of
going green. On a private repository its transcript is kept for 14 days as the
`claude-review-output-<pr>-<attempt>` artifact of the run; on a public one it
is not uploaded, because the action hides that output for a reason.
