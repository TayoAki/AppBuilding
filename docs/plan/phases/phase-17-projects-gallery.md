# Phase 17 — Projects Gallery (My Projects + Public Showcase)

| | |
|---|---|
| Video chapter(s) | Ch.19 "Display User Projects" (02:34:53) |
| Branch | `feat/17-projects-gallery` |
| Depends on | Phase 01 (home layout placeholder), Phase 07 (preview runtime + harness), Phase 08 (snapshots), Phase 14 (sharing, zip export) |
| Effort (solo dev + Claude Code) | ~3–4 days |
| New env vars | none |

## Goal
Replace the home-page placeholder grid with real data. Signed-in users see **their projects** (owned and shared), each with a thumbnail, title, status, "edited 2 h ago", and a menu of actions. Signed-out visitors see a **public showcase** of projects that owners marked public. Each public project has a read-only page with a live, interactive preview.

## What we do differently from the video (and why)
- **Static thumbnails, not live iframes, in the grid.** The video renders live previews in the cards. Twenty iframes, each booting react-native-web, make the home page slow and memory-hungry. A worker renders a PNG thumbnail after every successful run, using the preview harness (`07-preview-runtime.md` §8). The live preview loads only on hover (desktop, optional) or on the project page.
- **Public previews never touch the real backend.** We keep the video's decision: the public preview endpoint **strips `plan.backend`** before materializing, so the app runs on the local provider with seed data even if the project has a connected backend. Visitor actions stay in their own browser (preview storage namespaced `demo:<projectId>`, separate from the workspace's data). No visitor can read or write the owner's production data.
- **Cursor pagination + indexes** from `03-data-model.md` §9 instead of loading everything.
- **Duplicate = copy at a snapshot**, a cheap DB copy of revisions. No LLM call, no credits.

**Reference app notes** (`13-reference-implementation.md`, R20)
- **Live cards (alternative to thumbnails):** the reference app stores no screenshots. Each card runs the app itself in the sandboxed preview frame, non-interactive, loaded when it **nears the viewport** and stopped once scrolled away, so it always shows the latest screens. That's acceptable if you'd rather skip the thumbnail job. Use `IntersectionObserver` with a cap of about 6 live frames at once, and keep the static placeholder as the fallback. Our default stays static thumbnails, for home-page performance (`13` §3).
- **Visitor list rules:** only `public` projects whose status is **`ready`** (a project mid-build waits until it's done), most recently edited first. The page size is 24 in the reference app. Cards show the **author's display name only, never an email**.
- **Duplicate and delete wait for the agent:** both are refused (409, "Wait for the current build to finish") while a run is `queued`/`running` for the project. A copy gets the chat, screens and files, but starts private and without a backend.
- **Delete is honest about what it doesn't remove:** it deletes the project's rows and uploaded files, but **not** its published website, GitHub repo, or the backend in the user's provider account. The confirm dialog says so and links to each.
- Visibility is owner-only, from the share dialog, and only affects the list and the preview route. It never opens the workspace to anyone.

## Steps

### 1. Thumbnail job `thumbnail.render`
- Enqueued at `run.done` for build/edit runs and on theme switch (debounced 30 s per project).
- Worker: Playwright loads `/preview-runtime/harness.html` with the project's files (local adapter + seed data), renders the first tab route at 390×844, takes a screenshot, writes `thumbnails/{projectId}/{snapshotId}.png` (plus a small WebP), and updates `projects.thumbnail_key`.
- Projects without a thumbnail fall back to a generated placeholder (app name initials on the theme's primary color).

### 2. Queries
`lib/db/queries/projects.ts`:
- `listMyProjects(userId, cursor)`: owned + accepted memberships, not deleted, ordered by `updated_at desc`, page size 24. Returns `{id, title, status, visibility, thumbnailUrl, updatedAt, role}`.
- `listPublicProjects(cursor)`: `visibility = 'public'` **and `status = 'ready'`**, not deleted, `updated_at desc` (later: "featured" flag / likes). Join the owner's display name; never select email.
- Bump `projects.updated_at` on every run completion and file write (in `lib/vfs`), so "last edited" is accurate.

### 3. Home page sections
- Signed in: "My projects" grid + "Shared with me" filter chip; empty state ("Describe your first app above").
- Signed out: "Built with AppBuilding" showcase grid; card click → public project page.
- Card: thumbnail (phone-shaped crop), title, status pill (Planning / Building / Ready / Error), relative time ("edited 12 h ago"), role badge if shared.
- Infinite scroll or a "Load more" button with the cursor.

### 4. Card actions menu
| Action | Who | Behaviour |
|---|---|---|
| Open | member | `/workspace/:id` |
| Open in new tab | member | same, `target=_blank` |
| Rename | editor+ | inline dialog → `PATCH /api/projects/:id { title }` |
| Duplicate | owner (or editor) | `POST /api/projects/:id/duplicate` → **409 while a run is active**; otherwise a new project owned by the caller: chat, `projects.plan` (minus `backend`), screens and files copied from the latest snapshot, `visibility = private`, no backend/GitHub/builds/web deployment/members copied |
| Share | owner | opens the Phase 14 share dialog |
| Make public / unlisted / private | owner | `PATCH visibility`; public shows a confirmation that code + preview become visible to everyone |
| Export as zip | editor+ | Phase 14 zip job |
| Delete | owner | **409 while a run is active**; confirm dialog lists what is *not* removed (published website, GitHub repo, backend project) → soft delete (`deleted_at`); hidden everywhere; hard-delete job after 30 days removes rows and uploaded files |

Duplicate and delete count against plan limits (Phase 18), checked server-side.

### 5. Public project page `/p/:projectId`
- Only for `public` or `unlisted` projects (unlisted: reachable by link, not listed).
- Shows: title, author display name, description from the plan, screens strip (thumbnails per screen if available), one **interactive preview** iframe in `mode: "interactive"` with `plan.backend` stripped (local provider + seed data, storage `demo:<projectId>`), "Remix this app" (= duplicate into the visitor's account; requires sign-in), and an optional QR code for Expo Go (Phase 11 Snack, read-only session).
- Public code view (read-only file tree), as the video allows. Run the secret scanner (Phase 14) before making a project public, and block if it finds anything.
- SEO: `generateMetadata` with title/description and the thumbnail as the OG image.

### 6. Authz
- All mutations go through `requireProjectAccess(projectId, role)`.
- Public endpoints return only whitelisted fields; never `secret_config_enc`, connections, member emails, or build artifact URLs.
- The public page loads files via a separate read-only endpoint that checks `visibility in ('public','unlisted')` and serves the **latest snapshot**, not in-progress revisions.

## Files to create / change
- `worker/handlers/thumbnail-render.ts`, `preview-runtime/harness.html` (shared with the smoke test)
- `lib/db/queries/projects.ts`
- `app/(marketing)/page.tsx` (grid sections), `components/home/{ProjectGrid.tsx,ProjectCard.tsx,ProjectCardMenu.tsx,ShowcaseGrid.tsx}`
- `app/p/[projectId]/page.tsx`, `components/public/{PublicPreview.tsx,PublicCodeView.tsx}`
- `app/api/projects/[projectId]/{route.ts (PATCH/DELETE),duplicate/route.ts}`, `app/api/public/projects/route.ts`, `app/api/public/projects/[projectId]/files/route.ts`
- `lib/vfs/duplicate.ts`

## Database changes
- Uses `projects.visibility`, `projects.thumbnail_key`, `projects.deleted_at`, `project_snapshots`.
- Indexes (if not yet created): `projects (owner_id, deleted_at, updated_at desc)`, partial `projects (visibility, updated_at desc) where visibility = 'public'`.
- Optional: `projects.remixed_from_project_id text null` (attribution for "Remix"). Add it to `03-data-model.md` if adopted.
- New `runs.kind` value in use: `thumbnail`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/projects?cursor=&filter=mine\|shared` | session | → `{ items: ProjectCard[], nextCursor }` |
| GET | `/api/public/projects?cursor=` | none | → `{ items: PublicProjectCard[], nextCursor }` |
| GET | `/api/public/projects/:id` | none (public/unlisted only) | → `{ title, description, screens[], thumbnailUrl }` |
| GET | `/api/public/projects/:id/files` | none (public/unlisted only) | → `{ files: {path, sha, content}[] }` from latest snapshot, materialized with `plan.backend` removed |
| PATCH | `/api/projects/:id` | editor (title) / owner (visibility) | `{ title?, visibility? }` → project |
| POST | `/api/projects/:id/duplicate` | member (source must be accessible or public) + plan limit | → `{ projectId }` |
| DELETE | `/api/projects/:id` | owner | → `{ ok }` (soft) |

## Acceptance checklist
- [ ] Signed-in home shows only the user's own + shared projects, newest first, with thumbnails and correct relative times.
- [ ] Signed-out home shows only `public` + `ready` projects; `unlisted`, `private` and mid-build projects never appear; cards show the author's name and no email anywhere in the payload.
- [ ] Duplicate and delete return 409 while a run is active; the delete dialog names the website, repo and backend that remain.
- [ ] The thumbnail updates within ~1 min after a build/edit/theme change.
- [ ] Rename, duplicate, share, visibility toggle, zip export and delete all work from the card menu, with correct role restrictions (tests for viewer/editor/owner/non-member).
- [ ] Duplicate copies files only (no backend, GitHub, builds, members); the new project opens and renders.
- [ ] The public page preview runs on seed data even when the source project has a connected backend (test asserts `plan.backend` is stripped and `lib/backend/config.ts` selects the local provider); visitor data lives under `demo:<projectId>`.
- [ ] Making a project public is blocked if the secret scanner finds anything.
- [ ] Public endpoints never return private fields (snapshot test of response shapes).
- [ ] The grid with 50 projects loads without any iframes; Lighthouse performance on home stays ≥ 85.

## Claude Code prompt
```text
We're implementing Phase 17 (projects gallery + public showcase) of AppBuilding.

Read first:
- docs/plan/phases/phase-17-projects-gallery.md (scope + acceptance checklist)
- docs/plan/07-preview-runtime.md §7–§8 (frames, harness)
- docs/plan/03-data-model.md §2, §5, §9 (projects, snapshots, indexes)
- docs/plan/08-security.md (authz, public data exposure)

Implement:
1. thumbnail.render worker job (Playwright + preview harness, local adapter + seed data), enqueued
   on run.done and debounced theme switches; placeholder fallback.
2. Cursor-paginated queries for my/shared projects and public projects (public = visibility public
   AND status ready; author display name only, never email).
3. Home grid sections (signed-in vs signed-out), project cards with status + relative time, card menu:
   open, new tab, rename, duplicate, share (reuse Phase 14 dialog), visibility, export zip (reuse), soft delete.
   Duplicate/delete → 409 while a run is active; delete dialog lists website/repo/backend that remain.
4. Public project page /p/:id with interactive preview (plan.backend stripped → local provider + seed
   data, storage namespace demo:<id>; never the real backend),
   read-only code view, Remix (duplicate into signed-in visitor's account), SEO metadata.
5. Secret scan gate before visibility → public.

Definition of done = Acceptance checklist; authz tests for every route and response-shape tests for public endpoints.

Do NOT build: likes/featured ranking, comments, plan-limit enforcement (stub the check; Phase 18 wires it).
Finish by running lint, typecheck, tests; summarize changes.
```

## Pitfalls & notes
- **Stale "edited" times.** If `updated_at` only changes on metadata edits, cards lie. Update it in the VFS write path and on run completion.
- **Thumbnail thrash.** Debounce per project and skip if the snapshot hasn't changed (key by snapshot id).
- **Shared-with-me duplicates.** A project can appear both as owned and as a membership row in naive SQL. Use `DISTINCT` or a union with role precedence.
- **Public preview data.** Visitors interact with seed data in their own browser storage (namespaced per visitor + project). Nothing is persisted server-side.
- **The video's showcase was empty** until owners toggled "Show on homepage". Seed a few well-built example projects yourself before launch.
