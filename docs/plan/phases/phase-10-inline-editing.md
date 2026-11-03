# Phase 10 — Inline Visual Editing (Inspector, Live Overrides, Source Splicing)

| | |
|---|---|
| Video chapter(s) | Ch.13 "Inline Editing with AI" (1:49:41); demo at 3:31. Reference repo: Prompt 8.4 (popover vs sidebar), 8.5 (inline editing) |
| Branch | `feat/10-inline-editing` |
| Depends on | Phase 07 (preview runtime + `afid` instrumentation), Phase 08 (one-element-per-line generated code), Phase 09 (`selectedElement` chat payload) |
| Effort (solo dev + Claude Code) | ~4–5 days |
| New env vars | none |

## Goal
The user clicks the **Edit** icon on a canvas frame to enter Edit mode. Clicks then select elements instead of pressing them. A **quick-edit popover** opens near the element, with a header and categorized, labelled icon sections: text, colors, typography, spacing, layout, borders, background. An **Advanced** mode opens the full panel next to the frame. Changes show **instantly in every frame** as live overrides on the running code, with no reload and no regeneration. Shortly after, they're **spliced into the source file** by deterministic code: no LLM, no credits, undoable. **Edit with AI** attaches the selected element to the chat prompt box, so "change this button's background to blue" is understood (Phase 09).

## What we do differently from the video (and why)
- **No LLM for manual edits.** Codemods are pure code, so edits are free and instant.
- **Source stays clean.** `afid` stamps exist only in the preview transpile step (`07` §5). Exported code has no editor noise.
- **Theme-aware colors first.** The picker offers theme tokens (`colors.primary`, `colors.surface` …) and writes the token reference. A custom hex is allowed. It's written as a **literal on that element and doesn't follow theme switches**, and the inspector says so.
- **Dynamic text is read-only**, as in the video ("dynamic value cannot be changed"). The panel explains and offers **Edit with AI**.

### Reference app notes
- **R9:** live override first, then splice into source. Frames keep running the code they compiled, so stamps stay valid, and each save re-applies **all** pending edits to that same base revision. That avoids line/column drift entirely.
- **R9:** generated code has **one element per line with its own named style** (`06` §9). That makes selection → source mapping reliable and keeps StyleSheet entries effectively unshared. The reference app stamped `file:line:type` (no column), so two same-type elements on one line were AI-only. We stamp `line:col`.
- **R10:** person-chosen literal colors are kept by later AI edits, and the validator accepts them because they're already in the file.
- **R6:** an element inside a shared component (`components/<Name>.tsx`) edits the **component**, so the change appears everywhere it's used. The inspector shows "Used by N screens".
- Image actions (Replace, Generate AI image, Apply AI transformation) are part of this inspector, and Phase 12 implements the AI parts.

## Steps

### 1. Edit mode in the runtime (`preview-runtime/src/inspect.ts`)
- `inspect.mode {enabled}` → capture-phase `pointermove`/`click` listeners. On hover, draw an outline overlay inside the iframe (`pointer-events:none`) around the nearest `[data-afid]`, with a small component-name label. Clicks don't reach app handlers while Edit mode is on.
- `element.select` → `{afid, rect, component, props: {style (flattened, resolved), text?}, isDynamic}`.
- `edit.override {afid, style?, text?, imageUri?}` (`07` §4): merge the override onto that element at render time (a wrapper in the instrumented `createElement`) and re-render only that element. **Broadcast overrides to every frame** showing the same file, plus the interactive preview.
- The selection persists until the file gets a new **agent** revision (frames recompile) or the page reloads.

### 2. Inspector UI (`components/inspector/`)
- **Quick-edit popover** anchored to the element rect (flips at frame edges). Its header shows the element label and file kind ("Screen" / "Component · used by 3 screens"), plus tabs with icons: Content · Colors · Type · Spacing · Layout · Border · Background. An **Advanced** button docks the full panel beside the frame.
- Sections:
  - **Content**: literal text (`JSXText` only), `accessibilityLabel`, `placeholder`, and image source (URL field; Phase 12 adds Generate / Transform).
  - **Colors / Background**: token swatches from the active theme first, then custom hex (with "won't follow theme switches").
  - **Type**: fontSize, fontWeight, lineHeight, letterSpacing, textAlign, textTransform.
  - **Spacing**: padding/margin (all and per side), gap.
  - **Layout**: flexDirection, justifyContent, alignItems, flex, width/height, position.
  - **Border**: borderWidth, borderColor, borderRadius (8-pt presets), shadow presets (iOS shadow* + Android elevation), opacity.
- Each control shows where its value comes from: **named style** (the normal case), **inline**, **token**, or **dynamic** (disabled).
- **Edit with AI** fills the chat input with a `selectedElement` chip `{file, line, col, label, screenId, code}` (Phase 09 schema). The chip is also attached automatically whenever an element is selected and the chat box is focused, as in the reference app.

### 3. Server-side resolution (`lib/codemods/resolve.ts`)
- Input `{afid, baseRevisionId}`. Parse the base revision with `@babel/parser` (`typescript`, `jsx`) through **recast** (formatting-preserving) and find the `JSXOpeningElement` at `line:col`.
- Classify `style`:
  - `styles.x` from `StyleSheet.create` in the same file → **named** (`sharedCount` = usages of `styles.x`; normally 1)
  - an inline object → **inline**
  - an array → **mixed**
  - anything else → **dynamic**
- Classify text: a single `JSXText` is editable; a `JSXExpressionContainer` is dynamic.

### 4. Codemod operations (`lib/codemods/ops.ts`)
Pure functions `(source, target, op) → newSource`, unit-tested.

| op | payload | behaviour |
|---|---|---|
| `setStyle` | `{prop, value}` | Set on the element's named style entry. If `sharedCount > 1`, split it into a new named entry for this element (keeps one style per element). Inline/mixed → set inline. |
| `removeStyle` | `{prop}` | inverse |
| `setStyleToken` | `{prop, token}` | writes `colors.<token>`. `import { colors } from "@/theme"` already exists (contract), and tokens work inside `StyleSheet.create` (`06` §5). |
| `setText` | `{text}` | replace a literal `JSXText` (escape `{`, `}`, `<`); rejects dynamic |
| `setProp` | `{name, value}` for an allowlist: `accessibilityLabel`, `numberOfLines`, `placeholder`, `source.uri` | string/number literals only |

- Value validation: fontSize 6–96, spacing 0–200, enums for layout props, `^#[0-9a-fA-F]{6}$` for hex, and tokens must exist in the theme token list.

### 5. Persist: splice edits into the base (`PUT /api/projects/:id/files`)
- The client keeps, per file, `{ baseRevisionId, edits: [{afid, op}] }` for edits since the frames last compiled.
- Debounced at 400 ms, it sends **all** pending edits for that file with `baseRevisionId`. The server re-applies the full list to the base revision's source (deterministic). It re-parses (must succeed) and runs the syntax, import and theme validators: literals are allowed because they're person-chosen, and **R10** keeps them on later AI edits. Then it writes a new revision with `author: "user_inline"`.
- **Conflict rule:** if the file's current revision isn't the base or a `user_inline` descendant of it (the agent changed the file meanwhile), return `409 stale`. The client drops overrides, frames recompile, and the user re-selects.
- Idempotent per `clientEditId`. Snack (Phase 11) and the canvas receive the new revision as a normal `file.written`. Frames in Edit mode **don't recompile** on `user_inline` revisions they produced themselves; overrides already show it.

### 6. Undo / redo
- Per-session stack of edit lists. Undo removes the last edit from the pending list (or from a new list based on the latest base), re-persists, and updates overrides. ⌘Z / ⌘⇧Z while the inspector is focused.
- Older history goes through snapshots (Phase 09).

### 7. Snapshot policy
- A "Manual edits" `project_snapshots` row when Edit mode is turned off, or after 2 minutes idle, if any `user_inline` revisions exist since the last snapshot.

## Files to create / change
```
preview-runtime/src/inspect.ts             (hover overlay, select, edit.override application)
preview-runtime/src/bridge.ts              (+ edit.override)
lib/codemods/{resolve.ts,ops.ts,apply.ts,print.ts,tokens.ts}
lib/codemods/__tests__/*.test.ts
app/api/projects/[projectId]/files/route.ts           (PUT: splice pending edits onto base)
app/api/projects/[projectId]/elements/resolve/route.ts
components/inspector/{QuickEditPopover.tsx,AdvancedPanel.tsx,sections/*.tsx,TokenColorPicker.tsx,LiteralColorNote.tsx}
components/canvas/PhoneFrame.tsx           (+ Edit icon per frame)
hooks/use-inline-editor.ts                 (pending edits per file, overrides broadcast, persist, undo)
```

## Database changes
- None new. Uses `file_revisions.author = user_inline` and `project_snapshots` (label "Manual edits").
- Manual edits write **no** `ai_usage` rows and no `credit_ledger` entries.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/elements/resolve` | viewer | `{ afid, baseRevisionId }` → `{ kind: "named"\|"inline"\|"mixed"\|"dynamic", sharedCount, fileKind: "screen"\|"component", usedBy[], style, text?: {value, editable}, props }` |
| PUT | `/api/projects/:id/files` | editor | `{ path, baseRevisionId, clientEditId, edits: [{afid, op}] }` → `{ revisionId }`; 400 invalid value; 409 stale; 422 dynamic target; 403 platform-owned file |

## Acceptance checklist
- [ ] The Edit icon on a frame turns on Edit mode. Hover outlines appear, a click selects, and the quick popover shows current values. **Advanced** opens the full panel.
- [ ] Changing a heading's text, color (token), font size and padding updates **all frames** showing that screen within ~100 ms, with no app restart (state such as a toggled habit is kept).
- [ ] After the debounce, the file has a new `user_inline` revision with a minimal, well-formatted diff, and nothing else in the file changes.
- [ ] Ten rapid edits to one element produce correct source (all applied to the same base) and no stale-position errors.
- [ ] Selecting an element inside a shared component shows "Used by N screens". Editing it changes the component file, and every screen using it updates.
- [ ] Dynamic text (`{habit.name}`) is read-only, with **Edit with AI**.
- [ ] **Edit with AI** attaches the element to chat. "Change this button's background to blue" changes only that element, with a literal hex (Phase 09).
- [ ] A custom hex color survives a later AI edit of that screen, and the inspector notes that it doesn't follow theme switches. Token colors do follow theme switches.
- [ ] If the agent changes the file while edits are pending, the server returns 409 and the UI recovers (re-select prompt), with no corrupted source.
- [ ] Undo/redo works for the last 20 manual edits in the session.
- [ ] No `ai_usage` or credit rows are created by manual edits.
- [ ] **Security:** ops accept only allowlisted props and validated values. `setText` with `}</Text><Evil/>{` is escaped and round-trips as text. Platform-owned files return 403.
- [ ] **AuthZ:** viewers can resolve but not edit (403); non-members get 404.
- [ ] Codemod unit tests cover: named style (shared 1 and shared 3 → split), inline, array, conditional (read-only), token write, JSX text with entities, edits re-applied to a base in sequence.

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 10 (inline visual editing) only.

Read first:
- docs/plan/07-preview-runtime.md (§4 protocol incl. edit.override, §5 instrumentation +
  "live override, then persist")
- docs/plan/06-generated-app-template.md (§3 ownership, §5 colors/tokens, §8 components,
  §9 one element per line)
- docs/plan/13-reference-implementation.md (R6, R9, R10)
- docs/plan/phases/phase-09-chat-edits.md (selectedElement payload)
- docs/plan/phases/phase-10-inline-editing.md (this phase)
- Existing: preview-runtime/src/{bridge.ts,inspector-plugin.ts}, lib/vfs, lib/validate

Implement, one commit per phase-doc step:
1. Edit mode inside the runtime: hover overlay, select, edit.override applied to every frame
   without restarting the app.
2. Quick-edit popover (header + categorized icon tabs) and Advanced panel; tokens first;
   custom hex marked as not following theme switches.
3. lib/codemods/resolve.ts (@babel/parser + recast, formatting-preserving).
4. lib/codemods/ops.ts with strict value validation + unit tests.
5. PUT /files: re-apply ALL pending edits to baseRevisionId, validate, save user_inline
   revision; 409 if the agent changed the file meanwhile. /elements/resolve.
6. Undo/redo; "Manual edits" snapshot policy.

Rules: NO LLM calls for manual edits, no credit/ai_usage rows. Edit with AI only attaches a
selectedElement chip to chat (Phase 09 handles it). Dynamic values are read-only. Never edit
platform-owned files. requireProjectAccess(): viewer for resolve, editor for PUT.
Do NOT build: AI image generation/transform (Phase 12; just the image URL field), Snack, export,
a free-form code editor.

Definition of done = Acceptance checklist in the phase doc.
Finish with lint, typecheck, tests, and a short summary with before/after diffs of 3 sample edits.
```

## Pitfalls & notes
- **Don't recompile frames on your own inline revisions.** That would reset app state and invalidate stamps. Recompile only on agent revisions or reload (R9).
- **recast + Babel TS:** use `recast/parsers/babel-ts`, or wrap `@babel/parser` with the same plugin set as the validator. Mismatched plugins cause parse failures on valid files.
- **Shadows** differ between iOS (`shadow*`), Android (`elevation`) and web (`boxShadow` via react-native-web). Presets write all of them.
- **Old projects** whose screens don't follow one-element-per-line still work: two same-type elements on one line remain selectable by column, but splicing may touch the shared line. Show "Edit with AI" as the safer option there.
- Keep the inspector focused. Arbitrary-prop editing and a code editor belong in a later phase, if at all.
