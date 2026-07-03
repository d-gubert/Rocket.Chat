# Phase 2 — Suite partitioning: domain tags and subset invocation

## Goal

Make both E2E suites addressable in subsets, without changing what CI runs yet:

1. Every Playwright spec file carries a domain tag (`@omnichannel`, `@admin`, …) and a small
   set of specs carries `@smoke`.
2. The mocha API suite's single spec glob is split into named domain groups that can be run
   individually.
3. Both suites accept an "run only this subset" invocation that later phases' selection
   logic will drive.

This phase is a no-op for CI behavior (full suites still run, same wall-clock). It can be
executed in parallel with phase 1.

## Prerequisites

Read `00-context.md`. No other phase required.

## Work items

### 2a. Tag Playwright specs by domain

Playwright supports tags natively: `test.describe('...', { tag: '@omnichannel' }, () => …)`
and CLI `--grep '@omnichannel'`. The suite is on a Playwright version ≥1.42 (verify in
`apps/meteor/package.json`; if older, tags-in-titles + `--grep` works identically).

- Define the domain taxonomy in a new file `apps/meteor/tests/e2e/config/tags.ts` exporting
  a const list. Derive domains from the existing file naming, roughly:
  `@accounts` (account-*, login, forgot-password, delete-account, enforce-2FA, oauth, iframe-auth),
  `@admin` (admin-*, administration*, permissions, settings-related),
  `@messaging` (message-*, messaging*, emojis, jump-to-thread, mark-unread, prune, export-messages),
  `@rooms` (channel-management, create-channel, create-direct, create-discussion, teams-related, preview-public-channel),
  `@omnichannel` (omnichannel/**),
  `@e2ee` (e2e-encryption/**),
  `@files` (file-upload, files-management, image-gallery, image-upload, avatar-settings),
  `@apps` (apps/**),
  `@search` (global-search),
  `@misc` (everything left: homepage, presence, feature-preview, notification-sounds, calendar, imports, email-inboxes, embedded-layout, anonymous-user, banners…).
  The exact grouping is the executor's judgment call — the requirements are: every spec file
  has exactly one domain tag, dirs map to a single tag, and the taxonomy is written down in
  `apps/meteor/tests/e2e/README.md`.
- Apply the tag on the top-level `test.describe` of each spec file. Files with multiple
  top-level describes get the tag on each.
- Add an ESLint guard or a small check script `apps/meteor/tests/e2e/config/check-tags.mts`
  (run in the existing lint/`checks` pipeline or as a Playwright `globalSetup` assertion in
  local runs) that fails when a spec file has no domain tag. Without enforcement the
  taxonomy rots.

### 2b. Smoke subset

Tag with `@smoke` a minimal high-signal set (~10–15 minutes total, target ≤10 spec files):
login, send/receive message in a channel, create channel, basic admin settings page load,
one omnichannel conversation flow, one API-adjacent UI flow. Selection criteria: broad
server-path coverage per minute, historically LOW flakiness (consult the team or the
reporter dashboards; when in doubt prefer boring stable specs). `@smoke` is additive to the
domain tag.

### 2c. Split mocha API suite into named groups

In `apps/meteor/.mocharc.api.js` the spec list is a single array. Refactor:

- New file `apps/meteor/tests/end-to-end/api/groups.json` mapping group name → array of
  globs, covering **exactly** the current spec set (union of groups == current globs;
  enforce with a check script, see below). Suggested groups by file: `rooms`
  (channels, groups, rooms, direct-message, teams, invites, subscriptions), `chat`
  (chat, threads-related, moderation, autotranslate), `users` (users, roles, permissions,
  presence, failed-login-attempts, guest-permissions, personal-tokens), `admin`
  (settings, licenses, statistics, banners, custom-sounds, custom-user-status,
  emoji-custom, assets*, oauth*, incoming/outgoing-integrations, import, cloud, push),
  `misc` (everything else incl. `methods/**`, `helpers/**` stays always-loaded),
  `apps` (`tests/end-to-end/apps/*`).
- `.mocharc.api.js` reads `API_TEST_GROUPS` env (comma-separated group names; empty/unset →
  all groups) and builds `spec` from the JSON. Keep `helpers/**` in every invocation.
- Add `apps/meteor/tests/end-to-end/api/check-groups.mjs` verifying every file matched by
  the historical globs is claimed by ≥1 group and that groups don't reference missing
  files. Wire it into `testapi`'s pretest or the unit-test job.
- Livechat config stays as-is (it is already a partition).

### 2d. Subset invocation contract

Later phases pass subsets via env vars, so the reusable workflow only needs env changes:

- UI: `E2E_GREP` (Playwright `--grep` value, e.g. `@omnichannel|@smoke`) and/or
  `E2E_SPEC_FILES` (space-separated file args). Modify the `yarn test:e2e` step contract:
  when `E2E_GREP` is set, append `--grep "$E2E_GREP"`. Document in
  `apps/meteor/tests/e2e/README.md`. (The actual CI wiring lands in phase 3; this phase
  only makes local invocation work: `E2E_GREP='@smoke' yarn test:e2e` must run only smoke.)
- API: `API_TEST_GROUPS=rooms,chat npm run testapi` runs two groups.

## Files to create/modify

- ~90 files under `apps/meteor/tests/e2e/**/*.spec.ts` (tag insertion — mechanical; a
  codemod/regex pass per file group is acceptable, review the diff).
- `apps/meteor/tests/e2e/config/tags.ts`, `apps/meteor/tests/e2e/config/check-tags.mts`
- `apps/meteor/tests/e2e/README.md` (taxonomy + invocation docs)
- `apps/meteor/.mocharc.api.js`, `apps/meteor/tests/end-to-end/api/groups.json`,
  `apps/meteor/tests/end-to-end/api/check-groups.mjs`

## Acceptance criteria

1. `yarn test:e2e --list` (no server needed) succeeds and every spec file appears with a
   domain tag; the check script passes and fails when a tag is removed.
2. `E2E_GREP='@smoke' yarn test:e2e --list` lists only the smoke specs, total ≤ ~12 files.
3. `API_TEST_GROUPS=rooms npm run testapi -- --dry-run` (or equivalent listing; mocha has no
   dry-run — acceptable substitute: a tiny script printing the resolved `spec` array from
   the config) resolves to only the `rooms` group globs + helpers.
4. `check-groups.mjs` proves group union == previous glob coverage; CI full runs are
   byte-identical in which test files execute (compare test counts against a develop run).
5. No test logic changed — diff on spec files is tags only.

## Verification

- Run `yarn test:e2e --list` before and after; assert same test count.
- Run one full `testapi` locally against a dev server OR rely on the next CI run's test
  count comparison (report both counts in the PR description).

## Rollback

Tags and groups are inert metadata; revert the PR if anything misbehaves. No CI behavior
depends on this phase until phase 3.

## Out of scope

Any CI wiring (phase 3), duration-based sharding (phase 5), reducing the CE/EE duplication.
