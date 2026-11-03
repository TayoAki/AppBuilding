# Phase 14 — Export (Zip + GitHub) and Sharing

| | |
|---|---|
| Video chapter(s) | Ch.16 "Export Project" (02:09:19) |
| Branch | `feat/14-export-and-sharing` |
| Depends on | Phase 05 (jobs), Phase 08 (VFS + snapshots + `materialize`) |
| Effort (solo dev + Claude Code) | ~4–5 days |
| New env vars | `GITHUB_APP_ID`, `GITHUB_APP_SLUG`, `GITHUB_APP_CLIENT_ID`, `GITHUB_APP_CLIENT_SECRET`, `GITHUB_APP_PRIVATE_KEY` |

## Goal
Users own their code. From the workspace **Export** menu they can (1) download the full Expo project as a zip that runs with `npm install && npx expo start`, (2) push it to a GitHub repo: a push commits to the default branch **only if the repo is empty**, and every other push goes to a new `appbuilding/update-…` branch with a pull request, so nothing already on GitHub is ever overwritten, and (3) share the project with other people by email or link with viewer/editor roles.

## What we do differently from the video (and why)
- **GitHub App instead of an OAuth App.** The video registers an OAuth App, which for private repos needs the `repo` scope: full read/write to **every** repo the user can access, with no per-repo choice. A GitHub App lets the user pick which repos (or "only repos created by AppBuilding") to grant. It issues short-lived user-to-server tokens with fine-grained permissions (`contents:write`, `pull_requests:write`, `administration:write` only if we create repos), and can be revoked per repo. It's slightly more setup, but much safer for users and for us.
- **Git Data API, no `git` binary.** Pushing goes blobs → tree → commit → ref update via the REST API. This works on serverless and in the worker without a checkout. Large binary assets are uploaded as base64 blobs.
- **Repo name validation up front.** The video failed with "repository name may only contain…" after entering a name with spaces. We slugify a suggestion (`PennyWise Expense Tracker` → `pennywise-expense-tracker`) and validate `^[A-Za-z0-9._-]{1,100}$` client-side and server-side.
- **Secret scan before every export.** Materialized output is scanned for secret-looking strings (service keys, private keys, tokens) before zipping or pushing. The push is blocked with a clear message.
- **Zips are built in the worker and stored**, with a short-lived signed URL. They're not streamed from a request handler, so big projects don't hit timeouts.

**Reference app notes** (`13-reference-implementation.md`)
- **R17:** the first push goes to the default branch only if the repository is empty. Every other push, including a first push into an existing non-empty repo, goes to a new branch. We adopt this rule as-is.
- **One exporter for zip and GitHub (and later EAS):** the reference app's `export-project.ts` feeds both. Ours is `materialize({ target: "export" })` (`06` §10), with dependency versions from the pinned `expo`'s `bundledNativeModules.json`, a static `app.json` with `web.output: "single"`, `eas.json`, and identity fields when set.
- **Sharing scope:** the reference app ships **view-only** sharing for **existing accounts** only (every mutating route stays owner-only). Our plan has viewer/editor roles plus email invites. v1 may ship the reference app's simpler version (steps 7a only, and about 1 day less). The `project_members.role` column keeps the upgrade path open.
- The reference app used a GitHub **OAuth App**. We keep the GitHub App (§3 of `13`: where we differ).

## Steps

### 1. README + materialize check
- Use `materialize({ target: "export" })` (see `06-generated-app-template.md` §10). It renders platform files from `projects.plan` (themes, schema, seed, layouts, `lib/backend/config.ts`), overlays `project_files`, and writes `package.json` with versions from `expo/bundledNativeModules.json` for the pinned SDK, a static `app.json` (`web.output: "single"`, identity from `projects.slug` / `android_package` / `bundle_id` if already set), and `eas.json`. It also renders `README.md`: app name, description from the plan, "How to run" (Node version, `npm install`, `npx expo start`, Expo Go QR), "How to build" (EAS), backend notes (which provider, where config lives, **no secrets included**), "Generated with AppBuilding".
- Add `.gitignore` (node_modules, .expo, dist, web-build, *.jks, credentials.json) to template platform-owned files. Also make sure the **platform repo's own** `.gitignore` excludes local export/build scratch output (`tmp/`, `.exports/`, `*.tgz`). The reference app accidentally committed 10K+ generated build files.
- Unit test: a materialized fixture project passes `npm install && npx tsc --noEmit` in CI (nightly job; too slow per PR).

### 2. Secret scanner
`lib/validate/secrets.ts`: regex + entropy checks (private key headers, `sk_live_`, `service_role`, `ghp_`, `github_pat_`, AWS keys, JWTs with a `service_role` claim). It runs on materialized files and returns findings with path + line. Used by zip, GitHub, and later EAS/web.

### 3. Zip export job
- `POST /api/projects/:id/export/zip` → `requireProjectAccess(editor)` (per the `08` §2 matrix) → in one transaction insert the run (`kind = export_zip`, `created_by`, `billed_user_id` = owner) and a `job_outbox` row (ADR-003 #7). The outbox dispatcher enqueues the job with payload `{runId, projectId, userId}`.
- Worker `export-zip.ts`: materialize → secret scan → stream into a zip (`archiver`/`yazl`) → upload to `exports/{projectId}/{runId}.zip` → `runs.output = { storageKey, bytes }`.
- `GET /api/runs/:runId/download` → authz → 302 to a signed URL (5 min TTL).
- UI: Export menu → "Download as zip" → progress "Packaging your project…" (run events) → auto-download.

### 4. GitHub App setup + connect
- Create the GitHub App (manual, documented in `docs/plan/09-dev-workflow.md` or this file's notes): permissions Contents RW, Pull requests RW, Metadata R, Administration RW (only needed to create repos for the user); "Request user authorization (OAuth) during installation" on; callback URL `/api/github/callback`; one app per environment (dev/prod) to keep callback URLs clean.
- `GET /api/github/connect?projectId=` → redirect to the app's installation/authorize URL with signed `state`.
- `GET /api/github/callback` → exchange code for a user-to-server token (and refresh token if expiring tokens are enabled) → `oauth_connections` (`provider = "github"`, encrypted) → redirect back with `?github=connected`.

### 5. Repo picker / create
- `GET /api/github/repos` → list repos the installation can access (paginate, search).
- `POST /api/github/repos` `{ name, private, description }` → validate the name → create the repo under the user's account with **`auto_init: true`** → returns `{ owner, repo, defaultBranch }`.
- Dialog: "Create new repo" (prefilled slug, private toggle, description "Built with AppBuilding") or "Use existing repo".

### 6. Push job `export.github`
Worker `export-github.ts`:
1. Materialize the current snapshot (create one if needed); secret scan.
2. Check whether the repo is **empty** (no commits / no default-branch ref). If it is, write the **first commit with the Contents API** (`PUT /repos/{owner}/{repo}/contents/{path}` for one small file, e.g. `README.md`). The Git Data API returns `409 Git Repository is empty` on repos with no commits (ADR-003 #16). Then add the remaining files with the Git Data API (blobs → tree with `base_tree` = that commit's tree → commit → update `refs/heads/<default>`). This is the only case that writes to the default branch. When **we** create the repo (step 5), create it with `auto_init: true` instead and treat it as empty: the only commit is the auto README, which our first push replaces on the default branch.
3. Otherwise (a later push, *or* a first push into an existing repo that already has commits): diff against `last_pushed_snapshot_id` if set (skip unchanged blobs), create branch `appbuilding/update-<yyyyMMdd-HHmmss>` from the default head, commit with a meaningful message ("Add Settings screen; restyle HabitCard", built from the run summaries since the last push), then open a PR with a body listing changed screens/components/files.
4. Save `last_pushed_snapshot_id`, `last_commit_sha` (for update pushes, the branch commit; merge state is the user's business).
5. `runs.output = { commitUrl, prUrl? }`.
- UI: "Commit & push" → progress → success state with "Open repo" / "Open pull request" buttons.
- Handle a non-fast-forward or deleted repo: re-fetch the head and retry once; if the repo is gone, clear `project_github_repos` and ask the user to pick again.

### 7. Sharing
**7a (v1 minimum, reference-app parity):** view-only sharing with existing accounts. The owner enters an email, it resolves to an existing user (otherwise "No AppBuilding account with this email"), and a `project_members` row with `role = viewer` is created. All mutating routes stay owner-only.

**7b (full plan):**
- Share dialog (from the Export menu, matching the video's placement): email input + role select (viewer/editor) → `POST /api/projects/:id/members`.
- If the email belongs to an existing user → immediate access; otherwise create a pending `project_members` row and send an invite email with a signed accept link (`/invite/:token`). Accepting while signed in as that email binds `user_id` and sets `accepted_at`.
- "Copy project link" → `/workspace/:id`. Access still requires membership, unless the project is public/unlisted (Phase 17 adds a read-only public page).
- Member list with role change/remove (owner only).
- Update `requireProjectAccess()` to consult `project_members` (viewer < editor < owner).

## Files to create / change
- `lib/vfs/materialize.ts` (README, .gitignore), `templates/expo-app/README.md.hbs`, `templates/expo-app/.gitignore`
- `lib/validate/secrets.ts`
- `lib/providers/github/{app.ts,client.ts,gitdata.ts}` (JWT for app auth via `GITHUB_APP_PRIVATE_KEY`, user-token client, tree/commit helpers)
- `worker/handlers/{export-zip.ts,export-github.ts}`
- `app/api/projects/[projectId]/export/{zip,github}/route.ts`, `app/api/runs/[runId]/download/route.ts`
- `app/api/github/{connect,callback,repos}/route.ts`
- `app/api/projects/[projectId]/members/route.ts`, `app/invite/[token]/page.tsx`
- `lib/auth/require.ts` (`requireProjectAccess` roles)
- `components/workspace/export/{ExportMenu.tsx,GithubPushDialog.tsx,ShareDialog.tsx}`
- Email sending helper `lib/email/send.ts` (provider of choice; console transport in dev)

## Database changes
- `oauth_connections`: rows with `provider = 'github'`.
- `project_github_repos`: `project_id`, `owner`, `repo`, `default_branch`, `last_pushed_snapshot_id`, `last_commit_sha`.
- `project_members`: `project_id`, `user_id` (null until accepted), `email`, `role`, `invited_by`, `accepted_at`; unique (project_id, email). Add `invite_token_hash` + `invite_expires_at`.
- `project_snapshots` used as the export source.
- New `runs.kind` values in use: `export_zip`, `export_github`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/export/zip` | editor | → `{ runId }` |
| GET | `/api/runs/:runId/download` | member of run's project | → 302 signed URL |
| GET | `/api/github/connect?projectId=` | session | → 302 GitHub |
| GET | `/api/github/callback` | session + state | → 302 workspace |
| GET | `/api/github/status` | session | → `{ connected, login }` |
| GET | `/api/github/repos?q=&page=` | session + github | → `{ repos: [{owner, name, private, defaultBranch}] }` |
| POST | `/api/github/repos` | session + github | `{ name, private, description? }` → `{ owner, repo, defaultBranch }` |
| POST | `/api/projects/:id/export/github` | editor | `{ owner?, repo? }` → `{ runId }` |
| GET/POST/PATCH/DELETE | `/api/projects/:id/members` | owner (viewer may GET) | `{ email, role }` → member list |
| POST | `/api/invites/:token/accept` | session (email must match) | → `{ projectId }` |

## Acceptance checklist
- [ ] Downloaded zip unpacks to a project that runs with `npm install && npx expo start` and opens in Expo Go.
- [ ] Zip contains README with run instructions; contains no `*.jks`, no secrets (scanner test with a planted fake key blocks export).
- [ ] Connect GitHub → create repo with an invalid name shows inline validation; valid name creates the repo.
- [ ] Pushing to a new empty repo (created outside AppBuilding with no commits, **and** one created by AppBuilding with `auto_init`) lands on the default branch with all files, with no 409 from the Git Data API. A second push after a theme change creates `appbuilding/update-*` and a PR with only changed files. A first push into an existing non-empty repo also goes to a new branch (the default branch is untouched).
- [ ] GitHub tokens are stored encrypted; disconnecting GitHub deletes them.
- [ ] Share by email: existing user gets access immediately; a new email gets an invite that works only for that email; expired tokens are rejected.
- [ ] Viewer can open the workspace read-only (chat input, inline edit, zip export and GitHub push disabled, per the `08` §2 matrix); non-member gets 404 on every project route.
- [ ] Every export run records `created_by` (acting user) and `billed_user_id` (owner).
- [ ] Download URLs expire and are only issued to project members.

## Claude Code prompt
```text
We're implementing Phase 14 (export: zip + GitHub, and project sharing) of AppBuilding.

Read first:
- docs/plan/phases/phase-14-export-and-sharing.md (scope + acceptance checklist)
- docs/plan/06-generated-app-template.md §10 (materialize)
- docs/plan/03-data-model.md §2 (project_members), §5 (snapshots), §7 (oauth_connections, project_github_repos)
- docs/plan/08-security.md (§2 role matrix is authoritative, token encryption, secrets)
- docs/plan/12-decisions-and-open-questions.md ADR-003 (#3 actors, #7 outbox, #16 empty repos)

Implement:
1. README/.gitignore in materialize + lib/validate/secrets.ts scanner (block exports on findings).
2. export_zip worker job → storage → signed download URL route.
3. GitHub App integration (NOT an OAuth App): app JWT + user-to-server token, connect/callback,
   repo list + create with name validation (^[A-Za-z0-9._-]{1,100}$, slugified suggestion).
4. export_github worker job via the Git Data API: commit to the default branch ONLY if the repo is
   empty (root commit); every other push (incl. first push into a non-empty repo) goes to branch
   appbuilding/update-<timestamp> + a PR with a meaningful message; only changed blobs; store last_commit_sha.
   Empty repos: first commit via the Contents API (Git Data API 409s on repos with no commits); repos we
   create use auto_init. Use materialize({target:"export"}) (versions from bundledNativeModules.json,
   static app.json with web.output "single").
   Jobs are enqueued via the job_outbox in the same transaction as the run row (ADR-003 #7).
5. Sharing: start with 7a (view-only, existing accounts, mutations owner-only); then 7b if in scope:
   viewer/editor roles, invite email with hashed token, accept flow. requireProjectAccess role checks
   on every /api/projects/* route; viewer = read-only UI.

Verify current GitHub App docs (user-to-server tokens, token expiry/refresh, repo creation permission)
before coding; note anything that differs from the phase doc.

Definition of done = the Acceptance checklist. Write tests for: repo name validation, secret scanner,
diffing changed files between snapshots, invite token accept (wrong email / expired), and authz on
every new route (non-member, viewer, editor, owner).

Do NOT build: native builds (Phase 15), web publish (Phase 16), public showcase (Phase 17).
Finish by running lint, typecheck, tests; summarize changes and the manual GitHub App setup steps I need to do.
```

## Pitfalls & notes
- **GitHub App vs repo creation.** Creating a repo for a *user* account through an App requires the Administration permission and the user's authorization. If that's awkward, fall back to "create the repo on GitHub, then select it": the picker still works. Decide during implementation after reading current docs.
- **Empty repos.** The Git Data API refuses blobs/trees/commits on a repo with **no commits** (409 "Git Repository is empty"). Seed the first commit through the Contents API, or create repos with `auto_init: true`. A repo whose only commit is GitHub's auto-generated README counts as "empty" for the default-branch rule.
- **Rate limits.** One blob per file is fine for ~100 files. Skip unchanged blobs by sha, and batch with modest concurrency (5).
- **Binary assets.** Upload as base64 blobs; keep images reasonably sized (Phase 12 uses URLs, so few binaries are expected).
- **The callback URL must match exactly.** The video's push failed because of a stray `.` in the registered redirect URI. Copy it from `NEXT_PUBLIC_APP_URL` + path, and register separate dev/prod apps.
- **Never auto-merge** the update PR. That's the user's call (the video's behavior, and the right one).
- **Viewer export.** Viewers can't export or push (`08` §2 matrix). Make it an owner toggle later if needed.
