# Phase 18 — Plans, Credits and Billing (Free & Pro)

| | |
|---|---|
| Video chapter(s) | Ch.20 "Free & Pro Plan Setup" (02:40:52) |
| Branch | `feat/18-plans-and-billing` |
| Depends on | Phase 02 (users), Phase 05 (runs/jobs), Phase 06 (`ai_usage` logging), Phases 15–17 (gated features) |
| Effort (solo dev + Claude Code) | ~4–5 days |
| New env vars | `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_PRO_MONTHLY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`, `CREDIT_USD_VALUE` |

## Goal
Turn the builder into a SaaS. Every AI run costs **credits**, derived from the real token cost recorded in `ai_usage`. Free users get a one-time credit grant and limited projects and screens. Pro subscribers get a monthly grant, unlimited projects and native builds. Limits are enforced on the server, both before a job is enqueued and inside the worker. Stripe handles checkout, the customer portal and renewals.

## What we do differently from the video (and why)
- **Usage was logged long before billing.** In the video, the `credits = 100` column never changed until this chapter. We've written an `ai_usage` row per API response since Phase 06, so pricing here is based on measured cost, not guesses.
- **Append-only ledger, not a mutable counter.** `credit_ledger` is the source of truth; `user.credits_balance` is a cache updated in the same transaction. This gives auditability, refunds, and no lost updates under concurrency.
- **Reserve → settle → release per run.** Credits are held when a run is enqueued, so two parallel runs can't overspend. The actual cost is charged at the end, and the unused hold is released. The video's flow deducted after the fact.
- **Credit value comes from cost, with a margin, and is configurable.** `credits = ceil(cost_usd × CREDIT_MARKUP / CREDIT_USD_VALUE)`. Plan numbers are placeholders, set in `11-costs-and-pricing.md`.
- **Billing behind `BillingProvider`.** Stripe is the default. Regional providers (e.g. Razorpay, as the video mentions for India) can be added without touching gates or the ledger.
- **RevenueCat is not for this.** RevenueCat handles in-app purchases inside *generated mobile apps*, a possible future feature for users' apps. This web SaaS bills through Stripe (or another web billing provider).

## Steps

### 1. Plan config
`lib/credits/plans.ts` (single source of truth; numbers are placeholders, finalize in `11-costs-and-pricing.md`):
```ts
export const PLANS = {
  free: { signupCredits: 100, monthlyCredits: 0, maxProjects: 2, maxScreensPerProject: 5,
          nativeBuilds: false, webPublish: true, exportZip: true, githubPush: true, queuePriority: 0 },
  pro:  { signupCredits: 0, monthlyCredits: 2000, maxProjects: Infinity, maxScreensPerProject: 30,
          nativeBuilds: true, webPublish: true, exportZip: true, githubPush: true, queuePriority: 10 },
} as const;
```

### 2. Pricing function
`lib/credits/pricing.ts`:
- `costUsdMicros(usageRow)` uses a per-model price table (input, output, cache read, cache write). Keep it in code next to `AI_MODEL` handling (`05-ai-agent-design.md` §1) and update it when the model changes.
- `toCredits(costUsdMicros) = ceil(costUsd × CREDIT_MARKUP / CREDIT_USD_VALUE)` (internally micro-credits; display whole credits with one decimal).
- `estimateRunCredits(kind, { screens })` uses p90 cost per run kind, measured in evals (`10-testing-and-evals.md`), × 1.5 safety factor.

### 3. Ledger operations
`lib/credits/ledger.ts`. All operations run in a DB transaction that inserts a ledger row **and** updates `user.credits_balance`, using `SELECT … FOR UPDATE` on the user row:
- `grant(userId, amount, reason)`: `signup_grant`, `monthly_grant`, `purchase`, `admin_adjust`
- `reserve(userId, runId, amount)`: fails with `INSUFFICIENT_CREDITS` if the balance is too low
- `settle(runId)`: `charge` the actual cost (sum of `ai_usage.cost_usd_micros` for the run → credits) and `release` the reservation; sets `runs.credits_charged`
- `refund(runId)`: for runs that failed due to **our** error (non-retryable platform fault), refund the charge

Signup grant: a Better Auth `user.create` hook calls `grant(signup_grant)`. Grant only once per verified email, to discourage multi-account abuse (`08-security.md`).

### 4. Gates
`lib/credits/gate.ts`. Call from API routes **before** enqueueing, and again in the worker at job start:
- `canCreateProject(user)`: count non-deleted owned projects < `maxProjects`
- `canAddScreens(user, project, n)`: current screens + n ≤ `maxScreensPerProject`. The Planner also receives the cap (`05-ai-agent-design.md` §10).
- `canNativeBuild(user)`: `PLANS[user.plan].nativeBuilds` (replaces the Phase 15 stub)
- `canStartRun(user, kind, estimate)`: `reserve` succeeds
- Worker per-turn guard: if the run's accumulated cost exceeds `credits_reserved` + the remaining balance, stop gracefully with `run.error { code: "CREDITS_EXHAUSTED" }` after the current tool step, and keep the files written so far.
- Map failures to `402` with `{ code, upgradeUrl }`. The UI shows an upgrade dialog, not a generic error.
- Queue priority: pg-boss `priority` from `queuePriority`.

### 5. Stripe integration
`lib/providers/billing/{types.ts,stripe.ts}`:
- `POST /api/billing/checkout` → Checkout Session (mode `subscription`, price `STRIPE_PRICE_PRO_MONTHLY`, `client_reference_id = userId`, existing customer if any) → `{ url }`.
- `POST /api/billing/portal` → Customer Portal session → `{ url }` (cancel, update card, invoices).
- `POST /api/webhooks/stripe`: verify the signature with `STRIPE_WEBHOOK_SECRET` on the **raw** body. Handle:
  - `checkout.session.completed` → upsert `subscriptions`, set `user.plan = 'pro'`
  - `customer.subscription.updated|deleted` → status/plan/`current_period_end`; downgrade to free when canceled or ended
  - `invoice.paid` (subscription create/cycle) → `grant(monthly_grant)` with `stripe_event_id` = event id. The unique index makes retries idempotent.
- Decide (placeholder) whether unused Pro credits roll over or reset each period. Default: reset monthly grants, keep purchased top-ups.
- Optional: one-time top-up packs via Checkout mode `payment` → `grant(purchase)`.

### 6. UI
- `/pricing` page: plan comparison table (from `PLANS`), FAQ ("What is a credit?", "Do credits expire?"), CTA → checkout (or sign-in first).
- Header credit chip → popover: plan, credits left, used this period, "Upgrade to Pro" / "Manage billing".
- After each run, show the cost in the build summary ("This build used 23.4 credits").
- Locked features show a lock icon + upgrade dialog (e.g. Publish → App Store tab on Free).
- Account → Billing page: subscription status, next renewal, ledger history (last 50 rows).

### 7. Admin tooling (minimal)
A CLI script `scripts/credits.ts` for `grant`/`adjust`/`inspect` (support cases) that writes `admin_adjust` rows with a note.

## Files to create / change
- `lib/credits/{plans.ts,pricing.ts,ledger.ts,gate.ts}`
- `lib/providers/billing/{types.ts,stripe.ts,index.ts}`
- `app/api/billing/{checkout,portal}/route.ts`, `app/api/webhooks/stripe/route.ts`
- `app/pricing/page.tsx`, `app/account/billing/page.tsx`, `components/billing/{CreditChip.tsx,UpgradeDialog.tsx,PlanTable.tsx}`
- Enqueue helpers in `lib/jobs/enqueue.ts` (call gate + reserve), `worker/handlers/*` (settle in `finally`)
- Better Auth hook in `lib/auth/server.ts` (signup grant)
- `scripts/credits.ts`

## Database changes
- `credit_ledger` (append-only) with unique `stripe_event_id` where not null; index `(user_id, created_at)`.
- `subscriptions`: `user_id`, `provider`, `customer_id`, `subscription_id`, `plan`, `status`, `current_period_end`.
- `user.plan`, `user.credits_balance` (cache).
- `runs.credits_reserved`, `runs.credits_charged`.
- `ai_usage.cost_usd_micros` (already written since Phase 06; backfill cost for any rows missing it).

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/billing/me` | session | → `{ plan, creditsBalance, periodEnd?, limits }` |
| POST | `/api/billing/checkout` | session | `{ plan: "pro" }` → `{ url }` |
| POST | `/api/billing/portal` | session + customer | → `{ url }` |
| GET | `/api/billing/ledger?cursor=` | session | → `{ items, nextCursor }` |
| POST | `/api/webhooks/stripe` | Stripe signature | event → 200 |
| (all run-starting routes) | — | — | may return `402 { code: "INSUFFICIENT_CREDITS" \| "PLAN_LIMIT", upgradeUrl }` |

## Acceptance checklist
- [ ] New user gets the signup grant exactly once (a second account with the same verified email doesn't).
- [ ] A build reserves credits at enqueue, charges the actual cost at the end, and releases the rest; the balance equals the ledger sum (property test).
- [ ] Two simultaneous runs can't push the balance below zero (concurrency test).
- [ ] A run that exhausts credits mid-way stops gracefully with written files preserved.
- [ ] Free user: a 3rd project → 402 with upgrade dialog; a 6th screen → blocked; App Store build → locked; web publish and zip export work.
- [ ] Stripe test mode: checkout → plan becomes Pro → monthly grant posted once even if the webhook is replayed 3×.
- [ ] Cancel in the portal → plan returns to Free at period end.
- [ ] Webhook with an invalid signature → 400; the raw body is used for verification.
- [ ] Gates are enforced server-side. Calling run-starting APIs directly with a free account is still blocked (tests), and the worker re-checks.

## Claude Code prompt
```text
We're implementing Phase 18 (free/pro plans, credit ledger, Stripe billing) of AppBuilding.

Read first:
- docs/plan/phases/phase-18-plans-and-billing.md (scope + acceptance checklist)
- docs/plan/11-costs-and-pricing.md (credit value, markup, plan numbers — use its values)
- docs/plan/03-data-model.md §1 and §8 (user extensions, ai_usage, credit_ledger, subscriptions)
- docs/plan/05-ai-agent-design.md §1 and §10 (usage recording, limits)
- docs/plan/08-security.md (webhooks, abuse)

Implement:
1. lib/credits/plans.ts (single source of plan limits), pricing.ts (cost → credits with CREDIT_USD_VALUE
   and markup), ledger.ts (grant/reserve/settle/refund in transactions with row locks; balance cache),
   gate.ts (project/screen/native-build/run gates) wired into every run-starting route AND re-checked in
   the worker; per-turn credit guard in the agent loop.
2. BillingProvider interface + Stripe implementation: checkout, customer portal, webhook
   (raw-body signature verification; idempotent monthly grants via unique stripe_event_id).
3. UI: /pricing, header credit chip + popover, upgrade dialog on 402, per-run cost in build summary,
   account billing page with ledger history.
4. Signup grant once per verified email; scripts/credits.ts admin tool.

Definition of done = Acceptance checklist. Tests: ledger invariants (balance == sum), concurrent
reservations, webhook replay idempotency, 402 paths, worker re-check. Use Stripe test mode / mocks; no live keys.

Do NOT build: top-up packs unless trivial, team plans, regional payment providers, RevenueCat.
Finish by running lint, typecheck, tests; summarize changes and the Stripe dashboard setup steps.
```

## Pitfalls & notes
- **Floating point.** Never store money or credits as floats. Use integer micro-units everywhere and round once at display.
- **Webhook ordering.** Stripe can deliver `invoice.paid` before `checkout.session.completed`. Make each handler upsert what it needs and not assume order.
- **Raw body in Next.js.** Read `await req.text()` in the route handler for signature verification. Don't parse JSON first.
- **Price changes after model changes.** If `AI_MODEL` or its prices change, update the price table and re-measure estimates. Old `ai_usage` rows keep their recorded cost.
- **Failed runs.** Charge for tokens consumed on user-caused failures (e.g. canceled runs), refund on platform faults. Write the policy on the pricing FAQ.
- **Abuse.** Free credits attract scripted sign-ups. Require email verification before the grant, rate-limit project creation per IP, and keep the free grant small.
- **Tax/VAT.** Enable Stripe Tax (or equivalent) before charging customers in regions that require it.
