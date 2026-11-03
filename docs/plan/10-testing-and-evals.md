# 10 — Testing & Agent Evals

The platform code is tested like any web app. The **agent** needs its own measurable quality bar, because a prompt or model change can silently drop the share of apps that compile and render. The video judged quality by eye on single demos. We measure it.

---

## 1. Test layers

| Layer | Tool | What | Where |
|---|---|---|---|
| Unit | Vitest | `lib/crypto`, `lib/credits` (ledger math, reserve/settle), `lib/validate/*`, `lib/codemods/*` (inline-edit ops), `lib/vfs` path rules, plan-limit enforcement, theme contrast fixer, route table builder | `**/*.test.ts` |
| Integration | Vitest + test Postgres (docker / testcontainers) | Route handlers with real DB: authz (other user → 404), project creation, project-scoped run events append/tail, charge + run + outbox in one transaction → dispatcher → pg-boss → handler | `tests/integration/` |
| Preview runtime | Vitest (jsdom) + Playwright | Loader resolution, router shim behaviour, bridge message validation; fixture app renders all routes without errors in Chromium + WebKit | `preview-runtime/**/*.test.ts`, `tests/runtime/` |
| E2E | Playwright | Golden path with a **recorded/mocked model** (§3): sign up → prompt → plan → build → preview → theme → inline edit → export zip | `tests/e2e/` |
| Agent evals | custom harness (`evals/`) | Real model calls on a fixed prompt set; scored automatically (§2) | `npm run eval` |

CI runs unit + integration + runtime + e2e (mocked model) on every PR. The live-model eval runs **on demand** and nightly on `main`, because it costs money.

## 2. Agent eval harness

### 2.1 Dataset (`evals/prompts.jsonl`)
**30 prompts** (ADR-003 #12), grown later from real user prompts (with consent) and from every bug report. **15 are held out** (never looked at while tuning prompts) and are the headline number.

| Category | Examples | Count (tune / held-out) |
|---|---|---|
| Classic CRUD | habit tracker, expense tracker with budgets, todo with tags | 4 / 4 |
| Content/catalog | grocery store with cart, recipe browser, events list | 3 / 3 |
| Media-heavy | travel journal with photos, fitness workout library | 2 / 2 |
| Forms/flows | onboarding + sign-in + profile, booking flow | 3 / 3 |
| Tricky | very short ("notes app"), very long multi-feature, non-English prompt, reference-image driven, conflicting requirements | 3 / 3 |

Plus an **edit set** (`evals/edits.jsonl`, 12 cases), each a starting fixture + instruction:
- "add a settings screen with currency, dark mode and notifications" (must add a screen; the video's P4 bug),
- "add a welcome screen" (must become `firstScreenId` and link onward),
- "change the icon on this card to a leaf" with a `<selected-element>` block,
- "sign up throws Cannot read properties of undefined (reading 'user')" with a seeded bug,
- "make it dark" (must use `set_theme`, no screen rebuilt),
- "add a screen" after an earlier request for a different screen in the same session (must add only the latest),
- "make the cards rounder" (must edit one component, not screens),
- remove a screen that two others link to (plan-guard must force those into `editScreens`),
- a hand-edited screen followed by an unrelated change (hand edit survives, Phase 10b),
- plus 3 regression cases from bug reports.

Each case runs with **2 seeds** (two independent runs) per eval, because a single run can't separate a real regression from variance.

### 2.2 Automatic scores

Targets are **counts** on the held-out set across 2 seeds (15 prompts × 2 = 30 runs), so "≥ 27/30" is unambiguous.

| Metric | How | Target (v1, held-out) |
|---|---|---|
| Plan valid | `propose_plan` input passes plan-guard first try (cap, unique ids, reachability, tabs 3–5, `firstScreenId`) | ≥ 28/30 (and 30/30 after the guard's tool-error retry) |
| Build completes | build run reaches `run.done` (`succeeded`), no screen left `error` | ≥ 27/30 |
| Files pass validation after ≤ 1 repair | validators (`05` §6) clean for every generated file after its single repair round | ≥ 27/30 runs with 0 failing files |
| First-try pass rate | share of generated files that pass validation **without** a repair | track (≥ 70% of files); alert on −10 points |
| Renders | render smoke test (sandbox): every route mounts, 0 uncaught errors | ≥ 27/30 |
| Interactions | scripted Playwright taps on primary buttons don't throw; every `router.push` target exists | ≥ 25/30 |
| Theme discipline | 0 color literals in generated files (except person-chosen ones in edit cases) | ≥ 28/30 |
| Component use | screens import ≥ 1 shared component; no component redrawn inline (heuristic) | ≥ 25/30 |
| Edit success | edit set: expected files/screens changed, others untouched (diff check), correct tool chosen | ≥ 21/24 (12 cases × 2 seeds) |
| Cost | USD per app (from `ai_usage`, incl. repairs), p50/p90 | track; alert on +20%; p50 within 1.5× of `11` §2 |
| Latency | wall-clock confirm → `run.done`, p50/p90 | track; the target is set from S2 measurements (ADR at G0) |
| Cache hit | `cache_read_input_tokens / input_tokens` on Screen Builder calls after the first screen of a build | ≥ 60% |

### 2.3 Design quality (semi-automatic)
- Screenshot every screen (sandbox render). Score each against a short rubric (hierarchy, spacing consistency, contrast, empty states, realistic data, no overflow/clipping) using a model-as-judge call with the screenshot and rubric. Spot-check about 10% by hand each run.
- Keep a `evals/report.html` with the screenshot grid per run, so prompt changes can be compared visually side by side.

### 2.4 Process
- Every change to `lib/ai/prompts/**`, `templates/expo-app/CONTRACT.md`, the generator rules, Planner tools, or `AI_MODEL`/`AI_BUILDER_MODEL`/effort settings must include an eval run in the PR description (before → after on the metrics above, held-out counts first).
- Tune on the 15 tuning prompts only. Report the 15 held-out prompts as the headline. Rotate a few new held-out prompts in each quarter so the set doesn't get memorized by prompt tweaks.
- Budget: estimate cost per eval run first (30 runs × p50 cost per app, plus edits) and get approval for a spend ceiling. `npm run eval -- --limit 5 --seeds 1` is the quick smoke variant.

## 3. Mocking the model for CI

- `lib/ai/client.ts` accepts an injected transport. In tests, use **recorded fixtures**: real responses captured once with `RECORD=1`, keyed by a hash of (mode, stage, screen id). Replay is deterministic and free.
- Fixtures must contain realistic Planner tool calls (`propose_plan`, `update_app`, …) and generator outputs (foundation JSON, component library JSON, screen files, including one that needs a repair), so the e2e test exercises the real tool executor, pipeline steps, VFS, validators, events and preview.
- Re-record fixtures when prompts change materially. The diff in fixture files is itself a useful review artifact.

## 4. What "done" means for the agent work (Phases 06, 08, 09)

- Unit tests for: plan-guard (incl. `update_app` resulting-plan checks), generator output schemas, validators, the repair round (one rewrite, then keep-old / mark-error), the Planner loop's stop-reason handling (`refusal`, `max_tokens`, `pause_turn`, tool error → `is_error`), PROJECT STATE placement (last entry, after the user turn), and **append-only replay** (a test that fails if any earlier `messages.payload.api` entry changes between turns).
- Worker tests: step jobs never finish the run early; a retried step after a crash takes over the lease; `failRun` only on the final attempt; outbox dispatch is idempotent.
- Mocked-model e2e passes.
- A live eval run meets the §2.2 held-out targets, or the PR documents the gap and a plan to close it.
