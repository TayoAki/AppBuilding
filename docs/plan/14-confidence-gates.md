# 14 — Confidence: Assumptions, Gates, Evidence

Building with confidence doesn't come from a longer plan. It comes from three habits:
1. **Name every assumption** the plan rests on, and say how it's known (reference app, vendor docs, or not yet).
2. **Test the risky ones first**, in throwaway spikes before feature work (Phase 00b). Each has pass criteria and a written fallback.
3. **Demand evidence at every gate**: tests, checklists, screenshots and eval numbers in each PR, not "it works on my machine".

If every assumption below is green before the phase that depends on it starts, the remaining risk is ordinary engineering effort, not unknowns.

---

## 1. Assumption register

Status: **✅ proven** (shipped in the reference app or verified in official docs) · **🟡 likely** (sound, documented, but not yet run by us) · **🔴 unproven** (our own design choice, untested anywhere).

| # | Assumption | Status | Evidence / source | Proven by (gate) | Fallback if it fails |
|---|---|---|---|---|---|
| A1 | Generated Expo code can run in the browser on react-native-web inside a sandboxed iframe, with element selection | ✅ | Reference app's `preview-runtime/` + inspector (`13` R8–R9) | Spike S1 | Sandpack + router shim; Snack web player for the interactive preview only |
| A2 | Snack + Expo Go run generated apps (SDK 54, React Navigation shim, relative imports) with live updates | ✅ | Reference app (`13` R11) | S3 | "Open in browser" fallback link |
| A3 | Store Expo Go still opens the pinned SDK on the day we build | 🟡 | Changes with Expo releases | S0 + S3 | Pin to the SDK both Snack and Expo Go support; else ship web fallback first and re-pin when `snack-sdk` updates |
| A4 | Planner + single-purpose generators produce apps that compile and render ≥ 90% of the time after 1 repair | ✅ pattern / 🟡 rate | Reference app ships it; the rate is unmeasured for our prompts | S2, then Phase 08 evals | More exemplars in prompts; a typecheck-feedback repair; higher effort; Opus vs Sonnet (D9) |
| A5 | Prompt caching: APP CONTEXT shared across screens gives cache hits | 🟡 | Anthropic docs; our layout (`05` §7) | S2 (check `cache_read_input_tokens > 0`) | Reorder blocks; costs about 2× without it (`11`) |
| A6 | Planner state as a mid-conversation system message on Opus 5.5 | 🟡 | Anthropic docs (confirming via S0) | S2 | `<project-state>` block in the user turn (the reference app's approach rebuilds the system prompt, which also works but isn't cached) |
| A7 | Babel `line:col` stamps + one element per line give reliable source mapping | 🔴 (improves on ✅ `file:line:type`) | Reference app used line-only | S1 | Fall back to `file:line:type`, as in the reference |
| A8 | pg-boss + `LISTEN/NOTIFY` SSE works on the chosen hosts | 🟡 | Standard Postgres; needs a direct connection (`04`) | S0 (host check) + Phase 05 | Polling fallback (Phase 05) |
| A9 | Long SSE on Vercel works via reconnect + `Last-Event-ID` | 🟡 | Design (`02` A3) | Phase 05 demo job | Serve SSE from the worker host instead |
| A10 | EAS builds an Android APK/AAB non-interactively from an uploaded project | ✅ | Reference app (`13` R18) + EAS docs | S4 | Keystore via our `keytool` + `credentials.json` (Phase 15) |
| A11 | iOS IPA builds for users with an Apple Developer account | 🟡 | EAS docs; reference app shows setup steps instead of building | S4 (if an Apple account is available) | Ship Android + web first (D13) |
| A12 | EAS webhooks deliver build status with a verifiable signature | 🟡 | EAS docs (S0 confirms the header and algorithm) | S4 | Poll-on-view like the reference app |
| A13 | EAS Hosting deploys non-interactively with the user's token; the URL is stable across publishes | ✅ | Reference app (`13` R19) | S5 | Our own static hosting (D14) |
| A14 | ImageKit generation URLs work and can be awaited server-side | ✅ | Reference app (`13`, Phase 12) | S7 | Placeholders (`IMAGEKIT_ENABLED=false`) |
| A15 | Backend provider: OAuth connect → create project → apply derived DDL → app reads/writes with per-user security | ✅ for CloudHike (reference) · 🔴 for Supabase | Reference app (CloudHike + gateway); Supabase docs (S0) | S6 | Gateway mode (works with any Postgres); switch provider (D10) |
| A16 | GitHub push via App/OAuth + Git Data API, branch-per-push | ✅ | Reference app (OAuth) + GitHub docs | Phase 14 | OAuth App with minimal scopes |
| A17 | Stripe subscription + two-balance credits are correct under out-of-order webhooks | ✅ pattern | Reference app (`13` R15–R16) | Phase 18 tests (Stripe CLI fixtures) | — |
| A18 | A small app costs ≈ $1.20 in model fees on Opus 5.5 | 🔴 | Estimate (`11` §2) | S2 measures it | Adjust prices or effort, or switch builder model (D9, D16) |
| A19 | Free-tier abuse stays manageable | 🟡 | Rate limits + verification (`08` §7) | Pre-launch review | Require email verification; lower free credits |
| A20 | Hosting user-generated apps and showcase content is legally OK with ToS | 🟡 | Q6 | Pre-launch review | Disable the public showcase; EAS Hosting stays on users' own accounts |

**Rule:** a phase may not start while an assumption it depends on is 🔴. Each spike's result is recorded as an ADR in `12-decisions-and-open-questions.md`, and the row's status is updated here.

## 2. Gates

| Gate | When | Pass criteria (evidence in the PR / ADR) | If it fails |
|---|---|---|---|
| **G0 Architecture go/no-go** | End of Phase 00b | S0–S7 pass or have an accepted fallback; A1–A15 are no longer 🔴 | Re-plan the affected phases before writing feature code |
| **G1 Shell** (M1) | After Phase 04 | Manual QA steps 1, 10 (`09` §6); authz tests green | — |
| **G2 Generation quality** (M2) | After Phase 08 | Eval targets (`10` §2.2) met on the held-out set; measured cost per app within 1.5× of `11` §2 | Prompt iteration (time-box 3 days), then D9 or re-architecture review |
| **G3 Editing + device** (M3) | After Phase 11 | Full manual QA script (`09` §6) passes on a real iPhone and a real Android phone | Fix before continuing |
| **G4 MVP launch** | After the MVP track (`00-index`) | Security checklist (`08` §10) incl. two-account pentest; load test (20 concurrent builds); backups and restore tested; cost alerts on; ToS/privacy live | No launch |
| **G5 Full launch** | After all phases | G4 again, plus native builds and backend connect verified end to end on fresh accounts | — |

## 3. Evidence required in every phase PR

- The phase's acceptance checklist, each item ticked **with how it was verified** (test name, screenshot, recording, log excerpt).
- `lint`, `typecheck`, `test` (and `test:e2e` from Phase 04) green in CI.
- From Phase 08 on, for prompt or model changes: eval before/after numbers (`10` §2.4).
- From Phase 05 on: no route or job without an authz test.
- Docs updated: the phase doc and any reference doc that changed, plus this register.

## 4. Things no plan can make certain, and how we watch them

| Risk | Watch | Response |
|---|---|---|
| Vendors change (Expo SDK cadence, `snack-sdk`, EAS, ImageKit, provider APIs) | Monthly check of release notes; nightly smoke (Snack load, EAS dry run, image fetch) from Phase 19 | Template version bump (`06` §11); keep fallbacks wired |
| Model behaviour drift or deprecation | Nightly eval on `main` (`10`); `ai_usage` cost dashboard | Pin the model id in env; re-tune prompts with evals before switching |
| Cost overruns | Alerts on daily spend and on cost per credit by action (`11` §5) | Adjust prices or effort; per-round file cap |
| Security regressions | `security-review` on sensitive PRs; quarterly two-account pentest | Fix forward; incident runbook (Phase 19) |
