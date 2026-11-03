# Phase 10b — Code Editing in the Workspace

| | |
|---|---|
| Video chapter(s) | New. The video and the reference app show code read-only and say "you own the code" only via export. |
| Branch | `feat/10b-code-editing` |
| Depends on | Phase 07 (VFS, Code tab), Phase 08 (validators), Phase 10 (splice/persist path, `PUT /api/projects/:id/files`) |
| Effort (solo dev + Claude Code) | ~1.5–2 days |
| New env vars | none |

## Goal
People can edit screen and component code directly in the Code tab, save, and see the change live in every frame and on connected phones. Edits are validated the same way agent output is, are versioned and revertible, and survive later AI changes.

## What we do differently from the video (and why)
- Code isn't read-only. A builder that claims "you own the code" should let you change it without exporting.
- Only **screens and components** are editable. Platform-derived files (themes, schema, seed, layouts, `lib/backend`, configs) open read-only, with a note on where to change them (theme picker, chat, plan). Otherwise the next render would silently overwrite the edit.

## Steps

### 1. Editor
- Swap the read-only viewer for **Monaco** (lazy-loaded) on editable paths, with TypeScript/TSX highlighting and a `.d.ts` bundle for the template's modules (`@/theme`, `@/lib/backend`, `@/data/schema`, components' props), so completion and red squiggles work.
- Header: path, "Edited by you / by AI · time", **Save** (⌘S), **Revert to previous**, **Diff** (Monaco diff vs the previous revision).

### 2. Save path
- `PUT /api/projects/:id/files` (exists from Phase 10) with `{ path, content, baseRevisionId }`.
- **Conflict check:** if the current revision ≠ `baseRevisionId` (the agent or an inline edit changed it), return `409` with the latest content. The UI offers a 3-way merge view or "keep mine / take theirs".
- **Run lock:** while a build/change run is writing this file (`runs.steps` says `screen:<id>` is running), saves to that file return `423 Locked`, with "AI is changing this screen". Other files stay editable.
- Validate with the same pipeline as generators (`05` §6). Syntax errors and disallowed imports block the save (the error is shown inline). Theme or data-discipline issues are **warnings** that allow saving: a person's literal colors are legitimate (`06` §5).
- Write `file_revisions(author = "user_code")` and emit `file.written`. Frames hot-swap and Snack sessions update. No credits charged.

### 3. AI awareness
- PROJECT STATE (`05` §3.1) lists screens and components the person edited by hand since the last AI change. The Planner rule: "When changing a file the person edited, keep their changes unless the request says otherwise."
- The Screen and Component Builders already receive the **current file** for edits, so hand changes are preserved by the "keep everything else" rule. Eval case: hand-edit a screen, then ask for an unrelated change, and check the hand edit survives.

### 4. Guardrails
- Editing an exported component's props type shows a warning listing the screens that use it (from the same analysis as Phase 10's "Used by N screens").
- File size cap is the same as for generators (60 KB).
- Restore from History (Phase 09) also covers user revisions.

## Files to create / change
```
components/workspace/CodeEditor.tsx        (Monaco, lazy)
components/workspace/MergeDialog.tsx
lib/vfs/editable.ts                        (isEditablePath, readOnlyReason)
lib/validate/forUserSave.ts                (error vs warning policy)
app/api/projects/[projectId]/files/route.ts   (PUT: baseRevisionId, 409/423)
lib/ai/prompts/state.ts                    (hand-edited files list in PROJECT STATE)
```

## Database changes
None. Uses `file_revisions.author = 'user_code'` and `parent_revision_id`.

## API surface
| Method | Path | Auth | Body → Response |
|---|---|---|---|
| PUT | `/api/projects/:id/files` | editor | `{path, content, baseRevisionId}` → `200 {revisionId, warnings[]}` · `409 {latest}` · `422 {errors[]}` · `423 {reason}` |
| GET | `/api/projects/:id/files/:path/revisions` | viewer | → `[{revisionId, author, createdAt}]` |

## Acceptance checklist
- [ ] Editing a screen and saving updates all frames and a connected phone without reload/rescan.
- [ ] A syntax error blocks the save with an inline marker; a literal color saves with a warning.
- [ ] Platform-derived files open read-only with a reason and a pointer to where they're changed.
- [ ] A concurrent AI change produces `409` and a working merge dialog; saving during an AI write to that file gives `423`.
- [ ] After a hand edit, an unrelated AI change to the same screen keeps the hand edit (eval case added).
- [ ] Viewers can't save (`404` on another user's project, `403`-equivalent UI for viewers).
- [ ] Revert and History restore user revisions.

## Claude Code prompt
```text
Read docs/plan/phases/phase-10b-code-editing.md, docs/plan/06-generated-app-template.md (§3
ownership), docs/plan/05-ai-agent-design.md (§3.1 PROJECT STATE, §6 validation) and
docs/plan/phases/phase-10-inline-editing.md (save path).

Implement Phase 10b exactly:
1. Monaco editor (lazy) for editable paths only (screens, components); read-only with a
   reason for platform-derived files. Add type stubs so completion works.
2. PUT /api/projects/:id/files with baseRevisionId → 409 with latest content on conflict,
   423 while a run is writing that file, 422 on blocking validation errors, warnings otherwise.
3. Save writes file_revisions(author=user_code), emits file.written; frames + Snack update.
4. Merge dialog (keep mine / take theirs / manual) and Diff/Revert.
5. PROJECT STATE lists hand-edited files; Planner rule to preserve hand edits.
6. Tests: editable-path rules, conflict/lock responses, authz, and an eval case that a hand
   edit survives an unrelated AI change.
Don't change the inline editor or generators beyond what's listed. Run lint/typecheck/tests
and summarize with a short screen recording description of the flow.
```

## Pitfalls & notes
- Don't autosave on every keystroke. Hot-swapping half-typed code floods the frames with errors. Save on ⌘S, or after 2 s idle with a valid parse.
- Keep `afid` stamps consistent: frames re-transpile after a user save, so the inspector must refresh its selection (same as after agent writes).
