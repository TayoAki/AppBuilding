# 02 — System Architecture

This is the target architecture for the whole platform. Every phase doc builds one slice of it. If a phase doc and this file disagree, fix one of them before writing code.

Working name: **AppBuilding** (the repo name). Pick the real brand before launch. Don't reuse the video's "AppForge" name or assets.

---

## 1. One-paragraph summary

A **Next.js web app** (the "platform") lets a signed-in user type a prompt. The platform saves a **project** and puts a **generation job** on a queue. A separate **worker** process picks up the job and runs the **AI agent** (Claude via the Anthropic TypeScript SDK). The agent first proposes a **plan** (the screens to build). Once the user confirms, it writes **React Native / Expo source files** into the project's **virtual file system**, which lives in Postgres. The worker streams **events** back to the browser on a project-scoped stream. Anything that runs generated code on our servers (`expo export`, headless renders, device-preview dev servers) runs in a separate **sandbox service** with no secrets. The browser shows the files live on a **canvas** of phone frames and in an interactive **preview**. Both run in a sandboxed **preview runtime** (react-native-web inside an iframe). From there the user can edit through chat or a visual inspector, open the app on a real phone in **Expo Go** (via a hosted Expo dev server; Snack when it supports the SDK), connect a **backend** for the generated app, **export** it (zip / GitHub), **build** store binaries (EAS Build) and **publish** it to the web (EAS Hosting).

---

## 2. Component diagram

```
┌──────────────────────────── Browser ─────────────────────────────┐
│  Next.js UI (React, shadcn/ui)                                   │
│  ├─ Home (prompt box, projects grid)                             │
│  ├─ Workspace  ┌──────────┬──────────────────────┬────────────┐  │
│  │             │ Chat     │ Canvas (N iframes)   │ Preview /  │  │
│  │             │ panel    │  one per screen      │ Code / QR  │  │
│  │             └──────────┴──────────────────────┴────────────┘  │
│  └─ Preview runtime iframes (sandboxed, opaque origin)           │
│        react-native-web + module registry + router shim          │
│        ⇅ postMessage bridge (select element, errors, nav)        │
└───────────────▲───────────────────────────────▲──────────────────┘
                │ HTTPS (server actions / route  │ SSE: /api/projects/:id/events
                │ handlers)                      │ (tails run_events for the project)
┌───────────────┴───────────────────────────────┴──────────────────┐
│  Next.js server (Vercel or a Node host)                          │
│  ├─ Auth (Better Auth)        ├─ Project/file APIs               │
│  ├─ Upload presign (S3 API)   ├─ Runs + job_outbox (one txn)     │
│  ├─ Webhooks: Stripe, EAS     ├─ OAuth callbacks: GitHub, backend│
│  ├─ App data gateway /api/apps/[projectId]/… (gateway mode)      │
│  └─ Credit charge / plan gating                                  │
└───────────────┬──────────────────────────────────────────────────┘
                │ SQL                                     ▲ NOTIFY
┌───────────────▼─────────────────────────────────────────┴────────┐
│  Postgres  (platform DB: users, projects, files, runs, events,   │
│             credits, outbox, connections … + pg-boss schema)     │
└───────────────▲──────────────────────────────────────────────────┘
                │ SQL / outbox dispatch / job fetch (DATABASE_URL_DIRECT)
┌───────────────┴──────────────────────────────────────────────────┐
│  Worker service (long-running Node process, has secrets)         │
│  ├─ outbox dispatcher → pg-boss                                  │
│  ├─ agent.chat (Planner) / agent.step (generators) → Anthropic   │
│  ├─ export.github     → GitHub API (Contents + Git Data API)     │
│  ├─ build.native      → EAS Build (owner's Expo token)           │
│  ├─ publish.web       → sandbox export → EAS Hosting deploy      │
│  ├─ backend.provision → backend provider API (per generated app) │
│  └─ device.session    → asks the sandbox for a dev server        │
└───────────────┬──────────────────────────────────┬───────────────┘
                │ HTTPS: per-job token +           │
                │ signed storage URLs              │
┌───────────────▼──────────────────────┐           │
│  Sandbox service (NO secrets, NO DB) │           │
│  ├─ Playwright renders / thumbnails  │           │
│  ├─ `expo export -p web`             │           │
│  └─ Path B Expo dev servers ◀── Expo Go on phones (session URL)
└──────────────────────────────────────┘           │
                                                   ▼
     Object storage (S3-compatible): uploads, assets, project tarballs, zips, thumbnails
```

---

## 3. Key architectural decisions (and why)

| # | Decision | Why | Video did |
|---|----------|-----|-----------|
| A1 | **The agent runs in a worker, not in a request handler**, from day one. | Generating 3–8 screens takes minutes. Serverless functions time out at 15 s–5 min. Building on request handlers first and moving later (the video moves in Ch.11) means rewriting the agent loop. | Request handler first, then CloudHike Jobs + webhook back to an ngrok URL |
| A2 | **Postgres-backed queue (pg-boss, pinned major ≥ 10)** plus a dedicated worker service, fed by a **transactional outbox** (`job_outbox`). | No new vendor. A run, its credit charge and its job are created in one transaction on the web's connection; the worker's dispatcher moves outbox rows to pg-boss. Local dev needs no tunnel: the worker connects to the DB directly. You can swap in Inngest/Trigger.dev later behind `lib/jobs`. | CloudHike Jobs → webhook to public URL (requires ngrok locally) |
| A3 | **Events are persisted** (`run_events`, project-scoped, `run_id` nullable) and streamed over SSE (`/api/projects/:id/events`) with `Last-Event-ID` resume. Writes outside runs (inline edits, code saves, theme switches, restores) use the same stream. | Refreshing or reconnecting loses nothing. The SSE handler can be short-lived on serverless because the client reconnects and resumes. | Unclear; the video had to refresh to see progress |
| A4 | **Project source lives in Postgres as a virtual file system** (`project_files` + `file_revisions`), not on disk. | Every edit is versioned (undo, diff, "restore version"). Export, device preview and builds all start from the same canonical state. | Files saved in DB + storage |
| A5 | **The generated app follows a fixed template contract** (see `06-generated-app-template.md`). Generators write only screens and shared components; the platform writes themes, schema, seed data, layouts and configs from `projects.plan`; allowlisted imports only; `package.json` is never agent-written. | This is the biggest lever for quality and security. It makes the browser preview, device preview, export and EAS builds work reliably, and it stops the agent from adding dependencies or build scripts that would run arbitrary code in the worker. | Free-form generation; hit Expo SDK version mismatches |
| A6 | **Our own browser preview runtime** for the canvas and the interactive preview. **Real devices use a hosted Expo dev server** (Path B) by default; Snack (Path A) only when its published SDK matches the template (`06` §1). | The canvas shows N screens at once and needs element-level instrumentation and instant theme switching. Store Expo Go supports only SDK 58, while published `snack-sdk` stops at 54 (`15` §1). | Appears to be a web preview plus Snack for QR |
| A7 | **Themes are data, not code.** `projects.plan.themePackage` is the only source; generated code imports `colors` from `@/theme` (CSS variables in the preview). Switching a theme changes `activeThemeId`, with no LLM call. | Theme switching is instant and free, which the video highlights as a feature. | Same idea |
| A8 | **Anthropic TS SDK directly**: `claude-opus-5-5` (env), adaptive thinking, streaming, prompt caching, strict tools for the Planner, structured outputs for JSON generators, server-side refusal fallbacks. | First-party features (caching, structured outputs, fallbacks, usage reporting for credit metering) with no gateway in between. | OpenAI Agents SDK through an OpenAI-compatible endpoint (CloudHike, later Anthropic's) |
| A9 | **Provider adapters** for every third-party dependency: `StorageProvider`, `BackendProvider` (generated apps), `ImageProvider`, `BuildProvider`, `HostingProvider`, `BillingProvider`. | The video ties every layer to one sponsor (CloudHike). Adapters let you start with any vendor and switch without touching the agent or UI. | Hard-wired to CloudHike |
| A10 | **All third-party tokens are encrypted at rest** (AES-256-GCM, versioned key), and **every project route goes through `requireProjectAccess()`**. | These are the classic AI-builder bugs: leaked OAuth tokens and IDOR on project IDs. | Added an encryption key later; CodeRabbit flagged ownership issues |
| A11 | **A sandbox service with no secrets and no DB access** runs every step that processes generated code on our servers (renders, `expo export`, device-preview dev servers). | Generated code is untrusted. The worker holds secrets, so it must never run it (`08` §5). | Not discussed |

---

## 4. Request and data flows

### 4.1 Prompt → plan → build

```
User submits prompt (+ ≤5 images)
  → POST /api/projects            (create project; assets uploaded on Send via presigned PUT)
  → one txn: insert project + first message + run(kind=chat, created_by, billed_user_id = owner)
             + job_outbox row "agent.chat" {runId, projectId, userId}
  → redirect /workspace/:projectId
Browser opens SSE /api/projects/:projectId/events (stays open while the workspace is open)
Worker: agent.chat (Planner, see 05 §3)
  → streams prose (assistant.delta) → calls propose_plan(AppPlan)
  → server validates + caps plan → plan_card message → projects.plan (draft) → plan.ready
User: "only first 3 screens" → POST /api/projects/:id/messages → agent.chat → propose_plan again
User: [Confirm & build]  (or "looks good" → confirm_plan)
  → one txn: charge credits (ledger charge:<runId>) + run(kind=build, credits_charged)
             + job_outbox "agent.step" {runId, projectId, userId, step:"foundation"}
Worker: agent.step chain (one job per step, each enqueues the next; 05 §4)
  1. foundation   → JSON: 5 themes + collections → platform writes theme/themes.ts, data/schema.ts, data/seed.ts
  2. components   → JSON: 5–10 shared components → components/<Name>.tsx (validated, 1 repair)
  3. screen:<id>  → one .tsx per screen, streamed (file.writing) → validate → 1 repair → file.written
  4. platform     → app/_layout.tsx + app/(tabs)/_layout.tsx from the plan; backend setup if connected
  5. finish       → render smoke test → snapshot → build_summary → run.done
Canvas iframes receive file revisions → hot-swap modules → re-render
```

### 4.2 Edit flows

- **Chat edit / add screen**: the Planner calls `update_app(AppChange)`. The change pipeline regenerates only the affected data, components and screens; edited screens keep their old code until the new code passes validation. See `05` §3.5, §4.2.
- **Inline visual edit**: the frames apply an `edit.override` instantly; then `POST /api/projects/:id/files/edits` with structured operations (`{afid, op: "setStyle", prop, value}` …) and `baseRevisionId`. The server applies them as deterministic AST transforms (no LLM), writes a revision (`author = user_inline`) and emits `file.written` on the project stream.
- **Code save**: `PUT /api/projects/:id/files/content` `{path, content, baseRevisionId}` → validate → revision (`author = user_code`) → `file.written` (Phase 10b). Both routes return 409 when `baseRevisionId` isn't current.
- **Theme switch**: `POST /api/projects/:id/theme` (or the Planner's `set_theme`) → update `activeThemeId` in `projects.plan.themePackage` → `plan.updated`. Instant: the preview swaps CSS variables, and device sessions get the re-rendered `theme/themes.ts`.

### 4.3 Device preview (Phase 11; `06` §1)

- **Path B (default): hosted Expo dev server.** When the person opens the QR popover, the worker asks the sandbox service for a session (`device.session`). The sandbox materializes the project (`devserver` target) with the cached SDK 58 `node_modules`, starts `expo start` for that session, and exposes it through a per-session unguessable URL. The exact mechanism (`--tunnel` or reverse proxy + `EXPO_PACKAGER_PROXY_URL`) is settled by spike S3. The QR shows the session's `exp://` URL. Later `file.written` / `plan.updated` events rewrite single files in the session dir, and Metro Fast Refresh updates connected phones without rescanning. Sessions stop after 15 idle minutes.
- **Path A: Snack**, only when a published `snack-sdk` supports the template SDK (`DEVICE_PREVIEW_MODE=snack`). The live session runs in the browser tab; `snack.sync` makes persistent saves.
- **Path C (fallback):** "Open on phone in browser" link to the web preview.

### 4.4 Export / build / publish

All four start the same way: the project is **materialized** (template files plus project files) into a tarball in storage or an in-memory tree for GitHub. Steps that run generated code go to the sandbox service.
- Zip: stream to storage → signed download URL.
- GitHub: the first push goes to the default branch **only if the repo is empty**; it creates the first commit with the Contents API (or the repo is created with `auto_init`), because the Git Data API returns 409 on an empty repo. After that, Git Data API (blobs → tree → commit), always on a new branch `appbuilding/update-<ts>` with a PR.
- Native build (owner-only): `EAS_NO_VCS=1 eas build --non-interactive --no-wait --json` with the **owner's** Expo token. Status arrives via the project's EAS webhook (per-project secret), which is re-verified by fetching the build from EAS. The run ends at submit; status lives in `native_builds`.
- Web (owner-only): the sandbox runs `npx expo export -p web` (`web.output: "single"`), then `eas deploy --prod --dev-domain <slug>` (EAS Hosting) with the owner's token → stable public URL.

---

## 5. Repository layout (platform)

```
/                         Next.js app (App Router)
├─ app/
│  ├─ (marketing)/page.tsx            Home: prompt box + projects
│  ├─ workspace/[projectId]/page.tsx  Workspace
│  ├─ pricing/page.tsx
│  └─ api/
│     ├─ auth/[...all]/route.ts       Better Auth
│     ├─ projects/…                   CRUD, messages, files/edits, files/content, theme, device, export
│     ├─ projects/[projectId]/events/route.ts  SSE (project stream; primary)
│     ├─ runs/[runId]/events/route.ts SSE (same stream filtered to one run)
│     ├─ uploads/presign/route.ts
│     ├─ github/{connect,callback}/route.ts
│     ├─ backend/{connect,callback}/route.ts
│     ├─ apps/[projectId]/…            data gateway for generated apps (Phase 13, gateway mode)
│     └─ webhooks/{stripe,eas}/route.ts
├─ components/                         shadcn/ui + feature components
│  ├─ home/  workspace/  canvas/  inspector/  chat/
├─ lib/
│  ├─ db/ (schema.ts, index.ts, queries/)
│  ├─ auth/ (server.ts, client.ts, require.ts)
│  ├─ jobs/ (queue.ts, outbox.ts, dispatcher.ts, types.ts, wrappers.ts: singleShot / step)
│  ├─ runs/ (events.ts: append/tail project-scoped run_events)
│  ├─ vfs/ (project file system: read/write/revisions/materialize)
│  ├─ ai/ (client.ts, models.ts, pricing.ts, prompts/{rules,state,version}.ts, planner/, generators/, usage.ts)
│  ├─ codemods/ (AST edit operations for inline editing)
│  ├─ device/ (devserver client, snack/ toSnackFiles.ts + useSnackSession.ts for Path A)
│  ├─ backend-schema/ (collections → app_<name> DDL, additive sync) · render/ (themes, schema, seed, layouts, routes)
│  ├─ dto.ts (API response shapes) · errors.ts (typed app errors → HTTP)
│  ├─ validate/ (import allowlist, typecheck-lite, route checks)
│  ├─ providers/ (storage/, backend/, images/, build/, hosting/, billing/)
│  ├─ crypto/ (encrypt.ts)
│  ├─ credits/ (ledger.ts, pricing.ts, gate.ts)
│  └─ env.ts (zod-validated env)
├─ worker/
│  ├─ index.ts (outbox dispatcher + pg-boss createQueue/work for all handlers)
│  └─ handlers/ (agent-chat.ts, agent-step.ts, device-session.ts, snack-sync.ts, export-*.ts, …)
├─ sandbox/                separate service: NO secrets, NO DB (08 §5)
│  ├─ server.ts (POST /jobs with per-job token; render, export-web, devserver start/stop/update)
│  └─ Dockerfile (Node 22, Chromium, cached template node_modules, public/preview-runtime)
├─ preview-runtime/        separately bundled; served as static assets
│  ├─ src/ (loader.ts, transpile.ts, router-shim/, bridge.ts, shims/)
│  └─ build.mjs (esbuild → public/preview-runtime/*.js)
├─ templates/expo-app/     the generated-app template (see 06)
├─ drizzle/                migrations
├─ evals/                  agent eval harness (see 10)
└─ docs/plan/              this plan
```

---

## 6. Environments and processes

| Process | Local dev | Production |
|---------|-----------|------------|
| Web | `npm run dev` (Next.js) | Vercel (or Railway/Render Node service) |
| Worker | `npm run worker` (tsx watch) | Railway / Fly / Render long-running service: Node 22+, `eas-cli`, writable temp disk. Holds secrets; never runs generated code. A JDK is needed only if you use the `keytool` keystore fallback. |
| Sandbox | `npm run sandbox` (local, same machine) | Separate Railway / Fly / Render service: Node 22+, Chromium (Playwright), cached SDK 58 template `node_modules`, the built `public/preview-runtime/`, Expo CLI for dev servers. **No secrets, no DB access**; reached only by the worker (`SANDBOX_URL` + per-job tokens signed with `SANDBOX_TOKEN_SECRET`) and, for device sessions, by phones on per-session URLs. Restrict egress where the host allows. |
| DB | Docker Postgres or a hosted dev DB | Managed Postgres (Neon, Railway, Supabase, CloudHike …). pg-boss and SSE `LISTEN` need a **direct/session-mode** connection (`DATABASE_URL_DIRECT`); transaction-mode poolers break `LISTEN`. |
| Storage | MinIO (docker) or hosted bucket | S3 / R2 / Railway bucket / CloudHike storage |
| Preview runtime origin | `http://localhost:3000/preview-runtime/` loaded into `sandbox="allow-scripts"` iframes (opaque origin) | Same, or a separate domain like `preview.<brand>.com` for defense-in-depth |

Tunnels such as ngrok are needed **only** for inbound third-party webhooks (Stripe/EAS) and OAuth callbacks when you test them locally. The job system does not need them. Path B device preview may use Expo's own `--tunnel` inside the sandbox if spike S3 picks that mechanism; that's a sandbox concern, not a dev-machine requirement.

---

## 7. Non-goals for v1

- No real-time multi-user co-editing (sharing is read/edit access, not live collaboration).
- No arbitrary npm dependencies in generated apps. They use the template's allowlist only.
- No custom native modules (anything requiring a dev client instead of Expo Go).
- No in-platform App Store/Play Store submission (we produce AAB/APK/IPA; submission is a stretch goal via `eas submit`).
