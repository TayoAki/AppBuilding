# Phase 13 — Generated-App Backend (Connect a Real Backend)

| | |
|---|---|
| Video chapter(s) | Ch.12 "Generated App Backend Setup" (01:32:00) |
| Branch | `feat/13-generated-app-backend` |
| Depends on | Phase 05 (jobs), Phase 08 (template data layer), Phase 09 (editor agent) |
| Effort (solo dev + Claude Code) | ~5–7 days (first adapter); ~2–3 days per extra adapter |
| New env vars | `BACKEND_PROVIDER`, `BACKEND_OAUTH_CLIENT_ID`, `BACKEND_OAUTH_CLIENT_SECRET` (`ENCRYPTION_KEY`, `ENCRYPTION_KEY_VERSION` if not already added) |

## Goal
A user clicks **Connect backend** in the workspace header, signs in to their own account at a backend provider, and the platform provisions a database (plus auth) for this generated app. The agent writes **one table per entity**, with ownership and row-level security. The app's data layer switches from the local AsyncStorage adapter to the remote adapter **without touching screen code**. Auth screens in the generated app (sign in / sign up / sign out) work against the connected backend.

## What we do differently from the video (and why)
- **Provider adapter, not a hard-wired vendor.** The video only supports CloudHike. We define a `BackendProvider` interface and pick the first adapter with `BACKEND_PROVIDER`. Which vendor ships first is a user decision (`12-decisions-and-open-questions.md`). Both candidates are described neutrally below.
- **Relational schema from the start.** The video's first version stored everything in one `app_records` JSON table and needed a second prompt to fix it. Our agent writes `backend/migrations/NNN_*.sql` with one table per `plan.entities` entry from day one.
- **Ownership + row-level security by default.** Every entity table gets `owner_id` and policies so users only see their own rows. The video showed "device" owner ids because auth wasn't wired. We use a stable anonymous device id until the generated app has auth, then migrate to the authenticated user id.
- **The agent never touches the remote database.** It writes migration files. A provisioning job (`backend.provision`) applies them through the provider API, records `schema_version`, and is idempotent.
- **No secret keys in the generated app.** Only `public_config` (URL + anon/public key) is written into `lib/data/config.ts`. Service/admin keys stay encrypted in `project_backends.secret_config_enc` and are used only by the worker.
- **Screens don't change on connect.** Screens use `lib/store/*` → `DataAdapter` (see `06-generated-app-template.md` §6). Connecting swaps the adapter in a platform-owned file.

## Provider options (pick one first; verify their current docs before coding)

| | Supabase | CloudHike (video) |
|---|---|---|
| Connect | OAuth app for third-party integrations; user authorizes our app to manage their org/projects | Platform-level "developer app" (client id + secret) registered in the provider's settings; user signs in with their own account |
| Provision | Management API: create project, wait for healthy, fetch URL + anon key | Provider auto-deploys DB/auth for the connected app (video uses its MCP/API) |
| Migrations | Run SQL through the Management API (or a DB connection using a service credential) | Provider's SQL/DB API |
| Auth | Built-in auth; email/password works immediately; Google/GitHub need the user's own OAuth client configured in the backend | Built-in auth (Better Auth-based per the video); social providers need configuring per app |
| RLS | Native Postgres RLS policies (`auth.uid()`) | Verify what row-level access control the provider offers; otherwise enforce in a provider-side function/API layer |
| Client | `@supabase/supabase-js` (adds an allowlisted import; needs template version bump + web shim check) | Provider REST/SDK (same allowlist/shim requirement) |

> Implementer: before Step 2, read the chosen provider's **current** OAuth-integration and management/provisioning docs. Endpoints, scopes and rate limits change. Write the findings into `lib/providers/backend/<name>/README.md`.

## Steps

### 1. Define the `BackendProvider` interface
`lib/providers/backend/types.ts`:
```ts
export interface BackendProvider {
  id: string;                                   // "supabase" | "cloudhike"
  getAuthorizeUrl(state: string, redirectUri: string): string;
  exchangeCode(code: string, redirectUri: string): Promise<OAuthTokens>;
  refresh?(tokens: OAuthTokens): Promise<OAuthTokens>;
  createProject(ctx: ConnCtx, input: { name: string; region?: string }): Promise<{ externalProjectId: string }>;
  waitUntilReady(ctx: ConnCtx, externalProjectId: string): Promise<void>;
  applyMigrations(ctx: ConnCtx, externalProjectId: string, migrations: { name: string; sql: string }[]): Promise<{ applied: string[] }>;
  getPublicConfig(ctx: ConnCtx, externalProjectId: string): Promise<PublicBackendConfig>;   // safe to ship
  getSecretConfig?(ctx: ConnCtx, externalProjectId: string): Promise<SecretBackendConfig>;   // never shipped
  enableAuthProviders(ctx: ConnCtx, externalProjectId: string, p: { emailPassword: boolean; google?: GoogleOAuthConfig }): Promise<void>;
  status(ctx: ConnCtx, externalProjectId: string): Promise<BackendStatus>;   // services, health, dashboard URL
  disconnect(ctx: ConnCtx): Promise<void>;     // revoke token if the provider supports it
}
```
`lib/providers/backend/index.ts` picks the implementation from `BACKEND_PROVIDER`. Add a `fake` provider for tests and evals: an in-memory Postgres schema or pg-mem.

### 2. OAuth connect flow
- `GET /api/backend/connect?projectId=` → `requireProjectAccess(projectId, "editor")`, create a signed `state` (projectId + userId + nonce, short TTL, stored in a cookie), redirect to `getAuthorizeUrl`.
- `GET /api/backend/callback?code&state` → verify state, `exchangeCode`, upsert `oauth_connections` (`provider = "backend:<id>"`, tokens encrypted with `lib/crypto/encrypt.ts`), then enqueue `backend.provision` for the project and redirect to the workspace with `?backend=connecting`.
- Register the callback URL in the provider's developer settings (local, preview and prod URLs). The video forgot this and had to add it afterwards.

### 3. Agent: write entity migrations
- Add the `backend_migration` tool to the Editor/Builder toolsets (see `05-ai-agent-design.md` §3). It writes `backend/migrations/NNN_<name>.sql`; the path allowlist allows only that folder.
- Prompt rules (add to CONTRACT.md §backend): one table per entity, `id uuid primary key default gen_random_uuid()`, `owner_id` (uuid/text), `created_at`, `updated_at`, snake_case columns mapped from entity fields, foreign keys for `ref` fields, RLS enabled with select/insert/update/delete policies `owner_id = <current user>`, indexes on `owner_id` and foreign keys. **Never** a generic `records(jsonb)` table.
- Generator helper `lib/ai/backend/entitiesToSql.ts` produces a deterministic baseline migration from `plan.entities`. The agent is asked to review/extend it rather than write from scratch. This is cheaper and more consistent.
- Validator rule `backend-sql`: parse SQL (`pgsql-ast-parser` or `libpg-query`), reject `DROP`, `TRUNCATE`, `GRANT`, `ALTER ROLE`, extensions not on the allowlist, and tables without RLS.

### 4. Provisioning job `backend.provision`
Worker handler `worker/handlers/backend-provision.ts`:
1. Load connection, decrypt tokens, refresh if expired.
2. If there's no `project_backends` row: `createProject` → `waitUntilReady` (poll with backoff, emit `run.status` events).
3. If there are no migrations yet, ask the agent to generate them (short edit run), then `applyMigrations` for files not yet recorded (track applied names in `project_backends` or a `schema_migrations` table on the remote DB).
4. `enableAuthProviders({ emailPassword: true })`.
5. `getPublicConfig` → store `public_config`; `getSecretConfig` → `secret_config_enc`.
6. Write the platform-owned `lib/data/config.ts` (adapter = remote, public config) as a new file revision with `author: "system"`.
7. Emit `run.done`; the preview reloads data (`reset.data` message to frames).

### 5. RemoteAdapter + auth in the template
- `lib/data/remote.ts` implements `DataAdapter` against the provider client using `public_config`, mapping collection names ↔ table names (`habits` → `habits`), camelCase ↔ snake_case.
- `lib/data/auth.ts`: `useSession`, `signIn`, `signUp`, `signOut`, `signInWithOAuth`. With the local adapter it's a fake session (Phase 08); with the remote adapter it uses the provider's auth.
- Anonymous mode: before the generated app has auth screens, use a persisted device id as `owner_id` (the RLS policy allows `owner_id = device id` via a header/claim, or the adapter signs in anonymously if the provider supports it). When auth screens are added, the agent writes a migration step to claim device rows on first sign-in.
- Preview runtime: add the provider client to the vendor bundle and the CSP `connect-src` (only the connected backend URL; see `07-preview-runtime.md` §6). Bump the template version.

### 6. Workspace UI
- Header button **Connect backend** → dialog explaining what will be created and which account → **Connect <Provider>**.
- After connect: header pill "Backend connected" → popover "Connected services": database ✓, auth ✓ (email/password), storage (if enabled), link "Open dashboard", **Disconnect**.
- Disconnect: confirm dialog; switch `lib/data/config.ts` back to local, mark `project_backends.status = 'disconnected'`, call `disconnect()`. **Never delete** the user's remote project automatically.

### 7. Social sign-in in generated apps
Google/GitHub sign-in requires the user to create OAuth credentials and configure them in **their** backend (redirect URLs for the Expo scheme + web). The video's agent added a Google button that didn't work for exactly this reason. Implement as a guided checklist in the Connected-services popover ("Enable Google sign-in" → instructions + fields → `enableAuthProviders({ google })`). Until it's configured, the agent hides social buttons (`config.authProviders` flag).

## Files to create / change
- `lib/providers/backend/{types.ts,index.ts,fake.ts,<provider>/index.ts,<provider>/README.md}`
- `app/api/backend/connect/route.ts`, `app/api/backend/callback/route.ts`, `app/api/projects/[projectId]/backend/route.ts`
- `worker/handlers/backend-provision.ts`
- `lib/ai/tools/backend-migration.ts`, `lib/ai/backend/entitiesToSql.ts`
- `lib/validate/backend-sql.ts`
- `templates/expo-app/lib/data/{remote.ts,auth.ts,config.ts.hbs}`, `templates/expo-app/CONTRACT.md` (backend section), `templates/expo-app/ALLOWED_IMPORTS.json`, `templates/expo-app/VERSION`
- `preview-runtime/src/vendor.ts` (provider client), CSP builder
- `components/workspace/backend/{ConnectBackendDialog.tsx,ConnectedServicesPopover.tsx,SocialAuthChecklist.tsx}`

## Database changes
- `oauth_connections` (if not created earlier): rows with `provider = 'backend:<id>'`.
- `project_backends`: `project_id`, `provider`, `external_project_id`, `public_config`, `secret_config_enc`, `schema_version`, `status`.
- New `runs.kind` value in use: `backend_provision`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/backend/connect?projectId=` | session + editor | → 302 provider authorize URL |
| GET | `/api/backend/callback?code&state` | session + signed state | → 302 `/workspace/:id?backend=connecting` |
| GET | `/api/projects/:id/backend` | viewer | → `{ provider, status, services[], dashboardUrl, authProviders }` (no secrets) |
| POST | `/api/projects/:id/backend/provision` | editor | → `{ runId }` (re-run / re-apply migrations) |
| POST | `/api/projects/:id/backend/auth-providers` | editor | `{ google?: { clientId, clientSecret } }` → `{ ok }` |
| DELETE | `/api/projects/:id/backend` | owner | → `{ ok }` (disconnect, keeps remote project) |

## Acceptance checklist
- [ ] Connect → provider consent → back in workspace; provisioning progress streams in chat; "Backend connected" pill appears.
- [ ] Remote DB has one table per entity, with `owner_id`, timestamps, foreign keys and RLS enabled (verified in the provider dashboard).
- [ ] Adding a habit in the preview creates a row in `habits`; reloading the preview shows it; a second test user can't see it.
- [ ] Email/password sign-up/sign-in works in the preview and in Expo Go (Phase 11).
- [ ] `lib/data/config.ts` contains only public config; a test asserts no `service_role`/secret patterns in any materialized file.
- [ ] Tokens in `oauth_connections` and `secret_config_enc` are encrypted (DB inspection shows `v1:` ciphertext).
- [ ] A viewer (or non-member) calling `connect`, `provision` or `DELETE` gets 403; `state` tampering is rejected.
- [ ] Disconnect reverts to the local adapter; the remote project is untouched.
- [ ] Re-running provision is idempotent (no duplicate tables, no re-applied migrations).
- [ ] The validator rejects a migration with `DROP TABLE` or a table without RLS.

## Claude Code prompt
```text
We're implementing Phase 13 (generated-app backend connect) of AppBuilding.

Read these first, in order:
- docs/plan/phases/phase-13-generated-app-backend.md (this phase — scope + acceptance checklist)
- docs/plan/06-generated-app-template.md §2, §3, §6, §7 (data layer + ownership rules)
- docs/plan/05-ai-agent-design.md §3 (tools) and §4 (validation)
- docs/plan/03-data-model.md §7 (oauth_connections, project_backends)
- docs/plan/08-security.md (token encryption, authz, secrets)

Implement exactly this phase:
1. BackendProvider interface + a `fake` provider + the provider selected by BACKEND_PROVIDER
   (<FILL IN: supabase | cloudhike>). Before writing that adapter, fetch and read the provider's
   current OAuth-integration and management API docs and summarize them in
   lib/providers/backend/<name>/README.md. Don't guess endpoints.
2. OAuth connect + callback routes with signed state; encrypted tokens in oauth_connections.
3. backend_migration agent tool + deterministic entitiesToSql baseline + backend-sql validator
   (one table per entity, owner_id, RLS; reject DROP/TRUNCATE/GRANT; never a JSON records table).
4. backend.provision worker job (idempotent), writing public config only into lib/data/config.ts.
5. Template: RemoteAdapter + auth.ts, CONTRACT.md backend section, template version bump,
   preview-runtime vendor + CSP connect-src for the backend URL.
6. Connect dialog, Connected-services popover, disconnect, social-auth checklist.

Definition of done = every item in the phase's Acceptance checklist; add tests for state verification,
authz (viewer/non-member → 403), SQL validator rules, and "no secrets in materialized files" using the fake provider.

Do NOT build: export/GitHub (Phase 14), native builds (Phase 15), billing gates (Phase 18).
Do NOT delete remote projects on disconnect.

Finish by running lint, typecheck and tests; then summarize what changed, any provider-doc surprises,
and any manual setup I must do (registering callback URLs in the provider dashboard).
```

## Pitfalls & notes
- **Callback URLs per environment.** Register local, preview and prod callback URLs with the provider. Tunnels (ngrok) change URLs on each run unless you reserve a domain. The video lost time to this twice.
- **Provider rate limits on project creation.** Free tiers often cap active projects per account. Surface the provider's error verbatim ("You've reached N free projects") rather than a generic failure.
- **Token refresh.** Long-lived projects outlive access tokens. Refresh in the worker before every provider call and handle revoked tokens by marking the connection `needs_reauth` and showing a reconnect banner.
- **Schema drift.** If the user edits tables in the provider dashboard, `applyMigrations` may fail. Show the SQL error and offer "Ask AI to fix migration"; never auto-drop.
- **Web vs native auth redirects.** OAuth sign-in in the generated app needs redirect URIs for the Expo scheme (`<scheme>://`), Expo Go (`exp://…`) and the web publish URL (Phase 16). Document all three in the checklist.
- **Preview data isolation.** Canvas frames and the interactive preview share the remote backend once connected. Seeding must not run against the remote DB (seed only into the local adapter). Show an empty state the first time, as the video does.
- **Public showcase previews must never use the remote adapter** (Phase 17). Materialize them with the local adapter regardless of `project_backends`.
