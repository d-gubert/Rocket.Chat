# Phase 6 — Observability, rollout policy, and the vendor alternative

## Goal

Make the selection system measurable, debuggable, and governable. Items here are
incremental and should be interleaved with phases 1–5 (the ones marked **[pre-3]** must
land before phase 3 activates spec-level skipping).

## Work items

### 6a. Decision telemetry **[pre-3]**

Every `select-tests` run must leave an audit trail:

- Step summary (already required by phase 1): changed files count, rule/reason, outputs.
- Machine-readable artifact `selection-decision.json` per run: event, base/head SHAs,
  changed files, reason, selected suites/specs, map commit (when applicable), durations.
  Retention 90 days. This is what you grep when someone asks "why didn't my test run on
  PR #X" or "would selection have caught the regression that slipped through".

### 6b. Escape-fraction monitoring

The core risk metric: regressions that PR-selection skipped but the merge queue or develop
caught. Process (can be a lightweight scheduled workflow or a runbook):

- Whenever a `merge_group` or develop run fails a test that the corresponding PR run
  skipped, that's an **escape**. Correlate via `selection-decision.json` + the failing
  run's PR.
- Track escapes per month. Target: <1/month. Two same-root-cause escapes → add a curated
  rule or map fix; systemic escapes → widen the fallback (or disable via kill switch)
  until fixed.

### 6c. Savings dashboard

Report weekly (a scheduled workflow posting to a channel, or just a documented query):

- median & p90 PR CI wall-clock (`tests-done` completion − run start), PR job-minutes
  total, % of PRs with full run vs selected run, reason distribution.
- Baseline: capture 2 weeks of these numbers BEFORE phase 1 merges (executor of phase 1:
  record them in this file under "Baseline" when starting).

### 6d. Flaky-test interaction

Selection shrinks the flaky surface per PR, but a flaky `@smoke` test hits *every* PR.
Policy: smoke set membership requires <0.5% historical failure-then-pass-on-retry rate
(from the rocketchat reporter data / Qase). Review smoke membership monthly. Any spec
quarantined for flakiness must be removed from `@smoke`.

### 6e. Docs & runbook **[pre-3]**

`docs/plans/e2e-test-impact/RUNBOOK.md` (create in this phase) covering:
- how to read the step summary / decision artifact;
- how to force a full run on one PR (label `ci-full-run` — implement: selector checks
  `github.event.pull_request.labels`, any occurrence → full, reason `label-override`);
- kill switch / mode variables (`E2E_SELECTION_DISABLED`, `E2E_SELECTION_MODE`);
- how to add a curated map rule; how to interpret map validation failures.

### 6f. Vendor track (optional, parallel evaluation)

If build-vs-buy is still open after phase 3, evaluate against the in-house track:

- **Datadog Test Impact Analysis** (Test Optimization): per-test coverage + automatic
  skipping for JS runners. Evaluation questions: current Playwright/mocha support level;
  whether per-test coverage collection works when the system under test is a separate
  Docker container (our case — the answer likely requires the same instrumented-image
  plumbing phase 4 builds, which weakens the buy case); pricing per committed test span.
- **Launchable / CloudBees**: predictive selection from test results + diff history, no
  coverage needed. Works with any runner via JUnit/JSON reports. Evaluation: prediction
  quality on ~6 months of our history (they offer offline evaluation), data egress
  (source-file names + test results leave the org — needs approval), subset API ergonomics
  with Playwright sharding.
- Decision criterion: adopt a vendor only if it beats the phase 5 design on (a) escape
  rate, (b) integration effort for the Docker-boundary coverage problem, (c) cost vs
  saved compute. Record the decision here.

## Acceptance criteria

- 6a and 6e merged before phase 3 activation; label override works.
- Baseline metrics recorded below before phase 1 merges.
- First monthly escape review completed after phase 3 (record findings here).

## Baseline (fill in during execution)

- date range: _TBD_
- median PR CI wall-clock: _TBD_
- p90 PR CI wall-clock: _TBD_
- median PR job-minutes (sum across jobs): _TBD_
- E2E failure rate on PRs unrelated to the change (flakiness proxy): _TBD_
