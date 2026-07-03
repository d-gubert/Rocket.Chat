# Phase 4 — Per-spec coverage collection and the test→source map

## Goal

Build the empirical dependency graph: for every E2E spec file, the set of product source
files it exercises (server- and client-side). Generate it automatically on `develop` runs
(where the instrumented `-cov` image and full suites already run) and publish it as an
artifact. This phase only PRODUCES the map; phase 5 consumes it.

## Prerequisites

Phase 2 (per-spec addressability helps validation, and the mocha root-hook work overlaps).
Independent of phases 1/3. Read `00-context.md`, especially "Server-side coverage
instrumentation".

## Design

### 4a. Server-side per-spec coverage flush

Today `apps/meteor/packages/rocketchat-coverage` dumps `globalThis.__coverage__` once on
process exit. Add an HTTP flush endpoint, active only in test mode:

- In the meteor app (suggested location: a small server module registered only when
  `process.env.TEST_MODE` is truthy — follow how other TEST_MODE-only endpoints are
  registered; search `apps/meteor/server` for `TEST_MODE` to find the pattern), expose:
  - `POST /internal/coverage/snapshot` with JSON body `{ "label": "<spec-file-id>" }` —
    serializes the current `globalThis.__coverage__` coverage map (same
    `istanbul-lib-coverage` API the exit hook uses) to
    `$COVERAGE_DIR/per-spec/<sanitized-label>.json`, then **resets** counters
    (`istanbul-lib-coverage` `CoverageMap` doesn't reset in place — rebuild by zeroing
    the `s`/`f`/`b` counters of each FileCoverage, or snapshot-and-diff instead of reset;
    diffing consecutive cumulative snapshots is simpler and avoids mutating live counters —
    prefer the diff approach: endpoint just dumps cumulative state, the map builder diffs).
  - Guard: 404 unless TEST_MODE; no auth needed beyond that (CI-internal network), but keep
    the path under a clearly internal prefix.
- Decision to record in code comments: cumulative-snapshot + offline diff (recommended)
  vs in-process reset. Cumulative is race-tolerant (background jobs attribute noise to the
  next spec either way) and cannot corrupt the final whole-run report that the existing
  exit hook still produces.

### 4b. Snapshot triggers from the test runners

- **Playwright**: in the shared fixture file (`apps/meteor/tests/e2e/utils/test.ts`), add a
  worker-scoped auto fixture or `afterAll` per spec file that, when `E2E_MAP_BUILD=true`,
  POSTs the snapshot endpoint with the spec file's repo-relative path. Also collect the
  client-side coverage per spec: the existing `collectIstanbulCoverage` machinery writes
  per-page snapshots into `.nyc_output/` — extend it (when `E2E_MAP_BUILD=true`) to record
  which spec file was active, e.g. write into `.nyc_output/per-spec/<spec>/...`. Note
  `workers: 1` in CI makes per-spec attribution unambiguous — do not enable map-build with
  more workers.
- **Mocha API suite**: root hooks in a new file loaded via the mocharc `file`/`require`
  option: after each top-level spec file (use `afterAll` per file via `--file` semantics or
  a root `afterEach` that detects file transitions from `this.currentTest.file`), POST the
  snapshot endpoint labeled with the file path. Only when `E2E_MAP_BUILD=true`.

### 4c. Map builder `.github/scripts/build-test-map.mjs`

Input: the `per-spec/` directories from all suites/shards of one develop run (downloaded
as artifacts), plus the run's commit SHA. Processing:

1. Order cumulative server snapshots per suite/shard by sequence, diff consecutive pairs →
   per-spec server file sets (a file is "touched" if any statement counter increased).
2. Convert instrumented paths to repo-relative source paths (Meteor build paths differ from
   repo paths — inspect actual snapshot keys early; budget time for path mapping, it is the
   most likely surprise in this phase. `report-coverage` in `ci.yml` already produces lcov
   from these coverage objects, so a working path mapping exists — reuse its config).
3. Merge client-side per-spec sets (nyc json → file lists, same repo-relative mapping via
   the existing sourcemap handling in the UI coverage flow).
4. Emit `test-map.json`:
   ```json
   {
     "version": 1,
     "commit": "<sha>",
     "generatedAt": "<iso>",
     "specs": {
       "tests/e2e/admin-users.spec.ts": { "sources": ["apps/meteor/client/views/admin/...", ...] },
       "tests/end-to-end/api/rooms.ts": { "sources": [...] }
     }
   }
   ```
   Plus an inverted index `sources → specs` if size permits (compute at load time otherwise).
   Compress (`test-map.json.gz`); expect a few MB.

### 4d. CI wiring (develop runs only)

- `ci-test-e2e.yml`: new optional input `map-build` (boolean). When true AND
  `inputs.coverage == matrix.mongodb-version`, set `E2E_MAP_BUILD=true` on the test steps
  and upload `$COVERAGE_DIR/per-spec` + UI per-spec output as artifact
  `test-map-raw-<type>-<shard>`.
- `ci.yml`: pass `map-build: ${{ github.ref == 'refs/heads/develop' }}` on the three EE
  jobs (they're the coverage-enabled ones). Add job `build-test-map` (needs the EE jobs,
  develop only): download `test-map-raw-*`, run the builder, upload artifact `test-map`
  with `retention-days: 30`. Phase 5 fetches "latest develop run's `test-map` artifact"
  via the Actions API.
- Overhead guard: snapshotting is IO-light per spec (~90 UI + ~50 API snapshots of a
  few-MB JSON). Measure the delta on develop wall-clock in the phase PR; >5% needs
  discussion before merge. PR runs are untouched (`E2E_MAP_BUILD` unset).

### 4e. Map quality validation `.github/scripts/validate-test-map.mjs`

Sanity checks run right after building (fail the `build-test-map` job on violation):
- every spec file discovered by `--list`/groups.json appears in the map;
- no spec has an empty source set;
- spot-fixture: `tests/e2e/omnichannel/**` specs must include at least one
  `livechat`-path source; `admin-users.spec.ts` must include an admin view file.
- map size/spec-count drift vs previous artifact within ±15% (warn, don't fail, on first
  runs).

## Files to create/modify

- `apps/meteor/packages/rocketchat-coverage/**` or a TEST_MODE server module (endpoint)
- `apps/meteor/tests/e2e/utils/test.ts` (Playwright per-spec hooks)
- `apps/meteor/tests/end-to-end/` root-hook file + mocharc `require` wiring
- `.github/scripts/build-test-map.mjs`, `.github/scripts/validate-test-map.mjs` + tests
- `.github/workflows/ci-test-e2e.yml`, `.github/workflows/ci.yml`

## Acceptance criteria

1. A develop run produces a `test-map` artifact passing all validation checks.
2. PR runs are bitwise-unchanged (no `E2E_MAP_BUILD`, no new steps executed).
3. The whole-run coverage report (`report-coverage` → codecov) is unchanged (cumulative
   snapshots don't disturb the exit-hook dump).
4. Develop E2E wall-clock overhead measured and ≤5% (documented in PR).
5. Manual spot check: pick 3 specs, eyeball their source sets for plausibility; pick one
   source file, confirm the specs listing it look right.

## Verification

- Local: run the CE compose stack with the `-cov` image + `TEST_MODE`, run 2–3 specs with
  `E2E_MAP_BUILD=true`, inspect snapshots and a locally-built mini-map.
- CI: temporarily enable `map-build` on a draft PR (own branch) to exercise the full path
  before wiring to develop-only.

## Rollback

`map-build` input defaults false; removing the `ci.yml` develop wiring stops map
production. The endpoint is inert outside TEST_MODE. Phase 5 must treat a missing/stale
map as "fail open", so stopping production degrades gracefully.

## Out of scope

Consuming the map on PRs (phase 5). Per-test (vs per-spec-file) granularity. Federation
suite mapping (its jest harness differs; add later if worthwhile).
