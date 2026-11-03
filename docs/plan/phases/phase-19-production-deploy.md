# Phase 19 — Production Deploy & Operations

| | |
|---|---|
| Video chapter(s) | Ch.21 "Deploy App to Production" (02:46:49) |
| Branch | `feat/19-production-deploy` |
| Depends on | All previous phases merged to `main` |
| Effort (solo dev + Claude Code) | ~3–4 days (+ ongoing ops) |
| New env vars | `SENTRY_DSN`, `PREVIEW_ORIGIN` (if not already set); every canonical var gets a production value |

## Goal
Run AppBuilding in production with a custom domain: the web app, a separate long-running worker, managed Postgres, object storage, and every third-party integration pointed at production URLs. Add the operational basics (error tracking, logs, rate limits, backups, spend alerts, rollback) so the platform can take real users safely.

## What we do differently from the video (and why)
- **Two services, not one.** The video deploys only the Next.js app to Vercel and relies on CloudHike Jobs calling back into it. Our agent, builds and exports run in a **worker** (`02-architecture.md` §6) with no request timeout, `eas-cli`, Chromium and a writable temp disk. That's a Railway/Fly/Render service, not a serverless function.
- **Migrations are a release step.** We don't run `db:push` from a laptop. We generate SQL migrations (`drizzle-kit generate`) during development and apply them (`drizzle-kit migrate`) in CI before the new code goes live.
- **Separate dev and prod everywhere.** The video runs on its provider's production environment from the start. We keep separate OAuth apps, Stripe modes, buckets, DBs and Anthropic API keys per environment, so testing never touches real users or real money.
- **Public URL vars set deliberately.** The video had to delete and re-create `NEXT_PUBLIC_APP_URL` because it was stored as a sensitive value. `NEXT_PUBLIC_*` values are inlined at **build** time and aren't secrets: store them as plain vars and **redeploy** after changing them.

## Steps

### 1. Provision production infrastructure
| Piece | Choice (examples) | Notes |
|---|---|---|
| Web | Vercel (or a Node service on Railway/Render) | Region close to the DB |
| Worker | Railway / Fly / Render long-running service | Docker image from `worker/Dockerfile`: Node 22, pinned `eas-cli`, JDK (keytool), Playwright Chromium, cached template `node_modules`; ≥ 2 GB RAM, ephemeral disk ≥ 5 GB; 1–N replicas |
| Postgres | Managed (Neon, Railway, Supabase, CloudHike, RDS…) | PITR backups on; **direct (session) connection** for pg-boss and LISTEN/NOTIFY; pooled URL OK for plain queries |
| Object storage | S3 / R2 / provider bucket | Private bucket; versioning on; CORS for presigned PUT from `NEXT_PUBLIC_APP_URL` |
| Email | Transactional provider | Invites (Phase 14), auth verification |
| Errors | Sentry (web + worker) | `SENTRY_DSN`, environment tag, release = git sha |

### 2. Environment variables checklist
Set per environment (dev / prod). **W** = web, **K** = worker.

| Var | W | K | Notes |
|---|---|---|---|
| `DATABASE_URL` | ✅ | ✅ | worker: direct connection |
| `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL` | ✅ | | URL = prod origin |
| `NEXT_PUBLIC_APP_URL` | ✅ | ✅ | worker uses it for webhook/callback URLs; rebuild web after change |
| `GOOGLE_CLIENT_ID/SECRET`, `GITHUB_LOGIN_CLIENT_ID/SECRET` | ✅ | | prod OAuth clients |
| `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | ✅ | ✅ | |
| `ANTHROPIC_API_KEY`, `AI_MODEL` | | ✅ | separate prod API key/workspace with its own spend limit |
| `MAX_SCREENS_DEFAULT`, `AGENT_MAX_TURNS`, `AGENT_MAX_FIX_ITERATIONS`, `AGENT_SCREEN_CONCURRENCY` | ✅ | ✅ | |
| `ENCRYPTION_KEY`, `ENCRYPTION_KEY_VERSION` | ✅ | ✅ | **same** values on both; backed up in a secret manager |
| `GITHUB_APP_ID`, `GITHUB_APP_SLUG`, `GITHUB_APP_CLIENT_ID`, `GITHUB_APP_CLIENT_SECRET`, `GITHUB_APP_PRIVATE_KEY` | ✅ | ✅ | prod GitHub App |
| `IMAGEKIT_URL_ENDPOINT`, `IMAGEKIT_PRIVATE_KEY` | ✅ | ✅ | |
| `BACKEND_PROVIDER`, `BACKEND_OAUTH_CLIENT_ID`, `BACKEND_OAUTH_CLIENT_SECRET` | ✅ | ✅ | prod developer app |
| `EAS_WEBHOOK_SECRET` | ✅ | ✅ | |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_PRO_MONTHLY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | ✅ | | **live** mode only in prod |
| `CREDIT_USD_VALUE` | ✅ | ✅ | |
| `SENTRY_DSN` | ✅ | ✅ | |
| `PREVIEW_ORIGIN` | ✅ | ✅ | separate preview domain or the app origin (see `07-preview-runtime.md` §6) |

`lib/env.ts` (Zod) must fail fast at boot when a required var is missing. Add a CI check that `.env.example` lists every key `lib/env.ts` reads.

### 3. Database release flow
- Commit generated migrations under `drizzle/`.
- CI job on `main`: `drizzle-kit migrate` against prod, **then** deploy web + worker. Use expand → migrate → contract for breaking changes (add new columns first, backfill, remove old later), so a rollback of code doesn't need a DB rollback.
- Exclude the `pgboss` schema from Drizzle (`schemaFilter: ["public"]`).

### 4. Domains and callback URLs
Attach the custom domain (e.g. `app.<brand>.com`) and set `NEXT_PUBLIC_APP_URL` + `BETTER_AUTH_URL` to it. Then update **every** redirect/callback URL:

| Integration | Where | URL |
|---|---|---|
| Google sign-in | Google Cloud console OAuth client | `<APP_URL>/api/auth/callback/google` |
| GitHub sign-in | GitHub OAuth app (login) | `<APP_URL>/api/auth/callback/github` |
| GitHub App (export) | GitHub App settings | callback `<APP_URL>/api/github/callback`, homepage `<APP_URL>` |
| Backend provider | provider developer app | `<APP_URL>/api/backend/callback` |
| Stripe | Dashboard → Webhooks | `<APP_URL>/api/webhooks/stripe` (live mode) |
| EAS | per user Expo account (created by the worker at Expo connect) | `<APP_URL>/api/webhooks/eas` |
| Storage CORS | bucket config | allow origin `<APP_URL>` for PUT |

If the domain ever changes, run `scripts/rotate-eas-webhooks.ts` to update existing users' EAS webhooks. The polling fallback (Phase 15) covers the gap meanwhile.

### 5. Security and abuse controls
- Rate limits (per user + per IP) on: project create, message send, run-starting routes, uploads, invites, public endpoints. Use a Redis-backed limiter, or a Postgres token bucket for low volume.
- Security headers (CSP for the app shell, HSTS, `frame-ancestors`), cookie flags (`Secure`, `HttpOnly`, `SameSite=Lax`).
- Anthropic workspace **spend limit** set in the console, plus our own alert (step 6).
- Admin kill switch: env/DB flag `GENERATION_DISABLED`, which makes run-starting routes return 503 with a status message.

### 6. Observability
- Structured JSON logs (`pino`) with `runId`, `projectId`, `userId` (hashed), and job kind. Never log tokens, prompts with secrets, or file contents.
- Sentry for exceptions, with source maps uploaded on build.
- Metrics/alerts (Sentry crons, provider dashboards, or a tiny `/api/health` + uptime monitor):
  - queue depth and oldest job age per kind (alert if a build waits > 5 min)
  - run failure rate per kind (alert > 10% over 1 h)
  - Anthropic cost per hour/day from `ai_usage` (alert above budget); cost per completed build trend
  - webhook failures (Stripe/EAS 4xx/5xx)
- Health endpoints: web `/api/health` (DB ping), worker heartbeat row updated every 30 s.

### 7. Backups and key management
- Postgres PITR enabled; monthly restore drill into a scratch DB.
- Storage bucket versioning + lifecycle (expire `exports/` after 7 days, keep thumbnails).
- `ENCRYPTION_KEY` stored in a secret manager with offline backup. Losing it makes every stored token and keystore unreadable. Key rotation: add the new key as `ENCRYPTION_KEY_VERSION+1`, re-encrypt in a background job, retire the old one.
- **Android keystores** (`project_signing`): nightly encrypted export to a separate bucket/account. A lost keystore means users can never update their Play Store app.

### 8. Deploy, smoke test, rollback
Post-deploy smoke test (scripted where possible with Playwright against prod, test account):
- [ ] Sign up (email) + Google + GitHub sign-in
- [ ] Create project with an image → plan → confirm → 3 screens build; canvas + preview render
- [ ] Chat edit + inline edit + theme switch
- [ ] QR / Snack opens in Expo Go
- [ ] Connect backend (test provider account) → data persists
- [ ] Zip export downloads; GitHub push to a test repo
- [ ] Web publish URL loads
- [ ] Android APK build reaches "finished" (can be async)
- [ ] Stripe live checkout with a real card, then refund; credits granted once
- [ ] Public showcase visible signed-out

Rollback: Vercel instant rollback to the previous deployment; worker redeploys the previous image tag. DB migrations are forward-only (expand/contract), so the old code keeps working on the new schema. Document this in `docs/runbook.md`.

## Files to create / change
- `worker/Dockerfile`, `railway.json` / `fly.toml` / `render.yaml` (whichever host)
- `.github/workflows/{ci.yml,release.yml}` (lint/typecheck/test on PR; migrate + deploy on `main`)
- `lib/env.ts` (complete schema), `.env.example` (all canonical vars, no values)
- `lib/observability/{logger.ts,sentry.ts}`, `sentry.*.config.ts`
- `lib/ratelimit.ts`, `middleware.ts` (headers, rate limits)
- `app/api/health/route.ts`, `worker/heartbeat.ts`
- `scripts/{rotate-eas-webhooks.ts,backup-keystores.ts,reencrypt.ts}`
- `docs/runbook.md` (incidents, rollback, key rotation, takedown requests)

## Database changes
- Optional `system_flags` table (`key`, `value`) for the kill switch and maintenance banner. Add it to `03-data-model.md` if adopted.
- Optional `worker_heartbeats` (`worker_id`, `last_seen_at`).

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/health` | none | → `{ ok, db: "ok", worker: { lastSeenSecondsAgo } }` (no secrets, no versions of deps) |
| (all) | — | — | may return `429` (rate limited) or `503` (generation disabled) |

## Acceptance checklist
- [ ] Production domain serves the app over HTTPS; HSTS and security headers present.
- [ ] Web and worker run from the same git sha; Sentry shows releases for both.
- [ ] CI on `main` applies migrations before deploying; a failing migration blocks the deploy.
- [ ] Every row of the callback/webhook table is configured and verified by the smoke test.
- [ ] Stripe is in live mode in prod and test mode elsewhere; prod webhooks verify.
- [ ] Rate limits return 429 under a load-test burst; the kill switch returns 503 for run-starting routes.
- [ ] Spend alert fires in a dry run (threshold temporarily lowered).
- [ ] PITR restore drill and keystore backup restore both succeed.
- [ ] No secret values appear in logs (log scan test) and `/api/health` exposes nothing sensitive.
- [ ] Rollback rehearsed once: previous web deployment + previous worker image run against the current DB.

## Claude Code prompt
```text
We're implementing Phase 19 (production deploy & operations) of AppBuilding.

Read first:
- docs/plan/phases/phase-19-production-deploy.md (scope + acceptance checklist)
- docs/plan/02-architecture.md §6 (processes and environments)
- docs/plan/04-tech-stack.md (chosen hosts/providers) and docs/plan/08-security.md
- docs/plan/12-decisions-and-open-questions.md (which hosts I picked)

Implement the code/config side (I will click through provider dashboards myself):
1. worker/Dockerfile (Node 22, pinned eas-cli, JDK, Playwright Chromium, cached template deps) and the
   host config file for <FILL IN: Railway | Fly | Render>.
2. CI: lint/typecheck/test on PRs; on main: drizzle-kit migrate then deploy web + worker.
3. lib/env.ts complete Zod schema for every canonical var + CI check against .env.example.
4. Structured logging with redaction, Sentry for web and worker, /api/health, worker heartbeat.
5. Rate limiting + security headers middleware; GENERATION_DISABLED kill switch.
6. Spend alert job from ai_usage (hourly), queue-depth/oldest-job alert.
7. Scripts: rotate-eas-webhooks, backup-keystores, reencrypt (key rotation); docs/runbook.md.
8. A Playwright smoke-test suite runnable against any base URL with a test account.

Produce a step-by-step manual checklist (in docs/runbook.md) of every dashboard change needed:
domains, OAuth redirect URIs, GitHub App, backend provider app, Stripe live webhook, storage CORS.

Definition of done = Acceptance checklist items that can be verified in code/CI; list the rest for me.
Finish by running lint, typecheck, tests; summarize what I must do manually, in order.
```

## Pitfalls & notes
- **LISTEN/NOTIFY through poolers.** Transaction-mode poolers (PgBouncer, some serverless drivers) break `LISTEN`. pg-boss and the SSE tailer need a session/direct connection. On serverless web, the SSE route can fall back to short polling of `run_events` (by `id > lastEventId`) if LISTEN isn't available.
- **SSE on serverless.** Function max duration still applies to streaming responses. Return after ~250 s with a `retry:` hint; the client reconnects with `Last-Event-ID` (`02-architecture.md` A3).
- **Build-time public vars.** Changing `NEXT_PUBLIC_APP_URL` without a rebuild leaves the old value in client bundles. Always redeploy after changing it, as the video did.
- **Worker scale-out.** Multiple worker replicas are fine with pg-boss (row locks). Watch Anthropic rate limits, and set per-kind concurrency (`teamSize`/`batchSize`) so builds don't starve thumbnails or vice versa.
- **Region latency.** Keep web, worker and DB in the same region. The SSE tail and VFS writes are chatty.
- **Cost surprises.** Free-tier signups plus a viral showcase can burn API credit quickly. Keep the console spend limit below what you can afford to lose in a day.
