# Phase 12 — AI Images for Generated Apps (ImageProvider + ImageKit adapter)

| | |
|---|---|
| Video chapter(s) | Ch.15 "ImageKit AI Image Transformation" (2:01:53). Reference repo: Prompts 10.1–10.3 |
| Branch | `feat/12-ai-images` |
| Depends on | Phase 08 (generators + image rules), Phase 10 (inspector, `edit.override`, `POST …/files/edits`), Phase 03 (storage), Phase 18 for charging (logged until then) |
| Effort (solo dev + Claude Code) | ~2–3 days |
| New env vars | `IMAGEKIT_ENABLED`, `IMAGEKIT_URL_ENDPOINT`, `IMAGEKIT_PRIVATE_KEY` (server/worker only) |

## Goal
Generated apps that need imagery (a grocery store's products, a recipe app's dishes, onboarding illustrations) get **real, relevant images** instead of grey boxes. With `IMAGEKIT_ENABLED=true`, the generators write ImageKit **generation URLs** where an image goes (`06` §7). Before saving, the server **waits for each image, copies it into the project's folder and signs it**, so phones never see a "still preparing" page. In Edit mode, selecting an image offers **Replace image**, **Generate AI image** and **Apply AI transformation**. The result appears in every frame at once and is saved into the code without regenerating the screen.

## What we do differently from the video (and why)
- **A provider interface instead of hard-wiring a sponsor.** ImageKit is the first adapter because URL-based generation fits generators that write one file. The interface also allows a direct image model, Cloudinary, or stock libraries.
- **No unsigned generation URLs ship.** A raw generation URL built from the public endpoint lets anyone who learns the pattern generate images on your account. We enable ImageKit's "restrict unsigned URLs" setting (verify its name), sign server-side with `IMAGEKIT_PRIVATE_KEY`, and the final app code only contains signed stored copies.
- **Budgets and pricing (ADR-002, `15` §5).** A text-to-image generation is 65 ImageKit units, about **$0.12** at overage rates, so images are priced honestly:

| Action | ImageKit units | Credits |
|---|---|---|
| Generate AI image (build or Edit mode) | 65 | **4** |
| AI transformation via prompt (`e-edit` / `e-changebg`) | 50–60 | **4** |
| Background removal (`e-bgremove`; not the 130-unit `e-removedotbg`) | 10 | **1** |
| Replace / resize an existing image | ~0 | free |

  - **Free plan:** no AI generation. Generators use placeholder images, and the Edit-mode AI actions show an upgrade prompt.
  - **Pro builds:** up to **8 AI images per build**, at most 4 per screen. Each is charged 4 credits and **included in the build estimate** shown before confirming. Extras become placeholders.
  - Dedupe by prompt. Each unique prompt is a billable generation.
  - Production needs a paid ImageKit plan: the free 650 units cover about 10 images for the whole platform.

### Reference app notes
- **R1:** there are no image tools for the model. Image URLs are just strings in the generated code or sample data, following IMAGE_RULES (`05` §5), and code post-processes them. This replaces the earlier `image_url` / `use_uploaded_image` agent tools.
- The reference app found that **the server must wait for ImageKit's AI to finish before the app gets the URL**: a phone can't show the "still preparing" page. The build copies images into the project's folder (`/<brand>/<projectId>`) and signs them; past 8 per step, or on failure, it uses placeholders.
- Prompt 10.1/10.3: inspector actions **Replace / Generate / Transform**. The new URL is a normal inline edit (R9), shown in frames at once and saved in the screen's code. **Images that come from app data** (record fields) can't be changed this way: the inspector says so and offers chat instead.
- **R15 (amended by ADR-002):** the reference app charged 1 credit per Edit-mode image and included build images in the screen price. Verified ImageKit pricing makes that a loss, so images are charged per the table above.

> **Verified (`15` §5, 2026-11-03):**
> - Syntax: `{URL_ENDPOINT}/ik-genimg-prompt-{percent-encoded text}/{path}/{file.ext}`.
> - The first request may return **200 HTML** with header **`is-intermediate-response: true`** while generating (seconds to a minute).
> - The **same path returns the cached image even if the prompt changes**, so regenerate by changing the filename.
> - No documented prompt length limit; we cap at 200 chars.
> - Still to check in S7: the exact name of the "restrict unsigned URLs" setting and signing details.

## Steps

### 1. `ImageProvider` interface (`lib/providers/images/`)
```ts
export interface ImageProvider {
  isGenerationUrl(url: string): boolean;
  /** Fetch (signed) until the image is ready; copy into the project folder; return a signed stable URL. */
  finalize(input: { url: string; projectId: string; name: string }): Promise<{ url: string; providerRef: string }>;
  generate(input: { prompt: string; projectId: string; name: string; width?: number; height?: number }): Promise<{ url: string; providerRef: string }>;
  transform(input: { sourceUrl: string; prompt?: string; removeBackground?: boolean; width?: number; height?: number; projectId: string }): Promise<{ url: string; providerRef: string }>;
}
```
- `lib/providers/images/index.ts` picks `imagekit` when `IMAGEKIT_ENABLED=true` and both keys are set, otherwise `none`. With `none`, IMAGE_RULES tell generators to use `https://picsum.photos/seed/<word>/400/300` placeholders, and Edit mode shows only the URL field.

### 2. ImageKit adapter (`lib/providers/images/imagekit.ts`)
- `generationUrl(description, name)` = `{IMAGEKIT_URL_ENDPOINT}/ik-genimg-prompt-<encoded>/<projectId>/<name>-<salt>.jpg`. Sanitize the description: strip control characters, newlines, commas, slashes, colons and quotes (IMAGE_RULES); collapse whitespace; **cap at 200 chars**; percent-encode (spaces as `%20`). The `<salt>` in the filename is what makes **regeneration** produce a new image, because ImageKit caches by path.
- `finalize`: sign the URL, then poll with backoff (≤ 60 s) while the response carries **`is-intermediate-response: true`** (or isn't an `image/*` content-type), until a real image arrives. Upload or copy the result into the project's folder in the ImageKit media library and return a signed URL with a long expiry, or an unguessable path, because the URL ends up in exported code.
- `transform`: AI edit or background removal on an existing image, saved as **its own file** in the project folder.
- Dedupe: `seedKey = hash(projectId, normalizedDescription, name)`. An existing `project_assets` row with the same `seed_key` returns its URL with no new generation.

### 3. Server post-processing in generator steps (`lib/images/postprocess.ts`)
- Runs in `agent.step` after validation and **before** a screen, component or data file is saved:
  1. Collect string literals with `isGenerationUrl` from the file AST (and from `items` of collections in foundation/data steps).
  2. Apply the plan and budget: Free → all placeholders. Pro → keep up to the build's remaining image allowance (8 per build, 4 per screen file), already charged in the build estimate; replace the rest with placeholders.
  3. `finalize` in parallel (`p-limit(4)`). On failure, retry once with a simplified description, then use a placeholder.
  4. Rewrite the literals to the final URLs (deterministic codemod), and insert `project_assets` rows (`kind: app_image`, `external_url`, `prompt`, `provider`, `provider_ref`, `seed_key`).
- Emit `file.writing` progress ("Preparing 3 images…") so the wait is visible.
- The validator rejects image URLs from other hosts when ImageKit is enabled (IMAGE_RULES), so there are no random image sites or local assets.

### 4. Inspector image actions (Phase 10 shell)
- When the selected element is an `Image` with a literal `source={{ uri: "…" }}`:
  - **Replace image**: paste a URL or pick a project asset (user upload). The server transforms or resizes it into the project folder.
  - **Generate AI image**: a prompt → `POST /images/generate` → `generate` + `finalize` (the server waits) → returns the URL.
  - **Apply AI transformation**: a prompt or "remove background" → `POST /images/transform` → new file → URL.
- The returned URL is applied as `edit.override {afid, imageUri}` in every frame, then persisted through `POST /api/projects/:id/files/edits` (`setProp source.uri`), like any inline edit.
- If the `uri` is an expression (record data), the actions are disabled: "This image comes from the app's data. Ask in chat to change it."
- Generate and AI transform charge **4 credits**; background removal charges **1**; Replace is free. Enforced in Phase 18, logged until then. Free plans see an upgrade prompt for the AI actions.

### 5. User-uploaded images via chat
- Images attached to a chat message are uploaded to the project (`project_assets` `chat_attachment`) and finalized to signed URLs. The PROJECT STATE lists them (`name → url`), so the Planner can put the exact URL into an `update_app` `request` ("use <url> as the hero image"). The Screen Builder keeps such URLs unchanged, because they're person-chosen.

### 6. Template + preview support
- Generated code uses `Image` from `react-native` with `{ uri }` (`expo-image` isn't in the allowlist, `06` §4). No platform wrapper is required.
- Preview runtime CSP `img-src` allows the ImageKit endpoint origin and our storage origin (`07` §6).

### 7. Export considerations
- Exported apps keep the signed stored URLs. Optional stretch for Phase 14: `materialize({ bundleImages: true })` downloads them into `assets/images/` and rewrites the literals to `require()`, so apps don't depend on the provider long-term.

## Files to create / change
```
lib/providers/images/{index.ts,types.ts,imagekit.ts,none.ts}
lib/images/postprocess.ts                     (collect → cap → finalize → rewrite → record)
lib/codemods/imageUrls.ts                     (deterministic literal rewrite)
lib/ai/prompts/rules.ts                       (IMAGE_RULES switch on IMAGEKIT_ENABLED)
lib/validate/images.ts                        (host allowlist, per-file cap)
worker/handlers/agent-step.ts                 (+ postprocess before save)
app/api/projects/[projectId]/images/{generate,transform,replace}/route.ts
components/inspector/sections/ImageSection.tsx
preview-runtime/src/csp.ts                    (+ image origins)
lib/env.ts                                    (+ IMAGEKIT_* — private key server-only)
```

## Database changes
- `project_assets` with `kind = app_image` and the nullable columns `external_url`, `prompt`, `provider`, `provider_ref`, `seed_key` (already in `03` §2). `kind = chat_attachment` for chat uploads.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/images/generate` | editor | `{ prompt, name? }` → `{ url, assetId }` (waits for the image; 4 credits; 402 out of credits / plan gate in Phase 18) |
| POST | `/api/projects/:id/images/transform` | editor | `{ sourceUrl, prompt?, removeBackground? }` → `{ url, assetId }` (4 credits with prompt, 1 for background removal) |
| POST | `/api/projects/:id/images/replace` | editor | `{ url? , assetId? }` → `{ url }` |

## Acceptance checklist
- [ ] With `IMAGEKIT_ENABLED=true`, prompting "a grocery store app for online selling" produces product cards with relevant images (banana, strawberries, milk …) in preview and in Expo Go, and no "preparing" placeholders.
- [ ] Saved code contains only finalized, signed project-folder URLs, never raw unsigned generation URLs.
- [ ] More than 8 image URLs in one build (Pro): the extras become placeholders and the build still succeeds. On Free, all images are placeholders.
- [ ] A 200 `is-intermediate-response: true` response is polled until the real image arrives (unit test with a recorded response). Regenerate produces a different image (new filename salt).
- [ ] The build estimate includes 4 credits per planned AI image (Pro).
- [ ] With ImageKit disabled or unset, builds succeed with placeholder images.
- [ ] A failed generation degrades to a placeholder, with no broken image icons.
- [ ] Edit mode: Generate / Transform / Replace update the selected image in every frame instantly, and the screen's code is saved without regeneration. Reopening the project shows the new image.
- [ ] A data-driven image shows the actions disabled, with a "change it in chat" hint.
- [ ] Re-requesting the same description in the same project reuses the URL (no new asset row).
- [ ] **Security:** `IMAGEKIT_PRIVATE_KEY` appears in no client bundle, device-preview payload, export or generated file (grep test). Unsigned generation URLs are rejected by the ImageKit account settings (manual check, documented in the PR).
- [ ] **Security:** descriptions with newlines, `../`, `?`, `#`, `,` or very long text are sanitized and encoded. A unit test covers each.
- [ ] **AuthZ:** image routes require editor; `replace` rejects asset ids from another project; non-members get 404.

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 12 (AI images via an ImageProvider; ImageKit
adapter first) only.

Read first:
- docs/plan/05-ai-agent-design.md (§5 IMAGE_RULES, §6 validation, §10 image caps)
- docs/plan/06-generated-app-template.md (§7 images)
- docs/plan/13-reference-implementation.md (R1, R9, R15)
- docs/plan/07-preview-runtime.md (§4 edit.override, §6 CSP)
- docs/plan/phases/phase-10-inline-editing.md (inspector + POST /files/edits)
- docs/plan/15-vendor-facts.md §5 and ADR-002 (verified syntax, intermediate response, pricing)
- docs/plan/phases/phase-12-ai-images.md (this phase)
- ImageKit's CURRENT docs to re-confirm syntax, URL signing and "restrict unsigned URLs".
  Quote the sections you followed in the PR description. Don't guess.

Implement, one commit per phase-doc step:
1. ImageProvider interface + "none" adapter; IMAGEKIT_ENABLED switch in IMAGE_RULES.
2. ImageKit adapter: sanitize/encode (200-char cap), salted filenames, sign, poll while
   is-intermediate-response: true, copy into the project folder, transform, seedKey dedupe.
3. lib/images/postprocess.ts wired into agent-step before save (Free: placeholders; Pro: ≤8 per
   build, 4/screen, charged in the build estimate; placeholders on overflow/failure;
   deterministic URL rewrite; project_assets rows).
4. Inspector Image section: Replace / Generate / Transform → edit.override + POST /files/edits.
5. Chat attachments listed in PROJECT STATE with signed URLs.
6. Validator host allowlist + preview CSP img-src.

Rules: no image tools for the model. IMAGEKIT_PRIVATE_KEY is server/worker-only (enforce in
lib/env.ts). Every route uses requireProjectAccess(). Record every generation so Phase 18 can
charge 4 credits per AI image (1 for background removal).
Do NOT build: bundling images into exports (stretch), credit enforcement (Phase 18).
Definition of done = Acceptance checklist. Finish with lint, typecheck, tests, summary.
```

## Pitfalls & notes
- **Latency:** generation can take several seconds per image. Finalize in parallel and show progress. A screen with 4 images shouldn't add more than ~15 s.
- **Consistency across a set:** product shots look better with a shared style phrase ("studio product photo, soft shadow, plain background"). Put it in IMAGE_RULES guidance, and let the Foundation step's sample records use it.
- **No text in images:** IMAGE_RULES ask for descriptions of real pictures without text. Generated text looks broken.
- **Content safety and brands:** keep the provider's safety filters on. No real brand logos or real people.
- **Cost visibility:** every generation is a `project_assets` row with `provider_ref`. Include image counts and cost in eval output. Watch the ImageKit unit balance (the default cap is 8,000 units/month on Pro) and alert at 80%.
- **Alternatives:** a direct text-to-image API with results stored in our bucket (more control, no URL-generation abuse surface), Cloudinary, or curated stock (Unsplash API; attribution rules apply). All fit the same interface.
