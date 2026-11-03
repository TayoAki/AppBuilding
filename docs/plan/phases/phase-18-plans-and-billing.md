# Phase 18 — Plans, Credits and Billing (Free & Pro)

| | |
|---|---|
| Video chapter(s) | Ch.20 "Free & Pro Plan Setup" (02:40:52) · reference repo ch. 15 |
| Branch | `feat/18-plans-and-billing` |
| Depends on | Phase 02 (users, ledger, signup grant), Phase 05 (runs/jobs), Phase 06 (`ai_usage` logging), Phases 08–09 (build/change pipelines), Phases 12 and 15–17 (gated features) |
| Effort (solo dev + Claude Code) | ~4 days (was 4–5; flat prices remove cost estimation and reserve/settle) |
| New env vars | `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_PRO_MONTHLY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`. `CREDIT_USD_VALUE` / `CREDIT_MARKUP` are used only by the optional cost-based mode and the margin report. |

## Goal
Turn the builder into a SaaS with **simple, predictable prices**: 10 credits per new screen, 5 per round of changes, 1 per AI image. Chatting, planning and theme switches are free. Free users get 100 one-time credits, 2 projects and 5 screens per project, and can publish to the web but not build native binaries. Pro subscribers get 1,000 monthly credits (spent first), unlimited projects, 10 screens per project and native builds. Limits live in one file and are enforced on the server. Stripe handles checkout, the customer portal and renewals.

## What we do differently from the video (and why)
- **Usage was logged long before billing.** In the video, `credits = 100` never changed until this chapter. We've written an `ai_usage` row per API response since Phase 06. It isn't used to price actions any more, but it powers a **margin report** (credits charged vs real model cost per plan), which tells you when prices or `AI_BUILDER_MODEL` need to change (`11-costs-and-pricing.md`).
- **Append-only ledger, not a mutable counter.** `credit_ledger` is the source of truth. The two balance columns are caches updated in the same statement or transaction. Every row has a **unique `reference`**, which makes refunds, renewals and webhook retries idempotent.
- **Billing behind `BillingProvider`.** Stripe is the default. Regional providers (e.g. Razorpay, as the video mentions for India) can be added without touching gates or the ledger.
- **RevenueCat is not for this.** RevenueCat handles in-app purchases inside *generated mobile apps*, a possible future feature for users' apps. This web SaaS bills through Stripe.

**Reference app notes** (`13-reference-implementation.md`, R15–R16)
- **R15, flat prices instead of cost-derived credits:** users understand "a screen costs 10" better than token math. A small 4-screen app is 40 credits, so Free's 100 ≈ "2 small apps + 4 edits" and Pro's 1,000 ≈ "25 small apps or 200 edits". We adopt this as the default. The cost-based formula from the first plan version stays as an optional mode (`PRICING_MODE = "cost"` in `plans.ts`).
- **Two balances:** `credits_balance` never expires (signup grant, refunds, top-ups). `monthly_credits_balance` is Pro's allowance and is **spent first**. A renewal *replaces* the monthly balance, ending Pro zeroes it, and Free credits survive both. **Refunds always go to `credits_balance`.**
- **Charge before anything is saved,** in **one conditional `UPDATE`** (`… WHERE monthly + credits >= cost`), so two tabs can't go below zero. Tool-side checks exist only so the Planner can explain why. The charge decides.
- **Refunds are computed from what's unbuilt:** `buildCost(unbuilt)` equals exactly what a retry would charge. Each refund has a unique `reference` (`refund:<runId>:<screenId>`), so a retried step can't pay out twice. **Resuming an interrupted build isn't charged again.**
- **Stripe (R16):** re-read the subscription from Stripe on every webhook, because events arrive out of order. Credits come **only** from a *paid* invoice whose `billing_reason` is `subscription_create` or `subscription_cycle`. Downgrade sets `cancel_at_period_end`, so Pro lasts until the paid period ends, when `customer.subscription.deleted` moves the user to Free.
- **Downgraded users** keep their projects and bigger apps and can edit them, but can't add beyond Free limits (no 3rd project, no new screen past 5, no native builds).
- **Upgrade prompts:** API errors from a limit carry `code` (an `UpgradeReason`), and the agent event stream has an `upgrade` event (`05` §8). Both open the same `UpgradeDialog`.

## Steps

### 1. Plan config (`lib/credits/plans.ts`, the single source of truth)
Every limit and price lives here. The pricing page, popups, gates and Planner state all read from it.
```ts
export const PRICES = { screen: 10, changeRound: 5, aiImage: 1 } as const;  // credits
export const MIN_BALANCE_TO_MESSAGE = 5;
export const FREE_SIGNUP_GRANT = 100;

export const PLANS = {
  free: { name: "Free", monthlyCredits: 0,    maxProjects: 2,        maxScreens: 5,
          nativeBuilds: false, webPublish: true, exportZip: true, githubPush: true, queuePriority: 0 },
  pro:  { name: "Pro",  monthlyCredits: 1000, maxProjects: Infinity, maxScreens: 10,
          nativeBuilds: true,  webPublish: true, exportZip: true, githubPush: true, queuePriority: 10 },
} as const;
export type UpgradeReason = "projects" | "screens" | "native_builds" | "credits";
```
Zip download and Push to GitHub aren't limited by plan. Store credits as integer micro-credits internally (`03-data-model.md` §8) and display whole credits.

### 2. Costs per action (`lib/credits/costs.ts`)
- `buildCost(plan, builtScreenIds)` = `PRICES.screen × (screens not yet built)`. It's used for both the charge and the refund, so they always agree.
- `changeCost(change)` = `PRICES.changeRound` per `update_app` round (any number of screens/components in one round). Plus `PRICES.screen` for each **added** screen.
- AI images in edit mode (Phase 12 inspector: generate/transform) cost `PRICES.aiImage` each. Images written by generators during a build are included in the screen price.
- Free (never charged): Planner chat, `propose_plan` and revisions, theme switches and `set_theme`, inline manual edits, exports.

### 3. Ledger operations (`lib/credits/ledger.ts`)
- `grant(userId, amount, balance, reason, reference)`: `signup_grant` (Phase 02, once per verified email), `monthly_grant`, `purchase`, `admin_adjust`.
- `charge(userId, amount, reference)`: **one conditional `UPDATE`** on the user row. It takes `min(amount, monthly)` from `monthly_credits_balance` first and the rest from `credits_balance`, `WHERE monthly_credits_balance + credits_balance >= amount`, `RETURNING` both. Zero rows means `INSUFFICIENT_CREDITS`. In the same transaction, insert one or two ledger rows (`balance = 'monthly' | 'credits'`, references `<reference>:monthly` / `:credits`) and set `runs.credits_charged`.
- `refund(userId, amount, reference)`: always to `credits_balance`; the unique `reference` makes it idempotent.
- `resetMonthly(userId, newAmount, invoiceId)`: in one transaction, a `monthly_reset` row (negative of the current monthly balance) plus a `monthly_grant` row, both referencing `invoice:<id>`.
- Property test: each balance equals the sum of its ledger rows.

### 4. Where charges happen
| Moment | Charge | Refund |
|---|---|---|
| Confirm & Build (button or `confirm_plan`) | `buildCost` **before** the build run is created | each screen that ends `error` after its repair round: `PRICES.screen`, reference `refund:<runId>:<screenId>` |
| Retry / resume of an interrupted build | `buildCost` of what's still unbuilt (finished screens are free) | as above |
| `update_app` round | `changeCost` before `pendingChanges` is stored | the change round is refunded if *nothing* in it could be applied; added screens that fail are refunded per screen |
| Inspector AI image | `PRICES.aiImage` before calling the provider | refunded if generation fails |

### 5. Gates (`lib/credits/gate.ts`)
Call them from API routes and Planner tool handlers **before** enqueueing, and again in the worker at job start (`08-security.md` §2):
- `canCreateProject(user)`: non-deleted owned projects < `maxProjects`.
- `canHaveScreens(user, project, total)`: `total ≤ maxScreens`. The Planner gets the cap through its project state (`05` §3.1/§10) and says so plainly when a tool refuses.
- `canNativeBuild(user)`: `PLANS[plan].nativeBuilds` (replaces the Phase 15 stub).
- `canMessage(user)`: total balance ≥ `MIN_BALANCE_TO_MESSAGE`. Below it, the agent doesn't take a message, and the chat shows the upgrade or top-up dialog.
- Failures return `402 { code: "PLAN_LIMIT" | "INSUFFICIENT_CREDITS", reason: UpgradeReason }`, and the run stream emits `upgrade { reason }`. Both open `UpgradeDialog`.
- Queue priority: pg-boss `priority` from `queuePriority`.

### 6. Stripe integration (`lib/providers/billing/{types.ts,stripe.ts}`)
Keep everything that depends on the provider in `stripe.ts`.
- `POST /api/billing/checkout` → Checkout Session (mode `subscription`, `STRIPE_PRICE_PRO_MONTHLY`, `client_reference_id = userId`, existing customer if any) → `{ url }`.
- `POST /api/billing/portal` → Customer Portal session (cancel, update card, invoices).
- `POST /api/webhooks/stripe`: verify the signature on the **raw** body. For every relevant event, **re-fetch the subscription from Stripe** and derive the state from that, not from the event payload:
  - `checkout.session.completed`, `customer.subscription.created|updated` → upsert `subscriptions`, set `user.plan` from the fetched status.
  - `invoice.paid` with `billing_reason ∈ {subscription_create, subscription_cycle}` → `resetMonthly(user, 1000, invoice.id)`. No other invoice grants credits.
  - Downgrade from our UI or the portal → `cancel_at_period_end = true` (Pro stays until period end).
  - `customer.subscription.deleted` → `user.plan = 'free'`, zero `monthly_credits_balance` (ledger row `monthly_reset`, reference `sub_deleted:<subId>`).
- Optional: one-time top-up packs (Checkout mode `payment`) → `grant(purchase)` into `credits_balance`.

### 7. UI
- `/pricing`: the plan table generated from `PLANS`, the price list from `PRICES` ("a new screen costs 10 credits…"), and FAQ ("Do credits expire?" Monthly credits reset at renewal, the rest never expire).
- Header credit chip → popover: plan, both balances, "Upgrade to Pro" / "Manage billing".
- The Build button shows the cost ("Build 4 screens · 40 credits"), and the build summary says what was charged and refunded.
- `/billing` page (from the profile dropdown): subscription status, next renewal, ledger history.
- `UpgradeDialog` opened by `402` responses and `upgrade` events, with copy per `UpgradeReason`.

### 8. Margin report + admin tooling
- `scripts/margins.ts`: per plan and per week, credits charged vs `sum(ai_usage.cost_usd_micros)` (plus image costs). Alert if model cost per credit exceeds the target in `11-costs-and-pricing.md`.
- `scripts/credits.ts`: `grant` / `adjust` / `inspect` (support), writing `admin_adjust` rows with a note. For testing Pro in development, flip `user.plan` in the dev DB only. Never in production (the reference app did this by hand).

## Files to create / change
- `lib/credits/{plans.ts,costs.ts,ledger.ts,gate.ts}`
- `lib/providers/billing/{types.ts,stripe.ts,index.ts}`
- `app/api/billing/{checkout,portal,me,ledger}/route.ts`, `app/api/webhooks/stripe/route.ts`
- `app/pricing/page.tsx`, `app/billing/page.tsx`, `components/billing/{CreditChip.tsx,UpgradeDialog.tsx,PlanTable.tsx}`
- Planner tool handlers (`confirm_plan`, `update_app`) and the Confirm route: charge before enqueue. Worker build/change steps: refunds on screen failure.
- `scripts/{credits.ts,margins.ts}`

## Database changes
- `user.plan`, `user.credits_balance`, `user.monthly_credits_balance` (`03-data-model.md` §1).
- `credit_ledger` with `balance` and unique `reference` (§8).
- `subscriptions`: `user_id`, `provider`, `customer_id`, `subscription_id`, `plan`, `status`, `current_period_end`.
- `runs.credits_charged` (`credits_reserved` is unused under flat pricing).
- `ai_usage.cost_usd_micros` (already written since Phase 06) for the margin report.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/billing/me` | session | → `{ plan, credits, monthlyCredits, periodEnd?, cancelAtPeriodEnd, limits, prices }` |
| POST | `/api/billing/checkout` | session | `{ plan: "pro" }` → `{ url }` |
| POST | `/api/billing/portal` | session + customer | → `{ url }` |
| GET | `/api/billing/ledger?cursor=` | session | → `{ items, nextCursor }` |
| POST | `/api/webhooks/stripe` | Stripe signature | event → 200 |
| (run-starting routes and Planner tools) | — | — | may return `402 { code, reason }` / emit `upgrade { reason }` |

## Acceptance checklist
- [ ] New user gets 100 credits exactly once (a second account with the same verified email doesn't).
- [ ] Confirming a 4-screen plan charges 40 credits **before** the build starts. A screen that fails after repair is refunded 10, and a retry charges only the unbuilt screens. Replaying a step can't double-refund (unique `reference`).
- [ ] Two tabs confirming at once can't push the balance below zero (concurrency test on the conditional `UPDATE`).
- [ ] Pro: monthly credits are spent before never-expiring credits; renewal replaces the monthly balance; cancellation keeps Pro until period end, then zeroes monthly credits; refunds land in never-expiring credits.
- [ ] Free user: a 3rd project → 402 + upgrade dialog; a 6th screen via chat → the Planner says plainly it's a plan limit; App Store build → locked; web publish, zip and GitHub push work.
- [ ] Below 5 credits the chat refuses new messages with the upgrade dialog; theme switches and inline edits still work.
- [ ] Stripe test mode: checkout → Pro → 1,000 monthly credits posted once even if `invoice.paid` is replayed 3× or arrives before `checkout.session.completed`. A non-subscription invoice grants nothing.
- [ ] A downgraded user can open and edit a 9-screen app but can't add a 10th screen or a 3rd project.
- [ ] Webhook with an invalid signature → 400; the raw body is used for verification.
- [ ] Gates are enforced server-side and re-checked in the worker (tests call APIs directly with a free account).

## Claude Code prompt
```text
We're implementing Phase 18 (free/pro plans, flat credit prices, ledger, Stripe billing) of AppBuilding.

Read first:
- docs/plan/phases/phase-18-plans-and-billing.md (scope + acceptance checklist)
- docs/plan/11-costs-and-pricing.md (prices, plan numbers, margin targets — use its values)
- docs/plan/13-reference-implementation.md R15–R16 (what the reference app proved)
- docs/plan/03-data-model.md §1 and §8 (two balances, credit_ledger with unique reference, subscriptions)
- docs/plan/05-ai-agent-design.md §3.3 and §8 (Planner tools, upgrade event)
- docs/plan/08-security.md (webhooks, abuse)

Implement:
1. lib/credits/plans.ts as the single source of every limit and price (10/screen, 5/change round,
   1/AI image; chat, plans, theme switches free; min 5 credits to message; Free 100 one-time, 2 projects,
   5 screens, no native builds; Pro 1000 monthly spent first, unlimited projects, 10 screens, native builds).
2. costs.ts (buildCost from unbuilt screens, changeCost), ledger.ts (charge = ONE conditional UPDATE
   spending monthly first; refunds to never-expiring credits; unique reference on every row; resetMonthly),
   gate.ts wired into the Confirm route and Planner tools (confirm_plan, update_app) AND re-checked in the
   worker; refunds for failed screens; resumed builds not charged twice.
3. UpgradeReason codes on 402 responses + `upgrade` run event → one UpgradeDialog.
4. BillingProvider + Stripe: checkout, portal, webhook that re-fetches the subscription on every event,
   grants credits only for paid subscription_create/subscription_cycle invoices, cancel_at_period_end
   downgrades, subscription.deleted → Free + zero monthly credits.
5. UI: /pricing, credit chip, cost on the Build button, /billing page with ledger history.
6. scripts/margins.ts (credits charged vs ai_usage cost) and scripts/credits.ts.

Definition of done = Acceptance checklist. Tests: ledger invariants per balance, concurrent charges,
refund idempotency, webhook replay/out-of-order, downgrade rules, worker re-check. Stripe test mode /
mocks only; no live keys.

Do NOT build: team plans, regional payment providers, RevenueCat, cost-based pricing mode (leave the hook).
Finish by running lint, typecheck, tests; summarize changes and the Stripe dashboard setup steps.
```

## Pitfalls & notes
- **Integers only.** Never store credits or money as floats. Use micro-credits and round once at display.
- **Charge first, then save.** If you create the build run and then charge, a failed charge leaves an orphan run, and a double-click can charge twice. The conditional `UPDATE` plus a unique `reference` (`charge:<runId>`) closes both.
- **Webhook ordering.** Stripe can deliver `invoice.paid` before `checkout.session.completed`. Re-fetching the subscription and upserting makes order irrelevant.
- **Raw body in Next.js.** Read `await req.text()` in the route handler for signature verification. Don't parse JSON first.
- **Margins move with the model.** If `AI_BUILDER_MODEL` changes (decision D9), re-run the margin report. Flat prices hide cost changes from users, but not from you.
- **Abuse.** 100 free credits ≈ 10 screens of model spend per signup. Require email verification before the grant, rate-limit project creation per IP, and watch the margin report for free-tier spikes.
- **Tax/VAT.** Enable Stripe Tax (or equivalent) before charging customers in regions that require it.
