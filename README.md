# ewws-workflows

Reusable GitHub Actions workflows. Each one is exposed via `workflow_call`, so a
consumer repository calls it with a single `uses:` entry.

`@v1` is the stable ref; `@main` is the rolling tip.

## Workflows

| Workflow | Purpose | Inputs |
| --- | --- | --- |
| `test-pr-title.yml` | Conventional-Commits pull-request-title gate. | `types` |
| `release-please.yml` | Cuts release pull requests and tags from Conventional Commits. Outputs `released` and `tag-name`. | `release-type`, `package-name`, `config-file`, `manifest-file` |
| `build-and-push.yml` | Docker build (amd64 + arm64 by default), pushed to `ghcr.io` under the lower-cased `image-name`; never pushes `latest`. Outputs `sha-tag`, `image`, `tags`, `version`, `digest`. | `image-name` (required), `image-ownership-mode`, `dockerfile`, `context`, `platforms`, `push`, `runner`, `cache`, `submodules`, `timeout-minutes`, `build-args`, `extra-tags`; optional secret `gh-pat`. |
| `write-registry-tag.yml` | Writes `tags/<slug>/<stage>.yaml` in `ewws-apps-registry` through a pull request it merges itself. Outputs `pr-url` and `merge-sha`. | `slug`, `stage` (required), `backend-tag`, `frontend-tag`, `registry-repo`, `app-id`, `merge-timeout-minutes` |
| `write-registry-secrets.yml` | Checks the caller's SealedSecrets against `sops.mac` of the SOPS source and copies them to `secrets/<slug>/<stage>/` in `ewws-apps-registry` through a pull request it merges itself. Never seals, no Azure credentials. Outputs `pr-url` and `merge-sha`. | `slug`, `stage` (required), `sealed-dir`, `sops-file`, `registry-repo`, `app-id`, `merge-timeout-minutes` |
| `go-lint.yml` | `golangci-lint` v2 for Go modules. | `go-version`, `golangci-lint-version`, `working-directory` |
| `claude-code-review.yml` | One Claude review per non-draft pull request, triggered by a human or an `allowed-bots` App; comments only. Guidelines are read from the base branch, out of the pull request's reach. | `runner`, `timeout-minutes`, `guidelines`, `extra-instructions`, `ci-workflow`, `allowed-bots`, `model` |
| `platform-check.yml` | App-platform guard rails: no committed `.env`, no binary SOPS files, `*.sops.yaml` only on the vibe key, only SealedSecrets under `deploy/sealed/`, no cluster/SOPS credentials in CI, no plaintext secrets (gitleaks). | `runner`, `extra-forbidden-refs`, `scan-history` |
| `trivy.yml` | Trivy filesystem scan (vulnerable dependencies, secrets, optionally misconfigurations). Fails the job on findings, or keeps one tracking issue up to date. | `runner`, `scanners`, `severity`, `skip-dirs`, `fail-on-findings`, `tracking-issue` |

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
```

`secrets: inherit` is optional here: the workflow logs in to ghcr.io with
`GITHUB_TOKEN`, which a called workflow always has, with the permissions the
caller grants. The only secret it reads is the optional `gh-pat` below.

**Ein Image gehört genau einem Repository. Nur dessen CI pusht es.** (An image
belongs to exactly one repository, and only that repository's CI pushes it.)

- The name is `ghcr.io/meko-tech/<repo>/<component>`; a repository with a
  single image uses `ghcr.io/meko-tech/<repo>`.
- `<repo>` is the repository's current GitHub name, underscores kept:
  `${{ github.event.repository.name }}`, not the local directory name, an old
  image name or a registry slug. `build-and-push.yml` writes the name in lower
  case itself, so that expression works for a repository whose name has
  capitals too; the ownership check, the tags, the cache ref and the push all
  use the lower-cased name, and the `image` output returns it.
- Slash, not hyphen: `<repo>-backend` collides between repositories such as
  `mw_ruestprotokolle_mfmw` and `mw_ruestprotokolle_mfmw_split`.
- No generic names such as `app-backend`, no `latest` tag:
  `build-and-push.yml` never pushes `latest` (`flavor: latest=false`, and
  `extra-tags` refuses it).
- Images are pushed only through this workflow, with `GITHUB_TOKEN` and
  `permissions: packages: write`, not with the org PAT `EWWS_GITHUB_TOKEN`.

The job first checks that the lower-cased `image-name` is a plain OCI image
name: a host (optional port) and at least one path segment, no tag, no
digest. Anything else (`,`, `=`, `:tag`, `@sha256:…`, whitespace) fails the
job in either `image-ownership-mode`; a comma would otherwise let
metadata-action split the value into a second, foreign image. It then checks
the name against the caller repository (`github.repository`). `image-ownership-mode` decides what
a name outside `ghcr.io/meko-tech/<repo>` does: `warn` (default) writes an
`image ownership` warning and builds anyway, `error` fails the job.

Tags are the metadata-action rules `type=ref,event=branch`,
`type=ref,event=pr`, `type=semver` (`{{version}}`, `{{major}}.{{minor}}`,
`{{major}}`) and `type=sha,prefix=sha-`, plus one `type=raw` per line of
`extra-tags`. A line of `extra-tags` is a raw tag value (`1.5.0-6c98a25`, not
`type=…`); blank lines are skipped, and a value that is not a Docker tag
(`[A-Za-z0-9_][A-Za-z0-9_.-]{0,127}`) or is `latest` fails the job.
`build-args` (one `KEY=VALUE` per line) goes to docker/build-push-action
unchanged.

A release build that also tags `<semver>-<sha7>` and bakes the version into
the image, and a deploy job that picks up what was pushed:

```yaml
on:
  push:
    tags: ["v*"]

permissions:
  contents: read
  packages: write

jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.v.outputs.version }}
      tag: ${{ steps.v.outputs.tag }}
    steps:
      - id: v
        env:
          REF_NAME: ${{ github.ref_name }}
          SHA: ${{ github.sha }}
        run: |
          version="${REF_NAME#v}"
          echo "version=${version}" >>"$GITHUB_OUTPUT"
          echo "tag=${version}-${SHA::7}" >>"$GITHUB_OUTPUT"

  build-backend:
    needs: version
    uses: MeKo-Tech/ewws-workflows/.github/workflows/build-and-push.yml@v1
    with:
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}/backend
      build-args: |
        VERSION=${{ needs.version.outputs.version }}
      extra-tags: |
        ${{ needs.version.outputs.tag }}

  deploy:
    needs: build-backend
    runs-on: ubuntu-latest
    steps:
      - env:
          IMAGE: ${{ needs.build-backend.outputs.image }}
          SHA_TAG: ${{ needs.build-backend.outputs.sha-tag }}
          TAGS: ${{ needs.build-backend.outputs.tags }}
          DIGEST: ${{ needs.build-backend.outputs.digest }}
        run: |
          echo "deploying ${IMAGE}:${SHA_TAG} (${DIGEST})"
          printf 'pushed: %s\n' "$TAGS"
```

The outputs:

| Output | Value |
| --- | --- |
| `sha-tag` | The `sha-<7>` tag that was pushed (e.g. `sha-c69034c`); empty when nothing was pushed (`push: false` or a pull request). |
| `image` | `image-name` in lower case, as built and pushed (e.g. `ghcr.io/meko-tech/ipo-foo/backend`). |
| `tags` | Every tag metadata-action computed, one `<image>:<tag>` per line. Also set when nothing was pushed; check `sha-tag` for that. |
| `version` | metadata-action's `version`, its highest-priority tag: `1.2.3` on a semver tag push, the branch name as metadata-action writes it on a branch push (characters outside `[A-Za-z0-9._-]` become `-`, so `deploy/foo` gives `deploy-foo`), `pr-123` on a pull request. On a tag push that is not semver no rule names the tag (there is no `type=ref,event=tag`; adding it would push an extra `v1.2.3` tag on every release of existing callers), so `version` is the first `extra-tags` value, or else the `sha-<7>` tag, never the git tag. |
| `digest` | The image digest (`sha256:…`) docker/build-push-action reports. |

A Dockerfile that fetches private Go modules (for example `ewws-auth`) gets a
token through the optional `gh-pat` secret. It reaches BuildKit as secret
`gh_pat`. When the caller passes no `gh-pat`, `/run/secrets/gh_pat` does not
exist.

Rule for the Dockerfile: the token never goes into a file. `git config
--global` would write it into `~/.gitconfig` of that layer, and the layer ends
up in the build cache and possibly in the image. Pass the URL rewrite through
`GIT_CONFIG_*` environment variables scoped to the one command that needs it,
as below.

With `gh-pat` set, the workflow exports the build cache (`cache: gha` or
`registry`) with `mode=min`: only the layers of the final image, not the
builder stages. A builder stage that ran `go mod download` against private
modules holds their source, and `mode=max` would put that into
`<image-name>:buildcache` or the Actions cache, where anyone who can read that
package or cache can read it. The builder stages stay on the runner, so they are
rebuilt on a cold runner. Without `gh-pat` the cache keeps `mode=max`.

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
    uses: MeKo-Tech/ewws-workflows/.github/workflows/build-and-push.yml@v1
    with:
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}/backend
      dockerfile: ./backend/deployment/Dockerfile
    secrets:
      gh-pat: ${{ secrets.EWWS_GITHUB_TOKEN }}
```

The build runs on `ubuntu-latest` unless `runner` names one of our own pools.
Use one that has Docker: `ewws-arc-ci-onprem`, the pool the image builds of the
other repositories run on. `ewws-arc-ci-small` has no Docker, and the deploy pools are
for deploy jobs only.

```yaml
jobs:
  build-backend:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/build-and-push.yml@v1
    with:
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}/backend
      dockerfile: ./backend/Dockerfile
      context: ./backend
      runner: ewws-arc-ci-onprem
      platforms: linux/amd64
      cache: registry
      timeout-minutes: 30
```

On our own runners the workflow hands buildx the pool's registry mirror and
its pinned BuildKit image, so base images come from the in-cluster mirror
instead of Docker Hub. Three things differ from a GitHub-hosted build:

- **`platforms: linux/amd64` is required.** The pools are amd64 only, and
  QEMU would register its emulators with the kernel of a node that other pods
  share. The job fails in its first step otherwise.
- **`cache: registry`** keeps the layer cache as `<image-name>:buildcache`
  next to the image. `gha` works too, but sends every layer to GitHub's cache
  over the pool's uplink and counts against the repository's cache quota there.
  Pull requests read the registry cache and do not write it.
- **`timeout-minutes`** defaults to GitHub's 360. Pass a real one: a hung
  build keeps a runner slot until then.

`submodules` goes to `actions/checkout` unchanged (`false`, `true`,
`recursive`) for a Dockerfile that copies a submodule.

#### Pull access

Every cluster (chungus and DigitalOcean) pulls with the org account
`mekoAutomation` through the `regcred` pull secret, which needs only
`read:packages`. A new package needs no manual grant:

- The first `GITHUB_TOKEN` push from this workflow creates the package linked
  to the repository, with "Inherit access from source repository" on.
- The org base permission is `write`, so `mekoAutomation`, an org member, can
  read every repository and therefore every package that inherits from one.
  Pulling needs only `read`; setting the base permission to "No permission"
  without another grant breaks this for every inheriting package.
- A package that does not inherit (direct members only) needs the team
  `@ewws` with Read; `mekoAutomation` is in that team.
- A Dockerfile whose `FROM` is a private package of another repository needs
  "Manage Actions access: Read" for the building repository in that package's
  settings. `GITHUB_TOKEN` reads no other package without it.

Renaming an image that something deploys: (1) the workflow pushes the new
name, (2) the consumer switches (registry claim or Helm values), (3) the old
package is deleted after sign-off. There is no PAT mirror step any more.

#### The org PAT has no `write:packages`

Since 25.11.2026 `EWWS_GITHUB_TOKEN` (`mekoAutomation`) cannot push to
ghcr.io. A `docker login ghcr.io` with it followed by a push fails; push
through this workflow with `GITHUB_TOKEN` instead. In this repository the
PAT appears only as the optional `gh-pat` input of `build-and-push.yml` for
private Go modules (BuildKit secret `gh_pat`): `release-please.yml` uses
`GITHUB_TOKEN`, and `write-registry-tag.yml` uses the `ewws-registry-tagger`
App. Callers still use the PAT for their own release-please workflows and
checkouts of other private repositories.

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
`merge-timeout-minutes` (default 25; a failed GitHub API call while polling is
retried with backoff) and fails the job when the checks fail or
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
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}/backend
      context: backend
    secrets: inherit # optional for build-and-push.yml

  frontend:
    uses: MeKo-Tech/ewws-workflows/.github/workflows/build-and-push.yml@v1
    with:
      image-name: ghcr.io/meko-tech/${{ github.event.repository.name }}/frontend
      context: frontend
    secrets: inherit # optional for build-and-push.yml

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
`id-token: write`** (job-level `permissions:` as in the example, or workflow-level); without it the run fails
to start, and the registry rejects a tagger pull request without a token.

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
| `merge-timeout-minutes` | `25` | |

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

### `trivy.yml`

Scans the repository with [Trivy](https://trivy.dev/): vulnerable dependencies
in lockfiles and `go.mod`, plaintext secrets and, with `misconfig` in
`scanners`, Dockerfiles and Helm charts. Only `CRITICAL` and `HIGH` findings
count by default, and only vulnerabilities that have a fix.

Findings surface in one of two ways:

- **Gate** (default): the job fails. Use it for `pull_request`.
- **Tracking issue** (`tracking-issue: true`): the job stays green and keeps one
  issue labelled `trivy` up to date. The issue is created on the first finding,
  rewritten on every run and closed when a scan is clean. Use it for
  `schedule`. Our private repositories have no code scanning, so there is no
  Security tab to report to.

Trivy is a pinned release binary, verified against its checksum. To accept a
finding, add its ID to `.trivyignore` in the repository root, with a comment
that says why.

```yaml
on:
  pull_request:
  schedule:
    - cron: "0 4 * * 1"
  workflow_dispatch:

permissions: {}

jobs:
  # Pull requests: a read-only gate.
  scan:
    if: github.event_name == 'pull_request'
    permissions:
      contents: read
    uses: MeKo-Tech/ewws-workflows/.github/workflows/trivy.yml@v1
    with:
      runner: ewws-arc-ci-hetzner

  # Weekly and manual runs: keep the tracking issue up to date.
  report:
    if: github.event_name != 'pull_request'
    permissions:
      contents: read
      issues: write
    uses: MeKo-Tech/ewws-workflows/.github/workflows/trivy.yml@v1
    with:
      runner: ewws-arc-ci-hetzner
      scanners: vuln,secret,misconfig
      tracking-issue: true
```

The workflow declares no permissions of its own: it runs with what the calling
job grants. Only the tracking issue needs `issues: write`, so the pull-request
job gets by with `contents: read`. A repository that already has findings lists
them in `.trivyignore` rather than switching the gate off with
`fail-on-findings: false`, which would also silence the secret scanner.
