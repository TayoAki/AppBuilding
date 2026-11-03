# Phase 08 — App Generation (Builder Agent) & Theme Packs

| | |
|---|---|
| Video chapter(s) | Ch.11 "Generate Mobile App with AI" (1:11:10), first half up to ~1:27:00 |
| Branch | `feat/08-app-generation-and-themes` |
| Depends on | Phase 05 (jobs & realtime), Phase 06 (planner + confirmed plan), Phase 07 (template, VFS, preview runtime) |
| Effort (solo dev + Claude Code) | ~6–8 days (plus ongoing prompt tuning with evals) |
| New env vars | `AGENT_MAX_TURNS` (40), `AGENT_MAX_FIX_ITERATIONS` (3), `AGENT_SCREEN_CONCURRENCY` (2). `MAX_SCREENS_DEFAULT` (3) and `AI_MODEL` already exist from Phase 06. |

## Goal
When the user clicks **Confirm & build**, the Builder agent turns the confirmed `AppPlan` into a working Expo app inside the template. That means 5 theme packs, data types and seed data, a shared store, shared components, and one functional screen per planned screen. Progress streams live onto the canvas: each frame goes from placeholder → building shimmer → real screen, with "Writing `today.tsx` · 142 lines" in chat. The build ends validated (typecheck-lite, import allowlist, routes, render smoke test) and snapshotted. A theme picker switches between the 5 packs instantly, with no LLM call.

## What we do differently from the video (and why)
- **The background job is already in place.** The video builds generation in a request handler, then moves it into CloudHike Jobs + an ngrok webhook (~1:20:46) after hitting serverless timeouts. Our `agent.build` handler runs in the worker from the start (Phase 05). No tunnel is needed in dev.
- **Orchestrated stages instead of one long free-form loop** (`05-ai-agent-design.md` §2.2). That gives per-screen status, per-screen retry, 2–3 screens in parallel, and a cached shared prefix so screens 2..N are cheap.
- **Generation runs inside the contract.** The agent can only write allowlisted paths and imports, and validation is a gate rather than a hope. The video hit SDK and Expo Router issues later in Ch.14.
- **Scrolling is in the contract from the start.** The video had to ask for scrollable screens after generation (~1:16:00), because its "no scroll" rule hid content.
- **Theme contrast is fixed deterministically** in code after the agent writes `themes.ts`, not by asking the model to "keep text readable".
- **Usage is metered on every API response** (`ai_usage`). Credits are not enforced until Phase 18, but the data exists from day one.

## Steps

### 1. AI client plumbing for long runs
- Extend `lib/ai/client.ts` (Phase 06) with `streamTurn()`: streaming `client.beta.messages.stream(...)` with the settings in `05` §1 (adaptive thinking, `display: "summarized"`, explicit `effort`, `fallbacks: "default"` with its beta header, `max_tokens: 64000`). It returns `finalMessage()` and records `ai_usage`.
- `lib/ai/prompts/builder/`: `role.md`, a generated contract block (from `templates/expo-app/CONTRACT.md`), `uiux-guide.md`, the component docs (from `COMPONENT_DOCS.md`), and `exemplars/` (two hand-written high-quality screens, taken from the Phase 07 fixture). The system blocks are assembled in a **fixed order** with `cache_control` on the last stable block (`05` §5). Snapshot-test the assembled prompt so accidental byte changes show up in review.

### 2. Tools (`lib/ai/tools/`)
- Implement `list_files`, `read_file`, `write_file`, `edit_file`, `delete_file`, `validate`, `set_screen_status` per `05` §3, each as a Zod schema plus `run(ctx, input)`.
- Use `strict: true` on every tool, and `eager_input_streaming: true` on `write_file` and `edit_file`.
- Every tool call is validated against its Zod schema before running (eager streaming disables server validation). Failures return a `tool_result` with `is_error: true`.
- `ctx = { projectId, runId, allowedPaths, readShas }`. `allowedPaths` changes per stage (S1 vs S2 vs S3). `readShas` drives the `edit_file` staleness check.
- All writes go through `lib/vfs.writeFile(..., author: "agent", runId)` and emit `file.written {path, revisionId}`.

### 3. The manual streaming loop (`worker/agent/loop.ts`)
- Follow the sketch in `05` §3: persist each assistant turn to `runs.transcript` before running tools; return all parallel `tool_result`s in one user message; handle `refusal`, `pause_turn` and `max_tokens`; check the cancel flag between turns; enforce the `AGENT_MAX_TURNS` cap.
- `pipeStreamToRunEvents(stream, runId)`: thinking-summary deltas become `thinking.summary` (throttled). Partial `write_file` input JSON becomes `file.writing {path, lines}` (throttled ≤ 4/s). Parse `path` as soon as it appears in the partial JSON, and count `\n` in the partial content.

### 4. Validation pipeline (`lib/validate/`)
- Path allowlist, Babel syntax parse, import allowlist (from `templates/expo-app/ALLOWED_IMPORTS.json`), typecheck-lite (TypeScript compiler API, in-memory host with template `.d.ts` stubs, `noEmit`, first 20 errors), route check (plan screens ↔ files; literal `router.push` / `<Link href>` targets), theme discipline (no color literals outside `theme/`), data discipline (data only via `lib/store`).
- Output: `{ errors: Issue[], warnings: Issue[] }`, rendered to compact text (≤ ~4 KB) for the agent. Emits `validation.result`.
- Unit tests with a broken-on-purpose copy of the fixture (one test per rule).

### 5. Builder orchestration (`worker/handlers/agent-build.ts`)
```
load run + confirmed plan (messages.kind = plan_card, confirmed)
truncate plan.screens to plan cap (MAX_SCREENS_DEFAULT for first build)
upsert screens rows (status = planned) → emit run.status(running)
S1 Foundation  (allowedPaths: theme/themes.ts, data/, lib/store/, components/shared/, app/_layout.tsx)
   → after themes.ts written: post-process contrast (step 6) → emit theme.ready
   → platform writes theme/active.ts + regenerates app/(tabs)/_layout.tsx from screens + plan.navigation
S2 Screens     p-limit(AGENT_SCREEN_CONCURRENCY) over screens:
   shared cached prefix + "Build screen <id>" → screen.started … file.written … screen.done
   (allowedPaths: that screen's route file + components/<screenId>/)
   skip screens already status=ready (resume after crash/retry)
S3 Integrate   validate(all) + render smoke test → if errors: fix loop ≤ AGENT_MAX_FIX_ITERATIONS
create project_snapshots row → messages(kind=build_summary) → run.done {summaryMessageId, snapshotId}
projects.status = ready (or error with a retryable run.error)
```
- Each stage is a fresh conversation built from the same cached prefix. Each S2 screen loop is its own transcript (stored under `runs.transcript.screens[screenId]`). Transcripts are append-only (`05` §9).
- Reference images: only screens whose `referenceAssetIds` include an asset receive that image block.
- Cancel button: `POST /api/runs/:id/cancel` sets a flag. The loop stops at the next turn boundary and keeps finished screens.

### 6. Theme packs
- The S1 prompt requires `theme/themes.ts` to export 5 `Theme` objects. The first follows `plan.themeSeed`; the others are named variations (light and dark mixed).
- `lib/themes/postprocess.ts`: parse `themes.ts` with Babel, apply `ensureContrast()` to `onPrimary`/`onSecondary`/`text`/`textMuted` against their backgrounds (WCAG AA 4.5:1; 3:1 for large text tokens), print the file back, and write a new revision with `author: "system"`. Cache the parsed swatches in `projects.theme_pack` and set `projects.active_theme_id`.
- `POST /api/projects/:id/theme {themeId}` writes `theme/active.ts` (`author: "theme_switch"`), updates `projects.active_theme_id`, and emits `file.written`. The preview applies it via `theme.set`. No credits, no LLM.
- Header UI: theme dots (swatches from `theme_pack`), with active state and tooltips showing the theme name.

### 7. Canvas & chat UX during a build
- Frames listen for `screen.started` (shimmer + "Building Today…"), `file.written` (hot swap) and `screen.done` (live render).
- Chat shows a collapsible build timeline: Foundation ✓ → Today (writing · 142 lines) → Progress ⏳ … plus validation results, the final `build_summary`, and the ms/credits used (credits displayed but not charged yet).
- After a page reload, the run timeline rebuilds from `run_events` (SSE `Last-Event-ID` from Phase 05).

### 8. Render smoke test (`worker/lib/smoke.ts`)
- Playwright + Chromium loads `/preview-runtime/harness.html`, pushes the same file set the canvas receives (project files plus the template's runtime files such as `theme/tokens.ts` and `lib/data/*`), visits each route, and collects `runtime.error` messages for 1.5 s per route.
- Errors are fed to S3 like validator errors. The harness is reused for thumbnails in Phase 17.

### 9. Snapshot & summary
- At the end of the run, `project_snapshots` stores `{path → revisionId}`. A "Restore" UI comes in Phase 09, but the data exists now.
- Eval hook: `evals/` gets 5 golden prompts (habit tracker, expense tracker, grocery store, workout log, notes). Record pass/fail, validation counts, duration, cost and `cache_read_input_tokens` per S2 call. See `10-testing-and-evals.md`.

## Files to create / change
```
lib/ai/client.ts                      (+ streamTurn, usage recording)
lib/ai/prompts/builder/{role.md,uiux-guide.md,exemplars/*.tsx,index.ts}
lib/ai/tools/{list-files,read-file,write-file,edit-file,delete-file,validate,set-screen-status,index}.ts
lib/ai/build.ts                       (stage prompts + allowedPaths per stage)
lib/validate/{paths,syntax,imports,typecheck,routes,theme,data,index}.ts
lib/themes/{postprocess.ts,contrast.ts}
worker/agent/loop.ts
worker/agent/stream-events.ts
worker/handlers/agent-build.ts
worker/lib/smoke.ts
public/preview-runtime/harness.html   (built by preview-runtime/build.mjs)
app/api/projects/[projectId]/build/route.ts
app/api/projects/[projectId]/theme/route.ts
app/api/runs/[runId]/cancel/route.ts
components/workspace/{ThemePicker.tsx,BuildTimeline.tsx}
components/canvas/PhoneFrame.tsx      (+ building/error states)
evals/golden/*.json, evals/run-build-eval.ts
```

## Database changes
- Uses `runs` (kind `build`), `run_events`, `screens.status`, `project_files`, `file_revisions` (`author`: `agent` | `system` | `theme_switch`), `ai_usage`.
- New table: `project_snapshots` (`03` §5).
- New column usage: `projects.theme_pack` (jsonb), `projects.active_theme_id`, `projects.status` transitions `awaiting_confirmation → building → ready | error`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/build` | editor | `{ planMessageId }` → `{ runId }`. Rejects if a build/edit run is already `queued`/`running` for the project (409). |
| POST | `/api/projects/:id/theme` | editor | `{ themeId }` → `{ revisionId }`. 400 if `themeId` isn't in `theme_pack`. |
| POST | `/api/runs/:runId/cancel` | editor of run's project | → `202` |
| GET | `/api/runs/:runId/events` | viewer | SSE (from Phase 05) — new event types per `05` §7 |

## Acceptance checklist
- [ ] Confirming a 3-screen habit-tracker plan produces a working app in a few minutes. All frames render, and navigation, add, toggle and delete work in the interactive preview.
- [ ] The canvas shows shimmer → live screen per screen as each finishes. Chat shows "Writing `<file>` · N lines" updating live.
- [ ] Reloading the page mid-build loses nothing; the timeline resumes.
- [ ] Killing the worker mid-S2 and restarting: the job retries and skips screens already `ready`.
- [ ] Validation errors injected by a test (e.g. a forbidden import) are fixed by S3, or the run ends in `error` with a readable message and a Retry button.
- [ ] 5 theme dots appear. Switching updates every frame in < 300 ms with zero `ai_usage` rows added.
- [ ] Every theme passes the contrast check (unit test over generated packs).
- [ ] Every screen with lists scrolls. No content is unreachable (smoke test asserts that the scroll container exists for long content).
- [ ] `ai_usage` has a row per API response. S2 calls after the first show `cache_read_input_tokens > 0`.
- [ ] **Security:** a crafted plan or user message instructing the agent to write `package.json`, `../../etc/x` or `lib/data/config.ts` results in tool errors and no file changes (test with a mocked model response).
- [ ] **AuthZ:** `build`, `theme` and `cancel` return 404 for non-members and 403 for viewers.
- [ ] 5/5 golden eval prompts complete with 0 validation errors (record the baseline numbers in `10-testing-and-evals.md`).

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 08 (Builder agent + theme packs) only.

Read first:
- docs/plan/05-ai-agent-design.md (entire file — §1 model config, §2.2 stages, §3 tools
  and loop, §4 validation, §5 caching, §7 events, §9 transcripts, §10 limits)
- docs/plan/06-generated-app-template.md (contract, ownership, themes)
- docs/plan/07-preview-runtime.md (§8 smoke test harness)
- docs/plan/03-data-model.md (§4 runs/events, §5 VFS, §8 ai_usage)
- docs/plan/phases/phase-08-app-generation-and-themes.md (this phase)
- Existing code: lib/ai (Phase 06), lib/jobs + lib/runs (Phase 05), lib/vfs + preview-runtime (Phase 07)

Before writing SDK code, check the installed @anthropic-ai/sdk version's docs for
streaming, eager_input_streaming, cache_control and the fallbacks beta. Don't guess signatures.
Model id comes from env.AI_MODEL — never hard-code it.

Build, one commit per phase-doc step: streamTurn + prompts (with a prompt snapshot test),
tools with Zod validation + path allowlist, manual streaming loop, validators with tests,
S1/S2/S3 orchestration in worker/handlers/agent-build.ts, theme post-processing +
POST /theme + ThemePicker, canvas/chat build UX, smoke test harness, snapshot + build_summary,
and a 5-prompt eval script.

Rules: agent writes ONLY via lib/vfs; no bash tool; tool_choice auto only; transcripts
append-only; every API route uses requireProjectAccess(); events never contain file contents.
Do NOT build: chat edits/add-screen (Phase 09), inline editing (Phase 10), Snack/QR (11),
credit enforcement (18).

Definition of done = Acceptance checklist in the phase doc. Use a mocked Anthropic client in
unit tests; run the real golden evals once and report pass rate, duration and cost.
Finish with lint, typecheck, tests, and a summary including any prompt changes you made.
```

## Pitfalls & notes
- **Cache silently missing:** a timestamp, a project name in `system`, or tools assembled in a different order breaks the shared prefix. Keep all project-specific content in `messages`.
- **Partial JSON parsing** for `file.writing`: use a tolerant parser on the accumulated `input_json` deltas. Only the *final*, schema-validated input is written.
- **Parallel screens editing shared files:** S2 loops may only write their own route file and `components/<screenId>/`. If a screen needs a shared component change, it reports it, and S3 makes the change. That avoids write races.
- **Tab layout drift:** the platform (not the agent) generates `app/(tabs)/_layout.tsx` deterministically from the `screens` list + `plan.navigation`, after S1 and on every add/remove screen (Phase 09). It's outside every agent allowlist (`06` §3).
- Keep the first-build cap (`MAX_SCREENS_DEFAULT=3`) during development for speed and cost. The video does the same, and users can ask for more screens via chat in Phase 09.
- If quality is poor, improve the **exemplars and component library** before adding more prompt rules. Long rule lists make output worse on current models.
- Expect to revisit `uiux-guide.md` repeatedly. Every change must be followed by an eval run.
