# 08 — Security Model

An AI app builder runs **untrusted, AI-generated code** for **untrusted users**, and it holds **other services' credentials** on their behalf. Those three facts drive everything below. Treat this file as a checklist each phase must satisfy. Every phase's acceptance checklist includes at least one item from here.

---

## 1. Threat model (short)

| Asset | Threat | Primary control |
|---|---|---|
| Platform sessions/cookies | XSS from generated code in the preview | Opaque-origin sandboxed iframes (§4) |
| Other users' projects | IDOR: guessing/using another `projectId` | `requireProjectAccess()` on every route + job (§2) |
| Users' GitHub / Expo / backend tokens | DB leak, logs, over-scoped tokens | AES-256-GCM at rest, least-privilege scopes, no logging (§3) |
| Build/export workers | Code execution through generated code during `expo export` / EAS / npm install | Fixed `package.json`, no lifecycle scripts, ephemeral containers (§5) |
| Our Anthropic bill / credits | Abuse via free accounts, prompt loops | Credit reservation before enqueue, rate limits, turn caps (§7) |
| The agent | Prompt injection via prompts, uploaded images, pasted errors | Narrow tools, path allowlist, no shell, no secrets in context (§6) |
| Generated apps' end users | Service keys shipped in app bundle; missing row security | Public config only; owner checks in platform-derived tables / gateway (§8) |
| Payment state | Forged webhooks, double-crediting | Signature verification + idempotency keys (§9) |

## 2. Authorization

- `lib/auth/require.ts` exports `requireUser()` and `requireProjectAccess(projectId, minRole: "viewer" | "editor" | "owner")`. **Every** route handler, server action and SSE endpoint that takes a `projectId` or `runId` calls one of them first. A run is authorized through its project.
- Workers re-check: job payloads are `{runId, projectId, userId}` (the run also records `created_by` and `billed_user_id`), and the handler verifies that `userId` still has the needed role on `projectId` before doing anything. Never trust a job payload just because it came from our own queue.
- Public projects (`visibility = public`) allow anonymous **read** of files and preview only, never runs, edits or exports of private metadata.
- Add a test per route that calls it as a different user and expects 404, not 403, so existence isn't leaked.

**Role matrix**: the single authoritative source (ADR-003 #23). Phase docs reference it rather than restating it.

| Action | viewer | editor | owner |
|---|---|---|---|
| View files, chat history, preview, QR / device preview | ✅ | ✅ | ✅ |
| Send chat messages and edits, inline edits, code saves, theme switch (credits are always charged to the **project owner**, `runs.billed_user_id`) | — | ✅ | ✅ |
| Rename project, duplicate | — | ✅ | ✅ |
| Export zip, push to GitHub (with the acting user's GitHub connection) | — | ✅ | ✅ |
| Native build (owner's Expo connection; one `eas_project_id` per project) | — | — | ✅ |
| Web publish / unpublish (owner's Expo connection) | — | — | ✅ |
| Connect/disconnect backend, change visibility, request showcase, share/invite, delete | — | — | ✅ |
| Download Android signing key | — | — | ✅ + recent re-auth, audit-logged |
| Approve a project for the homepage showcase (D17) | admin only | | |

## 3. Secrets and tokens

- `lib/crypto/encrypt.ts`: AES-256-GCM, random 12-byte IV, output `v{n}:{iv}:{ct}:{tag}` (base64). The key comes from `ENCRYPTION_KEY`; the version from `ENCRYPTION_KEY_VERSION`. Decrypt supports old versions, so keys can be rotated with a re-encrypt script.
- Encrypted columns: `oauth_connections.*_token_enc`, `project_backends.secret_config_enc` (incl. Supabase `db_pass`), `project_signing.*_enc`, `projects.eas_webhook_secret_enc`.
- Never log tokens or full request bodies from OAuth/webhook routes. Add a log redaction helper in Phase 00.
- OAuth flows use `state` (random, bound to the session, single-use, 10-min TTL) and PKCE where the provider supports it.
- Least privilege:
  - GitHub: a **GitHub App** with `contents: write`, `pull_requests: write` and `metadata: read` on repos the user selects. Not an OAuth App with `repo`.
  - Expo: the user creates a token scoped to their account. We store it and use it only inside build/publish jobs.
- Disconnect = revoke at the provider (where an API exists) + delete our row.

## 4. Running generated code in the browser

See `07-preview-runtime.md` §6. In summary:
- `<iframe sandbox="allow-scripts">`, **never** `allow-same-origin` (both together let the frame remove its own sandbox).
- The runtime receives files via `postMessage`; it has no API access or tokens. Validate every message with Zod, and check `event.source` matches the expected frame.
- CSP on the runtime document: `default-src 'none'; script-src <runtime origin> 'unsafe-eval'` (needed to evaluate transpiled modules), `style-src 'unsafe-inline'`, **`font-src <runtime origin> data:`** (Ionicons), `img-src https: data: blob:`, `connect-src` limited to the project's connected backend URL or our gateway (`07` §6).
- The platform's own pages get a strict CSP and `frame-ancestors 'self'`.

## 5. Running generated code on servers: the sandbox service (ADR-003 #18)

Generated code is processed on our infrastructure during **render smoke tests and thumbnails** (Playwright), **`expo export`** (web publish), **the Path B device-preview dev servers** (Metro) and **materialization for EAS** (EAS builds on Expo's machines, but `eas-cli` reads project config locally).

- **Sandbox service:** a separate deployable (`sandbox/`) that runs every one of those steps. It has **no secrets and no database access**: no `DATABASE_URL`, no `ANTHROPIC_API_KEY`, no encryption key, no OAuth secrets.
  - The worker calls it over HTTP: `POST {SANDBOX_URL}/jobs` with a per-job token (HMAC with `SANDBOX_TOKEN_SECRET`, scoped to one job, 15-min expiry), **signed storage URLs** for inputs (the materialized project tarball) and outputs (screenshots, export dir, logs), and, only for publish/build steps, the owner's Expo token as a per-job value (never persisted, redacted in logs).
  - It returns results by uploading to the signed output URLs and answering the HTTP call (or a callback to the worker with the same per-job token).
  - It runs as **non-root** with an **env allowlist**, a temp workdir per job, CPU/memory/time limits, and **egress restricted** where the host allows: npm registry, Expo APIs, the image provider, our storage. It can't reach our DB or internal admin endpoints.
  - Railway/Fly/Render can each run it as its own service. If the host can't restrict egress, rely on the no-secrets property and keep it on a separate private network from the DB.
- `package.json`, `app.json` (static), `babel.config.js` and `metro.config.js` are **platform-generated**; the agent can't write them (`06` §3). So there are no custom Babel plugins, no Metro config tricks, and no `postinstall` scripts.
- Install dependencies from a **prebuilt, cached `node_modules`** for the pinned template version baked into the sandbox image, or with `npm ci --ignore-scripts` against a committed lockfile. Never install agent-chosen packages.
- Path B dev servers are capped per sandbox instance (e.g. 4 concurrent sessions), idle-stopped after 15 minutes without a connected client, and reachable only through the session's unguessable URL (`11` / Phase 11).

## 6. Prompt injection and agent safety

- The agent has **no shell, no network tool and no secrets in its context**. Its tools only read and write files inside the project's allowlisted paths (`05` §3).
- Reference images and pasted error text are untrusted. The system prompt tells the model to treat them as data. Tools can't do anything worse than write project files, which the user can see and undo anyway.
- The agent never writes SQL and never gets provider/MCP tools. App tables are **derived by platform code** from `plan.collections` (`03` §10), and `setup_backend` is a narrow, typed tool executed by server code. The reference app chose this because raw MCP tools would let the model read secrets and reach the person's other projects.
- ImageKit generation URLs written by generators: the server extracts them, caps the count, URL-encodes descriptions, strips control characters and length-limits them before fetching (Phase 12).

## 7. Abuse and cost controls

- Credits are **charged before** any agent job is enqueued, in the same transaction as the run and its outbox row (`11` §3, ADR-003 #7). Phase 06 already logs usage and enforces `DAILY_SPEND_CAP_USD` per user until billing exists.
- Rate limits (per user + per IP) on: project creation, messages, sign-up, uploads, share invites, publish. Use a Postgres token bucket or Upstash.
- One repair round per generated file and a cap on Planner tool calls per message (`05` §10). Runs can be canceled by the user and by an admin.
- Uploads: max 5 files × 10 MB, MIME allowlist (`image/png`, `image/jpeg`, `image/webp`, `image/gif`; GIFs are flattened to their first frame before going to the model), magic-byte sniffing after upload, re-encode or strip EXIF before passing to the model or the app.
- Web-published apps and public showcase projects need a ToS, a "report" link, and an admin unpublish switch. You're hosting user content.

## 8. Generated apps' security (what we ship to *their* users)

- Only **public** backend config (URL + **publishable** key) goes into `lib/backend/config.ts`. Secret/service keys and the database password stay in `project_backends.secret_config_enc` and are only used server-side during provisioning.
- Every derived `app_<name>` table has `owner_id`. How it's enforced depends on the mode (ADR-003 #19):
  - **Direct mode** (provider client + row-level security): **auth is always deployed**, even if the app has no sign-in screen. The app signs in **anonymously** on first launch, and `owner_id = auth.uid()` in the RLS policies. A client-supplied install id is never trusted, because anyone with the publishable key could spoof it. When the person later signs up, the provider links the anonymous user to the real one.
  - **Gateway mode** (platform `/api/apps/[id]/…`): used when the provider has no RLS/anonymous auth, or when the person doesn't want auth deployed. The gateway issues a signed per-install token on first launch and enforces `owner_id` server-side.
  - Unknown fields or wrong types → 400, so nothing is silently dropped.
- Shared/public collections (e.g. a product catalog) must be marked explicitly (`shared: true` in the collection). The policy then allows public read, owner-only write. The video's CodeRabbit finding about `shared = true` came from exactly this ambiguity.
- Secret scanning before zip/GitHub export (regex for common key formats + entropy check) blocks anything that looks like a secret.

## 9. Webhooks

- Stripe: verify `stripe-signature` with `STRIPE_WEBHOOK_SECRET`. Insert `(provider, event_id)` into `processed_webhook_events` in the handler's transaction (duplicate → already processed, return 200). Re-read the subscription from Stripe instead of trusting event order.
- EAS (ADR-003 #17): webhooks are created **per EAS project** after `eas init` (`eas webhook:create --event BUILD`), each with its **own random secret** stored in `projects.eas_webhook_secret_enc`. Verify `expo-signature` (HMAC-SHA1 of the raw body, `sha1=<hex>`, `15` §3) with that project's secret, resolved via `eas_build_id → native_builds → project`. **Treat the payload as a hint:** re-fetch the build from EAS with the owner's token before writing `status` or `artifact_url`, so a leaked secret can't forge artifacts.
- Webhook routes use the raw body for verification, return 2xx fast, and enqueue the real work.

## 10. Process

- **CodeRabbit** (or another automated reviewer) on every PR, plus a `security-review` pass on PRs touching `lib/auth`, `lib/crypto`, `api/webhooks`, `preview-runtime`, `worker/handlers/build-*`, `sandbox/`.
- `npm audit` / Dependabot weekly; pin GitHub Actions by SHA.
- Before launch: one manual pentest pass against §1 (two accounts, try to read and modify each other's projects, runs, files, exports; try XSS from a generated screen; try to make the agent write outside allowed paths).

## 11. Data lifecycle: retention, deletion, export

| Data | Retention |
|---|---|
| `tmp/` uploads (before a project exists) | 1 day (bucket lifecycle rule) |
| Export zips | 7 days, then deleted (re-export anytime) |
| `run_events` (project-scoped stream) | 30 days (run summaries stay in `runs`; file history stays in `file_revisions`) |
| Planner API turns (`messages.payload.api`) | life of the project. Deleted with it |
| `ai_usage` | kept for accounting; `user_id` nulled on account deletion |
| Deleted projects | soft-deleted for 30 days (restorable), then hard-deleted by a job |
| Stripe records | per Stripe's and tax obligations; we keep only ids |

**Delete project.** Waits for active runs. Soft delete (`deleted_at`), then a hard-delete job after 30 days removes project rows, revisions, snapshots, assets and storage objects. It doesn't touch the person's GitHub repo, EAS project/website or backend project; the dialog says so (as in the reference app) and links to them.

**Delete account** (`runs.kind = account_delete`, settings page, re-auth required):
1. Cancel any Stripe subscription immediately (no refund unless policy says so) and detach payment methods.
2. Revoke provider tokens where an API exists (GitHub, backend provider; Expo tokens are user-created, so tell them to revoke theirs), then delete `oauth_connections` and `project_signing` rows (offer keystore download first).
3. Delete all owned projects (immediate hard delete, with the same scope as above), memberships and pending invites.
4. Anonymize `ai_usage` and `credit_ledger` (`user_id` → null), and delete the Better Auth user, sessions and accounts. Keep the `signup_grants` row (email **hash** only), so re-signing up with the same email doesn't re-grant free credits.
5. Email a confirmation. The whole job is idempotent and resumable.

**Export my data.** A settings action produces a zip: profile, projects (materialized `export` target for each), chat history, ledger. It reuses the Phase 14 export job.

**Privacy policy must state:** prompts, uploads and generated code are sent to the model provider (Anthropic) under our API terms for processing. We don't use them for training, and we don't enable any data-sharing programs (`01` P14).

