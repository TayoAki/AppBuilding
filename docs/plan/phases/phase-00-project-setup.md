# Phase 00 — Project Setup, CI and Code Review

| | |
|---|---|
| Video chapter(s) | Ch.4 "Project Setup" (09:51), Ch.5 "CodeRabbit Setup" (18:32) |
| Branch | `feat/00-project-setup` (first commit can go straight to `main`, then branch for the rest) |
| Depends on | — |
| Effort (solo dev + Claude Code) | ~1 day |
| New env vars | `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `NEXT_PUBLIC_APP_URL`, `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` (declared in `lib/env.ts`; most are optional until the phase that uses them) |

## Goal
Create the empty-but-production-shaped Next.js repo that every later phase builds on: TypeScript strict, shadcn/ui, Drizzle + Postgres, Better Auth installed (wired in Phase 02), validated env, lint/format/test tooling, CI on every PR, and an automated PR reviewer. Also add local infra (Postgres + S3-compatible storage) with Docker, so nobody needs a hosted vendor account to start.

## What we do differently from the video (and why)
- **No boilerplate generator required.** The video uses a "FastNext" CLI. We use `create-next-app` and add each piece ourselves, so we know exactly what's in the repo. Using a boilerplate is fine if you prefer it; the end state must match this doc.
- **Validated env from day one** (`lib/env.ts` with Zod). The video copies env vars around by hand and finds mistakes at runtime.
- **CI runs lint + typecheck + tests on every PR.** The video relies on CodeRabbit alone. AI review is a complement to CI, not a replacement.
- **Local Postgres + MinIO via docker-compose.** Hosted services (CloudHike, Neon, Railway, Supabase, R2…) plug in later through `DATABASE_URL` / `S3_*` only.
- **`CLAUDE.md` exists before feature work**, pointing Claude Code at `docs/plan/`, so every new session starts with the conventions.
- **Decide the id strategy now** (prefixed text ids) so Phase 02+ tables are consistent.

## Steps

### 1. Scaffold Next.js
- `npx create-next-app@latest` → TypeScript, ESLint, Tailwind, App Router, `src/` **off** (paths in `02-architecture.md` §5 are root-level `app/`, `lib/`, `components/`), import alias `@/*`.
- Pin the Node version (`.nvmrc` / `engines.node` ≥ 22) and use npm (or pnpm, but pick one and commit the lockfile).
- `tsconfig.json`: `"strict": true`, `"noUncheckedIndexedAccess": true`, `"noImplicitOverride": true`.

### 2. UI kit
- `npx shadcn@latest init` (neutral base color, CSS variables).
- Add the primitives later phases need: `button input textarea dialog dropdown-menu tabs tooltip toast/sonner avatar badge card separator scroll-area resizable skeleton switch toggle-group`.
- Add `lucide-react` (it comes with shadcn) and `next-themes` for platform dark mode.

### 3. Database tooling
- `npm i drizzle-orm postgres` and `npm i -D drizzle-kit`.
- `lib/db/index.ts` exports a single `db` (postgres-js client; `max` pool size from env; reuse it across hot reloads in dev via `globalThis`).
- `lib/db/schema.ts` stays empty except for a placeholder comment. Phase 02 adds the auth tables.
- `drizzle.config.ts`: `schema: "./lib/db/schema.ts"`, `out: "./drizzle"`, `dialect: "postgresql"`, `schemaFilter: ["public"]` (keeps the pg-boss schema from Phase 05 out of migrations).
- Scripts: `db:generate` (`drizzle-kit generate`), `db:migrate` (`drizzle-kit migrate`), `db:push` (local only), `db:studio`.

### 4. Id strategy
- `lib/ids.ts`: `newId(prefix: "usr" | "prj" | "ast" | "msg" | "run" | "rev" | "scr" | …)` → `${prefix}_${nanoid-ish 21 chars}` using `crypto.getRandomValues` (or the `nanoid` package).
- Rule: all platform tables use `text` primary keys from `newId`. The exception is Better Auth's own tables, which keep whatever id generation Better Auth configures. Configure Better Auth's `advanced.database.generateId` to use `newId("usr")` etc. if you want consistency there too.

### 5. Env validation
- `lib/env.ts` using `zod` (or `@t3-oss/env-nextjs`): split `server` and `client` (`NEXT_PUBLIC_*`) schemas. Parse once at import; on failure, throw with a readable list of missing vars.
- Variables not needed yet are `.optional()`. The phase that starts using one makes it required.
- `.env.example` lists **every** canonical variable from the plan, with comments. `.env*` (except `.env.example`) is git-ignored.

### 6. Better Auth (install only)
- `npm i better-auth`. Create `lib/auth/server.ts` and `lib/auth/client.ts` stubs with a `TODO(phase-02)`. Don't mount the route handler yet.

### 7. Lint, format, test
- ESLint (Next config) + `eslint-config-prettier`; Prettier with `prettier-plugin-tailwindcss`.
- Vitest + `@testing-library/react` + `jsdom`; a trivial `lib/ids.test.ts` so CI has something to run.
- Scripts: `lint`, `typecheck` (`tsc --noEmit`), `test`, `format`, `format:check`.
- Optional: `husky` + `lint-staged` pre-commit (format + eslint on staged files).

### 8. Local infrastructure
- `docker-compose.yml`: `postgres:16` (port 5432, volume) and `minio/minio` (ports 9000/9001) plus a `minio/mc` init container that creates the dev bucket.
- `.env.example` defaults point at these (`S3_ENDPOINT=http://localhost:9000`, `S3_REGION=us-east-1`, force path-style in the S3 client later).

### 9. GitHub repo and CI
- Push to GitHub. Protect `main`: require PR + passing CI.
- `.github/workflows/ci.yml` on `pull_request` and `push: main`: checkout → setup-node (cache npm) → `npm ci` → `lint` → `typecheck` → `test` → `next build` (with dummy env vars or `SKIP_ENV_VALIDATION=1`).
- Add Dependabot (`.github/dependabot.yml`) for npm + GitHub Actions, weekly.

### 10. AI code review (optional but recommended)
- Install the CodeRabbit GitHub App on the repo (as in the video), or any equivalent reviewer.
- `.coderabbit.yaml`: review profile `assertive`; path instructions such as "`app/api/**`: check auth and project ownership on every handler", "`lib/crypto/**`: flag any key or token logged", "`worker/**`: check idempotency and retries".
- Rule for the team: **AI review comments are advisory; CI must be green and a human must read the diff before merging.**

### 11. CLAUDE.md
Create a short root `CLAUDE.md` (template in `docs/plan/09-dev-workflow.md`):
- "Read `docs/plan/00-index.md` and the current phase doc before coding."
- Commands (`npm run dev|worker|lint|typecheck|test|db:generate|db:migrate`).
- Conventions: ids via `newId`, env via `lib/env.ts`, every project route uses `requireProjectAccess`, no new dependencies without noting why in the PR.

### 12. Hello-world page
- Replace `app/page.tsx` with a minimal page and confirm `npm run dev` shows it on `http://localhost:3000`.

## Files to create / change
```
.nvmrc
.env.example
.gitignore
.prettierrc, .prettierignore
eslint.config.mjs
vitest.config.ts
docker-compose.yml
drizzle.config.ts
tsconfig.json
package.json                (scripts listed above)
CLAUDE.md
.coderabbit.yaml
.github/workflows/ci.yml
.github/dependabot.yml
app/layout.tsx, app/page.tsx, app/globals.css
components/ui/*             (shadcn generated)
lib/env.ts
lib/ids.ts, lib/ids.test.ts
lib/db/index.ts, lib/db/schema.ts
lib/auth/server.ts, lib/auth/client.ts   (stubs)
lib/utils.ts                (shadcn cn())
```

## Database changes
None (empty schema; the connection is verified).

## API surface
None. Optionally add `GET /api/health` → `{ ok: true, db: "up" | "down" }`, running `select 1`. It's useful for deploy checks in Phase 19.

## Acceptance checklist
- [ ] `docker compose up -d` starts Postgres and MinIO; the dev bucket exists.
- [ ] `npm run dev` serves the placeholder home page.
- [ ] `npm run lint`, `npm run typecheck`, `npm test`, `npm run build` all pass locally.
- [ ] Removing a required env var makes startup fail with a clear message naming the variable.
- [ ] `GET /api/health` reports `db: "up"` against local Postgres.
- [ ] A test PR triggers CI (green) and the AI reviewer (if installed) comments.
- [ ] `main` is protected; direct pushes are blocked.
- [ ] No secrets are committed: `.env` is ignored and `.env.example` holds placeholders only.
- [ ] `CLAUDE.md` exists and links to `docs/plan/00-index.md`.

## Claude Code prompt
```text
We are starting the AppBuilding platform. Before writing code, read:
- docs/plan/00-index.md
- docs/plan/02-architecture.md (§5 repository layout, §6 environments)
- docs/plan/03-data-model.md (conventions section only)
- docs/plan/09-dev-workflow.md (CLAUDE.md template)
- docs/plan/phases/phase-00-project-setup.md (this phase)

Implement Phase 00 exactly as described in phase-00-project-setup.md:
1. Scaffold Next.js (App Router, TypeScript strict, Tailwind, no src/ dir, @/* alias).
2. Init shadcn/ui and add the listed primitives.
3. Add Drizzle + postgres-js with lib/db/index.ts, an empty lib/db/schema.ts,
   drizzle.config.ts (schemaFilter ["public"]), and the db:* scripts.
4. Add lib/ids.ts (newId(prefix)) with a unit test.
5. Add lib/env.ts with Zod server/client schemas covering every variable in
   .env.example (unused ones optional). Create .env.example with comments.
6. Install better-auth and create stub lib/auth/server.ts + client.ts (no route yet).
7. ESLint + Prettier (tailwind plugin), Vitest + Testing Library, and the
   lint/typecheck/test/format scripts.
8. docker-compose.yml with postgres:16 and MinIO + bucket init.
9. .github/workflows/ci.yml (lint, typecheck, test, build) and dependabot.yml.
10. .coderabbit.yaml with the path instructions from the phase doc.
11. Root CLAUDE.md using the template in 09-dev-workflow.md.
12. GET /api/health that runs `select 1`.

Do NOT build any product UI, auth flows, or tables yet. Those are later phases.
Don't add dependencies beyond the ones named here without explaining why.

Definition of done = the Acceptance checklist in the phase doc.
When finished, run npm run lint, npm run typecheck, npm test and npm run build,
fix anything failing, and give me a summary: files created, commands to run
locally, and anything I must do by hand (GitHub branch protection, CodeRabbit install).
```

## Pitfalls & notes
- `next build` in CI fails if `lib/env.ts` requires vars CI doesn't have. Either give CI dummy values or support `SKIP_ENV_VALIDATION=1` for the build step only.
- postgres-js in dev: without a `globalThis` cache, Next.js hot reload opens a new pool on every change and you run out of connections.
- Keep `schemaFilter: ["public"]` in drizzle config. Otherwise `drizzle-kit` will try to drop pg-boss's tables in Phase 05.
- Check that the MinIO console port (9001) and API port (9000) don't clash with other local services.
- An AI reviewer can be confidently wrong. Treat "major" findings as prompts to look, not as truth. The video also ignores many minor findings.
- Don't commit `.env` even "temporarily". The video pastes connection strings and access tokens straight into prompts and env files; rotate anything that ends up in git history.
