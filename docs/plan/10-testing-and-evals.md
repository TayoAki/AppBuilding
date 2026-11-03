# 10 — Testing & Agent Evals

The platform code is tested like any web app. The **agent** needs its own measurable quality bar, because a prompt or model change can silently drop the share of apps that compile and render. The video judged quality by eye on single demos. We measure it.

---

## 1. Test layers

| Layer | Tool | What | Where |
|---|---|---|---|
| Unit | Vitest | `lib/crypto`, `lib/credits` (ledger math, reserve/settle), `lib/validate/*`, `lib/codemods/*` (inline-edit ops), `lib/vfs` path rules, plan-limit enforcement, theme contrast fixer, route table builder | `**/*.test.ts` |
| Integration | Vitest + test Postgres (docker / testcontainers) | Route handlers with real DB: authz (other user → 404), project creation, run events append/tail, pg-boss enqueue → handler | `tests/integration/` |
| Preview runtime | Vitest (jsdom) + Playwright | Loader resolution, router shim behaviour, bridge message validation; fixture app renders all routes without errors in Chromium + WebKit | `preview-runtime/**/*.test.ts`, `tests/runtime/` |
| E2E | Playwright | Golden path with a **recorded/mocked model** (§3): sign up → prompt → plan → build → preview → theme → inline edit → export zip | `tests/e2e/` |
| Agent evals | custom harness (`evals/`) | Real model calls on a fixed prompt set; scored automatically (§2) | `npm run eval` |

CI runs unit + integration + runtime + e2e (mocked model) on every PR. The live-model eval runs **on demand** and nightly on `main`, because it costs money.

## 2. Agent eval harness

### 2.1 Dataset (`evals/prompts.jsonl`)
Start with **20 prompts**, then grow it from real user prompts (with consent) and from every bug report.

| Category | Examples | Count |
|---|---|---|
| Classic CRUD | habit tracker, expense tracker with budgets, todo with tags | 5 |
| Content/catalog | grocery store with cart, recipe browser, events list | 4 |
| Media-heavy | travel journal with photos, fitness workout library | 3 |
| Forms/flows | onboarding + sign-in + profile, booking flow | 3 |
| Tricky | very short ("notes app"), very long multi-feature, non-English prompt, reference-image driven, conflicting requirements | 5 |

Plus an **edit set** (`evals/edits.jsonl`), each a starting fixture + instruction:
- "add a settings screen with currency, dark mode and notifications" (must add a screen; the video's P4 bug),
- "add a welcome screen" (must appear first),
- "change the icon on this card to a leaf" with an `element_ref`,
- "sign up throws Cannot read properties of undefined (reading 'user')" with a seeded bug,
- "make it dark" (should prefer a theme switch).

### 2.2 Automatic scores (per app)

| Metric | How | Target (v1) |
|---|---|---|
| Plan valid | `AppPlanSchema` parses; screen count ≤ cap; routes unique | 100% |
| Build completes | run `succeeded` without hitting the turn cap | ≥ 95% |
| Static clean | 0 validator errors (`05` §4) after S3 | ≥ 95% |
| Renders | render smoke test: every route mounts, 0 uncaught errors | ≥ 90% |
| Interactions | scripted Playwright clicks on primary buttons don't throw; navigation targets exist | ≥ 85% |
| Theme discipline | 0 literal colors outside `theme/` | ≥ 95% |
| Edit success | edit set: expected file/screen changed, others untouched (diff check) | ≥ 90% |
| Cost | USD per app (from `ai_usage`), p50/p90 | track; alert on +20% |
| Latency | wall-clock to `run.done`, p50/p90 | track |
| Cache hit | `cache_read_input_tokens / input_tokens` on S2 calls | ≥ 60% |

### 2.3 Design quality (semi-automatic)
- Screenshot every screen. Score each against a short rubric (hierarchy, spacing consistency, contrast, empty states, realistic data, no overflow/clipping) using a model-as-judge call with the screenshot and rubric. Spot-check about 10% by hand each run.
- Keep a `evals/report.html` with the screenshot grid per run, so prompt changes can be compared visually side by side.

### 2.4 Process
- Every change to `lib/ai/prompts/**`, `templates/expo-app/CONTRACT.md`, the component library, tools, or `AI_MODEL`/effort settings must include an eval run in the PR description (before → after on the metrics above).
- Use a **train/test split**. Tune prompts on 14 prompts, and report the 6 held-out ones as the headline number, so you don't overfit to the demo prompts.
- Budget: estimate cost per eval run first (prompts × p50 cost per app) and get approval for a spend ceiling. `npm run eval -- --limit 5` is the quick smoke variant.

## 3. Mocking the model for CI

- `lib/ai/client.ts` accepts an injected transport. In tests, use **recorded fixtures**: real responses captured once with `RECORD=1`, keyed by a hash of (mode, stage, screen id). Replay is deterministic and free.
- Fixtures must contain realistic tool calls (`write_file`, `validate`, …), so the e2e test exercises the real tool executor, VFS, validators, events and preview.
- Re-record fixtures when prompts change materially. The diff in fixture files is itself a useful review artifact.

## 4. What "done" means for the agent work (Phases 06, 08, 09)

- Unit tests for schemas, tools (path allowlist, staleness check), the validators and the loop's stop-reason handling (`refusal`, `max_tokens`, `pause_turn`, tool error → `is_error`).
- Mocked-model e2e passes.
- A live eval run on the full set meets the v1 targets in §2.2, or the PR documents the gap and a plan to close it.
