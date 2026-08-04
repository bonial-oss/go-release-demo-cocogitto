<!--
SPDX-FileCopyrightText: 2026 Bonial International GmbH
SPDX-License-Identifier: Apache-2.0
-->

# Rulesets

This directory holds the canonical JSON for the repo's two rulesets:

- `main.json` — branch ruleset `protect-main`. Enforcement: active.
- `tags.json` — tag ruleset `protect-version-tags`. Enforcement: evaluate.

Both are also applied live via `gh api` calls documented in the plan
(Tasks 18 and 19 of `docs/superpowers/plans/2026-07-31-plan-4-combo3-cocogitto.md`
in the meta repo).

## Why these shapes (PoC baseline)

### `main` ruleset

- `required_approving_review_count: 0` — solo-dev PoC. Raise to 1+ once
  co-reviewers exist. Follow-up in the meta ledger.
- No `required_signatures` rule — solo-dev PoC without commit-signing
  set up. Re-add once commit-signing is available. Follow-up in the
  meta ledger.
- Required status checks: `test-and-lint`, `commitlint`. Bare job names,
  NO `<workflow> / <job>` prefixes.
- `bypass_actors: []` — no bypasses. Unlike Combo 1's revisit (Plan 3b),
  Combo 3 has no Release-PR merge path that gets throttled by
  chain-prevention on `GITHUB_TOKEN`, so no Admin bypass is needed.
  The release-triggering PR's merge commit is authored by a human,
  downstream CI fires normally on `push: main`, and the atomic release
  pipeline runs without special handling.

### `tags` ruleset

- `enforcement: "evaluate"` — the bonial-oss org rejects the standard
  `Integration + actor_id 15368` bypass with 422 "Actor GitHub Actions
  integration must be part of the ruleset source or owner organization".
  Evaluate-mode gives us the audit trail without the block. Flip to
  `active` once the GH-App follow-up lands (see below).
- `bypass_actors: []` — no bypasses yet. Under evaluate-mode this is
  fine; the workflow's tag push produces a warning, not a failure.

## Follow-ups

1. **Dedicated `bonial-release` GitHub App.** Same follow-up as Plans 2
   and 3. The App identity would (a) let the tag ruleset flip to `active`
   with the App on the bypass list, (b) trigger `verify-release.yaml`'s
   `released` event auto-firing (chain-prevention on `GITHUB_TOKEN` no
   longer blocks under an App identity), (c) enable the CHANGELOG.md-
   at-HEAD path from the impl repo README's Future work section.
2. **Approvals + signatures.** Raise `required_approving_review_count`
   to 1+ and re-add the `required_signatures` rule once co-reviewers
   exist and commit-signing is set up.
