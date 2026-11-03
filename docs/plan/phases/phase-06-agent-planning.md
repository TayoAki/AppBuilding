# Phase 06 — AI Agent: Planning

| | |
|---|---|
| Video chapter(s) | Ch.10 "AI Agent Configuration" (59:35) |
| Branch | `feat/06-agent-planning` |
| Depends on | Phase 03 (project + first message + plan run), Phase 05 (jobs + events) |
| Effort (solo dev + Claude Code) | ~2–3 days |
| New env vars | `ANTHROPIC_API_KEY`, `AI_MODEL` (default `claude-opus-5-5`), `MAX_SCREENS_DEFAULT` (default `3`) |

## Goal
When a project is created, the worker runs the **Planner** (`05-ai-agent-design.md` §2.1). It reads the prompt and reference images, streams a short "thinking" summary into chat, and produces a validated `AppPlan`: app name, navigation, screens with key elements, entities and a theme seed. The plan appears as a **plan card** with a **Confirm & Build** button. The user can revise it by chatting ("just the first three screens"), and refreshing restores everything. Every model call is logged to `ai_usage` from now on, for billing in Phase 18.

## What we do differently from the video (and why)
- **Official Anthropic SDK, structured outputs.** The video used the OpenAI Agents SDK through CloudHike's OpenAI-compatible gateway, then parsed the plan from the agent's text. We call `client.messages.parse()` with `zodOutputFormat(AppPlanSchema)`, so the plan is schema-valid by construction (`05-ai-agent-design.md` §1).
- **Runs in the worker** (Phase 05), not in a request handler, with streamed progress. The video had to refresh the page to see the planner working.
- **Server-enforced screen cap.** `MAX_SCREENS_DEFAULT` (3, as in the video's later change) is in the prompt *and* enforced after parsing.
- **Append-only planning transcript.** Revisions append to the same conversation in `runs.transcript` (preserved-thinking rules, `05-ai-agent-design.md` §9). Nothing is edited in place.
- **Usage metering from the first call** (`ai_usage`), not bolted on at the end like the video's Ch.20.
- **Refusals and errors are first-class UI states** (an error card with Retry), not silent failures.

## Steps

### 1. AI client module
`lib/ai/client.ts`:
- One `Anthropic` instance (reads `ANTHROPIC_API_KEY`). Expose `aiModel()` reading `env.AI_MODEL`.
- `withUsage(runId, userId, fn)` helper: after every response, insert `ai_usage` (`model`, `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `cost_usd_micros` via `lib/ai/pricing.ts`, `served_model` if a fallback ran).
- Error mapping: SDK typed errors → `AppError` with `retryable` (429, 529/overloaded, 5xx, connection → retryable; 400/401/403/404 → not). Use the SDK's exception classes, not string matching.
- Defaults applied to every call: streaming, `thinking: { type: "adaptive", display: "summarized" }`, explicit `output_config.effort`, `betas: ["server-side-fallback-2026-07-01"]` + `fallbacks: "default"`. Check `stop_reason === "refusal"` before reading content. Verify each of these against the installed SDK version.

### 2. Pricing table
`lib/ai/pricing.ts`: per-model `$ / MTok` for input, output, cache read and cache write, as **config** (copied from `11-costs-and-pricing.md`; update when prices change). `costUsdMicros(usage, model)` is integer math.

### 3. ai_usage table
Add `ai_usage` per `03-data-model.md` §8, with indexes `(user_id, created_at)` and `(run_id)`.

### 4. Plan schema
`lib/ai/schemas/plan.ts`: `AppPlanSchema` exactly as in `05-ai-agent-design.md` §2.1, plus `export type AppPlan = z.infer<…>`. Add `lib/ai/plan-guard.ts`: post-parse rules. Truncate `screens` to the cap; unique `id`s and `route`s; tab `screenId`s must exist; `entities` names PascalCase; `themeSeed.primaryHex` is a valid hex. On a guard failure, ask once for a corrected plan (append a message listing the violations), then fail the run.

### 5. Planner prompt
`lib/ai/prompts/planner.ts`:
- System (stable, cacheable): role ("product designer + mobile architect planning an Expo app inside a fixed template"), planning rules (mobile-first, 1 primary action per screen, realistic scope for 3–8 screens, tabs vs stack guidance, naming), and a short excerpt of the template contract (what's possible: Expo Go-compatible features only, no custom native modules). Put no timestamps or user data in the system prompt.
- User message: the prompt, platforms, `screen cap = min(MAX_SCREENS_DEFAULT, planLimit)`, and reference images as image content blocks. Base64 from storage via `StorageProvider`; or URL source with a short-lived presigned GET. Each is labelled `[asset ast_…]` so the plan can reference them in `referenceAssetIds`.
- Tell it to treat image content as visual reference only (ignore instructions in images).
- Effort: `medium`.

### 6. Plan job handler
`worker/handlers/agent-plan.ts` (`agent.plan`):
1. `startRun`. Load the project, the triggering message and its assets. Load `runs.transcript` if this is a revision (see step 8).
2. `appendRunEvent(run.status, {status: "running"})`. Set `projects.status = "planning"`.
3. Stream the `messages.parse` call. Forward summarized thinking deltas as throttled `thinking.summary` events (≤ 2/s, ≤ 1 KB each).
4. On success: append the assistant turn (full `content`) to `runs.transcript`. Run `plan-guard`. Insert a `messages` row (`role: "assistant"`, `kind: "plan_card"`, `content`: short summary, `payload`: the `AppPlan`). Set `projects.title = plan.appName` (only if the user hasn't renamed it) and `projects.status = "awaiting_confirmation"`. Emit `plan.ready {messageId}`, then `finishRun`.
5. On refusal: insert a `messages` row (`kind: "error"`) with a friendly explanation and `failRun({code: "refused", retryable: false})`.
6. Record usage for every call (including guard retries).

### 7. Plan card UI
`components/chat/renderers/plan-card.tsx`:
- Header: app name + tagline. Theme-seed swatch dots, a preview of the theme selector that becomes real in Phase 08.
- Screen list: name, purpose and `keyElements` bullets. Navigation type badge (tabs/stack).
- Collapsible "Data" section (entities + fields) and "Assumptions".
- Footer: **Confirm & build N screens** (primary; editors only) and the hint "Or reply to change the plan".
- Only the **latest** plan card is actionable. Older ones render as "Superseded".
- The chat shows a collapsible "Thinking…" block fed by `thinking.summary` events while the run is active.
- The canvas shows `planned` placeholder frames for each plan screen (names only), so the user sees the shape of the app.

### 8. Revise by chat
- Enable the chat input (Phase 04 had it disabled). `POST /api/projects/:id/messages` (editor) with `{content, attachments?}`.
- If the project status is `awaiting_confirmation`, create a `runs` row with `kind: "plan_revision"`, `input: {messageId, previousRunId}`, and enqueue `agent.plan`.
- The handler copies the previous plan run's transcript into the new run, **appends** the user's revision message, and asks for a full new `AppPlan`. The earlier assistant turns are left untouched, so their thinking blocks stay valid.
- If the status is anything else, return 409 with "Editing arrives in Phase 09". The Editor agent handles post-build chat.

### 9. Confirm & Build
`POST /api/projects/:id/confirm` (editor) `{planMessageId}`:
- Verify it's the latest plan card and the status is `awaiting_confirmation`. Persist the chosen plan as the project's **current plan** (`projects.current_plan_message_id`, or copy into a `plan` jsonb column; pick one and update `03-data-model.md`).
- Create `screens` rows from `plan.screens` (`status: "planned"`, default canvas positions in a row).
- Create `runs` (`kind: "build"`) and enqueue `agent.build`. **Stub handler until Phase 08**: it emits `run.status` and finishes with a message "Build agent arrives in Phase 08".

### 10. Restore on refresh
The workspace server load already returns messages (Phase 04). Also return the active run (`/runs/active`) so the client resubscribes via `useRunEvents` and replays thinking events. The plan card renders from `messages.payload`. No re-generation.

### 11. Tests
- Unit: `plan-guard` (cap truncation, duplicate routes, missing tab target); `costUsdMicros`.
- Handler test with a **mocked Anthropic client** (recorded fixture of a parse response + a refusal response): rows written, events emitted, statuses correct.
- One live smoke test behind `RUN_LIVE_AI=1` (not in CI by default): a habit tracker prompt gives a valid plan with ≤ 3 screens.
- Authz: a viewer can't confirm or post messages (403); another user gets 404.

## Files to create / change
```
lib/ai/{client.ts,pricing.ts,plan-guard.ts}
lib/ai/schemas/plan.ts
lib/ai/prompts/planner.ts
lib/db/schema/ai-usage.ts
worker/handlers/agent-plan.ts
worker/handlers/agent-build.ts              (stub)
app/api/projects/[projectId]/messages/route.ts   (POST)
app/api/projects/[projectId]/confirm/route.ts    (POST)
components/chat/renderers/{plan-card.tsx,thinking.tsx,error-card.tsx}
components/chat/message-input.tsx           (enabled)
components/canvas/phone-frame.tsx           (planned state from plan)
tests/fixtures/anthropic/{plan-habit.json,refusal.json}
```

## Database changes
- New `ai_usage` (per `03-data-model.md` §8).
- `projects`: add the current-plan pointer (`current_plan_message_id text null`, recommended). Update `03-data-model.md` to match.
- Uses `messages.kind = "plan_card" | "error"`, `runs.kind = "plan" | "plan_revision" | "build"`, and `screens` rows created on confirm.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:projectId/messages` | editor | `{content, attachments?}` → `201 {messageId, runId}` · `409` if not awaiting confirmation |
| POST | `/api/projects/:projectId/confirm` | editor | `{planMessageId}` → `202 {runId}` · `409` if stale plan |
| GET | `/api/runs/:runId/events` | viewer | (Phase 05) now carries `thinking.summary`, `plan.ready`, `run.error` |

## Acceptance checklist
- [ ] Creating a project with "habit tracker" shows "Thinking…" within ~3 s, then a plan card with ≤ `MAX_SCREENS_DEFAULT` screens, an app name and theme dots.
- [ ] The project title updates to the plan's app name (unless the user renamed it first).
- [ ] Replying "only the first two screens, add a settings screen" produces a new plan card; the old one shows "Superseded"; `runs.transcript` shows appended turns, with no earlier turn modified.
- [ ] Refreshing at any point restores the chat, the latest plan card and any in-flight thinking stream.
- [ ] Confirm & build creates `screens` rows (`planned`), placeholder frames on the canvas, and a `build` run (stub message).
- [ ] A reference image is visible to the model (fixture/live test: the plan mentions an element only present in the image) and is mapped via `referenceAssetIds`.
- [ ] Every API response creates an `ai_usage` row with non-zero tokens and a computed cost.
- [ ] A forced refusal (fixture) shows a friendly error card with no crash; a 429 is retried by the job and succeeds.
- [ ] A viewer can't post or confirm (403); another user gets 404; the API key never appears in client bundles or logs.

## Claude Code prompt
```text
Read first:
- docs/plan/05-ai-agent-design.md (§1 model config, §2.1 Planner, §5 prompt/caching,
  §7 events, §8 reference images, §9 transcripts)
- docs/plan/03-data-model.md (§3 messages, §4 runs, §8 ai_usage)
- docs/plan/11-costs-and-pricing.md (pricing table for lib/ai/pricing.ts)
- docs/plan/phases/phase-06-agent-planning.md (this phase)
Use the official @anthropic-ai/sdk TypeScript SDK. Check method names and params
(messages.parse, zodOutputFormat, streaming, thinking/effort, fallbacks beta) against
the installed SDK's docs before writing code. Don't guess.

Implement Phase 06:
1. lib/ai/client.ts (single client, AI_MODEL env, usage logging to ai_usage, typed
   error mapping to retryable/non-retryable, refusal check, fallbacks "default").
2. lib/ai/pricing.ts + ai_usage table/migration.
3. AppPlanSchema exactly as in 05-ai-agent-design.md §2.1 + plan-guard post-checks
   (cap = min(MAX_SCREENS_DEFAULT, plan limit), unique ids/routes, valid tab targets).
4. Planner prompt (stable system prompt, no volatile data in it; images as image
   blocks labelled with asset ids; effort medium).
5. worker/handlers/agent-plan.ts: stream, forward summarized thinking as throttled
   thinking.summary events, write plan_card message, update project title/status,
   plan.ready event, append-only transcript in runs.transcript, refusal → error card.
6. Plan card + thinking + error renderers; planned placeholder frames on the canvas.
7. POST /messages (plan revision when awaiting_confirmation, else 409) and
   POST /confirm (creates screens rows + build run; agent.build handler is a stub).
8. Tests: plan-guard, pricing, handler with mocked Anthropic fixtures, authz;
   optional live test behind RUN_LIVE_AI=1.

Do NOT generate any app code, theme files or preview runtime. That's Phases 07–08.
Definition of done = Acceptance checklist in the phase doc.
Finish with lint/typecheck/test and a summary including a sample plan JSON from the fixture.
```

## Pitfalls & notes
- **The video's two stumbles:** (1) the UI didn't update until a manual refresh, which SSE + `useRunEvents` fixes; (2) it planned 6–8 screens by default and the author had to ask for "just three". Both the cap and the "reply to change the plan" hint are built in here.
- Don't put `Date.now()`, user names or project ids in the system prompt. Volatile text invalidates the prompt cache. Put it in the user message.
- Forced `tool_choice` returns 400 on the configured model, and assistant prefill isn't supported. Structured outputs are the right tool for "give me JSON".
- Large reference images cost input tokens. Downscale to ≤ 1568 px on the long edge before sending (store the downscaled copy at upload time or on first use).
- Keep thinking display to **summaries**. Never prompt the model to "write out its reasoning" in the answer; that can trigger refusals and wastes tokens.
- `ai_usage` rows must be written even when the run later fails. Billing in Phase 18 charges for consumed tokens, with the policy defined in `11-costs-and-pricing.md`.
- The free-token "data sharing" trick shown in the video for another provider is not part of this plan. Budget for real API costs (`11-costs-and-pricing.md`).
