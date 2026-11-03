# 04 — Tech Stack & Environment Variables

"Default" is what the plan assumes. "Alternatives" are drop-ins behind the same adapter. Open choices are collected in `12-decisions-and-open-questions.md`.

---

## 1. Platform (the builder itself)

| Layer | Default | Alternatives | Notes |
|---|---|---|---|
| Framework | **Next.js** (latest stable, App Router, TypeScript strict) | — | Pin the exact version in Phase 00 |
| UI | **Tailwind CSS + shadcn/ui** + lucide icons | DaisyUI | Matches the video |
| Panels / canvas | `react-resizable-panels`; `react-zoom-pan-pinch` (or a small custom transform layer) | `@xyflow/react` | Canvas frames are absolutely positioned iframes |
| Code viewer | **Shiki** (read-only) → Monaco when editing is added | CodeMirror 6 | |
| Client state | **Zustand** (workspace store: selection, frames, run state) | Jotai | Server data via fetch + SSE, no React Query needed initially |
| Long lists | `@tanstack/react-virtual` | — | Chat history, file tree |
| Worker build | `tsx` in dev; **tsup** + `tsconfig.worker.json` for the production bundle | esbuild script | Worker shares `lib/` with the web app |
| Database | **Postgres** | Neon · Railway · Supabase · CloudHike · RDS | Only `DATABASE_URL` matters |
| ORM | **Drizzle ORM** + drizzle-kit migrations | Prisma | `push` locally, `generate`+`migrate` in prod |
| Auth | **Better Auth** (email/password + Google + GitHub) | Provider-hosted Better Auth (e.g. CloudHike) · Clerk · Auth.js | Self-hosted Better Auth keeps users in our DB |
| Object storage | **S3-compatible** via `@aws-sdk/client-s3` + presigner | R2 · Railway buckets · MinIO (dev) · CloudHike storage | `StorageProvider` adapter |
| Queue / jobs | **pg-boss** (pinned major ≥ 10; `createQueue()` per queue, explicit policies) on the same Postgres, fed by a **transactional outbox** (`job_outbox`) | Inngest · Trigger.dev · BullMQ+Redis · CloudHike Jobs | `lib/jobs` hides the choice |
| Realtime | **SSE** (`/api/projects/:id/events`) + Postgres `LISTEN/NOTIFY` | Pusher/Ably · Redis pub/sub | Events persisted in project-scoped `run_events`. `LISTEN` needs a **direct/session-mode** connection; it doesn't work through transaction-mode poolers (PgBouncer transaction mode, some serverless drivers). Phase 05 adds a polling fallback. Check this when picking hosted Postgres. |
| AI | **Anthropic TypeScript SDK**, `AI_MODEL=claude-opus-5-5` | — | See `05-ai-agent-design.md` |
| Validation | **Zod** (env, API bodies, tool inputs, postMessage) | — | |
| AST / codemods | `@babel/parser`, `@babel/traverse`, `recast` (formatting-preserving prints) | ts-morph | Inline editing + validators |
| Type checking generated code | `typescript` compiler API (in-memory host) | — | Worker only |
| Headless browser | **Playwright + Chromium** | Puppeteer | Smoke tests + thumbnails, in the **sandbox service** (no secrets) |
| Device preview | **Path B (default): hosted Expo dev server** (`expo start` per session in the sandbox) + store Expo Go | **Path A: Expo Snack** (`snack-sdk`) once a published version supports the template SDK · **Path C**: web preview in the phone browser | `DEVICE_PREVIEW_MODE` (`15` §1, ADR-002) |
| Native builds | **EAS Build** via `eas-cli` (`EAS_NO_VCS=1`, `--non-interactive`) with the **project owner's** Expo token | GitHub Actions + local `expo prebuild` (video tried and dropped) | |
| Web publish | `expo export -p web` (in the sandbox) + **EAS Hosting** (`eas deploy --prod --dev-domain <slug>`) | Cloudflare Pages · our own static hosting | `HostingProvider` |
| AI images | **ImageKit** URL generation/transforms | Image model + our storage · Unsplash for stock | `ImageProvider` |
| Generated-app backend | `BackendProvider` with one adapter first: **Supabase** or **CloudHike** (decide) | Firebase · our own multi-tenant API | Phase 13 |
| GitHub export | **GitHub App** (user-to-server token) + Octokit: Contents API for the first commit to an empty repo, Git Data API after | OAuth App (`repo` scope) — avoid | Phase 14 |
| Payments | **Stripe** Checkout + Customer Portal + webhooks | Razorpay · Paddle · Lemon Squeezy | `BillingProvider` |
| Email (verification, invites, receipts) | Resend / Postmark / provider SMTP behind `lib/email` | CloudHike email | **Needed from Phase 02** (signup credits are granted on email verification, ADR-003 #22); invites use it in Phase 14 |
| Error tracking | Sentry | — | Web + worker |
| Tests | **Vitest** (unit), **Playwright** (e2e) | — | `10-testing-and-evals.md` |
| CI | GitHub Actions: lint, typecheck, test, build | — | + CodeRabbit on PRs (optional) |
| Hosting | **Vercel** (web) + **Railway/Fly/Render** (worker + **sandbox** as separate services) | Everything on one Railway project | Phase 19 |

## 2. Generated apps

| Layer | Choice |
|---|---|
| Runtime | Expo **SDK 58** (store Expo Go's only SDK; D7) |
| Navigation | expo-router (file-based) |
| Language | TypeScript |
| Styling | `StyleSheet.create` + `colors` tokens from `@/theme` (CSS variables in preview) |
| Icons | `@expo/vector-icons` |
| Local data | AsyncStorage via `lib/backend/local.ts` (generic `useCollection` / `useStoredValue` hooks) |
| Remote data/auth | `lib/backend/<provider>.ts` for the connected `BackendProvider` (direct + RLS with anonymous sign-in, or the platform gateway) |

## 3. Environment variables (canonical names)

Validated at boot by `lib/env.ts` (Zod). The web app and the worker share the same schema, with per-process "required" flags. **Never** prefix a secret with `NEXT_PUBLIC_`.

| Variable | Used by | Phase | Secret | Notes |
|---|---|---|---|---|
| `DATABASE_URL` | web, worker | 00 | ✅ | Postgres connection string (pooled for web if using serverless) |
| `DATABASE_URL_DIRECT` | web (SSE `LISTEN`), worker (pg-boss), migrations | 05 | ✅ | direct/session-mode connection; optional if `DATABASE_URL` is already direct |
| `NEXT_PUBLIC_APP_URL` | web | 00 | — | e.g. `http://localhost:3000`; prod domain later |
| `BETTER_AUTH_SECRET` | web | 02 | ✅ | `openssl rand -base64 32` |
| `BETTER_AUTH_URL` | web | 02 | — | same as app URL |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | web | 02 | ✅ | Platform sign-in |
| `GITHUB_LOGIN_CLIENT_ID` / `GITHUB_LOGIN_CLIENT_SECRET` | web | 02 | ✅ | Platform sign-in OAuth app (separate from the export GitHub App) |
| `S3_ENDPOINT` / `S3_REGION` / `S3_BUCKET` / `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` | web, worker | 03 | ✅ | Any S3-compatible store |
| `ANTHROPIC_API_KEY` | worker | 06 | ✅ | Worker only. The web tier never calls the model directly. |
| `AI_MODEL` | worker | 06 | — | default `claude-opus-5-5` |
| `AI_BUILDER_MODEL` | worker | 08 | — | model for all generators (Foundation, Data, Components, Screens); default `claude-opus-5-5`; `AI_MODEL` is the Planner. Change only with eval evidence (D9) |
| `MAX_SCREENS_DEFAULT` | worker | 06 | — | default `3` |
| `AGENT_SCREEN_CONCURRENCY` | worker | 08 | — | screens built in parallel (default 1 = sequential, so the first screen can serve as style reference; 2 max) |
| `ENCRYPTION_KEY` / `ENCRYPTION_KEY_VERSION` | web, worker | 02 | ✅ | 32 random bytes base64; versioned for rotation |
| `PREVIEW_ORIGIN` | web | 07 | — | optional separate origin for preview runtime |
| `SANDBOX_URL` | worker | 08 | — | base URL of the sandbox service (`08` §5) |
| `SANDBOX_TOKEN_SECRET` | worker, sandbox | 08 | ✅ | HMAC key for per-job tokens; the **only** secret the sandbox holds |
| `DEVICE_PREVIEW_MODE` | web, worker | 11 | — | `devserver` (default, Path B) \| `snack` (Path A, only if a published `snack-sdk` supports the template SDK) |
| `DEVICE_PREVIEW_BASE_DOMAIN` | sandbox, worker | 11 | — | domain for per-session dev-server URLs (Path B, if S3 picks the reverse-proxy mechanism) |
| `EMAIL_PROVIDER_API_KEY` / `EMAIL_FROM` | web, worker | 02 | ✅ / — | verification and invite emails (`lib/email`) |
| `SMTP_URL` | web, worker | 02 | ✅ | dev only: Mailpit SMTP (`smtp://localhost:1025`); production uses the provider API key |
| `DAILY_SPEND_CAP_USD` | worker | 06 | — | per-user model-spend cap before billing exists (default `5`); kept as a safety valve after Phase 18 |
| `IMAGEKIT_URL_ENDPOINT` | worker, generated apps | 12 | — | public |
| `IMAGEKIT_PRIVATE_KEY` | worker | 12 | ✅ | never sent to clients or generated apps |
| `IMAGEKIT_ENABLED` | worker | 12 | — | `true` → generators write ImageKit generation URLs; otherwise placeholders |
| `BACKEND_PROVIDER` | web, worker | 13 | — | e.g. `supabase` \| `cloudhike` |
| `BACKEND_OAUTH_CLIENT_ID` / `BACKEND_OAUTH_CLIENT_SECRET` | web | 13 | ✅ | The platform's developer app at the backend provider |
| `GITHUB_APP_ID` / `GITHUB_APP_SLUG` / `GITHUB_APP_CLIENT_ID` / `GITHUB_APP_CLIENT_SECRET` / `GITHUB_APP_PRIVATE_KEY` | web, worker | 14 | ✅ | Export GitHub App |
| `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_SECRET` / `STRIPE_PRICE_PRO_MONTHLY` | web | 18 | ✅ | |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | web | 18 | — | |
| `CREDIT_USD_VALUE` / `CREDIT_MARKUP` | web, worker | 18 | — | only for the optional cost-based credit mode (`11` §3); default flat pricing ignores them |
| `PUBLISH_HOSTING_PROVIDER` | worker | 16 | — | optional; default `eas` |
| `GENERATION_DISABLED` | web, worker | 19 | — | kill switch: run-starting routes return 503 (also settable as a DB flag in `system_flags`) |
| `SENTRY_DSN` | web, worker | 19 | — | |

Per-user and per-project secrets (GitHub user tokens, Expo access tokens, backend provider tokens and `db_pass`, Android keystores, per-project EAS webhook secrets) are **not** env vars. They live encrypted in the DB (`03-data-model.md` §7).

## 4. Local development services

`docker-compose.yml` (Phase 00):
- `postgres:16` on 5432
- `minio` on 9000/9001 with a bootstrap bucket
- `mailpit` for verification and invite emails (from Phase 02)

Scripts (`package.json`):
- `dev`: Next.js
- `worker`: `tsx watch worker/index.ts`
- `sandbox`: `tsx watch sandbox/server.ts` (local stand-in for the sandbox service; same no-secrets env allowlist)
- `db:generate` / `db:migrate` / `db:push` / `db:studio`
- `preview-runtime:build`: esbuild bundle of `preview-runtime/`
- `test`, `test:e2e`, `eval`: agent eval suite
- `lint`, `typecheck`
