# AppBuilding — Build Plan (Index)

A step-by-step plan to build an AI mobile-app builder: prompt → AI-planned screens → generated Expo app → visual and chat editing → browser, canvas and phone preview → backend connect → zip/GitHub export → Android/iOS builds → web publish → free/pro plans → production.

It's based on a review of the TubeGuruji course *"Build Lovable for Mobile Apps with AI Agents & Claude Code"* (Nov 2026). It follows the video's feature order but fixes the problems the video ran into, and it spells out the engineering the video left inside unpublished prompts.

---

## How to read this

1. **`01-video-review.md`** (10 min): what the video builds, what goes wrong in it, what we change.
2. **`02-architecture.md`** + **`03-data-model.md`**: the system we're building, and the canonical names.
3. Skim **`05`–`08`**: agent, app template, preview runtime, security. The phases refer back to them constantly.
4. Work through **`phases/`** in order, one PR per phase, using the loop in **`09-dev-workflow.md`**.

## Reference docs

| File | What it covers |
|---|---|
| [01-video-review.md](01-video-review.md) | Review of the source video: feature inventory, 17 problems and our fixes, effort reality check |
| [02-architecture.md](02-architecture.md) | Components, key decisions A1–A10, request flows, repo layout, environments |
| [03-data-model.md](03-data-model.md) | Every platform table with columns, indexes, ownership |
| [04-tech-stack.md](04-tech-stack.md) | Chosen libraries/vendors + alternatives; canonical env var list |
| [05-ai-agent-design.md](05-ai-agent-design.md) | Planner / Builder / Editor, model config, tools, loop, validation, caching, events |
| [06-generated-app-template.md](06-generated-app-template.md) | The fixed Expo template every generated app lives in: ownership rules, imports, theme, data layer |
| [07-preview-runtime.md](07-preview-runtime.md) | Running generated code in sandboxed iframes: canvas, live preview, element inspector |
| [08-security.md](08-security.md) | Threat model and controls; per-phase security checklist source |
| [09-dev-workflow.md](09-dev-workflow.md) | Claude Code + git + PR + review loop, `CLAUDE.md` template, manual QA script |
| [10-testing-and-evals.md](10-testing-and-evals.md) | Test layers, agent eval harness and quality targets, model mocking for CI |
| [11-costs-and-pricing.md](11-costs-and-pricing.md) | Cost per app/edit estimate, credit formula, what plan limits really cost |
| [12-decisions-and-open-questions.md](12-decisions-and-open-questions.md) | Decisions D1–D17 with defaults, open research questions, ADR log |

## Phases

Each phase doc has: goal · what differs from the video · steps · files · DB changes · API · acceptance checklist · **ready-to-paste Claude Code prompt** · pitfalls.

| # | Phase | Video ch. | Milestone |
|---|---|---|---|
| 00 | [Project setup](phases/phase-00-project-setup.md) | 4, 5 | Repo, CI, review bot, local services |
| 01 | [Home screen](phases/phase-01-home-screen.md) | 6 | |
| 02 | [Auth & database](phases/phase-02-auth-and-database.md) | 7 | |
| 03 | [Create project & uploads](phases/phase-03-create-project.md) | 8 | |
| 04 | [Workspace layout](phases/phase-04-workspace-layout.md) | 9 | **M1: clickable shell** |
| 05 | [Background jobs & realtime](phases/phase-05-jobs-and-realtime.md) | (11) new | |
| 06 | [Agent: planning](phases/phase-06-agent-planning.md) | 10 | |
| 07 | [App template & preview runtime](phases/phase-07-template-and-preview-runtime.md) | new | ⚠️ highest-risk spike |
| 08 | [Agent: app generation & themes](phases/phase-08-app-generation-and-themes.md) | 11 | **M2: prompt → working app** |
| 09 | [Chat edits & add screens](phases/phase-09-chat-edits.md) | 11 | |
| 10 | [Inline visual editing](phases/phase-10-inline-editing.md) | 13 | |
| 11 | [Device preview (QR / Expo Go)](phases/phase-11-device-preview-qr.md) | 14 | **M3: editable app on a real phone** |
| 12 | [AI images](phases/phase-12-ai-images.md) | 15 | |
| 13 | [Generated-app backend](phases/phase-13-generated-app-backend.md) | 12 | |
| 14 | [Export (zip, GitHub) & sharing](phases/phase-14-export-and-sharing.md) | 16 | |
| 15 | [Native builds (EAS)](phases/phase-15-native-builds.md) | 17 | |
| 16 | [Web publish](phases/phase-16-web-publish.md) | 18 | **M4: feature-complete** |
| 17 | [Projects gallery & showcase](phases/phase-17-projects-gallery.md) | 19 | |
| 18 | [Plans, credits & billing](phases/phase-18-plans-and-billing.md) | 20 | |
| 19 | [Production deploy & ops](phases/phase-19-production-deploy.md) | 21 | **M5: launch** |

**Order changes vs. the video**
- **Background jobs (05)** move before any AI work. The video added them after generation already worked, which forced a rewrite and an ngrok tunnel.
- **Template & preview runtime (07)** is a new phase. It proves the runtime with a hand-written app *before* the AI writes code.
- **Generated-app backend (13)** moves after editing/preview. Those phases don't depend on it, and doing them first ships a usable product sooner.

## Dependency graph

```
00 → 01 → 02 → 03 → 04 ─┬─→ 05 → 06 ─┐
                        └────→ 07 ───┴─→ 08 → 09 → 10
                                          │
                                          ├─→ 11 (needs 07 materialize)
                                          ├─→ 12
                                          ├─→ 13 → (14, 15, 16 use backend config if present)
                                          ├─→ 14 → 15 → 16
                                          └─→ 17
                       06 logs usage ───────────→ 18 (enforcement) → 19
```

Phases 05 and 07 can be built in parallel by two people (or two worktrees).

## Status tracker

Tick as phases merge. Keep this the single source of truth for progress.

- [ ] 00 · [ ] 01 · [ ] 02 · [ ] 03 · [ ] 04 · [ ] 05 · [ ] 06 · [ ] 07 · [ ] 08 · [ ] 09
- [ ] 10 · [ ] 11 · [ ] 12 · [ ] 13 · [ ] 14 · [ ] 15 · [ ] 16 · [ ] 17 · [ ] 18 · [ ] 19
