# 11 — Costs, Credits & Pricing

The model bill is the main variable cost. This doc gives a **first estimate** so plan limits aren't pulled out of thin air (the video's weren't). Replace the estimates with measured numbers from `ai_usage` and the eval harness as soon as Phase 08 runs.

---

## 1. Model prices used here

Anthropic list prices for the default model (Nov 2026; re-check before launch):

| Model | Input $/MTok | Output $/MTok | Cache read $/MTok |
|---|---|---|---|
| `claude-opus-5-5` (default) | 4.00 | 20.00 | 0.20 |
| `claude-sonnet-5-5` | 2.00 | 10.00 | 0.20 |
| `claude-haiku-5-5` | 0.10 | 0.50 | — |

`lib/ai/pricing.ts` holds this table as code (`{ model → {inputPerMTok, outputPerMTok, cacheReadPerMTok, cacheWritePerMTok} }`) and is the only place `ai_usage.cost_usd_micros` is computed from. Unknown model → throw, so a model change can't silently bill at $0.

Cache **writes** are billed at a premium over base input. The estimates below assume about 1.25× base input; check current pricing. Thinking tokens count as output tokens.

## 2. Estimated cost per action (Opus 5.5, with prompt caching)

Assumptions: system prefix (rules + contract + UI guide + component docs + exemplars) ≈ 15k tokens; a 3-screen app; foundation files ≈ 15k tokens; one screen file ≈ 5k tokens; thinking about equal to code output.

| Action | Input (mostly cached) | Output (code + thinking) | Est. cost |
|---|---|---|---|
| Plan (or one revision) | ~6k uncached | ~2.5k | **~$0.08** |
| S1 Foundation (≈6 turns) | ~17k cache write + ~130k cache reads | ~25k | **~$0.60** |
| S2 one screen (≈4 turns) | ~140k mostly cache reads + ~10k new | ~11k | **~$0.27** |
| S3 fix pass | — | ~5k | **~$0.15** |
| **Small app (plan + 3 screens)** | | | **≈ $1.60 (range $1–3)** |
| Chat edit (new conversation, ≈3 turns) | ~40k cache write + reads | ~5k | **≈ $0.35 (range $0.20–0.60)** |
| Add screen via chat | | | ≈ $0.40 |
| Theme switch / inline visual edit | none | none | **$0** |
| AI image (ImageKit generation) | provider-priced | | per image, check ImageKit pricing |

Without caching the S2 stage would cost about 3–5× more. Watch `cache_read_input_tokens` (`10-testing-and-evals.md` §2.2).

## 3. Credits

- Internal unit: **micro-credits** (`bigint`). 1 credit = 1,000,000 micro-credits.
- Charge per run: `credits = ceil(sum(ai_usage.cost_usd for run) × CREDIT_MARKUP / CREDIT_USD_VALUE)`, plus fixed fees for non-LLM actions that cost us money (e.g. AI images). `CREDIT_MARKUP` defaults to `1.0`. Margin normally comes from the plan price vs. included credits (§4); raise the markup only if top-up packs need it.
- `CREDIT_USD_VALUE` is the model cost one credit covers. Starting point: **`0.05`**, so a small app is ≈ 32 credits and an edit ≈ 7 credits.
- Reservation (Phase 18): at enqueue, reserve `estimate × 1.5` (plan: 3 credits; build: 15 + 8 × screens; edit: 12). At the end, settle to actual and release the rest. If the balance can't cover the reservation, block before enqueueing.
- Show the user the estimate up front ("Building 3 screens ≈ 35 credits").
- **Non-LLM metered actions:** each AI image generation is charged a flat `IMAGE_GENERATION_CREDITS` (start at 2), because `ai_usage` only covers model calls. Count generations from `project_assets` rows with `provider_ref`.
- **Signup grant:** `FREE_SIGNUP_GRANT` in `lib/credits/plans.ts` defaults to **60 credits** (≈ 1 small app + 3 edits ≈ $2.65 model cost; decision D16). The video's 100 credits would cost ≈ $4.60 per signup (§4).

## 4. What the video's plan limits would cost us

| Plan (from the video) | Allowance | Model cost at Opus 5.5 list |
|---|---|---|
| Free: 100 credits ≈ "2 small apps + 4 edits" | 2 × $1.60 + 4 × $0.35 | **≈ $4.60 per signup** |
| Pro: "25 small apps + 200 edits" per month | 25 × $1.60 + 200 × $0.35 | **≈ $110 per Pro user/month** |

**Implications:**
- At ≈ $4.60 a signup, the free tier needs abuse controls (email verification, rate limits, maybe phone or card for higher limits). Or make the free tier smaller: e.g. 1 app + 3 edits ≈ $2.65.
- A Pro plan with the video's allowance needs a price around **$200+/mo** for healthy margins. Alternatively, keep a lower price with a smaller allowance plus top-up packs. Example, aiming for model cost ≤ 40% of revenue:

| Pro price | Model budget (40%) | Credits @ $0.05 | ≈ Apps or edits |
|---|---|---|---|
| $29/mo | $11.60 | 230 | ~7 apps, or ~33 edits |
| $49/mo | $19.60 | 390 | ~12 apps, or ~55 edits |
| $99/mo | $39.60 | 790 | ~24 apps, or ~110 edits |

These are planning numbers. Set real prices after a week of measured usage.

## 5. Levers to reduce cost (measure each with evals before adopting)

Free wins, adopt by default:
1. **Prompt caching**, arranged as in `05` §5.
2. **Per-route effort**: plan `medium`, edit `medium`, build `high`. Lower effort cuts thinking tokens.
3. **Send only relevant files** to the Editor (selected screen + deps), not the whole project.
4. **Theme switches and inline edits never call the model.**
5. **Max screens on first build** (`MAX_SCREENS_DEFAULT=3`).

Trade-offs, which are your decision (see `12-decisions-and-open-questions.md`):
6. Run specific stages on a cheaper model, e.g. `claude-sonnet-5-5` for S2 screens or edits, or `claude-haiku-5-5` for small text-only edits. Each is a config change (`AI_MODEL_<STAGE>`). Adopt one only if the eval quality metrics hold.
7. A per-run `task_budget` (beta) to cap runaway loops.
8. Batch API (50% off) for **non-interactive** work only, e.g. nightly evals or thumbnail captions. Never for user-facing builds.

## 6. Other running costs

| Item | Who pays | Notes |
|---|---|---|
| Web hosting (Vercel) | us | free → Pro tier |
| Worker host (Railway/Fly/Render) | us | ~$5–25/mo to start; scales with concurrent builds |
| Postgres + storage | us | small at first |
| ImageKit generation/bandwidth | us (platform key) | meter as credits per generated image |
| EAS Build / EAS Hosting | **the user** (their Expo token/account) | Free tier has queues and limits; say so in the UI |
| Generated-app backend (Supabase/CloudHike/…) | **the user** (their account) | |
| Expo Snack / Expo Go | free | |
| Stripe | ~2.9% + 30¢ per charge (region-dependent) | |
| CodeRabbit | us (dev tooling) | optional |
