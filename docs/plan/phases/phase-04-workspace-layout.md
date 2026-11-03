# Phase 04 — Workspace Layout

| | |
|---|---|
| Video chapter(s) | Ch.9 "Workspace Layout" (52:50); QR placement from Ch.14 (1:54:57) |
| Branch | `feat/04-workspace-layout` |
| Depends on | Phase 03 |
| Effort (solo dev + Claude Code) | ~2 days |
| New env vars | none |

## Goal
Build the `/workspace/[projectId]` shell where users spend most of their time: header, resizable 3-panel layout (chat | canvas | preview/code), a pan/zoom canvas with draggable phone frames whose positions persist, and a right panel with Preview and Code tabs. Everything renders **placeholder data** behind typed props and hooks, so Phases 05–08 only swap the data sources. The page is protected by `requireProjectAccess`.

## What we do differently from the video (and why)
- **The QR / "Open on device" button goes in the right-panel header from the start.** The video put it below the preview, then moved it in Ch.14 to give the preview more height.
- **Frame positions persist** (`screens.canvas_x/y`) and the layout state (panel sizes, zoom) is saved per user in `localStorage`. The video's canvas resets on reload.
- **Typed workspace store** (Zustand) with selectors for `screens`, `files`, `selectedScreenId`, `selectedElement`, `activeThemeId`, `runStatus`. Later phases plug in SSE events and the preview runtime without prop-drilling rewrites.
- **Phone frames are a component with a `content` slot.** In this phase the slot is a placeholder; Phase 07 mounts preview-runtime iframes in it (`07-preview-runtime.md`).
- **Header action buttons exist but are disabled with tooltips** ("Coming soon"), so the layout doesn't shift as features land.

## Steps

### 1. Route and data loading
`app/workspace/[projectId]/page.tsx` (server component):
- `requireProjectAccess(projectId, "viewer")` → 404 page if denied.
- Load `project`, `messages` (latest 100), `screens` (empty now), the file list (empty now) and the latest `runs` row.
- Pass a serializable `WorkspaceBootstrap` to the `<Workspace>` client component.
- `role` is passed down. Viewers get a read-only UI (chat input disabled, frames not draggable).

### 2. Workspace store
`components/workspace/store.ts` (Zustand):
```ts
type WorkspaceState = {
  project: ProjectDTO; role: "viewer" | "editor" | "owner";
  messages: MessageDTO[]; screens: ScreenDTO[]; files: FileMetaDTO[];
  activeThemeId: string | null; runStatus: RunStatus | null;
  selectedScreenId: string | null; selectedElement: ElementRef | null;
  canvas: { zoom: number; x: number; y: number; tool: "select" | "hand" };
  rightTab: "preview" | "code"; openFilePath: string | null;
  // actions …
};
```

### 3. Header
`components/workspace/header.tsx`: logo (links home) · editable project title (inline rename → `PATCH /api/projects/:id` for owners/editors) · project status pill (`planning`/`awaiting_confirmation`/`building`/`ready`/`failed`, per `03` §2) · **theme dots slot** (empty until Phase 08) · right side: **Connect backend**, **Share**, **Export**, **Publish** (all disabled stubs with tooltips) · user avatar menu.

### 4. Three-panel layout
shadcn `Resizable` (`react-resizable-panels`): left chat (default 26%, min 20%), center canvas (flex), right preview/code (default 30%, min 24%). Save sizes per user in `localStorage` (`autoSaveId`). Under 1024 px, switch to a tab bar (Chat / Canvas / Preview) instead of side-by-side panels.

### 5. Chat panel (shell)
`components/chat/chat-panel.tsx`: virtualized message list (render existing `messages`: user prompt bubble, assistant text), message input with attach button (reuses `attachment-strip`), and a send button that's disabled with the hint "Agent arrives in Phase 06". Message renderers are a `kind → component` map (`text`, `plan_card`, `build_summary`, `error`, `element_ref`), so Phase 06 only adds the `plan_card` renderer.

### 6. Canvas
`components/canvas/canvas.tsx`:
- Pan/zoom surface: `react-zoom-pan-pinch`, or a custom transform with pointer events. Wheel+ctrl/pinch = zoom (10%–200%), space+drag or the hand tool = pan.
- Toolbar (bottom-center): Select / Hand, zoom −/+ and %, "Fit all", fullscreen (Fullscreen API on the canvas container).
- Dot-grid background that scales with zoom.
- `PhoneFrame` (`components/canvas/phone-frame.tsx`): fixed logical size (e.g. 390×844) with a notch/status-bar chrome and the screen name label above. Click selects it (outline). The drag handle on the label moves it, and on drag end it calls `PATCH /api/projects/:id/screens/:screenId` `{canvasX, canvasY}` (debounced).
- Frame states: `planned` (skeleton + name), `building` (shimmer), `ready` (content slot), `error` (red outline + message).
- Hover toolbar on a selected frame: "Edit" (inline editing, Phase 10), "Open in preview", "View code". Disabled for now.
- Until real screens exist: if `screens` is empty, show 3 placeholder frames from a fixture so the layout can be judged.

### 7. Right panel
`components/workspace/right-panel.tsx` with tabs:
- **Preview**: a single larger phone frame (content slot, Phase 07 mounts the interactive runtime) and a route dropdown. Header row: **Open on device (QR)** button (opens a popover; disabled stub until Phase 11), reload button, "Reset data" button.
- **Code**: file tree (`components/workspace/file-tree.tsx`, grouped by folder) plus a **read-only** viewer. Use Shiki (lighter) for v1. Phase 10b swaps in a Monaco editor for editable files. Copy-file button. It's empty until Phase 08 writes files, so add a demo fixture behind `?demo=1` to develop against.

### 8. Screens table (minimal)
Add the `screens` table now (`03-data-model.md` §5), since it holds canvas positions. **This is the only phase that creates it**; Phase 06 upserts rows on confirm and Phase 08 updates statuses.

### 9. Keyboard shortcuts
`V` select, `H` hand, `Cmd/Ctrl+0` fit, `Cmd/Ctrl+±` zoom, `Esc` deselect. Ignore them while focus is in the chat input.

### 10. Tests
Store reducer tests (select, move, zoom bounds). Playwright smoke: open a project → three panels render → drag a frame → reload → position kept. Cross-user access → 404.

## Files to create / change
```
app/workspace/[projectId]/page.tsx
app/workspace/[projectId]/not-found.tsx
components/workspace/{workspace.tsx,store.ts,header.tsx,right-panel.tsx,file-tree.tsx,code-viewer.tsx,status-pill.tsx}
components/chat/{chat-panel.tsx,message-list.tsx,message-input.tsx,renderers/text.tsx}
components/canvas/{canvas.tsx,toolbar.tsx,phone-frame.tsx,frame-placeholder.tsx}
lib/db/schema/screens.ts
lib/dto.ts                         (ProjectDTO, MessageDTO, ScreenDTO, FileMetaDTO)
app/api/projects/[projectId]/route.ts                  (PATCH title)
app/api/projects/[projectId]/screens/[screenId]/route.ts (PATCH position)
tests/e2e/workspace.spec.ts
```

## Database changes
- New `screens` table (`id`, `project_id`, `name`, `route`, `file_path`, `order`, `status`, `canvas_x`, `canvas_y`, `created_at`, `updated_at`).

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| PATCH | `/api/projects/:projectId` | editor | `{title}` → `{project}` |
| PATCH | `/api/projects/:projectId/screens/:screenId` | editor | `{canvasX, canvasY}` → `204` |

Every handler starts with `requireProjectAccess(projectId, "editor")` **and** checks that `screenId` belongs to `projectId` (no cross-project updates via a guessed screen id).

## Acceptance checklist
- [ ] `/workspace/{id}` shows the header, chat (with the original prompt bubble), canvas with placeholder frames, and the right panel with Preview/Code tabs.
- [ ] Panels resize; sizes persist across reloads. Under 1024 px the layout switches to tabs.
- [ ] Canvas pans, zooms (bounded 10–200%), fits all, and goes fullscreen; shortcuts work and don't fire while typing.
- [ ] Dragging a frame persists `canvas_x/y`; the position survives a reload.
- [ ] Inline title rename works for the owner; viewers see no edit affordances.
- [ ] Disabled header buttons (Connect backend, Share, Export, Publish, QR) show tooltips and cause no layout shift.
- [ ] Code tab renders the demo fixture tree and syntax-highlighted file (`?demo=1`).
- [ ] Another user gets 404 for the page **and** for both PATCH endpoints; a screen id from another project is rejected.

## Claude Code prompt
```text
Read first:
- docs/plan/02-architecture.md (§2 component diagram, §5 layout)
- docs/plan/03-data-model.md (§5 screens)
- docs/plan/07-preview-runtime.md (§3, §7 — only to understand the frame content slot)
- docs/plan/phases/phase-04-workspace-layout.md (this phase)

Implement Phase 04 (UI shell, placeholder data):
1. Server page /workspace/[projectId] with requireProjectAccess(…, "viewer"), loading
   project, messages, screens, latest run → WorkspaceBootstrap → <Workspace/> client.
2. Zustand workspace store exactly as typed in the doc.
3. Header with inline rename (PATCH /api/projects/:id), status pill, empty theme-dots
   slot, disabled Connect backend / Share / Export / Publish buttons with tooltips.
4. Resizable 3-panel layout (shadcn Resizable, autoSaveId), tabs layout < 1024px.
5. Chat panel shell with kind→renderer map (only "text" renderer now), disabled send.
6. Canvas: pan/zoom (react-zoom-pan-pinch or custom), toolbar, fit, fullscreen,
   dot grid, PhoneFrame component with states planned/building/ready/error and a
   content slot; draggable frames persisting canvas_x/y via
   PATCH /api/projects/:id/screens/:screenId (debounced). Placeholder frames when empty.
7. Right panel: Preview tab (frame + route dropdown + QR button in the header, disabled)
   and Code tab (file tree + Shiki read-only viewer; demo fixture with ?demo=1).
8. screens table + migration. Keyboard shortcuts.
9. Tests: store unit tests, Playwright smoke incl. cross-user 404.

Do NOT implement SSE, jobs, the AI agent, or the preview runtime iframe.
Definition of done = Acceptance checklist in the phase doc.
Finish with lint/typecheck/test (+ playwright) and summarize.
```

## Pitfalls & notes
- Don't render iframes inside a CSS-transformed canvas without testing pointer mapping. Scaling breaks hit-testing for some libraries. Phase 07 will mount iframes in the frame slot; check now that a plain `<iframe>` inside a scaled frame still receives clicks at the right coordinates.
- Persisting positions on every mousemove floods the DB. Persist on drag end, debounced.
- Long chats: virtualize the list (`@tanstack/react-virtual`) now. Agent runs produce many messages.
- Keep `WorkspaceBootstrap` small. Never ship file **contents** in the initial payload; fetch by revision later (`05-ai-agent-design.md` §8).
- The video's layout is a starting point; the user can tweak it. Prioritise the canvas and preview sizes, since those are what users judge the product by.
