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
| Generated apps' end users | Service keys shipped in app bundle; missing row security | Public config only; RLS / owner checks in migrations (§8) |
| Payment state | Forged webhooks, double-crediting | Signature verification + idempotency keys (§9) |

## 2. Authorization

- `lib/auth/require.ts` exports `requireUser()` and `requireProjectAccess(projectId, minRole: "viewer" | "editor" | "owner")`. **Every** route handler, server action and SSE endpoint that takes a `projectId` or `runId` calls one of them first. A run is authorized through its project.
- Workers re-check: job payloads carry `userId` + `projectId`, and the handler verifies the relationship before doing anything. Never trust a job payload just because it came from our own queue.
- Public projects (`visibility = public`) allow anonymous **read** of files and preview only, never runs, edits or exports of private metadata.
- Add a test per route that calls it as a different user and expects 404, not 403, so existence isn't leaked.

## 3. Secrets and tokens

- `lib/crypto/encrypt.ts`: AES-256-GCM, random 12-byte IV, output `v{n}:{iv}:{ct}:{tag}` (base64). The key comes from `ENCRYPTION_KEY`; the version from `ENCRYPTION_KEY_VERSION`. Decrypt supports old versions, so keys can be rotated with a re-encrypt script.
- Encrypted columns: `oauth_connections.*_token_enc`, `project_backends.secret_config_enc`, `project_signing.*_enc`.
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
- CSP on the runtime document: `default-src 'none'; script-src <runtime origin> 'unsafe-eval'` (needed to evaluate transpiled modules), `style-src 'unsafe-inline'`, `img-src https: data: blob:`, `connect-src` limited to the project's connected backend URL.
- The platform's own pages get a strict CSP and `frame-ancestors 'self'`.

## 5. Running generated code on servers

Generated code executes on our infrastructure during **thumbnail rendering**, **`expo export`** (web publish) and **materialization for EAS** (EAS runs it on Expo's machines, but `eas-cli` reads project config locally).

- `package.json`, `app.json`, `babel.config.js` and `metro.config.js` are **platform-generated**; the agent can't write them (`06` §3). So there are no custom Babel plugins, no Metro config tricks, and no `postinstall` scripts.
- Install dependencies from a **prebuilt, cached `node_modules`** for the pinned template version, or with `npm ci --ignore-scripts` against a committed lockfile. Never install agent-chosen packages.
- Run `expo export` and Playwright renders in an **ephemeral, unprivileged container**: no platform env vars except what the step needs, read-only root, a temp workdir, CPU/memory/time limits, and egress restricted to npm registry/Expo APIs when possible.
- The user's Expo token goes into that container's env only for the duration of the command.

## 6. Prompt injection and agent safety

- The agent has **no shell, no network tool and no secrets in its context**. Its tools only read and write files inside the project's allowlisted paths (`05` §3).
- Reference images and pasted error text are untrusted. The system prompt tells the model to treat them as data. Tools can't do anything worse than write project files, which the user can see and undo anyway.
- `backend_migration` only writes SQL **files**. Provisioning applies them with the user's backend credentials in a separate job. Validate migrations against a denylist (no `DROP SCHEMA`, no `GRANT` to `anon` beyond the RLS pattern, no extensions) and show the SQL to the user before the first apply.
- Validate `image_url` prompts: length cap, URL-encode, strip control characters.

## 7. Abuse and cost controls

- Credits are reserved **before** enqueueing any agent job (Phase 18). Phase 06 already logs usage, and has a hard per-user daily spend cap env until billing exists.
- Rate limits (per user + per IP) on: project creation, messages, sign-up, uploads, share invites, publish. Use a Postgres token bucket or Upstash.
- Turn and fix-iteration caps per run (`05` §10). Runs can be canceled by the user and by an admin.
- Uploads: max 5 files × 10 MB, MIME allowlist (`image/png`, `image/jpeg`, `image/webp`), magic-byte sniffing after upload, re-encode or strip EXIF before passing to the model or the app.
- Web-published apps and public showcase projects need a ToS, a "report" link, and an admin unpublish switch. You're hosting user content.

## 8. Generated apps' security (what we ship to *their* users)

- Only **public** backend config (URL + anon/public key) goes into `lib/data/config.ts`. Service keys stay in `project_backends.secret_config_enc` and are only used server-side during provisioning.
- Every agent-written table has `owner_id` (or `user_id`) and row-level security policies (or the provider's equivalent) so users can only read and write their own rows. The validator checks that each migration creating a table also enables RLS and creates owner policies.
- Shared/public collections (e.g. a product catalog) must be marked explicitly (`shared: true` in the plan entity). The policy then allows public read, owner-only write. The video's CodeRabbit finding about `shared = true` came from exactly this ambiguity.
- Secret scanning before zip/GitHub export (regex for common key formats + entropy check) blocks anything that looks like a secret.

## 9. Webhooks

- Stripe: verify `stripe-signature` with `STRIPE_WEBHOOK_SECRET`; record `event.id` with a unique constraint; handlers are idempotent.
- EAS: verify the signature header with `EAS_WEBHOOK_SECRET` (check the current header name/algorithm in Expo docs); map by `eas_build_id`.
- Webhook routes use the raw body for verification, return 2xx fast, and enqueue the real work.

## 10. Process

- **CodeRabbit** (or another automated reviewer) on every PR, plus a `security-review` pass on PRs touching `lib/auth`, `lib/crypto`, `api/webhooks`, `preview-runtime`, `worker/handlers/build-*`.
- `npm audit` / Dependabot weekly; pin GitHub Actions by SHA.
- Before launch: one manual pentest pass against §1 (two accounts, try to read and modify each other's projects, runs, files, exports; try XSS from a generated screen; try to make the agent write outside allowed paths).
