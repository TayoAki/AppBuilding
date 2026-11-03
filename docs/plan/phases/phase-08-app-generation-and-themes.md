# Phase 08 — App Generation (Generator Pipeline) & Theme Packs

| | |
|---|---|
| Video chapter(s) | Ch.11 "Generate Mobile App with AI" (1:11:10), first half up to ~1:27:00; reference repo Ch.6 (+ Ch.8 components, Ch.11 shared tab layout) |
| Branch | `feat/08-app-generation-and-themes` |
| Depends on | Phase 05 (jobs, step chaining), Phase 06 (Planner, confirmed plan), Phase 07 (template, renderers, VFS, preview runtime) |
| Effort (solo dev + Claude Code) | ~6–8 days (was 5–7; adds the sandbox service skeleton for smoke renders, ADR-003 #18), plus ongoing prompt tuning with evals |
| New env vars | `AGENT_SCREEN_CONCURRENCY` (default `1`; with `2`, the reference screen is built alone first, then the rest two at a time; ADR-003 #11). `SANDBOX_URL`, `SANDBOX_JOB_SECRET` (worker → sandbox service). `AI_BUILDER_MODEL` exists from Phase 06. |

## Goal
When the plan is confirmed (button or `confirm_plan`), a chain of `agent.step` jobs turns it into a working Expo app (`05-ai-agent-design.md` §4.1):
1. **Foundation** returns JSON. The platform turns it into 5 themes (with derived tokens and fixed contrast) plus the app's collections and sample records.
2. **Component Designer** returns 5–10 shared components.
3. **Screen Builder** writes one streamed `.tsx` per screen.

Each output is validated, with **one repair round** if needed. The platform renders the layouts and routes, takes a snapshot, and the app is ready. Progress streams live: each canvas frame goes placeholder → shimmer → real screen, and the Code tab shows the current file being written. A **theme picker** (and the Planner's `set_theme` tool) switches between themes instantly, with no screen rebuilt.

## What we do differently from the video (and why)
- **The background job is already in place.** The video moved generation into CloudHike Jobs + an ngrok webhook (~1:20:46) after hitting serverless timeouts. Our steps are pg-boss jobs pulled by the worker (Phase 05). No tunnel.
- **Single-purpose generators instead of one free-form agent.** Each step is one structured output or one file, checked by code. That gives real progress, per-screen retry, and small cacheable prompts.
- **Everything derived from data is rendered by the platform**: themes, schema, seed, tab layout and routes. The agent can't break them.
- **Scrolling and pinned headers are in the screen rules from the start.** The video had to ask for scrollable screens afterwards (~1:16:00).
- **Contrast is fixed deterministically** in `lib/theme/derive.ts`, not by asking the model to "keep text readable".
- **Usage is metered** on every response. The credit charge and refund hooks exist (no-ops until Phase 18).

**Reference app notes** (`13-reference-implementation.md`)
- R1: Foundation → Component Designer → Screen Builder, each validated with one repair, is the shipped pipeline. A failed screen doesn't lose finished ones.
- R5: screens use `useCollection` / `useStoredValue` / `useAuth` from `@/lib/backend`, and records come from collections with sample data, never hardcoded.
- R6: the component library is generated **before** any screen. Screens see only each component's props type and purpose.
- R7: the shared tab layout comes from `screen.tab`, so screens never draw a tab bar.
- R8: `import { colors } from "@/theme"`, with no color literals. In the preview a theme switch only sets CSS variables.
- R9: one element per line with its own named style. Phase 10's inline editing depends on it.
- R4: the default Sign In screen is built like any other screen, using `useAuth`.

## Steps

### 1. Generator helpers (`lib/ai/generate.ts`)
- `generateJson(schema, {system, user, effort})`: streams with `output_config.format = zodOutputFormat(schema)` and validates the final text with Zod. If the installed SDK's `messages.parse` supports streaming, use it; check the docs. Records `ai_usage`.
- `generateFile({system, user, effort, onDelta})`: streams plain text, calls `onDelta` for live code, strips an accidental leading/trailing code fence defensively, and returns the file text. Checks `stop_reason` (`refusal`, `max_tokens` → treated as a validation problem → repair).
- Both use `AI_BUILDER_MODEL`, adaptive thinking (`display: "summarized"`), explicit effort (`05` §1), fallbacks, and `max_tokens: 64000`.

### 2. Prompt modules (`lib/ai/prompts/`)
- `rules.ts`: the shared blocks from `05` §5 (THEME_RULES, DATA_RULES, SCREEN output/code/components/data/everything-works/layout/design/keep-editable, TAB rules, IMAGE_RULES, COMPONENT rules), **written in our own words**. One source, so the generators can't drift apart.
- `foundation.ts`, `components.ts`, `screen.ts`, `repair.ts`: static system prompts (cached) + request builders.
- `app-context.ts`: `appContext(plan)` renders app, design, collections (with one sample record each), component props APIs and routes. It's the shared prefix for every screen step, with a `cache_control` breakpoint after it (`05` §7). The screen spec, tab rules, style reference and (for edits) the current file come after the breakpoint.
- Snapshot-test every assembled request, so accidental byte changes to cached prefixes show up in review.

### 3. Step dispatcher (`worker/handlers/agent-step.ts`)
`agent.step {runId, projectId, userId, step}`, wrapped as a **step job** (Phase 05: never finishes the run itself; `failRun` only on the final attempt or a non-retryable error) → `claimStep` → dispatch by step name → `advance(nextStep)`:
```
foundation → components → screen:<referenceScreenId> → screen:<rest, in plan order> … → finish
```
- `referenceScreenId` = the first screen in plan order that **isn't `sign-in`** (content-rich, so it makes a good style reference). The remaining screens follow in plan order, `sign-in` included. With `AGENT_SCREEN_CONCURRENCY=2`, the reference screen is built alone and the rest two at a time (the dispatcher fans out with one `agent.step` per screen and a join counter in `runs.steps`).
- `runs.steps` records each step's status (`queued`/`running`/`done`/`failed`). Resume and refunds (Phase 18) read it.
- `finish` renders platform files (already derived from `projects.plan`), runs the render smoke test (step 8), calls the backend-setup hook (Phase 13, no-op now), creates a `project_snapshots` row (`files`, `screens`, `label: "Initial build"`), writes a `build_summary` message (template text: "Built 4 screens: Sign In, Today, …"), sets status `ready`, and emits `run.done`.
- Each step checks `isCanceled` and `assertLease` before writing, and skips screens already `ready` (resume after crash/retry, never charged twice).

### 4. Foundation step
- Schema `FoundationSchema = { themes: Theme[5], collections: Collection[] }` (base tokens per theme; `Collection` per `06` §6).
- Request: app name, tagline, summary, target users, confirmed design, screens with purposes and key elements.
- Validate: exactly 5 themes, the first matching `plan.design`, at least one in the opposite mode, distinct names; collections with valid names, `typeName`, field types and `references` to existing collections, 5–8 records each with ids and a value for every field of the right type, and no `id`/`createdAt`/`updatedAt` fields. On problems, one repair ("These have problems: … Write the complete JSON again").
- Then the platform: `deriveThemePackage()` (full tokens + contrast + tinted neutrals) → `projects.plan.themePackage = {themes, activeThemeId: themes[0].id}`, `projects.plan.collections = …` → re-render `theme/themes.ts`, `data/schema.ts`, `data/seed.ts` → emit `theme.ready`, `plan.updated`.
- Images (ADR-002, `11` §3): **Free projects always use placeholder URLs.** When `IMAGEKIT_ENABLED` and the project owner is on Pro, image fields may use generation URLs: at most 8 per build across all steps, each charged 4 credits and included in the build estimate (Phase 18). The wait-and-store post-processing is Phase 12; until then everything uses placeholders.

### 5. Components step
- Schema `{ components: [{name, purpose, code}] }`, 5–10 entries, PascalCase names, ordered simple-first.
- Validate each `code` with `lib/validate` (step 7), plus component rules: exported `<Name>Props` type + default export; imports only allowlisted packages, `@/theme`, `@/data/schema` (types) and **earlier** components; no `@/lib/backend` or `expo-router` imports (props only, no navigation); no tab bar; 20–80 lines (warning). Must include a button, a screen header and an empty state.
- One JSON repair round. Then write `components/<Name>.tsx` (`author: "agent"`), store `[{name, purpose}]` in `projects.plan.components`, emit `components.ready`.
- `lib/ai/component-api.ts`: extract each component's props type + purpose with an AST parse, not a regex. This is what screens see.

### 6. Screen steps
- `screen:<id>` → request = APP CONTEXT (cached) + "Build this screen (k of N)": name, purpose, key elements, links-to names, tab rules for this screen (tab → `SafeAreaView edges={["top"]}`, no tab bar; non-tab → no tab bar), and the **style reference** = the code of the first finished screen (absent for screen 1).
- Stream via `generateFile`. `onDelta` emits `file.writing {path, lines}` (throttled ≤ 4/s), and the Code tab shows the streamed text for the active file. Stream text never becomes a file revision until validated.
- Validate (step 7). On problems, one repair ("This file has problems: … Rewrite the complete file with these fixed and everything else kept. Reply with only the file.") and re-validate.
- Pass → `writeFile(screenFilePath(screen), …)` → `screens.status = ready` → `screen.done`. Fail after repair → `screens.status = error`, `screen.failed {error}`, `refundScreen()` hook (no-op until 18). The build continues with the next screen and finishes as `ready` with a summary noting the failed screen and "ask me to try again".

### 7. Validators (`lib/validate/`)
Per `05` §6:
- Syntax (`@babel/parser`).
- Import allowlist (`ALLOWED_IMPORTS.json` + `@/theme`, `@/lib/backend`, `@/data/schema`, existing `@/components/<Name>`).
- Theme discipline: no hex, rgb or named colors and no computed alphas, except `"transparent"` and literals already in the previous version or named in the request.
- Data discipline: no AsyncStorage/storage imports; `useCollection("<name>")` names exist; `add`/`update` object literals use only that collection's fields.
- Routes: `router.push/replace` and `<Link href>` literals resolve via `lib/render/routes.ts`.
- Editable markup: a warning when one line holds two JSX elements of the same type.
- Typecheck-lite: TypeScript compiler API, in-memory host with `.d.ts` stubs for `@/lib/backend`, `@/theme`, the rendered `data/schema.ts` and the project's components; `noEmit`; first 20 errors.

Output `{problems: Issue[], warnings: Issue[]}` → compact text (≤ ~4 KB) for repair prompts, plus a `validation.result` event. Unit tests: one broken-on-purpose fixture file per rule.

### 8. Render smoke test (`worker/agent/smoke.ts`)
At `finish`, the worker calls the **sandbox service** (ADR-003 #18): `POST $SANDBOX_URL/render` with a per-job HMAC token (`SANDBOX_JOB_SECRET`) and a signed storage URL of the materialized `preview` target. The sandbox service has no DB access and no platform secrets, runs non-root with an env allowlist, and restricts egress where the host allows. It serves the bundled `public/preview-runtime/` from a local static server, loads `harness.html` in Playwright + Chromium, renders each route and returns the collected `runtime.error`s. A screen that throws gets **one** repair with the runtime error text, then is re-validated and re-rendered. If it still fails, it's marked `error` as in step 6. Cap the total smoke time at 60 s.

### 9. Theme picker + `set_theme`
- **Theme dropdown** centered at the top of the canvas (where the reference app put it): theme names with swatches, the active one checked.
- `POST /api/projects/:id/theme {themeId}` (editor) → update `projects.plan.themePackage.activeThemeId` → `plan.updated` → all frames get `theme.set` (CSS variables, instant). The interactive preview keeps its state, and native targets get a re-rendered `theme/themes.ts`. No LLM, no credits.
- **Implement** the Planner tool `set_theme`. Its schema (`{themeId}` or `{newTheme: {name, mode, colors}}`) and its prompt section have existed since Phase 06, so the tool list and system text don't change (ADR-003 #1). A `newTheme` goes through `deriveThemePackage` (contrast-fixed) and is appended and activated. It emits `plan.updated {reason: "theme"}` on the project stream.
- **Interim behaviour for built apps** (until Phase 09): `update_app` is still the Phase 06 stub that answers "not available yet", so the Planner tells the person plainly. No prompt text changes now. Phase 09 replaces the stub implementation only, and the static prompt already says to ignore earlier "not available" replies (`05` §3.2 rule 3).

### 10. Chat and canvas progress UI
- Chat: a build progress card "Building 2 of 4: Today", with per-screen ticks/errors and a Cancel button (`POST /api/runs/:id/cancel`).
- Canvas: frames show `planned` → `building` (shimmer, with live line count) → the live screen on `screen.done`. A `screen.failed` frame shows an error badge with the message.
- Code tab: auto-selects the file being written and shows the stream.

### 11. Tests and evals
- Recorded fixtures (`RECORD=1`) for: foundation (valid + one repair), components (valid + an import-cycle violation → repair), 3 screens (one with a color literal → repair), smoke-test failure → repair.
- Integration: confirm → steps chain → files written → snapshot → `ready`. Kill the worker during `screen:2` → restart → resumes at `screen:2`, with no duplicate files and the charge hook called once.
- Determinism: the rendered `themes.ts`/`schema.ts`/layouts are unchanged by screen steps.
- `npm run eval -- --limit 5` (live, opt-in) reports the `10-testing-and-evals.md` §2.2 metrics, including the S2-equivalent cache hit (`cache_read_input_tokens > 0` on screens 2..N).

## Files to create / change
```
lib/ai/generate.ts
lib/ai/component-api.ts
lib/ai/prompts/{rules.ts,app-context.ts,foundation.ts,components.ts,screen.ts,repair.ts}
lib/ai/schemas/{foundation.ts,components.ts}
lib/ai/tools/planner-tools.ts              (set_theme implemented; schemas unchanged since Phase 06)
lib/validate/{index.ts,syntax.ts,imports.ts,theme.ts,data.ts,routes.ts,markup.ts,typecheck.ts,stubs/*.d.ts}
lib/theme/derive.ts                        (from Phase 07; + newTheme path)
worker/handlers/agent-step.ts              (real dispatcher)
worker/agent/{foundation.ts,components.ts,screen.ts,finish.ts,smoke.ts}
preview-runtime/harness.html               (from Phase 07)
sandbox/{index.ts,render.ts,auth.ts,Dockerfile}   (sandbox service skeleton: /render; reused by Phases 11, 16, 17)
lib/sandbox/client.ts                      (worker → sandbox: signed URL + per-job token)
app/api/projects/[projectId]/theme/route.ts
components/workspace/ThemePicker.tsx
components/chat/renderers/build-progress.tsx
components/canvas/PhoneFrame.tsx           (building / failed states)
tests/fixtures/anthropic/build/*
evals/{run.ts,prompts.jsonl}               (smoke subset)
```

## Database changes
- `projects.plan`: adds `themePackage`, `collections`, `components` (`03` §2).
- `screens.status`: `planned | building | ready | error`.
- `project_snapshots` rows at the end of each build (`files`, `screens`, `label`).
- `runs.step` / `runs.lease_id` / `runs.steps` used by the step chain. `messages.kind = build_summary`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/theme` | editor | `{themeId}` → `200 {activeThemeId}` (free, instant) |
| POST | `/api/runs/:runId/cancel` | editor | (Phase 05) now stops the step chain at the next checkpoint |
| (Planner tool) | `set_theme` | worker | `{themeId}` \| `{newTheme}` → `{activeThemeId}` |

## Acceptance checklist
- [ ] Confirming the habit-tracker plan produces Sign In + 3 screens **within the p90 build time measured in S2 and recorded in the G0 ADR** (sequential; report the concurrency-2 time too), with live frame states and live code in the Code tab.
- [ ] The app works in the interactive preview: sign in (local), add, complete and delete a habit, see the change on another tab. Lists scroll, and headers and main buttons stay pinned.
- [ ] Every screen imports shared components where they fit, draws no tab bar (tab apps) and contains no color literals (validator report is clean).
- [ ] Five themes appear in the picker. Switching is instant across all frames, with no LLM call and no `ai_usage` row. Asking in chat "make it dark" switches or adds a theme via `set_theme`, and no screen is rebuilt.
- [ ] Text is readable in all 5 themes (contrast unit test over the derived tokens).
- [ ] A screen with an injected validator failure (fixture) is repaired once and passes. A screen that still fails is marked `error` while the other screens finish.
- [ ] Killing the worker mid-build resumes from the current screen, with no duplicate files and no double charge-hook call.
- [ ] Refreshing mid-build resumes progress, with nothing missing.
- [ ] The materialized app (`export` target) runs with `npx expo start`.
- [ ] **Security:** generators can't write outside `app/<id>.tsx`, `app/(tabs)/<id>.tsx` and `components/<Name>.tsx` (vfs tests). Generated code runs only in sandboxed iframes and in the sandbox service, which rejects requests without a valid per-job token and has no DB credentials in its env (test asserts it).
- [ ] Free projects never get AI image URLs; Pro builds stay within 8 images.
- [ ] **AuthZ:** the theme endpoint and cancel return 404 for non-members and 403 for viewers.

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 08 only: the generator pipeline + themes.

Read first:
- docs/plan/05-ai-agent-design.md (entire file; §4 pipelines, §5 rule blocks, §6 validation
  and repair, §7 caching, §8 events)
- docs/plan/06-generated-app-template.md (entire file)
- docs/plan/07-preview-runtime.md (§3, §8 smoke harness)
- docs/plan/13-reference-implementation.md (R1, R4–R10)
- docs/plan/phases/phase-08-app-generation-and-themes.md (this phase)
Check the installed @anthropic-ai/sdk for streaming + structured outputs
(output_config.format / messages.parse) before writing code. Don't guess.

Build, one commit per numbered step:
1. lib/ai/generate.ts (generateJson with structured outputs, generateFile streaming text).
2. Prompt modules: rules.ts in our own words per 05 §5, app-context with a cache breakpoint,
   foundation/components/screen/repair prompts; snapshot tests of assembled requests.
3. agent.step dispatcher (stepJob wrapper, payload {runId, projectId, userId, step}) with
   claimStep/assertLease/advance: foundation → components → screen:<first non-sign-in> →
   remaining screens in plan order (optional concurrency 2 after the reference) → finish.
4. Foundation: schema, validation, one repair, deriveThemePackage, store in projects.plan,
   re-render themes/schema/seed, theme.ready.
5. Components: schema, per-component validation (ordered imports, props-only, no tab bar),
   one repair, write components/<Name>.tsx, component-api extraction via AST.
6. Screens: APP CONTEXT + spec + tab rules + style reference (first finished screen);
   stream file.writing; validate → one repair → write or mark error (refund hook no-op).
7. lib/validate per 05 §6 incl. typecheck-lite with .d.ts stubs; one failing fixture per rule.
8. Sandbox service skeleton (sandbox/, no DB access, no secrets, per-job token) and the render
   smoke test via its Playwright harness, with one repair on runtime errors.
9. Theme picker (top-center of canvas), POST /theme, implement the existing set_theme stub
   (+ newTheme); emit plan.updated. Don't change the tool list or system prompt.
10. Build progress UI (chat card, frame states, live Code tab), cancel.
11. Fixtures + integration tests (resume after worker kill), eval smoke subset.

Do NOT build: update_app / chat edits (Phase 09), inline editing (10), Snack (11), image
post-processing (12), backend (13), credit enforcement (18).
Definition of done = Acceptance checklist in the phase doc.
Finish with lint/typecheck/tests and a short eval run (--limit 3) with cost per app.
```

## Pitfalls & notes
- **Don't let stream text become a revision.** Only validated output is written; the stream is for display. This keeps frames from hot-swapping half-written code.
- **Style reference ordering:** the reference screen has no reference, so its quality sets the tone for the app. That's why the dispatcher builds the first *non-sign-in* screen first (step 3).
- **Structured outputs and long JSON:** the component library can be 10–15k output tokens. Use streaming with a large `max_tokens`, and treat `max_tokens` truncation as a repair case ("the JSON was cut off; return fewer, shorter components").
- **Cache invalidators:** component code in the APP CONTEXT would bust the cache whenever one component changes. Only the props APIs go into the APP CONTEXT, and the code never does.
- **CSS variables vs computed colors:** the theme rules forbid `colors.primary + "20"`. That would break CSS variables in the preview and theme switching on device. Keep the validator rule strict.
- **Sign-in redirect loops:** screens that redirect on `user` must wait for `loading === false`. That rule is in the screen rules, and the smoke test catches infinite navigation (add a navigation-count guard in the harness).
- **The reference app's bug to avoid:** after the first build its agent said "editing by chat isn't available", and later kept repeating it. Here the "not available yet" answer comes from the stub tool result, not the prompt. The prompt already tells the Planner to ignore earlier such replies once the tool works (Phase 09).
- Keep `MAX_SCREENS_DEFAULT=3` during development for speed and cost. Users can add screens by chat in Phase 09.
