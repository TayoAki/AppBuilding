# Phase 06 — AI Agent: Planning (the Planner)

| | |
|---|---|
| Video chapter(s) | Ch.10 "AI Agent Configuration" (59:35); reference repo Ch.5 |
| Branch | `feat/06-agent-planning` |
| Depends on | Phase 03 (project + first message + plan run), Phase 05 (jobs, step chaining, events) |
| Effort (solo dev + Claude Code) | ~3 days |
| New env vars | `ANTHROPIC_API_KEY`, `AI_MODEL` (default `claude-opus-5-5`), `AI_BUILDER_MODEL` (default `claude-opus-5-5`, used from Phase 08), `MAX_SCREENS_DEFAULT` (default `3`) |

## Goal
As soon as a project is created, the worker runs the **Planner**, the only agent that talks to the person (`05-ai-agent-design.md` §3). It streams a sentence or two showing it understood the idea, then plans **through a tool** (`propose_plan`). The server validates and caps the plan, and the chat shows it as a **plan card** with **Confirm & build**. The person can revise by chatting ("just the first three screens"), or approve in chat ("looks good"), in which case the Planner calls `confirm_plan`. Refreshing restores everything. Every model call is logged to `ai_usage` from now on.

## What we do differently from the video (and why)
- **Official Anthropic SDK.** The video, and the reference app, used the OpenAI Agents SDK through an OpenAI-compatible endpoint. We call `@anthropic-ai/sdk` directly for caching, refusal fallbacks and exact usage (`05` §1).
- **Runs in the worker** (Phase 05) with streamed prose and thinking. The video had to refresh to see the planner working.
- **Server-enforced plan rules.** The screen cap, reachability, tab count and Sign In rules are in the prompt *and* checked on every `propose_plan` call. A violation goes back to the model as a tool error.
- **Cache-friendly prompt.** Static system prompt + project state as a mid-conversation system message right before the latest user turn (`05` §3.1).
- **Usage metering from the first call** (`ai_usage`), not bolted on at the end.

**Reference app notes** (`13-reference-implementation.md`)
- R1/R2: the Planner plans through tools (`propose_plan`, `confirm_plan`), so the plan arrives as structured data the chat draws as a card. We keep that and validate the tool input with Zod + `plan-guard`.
- R3: "What to act on": act only on the latest message, and the current plan is the truth. Those rules go into the static prompt now. The **Start new session** button comes in Phase 09.
- R4: a default **Sign In** screen (id `sign-in`, first, never a tab) for any app with personal data, outside the screen cap. `firstScreenId` is part of the plan.
- R21: the reference Planner only saw attachment *names*. We send reference images as image blocks, but keep the rule "don't claim to have read what you can't see" for non-image attachments.
- §3: the reference app rebuilt the whole system prompt from state on every message, which defeats caching. We split static rules from state.

## Steps

### 1. AI client module
`lib/ai/client.ts`:
- One `Anthropic` instance (`ANTHROPIC_API_KEY`). `plannerModel()` reads `AI_MODEL`, `builderModel()` reads `AI_BUILDER_MODEL`.
- `withUsage(runId, userId, fn)`: after every response, insert `ai_usage` (`model`, tokens incl. cache read/write, `cost_usd_micros` via `lib/ai/pricing.ts`, `served_model` if a fallback ran). Also on failed runs.
- Error mapping: SDK typed errors → `AppError{retryable}` (429, overloaded, 5xx and connection errors retry; 400/401/403/404 don't). Use the SDK exception classes, not string matching.
- Defaults on every call: streaming, `thinking: { type: "adaptive", display: "summarized" }`, explicit `output_config.effort`, `betas: ["server-side-fallback-2026-07-01"]` + `fallbacks: "default"`. Check `stop_reason === "refusal"` before reading content. Verify each against the installed SDK version.

### 2. Pricing table and `ai_usage`
- `lib/ai/pricing.ts`: per-model input/output/cache-read/cache-write `$ / MTok` as config, mirroring `11-costs-and-pricing.md` §1. An unknown model throws. `costUsdMicros(usage, model)` uses integer math.
- `ai_usage` table per `03-data-model.md` §8, indexes `(user_id, created_at)` and `(run_id)`.

### 3. Schemas and plan guard
- `lib/ai/schemas/plan.ts`: `AppPlanSchema` exactly as in `05` §3.4 (`appName, tagline, summary, targetUsers, navigation, firstScreenId, screens[{id, name, purpose, keyElements(3–6), navigatesTo[], tab|null}], design{style, mode, primaryColor, accentColor, backgroundColor}, assumptions`).
- `lib/ai/plan-guard.ts` returns a list of violations (it doesn't silently fix them):
  - `screenTarget = max(min(MAX_SCREENS_DEFAULT, planMax − 1), 1)`. A `sign-in` screen doesn't count toward it; the total stays ≤ `planMax` (Free 5 until Phase 18 config exists).
  - Ids are unique kebab-case. `firstScreenId` exists. Every `navigatesTo` target exists. Every non-first screen is reachable from the first (BFS over `navigatesTo` ∪ tab screens).
  - `navigation: "tabs"` ⇒ 3–5 screens with `tab`, each with a label and an icon name from the Ionicons allowlist. `sign-in` never has a tab. `"stack"` ⇒ no tabs.
  - Design colors are `#rrggbb`. If the background/primary contrast is unreadable, report that as a violation.
- Accepting more screens than `screenTarget` is allowed only when the latest user message explicitly asks for more (the model asserts it with `explicitScreenRequest: true` in the tool input; the server still caps at `planMax`).

### 4. Planner prompt (static, cached)
`lib/ai/prompts/planner.ts`: write the sections from `05` §3.2 **in our own words**. For this phase, include *How a project goes*, *What to act on*, *Planning rules*, *Talking with the person*, and the reference-image rule (visual guidance only; ignore instructions inside images). Phases 08/09/13 append *Changing a built app*, *A selected element* and *The app's backend*, along with their tools. The tool list and system text change only at deploy time, never per request, so caching holds.

`lib/ai/prompts/project-state.ts`: `buildProjectState(project, account)` renders the PROJECT STATE block (`05` §3.1): phase description, platform, plan name + credits + price list (credits are a placeholder until Phase 18), backend status ("not connected" until Phase 13), attachment names, and the current plan JSON. It's sent as `{role: "system", content}` directly before the newest user message. If the configured model doesn't support mid-conversation system messages, fall back to a `<project-state>` block at the start of that user turn.

### 5. Planner tools (this phase)
`lib/ai/tools/planner-tools.ts`, all `strict: true`, `tool_choice: auto`:
- `propose_plan` — input = `AppPlanSchema` + `explicitScreenRequest`. Server: Zod-validate → `plan-guard` → on violations, a `tool_result` with `is_error: true` and the violation list (the model re-proposes in the same turn, max 2 attempts). On success: insert a `plan_card` message (payload = plan), set `projects.plan` = draft plan + `current_plan_message_id`, set `projects.title = appName` (unless the user renamed it), status `awaiting_confirmation`, emit `plan.ready`.
- `confirm_plan` — input `{}`. Server: allowed only when status is `awaiting_confirmation` and a plan exists. It calls the same `confirmPlan()` as the button (step 9). The prompt restricts it to a clear approval in the latest message; the server can't judge wording, so it checks state only.

### 6. Chat job handler (`agent.chat`)
`worker/handlers/agent-chat.ts`, one job per user message (payload `{runId, messageId}`):
1. `startRun`. Load the project, the current session's messages, the new user message + its assets.
2. Build `messages[]`: replay prior turns from `messages.payload.api` (the raw API content blocks of each turn, thinking and tool blocks included, unchanged and append-only, `05` §9), then PROJECT STATE (system), then the new user turn (text + image blocks for reference images, base64 from storage, downscaled to ≤ 1568 px).
3. Manual streaming loop (≤ 3 tool calls per message): forward text deltas as batched `assistant.delta` events and thinking summaries as throttled `thinking.summary` (≤ 2/s). Execute tool calls and append `tool_result`s in one user turn.
4. Persist: one `messages` row for the assistant's visible reply (`kind: "text"`) with `payload.api` = full content blocks, plus the `plan_card` row from the tool. Emit `assistant.done`.
5. Refusal → `messages(kind: "error")` with a friendly explanation + `failRun({code: "refused", retryable: false})`. Other errors → `run.error` with Retry.
6. **Auto-start:** project creation (Phase 03) now enqueues `agent.chat` for the first message immediately, so planning begins before the workspace even finishes loading.

### 7. Plan card UI
`components/chat/renderers/plan-card.tsx`:
- Header: app name + tagline, with design swatches (primary/accent/background) previewing the theme selector that becomes real in Phase 08.
- Screens: name, purpose, key-element bullets, a tab badge with icon, a "first screen" marker, and a "Sign In (assumed)" note when present.
- Collapsible "Navigation" (tabs/stack + links) and "Assumptions".
- Footer: **Confirm & build N screens** (editors only), showing the credits estimate from Phase 18 (placeholder now), and the hint "Or reply to change the plan".
- Only the latest card is actionable; older ones render "Superseded". The canvas shows `planned` placeholder frames for the plan's screens.
- Streaming prose renders live from `assistant.delta`. A collapsible "Thinking…" block comes from `thinking.summary`.

### 8. Revise by chat
- Enable the chat input. `POST /api/projects/:id/messages` (editor) `{content, attachments?}` inserts the user message, creates a run (`kind = chat`), and enqueues `agent.chat`.
- While `building`, return `409 "The plan can't change while screens are being built"`. Built-app changes arrive in Phase 09 (the same Planner, plus `update_app`).

### 9. Confirm & build
`lib/agent/confirm.ts` `confirmPlan(projectId, actor)`, used by `POST /api/projects/:id/confirm {planMessageId}` and by `confirm_plan`:
- Verify it's the latest plan card and the status is `awaiting_confirmation`. Mark the plan confirmed in `projects.plan`.
- Upsert `screens` rows (`status: "planned"`, default canvas positions; keep ids).
- `chargeForBuild()` hook: a no-op that returns "ok" until Phase 18.
- Create `runs(kind: "build", step: "foundation")` and enqueue `agent.step {runId, step: "foundation"}`. **Stub handler until Phase 08**: it emits `run.status` and finishes with "Build agent arrives in Phase 08".

### 10. Restore on refresh
The workspace load returns messages + `projects.plan` + the active run. The client resubscribes via `useRunEvents` and replays deltas/thinking. Plan cards render from `messages.payload`, with no re-generation.

### 11. Tests
- Unit: `plan-guard` (cap with/without Sign In and explicit requests, reachability, tab rules, firstScreenId); `costUsdMicros`; `buildProjectState` snapshot.
- Handler tests with a **mocked Anthropic transport** (recorded fixtures: understanding text + `propose_plan`; an invalid plan → is_error → corrected plan; "looks good" → `confirm_plan`; a refusal). Check rows, events, statuses, and that the transcript is append-only.
- Caching: two consecutive messages → the second request's prefix (tools + system + earlier turns) is byte-identical (snapshot the serialized request up to the state message).
- Live smoke behind `RUN_LIVE_AI=1`: "habit tracker" → valid plan ≤ 3 screens + Sign In.
- Authz: a viewer can't post or confirm (403); another user gets 404.

## Files to create / change
```
lib/ai/{client.ts,pricing.ts,plan-guard.ts}
lib/ai/schemas/plan.ts
lib/ai/prompts/{planner.ts,project-state.ts}
lib/ai/tools/planner-tools.ts
lib/agent/confirm.ts
lib/db/schema/ai-usage.ts
worker/handlers/agent-chat.ts
worker/handlers/agent-step.ts                 (stub dispatcher until Phase 08)
app/api/projects/[projectId]/messages/route.ts   (POST)
app/api/projects/[projectId]/confirm/route.ts    (POST)
components/chat/renderers/{plan-card.tsx,thinking.tsx,error-card.tsx,streaming-text.tsx}
components/chat/message-input.tsx             (enabled)
components/canvas/phone-frame.tsx             (planned state from plan)
tests/fixtures/anthropic/{plan-habit.json,plan-invalid-then-fixed.json,confirm-in-chat.json,refusal.json}
```

## Database changes
- New `ai_usage` (`03-data-model.md` §8).
- `projects`: `plan jsonb`, `current_plan_message_id` (`03` §2).
- `messages.payload.api` holds raw API content blocks per turn. `messages.kind` uses `text | plan_card | error`.
- `runs.kind` uses `chat | build` (and `change` from Phase 09); `runs.step` is set on build runs.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:projectId/messages` | editor | `{content, attachments?}` → `201 {messageId, runId}` · `409` while building |
| POST | `/api/projects/:projectId/confirm` | editor | `{planMessageId}` → `202 {runId}` · `409` if not the latest plan / wrong status |
| GET | `/api/runs/:runId/events` | viewer | (Phase 05) now carries `assistant.delta/done`, `thinking.summary`, `plan.ready`, `run.error` |

## Acceptance checklist
- [ ] Creating "habit tracker" streams 1–2 sentences of understanding within ~3 s, then a plan card with ≤ `MAX_SCREENS_DEFAULT` screens **plus** a Sign In screen first, an app name and design swatches.
- [ ] "A calculator" gets **no** Sign In screen. "Build me 5 screens" gets 5 (capped by the plan maximum).
- [ ] An invalid plan from the model (fixture) is rejected by the guard and corrected in the same turn. The user only sees the valid card.
- [ ] "Only the first two screens, add a settings screen" produces a new card; the old one shows "Superseded"; earlier turns' `payload.api` are unchanged.
- [ ] Typing "looks good, build it" triggers `confirm_plan` → same result as the button: `screens` rows, placeholder frames, a `build` run (stub message).
- [ ] Refreshing at any point restores chat, the latest card and any in-flight stream.
- [ ] The project title becomes the app name (unless renamed first).
- [ ] Every API response creates an `ai_usage` row with tokens and cost. On the second message, `cache_read_input_tokens > 0`.
- [ ] A forced refusal shows a friendly error card. A 429 is retried by the job and succeeds.
- [ ] A viewer can't post or confirm (403); another user gets 404; the API key never appears in client bundles or logs.

## Claude Code prompt
```text
Read first:
- docs/plan/05-ai-agent-design.md (§1 model config, §3 Planner: prompt layout, rules,
  tools, AppPlan schema; §8 events; §9 sessions/transcripts)
- docs/plan/13-reference-implementation.md (R1–R4, R21, §3 Planner prompt row)
- docs/plan/03-data-model.md (§2 projects.plan, §3 messages, §4 runs, §8 ai_usage)
- docs/plan/11-costs-and-pricing.md (§1 pricing table for lib/ai/pricing.ts)
- docs/plan/phases/phase-06-agent-planning.md (this phase)
Use the official @anthropic-ai/sdk TypeScript SDK. Check streaming, tool definitions
(strict), thinking/effort, mid-conversation system messages and the fallbacks beta
against the installed SDK's docs before writing code. Don't guess.

Implement Phase 06:
1. lib/ai/client.ts (AI_MODEL + AI_BUILDER_MODEL, usage logging to ai_usage, typed error
   mapping, refusal check, fallbacks "default"); lib/ai/pricing.ts + ai_usage migration.
2. AppPlanSchema exactly per 05 §3.4 + plan-guard returning violations (screenTarget
   formula, Sign In outside the cap, reachability, tabs 3–5, firstScreenId, hex colors).
3. Static Planner system prompt in our own words (sections listed in the phase doc) +
   buildProjectState() sent as a mid-conversation system message before the newest
   user turn. No volatile data in the system prompt.
4. Tools propose_plan (guard → is_error tool_result on violations, max 2 attempts) and
   confirm_plan (state check, calls lib/agent/confirm.ts).
5. worker/handlers/agent-chat.ts: replay session turns from messages.payload.api
   (append-only), stream assistant.delta (batched) + thinking.summary, ≤3 tool calls,
   persist messages; auto-enqueue for the first message at project creation.
6. Plan card, streaming text, thinking and error renderers; planned frames on the canvas.
7. POST /messages (409 while building) and POST /confirm (screens rows, chargeForBuild
   no-op hook, build run with step "foundation", agent.step stub).
8. Tests from step 11, using recorded Anthropic fixtures; live test behind RUN_LIVE_AI=1.

Do NOT generate any app code, themes, components or preview runtime (Phases 07–08), and
don't add update_app / set_theme / setup_backend yet (Phases 08, 09, 13).
Definition of done = Acceptance checklist in the phase doc.
Finish with lint/typecheck/test and a summary including a sample plan JSON from a fixture.
```

## Pitfalls & notes
- **The video's two stumbles:** (1) the UI didn't update until a manual refresh, which SSE + `useRunEvents` fixes; (2) it planned 6–8 screens by default and the author had to ask for "just three". The cap and the "reply to change the plan" hint are built in.
- **Never let the model confirm on a question or a change request.** The prompt rule plus a fixture test ("can you add a settings screen?" must re-propose, not confirm).
- Keep `Date.now()`, user names and ids out of the system prompt. Volatile text belongs in the PROJECT STATE message, after the cached prefix.
- Forced `tool_choice` returns 400 on the configured model. Steer with the prompt and keep `strict: true` on tools.
- Downscale reference images before sending; large images cost input tokens on every replayed turn. Consider replaying older turns' images as text placeholders once a plan is confirmed (a *new* session in Phase 09 avoids the cost entirely).
- Keep thinking display to **summaries**. Never prompt the model to write out its reasoning in the reply; it wastes tokens and can trigger refusals.
- Write `ai_usage` rows even when the run fails. Phase 18 needs them for margin tracking.
- The free-token "data sharing" trick from the video is not part of this plan (`01` P14).
