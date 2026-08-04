<!--
SPDX-FileCopyrightText: 2026 Bonial International GmbH
SPDX-License-Identifier: Apache-2.0
-->

# go-release-demo-cocogitto

**Combo 3** reference implementation of immutable Go releases, using
**cocogitto + GoReleaser**. Part of the
[go-release-demo](https://github.com/bonial-oss/go-release-demo)
evaluation project comparing three release toolchains.

## What this demonstrates

Cutting a versioned Go release where:

- The git tag cannot be moved, deleted, or re-created (tag ruleset).
- The release assets cannot be edited after publication (immutable-releases setting).
- The artifacts are signed by a verifiable workflow identity via **cosign keyless** (Sigstore, OIDC).
- The build has **SLSA L3 provenance attestations** proving how it was built.
- The build is **reproducible**: the same commit produces byte-identical binaries, testable via a rebuild-and-diff CI job.
- The release is **atomic**: the tag only exists if the full build, sign, and publish succeeded — no partial state ever reaches a consumer.

## Toolchain

| Concern | Tool |
|---|---|
| Next version from Conventional Commits | [`cocogitto`](https://docs.cocogitto.io/) (`cog bump --auto --dry-run`) |
| Release notes from Conventional Commits | [`cocogitto`](https://docs.cocogitto.io/) (`cog changelog --at <tag>`) |
| Build, sign, SBOM, release | [`goreleaser`](https://goreleaser.com) |
| Keyless signing | [`cosign`](https://docs.sigstore.dev/cosign/) |
| Build provenance | [`slsa-github-generator`](https://github.com/slsa-framework/slsa-github-generator) |
| SBOM | [`syft`](https://github.com/anchore/syft) (invoked by GoReleaser) |

## Using the released binary

Download for your platform from the [latest release](https://github.com/bonial-oss/go-release-demo-cocogitto/releases/latest):

```bash
# Example: linux/amd64
curl -fsSL -o demo.tar.gz \
  https://github.com/bonial-oss/go-release-demo-cocogitto/releases/download/v0.1.0/demo_0.1.0_linux_amd64.tar.gz
tar xzf demo.tar.gz
./demo version
./demo verify
```

## Verifying the release

See [`docs/verification.md`](docs/verification.md) for cut-and-paste commands
that check the signature, provenance, and reproducibility of any downloaded
archive.

The bundled `demo verify` subcommand does the same check in-binary — the
binary you downloaded proves to you that it's the binary the release
workflow produced.

**Post-release verification is triggered manually.** GitHub's chain-prevention
rule suppresses the `released` event when the release was published by a
workflow-authenticated token, so `verify-release.yaml` will not auto-fire.
After a release publishes, run:

```bash
gh workflow run verify-release.yaml --repo bonial-oss/go-release-demo-cocogitto --ref main --field tag=v0.1.0
```

## How releases are cut

1. Merge a PR to `main` containing at least one `feat:` or `fix:` commit.
2. The `Release` workflow's `propose` job runs `cog bump --auto --dry-run`
   against Conventional Commits since the last tag. If a bump is warranted,
   it proceeds to the atomic release pipeline.
3. Three sequential jobs: **goreleaser** (build, sign, SBOM, create draft
   release, push tag as the last action) → **slsa** (attach SLSA L3
   provenance to the draft) → **promote** (`gh release edit --draft=false`
   — the only write to a published release).
4. If any step fails, no tag exists, no release is visible to consumers.
   Re-dispatching the workflow retries cleanly.

A manual `workflow_dispatch` path is supported for dry-run previews — see
[`.github/workflows/release.yaml`](.github/workflows/release.yaml).

## cocogitto configuration

The repo uses `cog.toml` with `ignore_merge_commits = true` and
`disable_bump_commit = true`:

- `ignore_merge_commits = true` lets cocogitto see conventional-commit
  signals inside merged branches, which the default first-parent walk
  would miss under `--merge` merge policy. Without this, `cog bump --auto`
  would fail to detect `feat:` commits on merged branches and either
  propose no bump or fall back to a default. This is Combo 3's defense
  against the same class of issue that release-please's default config
  hit in Combo 1's revisit (Plan 3b).
- `disable_bump_commit = true` prevents cocogitto from writing anything
  back to the repo during the release workflow. GoReleaser owns the tag
  push; cocogitto only proposes.

## Future work: CHANGELOG.md at HEAD

This repo intentionally does **not** maintain `docs/CHANGELOG.md` at HEAD
in the PoC. The changelog is generated at release time only, from
`cog changelog --at <tag>`, and surfaces on the GitHub release page. This
deviates from the design spec §4.6 comparison summary, which lists
"CHANGELOG.md at HEAD" as a Combo 3 property.

Three paths would activate it in a follow-up:

**Path A — client-side hooks.** cocogitto ships git hooks that update
`docs/CHANGELOG.md` when contributors run `cog commit` and `cog bump`
locally. Install with `cog install-hook`. Requires every contributor to
use the cocogitto CLI; contributors using vanilla `git commit` bypass
the hook. Simplest to enable but weakest guarantee — the file drifts
the moment a non-cog contributor lands a commit.

**Path B — flip `disable_bump_commit = false` and grant workflow write.**
Let cocogitto commit CHANGELOG.md and push during the release workflow.
Requires the workflow identity on the `main`-ruleset bypass list (or
a dedicated `bonial-release` GitHub App, per Plan 2/3's shared
follow-up). This is a real direct-to-main push under CI — the same
trust boundary the tag push already crosses, but wider (arbitrary
code write, not just a tag ref).

**Path C — Combo-1-style PR flow.** Add a step that generates
CHANGELOG.md, opens a "Release" PR against `main` with the changelog +
version bump, and only runs the atomic release pipeline once the PR
is merged. Closest analogue to release-please's Release-PR model but
reintroduces the chain-prevention friction described above (Release
PRs need CI to fire, but PRs opened by `GITHUB_TOKEN` don't trigger
workflows).

**Recommendation for a hypothetical migration:** Path B under a
dedicated GitHub App, since the same App is already the answer to
multiple other spec follow-ups (tag ruleset in `active` mode,
`verify-release.yaml` auto-trigger, Combo 1's admin bypass removal).
One App identity resolves them all at once.

## Development

```bash
make install-tools   # install reuse into .venv-reuse
make lint            # reuse-lint + md-lint + go-lint
make test            # go test ./... -race -cover
make build           # go build ./...
```

Individual commits must be [Conventional Commits](https://www.conventionalcommits.org/); this is enforced by CI (`.github/workflows/commitlint.yaml`). The `main` branch protects against direct pushes; merge via PR.

## License

Apache 2.0. See [LICENSE](LICENSE) and [REUSE.toml](REUSE.toml).
