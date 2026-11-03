# Phase 13 — Generated-App Backend (Connect a Real Backend)

| | |
|---|---|
| Video chapter(s) | Ch.12 "Generated App Backend Setup" (01:32:00). Reference repo: Prompt 7.1 (connect + deploy) and Prompt 8.1 (real tables) |
| Branch | `feat/13-generated-app-backend` |
| Depends on | Phase 05 (jobs), Phase 08 (collections + `@/lib/backend` hooks), Phase 09 (Planner tools, change pipeline) |
| Effort (solo dev + Claude Code) | ~5–7 days for the first adapter in **direct + RLS** mode; ~6–8 days if it needs the **platform gateway** (R14); ~2–3 days per extra adapter |
| New env vars | `BACKEND_PROVIDER`, `BACKEND_OAUTH_CLIENT_ID`, `BACKEND_OAUTH_CLIENT_SECRET` (`ENCRYPTION_KEY`, `ENCRYPTION_KEY_VERSION` if not already added) |

## Goal
The user clicks **Connect Backend** in the workspace header and signs in to **their own** account at a backend provider. The platform then creates a backend project for this app. It always deploys a database. In **direct + RLS** mode it **always deploys auth too**, using anonymous sign-in for apps without sign-in screens (ADR-003 #19). In gateway mode it deploys auth only if the code uses `useAuth`. Storage is deployed only if the code uses `uploadFile`. The app's tables are **derived from `projects.plan.collections` by platform code**: one typed table per collection, ownership enforced. The app's `@/lib/backend` switches from the local provider to the remote one **without any screen changing**. Sign-in screens work against the connected auth. The header shows **Backend connected**, with a services popover.

## What we do differently from the video (and why)
- **A provider adapter, not a hard-wired vendor.** The video supports only CloudHike. A `BackendProvider` interface plus `BACKEND_PROVIDER` picks the first adapter (decision D10).
- **Relational schema from the start**, not one JSON record table that has to be migrated later (the video and the reference app both hit that).
- **The agent never writes SQL and never touches the remote backend.** Tables come from collections deterministically. The model gets one narrow tool, `setup_backend`.
- **No credentials in the app.** Only public config ships (URL, publishable key, or the platform gateway URL). Service keys stay encrypted in `project_backends.secret_config_enc` and are used only by server code.

### Reference app notes
- **R12:** the Planner's only backend tool is `setup_backend`, and server code calls the provider (MCP `create_project` / `deploy_services` in the CloudHike case). Raw MCP tools would let the model read secrets (`get_environment`) and reach the person's other projects.
- **R13:** tables are `app_<name>` with `id`, `owner_id`, timestamps and one typed column per field (`string`→`text`, `number`→`double precision`, `boolean`→`boolean`, `string[]`→`text[]`). `references` becomes an FK `on delete set null`. `useStoredValue` goes into one `app_values` table, the only JSON column. Migrations are **additive only** (removing a field leaves its column) and run in one locked transaction, from `setup_backend` and from the first data request after the collections change. A write with an unknown field or wrong type is refused with 400.
- **R14:** apps can't hold database credentials. When the provider has no row-level security or functions for app code (CloudHike at the time), **our platform serves the app's data API** (`/api/apps/[projectId]/…`), checks the user's token against the project's auth, and writes to the person's database.
- Ownership: a record's owner is the signed-in person, or else a random per-install id. Collections are private to their owner unless `shared`. Only the creator can change a record.
  - **Amended (ADR-003 #19):** in **direct + RLS** mode a client-supplied install id can't be verified, because anyone with the publishable key could claim it. So direct mode **always deploys auth and signs every install in anonymously** (`owner_id = auth.uid()`). If the provider has no anonymous sign-in, use **gateway mode** for that app. Gateway mode can keep install ids, because the platform issues and signs them (`X-Install-Token`, HMAC with the project's secret).
- Deploy only what's used: the database always; auth if the code uses `useAuth` (and **always in direct mode**, see above); storage if it uses `uploadFile`. Email requires the person's explicit acceptance of the provider's email policy. Jobs/Redis can deploy but aren't usable without server code, so the Planner says so.
- Sync: screens load records on open. **Only preview frames poll (every 5 s), and only while the agent is changing screens.** There's no offline write queue.

## Provider options (pick one first; verify their current docs before coding)

| | Supabase-style (direct + RLS) | CloudHike (reference app; gateway) |
|---|---|---|
| Connect | OAuth app (org settings → OAuth Apps); `https://api.supabase.com/v1/oauth/authorize` + `/v1/oauth/token`, **PKCE**; scopes set on the app (`15` §6) | Platform-level "developer app" (client id + secret); user connects their own account (`/connect` flow) |
| Provision | `POST /v1/projects` (`projects:write`; needs `db_pass`, `name`, `organization_slug`). **Store `db_pass` encrypted**: it can't be read back. Then wait until healthy; `GET /v1/projects/{ref}/api-keys?reveal=` (`secrets:read`) | Server calls MCP `create_project` / `deploy_services`; status and .env via the REST API |
| Schema | Derived DDL via `POST /v1/projects/{ref}/database/query` (`database:write`, **Beta**: wrap it behind `runSql` so a change is one file) | Platform applies derived DDL with the stored DB credential |
| Keys | New **`sb_publishable_…`** (ships in the app) and **`sb_secret_…`** (server only; 401 from browsers). Don't use the legacy anon/service_role keys, which are deprecated by end of 2026 | Publishable `authKey` only in the app |
| Data path | App → provider client with the publishable key; **RLS** policies `owner_id = auth.uid()`, with **anonymous sign-in** for apps without sign-in screens | App → **platform gateway** → person's DB (no app credentials) |
| Auth | Built-in; email/password immediately; Google/GitHub need the person's OAuth client | Built-in (Better Auth-based); social providers configured per app |

> Supabase facts above were verified on 2026-11-03 (`15` §6). Before Step 2, re-check the chosen provider's **current** docs (rate limits, anonymous sign-in support, project-creation limits per plan) and write the findings into `lib/providers/backend/<name>/README.md`. Spike S6 must have passed for this provider (`14` A15).

## Steps

### 1. `BackendProvider` interface (`lib/providers/backend/types.ts`)
```ts
export interface BackendProvider {
  id: string;                                         // "supabase" | "cloudhike" | "fake"
  dataPath: "direct" | "gateway";
  getAuthorizeUrl(state: string, redirectUri: string): string;
  exchangeCode(code: string, redirectUri: string): Promise<OAuthTokens>;
  refresh?(t: OAuthTokens): Promise<OAuthTokens>;
  ensureProject(ctx: ConnCtx, input: { name: string }): Promise<{ externalProjectId: string }>;  // idempotent
  ensureServices(ctx: ConnCtx, id: string, services: ("database"|"auth"|"storage"|"email")[]): Promise<ServiceStatus[]>;
  runSql(ctx: ConnCtx, id: string, sql: string): Promise<void>;          // used only by derived DDL
  getPublicConfig(ctx: ConnCtx, id: string): Promise<PublicBackendConfig>;   // safe to ship
  getSecretConfig(ctx: ConnCtx, id: string): Promise<SecretBackendConfig>;   // never shipped
  verifyUserToken?(cfg: SecretBackendConfig, token: string): Promise<{ userId: string } | null>; // gateway mode
  status(ctx: ConnCtx, id: string): Promise<BackendStatus>;
  disconnect(ctx: ConnCtx): Promise<void>;
}
```
Add a `fake` provider (pg-mem or a local schema) for tests and evals.

### 2. OAuth connect flow
- `GET /api/projects/:id/backend/connect` → `requireProjectAccess(id, "owner")`, random `state` in an HttpOnly cookie bound to user + project (short TTL), redirect to `getAuthorizeUrl`.
- `GET /api/backend/callback?code&state` → verify state, `exchangeCode`, upsert `oauth_connections` (`provider = "backend:<id>"`, one per person, tokens encrypted). Then enqueue `backend.provision` and redirect to the workspace with `?backend=connecting`.
- Register the exact callback URL per environment. Open the workspace at `NEXT_PUBLIC_APP_URL` locally, because that's where the state cookie lives (the reference app documents this).

### 3. Schema derivation (`lib/backend-schema/derive.ts`)
- `deriveDdl(collections, current)` → additive DDL only: `create table if not exists app_<name>(id text primary key, owner_id text not null, created_at timestamptz default now(), updated_at timestamptz default now(), …typed columns)`, `alter table … add column if not exists …` for new fields, FKs `on delete set null` for `references`, an `owner_id` index, and `app_values(owner_id, key, value jsonb)`.
- Direct mode adds RLS policies: private collections `owner_id = auth.uid()` for every operation (installs without sign-in are anonymous users, so `auth.uid()` always exists); shared collections allow authenticated select (anonymous included) with owner-only writes.
- `applySchema(projectId)` takes `pg_advisory_xact_lock(hash(projectId))` (or the provider's equivalent) in one transaction, then records `project_backends.schema_version` (the hash of the collections).
- Unit-test the DDL for every field type, references, shared vs private, and the "remove a field → column stays" case.

### 4. `setup_backend` Planner tool + provisioning job
- Tool input: `{ extras?: ("storage"|"email")[], sharedCollections?: string[], acceptEmailPolicy?: boolean }`. Server checks: the backend is connected, the app is built, and email only if `acceptEmailPolicy` (and the latest user message accepted it).
- `worker/handlers/backend-provision.ts` (idempotent; runs from the tool, after connect, and **automatically at the end of any build/change of a connected app**):
  1. Load the connection, decrypt, refresh if expired (`project_backends.status = reconnect_required` on failure).
  2. `ensureProject` (one backend project per app).
  3. Services: always `database`. Add `auth` if any file imports `useAuth`, **or always in direct mode** (enable anonymous sign-in there). Add `storage` if `uploadFile` is used, plus the requested extras.
  4. `applySchema`.
  5. `getPublicConfig` → `project_backends.public_config`; `getSecretConfig` → `secret_config_enc`.
  6. Update `projects.plan.backend` (public fields only) and re-render `lib/backend/config.ts` (`provider`, `apiUrl` = gateway URL in gateway mode, from `NEXT_PUBLIC_APP_URL`, or the provider URL + publishable key in direct mode, `authProviders`).
  7. Emit `backend.status`. Frames reload data. **Never seed the remote DB**; people start empty, so empty states matter.
- The Planner names what it's deploying and that it's billed to the person's account (`05` §3.2 rule 5).

### 5. Template: remote providers (`templates/expo-app/lib/backend/`)
- `direct-<provider>.ts`: implements `BackendProvider` (the app-side interface behind `useCollection` etc.) with the provider client, mapping camelCase ↔ snake_case.
- `gateway.ts`: `fetch(apiUrl + "/collections/<name>")` with `Authorization: Bearer <user token>`, or `X-Install-Token` (issued and signed by the gateway on first launch, stored on the device).
- `useAuth`: provider auth in direct mode, with a **silent anonymous sign-in on first launch** when no user is signed in. `user` stays `null` for the screens (anonymous isn't "signed in"), while `auth.uid()` scopes the data. In gateway mode, the provider's auth client (publishable `authKey` + `authUrl` only).
- Both enforce field/type validation client-side too (same as `local.ts`, `06` §6).
- Preview runtime: bundle the provider client; CSP `connect-src` allows only this project's backend/gateway origin (`07` §6). This is a template **revision** within SDK 58 that renders from HEAD for all projects and must stay backward compatible (ADR-003 #9). It isn't a versioned upgrade.

### 6. Platform data gateway (only if `dataPath = "gateway"`) (`app/api/apps/[projectId]/[...path]/route.ts`)
- Routes: `GET/POST /collections/:name`, `PATCH/DELETE /collections/:name/:id`, `GET/PUT /values/:key`, `POST /upload` (storage), `POST /email/me`.
- Per request: look up `project_backends` (cached), resolve the caller (`verifyUserToken` with the project's auth, else a gateway-signed install token; unsigned ids are rejected), check collection exists and `shared` rules, validate fields and types (unknown field or wrong type → 400), and run parameterized SQL with the decrypted credential. It never returns credentials, connection strings or stack traces.
- On the first data request after `schema_version` changes, run `applySchema` first.
- Rate-limit per project + caller. Add CORS for the app origins (Expo web publish URL, device-preview origins, preview runtime).

### 7. Workspace UI
- Header **Connect Backend** → dialog (what will be created, billed to whom) → **Connect <Provider>**. The **Share** option lives under the Export menu.
- After connect: a **Backend connected** pill → popover with services (database ✓, auth ✓, storage …), "Open dashboard", **Disconnect**.
- Disconnect: confirm (**owner only**); `lib/backend/config.ts` goes back to local; delete the `project_backends` row (keep a log line with the external project id) and call `disconnect()`. **Never delete** the person's remote project. Emit `plan.updated` on the project stream.
- Social sign-in checklist: Google/GitHub need the person's own OAuth client configured in their backend (redirect URIs for the Expo scheme, `exp://` and the web publish URL). Until it's configured, `authProviders` hides social buttons. The video's Google button failed for exactly this reason.

## Files to create / change
- `lib/providers/backend/{types.ts,index.ts,fake.ts,<provider>/index.ts,<provider>/README.md}`
- `lib/backend-schema/{derive.ts,apply.ts}` + tests
- `lib/ai/tools/planner/setup-backend.ts`
- `worker/handlers/backend-provision.ts`
- `app/api/projects/[projectId]/backend/{route.ts,connect/route.ts}`, `app/api/backend/callback/route.ts`
- `app/api/apps/[projectId]/[...path]/route.ts` (gateway mode only)
- `templates/expo-app/lib/backend/{direct-<provider>.ts,gateway.ts}`, `config.ts.hbs`, `VERSION`
- `preview-runtime/src/vendor.ts` (provider client), CSP builder
- `components/workspace/backend/{ConnectBackendDialog.tsx,BackendPopover.tsx,SocialAuthChecklist.tsx}`

## Database changes
- `oauth_connections` rows with `provider = 'backend:<id>'` (one per person).
- `project_backends` exactly as in `03` §7: `mode` (`direct`/`gateway`), `external_project_id`, `public_config`, `secret_config_enc` (Supabase: secret key **and `db_pass`**), `services` jsonb, `schema_version`, `status` (`connecting` · `provisioning` · `ready` · `failed` · `reconnect_required`), `error`.
- `projects.plan.backend` (public fields only). `runs.kind = backend_provision`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/projects/:id/backend/connect` | owner | → 302 provider authorize URL |
| GET | `/api/backend/callback?code&state` | session + state cookie | → 302 `/workspace/:id?backend=connecting` |
| GET | `/api/projects/:id/backend` | viewer | → `{ provider, status, services[], dashboardUrl, authProviders }` (no secrets) |
| POST | `/api/projects/:id/backend/provision` | editor | → `{ runId }` (re-run, idempotent) |
| POST | `/api/projects/:id/backend/auth-providers` | owner | `{ google?: { clientId, clientSecret } }` → `{ ok }` |
| DELETE | `/api/projects/:id/backend` | owner | → `{ ok }` (disconnect, keeps remote project) |
| * | `/api/apps/:projectId/...` | app user token / install id | gateway (mode-dependent), see Step 6 |
| (Planner tool) | `setup_backend` | worker ctx | see Step 4 |

## Acceptance checklist
- [ ] Connect → provider consent → back in the workspace; provisioning progress shows in chat; the "Backend connected" pill appears.
- [ ] The remote DB has one `app_<name>` table per collection with typed columns, `owner_id`, timestamps and FKs (verified in the provider dashboard). No JSON record table except `app_values`.
- [ ] Adding a habit in the preview creates a row. Reloading shows it. A second test user can't see it; a `shared` collection is visible to both but editable only by its creator.
- [ ] **Two installs without sign-in** (two browsers or devices) can't read or modify each other's private records, in direct mode (anonymous `auth.uid()`) and in gateway mode (signed install tokens). A forged install id is rejected.
- [ ] A write with an unknown field or a wrong type is refused (400 / provider error) and the screen shows the error inline.
- [ ] Adding a field via chat (Data Designer) adds a column on the next sync. Removing a field from the plan leaves the column.
- [ ] Gateway mode deploys auth only when a screen uses `useAuth`; direct mode always deploys auth with anonymous sign-in. Storage is deployed only with `uploadFile`. Email/password sign-up and sign-in work in the preview and in Expo Go.
- [ ] Supabase adapter: only the `sb_publishable_…` key appears in materialized output; `sb_secret_…` and `db_pass` are encrypted in `secret_config_enc`.
- [ ] The model never receives provider tokens, keys or raw provider tools (unit test on the Planner tool list and tool results).
- [ ] `lib/backend/config.ts` and every materialized file contain only public config. A test asserts no secret patterns in materialized output.
- [ ] Tokens and `secret_config_enc` are encrypted (DB inspection shows `v1:` ciphertext).
- [ ] **AuthZ:** a viewer or non-member calling connect, provision, auth-providers or DELETE gets 403/404 (connect/disconnect/auth-providers are owner-only per the `08` §2 matrix). State tampering is rejected. Gateway mode: one app user can't read or modify another's private records (test).
- [ ] Disconnect reverts to local; the remote project is untouched. Re-running provision is idempotent.

## Claude Code prompt
```text
We're implementing Phase 13 (generated-app backend connect) of AppBuilding.

Read these first, in order:
- docs/plan/phases/phase-13-generated-app-backend.md (this phase)
- docs/plan/13-reference-implementation.md (R12, R13, R14)
- docs/plan/06-generated-app-template.md §2, §3, §6 (collections, @/lib/backend hooks, ownership)
- docs/plan/05-ai-agent-design.md §3.2 rule 5 and §3.3 (setup_backend)
- docs/plan/03-data-model.md §7 and §10 (oauth_connections, project_backends, derived app tables)
- docs/plan/08-security.md (token encryption, authz matrix, secrets)
- docs/plan/15-vendor-facts.md §6 and ADR-003 #19 (anonymous sign-in in direct mode)

Implement exactly this phase:
1. BackendProvider interface + `fake` provider + the adapter for BACKEND_PROVIDER
   (<FILL IN: supabase | cloudhike>), including its dataPath (direct | gateway). Before writing
   the adapter, read the provider's CURRENT OAuth + provisioning docs and summarize them in
   lib/providers/backend/<name>/README.md. Don't guess endpoints.
2. OAuth connect + callback with a state cookie; encrypted tokens in oauth_connections.
3. lib/backend-schema: deterministic additive DDL from plan.collections (app_<name> tables,
   typed columns, FKs on delete set null, app_values, RLS owner_id = auth.uid() in direct mode),
   applied in one locked transaction; unit tests.
4. setup_backend Planner tool (narrow) + idempotent backend.provision job (database always;
   auth always in direct mode with anonymous sign-in, otherwise only if useAuth is used;
   storage only if uploadFile; auto-run after builds of connected apps); public config only
   into lib/backend/config.ts. Supabase: publishable/secret keys, db_pass stored encrypted.
5. Template remote providers (direct-<provider>.ts or gateway.ts) + preview vendor/CSP.
6. Gateway routes /api/apps/[projectId]/... ONLY if dataPath = gateway (token check,
   shared/private rules, field/type validation, parameterized SQL, rate limits, CORS).
7. Connect dialog, Backend popover, disconnect, social-auth checklist.

The model must never see tokens, keys or raw provider/MCP tools. The agent writes no SQL.
Definition of done = the Acceptance checklist; tests for state verification, authz, DDL
derivation, gateway isolation between users, and "no secrets in materialized files".
Do NOT build: export/GitHub (14), native builds (15), billing gates (18). Never delete remote
projects on disconnect.

Finish with lint, typecheck, tests; summarize changes, provider-doc surprises, and manual setup
(callback URLs, developer app registration).
```

## Pitfalls & notes
- **Callback URLs per environment.** Register local, preview and prod callback URLs exactly (the video and the reference app both lost time to this).
- **Provider limits on project creation.** Free tiers often cap active projects. Surface the provider's error verbatim.
- **Token refresh.** Refresh before every provider call; on revocation, set `status = reconnect_required` and show a reconnect banner. The Planner's state says "needs connecting again".
- **Schema drift.** If the person edits tables in their dashboard, additive DDL may fail. Show the SQL error, never auto-drop.
- **Gateway cost and latency.** In gateway mode our platform serves every app request. Cache `project_backends` lookups, use a pooled connection per project with idle timeouts, and set rate limits. Plan to move this to provider-side functions when the provider offers them (the reference app's note).
- **Gateway URLs and published apps.** `apiUrl` comes from `NEXT_PUBLIC_APP_URL`, so publish (Phase 16) from a deployed platform, not localhost, if the app has a backend.
- **Preview polling.** Poll only while a run is changing screens. The reference app had a render loop that hammered the dev server.
- **Public showcase previews never use the remote backend** (Phase 17): strip `plan.backend` and materialize with the local provider.
- **Supabase SQL endpoint is Beta.** Keep every call behind `runSql()` and add a contract test in the nightly vendor smoke (Phase 19b).
