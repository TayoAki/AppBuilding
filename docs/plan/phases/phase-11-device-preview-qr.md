# Phase 11 — Device Preview via QR (Expo Snack + Expo Go)

| | |
|---|---|
| Video chapter(s) | Ch.14 "App Preview & Build (QR Scan)" (1:54:57); demo at 5:41 |
| Branch | `feat/11-device-preview-qr` |
| Depends on | Phase 07 (template pinned to a Snack-supported SDK, `materialize()`), Phase 08 (generated apps) |
| Effort (solo dev + Claude Code) | ~2–3 days |
| New env vars | none (`snack-sdk` needs no key for anonymous snacks; add `EXPO_SNACK_*` only if you later save snacks under an owned Expo account) |

## Goal
The workspace header gets a **Scan QR** button. The popover shows a QR code that opens the current project in **Expo Go** on a real phone. It also shows how many devices are connected. When the agent or the user changes a file, connected phones **live-reload without rescanning**. The app runs against whatever backend is configured: local storage by default, the connected backend after Phase 13.

## What we do differently from the video (and why)
- **The SDK is pinned up front** (Phase 07) to a version Snack and current Expo Go both support. The video lost time when generated code targeted SDK 58 while its Snack flow worked only on SDK 54 (~1:57:50). Our template version makes the mismatch impossible by construction.
- **The live session runs in the browser** (snack-sdk works client-side; snack.expo.dev itself does this). The worker job `snack.sync` only **saves** a persistent snapshot for share links. A worker-held session would need a long-lived connection per open workspace and a cleanup story. The browser tab already has the files and the lifetime we want.
- **The QR lives in the header popover**, so the right panel keeps the bigger preview. The video moves it there in this chapter too.
- **Dependencies come from the template's `package.json`**, not from whatever the agent imported. The import allowlist guarantees every import is satisfiable in Expo Go.

> ⚠️ Verify the current `snack-sdk` API (constructor options, `updateFiles`, `updateDependencies`, `setOnline`, `getUrlAsync`, state listener fields such as connected clients, `saveAsync`) against its README at implementation time. The names below follow recent versions but may have changed.

## Steps

### 1. Snack file adapter (`lib/snack/toSnackFiles.ts`)
- `materialize(projectId)` → filter to code and assets → the Snack `files` map (`{ [path]: { type: "CODE", contents } }`; assets as `{ type: "ASSET", contents: <url> }` using signed storage URLs).
- Entry point: Snack + expo-router needs the router entry (e.g. `index.js` with `import "expo-router/entry"`, plus `"main"` in package.json). Confirm the required shape for the pinned SDK on a minimal test snack first.
- `dependencies`: take exact versions from `templates/expo-app/package.json` for every module in `ALLOWED_IMPORTS.json` that the project actually imports (use `lib/validate/imports` to collect them).
- Keep platform-owned files that Snack doesn't need (`eas.json`, build configs) out of the map.

### 2. Browser live session (`lib/snack/useSnackSession.ts`)
- Create a `Snack` instance when the QR popover first opens (lazy; costs nothing for users who never scan). Use `sdkVersion` from `projects.template_version`, the files and dependencies from step 1, and `online: true`.
- Subscribe to the state listener to get the `exp://` URL (render it with `qrcode.react`), connected clients (badge: "iPhone connected" / "2 devices"), and errors and logs from devices. Device runtime errors go into the same "Fix with AI" flow as preview errors (Phase 09).
- Subscribe to the same `file.written` stream the canvas uses. On change, `updateFiles({[path]: …})` with only the changed files, debounced at 300 ms. Deleted files → `null`.
- Theme switch is just a `theme/active.ts` change, so it propagates the same way.
- Tear down when the workspace unmounts. Phones then show "disconnected"; rescanning after reopening works.

### 3. Persistent save (`worker/handlers/snack-sync.ts`)
- `runs.kind = snack_sync`, triggered by **Share to device** (a stable link that works while the workspace tab is closed) and at the end of each successful build/edit run, if the project has been opened in Snack before.
- Worker: `new Snack({...})` → `saveAsync()` → store `{ snackId, url, savedAt, snapshotId }` in `projects.snack` (jsonb). The QR popover shows "Live" vs "Saved version" tabs.

### 4. QR popover UI (`components/workspace/QrPopover.tsx`)
- A header button with a phone icon and a connected-count badge.
- The popover has the QR code, "Open Expo Go → scan" instructions (iOS: Camera app scans `exp://` directly; Android: scan from within Expo Go), a copyable URL, the connected-device list, a Live/Saved toggle, and an SDK note ("Requires Expo Go for SDK N").
- Help link: "Looks different from the preview? Expo Go is the accurate one". Tab bar and safe-area rendering differ slightly from react-native-web.

### 5. Compatibility guard
- `GET /api/projects/:id/snack/compat` (or computed client-side): if `projects.template_version` isn't the current supported template (an old project), show "This project uses SDK X. Upgrade template to preview on device." The template upgrade itself is a later migration task.
- Add a CI check that fails if `templates/expo-app/package.json`'s SDK is not in a hard-coded `SNACK_SUPPORTED_SDKS` list. Update the list when Expo releases.

### 6. Manual device test script
- Document in `docs/plan/10-testing-and-evals.md`: scan with iOS and Android, edit text via inspector → reload in < 3 s, add a screen via chat → appears, switch theme → applies, kill the tab → saved link still opens.

## Files to create / change
```
lib/snack/{toSnackFiles.ts,useSnackSession.ts,compat.ts}
worker/handlers/snack-sync.ts
components/workspace/QrPopover.tsx
components/workspace/Header.tsx                 (+ QR button)
app/api/projects/[projectId]/snack/route.ts     (POST save → runId; GET saved info)
app/api/projects/[projectId]/snack/files/route.ts (GET materialized snack payload for the browser)
lib/db/schema.ts                                (+ projects.snack jsonb)
```

## Database changes
- `projects.snack` (jsonb, nullable): `{ snackId, url, savedAt, snapshotId }`. A small addition to `03-data-model.md` §2.
- `runs.kind = snack_sync` (already listed).

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/projects/:id/snack/files` | viewer | → `{ sdkVersion, files, dependencies }` (materialized; asset URLs signed, ≤ 1 h) |
| POST | `/api/projects/:id/snack` | editor | → `{ runId }` (persistent save job) |
| GET | `/api/projects/:id/snack` | viewer | → `{ saved: { url, savedAt } \| null, compatible: boolean, sdkVersion }` |

## Acceptance checklist
- [ ] Scanning the QR with Expo Go (iOS and Android) opens the generated app. Navigation, add/edit/delete and theme all work.
- [ ] The connected-device badge updates when a phone connects or disconnects.
- [ ] An inline edit or chat edit reloads the app on the phone without rescanning (< 3 s typical).
- [ ] A saved snack link opens on a phone after the workspace tab is closed.
- [ ] A device runtime error shows up in the workspace with **Fix with AI**.
- [ ] The CI SDK-compat check exists and fails on a deliberately unsupported version.
- [ ] **Security:** the materialized Snack payload contains no platform secrets and no `secret_config_enc` values. Only `public_config` (Phase 13) is included. Asset URLs are short-lived signed URLs. Test asserts no env var values appear in the payload.
- [ ] **AuthZ:** `/snack/files` returns 404 for non-members. Saved snack URLs for private projects aren't shown to viewers who lack access to the project.

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 11 (device preview via Expo Snack + QR) only.

Read first:
- docs/plan/06-generated-app-template.md (§1 SDK pinning, §4 allowed imports, §10 materialize)
- docs/plan/02-architecture.md (§4.3 device preview)
- docs/plan/phases/phase-11-device-preview-qr.md (this phase)
- The CURRENT snack-sdk README/typings in node_modules after installing it. Verify every
  method name before using it (constructor options, updateFiles, updateDependencies,
  setOnline, getUrlAsync, state listener fields, saveAsync).

Implement, one commit per phase-doc step:
1. lib/snack/toSnackFiles.ts (materialize → snack files + exact dependency versions from the
   template package.json; expo-router entry). First prove the shape on a minimal snack.
2. Browser live session hook (lazy create, QR from exp:// URL, connected clients, device
   errors → existing Fix with AI flow, incremental updateFiles on file.written, teardown).
3. worker/handlers/snack-sync.ts persistent save → projects.snack.
4. QrPopover in the header (Live/Saved, instructions, copy URL, SDK note).
5. SDK compat guard + CI check.

Rules: the payload must never contain secrets; every route uses requireProjectAccess().
Do NOT build: EAS builds, web publish, export.
Definition of done = Acceptance checklist in the phase doc (document the manual device test
results in the PR description). Finish with lint, typecheck, tests, and a summary.
```

## Pitfalls & notes
- **SDK drift is the number-one failure mode.** Expo Go in the stores supports only the latest SDK (and sometimes one previous). When Expo ships a new SDK, old projects stop opening in Expo Go. Plan a template-upgrade path (new template version + codemods), and show a clear message until it's done.
- **expo-router on Snack** has had quirks (tab bar and layout differences, the entry file shape). Test the pinned SDK with the fixture app before relying on it. The video saw "a little bit odd" tab navigation on Snack that was fine in real builds.
- **Assets:** Snack fetches asset URLs from the device, so signed URLs must still be valid when the phone loads them. Use ≥ 1 h expiry and refresh on reconnect.
- **Rate limits/abuse:** anonymous snacks are public by URL. Don't save snacks for private projects unless the user explicitly clicks "Share to device". Note this in the UI.
- **Backend config (Phase 13):** the Snack gets the same `lib/data/config.ts` as the export, so phones talk to the real backend once it's connected. Only public keys go in it.
- Snack also offers a web player. We deliberately don't use it for the canvas (`07-preview-runtime.md` §2).
