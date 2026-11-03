# Phase 11 — Device Preview via QR (Expo Snack + Expo Go)

| | |
|---|---|
| Video chapter(s) | Ch.14 "App Preview & Build (QR Scan)" (1:54:57); demo at 5:41. Reference repo: Prompts 9.1–9.4 |
| Branch | `feat/11-device-preview-qr` |
| Depends on | Phase 07 (template pinned to a Snack-supported SDK, `materialize()`), Phase 08 (generated apps), Phase 10 (inline edits arrive as revisions) |
| Effort (solo dev + Claude Code) | ~3–4 days (was 2–3; the Snack materialize target with an expo-router shim on React Navigation adds about a day) |
| New env vars | none (anonymous snacks need no key) |

## Goal
The App Preview header gets a **Scan QR Code** button. Its popover shows a QR code that opens the current project in **Expo Go** on a real phone, plus connection status. When the agent, the inline editor or the user changes anything, connected phones **live-reload without rescanning**. If Expo Go can't open the app, an **Open in browser** fallback shows the app on the phone's browser. Expo Go is always offered first, for native features. The app runs against whatever backend is configured: local device storage by default, the connected backend after Phase 13.

## What we do differently from the video (and why)
- **The SDK is pinned up front** (Phase 07, D7) to what the published `snack-sdk` and store Expo Go both support. The video lost time when generated code targeted a newer SDK than the Snack flow supported (~1:57:50).
- **A dedicated `snack` materialization target** (`06` §10) adapts the same project files for Snack. Screens, components and the web preview don't change.
- **The QR lives in a popover at the top of App Preview**, so the preview gets the bigger area. The video moves it there in this chapter too.

### Reference app notes
- **R11:** the reference app pins **SDK 54**, because the published `snack-sdk` didn't go past 54 and newer Expo Go builds refuse older SDKs. Re-check both at implementation time and record the result in D7.
- **R11:** Snack has no path aliases and runs generated apps more reliably on **React Navigation** than on expo-router. So the snack target rewrites `@/…` imports to relative paths, adds an `App.js` entry that replaces `app/_layout.tsx`, and turns `expo-router` into a small shim on React Navigation, with `@react-navigation/bottom-tabs` for tab apps. `router.replace` between tabs doesn't animate (a slide made each tab look like a new page on a real device).
- **R11:** the live session runs in the browser tab and stays online while the tab is open. Every code change, including inline edits, goes straight to connected phones.
- **R7:** the shared tab layout fixes the "each tab looks like a new page" problem the reference app saw on devices (Prompt 9.4).
- Prompt 9.3: when Expo Go fails, offer the web fallback, but keep Expo Go as the first choice.

> ⚠️ Verify the current `snack-sdk` API against its README and typings at implementation time: constructor options, `updateFiles`, `updateDependencies`, `setOnline`, `getUrlAsync`, the state listener (connected clients, errors), `saveAsync`.

## Steps

### 1. Snack target in `materialize()` (`lib/vfs/targets/snack.ts`)
- Start from `materialize(projectId, {target: "snack"})` (`06` §10): platform files rendered from `projects.plan` plus project files.
- Transforms:
  - **Imports:** rewrite `@/theme`, `@/lib/backend`, `@/data/schema` and `@/components/<Name>` to relative paths (parse with Babel; don't regex).
  - **Entry:** generate `App.js` that wraps providers (SafeAreaProvider, backend provider) around a React Navigation container built from the plan. A stack for the stack screens, and a bottom-tab navigator for screens with `tab` (labels and Ionicons from `screen.tab`), with `initialRouteName` = `firstScreenId`. Drop `app/_layout.tsx` and `app/(tabs)/_layout.tsx` from the payload.
  - **expo-router shim** (`expo-router.js` in the payload): `router.push/replace/back`, `useRouter`, `useLocalSearchParams`, `Link`, `Redirect`, `Stack.Screen` (options no-op), mapped onto React Navigation route names derived from `screenRoute(id)`. `replace` to a tab → `navigation.navigate(tab)` without animation.
- `dependencies`: exact versions for the pinned SDK (from `bundledNativeModules.json`) for every allowlisted module the project imports, plus `@react-navigation/native`, `@react-navigation/native-stack` and `@react-navigation/bottom-tabs`, and their peers.
- Assets as `{ type: "ASSET", contents: <signed URL> }`. Leave out config files Snack doesn't need (`eas.json`).
- Unit-test the transform on the fixture app and on a generated tab app. Prove the shape on a minimal snack first.

### 2. Browser live session (`hooks/use-expo-snack.ts`)
- Create a `Snack` lazily when the popover first opens, with `sdkVersion` from the template, files and dependencies from `GET /snack/files`, and `online: true`.
- From the state listener: the `exp://` URL → QR (`qrcode.react`), the connected-client count ("iPhone connected" / "2 devices", if the SDK exposes it; Q1), and device errors and logs. Device runtime errors join the same **Fix with AI** flow as preview errors (Phase 09).
- Subscribe to the workspace's `file.written` stream (agent revisions **and** `user_inline` revisions). Re-run the snack transform for changed files only (an import rewrite needs the file list) and `updateFiles({[path]: …})`, debounced at 300 ms. Deleted files → `null`. A plan change (added screen or tab) regenerates `App.js` too.
- Theme switch: `activeThemeId` changes in `theme/themes.ts`, which propagates the same way.
- Tear down when the workspace unmounts. Phones then show "disconnected", and rescanning after reopening works.

### 3. Persistent save (`worker/handlers/snack-sync.ts`)
- `runs.kind = snack_sync`, triggered by **Share to device** (a stable link that works while the workspace tab is closed). Optionally also at the end of successful build/change runs, if the project was opened in Snack before.
- Worker: `materialize({target:"snack"})` → `new Snack({...})` → `saveAsync()` → `projects.snack = { snackId, url, savedAt, snapshotId }`. The popover shows "Live" and "Saved version" tabs.

### 4. QR popover UI (`components/workspace/QrPopover.tsx`)
- A **Scan QR Code** button at the top of the App Preview section, with a connected badge.
- The popover has: the QR; instructions (iOS: Camera app scans `exp://`; Android: scan from Expo Go); a copy URL button; status (connecting / online / N devices / error); Live/Saved; the SDK note "Requires Expo Go for SDK `<pinned>`"; and **Open in browser instead**. That's a QR or link to a short-lived signed URL of the project's preview page, rendered by our sandboxed runtime, for phones where Expo Go fails.
- Help text: "Looks different from the preview? Expo Go is the accurate one."

### 5. Compatibility guard
- `GET /api/projects/:id/snack` returns `compatible: false` when `projects.template_version` targets an SDK the current Snack/Expo Go no longer support. The popover then says "This project uses SDK X; upgrade the template to preview on device." The upgrade is a separate migration task.
- CI check: fail if `templates/expo-app/package.json`'s SDK isn't in `SNACK_SUPPORTED_SDKS` (a constant updated when Expo releases).

### 6. Manual device test script
- Add to `10-testing-and-evals.md`:
  - scan with iOS and Android;
  - inline edit → phone updates in < 3 s;
  - add a tab via chat → the bar updates;
  - switch theme → applies;
  - device error → Fix with AI;
  - close the tab → the saved link still opens;
  - force an Expo Go failure → the browser fallback opens.

## Files to create / change
```
lib/vfs/targets/snack.ts                    (imports → relative, App.js, router shim, deps)
lib/vfs/targets/snack/{app-entry.ts,expo-router-shim.js,deps.ts}
hooks/use-expo-snack.ts
worker/handlers/snack-sync.ts
components/workspace/QrPopover.tsx
components/workspace/AppPreviewHeader.tsx   (+ Scan QR Code button)
app/api/projects/[projectId]/snack/route.ts       (POST save → runId; GET saved info + compat)
app/api/projects/[projectId]/snack/files/route.ts (GET snack-target payload for the browser)
app/(public)/preview/[projectId]/page.tsx         (signed-token browser fallback, sandboxed runtime)
```

## Database changes
- `projects.snack` (jsonb, nullable): `{ snackId, url, savedAt, snapshotId }` (already in `03` §2).
- `runs.kind = snack_sync` (already listed).

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/projects/:id/snack/files` | viewer | → `{ sdkVersion, files, dependencies }` (snack target; asset URLs signed ≥ 1 h) |
| POST | `/api/projects/:id/snack` | editor | → `{ runId }` (persistent save job) |
| GET | `/api/projects/:id/snack` | viewer | → `{ saved: { url, savedAt } \| null, compatible, sdkVersion }` |
| POST | `/api/projects/:id/preview-link` | viewer | → `{ url, expiresAt }` (signed browser-fallback link, ≤ 1 h) |

## Acceptance checklist
- [ ] Scanning the QR with Expo Go (iOS and Android) opens the generated app. Navigation, tabs (one shared bar, no slide between tabs), add/edit/delete and theme all work.
- [ ] Screens, components and the web preview are unchanged by the snack transform (diff the project files: none modified).
- [ ] An inline edit, a chat change or a theme switch reloads the app on the phone without rescanning (< 3 s typical).
- [ ] Adding a tab via chat updates the phone's tab bar.
- [ ] The connection status updates when a phone connects or disconnects (or the indicator is hidden if the SDK can't report it; Q1).
- [ ] A saved snack link opens on a phone after the workspace tab is closed.
- [ ] **Open in browser instead** shows the app on a phone browser through the sandboxed runtime, and the link expires.
- [ ] A device runtime error shows up in the workspace with **Fix with AI**.
- [ ] The CI SDK-compat check exists and fails on a deliberately unsupported version.
- [ ] **Security:** the snack payload contains no platform secrets and no `secret_config_enc` values. Only the backend's public config (Phase 13) is included. A test asserts no env var values appear in the payload. Asset URLs are short-lived signed URLs.
- [ ] **AuthZ:** `/snack/files` and `/preview-link` return 404 for non-members. Saved snack URLs for private projects aren't shown to users without access.

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 11 (device preview via Expo Snack + QR) only.

Read first:
- docs/plan/06-generated-app-template.md (§1 SDK pinning, §2 layout, §4 allowed imports,
  §10 materialize targets — the snack target)
- docs/plan/13-reference-implementation.md (R7, R11)
- docs/plan/02-architecture.md (§4.3 device preview: live session in the browser)
- docs/plan/phases/phase-11-device-preview-qr.md (this phase)
- The CURRENT snack-sdk README/typings after installing it. Verify every method name.

Implement, one commit per phase-doc step:
1. lib/vfs/targets/snack.ts: @/ imports → relative (Babel, not regex), App.js entry with a
   React Navigation stack + bottom tabs from the plan (initialRouteName = firstScreenId),
   expo-router shim on React Navigation (no animation for replace between tabs), exact
   dependency versions for the pinned SDK. Unit tests on the fixture + a generated tab app.
   Prove the shape on a minimal snack first.
2. hooks/use-expo-snack.ts: lazy session, QR from exp:// URL, status/devices, device errors →
   Fix with AI, incremental updateFiles on every file.written (agent + user_inline), teardown.
3. worker/handlers/snack-sync.ts persistent save → projects.snack.
4. QrPopover at the top of App Preview (Live/Saved, instructions, copy URL, SDK note,
   "Open in browser instead" signed fallback link).
5. SDK compat guard + CI check.

Rules: never modify project files for Snack (transform at materialization only); the payload
must never contain secrets; every route uses requireProjectAccess().
Do NOT build: EAS builds, web publish, export.
Definition of done = Acceptance checklist in the phase doc (put the manual device test
results in the PR description). Finish with lint, typecheck, tests, and a summary.
```

## Pitfalls & notes
- **SDK drift is the number-one failure mode.** Store Expo Go supports only the latest SDK (sometimes one previous), and `snack-sdk` may lag behind Expo releases. When they diverge, old projects stop opening. Plan a template-upgrade path (new template version + codemods), and show a clear message until it's done.
- **Shim surface = contract surface.** The expo-router shim only needs the router API the generator rules allow (`05` §5: `router.push({pathname, params})`, `replace`, `back`, `useLocalSearchParams`, `Link`). Keep the validator and the shim in sync. If the Screen Builder starts using new router APIs, the validator should reject them.
- **Assets:** Snack fetches asset URLs from the device, so signed URLs must still be valid when the phone loads them. Use ≥ 1 h expiry and refresh on reconnect.
- **Anonymous snacks are public by URL.** Don't save snacks for private projects unless the user clicks **Share to device**, and say so in the UI.
- **Backend config (Phase 13):** the snack gets the same `lib/backend/config.ts` as the export, so phones talk to the real backend (or the platform gateway) once it's connected. Only public config is included.
- **Dev tunnels:** the reference app needed ngrok for its job service, which broke Next.js HMR websockets. Our worker doesn't need a tunnel, and Snack sessions run from the browser, so local dev stays on localhost.
- Snack also offers a web player. We don't use it for the canvas (`07` §2).
