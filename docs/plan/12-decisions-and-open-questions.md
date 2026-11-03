# 12 — Decisions & Open Questions

Each open decision lists the recommended default, so work can proceed if you don't decide. Change a default by editing this file, and the affected phase docs, in the same PR.

---

## 1. Decisions to make (in phase order)

| # | Decision | Options | Recommended default | Decide by |
|---|---|---|---|---|
| D1 | **Product name / brand** | anything but the video's "AppForge" | Working name "AppBuilding" until chosen | Phase 01 (UI copy) |
| D2 | **Platform infra vendor** (DB, storage, worker host) | CloudHike (as video) · Railway (Postgres + buckets + worker in one project) · Neon + R2 + Fly · Supabase | **Railway** for DB + bucket + worker, **Vercel** for web. Everything is behind `DATABASE_URL` / S3 env vars, so it's reversible. | Phase 00 |
| D3 | **Id strategy** | prefixed text ids (`prj_…`) · uuid | Prefixed text ids (readable logs) | Phase 00 |
| D4 | **Use a boilerplate?** | create-next-app + manual setup · a Next.js boilerplate like the video's | create-next-app (fewer unknown files) | Phase 00 |
| D5 | **Code reviewer** | CodeRabbit (video) · Claude Code `/code-review` in CI · none | CodeRabbit on PRs, plus CI checks required | Phase 00 |
| D6 | **Job system** | pg-boss + own worker · Inngest · Trigger.dev · CloudHike Jobs | **pg-boss + worker** (no extra vendor, no tunnel) | Phase 05 |
| D7 | **Expo SDK version** for generated apps | newest SDK supported by store Expo Go · an older SDK that Snack supports | **SDK 58** (decided, ADR-002): store Expo Go supports only 58, and published `snack-sdk` stops at 54. Phone preview uses a hosted Expo dev server (Path B); Snack only once it supports the template SDK. Re-check at Phase 00b S0. | Phase 00b |
| D8 | **Preview runtime fallback** if the custom loader spike fails | Sandpack · Snack web player | Decide only if the spike fails its exit criteria (`07` §9) | Phase 07 |
| D9 | **Models** (`AI_MODEL` Planner, `AI_BUILDER_MODEL` generators) | `claude-opus-5-5` · `claude-sonnet-5-5` (what the reference app runs for all agents, ≈ half the cost) | **Opus 5.5 for both** to start; S2 measures both. Switch `AI_BUILDER_MODEL` (and/or `AI_MODEL`) to Sonnet 5.5 only if the held-out eval counts hold (`10` §2.2, `11` §4). Note that mid-conversation system messages aren't available on Sonnet 5 (fine on 5.5) | Phase 00b → 08 |
| D10 | **Generated-app backend provider** (first adapter) | Supabase (public OAuth-for-integrations + Management API, RLS) · CloudHike (as video) · Firebase · own multi-tenant API | **Supabase** first (widely documented, Lovable-proven pattern), CloudHike as second adapter if you want parity with the video | Phase 13 |
| D11 | **AI images provider** | ImageKit (video) · image model + our storage · stock photos | ImageKit | Phase 12 |
| D12 | **GitHub integration type** | GitHub App · OAuth App with `repo` scope (video) | **GitHub App** | Phase 14 |
| D13 | **iOS builds in v1?** | Yes (needs the owner's Apple Developer account, an ASC API key, and a distribution certificate created once, interactively or uploaded) · Android + web only at launch | Ship **Android + web first**; iOS behind a "beta" label with a guided one-time certificate setup (`15` §2) | Phase 15 |
| D14 | **Web hosting for published apps** | EAS Hosting on the user's Expo account · our own static hosting under `*.apps.<brand>` | EAS Hosting (the user owns it; we don't host their content) | Phase 16 |
| D15 | **Payments provider** | Stripe · Razorpay (India) · Paddle/Lemon Squeezy (merchant of record) | Stripe; consider a merchant of record if you need sales-tax handling | Phase 18 |
| D16 | **Plan limits and prices** | see `11` §4 | Reference-app limits: Free 100 one-time credits (on email verification), 2 projects, 5 screens/project, placeholder images, web publish but no native builds. Pro 1,000 monthly credits, unlimited projects, 10 screens/project, AI images (4 credits each), native builds. **Pro price to decide** ($49–99 on Opus, $25–49 on Sonnet per `11` §4) | Phase 18 |
| D18 | **Credit model** | flat per action (reference app: 10/screen, 5/change round, 1/image) · cost-based (`CREDIT_MARKUP`/`CREDIT_USD_VALUE`) | **Flat**: 10/screen, 5 per change round for ≤ 2 files (+3/extra file), **4 per AI image** (ADR-002/003); cost-based stays a fallback config | Phase 18 |
| D17 | **Public showcase moderation** | auto-publish · admin approval | **Admin approval** for "show on homepage" (`projects.showcase_status`); a public link (`visibility = public`) works immediately | Phase 17 |

## 2. Open questions (need research at implementation time)

- **Q1. Answered (`15` §1):** `snack-sdk` exposes `state.connectedClients` (name, platform, status), so the indicator is possible on Path A. For Path B, the dev server's connected-client info is checked in S3; show "connected" only if available.
- **Q2. Answered (`15` §5):** `{endpoint}/ik-genimg-prompt-{percent-encoded}/{path}/{file}`; the intermediate response has `is-intermediate-response: true`; cached by path; 65 units per image (≈ $0.12 at overage). Prompt length limit unknown (cap 200 chars).
- **Q3. Answered for Supabase (`15` §6):** OAuth apps + Management API (create project, SQL query endpoint (beta), API keys with publishable/secret types). Store `db_pass`. Still open: CloudHike equivalents, if CloudHike is picked as an adapter.
- **Q4. Answered (`15` §2–§3):** Android: EAS generates a keystore non-interactively, or use `credentials.json` + `credentialsSource: "local"`. iOS: ASC key env vars work, but a missing distribution certificate fails non-interactively. Webhooks: `expo-signature`, HMAC-SHA1, `sha1=<hex>`, per project. Use `EAS_NO_VCS=1`; set `owner`.
- **Q5. Answered (`15` §4):** `eas deploy --prod --dev-domain <slug> --non-interactive --json`; free plan 100k requests/mo, 1 GB, 5 aliases, no custom domain; auth via `EXPO_TOKEN` (standard, 🟡).
- **Q6.** Legal: ToS for hosted user content, the privacy policy (prompts/code go to the model provider under our API terms), and data retention for prompts and uploads.

## 3. Decision log (ADR-lite)

Append one entry per decision taken. Never rewrite old entries; supersede them.

```
### ADR-001 — <title>   (YYYY-MM-DD, Phase NN)
Decision: …
Why: …
Alternatives rejected: …
Consequences / follow-ups: …
```

### ADR-000 — Plan baseline (2026-11-03)
Decision: Adopt the architecture in `02-architecture.md`: queue + worker from Phase 05, a fixed Expo template, our own browser preview runtime with Snack only for devices, the Anthropic SDK directly, and vendor adapters everywhere.
Why: Avoids the rework and breakages seen in the source video (`01-video-review.md` §3).
Alternatives rejected: follow the video's vendor stack one-to-one.
Consequences: Phase 05 and Phase 07 are new phases that come before generation; total effort is higher up front but later phases are simpler.

### ADR-001 — Adopt the reference implementation's agent and app architecture (2026-11-03)
Decision: Replace the single tool-looping Builder/Editor with a conversational Planner (tools `propose_plan`, `confirm_plan`, `update_app`, `set_theme`, `setup_backend`) plus single-purpose generators (Foundation, Data, Component Designer/Builder, Screen Builder) with one code-driven repair. Generated apps use collections-as-data with generic `@/lib/backend` hooks, a per-app shared component library, a platform-generated tab layout, and `colors` tokens (CSS variables in preview). App tables are derived from collections by platform code. Flat credit pricing.
Why: The shipped reference product (`13-reference-implementation.md`) proves these patterns. They're cheaper per app (≈ $1.20 vs ≈ $1.60 for a small app), simpler to validate, and remove agent-written plumbing and SQL.
Alternatives rejected: keep the v1 tool loop (more flexible, but costlier and harder to make reliable).
Consequences: Phase 06/08/09/10/12/13/18 docs rewritten accordingly. Our own choices stay where they're stronger: the Anthropic SDK directly, pg-boss worker, structured outputs, caching layout.


### ADR-002 — Vendor facts verified (2026-11-03; details in `15-vendor-facts.md`)
Decision:
- **D7 = Expo SDK 58** for the template (store Expo Go supports only 58; the published `snack-sdk` supports ≤ 54, and `main` ≤ 56).
- **Phone preview defaults to Path B, a hosted Expo dev server**: `expo start` per active session in the sandbox service, reachable by Expo Go, with Fast Refresh on file writes. S3 verifies the exact connection mechanism; candidates are `--tunnel`, or a reverse proxy per session with `EXPO_PACKAGER_PROXY_URL`. **Path A (Snack)** is used only when a published `snack-sdk` supports the template SDK. **Path C (fallback):** open the web preview URL on the phone.
- EAS: let EAS generate the Android keystore non-interactively (keytool + local credentials as fallback). Use `EAS_NO_VCS=1` instead of `git init`. iOS needs a distribution certificate created once (interactive or uploaded) plus an ASC API key. Set `owner` for project linking.
- EAS Hosting: pass `--dev-domain <slug>`. Set `web.output: "single"` explicitly (SDK 58 defaults to `server`).
- ImageKit: handle `is-intermediate-response: true`; regenerate by changing the filename; cap prompts at 200 chars. A generated image costs us ≈ $0.12, so **AI images cost 4 credits**. Free plans use placeholder images. Pro builds may include up to 8 AI images, charged 4 credits each and shown in the build estimate. Production needs a paid ImageKit plan.
- Supabase: publishable/secret keys (not anon/service_role); store `db_pass` encrypted; the SQL endpoint is beta.
- Anthropic: the PROJECT STATE system message goes **after** the latest user message, as the last entry.
Why: verified primary sources (`15`).
Consequences: Phase 11 gains Path B (+1–2 d). `06` §1/§10, `11` §3, Phases 12/13/15/16 updated.

### ADR-003 — Fixes from the independent buildability review (2026-11-03)
Decisions (each closes a review finding; phase docs follow these):
1. **Planner history is append-only.** Each turn's PROJECT STATE message is stored in `messages.payload.api` with that turn and replayed byte-for-byte, never rebuilt or removed. No history rewriting of any kind (no image-placeholder swaps). The **full Planner tool set (all 5 tools) is defined from Phase 06**; not-yet-built tools return "not available yet" results, so the tool list never changes mid-session. Each session is stamped with `prompt_version`; a deploy that changes the prompts or tools inserts a `session_break` automatically. Render progress-update `thinking` blocks, since pre-tool prose may arrive there.
2. **Two worker wrappers.** Single-shot jobs finish/fail the run. **Step jobs** never finish the run (only the final `advance(null)` does) and call `failRun` only on the final attempt or a non-retryable error. The stuck-run sweeper uses a per-step heartbeat (`runs.heartbeat_at`) and skips kinds waiting on external systems. On a retried `agent.chat`, emit `assistant.reset` before re-streaming.
3. **Actor on every run.** `runs.created_by` (acting user) and `runs.billed_user_id` (project owner, who pays). Job payloads are `{runId, projectId, userId}`. Native builds and web publish are **owner-only** in v1, using the owner's Expo connection (one `eas_project_id` per project).
4. **One theme source:** `projects.plan.themePackage {themes, activeThemeId}`. Drop the `projects.active_theme_id`/`theme_pack` columns and the `theme_switch` author. A theme switch emits `plan.updated`.
5. **Project-scoped event stream.** `run_events` gets `project_id NOT NULL` and a nullable `run_id`. The primary SSE is `/api/projects/:id/events` (open while the workspace is open; `/api/runs/:id/events` filters it). Non-run writes (inline edits, code saves, theme switches, restores) emit `file.written {author}` / `plan.updated` on it.
6. **Schemas built verbatim from `03`** in Phases 02/05: `credit_ledger` with `balance` + unique `reference` (signup reference `signup:<userId>`); `monthly_credits_balance`; `runs.steps`, `credits_charged`, `credits_refunded`, `step`, `lease_id`, `heartbeat_at`, `created_by`, `billed_user_id`; a `processed_webhook_events(provider, event_id) pk` table. Phase 03 uses `runs.kind = "chat"`.
7. **Pricing mechanics:**
   - A change round costs 5 credits for ≤ 2 regenerated files, +3 per extra file. The Planner is told the price.
   - **Resume** = same run, automatic, never charged again. **Retry** = a new run after a refund, charging what's unbuilt.
   - Charge flow: generate the runId → in one transaction: conditional balance `UPDATE`, ledger `charge:<runId>`, insert the run with `credits_charged`, insert a **`job_outbox`** row. A dispatcher in the worker sends outbox rows to pg-boss. This is a transactional outbox, so no cross-connection pg-boss transaction is needed.
8. **MVP web publish** includes the Expo connect, EAS init and identity steps (Phase 15 steps 1–2 move into Phase 16's MVP scope; +1–1.5 d).
9. **Template versioning rule:** template revisions within one SDK render from HEAD for every project and must stay backward compatible. Only SDK changes are versioned. The upgrade machinery (`upgradeProject`, banner, `template_upgrade` runs) is built in **Phase 19b**, before the first SDK bump.
10. **S1 spike code graduates** into `preview-runtime/` (Phase 07 hardens and tests it). One fixture: sign-in + 2 tab screens + 1 stack detail screen, 4 components, 2 collections. S2 may hand-write renderers and use syntax+import validation only.
11. **Build-time target** comes from S2's measured p50/p90 (ADR at G0). After screen 1, concurrency 2 is allowed.
12. **Evals:** metrics rewritten for the generator pipeline. The eval set is 30 prompts with a **15-prompt held-out set**; targets are stated as counts across 2 seeds.
13. **`update_app` semantics:** plan-guard runs at tool time on the resulting plan. Screens linking to a removed id must be in `editScreens`; tabs stay 3–5 (or the app switches to stack); `firstScreenId` can only be removed if replaced. The target plan is applied at tool time with per-screen status. A failed new screen is marked `error` (retry from canvas, credits refunded), and a failed edit keeps the old code.
14. **Two file-write routes:** `POST /api/projects/:id/files/edits` (inline edit ops, Phase 10) and `PUT /api/projects/:id/files/content` (whole-file saves, Phase 10b). Both take `baseRevisionId`, and 409 means the base revision isn't current. The lock is based on `runs.step` + run status.
15. **Runtime CSP** adds `font-src <runtime origin> data:` (Ionicons).
16. **GitHub first push to an empty repo** uses the Contents API for the first commit, or creates the repo with `auto_init`, then the Git Data API.
17. **EAS webhooks per EAS project** with a per-project secret (encrypted). Payloads are hints: re-fetch the build with the owner's token before writing. A `build_native` run ends at submit, and status lives in `native_builds`. The global `EAS_WEBHOOK_SECRET` env var is dropped.
18. **Sandbox service:** a separate service with **no secrets and no DB access** runs untrusted-code steps (`expo export`, Playwright renders/thumbnails, the Path B dev servers). It receives work from the worker over HTTP with signed storage URLs and a per-job token. It runs as non-root with an env allowlist and restricted egress where the host allows.
19. **Direct + RLS backends** always deploy auth and use anonymous sign-in (`owner_id = auth.uid()`); otherwise gateway mode.
20. **Phase 19 splits** into 19a (deploy, CI, observability; 3–4 d) and 19b (data lifecycle, template-upgrade machinery, vendor watch, G4 evidence; 5–7 d).
21. **pg-boss** is pinned to a major version (≥ 10): `createQueue()` per queue, policies explicit.
22. **Signup credits are granted on email verification** (social sign-ins count as verified). Phase 02 adds an email sender. `signup_grants(email_hash pk)` prevents repeat grants after account deletion.
23. **Roles:** `08` §2's matrix is authoritative. Rename = editor; builds and publish = owner. The homepage showcase needs admin approval (D17); a public link is immediate.
24. Naming: `lib/backend` (not `lib/data`); `lib/render/*`; `lib/ai/prompts/state.ts`; `tests/e2e/`; the `screens` table is created in Phase 04 only; image MIME allowlist png/jpeg/webp/gif; `DAILY_SPEND_CAP_USD` env var.
