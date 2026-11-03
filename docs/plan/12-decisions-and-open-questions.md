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
| D7 | **Expo SDK version** for generated apps | newest SDK supported by both Snack and store Expo Go | Check both lists at Phase 07 start; record here | Phase 07 |
| D8 | **Preview runtime fallback** if the custom loader spike fails | Sandpack · Snack web player | Decide only if the spike fails its exit criteria (`07` §9) | Phase 07 |
| D9 | **Model per stage** | `claude-opus-5-5` everywhere · cheaper models on some stages | **Opus 5.5 everywhere**; consider `AI_MODEL_<STAGE>` overrides only with eval evidence (`11` §5) | Phase 08 |
| D10 | **Generated-app backend provider** (first adapter) | Supabase (public OAuth-for-integrations + Management API, RLS) · CloudHike (as video) · Firebase · own multi-tenant API | **Supabase** first (widely documented, Lovable-proven pattern), CloudHike as second adapter if you want parity with the video | Phase 13 |
| D11 | **AI images provider** | ImageKit (video) · image model + our storage · stock photos | ImageKit | Phase 12 |
| D12 | **GitHub integration type** | GitHub App · OAuth App with `repo` scope (video) | **GitHub App** | Phase 14 |
| D13 | **iOS builds in v1?** | Yes (needs the user's Apple Developer account + ASC API key) · Android + web only at launch | Ship **Android + web first**; iOS behind a "beta" label | Phase 15 |
| D14 | **Web hosting for published apps** | EAS Hosting on the user's Expo account · our own static hosting under `*.apps.<brand>` | EAS Hosting (the user owns it; we don't host their content) | Phase 16 |
| D15 | **Payments provider** | Stripe · Razorpay (India) · Paddle/Lemon Squeezy (merchant of record) | Stripe; consider a merchant of record if you need sales-tax handling | Phase 18 |
| D16 | **Plan limits and prices** | see `11` §4 | Free: 60 credits (~1 app + 3 edits), 2 projects, 5 screens/project, no native builds. Pro: $49/mo, 400 credits, unlimited projects, native builds. Revisit after measurement. | Phase 18 |
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
