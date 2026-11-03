# Phase 02 — Authentication and Database

| | |
|---|---|
| Video chapter(s) | Ch.7 "Auth & Database Setup" (29:17) |
| Branch | `feat/02-auth-db` |
| Depends on | Phase 00, Phase 01 |
| Effort (solo dev + Claude Code) | ~1.5–2 days |
| New env vars | `DATABASE_URL` (now required), `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `NEXT_PUBLIC_APP_URL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GITHUB_LOGIN_CLIENT_ID`, `GITHUB_LOGIN_CLIENT_SECRET` |

## Goal
Users can sign up and sign in with email/password, Google or GitHub. Their user row carries `plan` and `credits_balance`, and a `signup_grant` ledger entry records their starting credits. Submitting a prompt while signed out opens the auth dialog, **remembers the prompt (and attachments)**, and continues after sign-in. Authorization helpers (`requireUser`, `requireProjectAccess`) are created now, so every later route uses them from its first line.

## What we do differently from the video (and why)
- **Self-hosted Better Auth on our own Postgres, against any provider.** The video swaps Better Auth for CloudHike's hosted auth service. We keep auth in our app (one less vendor dependency), and the DB is just a `DATABASE_URL`: local Docker, Neon, Railway, Supabase or CloudHike Postgres all work.
- **Credits ledger from day one.** The video sets `credits = 100` on the user row. We add `credit_ledger` now with a `signup_grant` row and keep `user.credits_balance` as a cache updated in the same transaction (see `03-data-model.md` §8). Phase 18 adds charging.
- **Migrations, not just `db push`.** `drizzle-kit generate` + `migrate` produce reviewable SQL in `drizzle/`. `db:push` is allowed only against a local DB.
- **Pending prompt survives sign-in**, including OAuth redirects (the video re-asks after sign-in).
- **Authorization helpers before any project routes exist**, so IDOR bugs can't creep in later.

## Steps

### 1. Better Auth server
`lib/auth/server.ts`:
- `betterAuth({ database: drizzleAdapter(db, { provider: "pg" }), emailAndPassword: { enabled: true, minPasswordLength: 8 }, socialProviders: { google: {...}, github: { clientId: env.GITHUB_LOGIN_CLIENT_ID, clientSecret: env.GITHUB_LOGIN_CLIENT_SECRET } }, user: { additionalFields: { plan, creditsBalance, onboardedAt } } })`.
- Use `BETTER_AUTH_SECRET` and `BETTER_AUTH_URL`. Set `trustedOrigins` to `NEXT_PUBLIC_APP_URL` (plus a tunnel URL in dev if you test OAuth through one).
- Session cookie: `httpOnly`, `secure` in production, `sameSite: "lax"`.
- `databaseHooks.user.create.after`: insert the `credit_ledger` `signup_grant` row and set `credits_balance` (same transaction if the adapter allows; otherwise an idempotent upsert keyed on `(user_id, reason='signup_grant')`).

### 2. Generate the auth schema
- Run the Better Auth CLI to generate the Drizzle schema for `user`, `session`, `account`, `verification` into `lib/db/schema/auth.ts`, then add the extension columns.
- Split the schema into files: `lib/db/schema/{auth.ts,credits.ts,index.ts}`, re-exported from `lib/db/schema.ts`.

### 3. Credit ledger table
`lib/db/schema/credits.ts`: `credit_ledger` exactly as in `03-data-model.md` §8 (`amount` bigint micro-credits, `reason`, `run_id` null, `stripe_event_id` unique where not null). `lib/credits/ledger.ts`: `grant(userId, amount, reason)` that inserts the ledger row and increments `user.credits_balance` in one transaction. The signup grant amount comes from plan config in `lib/credits/plans.ts` (`FREE_SIGNUP_GRANT`). Its value is set in `11-costs-and-pricing.md`.

### 4. Migrations
- `npm run db:generate` → review the SQL in `drizzle/` → `npm run db:migrate`.
- Add `db:migrate` to the deploy process later (Phase 19). Document in `README` that `db:push` is local-only.

### 5. Route handler and client
- `app/api/auth/[...all]/route.ts` → `toNextJsHandler(auth)`.
- `lib/auth/client.ts` → `createAuthClient()` exporting `signIn`, `signUp`, `signOut`, `useSession`.

### 6. Wire the dialog (from Phase 01)
- Email/password forms with Zod validation and inline errors (wrong password, email taken, weak password).
- Google / GitHub buttons call `signIn.social({ provider, callbackURL })`. `callbackURL` = current path plus `?resume=1`.
- Header: when signed in, show an avatar menu (name, credits pill, "My projects", "Sign out") instead of the buttons.

### 7. Pending prompt across sign-in
- `lib/pending-prompt.ts`: on a signed-out submit, save `{prompt, platforms}` to `sessionStorage`. Save attachments to **IndexedDB** (Files survive an OAuth redirect there; they don't in memory). Then open the dialog.
- After a successful sign-in (in-dialog), or on load with `?resume=1` (OAuth return), restore the prompt box state and show "Ready, press Create to continue". Auto-submit is optional; leave it off by default so the user confirms.
- Clear the pending state after a successful submit or after 1 hour.

### 8. Authorization helpers
`lib/auth/require.ts`:
```ts
export async function getSessionUser(): Promise<User | null>;
export async function requireUser(): Promise<User>;             // throws 401 AppError
export async function requireProjectAccess(
  projectId: string, need: "viewer" | "editor" | "owner"
): Promise<{ user: User; project: Project; role: "viewer" | "editor" | "owner" }>;
```
- `requireProjectAccess` is defined now (owner check only). The `projects` table arrives in Phase 03, and `project_members` roles in Phase 14. Until Phase 03, put a typed stub with a `TODO` and a unit test that it rejects non-owners.
- It returns 404 (not 403) for projects the user can't see, so IDs can't be enumerated.
- `lib/errors.ts`: `AppError(code, status, message)` + `toResponse(err)` used by every route handler.

### 9. Protect future routes
`middleware.ts` (or per-route checks): redirect `/workspace/*` to `/` with the sign-in dialog open when there's no session cookie. This is only a UX convenience; **real checks happen in handlers** via `requireUser` / `requireProjectAccess`.

### 10. Tests
Unit-test `grant()` (balance equals the sum of the ledger) and `requireProjectAccess` (owner OK, other user 404). Add an integration test hitting `/api/auth/sign-up/email` against the docker Postgres in CI (add a Postgres service to `ci.yml`).

## Files to create / change
```
lib/auth/server.ts, lib/auth/client.ts, lib/auth/require.ts
lib/db/schema.ts → lib/db/schema/{index.ts,auth.ts,credits.ts}
lib/credits/ledger.ts, lib/credits/plans.ts
lib/errors.ts
lib/pending-prompt.ts
app/api/auth/[...all]/route.ts
middleware.ts
components/auth/sign-in-dialog.tsx      (wired)
components/home/site-header.tsx         (user menu + credits pill)
components/home/prompt-box.tsx          (auth gate + resume)
drizzle/0001_auth.sql (generated)
.github/workflows/ci.yml                (postgres service for integration tests)
```

## Database changes
- Better Auth tables: `user`, `session`, `account`, `verification`.
- `user` extended with `plan text default 'free'`, `credits_balance bigint default 0`, `onboarded_at timestamptz null`.
- New `credit_ledger` (`id`, `user_id`, `amount`, `reason`, `run_id`, `stripe_event_id` unique-where-not-null, `created_at`) + index `(user_id, created_at)`.

## API surface
| Method | Path | Auth | Notes |
|---|---|---|---|
| * | `/api/auth/*` | public | Better Auth (sign-up/in/out, OAuth callbacks, session) |
| GET | `/api/me` | user | `{ id, name, email, image, plan, creditsBalance }` (credits formatted from micro-credits on the client) |

## Acceptance checklist
- [ ] Sign up with email/password creates `user` + `account` rows, a `credit_ledger` `signup_grant` row, and `credits_balance` equal to the grant.
- [ ] Sign in works with email/password, Google and GitHub (dev OAuth apps with `http://localhost:3000/api/auth/callback/{google,github}` redirect URIs).
- [ ] Wrong password / existing email / weak password show inline errors without leaking which emails exist beyond the standard sign-up conflict.
- [ ] Signed-out submit → dialog opens → after sign-in (including OAuth round-trip) the prompt text, platforms **and attachments** are restored.
- [ ] Header shows the avatar menu with credits when signed in; sign out works.
- [ ] `/workspace/anything` while signed out redirects home with the dialog open.
- [ ] `requireProjectAccess` unit test: a different user gets 404.
- [ ] Migrations are committed under `drizzle/`, and `npm run db:migrate` on a fresh DB reproduces the schema.
- [ ] Session cookie is `HttpOnly`, and `Secure` in production builds.

## Claude Code prompt
```text
Read first:
- docs/plan/03-data-model.md (§1 Auth, §8 credit_ledger, conventions)
- docs/plan/08-security.md (auth + authorization sections)
- docs/plan/phases/phase-02-auth-and-database.md (this phase)
- CLAUDE.md

Implement Phase 02:
1. Better Auth with the Drizzle adapter (pg): email+password, Google, GitHub
   (env GITHUB_LOGIN_CLIENT_ID/SECRET), user additionalFields plan/creditsBalance/onboardedAt.
   Mount app/api/auth/[...all]/route.ts. Create lib/auth/client.ts.
2. Generate auth schema into lib/db/schema/auth.ts; add lib/db/schema/credits.ts
   with credit_ledger exactly per 03-data-model.md; re-export from lib/db/schema.ts.
3. lib/credits/ledger.ts grant() — ledger insert + balance update in one transaction;
   call it from a Better Auth user.create.after hook with reason "signup_grant",
   amount from lib/credits/plans.ts FREE_SIGNUP_GRANT. Make it idempotent.
4. Use drizzle-kit generate + migrate (commit the SQL). db:push is local-only.
5. Wire the SignInDialog from Phase 01 (validation, inline errors, social buttons).
6. lib/pending-prompt.ts: persist prompt+platforms in sessionStorage and attachments
   in IndexedDB on signed-out submit; restore after sign-in or ?resume=1.
7. lib/auth/require.ts: getSessionUser, requireUser, requireProjectAccess (404 for
   no access; projects table comes in Phase 03, so stub the lookup behind an interface
   and unit-test the rule). lib/errors.ts AppError + toResponse.
8. middleware.ts redirect for /workspace/* when there is no session (UX only).
9. GET /api/me. Header avatar menu with credits pill.
10. Tests listed in step 10; add a Postgres service to CI.

Do NOT create projects, uploads, workspace pages or billing. No Stripe.
Definition of done = Acceptance checklist in the phase doc.
Finish with npm run lint, typecheck, test; fix failures; summarize, including the
exact OAuth redirect URIs I need to register with Google and GitHub.
```

## Pitfalls & notes
- The video hit "sign-in link expired" through ngrok because `NEXT_PUBLIC_APP_URL` / the auth base URL still said `localhost`. `BETTER_AUTH_URL`, `NEXT_PUBLIC_APP_URL` and every OAuth redirect URI must agree on the exact origin you're browsing.
- Use **separate** GitHub OAuth apps for *platform sign-in* (`GITHUB_LOGIN_*`, this phase) and for *pushing generated code* (a GitHub App, Phase 14). Repo-write scopes must never be requested at login.
- Google OAuth in "testing" mode only allows listed test users. Add yourself, or the callback fails with a vague error.
- Don't store the attachment `File`s in `localStorage` (the size limit and base64 bloat get in the way). IndexedDB stores Blobs natively.
- Credits are integers (micro-credits). Never `float` or `numeric` money math in JS.
- If you later move to a hosted auth service (CloudHike/Supabase auth), keep the `requireUser()` interface so no route changes.
