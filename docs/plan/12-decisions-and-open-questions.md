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
| D7 | **Expo SDK version** for generated apps | newest SDK supported by the published `snack-sdk` and by store Expo Go | **SDK 54** unless `snack-sdk` now supports newer (the reference app is pinned to 54 for this reason). Re-check at Phase 07 start. | Phase 07 |
| D8 | **Preview runtime fallback** if the custom loader spike fails | Sandpack · Snack web player | Decide only if the spike fails its exit criteria (`07` §9) | Phase 07 |
| D9 | **Models** (`AI_MODEL` Planner, `AI_BUILDER_MODEL` generators) | `claude-opus-5-5` · `claude-sonnet-5-5` (what the reference app runs for all agents, ≈ half the cost) | **Opus 5.5 for both** to start; switch `AI_BUILDER_MODEL` (and/or `AI_MODEL`) to Sonnet 5.5 only if the eval suite shows equal quality (`10`, `11` §4) | Phase 08 |
| D10 | **Generated-app backend provider** (first adapter) | Supabase (public OAuth-for-integrations + Management API, RLS) · CloudHike (as video) · Firebase · own multi-tenant API | **Supabase** first (widely documented, Lovable-proven pattern), CloudHike as second adapter if you want parity with the video | Phase 13 |
| D11 | **AI images provider** | ImageKit (video) · image model + our storage · stock photos | ImageKit | Phase 12 |
| D12 | **GitHub integration type** | GitHub App · OAuth App with `repo` scope (video) | **GitHub App** | Phase 14 |
| D13 | **iOS builds in v1?** | Yes (needs the user's Apple Developer account + ASC API key) · Android + web only at launch | Ship **Android + web first**; iOS behind a "beta" label | Phase 15 |
| D14 | **Web hosting for published apps** | EAS Hosting on the user's Expo account · our own static hosting under `*.apps.<brand>` | EAS Hosting (the user owns it; we don't host their content) | Phase 16 |
| D15 | **Payments provider** | Stripe · Razorpay (India) · Paddle/Lemon Squeezy (merchant of record) | Stripe; consider a merchant of record if you need sales-tax handling | Phase 18 |
| D16 | **Plan limits and prices** | see `11` §4 | Reference-app limits: Free 100 one-time credits, 2 projects, 5 screens/project, web publish but no native builds. Pro 1,000 monthly credits, unlimited projects, 10 screens/project, native builds. **Pro price to decide** ($49–99 on Opus, $25–49 on Sonnet per `11` §4) | Phase 18 |
| D18 | **Credit model** | flat per action (reference app: 10/screen, 5/change round, 1/image) · cost-based (`CREDIT_MARKUP`/`CREDIT_USD_VALUE`) | **Flat**, plus a per-round file cap (`11` §5); keep cost-based as a fallback config | Phase 18 |
| D17 | **Public showcase moderation** | auto-publish · admin approval | Admin approval queue for "show on homepage" | Phase 17 |

## 2. Open questions (need research at implementation time)

- **Q1.** Does `snack-sdk` expose connected-device presence for the "iPhone connected" indicator shown in the video? If not, drop the indicator. (Phase 11)
- **Q2.** Exact ImageKit generation URL syntax, limits and per-image price. (Phase 12)
- **Q3.** The chosen backend provider's OAuth scopes and how to provision a project, apply SQL, and read keys via API. Is there a sandbox for testing? (Phase 13)
- **Q4.** EAS non-interactive credentials: confirm `credentials.json` local Android credentials and the iOS ASC API key env vars work with `--non-interactive`. Confirm the EAS webhook signature header and algorithm. (Phase 15)
- **Q5.** EAS Hosting: per-account limits, custom domains, and whether `eas deploy` can run fully non-interactively with an access token. (Phase 16)
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

