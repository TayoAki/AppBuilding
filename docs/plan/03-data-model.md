# 03 — Data Model (Platform Database)

Postgres + Drizzle ORM. This is the **target** schema. Each phase adds only the tables it needs; the "Phase" column says when. Names here are canonical. Phase docs and Claude Code prompts refer to them by these names.

Conventions:
- Primary keys are `text` ids from a single helper, `newId(prefix)` (e.g. `prj_…`, `run_…`), so logs are readable. Alternatively use `uuid` with `defaultRandom()`. Pick one in Phase 00 and never mix (decision D3).
- Canonical prefixes: `prj` project · `ast` asset · `upl` temporary upload (not a table, see §2) · `msg` message · `run` run · `rev` file revision · `snp` snapshot · `scr` screen · `mbr` project member · `con` oauth connection · `bld` native build · `dep` web deployment · `use` ai_usage · `led` ledger entry. Better Auth keeps its own ids for `user`/`session`/`account`.
- Every table has `created_at timestamptz default now()`. Mutable tables also get `updated_at`.
- Soft delete where users can delete things (`deleted_at`).
- Money is stored as integer **micro-credits**, never floats (see §8).
- Encrypted columns end in `_enc` and hold `v1:<iv>:<ciphertext>:<tag>` strings produced by `lib/crypto/encrypt.ts`.

---

## 1. Auth (Phase 02)

Better Auth owns `user`, `session`, `account`, `verification`. Generate them with the Better Auth CLI / Drizzle adapter. Don't hand-write them. We **extend** `user` with:

| column | type | notes |
|---|---|---|
| `plan` | `text` default `'free'` | `free` \| `pro` (denormalized from subscription for fast gating) |
| `credits_balance` | `bigint` default `0` | never-expiring credits (signup grant, refunds, top-ups); cache of its ledger sum |
| `role` | `text` default `'user'` | `user` \| `admin` (admin pages: showcase approval, unpublish, kill switch) |
| `monthly_credits_balance` | `bigint` default `0` | Pro monthly allowance; **spent first**; replaced on renewal, zeroed when Pro ends (Phase 18) |
| `onboarded_at` | `timestamptz` null | |

> The video stores `credits = 100` directly on the user row. We keep a balance column for speed, but the **ledger is the source of truth** (Phase 18).

## 2. Projects (Phase 03)

### `projects`
| column | type | notes |
|---|---|---|
| `id` | text pk | `prj_…` |
| `owner_id` | text fk → user | |
| `title` | text | AI-suggested app name, user-renamable |
| `prompt` | text | original prompt |
| `platforms` | text[] | `{ios,android}` |
| `status` | text | `planning` (no plan yet) · `awaiting_confirmation` (plan proposed) · `building` · `ready` · `failed` (with N screens possibly built) |
| `visibility` | text default `'private'` | `private` · `unlisted` · `public` (public = shown on home showcase) |
| `current_plan_message_id` | text null fk → messages | the latest `plan_card` message (Phase 06) |
| `plan` | jsonb null | **the current plan is the truth** (`05` §3.2): `AppPlan` + `themePackage {themes, activeThemeId}` (the **only** source of themes, §6) + `collections` + `components` (name, purpose) + `pendingChanges` + `backend` (public config only). Platform files are rendered from it (`06` §10). Every write emits `plan.updated`. |
| `slug` | text null | set once at first build/publish with a project-id suffix (Phase 15) |
| `snack` | jsonb null | Path A only: `{snackId, url, savedAt, snapshotId}` of the last persistent Snack save (Phase 11) |
| `bundle_id`, `android_package` | text null | iOS bundle identifier / Android package. Derived from slug, **locked after the first store build** (Phase 15) |
| `eas_project_id` | text null | EAS project id on the **owner's** Expo account (Phase 15/16; builds and publish are owner-only, ADR-003 #3) |
| `eas_webhook_id`, `eas_webhook_secret_enc` | text null | per-project EAS webhook and its secret (created after EAS init; ADR-003 #17) |
| `showcase_reviewed_by`, `showcase_reviewed_at` | text / timestamptz null | admin who approved/rejected (Phase 17) |
| `showcase_status` | text default `'none'` | `none` · `pending` · `approved` · `rejected`: homepage showcase needs admin approval (D17); `visibility = public` alone only makes the link work |
| `remixed_from_project_id` | text null | set when duplicated from a public project (Phase 17) |
| `template_version` | text | e.g. `expo-58@1` — which template the project was generated against |
| `thumbnail_key` | text null | storage key of latest screenshot |
| `deleted_at` | timestamptz null | |
| `updated_at` | timestamptz | |

### `project_assets`
User uploads (reference images) and generated assets.
| column | type | notes |
|---|---|---|
| `id` | text pk | `ast_…` |
| `project_id` | fk | |
| `kind` | text | `reference_image` · `chat_attachment` · `app_image` · `icon` · `splash` |
| `storage_key` | text | `users/{userId}/projects/{projectId}/assets/{assetId}.{ext}` |
| `mime_type`, `size_bytes`, `width`, `height` | | validated server-side |
| `status` | text | `pending_upload` · `ready` · `rejected` |
| `external_url`, `prompt`, `provider`, `provider_ref` | text null | generated images (Phase 12): provider URL, generation prompt, provider name, provider id |
| `seed_key` | text null | stable registry key for generated images; unique per project |

Uploads made **before** a project exists (the home prompt box) don't create rows. They go to `tmp/{userId}/{uploadId}` with an HMAC upload token, and `POST /api/projects` verifies them, copies them to final keys and inserts `project_assets` rows in the same transaction as the project (Phase 03). A bucket lifecycle rule expires `tmp/` after 1 day. `pending_upload` is only used for attachments added inside an existing project (chat, Phase 09).

### `project_members` (Phase 14)
| `project_id` | `user_id` null | `email` | `role` (`viewer`/`editor`) | `invited_by` | `invite_token_hash` | `invite_expires_at` | `accepted_at` |
Unique (project_id, email). The owner is implicit via `projects.owner_id`.

## 3. Conversation (Phase 06)

### `messages`
| column | type | notes |
|---|---|---|
| `id` | text pk | `msg_…` |
| `project_id` | fk | |
| `run_id` | fk null | the run that produced / was triggered by this message |
| `role` | text | `user` · `assistant` · `system_note` |
| `kind` | text | `text` · `plan_card` · `build_summary` · `error` · `element_ref` · `session_break` (Start new session; the Planner replays only messages after the last one) |
| `content` | text | markdown text shown in chat |
| `payload` | jsonb | e.g. `AppPlan` for plan cards; `{afid, file, snippet}` for element refs |
| `attachments` | jsonb | asset ids |
| `prompt_version` | text null | Planner prompt + tool-set version of the session (set on each Planner turn). If the deployed version differs from the latest message's, the server inserts a `session_break` before the next turn (ADR-003 #1) |


> The Planner's raw API turns are stored unchanged in `messages.payload.api`: the user message, **that turn's PROJECT STATE system message**, and the assistant turns including tool calls and thinking blocks. They're replayed byte-for-byte, append-only, for the current session; earlier state messages are never rebuilt or removed (`05` §3.1, §9). Generators are stateless single calls (plus at most one repair), so they keep no transcript. UI `content` is the user-facing projection.

## 4. Runs, jobs and events (Phase 05)

### `runs`
One row per agent/worker task the user can see progress for.
| column | type | notes |
|---|---|---|
| `id` | text pk | `run_…` |
| `project_id` | fk | |
| `created_by` | text fk → user | the acting user (who clicked or typed) |
| `billed_user_id` | text fk → user | who pays: always the project **owner** (ADR-003 #3); `ai_usage.user_id` and ledger rows use it |
| `kind` | text | `chat` (one Planner turn) · `build` (step chain after confirm) · `change` (step chain after `update_app`) · `snack_sync` · `export_zip` · `export_github` · `build_native` · `publish_web` · `backend_provision` · `thumbnail` · `template_upgrade` · `account_delete` · `export_my_data` |
| `status` | text | `queued` · `running` · `succeeded` · `failed` · `canceled` |
| `input` | jsonb | job params |
| `output` | jsonb | result summary (urls, file list…) |
| `error` | jsonb null | `{code, message, retryable}` |
| `steps` | jsonb | per-step status for build/change runs (`{foundation: done, components: done, "screen:today": failed, …}`), used to resume and to compute refunds |
| `credits_charged` / `credits_refunded` | bigint | micro-credits charged up front for this run / refunded for unbuilt work (Phase 18) |
| `started_at`, `finished_at` | timestamptz | |
| `job_id` | text | pg-boss job id of the current step |
| `step` | text null | current pipeline step (`foundation`, `components`, `screen:<id>`, …). A redelivered job for an earlier step is ignored. |
| `lease_id` | text null | the execution that owns the step; a retry takes over the lease, and the older execution stops at its next write |
| `heartbeat_at` | timestamptz null | updated by the running step every ~30 s; the stuck-run sweeper uses it, never `started_at` (ADR-003 #2) |
| `attempt` | int default 1 | attempt number of the current step/job; a retried `agent.chat` emits `assistant.reset` first |

Job payloads are always `{runId, projectId, userId}`; handlers re-check that `userId` may act on `projectId` (`08` §2). Single-shot jobs finish or fail the run themselves. Step jobs never finish the run; only the final `advance(null)` does, and `failRun` runs only on the last attempt or a non-retryable error.

### `run_events`
Append-only, **project-scoped** stream that powers SSE (ADR-003 #5). Run events and non-run writes (inline edits, code saves, theme switches, restores) share it.
| column | type | notes |
|---|---|---|
| `id` | bigserial pk | **monotonic**, used as SSE `id` / `Last-Event-ID` |
| `project_id` | fk, not null | index (project_id, id) |
| `run_id` | fk null | null for writes outside a run; index (run_id, id) |
| `type` | text | see the event catalogue in `05-ai-agent-design.md` §8 |
| `data` | jsonb | small payloads only (file paths + revision ids + author, not file contents) |

After each insert, call `NOTIFY project_events, '<project_id>'`. The primary SSE endpoint is `/api/projects/:id/events`, open while the workspace is open. `/api/runs/:id/events` is the same stream filtered to one run. Retention: 30 days (`08` §11).

### `job_outbox` (Phase 05)
Transactional outbox, so a run, its credit charge and its job are created atomically on the web's connection (ADR-003 #7).
| `id` bigserial pk | `queue` text | `payload` jsonb | `run_id` fk | `created_at` | `dispatched_at` null |
A dispatcher loop in the worker sends undispatched rows to pg-boss (on `DATABASE_URL_DIRECT`) and stamps `dispatched_at`. Jobs are idempotent by `runId` + step, so a double dispatch is harmless.

### pg-boss
pg-boss (pinned major ≥ 10) creates its own `pgboss` schema. Don't model it in Drizzle. Exclude it from `drizzle-kit push` with `schemaFilter: ["public"]`. Every queue is created with `createQueue()` and an explicit policy (Phase 05).

## 5. Virtual file system (Phase 07/08)

### `project_files`
The current state of each file.
| column | type | notes |
|---|---|---|
| `project_id` | fk | pk part |
| `path` | text | pk part — POSIX relative path, e.g. `app/(tabs)/index.tsx` |
| `current_revision_id` | fk → file_revisions | |
| `kind` | text | `screen` · `component` · `theme` · `data` · `lib` · `backend` · `config` |
| `screen_id` | text null | links files to plan screens |
| `deleted` | bool default false | |

### `file_revisions`
| column | type | notes |
|---|---|---|
| `id` | text pk | `rev_…` |
| `project_id`, `path` | | |
| `content` | text | full contents (files are small; skip delta storage in v1) |
| `sha256` | text | dedupe and cache key for the preview transpiler |
| `author` | text | `agent` · `user_inline` · `user_code` · `system` (template upgrades) |
| `run_id` | fk null | |
| `parent_revision_id` | fk null | |

### `project_snapshots` (Phase 08, used by export/undo)
A point-in-time map `files: {path → revision_id}` (jsonb), plus `plan` (jsonb copy of `projects.plan`, so restore brings back themes, collections and components), `screens` (jsonb, the screen list at that time, so restore can bring back deleted screens), `label` (text, e.g. "After: add settings screen"), and `run_id`. Created at the end of every successful run, so "Restore previous version" is one row copy.

### `screens`
Denormalized list for the canvas and plan ↔ code linkage.
| `id` (`scr_…`) | `project_id` | `name` | `route` (e.g. `/(tabs)/index`) | `file_path` | `order` | `status` (`planned`/`building`/`ready`/`error`) | `canvas_x`, `canvas_y` |

## 6. Themes (Phase 08)

`projects.plan.themePackage {themes, activeThemeId}` is the **only** source of truth for themes (ADR-003 #4). `theme/themes.ts` is a platform-rendered file, never stored in `project_files`, and re-rendered from the plan on demand. The theme picker reads swatches straight from the plan. A theme switch (picker or the Planner's `set_theme`) updates `activeThemeId`, emits `plan.updated`, and the preview swaps CSS variables (`06` §5). Snack/dev-server sessions receive the re-rendered `theme/themes.ts`.

## 7. Integrations

### `oauth_connections` (Phase 13/14/15)
One table for every third-party account the user links.
| column | type | notes |
|---|---|---|
| `id` | text pk | |
| `user_id` | fk | |
| `provider` | text | `github` · `expo` · `backend:<name>` (e.g. `backend:supabase`, `backend:cloudhike`) |

> `provider = 'github'` here is the **export GitHub App** user-to-server token (Phase 14). It's unrelated to Better Auth's GitHub *sign-in* account, which lives in Better Auth's `account` table.

| `external_account_id`, `external_login` | text | |
| `access_token_enc`, `refresh_token_enc` | text | encrypted |
| `scopes` | text | |
| `expires_at` | timestamptz null | |
Unique (user_id, provider).

### `project_backends` (Phase 13)
| `project_id` pk | `provider` | `mode` (`direct` = provider client + RLS with auth always deployed and anonymous sign-in · `gateway` = platform `/api/apps/[id]/…`) | `external_project_id` | `public_config` jsonb (URL + **publishable** key: safe to ship in the app) | `secret_config_enc` (secret/service keys and `db_pass`: never shipped) | `services` jsonb (`[{type, status}]`) | `schema_version` int | `status` (`connecting` · `provisioning` · `ready` · `failed` · `reconnect_required`) | `error` |

### `device_sessions` (Phase 11, Path B)
| `id` | `project_id` | `started_by` | `mode` (`devserver` \| `snack`) | `sandbox_instance` | `url` (exp:// URL shown as QR; unguessable) | `status` (`starting` · `ready` · `stopped` · `failed`) | `clients` jsonb (if the dev server reports them) | `last_activity_at` | `error` |
One live session per project; idle-stopped after 15 minutes without a connected client (`08` §5).

### `project_github_repos` (Phase 14)
| `project_id` pk | `owner` | `repo` | `default_branch` | `last_pushed_snapshot_id` | `last_commit_sha` |

### `native_builds` (Phase 15)
| `id` | `project_id` | `requested_by` | `platform` (`android`/`ios`) | `artifact` (`apk`/`aab`/`ipa`) | `version`, `build_number` | `eas_build_id` | `status` (`queued` · `submitting` · `in_queue` · `building` · `finished` · `errored` · `canceled` · `needs_ios_setup`) | `artifact_url` | `logs_url` | `error` |

### `project_signing` (Phase 15)
| `project_id` pk | `android_keystore_enc` (base64 jks, encrypted) | `android_keystore_password_enc` | `android_key_alias` | `android_key_password_enc` | `ios_asc_key_enc` (p8) | `ios_asc_key_id` | `ios_asc_issuer_id` | `apple_team_id` |
Also: `android_fingerprint` (SHA-256 of the signing cert, recorded whichever way the keystore was created) and `apple_team_type` (`INDIVIDUAL` · `COMPANY_OR_ORGANIZATION` · `IN_HOUSE`).
Back this up. Losing the keystore means you can never update the app on Google Play.

### `web_deployments` (Phase 16)
| `id` | `project_id` | `provider` (`eas_hosting`) | `url` | `alias` | `status` | `snapshot_id` | `log_tail` (last ~200 lines for the UI) |
Also: `deployment_id` (EAS Hosting deployment id of the live version).
One row per project (`project_id` unique). `url` / live URL changes only when a publish succeeds, so a failed update leaves the old version online.

## 8. Billing and credits (Phase 06 logs usage; Phase 18 enforces)

### `ai_usage`
One row per Anthropic API response. Written from Phase 06 on, even before billing exists.
| `id` | `run_id` | `user_id` | `model` | `input_tokens` | `output_tokens` | `cache_read_tokens` | `cache_write_tokens` | `cost_usd_micros` | `served_model` (if a fallback ran) |

### `credit_ledger`
Append-only. Each balance equals the sum of its own rows: `credits_balance = sum(amount where balance='credits')`, `monthly_credits_balance = sum(amount where balance='monthly')`.
| `id` | `user_id` | `amount` bigint (±micro-credits) | `balance` (`credits` \| `monthly`) | `reason` (`signup_grant`, `monthly_grant`, `monthly_reset`, `charge`, `refund`, `purchase`, `admin_adjust`) | `run_id` null | `reference` text **unique** (idempotency: `signup:<userId>`, `charge:<runId>`, `refund:<runId>:<screenId>`, `invoice:<stripeInvoiceId>`, `monthly_reset:<subscriptionId>:<periodEnd>`) |

### `processed_webhook_events`
| `provider` (`stripe` · `eas` · …) | `event_id` | `processed_at` | (pk `provider, event_id`)
Insert first, in the handler's transaction; a duplicate key means the event was already processed, so return 200.

### `signup_grants`
| `email_hash` pk (sha256 of the normalized, verified email) | `granted_at` |
The signup credit grant happens **on email verification** (social sign-ins count as verified) with ledger reference `signup:<userId>`, and only if no row exists for the email hash. Rows survive account deletion, so re-signing up doesn't re-grant (ADR-003 #22).

### `subscriptions`
| `user_id` pk | `provider` (`stripe`) | `customer_id` | `subscription_id` | `plan` | `status` | `current_period_end` |

## 8b. Ops (Phase 19)

- `system_flags` (`key` pk, `value` jsonb, `updated_by`, `updated_at`): e.g. `generation_disabled`, read with a short cache.
- `worker_heartbeats` (`worker_id` pk, `host`, `version`, `last_seen_at`): powers the "worker down" alert and admin page.

## 9. Indexes to create from the start

- `projects (owner_id, deleted_at, updated_at desc)` — home grid
- `projects (visibility, updated_at desc) where visibility = 'public'` — showcase
- `run_events (project_id, id)`, `run_events (run_id, id)`
- `job_outbox (dispatched_at) where dispatched_at is null`
- `runs (project_id, created_at desc)`
- `file_revisions (project_id, path, created_at desc)`
- `messages (project_id, created_at)`
- `credit_ledger (user_id, created_at)`; unique `(reference)`

## 10. Generated-app databases are separate

The tables above belong to **the platform**. Each **generated app's** data (habits, expenses …) lives in the backend the user connects in Phase 13, in tables **derived deterministically from `projects.plan.collections`** by platform code. Each is an `app_<name>` table with `id`, `owner_id`, timestamps and one typed column per field (`string`→`text`, `number`→`double precision`, `boolean`→`boolean`, `string[]`→`text[]`, `references`→ FK `on delete set null`), plus one `app_values` table for `useStoredValue`. Migrations are **additive only** and run in one locked transaction. The agent never writes SQL. Never store generated-app user data in the platform DB, and never as a single JSON blob table (the reference app had to migrate away from exactly that).
