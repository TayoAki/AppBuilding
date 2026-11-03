# Phase 09 — Chat Changes (update_app), Fix-with-AI, New Session & Version Restore

| | |
|---|---|
| Video chapter(s) | Ch.11 "Generate Mobile App with AI", second half (~1:26:40 to 1:30:00): the "add welcome screen" request the agent refused, and the fix. Also the error-paste fix in Ch.12 (~1:42:54). Reference repo: Prompt 6.7 (`update_app`, `set_theme`) and Prompt 8.2 ("act only on the latest request", Start New Session). |
| Branch | `feat/09-chat-edits` |
| Depends on | Phase 06 (Planner + `propose_plan`/`confirm_plan`), Phase 08 (generators, validation + repair, `agent.step` pipeline, snapshots) |
| Effort (solo dev + Claude Code) | ~3–4 days |
| New env vars | none |

## Goal
After the first build, the user keeps working in chat. They can ask questions, change one element or a whole screen, add or remove screens ("add a welcome screen" becomes the first screen), add or change shared components, add data, switch or add a theme, paste an error, or click **Fix with AI** on a frame that threw. All of it goes through the **Planner** (the only conversational agent, `05` §3). The Planner turns a request into one structured `update_app` change (`05` §3.5) or a `set_theme` call. The **change pipeline** (`05` §4.2) then regenerates only the affected files. Every change ends with a validated snapshot and a `build_summary` message, and any snapshot can be **restored**.

## What we do differently from the video (and why)
- In the video, the agent answered "adding/editing screens by chat isn't available" (~1:27:20), because its instructions only covered first-time generation. Our Planner prompt has a "Changing a built app" section and `update_app` from day one. It's also told explicitly to ignore earlier replies that claimed changes weren't possible. An eval for "add a screen" guards against regressions.
- **No tool-looping editor and no file tools.** The Planner emits a typed `AppChange`. Code executes it with the same single-purpose generators as the first build (Data Designer, Component Builder, Screen Builder). That gives one code path, one validator, and predictable cost.
- **Runtime errors are captured automatically** from the preview runtime (`runtime.error`) and attached as structured context. Pasting still works.
- **Every change is reversible** through `project_snapshots`. The video has no undo besides git.

### Reference app notes
- **R1/R2:** the Planner plus generators replace our earlier Editor tool loop. `update_app` and `set_theme` are the only change tools.
- **R3:** "What to act on". Do only what the latest message asks; the current plan (`projects.plan`) is the truth about what exists. The **Start new session** button exists because the reference agent re-added screens requested earlier in the chat.
- **R4:** welcome, onboarding and sign-in screens become `firstScreenId` and link onward to the previous first screen.
- **R6:** requests about something a shared component draws go to `editComponents`. Every screen using it updates without being rewritten.
- **R10:** literal colors already in a file were chosen by the person and are kept on edits.

## Steps

### 1. Planner: change rules + state block
- Extend the Planner's static system prompt (Phase 06) with the "Changing a built app", "A selected element" and "What to act on" sections from `05` §3.2. Keep the text static so the cached prefix stays byte-identical.
- The PROJECT STATE mid-conversation system message (`05` §3.1) now includes the phase `ready`/`failed` (with N built), components with their used-by screens, themes with the active one, collections with field names, and the price list (10 credits per new screen, 5 per change round).
- Phase-dependent behaviour: while a build/change run is active, the state says "building, the plan can't change right now". `update_app` returns a tool error ("A change is in progress") and the Planner tells the person to wait. Questions are still answered.

### 2. `update_app` and `set_theme` tools (`lib/ai/tools/planner/*.ts`)
- `update_app` takes the `AppChange` input (`05` §3.5), `strict: true`. Server checks:
  - every `editScreens`/`removeScreens` id exists; `addScreens` ids are new and kebab-case;
  - screen count ≤ plan max (Free 5, Pro 10) and tab count ≤ `MAX_TABS`;
  - each added screen is reachable (some screen lists it in `navigatesTo` **and** is in `editScreens`, or it has a `tab`, or it's the new `firstScreenId`);
  - `editComponents` names exist; `addComponents` names are new and PascalCase;
  - a credit pre-check (enforced in Phase 18; until then, only logged).
- Failures return a short tool error that the Planner relays plainly. It never claims the work happened.
- On success: store the change in `projects.plan.pendingChanges`, create `runs(kind=change)`, and enqueue the first `agent.step`. The Planner then replies with one sentence on what's starting.
- `set_theme {themeId}` writes `activeThemeId` in `projects.plan.themePackage` and re-renders `theme/themes.ts`. `set_theme {newTheme}` runs the derivation + contrast pipeline (`06` §5) and adds the theme. Both are free and need no run; the frames get `theme.set`.

### 3. Change pipeline (`worker/handlers/agent-step.ts`, change steps)
Order (`05` §4.2):
1. `data` (if `dataChange`): Data Designer returns **all** collections, **additive only**. The server diffs against the current collections and rejects any removal or type change. A new field needs a value on every existing record. Then it re-renders `data/schema.ts` and `data/seed.ts`.
2. `component:<Name>` for each add or edit: Component Builder gets the app context, earlier components' props APIs, and, for edits, the current file plus the request with the rule "keep every existing prop working; new props optional".
3. `screen:<id>` for each add or edit: Screen Builder with the app context, routes, the first screen as style reference and, for edits, the current file plus the request and the CHOSEN_COLORS rule.
4. Remove: soft-delete the route files of `removeScreens`. Validation fails the step if any remaining file still navigates to a removed route; the error lists the files, so the Planner can include them in a follow-up.
5. Platform: regenerate `app/_layout.tsx` (first-screen redirect) and `app/(tabs)/_layout.tsx` (tab bar) from the plan. No screen is rewritten for a tab add or rename.
6. Backend schema sync if connected (Phase 13), then snapshot, then a `build_summary` message `{changedFiles[], addedScreens[], removedScreens[], summary}`, then `run.done`.
- **Keep old code until new passes.** Each generated file goes through validation + **one repair round** (`05` §6). Only a passing file becomes the current revision, so an edited screen keeps running its old code meanwhile. If a new screen still fails, mark it `error` (credits refunded in Phase 18). If an edit fails, keep the old code and add a note to the summary.
- Concurrency: one build/change run per project at a time (409 / tool error otherwise).

### 4. Selected element + Fix with AI
- Message payload schema (Phase 10 adds the UI): `{ selectedElement?: { file, line, col, label, screenId, code } }`. The server renders it as the `<selected-element file line label screen>…code…</selected-element>` block at the start of the user turn. It caps `code` at 4 KB and escapes it as data.
- **Fix with AI:** when a frame posts `runtime.error`, it shows a red badge and a **Fix with AI** button. The button posts a message "Fix this error on <screen>" with `payload.error = {message, stack, path, line}`. The Planner gets it as quoted data and normally answers with `update_app.editScreens` (or `editComponents` if `path` is a component) whose `request` contains the error and the failing line.
- **Pasted errors:** if a message looks like a JS error or stack, the server attaches parsed `{path, line}` hints in the state block.

### 5. Start new session
- A chat header button inserts a `messages(role=system_note, kind=session_break)` row. The Planner transcript starts after the last `session_break` (UI history above it stays visible but dimmed). Closing the confirm dialog on click is part of the UX (reference follow-up).
- The plan and state carry everything, so a fresh session loses nothing (`05` §9).

### 6. Restore previous version
- Each `build_summary` has **Restore this version**; the header gets a "History" popover listing snapshots (time, label, run).
- `POST /api/projects/:id/snapshots/:snapshotId/restore` writes a **new** revision (`author: "system"`) for every path whose content differs. It soft-deletes files absent from the snapshot, restores `screens` rows **and `projects.plan`** from the snapshot, and creates a new snapshot "Restored to …". History is never rewritten.

### 7. Eval additions (`evals/edits.jsonl`)
- "add a welcome screen": new `firstScreenId`, links onward, no other screen rewritten.
- "add a settings screen with currency, dark mode and notifications": a new tab via `tab`, with no screen changes for the bar.
- "make the cards rounder": an `editComponents` edit, not screen edits.
- "remove the settings screen": no dangling links.
- "what does the progress screen show?": no tool call.
- A selected-element request "change this icon to a leaf": only that element changes.
- A seeded runtime error: Fix with AI fixes it.
- A two-turn eval: add screen A, then "add screen B". The second change must not re-add A (R3).

## Files to create / change
```
lib/ai/prompts/planner/{change-rules.md,selected-element.md,act-on-latest.md}
lib/ai/planner/state.ts                    (PROJECT STATE block builder)
lib/ai/tools/planner/{update-app.ts,set-theme.ts}
lib/ai/schemas/change.ts                   (AppChange Zod + server checks)
worker/handlers/agent-step.ts              (change steps: data, component:<Name>, screen:<id>, remove)
lib/navigation/layouts.ts                  (deterministic _layout.tsx + (tabs)/_layout.tsx writers)
app/api/projects/[projectId]/messages/route.ts        (always → agent.chat)
app/api/projects/[projectId]/session/route.ts         (POST new session)
app/api/projects/[projectId]/snapshots/route.ts       (GET list)
app/api/projects/[projectId]/snapshots/[snapshotId]/restore/route.ts
components/chat/{BuildSummaryCard.tsx,SelectedElementChip.tsx,NewSessionButton.tsx}
components/canvas/FrameErrorBadge.tsx
components/workspace/HistoryPopover.tsx
evals/edits.jsonl
```

## Database changes
- `runs.kind = change` (in the enum); `runs.step` / `runs.lease_id` (Phase 05/08).
- `projects.plan.pendingChanges` (inside the `plan` jsonb).
- `messages.kind`: `build_summary`, `session_break` (new value), `element_ref` (existing). `payload` carries `selectedElement` / `error`.
- `project_snapshots`: `screens jsonb`, `label text` (already in `03` §5), plus the `plan` jsonb at snapshot time (needed so restore brings back collections, components and themes).
- `file_revisions.author = system` for restores.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/messages` | editor | `{ content, attachments?: assetId[], selectedElement?, error? }` → `{ messageId, runId }` (an `agent.chat` run) |
| POST | `/api/projects/:id/session` | editor | → `{ messageId }` (session_break) |
| GET | `/api/projects/:id/snapshots` | viewer | → `{ snapshots: [{ id, label, createdAt, runId }] }` |
| POST | `/api/projects/:id/snapshots/:snapshotId/restore` | editor | → `{ snapshotId (new), changedPaths[] }`; 409 if a run is active |
| (Planner tool) | `update_app` / `set_theme` | worker ctx | see `05` §3.3–§3.5 |

## Acceptance checklist
- [ ] "Add a welcome screen for the app" adds a new first frame. The app opens on it in the interactive preview, and its button leads to the previous first screen.
- [ ] "Add a settings screen with currency, dark mode and notification preferences" adds a tab. The tab bar updates and **no existing screen file gets a new revision**.
- [ ] "Make the cards rounder" changes one component file, and every screen using it updates.
- [ ] "Remove the X screen" removes the frame and leaves no broken links.
- [ ] A question gets a chat answer, with no tool call and no new revisions.
- [ ] A seeded runtime error shows a badge; **Fix with AI** fixes it and the badge clears.
- [ ] A pasted error (the video's sign-up error pattern, seeded) gets fixed.
- [ ] Asking for screen B after screen A was added doesn't re-add A. After **Start new session**, the Planner still knows the app's screens.
- [ ] While a change runs, the edited screen keeps showing its old code, then swaps in the new code once it's validated. A change that fails validation twice leaves the old code in place, with a note.
- [ ] **Restore** brings back the exact files, screens and plan of an earlier snapshot, including a screen deleted later.
- [ ] **Security:** `selectedElement` and `error` payloads are Zod-validated and size-capped (code ≤ 4 KB, stack ≤ 8 KB) and passed to the model as quoted data. `update_app` server checks reject unknown ids and over-limit changes.
- [ ] **AuthZ:** viewers can't post messages, start sessions or restore (403); non-members get 404.
- [ ] Edit evals pass, including "add screen" and the two-turn "don't re-add" case.

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 09 only: chat changes through the Planner's
update_app / set_theme tools, the change pipeline, Fix with AI, Start new session, and
snapshot restore.

Read first:
- docs/plan/05-ai-agent-design.md (§3 Planner incl. §3.2 rules, §3.3 tools, §3.5 AppChange;
  §4.2 change pipeline; §5 rule blocks; §6 validation + one repair; §9 sessions)
- docs/plan/06-generated-app-template.md (§2–§3 ownership: layouts are platform-written;
  §6 additive collections; §8 components)
- docs/plan/13-reference-implementation.md (R1–R4, R6, R10)
- docs/plan/03-data-model.md (projects.plan, messages, runs.step/lease, project_snapshots)
- docs/plan/phases/phase-09-chat-edits.md (this phase)
- Existing code: Planner (Phase 06), generators + validator + agent-step handler (Phase 08)

Key requirements:
- The Planner MUST be able to add, edit, rename and remove screens and components after the
  first build (the original agent refused). Add the eval.
- Act only on the latest message; the current plan is the truth (no re-adding old requests).
- No file tools for the model. update_app input is validated by server code, then the
  pipeline regenerates only affected files. Old code stays live until new code passes.

Implement, one commit per phase-doc step:
1. Planner static prompt additions + PROJECT STATE builder (mid-conversation system message).
2. update_app (AppChange schema + server checks) and set_theme.
3. Change steps in agent-step.ts: data (additive diff), component:<Name>, screen:<id>,
   remove, deterministic layouts, snapshot, build_summary.
4. selectedElement payload + Fix with AI + pasted-error hints.
5. Start new session (session_break).
6. Snapshot list + restore (incl. plan jsonb; new revisions only).
7. Edit evals.

Do NOT build: the inspector UI (Phase 10; only the selectedElement schema), Snack, export,
credit enforcement (log estimated cost only). Every route uses requireProjectAccess().

Definition of done = Acceptance checklist in the phase doc.
Finish with lint, typecheck, tests, eval results, and a short summary.
```

## Pitfalls & notes
- **Cache reuse:** keep the Planner's system prompt static. All state goes in the mid-conversation system message right before the latest user turn. Check `cache_read_input_tokens` on the second message of a session.
- **`request` quality decides edit quality.** The Screen Builder sees only `request` plus the current file. The Planner prompt must say to spell out exact copy, what moves where, and what to keep. Eval a few vague requests.
- **Over-rewriting:** the Screen Builder returns whole files for edits. Track "lines changed outside the request's target" in evals (diff vs. previous revision) and tighten the edit rule if it drifts.
- **Tab layout writer:** icon names must exist in the Ionicons glyph map; validate them before writing.
- **Restore + Snack/GitHub:** a restore is a normal new revision set, so Phase 11 (live Snack updates) and Phase 14 (GitHub push) need no special handling.
- **Billing later:** a change round costs 5 credits and each new screen 10 (`11`). Phase 18 charges before the pipeline starts and refunds unbuilt work. Keep per-change model cost in eval output.
