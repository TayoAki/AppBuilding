# Phase 07 — Generated-App Template, Virtual File System & Preview Runtime

| | |
|---|---|
| Video chapter(s) | New (not in video). It is the foundation the video's Ch.11 "Generate Mobile App with AI" (1:11:10) assumes. |
| Branch | `feat/07-template-and-preview-runtime` |
| Depends on | Phase 04 (workspace layout), Phase 05 (jobs & realtime; for `file.written` events) |
| Effort (solo dev + Claude Code) | ~6–8 days. The spike is ~3 days, and this is the riskiest phase. |
| New env vars | `PREVIEW_ORIGIN` (optional; defaults to same-origin `srcdoc` iframes) |

## Goal
Build the three things every later phase stands on:
1. **`templates/expo-app/`**, the fixed Expo template ("the contract") described in `docs/plan/06-generated-app-template.md`.
2. **`lib/vfs`**, the project's virtual file system in Postgres: revisions, path allowlist, `materialize()`.
3. **`preview-runtime/`**, react-native-web in sandboxed iframes, which renders real project files on the canvas and in the interactive preview.

**No AI in this phase.** Everything is proven with a hand-written fixture app, so that when the agent arrives in Phase 08, any failure is the agent's fault and not the runtime's.

## What we do differently from the video (and why)
- The video jumps straight from "workspace placeholder" to "AI generates screens". We first prove rendering, hot swap and element mapping with a fixture. Debugging a new runtime and a new agent at the same time is the slowest way to ship.
- We pin **one Expo SDK** that Expo Snack and the current Expo Go both support, *before* any code is generated. The video discovered the SDK mismatch only when the QR code failed (Ch.14, ~1:57:50).
- The canvas runs our own preview runtime, not one Snack per screen. That gives N frames, instant theme switch and element instrumentation (see `docs/plan/07-preview-runtime.md` §2).
- Platform-owned files (`package.json`, `app.json`, configs, `components/ui`, `lib/data`) are **rendered at materialization** and never stored per project, so one template fix repairs every project.

## Steps

### 1. Pin the Expo SDK and scaffold the template
- Check the Expo Snack supported-SDK list and the latest Expo Go release notes. Choose the newest SDK in both. Record the choice and the date in `templates/expo-app/VERSION` (e.g. `expo-<sdk>@1`) and in `docs/plan/12-decisions-and-open-questions.md`.
- Create a real Expo Router app in `templates/expo-app/` with `npx create-expo-app` at that SDK, then strip it to the layout in `06` §2.
- Turn `package.json`, `app.json`, `eas.json` and `babel.config.js` into templates with `{{appName}}`, `{{slug}}`, `{{bundleId}}`, `{{scheme}}`, `{{version}}`, `{{buildNumber}}` and `{{splashBg}}` placeholders.
- Commit `templates/expo-app/ALLOWED_IMPORTS.json` exactly as in `06` §4.

### 2. Theme system (platform-owned parts)
- `theme/tokens.ts`: `Theme` type, `contrastRatio()`, `ensureContrast(fg, bg, min=4.5)`, `alpha()`.
- `theme/ThemeProvider.tsx`, `theme/useTheme.ts`: reads `ACTIVE_THEME_ID` from `theme/active.ts` and the packs from `theme/themes.ts`. Exposes `useTheme()` returning the active `Theme`.
- A default `theme/themes.ts` with 2 packs (light and dark) so the template runs standalone.

### 3. Data layer (platform-owned parts)
- `lib/data/adapter.ts` (`DataAdapter` interface), `lib/data/local.ts` (AsyncStorage, seeded on first run), `lib/data/config.ts` (`export const adapter = localAdapter`).
- `lib/store/createCollection.ts`: `createCollection<T>(name, { seed })` → `{ useAll, useOne, add, update, remove }`, built on `useSyncExternalStore` plus adapter calls.
- `lib/data/auth.ts`: a fake single-user session for the local adapter (`useSession`, `signIn`, `signUp`, `signOut`, `signInWithOAuth`). Real providers come in Phase 13.

### 4. Component library `components/ui`
- Build the list in `06` §8. Every component uses `useTheme()` only, forwards `testID` and `dataSet`, and has a doc block.
- `scripts/build-component-docs.ts` concatenates the doc blocks into `templates/expo-app/COMPONENT_DOCS.md`. The agent prompt in Phase 08 uses it.
- Check it on a device: `npx expo start` inside the template renders a "kitchen sink" screen on iOS, Android and web.

### 5. Write `CONTRACT.md`
- `templates/expo-app/CONTRACT.md`: folder layout, ownership table, import allowlist, theme rules, data rules, routing conventions, scrolling rule (screens must scroll), and a short example screen. Keep it under ~4k tokens. Check with a token count.

### 6. Fixture app
- `preview-runtime/fixtures/habit-tracker/` is a hand-written app following the contract. It has 3 tab screens (`(tabs)/index`, `(tabs)/progress`, `(tabs)/habits`), 1 stack screen (`habit/[id]`), 1 modal (`add-habit`), `data/types.ts`, `data/seed.ts`, `lib/store/habits.ts`, and `theme/themes.ts` with 5 packs.
- It must also run under real Expo (`materialize` it to a temp dir → `npx expo start`). That proves the fixture is valid Expo, not just valid for our runtime.

### 7. Virtual file system `lib/vfs`
- Drizzle tables `project_files`, `file_revisions`, `screens` (columns per `03` §5).
- `lib/vfs/paths.ts`: `assertWritable(path, author)` enforces the ownership table (`06` §3): POSIX-normalize, reject `..`, absolute paths, NUL bytes and platform-owned prefixes.
- `lib/vfs/files.ts`: `readFile`, `listFiles`, `writeFile({projectId, path, content, author, runId})`. `writeFile` computes sha256, skips no-op writes, inserts a revision and updates `project_files.current_revision_id` in one transaction. Plus `deleteFile` (soft).
- `lib/vfs/materialize.ts`: per `06` §10. Returns `Map<path, string>`, with an optional `writeToDir(dir)` helper for later phases.
- Unit tests: path traversal rejection, ownership rules, revision chain, no-op writes, and that `materialize` output contains rendered `package.json` and project files.

### 8. Preview runtime spike (per `07-preview-runtime.md` §9)
- 8a. `preview-runtime/src/vendor.ts` and `build.mjs` (esbuild) → `public/preview-runtime/vendor.js` exposing `window.__AB_MODULES__`.
- 8b. `loader.ts` + `transpile.ts` (`@babel/standalone`, TS + JSX → CJS). Resolve `./`, `@/`, extensions and index files. Cache by sha.
- 8c. `shims/expo-router.tsx` (memory history; Stack, Tabs, Slot, Link, Redirect, router, useRouter, useLocalSearchParams, useSegments, usePathname) + `routes.ts` (route table from `app/**`).
- 8d. `bridge.ts` (Zod-validated protocol, `07` §4) + `entry.tsx`. Get the fixture rendering in one interactive iframe.
- 8e. Frame mode with `initialRoute`, then a dev page `/dev/preview-lab` showing 4 frames side by side.
- 8f. Hot swap: textarea edit → `file.update` → module invalidation → re-render in < 300 ms.
- 8g. Inspector Babel plugin (`07` §5) adds `dataSet={{afid}}`; on click, log `element.select` to the console.
- **Gate:** all of 8a–8g work in Chrome and Safari. If the router shim isn't viable by day 3, use the fallback in `07` §9 and update `12-decisions-and-open-questions.md`.

### 9. Wire the workspace to real files
- `components/canvas/Canvas.tsx`: pan/zoom canvas from Phase 04. One `<PhoneFrame mode="frame">` per `screens` row at `canvas_x/canvas_y`. Frames outside the viewport show a snapshot instead of a live iframe (`07` §7).
- `components/workspace/InteractivePreview.tsx`: one `mode="interactive"` iframe, with a route selector and a "Reset data" button.
- `components/workspace/CodeView.tsx`: file tree from `listFiles`, read-only Monaco/CodeMirror viewer, copy button. Read-only for now.
- Client store `useProjectFiles(projectId)`: initial fetch of path + sha + revisionId, then fetch contents by revision id (cache forever, keyed by revision). It subscribes to `file.written` run events from Phase 05 and pushes `file.update` into every mounted iframe (coalesced per 150 ms).
- Runtime errors (`runtime.error`) show as a red badge on the frame with the message. "Fix with AI" is added in Phase 09.

### 10. Dev-only fixture seeding
- `scripts/seed-fixture-project.ts --user <email>` creates a `projects` row, writes all fixture files through `lib/vfs` with `author: "system"`, and creates `screens` rows. Guard it with `NODE_ENV !== "production"`.

## Files to create / change
```
templates/expo-app/{VERSION,CONTRACT.md,COMPONENT_DOCS.md,ALLOWED_IMPORTS.json}
templates/expo-app/{package.json,app.json,eas.json,babel.config.js,tsconfig.json,metro.config.js}.hbs
templates/expo-app/{theme,lib/data,lib/store,components/ui}/**
lib/vfs/{paths.ts,files.ts,materialize.ts,index.ts}
lib/db/schema.ts                       (+ project_files, file_revisions, screens)
preview-runtime/src/{vendor.ts,loader.ts,transpile.ts,routes.ts,bridge.ts,entry.tsx}
preview-runtime/src/shims/{expo-router.tsx,expo-image.tsx,expo-linear-gradient.tsx,expo-haptics.ts,expo-constants.ts,async-storage.ts}
preview-runtime/src/inspector-plugin.ts
preview-runtime/fixtures/habit-tracker/**
preview-runtime/build.mjs              → public/preview-runtime/*
components/canvas/{Canvas.tsx,PhoneFrame.tsx}
components/workspace/{InteractivePreview.tsx,CodeView.tsx}
lib/preview/{useProjectFiles.ts,frameBridge.ts}
app/dev/preview-lab/page.tsx           (dev only)
app/api/projects/[projectId]/files/route.ts
app/api/projects/[projectId]/files/[...path]/route.ts
scripts/{seed-fixture-project.ts,build-component-docs.ts}
```

## Database changes
- New tables: `project_files`, `file_revisions`, `screens` (exact columns in `03-data-model.md` §5).
- New column usage: `projects.template_version`.
- Indexes: `file_revisions (project_id, path, created_at desc)`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/projects/:id/files` | `requireProjectAccess(id, "viewer")` | → `{ files: [{ path, sha, revisionId, kind, screenId }] }` |
| GET | `/api/projects/:id/files/:path*?rev=<revisionId>` | viewer | → `text/plain` content; `Cache-Control: private, max-age=31536000, immutable` when `rev` is given |
| GET | `/api/projects/:id/screens` | viewer | → `{ screens: [{ id, name, route, filePath, order, status, canvasX, canvasY }] }` |
| PATCH | `/api/projects/:id/screens/:screenId` | editor | `{ canvasX, canvasY }` → `204` (persist frame drag) |

No write API for file contents yet. Writes come from the agent (Phase 08), inline edits (Phase 10) and the theme switch (Phase 08).

## Acceptance checklist
- [ ] `templates/expo-app` materialized with sample metadata runs with `npx expo start` on iOS (Expo Go), Android (Expo Go) and web.
- [ ] The pinned SDK is documented, with the date checked against the Snack and Expo Go support lists.
- [ ] `CONTRACT.md` is ≤ ~4k tokens.
- [ ] `lib/vfs` unit tests pass: `../`, absolute paths, `package.json`, `components/ui/x.tsx` and `lib/data/x.ts` writes are rejected for `author: "agent"`.
- [ ] The fixture seeded via the script shows 5 frames on the canvas, an interactive preview where navigation, add habit and toggle-complete work, and a Code tab listing every file.
- [ ] Editing a fixture file in the DB (e.g. via the script with a changed string) and emitting `file.written` updates the frames in < 1 s without a reload.
- [ ] Clicking an element in the dev preview lab logs an `afid` that maps to the correct `path:line:col`.
- [ ] **Security:** iframes have `sandbox="allow-scripts"` and no `allow-same-origin`. Code in the fixture cannot read `document.cookie` of the parent or `fetch('/api/projects')` with credentials (add a test that asserts this).
- [ ] **AuthZ:** file and screen routes return 404 for a project the user isn't a member of (test with two users).
- [ ] Frames outside the viewport are not live iframes. With 10 frames, the canvas stays at 60 fps panning on a mid-range laptop.

## Claude Code prompt
```text
We are building AppBuilding, an AI mobile-app builder. Implement Phase 07 only.

Read first, in order:
- docs/plan/02-architecture.md (§3 decisions A4–A7, §5 repo layout)
- docs/plan/03-data-model.md (§5 virtual file system)
- docs/plan/06-generated-app-template.md (entire file — this is the contract)
- docs/plan/07-preview-runtime.md (entire file)
- docs/plan/phases/phase-07-template-and-preview-runtime.md (this phase)

Scope, in this order, one commit per numbered step of the phase doc:
1. templates/expo-app pinned to ONE Expo SDK that Expo Snack and current Expo Go
   both support. Tell me which SDK you chose and why before scaffolding.
2. Theme tokens/ThemeProvider, lib/data adapter + LocalAdapter + createCollection,
   components/ui library, CONTRACT.md (<4k tokens).
3. Hand-written fixture app in preview-runtime/fixtures/habit-tracker.
4. lib/vfs (paths allowlist, files with revisions, materialize) + Drizzle tables
   project_files, file_revisions, screens + unit tests.
5. Preview runtime spike (steps 8a–8g). STOP and report after 8d if the router shim
   looks infeasible — don't silently switch approaches.
6. Wire canvas frames, interactive preview and read-only Code tab to real project
   files; subscribe to file.written events from lib/runs.
7. scripts/seed-fixture-project.ts (dev only).

Rules:
- NO AI/LLM calls in this phase. No agent, no prompts.
- Preview iframes: sandbox="allow-scripts" only (no allow-same-origin). Validate every
  postMessage with Zod and check event.source.
- Every API route uses requireProjectAccess().
- Platform-owned template files are rendered at materialization, never stored per project.
Do NOT build: theme switching UI, inline editing UI, Snack/QR, export.

Definition of done = the Acceptance checklist in the phase doc.
Finish by running lint, typecheck and tests. Summarize what works, what doesn't, and
any decision you had to make that should go into docs/plan/12-decisions-and-open-questions.md.
```

## Pitfalls & notes
- **react-native-web version must match** the React Native version the template pins. Build the vendor bundle from the template's own `package.json` versions so the preview and the real app don't drift.
- `@babel/standalone` is ~2 MB. Load it once in the vendor bundle and rely on HTTP caching. Transpile only changed files (sha cache).
- Expo Router conventions to cover in `routes.ts`: `(group)` folders don't add path segments, `[param]` and `[...rest]`, `_layout.tsx` nesting, `index.tsx`, and `+not-found.tsx`. Write table-driven tests.
- `@expo/vector-icons` on web needs the font files. Ship them with the vendor bundle and register `@font-face` in the runtime HTML.
- AsyncStorage shim keys must be namespaced by `projectId + frameId`, or one frame's data leaks into another. Opaque-origin iframes can't use `localStorage` at all (it throws), so implement the shim as in-memory with the parent persisting a snapshot via `postMessage` if you want data to survive reloads.
- Safari is the usual troublemaker (iframe transforms, `ResizeObserver` timing). Test early.
- Keep the expo-router shim deliberately small. A pattern the shim can't support gets banned in `CONTRACT.md`, not added to the shim.
