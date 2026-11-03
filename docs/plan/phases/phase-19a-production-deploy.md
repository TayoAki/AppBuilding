# Phase 19a — Production Deploy & Operations

| | |
|---|---|
| Video chapter(s) | Ch.21 "Deploy App to Production" (02:46:49) |
| Branch | `feat/19a-production-deploy` |
| Depends on | All previous phases in the chosen track (MVP or full) merged to `main` |
| Effort (solo dev + Claude Code) | ~3–4 days (+ ongoing ops). Data lifecycle, template upgrades, vendor watch and launch-gate evidence are in **Phase 19b** |
| New env vars | `SENTRY_DSN`, `PREVIEW_ORIGIN`, `DATABASE_URL_DIRECT`, `GENERATION_DISABLED`, `DAILY_SPEND_CAP_USD` (if not already set); every canonical var gets a production value |

## Goal
Run AppBuilding in production with a custom domain. That means the web app, a long-running **worker**, a separate **sandbox service** for untrusted generated code, managed Postgres, object storage, and every third-party integration pointed at production URLs. Add the operational basics: error tracking, logs, rate limits, backups, spend alerts and rollback.

## What we do differently from the video (and why)
- **Three services, not one** (ADR-003 #18). The video deploys only the Next.js app to Vercel and relies on CloudHike Jobs calling back into it. Ours:
  - **web**: the Next.js app;
  - **worker**: agent, pipelines, outbox dispatcher. No request timeout. It holds DB/AI/provider secrets but never runs generated code;
  - **sandbox**: `expo export`, Playwright renders/thumbnails, `eas-cli` commands and the Path B Expo dev servers. It has **no DB access and no platform secrets**: only `SANDBOX_TOKEN_SECRET` to verify per-job tokens, and per-job user tokens that are never stored.
- **Migrations are a release step.** We generate SQL migrations (`drizzle-kit generate`) during development and apply them (`drizzle-kit migrate`) in CI before the new code goes live.
- **Separate dev and prod everywhere**: OAuth apps, Stripe modes, buckets, DBs and Anthropic API keys per environment.
- **Public URL vars set deliberately.** The video had to delete and re-create `NEXT_PUBLIC_APP_URL` because it was stored as a sensitive value. `NEXT_PUBLIC_*` values are inlined at **build** time and aren't secrets: store them as plain vars and **redeploy** after changing them.

### Reference app notes
- **No tunnel in any environment.** The reference app's job service refused localhost, so dev needed an ngrok tunnel, which broke HMR websockets and let a render loop hit the dev server 1,000+ times. Our worker pulls jobs from Postgres. Tunnels are only for testing inbound Stripe/EAS webhooks locally.
- **Builds progress whether or not anyone is watching.** In the reference app, an unwatched build "shows as building until the workspace is opened again". Here the worker, per-project EAS webhooks and polling handle it, and stuck runs are still monitored (§6).
- **`.gitignore` generated output.** The reference app's repo picked up 10K+ Expo build/export files (step 1b).
- **Publish generated apps from production**, because a backend-connected app's API base URL comes from `NEXT_PUBLIC_APP_URL` (Phase 16).
- **OAuth state cookies are per origin.** Use exactly `NEXT_PUBLIC_APP_URL` (no `www.` variant) when connecting backends/GitHub.

## Steps

### 1. Provision production infrastructure
| Piece | Choice (examples) | Notes |
|---|---|---|
| Web | Vercel (or a Node service on Railway/Render) | Region close to the DB |
| Worker | Railway / Fly / Render long-running service | Node 22 image from `worker/Dockerfile`; 1–N replicas; no Chromium/eas-cli needed |
| **Sandbox** | Railway / Fly / Render service in an isolated project/network | `sandbox/Dockerfile`: Node 22, pinned `eas-cli`, Playwright Chromium, prebuilt `node_modules` per template version (read-only), JDK only if keystore option B. Non-root, ≥ 4 GB RAM per instance (Metro sessions), ephemeral disk ≥ 10 GB. **No DB network access**; egress restricted where the host allows (npm registry, Expo/EAS, ImageKit, storage). Wildcard DNS + TLS for `*.<DEVICE_PREVIEW_BASE_DOMAIN>` if Path B1 is used |
| Postgres | Managed (Neon, Railway, Supabase, CloudHike, RDS…) | PITR on; **direct (session) connection** for pg-boss, outbox and LISTEN; pooled URL OK for plain queries |
| Object storage | S3 / R2 / provider bucket | Private; versioning on; CORS for presigned PUT from `NEXT_PUBLIC_APP_URL`; signed URLs are how the sandbox reads and writes |
| Email | Transactional provider (from Phase 02) | Verification, invites, account notices |
| Errors | Sentry (web, worker, sandbox) | `SENTRY_DSN`, environment tag, release = git sha |

### 1b. Repository hygiene
- Root `.gitignore`: `node_modules`, `.next`, `.env*` (except `.env.example`), `public/preview-runtime/*.js`, `tmp/`, `.exports/`, `*.tgz`, `dist/`, `web-build/`, `.expo/`, `*.jks`, `credentials.json`, Playwright reports, `spikes/**/.env.spike`. Add a CI check that fails if a PR adds more than ~500 files or any file under these paths.

### 2. Environment variables checklist
Per environment (dev / prod). **W** = web, **K** = worker, **S** = sandbox.

| Var | W | K | S | Notes |
|---|---|---|---|---|
| `DATABASE_URL` | ✅ | ✅ | ❌ | pooled is fine for plain queries on serverless web |
| `DATABASE_URL_DIRECT` | ✅ | ✅ | ❌ | **direct/session-mode**: pg-boss + outbox (worker), SSE `LISTEN` (web), migrations (CI) |
| `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL` | ✅ | | ❌ | URL = prod origin |
| `NEXT_PUBLIC_APP_URL` | ✅ | ✅ | ❌ | the worker uses it for callback/webhook URLs; rebuild web after a change |
| `GOOGLE_CLIENT_ID/SECRET`, `GITHUB_LOGIN_CLIENT_ID/SECRET` | ✅ | | ❌ | prod OAuth clients |
| `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | ✅ | ✅ | ❌ | the sandbox only gets signed URLs |
| `ANTHROPIC_API_KEY`, `AI_MODEL`, `AI_BUILDER_MODEL` | | ✅ | ❌ | separate prod key/workspace with its own spend limit |
| `MAX_SCREENS_DEFAULT`, `AGENT_SCREEN_CONCURRENCY` | ✅ | ✅ | | |
| `ENCRYPTION_KEY`, `ENCRYPTION_KEY_VERSION` | ✅ | ✅ | ❌ | **same** values on web and worker; backed up in a secret manager |
| `GITHUB_APP_ID`, `GITHUB_APP_SLUG`, `GITHUB_APP_CLIENT_ID`, `GITHUB_APP_CLIENT_SECRET`, `GITHUB_APP_PRIVATE_KEY` | ✅ | ✅ | ❌ | prod GitHub App |
| `IMAGEKIT_ENABLED`, `IMAGEKIT_URL_ENDPOINT`, `IMAGEKIT_PRIVATE_KEY` | ✅ | ✅ | ❌ | `06` §7, Phase 12 |
| `BACKEND_PROVIDER`, `BACKEND_OAUTH_CLIENT_ID`, `BACKEND_OAUTH_CLIENT_SECRET` | ✅ | ✅ | ❌ | prod developer app |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_PRO_MONTHLY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | ✅ | | ❌ | **live** mode only in prod |
| `CREDIT_USD_VALUE`, `CREDIT_MARKUP` | ✅ | ✅ | ❌ | margin report / optional cost-based mode |
| `DEVICE_PREVIEW_MODE`, `DEVICE_PREVIEW_BASE_DOMAIN` | ✅ | ✅ | ✅ | Phase 11 |
| `SANDBOX_URL` | | ✅ | | worker → sandbox |
| `SANDBOX_TOKEN_SECRET` | | ✅ | ✅ | HMAC for per-job tokens; the only secret the sandbox holds |
| `GENERATION_DISABLED`, `DAILY_SPEND_CAP_USD` | ✅ | ✅ | | kill switch (also a `system_flags` row); hard daily model-spend cap |
| `SENTRY_DSN` | ✅ | ✅ | ✅ | |
| `PREVIEW_ORIGIN` | ✅ | ✅ | ✅ | separate preview domain or the app origin (`07` §6) |

There's no `EAS_WEBHOOK_SECRET`: EAS webhook secrets are per project, encrypted in the DB (ADR-003 #17). `lib/env.ts` (Zod) fails fast at boot, per process, and **the sandbox's schema rejects any unexpected secret-looking var**. CI checks that `.env.example` lists every key each schema reads.

### 3. Database release flow
- Commit generated migrations under `drizzle/`.
- CI job on `main`: `drizzle-kit migrate` against prod, **then** deploy web, worker and sandbox. Use expand → migrate → contract for breaking changes, so a code rollback never needs a DB rollback.
- Exclude the `pgboss` schema from Drizzle (`schemaFilter: ["public"]`). Pin the pg-boss major version (ADR-003 #21).

### 4. Domains and callback URLs
Attach the custom domain (e.g. `app.<brand>.com`) and set `NEXT_PUBLIC_APP_URL` + `BETTER_AUTH_URL`. Then update **every** redirect/callback URL:

| Integration | Where | URL |
|---|---|---|
| Google sign-in | Google Cloud console OAuth client | `<APP_URL>/api/auth/callback/google` |
| GitHub sign-in | GitHub OAuth app (login) | `<APP_URL>/api/auth/callback/github` |
| GitHub App (export) | GitHub App settings | callback `<APP_URL>/api/github/callback`, homepage `<APP_URL>` |
| Backend provider | provider developer app | `<APP_URL>/api/backend/callback` |
| Stripe | Dashboard → Webhooks | `<APP_URL>/api/webhooks/stripe` (live mode) |
| EAS | **per EAS project**, created by `ensureEasWebhook` (Phase 15) | `<APP_URL>/api/webhooks/eas/<projectId>` |
| Device preview (Path B1) | DNS + TLS | `*.<DEVICE_PREVIEW_BASE_DOMAIN>` → sandbox proxy |
| Storage CORS | bucket config | allow origin `<APP_URL>` for PUT |

If the domain ever changes, `scripts/rotate-eas-webhooks.ts` re-creates each project's EAS webhook with the new URL, using the owner's token and the stored per-project secret. Projects whose owner disconnected Expo fall back to polling.

### 5. Security and abuse controls
- Rate limits (per user + per IP) on: project create, message send, run-starting routes, uploads, invites, device-preview sessions, public endpoints. Use a Postgres token bucket, or Redis at volume.
- Security headers (CSP for the app shell, HSTS, `frame-ancestors`), cookie flags (`Secure`, `HttpOnly`, `SameSite=Lax`).
- Anthropic workspace **spend limit** set in the console, plus our `DAILY_SPEND_CAP_USD` check before enqueueing agent runs (sum of today's `ai_usage`).
- Admin kill switch: `GENERATION_DISABLED` env or a `system_flags` row → run-starting routes return 503 with a status message.

### 6. Observability
- Structured JSON logs (`pino`) with `runId`, `projectId`, `userId` (hashed), job kind and step. Never log tokens, device-preview URLs, prompts with secrets, or file contents.
- Sentry for exceptions in all three services, with source maps.
- Alerts:
  - queue depth and oldest job age per queue (alert if a build waits > 5 min), plus **unsent `job_outbox` rows older than 1 min**;
  - run failure rate per kind (> 10% over 1 h) and stuck steps (no `runs.heartbeat_at` update for 10 min, excluding kinds that wait on external systems);
  - Anthropic cost per hour/day from `ai_usage` against budget; ImageKit unit usage at 80%;
  - webhook failures (Stripe/EAS 4xx/5xx); sandbox capacity (sessions in use / max).
- Health: web `/api/health` (DB ping), worker heartbeat row every 30 s (`worker_heartbeats`), and sandbox `/health` checked by the worker.

### 7. Backups and key management
- Postgres PITR enabled. The restore drill is part of Phase 19b's G4 evidence.
- Storage bucket versioning + lifecycle (expire `exports/` after 7 days, `tmp/` after 1 day, keep thumbnails).
- `ENCRYPTION_KEY` in a secret manager with an offline backup. Losing it makes every stored token and keystore unreadable. Rotation: add version +1, re-encrypt in a background job (`scripts/reencrypt.ts`), retire the old key.
- **Android keystores** (only option B, Phase 15): nightly encrypted export to a separate bucket/account.

### 8. Deploy, smoke test, rollback
Post-deploy smoke test (Playwright against prod, test account):
- [ ] Sign up (email, with verification) + Google + GitHub sign-in
- [ ] Create project with an image → plan → confirm → screens build; canvas + preview render
- [ ] Chat edit + inline edit + code edit + theme switch
- [ ] Device preview QR opens in **store Expo Go** (Path B); Path C link opens on a phone
- [ ] Connect backend (test provider account) → data persists
- [ ] Zip export downloads; GitHub push to a test repo (empty and non-empty)
- [ ] Web publish URL loads (a backend-connected test app loads data through the **production** gateway)
- [ ] Android APK build reaches "finished" via the per-project webhook (can be async)
- [ ] Stripe live checkout with a real card, then refund; credits granted once
- [ ] Public showcase shows only approved projects when signed out

Rollback: Vercel instant rollback to the previous deployment; worker and sandbox redeploy the previous image tags. Migrations are forward-only (expand/contract), so old code keeps working on the new schema. Document this in `docs/runbook.md`.

## Files to create / change
- `worker/Dockerfile`, `sandbox/Dockerfile`, host config (`railway.json` / `fly.toml` / `render.yaml`)
- `.github/workflows/{ci.yml,release.yml}` (lint/typecheck/test on PR; migrate + deploy on `main`)
- `lib/env.ts` (per-process schemas incl. the sandbox allowlist), `.env.example`
- `lib/observability/{logger.ts,sentry.ts}`, `sentry.*.config.ts`
- `lib/ratelimit.ts`, `middleware.ts` (headers, rate limits), `lib/credits/spend-cap.ts`
- `app/api/health/route.ts`, `worker/heartbeat.ts`, `sandbox/health.ts`
- `scripts/{rotate-eas-webhooks.ts,backup-keystores.ts,reencrypt.ts}`
- `docs/runbook.md` (incidents, rollback, key rotation, takedown requests)

## Database changes
- `system_flags` and `worker_heartbeats` (`03` §8b).

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/health` | none | → `{ ok, db: "ok", worker: { lastSeenSecondsAgo }, sandbox: "ok" }` (no secrets, no dependency versions) |
| (all) | — | — | may return `429` (rate limited) or `503` (generation disabled / spend cap reached) |

## Acceptance checklist
- [ ] The production domain serves the app over HTTPS, with HSTS and security headers present.
- [ ] Web, worker and sandbox run from the same git sha, and Sentry shows releases for all three.
- [ ] CI on `main` applies migrations before deploying; a failing migration blocks the deploy.
- [ ] Every row of the callback/webhook table is configured and verified by the smoke test.
- [ ] The sandbox can't reach the DB (connection attempt fails) and has no platform secrets in its env (env dump test against the allowlist).
- [ ] Stripe is in live mode in prod and test mode elsewhere; prod webhooks verify.
- [ ] Rate limits return 429 under a burst; the kill switch and the daily spend cap return 503 for run-starting routes.
- [ ] The spend alert and the outbox-lag alert fire in a dry run (thresholds temporarily lowered).
- [ ] No secret values appear in logs (log scan test), and `/api/health` exposes nothing sensitive.
- [ ] Rollback rehearsed once: previous web deployment + previous worker/sandbox images run against the current DB.

## Claude Code prompt
```text
We're implementing Phase 19a (production deploy & operations) of AppBuilding.

Read first:
- docs/plan/phases/phase-19a-production-deploy.md (scope + acceptance checklist)
- docs/plan/02-architecture.md §6, docs/plan/04-tech-stack.md, docs/plan/08-security.md
- docs/plan/12-decisions-and-open-questions.md (hosts I picked; ADR-003 #17, #18, #21)

Implement the code/config side (I will click through provider dashboards myself):
1. worker/Dockerfile (Node only) and sandbox/Dockerfile (Node 22, pinned eas-cli, Playwright Chromium,
   prebuilt template node_modules, non-root) + host config for <FILL IN: Railway | Fly | Render>.
2. CI: lint/typecheck/test on PRs; on main: drizzle-kit migrate, then deploy web + worker + sandbox.
3. lib/env.ts per-process schemas (web, worker, sandbox); the sandbox schema rejects unexpected
   secret-looking vars; CI check against .env.example. DATABASE_URL_DIRECT for pg-boss/outbox/LISTEN.
3b. Root .gitignore for all build/export/scratch output + CI guard against huge/generated-file PRs.
4. Structured logging with redaction; Sentry for all three services; /api/health, worker heartbeat,
   sandbox health.
5. Rate limiting + security headers; GENERATION_DISABLED kill switch; DAILY_SPEND_CAP_USD check.
6. Alerts: queue age, outbox lag, stuck steps (heartbeat), failure rate, spend, ImageKit units,
   webhooks, sandbox capacity.
7. Scripts: rotate-eas-webhooks (per project), backup-keystores (option B only), reencrypt; docs/runbook.md.
8. A Playwright smoke-test suite runnable against any base URL with a test account.

Produce a step-by-step manual checklist in docs/runbook.md of every dashboard change needed
(domains, wildcard DNS for device preview, OAuth redirect URIs, GitHub App, backend provider app,
Stripe live webhook, storage CORS).
Do NOT build: data lifecycle, template upgrades, vendor watch, G4 evidence (Phase 19b).
Definition of done = Acceptance checklist items verifiable in code/CI; list the rest for me.
Finish by running lint, typecheck, tests; summarize what I must do manually, in order.
```

## Pitfalls & notes
- **LISTEN/NOTIFY through poolers.** Transaction-mode poolers break `LISTEN`. pg-boss, the outbox dispatcher and the SSE tailer need `DATABASE_URL_DIRECT`. On serverless web, the SSE route can fall back to short polling by `id > lastEventId`.
- **SSE on serverless.** Function max duration still applies to streaming responses. Return after ~250 s with a `retry:` hint; the client reconnects with `Last-Event-ID` (`02` A3).
- **Build-time public vars.** Changing `NEXT_PUBLIC_APP_URL` without a rebuild leaves the old value in client bundles. Always redeploy.
- **Worker scale-out.** Multiple worker replicas are fine with pg-boss row locks. Set per-queue concurrency, so builds don't starve thumbnails, and watch Anthropic rate limits.
- **Sandbox sizing.** Metro dev servers dominate memory (~300–600 MB each). Size instances and per-user caps from S3 measurements, and scale horizontally behind the proxy.
- **Region latency.** Keep web, worker, sandbox and DB in the same region.
- **Cost surprises.** Keep the console spend limit and `DAILY_SPEND_CAP_USD` below what you can afford to lose in a day.
