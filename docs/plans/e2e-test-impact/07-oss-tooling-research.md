# Phase 7 (reference) — OSS tooling research for test impact analysis

Research snapshot **as of 2026-07-03**, verified against live GitHub/npm metadata and
official docs. Scope: open-source only; commercial services (Launchable, Datadog TIA,
Sealights, Teamscale, Knapsack Pro, Currents, Develocity, CloudBees Smart Tests) were
deliberately excluded. Low-authority repositories (abandoned, toy/POC, <100 stars with no
adoption signal) were checked and excluded — listed by name at the end so nobody re-treads
them.

## Headline finding

**No authoritative off-the-shelf OSS JavaScript/TypeScript test-impact-analysis tool exists
for our situation** (Playwright + mocha E2E suites exercising a server across an HTTP
boundary inside Docker). Everything OSS in the JS ecosystem falls into one of:

1. Runner-built-in *static import-graph* selection — Playwright `--only-changed` (v1.46+),
   Jest `--changedSince`, Vitest `--changed`. Structurally blind to server code reached
   over HTTP: a spec file imports page objects, not the API handlers it exercises.
2. *Package-level* monorepo gating — Turborepo `--affected`, Nx `affected`. Useless inside
   `apps/meteor`, which is one package owning the entire E2E surface.
3. *Path-rule routing* — dorny/paths-filter. Human-curated, no analysis.
4. Zero-adoption POCs (see exclusions).

An exhaustive GitHub search for "test impact analysis" in JS/TS returns only zero-star
2025–2026 POC/hackathon projects. The mature coverage-map implementations live in other
ecosystems, and the one dedicated Java OSS TIA product (skippy) was archived in Oct 2025.
**Conclusion: the build-on-primitives approach in phases 1–5 is the realistic path**, and
the primitives below are healthy.

## Recommended stack (maps to plan phases)

| Layer | Tool | Authority signals (2026-07-03) | Plan phase |
|---|---|---|---|
| Package-graph gating | **Turborepo** `--affected` / `turbo ls --affected --output=json` | Already in repo at 2.9.14 (feature needs ≥2.1; current stable 2.10.2); Vercel-backed, ~30.6k★, MIT | 1 |
| Path-rule gating | **dorny/paths-filter** v4, **pinned by commit SHA** | 3.2k★, v4.0.2 released 2026-07-02, ~57.6k dependent repos (Sentry, GoogleChrome); minimal permission surface | 1, 3 |
| Test-code-direction signal | **Playwright `--only-changed=<ref>`** | Built into Playwright since v1.46 (Microsoft); transitive import graph of spec files; needs `fetch-depth: 0` | 3/5 (auxiliary) |
| Coverage data plane | **istanbul-lib-coverage** + **babel-plugin-istanbul** | Already our instrumentation. babel-plugin-istanbul v8.0.0 (2026-04), nyc v18 (2026-02). Ecosystem is in *caretaker mode* (reactive, dependency-driven releases; primary maintainer bcoe) but the istanbul JSON format is a de-facto standard (jest, vitest, codecov consume it) and `CoverageMap.merge()/toJSON()` is exactly the snapshot/diff primitive phase 4 needs. Pin versions; don't count on fast upstream fixes; format-churn risk ≈ zero | 4 |
| Per-spec Playwright attribution | **Vendored fixture, mxschmitt pattern** | mxschmitt/playwright-test-coverage (142★, pushed 2026-05) is a *demo repo by a Microsoft Playwright team member*, not an npm dep — vendor the ~50-line fixture, write one JSON per `testInfo.testId`. Matches our existing `window.__coverage__` machinery | 4 |
| Per-spec mocha attribution | **Custom root-hook plugin** (~100 lines) | Upstream confirmed no package exists (mochajs/mocha#4534 closed unresolved; mocha-istanbul dead pre-2016). Mocha Root Hook Plugins (v8+, `--require`) are the supported primitive; snapshot the server coverage endpoint in `afterEach`/per-file | 4 |
| Static import graph (client-side mapping aid) | **dependency-cruiser** v18 | 6.8k★, v18.0.0 2026-06-25, 36 open issues across 388 releases — strongest maintenance signal in its category; tsconfig `paths`, workspace-aware, JSON output, `reaches` rules do "which entrypoints reach this changed file" natively | 3 (optional map authoring aid) |
| Duration-based shard balancing | **Custom greedy bin-packer** (~40 lines over Playwright JSON-report durations) | Playwright sharding is still file/count-based; timing-based sharding is feature request microsoft/playwright#17969, open since 2022, `P3-collecting-feedback`. v1.57's "Speedboard" is observability only. No credible OSS balancer exists (only blog gists and SaaS). Blogs claiming a `--shard-weights` flag in 1.57 could not be verified in official release notes — treat as false | 5 |
| Merged human-readable reports (optional) | **monocart-coverage-reports** | 155★ but better authority than stars suggest: embedded by c8 (v10.1+), active (v2.12.12, 2026-05), used in Playwright ecosystem; single maintainer (bus-factor 1) — use as reporting garnish, not a load-bearing format dependency | 4 (optional) |

## Production-proven reference designs (other ecosystems — steal the architecture)

- **GitLab's crystalball setup (Ruby/RSpec)** — the closest production blueprint to phases
  4–5: scheduled full instrumented runs produce a packed source-file→example mapping
  (`packed-mapping.json.gz`); MR pipelines download it and select predictive specs; hard
  fallback-to-full rules. Documented in GitLab's pipeline development docs. Upstream
  `pluff/crystalball` (352★) is dormant since 2023, but GitLab maintains its own fork and
  ran it in production at massive scale — the *design* is validated even if the gem isn't
  ours to use.
- **pytest-testmon (Python)** — 993★, actively maintained (pushed 2026-07-02). Per-test
  Coverage.py data → SQLite DB of test↔code dependencies with method/class fingerprints;
  invalidates tests whose fingerprinted deps changed; always runs new/failed tests. The
  most mature OSS coverage-map RTS anywhere; its invalidation rules are worth copying
  into phase 5's selector.
- **Ekstazi (Java, UT Austin research)** — the academic gold standard for *safe* dynamic
  RTS; validated that **file-level-but-dynamic beats fine-grained-but-static** selection
  (studies vs STARTS). Supports phase 4's choice of file-granularity coverage maps over
  static analysis.

## Supply-chain due-diligence findings

- **tj-actions/changed-files — do not standardize on it.** CVE-2025-30066 (2025-03-14):
  attackers rewrote nearly all its version tags to a malicious commit that dumped runner
  memory (secrets) to logs; ~23k repos affected; CISA alert. Patched, still maintained,
  but it demonstrated the mutable-tag attack and has a larger code surface than the
  alternative. Use dorny/paths-filter instead, and **pin every third-party action by full
  commit SHA** (the repo already does this — keep it that way for new actions).
- **Nx "s1ngularity" compromise (Aug 2025, CVE-2025-10894):** malicious `nx` npm versions
  live for ~4h stole tokens/keys via postinstall; 190+ orgs impacted. Nx published a
  transparent postmortem and moved to trusted publishing. Not disqualifying, but a reason
  to prefer sticking with Turborepo (equivalent affected-detection, already installed)
  over adding Nx side-by-side. If Nx is ever adopted: exact-version pins, `--ignore-scripts`
  in CI.
- **istanbul caretaker-mode risk** is acceptable: the July-2023 "unmaintained" scare
  (vitest discussion #3786) ended with releases resuming; 2026 releases exist for nyc and
  babel-plugin-istanbul. The mitigation is pinning + owning our thin layer on top of
  `istanbul-lib-coverage`, which phases 4–5 already assume.

## Evaluated and excluded (with reasons — don't re-tread)

**Monorepo/build-graph tier:**
- **Nx** — healthy OSS (v23, MIT, ~29k★) and `nx affected` can be adopted plugin-less next
  to yarn workspaces, but redundant with Turborepo for our purposes + s1ngularity history.
  Situational, not recommended here.
- **moonrepo (moon)** — good tech (task-level file inputs, `moon query touched-files`,
  v2.3.5 active), but requires re-platforming task orchestration off Turborepo and is a
  2-founder YC company with bus-factor risk. Lean exclude.
- **Bazel + rules_js/bazel-diff/target-determinator** — the tools are healthy (bazel-diff
  v30, 2026-07; target-determinator v0.34) and give true target-level selection, but
  `apps/meteor` is built by Meteor's isobuild → one opaque mega-target, destroying the
  granularity you'd be buying; plus rules_js requires a pnpm lockfile (we're yarn 4).
  Multi-quarter migration for negated payoff. Exclude.
- **Pants** — JS/TS backend still experimental after 3+ years; yarn Berry unsupported. Exclude.

**JS TIA tier:** **testpick** (V8-runtime-coverage selection for Jest/Vitest — architecturally
the right idea, but 2 stars, two weeks old, solo author; watch, don't use) · **BuildLens**,
**test-impact-analyzer**, **testmate**, **test-funnel** (0–22★ POCs/abandoned) · **skippy**
(Java, archived 2025-10 — cautionary tale about niche TIA dependencies) · **pytest-rts**
(abandoned 2021).

**Coverage/graph tier:** **anishkny/playwright-test-coverage** npm package (stale since
2023-03, <100★ — vendor the pattern instead) · **bgotink/playwright-coverage** (51★,
V8-based → Chromium-only/per-page, wrong fit) · **madge** (10k★ but caretaker mode, last
release 2024-08, maintainer seeking funding — don't start new tooling on it) · **ts-morph**
(healthy but wrong altitude — you'd hand-roll module resolution dependency-cruiser already
does) · **Playwright's native `page.coverage` API** (Chromium-only, per-page, coverage not
guaranteed across navigation — worse than our existing istanbul route) · **vite-plugin-istanbul**
(healthy but N/A — our client build isn't Vite) · blog-gist shard balancers (toys).

## Implications for the plan (deltas to fold into phases)

1. **Phase 1**: implement the workspace-closure step with `turbo ls --affected
   --output=json` (available at our 2.9.14; `--output` is still flagged experimental —
   keep the static-map fallback the phase already specifies). Known sharp edge: root
   `package.json`/`yarn.lock`/`turbo.json` changes mark *everything* affected — consistent
   with our global-invalidation list, but expect frequent full runs on dep-bump PRs.
   dorny/paths-filter (SHA-pinned) is a sanctioned alternative to the hand-rolled glob
   matcher if the dependency is acceptable.
2. **Phase 3**: optionally add Playwright `--only-changed` as an *additive* signal for the
   test-code direction (changed helpers/POMs select their dependent specs automatically);
   never as the sole selector. dependency-cruiser can help *author* curated-map rules
   (client-side reachability) but shouldn't run in the PR hot path initially.
3. **Phase 4**: vendor the mxschmitt fixture pattern for per-spec client coverage keyed by
   `testId`; the mocha side is confirmed custom-code territory (root hooks). Keep the
   cumulative-snapshot+diff design — it matches how crystalball/testmon handle shared-server
   attribution noise. Attribution passes must run with concurrency 1 (our CI already uses
   `workers: 1`).
4. **Phase 5**: confirmed there is no OSS duration-based shard balancer worth adopting —
   the custom bin-packer stays in scope. Steal testmon's invalidation rules (always run
   new + previously-failed specs).
5. **Phase 6 (vendor track)**: the OSS survey strengthens the build case — the buy-side
   evaluation can be deprioritized until phases 1–3 have shipped and produced savings data.
