# Phase 09 — Chat Edits, Add/Remove Screens, Fix-with-AI & Version Restore

| | |
|---|---|
| Video chapter(s) | Ch.11 "Generate Mobile App with AI", second half (~1:26:40 to 1:30:00). Covers the "add welcome screen" request the agent refused, and the fix the video applies after it. Also the error-paste fix in Ch.12 (~1:42:54). |
| Branch | `feat/09-chat-edits` |
| Depends on | Phase 08 (Builder, tools, validation, snapshots) |
| Effort (solo dev + Claude Code) | ~3–4 days |
| New env vars | none |

## Goal
After the first build, the user keeps working through chat. They can ask questions, change one element or a whole screen, add or remove screens ("add a welcome screen" goes in first position), paste an error message, or click **Fix with AI** on a frame that threw a runtime error. All of these run through one **Editor** agent loop. Every edit run ends with a validated snapshot and a `build_summary` message, and the user can **restore** any previous snapshot.

## What we do differently from the video (and why)
- In the video, the agent answered "adding/editing screens by chat isn't available" (~1:27:20) because its instructions were written only for first-time generation. Our Editor's system prompt and toolset include `add_screen` and `remove_screen` from the start, and there's an eval prompt for "add a screen" so this can't silently regress.
- **One Editor loop decides the request type itself.** There's no separate intent-classifier call (`05` §2.3), so there's one less prompt to maintain.
- **Runtime errors are captured automatically** from the preview runtime (`runtime.error`) and attached as structured context. The user doesn't have to copy and paste stack traces, though pasting still works.
- **Every run is reversible** through `project_snapshots`. The video has no undo besides git.

## Steps

### 1. Editor prompt and context builder (`lib/ai/edit.ts`)
- The system blocks reuse the Builder's stable prefix: role, contract, UI/UX guide, component docs, exemplars. They add an **Editor addendum**: the request types and expected behaviour from `05` §6, "you may add, modify and delete screens", "prefer the smallest change", "end with a short summary".
- Volatile context goes in `messages[0]`: the current plan with its screen list and routes, the file tree, full contents of the selected screen and its components, `theme/themes.ts`, the last validation result, and the selected element or runtime error context when present.
- Context budget: if the selected files exceed ~60k tokens, include only the file tree plus selected files and let the agent use `read_file` for the rest.

### 2. Screen tools
- `add_screen {id, name, route, tab?: {label, icon}, position?}` inserts a `screens` row (`status: building`). If `tab` is given, it regenerates `app/(tabs)/_layout.tsx` deterministically from the screens list. Platform code writes that file, not the agent, so the tab order is always consistent. `position: "first"` handles welcome/onboarding screens. For onboarding, the tool also writes an `app/index.tsx` redirect stub that the agent then fills in.
- `remove_screen {id}` soft-deletes the route file and the `components/<id>/` folder, removes the screen from tabs, and fails if other files still link to its route. The error lists the referencing files, so the agent fixes them first.
- Both tools emit `screen.started` / `screen.done` so the canvas adds or removes frames live.

### 3. Edit handler (`worker/handlers/agent-edit.ts`)
- `runs.kind = edit`. One manual streaming loop (`worker/agent/loop.ts` from Phase 08) at effort `medium`, with the full toolset and `allowedPaths` = all agent-owned paths.
- The loop must call `validate` before finishing. After the loop, the handler runs `validate` plus a render smoke test **only for the screens touched**, then up to `AGENT_MAX_FIX_ITERATIONS` fix turns appended to the same transcript.
- At the end: create a `project_snapshots` row, insert `messages(kind=build_summary)` with `{changedFiles[], addedScreens[], removedScreens[], summary}`, then emit `run.done`.
- Concurrency: one build/edit run per project at a time. A new chat message while a run is active is queued, with a "Waiting for current change…" status on the message.

### 4. Chat input → edit run
- `POST /api/projects/:id/messages` gets a new behaviour. If the project has a confirmed build (`status` is `ready` or `error`), a user message starts an `edit` run. Before the first build it still revises the plan (Phase 06).
- Attachments: images (reference) go as image blocks. `element_ref` chips come from the inspector (Phase 10 adds the UI; the payload shape is defined here: `{afid, filePath, line, col, snippet, componentName}`).

### 5. Fix with AI
- When a frame posts `runtime.error`, the frame shows a red badge and a **Fix with AI** button.
- The button creates a user message (`kind: text`, content "Fix this error on <screen>") whose `payload` carries `{error: {message, stack, path, line}, screenId}`. The edit handler puts the error at the top of the context.
- Pasted errors: if a user message contains something that looks like a JS stack or `Error:` text, the context builder parses file/line hints and preloads those files.

### 6. Restore previous version
- The chat timeline shows each `build_summary` with a **Restore this version** action, and the header has a "History" popover listing snapshots (time, summary, author).
- `POST /api/projects/:id/snapshots/:snapshotId/restore` writes, for every path in the snapshot whose current revision differs, a **new** revision copying the old content (`author: "system"`). It soft-deletes files absent from the snapshot, restores `screens` rows from the snapshot metadata, and creates a new snapshot "Restored to …". History is never rewritten.
- Store `screens` state inside `project_snapshots` (add a `screens` jsonb field) so restore can bring back deleted screens.

### 7. Eval additions
- Add edit evals to `evals/`: "add a welcome screen" (expects a new first screen and nav wiring), "change the header color to the secondary color" (expects a theme-token use, no hex literal), "remove the settings screen" (no dangling links), "what does the progress screen show?" (expects no file writes), and "fix: Cannot read properties of undefined (reading 'map')" with a seeded bug.

## Files to create / change
```
lib/ai/edit.ts                       (context builder + editor addendum)
lib/ai/prompts/editor/addendum.md
lib/ai/tools/{add-screen,remove-screen}.ts
lib/navigation/tabsLayout.ts         (deterministic (tabs)/_layout.tsx writer)
worker/handlers/agent-edit.ts
app/api/projects/[projectId]/messages/route.ts        (route plan-revision vs edit)
app/api/projects/[projectId]/snapshots/route.ts       (GET list)
app/api/projects/[projectId]/snapshots/[snapshotId]/restore/route.ts
components/chat/{BuildSummaryCard.tsx,ElementRefChip.tsx,QueuedBadge.tsx}
components/canvas/FrameErrorBadge.tsx
components/workspace/HistoryPopover.tsx
evals/golden/edits/*.json
```

## Database changes
- `runs.kind = edit` (already in the enum).
- `messages.kind`: `build_summary`, `element_ref` (already defined). `payload` carries error or element context.
- `project_snapshots`: add a `screens jsonb` column (the screen list at snapshot time) and a `label text` column.
- `file_revisions.author = system` for restores.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/messages` | editor | `{ content, attachments?: assetId[], elementRef?, error? }` → `{ messageId, runId, queued: boolean }` |
| GET | `/api/projects/:id/snapshots` | viewer | → `{ snapshots: [{ id, label, createdAt, runId }] }` |
| POST | `/api/projects/:id/snapshots/:snapshotId/restore` | editor | → `{ snapshotId (new), changedPaths[] }`; 409 if a run is active |

## Acceptance checklist
- [ ] "Add a welcome screen for the app" adds a new first frame on the canvas. The app opens on it in the interactive preview, with a working "Get started" button to the main tabs.
- [ ] "Add a settings screen with currency, dark mode and notification preferences" adds a tab and a working screen.
- [ ] "Remove the X screen" removes the frame and leaves no broken links (validation clean).
- [ ] A question gets a chat answer and creates no file revisions.
- [ ] A seeded runtime error shows a badge; **Fix with AI** fixes it and the badge clears.
- [ ] A pasted error message is fixed (reproduce the video's sign-up error pattern with a seeded bug).
- [ ] **Restore** brings back the exact file set and screen list of an earlier snapshot, including a screen deleted later. Current history remains visible.
- [ ] Sending a message during an active run queues it, and it runs afterwards.
- [ ] **Security:** element and error payloads are validated with Zod and size-capped (e.g. snippet ≤ 4 KB, stack ≤ 8 KB). Error text is passed to the model as quoted data, never as instructions.
- [ ] **AuthZ:** viewers can't post messages or restore (403); non-members get 404.
- [ ] Edit evals pass (5/5), including "add screen".

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 09 (Editor agent: chat edits, add/remove
screens, Fix with AI, snapshot restore) only.

Read first:
- docs/plan/05-ai-agent-design.md (§2.3 Editor, §3 tools, §6 request types, §9 transcripts)
- docs/plan/06-generated-app-template.md (§2–§3 ownership — tabs layout is platform-written)
- docs/plan/03-data-model.md (§3 messages, §5 project_snapshots, screens)
- docs/plan/phases/phase-09-chat-edits.md (this phase)
- Existing code from Phase 08: worker/agent/loop.ts, lib/ai/tools, lib/validate, agent-build.ts

Key requirement: the Editor MUST be able to add and remove screens. In the original video,
the agent refused because its prompt only covered first-time generation. Add an eval for it.

Implement, one commit per phase-doc step:
1. lib/ai/edit.ts context builder + editor addendum prompt (reuse the Builder's cached prefix
   byte-for-byte; volatile context goes in messages).
2. add_screen / remove_screen tools + deterministic (tabs)/_layout.tsx writer.
3. worker/handlers/agent-edit.ts (validate + targeted smoke test + fix turns, snapshot,
   build_summary message, one active run per project with queueing).
4. messages route: plan revision before first build, edit run after.
5. Fix with AI from runtime.error + pasted-error parsing.
6. Snapshot list + restore (new revisions only, never rewrite history).
7. Edit evals.

Do NOT build: inspector UI (Phase 10 — only define the elementRef payload schema), Snack,
export, credits enforcement. Every route uses requireProjectAccess().

Definition of done = Acceptance checklist in the phase doc.
Finish with lint, typecheck, tests, eval results, and a short summary.
```

## Pitfalls & notes
- **Cache reuse:** the Editor should share the Builder's prefix bytes exactly so edits hit the cache. Put the Editor addendum *after* the shared cached blocks, or as the first message, not inside them.
- **Over-editing:** models tend to rewrite whole files. The addendum should prefer `edit_file`. Track "lines changed per edit" in evals and flag regressions.
- **Element-scoped requests** ("change icon to leaf") should touch only the referenced element. Pass the exact `line:col` and snippet. Also tell the agent the file sha, so a stale `afid` is detected.
- **Tabs layout writer:** route groups and icon names must match the allowlisted icon set (`@expo/vector-icons` family named in the contract). Validate icon names against the family's glyph map.
- **Restore + Snack/GitHub:** a restore is a normal new revision set, so Phase 11 (Snack live updates) and Phase 14 (GitHub push) need no special handling.
- **Billing later:** edit runs will reserve and charge credits in Phase 18. Keep the cost per edit in eval output so pricing can be set from real data.
