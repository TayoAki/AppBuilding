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
| `credits_balance` | `bigint` default `0` | micro-credits; **cache** of `credit_ledger` sum, updated in the same transaction |
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
| `status` | text | `planning` · `awaiting_confirmation` · `building` · `ready` · `error` |
| `visibility` | text default `'private'` | `private` · `unlisted` · `public` (public = shown on home showcase) |
| `active_theme_id` | text null | id inside the project's theme pack |
| `theme_pack` | jsonb null | parsed palette list for the theme picker (§6) |
| `current_plan_message_id` | text null fk → messages | the confirmed `plan_card` message (Phase 06) |
| `snack` | jsonb null | `{snackId, url, savedAt, snapshotId}` of the last persistent Snack save (Phase 11) |
| `bundle_id`, `android_package` | text null | iOS bundle identifier / Android package. Derived from slug, **locked after the first store build** (Phase 15) |
| `eas_project_id` | text null | EAS project id on the user's Expo account (Phase 15) |
| `remixed_from_project_id` | text null | set when duplicated from a public project (Phase 17) |
| `template_version` | text | e.g. `expo-55@1` — which template the project was generated against |
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
| `kind` | text | `text` · `plan_card` · `build_summary` · `error` · `element_ref` |
| `content` | text | markdown text shown in chat |
| `payload` | jsonb | e.g. `AppPlan` for plan cards; `{afid, file, snippet}` for element refs |
| `attachments` | jsonb | asset ids |

> The **Anthropic message history** (with tool calls, thinking blocks, compaction blocks) is **not** the same as UI messages. We store the raw API transcript per run in `runs.transcript` (jsonb, append-only), so the agent can resume and keep thinking blocks valid. UI `messages` are a curated, user-facing projection.

## 4. Runs, jobs and events (Phase 05)

### `runs`
One row per agent/worker task the user can see progress for.
| column | type | notes |
|---|---|---|
| `id` | text pk | `run_…` |
| `project_id` | fk | |
| `kind` | text | `plan` · `plan_revision` · `build` · `edit` · `snack_sync` · `export_zip` · `export_github` · `build_native` · `publish_web` · `backend_provision` · `thumbnail` |
| `status` | text | `queued` · `running` · `succeeded` · `failed` · `canceled` |
| `input` | jsonb | job params |
| `output` | jsonb | result summary (urls, file list…) |
| `error` | jsonb null | `{code, message, retryable}` |
| `transcript` | jsonb | Anthropic message array for agent runs (append-only) |
| `credits_reserved` / `credits_charged` | bigint | micro-credits |
| `started_at`, `finished_at` | timestamptz | |
| `job_id` | text | pg-boss job id |

### `run_events`
Append-only stream that powers SSE.
| column | type | notes |
|---|---|---|
| `id` | bigserial pk | **monotonic** — used as SSE `id` / `Last-Event-ID` |
| `run_id` | fk | index (run_id, id) |
| `type` | text | see event catalogue in `05-ai-agent-design.md` §7 |
| `data` | jsonb | small payloads only (file paths + revision ids, not file contents) |

After each insert, call `NOTIFY run_events, '<run_id>'` so SSE handlers wake up without polling.

### pg-boss
pg-boss creates its own `pgboss` schema. Don't model it in Drizzle. Exclude it from `drizzle-kit push` with `schemaFilter: ["public"]`.

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
| `author` | text | `agent` · `user_inline` · `user_code` · `theme_switch` · `system` |
| `run_id` | fk null | |
| `parent_revision_id` | fk null | |

### `project_snapshots` (Phase 08, used by export/undo)
A point-in-time map `files: {path → revision_id}` (jsonb), plus `screens` (jsonb, the screen list at that time, so restore can bring back deleted screens), `label` (text, e.g. "After: add settings screen"), and `run_id`. Created at the end of every successful run, so "Restore previous version" is one row copy.

### `screens`
Denormalized list for the canvas and plan ↔ code linkage.
| `id` (`scr_…`) | `project_id` | `name` | `route` (e.g. `/(tabs)/index`) | `file_path` | `order` | `status` (`planned`/`building`/`ready`/`error`) | `canvas_x`, `canvas_y` |

## 6. Themes (Phase 08)

Themes are stored **in the project's files** (`theme/themes.ts`) because the generated app needs them. Also cache the parsed palette list in `projects.theme_pack` (jsonb) so the theme picker can render swatches without parsing TS.

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
| `project_id` pk | `provider` | `external_project_id` | `public_config` jsonb (URL + anon/public key — safe to ship in the app) | `secret_config_enc` (service keys — never shipped) | `schema_version` int | `status` |

### `project_github_repos` (Phase 14)
| `project_id` pk | `owner` | `repo` | `default_branch` | `last_pushed_snapshot_id` | `last_commit_sha` |

### `native_builds` (Phase 15)
| `id` | `project_id` | `platform` (`android`/`ios`) | `artifact` (`apk`/`aab`/`ipa`) | `version`, `build_number` | `eas_build_id` | `status` | `artifact_url` | `logs_url` | `error` |

### `project_signing` (Phase 15)
| `project_id` pk | `android_keystore_enc` (base64 jks, encrypted) | `android_keystore_password_enc` | `android_key_alias` | `android_key_password_enc` | `ios_asc_key_enc` (p8) | `ios_asc_key_id` | `ios_asc_issuer_id` | `apple_team_id` |
Back this up. Losing the keystore means you can never update the app on Google Play.

### `web_deployments` (Phase 16)
| `id` | `project_id` | `provider` (`eas_hosting`) | `url` | `alias` | `status` | `snapshot_id` | `log_tail` (last ~200 lines for the UI) |

## 8. Billing and credits (Phase 06 logs usage; Phase 18 enforces)

### `ai_usage`
One row per Anthropic API response. Written from Phase 06 on, even before billing exists.
| `id` | `run_id` | `user_id` | `model` | `input_tokens` | `output_tokens` | `cache_read_tokens` | `cache_write_tokens` | `cost_usd_micros` | `served_model` (if a fallback ran) |

### `credit_ledger`
Append-only. `credits_balance = sum(amount)`.
| `id` | `user_id` | `amount` bigint (±micro-credits) | `reason` (`signup_grant`, `monthly_grant`, `reserve`, `release`, `charge`, `purchase`, `refund`, `admin_adjust`) | `run_id` null | `stripe_event_id` null (unique — idempotency) |

### `subscriptions`
| `user_id` pk | `provider` (`stripe`) | `customer_id` | `subscription_id` | `plan` | `status` | `current_period_end` |

## 8b. Ops (Phase 19)

- `system_flags` (`key` pk, `value` jsonb, `updated_by`, `updated_at`): e.g. `generation_disabled`, read with a short cache.
- `worker_heartbeats` (`worker_id` pk, `host`, `version`, `last_seen_at`): powers the "worker down" alert and admin page.

## 9. Indexes to create from the start

- `projects (owner_id, deleted_at, updated_at desc)` — home grid
- `projects (visibility, updated_at desc) where visibility = 'public'` — showcase
- `run_events (run_id, id)`
- `runs (project_id, created_at desc)`
- `file_revisions (project_id, path, created_at desc)`
- `messages (project_id, created_at)`
- `credit_ledger (user_id, created_at)`; unique `(stripe_event_id)` where not null

## 10. Generated-app databases are separate

The tables above belong to **the platform**. Each **generated app's** data (habits, expenses …) lives in the backend the user connects in Phase 13. That backend gets its own per-entity tables, created by agent-written migrations (see `phases/phase-13-generated-app-backend.md`). Never store generated-app user data in the platform DB, and never as a single JSON blob table. The video had to fix exactly that.
