# Phase 05 — Background Jobs and Realtime Events

| | |
|---|---|
| Video chapter(s) | New (not in video as a phase). It pulls forward the background-job fix from Ch.11 "Generate Mobile App with AI" (~1:20:46) |
| Branch | `feat/05-jobs-realtime` |
| Depends on | Phase 03 (runs table), Phase 04 (workspace store) |
| Effort (solo dev + Claude Code) | ~2 days |
| New env vars | `DATABASE_URL_DIRECT` (optional; direct/session-mode connection for pg-boss and `LISTEN`). Optional: `WORKER_CONCURRENCY`, `JOBS_SCHEMA` (default `pgboss`) |

## Goal
Build the plumbing every long task uses: a Postgres-backed queue (pg-boss), a separate long-running **worker** process, `runs` + `run_events` tables, an `appendRunEvent()` that wakes listeners via `NOTIFY`, and an SSE endpoint with `Last-Event-ID` resume that the workspace subscribes to. Prove it end-to-end with a demo `echo` job before any AI exists. After this phase, the agent (Phase 06+) never runs inside a request handler.

## What we do differently from the video (and why)
- **The worker exists before the agent.** The video built generation inside an API route and found in Ch.11 that Vercel kills functions after 15 s–5 min, then retrofitted CloudHike Jobs. We design for minutes-long work from day one (`02-architecture.md` A1).
- **No inbound webhook to our app, so no ngrok in dev.** The video's job service called back to a public URL, which forced a tunnel and broke sign-in (redirect/origin mismatch). Our worker connects **outbound** to Postgres, both locally (`npm run worker`) and in production.
- **Events are persisted**, so a refresh or reconnect replays everything missed. The video had to refresh to see the agent working.
- **Vendor-neutral `lib/jobs` interface.** pg-boss today; Inngest, Trigger.dev or CloudHike Jobs can replace it behind the same `enqueue()` / `registerHandler()` API.
- **Step chaining with leases from the start.** Agent pipelines run one step per job, with each step enqueueing the next (`05-ai-agent-design.md` §4). This phase provides the `runs.step` / `runs.lease_id` primitives.

**Reference app notes** (`13-reference-implementation.md`)
- R1/§3: the reference app runs each build step as its own background job that queues the next one. A redelivered earlier step is ignored (`builds.step`), and a retry takes over a lease (`builds.lease`), which stops the older execution at its next write. We adopt the same pattern on `runs.step` / `runs.lease_id`.
- §3: their job service only returns a run's final answer, so the worker saved batched events to a table and the browser replayed them. That's the same idea as our `run_events` + SSE. But their jobs called back to a public URL (ngrok locally), and a build nobody watched showed "building" until the workspace was reopened. Our worker pulls from Postgres, so neither problem exists.
- §4: preview frames polled constantly, and a render loop flooded the dev server. Clients here subscribe to SSE only while a run is active and never poll the files API in a loop.

## Steps

### 1. Tables
Complete `runs` per `03-data-model.md` §4 (add `transcript`, `credits_reserved`, `credits_charged`, `started_at`, `finished_at`, `job_id`). Add `run_events` (`id bigserial`, `run_id`, `type`, `data jsonb`, `created_at`) with index `(run_id, id)`. Make sure `drizzle.config.ts` keeps `schemaFilter: ["public"]`.

### 2. Queue
`lib/jobs/queue.ts`: a singleton `PgBoss` (`new PgBoss({ connectionString: env.DATABASE_URL, schema: "pgboss" })`), started lazily. **Web** processes only `send`; the **worker** also `work`s.

`lib/jobs/types.ts`: one discriminated union of job payloads, so producers and handlers share types:
```ts
export type JobMap = {
  "demo.echo":   { runId: string; steps: number };
  "demo.chain":  { runId: string; step: string };   // proves step chaining (this phase)
  "agent.chat":  { runId: string; messageId: string }; // Phase 06: one Planner turn per user message
  "agent.step":  { runId: string; step: string };   // Phase 08/09: one pipeline step
                                                    // ("foundation", "components", "screen:<id>", …)
  "maintenance.cleanup-uploads": {};                 // from Phase 03
  // later: snack.sync, export.zip, export.github, build.native, publish.web, backend.provision, thumbnail.render
};
```

`lib/jobs/enqueue.ts`: `enqueue<K extends keyof JobMap>(name, data, opts?)`. Default options: `retryLimit: 2`, `retryDelay: 15`, `retryBackoff: true`, `expireInMinutes: 15` per step, `singletonKey: <runId>:<step>` (no duplicate jobs per step). It stores the returned job id in `runs.job_id`. It replaces the Phase 03 stub.

Connect pg-boss with `DATABASE_URL_DIRECT ?? DATABASE_URL`. pg-boss and `LISTEN` both need a direct/session-mode connection.

### 3. Run lifecycle helpers
`lib/runs/lifecycle.ts`:
- `createRun(projectId, kind, input)` → row with `status: "queued"`.
- `startRun(runId)` → `running`, `started_at`; **no-op if already terminal** (idempotent on retry).
- `finishRun(runId, output)` / `failRun(runId, {code, message, retryable})` → terminal + `finished_at` + `run.done` / `run.error` event.
- `requestCancel(runId)` → sets `runs.status = "canceled"` if not terminal. Handlers poll `isCanceled(runId)` between steps.
- Also update `projects.status` where relevant (Phase 06+).

### 3b. Step chaining + leases (`lib/runs/steps.ts`)
The pattern every agent pipeline uses (`05-ai-agent-design.md` §4):
```ts
// claim: only proceeds if this delivery is for the run's current step
async function claimStep(runId, step): Promise<{ leaseId } | null>
//   UPDATE runs SET lease_id = $newLease
//   WHERE id = $runId AND step = $step AND status IN ('queued','running')
//   RETURNING lease_id      → null means a stale/earlier delivery: ack and exit
async function assertLease(runId, leaseId)   // before every write; throws LeaseLost if taken over
async function advance(runId, leaseId, nextStep | null)
//   in one transaction: check lease → set step = nextStep → enqueue agent.step(nextStep)
//   (or finishRun when nextStep is null)
```
- A **redelivery of an earlier step** finds `runs.step` already moved on and exits without doing anything.
- A **retry of the current step** takes over the lease. If the old execution is somehow still alive, its next `assertLease` throws, and it stops without writing.
- Writes inside a step (files, events) call `assertLease` first, so they're safe to repeat.
- Enqueueing the next step happens in the same transaction as `step = next`. pg-boss writes to the same Postgres, so that's possible, and there's no window where a step completes but its successor is never queued.

### 4. Event append + NOTIFY
`lib/runs/events.ts`:
```ts
export async function appendRunEvent(runId: string, type: RunEventType, data: object) {
  // INSERT … RETURNING id;  then  SELECT pg_notify('run_events', runId)
}
export async function readRunEvents(runId: string, afterId: bigint, limit = 500): Promise<RunEvent[]>;
```
`RunEventType` is the catalogue from `05-ai-agent-design.md` §8 (`run.status`, `thinking.summary`, `assistant.delta`, `assistant.done`, `plan.ready`, `theme.ready`, `components.ready`, `screen.started`, `screen.done`, `screen.failed`, `file.writing`, `file.written`, `validation.result`, `upgrade`, `backend.status`, `export.progress`, `build.status`, `publish.step`, `run.error`, `run.done`) plus `demo.tick` for this phase. `assistant.delta` is high-frequency: batch deltas (flush every ~150 ms) into one row so a long reply doesn't become thousands of rows. Reject payloads > 8 KB. Events carry ids and paths, never file contents.

### 5. Worker process
`worker/index.ts`:
- Boot: validate env, start pg-boss, `registerHandler()` for each job in `worker/handlers/*`.
- `WORKER_CONCURRENCY` (default 4) via pg-boss `teamSize`/`batchSize` options (check the pg-boss version's option names).
- Each handler is wrapped: `startRun` → handler → `finishRun`, with `failRun` on throw. Unknown errors are retryable once; `AppError` with `retryable: false` → `fail` without retry.
- Graceful shutdown on SIGTERM: stop fetching, let in-flight jobs reach a checkpoint (give them up to 25 s), then `boss.stop()`.
- Schedule `maintenance.cleanup-uploads` daily (pg-boss `schedule`).
- `package.json` scripts: `"worker": "tsx watch worker/index.ts"`, `"worker:start": "node dist/worker/index.js"` (build with `tsup` or `tsc -p tsconfig.worker.json`).
- Share code with the web app via `lib/*`. The worker must not import anything from `app/` or Next-only modules (`next/headers`, `server-only`).

### 6. SSE endpoint
`app/api/runs/[runId]/events/route.ts` (GET, `runtime = "nodejs"`, `dynamic = "force-dynamic"`):
- Auth: load the run → `requireProjectAccess(run.projectId, "viewer")`.
- Resume point: the `Last-Event-ID` header or the `?after=` query param (default 0).
- Send `retry: 2000`, then backlog via `readRunEvents(after)`, then wait for `NOTIFY run_events` (a dedicated `LISTEN` connection per server instance, fanning out in-process to subscribers keyed by runId) and send new rows. Each message is `id: <event.id>\nevent: <type>\ndata: <json>\n\n`.
- Heartbeat comment `: ping` every 15 s.
- Close after `run.done` / `run.error` + flush, on client abort, or after **~4 min** (serverless limit; the client reconnects and resumes losslessly).
- Fallback: if LISTEN isn't available (e.g. pooled serverless connections that don't support it), poll `readRunEvents` every 1 s instead. Keep both paths behind the same function.

### 7. Client hook
`lib/runs/use-run-events.ts`: `useRunEvents(runId, { onEvent })` using `EventSource` (automatic reconnect with `Last-Event-ID`). It keeps `lastEventId` in a ref and dedupes by id. On a terminal event it closes. It exposes `status` (`connecting`/`open`/`closed`). The workspace store gets one reducer, `applyRunEvent(event)`.

Also add `GET /api/projects/:id/runs/active` → the latest non-terminal run, so the workspace knows what to subscribe to on load.

### 8. Cancel endpoint
`POST /api/runs/:runId/cancel` (editor): `requestCancel` → `run.status` event. The handler stops at its next checkpoint.

### 9. Demo echo job (proves the pipe)
- `worker/handlers/demo-echo.ts`: for `i in 1..steps`, `appendRunEvent(runId, "demo.tick", {i})`, `sleep(1000)`, check cancel.
- `POST /api/dev/echo` (dev-only, `NODE_ENV !== "production"`) creates a run + enqueues `demo.echo`.
- `worker/handlers/demo-chain.ts`: three steps `a → b → c` using `claimStep` / `advance`, each emitting a tick. `?chain=1` on the dev endpoint runs it. It proves chaining, redelivery-ignore and lease takeover before Phase 08 depends on them.
- A small dev panel in the workspace (behind `?debug=1`) shows the live ticks with a Cancel button.

### 10. Tests
- Unit: `appendRunEvent` ids are monotonic; payload size limit.
- Integration (CI Postgres): enqueue echo → run the worker in-process → events 1..N persisted in order → run `succeeded`.
- SSE: connect, read 3 events, disconnect, reconnect with `Last-Event-ID` → receive only the rest. No gaps, no duplicates.
- Idempotency: run the same job twice (simulated retry) → `startRun` doesn't reset a terminal run.
- Chaining: deliver step `a` twice after it advanced → the second delivery is a no-op. Retry step `b` while a fake "old" execution holds the lease → the old one's `assertLease` throws, and `c` is enqueued exactly once.
- Authz: another user gets 404 on the SSE and cancel endpoints.

## Files to create / change
```
lib/db/schema/runs.ts                 (complete runs + run_events)
lib/jobs/{queue.ts,types.ts,enqueue.ts}
lib/runs/{lifecycle.ts,steps.ts,events.ts,listen.ts,use-run-events.ts}
worker/index.ts
worker/handlers/{demo-echo.ts,demo-chain.ts,cleanup-uploads.ts}
app/api/runs/[runId]/events/route.ts
app/api/runs/[runId]/cancel/route.ts
app/api/projects/[projectId]/runs/active/route.ts
app/api/dev/echo/route.ts
components/workspace/debug-run-panel.tsx
tsconfig.worker.json
package.json                          (worker scripts)
drizzle/000N_runs_events.sql
```

## Database changes
- `runs`: add `transcript jsonb default '[]'`, `credits_reserved bigint default 0`, `credits_charged bigint default 0`, `started_at`, `finished_at`, `job_id text`, `step text null`, `lease_id text null` (`03-data-model.md` §4).
- New `run_events` (`id bigserial pk`, `run_id fk`, `type text`, `data jsonb`, `created_at`) + index `(run_id, id)`.
- pg-boss creates/migrates its own `pgboss` schema on start (excluded from Drizzle).

## API surface
| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/api/runs/:runId/events` | viewer | SSE; `Last-Event-ID` / `?after=`; closes on terminal event or ~4 min |
| POST | `/api/runs/:runId/cancel` | editor | → `202` |
| GET | `/api/projects/:projectId/runs/active` | viewer | → `{run | null}` |
| POST | `/api/dev/echo` | user, dev only | → `{runId}` |

## Acceptance checklist
- [ ] `npm run dev` + `npm run worker` running: triggering the echo job shows ticks live in the debug panel.
- [ ] Refreshing mid-run replays earlier ticks and continues live, with no duplicates or gaps.
- [ ] Killing the worker mid-run, then restarting it: pg-boss retries, and the run either resumes or fails cleanly with `run.error`. It never hangs in `running` forever (an expiry marks it failed).
- [ ] Cancel stops the job within one step and the run ends `canceled`.
- [ ] The `demo.chain` job runs a → b → c. Re-sending an old step is ignored, and a lease takeover stops the stale execution (covered by tests).
- [ ] The SSE response closes by itself after ~4 min and the client reconnects transparently.
- [ ] No tunnel/ngrok is needed for any of the above locally.
- [ ] Another user gets 404 from the SSE, cancel and active-run endpoints.
- [ ] `drizzle-kit generate` doesn't try to touch the `pgboss` schema.

## Claude Code prompt
```text
Read first:
- docs/plan/02-architecture.md (§3 decisions A1–A3, §6 processes)
- docs/plan/03-data-model.md (§4 runs, run_events, pg-boss note)
- docs/plan/05-ai-agent-design.md (§4 pipelines/step chaining, §8 run event catalogue)
- docs/plan/13-reference-implementation.md (§3 background work, R1)
- docs/plan/phases/phase-05-jobs-and-realtime.md (this phase)

Implement Phase 05:
1. Complete runs + add run_events (migration). Keep drizzle schemaFilter ["public"].
2. lib/jobs: pg-boss singleton (schema "pgboss", DATABASE_URL_DIRECT ?? DATABASE_URL),
   typed JobMap (demo.echo, demo.chain, agent.chat, agent.step, maintenance.*), enqueue() with
   retry/backoff/expiry/singletonKey=<runId>:<step>; replace the Phase 03 enqueue stub.
3. lib/runs/lifecycle.ts (createRun/startRun/finishRun/failRun/requestCancel/isCanceled,
   idempotent) and lib/runs/events.ts (appendRunEvent with pg_notify, readRunEvents,
   8 KB payload cap, typed RunEventType from the 05 §8 catalogue + demo.tick, batched
   assistant.delta).
3b. lib/runs/steps.ts: claimStep / assertLease / advance (runs.step + runs.lease_id;
   advance sets the next step and enqueues it in one transaction).
4. worker/index.ts: separate Node process (npm run worker = tsx watch), registers
   handlers, wraps them with lifecycle, graceful SIGTERM shutdown, daily cleanup schedule.
   The worker must not import from app/ or Next-only modules.
5. SSE route /api/runs/[runId]/events: auth via requireProjectAccess, Last-Event-ID
   resume, backlog then LISTEN/NOTIFY fan-out (one LISTEN connection per instance),
   polling fallback, 15s heartbeat, close on terminal event or ~4 min.
6. useRunEvents hook (EventSource, dedupe by id) + workspace store applyRunEvent.
7. Cancel endpoint, active-run endpoint, dev-only echo endpoint + demo.echo and demo.chain
   handlers, debug panel behind ?debug=1.
8. Tests listed in step 10 (integration tests run against the CI Postgres service).

Do NOT add any AI/Anthropic code yet, and do not add webhooks or tunnels.
Definition of done = Acceptance checklist in the phase doc.
Finish with lint/typecheck/test, then tell me exactly how to run web + worker locally.
```

## Pitfalls & notes
- **Connection budget:** each web instance holds one `LISTEN` connection plus its pool; the worker holds its own pool plus pg-boss's. Hosted Postgres plans with low connection limits need a pooler. LISTEN/NOTIFY **does not work through transaction-mode poolers** (e.g. PgBouncer transaction mode, some serverless drivers), so use a direct connection for LISTEN, or the polling fallback.
- `pg_notify` payloads are limited to 8000 bytes. Send only the `runId` and let listeners read the row.
- Vercel and other serverless hosts cap function duration, so SSE responses *will* be cut. That's why resume-by-id is mandatory, not a nice-to-have.
- Make every handler step **idempotent**, because pg-boss retries can re-run a job. Phase 08 relies on `claimStep`/`assertLease` plus "skip screens already `ready`". A resumed build is never charged twice (Phase 18).
- A job stuck in `running` after a crash: rely on pg-boss expiry + a `failRun` in the `onComplete`/failed hook, and add a periodic sweeper that fails runs with `started_at` older than 1 h.
- The worker needs the same env as the web app (DB, storage, Anthropic key later). In production it's a separate service (Railway/Fly/Render, see Phase 19), never a Vercel function.
