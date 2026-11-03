# Phase 14 — Export (Zip + GitHub) and Sharing

| | |
|---|---|
| Video chapter(s) | Ch.16 "Export Project" (02:09:19) |
| Branch | `feat/14-export-and-sharing` |
| Depends on | Phase 05 (jobs), Phase 08 (VFS + snapshots + `materialize`) |
| Effort (solo dev + Claude Code) | ~4–5 days |
| New env vars | `GITHUB_APP_ID`, `GITHUB_APP_SLUG`, `GITHUB_APP_CLIENT_ID`, `GITHUB_APP_CLIENT_SECRET`, `GITHUB_APP_PRIVATE_KEY` |

## Goal
Users own their code. From the workspace **Export** menu they can (1) download the full Expo project as a zip that runs with `npm install && npx expo start`, (2) push it to a GitHub repo: the first push creates the repo and commits to `main`, later pushes open a pull request, and (3) share the project with other people by email or link with viewer/editor roles.

## What we do differently from the video (and why)
- **GitHub App instead of an OAuth App.** The video registers an OAuth App, which for private repos needs the `repo` scope: full read/write to **every** repo the user can access, with no per-repo choice. A GitHub App lets the user pick which repos (or "only repos created by AppBuilding") to grant. It issues short-lived user-to-server tokens with fine-grained permissions (`contents:write`, `pull_requests:write`, `administration:write` only if we create repos), and can be revoked per repo. It's slightly more setup, but much safer for users and for us.
- **Git Data API, no `git` binary.** Pushing goes blobs → tree → commit → ref update via the REST API. This works on serverless and in the worker without a checkout. Large binary assets are uploaded as base64 blobs.
- **Repo name validation up front.** The video failed with "repository name may only contain…" after entering a name with spaces. We slugify a suggestion (`PennyWise Expense Tracker` → `pennywise-expense-tracker`) and validate `^[A-Za-z0-9._-]{1,100}$` client-side and server-side.
- **Secret scan before every export.** Materialized output is scanned for secret-looking strings (service keys, private keys, tokens) before zipping or pushing. The push is blocked with a clear message.
- **Zips are built in the worker and stored**, with a short-lived signed URL. They're not streamed from a request handler, so big projects don't hit timeouts.

## Steps

### 1. README + materialize check
- Extend `materialize()` (see `06-generated-app-template.md` §10) to render `README.md`: app name, description from the plan, "How to run" (Node version, `npm install`, `npx expo start`, Expo Go QR), "How to build" (EAS), backend notes (which provider, where config lives, **no secrets included**), "Generated with AppBuilding".
- Add `.gitignore` (node_modules, .expo, dist, *.jks, credentials.json) to template platform-owned files.
- Unit test: a materialized fixture project passes `npm install && npx tsc --noEmit` in CI (nightly job; too slow per PR).

### 2. Secret scanner
`lib/validate/secrets.ts`: regex + entropy checks (private key headers, `sk_live_`, `service_role`, `ghp_`, `github_pat_`, AWS keys, JWTs with a `service_role` claim). It runs on materialized files and returns findings with path + line. Used by zip, GitHub, and later EAS/web.

### 3. Zip export job
- `POST /api/projects/:id/export/zip` → `requireProjectAccess(viewer)` (owner can decide later whether viewers may export; default **editor**) → run `export_zip` → job.
- Worker `export-zip.ts`: materialize → secret scan → stream into a zip (`archiver`/`yazl`) → upload to `exports/{projectId}/{runId}.zip` → `runs.output = { storageKey, bytes }`.
- `GET /api/runs/:runId/download` → authz → 302 to a signed URL (5 min TTL).
- UI: Export menu → "Download as zip" → progress "Packaging your project…" (run events) → auto-download.

### 4. GitHub App setup + connect
- Create the GitHub App (manual, documented in `docs/plan/09-dev-workflow.md` or this file's notes): permissions Contents RW, Pull requests RW, Metadata R, Administration RW (only needed to create repos for the user); "Request user authorization (OAuth) during installation" on; callback URL `/api/github/callback`; one app per environment (dev/prod) to keep callback URLs clean.
- `GET /api/github/connect?projectId=` → redirect to the app's installation/authorize URL with signed `state`.
- `GET /api/github/callback` → exchange code for a user-to-server token (and refresh token if expiring tokens are enabled) → `oauth_connections` (`provider = "github"`, encrypted) → redirect back with `?github=connected`.

### 5. Repo picker / create
- `GET /api/github/repos` → list repos the installation can access (paginate, search).
- `POST /api/github/repos` `{ name, private, description }` → validate the name → create the repo under the user's account → returns `{ owner, repo, defaultBranch }`.
- Dialog: "Create new repo" (prefilled slug, private toggle, description "Built with AppBuilding") or "Use existing repo".

### 6. Push job `export.github`
Worker `export-github.ts`:
1. Materialize the current snapshot (create one if needed); secret scan.
2. If `project_github_repos.last_commit_sha` is null → **first push**: create blobs for each file, a tree, a commit on top of the default branch head (or root commit for an empty repo), update `refs/heads/<default>`.
3. Otherwise → **update push**: diff against `last_pushed_snapshot_id` (skip unchanged blobs), create branch `appbuilding/update-<yyyyMMdd-HHmmss>` from the default head, commit, then open a PR titled "AppBuilding update: <summary of runs since last push>", with a body listing changed screens/files.
4. Save `last_pushed_snapshot_id`, `last_commit_sha` (for update pushes, the branch commit; merge state is the user's business).
5. `runs.output = { commitUrl, prUrl? }`.
- UI: "Commit & push" → progress → success state with "Open repo" / "Open pull request" buttons.
- Handle a non-fast-forward or deleted repo: re-fetch the head and retry once; if the repo is gone, clear `project_github_repos` and ask the user to pick again.

### 7. Sharing
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
- [ ] First push lands on `main` with all files; a second push after a theme change creates `appbuilding/update-*` and a PR with only changed files.
- [ ] GitHub tokens are stored encrypted; disconnecting GitHub deletes them.
- [ ] Share by email: existing user gets access immediately; a new email gets an invite that works only for that email; expired tokens are rejected.
- [ ] Viewer can open the workspace read-only (chat input, inline edit, export-to-GitHub disabled); non-member gets 404/403 on every project route.
- [ ] Download URLs expire and are only issued to project members.

## Claude Code prompt
```text
We're implementing Phase 14 (export: zip + GitHub, and project sharing) of AppBuilding.

Read first:
- docs/plan/phases/phase-14-export-and-sharing.md (scope + acceptance checklist)
- docs/plan/06-generated-app-template.md §10 (materialize)
- docs/plan/03-data-model.md §2 (project_members), §5 (snapshots), §7 (oauth_connections, project_github_repos)
- docs/plan/08-security.md (authz roles, token encryption, secrets)

Implement:
1. README/.gitignore in materialize + lib/validate/secrets.ts scanner (block exports on findings).
2. export_zip worker job → storage → signed download URL route.
3. GitHub App integration (NOT an OAuth App): app JWT + user-to-server token, connect/callback,
   repo list + create with name validation (^[A-Za-z0-9._-]{1,100}$, slugified suggestion).
4. export_github worker job via the Git Data API: first push to default branch; later pushes to
   branch appbuilding/update-<timestamp> + open a PR; only changed blobs; store last_commit_sha.
5. Sharing: project_members with viewer/editor, invite email with hashed token, accept flow,
   requireProjectAccess role checks applied to every /api/projects/* route; viewer = read-only UI.

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
- **Empty repos.** A brand-new repo with no commits has no ref to base on. Create a root commit (no parents) and then create `refs/heads/main`. Creating with `auto_init` avoids this but adds an unwanted README commit.
- **Rate limits.** One blob per file is fine for ~100 files. Skip unchanged blobs by sha, and batch with modest concurrency (5).
- **Binary assets.** Upload as base64 blobs; keep images reasonably sized (Phase 12 uses URLs, so few binaries are expected).
- **The callback URL must match exactly.** The video's push failed because of a stray `.` in the registered redirect URI. Copy it from `NEXT_PUBLIC_APP_URL` + path, and register separate dev/prod apps.
- **Never auto-merge** the update PR. That's the user's call (the video's behavior, and the right one).
- **Viewer export.** Default: viewers can't export or push. Make it an owner toggle later if needed.
