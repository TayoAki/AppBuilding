# 08 — Security Model

An AI app builder runs **untrusted, AI-generated code** for **untrusted users**, and it holds **other services' credentials** on their behalf. Those three facts drive everything below. Treat this file as a checklist each phase must satisfy. Every phase's acceptance checklist includes at least one item from here.

This file is the authoritative source for security rules (ADR-005). If a phase doc disagrees with it, this file wins and the phase doc gets fixed.

---

## 1. Threat model (short)

| Asset | Threat | Primary control |
|---|---|---|
| Platform sessions/cookies | XSS from generated code in the preview | Opaque-origin sandboxed iframes; generated code is never served from, or same-site with, the platform origin (§4) |
| Other users' projects | IDOR: guessing/using another `projectId` or a child id from another project | `requireProjectAccess()` on every route + job; child ids bound to the project (§2) |
| Private chat and prompts | Leaking through public pages, remixes or shares | Visibility never opens the workspace; remix copies files only (§2) |
| Users' GitHub / Expo / backend tokens | DB leak, logs, over-scoped tokens, untrusted code running next to them | AES-256-GCM at rest, least-privilege scopes, no logging (§3); credentials only ever reach the runner, never the sandbox (§5) |
| Build/export workers | Code execution through generated code during `expo export` / EAS / npm install | Fixed `package.json`, no lifecycle scripts, a sandbox with no secrets and per-job isolation (§5) |
| Our Anthropic bill / credits | Abuse via free accounts, prompt loops, model-triggered charges | Credit reservation before enqueue, rate limits, turn caps, spend cap, click-bound charges (§6, §7) |
| The agent | Prompt injection via prompts, uploaded images, pasted errors, copied projects | Narrow tools, path allowlist, no shell, no secrets in context, no charge or exposure change without a click (§6) |
| Generated apps' end users | Service keys shipped in app bundle; missing row security; private data made public | Public config only; owner checks in platform-derived tables / gateway; `shared` is owner-confirmed (§8) |
| Payment state | Forged webhooks, double-crediting, refunds and disputes | Signature verification + idempotency keys + clawback (§9) |

## 2. Authorization

- `lib/auth/require.ts` exports `requireUser()` and `requireProjectAccess(projectId, minRole: "viewer" | "editor" | "owner")`. **Every** route handler, server action and SSE endpoint that takes a `projectId` or `runId` calls one of them first. A run is authorized through its project.
- **Account-level runs** (`account_delete`, `export_my_data`) have no project. They're authorized by `runs.created_by = session user`, on every `/api/runs/:id/*` route.
- Workers re-check: job payloads are `{runId, projectId, userId}` (the run also records `created_by` and `billed_user_id`), and the handler verifies that `userId` still has the needed role on `projectId` before doing anything. Never trust a job payload just because it came from our own queue.
- **Child ids are bound to the project.** Every child object id a route accepts (`screenId`, `revisionId` / `baseRevisionId`, `snapshotId`, `assetId`, `messageId`, `sessionId`, `buildId`) is loaded with `WHERE id = ? AND project_id = ?`; a miss is 404. Each such route gets a test that passes an id from another project.
- **Visibility never satisfies `requireProjectAccess`.** `public` and `unlisted` projects are served only by the Phase 17 public endpoints, which return the latest snapshot's files, the stripped plan and the preview, and which also require `deleted_at is null`. Chat history, the event stream, uploaded assets (reference images, attachments), member lists, device sessions and exports are never public. No runs, edits or exports for non-members.
- **Remix** (duplicate by a non-member of a public/unlisted project) copies the latest snapshot's files and `projects.plan` minus `backend` only. It copies **no messages, no `messages.payload.api`, no assets** of kind `reference_image` / `chat_attachment`. A member's duplicate copies the UI chat text but never `payload.api`; the copy's Planner always starts a fresh session (`05` §3.1). `payload.api` is never sent to any client.
- Add a test per route that calls it as a different user and expects 404, not 403, so existence isn't leaked.
- **Membership.** Rows are keyed by `user_id`, never by email. `requireProjectAccess` requires a **verified** email. **Every share creates a pending invite**, whether or not the address has an account, and the member list shows "Invited" until it's accepted, so sharing can't be used to test which emails have accounts. A verified existing user also sees the invite in the app. Invite tokens: 32 random bytes, stored hashed, 7-day expiry, single-use, bound to the invited email and role at creation, void when the invite is removed or its role changes. Accepting requires a verified session email equal to the invited one. Changing your email doesn't inherit pending invites.
- **Sign-in.** Unverified users can't sign in (`requireEmailVerification: true`). An OAuth sign-in links to an existing account only when the provider asserts a verified email and the existing account's email is verified; an unverified password account for that address is replaced, not merged.
- **Server-owned user fields.** `plan`, `creditsBalance`, `monthlyCreditsBalance` and `role` are declared with `input: false` in the Better Auth config, and a test posts them to sign-up and update-user and expects them ignored (check the current Better Auth docs for the default when you write Phase 02).
- **Admin.** `role = 'admin'` is set only by `scripts/grant-admin.ts` against the DB, never through an API. Admin routes require a recent re-auth and write an audit log row.
- **CSRF.** Every state-changing route handler checks `Origin` (or `Sec-Fetch-Site: same-origin`) against the app URL, in addition to `SameSite=Lax`. Webhooks and the `/api/apps/*` gateway are exempt because they never read the session cookie (§8, §9).
- **Revocation.** Removing a member or lowering a role stops that member's device sessions on the project and invalidates their preview links in the same transaction. Open SSE streams end at the next forced reconnect (≤ 4 min, Phase 05), which re-authorizes.
- **Editor spend.** Editors charge the owner. Each membership has `daily_credit_cap` (default 50, owner-adjustable in the share dialog); the charge transaction enforces it, and the owner's ledger shows who spent what.

**Role matrix**: the single authoritative source (ADR-003 #23). Phase docs reference it rather than restating it.

| Action | viewer | editor | owner |
|---|---|---|---|
| View files, chat history, preview, QR / device preview (each member starts their own session; its URL is returned only to that member, never broadcast on the event stream) | ✅ | ✅ | ✅ |
| Send chat messages and edits, inline edits, code saves, theme switch (credits are always charged to the **project owner**, `runs.billed_user_id`, within the member's `daily_credit_cap`) | — | ✅ | ✅ |
| Rename project, duplicate | — | ✅ | ✅ |
| Export zip and download it, push to GitHub (with the acting user's GitHub connection, to a repo that user can access) | — | ✅ | ✅ |
| Native build (owner's Expo connection; one `eas_project_id` per project) | — | — | ✅ |
| Web publish / unpublish (owner's Expo connection) | — | — | ✅ |
| Connect/disconnect backend, provision or re-provision it, add backend extras (storage, email), mark a collection `shared`, change visibility, public Snack save, request showcase, share/invite, set member credit caps, delete | — | — | ✅ |
| Download Android signing key | — | — | ✅ + recent re-auth, audit-logged |
| Approve a project for the homepage showcase (D17) | admin only | | |

Any verified signed-in user can **Remix** a public/unlisted project (files and stripped plan only, see above).

## 3. Secrets and tokens

- `lib/crypto/encrypt.ts`: AES-256-GCM, random 12-byte IV, output `v{n}:{iv}:{ct}:{tag}` (base64). The key comes from `ENCRYPTION_KEY`; the version from `ENCRYPTION_KEY_VERSION`. Decrypt supports old versions, so keys can be rotated with a re-encrypt script.
- **Bind each ciphertext to its place:** pass `"{table}.{column}:{rowId}"` as the GCM additional data on encrypt and decrypt, so a ciphertext copied to another row or column fails to decrypt.
- Encrypted columns: `oauth_connections.*_token_enc`, `project_backends.secret_config_enc` (incl. Supabase `db_pass`), `project_backends.install_token_secret_enc`, `project_signing.*_enc`, `projects.eas_webhook_secret_enc`.
- `ENCRYPTION_KEY` lives on web and worker only. The sandbox and the runner never have it. Web decrypts only in the OAuth callbacks and the gateway; everything else decrypts in the worker.
- Never log tokens or full request bodies from OAuth/webhook routes. Add a log redaction helper in Phase 00.
- OAuth flows use `state` (random, bound to the session, single-use, 10-min TTL) and PKCE where the provider supports it. A callback without a valid `state` (for example an install started on the provider's site) stores nothing and redirects to the connect page.
- Least privilege:
  - GitHub: a **GitHub App** with `contents: write`, `pull_requests: write` and `metadata: read` on repos the user selects (plus `administration: write` only if we keep "create a repo for me", Phase 14). Not an OAuth App with `repo`. **Every GitHub call made for a user uses that user's user-to-server token.** We never mint installation tokens and never trust an `installation_id` from a callback or request. The target `{owner, repo}` is re-checked against the repos that token can reach (`GET /user/installations/:id/repositories`) when the export is requested and again in the worker.
  - Expo: the user creates a token scoped to their account. We store it and use it only inside build/publish jobs, and only in the runner (§5).
  - Backend provider: the OAuth token is organization-wide at the provider (`15` §5). We always use the **project owner's** connection, never the acting user's; we store the chosen organization and the `external_project_id` we created, and every later call targets that id (never a lookup by name).
- Disconnect = revoke at the provider (where an API exists) + delete our row.

## 4. Running generated code in the browser

See `07-preview-runtime.md` §6. In summary:
- `<iframe sandbox="allow-scripts">`, **never** `allow-same-origin` (both together let the frame remove its own sandbox). No `allow=` feature delegation (no geolocation, camera or microphone); the shims return permission-denied.
- **Everything under the preview runtime path is served with `Content-Security-Policy: sandbox allow-scripts`** plus the policy below, so the documents get an opaque origin even when someone opens them top-level instead of in our iframe.
- The runtime document is loaded **by URL from `PREVIEW_ORIGIN`** with a header CSP. Don't use `srcdoc`: a `srcdoc` document inherits the embedding page's CSP, which would force `'unsafe-eval'` into the app shell.
- **`PREVIEW_ORIGIN` and `DEVICE_PREVIEW_BASE_DOMAIN` are on a separate registrable domain from the app in production** (for example `brand-preview.dev`, not `preview.brand.com`). A sibling subdomain is same-site: it would receive `SameSite=Lax` cookies and could set cookies for the parent domain. Local dev may use `localhost`.
- **`harness.html` (the server-side render page) is never deployed with the web app.** It exists only in the sandbox image, and it renders project code inside the same sandboxed iframe and CSP as production.
- The runtime receives files via `postMessage`; it has no API access or tokens. Validate every message with Zod, and check `event.source` matches the expected frame. Strings from the frame (`runtime.error`, console lines, element labels) are rendered as text, never as HTML. `storage.snapshot` is size-capped and stored under the parent's own `projectId`.
- CSP on the runtime document: `default-src 'none'; script-src <runtime origin> 'unsafe-eval'` (needed to evaluate transpiled modules), `style-src 'unsafe-inline'`, **`font-src <runtime origin> data:`** (Ionicons), `img-src <image provider> <our storage> https://picsum.photos data: blob:` (a host list, not `https:`, so a public project can't beacon to arbitrary hosts), `connect-src` limited to the project's connected backend URL or our gateway (`07` §6).
- The platform's own pages get a strict CSP (no `'unsafe-eval'`) and `frame-ancestors 'self'`.
- Chat markdown, plan cards and build summaries are rendered with raw HTML disabled, links limited to `https:` / `mailto:`, and no remote images (only our own asset URLs).

## 5. Running generated code on servers: the sandbox and the runner (ADR-003 #18, ADR-005)

Two separate deployables sit behind the worker. The **sandbox** runs generated code and never sees a credential. The **runner** holds a user's credential for one job and never runs generated code.

**Sandbox service** (`sandbox/`): **render smoke tests and thumbnails** (Playwright), **`expo export`** (web publish) and **the Path B device-preview dev servers** (Metro).
- It has **no secrets and no database access**: no `DATABASE_URL`, no `ANTHROPIC_API_KEY`, no encryption key, no OAuth secrets, **no user tokens** (no Expo token, no App Store Connect key, no webhook secret).
- The worker calls it over HTTP: `POST {SANDBOX_URL}/jobs` with a per-job token and **signed storage URLs** for inputs (the materialized project tarball) and outputs (screenshots, export dir, logs).
- **Job tokens are asymmetric.** The worker signs them with `SANDBOX_SIGNING_KEY` (Ed25519 private key); the sandbox and runner hold only `SANDBOX_VERIFY_KEY` (the public key). A token names one job and one service, and expires in 15 minutes. Code that gets execution in the sandbox can't mint tokens.
- **Results come back only through the signed output URLs** and the HTTP response. There are no callbacks into the worker, and the worker validates everything it reads back (sizes, JSON shape, file list).
- **Isolation per job and per session:** each job or dev-server session runs in its own container or microVM where the host supports it; otherwise at least its own OS user, working directory, and network namespace with loopback not shared. One session can't read another's files, environment or ports.
- It runs as **non-root** with an **env allowlist**, CPU/memory/time limits, and Chromium's own sandbox left on (no `--no-sandbox`).
- **Egress:** allow only the npm registry, Expo APIs, the image provider and our storage. Block loopback, link-local (cloud metadata) and private ranges for Chromium and for anything generated code can drive. If the host can't restrict egress, pick a different host for this service; the no-secrets property is not a substitute.
- **Paths are validated again here.** When unpacking a tarball or handling a file update for a session, reject absolute paths, `..`, symlinks and hard links, and anything outside the allowed tree (`06` §3). Don't rely on the VFS having checked.
- Path B dev servers are capped per sandbox instance (e.g. 4 concurrent sessions), idle-stopped when there has been no workspace heartbeat and no connected device for 5 minutes (Phase 11), and reachable only through the session's unguessable URL. The proxy forwards only the manifest, bundle, asset and hot-reload paths; Metro's debugger, inspector, symbolicate and open-in-editor endpoints are not exposed (`11` / Phase 11).

**Runner service** (`runner/`): every `eas-cli` step that needs the owner's Expo token or an App Store Connect key (`whoami`, `init`, `webhook:create`, `build`, `build:view`, `deploy`).
- It receives the credential as a per-job value (never persisted, redacted in logs) with the same kind of signed job token.
- It runs **one job per fresh container** (or process under a fresh OS user with its own temp dir, wiped afterwards), with no Metro, no Chromium and no `expo export`.
- It never evaluates project code. For a native build it unpacks the tarball and lets `eas-cli` upload it (`EAS_NO_VCS=1`); the configs `eas-cli` reads are platform-generated and static. For web publish it receives only the static `dist/` the sandbox exported, and runs `eas deploy` on it. Spikes S4 and S5 confirm that these commands execute no project module (Phase 00b).
- No DB access, no platform secrets, egress limited to Expo and our storage.

Both:
- `package.json`, `app.json` (static), `babel.config.js` and `metro.config.js` are **platform-generated**; the agent can't write them (`06` §3). So there are no custom Babel plugins, no Metro config tricks, and no `postinstall` scripts. Route files that run in Node (`+api`, `+html`, `+middleware`) are rejected at the path layer.
- Install dependencies from a **prebuilt, cached `node_modules`** for the pinned template version baked into the image, mounted read-only, or with `npm ci --ignore-scripts` against a committed lockfile. Never install agent-chosen packages.
- Railway/Fly/Render can each run them as their own services, on a private network separate from the DB.

## 6. Prompt injection and agent safety

- The agent has **no shell, no network tool and no secrets in its context**. Its tools only read and write files inside the project's allowlisted paths (`05` §3).
- Reference images and pasted error text are untrusted. The system prompt tells the model to treat them as data. So is anything a collaborator typed, and anything in the project's files.
- **The model can't spend credits or change exposure on its own.** Wording can be injected, so the server doesn't rely on the prompt:
  - `confirm_plan` only shows the Confirm button again with a note; the build starts from the button click.
  - `update_app` above a threshold (more than 11 credits, or any added screen) returns a cost card; the run starts when the person clicks, and the click is bound to a hash of the normalized `AppChange`.
  - `setup_backend` never takes `sharedCollections` or `acceptEmailPolicy` from the model. It only proposes; the owner confirms extras and shared collections in a dialog (§8).
- **Copied projects carry no agent history.** Remix and duplicate never copy `messages.payload.api` (§2).
- The agent never writes SQL and never gets provider/MCP tools. App tables are **derived by platform code** from `plan.collections` (`03` §10), and `setup_backend` is a narrow, typed tool executed by server code. The reference app chose this because raw MCP tools would let the model read secrets and reach the person's other projects.
- **Model output that becomes code or SQL is validated, not trusted:**
  - Collection and field names must match `^[a-z][a-zA-Z0-9]{0,40}$`, must not be reserved words, and are always quoted as identifiers in DDL and gateway SQL. Reject at Foundation/Data validation.
  - Platform renderers (`themes.ts`, `schema.ts`, `seed.ts`, layouts) emit every plan string through `JSON.stringify` and every identifier through the same regex check. `AppPlanSchema` strings have maximum lengths. Screen ids are kebab-case at the schema level.
- **Server-side image fetches** (ImageKit generation URLs written by generators, and "replace image" by URL): `https` only, host pinned to `IMAGEKIT_URL_ENDPOINT` and this project's folder for generation URLs; for pasted URLs resolve the host and refuse loopback, link-local and private addresses, follow no redirects, cap size and time, and re-encode the result. The server extracts generation URLs, caps the count, URL-encodes descriptions, strips control characters and length-limits them before fetching (Phase 12).
- The eval suite includes injection cases: an image, a pasted error and a file comment that each tell the agent to confirm, charge or share. Expected: no state change without a click (`10`).

## 7. Abuse and cost controls

- Credits are **charged before** any agent job is enqueued, in the same transaction as the run and its outbox row (`11` §3, ADR-003 #7). The plan-status compare-and-set happens inside that transaction, with ledger reference `charge:build:<planMessageId>`, so a button click and a second tab can't start two builds of one plan.
- **`DAILY_SPEND_CAP_USD` is enforced from Phase 06**, in `lib/ai/client.ts` before every model call (sum of today's `ai_usage` for the billed user), and it stays on after billing exists.
- **Planner chat is free but bounded:** a per-user daily message quota (default 100; then 1 credit per message), and a forced session break when the replayed history passes a token ceiling.
- **Images are charged where they're generated.** A change run counts the generation URLs kept after post-processing and charges for each; a per-user daily image cap applies.
- **Refunds go back to the balance they came from.** The ledger rows record the monthly/permanent split; monthly refunds after the period ended are dropped. A canceled run refunds only steps that hadn't started.
- **Signup grants** are keyed on a canonical email (lowercase, `+tag` removed, provider dot rules applied), refuse disposable domains, and are capped per IP per day.
- Rate limits (per user + per IP) on: project creation, messages, sign-up, uploads, share invites, publish, public reports. Use a Postgres token bucket or Upstash.
- One repair round per generated file and a cap on Planner tool calls per message (`05` §10). Runs can be canceled by the user and by an admin.
- Uploads: max 5 files × 10 MB, MIME allowlist (`image/png`, `image/jpeg`, `image/webp`, `image/gif`; GIFs are flattened to their first frame before going to the model), magic-byte sniffing after upload, re-encode or strip EXIF before passing to the model or the app.
- Web-published apps and public showcase projects need a ToS, a "report" link, and an admin unpublish switch. You're hosting user content. A report only adds an entry to the admin queue; it never changes a project's status by itself.

## 8. Generated apps' security (what we ship to *their* users)

- Only **public** backend config (URL + **publishable** key) goes into `lib/backend/config.ts`. Secret/service keys and the database password stay in `project_backends.secret_config_enc` and are only used server-side during provisioning.
- Every derived `app_<name>` table has `owner_id`. How it's enforced depends on the mode (ADR-003 #19):
  - **Direct mode** (provider client + row-level security): **auth is always deployed**, even if the app has no sign-in screen. The app signs in **anonymously** on first launch, and `owner_id = auth.uid()` in the RLS policies. A client-supplied install id is never trusted, because anyone with the publishable key could spoof it. When the person later signs up, the provider links the anonymous user to the real one. Row-level security is **enabled and forced on every `app_*` table, `app_values` included**, with default deny.
  - **Gateway mode** (platform `/api/apps/[id]/…`): used when the provider has no RLS/anonymous auth, or when the person doesn't want auth deployed. The gateway issues a signed per-install token on first launch and enforces `owner_id` server-side.
  - Unknown fields or wrong types → 400, so nothing is silently dropped.
- **Gateway limits.** The projectId is in every app bundle, so the gateway is a public API:
  - Install tokens are signed with a per-project key (`project_backends.install_token_secret_enc`), bind the projectId, and expire (30 days, refreshed on use).
  - Issuance is rate-limited per IP and per project. Each install and each project has row, byte and upload quotas.
  - `/email/me` works only for a verified signed-in user of the app, never an anonymous install.
  - The gateway never reads the platform session cookie and never sends `Access-Control-Allow-Credentials`. The `null`-origin allowance applies to `/api/apps/*` only.
- Shared/public collections (e.g. a product catalog) must be marked explicitly (`shared: true` in the collection). The policy then allows public read, owner-only write. The video's CodeRabbit finding about `shared = true` came from exactly this ambiguity.
- **`shared` can't change silently.** Once a collection has been applied to a backend, turning `shared` on (private → public read) needs an owner confirmation in the UI that names the collection; it's never taken from model tool input or from an editor's chat. The policy change is an explicit drop-and-create in one transaction.
- Provisioning always uses the project owner's provider connection and the stored `external_project_id` (§3).
- Secret scanning before zip/GitHub export, before a project becomes public, and on every snapshot a public endpoint serves (regex for common key formats, including Stripe `sk_live_` / `sk_test_` / `whsec_`, Supabase `sb_secret_`, Anthropic `sk-ant-`, Expo and GitHub tokens, plus an entropy check) blocks anything that looks like a secret.

## 9. Webhooks

- Stripe: verify `stripe-signature` with `STRIPE_WEBHOOK_SECRET`. Insert `(provider, event_id)` into `processed_webhook_events` in the handler's transaction (duplicate → already processed, return 200). Re-read the subscription from Stripe instead of trusting event order. Grant credits only for an invoice with `amount_paid > 0` whose customer maps to a user we created the customer for. `charge.refunded` and `charge.dispute.created` claw back the credits granted by that invoice (the balance may go to zero, never below) and flag the account.
- EAS (ADR-003 #17): webhooks are created **per EAS project** after `eas init` (`eas webhook:create --event BUILD`), each with its **own random secret** stored in `projects.eas_webhook_secret_enc`. The route is `/api/webhooks/eas/:projectId`: load that project's secret from the URL's id, verify `expo-signature` (HMAC-SHA1 of the raw body, `sha1=<hex>`, `15` §3) **before parsing the body**, then require `native_builds.project_id = :projectId` for the build id in the payload. **Treat the payload as a hint:** re-fetch the build from EAS with the owner's token before writing `status` or `artifact_url`, so a leaked secret can't forge artifacts.
- Webhook routes use the raw body for verification, return 2xx fast, and enqueue the real work. The worker has `STRIPE_SECRET_KEY` for that work (re-reading subscriptions, canceling on account deletion).

## 10. Process

- **CodeRabbit** (or another automated reviewer) on every PR, plus a `security-review` pass on PRs touching `lib/auth`, `lib/crypto`, `api/webhooks`, `api/apps`, `preview-runtime`, `worker/handlers/build-*`, `sandbox/`, `runner/`.
- `npm audit` / Dependabot weekly; pin GitHub Actions by SHA.
- Before launch: one manual pentest pass against §1 (two accounts, try to read and modify each other's projects, runs, files, exports; pass child ids from the other account's project; try XSS from a generated screen; open the runtime and device-preview URLs top-level; try to make the agent write outside allowed paths, confirm a plan or share a collection from injected text).

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

**Delete project.** Waits for active runs. Soft delete (`deleted_at`), then a hard-delete job after 30 days removes project rows, revisions, snapshots, assets and storage objects. A soft-deleted project is gone from every route at once, public pages and remix included. It doesn't touch the person's GitHub repo, EAS project/website or backend project; the dialog says so (as in the reference app) and links to them.

**Delete account** (`runs.kind = account_delete`, settings page, re-auth required):
1. Cancel any Stripe subscription immediately (no refund unless policy says so) and detach payment methods.
2. Revoke provider tokens where an API exists (GitHub, backend provider; Expo tokens are user-created, so tell them to revoke theirs), then delete `oauth_connections` and `project_signing` rows (offer keystore download first).
3. Delete all owned projects (immediate hard delete, with the same scope as above), their export zips and device sessions, memberships and pending invites. Remixes other people made stay theirs; public Snack saves are listed for the person to remove.
4. Anonymize `ai_usage` and `credit_ledger` (`user_id` → null), null `runs.created_by` and `project_members.invited_by` in other owners' projects, and delete the Better Auth user, sessions and accounts. Keep the `signup_grants` row (email **hash** only), so re-signing up with the same email doesn't re-grant free credits.
5. Email a confirmation. The whole job is idempotent and resumable.

**Export my data.** A settings action produces a zip: profile, projects (materialized `export` target for each), chat history, ledger. It reuses the Phase 14 export job. The email links to a signed-in settings page that issues a 5-minute download URL; it never contains the download link itself.

**Privacy policy must state:** prompts, uploads and generated code are sent to the model provider (Anthropic) under our API terms for processing. We don't use them for training, and we don't enable any data-sharing programs (`01` P14).
