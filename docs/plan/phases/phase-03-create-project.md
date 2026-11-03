# Phase 03 — Create New Project (Uploads + Project Record)

| | |
|---|---|
| Video chapter(s) | Ch.8 "Create New Project" (44:59) |
| Branch | `feat/03-create-project` |
| Depends on | Phase 02 |
| Effort (solo dev + Claude Code) | ~1.5 days |
| New env vars | `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` (now required) |

## Goal
When a signed-in user submits the prompt box, the platform uploads attachments directly to object storage, creates a `projects` row with its `project_assets`, saves the prompt as the first `messages` row, creates a `runs` row of kind `plan`, and redirects to `/workspace/[projectId]`. The actual job enqueue is a stub until Phase 05/06. A failed or partial upload never leaves orphan "ready" assets.

## What we do differently from the video (and why)
- **Presigned PUT uploads straight to storage**, not files posted through a Next.js route. Serverless body limits are ~4.5 MB, while our limit is 10 MB × 5. It's also cheaper and faster.
- **Server-side verification after upload:** `HEAD` for the real size, plus a magic-byte sniff of the first bytes to confirm it's actually an image. Assets go `pending_upload → ready | rejected`. The video trusts the client.
- **`StorageProvider` interface.** The S3 API covers MinIO, AWS S3, Cloudflare R2, Railway buckets and CloudHike storage. Switching vendors is an env change, not a code change.
- **Uploads first, project second, with no DB rows until create succeeds.** Files go to a temporary prefix (`tmp/`) with signed upload tokens. `POST /api/projects` verifies them, copies them to their final keys and inserts everything in one transaction. Abandoned uploads are deleted by a bucket lifecycle rule, so there are no orphan rows and no FK problems from assets created before their project.
- **`runs` + `messages` rows are created now** (tables from `03-data-model.md`), so Phase 05/06 only adds the enqueue call.

## Steps

### 1. Storage provider
`lib/providers/storage/index.ts`:
```ts
export interface StorageProvider {
  presignPut(key: string, opts: { contentType: string; contentLength: number; expiresIn: number }): Promise<{ url: string; headers: Record<string,string> }>;
  presignGet(key: string, expiresIn: number): Promise<string>;
  head(key: string): Promise<{ size: number; contentType?: string } | null>;
  getRange(key: string, start: number, end: number): Promise<Uint8Array>;
  put(key: string, body: Uint8Array | ReadableStream, contentType: string): Promise<void>;
  copy(srcKey: string, dstKey: string): Promise<void>;
  delete(key: string): Promise<void>;
}
```
`lib/providers/storage/s3.ts` implements it with `@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner` (`forcePathStyle: true` when `S3_ENDPOINT` is set, for MinIO). Sign `ContentLength` and `ContentType` into the presigned request, so the client can't upload a different size or type than it declared.

### 2. Bucket CORS and lifecycle
- CORS: allow `PUT` from `NEXT_PUBLIC_APP_URL` with `Content-Type`.
- Lifecycle rule: expire objects under `tmp/` after 1 day.
- Add both to the MinIO init in docker-compose and document them for hosted buckets.

### 3. Schema
Add `projects` and `project_assets` (columns per `03-data-model.md` §2), plus the minimal `messages` (§3) and `runs` (§4) tables. Phase 05 adds `run_events` and the job columns' behaviour. For now `runs.status` is just set to `queued`.

### 4. Key layout
- Temporary: `tmp/{userId}/{uploadId}` (no extension; expires by lifecycle rule).
- Final: `users/{userId}/projects/{projectId}/assets/{assetId}.{ext}`. The extension is derived from the **verified** MIME type, never from the user's file name. Keep the original file name only as metadata.

### 5. Presign endpoint
`POST /api/uploads/presign` (requireUser):
- Body: `{ files: [{ name, size, type }] }`, validated against `UPLOAD_LIMITS` (`lib/limits.ts`).
- For each file: `uploadId = newId("upl")`, presigned PUT to `tmp/{userId}/{uploadId}`, plus an `uploadToken` = HMAC (keyed with `BETTER_AUTH_SECRET`, or a dedicated secret) over `{userId, uploadId, size, type, exp}`.
- No DB writes.
- Return `{ uploads: [{ uploadId, uploadToken, url, headers }] }`.

### 6. Client upload with progress
`lib/uploads/client.ts`: `XMLHttpRequest` PUT (fetch has no upload progress) with an `onProgress` per file, concurrency 3, one retry on network error. The prompt box shows per-file progress and an overall "Uploading 2 assets… 45%" (as in the video).

### 7. Create endpoint
`POST /api/projects` (requireUser):
- Body: `{ prompt, platforms, uploads: [{ uploadId, uploadToken }] }` (Zod: prompt length per `PROMPT_LIMITS`, platforms non-empty, ≤ 5 uploads).
- For each upload: verify the token (signature, `userId` matches the session, not expired). `head(tmp key)` → size matches the token. `getRange(0, 15)` → magic bytes match PNG/JPEG/WEBP/GIF. If any upload fails, delete those tmp objects and return 422 listing them.
- `projectId = newId("prj")`. Copy each verified tmp object to its final key (`assetId = newId("ast")`).
- In **one transaction**: insert `projects` (`status: "planning"`, `title`: placeholder from the first words of the prompt; the agent renames it in Phase 06, `template_version` from `templates/expo-app/VERSION` or `"unset"` until Phase 07). Insert `project_assets` rows (`kind: "reference_image"`, `status: "ready"`, verified mime/size, width/height if cheaply available). Insert `messages` (`role: "user"`, `kind: "text"`, `content: prompt`, `attachments: assetIds`). Insert `runs` (`kind: "plan"`, `status: "queued"`, `input: { messageId }`).
- If the transaction fails, delete the copied final objects (best effort). The tmp objects expire anyway.
- After commit: `await enqueuePlan(runId)` — **a stub in `lib/jobs/enqueue.ts` that just logs** until Phase 05 wires pg-boss and Phase 06 the handler.
- Return `{ projectId, runId }`.

### 8. Prompt box integration
Signed-in submit → presign → upload with progress → create → `router.push(/workspace/${projectId})`. Status text cycles "Uploading…", "Creating project…", "Opening your workspace…". On failure, keep the prompt and files in the box and show the error.

### 9. Orphan cleanup
- Abandoned uploads: handled by the `tmp/` lifecycle rule (step 2).
- Final-key objects with no `project_assets` row (a crash between copy and commit): `lib/uploads/cleanup.ts` lists `users/*/projects/*/assets/` objects older than 24 h that have no matching row, and deletes them. Phase 05 schedules it with pg-boss cron.
- `project_assets.status = 'pending_upload'` is reserved for in-workspace chat attachments (Phase 09), which may use a row-first flow.

### 10. Workspace placeholder
`app/workspace/[projectId]/page.tsx`: server component → `requireProjectAccess(projectId, "viewer")` → renders the title, prompt and asset thumbnails (via `presignGet`). Phase 04 replaces the layout.

### 11. Implement `requireProjectAccess` for real
Now that `projects` exists: owner check (members come in Phase 14), `deleted_at is null`, 404 otherwise.

## Files to create / change
```
lib/providers/storage/{index.ts,s3.ts}
lib/uploads/{client.ts,verify.ts,cleanup.ts}
lib/db/schema/{projects.ts,messages.ts,runs.ts}
lib/jobs/enqueue.ts                      (stub)
lib/auth/require.ts                      (real project lookup)
app/api/uploads/presign/route.ts
app/api/projects/route.ts                (POST)
app/workspace/[projectId]/page.tsx       (placeholder)
components/home/prompt-box.tsx           (upload progress + create)
docker-compose.yml                       (bucket CORS)
drizzle/0002_projects.sql
```

## Database changes
- `projects` (§2), `project_assets` (§2), `messages` (§3), `runs` (§4, minimal: id, project_id, kind, status, input, output, error, created_at).
- Indexes: `projects (owner_id, deleted_at, updated_at desc)`, `messages (project_id, created_at)`, `runs (project_id, created_at desc)`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/uploads/presign` | user | `{files[{name,size,type}]}` → `{uploads[{uploadId,uploadToken,url,headers}]}` |
| POST | `/api/projects` | user | `{prompt, platforms, uploads[{uploadId,uploadToken}]}` → `201 {projectId, runId}` · `422 {rejected:[uploadId]}` |
| GET | `/api/projects/:id/assets/:assetId` | viewer | 302 to a short-lived presigned GET |

## Acceptance checklist
- [ ] Submitting with 2 images shows per-file progress, then lands on `/workspace/{id}` with the prompt and thumbnails.
- [ ] DB has the `projects`, 2 `project_assets` (`ready`), 1 `messages` and 1 `runs` (`plan`, `queued`) rows. Objects exist in the bucket under the documented key layout.
- [ ] A file renamed `evil.png` that is really HTML/SVG/EXE is rejected server-side (magic bytes), its object is deleted, and the UI explains why.
- [ ] Changing the declared size or type after presigning makes the PUT fail (signature mismatch).
- [ ] A tampered, expired or other user's `uploadToken` makes `POST /api/projects` fail (403) and nothing is created.
- [ ] User B opening user A's `/workspace/{id}` gets a 404.
- [ ] Killing the browser mid-upload leaves no DB rows; the tmp object expires via the lifecycle rule. `cleanup()` removes final-key objects that have no row (unit test with a fake storage + clock).
- [ ] Works against MinIO locally; S3/R2 config is documented in `.env.example`.

## Claude Code prompt
```text
Read first:
- docs/plan/02-architecture.md (§3 A9 provider adapters, §5 layout)
- docs/plan/03-data-model.md (§2 projects/project_assets, §3 messages, §4 runs)
- docs/plan/08-security.md (uploads section)
- docs/plan/phases/phase-03-create-project.md (this phase)

Implement Phase 03:
1. lib/providers/storage StorageProvider interface + S3 implementation
   (@aws-sdk/client-s3, presigner; forcePathStyle when S3_ENDPOINT is set; sign
   ContentLength + ContentType).
2. Drizzle tables projects, project_assets, messages, runs (minimal) exactly per the
   data model doc; generate + commit the migration.
3. POST /api/uploads/presign with Zod validation against lib/limits.ts: presigned PUT
   to tmp/{userId}/{uploadId} + an HMAC uploadToken per file, no DB writes.
4. Client upload helper using XHR PUT with progress (concurrency 3, 1 retry), and the
   prompt-box flow: presign → upload → create → redirect, with status text.
5. POST /api/projects: verify each uploadToken + object (HEAD size + magic bytes for
   png/jpeg/webp/gif), copy to final keys, then in ONE transaction create project +
   project_assets(ready) + message + run(kind=plan,status=queued);
   after commit call enqueuePlan(runId) from lib/jobs/enqueue.ts, which is a logging
   stub for now.
6. Real requireProjectAccess (owner only, 404 otherwise) and a placeholder
   /workspace/[projectId] page showing prompt + thumbnails via presigned GET.
7. lib/uploads/cleanup.ts for final-key objects with no asset row (not scheduled yet) + unit test.
8. Bucket CORS + tmp/ lifecycle rule in docker-compose MinIO init.

Do NOT build the workspace layout, pg-boss, SSE, or any AI calls.
Definition of done = Acceptance checklist in the phase doc (write tests for the
security items: magic-byte rejection, tampered/foreign upload tokens, cross-user 404).
Finish with lint/typecheck/test and a summary.
```

## Pitfalls & notes
- SVG is **not** allowed as a reference image. It can carry script, and we serve assets back to browsers.
- Re-encoding images (e.g. with `sharp`) to strip EXIF/GPS is a cheap privacy win. Add it if you serve reference images publicly (showcase).
- Presigned URL expiry: 10 minutes is plenty. Never return a presigned GET with a long expiry for private assets.
- Don't put the original filename in the storage key (path traversal, Unicode weirdness, PII).
- The video stored a "status: planning" project and then navigated to a placeholder page. Same here. The plan job actually runs from Phase 06.
