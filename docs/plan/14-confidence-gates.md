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
| A2 | Expo Go on phones runs generated apps with live updates via a **hosted Expo dev server** (Path B: `expo start` per session in the sandbox, Fast Refresh on file writes) | 🟡 | Standard Expo CLI workflow; the exact public-reachability mechanism (`--tunnel` vs reverse proxy + `EXPO_PACKAGER_PROXY_URL`) isn't run by us yet | S3 | Path C: web preview in the phone browser; Path A (Snack) once it supports the SDK |
| A3 | The template SDK matches store Expo Go | ✅ (decided) | Store Expo Go 58.0.5 supports only SDK 58; we pin 58 (`15` §1, ADR-002). The reference app's SDK 54 + Snack path doesn't open in today's store Expo Go (published `snack-sdk` ≤ 54) | S0 re-check + S3 | Re-pin when Expo ships SDK 59 (`06` §11); Path C meanwhile |
| A4 | Planner + single-purpose generators produce apps that compile and render ≥ 90% of the time after 1 repair | ✅ pattern / 🟡 rate | Reference app ships it; the rate is unmeasured for our prompts | S2, then Phase 08 evals | More exemplars in prompts; a typecheck-feedback repair; higher effort; Opus vs Sonnet (D9) |
| A5 | Prompt caching: APP CONTEXT shared across screens gives cache hits | 🟡 | Anthropic docs; our layout (`05` §7) | S2 (check `cache_read_input_tokens > 0`) | Reorder blocks; costs about 2× without it (`11`) |
| A6 | Planner state as a mid-conversation system message on Opus 5.5, **placed after the latest user turn as the last entry**, stored and replayed append-only | ✅ API support / 🟡 our usage | Verified in Anthropic docs, including the placement rule (`15` §7) | S2 (one real call + an append-only replay test) | `<project-state>` block at the end of the latest user turn |
| A7 | Babel `line:col` stamps + one element per line give reliable source mapping | 🔴 (improves on ✅ `file:line:type`) | Reference app used line-only | S1 | Fall back to `file:line:type`, as in the reference |
| A8 | pg-boss + `LISTEN/NOTIFY` SSE works on the chosen hosts | 🟡 | Standard Postgres; needs a direct connection (`04`) | S0 (host check) + Phase 05 | Polling fallback (Phase 05) |
| A9 | Long SSE on Vercel works via reconnect + `Last-Event-ID` | 🟡 | Design (`02` A3) | Phase 05 demo job | Serve SSE from the worker host instead |
| A10 | EAS builds an Android APK/AAB non-interactively from an uploaded project | ✅ | Reference app (`13` R18) + eas-cli 27.3.0 source: generates a keystore non-interactively; `EAS_NO_VCS=1` avoids the git prompt (`15` §2) | S4 | Keystore via our `keytool` + `credentials.json` (`credentialsSource: "local"`) |
| A11 | iOS IPA builds for owners with an Apple Developer account | 🟡 | Verified: ASC API key env vars create profiles non-interactively, but a **missing distribution certificate fails** (`15` §2) | S4 (if an Apple account is available) | Guided one-time certificate setup; ship Android + web first (D13) |
| A12 | EAS webhooks deliver build status with a verifiable signature | ✅ | Verified: per-project webhooks, `expo-signature` HMAC-SHA1 `sha1=<hex>` (`15` §3). Payloads treated as hints and re-fetched (ADR-003 #17) | S4 | Poll-on-view like the reference app |
| A13 | EAS Hosting deploys non-interactively with the owner's token; the URL is stable across publishes | ✅ | Reference app (`13` R19) + docs: `--dev-domain` required non-interactively (`15` §4) | S5 | Our own static hosting (D14) |
| A14 | ImageKit generation URLs work and can be awaited server-side | ✅ | Reference app + docs: `is-intermediate-response: true` until ready; cached by path; ≈ $0.12/image (`15` §5) | S7 | Placeholders (`IMAGEKIT_ENABLED=false`; always on Free) |
| A15 | Backend provider: OAuth connect → create project → apply derived DDL → app reads/writes with per-user security | ✅ CloudHike (reference, gateway) · 🟡 Supabase (APIs verified in docs, `15` §6; not run) | Direct mode always deploys auth + anonymous sign-in (ADR-003 #19) | S6 (incl. two-install privacy check) | Gateway mode (works with any Postgres); switch provider (D10) |
| A16 | GitHub push via App/OAuth + Git Data API, branch-per-push | ✅ | Reference app (OAuth) + GitHub docs | Phase 14 | OAuth App with minimal scopes |
| A17 | Stripe subscription + two-balance credits are correct under out-of-order webhooks | ✅ pattern | Reference app (`13` R15–R16) | Phase 18 tests (Stripe CLI fixtures) | — |
| A18 | A small app costs ≈ $1.20 in model fees on Opus 5.5 | 🔴 | Estimate (`11` §2) | S2 measures it | Adjust prices or effort, or switch builder model (D9, D16) |
| A19 | Free-tier abuse stays manageable | 🟡 | Rate limits + verification (`08` §7) | Pre-launch review | Require email verification; lower free credits |
| A20 | Hosting user-generated apps and showcase content is legally OK with ToS | 🟡 | Q6 | Pre-launch review | Disable the public showcase; EAS Hosting stays on users' own accounts |
| A21 | A **sandbox service** with no secrets and no DB access can run Playwright renders, `expo export` and Path B dev servers, reached only via per-job tokens and signed URLs | 🟡 | Our design (`08` §5, ADR-003 #18); standard services on Railway/Fly/Render | S1/S3/S5 run through a local sandbox; Phase 08 smoke test | If egress can't be restricted on the host: rely on no-secrets + private networking; move to per-job VMs (e.g. Fly Machines) later |
| A22 | Path B dev servers stay affordable: ~4 concurrent sessions per sandbox instance, 15-min idle stop | 🔴 | Estimate; Metro memory per session unmeasured | S3 measures memory/CPU per session | Fewer concurrent sessions per instance; Pro-only device preview; Path C for Free |

**Rule:** a phase may not start while an assumption it depends on is 🔴. Each spike's result is recorded as an ADR in `12-decisions-and-open-questions.md`, and the row's status is updated here.

## 2. Gates

| Gate | When | Pass criteria (evidence in the PR / ADR) | If it fails |
|---|---|---|---|
| **G0 Architecture go/no-go** | End of Phase 00b | S0–S7 pass or have an accepted fallback; A1–A15, A21, A22 are no longer 🔴 | Re-plan the affected phases before writing feature code |
| **G1 Shell** (M1) | After Phase 04 | Manual QA steps 1, 10 (`09` §6); authz tests green | — |
| **G2 Generation quality** (M2) | After Phase 08 | `10` §2.2 held-out **counts** met across 2 seeds (e.g. build completes ≥ 27/30, renders ≥ 27/30, edit success ≥ 21/24); measured p50 cost per app within 1.5× of `11` §2; latency within the S2-based target | Prompt iteration (time-box 3 days), then D9 or re-architecture review |
| **G3 Editing + device** (M3) | After Phase 11 | Full manual QA script (`09` §6) passes on a real iPhone and a real Android phone with **store** Expo Go (Path B) | Fix before continuing |
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
