# Phase 10 — Inline Visual Editing (Inspector + Deterministic Codemods)

| | |
|---|---|
| Video chapter(s) | Ch.13 "Inline Editing with AI" (1:49:41); demo at 3:31 |
| Branch | `feat/10-inline-editing` |
| Depends on | Phase 07 (preview runtime + `afid` instrumentation), Phase 09 (`element_ref` chat payload) |
| Effort (solo dev + Claude Code) | ~4–5 days |
| New env vars | none |

## Goal
The user toggles **Edit mode** on a frame, hovers to see element outlines, clicks an element, and edits it in an inspector panel: text, typography, colors (theme tokens first), spacing, layout, border, radius and shadow. Each change is applied as a **deterministic AST transform** on the source file. No LLM is involved, no credits are spent, and it's undoable. **Edit with AI** sends the selected element into chat as context for the Editor agent (Phase 09).

## What we do differently from the video (and why)
- **No LLM for manual edits.** The video uses the same idea ("you don't need AI every time"). We make it explicit: the codemod is pure code, and edits are free and instant.
- **Source stays clean.** The `afid` attributes exist only in the preview transpile step (`07-preview-runtime.md` §5), so exported code has no editor noise.
- **Theme-aware colors.** The color picker offers theme tokens first (`colors.primary`, `colors.surface` …) and writes `theme.colors.x` references. Raw hex is allowed but flagged, because it breaks theme switching.
- **Shared styles are handled honestly.** If the element's style comes from a `StyleSheet.create` entry used by several elements, the inspector warns "This style is shared by N elements". The user chooses between editing the shared style and overriding just this element.
- **Dynamic text is read-only**, the same as the video ("dynamic value cannot be changed"). The panel says why and offers **Edit with AI** instead.

## Steps

### 1. Inspect mode in the runtime
- `inspect.mode {enabled}` → the runtime adds capture-phase `pointermove`/`click` listeners. On hover it draws an outline overlay *inside* the iframe (absolutely positioned div, `pointer-events:none`) around the nearest `[data-afid]`, with a small label showing the component name.
- `element.select` sends `{afid, rect, component, props: {style (flattened, resolved), text?}, isDynamic}` to the parent. Clicks don't trigger app handlers while inspect mode is on.
- Selection persists across hot swaps if the same `afid` still exists. If it doesn't, the selection clears with a toast.

### 2. Inspector panel UI (`components/inspector/`)
- It opens docked next to the selected frame (the video's "advanced edit" container), with a compact quick-edit popover near the element and an **Expand** button for the full panel.
- Sections:
  - **Content**: text (only for literal `JSXText`), image source for `Image`/`expo-image`, `accessibilityLabel`.
  - **Typography**: fontSize, fontWeight, lineHeight, letterSpacing, textAlign, textTransform, color.
  - **Colors**: background color and text color. Token swatches from the active theme come first, then a custom hex picker.
  - **Spacing**: padding/margin (all sides plus individual sides), gap.
  - **Layout**: flexDirection, justifyContent, alignItems, flex, width/height (number, % or auto), position.
  - **Border & effects**: borderWidth, borderColor, borderRadius (token radii first), shadow (preset sm/md/lg mapped to cross-platform shadow + elevation), opacity.
- Each control shows whether the value is **inline**, comes from a **StyleSheet entry** (shared ×N), is a **theme token**, or is **computed/dynamic** (disabled).
- **Edit with AI** inserts an `element_ref` chip into the chat input with the payload defined in Phase 09.

### 3. Server-side element resolution (`lib/codemods/resolve.ts`)
- Input: `{path, line, col, sha}`. Load the current revision; if the sha differs, return `409 stale` and have the client re-select after the hot swap.
- Parse with `@babel/parser` (`typescript`, `jsx`, `tokens: true`) via **recast**, so printing keeps the original formatting. Find the `JSXOpeningElement` at `line:col`.
- Classify the `style` prop:
  - `style={{...}}` inline object → **inline**
  - `style={styles.card}` with a resolvable `StyleSheet.create` entry in the same file → **sheet** (count other usages of `styles.card` in the file → `sharedCount`)
  - `style={[styles.a, {…}]}` array → mixed (edit the inline part, or append one)
  - anything else (function call, conditional, imported style) → **dynamic** (read-only)
- Classify the text child: a single `JSXText` → editable. A `JSXExpressionContainer` → dynamic.

### 4. Codemod operations (`lib/codemods/ops.ts`)
All operations are pure functions `(source, target, op) → newSource`, with unit tests.

| op | payload | behaviour |
|---|---|---|
| `setStyle` | `{prop, value, scope: "element" \| "shared"}` | `element`: set on the inline object (convert `styles.x` → `[styles.x, {prop: value}]` if needed). `shared`: set on the StyleSheet entry. |
| `removeStyle` | `{prop, scope}` | inverse |
| `setStyleToken` | `{prop, token: "colors.primary"}` | writes `theme.colors.primary`. Ensures `const theme = useTheme()` exists in the component (adds the import and hook call if missing; if styles are module-level `StyleSheet.create`, it falls back to an element-scoped inline override). |
| `setText` | `{text}` | replace a literal `JSXText` (escape `{`, `}`, `<`). Rejects dynamic text. |
| `setProp` | `{name, value}` for an allowlist: `accessibilityLabel`, `numberOfLines`, `placeholder`, `source` (image URL) | string/number literals only |

- After applying the op: re-parse (it must succeed), run the `syntax` and `theme` validators on the file, and write the new revision with `author: "user_inline"`. Respond with `{revisionId, newAfid}`. The line/col may shift, so the server recomputes the target's new location.
- Value validation: numeric ranges (fontSize 6–96, spacing 0–200), enums for layout props, `^#[0-9a-f]{6}$` for hex, and tokens must exist in the `Theme` type.

### 5. Edits API and batching
- The client debounces slider and color changes (250 ms). It sends the latest value only, plus a `clientEditId` for idempotency.
- **Optimistic preview:** the runtime applies the style immediately via a temporary override keyed by `afid` (`inspect.preview {afid, style}`). The confirmed revision then replaces it via `file.update`.

### 6. Undo / redo
- A per-session client stack of `{path, beforeRevisionId, afterRevisionId}`. Undo calls `POST /edits/undo {revisionId}`, which writes a new revision copying `beforeRevisionId`'s content, but only if the current revision is still `afterRevisionId`. Otherwise it returns 409 and asks the user to use History (Phase 09).
- Keyboard: ⌘Z / ⌘⇧Z while the inspector is focused.

### 7. Snapshot policy for manual edits
- Don't snapshot on every keystroke. Create a `project_snapshots` row labelled "Manual edits" when inspect mode is turned off or after 2 minutes idle, if any `user_inline` revisions exist since the last snapshot.

## Files to create / change
```
preview-runtime/src/inspect.ts             (hover overlay, select, temporary style overrides)
preview-runtime/src/bridge.ts              (+ inspect.preview message)
lib/codemods/{resolve.ts,ops.ts,print.ts,tokens.ts,index.ts}
lib/codemods/__tests__/*.test.ts           (fixtures per style shape)
app/api/projects/[projectId]/edits/route.ts
app/api/projects/[projectId]/edits/undo/route.ts
app/api/projects/[projectId]/elements/resolve/route.ts
components/inspector/{InspectorPanel.tsx,QuickEdit.tsx,sections/*.tsx,TokenColorPicker.tsx,SharedStyleWarning.tsx}
components/canvas/PhoneFrame.tsx           (+ Edit mode toggle per frame)
lib/preview/useInspector.ts
```

## Database changes
- None new. Uses `file_revisions.author = user_inline` and `project_snapshots` (label "Manual edits").
- Manual edits write **no** `ai_usage` rows and no `credit_ledger` entries.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/elements/resolve` | viewer | `{ afid, sha }` → `{ kind: "inline"\|"sheet"\|"mixed"\|"dynamic", sharedCount, style, text?: {value, editable}, props }` or `409 { reason: "stale" }` |
| POST | `/api/projects/:id/edits` | editor | `{ afid, sha, clientEditId, op }` (op per §4) → `{ revisionId, newAfid }`; 400 invalid value; 409 stale; 422 dynamic target |
| POST | `/api/projects/:id/edits/undo` | editor | `{ path, beforeRevisionId, afterRevisionId }` → `{ revisionId }` or 409 |

## Acceptance checklist
- [ ] Toggle Edit mode on a frame → hover outlines appear → click selects → the inspector shows current values.
- [ ] Changing a heading's text, color (token), font size and padding updates the frame within ~300 ms. The source diff shows minimal, well-formatted changes, and the rest of the file is untouched (recast preserves formatting).
- [ ] Selecting an element whose style comes from a StyleSheet entry used 3 times shows "shared by 3 elements". Both scopes work as described.
- [ ] Dynamic text (`{habit.name}`) is shown read-only with an **Edit with AI** button.
- [ ] **Edit with AI** puts an element chip in chat. Sending "change icon to leaf" changes only that element (Phase 09 Editor).
- [ ] Undo/redo works for the last 20 manual edits in the session.
- [ ] Theme switching still works after token-based edits. A raw-hex edit is flagged with a warning badge.
- [ ] No `ai_usage` or credit rows are created by manual edits.
- [ ] **Security:** codemod ops accept only allowlisted props and validated values. String values can't inject code (e.g. `setText` with `}</Text><Evil/>{` is escaped and round-trips as text). Edits on platform-owned files return 403.
- [ ] **AuthZ:** viewers can resolve but not edit (403); non-members get 404.
- [ ] Codemod unit tests cover: inline object, StyleSheet entry, style array, conditional style (read-only), missing `useTheme` (hook inserted), JSX text with entities.

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 10 (inline visual editing) only.

Read first:
- docs/plan/07-preview-runtime.md (§4 bridge protocol, §5 element instrumentation)
- docs/plan/06-generated-app-template.md (§3 ownership, §5 theme tokens)
- docs/plan/phases/phase-09-chat-edits.md (element_ref payload)
- docs/plan/phases/phase-10-inline-editing.md (this phase)
- Existing: preview-runtime/src/{bridge.ts,inspector-plugin.ts}, lib/vfs, lib/validate

Implement, one commit per phase-doc step:
1. Inspect mode inside the runtime (hover overlay, select, temporary style overrides).
2. Inspector panel UI with sections per the doc; theme tokens first for colors/radii.
3. lib/codemods/resolve.ts using @babel/parser + recast (preserve formatting).
4. lib/codemods/ops.ts: setStyle/removeStyle/setStyleToken/setText/setProp with strict
   value validation, plus unit tests for every style shape listed in the checklist.
5. POST /edits (debounced client, clientEditId idempotency, stale-sha 409),
   /elements/resolve, /edits/undo.
6. Undo/redo stack; "Manual edits" snapshot policy.

Rules: NO LLM calls for manual edits and no credit/ai_usage rows. Edit with AI only attaches
an element_ref chip to chat (Phase 09 handles it). Dynamic values are read-only. Never edit
platform-owned files. Every route uses requireProjectAccess() (viewer for resolve, editor for edits).
Do NOT build: Snack/QR, export, a free-form code editor.

Definition of done = Acceptance checklist in the phase doc.
Finish with lint, typecheck, tests, and a short summary with before/after diffs of 3 sample edits.
```

## Pitfalls & notes
- **Line/col drift:** every write can shift positions. Always return `newAfid` and re-key the selection, and always send `sha` with edits.
- **recast + Babel TS:** configure recast with `parser: require("recast/parsers/babel-ts")`, or wrap `@babel/parser` with the same plugin set the validator uses. Mismatched plugin sets cause parse failures on valid files.
- **`useTheme()` in module-level StyleSheets:** tokens can't be referenced inside `StyleSheet.create` at module scope. Hence the inline-override fallback in `setStyleToken`. Document in `CONTRACT.md` that generated components should build theme-dependent styles inside the component (`useMemo(() => makeStyles(theme), [theme])`). That makes token edits on shared styles possible.
- **Shadows** differ between iOS (`shadow*`) and Android (`elevation`). Use presets that write both. Web preview uses `boxShadow` via react-native-web.
- **Components from `components/ui`:** their internals are platform-owned. Selecting one edits the *usage site* props/style only, never the library file. The resolve endpoint must target the JSX element in the agent-owned file.
- Keep the inspector focused. Arbitrary-prop editing and a code editor belong in a later phase, if at all.
