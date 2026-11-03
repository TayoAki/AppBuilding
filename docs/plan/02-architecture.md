# 02 — System Architecture

This is the target architecture for the whole platform. Every phase doc builds one slice of it. If a phase doc and this file disagree, fix one of them before writing code.

Working name: **AppBuilding** (the repo name). Pick the real brand before launch. Don't reuse the video's "AppForge" name or assets.

---

## 1. One-paragraph summary

A **Next.js web app** (the "platform") lets a signed-in user type a prompt. The platform saves a **project** and puts a **generation job** on a queue. A separate **worker** process picks up the job and runs the **AI agent** (Claude via the Anthropic TypeScript SDK). The agent first proposes a **plan** (the screens to build). Once the user confirms, it writes **React Native / Expo source files** into the project's **virtual file system**, which lives in Postgres. The worker streams **run events** back to the browser. The browser shows the files live on a **canvas** of phone frames and in an interactive **preview**. Both run in a sandboxed **preview runtime** (react-native-web inside an iframe). From there the user can edit through chat or a visual inspector, open the app on a real phone through **Expo Snack / Expo Go**, connect a **backend** for the generated app, **export** it (zip / GitHub), **build** store binaries (EAS Build) and **publish** it to the web (EAS Hosting).

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
                │ HTTPS (server actions / route  │ SSE: /api/runs/:id/events
                │ handlers)                      │ (tails run_events)
┌───────────────┴───────────────────────────────┴──────────────────┐
│  Next.js server (Vercel or a Node host)                          │
│  ├─ Auth (Better Auth)        ├─ Project/file APIs               │
│  ├─ Upload presign (S3 API)   ├─ Enqueue jobs (pg-boss)          │
│  ├─ Webhooks: Stripe, EAS     ├─ OAuth callbacks: GitHub, backend│
│  └─ Credit checks / plan gating                                  │
└───────────────┬──────────────────────────────────────────────────┘
                │ SQL                                     ▲ NOTIFY
┌───────────────▼─────────────────────────────────────────┴────────┐
│  Postgres  (platform DB: users, projects, files, runs, events,   │
│             credits, connections … + pg-boss queue schema)       │
└───────────────▲──────────────────────────────────────────────────┘
                │ SQL / job fetch
┌───────────────┴──────────────────────────────────────────────────┐
│  Worker service (long-running Node process, no request timeout)  │
│  ├─ agent.plan / agent.build / agent.edit  → Anthropic API       │
│  ├─ snack.sync        → Expo Snack (persistent save only)        │
│  ├─ export.github     → GitHub API (Git Data API)                │
│  ├─ build.native      → EAS Build (user's Expo token)            │
│  ├─ publish.web       → `expo export -p web` + EAS Hosting       │
│  ├─ backend.provision → backend provider API (per generated app) │
│  └─ thumbnail.render  → headless Chromium screenshot             │
└───────────────┬──────────────────────────────────────────────────┘
                │
     Object storage (S3-compatible): uploads, assets, zips, thumbnails
```

---

## 3. Key architectural decisions (and why)

| # | Decision | Why | Video did |
|---|----------|-----|-----------|
| A1 | **The agent runs in a worker, not in a request handler**, from day one. | Generating 3–8 screens takes minutes. Serverless functions time out at 15 s–5 min. Building on request handlers first and moving later (the video moves in Ch.11) means rewriting the agent loop. | Request handler first, then CloudHike Jobs + webhook back to an ngrok URL |
| A2 | **Postgres-backed queue (pg-boss)** plus a dedicated worker service. | It needs no new vendor, jobs are transactional with app writes, and local dev needs no tunnel: the worker connects to the DB directly. You can swap in Inngest/Trigger.dev later behind `lib/jobs`. | CloudHike Jobs → webhook to public URL (requires ngrok locally) |
| A3 | **Run events are persisted** (`run_events` table) and streamed over SSE with `Last-Event-ID` resume. | Refreshing or reconnecting loses nothing. The SSE handler can be short-lived on serverless because the client reconnects and resumes. | Unclear; the video had to refresh to see progress |
| A4 | **Project source lives in Postgres as a virtual file system** (`project_files` + `file_revisions`), not on disk. | Every edit is versioned (undo, diff, "restore version"). Export, Snack and builds all start from the same canonical state. | Files saved in DB + storage |
| A5 | **The generated app follows a fixed template contract** (see `06-generated-app-template.md`). The agent may only write inside allowed folders, use allowlisted imports, and never edits `package.json`. | This is the biggest lever for quality and security. It makes the browser preview, Snack, export and EAS builds work reliably, and it stops the agent from adding dependencies or build scripts that would run arbitrary code in the worker. | Free-form generation; hit Expo SDK version mismatches |
| A6 | **Our own browser preview runtime** for the canvas and the interactive preview. **Expo Snack is used only for real devices** (QR). | The canvas shows N screens at once and needs element-level instrumentation for inline editing. It also needs instant theme switching. Snack is one session per app and is SDK-version-limited. | Appears to be a web preview plus Snack for QR |
| A7 | **Themes are data, not code.** Generated code reads colors/spacing from `theme/tokens`. Switching a theme rewrites one file, with no LLM call. | Theme switching is instant and free, which the video highlights as a feature. | Same idea |
| A8 | **Anthropic TS SDK directly**: model `claude-opus-5-5`, adaptive thinking, streaming, prompt caching, strict tools, structured outputs for the plan, server-side refusal fallbacks. Model id comes from env. | First-party features (caching, structured outputs, fallbacks, usage reporting for credit metering) with no gateway in between. | OpenAI Agents SDK through CloudHike's OpenAI-compatible gateway |
| A9 | **Provider adapters** for every third-party dependency: `StorageProvider`, `BackendProvider` (generated apps), `ImageProvider`, `BuildProvider`, `HostingProvider`, `BillingProvider`. | The video ties every layer to one sponsor (CloudHike). Adapters let you start with any vendor and switch without touching the agent or UI. | Hard-wired to CloudHike |
| A10 | **All third-party tokens are encrypted at rest** (AES-256-GCM, versioned key), and **every project route goes through `requireProjectAccess()`**. | These are the classic AI-builder bugs: leaked OAuth tokens and IDOR on project IDs. | Added an encryption key later; CodeRabbit flagged ownership issues |

---

## 4. Request and data flows

### 4.1 Prompt → plan → build

```
User submits prompt (+ ≤5 images)
  → POST /api/projects            (create project, assets already uploaded via presigned PUT)
  → insert run(kind=plan), enqueue job "agent.plan" {runId}
  → redirect /workspace/:projectId
Browser opens SSE /api/runs/:runId/events
Worker: agent.plan
  → Claude (structured output: AppPlan JSON)
  → insert message(role=assistant, kind=plan_card, payload=AppPlan)
  → run_events: status, thinking_summary, plan_ready
User: "only first 3 screens" → POST /api/projects/:id/messages
  → run(kind=plan_revision) → agent.plan (with previous plan in context) → new plan_card
User: [Confirm & build]
  → run(kind=build) → enqueue "agent.build"
Worker: agent.build (tool loop)
  1. write theme/tokens + 5 theme packs
  2. write data/ seed + lib/store
  3. write shared components/
  4. for each screen: write app/<route>.tsx → validate → emit file_written events
  5. final validation pass (typecheck-lite, import allowlist, render smoke test)
  → run_events stream to UI: screen_started, file_written(path, rev), screen_done, run_done
Canvas iframes receive file revisions → hot-swap modules → re-render
```

### 4.2 Edit flows

- **Chat edit / add screen**: same as build, run kind `edit`. An intent step decides whether this is add_screen, modify_screen, restyle, fix_error or question. See `05-ai-agent-design.md` §6.
- **Inline visual edit**: browser → `POST /api/projects/:id/edits` with a structured operation (`{afid, op: "setStyle", prop, value}`). The server applies it as a deterministic AST transform (no LLM) and writes a new file revision. Canvas hot-reloads.
- **Theme switch**: `POST /api/projects/:id/theme` → rewrite `theme/active.ts` → new revision. Instant.

### 4.3 Device preview

The **live** Snack session runs **in the browser tab** (`snack-sdk` works client-side). When the QR popover opens, the workspace builds the Snack files from the materialized project and gets the `exp://` URL → QR. On every `file.written` event it pushes only the changed files, so phones live-reload without rescanning. The worker job `snack.sync` only makes a **persistent save** (a stable link that works with the workspace closed), stored in `projects.snack`. See Phase 11.

### 4.4 Export / build / publish

All three start the same way. The worker **materializes** the project: template files plus project files go into a clean temp dir (or an in-memory tree for GitHub).
- Zip: stream to storage → signed download URL.
- GitHub: Git Data API (blobs → tree → commit). The first push goes to `main`; later pushes go to a new branch `appbuilding/update-<ts>` with a PR.
- Native build: `eas build --non-interactive --no-wait` with the **user's** Expo token. Status arrives via EAS webhook.
- Web: `npx expo export -p web` in the worker, then `eas deploy` (EAS Hosting) → public URL.

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
│     ├─ projects/…                   CRUD, messages, edits, theme, snack, export
│     ├─ runs/[runId]/events/route.ts SSE
│     ├─ uploads/presign/route.ts
│     ├─ github/{connect,callback}/route.ts
│     ├─ backend/{connect,callback}/route.ts
│     └─ webhooks/{stripe,eas}/route.ts
├─ components/                         shadcn/ui + feature components
│  ├─ home/  workspace/  canvas/  inspector/  chat/
├─ lib/
│  ├─ db/ (schema.ts, index.ts, queries/)
│  ├─ auth/ (server.ts, client.ts, require.ts)
│  ├─ jobs/ (queue.ts, enqueue.ts, types.ts)
│  ├─ runs/ (events.ts — append/tail run_events)
│  ├─ vfs/ (project file system: read/write/revisions/materialize)
│  ├─ ai/ (client.ts, models.ts, prompts/, tools/, plan.ts, build.ts, edit.ts, usage.ts)
│  ├─ codemods/ (AST edit operations for inline editing)
│  ├─ snack/ (toSnackFiles.ts, useSnackSession.ts)
│  ├─ dto.ts (API response shapes) · errors.ts (typed app errors → HTTP)
│  ├─ validate/ (import allowlist, typecheck-lite, route checks)
│  ├─ providers/ (storage/, backend/, images/, build/, hosting/, billing/)
│  ├─ crypto/ (encrypt.ts)
│  ├─ credits/ (ledger.ts, pricing.ts, gate.ts)
│  └─ env.ts (zod-validated env)
├─ worker/
│  ├─ index.ts (pg-boss subscribe all handlers)
│  └─ handlers/ (agent-plan.ts, agent-build.ts, agent-edit.ts, snack-sync.ts, …)
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
| Worker | `npm run worker` (tsx watch) | Railway / Fly / Render long-running service: Node 22+, `eas-cli`, a JDK (`keytool` for Android keystores), Chromium (Playwright smoke tests + thumbnails), cached template `node_modules`, writable temp disk. |
| DB | Docker Postgres or a hosted dev DB | Managed Postgres (Neon, Railway, Supabase, CloudHike …). pg-boss and SSE `LISTEN` need a **direct/session-mode** connection (`DATABASE_URL_DIRECT`); transaction-mode poolers break `LISTEN`. |
| Storage | MinIO (docker) or hosted bucket | S3 / R2 / Railway bucket / CloudHike storage |
| Preview runtime origin | `http://localhost:3000/preview-runtime/` loaded into `sandbox="allow-scripts"` iframes (opaque origin) | Same, or a separate domain like `preview.<brand>.com` for defense-in-depth |

Tunnels such as ngrok are needed **only** for inbound third-party webhooks (Stripe/EAS) and OAuth callbacks when you test them locally. The job system does not need them.

---

## 7. Non-goals for v1

- No real-time multi-user co-editing (sharing is read/edit access, not live collaboration).
- No arbitrary npm dependencies in generated apps. They use the template's allowlist only.
- No custom native modules (anything requiring a dev client instead of Expo Go).
- No in-platform App Store/Play Store submission (we produce AAB/APK/IPA; submission is a stretch goal via `eas submit`).
