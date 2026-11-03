# Phase 11 — Device Preview via QR (Expo Go)

| | |
|---|---|
| Video chapter(s) | Ch.14 "App Preview & Build (QR Scan)" (1:54:57); demo at 5:41. Reference repo: Prompts 9.1–9.4 |
| Branch | `feat/11-device-preview-qr` |
| Depends on | Phase 07 (template on SDK 58, `materialize()`), Phase 08 (generated apps, sandbox service), Phase 10 (inline edits arrive as revisions), Phase 00b S3 (connection mechanism chosen) |
| Effort (solo dev + Claude Code) | ~4–6 days (Path B hosted dev server + Path C fallback; Path A Snack adds ~1–2 days if/when enabled) |
| New env vars | `DEVICE_PREVIEW_MODE` (`devserver` \| `snack`, default `devserver`), `DEVPREVIEW_BASE_DOMAIN` (e.g. `devpreview.<brand>.com`), plus the sandbox service vars from Phase 08 (`SANDBOX_URL`, `SANDBOX_SHARED_SECRET`) |

## Goal
The App Preview header gets a **Scan QR Code** button. Its popover shows a QR code that opens the current project in **Expo Go** on a real phone, plus connection status. When the agent, the inline editor, the code editor or a theme switch changes anything, connected phones **live-reload without rescanning**. If Expo Go can't open the app, **Open in browser** shows it in the phone's browser. Expo Go is always offered first, for native features. The app runs against whatever backend is configured: device storage by default, the connected backend after Phase 13.

## What we do differently from the video (and why)
- **Three paths, chosen by verified facts (`15` §1, ADR-002), not by assumption:**

| Path | When | How | Fidelity |
|---|---|---|---|
| **B: hosted Expo dev server** (default) | Always works with store Expo Go (SDK 58) | Sandbox service runs `expo start` for the project; Expo Go connects to it; Metro Fast Refresh pushes edits | Highest: the real project with real expo-router, no shim |
| **A: Expo Snack** | Only when a **published** `snack-sdk` supports the template SDK (not true on 2026-11-03: published ≤ 54, main ≤ 56) | Live Snack session from the browser tab; React Navigation shim | Good; cheaper (no server per session) |
| **C: browser fallback** | Always available | Signed link to our sandboxed preview page | Web only, no native APIs |

- **The SDK isn't pinned backwards for Snack's sake.** The video and reference app pinned SDK 54 for Snack, which store Expo Go (SDK 58 only) can no longer open, and iOS users can't install an older Expo Go.
- **The QR lives in a popover at the top of App Preview**, so the preview keeps the bigger area (as the video moves it in this chapter).

### Reference app notes
- **R11:** the reference app ran Snack on SDK 54 with a React Navigation shim (relative imports, `App.js` entry, `expo-router` shim). We keep that design as **Path A**, enabled only when Snack catches up with Expo Go.
- **R11:** live updates from the browser tab. In Path B, the tab pushes changes to the dev server instead.
- **R7:** the shared tab layout fixes "each tab looks like a new page" on devices (Prompt 9.4). Path B uses the real `app/(tabs)/_layout.tsx`.
- Prompt 9.3: when Expo Go fails, offer the web fallback, but keep Expo Go first.

## Steps

### 1. Dev-server materialization target (`lib/vfs/targets/devserver.ts`)
- `materialize(projectId, { target: "devserver" })` produces the **full Expo project**: the same as the `export` target (`06` §10), with real `app/` routes, `package.json` with SDK 58 versions from `bundledNativeModules.json`, a static `app.json` and the backend config. No transforms.
- It's packed as a tarball uploaded to storage. The sandbox gets a signed GET URL (it has no DB or storage credentials, ADR-003 #18).

### 2. Sandbox: dev server sessions (`sandbox/devserver/*`)
- `POST /sessions` (HMAC-signed by the worker with `SANDBOX_SHARED_SECRET`, per-job token): `{ sessionId, tarballUrl, templateVersion }` →
  - unpack to `/sessions/<id>/`; link the **prebuilt `node_modules`** for the template version (read-only cache; never `npm install` agent-influenced deps);
  - start `npx expo start --port <p>` as an unprivileged user with an env allowlist. **Don't set `CI=1`**: in CI mode Metro disables reloads (verify in S3);
  - expose it to phones with the mechanism S3 chose (record it in an ADR):
    - **B1 (default):** a reverse-proxy subdomain `d-<sessionId>.<DEVPREVIEW_BASE_DOMAIN>` → Metro port, with `EXPO_PACKAGER_PROXY_URL=https://d-<sessionId>.<…>` so the manifest's bundle URLs point at the proxy. QR = the `exp://`/`exps://` URL S3 confirmed Expo Go accepts;
    - **B2:** `expo start --tunnel` (Expo's ngrok-based tunnel). It's simpler but depends on a third party;
  - respond `{ url: "exp://…", expiresAt }`.
- `PUT /sessions/:id/files` `{ files: [{path, content|null}] }`: writes only changed files. Metro's watcher triggers **Fast Refresh** on connected Expo Go clients.
- `GET /sessions/:id/status`: `{ running, devices: n | null, lastLogs[] }`. Devices are counted from proxy websocket connections (HMR/message sockets). Device logs and errors come from Metro output (`client_log`). Verify both in S3; if either is unavailable, hide that indicator.
- `DELETE /sessions/:id`: kills the process and deletes the dir.
- **Lifecycle:** stop when there's been no workspace heartbeat for 5 min **and** no device connected for 5 min, or on hard max (Free 30 min, Pro 2 h, renewable from the UI). Caps: about 4 sessions per 4 GB sandbox instance. Per user: Free 1 concurrent, Pro 2 concurrent. Over capacity → "Preview servers are busy, try again in a minute," with the browser fallback offered.

### 3. Platform orchestration
- `POST /api/projects/:id/device-preview` (viewer): plan gate + caps → insert `device_sessions` row → worker job `devpreview.start` (single-shot wrapper) → materialize → sandbox `POST /sessions` → `{ sessionId, url }` back to the client via the project event stream (`device.ready`).
- The workspace sends `POST /api/projects/:id/device-preview/:sid/heartbeat` every 60 s while the popover or workspace is open.
- **Live updates:** the workspace already receives `file.written` (agent, `user_inline`, `user_code`) and `plan.updated` (themes, layouts, schema) on the **project-scoped stream** (ADR-003 #5). A small worker job `devpreview.sync` (debounced 300 ms) re-renders platform files on `plan.updated`, collects changed paths and calls `PUT /sessions/:id/files`. Theme switches therefore reach phones as a `theme/themes.ts` change.

### 4. Path A: Snack (feature flag `DEVICE_PREVIEW_MODE=snack`, only if supported)
- At boot, read the installed `snack-sdk`'s supported SDK list. If the template SDK isn't in it, Path A is **disabled** with a log line, and Path B is used.
- `lib/vfs/targets/snack.ts` (the reference app's design): `@/…` imports → relative paths (Babel, not regex); an `App.js` entry with a React Navigation stack + bottom tabs from the plan (`initialRouteName = firstScreenId`, reproducing the root layout's providers and screen options); an `expo-router` shim (`router.push/replace/back`, `useLocalSearchParams`, `Link`, `Redirect`) with no animation for `replace` between tabs; exact dependency versions for that SDK.
- Browser session (`hooks/use-expo-snack.ts`): `new Snack({...})` → `setOnline(true)` → the URL from **`state.url`** (there's no `getUrlAsync`); devices from **`state.connectedClients`**; logs via `addLogListener`; incremental `updateFiles` on `file.written` / `plan.updated`.
- **No auto-save for private projects.** `saveAsync()` only when the user clicks **Share to device**, which warns that saved snacks are public by URL. Result in `projects.snack`.

### 5. Path C: browser fallback
- `POST /api/projects/:id/preview-link` → a signed, short-lived (≤ 1 h) URL to `app/(public)/preview/[projectId]`, which renders the app with our sandboxed runtime (`07`) in interactive mode. It's shown as a second QR and link in the popover.

### 6. QR popover UI (`components/workspace/QrPopover.tsx`)
- The **Scan QR Code** button sits at the top of App Preview, with a status badge (starting / ready / N devices / stopped / error).
- The popover has:
  - the Expo Go QR;
  - instructions: on iOS, the Camera app scans the code; on Android, scan from Expo Go. "Requires Expo Go from the App Store / Play Store (SDK 58)";
  - a copy-URL button and a time-left indicator with **Keep alive**;
  - **Open in browser instead** (Path C QR);
  - device errors and logs with **Fix with AI** (Phase 09).
- Help text: "Looks different from the preview? Expo Go is the accurate one."

### 7. Manual device test script
Add to `10-testing-and-evals.md`, and run it on one iPhone and one Android phone:
- scan → app opens;
- inline edit → phone updates in < 3 s;
- add a tab via chat → the bar updates;
- switch theme → applies;
- device error → Fix with AI;
- idle 10 min → session stops cleanly;
- Path C link works and expires.

## Files to create / change
```
lib/vfs/targets/devserver.ts
lib/vfs/targets/snack.ts, lib/vfs/targets/snack/{app-entry.ts,expo-router-shim.js,deps.ts}   (Path A, flagged)
worker/handlers/{devpreview-start.ts,devpreview-sync.ts,devpreview-stop.ts}
sandbox/devserver/{server.ts,sessions.ts,proxy.ts,cache.ts}
hooks/use-device-preview.ts, hooks/use-expo-snack.ts (Path A)
components/workspace/QrPopover.tsx, components/workspace/AppPreviewHeader.tsx
app/api/projects/[projectId]/device-preview/route.ts            (POST start, GET status)
app/api/projects/[projectId]/device-preview/[sid]/heartbeat/route.ts
app/api/projects/[projectId]/preview-link/route.ts
app/(public)/preview/[projectId]/page.tsx                        (Path C)
```

## Database changes
- New `device_sessions` (`id` `dev_…`, `project_id`, `user_id`, `mode` (`devserver`/`snack`), `url`, `status` (`starting`/`ready`/`stopped`/`failed`), `started_at`, `last_heartbeat_at`, `expires_at`, `ended_at`). Index `(user_id, status)` for per-user caps. **Add to `03`.**
- `projects.snack` (jsonb) is used only by Path A saves.
- `runs.kind`: `devpreview` is unnecessary. The start/sync jobs are short single-shot jobs tracked in `device_sessions`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/device-preview` | viewer | → `202 { sessionId }`, then `device.ready {url}` on the project stream · `429` over caps |
| GET | `/api/projects/:id/device-preview` | viewer | → `{ session: {status, url, devices, expiresAt} \| null, mode }` |
| POST | `/api/projects/:id/device-preview/:sid/heartbeat` | viewer (session owner) | → `204` |
| DELETE | `/api/projects/:id/device-preview/:sid` | viewer (session owner) | → `204` |
| POST | `/api/projects/:id/preview-link` | viewer | → `{ url, expiresAt }` (≤ 1 h) |
| POST | `/api/projects/:id/snack` | editor | Path A only → `{ runId }` (explicit public save) |
| Sandbox (internal) | `POST /sessions`, `PUT /sessions/:id/files`, `GET /sessions/:id/status`, `DELETE /sessions/:id` | HMAC per-job token | as in step 2 |

## Acceptance checklist
- [ ] Scanning the QR with **store Expo Go (SDK 58)** on iOS and Android opens the generated app. Navigation, the shared tab bar, add/edit/delete and theme all work.
- [ ] Inline edits, code saves, chat changes and theme switches reload the phone app without rescanning (< 3 s typical, measured).
- [ ] Adding a tab via chat updates the phone's tab bar.
- [ ] Sessions stop after the idle rules and at the hard max; per-user and per-instance caps return `429` with the browser fallback offered.
- [ ] **Open in browser instead** works on a phone, and the link expires.
- [ ] A device runtime error appears in the workspace with **Fix with AI** (or the indicator is hidden if S3 showed logs aren't available).
- [ ] With `DEVICE_PREVIEW_MODE=snack` and an unsupported SDK, Path A is disabled automatically and Path B is used.
- [ ] **Security:**
  - The sandbox has no DB or storage credentials and no platform env vars (a test asserts its env allowlist).
  - The tarball and session contain only the public backend config.
  - Session URLs are unguessable and die with the session.
  - Private projects are never saved to Snack without an explicit, warned click.
- [ ] **AuthZ:** every route uses `requireProjectAccess(viewer)`. Heartbeat/stop only work for the session owner. Non-members get `404`.

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 11 (device preview via Expo Go) only.

Read first:
- docs/plan/15-vendor-facts.md §1 (SDK 58, Expo Go, snack-sdk facts)
- docs/plan/12-decisions-and-open-questions.md ADR-002 and ADR-003 (#5 project stream, #18 sandbox)
- docs/plan/06-generated-app-template.md (§1, §10 materialize targets)
- docs/plan/07-preview-runtime.md (Path C page uses the sandboxed runtime)
- docs/plan/phases/phase-00b-derisking-spikes.md S3 findings/ADR (the chosen B1/B2 mechanism)
- docs/plan/phases/phase-11-device-preview-qr.md (this phase)

Implement, one commit per step in the phase doc:
1. materialize target "devserver" (full Expo project tarball → storage, signed URL).
2. sandbox/devserver: sessions API (HMAC per-job token), prebuilt node_modules cache, expo start
   WITHOUT CI=1, the S3-chosen exposure mechanism, file sync → Fast Refresh, status/devices/logs,
   idle + hard-max lifecycle, capacity caps. The sandbox has no secrets and no DB access.
3. Platform: device_sessions table, start/heartbeat/stop routes, devpreview.start/sync jobs fed by
   the project-scoped event stream (file.written, plan.updated), device.ready event.
4. Path C signed preview link + public preview page.
5. QrPopover UI with status, keep-alive, browser fallback, Fix with AI for device errors.
6. Path A (Snack) behind DEVICE_PREVIEW_MODE=snack, auto-disabled when the published snack-sdk
   doesn't support the template SDK (state.url, state.connectedClients; no getUrlAsync).
Verify every CLI flag and SDK method against the installed versions. Don't guess.
Do NOT build: EAS builds, web publish, export.
Definition of done = Acceptance checklist (attach the manual device test results to the PR).
Finish with lint, typecheck, tests, and a summary.
```

## Pitfalls & notes
- **SDK drift is the number-one failure mode.** Expo Go supports one SDK at a time. When Expo ships SDK 59, store Expo Go moves and every project must follow: template upgrade (Phase 19b, `06` §11). Until then, the popover says "Upgrade this app to preview on the current Expo Go" and offers Path C.
- **Don't set `CI=1`** for the dev server. It disables Metro reloads, so phones stop updating (verify in S3).
- **Prebuilt `node_modules` per template version** keeps session start fast (target < 20 s to a QR) and prevents installing anything the agent could influence.
- **Metro memory** is roughly 300–600 MB per session. Size the sandbox and caps from S3 measurements.
- **Session URLs are capabilities.** Expo Go can't send auth headers, so the URL itself grants access. Keep it random, short-lived and session-scoped, and never log it in client-visible places.
- **Backend config (Phase 13):** the tarball gets the same `lib/backend/config.ts` as the export, so phones talk to the real backend (or the platform gateway) once it's connected. Only public config is included.
- **Path A, if enabled later:** keep the router shim in sync with the router API the generator rules allow (`05` §5), and anonymous saved snacks are public by URL.
