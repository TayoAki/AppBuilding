# 13 — Lessons from the Reference Implementation

**Sources** (shared by the project owner, Nov 2026):
- *AppForge: App Builder Roadmap*: the shipped product's architecture notes ("how X works").
- *AppForge Project Prompts*: every build prompt typed into Claude Code (15 chapters), plus the product's own runtime prompts (Planner, Foundation/Data/Component designers, Screen Builder, repair prompts).
- Course Eraser doc (`app.eraser.io/workspace/4ixx2EqOzixkpo17Bafn`): renders client-side and couldn't be fetched. The two files above appear to cover its content.
- Source code: `github.com/rrs301/appForge-Ai-agent-mobile-app-builder`, one branch per chapter. Not read for this plan; attach the repo to a session to diff against real code.

The video hid the engineering inside these prompts. Reading them, we changed the plan where the shipped product had found a better or proven answer, and kept our design where it's stronger. We use the ideas, rewritten in our own words, not their text, and keep our own product name.

---

## 1. Chapter mapping

| Repo ch. | Topic | Video ch. | Our phase |
|---|---|---|---|
| 1 | Home screen layout | 6 | 01 |
| 2 | Auth + DB (+ social sign-in, 100 credits once) | 7 | 02 |
| 3 | Create new project (upload on Send only) | 8 | 03 |
| 4 | Workspace layout | 9 | 04 |
| 5 | AI agent configuration (plan → confirm, streamed) | 10 | 06 |
| 6 | Build app screens (themes, local data, jobs, edit by chat) | 11 | 05, 07, 08, 09 |
| 7 | App backend (CloudHike) | 12 | 13 |
| 8 | Inline editing (+ real tables, act on latest, components) | 13 | 10, 09, 13 |
| 9 | Expo preview with QR (Snack) | 14 | 11 |
| 10 | ImageKit AI images | 15 | 12 |
| 11 | Export ZIP + GitHub (+ shared tab layout, default Sign In) | 16 | 14, 08 |
| 12 | Android + iOS build files (EAS) | 17 | 15 |
| 13 | Publish as website (EAS Hosting) | 18 | 16 |
| 14 | Projects on the home page | 19 | 17 |
| 15 | Free and Pro plans | 20 | 18 |

## 2. What we adopted (and where)

| # | Reference-app finding | Change in our plan | Docs |
|---|---|---|---|
| R1 | **One conversational Planner + single-purpose generators** (Foundation, Data, Component Designer/Builder, Screen Builder), each returning one JSON object or one file; code validates and asks for **one repair** | Replaced our tool-looping Builder/Editor with this pipeline; generators are stateless calls | `05` §2–§6, Phases 06, 08, 09 |
| R2 | Planner plans **through tools** (`propose_plan`, `confirm_plan`) and changes apps through `update_app` (structured change spec) and `set_theme` | Same tools; `AppChange` schema; the agent has no file tools at all | `05` §3.3–§3.5 |
| R3 | **"What to act on"**: act only on the latest message; the current plan is the truth; "Start new session" button | Built into Planner rules + chat UI | `05` §3.2, §9; Phase 09 |
| R4 | **Default Sign In screen** for any app with personal data; `firstScreenId` for welcome/onboarding/sign-in | Planning rules + plan schema | `05` §3.2, §3.4 |
| R5 | **Collections as data** (typed fields, `references`, `shared`, sample records) + generic hooks `useCollection` / `useStoredValue` / `useAuth` / `uploadFile` from `@/lib/backend` | Replaced agent-written `lib/store/*`; platform writes schema/seed from JSON | `06` §6 |
| R6 | **Per-app shared component library** (5–10, props-only, ordered imports, no tab bar); edits to one component update every screen | Replaced our fixed platform `components/ui` library | `06` §8, Phase 08/09 |
| R7 | **Shared tab layout** generated from the plan (`screen.tab`); adding a tab rewrites no screen | Platform-owned `app/(tabs)/_layout.tsx` | `06` §2–§3 |
| R8 | Themes: `import { colors } from "@/theme"`; **CSS variables in the preview** so a switch only sets variables; `activeThemeId` on native | Replaced `useTheme()`/`makeStyles` | `06` §5, `07` |
| R9 | **One element per line with its own named style**, so a click maps to source; edits apply as a **live override first, then are spliced into source** | Generator rule + runtime protocol `edit.override` | `06` §9, `07` §5, Phase 10 |
| R10 | Person-chosen literal colors are kept on edits; the validator allows literals already in the file or named in the request | Validator + edit prompt rule | `05` §6, Phase 10 |
| R11 | **Snack: SDK 54** (published `snack-sdk` max); Snack runs generated apps more reliably on **React Navigation**: relative imports, `App.js` entry, `expo-router` shim on React Navigation; QR session lives in the **browser tab**; web fallback when Expo Go fails | Kept as **Path A** (Snack materialization target) for when Snack supports the template SDK. **The SDK 54 pin is superseded by ADR-002**: store Expo Go now supports only SDK 58, so we pin 58 and default to a hosted Expo dev server (Path B) | `06` §1, §10; Phase 11; `15` §1 |
| R12 | Backend: the agent gets one narrow `setup_backend` tool; **server code** calls the provider (raw MCP would let the model read secrets and reach other projects); deploy Auth/Storage only if code uses them | Same | `05` §3.3, Phase 13 |
| R13 | App tables **derived from collections by platform code** (`app_<name>`, typed columns, FKs, additive-only, locked transaction); unknown fields → 400 | Replaced agent-written SQL migrations (`backend_migration` tool removed) | `03` §10, Phase 13 |
| R14 | Apps can't hold DB credentials → **the platform serves a data gateway** (`/api/apps/[id]/…`) when the provider has no row-level security or functions | Gateway is the default for providers without RLS; direct client + RLS for providers with it | Phase 13 |
| R15 | **Flat credit prices**: 10/new screen, 5/change round, 1/AI image; chat, plans and theme switches free; minimum 5 credits to message; two balances (never-expiring + monthly, monthly spent first); charge before build in one conditional `UPDATE`; refunds computed from what's unbuilt; unique `reference` per ledger row | Adopted as the default pricing model; `ai_usage` still tracks real cost for margins | `11`, `03` §8, Phase 18 |
| R16 | Stripe: re-read the subscription from Stripe on each webhook (events arrive out of order); credits only from paid `subscription_create` / `subscription_cycle` invoices; downgrade = `cancel_at_period_end` | Phase 18 webhook rules | Phase 18 |
| R17 | GitHub: first push to the default branch **only if the repo is empty**, otherwise always a new `<brand>/…` branch | Phase 14 rule | Phase 14 |
| R18 | EAS: identity (slug, package, bundle id) set once with a project-id suffix; `appVersionSource: remote`; the uploaded project needs a **git repo** or EAS prompts for `git init` and fails non-interactively (we use `EAS_NO_VCS=1` instead, `15` §2); iOS shows setup steps if EAS has no Apple credentials | Phase 15 pitfalls + identity rules | Phase 15 |
| R19 | Web publish: one row per project; `live_url` changes only on success (the old version stays up); `web.output: "single"`; reserve the `*.expo.app` name up front; app `apiUrl` comes from the platform URL, so publish from a deployed platform if the app has a backend | Phase 16 | Phase 16 |
| R20 | Public cards run the real app lazily (load near viewport, stop when scrolled away); public preview strips `plan.backend` so visitors use local sample data | Offered as an alternative to static thumbnails; backend stripping kept | Phase 17 |
| R21 | Reference images: the Planner sees only **file names** of attachments in their implementation | We pass images as image blocks (better); keep the "don't claim to have read what you can't see" rule | `05` §3.1 |
| R22 | Prompt-writing habits that worked: limit scope; mention what's coming later; describe what the person sees; link docs; "just tell me, don't implement"; have the agent write the next prompt; report what you see (paste raw logs) | Added to our workflow | `09` |

## 3. Where we deliberately differ

| Topic | Reference app | Our plan | Why |
|---|---|---|---|
| LLM access | OpenAI Agents SDK → Anthropic's OpenAI-compatible endpoint | Anthropic SDK directly | Prompt caching, structured outputs, refusal fallbacks, exact usage |
| Background work | CloudHike Jobs calling back to a public URL (needs ngrok locally; planning still in the request process; build "shows as building" if nobody watches) | pg-boss + own worker pulling from Postgres | No tunnel, no inbound webhook, builds progress whether or not anyone is watching |
| Planner prompt | Whole system prompt rebuilt per message | Static cached system + state as a mid-conversation system message | Caching |
| JSON generators | "Reply with ONLY JSON" | Structured outputs (`messages.parse`) | No parse failures |
| Build status (EAS) | Polled only while the dialog is open (no webhook; undocumented GraphQL API for workflows) | `eas-cli` in the worker + EAS webhooks; polling as fallback | Progress without an open tab; documented interfaces first |
| Sharing | View-only, existing accounts only, all mutations owner-only | Viewer/editor roles with email invites | Collaboration. v1 may ship view-only (it's simpler) |
| Card previews | Live sandboxed apps per card | Static thumbnails by default; live on hover/open | Home page performance (either is acceptable) |
| Default model | `claude-sonnet-5-5` for all agents | `claude-opus-5-5` default; Sonnet 5.5 via `AI_BUILDER_MODEL` if evals hold (D9) | Your call; the reference app is evidence that Sonnet 5.5 is good enough |

## 4. Things the reference app ran into (don't repeat them)

- The agent said "editing by chat isn't available" after the first build, because its instructions were written for first-time generation only.
- Re-adding screens requested earlier in the chat → fixed with "act only on the latest message" + "Start new session".
- One JSON-record table for app data → migrated to real tables (and kept a `_legacy` backup).
- Tab bars drawn per screen looked like separate pages on a real device → shared tab layout.
- A render loop hammered the dev server through ngrok (1000+ requests); preview frames polled constantly → poll only while the agent is changing screens.
- The workspace page scrolled and showed blank space → fixed-height layout.
- Generated Expo build output got into git → `.gitignore` the build/export folders.
- GitHub Actions for native builds was abandoned for EAS; EAS failed without a git repo in the upload.
- Publish dialog errors pushed the layout around and stayed after retry → collapsible error area, clear on retry.
