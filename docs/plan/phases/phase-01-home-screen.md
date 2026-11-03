# Phase 01 — Home Screen Layout

| | |
|---|---|
| Video chapter(s) | Ch.6 "Home Screen Layout" (23:35) |
| Branch | `feat/01-home-screen` |
| Depends on | Phase 00 |
| Effort (solo dev + Claude Code) | ~1–1.5 days |
| New env vars | none |

## Goal
Build the landing/home page where everything starts: header, hero, a Claude/Codex-style prompt box with image attachments and a platform toggle, suggestion chips, a projects section (placeholder data for now) and the sign-in dialog shell. This is UI only, with no persistence. The prompt box's public API should already match what Phase 03 needs, so it doesn't have to be rewritten.

## What we do differently from the video (and why)
- **The prompt box is a self-contained component with a typed `onSubmit(payload)` contract**, not page-local state. Phase 02 (auth gate) and Phase 03 (upload + create project) plug into it without rewriting it.
- **Attachment limits come from one constants file** (`lib/limits.ts`: 5 files, 10 MB each, `image/png|jpeg|webp|gif`). The client and server (Phase 03) share the same numbers.
- **Accessibility from the start:** keyboard submit (Enter / Shift+Enter for a newline), labelled buttons, focus trap in the dialog, drag-and-drop *and* paste-to-attach.
- **The projects section is data-driven from a typed placeholder array** with the same shape the real query returns in Phase 17. Swapping it in is a one-line change.

## Steps

### 1. Shared constants
`lib/limits.ts`:
```ts
export const UPLOAD_LIMITS = {
  maxFiles: 5,
  maxBytes: 10 * 1024 * 1024,
  mimeTypes: ["image/png", "image/jpeg", "image/webp", "image/gif"] as const,
};
export const PROMPT_LIMITS = { minChars: 8, maxChars: 4000 };
export const PLATFORMS = ["ios", "android"] as const;
```

### 2. Site header
`components/home/site-header.tsx`: logo + wordmark (AppBuilding placeholder), nav links (Pricing → `/pricing` placeholder, Showcase anchor), and right side **Sign in** (ghost) + **Get started** (primary). Both open the sign-in dialog (tab preselected). Make it responsive: collapse the nav into a sheet on mobile.

### 3. Hero
`components/home/hero.tsx`: headline ("Describe an app. Get a real mobile app."), a one-line subhead, and the prompt box centered below it. Keep it visually quiet; the prompt box is the focus.

### 4. Prompt box
`components/home/prompt-box.tsx`:
- Auto-growing `Textarea` (3–10 rows), character counter near the limit.
- **Attach** button + drag-drop zone + paste handler. Validate count, size and MIME type on the client and show a toast on rejection.
- Thumbnail strip: object-URL previews, a remove button on each, file name on hover. Revoke object URLs on unmount/remove.
- **Platform toggle** (`ToggleGroup`, multi-select): iOS, Android. Default both, at least one required.
- **Submit** button: disabled until the prompt is ≥ `minChars` and at least one platform is selected. Show a spinner while `onSubmit` resolves.
- Contract:
```ts
export type PromptSubmitPayload = {
  prompt: string;
  platforms: Array<"ios" | "android">;
  files: File[];
};
type Props = { onSubmit: (p: PromptSubmitPayload) => Promise<void>; disabled?: boolean };
```
- For now the home page's `onSubmit` just logs and toasts "Sign in to continue". Phase 02 replaces that.

### 5. Suggestion chips
`components/home/suggestion-chips.tsx`: 6–8 chips ("Habit tracker", "Expense tracker with budgets", "Grocery store with cart", "Workout log", "Recipe book", "Journal with moods"). Clicking one fills the textarea with a full sentence prompt (don't auto-submit) and focuses it.

### 6. Projects section
`components/home/projects-section.tsx`:
- Props: `{ mode: "mine" | "showcase"; projects: ProjectCardData[] }`.
- `ProjectCardData = { id, title, updatedAt, thumbnailUrl | null, status, visibility }`. This is the exact shape Phase 17 returns.
- Grid of `ProjectCard` (thumbnail, title, relative time "edited 1h ago", status badge). Empty state for `mine` ("Your apps will appear here").
- Signed out → heading "Built by the community"; signed in → "Your projects". Phase 01 always renders the showcase variant with placeholder data from `lib/placeholders.ts`.

### 7. Sign-in dialog shell
`components/auth/sign-in-dialog.tsx`: `Dialog` with `Tabs` (Sign in / Create account). Fields: name (sign-up only), email, password; buttons for "Continue with Google" / "Continue with GitHub". Every action is a no-op `TODO(phase-02)`. Export a small store or context (`useAuthDialog()` with `open(tab)`) so the header and prompt box can both open it.

### 8. Page composition and polish
`app/(marketing)/page.tsx` composes the header, hero (with the prompt box), chips and projects section. Add the `(marketing)` route group with its own layout. Check 375 px (mobile), 768 px and 1440 px widths. Verify dark mode via `next-themes`.

### 9. Component tests
Vitest + Testing Library: submit disabled/enabled rules; the 6th file is rejected; an 11 MB file is rejected; a non-image is rejected; clicking a chip fills the prompt; Enter submits and Shift+Enter adds a newline.

## Files to create / change
```
app/(marketing)/layout.tsx
app/(marketing)/page.tsx            (moves/replaces app/page.tsx)
components/home/site-header.tsx
components/home/hero.tsx
components/home/prompt-box.tsx
components/home/attachment-strip.tsx
components/home/suggestion-chips.tsx
components/home/projects-section.tsx
components/home/project-card.tsx
components/auth/sign-in-dialog.tsx
components/auth/auth-dialog-context.tsx
lib/limits.ts
lib/placeholders.ts
components/home/__tests__/prompt-box.test.tsx
```

## Database changes
None.

## API surface
None. The `onSubmit` contract is the interface for Phases 02/03.

## Acceptance checklist
- [ ] Home renders the header, hero, prompt box, chips and a projects grid at 375/768/1440 px, in light and dark.
- [ ] Submit stays disabled until the prompt has ≥ 8 characters and at least one platform is selected.
- [ ] Attaching via button, drag-drop and paste all work; previews show and can be removed.
- [ ] Attaching a 6th file, a file > 10 MB, or a non-image shows an error toast and isn't added.
- [ ] Clicking a chip fills the prompt and focuses the textarea without submitting.
- [ ] Sign in / Get started open the dialog on the right tab; Esc closes it; focus returns to the trigger.
- [ ] Every interactive element is keyboard reachable and has an accessible name (check with axe or Lighthouse a11y ≥ 95).
- [ ] Object URLs are revoked (no memory growth after attaching and removing 20 times).
- [ ] Component tests pass in CI.

## Claude Code prompt
```text
Read these first:
- docs/plan/02-architecture.md (§5 layout)
- docs/plan/phases/phase-01-home-screen.md (this phase)
- CLAUDE.md

Implement Phase 01 (home screen UI only) as specified:
- lib/limits.ts with UPLOAD_LIMITS, PROMPT_LIMITS, PLATFORMS (exact values in the doc).
- (marketing) route group: site header, hero, PromptBox, SuggestionChips,
  ProjectsSection with placeholder data typed as ProjectCardData, SignInDialog shell
  with an auth-dialog context so any component can open it.
- PromptBox must expose exactly:
    onSubmit(payload: { prompt: string; platforms: ("ios"|"android")[]; files: File[] }): Promise<void>
  with client-side validation against UPLOAD_LIMITS, drag-drop + paste + button attach,
  thumbnail previews (revoke object URLs), platform ToggleGroup (default both),
  Enter to submit / Shift+Enter newline.
- Use shadcn/ui components and Tailwind; support dark mode via next-themes.
- Write the component tests listed in step 9.

Do NOT implement authentication, API routes, uploads to storage, or any database
access. The home page onSubmit should only show a "Sign in to continue" toast.

Definition of done = the Acceptance checklist in the phase doc.
Finish by running npm run lint, npm run typecheck, npm test; fix failures; then
summarize the components created and screenshot-worthy states I should check by hand.
```

## Pitfalls & notes
- Don't read files into base64 on the client for previews. Use `URL.createObjectURL` and revoke it. The actual upload in Phase 03 goes directly to storage via presigned URLs, never through a Next.js route body.
- Client validation is UX only. Phase 03 re-validates on the server (size, MIME and magic bytes).
- The video's design is a wireframe. Spend design time on the prompt box and project cards, since those are what users see most. Everything else can stay simple.
- Keep the projects section a server component wrapper with a client grid inside, so Phase 17 can fetch on the server without a refactor.
