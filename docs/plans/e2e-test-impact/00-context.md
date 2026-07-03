# Phase 0 — Shared context (read this before executing any phase)

Facts about the current CI and test architecture. All paths are repo-relative. Line numbers
are approximate (verified 2026-07) — re-check before editing, but do not re-derive the
architecture from scratch.

## Repository layout

- Yarn workspaces + Turborepo monorepo. Workspaces: `apps/*` (`meteor` = the product,
  `uikit-playground`), `packages/*` (~55 shared packages), `ee/apps/*`, `ee/packages/*`
  (enterprise: `federation-matrix`, `license`, `presence`, `omnichannel-services`, …).
- `apps/meteor` is the server+client application. It imports most of `packages/*`, so
  package-graph analysis alone rarely excludes the main suites — but it *does* exclude leaf
  packages (`storybook-config`, `eslint-config`, `release-action`, docs, `uikit-playground`).
- `turbo run build --filter='...[<ref>]' --dry-run=json` prints the affected-package closure
  for a git range; `.packages[]` in the output lists impacted workspace names.

## CI entry point: `.github/workflows/ci.yml`

Triggers: `pull_request` (all branches, `paths-ignore: ['**.md']`), `merge_group`,
`push` to `develop`, `release: published`. Concurrency cancels superseded PR runs.

Job graph (names matter — branch protection depends on them):

```
release-versions ─┬─ notify-draft-services ── packages-build ── build (meteor) ── build-gh-docker ── build-gh-docker-publish
                  ├─ checks (ci-code-check.yml)                                                            │
                  ├─ test-unit / test-storybook (need packages-build)                                      │
                  └─ actionlint (only if .github/** changed)                                               │
test-api, test-api-livechat, test-ui, test-api-ee, test-api-livechat-ee, test-ui-ee  ◄── need [checks, build-gh-docker-publish, release-versions]
test-federation-matrix ◄── needs [checks, build-gh-docker-publish, packages-build, release-versions]
report-coverage ◄── needs the three *-ee jobs
tests-done ◄── needs [checks, test-unit, test-api, test-ui, test-api-ee, test-ui-ee, test-api-livechat, test-api-livechat-ee, test-federation-matrix], if: always()
```

Key details:

- `release-versions` already computes a diff-based output: `github-actions-changed` via
  `gh pr diff --name-only` (step id `diff`, ~line 144). Use the same technique for new
  diff-based outputs, or `git diff --name-only` against the merge base after a full checkout.
- `tests-done` (~line 836) fails unless **every** needed job's `result == 'success'`. Any
  gating added in phase 1 must update this job to accept `skipped` for intentionally
  skipped jobs.
- All E2E jobs call the reusable workflow `.github/workflows/ci-test-e2e.yml` with inputs:
  `type` (`api` | `api-livechat` | `ui`), `release` (`ce` | `ee`), `shard` (JSON array
  string), `total-shard`, `mongodb-version`, `coverage`, `retries`, plus version/tag params.
- UI suites are sharded: CE `[1,2,3,4]`/4, EE `[1,2,3,4,5]`/5 via Playwright
  `--shard=$E2E_SHARD/$E2E_TOTAL_SHARD` (count-based distribution, no duration awareness).
- API suites are NOT sharded and run twice (CE and EE variants of the same suite).
- Fork PRs: no registry secrets → docker images travel via artifacts (`Download Docker
  images` steps guarded by `github.event.pull_request.head.repo.full_name != github.repository`).
- Dependabot: docker build/publish steps are skipped (`github.actor != 'dependabot[bot]'`).

## The reusable E2E workflow: `.github/workflows/ci-test-e2e.yml`

Single job `test`, matrix over `mongodb-version × shard`. Relevant steps:

- Starts the server from docker images via `docker-compose-ci.yml`
  (CE: `rocketchat` container only; EE: full micro-services set + `--wait` on `ddp-streamer`).
- `type == 'api'` → `npm run testapi` in `apps/meteor`, then `docker compose stop`
  (the stop is what flushes server coverage — see below).
- `type == 'api-livechat'` → `npm run testapi:livechat`, same pattern.
- `type == 'ui'` → `yarn test:e2e --shard=...` in `apps/meteor`.
- Coverage is only collected when `inputs.coverage == matrix.mongodb-version` (currently
  only on the three EE jobs, with `coverage: '8.0'`); coverage artifacts are named
  `coverage-<type>-<shard>` and merged by `report-coverage` in `ci.yml` → codecov.

## Test suites

### Playwright UI suite — `apps/meteor/tests/e2e/`

- ~90 flat `*.spec.ts` files named by domain (`admin-*`, `omnichannel/`, `e2e-encryption/`,
  `message-*`, …) plus dirs `omnichannel/`, `e2e-encryption/`, `apps/`, `federation/` (ignored).
- Config `apps/meteor/playwright.config.ts`: `testDir: 'tests/e2e'`,
  `testIgnore: 'tests/e2e/federation/**'`, `workers: 1`, `maxFailures: 5` on CI,
  retries from `PLAYWRIGHT_RETRIES` (0 on PRs, 2 on develop/release).
- Reporters: `list`, custom `./reporters/rocketchat.ts` and `./reporters/jira.ts`
  (enabled when `REPORTER_ROCKETCHAT_REPORT=true`), `playwright-qase-reporter`.
  The rocketchat reporter posts per-test results incl. durations to an internal service —
  a source of historical durations for phase 5.
- Client-side coverage: `apps/meteor/tests/e2e/utils/test.ts` exposes
  `collectIstanbulCoverage` on the browser context when `E2E_COVERAGE` is set; per-page
  `window.__coverage__` snapshots are written to `.nyc_output/playwright_coverage_<uuid>.json`
  and merged by the workflow step `npx nyc merge .nyc_output ...`.

### Mocha REST API suite — `apps/meteor/tests/end-to-end/`

- `apps/meteor/.mocharc.api.js` — spec globs:
  `['tests/end-to-end/api/*.ts', 'tests/end-to-end/api/helpers/**/*', 'tests/end-to-end/api/methods/**/*', 'tests/end-to-end/apps/*']`
  (~50 top-level domain files: `channels.ts`, `chat.ts`, `rooms.ts`, `users.ts`, …).
- `apps/meteor/.mocharc.api.livechat.js` — `spec: ['tests/end-to-end/api/livechat/**/*']`,
  `bail: true`.
- Both use `file: 'tests/end-to-end/teardown.ts'` and a custom reporter.
- Many API test files check `IS_EE` env internally to toggle EE-only assertions.

### Server-side coverage instrumentation

- The `-cov` docker image is a Meteor build that includes the local Meteor package
  `apps/meteor/packages/rocketchat-coverage` (plugin
  `apps/meteor/packages/rocketchat-coverage/plugin/compile-version.js`).
- That plugin registers a `process.on('exit')` hook which serializes
  `globalThis['__coverage__']` via `istanbul-lib-coverage`/`istanbul-reports` into
  `$COVERAGE_DIR/$COVERAGE_FILE_NAME` (reporter from `$COVERAGE_REPORTER`, default `lcov`).
- `docker-compose-ci.yml` bind-mounts `${COVERAGE_DIR:-/tmp/coverage}` into the container
  and passes the three `COVERAGE_*` env vars through.
- Granularity today: one dump per container lifetime (i.e., per suite/shard), flushed when
  the workflow runs `docker compose stop`. Phase 4 adds per-spec flushing.

## Existing safety nets to preserve

- `merge_group` trigger runs the identical full pipeline before merge into `develop`.
- `develop` pushes run the full pipeline with coverage + production images.
- Draft PRs already skip result reporting but still run tests.

## Glossary

- **Selection**: deciding at PR time which suites/spec files run.
- **Suite-level gating**: skipping an entire job (`test-ui`, `test-federation-matrix`, …).
- **Spec-level selection**: running a subset of spec files within a job.
- **The map**: data mapping each spec file to the source files it exercises
  (curated in phase 3, coverage-derived in phase 4).
