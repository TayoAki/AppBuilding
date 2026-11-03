# Phase 00b — De-risking Spikes (before any feature work)

| | |
|---|---|
| Video chapter(s) | New (not in video) |
| Branch | `spike/00b-<name>` per spike. Spike code lives in `spikes/` and is **never merged into app code**; findings and reusable assets (fixture app, prompt drafts) are merged |
| Depends on | Phase 00 |
| Effort (solo dev + Claude Code) | ~7 days total (the Phase 07 runtime spike moves here, so Phase 07 shrinks by ~3 days) |
| New env vars | none (spikes use local `.env.spike`) |

## Goal
Turn every 🟡/🔴 assumption in `14-confidence-gates.md` §1 into a measured result before building features on it. Each spike has a time box, pass criteria and a fallback that's already decided. The phase ends at gate **G0**.

## What we do differently from the video (and why)
- The video (and the reference project) discovered each blocker mid-feature: Snack SDK limits, EAS keystore and git errors, JSON-blob data, the agent refusing edits. That cost rework each time. We pay about 7 days up front to find them on day 5 instead of week 6.

## Steps

### S0. Vendor fact check (0.5 day)
- Re-verify against today's docs: current `snack-sdk` version and supported SDKs; store Expo Go SDK; EAS non-interactive credentials, webhook header/algorithm, `eas deploy` flags; ImageKit generation URL syntax and pricing; the chosen backend provider's OAuth + management API; Anthropic mid-conversation system messages.
- Check the chosen hosts: does the Postgres plan give a direct (session-mode) connection for `LISTEN`? Can the worker host install Chromium + JDK?
- Record results in `12` (D7, Q1–Q5) and `14` §1.
- **Pass:** every fact has a source link and date. **Fail →** update the affected phase docs before G0.

### S1. Preview runtime (3 days), `spikes/preview-runtime/`
Exactly `07-preview-runtime.md` §9:
1. Hand-write the fixture app `preview-runtime/fixtures/habit-tracker/` following `06` (plan.json, 4 screens incl. sign-in + one stack screen, 4 components, 2 collections). **This fixture is kept** and reused by Phases 07–11 and tests.
2. Vendor bundle + CommonJS loader + expo-router shim → interactive iframe.
3. Frame mode with `initialRoute`; 4 frames on a page.
4. Hot swap from a textarea edit in < 300 ms.
5. Babel `afid` stamps (`line:col`) + click-to-select prints source location; `edit.override` restyles one element without restart.
6. CSS-variable theme switch across all frames.
- **Pass:** 1–6 work in Chrome and Safari; 8 frames stay usable. **Fallback:** `file:line:type` stamps (A7); Sandpack for the loader (A1).

### S2. Generator quality and cost (1.5 days), `spikes/generators/`
- Draft the Planner, Foundation, Component Designer and Screen Builder prompts from `05` §3 and §5, in our own words. **The drafts are kept** and become Phase 06/08 prompts.
- Script: for 5 prompts from `10` §2.1, run plan → foundation → components → screens with structured outputs, writing into the fixture template. Validate and repair once, then render each screen in the S1 harness.
- Measure the compile/render pass rate, repairs needed, cost per app (`ai_usage`-style accounting), wall-clock time, and `cache_read_input_tokens` on screens 2+.
- Send one Planner call with project state as a mid-conversation system message (A6).
- Run the same 5 prompts with `AI_BUILDER_MODEL=claude-sonnet-5-5` for D9 data.
- **Pass:** ≥ 4/5 apps render every screen after ≤ 1 repair per file; cache hits > 0; cost per app within 2× of `11` §2. **Fallback:** time-boxed prompt iteration (1 extra day), then record an ADR with the chosen mitigation (A4).

### S3. Snack on real phones (0.5 day)
- Materialize the fixture to the `snack` target (`06` §10): relative imports, `App.js`, React Navigation shim. Open via QR in Expo Go on one iPhone and one Android phone. Push one file edit and confirm the reload without rescanning.
- **Pass:** loads on both; the edit shows in < 5 s; tabs look native. **Fallback:** web fallback link; record the SDK situation (A2, A3).

### S4. EAS Android build (+ iOS if an Apple account is available) (1 day)
- With a test Expo account: materialize to `eas`, `git init` + commit in the temp dir, then build non-interactively with the chosen keystore approach (Phase 15 options). Receive the webhook (temporary tunnel is fine here) and verify the signature.
- **Pass:** the APK installs and runs on a device; the webhook verifies. **Fallback:** poll-on-view (A12); `keytool` keystore (A10).

### S5. EAS Hosting (0.25 day)
- `expo export -p web` (with `web.output: "single"`) + `eas deploy --prod` non-interactively; republish keeps the URL.
- **Pass:** URL stable across two publishes. **Fallback:** static hosting (A13).

### S6. Backend provider (1 day)
- With the D10 provider: OAuth connect a test account → create project → apply DDL derived from the fixture's collections → read/write from the fixture app in the browser and on a phone, with two users seeing only their own private records and the same shared records.
- **Pass:** all of the above. **Fallback:** gateway mode; or switch provider (A15).

### S7. ImageKit (0.25 day)
- Generate 3 images via URL; measure first-request latency and the "processing" behaviour; await-and-copy into storage; check cost in the dashboard.
- **Pass:** the server can await completion within 60 s per image. **Fallback:** placeholders (A14).

### G0 review (0.25 day)
- Write `ADR-00x` entries for each spike outcome. Update `14` §1 statuses, `11` §2 cost numbers (from S2) and `12` decisions.
- Go: start Phase 01. No-go: re-plan the affected phases first.

## Files to create / change
```
spikes/                         (throwaway; excluded from build, lint and CI except its own scripts)
  preview-runtime/  generators/  snack/  eas/  backend/  imagekit/
preview-runtime/fixtures/habit-tracker/   (kept)
lib/ai/prompts/drafts/                    (kept: prompt drafts from S2)
docs/plan/12-…, 14-…, 11-…               (updated with results)
```

## Database changes
None (spikes use local stores and scripts).

## API surface
None.

## Acceptance checklist
- [ ] S0 facts recorded with sources and dates in `12`/`14`.
- [ ] S1 pass criteria met (screen recording attached).
- [ ] S2 report: pass rate, repairs, cost per app (Opus and Sonnet), cache hit ratio, latency.
- [ ] S3 recording on iPhone and Android.
- [ ] S4 APK installed; webhook signature verified.
- [ ] S5 stable URL across two publishes.
- [ ] S6 two-user privacy check passes.
- [ ] S7 latency and cost measured.
- [ ] No spike credentials committed (`.env.spike` gitignored; tokens revoked after).
- [ ] G0 ADRs written; no assumption a phase depends on is still 🔴.

## Claude Code prompt
```text
Read docs/plan/14-confidence-gates.md, docs/plan/phases/phase-00b-derisking-spikes.md,
docs/plan/06-generated-app-template.md and docs/plan/07-preview-runtime.md.

We're doing spike S<N> only: <name>. This is throwaway code under spikes/<name>/ except
for the assets the phase doc says to keep. Optimize for learning fast, not code quality,
but never commit credentials (use .env.spike, which must be gitignored).

1. Restate the spike's pass criteria and fallback from the phase doc.
2. Ask me for any account or credential you need before starting.
3. Build the smallest thing that tests the pass criteria.
4. Run it and show me the evidence (output, numbers, or what I should check on a device).
5. Write a short findings note: what passed, what failed, surprises, exact versions used,
   and links to the docs you relied on. Propose the ADR text and the updates to
   14-confidence-gates.md §1 and the affected phase docs. Don't edit them until I agree.
Don't start any other spike or any feature work.
```

## Pitfalls & notes
- Time-box hard. A spike that runs over by 50% is itself a finding: write the ADR and take the fallback.
- Use separate test accounts (Expo, provider, ImageKit, Stripe test mode). Revoke the tokens afterwards.
- Spike results go stale. Re-run S0 whenever more than about 6 weeks pass before the phase that depends on them.
