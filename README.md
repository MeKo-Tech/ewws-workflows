# ewws-workflows

Reusable GitHub Actions workflows. Each one is exposed via `workflow_call`, so a
consumer repository calls it with a single `uses:` entry.

`@v1` is the stable ref; `@main` is the rolling tip.

## Workflows

| Workflow | Purpose | Inputs |
| --- | --- | --- |
| `test-pr-title.yml` | Conventional-Commits pull-request-title gate. | `types` |
| `release-please.yml` | Cuts release pull requests and tags from Conventional Commits. Outputs `released` and `tag-name`. | `release-type`, `package-name`, `config-file`, `manifest-file` |
| `build-and-push.yml` | Multi-arch (amd64 + arm64) Docker build, pushed to `ghcr.io`. Outputs `sha-tag`. | `image-name` (required), `dockerfile`, `context`, `platforms`, `push`; optional secret `gh-pat` |
| `write-registry-tag.yml` | Writes `tags/<slug>/<stage>.yaml` in `ewws-apps-registry` through a pull request it merges itself. Outputs `pr-url` and `merge-sha`. | `slug`, `stage` (required), `backend-tag`, `frontend-tag`, `registry-repo`, `app-id`, `merge-timeout-minutes` |
| `write-registry-secrets.yml` | Checks the caller's SealedSecrets against `sops.mac` of the SOPS source and copies them to `secrets/<slug>/<stage>/` in `ewws-apps-registry` through a pull request it merges itself. Never seals, no Azure credentials. Outputs `pr-url` and `merge-sha`. | `slug`, `stage` (required), `sealed-dir`, `sops-file`, `registry-repo`, `app-id`, `merge-timeout-minutes` |
| `go-lint.yml` | `golangci-lint` v2 for Go modules. | `go-version`, `golangci-lint-version`, `working-directory` |
| `claude-code-review.yml` | One Claude review per non-draft pull request, triggered by a human or an `allowed-bots` App; comments only. Guidelines are read from the base branch, out of the pull request's reach. | `runner`, `timeout-minutes`, `guidelines`, `extra-instructions`, `ci-workflow`, `allowed-bots`, `model` |
| `platform-check.yml` | App-platform guard rails: no committed `.env`, no binary SOPS files, `*.sops.yaml` only on the vibe key, only SealedSecrets under `deploy/sealed/`, no cluster/SOPS credentials in CI, no plaintext secrets (gitleaks). | `runner`, `extra-forbidden-refs`, `scan-history` |

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
      go-version: "1.27"
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

A Dockerfile that fetches private Go modules (for example `ewws-auth`) gets a
token through the optional `gh-pat` secret. It reaches BuildKit as secret
`gh_pat`. Keep the token out of every file: `git config --global` would write it
into `~/.gitconfig` of that layer, and `cache-to: mode=max` exports
intermediate layers to the Actions cache. Pass the URL rewrite through
command-scoped Git environment variables instead. When the caller passes no
`gh-pat`, `/run/secrets/gh_pat` does not exist.

```dockerfile
RUN --mount=type=secret,id=gh_pat \
	GIT_CONFIG_COUNT=1 \
	GIT_CONFIG_KEY_0="url.https://x-access-token:$(cat /run/secrets/gh_pat)@github.com/.insteadOf" \
	GIT_CONFIG_VALUE_0="https://github.com/" \
	go mod download
```

```yaml
jobs:
  build-backend:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/build-and-push.yml@main
    with:
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}-backend
      dockerfile: ./backend/deployment/Dockerfile
    secrets:
      gh-pat: ${{ secrets.EWWS_GITHUB_TOKEN }}
```

`sha-tag` is the `sha-<7>` tag that was pushed (e.g. `sha-c69034c`); it is
empty when nothing was pushed (`push: false` or a pull request).

### `write-registry-tag.yml`

Records which image an app runs on a stage: it writes
`tags/<slug>/<stage>.yaml` in `MeKo-Tech/ewws-apps-registry`,

```yaml
imageTags:
  backend: "sha-c758f9c"
  frontend: "sha-c758f9c"
```

and nothing else. A component whose input is empty keeps the value it already
has; an unchanged file is a successful no-op. The change goes through a pull
request on a fresh `tags/<slug>-<stage>-<run>-<attempt>` branch, which the
workflow squash-merges itself once the registry's required checks
(`validate-apps`, `tag-scope`) are green. It retries the merge for
`merge-timeout-minutes` (default 10) and fails the job when the checks fail or
the time runs out. A pull request that conflicts with or falls behind the
registry's default branch is re-applied on top of it: force-pushed when the
content stays the same (same pull request and token), else re-opened as a new
pull request with a new token (`-r<n>` branch). Only commits the run pushed
itself are merged (`--match-head-commit`). Runs for the same slug and stage
are serialised (`registry-tag-<slug>-<stage>`, within the caller
repository).

```yaml
on:
  push:
    branches: [main]
  release:
    types: [published]

permissions:
  contents: read
  packages: write

jobs:
  backend:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/build-and-push.yml@v1
    with:
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}-backend
      context: backend
    secrets: inherit

  frontend:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/build-and-push.yml@v1
    with:
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}-frontend
      context: frontend
    secrets: inherit

  tag-staging:
    needs: [backend, frontend]
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      id-token: write # OIDC token for the registry's tag-scope check
    uses: MeKo-Tech/ewws-workflows/.github/workflows/write-registry-tag.yml@v1
    with:
      slug: materialorder
      stage: staging
      backend-tag: ${{ needs.backend.outputs.sha-tag }}
      frontend-tag: ${{ needs.frontend.outputs.sha-tag }}
    secrets: inherit

  tag-prod:
    needs: [backend, frontend]
    if: github.event_name == 'release'
    permissions:
      contents: read
      id-token: write # OIDC token for the registry's tag-scope check
    uses: MeKo-Tech/ewws-workflows/.github/workflows/write-registry-tag.yml@v1
    with:
      slug: materialorder
      stage: prod
      backend-tag: ${{ needs.backend.outputs.sha-tag }}
      frontend-tag: ${{ needs.frontend.outputs.sha-tag }}
    secrets: inherit
```

A repository with only one component passes only that tag. The caller's
`GITHUB_TOKEN` is used read-only; everything in the registry is done with a
token of the GitHub App below, scoped to the registry repository alone.

Prerequisites:

- The GitHub App **`ewws-registry-tagger`**, installed on
  `MeKo-Tech/ewws-apps-registry` only, with the repository permissions
  *Contents: write*, *Pull requests: write*, *Checks: read* and
  *Commit statuses: read*. It is a bypass actor (mode `pull_request`) for the
  approval rule of the registry's default branch, but not for the ruleset that
  requires `validate-apps` and `tag-scope`.
- The org variable **`EWWS_REGISTRY_TAGGER_APP_ID`** (the App's ID; or pass
  `app-id`) and the org secret **`EWWS_REGISTRY_TAGGER_PRIVATE_KEY`**, both
  visible to the calling repository. `secrets: inherit` passes the secret; a
  caller without `inherit` passes it as `registry-tagger-private-key`.
- A claim `apps/<slug>.yaml` in the registry; the workflow refuses to tag an
  unregistered app.

The pull request body (and the commit body) carries a line
`Source-Repository: <owner>/<repo>` next to `Source run: <run url>`; the
registry's `tag-scope` check verifies it against the claim's `repo.url`.

That line is only text: anyone who can run a workflow in an org repository can
mint a tagger token and write any repository there. The proof is the line
`OIDC-Token: <jwt>` in the pull request body — a GitHub Actions OIDC token
requested right before the pull request is opened, with the audience
`ewws-apps-registry:<tags|secrets>/<slug>/<stage>@<sha>`, where `<sha>` is the
git object id of what the pull request writes (blob of the tag file, tree of
the secrets directory). `tag-scope` verifies GitHub's signature, that the
audience matches the pull request's head, and that the token was issued to the
claim's `repo.url`, by this workflow (`job_workflow_ref`), for this pull
request's branch (run id), within 15 minutes before the pull request was
opened, from an event/ref allowed for the stage. A commit someone else pushes
onto the branch no longer matches the token. **The calling job must grant
`id-token: write`** (job-level `permissions:` as in the example, or
workflow-level); without it the workflow opens the pull request without a
token, which the registry accepts with a warning only until the rollout is
complete.

### `write-registry-secrets.yml`

Publishes an app's SealedSecrets for one stage: it replaces
`secrets/<slug>/<stage>/` in `MeKo-Tech/ewws-apps-registry` with the files of
the caller's `deploy/sealed/<stage>/` (files gone there are deleted in the
registry too), where the vibe-tenants ApplicationSet picks them up.

**CI never seals and needs no Azure credentials.** Secret values live
SOPS-encrypted in `secrets/<stage>.sops.yaml` (vibe key `sops-key-vibe` in
`meko-vibe-vault`) and are sealed locally with `just secrets-seal`, which also
stamps each SealedSecret with the annotation `platform.meko.dev/sops-mac` — the
`sops.mac` of the source it was sealed from. Before copying anything the
workflow checks, without decrypting:

- `secrets/<stage>.sops.yaml` exists and has a `sops.mac`; `deploy/sealed/<stage>/`
  holds at least one file and only `<name>.yaml` files,
- every file is one `bitnami.com/v1alpha1` `SealedSecret` with
  `metadata.namespace: tenant-<slug>-<stage>`, a file name equal to
  `metadata.name` (`^[a-z][a-z0-9-]{1,40}[a-z0-9]$`), non-empty
  `encryptedData` and no `sealedsecrets.bitnami.com/cluster-wide` or
  `/namespace-wide` annotation (scope strict),
- its `platform.meko.dev/sops-mac` equals `sops.mac`. Otherwise the SOPS file changed
  after sealing and the job fails with
  `secrets geändert, aber nicht neu versiegelt – just secrets-seal laufen lassen`.

If the caller ships the template recipe `scripts/platform-secrets.sh`, its
`check` (yq only, no decryption) runs first as well.
A stage without `secrets/<stage>.sops.yaml` and without sealed files is
skipped with a notice, so the job can be wired unconditionally. Removing the
last secret of a stage from the registry is a human PR.

The rest works like `write-registry-tag.yml`: same GitHub App, same
prerequisites (including the claim `apps/<slug>.yaml`), a pull request
`build(secrets): <slug> <stage>` on a fresh
`secrets/<slug>-<stage>-<run>-<attempt>` branch, squash-merged once
`validate-apps` and `tag-scope` are green, re-applied on a conflict, runs
serialised per slug and stage (`registry-secrets-<slug>-<stage>`). An unchanged
directory is a successful no-op. The pull request body reads

```text
Written by the `write-registry-secrets` workflow.

Source-Repository: <owner>/<repo>
Source run: https://github.com/<owner>/<repo>/actions/runs/<id>
OIDC-Token: <jwt>
```

`tag-scope` limits a tagger pull request to `tags/<slug>/` and
`secrets/<slug>/` of a single slug and checks `Source-Repository` and the OIDC
token against the claim's `repo.url` (see above; the calling job needs
`id-token: write`).

| Input | Default | |
| --- | --- | --- |
| `slug` | — (required) | App slug. |
| `stage` | — (required) | `staging` or `prod`. |
| `sealed-dir` | `deploy/sealed/<stage>` | SealedSecrets in the caller repository. |
| `sops-file` | `secrets/<stage>.sops.yaml` | SOPS source they were sealed from. |
| `registry-repo` | `MeKo-Tech/ewws-apps-registry` | |
| `app-id` | `vars.EWWS_REGISTRY_TAGGER_APP_ID` | |
| `merge-timeout-minutes` | `10` | |

Staging is published on a push to `main`, prod on a release — only when the
sealed files changed:

```yaml
on:
  push:
    branches: [main]
    paths: ["deploy/sealed/**"]
  release:
    types: [published]

permissions:
  contents: read

jobs:
  secrets-staging:
    if: github.event_name == 'push'
    permissions:
      contents: read
      id-token: write # OIDC token for the registry's tag-scope check
    uses: MeKo-Tech/ewws-workflows/.github/workflows/write-registry-secrets.yml@main
    with:
      slug: materialorder
      stage: staging
    secrets: inherit

  secrets-prod:
    if: github.event_name == 'release'
    permissions:
      contents: read
      id-token: write # OIDC token for the registry's tag-scope check
    uses: MeKo-Tech/ewws-workflows/.github/workflows/write-registry-secrets.yml@main
    with:
      slug: materialorder
      stage: prod
    secrets: inherit
```

A release always runs the prod job; with unchanged files it ends as a no-op
without a pull request. `paths` cannot filter a `release` event.

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

### `platform-check.yml`

Guard rails for app repositories on the MeKo app platform. The job fails on

- committed dotenv files (`.env`, `.env.local`, `prod.env`, …); allowed are
  `*.example`, `*.sample`, `*.template`, `*.dist` and SOPS-encrypted `*.sops.env`,
- SOPS files in the binary format (`"data": "ENC[…]"`),
- committed `*.sops.yaml` files not encrypted with exactly the vibe key: every
  `sops.azure_kv[]` entry must be `https://meko-vibe-vault.vault.azure.net`
  with key `sops-key-vibe`. A `mekorp-vault*` URL, a file without any
  `azure_kv` entry, any `pgp`/`age`/`kms`/`gcp_kms`/`hc_vault` key or an
  unencrypted value fails — vibe apps may only use
  `meko-vibe-vault/sops-key-vibe`, never `mekorp-vault*`. Only metadata is
  read; nothing is decrypted,
- files under `deploy/sealed/` that are not one `bitnami.com/v1alpha1`
  `SealedSecret` each, or that carry an unencrypted `spec.template.data`,
- references to `EWWS_ON_PREM_KUBECONFIG`, `AZURE_EWWS_KEY_VAULT_CLIENT_SECRET`
  or `AZURE_EWWS_KEY_VAULT_CREDS` under `.github/` (plus `extra-forbidden-refs`),
- plaintext secrets found by gitleaks (pinned, checksum-verified). It scans the
  working tree and, on a pull request, every commit of the pull request;
  `scan-history: true` scans the whole history instead. A `.gitleaks.toml` in
  the repository root is honoured. In `*.sops.yaml` files and
  SealedSecrets only ciphertext-shaped lines are ignored (`ENC[AES256_GCM,…]`,
  sops `enc:`, `Ag…` values); comments, annotations and history are scanned.

Secret values belong SOPS-encrypted in `secrets/<stage>.sops.yaml` and sealed
with `just secrets-seal` into `deploy/sealed/<stage>/`; see
`write-registry-secrets.yml`.

All checks run even when an earlier one failed, so a single run lists every
finding.

```yaml
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  platform-check:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/platform-check.yml@main
```
