# 11 — Costs, Credits & Pricing

Model spend is the main variable cost. This doc sets the **credit model** (flat, per action, as in the reference app) and estimates what each action costs us. Replace the estimates with measured numbers from `ai_usage` and the eval harness as soon as Phase 08 runs.

---

## 1. Model prices

Anthropic list prices (Nov 2026; re-check before launch). `lib/ai/pricing.ts` holds this table as code (`{model → {inputPerMTok, outputPerMTok, cacheReadPerMTok, cacheWritePerMTok}}`). It's the only place `ai_usage.cost_usd_micros` is computed, and an unknown model throws.

| Model | Input $/MTok | Output $/MTok | Cache read $/MTok |
|---|---|---|---|
| `claude-opus-5-5` (default for both `AI_MODEL` and `AI_BUILDER_MODEL`) | 4.00 | 20.00 | 0.20 |
| `claude-sonnet-5-5` (what the reference app runs) | 2.00 | 10.00 | 0.20 |
| `claude-haiku-5-5` | 0.10 | 0.50 | — |

Cache writes cost a premium over base input; the estimates assume about 1.25×. Thinking tokens bill as output.

## 2. Estimated model cost per action (Opus 5.5, generator pipeline, with caching)

Assumptions: generator system prompts ≈ 3–4k tokens; APP CONTEXT (app, data, component APIs, routes) ≈ 3k; a screen ≈ 3.5k output tokens of code + ≈ 3k thinking; repairs needed on ~30% of files.

| Action | Input | Output (code + thinking) | Est. cost |
|---|---|---|---|
| Planner message (plan / revision / chat) | ~6k, mostly cached | ~3.5k | **~$0.06–0.09** |
| Foundation (5 themes + collections JSON) | ~3k | ~6k | **~$0.13** |
| Component library (5–10 components JSON) | ~5k | ~10k | **~$0.22** |
| One screen (new or rewritten) | ~10k, ~60% cached after the first | ~6.5k | **~$0.16** (+ ~$0.03 average repair) |
| One component (Component Builder) | ~5k | ~2.5k | **~$0.06** |
| **Small app: plan + 4 screens (incl. Sign In)** | | | **≈ $1.20 (range $0.80–2.00)** |
| Change round touching 1 screen | Planner + 1 screen | | **≈ $0.25** |
| Change round touching 2 screens + 1 component | | | **≈ $0.50** |
| Theme switch, inline visual edit, plan card confirm | — | — | **$0** |
| AI image (ImageKit) | provider-priced | | per image (check ImageKit pricing) |

On `claude-sonnet-5-5` the same actions cost about half.

## 3. Credit model (default: flat prices, as in the reference app)

All prices live in `lib/credits/plans.ts`. The pricing page, upgrade dialogs and the Planner's PROJECT STATE read them from there.

| Action | Credits |
|---|---|
| New screen (first build or added later) | **10** |
| Round of changes (one `update_app` that edits existing screens/components) | **5**, covering up to 2 regenerated files; +3 per additional file (our addition, see §5) |
| AI image generated or transformed in edit mode | **1** |
| Chat, planning, plan revisions, theme switches, inline visual edits | **free** |
| Minimum balance to send the Planner a message | **5** |

Mechanics (Phase 18):
- **Two balances.** `monthly_credits_balance` (Pro allowance, spent first, replaced at renewal, zeroed when Pro ends) and `credits_balance` (never expires: signup grant, refunds, top-ups).
- **Charge before work starts**, in one conditional `UPDATE … WHERE balance >= cost`, so two tabs can't overspend. Tools pre-check only so the Planner can explain a refusal; the charge decides.
- **Refunds** are computed from what's unbuilt (a failed build refunds exactly what retrying would charge), and each has a unique ledger `reference`, so a retried step can't refund twice. Resuming an interrupted build isn't charged again.
- `ai_usage` still records real cost per run, so margins are visible per action type (dashboard in Phase 19).

Why flat rather than cost-based: people can predict it ("4 screens = 40 credits"). Cost-based pricing (`credits = ceil(cost × CREDIT_MARKUP / CREDIT_USD_VALUE)`) stays available as a config option if margins drift.

## 4. Plans (defaults, from the reference app; decision D16)

| | Free | Pro |
|---|---|---|
| Credits | **100 one-time**, never expire (≈ 2 small apps + 4 edits) | **1,000 per month**, unused expire at renewal (≈ 25 small apps or 200 edits) |
| Projects | 2 | unlimited |
| Screens per project | 5 | 10 |
| ZIP export, Push to GitHub | ✅ | ✅ |
| Publish as web | ✅ | ✅ |
| Android/iOS builds | ❌ | ✅ |
| Price | $0 | **to decide** (see below) |

What it costs us at Opus 5.5 list prices, using §2:

| Usage | Model cost |
|---|---|
| Free signup that spends all 100 credits (2 apps + 4 one-screen edits) | ≈ 2 × $1.20 + 4 × $0.25 ≈ **$3.40** |
| Pro user spending 1,000 credits only on new screens (100 screens ≈ 25 apps) | ≈ **$30** |
| Pro user spending 1,000 credits only on 2-file change rounds (200 rounds) | ≈ **$100** |

**Pricing guidance.** Aim for model cost ≤ 40% of revenue on a typical mix. That puts Pro at roughly **$49–$99/month on Opus 5.5**, or **$25–$49 on Sonnet 5.5** if evals show equal quality (D9). The free tier at ≈ $3.40 per fully used signup needs email verification and rate limits (`08` §7).

## 5. Keeping cost per credit bounded

The flat model's risk is that change rounds vary: one round can regenerate 1 file or 6. Mitigations, in order:
1. **Per-round file cap**, as in §3: 5 credits covers up to 2 regenerated files, +3 per extra file. The Planner tells the person the cost before large changes.
2. **Change as little as possible**: Planner rules prefer one component edit over several screen rewrites (`05` §3.2).
3. **Prompt caching** with the APP CONTEXT prefix shared across a build (`05` §7). Without it, screens cost 2–3× more.
4. **Per-agent effort**, plus `AI_BUILDER_MODEL` (D9) once evals justify it.
5. Batch API (50% off) for **non-interactive** work only (evals, thumbnails), never user-facing builds.

## 6. Other running costs

| Item | Who pays | Notes |
|---|---|---|
| Web hosting (Vercel) | us | |
| Worker host (Railway/Fly/Render) | us | ~$5–25/mo to start |
| Postgres + storage | us | small at first |
| ImageKit generation/bandwidth | us (platform key) | charged as 1 credit per edit-mode image; build-time images (≤ 8 per step) are included in the screen price |
| EAS Build / EAS Hosting | **the user** (their Expo account/token) | Free tier queues; say so in the UI |
| Generated-app backend | **the user** (their provider account) | The Planner names what it deploys and that it's billed to them |
| Expo Snack / Expo Go | free | |
| Stripe | ~2.9% + 30¢ per charge (region-dependent) | |
