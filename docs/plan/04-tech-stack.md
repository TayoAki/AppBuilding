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
| Database | **Postgres** | Neon · Railway · Supabase · CloudHike · RDS | Only `DATABASE_URL` matters |
| ORM | **Drizzle ORM** + drizzle-kit migrations | Prisma | `push` locally, `generate`+`migrate` in prod |
| Auth | **Better Auth** (email/password + Google + GitHub) | Provider-hosted Better Auth (e.g. CloudHike) · Clerk · Auth.js | Self-hosted Better Auth keeps users in our DB |
| Object storage | **S3-compatible** via `@aws-sdk/client-s3` + presigner | R2 · Railway buckets · MinIO (dev) · CloudHike storage | `StorageProvider` adapter |
| Queue / jobs | **pg-boss** on the same Postgres | Inngest · Trigger.dev · BullMQ+Redis · CloudHike Jobs | `lib/jobs` hides the choice |
| Realtime | **SSE** + Postgres `LISTEN/NOTIFY` | Pusher/Ably · Redis pub/sub | Events persisted in `run_events` |
| AI | **Anthropic TypeScript SDK**, `AI_MODEL=claude-opus-5-5` | — | See `05-ai-agent-design.md` |
| Validation | **Zod** (env, API bodies, tool inputs, postMessage) | — | |
| AST / codemods | `@babel/parser`, `@babel/traverse`, `recast` (formatting-preserving prints) | ts-morph | Inline editing + validators |
| Type checking generated code | `typescript` compiler API (in-memory host) | — | Worker only |
| Headless browser | **Playwright + Chromium** | Puppeteer | Smoke tests + thumbnails (worker) |
| Device preview | **Expo Snack** (`snack-sdk`) + Expo Go | — | Pinned SDK |
| Native builds | **EAS Build** via `eas-cli` with the user's Expo token | GitHub Actions + local `expo prebuild` (video tried and dropped) | |
| Web publish | `expo export -p web` + **EAS Hosting** (`eas deploy`) | Cloudflare Pages · our own static hosting | `HostingProvider` |
| AI images | **ImageKit** URL generation/transforms | Image model + our storage · Unsplash for stock | `ImageProvider` |
| Generated-app backend | `BackendProvider` with one adapter first: **Supabase** or **CloudHike** (decide) | Firebase · our own multi-tenant API | Phase 13 |
| GitHub export | **GitHub App** (user-to-server token) + Octokit Git Data API | OAuth App (`repo` scope) — avoid | Phase 14 |
| Payments | **Stripe** Checkout + Customer Portal + webhooks | Razorpay · Paddle · Lemon Squeezy | `BillingProvider` |
| Email (invites) | Resend / Postmark / provider SMTP | CloudHike email | Only needed from Phase 14 |
| Error tracking | Sentry | — | Web + worker |
| Tests | **Vitest** (unit), **Playwright** (e2e) | — | `10-testing-and-evals.md` |
| CI | GitHub Actions: lint, typecheck, test, build | — | + CodeRabbit on PRs (optional) |
| Hosting | **Vercel** (web) + **Railway/Fly/Render** (worker) | Everything on one Railway project | Phase 19 |

## 2. Generated apps

| Layer | Choice |
|---|---|
| Runtime | Expo (pinned SDK supported by Snack + current Expo Go) |
| Navigation | expo-router (file-based) |
| Language | TypeScript |
| Styling | `StyleSheet` + theme tokens via `useTheme()` |
| Icons | `@expo/vector-icons` |
| Local data | AsyncStorage via `lib/data` LocalAdapter |
| Remote data/auth | `RemoteAdapter` for the connected `BackendProvider` |

## 3. Environment variables (canonical names)

Validated at boot by `lib/env.ts` (Zod). The web app and the worker share the same schema, with per-process "required" flags. **Never** prefix a secret with `NEXT_PUBLIC_`.

| Variable | Used by | Phase | Secret | Notes |
|---|---|---|---|---|
| `DATABASE_URL` | web, worker | 00 | ✅ | Postgres connection string (pooled for web if using serverless) |
| `NEXT_PUBLIC_APP_URL` | web | 00 | — | e.g. `http://localhost:3000`; prod domain later |
| `BETTER_AUTH_SECRET` | web | 02 | ✅ | `openssl rand -base64 32` |
| `BETTER_AUTH_URL` | web | 02 | — | same as app URL |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | web | 02 | ✅ | Platform sign-in |
| `GITHUB_LOGIN_CLIENT_ID` / `GITHUB_LOGIN_CLIENT_SECRET` | web | 02 | ✅ | Platform sign-in OAuth app (separate from the export GitHub App) |
| `S3_ENDPOINT` / `S3_REGION` / `S3_BUCKET` / `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` | web, worker | 03 | ✅ | Any S3-compatible store |
| `ANTHROPIC_API_KEY` | worker | 06 | ✅ | Worker only. The web tier never calls the model directly. |
| `AI_MODEL` | worker | 06 | — | default `claude-opus-5-5` |
| `AI_MODEL_PLAN` / `AI_MODEL_BUILD` / `AI_MODEL_EDIT` | worker | 08 | — | optional per-stage overrides; unset = `AI_MODEL`. Only change with eval evidence (`11` §5, D9) |
| `MAX_SCREENS_DEFAULT` | worker | 06 | — | default `3` |
| `AGENT_MAX_TURNS` / `AGENT_MAX_FIX_ITERATIONS` / `AGENT_SCREEN_CONCURRENCY` | worker | 08 | — | 40 / 3 / 2 |
| `ENCRYPTION_KEY` / `ENCRYPTION_KEY_VERSION` | web, worker | 02 | ✅ | 32 random bytes base64; versioned for rotation |
| `PREVIEW_ORIGIN` | web | 07 | — | optional separate origin for preview runtime |
| `IMAGEKIT_URL_ENDPOINT` | worker, generated apps | 12 | — | public |
| `IMAGEKIT_PRIVATE_KEY` | worker | 12 | ✅ | never sent to clients or generated apps |
| `BACKEND_PROVIDER` | web, worker | 13 | — | e.g. `supabase` \| `cloudhike` |
| `BACKEND_OAUTH_CLIENT_ID` / `BACKEND_OAUTH_CLIENT_SECRET` | web | 13 | ✅ | The platform's developer app at the backend provider |
| `GITHUB_APP_ID` / `GITHUB_APP_SLUG` / `GITHUB_APP_CLIENT_ID` / `GITHUB_APP_CLIENT_SECRET` / `GITHUB_APP_PRIVATE_KEY` | web, worker | 14 | ✅ | Export GitHub App |
| `EAS_WEBHOOK_SECRET` | web | 15 | ✅ | Verify EAS build webhooks |
| `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_SECRET` / `STRIPE_PRICE_PRO_MONTHLY` | web | 18 | ✅ | |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | web | 18 | — | |
| `CREDIT_USD_VALUE` | web, worker | 18 | — | model-cost USD per credit, see `11` |
| `SENTRY_DSN` | web, worker | 19 | — | |

Per-user secrets (GitHub user tokens, Expo access tokens, backend provider tokens, Android keystores) are **not** env vars. They live encrypted in the DB (`03-data-model.md` §7).

## 4. Local development services

`docker-compose.yml` (Phase 00):
- `postgres:16` on 5432
- `minio` on 9000/9001 with a bootstrap bucket
- optional `mailpit` for invite emails (Phase 14)

Scripts (`package.json`):
- `dev`: Next.js
- `worker`: `tsx watch worker/index.ts`
- `db:generate` / `db:migrate` / `db:push` / `db:studio`
- `preview-runtime:build`: esbuild bundle of `preview-runtime/`
- `test`, `test:e2e`, `eval`: agent eval suite
- `lint`, `typecheck`
