# Phase 05 — Background Jobs and Realtime Events

| | |
|---|---|
| Video chapter(s) | New (not in video as a phase). It pulls forward the background-job fix from Ch.11 "Generate Mobile App with AI" (~1:20:46) |
| Branch | `feat/05-jobs-realtime` |
| Depends on | Phase 03 (runs table), Phase 04 (workspace store) |
| Effort (solo dev + Claude Code) | ~2.5 days (was ~2; adds the transactional outbox, the project-scoped event stream and two handler wrappers, per ADR-003) |
| New env vars | `DATABASE_URL_DIRECT` (optional; direct/session-mode connection for pg-boss and `LISTEN`). Optional: `WORKER_CONCURRENCY`, `JOBS_SCHEMA` (default `pgboss`) |

## Goal
Build the plumbing every long task uses:
- a Postgres-backed queue (pg-boss), fed through a **transactional outbox**;
- a separate long-running **worker** process;
- the `runs`, `run_events` and `job_outbox` tables;
- an `appendEvent()` that wakes listeners via `NOTIFY`;
- a **project-scoped SSE stream** with `Last-Event-ID` resume, open whenever the workspace is open.

Prove it end to end with demo jobs before any AI exists. After this phase, the agent (Phase 06+) never runs inside a request handler.

## What we do differently from the video (and why)
- **The worker exists before the agent.** The video built generation inside an API route and found in Ch.11 that Vercel kills functions after 15 s–5 min, then retrofitted CloudHike Jobs. We design for minutes-long work from day one (`02-architecture.md` A1).
- **No inbound webhook to our app, so no ngrok in dev.** The video's job service called back to a public URL, which forced a tunnel and broke sign-in (redirect/origin mismatch). Our worker connects **outbound** to Postgres, both locally (`npm run worker`) and in production.
- **Events are persisted**, so a refresh or reconnect replays everything missed. The video had to refresh to see the agent working.
- **Vendor-neutral `lib/jobs` interface.** pg-boss today; Inngest, Trigger.dev or CloudHike Jobs can replace it behind the same `enqueue()` / `registerHandler()` API.
- **Step chaining with leases from the start.** Agent pipelines run one step per job, with each step enqueueing the next (`05-ai-agent-design.md` §4). This phase provides `runs.step` / `runs.lease_id` / `runs.heartbeat_at`.
- **Transactional outbox (ADR-003 #7, #21).** The web app never calls pg-boss. Producers insert a `job_outbox` row in the **same Drizzle transaction** as the run (and the credit charge, Phase 18). A dispatcher in the worker forwards it to pg-boss. This avoids needing pg-boss and Drizzle to share one connection or transaction (`DATABASE_URL` vs `DATABASE_URL_DIRECT`).

**Reference app notes** (`13-reference-implementation.md`)
- R1/§3: the reference app runs each build step as its own background job that queues the next one. A redelivered earlier step is ignored (`builds.step`), and a retry takes over a lease (`builds.lease`), which stops the older execution at its next write. We adopt the same pattern on `runs.step` / `runs.lease_id`.
- §3: their job service only returns a run's final answer, so the worker saved batched events to a table and the browser replayed them. That's the same idea as our `run_events` + SSE. But their jobs called back to a public URL (ngrok locally), and a build nobody watched showed "building" until the workspace was reopened. Our worker pulls from Postgres, so neither problem exists.
- §4: preview frames polled constantly, and a render loop flooded the dev server. Here the workspace holds **one** project event stream and never polls the files API in a loop.

## Steps

### 1. Tables
Complete `runs` per `03-data-model.md` §4 and ADR-003 #3/#6:
- `created_by` (acting user), `billed_user_id` (project owner),
- `step`, `lease_id`, `heartbeat_at`, `attempt int default 0`,
- `steps jsonb default '{}'`, `credits_charged bigint default 0`, `credits_refunded bigint default 0`,
- `started_at`, `finished_at`, `job_id`.

There's **no** `transcript` and no `credits_reserved`: Planner turns live in `messages.payload.api`, and generators are stateless (`05` §9).

`run_events`: `id bigserial`, **`project_id` NOT NULL**, **`run_id` nullable** (non-run writes like inline edits, code saves, theme switches and restores emit events too), `type`, `data jsonb`, `created_at`. Indexes `(project_id, id)` and `(run_id, id)`.

`job_outbox`: `id bigserial`, `name text`, `data jsonb`, `options jsonb`, `created_at`, `dispatched_at null`, `job_id null`. Partial index on `dispatched_at IS NULL`.

Make sure `drizzle.config.ts` keeps `schemaFilter: ["public"]`.

### 2. Queue
- **Pin pg-boss to one major version (≥ 10)** in `package.json` and check its README for that version. v10+ requires `createQueue(name, options)` before `send`/`work`. Queue `policy` (`standard`/`short`/`singleton`/`stately`) and option names (`expireInSeconds`, `retryLimit`, `retryDelay`, `retryBackoff`, `batchSize`) changed across majors.
- `lib/jobs/queue.ts`: a singleton `PgBoss` (`connectionString: env.DATABASE_URL_DIRECT ?? env.DATABASE_URL`, `schema: env.JOBS_SCHEMA ?? "pgboss"`), **worker only**. On boot the worker calls `createQueue` for every name in `JobMap` with explicit options:
  - `agent.step` / `agent.chat`: policy `standard`, `retryLimit: 2` (3 attempts total), `retryDelay: 15`, `retryBackoff: true`, `expireInSeconds: 900`.
  - `demo.*` and `maintenance.*`: defaults.
- `lib/jobs/types.ts`: one discriminated union of payloads. Every payload carries the actor (ADR-003 #3):
```ts
type Base = { runId: string; projectId: string; userId: string };
export type JobMap = {
  "demo.echo":   Base & { steps: number };
  "demo.chain":  Base & { step: string };          // proves step chaining (this phase)
  "agent.chat":  Base & { messageId: string };     // Phase 06: one Planner turn per user message
  "agent.step":  Base & { step: string };          // Phase 08/09: "foundation", "components", "screen:<id>", …
  "maintenance.cleanup-uploads": {};               // from Phase 03 (no run)
  "maintenance.sweep-runs": {};
  // later: snack.sync, export.zip, export.github, build.native, publish.web, backend.provision, thumbnail.render
};
```
- `lib/jobs/enqueue.ts`: `enqueue(tx, name, data, opts?)` **inserts a `job_outbox` row using the caller's transaction** and `pg_notify('job_outbox', '')` (delivered on commit). It replaces the Phase 03 stub with the same signature, so callers don't change.
- `worker/dispatcher.ts`: on `LISTEN job_outbox` (plus a 1 s poll fallback), selects up to 50 undispatched rows `FOR UPDATE SKIP LOCKED`, calls `boss.send(name, data, options)`, then sets `dispatched_at` and `job_id` (and `runs.job_id` for run jobs). Delivery is at-least-once: a crash between `send` and the update can duplicate a job. That's harmless, because handlers are idempotent (`startRun` no-ops on terminal runs, `claimStep` rejects stale steps) and step jobs use `singletonKey: <runId>:<step>` where the chosen policy supports it.

### 3. Run lifecycle helpers
`lib/runs/lifecycle.ts`:
- `createRun(tx, { projectId, kind, input, createdBy, billedUserId })` → row with `status: "queued"`.
- `startRun(runId)` → `running`, `started_at`, `heartbeat_at`. It's a **no-op if already terminal**.
- `heartbeat(runId)` → `heartbeat_at = now()`. Long handlers call it at least every 30 s; `assertLease` also updates it.
- `finishRun(runId, output)` / `failRun(runId, {code, message, retryable})` → terminal + `finished_at` + `run.done` / `run.error` event.
- `requestCancel(runId)` → `canceled` if not terminal. Handlers check `isCanceled(runId)` between steps.
- Also update `projects.status` where relevant (Phase 06+).

### 3b. Two handler wrappers (`worker/wrap.ts`, ADR-003 #2)
```ts
// Single-shot jobs (demo.echo, agent.chat, export.*, publish.web, …)
singleShot(handler) = async (job) => {
  await startRun(job.data.runId);  await bumpAttempt(job);
  try { const out = await handler(job); await finishRun(runId, out); }
  catch (e) {
    if (isFinalAttempt(job) || !isRetryable(e)) await failRun(runId, toRunError(e));
    else { await appendEvent(…, "run.status", { status: "retrying", attempt }); throw e; } // pg-boss retries
  }
};
// Step jobs (agent.step, demo.chain): never finish the run themselves
stepJob(handler) = async (job) => {
  const lease = await claimStep(runId, job.data.step);  if (!lease) return;   // stale delivery: ack
  await startRun(runId);
  try { const next = await handler(job, lease); await advance(runId, lease, next); } // advance(null) finishes
  catch (e) {
    if (e instanceof LeaseLost) return;                               // a newer execution owns it
    if (isFinalAttempt(job) || !isRetryable(e)) await failRun(runId, toRunError(e));
    else throw e;
  }
};
```
- `isFinalAttempt(job)`: `job.retryCount >= retryLimit` (check the field name in the pinned pg-boss version).
- `AppError` with `retryable: false` fails immediately; unknown errors are retryable.
- **Retried `agent.chat`:** on attempt > 1, emit `assistant.reset { messageId }` before re-streaming, so the client discards the partial text from the failed attempt instead of showing it twice.

### 3c. Step chaining + leases (`lib/runs/steps.ts`)
```ts
async function claimStep(runId, step): Promise<{ leaseId } | null>
//   UPDATE runs SET lease_id = $newLease, heartbeat_at = now()
//   WHERE id = $runId AND step = $step AND status IN ('queued','running')
//   RETURNING lease_id      → null = stale/earlier delivery or terminal run: ack and exit
async function assertLease(runId, leaseId)   // before every write; throws LeaseLost if taken over
async function advance(runId, leaseId, nextStep | null)
//   one transaction: check lease → set step = nextStep, steps[step] = "done"
//   → enqueue(tx, "agent.step", {…, step: nextStep}) via the outbox   (or finishRun when null)
```
- Because the run stays `running` while a failed step is retried, `claimStep` still matches. Only the final attempt marks it `failed`.
- A **redelivery of an earlier step** finds `runs.step` already moved on and exits.
- A **retry of the current step** takes over the lease. If the old execution is still alive, its next `assertLease` throws and it stops without writing.
- Writes inside a step (files, events) call `assertLease` first, so they're safe to repeat.

### 3d. Stuck-run sweeper
`maintenance.sweep-runs` runs every 5 min. It fails `running` runs whose `heartbeat_at` is older than 10 min (`run.error {code: "stalled", retryable: true}`). It **skips** kinds in `EXTERNAL_WAIT_KINDS` (config; empty today, because `build_native` ends at submit per ADR-003 #17). It never uses `started_at`, since long builds are legitimate.

### 4. Event append + NOTIFY
`lib/runs/events.ts`:
```ts
export async function appendEvent(projectId: string, runId: string | null, type: EventType, data: object) {
  // INSERT … RETURNING id;  then  SELECT pg_notify('project_events', projectId)
}
export async function readEvents(q: { projectId: string; runId?: string; afterId: bigint; limit?: number }): Promise<Event[]>;
```
`EventType` is the catalogue from `05-ai-agent-design.md` §8 plus:
- `plan.updated` (`{ reason: "theme" | "change" | "restore" }`: theme switches, applied changes and restores re-render platform files),
- `assistant.reset`,
- `run.status { status, attempt? }` (adds `retrying`),
- `file.written { path, revisionId, author }` (author covers `agent`, `user_inline`, `user_code`, `system`),
- `demo.tick` for this phase.

`assistant.delta` is high-frequency: batch deltas (flush every ~150 ms) into one row. Reject payloads > 8 KB. Events carry ids and paths, never file contents.

### 5. Worker process
`worker/index.ts`:
- Boot: validate env, start pg-boss, `createQueue` for all names, start the outbox dispatcher, `registerHandler()` for each job in `worker/handlers/*` using the right wrapper.
- `WORKER_CONCURRENCY` (default 4) via the pinned pg-boss version's `work()` options.
- Graceful shutdown on SIGTERM: stop fetching, let in-flight jobs reach a checkpoint (give them up to 25 s), then `boss.stop()`.
- Schedule `maintenance.cleanup-uploads` daily and `maintenance.sweep-runs` every 5 min (pg-boss `schedule`).
- `package.json` scripts: `"worker": "tsx watch worker/index.ts"`, `"worker:start": "node dist/worker/index.js"` (build with `tsup` or `tsc -p tsconfig.worker.json`).
- Share code with the web app via `lib/*`. The worker must not import anything from `app/` or Next-only modules (`next/headers`, `server-only`).

### 6. SSE endpoints
Primary: `app/api/projects/[projectId]/events/route.ts` (GET, `runtime = "nodejs"`, `dynamic = "force-dynamic"`):
- Auth: `requireProjectAccess(projectId, "viewer")`.
- Resume point: the `Last-Event-ID` header or `?after=`. A fresh connection (no id) starts from the latest event id; the client loads current state via normal GETs.
- Send `retry: 2000`, then the backlog via `readEvents({projectId, afterId})`, then wait for `NOTIFY project_events` (a dedicated `LISTEN` connection per server instance, fanning out in-process to subscribers keyed by projectId) and send new rows as `id: <event.id>\nevent: <type>\ndata: <json>\n\n`.
- Heartbeat comment `: ping` every 15 s. Close on client abort or after **~4 min** (serverless limit; the client reconnects and resumes losslessly).
- Fallback: if `LISTEN` isn't available (pooled serverless connections), poll `readEvents` every 1 s. Both paths sit behind the same function.

Secondary: `app/api/runs/[runId]/events/route.ts` is the same stream filtered by `run_id`. It closes after `run.done` / `run.error`, for places that follow a single run (e.g. the publish dialog).

### 7. Client hook
`lib/runs/use-project-events.ts`: `useProjectEvents(projectId, { onEvent })` using `EventSource` (automatic reconnect with `Last-Event-ID`). It keeps `lastEventId` in a ref, dedupes by id, and exposes `status` (`connecting`/`open`/`closed`). It's opened once per workspace mount, whether or not a run is active. The workspace store gets one reducer, `applyEvent(event)`. `useRunEvents(runId)` is a thin wrapper over the run endpoint.

Also add `GET /api/projects/:id/runs/active` → the latest non-terminal run, so the workspace can render progress state on load.

### 8. Cancel endpoint
`POST /api/runs/:runId/cancel` (editor): `requestCancel` → `run.status` event. The handler stops at its next checkpoint.

### 9. Demo jobs (prove the pipe)
- `worker/handlers/demo-echo.ts` (single-shot): for `i in 1..steps`, `appendEvent(projectId, runId, "demo.tick", {i})`, `heartbeat`, `sleep(1000)`, check cancel. A `?failOnce=1` flag throws on attempt 1, to prove retry and `run.status retrying`.
- `POST /api/dev/echo` (dev-only, `NODE_ENV !== "production"`) creates a run and enqueues via the outbox in one transaction.
- `worker/handlers/demo-chain.ts` (step job): three steps `a → b → c` using `claimStep` / `advance`, each emitting a tick. `?chain=1` on the dev endpoint runs it. It proves chaining, redelivery-ignore, lease takeover and that a retried step doesn't fail the run.
- A small dev panel in the workspace (behind `?debug=1`) shows the live ticks with a Cancel button.

### 10. Tests
- Unit: event ids are monotonic; payload size limit; `isFinalAttempt` logic.
- Integration (CI Postgres): enqueue echo → outbox dispatched → worker in-process → events 1..N persisted in order → run `succeeded`.
- Outbox: a rolled-back transaction leaves no job. A dispatcher crash after `send` but before the update causes a duplicate delivery that's harmless.
- SSE: connect, read 3 events, disconnect, reconnect with `Last-Event-ID` → receive only the rest. No gaps, no duplicates. A non-run event (`run_id` null) reaches project subscribers.
- Wrappers: a retryable error on attempt 1 keeps the run `running` and the retry succeeds. A non-retryable error fails immediately. A step job never sets `succeeded` before `advance(null)`.
- Chaining: deliver step `a` twice after it advanced → the second delivery is a no-op. Retry step `b` while a fake "old" execution holds the lease → the old one's `assertLease` throws, and `c` is enqueued exactly once.
- Sweeper: a run with a stale heartbeat is failed; a long run with fresh heartbeats isn't.
- Authz: another user gets 404 on both SSE endpoints and the cancel endpoint.

## Files to create / change
```
lib/db/schema/runs.ts                 (runs, run_events, job_outbox)
lib/jobs/{queue.ts,types.ts,enqueue.ts}
lib/runs/{lifecycle.ts,steps.ts,events.ts,listen.ts,use-project-events.ts,use-run-events.ts}
worker/index.ts  worker/dispatcher.ts  worker/wrap.ts
worker/handlers/{demo-echo.ts,demo-chain.ts,cleanup-uploads.ts,sweep-runs.ts}
app/api/projects/[projectId]/events/route.ts
app/api/runs/[runId]/events/route.ts
app/api/runs/[runId]/cancel/route.ts
app/api/projects/[projectId]/runs/active/route.ts
app/api/dev/echo/route.ts
components/workspace/debug-run-panel.tsx
tsconfig.worker.json
package.json                          (worker scripts, pinned pg-boss major)
drizzle/000N_runs_events_outbox.sql
```

## Database changes
- `runs`: add `created_by`, `billed_user_id`, `step text null`, `lease_id text null`, `heartbeat_at`, `attempt int default 0`, `steps jsonb default '{}'`, `credits_charged bigint default 0`, `credits_refunded bigint default 0`, `started_at`, `finished_at`, `job_id text` (`03-data-model.md` §4).
- New `run_events` (`id bigserial pk`, `project_id fk not null`, `run_id fk null`, `type text`, `data jsonb`, `created_at`) + indexes `(project_id, id)`, `(run_id, id)`.
- New `job_outbox` (`id bigserial pk`, `name`, `data jsonb`, `options jsonb`, `created_at`, `dispatched_at`, `job_id`) + partial index on undispatched rows.
- pg-boss creates/migrates its own `pgboss` schema on start (excluded from Drizzle).

## API surface
| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/api/projects/:projectId/events` | viewer | SSE, primary; `Last-Event-ID` / `?after=`; ~4 min then reconnect |
| GET | `/api/runs/:runId/events` | viewer (via project) | SSE filtered to one run; closes on terminal event |
| POST | `/api/runs/:runId/cancel` | editor | → `202` |
| GET | `/api/projects/:projectId/runs/active` | viewer | → `{run | null}` |
| POST | `/api/dev/echo` | user, dev only | `?chain=1`, `?failOnce=1` → `{runId}` |

## Acceptance checklist
- [ ] `npm run dev` + `npm run worker` running: triggering the echo job shows ticks live in the debug panel.
- [ ] Refreshing mid-run replays earlier ticks and continues live, with no duplicates or gaps.
- [ ] `?failOnce=1`: attempt 1 fails, the run shows `retrying`, attempt 2 succeeds, and the run ends `succeeded` (not `failed`).
- [ ] Killing the worker mid-run, then restarting it: the job is retried and the run resumes or fails cleanly with `run.error`. The sweeper fails it if heartbeats stop; it never hangs in `running` forever.
- [ ] Cancel stops the job within one step and the run ends `canceled`.
- [ ] The `demo.chain` job runs a → b → c. Re-sending an old step is ignored, and a lease takeover stops the stale execution (covered by tests).
- [ ] Creating a run whose transaction rolls back enqueues nothing (outbox test).
- [ ] An event with `run_id = null` reaches every open workspace tab of that project.
- [ ] The SSE response closes by itself after ~4 min and the client reconnects transparently.
- [ ] No tunnel/ngrok is needed for any of the above locally.
- [ ] Another user gets 404 from both SSE endpoints and the cancel and active-run endpoints.
- [ ] `drizzle-kit generate` doesn't try to touch the `pgboss` schema.

## Claude Code prompt
```text
Read first:
- docs/plan/02-architecture.md (§3 decisions A1–A3, §6 processes)
- docs/plan/03-data-model.md (§4 runs, run_events, pg-boss note)
- docs/plan/05-ai-agent-design.md (§4 pipelines/step chaining, §8 event catalogue)
- docs/plan/12-decisions-and-open-questions.md (ADR-003 items 2, 3, 5, 6, 7, 21)
- docs/plan/13-reference-implementation.md (§3 background work, R1)
- docs/plan/phases/phase-05-jobs-and-realtime.md (this phase)

Implement Phase 05:
1. Migration: complete runs (created_by, billed_user_id, step, lease_id, heartbeat_at, attempt,
   steps, credits_charged, credits_refunded, started_at, finished_at, job_id), run_events with
   project_id NOT NULL + nullable run_id, and job_outbox. Keep drizzle schemaFilter ["public"].
2. Pin pg-boss to one major (>=10) and read its README for that version. Worker-only PgBoss
   singleton (DATABASE_URL_DIRECT ?? DATABASE_URL), createQueue for every JobMap name with
   explicit policy/retry/expiry options. Typed JobMap where every run job carries
   {runId, projectId, userId}.
3. enqueue(tx, name, data) inserts a job_outbox row in the caller's transaction (replaces the
   Phase 03 stub, same call sites). worker/dispatcher.ts forwards rows to pg-boss
   (LISTEN job_outbox + 1s poll, FOR UPDATE SKIP LOCKED).
4. lib/runs/lifecycle.ts (createRun/startRun/heartbeat/finishRun/failRun/requestCancel,
   idempotent), lib/runs/steps.ts (claimStep/assertLease/advance; advance enqueues via outbox
   in the same transaction), lib/runs/events.ts (appendEvent(projectId, runId|null, …) with
   pg_notify('project_events'), readEvents, 8 KB cap, typed EventType incl. plan.updated,
   assistant.reset, run.status retrying, file.written{author}, batched assistant.delta).
5. worker/wrap.ts: singleShot and stepJob wrappers exactly as in step 3b (failRun only on final
   attempt or non-retryable; step jobs never finish the run except via advance(null)).
6. Sweeper (heartbeat-based, EXTERNAL_WAIT_KINDS skip list), daily upload cleanup.
7. SSE: /api/projects/[projectId]/events (primary) and /api/runs/[runId]/events (filtered),
   requireProjectAccess, Last-Event-ID resume, LISTEN fan-out + polling fallback, 15s heartbeat,
   ~4 min close. useProjectEvents hook (opened once per workspace) + store applyEvent.
8. Cancel + active-run endpoints, dev-only echo endpoint (?chain=1, ?failOnce=1), demo.echo and
   demo.chain handlers, debug panel behind ?debug=1.
9. Tests listed in step 10 (integration tests run against the CI Postgres service).

Do NOT add any AI/Anthropic code yet, and do not add webhooks or tunnels.
Definition of done = Acceptance checklist in the phase doc.
Finish with lint/typecheck/test, then tell me exactly how to run web + worker locally.
```

## Pitfalls & notes
- **Connection budget:** each web instance holds one `LISTEN` connection plus its pool; the worker holds its own pool plus pg-boss's. Hosted Postgres plans with low connection limits need a pooler. LISTEN/NOTIFY **does not work through transaction-mode poolers** (e.g. PgBouncer transaction mode, some serverless drivers), so use a direct connection for LISTEN, or the polling fallback.
- `pg_notify` payloads are limited to 8000 bytes. Send only the id and let listeners read the row. `NOTIFY` inside a transaction is delivered on commit, which is what the outbox relies on.
- Vercel and other serverless hosts cap function duration, so SSE responses *will* be cut. That's why resume-by-id is mandatory, not a nice-to-have.
- Make every handler step **idempotent**, because pg-boss retries and outbox redelivery can re-run a job. Phase 08 relies on `claimStep`/`assertLease` plus "skip screens already `ready`". A resumed build is never charged twice (Phase 18). Resume = the same run.
- pg-boss's `retryLimit: 2` means up to 3 attempts. Read the attempt counter from the job object of the pinned version, not from our own guess.
- The worker needs the same env as the web app (DB, storage, Anthropic key later). In production it's a separate service (Railway/Fly/Render, see Phase 19), never a Vercel function. Untrusted-code steps run in the separate **sandbox service** (ADR-003 #18), not here.
