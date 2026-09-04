# ewws-workflows

Reusable GitHub Actions workflows used across all `ewws`-tagged MeKo-Tech repos. The goal is **one source of truth** for the CI conventions: Conventional-Commits PR titles, release-please for semver, build-and-push for ghcr.io images, etc.

## Why

Every `ewws-*` repo used to copy-paste its own version of these workflows. That meant:

- Five different release-please configs, three of them broken
- Image-tag formats drifted (`sha-<7>` here, `<sha>-<branch>` there)
- PR-title regex differed by repo
- Whenever we wanted to bump a pinned action SHA, we had to PR it into every repo

This repo centralizes the workflows. Each is exposed via `workflow_call` so a consumer repo just needs a one-line `uses:` entry. Pin to a tag (`@v1`) for prod-stable; pin to `@main` for the rolling tip.

## Available workflows

| Workflow | Purpose | Inputs |
| --- | --- | --- |
| `test-pr-title.yml` | Conventional-Commits PR-title gate. Required because `release-please` keys off the title prefix. | none |
| `release-please.yml` | Cuts release PRs + tags from Conventional Commits. Default `release-type: simple` reads version from `version.txt`. | `release-type`, `version-file` |
| `build-and-push.yml` | Multi-arch (amd64+arm64) Docker build, pushes to `ghcr.io`. Tags: branch name, PR ref, semver, `sha-<short>`. | `image-name`, `dockerfile`, `platforms` |
| `go-lint.yml` | `golangci-lint v2` for Go modules. | `go-version` |
| `claude-code-review.yml` | Claude reviews every non-draft pull request once, and comments only — no write access to the repository. | `runner`, `timeout-minutes`, `guidelines`, `extra-instructions`, `ci-workflow` |

## Quick-start (consumer repo)

Replace the per-repo `ci.yml`, `release.yml`, etc. with:

```yaml
name: ci

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test-pr-title:
    if: github.event_name == 'pull_request'
    uses: MeKo-Tech/ewws-workflows/.github/workflows/test-pr-title.yml@main

  go-lint:
    if: hashFiles('go.mod') != ''
    uses: MeKo-Tech/ewws-workflows/.github/workflows/go-lint.yml@main
    with:
      go-version: "1.25.x"
```

And in `release.yml`:

```yaml
name: release

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write
  packages: write

jobs:
  release-please:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/release-please.yml@main

  build-and-push:
    if: needs.release-please.outputs.released == 'true' || github.ref == 'refs/heads/main'
    needs: release-please
    uses: MeKo-Tech/ewws-workflows/.github/workflows/build-and-push.yml@main
    with:
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}
    secrets: inherit
```

## `claude-code-review.yml`

One review per pull request, on `opened` / `ready_for_review` / `reopened`. No
`synchronize`: a push does not buy a second review. Drafts, `release-please--*`
and `dependabot/*` branches are skipped.

The caller declares the permissions — a reusable workflow cannot grant itself
more than it is given — and passes the organisation token:

```yaml
name: claude-code-review

on:
  pull_request:
    types: [opened, ready_for_review, reopened]

permissions:
  contents: read
  pull-requests: write
  issues: read
  id-token: write # the action exchanges the OIDC token for its GitHub App token

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/claude-code-review.yml@main
    with:
      runner: '["ewws-arc-ci-hetzner"]'
      guidelines: AGENTS.md and CLAUDE.md
      ci-workflow: tests.yaml
      extra-instructions: |
        Errors are wrapped with context and never dropped.
        No new dependency without a reason stated in the pull request.
    secrets:
      anthropic-token: ${{ secrets.ANTHROPIC_API_KEY }}
```

`extra-instructions` is where a repository's red lines go — the rules that get
buried among smaller findings unless the review is told to name them. Keep it to
what is specific: the generic half (correctness, scope, reuse, security, test
coverage, `file:line` references, blocker / should-fix / nice-to-have) is already
in the workflow.

Two things to know before adopting it:

- **One token, one rate limit.** `ANTHROPIC_API_KEY` is a single organisation
  secret, shared with the `pr-auto-rebase` sweeps. Every repository added here
  draws on it.
- **The pull request that adds the caller cannot be reviewed by it.** The action
  validates the caller's workflow file against the copy on the default branch and
  skips itself when the two differ — the guard that stops a pull request from
  rewriting the reviewer that is about to review it. That run goes green having
  posted nothing, and says so in one warning in the middle of the log. It starts
  working on the next pull request.

## Conventions enforced

1. **PR titles use Conventional Commits** (`feat:`, `fix:`, `chore:`, …). Anything else fails the `test-pr-title` check.
2. **Version comes from `version.txt`** at repo root. `release-please` bumps it; that file is the single source of truth in the build pipeline.
3. **Image tags include `sha-<short>`** for image-updater compatibility. The platform-UI drift detector treats `sha-<short>` as the git-commit prefix (see [`ewws-platform-ui/internal/drift/compare.go`](https://github.com/MeKo-Tech/ewws-platform-ui/blob/main/internal/drift/compare.go)).
4. **Images push to `ghcr.io/meko-tech/<repo>`** (lowercase). The chungus Kyverno policy only allows pulls from this registry plus a small allow-list.

## Migration plan

The intent is to migrate every `ewws`-tagged repo (and every tenant claim's source repo) to use these reusable workflows. Tracked in MEK-XXXX (TBD). Rough order:

1. `ewws-platform-ui` — meta-platform, low blast radius
2. `ewws-microscope`, `ewws-tailscale`, `ewws-monitoring` — already-archived; skip
3. `mekorp-backend`, `mekorp-webclient` — bigger, more workflows touch
4. Tenant repos (`radialforce-plotter`, …) — one per onboarding wave
