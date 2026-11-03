# 01 — Review of the Source Video

**Source:** "Build Lovable for Mobile Apps with AI Agents & Claude Code | Next.js Full Course 2026" (TubeGuruji, Nov 3 2026, ~2 h 53 min).
**Product built in the video:** a prompt-to-mobile-app builder. You type a prompt; an AI agent plans the screens and then generates a React Native / Expo app screen by screen. You can edit it visually or by chat, preview it in the browser and on a phone (QR / Expo Go), connect a backend, export (zip / GitHub), build APK/AAB/IPA and publish to the web. It has free/pro plans.

---

> **Update:** the course's prompt book and architecture roadmap were later shared with us. What they revealed, and the 22 patterns we adopted from them, are in `13-reference-implementation.md`. Several gaps listed below (hidden prompts, agent design) are now covered there.

## 1. Verdict

The video is a good **feature list and build order**. It's a weak **engineering spec**.

- **Good:** the scope and order make sense (UI → auth → projects → workspace → agent → generation → backend → editing → device preview → export → builds → publish → billing → deploy). It ships one feature per branch and PR, and tests every step by hand before moving on. We keep that rhythm (`09-dev-workflow.md`).
- **The real engineering isn't in the video.** Each chapter is "paste my prompt from the Eraser doc into Claude Code, wait, test". The prompts that encode the hard design (agent instructions, preview runtime, inline editor, Snack integration) live in an external doc we don't have. This plan replaces them with explicit specs (`02`–`07`) and our own per-phase prompts (`phases/*`).
- **It's sponsor-shaped.** CloudHike is presented as the backend for *everything*: platform DB, auth, storage, AI gateway, background jobs, app hosting, *and* every generated app's backend. It's the presenter's own product ("I just made this for you guys"). CodeRabbit and ImageKit are sponsors too. None of them are bad choices, but they shouldn't be load-bearing assumptions. We put every vendor behind an adapter (`02-architecture.md` A9) and let you choose (`12-decisions-and-open-questions.md`).

---

## 2. Feature inventory

| # | Feature (as demoed) | Video ch. | Keep? | Our phase |
|---|---|---|---|---|
| 1 | Next.js boilerplate (shadcn, Postgres + Drizzle, Better Auth) | 4 | ✅ (any boilerplate or `create-next-app`) | 00 |
| 2 | CodeRabbit PR reviews + "Secure" deep scans | 5 | ✅ optional; plus CI checks | 00, 09-workflow |
| 3 | Home: prompt box, image attach, iOS/Android toggle, suggestion chips, projects grid | 6 | ✅ | 01 |
| 4 | Email/password + Google/GitHub sign-in, 100 free credits | 7 | ✅ (credits via ledger) | 02, 18 |
| 5 | Create project: save prompt + up to 5 assets (10 MB each), redirect to workspace | 8 | ✅ | 03 |
| 6 | Workspace: chat \| canvas \| preview/code, header with share/export/publish | 9 | ✅ | 04 |
| 7 | Agent: analyze → plan 3–8 screens → revise by chat → confirm | 10 | ✅ (structured output) | 06 |
| 8 | Screen-by-screen generation with live code streaming | 11 | ✅ | 08 |
| 9 | 5 theme packs, instant switch, auto contrast | 11 | ✅ | 08 |
| 10 | Functional screens (local storage), scrollable lists | 11 | ✅ | 08 |
| 11 | Background jobs for long generations | 11 | ✅ **moved to Phase 05** | 05 |
| 12 | Add new screens by chat (initially broken) | 11 | ✅ designed in from the start | 09 |
| 13 | Connect backend for generated app (OAuth to provider, auto-provision DB/auth) | 12 | ✅ provider-agnostic | 13 |
| 14 | Proper per-entity tables instead of a JSON blob (fixed mid-chapter) | 12 | ✅ from the start | 13 |
| 15 | Inline visual editor (select element, style panel, text edit, "Edit with AI") | 13 | ✅ deterministic codemods | 10 |
| 16 | Expo Snack + Expo Go QR, live reload without rescan | 14 | ✅ with pinned SDK | 11 |
| 17 | AI images via ImageKit URL generation/transformation | 15 | ✅ behind ImageProvider | 12 |
| 18 | Download zip; push to GitHub (new repo, later pushes → branch + PR); share by email | 16 | ✅ GitHub App instead of broad OAuth scope | 14 |
| 19 | Android APK/AAB, iOS IPA via EAS (GitHub Actions attempt abandoned) | 17 | ✅ EAS directly | 15 |
| 20 | Publish to web (Expo web + EAS Hosting) | 18 | ✅ export in our worker | 16 |
| 21 | My projects + public showcase, card actions | 19 | ✅ static thumbnails | 17 |
| 22 | Free/Pro plans, credit deduction, Stripe | 20 | ✅ ledger + metering | 18 |
| 23 | Deploy to Vercel, update OAuth redirects and env | 21 | ✅ plus separate worker | 19 |

---

## 3. Problems seen in the video, and how this plan handles them

| # | What happened / what's missing | Risk | Our fix |
|---|---|---|---|
| P1 | Generation first ran inside an API route. Background jobs came in Ch.11 after the Vercel timeout problem was noticed, and they needed **ngrok** locally so the job service could call back. | Rework of the agent loop; brittle dev setup | Queue + worker from Phase 05. The worker pulls from Postgres, so no tunnel is needed (`02-architecture.md` A1–A3). |
| P2 | Progress only appeared after a manual page refresh (Ch.10). | Bad UX | Persisted `run_events` + SSE with resume (Phase 05) |
| P3 | Generated-app data was saved as **one JSON record**; fixed later with a second prompt. | Unqueryable data, migrations hell | One table per entity + migrations from the start (Phase 13, `06` §6) |
| P4 | The agent refused to **add a screen** by chat; its instructions only covered first-time generation. | Core feature broken | Planner `update_app` tool with an explicit "changing a built app" section, plus eval cases (Phase 09, `05` §3.2–§3.5) |
| P5 | Snack / Expo Go only worked on an older Expo SDK than the one generated. | QR preview broken | One pinned SDK in a fixed template (`06` §1) |
| P6 | "Can't create Android keystore in non-interactive build"; GitHub Actions path abandoned for EAS. | Builds fail on first run | Generate the keystore ourselves, store it encrypted, pass it as local credentials (Phase 15) |
| P7 | iOS IPA was shown as a button, but iOS signing (Apple Developer account, certs, profiles) was never really covered. | Users hit a wall | Explicit Apple setup flow (ASC API key) and honest UI copy (Phase 15) |
| P8 | Web publish waited **~12.7 min** in the free EAS queue. | Slow | Run `expo export -p web` in our worker; only *upload* via `eas deploy` (Phase 16) |
| P9 | OAuth-token encryption was added late; CodeRabbit flagged an ownership check that "accepts any 64-char hex value". | Account/project takeover class bugs | Encrypted token columns + `requireProjectAccess()` everywhere from Phase 02 (`08-security.md`) |
| P10 | GitHub export used an OAuth App (broad `repo` scope over **all** the user's repos). | Over-privileged tokens | GitHub App with per-repo installation (Phase 14) |
| P11 | Running AI-generated code (preview, `expo export`, EAS build) was never discussed from a sandboxing point of view. | XSS against the platform; code execution on build workers | Opaque-origin iframes; fixed `package.json`; ephemeral build containers (`07` §6, `08-security.md`) |
| P12 | Credits sat at 100 and never moved until Ch.20; no metering model. | No way to price | `ai_usage` from the first model call (Phase 06), ledger + reservation (Phase 18), cost model (`11-costs-and-pricing.md`) |
| P13 | No automated tests or agent evals; quality judged by eye per demo. | Silent regressions when prompts change | Unit/e2e tests + an agent eval suite with pass-rate metrics (`10-testing-and-evals.md`) |
| P14 | Suggested "free daily OpenAI tokens" by enabling **data sharing** on the API account. | Your users' prompts and code get shared with a model provider for training; a privacy/ToS problem for a SaaS | Don't. Use a normal paid API key; budget for it (`11`) |
| P15 | Live app previews for every card on the home page. | Heavy page, battery drain | Static thumbnails rendered by the worker; live preview on open (Phase 17) |
| P16 | `NEXT_PUBLIC_APP_URL` had to be deleted and recreated on Vercel; redirect URLs had a stray `.` that broke GitHub OAuth. | Deploy-day breakage | Env checklist + redirect-URI checklist (Phase 19) |
| P17 | The agent's tab navigation looked "odd" on Snack. | Fidelity gap | Accept it in preview; template + real builds are the fidelity source; keep the router surface small (`06`, `07`) |

---

## 4. What the video gets right that we keep

- **Plan → confirm → build.** Users catch a wrong plan in seconds, before minutes of generation are spent. It's cheap, and it builds trust.
- **Themes as data.** Switching palettes instantly without regenerating is a strong, low-cost differentiator.
- **Screen-by-screen generation with live progress.** It's engaging, and it maps naturally to per-screen retries.
- **Code ownership.** Zip and GitHub export with a README means no lock-in, which is a selling point against closed builders.
- **Later GitHub pushes go to a new branch + PR**, never straight to `main`.
- **Public showcase previews never touch the real backend.**
- **One feature per branch, PR, automated review, merge.**

## 5. Effort reality check

The video shows the happy path in ~3 hours of footage, with the hard parts hidden in prompts and pause-and-fix cuts ("I will pause the video and fix all the major issues"). A robust version is a multi-week build even with Claude Code. Per-phase estimates are in each phase doc, and the total is in `00-index.md`. The two riskiest pieces, which are worth a spike before committing to dates, are:
1. the **browser preview runtime** (Phase 07, `07-preview-runtime.md`), and
2. **generation quality**: whether apps reliably compile, render and look good (Phase 08 + evals).

## 6. Things to verify yourself before building

These come from the video and from vendor docs that change quickly. Check current docs when you reach the phase:
- Which Expo SDK versions Snack and the store version of Expo Go support right now (Phase 07/11).
- The ImageKit AI generation/transformation URL syntax and pricing (Phase 12).
- The chosen backend provider's OAuth-for-platforms and management APIs (Phase 13).
- `eas-cli` non-interactive flags, credentials handling, and EAS webhook signature format (Phase 15).
- EAS Hosting (`eas deploy`) limits and pricing (Phase 16).
- Anthropic model ids, prices and SDK signatures (`05`, `11`).
