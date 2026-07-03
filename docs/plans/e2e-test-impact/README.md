# E2E Test Impact Analysis — Execution Plan

Goal: stop running E2E test suites that are unrelated to the changes in a PR, cutting CI
wall-clock time, compute cost, and flaky-failure noise — **without** reducing the coverage
guarantees on `develop` and in the merge queue.

This plan is broken into self-contained phase documents so that each phase can be handed to a
separate agent (or human) with minimal extra context. Read `00-context.md` first — it contains
the codebase facts every phase relies on, so phase executors don't need to rediscover them.

## Phase index and dependency order

| Phase | File | Deliverable | Depends on |
|---|---|---|---|
| 0 | `00-context.md` | Shared context (read-only, no work) | — |
| 1 | `01-select-tests-job.md` | `select-tests` CI job + job-level gating by changed paths | — |
| 2 | `02-suite-partitioning.md` | Domain tags on Playwright specs + split mocha spec groups + subset invocation support | — (parallel with 1) |
| 3 | `03-curated-map.md` | Curated path→suite map, spec-level selection on PRs, smoke subset | 1, 2 |
| 4 | `04-coverage-map.md` | Per-spec coverage collection + test→source map built on `develop` runs | 2 |
| 5 | `05-dynamic-selection.md` | PR-time selection driven by the coverage map + duration-based sharding | 3, 4 |
| 6 | `06-observability-rollout.md` | Metrics, kill switch, rollout policy, optional vendor track | 1 |

Phases 1 and 2 can be executed in parallel. Phase 6 items should be picked up incrementally
alongside phases 1–5 (the kill switch must exist before phase 3 ships).

## Non-negotiable invariants (apply to every phase)

1. **Fail open.** Whenever the selection logic cannot confidently map a change to a test
   subset — unknown files, stale data, script error, missing artifact — the FULL suite runs.
   A bug in selection must never silently skip tests; it may only waste compute.
2. **PR-only.** Selection applies only when `github.event_name == 'pull_request'`.
   `merge_group`, pushes to `develop`, and `release` events always run the full suite.
   The merge queue is the safety net that makes aggressive PR-time skipping acceptable.
3. **Harness changes run everything.** Changes to `.github/**`, `docker-compose-ci.yml`,
   any `Dockerfile`, `yarn.lock`, `turbo.json`, root `package.json`, `apps/meteor/.meteor/**`,
   or the test harness itself (`apps/meteor/tests/e2e/{config,fixtures,page-objects,utils}/**`,
   `apps/meteor/tests/end-to-end/helpers/**`, `apps/meteor/playwright.config.ts`,
   `apps/meteor/.mocharc*`) invalidate all selection → full suite.
4. **New/changed spec files always run**, regardless of what the map says.
5. **Required checks must stay green when jobs are skipped.** The `tests-done` aggregation
   job must treat `skipped` as acceptable for any job that the selector intentionally skipped
   (see phase 1). Never leave a state where a legitimately skipped job blocks merging.
6. **Kill switch.** A repository variable `E2E_SELECTION_DISABLED` (checked by the
   `select-tests` job) must force full-suite behavior when set to `'true'`.

## Conventions for executing agents

- One phase = one PR. Keep diffs reviewable; do not mix phases.
- Every phase file has **Acceptance criteria** and **Verification** sections — do all of them
  before declaring the phase done.
- When editing workflows, run `actionlint` locally (the repo pins version 1.7.12 in
  `ci.yml`'s `actionlint` job) — CI only runs it when `.github/**` changed, which will be
  true for these PRs.
- Shell steps in workflows follow the existing style: `set -o xtrace` for debuggable steps,
  pinned action SHAs with version comments (copy pins from existing usage in `ci.yml`).
- New scripts go in `.github/scripts/` (create it), written in Node.js (no new runtime deps
  unless the phase says otherwise), executable via `node .github/scripts/<name>.mjs`.
- Do not rename existing jobs (`test-api`, `test-ui`, `tests-done`, …). They are referenced
  by branch protection / merge queue configuration outside the repo.
