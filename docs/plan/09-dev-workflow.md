# 09 — Development Workflow (Claude Code + Git + Review)

How we turn each phase doc into merged code. It's the video's rhythm (one feature → branch → PR → automated review → merge), made repeatable.

---

## 1. The loop for every phase

```
1. Read the phase doc yourself (5 min). Resolve any open decision it lists (12-decisions…).
2. git switch main && git pull && git switch -c feat/NN-slug
3. Start a NEW Claude Code session (fresh context = cheaper + more focused).
4. Paste the phase's "Claude Code prompt". For big phases, ask for a plan first
   (plan mode), review it, then let it implement.
5. Let it implement in small commits. Interrupt early if it drifts from the doc.
6. Run the acceptance checklist by hand in the browser. Paste any errors back.
7. npm run lint && npm run typecheck && npm test   (CI runs the same)
8. Push, open a PR using the template below. Wait for CI + CodeRabbit.
9. Fix every "major"/security finding; triage minor ones (fix, or note why not).
10. Squash-merge. Update the phase doc's checklist + 12-decisions if anything changed.
```

**Rules of thumb**
- One phase = one PR. If a phase grows past roughly 1,500 changed lines, split it (e.g. 07a runtime spike, 07b canvas wiring).
- Don't start phase N+1 on top of an unmerged phase N.
- If reality diverges from the plan (an API doesn't exist, a library won't work), **update the doc in the same PR**. The docs are the shared memory between sessions.

## 2. `CLAUDE.md` (create in Phase 00)

Put this at the repo root so every Claude Code session starts with the same context:

```markdown
# AppBuilding — instructions for Claude

This repo builds an AI mobile-app builder (Next.js platform + worker + Expo app template).
The full plan lives in docs/plan/. Before working on a phase, read:
- docs/plan/00-index.md (phase list and status)
- docs/plan/02-architecture.md, 03-data-model.md
- the phase doc in docs/plan/phases/ you were asked to implement
and any reference doc that phase links to.

## Conventions
- TypeScript strict. Zod for every external input (env, API bodies, tool inputs, postMessage).
- DB: Drizzle; table/column names exactly as in docs/plan/03-data-model.md.
- Every route/server action/job that takes projectId or runId calls
  requireProjectAccess() first (lib/auth/require.ts).
- Secrets: never log them; per-user tokens are encrypted with lib/crypto/encrypt.ts.
- Long-running work goes through lib/jobs (pg-boss) → worker/handlers/*, never in a request.
- AI calls only from the worker via lib/ai/client.ts (records ai_usage).
- Generated-app rules live in templates/expo-app/CONTRACT.md — don't loosen the
  import/path allowlists without updating that file and the evals.

## Commands
npm run dev | npm run worker | npm run db:push | npm run lint | npm run typecheck |
npm test | npm run test:e2e | npm run eval

## Definition of done
The phase doc's acceptance checklist passes, lint/typecheck/tests pass, the
phase doc is updated if anything changed, and no TODOs are left without an issue.
```

## 3. Branches, commits, PRs

- Branches: `feat/NN-slug` (e.g. `feat/05-jobs-and-realtime`); fixes `fix/<slug>`; docs `docs/<slug>`.
- Commits: Conventional Commits (`feat(workspace): add resizable panels`).
- PR template (`.github/pull_request_template.md`, Phase 00):

```markdown
## Phase
Phase NN — <title> (docs/plan/phases/phase-NN-*.md)

## What changed
-

## Acceptance checklist
- [ ] (copy from the phase doc, tick what you verified)

## Screenshots / recording

## Security notes
(authz, secrets, untrusted input touched?)

## Docs updated
- [ ] phase doc / reference docs reflect reality
```

## 4. Automated review

- **CI (required):** GitHub Actions on every PR runs `lint`, `typecheck`, `test`, `build`. Add `test:e2e` once Phase 04 exists, and `eval:smoke` (5 prompts) once Phase 08 exists.
- **CodeRabbit (optional, as in the video):** install the GitHub app on the repo and add `.coderabbit.yaml` with path instructions, e.g. "lib/auth/**: check every exported function enforces session; worker/handlers/**: check project ownership is re-verified". Treat its "major" findings like failing CI. Its "Prompt for AI agent" snippets can be pasted straight into Claude Code.
- **Claude Code review:** for security-sensitive PRs, run a fresh session with `/security-review` or `/code-review` on the branch before merging.

## 5. Working with Claude Code effectively on this project

- **Start each phase in a new session.** Reference docs by path instead of pasting them; Claude reads them itself.
- **Use plan mode for Phases 05, 07, 08, 10, 13, 15, 18.** They're the ones with real design risk. Approve the plan only if it matches the doc.
- **Give it the failure, not a guess.** Paste exact error text, stack traces, screenshots (the video does this well with the sign-up error).
- **Keep the fixture app handy.** Phases 07–10 can be exercised without spending credits by loading `preview-runtime/fixtures/habit-tracker`.
- **Record agent costs.** After Phase 06, check `ai_usage` totals after each manual test. Surprises there are bugs.
- **Don't let it "simplify" security.** If a session proposes skipping `requireProjectAccess`, sandbox flags or encryption to get something working, stop and re-read `08-security.md`.

## 6. Manual QA script (run before merging Phases ≥ 08)

1. Sign up as a new user → 100 free credits shown (Phase 18 onward: from the ledger).
2. Prompt "habit tracker with streaks" → plan with 3 screens → revise "only 2 screens" → confirm.
3. Watch screens build. Refresh mid-build; progress must resume.
4. Click through the interactive preview; add/complete/delete an item.
5. Switch theme twice; no credits change.
6. Inline-edit a heading color and text; undo.
7. Chat: "add a settings screen with dark mode toggle" → new screen appears and is navigable.
8. Scan the QR with Expo Go → app loads; make an edit → phone reloads.
9. Export zip → `npm install && npx expo start` works locally.
10. Second account: try opening the first account's workspace URL → 404.
