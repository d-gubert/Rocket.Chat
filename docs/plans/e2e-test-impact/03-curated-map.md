# Phase 3 — Curated path→suite map and spec-level selection on PRs

## Goal

When a PR touches `apps/meteor` (the case phase 1 can't narrow), consult a human-curated,
in-repo map from source path globs to test domains, and run only the mapped domains plus
the `@smoke` subset — falling back to the full suite whenever any changed file is unmapped.

This is the first phase that actually skips tests for typical product PRs. It ships behind
the phase 1 kill switch and the merge-queue full run remains the safety net.

## Prerequisites

Phases 1 (select-tests job, outputs contract, kill switch) and 2 (tags, API groups,
subset invocation) merged. Read `00-context.md`.

## Design

### 3a. The map: `.github/test-impact-map.yml`

Reviewed-in-repo YAML. Schema:

```yaml
version: 1
rules:
  - name: omnichannel
    sources:
      - 'apps/meteor/app/livechat/**'
      - 'apps/meteor/app/livechat-enterprise/**'
      - 'apps/meteor/client/views/omnichannel/**'
      - 'packages/livechat/**'
      - 'packages/omni-core/**'
      - 'ee/packages/omnichannel-services/**'
      - 'ee/packages/omni-core-ee/**'
    ui-tags: ['@omnichannel']
    api-groups: []            # regular API suite not needed
    api-livechat: true        # run the livechat mocha suite
  - name: admin
    sources:
      - 'apps/meteor/client/views/admin/**'
    ui-tags: ['@admin']
    api-groups: ['admin']
    api-livechat: false
```

Start with rules for the 5–10 highest-churn, best-isolated domains only (suggested:
omnichannel, admin UI, E2EE, message composer/actions client code, account/profile client
code, file upload). Use `git log --since='6 months' --name-only` to rank churn. **Every
rule's `sources` must be reviewed by a domain owner before merge** — a wrong mapping here
is the main correctness risk of this phase; the executor drafts, humans confirm.

Server-side directories are usually NOT well-isolated (e.g. `apps/meteor/app/lib/**`,
`apps/meteor/server/**` are shared) — do not write rules for them. Client `views/`
directories and clearly-scoped feature dirs are the good candidates. When in doubt, leave
the path unmapped (→ full run).

### 3b. Selector extension (`.github/scripts/select-tests.mjs`)

Extend the phase 1 decision procedure. In the `meteor-affected` branch, instead of
returning all-true immediately:

1. Match every changed file against the map's `sources` globs (add a tiny glob matcher or
   vendored minimatch-equivalent; still no runtime deps — a ~50-line glob-to-regex
   function is fine and must be unit-tested).
2. If ANY changed file matches no rule → all `true`, reason `unmapped-files` (list the
   first few unmapped files in the reason detail for the step summary).
3. Else union the matched rules and emit new outputs alongside the phase 1 booleans:
   - `ui-grep`: `'@smoke|@omnichannel|@admin'` (always includes `@smoke`)
   - `api-groups`: `'admin'` (comma-separated; empty string = suite not needed, but see
     smoke note below)
   - `run-api-livechat`: from the rules' `api-livechat` flags
   - `run-ui: true` when `ui-grep` is non-trivial; `run-api: true` when `api-groups`
     non-empty.
4. Changed spec files map to their own domain: a change under `tests/e2e/omnichannel/**`
   adds `@omnichannel` to `ui-grep`; a change under `tests/end-to-end/api/<file>.ts` adds
   that file's group. (Group/tag lookup reuses `groups.json` and file→tag inference from
   the phase 2 taxonomy — encode the file→tag mapping in `tags.ts`/a JSON manifest so the
   selector can read it without executing TypeScript.)
5. API smoke: there is no `@smoke` equivalent for mocha; when the UI suite runs but no API
   groups matched, still run one designated cheap group (`misc`) as an API sanity check.
   Encode this in the selector, not the workflow.

### 3c. Workflow wiring

- `.github/workflows/ci.yml`: pass new inputs through to the reusable workflow calls, e.g.
  `ui-grep: ${{ needs.select-tests.outputs.ui-grep }}` and
  `api-groups: ${{ needs.select-tests.outputs.api-groups }}` on all six suite jobs
  (CE and EE get the same selection).
- `.github/workflows/ci-test-e2e.yml`: add optional inputs `ui-grep` (string) and
  `api-groups` (string). Wire:
  - UI step: `yarn test:e2e --shard=... ${UI_GREP:+--grep "$UI_GREP"}` via env.
  - API steps: `API_TEST_GROUPS="$API_GROUPS" npm run testapi`.
  Empty inputs (develop/merge_group/full-run PRs) → identical behavior to today. Verify
  quoting survives regex metacharacters (`|`) in grep values.
- Sharding note: a selected UI run may have far fewer specs than shards. That's fine for
  now (idle shards finish fast because Playwright distributes files); duration-based
  shard-count reduction is phase 5. Do NOT reduce shard counts here.
- Step summary: print grep/groups decisions (already required by phase 1 conventions).

### 3d. Reporter caveat

The rocketchat/jira/qase reporters assume full-suite semantics on develop only
(`REPORTER_ROCKETCHAT_REPORT` is set from secrets presence and draft status;
`QASE_REPORT` only on develop). Confirm partial PR runs don't corrupt any dashboard that
assumes a fixed test population; if the internal reporter tracks "missing" tests per run,
coordinate with its owners (ask the requesting team) before enabling. Surface this in the
phase PR description.

## Files to create/modify

- `.github/test-impact-map.yml` (new)
- `.github/scripts/select-tests.mjs` + tests (extend)
- `.github/scripts/lib/glob.mjs` + tests (new, if needed)
- `.github/workflows/ci.yml`, `.github/workflows/ci-test-e2e.yml`
- `apps/meteor/tests/e2e/config/tags.ts` → ensure a machine-readable manifest exists
  (`tags.json` generated or hand-maintained with check script)

## Acceptance criteria

1. PR touching only `apps/meteor/client/views/admin/**`: UI jobs run with
   `--grep '@smoke|@admin'`, API jobs run only the `admin` group, livechat suites skipped,
   `tests-done` green.
2. PR touching `apps/meteor/app/lib/**` (unmapped): full suite, reason `unmapped-files`.
3. PR touching a mapped source AND an unmapped one: full suite.
4. PR touching only `tests/e2e/omnichannel/*.spec.ts`: UI runs `@smoke|@omnichannel`,
   API jobs skipped except the sanity group rule.
5. develop / merge_group: no grep, no groups — full suites (compare test counts with a
   pre-phase run).
6. Selector unit tests cover: rule union, unmapped fallback, smoke always included,
   spec-file self-mapping, metacharacter quoting.
7. Map file has an owner-approved review (humans, not the agent) for every rule.

## Verification

- Unit tests.
- Three real draft PRs (mapped-only, unmapped, mixed) observed end-to-end.
- One develop-simulation: dispatch or merge-queue run confirming zero behavioral diff.

## Rollback

`E2E_SELECTION_DISABLED=true` reverts to full runs (phase 1 switch covers this phase's
logic too — verify the switch short-circuits before map evaluation). Deleting all rules
from the map is an intermediate soft-disable (everything becomes `unmapped-files`).

## Out of scope

Coverage-derived mapping (phase 4/5), shard-count adaptation (phase 5), CE/EE dedup.
