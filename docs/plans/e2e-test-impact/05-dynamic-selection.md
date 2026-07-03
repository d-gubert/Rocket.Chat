# Phase 5 — Coverage-map-driven selection and duration-based sharding

## Goal

Replace the curated map as the primary selector with the empirical coverage map from
phase 4: on a PR, run exactly the spec files whose recorded source sets intersect the
changed files (plus smoke, plus new/changed specs). Adapt UI shard counts to the size of
the selection. The curated map and phase 1 rules remain as fallbacks.

## Prerequisites

Phases 3 and 4 merged; the `test-map` artifact has been produced on develop for ≥1 week
with validation green. Read `00-context.md`.

## Design

### 5a. Map retrieval in `select-tests`

- New step: query the Actions API for the most recent successful develop run of `ci.yml`
  that has a `test-map` artifact; download and unpack it. (Use `actions/download-artifact`
  cross-run via `run-id`, or `gh api` — `GITHUB_TOKEN` with `actions: read` suffices; the
  job already has it. For fork PRs the token can still read same-repo artifacts.)
- Staleness policy: reject the map if `commit` is older than N=7 days OR more than M=300
  commits behind the merge base (`git rev-list --count map.commit..merge-base`; if
  `map.commit` is unknown locally after `fetch-depth: 0`, treat as stale). Stale/missing →
  fall back to phase 3 behavior, reason `map-stale` / `map-missing`.

### 5b. Selection algorithm (extends `select-tests.mjs`)

Runs in the `meteor-affected` branch, before the curated-map logic:

1. Global invalidation & non-PR events: unchanged from phases 1/3 (checked first).
2. For each changed file, look up the inverted index `source → specs`.
   - File present in index → collect its specs.
   - File absent → is it *expected* to be absent? Maintain an "unmappable but harmless"
     allowlist (pure-type packages like `packages/core-typings/**`,
     `packages/rest-typings/**` compile away and won't appear in runtime coverage — their
     changes DO affect behavior via consumers, so they must NOT be treated as harmless;
     instead: absent file under a workspace that IS in the map's source universe → treat
     as unknown → full run, reason `uncovered-file`. Only genuinely non-runtime paths
     (tests, tooling, i18n? — no: i18n affects UI strings, keep it mapped or full) may be
     allowlisted. Start with an EMPTY allowlist; grow deliberately.)
3. Selection = union of collected specs + all `@smoke` specs + changed spec files
   themselves + specs added since the map's commit (`git diff --name-only map.commit..HEAD
   -- 'apps/meteor/tests/e2e/**' 'apps/meteor/tests/end-to-end/**'` intersected with
   spec patterns).
4. Safety valve: if selection covers >60% of all specs, just run the full suite (skip
   grep overhead and shard-count games), reason `selection-near-full`.
5. Emit:
   - `ui-spec-files`: newline/space-separated list (preferred over grep now — exact files);
   - `api-spec-files`: file list for mocha (mocharc accepts explicit spec paths via
     `API_TEST_SPECS` env — add alongside the phase 2 `API_TEST_GROUPS`);
   - `run-*` booleans and livechat flag derived from whether any selected spec belongs to
     each suite;
   - `ui-shard-total`: computed below.
6. Cross-check (during rollout only, see 5d): also compute the phase 3 curated-map
   decision; if curated says "run X" and coverage-map selection excludes all of X's specs,
   log a discrepancy warning in the step summary (signal for map quality issues).

### 5c. Duration-based shard adaptation (UI suites)

- Source of durations: prefer a `durations.json` emitted into the `test-map` artifact by
  phase 4's builder (per-spec wall-clock is available in the raw per-spec artifacts'
  timestamps, or from Playwright's JSON report — extend the phase 4 builder accordingly;
  fallback default 5 min/spec for unknown specs).
- `ui-shard-total = clamp(ceil(total-selected-duration / TARGET_SHARD_MINUTES), 1, current-max)`
  with `TARGET_SHARD_MINUTES ≈ 15`. Emit `ui-shard` as the JSON array `[1..n]`.
- `ci.yml` passes `shard: ${{ needs.select-tests.outputs.ui-shard }}` and
  `total-shard: ${{ needs.select-tests.outputs.ui-shard-total }}` (with full-run defaults
  preserved for non-PR events — keep today's 4/5 shards for full runs).
- Playwright's `--shard` splits the *filtered* set when spec files are passed as args, so
  file-list + shard compose naturally. Verify empirically with `--list`.

### 5d. Rollout: shadow mode first

Ship 5a/5b computing and LOGGING decisions (step summary + a `selection-shadow.json`
artifact) while phase 3 curated behavior still drives the actual gating, for ≥2 weeks of
PRs. Then compare:

- how often shadow selection ⊂ curated selection (expected: almost always);
- any PR where a test that FAILED in the actual run would have been skipped by shadow
  selection → blocking finding, investigate map quality before activation.

Activation = flipping a single env/var (`E2E_SELECTION_MODE=coverage` vs `curated`) read
by `select-tests`. Keep `curated` as automatic fallback on any map problem.

## Files to create/modify

- `.github/scripts/select-tests.mjs` (+ lib for map load/staleness/inverted index) + tests
- `.github/scripts/build-test-map.mjs` (durations)
- `apps/meteor/.mocharc.api.js` (`API_TEST_SPECS`)
- `.github/workflows/ci.yml`, `.github/workflows/ci-test-e2e.yml` (spec-file inputs,
  dynamic shard matrix)

## Acceptance criteria

1. Shadow mode ran ≥2 weeks; comparison report attached to the activation PR; zero
   would-have-skipped-a-real-failure findings (or each one root-caused + fixed).
2. With `E2E_SELECTION_MODE=coverage`: a PR touching one admin client view runs only the
   mapped specs (+smoke), on a reduced shard count, and `tests-done` is green.
3. Stale-map simulation (point the retrieval at an old run) → curated fallback, reason
   visible in summary.
4. Full-run paths (develop, merge_group, harness change, kill switch) byte-identical to
   phase 3 behavior.
5. Selector unit tests: inverted-index lookup, uncovered-file → full run, near-full valve,
   new-spec inclusion, shard-count math, staleness math.

## Verification

- Unit tests + three scripted draft-PR scenarios (mapped, uncovered file, huge selection).
- Measure and report: median PR E2E wall-clock and total job-minutes, week before vs
  week after activation (this is the number the whole project is judged on).

## Rollback

`E2E_SELECTION_MODE=curated` (instant), or `E2E_SELECTION_DISABLED=true` (full runs).

## Out of scope

ML-based selection, per-test granularity, CE/EE dedup (candidate follow-up: run CE API
suite only when EE isn't selected, since EE ⊃ CE for most files), federation mapping.
