# Phase 07 — Generated-App Template, Virtual File System & Preview Runtime

| | |
|---|---|
| Video chapter(s) | New (not in video). It's the foundation the video's Ch.11 "Generate Mobile App with AI" (1:11:10) assumes; reference repo Ch.6 + Ch.9 runtime notes |
| Branch | `feat/07-template-and-preview-runtime` |
| Depends on | Phase 00b (G0 passed; S1 runtime code + fixture already in `preview-runtime/`), Phase 04 (workspace layout, `screens` table), Phase 05 (project event stream, `file.written` / `plan.updated`) |
| Effort (solo dev + Claude Code) | ~3–5 days. The ~3-day runtime spike now runs earlier, as S1 in Phase 00b, and its fixture + findings are reused here. |
| New env vars | `PREVIEW_ORIGIN` (optional; defaults to sandboxed `srcdoc` iframes) |

## Goal
Build the three things every later phase stands on:
1. **`templates/expo-app/`**, the fixed Expo template described in `06-generated-app-template.md`. That covers the platform-owned `@/theme` and `@/lib/backend`, plus renderers that turn `projects.plan` (themes, collections, navigation) into `theme/themes.ts`, `data/schema.ts`, `data/seed.ts`, the layouts and `lib/backend/config.ts`.
2. **`lib/vfs`**, the project's virtual file system in Postgres: revisions, the ownership allowlist, and `materialize()` with targets.
3. **`preview-runtime/`**, react-native-web in sandboxed iframes, rendering real project files on the canvas and in the interactive preview, with **CSS-variable theme switching**.

**No AI in this phase.** Everything is proven with a hand-written fixture app (a fixture `plan.json` plus hand-written screens and components), so any failure in Phase 08 is the agent's, not the runtime's.

## What we do differently from the video (and why)
- The video jumps from "workspace placeholder" to "AI generates screens". We first prove rendering, hot swap, theme switching and element mapping with a fixture.
- We pin **one Expo SDK** before any code is generated. The video found the SDK mismatch only when the QR code failed (Ch.14, ~1:57:50).
- The canvas runs our own preview runtime, not one Snack per screen (`07-preview-runtime.md` §2).
- Everything derived from data is **rendered by the platform**, never stored per project and never written by the agent: themes, schema, seed, layouts, backend config and all configs.

**Reference app notes** (`13-reference-implementation.md`)
- R5: screens use generic hooks from `@/lib/backend` (`useCollection`, `useStoredValue`, `useAuth`, `uploadFile`) over a `BackendProvider` interface. The local provider is the default and in the preview keeps data in the workspace. Connecting a backend later changes no screen.
- R6/R7: no fixed UI library. Each app gets its own `components/<Name>.tsx`, and the tab bar is one platform-generated `app/(tabs)/_layout.tsx`.
- R8: `import { colors } from "@/theme"` works inside `StyleSheet.create`. In the preview the tokens are CSS variables, so a theme switch only sets variables.
- R11 + `15` §1: the reference app pinned SDK 54 because the published `snack-sdk` stops there. Store Expo Go now runs **only SDK 58**, so we pin **SDK 58** (ADR-002) and phones use the hosted dev server (Phase 11 Path B), not Snack.
- R9: generated markup is one element per line. The fixture follows that, so the inspector mapping is tested against realistic code.

## Steps

### 1. Pin the Expo SDK and scaffold the template
- **Pin SDK 58** (ADR-002: the SDK store Expo Go supports; confirm on the day, as in S0). Write `templates/expo-app/VERSION` = `expo-58@1` and record the date in D7. Record the published `snack-sdk` ceiling there too, so Phase 11 knows whether Path A is available.
- `npx create-expo-app` at SDK 58 into `templates/expo-app/`, then strip it to the layout in `06` §2.
- Templates for `package.json` (versions taken from `expo/bundledNativeModules.json` of the pinned `expo`), static `app.json` (`web.output: "single"` **set explicitly**: SDK 58 Expo Router defaults to `"server"`), `eas.json`, `babel.config.js` and `tsconfig.json` (with the `@/` alias), with placeholders `{{appName}} {{slug}} {{bundleId}} {{androidPackage}} {{scheme}} {{splashBg}}`.
- `templates/expo-app/ALLOWED_IMPORTS.json` exactly as in `06` §4.

### 2. Theme system
- `lib/theme/derive.ts` (platform code, shared by the web app and the worker) turns base tokens into the full token set from `06` §5: soft tints, `on*` colors, contrast fixes to WCAG AA, and neutrals tinted toward the primary hue. It's pure and table-tested. Phase 08's Foundation step and `set_theme` use it.
- `templates/expo-app/theme/index.ts` (platform-owned): `export const colors` resolved from `themes.ts` at `activeThemeId`; `useThemeId()`.
- `lib/render/themes.ts` renders `theme/themes.ts` from `projects.plan.themePackage` (`{themes[], activeThemeId}`).

### 3. Data layer (`@/lib/backend`, platform-owned)
- `provider.ts`: the `BackendProvider` interface (`list/get/insert/update/remove` for collections, `getValue/setValue`, auth methods, `uploadFile`).
- `local.ts`: AsyncStorage provider, seeded from `data/seed.ts` on first run. It **validates writes against `data/schema.ts`** (unknown field or wrong type → error), so behaviour matches the remote provider later.
- `index.ts`: `useCollection<T>(name)` → `{items, add, update, remove, loading}` (an external store shared across screens), `useStoredValue(key, default)`, `useAuth()` (fake local session with `loading`), `uploadFile(uri)` (local: returns the uri), `sendEmailToMe` (local: no-op + console notice).
- `lib/render/schema.ts`, `lib/render/seed.ts`: render `data/schema.ts` (TS types per collection, `typeName`) and `data/seed.ts` from `projects.plan.collections`. `lib/render/backend-config.ts` renders `config.ts` (local provider until Phase 13).

### 4. Layouts and routes
- `lib/render/layouts.ts`: from `projects.plan` (navigation, screens, `tab`, `firstScreenId`), render `app/_layout.tsx` (root Stack + providers + redirect to the first screen) and, for tab apps, `app/(tabs)/_layout.tsx` (shared tab bar with Ionicons). The output is deterministic, so the same plan always gives byte-identical files.
- `lib/render/routes.ts`: `screenRoute(screen)` → `/(tabs)/<id>` or `/<id>`; `screenFilePath(screen)`.

### 5. `CONTRACT.md` (human-readable)
`templates/expo-app/CONTRACT.md` documents the folder layout, ownership, imports, hooks, theme usage and editability rules for humans and code review. The **prompt** version of these rules lives in `lib/ai/prompts/rules.ts` (Phase 08, `05` §5); keep the two in sync.

### 6. Fixture app (finalize the S1 fixture)
`preview-runtime/fixtures/habit-tracker/` (created in Phase 00b S1; ADR-003 #10 fixes its shape):
- `plan.json`: an `AppPlan` with a `sign-in` stack screen (first), **2 tab screens (`today`, `habits`)**, a `habit-detail` stack screen, 2 collections (`habits`, `checkins` with `habitId` references) with sample records, and a 5-theme package.
- Hand-written `app/sign-in.tsx`, `app/(tabs)/{today,habits}.tsx`, `app/habit-detail.tsx` and **4 components** `components/{Button,ScreenHeader,EmptyState,HabitCard}.tsx`, following the rules in `05` §5: `colors` tokens only, `@/lib/backend` hooks only, one element per line, no tab bar in screens.
- It must also run under real Expo: `materialize({target: "export"})` to a temp dir → `npx expo start`.

### 7. Virtual file system `lib/vfs`
- Drizzle tables `project_files`, `file_revisions`, `project_snapshots` (`03` §5). `screens` already exists from Phase 04; don't recreate it.
- `lib/vfs/paths.ts`: `assertWritable(path, author)` enforces `06` §3. Agents may write only `app/(tabs)/<id>.tsx`, `app/<id>.tsx` and `components/<Name>.tsx` (PascalCase). It also normalizes POSIX paths and rejects `..`, absolute paths, NUL bytes and every platform-rendered path.
- `lib/vfs/files.ts`: `readFile`, `listFiles`, `writeFile({projectId, path, content, author, runId})` (sha256, no-op skip, revision + pointer update in one transaction), `deleteFile` (soft).
- `lib/vfs/materialize.ts`: `materialize(projectId, {target, snapshotId?})` per `06` §10. `preview` = project files + rendered platform files. `export` adds configs, README and assets. `snack`, `eas` and `web` are **stubs that throw "not yet implemented"** until Phases 11, 15 and 16.
- Unit tests: traversal rejection, ownership rules (agent can't write `theme/themes.ts`, `data/schema.ts`, `app/_layout.tsx` or `lib/backend/*`), the revision chain, no-op writes, and deterministic rendering (render twice → identical bytes).

### 8. Harden and test the preview runtime (S1 code graduated from Phase 00b)
S1 already proved 8a–8h (gate G0). This step turns the spike code into production code: types, tests, error handling, Safari fixes, performance, and the production CSP. The list below is the inventory to cover:
- 8a. `vendor.ts` + `build.mjs` (esbuild) → `public/preview-runtime/vendor.js` exposing `window.__AB_MODULES__`, built from the template's own pinned versions.
- 8b. `loader.ts` + `transpile.ts` (`@babel/standalone`, TS + JSX → CJS; resolves `./`, `@/`, extensions and index files; sha cache).
- 8c. Shims: `expo-router` (memory history; Stack, Tabs, Slot, Link, Redirect, `router.push/replace/back`, `useRouter`, `useLocalSearchParams`, `useSegments`, `usePathname`), `expo-haptics`, `expo-image-picker` (file input), `expo-clipboard`, `expo-linking`, `expo-location` and `expo-notifications` (permission-denied stubs), `expo-status-bar`, `react-native-safe-area-context`, and `@react-native-async-storage/async-storage` (**in-memory**, with the parent persisting snapshots via `storage.snapshot`/`storage.restore`, `07` §4). `routes.ts` builds the route table from `app/**`.
- 8d. **Preview `@/theme`**: the loader substitutes a runtime module where `colors.<token> = "var(--c-<token>)"`. `theme.set {themeId}` writes the CSS variables on the frame root. `@/lib/backend` is the template's real code, so the fixture exercises the real local provider.
- 8e. `bridge.ts` (Zod-validated protocol, `07` §4) + `entry.tsx`: the fixture renders in one interactive iframe.
- 8f. Frame mode with `initialRoute` → a dev page `/dev/preview-lab` with 4 frames.
- 8g. Hot swap: textarea edit → `file.update` → module invalidation → re-render in < 300 ms.
- 8h. Inspector Babel plugin (`07` §5) stamps `dataSet={{afid: "<path>:<line>:<col>"}}` on RN host elements and app components. A click logs `element.select`. `edit.override` handling is stubbed (Phase 10).
- 8i. Production CSP on the runtime document: `default-src 'none'; script-src <runtime origin> 'unsafe-eval'; style-src 'unsafe-inline'; font-src <runtime origin> data:; img-src https: data: blob:; connect-src <backend URL when connected>` (ADR-003 #15). Ionicons must render under it.
- 8j. **Harness for server-side renders:** `public/preview-runtime/harness.html` renders each route in turn from files posted to it. Phase 08's smoke test and Phase 17's thumbnails load it in the **sandbox service** from a local static server over the bundled `public/preview-runtime/` (ADR-003 #18), never from the web app's origin.
- Tests: loader resolution, router shim (push/replace/back, params, tabs), bridge message validation (bad messages ignored), transpile cache, `afid` stamps on the fixture, CSS-variable theme switch, icons under CSP.

### 9. Wire the workspace to real files
- `components/canvas/Canvas.tsx`: one `<PhoneFrame mode="frame">` per `screens` row at `canvas_x/canvas_y`. Off-screen frames show a snapshot instead of a live iframe (`07` §7).
- `components/workspace/InteractivePreview.tsx`: one `mode="interactive"` iframe, with a route selector and **Reset data**.
- `components/workspace/CodeView.tsx`: a file tree with agent/user files plus platform-rendered files shown read-only with a "generated" badge. Shiki viewer, copy button.
- `useProjectFiles(projectId)`: fetches path + sha + revisionId, then contents by revision id (cached forever). It subscribes to `file.written`, pushes `file.update` to every mounted iframe (coalesced 150 ms), and re-renders platform files when `projects.plan` changes (a `plan.updated` refresh).
- A theme dropdown is **not** built yet (Phase 08). The dev lab has a theme switch button to prove `theme.set`.
- `runtime.error` shows as a red badge on the frame. "Fix with AI" comes in Phase 09.

### 10. Dev-only fixture seeding
`scripts/seed-fixture-project.ts --user <email>` creates a project with `projects.plan` from the fixture `plan.json`, writes the fixture screens/components through `lib/vfs` (`author: "system"`), and creates `screens` rows. Guarded by `NODE_ENV !== "production"`.

## Files to create / change
```
templates/expo-app/{VERSION,CONTRACT.md,ALLOWED_IMPORTS.json}
templates/expo-app/{package.json,app.json,eas.json,babel.config.js,tsconfig.json}.hbs
templates/expo-app/theme/index.ts
templates/expo-app/lib/backend/{provider.ts,local.ts,index.ts}
lib/theme/derive.ts
lib/render/{themes.ts,schema.ts,seed.ts,layouts.ts,routes.ts,backend-config.ts,platform-files.ts}
lib/vfs/{paths.ts,files.ts,materialize.ts,index.ts}
lib/db/schema.ts                       (+ project_files, file_revisions, project_snapshots; screens from Phase 04)
preview-runtime/src/{vendor.ts,loader.ts,transpile.ts,routes.ts,bridge.ts,entry.tsx,theme-vars.ts}
preview-runtime/src/shims/{expo-router.tsx,async-storage.ts,expo-haptics.ts,expo-image-picker.ts,
                           expo-clipboard.ts,expo-linking.ts,expo-location.ts,expo-notifications.ts,
                           expo-status-bar.ts,safe-area-context.tsx}
preview-runtime/src/inspector-plugin.ts
preview-runtime/fixtures/habit-tracker/{plan.json,app/**,components/**}
preview-runtime/build.mjs              → public/preview-runtime/* (incl. harness.html)
preview-runtime/**/*.test.ts
components/canvas/{Canvas.tsx,PhoneFrame.tsx}
components/workspace/{InteractivePreview.tsx,CodeView.tsx}
lib/preview/{useProjectFiles.ts,frameBridge.ts}
app/dev/preview-lab/page.tsx           (dev only)
app/api/projects/[projectId]/files/route.ts
app/api/projects/[projectId]/files/[...path]/route.ts
scripts/seed-fixture-project.ts
```

## Database changes
- New tables: `project_files`, `file_revisions`, `project_snapshots` (exact columns in `03-data-model.md` §5). `screens` exists from Phase 04.
- Uses `projects.plan` (jsonb) and `projects.template_version`.
- Index: `file_revisions (project_id, path, created_at desc)`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/projects/:id/files` | viewer | → `{ files: [{ path, sha, revisionId, kind, screenId, generated: boolean }] }` (includes platform-rendered files) |
| GET | `/api/projects/:id/files/:path*?rev=<revisionId>` | viewer | → `text/plain`; `immutable` caching when `rev` is given. Rendered files are keyed by plan hash. |
| GET | `/api/projects/:id/screens` | viewer | → `{ screens: [{ id, name, route, filePath, order, status, canvasX, canvasY }] }` |
| PATCH | `/api/projects/:id/screens/:screenId` | editor | `{ canvasX, canvasY }` → `204` |

No write API for file contents yet. Writes come from the generators (Phase 08), inline edits (Phase 10) and theme switches (Phase 08).

## Acceptance checklist
- [ ] The fixture, materialized with target `export`, runs with `npx expo start` on iOS and Android (Expo Go) and on web.
- [ ] The template is pinned to SDK 58 (store Expo Go), with the date and the published `snack-sdk` ceiling recorded in D7. `app.json` sets `web.output: "single"` explicitly.
- [ ] `lib/vfs` tests pass: `../`, absolute paths, `package.json`, `theme/themes.ts`, `data/seed.ts`, `app/_layout.tsx` and `lib/backend/x.ts` are rejected for `author: "agent"`; `components/habitCard.tsx` (not PascalCase) is rejected.
- [ ] Platform rendering is deterministic: the same plan gives byte-identical `themes.ts`, `schema.ts`, `seed.ts` and layouts.
- [ ] The seeded fixture shows 4 frames on the canvas. The interactive preview signs in (fake session), navigates the tabs, adds and completes a habit (data persists across a frame reload via parent snapshots), and **Reset data** restores the seed.
- [ ] A local-provider write with an unknown field or a wrong type is rejected.
- [ ] Switching theme in the dev lab recolors all frames instantly, with no re-transpile (verify in the transpile counter) and no app restart (in-memory state preserved).
- [ ] Editing a fixture file and emitting `file.written` updates frames in < 1 s without a reload.
- [ ] Clicking an element logs an `afid` mapping to the correct `path:line:col`.
- [ ] Ionicons (tab bar and screen icons) render inside a sandboxed frame **under the production CSP** (`font-src <runtime origin> data:`); a test asserts icon glyphs aren't blank.
- [ ] `harness.html` renders every fixture route headlessly (Playwright, run locally the same way the sandbox service will).
- [ ] **Security:** iframes have `sandbox="allow-scripts"`, no `allow-same-origin`. Fixture code can't read the parent's cookies or `fetch('/api/projects')` with credentials (test asserts it).
- [ ] **AuthZ:** file and screen routes return 404 for non-members (two-user test).
- [ ] With 10 frames, off-screen frames aren't live iframes, and panning stays smooth on a mid-range laptop.

## Claude Code prompt
```text
We are building AppBuilding, an AI mobile-app builder. Implement Phase 07 only.

Read first, in order:
- docs/plan/02-architecture.md (§3 decisions A4–A7, §5 repo layout)
- docs/plan/03-data-model.md (§2 projects.plan, §5 virtual file system)
- docs/plan/06-generated-app-template.md (entire file: this is the contract)
- docs/plan/07-preview-runtime.md (entire file)
- docs/plan/13-reference-implementation.md (R5–R9, R11)
- docs/plan/phases/phase-07-template-and-preview-runtime.md (this phase)

Scope, one commit per numbered step of the phase doc:
1. templates/expo-app pinned to Expo SDK 58 (store Expo Go; see docs/plan/15-vendor-facts.md
   §1 and ADR-002). Confirm Expo Go's current SDK before scaffolding; set web.output "single".
2. lib/theme/derive.ts (token derivation + contrast) and the platform renderers in
   lib/render (themes, schema, seed, layouts, routes, backend config). Deterministic output.
3. Template @/theme (colors) and @/lib/backend (provider interface, local provider with
   write validation, useCollection/useStoredValue/useAuth/uploadFile hooks).
4. Finalize the S1 fixture: sign-in + 2 tabs (today, habits) + habit-detail, 4 components,
   2 collections, following docs/plan/05-ai-agent-design.md §5 rules.
5. lib/vfs (ownership allowlist per 06 §3, revisions, materialize with targets; snack/eas/web
   targets throw "not yet implemented") + tables + unit tests.
6. Harden and test the preview runtime that graduated from spike S1 (8a–8j): types, tests,
   Safari, performance, production CSP incl. font-src, and harness.html for server-side renders.
7. Wire canvas frames, interactive preview (Reset data) and read-only Code tab (generated
   files badged) to real project files; subscribe to file.written.
8. scripts/seed-fixture-project.ts (dev only).

Rules:
- NO AI/LLM calls in this phase.
- Iframes: sandbox="allow-scripts" only. Validate every postMessage with Zod and check event.source.
- Every API route uses requireProjectAccess().
- Platform-derived files (themes, schema, seed, layouts, configs, lib/backend) are rendered,
  never stored per project and never writable by agents.
Do NOT build: theme picker UI, inline editing UI, Snack/QR, export endpoints.

Definition of done = the Acceptance checklist in the phase doc.
Finish by running lint, typecheck and tests. Summarize what works, what doesn't, and any
decision that should go into docs/plan/12-decisions-and-open-questions.md.
```

## Pitfalls & notes
- **react-native-web must match** the template's React Native version. Build the vendor bundle from the template's own `package.json`.
- **CSS variables and react-native-web:** check that RNW passes `var(--…)` strings through for every style prop the generators use (colors, borderColor, shadowColor, backgroundColor, icon `color`). Props RNW converts in JS, such as some shadow handling, may need the runtime to resolve the token to a literal and re-render on `theme.set`. Write a test screen covering every token usage.
- `@babel/standalone` is ~2 MB. Load it once in the vendor bundle; transpile only changed files.
- Expo Router conventions for `routes.ts`: `(group)` folders don't add segments; `_layout.tsx` nesting; `index.tsx`; `+not-found.tsx`. The generators only produce flat `/<id>` and `/(tabs)/<id>` routes, so keep the shim small. A pattern the shim can't support gets banned in the rules, not added to the shim.
- `@expo/vector-icons` on web needs font files. Ship them with the vendor bundle, register `@font-face`, and allow them in the CSP (`font-src`). Without that, icons render blank.
- Opaque-origin iframes can't use `localStorage` (it throws). The AsyncStorage shim is in-memory, namespaced by `projectId + frameId`, and the parent persists snapshots.
- Safari is the usual troublemaker (iframe transforms, `ResizeObserver` timing). Test early.
- Don't build a platform UI component library. The reference app showed per-app generated components work and make edits cheap (R6).
