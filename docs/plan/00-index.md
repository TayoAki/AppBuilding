# AppBuilding — Build Plan (Index)

A step-by-step plan to build an AI mobile-app builder: prompt → AI-planned screens → generated Expo app → visual and chat editing → browser, canvas and phone preview → backend connect → zip/GitHub export → Android/iOS builds → web publish → free/pro plans → production.

It's based on a review of the TubeGuruji course *"Build Lovable for Mobile Apps with AI Agents & Claude Code"* (Nov 2026). It follows the video's feature order but fixes the problems the video ran into, and it spells out the engineering the video left inside unpublished prompts.

---

## How to read this

1. **`01-video-review.md`** (10 min): what the video builds, what goes wrong in it, what we change. Then **`13-reference-implementation.md`**: what the course's actual prompts and architecture notes taught us.
2. **`02-architecture.md`** + **`03-data-model.md`**: the system we're building, and the canonical names.
3. Skim **`05`–`08`**: agent, app template, preview runtime, security. The phases refer back to them constantly.
4. Read **`14-confidence-gates.md`**: what's proven, what isn't yet, and the gates that catch problems early.
5. Work through **`phases/`** in order, starting with **00 → 00b (spikes)**, one PR per phase, using the loop in **`09-dev-workflow.md`**.

## Reference docs

| File | What it covers |
|---|---|
| [01-video-review.md](01-video-review.md) | Review of the source video: feature inventory, 17 problems and our fixes, effort reality check |
| [02-architecture.md](02-architecture.md) | Components (web, worker, **sandbox service**, Postgres, storage), key decisions A1–A11, request flows, repo layout, environments |
| [03-data-model.md](03-data-model.md) | Every platform table with columns, indexes, ownership |
| [04-tech-stack.md](04-tech-stack.md) | Chosen libraries/vendors + alternatives; canonical env var list |
| [05-ai-agent-design.md](05-ai-agent-design.md) | Planner + single-purpose generators, model config, tools, append-only prompt layout, pipelines, validation + repair, caching, event catalogue |
| [06-generated-app-template.md](06-generated-app-template.md) | The fixed Expo template every generated app lives in: ownership rules, imports, theme, data layer |
| [07-preview-runtime.md](07-preview-runtime.md) | Running generated code in sandboxed iframes: canvas, live preview, element inspector |
| [08-security.md](08-security.md) | Threat model and controls; per-phase security checklist source |
| [09-dev-workflow.md](09-dev-workflow.md) | Claude Code + git + PR + review loop, `CLAUDE.md` template, manual QA script |
| [10-testing-and-evals.md](10-testing-and-evals.md) | Test layers, agent eval harness and quality targets, model mocking for CI |
| [11-costs-and-pricing.md](11-costs-and-pricing.md) | Cost per app/edit estimate, credit formula, what plan limits really cost |
| [12-decisions-and-open-questions.md](12-decisions-and-open-questions.md) | Decisions D1–D18 with defaults, open research questions, ADR log |
| [13-reference-implementation.md](13-reference-implementation.md) | What the course's real source roadmap + prompt book revealed: chapter map, 22 adopted patterns (R1–R22), where we differ, pitfalls they hit |
| [14-confidence-gates.md](14-confidence-gates.md) | Assumption register (A1–A22, each proven / likely / unproven with its test and fallback), gates G0–G5, evidence rules, what we monitor |
| [15-vendor-facts.md](15-vendor-facts.md) | Vendor facts verified on 2026-11-03 (Expo SDK/Expo Go/Snack, EAS Build/webhooks/Hosting, ImageKit, Supabase, Anthropic) and the plan changes they forced |

## Phases

Each phase doc has: goal · what differs from the video · steps · files · DB changes · API · acceptance checklist · **ready-to-paste Claude Code prompt** · pitfalls.

| # | Phase | Video ch. | Effort* | Milestone |
|---|---|---|---|---|
| 00 | [Project setup](phases/phase-00-project-setup.md) | 4, 5 | 1 d | Repo, CI, review bot, local services |
| 00b | [De-risking spikes](phases/phase-00b-derisking-spikes.md) | new | 7 d | **G0: architecture go/no-go**. Every risky assumption measured; S1's runtime code and the one canonical fixture graduate into `preview-runtime/` |
| 01 | [Home screen](phases/phase-01-home-screen.md) | 6 | 1–1.5 d | |
| 02 | [Auth & database](phases/phase-02-auth-and-database.md) | 7 | 2–2.5 d | incl. email sender + verification-gated signup credits |
| 03 | [Create project & uploads](phases/phase-03-create-project.md) | 8 | 1.5 d | |
| 04 | [Workspace layout](phases/phase-04-workspace-layout.md) | 9 | 2 d | **M1: clickable shell** |
| 05 | [Background jobs & realtime](phases/phase-05-jobs-and-realtime.md) | (11) new | 2.5–3 d | incl. outbox, two worker wrappers, project event stream |
| 06 | [Agent: planning](phases/phase-06-agent-planning.md) | 10 | 3 d | |
| 07 | [App template & preview runtime](phases/phase-07-template-and-preview-runtime.md) | new | 3–5 d | hardens the S1 runtime |
| 08 | [Agent: app generation & themes](phases/phase-08-app-generation-and-themes.md) | 11 | 6–8 d | incl. sandbox service skeleton (smoke tests). **M2: prompt → working app** |
| 09 | [Chat edits & add screens](phases/phase-09-chat-edits.md) | 11 | 3–4 d | |
| 10 | [Inline visual editing](phases/phase-10-inline-editing.md) | 13 | 4–5 d | |
| 10b | [Code editing](phases/phase-10b-code-editing.md) | new | 1.5–2 d | |
| 11 | [Device preview (QR / Expo Go)](phases/phase-11-device-preview-qr.md) | 14 | 4–6 d | Path B hosted dev server. **M3: editable app on a real phone** |
| 12 | [AI images](phases/phase-12-ai-images.md) | 15 | 2–3 d | |
| 13 | [Generated-app backend](phases/phase-13-generated-app-backend.md) | 12 | 5–8 d | |
| 14 | [Export (zip, GitHub) & sharing](phases/phase-14-export-and-sharing.md) | 16 | 4–5 d | |
| 15 | [Native builds (EAS)](phases/phase-15-native-builds.md) | 17 | 4–6 d | Expo connect / EAS init / identity now built in 16 |
| 16 | [Web publish](phases/phase-16-web-publish.md) | 18 | 4–4.5 d | incl. Expo connect, EAS init, identity. **M4: feature-complete** |
| 17 | [Projects gallery & showcase](phases/phase-17-projects-gallery.md) | 19 | 3–4 d | |
| 18 | [Plans, credits & billing](phases/phase-18-plans-and-billing.md) | 20 | 4 d | |
| 19a | [Production deploy & ops](phases/phase-19-production-deploy.md) | 21 | 3–4 d | deploy, CI, observability |
| 19b | [Data lifecycle, upgrades & launch gate](phases/phase-19-production-deploy.md) | new | 5–7 d | retention/deletion/export, template-upgrade machinery, vendor watch, G4 evidence. **M5: launch** |

\* Working days for one developer using Claude Code, reflecting the ADR-003 scope changes. If a phase doc's header disagrees, the phase doc wins and this table must be updated in the same PR.

**Full plan ≈ 76–97 working days (~15–19 weeks)**, before a recommended 15–20% buffer. Cumulative:

| Gate / milestone | After | Cumulative |
|---|---|---|
| G0 | 00, 00b | ≈ 8 d |
| M1 / G1 | 01–04 | ≈ 14.5–15.5 d |
| M2 / G2 | 05–08 | ≈ 29–34.5 d |
| M3 / G3 | 09, 10, 10b, 11 | ≈ 41.5–51.5 d |
| M4 | 12–16 | ≈ 60.5–78 d |
| M5 / G5 | 17, 18, 19a, 19b | ≈ 75.5–97 d |

## MVP track (launch first, then v1.1)

A launchable beta needs only part of the plan:

| Track | Phases | Effort |
|---|---|---|
| **MVP** | 00, 00b, 01–11, 10b, 14 *(zip export only, ≈ 1.5 d)*, 16 *(web publish incl. Expo connect, EAS init and identity)*, 17 *("my projects" grid only, ≈ 1.5 d)*, 18, 19a, 19b | **≈ 61–75 d (~12–15 weeks)** |
| **v1.1** | 12 (AI images), 13 (generated-app backend), 14 *(GitHub push + sharing)*, 15 (native builds), 17 *(public showcase + approval queue)* | ≈ 15–23 d |

MVP arithmetic (low–high): G0 8 · 01–04 6.5–7.5 · 05–08 14.5–19.5 · 09/10/10b/11 12.5–17 · 14-zip 1.5 · 16 4–4.5 · 17-mine 1.5 · 18 4 · 19a 3–4 · 19b 5–7.

At MVP, people can prompt → plan → build, edit by chat, inspector or code, test on their phone in store Expo Go (Path B), export the code and publish to the web. Apps keep data on the device and use placeholder images until v1.1 adds backends and AI images. Gate **G4** (`14` §2) must pass before launch.

**Order changes vs. the video**
- **Background jobs (05)** move before any AI work. The video added them after generation already worked, which forced a rewrite and an ngrok tunnel.
- **Template & preview runtime (07)** is a new phase. The runtime is first proven in spike S1 (00b) with the canonical hand-written fixture (sign-in + 2 tabs + 1 stack detail, 4 components, 2 collections), *before* the AI writes code; Phase 07 hardens it.
- **Generated-app backend (13)** moves after editing/preview. Those phases don't depend on it, and doing them first ships a usable product sooner.

## Dependency graph

```
00 → 00b(G0) → 01 → 02 → 03 → 04 ─┬─→ 05 → 06 ─┐
                                  └────→ 07 ───┴─→ 08 (+ sandbox) → 09 → 10 → 10b
                                                    │
                                                    ├─→ 11 (needs 07 materialize + 08 sandbox)
                                                    ├─→ 12
                                                    ├─→ 13 → (14, 15, 16 use backend config if present)
                                                    ├─→ 16 (Expo connect, EAS init, identity) → 15
                                                    ├─→ 14
                                                    └─→ 17
                       06 logs usage ───────────→ 18 (enforcement) → 19a → 19b (G4)
```

Phases 05 and 07 can be built in parallel by two people (or two worktrees).

## Status tracker

Tick as phases merge. Keep this the single source of truth for progress.

- [ ] 00 · [ ] 00b/G0 · [ ] 01 · [ ] 02 · [ ] 03 · [ ] 04 · [ ] 05 · [ ] 06 · [ ] 07 · [ ] 08 · [ ] 09
- [ ] 10 · [ ] 10b · [ ] 11 · [ ] 12 · [ ] 13 · [ ] 14 · [ ] 15 · [ ] 16 · [ ] 17 · [ ] 18 · [ ] 19a · [ ] 19b/G4
