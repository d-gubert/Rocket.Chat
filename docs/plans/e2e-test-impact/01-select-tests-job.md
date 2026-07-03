# Phase 1 — `select-tests` job and suite-level gating

## Goal

Introduce a single CI job that decides, per PR, which test **jobs** need to run, and gate the
E2E jobs on its outputs. This phase only skips whole jobs based on changed paths and the
workspace graph — no spec-level selection yet. It also establishes the plumbing every later
phase reuses: the changed-file computation, the outputs contract, the kill switch, and the
`tests-done` skip handling.

Expected payoff: PRs touching only leaf packages, docs, storybook, or a single isolated
domain (e.g. `ee/packages/federation-matrix`) stop running the full E2E fleet. PRs touching
`apps/meteor` still run everything (that's phases 3–5).

## Prerequisites

Read `00-context.md`. No other phase required.

## Design

### New job `select-tests` in `.github/workflows/ci.yml`

Runs after `release-versions` (needs it only for ordering; it must be fast, ~30s, on
`ubuntu-24.04-arm`). Steps:

1. Full checkout with history for the merge base:
   `actions/checkout` with `fetch-depth: 0` (copy the pinned SHA used elsewhere in `ci.yml`).
2. Compute the changed file list:
   - `pull_request` event: `git diff --name-only "$(git merge-base origin/$GITHUB_BASE_REF HEAD)" HEAD`
     (fetch the base branch first: `git fetch origin "$GITHUB_BASE_REF"`).
   - Any other event (`merge_group`, `push`, `release`): emit no list and set every output
     to `true` (full run). Do not try to be clever outside PRs.
3. Run the selector script (below) and write its JSON result to `$GITHUB_OUTPUT`.
4. Print a human-readable decision summary to `$GITHUB_STEP_SUMMARY`: changed-file count,
   which rule fired, every output value. This summary is the primary debugging tool —
   make it good.

### Selector script `.github/scripts/select-tests.mjs`

Node ESM script, no external dependencies. Input: newline-separated changed file list on
stdin (or a `--files-from` arg) plus env `GITHUB_EVENT_NAME` and `E2E_SELECTION_DISABLED`.
Output: JSON on stdout:

```json
{
  "reason": "leaf-packages-only",
  "run-api": false,
  "run-api-livechat": false,
  "run-ui": false,
  "run-federation": false,
  "run-unit": true,
  "run-storybook": true,
  "needs-docker-build": false
}
```

Decision procedure (first match wins):

1. `E2E_SELECTION_DISABLED == 'true'` or event is not `pull_request` → all `true`,
   reason `selection-disabled` / `non-pr-event`.
2. Any changed file matches the **global-invalidation list** (from README invariant 3:
   `.github/**`, `docker-compose-ci.yml`, `**/Dockerfile*`, `yarn.lock`, `.yarnrc.yml`,
   `turbo.json`, root `package.json`, `apps/meteor/.meteor/**`, `.tool-versions`) → all
   `true`, reason `harness-change`.
3. All changed files match **no-test globs** (`docs/**`, `**/*.md` — already
   `paths-ignore`d but the merge-base diff can include them, `.vscode/**`,
   `development/**`) → all E2E outputs `false`, `needs-docker-build: false`,
   reason `docs-only`.
4. Compute the affected-workspace closure. Preferred: parse
   `turbo run build --filter='...[<merge-base-sha>]' --dry-run=json` (requires
   `yarn install` — if that is too slow in practice, fall back to a static map of
   `directory prefix → workspace` plus each workspace's `dependencies` from its
   `package.json`; keep the implementation behind a function so it can be swapped).
   Then apply suite rules:
   - `@rocket.chat/meteor` (i.e. `apps/meteor/**` or any workspace it depends on) affected
     → all E2E outputs `true`, reason `meteor-affected`. (Phases 3/5 refine this case.)
   - Only `ee/packages/federation-matrix` (and workspaces only it depends on) affected
     → `run-federation: true`, everything else E2E `false`.
   - Changes confined to `apps/meteor/tests/e2e/**` → `run-ui: true`, API/federation `false`
     (and vice versa for `apps/meteor/tests/end-to-end/**` → API suites only).
   - Workspaces with no path into `@rocket.chat/meteor`'s dependency closure and no rule
     above (e.g. `packages/storybook-config`, `packages/eslint-config`,
     `packages/release-action`, `apps/uikit-playground`) → E2E outputs `false`,
     `run-storybook`/`run-unit` stay `true` (unit/storybook are cheap; do not gate them in
     this phase).
5. Anything unrecognized → all `true`, reason `fallback-unknown-files`. **Fail open.**

Ship the script with unit tests (plain `node --test` in
`.github/scripts/select-tests.test.mjs`) covering each rule, especially the fail-open paths.

### Wiring in `ci.yml`

- Add `needs: [release-versions, select-tests]` and
  `if: needs.select-tests.outputs.run-<x> == 'true'` to: `test-api`, `test-api-livechat`,
  `test-ui`, `test-api-ee`, `test-api-livechat-ee`, `test-ui-ee`, `test-federation-matrix`.
- Gate `build-gh-docker` (and therefore `build-gh-docker-publish`, `build`,
  `packages-build`? **No** — keep `packages-build` and `build` ungated in this phase; only
  gate `build-gh-docker` + `build-gh-docker-publish` with `needs-docker-build`, and ONLY
  when all E2E outputs are false. Note `deploy`/`track-image-sizes` depend on publish but
  are already `if`-guarded to develop/release, where selection never gates anything).
  If this proves too entangled during implementation, ship phase 1 gating only the seven
  test jobs and leave docker builds always-on; record that as a follow-up. Skipping the
  docker build is an optimization, not the goal of this phase.
- `report-coverage` needs the EE jobs: add `if: always() &&` logic or make it tolerate
  skipped upstreams (`needs.*.result` checks) so a gated run doesn't fail it.
- **`tests-done`**: rewrite the aggregation to accept, per job, either `success` or
  (`skipped` **and** the corresponding `select-tests` output was `false`). Any `failure`,
  `cancelled`, or unexpected `skipped` → exit 1. Implement as a small inline script looping
  over a job→output mapping rather than nine copy-pasted `if` blocks.

### Kill switch

Repository variable `E2E_SELECTION_DISABLED` (Settings → Variables). The selector reads it
via `${{ vars.E2E_SELECTION_DISABLED }}` passed as env. Document it in the step summary
output when active.

## Files to create/modify

- `.github/workflows/ci.yml` — add `select-tests`, gate jobs, rewrite `tests-done`.
- `.github/scripts/select-tests.mjs` — selector.
- `.github/scripts/select-tests.test.mjs` — tests.
- `docs/plans/e2e-test-impact/01-select-tests-job.md` — update status when done.

## Acceptance criteria

1. PR changing only `packages/eslint-config/**`: all seven E2E jobs show as skipped,
   `tests-done` succeeds, unit/storybook/checks still run.
2. PR changing only `ee/packages/federation-matrix/**`: only `test-federation-matrix` runs
   among E2E jobs; `tests-done` succeeds.
3. PR changing `apps/meteor/server/**`: all E2E jobs run (reason `meteor-affected`).
4. PR changing `.github/workflows/anything.yml`: all jobs run (reason `harness-change`).
5. `merge_group` and `develop` push runs: selector reports `non-pr-event`, nothing skipped.
6. With `E2E_SELECTION_DISABLED=true`: nothing skipped on any event.
7. `node --test .github/scripts/` passes; `actionlint` passes on the modified workflow.
8. A skipped-jobs PR is mergeable (branch protection satisfied by `tests-done`).

## Verification

- Unit tests for the selector (rule table above, one test per rule + fail-open).
- Open a draft PR against this branch touching only a leaf package and observe the run.
- Manually inspect the `$GITHUB_STEP_SUMMARY` of `select-tests` for each scenario.

## Rollback

Set `E2E_SELECTION_DISABLED=true` (immediate, no deploy), or revert the `ci.yml` changes.
The selector script is inert without the workflow wiring.

## Out of scope

Spec-level selection (phase 3), smoke subset (phase 3), gating `test-unit`/`test-storybook`,
skipping `packages-build`/`build`.
