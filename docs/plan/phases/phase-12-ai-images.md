# Phase 12 — AI Images for Generated Apps (ImageProvider + ImageKit adapter)

| | |
|---|---|
| Video chapter(s) | Ch.15 "ImageKit AI Image Transformation" (2:01:53) |
| Branch | `feat/12-ai-images` |
| Depends on | Phase 08 (Builder tools), Phase 03 (uploads/storage), Phase 09 (Editor) |
| Effort (solo dev + Claude Code) | ~2–3 days |
| New env vars | `IMAGEKIT_URL_ENDPOINT`, `IMAGEKIT_PRIVATE_KEY` (server/worker only) |

## Goal
Generated apps that need imagery (a grocery store's products, a recipe app's dishes, onboarding illustrations) get **real, relevant images** instead of grey boxes or random stock photos. The agent calls an `image_url` tool with a prompt and size. It gets back a URL from the configured **ImageProvider**: AI text-to-image generation plus transformations such as resize, smart crop and background removal. The agent references those URLs from an agent-owned image registry. Users can also tell the agent to use their own uploaded images in the app.

## What we do differently from the video (and why)
- **A provider interface instead of hard-wiring a sponsor.** ImageKit is the first adapter because its URL-based generation fits the agent well. The interface also allows a direct image model, Cloudinary, or static stock libraries.
- **Signed URLs, created server-side.** In the video the generation URL is built from the public endpoint. Anyone who learns the pattern could generate images billed to your account. We enable ImageKit's "restrict unsigned URLs" option (verify the setting name), sign every URL in the worker with `IMAGEKIT_PRIVATE_KEY`, and never give the key to clients or generated apps.
- **Pre-warming.** URL-based generation happens on the first fetch. The worker fetches each new URL once during the build, so the first app user doesn't wait and failed generations show up immediately (and get retried or replaced).
- **Budgets.** There's a per-build image cap and prompt dedupe, because each unique prompt is a billable generation.

> ⚠️ **Verify ImageKit's exact URL syntax in their current docs** before implementing: the generation path segment (the video shows an `ik-genimg-prompt-<prompt>` style path), transformation parameters (e.g. width/height, focus/smart crop, background removal effect), signing algorithm and expiry parameters, and pricing per generation. The shapes below are illustrative.

## Steps

### 1. `ImageProvider` interface (`lib/providers/images/`)
```ts
export interface ImageProvider {
  /** Returns a stable, signed, publicly fetchable URL. Never returns secrets. */
  generate(input: { prompt: string; width: number; height: number; style?: string; seedKey: string }): Promise<{ url: string; providerRef: string }>;
  transform(input: { sourceUrl: string; width?: number; height?: number; crop?: "smart" | "center"; removeBackground?: boolean }): Promise<{ url: string }>;
  warm(url: string): Promise<{ ok: boolean; status: number; bytes?: number }>;
}
```
- `lib/providers/images/index.ts` picks the adapter (`imagekit` | `none`). With `none`, the tool returns a themed placeholder (`assets/placeholder.png` + tint), so apps still render with no provider configured.

### 2. ImageKit adapter (`lib/providers/images/imagekit.ts`)
- Build the generation URL: `{IMAGEKIT_URL_ENDPOINT}/<generation-segment-with-encoded-prompt>/<seedKey>.jpg?tr=w-<w>,h-<h>` (illustrative).
- Prompt sanitization: strip control characters and newlines, collapse whitespace, cap at ~200 characters, then URL-encode the segment the way ImageKit requires (check whether they need double encoding or forbid `/` and `,`). Reject prompts that are empty after sanitizing.
- `seedKey` = a short hash of `(projectId, normalizedPrompt, w, h)`, so the same request reuses the same URL (cache hit, no new generation).
- Sign the final URL with `IMAGEKIT_PRIVATE_KEY` (official SDK or its documented HMAC scheme). Use no expiry, or a very long one, because the URL ends up in exported app code.
- `transform`: for user-uploaded images, use ImageKit with our bucket as an external origin, or upload them to the ImageKit media library first (decide in implementation). Then apply resize/crop/bg-removal parameters and sign.
- `warm`: `GET` with a 60 s timeout. Treat non-2xx or a non-image content type as failure.

### 3. Agent tool `image_url` (`lib/ai/tools/image-url.ts`)
- Input (strict): `{ purpose: string, prompt: string, width: int 64–2048, height: int 64–2048, key: /^[a-z][a-zA-Z0-9]{1,40}$/ }`.
- Behaviour: check the per-run cap (`IMAGE_CAP_PER_RUN`, a code constant, e.g. 12 for builds and 4 for edits) and dedupe by `seedKey`. Call `provider.generate`, then `warm`. Insert a `project_assets` row (`kind: app_image`, provider URL, prompt). Return `{ key, url, width, height }`. Ask the model for 2× the display size for retina.
- The tool result tells the agent to add the URL to `data/images.ts` under `key`, rather than inlining the URL in screens.
- Add the tool to the Builder (S1/S2) and Editor toolsets. Update the Builder prompt: "Use `image_url` only where imagery adds real value (products, food, places, onboarding). Use icons for UI chrome. Keep prompts concrete: subject, style, background, lighting, consistent across a set."

### 4. Template support
- `data/images.ts` (agent-owned): `export const images = { banana: { uri: "https://…", width: 600, height: 600 }, … } as const;`.
- `lib/images.ts` (platform-owned): `RemoteImage` wrapper over `expo-image` with a themed placeholder, fade-in, `contentFit`, and an error fallback. Screens use `<RemoteImage source={images.banana} />`.
- Contract update (`CONTRACT.md`): image rules, and no raw image URLs in screens (validator warning `image-discipline`).
- Preview runtime: the CSP `img-src` allows `IMAGEKIT_URL_ENDPOINT`'s origin plus our storage origin.

### 5. User-uploaded images in the app
- The Editor gets a tool `use_uploaded_image { assetId, key, width?, height?, removeBackground? }`. It verifies the asset belongs to the project and is `ready`, produces a transformed signed URL (or a public storage URL under an unguessable path), and adds it to the registry, the same way `image_url` does.
- Chat UX: an image attached to a message plus "use this as the hero image" is enough for the agent to call `use_uploaded_image` with that attachment's id.

### 6. Failure handling and regeneration
- If warming fails, retry once with a simplified prompt. If it still fails, return a tool error, and the agent uses a placeholder plus an icon.
- Inspector (Phase 10): when an `Image`/`RemoteImage` is selected, show a **Regenerate image** action. It calls `POST /api/projects/:id/images/regenerate {key, prompt?}` → new `seedKey` (add a salt) → update `data/images.ts` with a deterministic codemod (no LLM).

### 7. Export considerations
- Generated apps keep remote URLs (signed, stable). Optional stretch for Phase 14: `materialize({ bundleImages: true })` downloads them into `assets/images/` and rewrites `data/images.ts` to `require()`, so apps work offline and don't depend on the provider long-term.

## Files to create / change
```
lib/providers/images/{index.ts,types.ts,imagekit.ts,none.ts}
lib/ai/tools/{image-url.ts,use-uploaded-image.ts}
lib/ai/prompts/builder/uiux-guide.md         (+ image guidance)
lib/validate/images.ts                        (no raw URLs in screens)
lib/codemods/imageRegistry.ts                 (update data/images.ts entries)
templates/expo-app/lib/images.ts              (RemoteImage)
templates/expo-app/CONTRACT.md                (+ image rules)
preview-runtime/src/csp.ts                    (+ image origins)
app/api/projects/[projectId]/images/regenerate/route.ts
lib/env.ts                                    (+ IMAGEKIT_URL_ENDPOINT, IMAGEKIT_PRIVATE_KEY, server-only)
```

## Database changes
- Uses `project_assets` with `kind = app_image`. Add nullable columns: `external_url text`, `prompt text`, `provider text`, `provider_ref text`, `seed_key text` (unique per project). This is a small extension to `03-data-model.md` §2.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/images/regenerate` | editor | `{ key, prompt? }` → `{ url, revisionId }`; 429 if the image cap is hit |
| (agent tool) | `image_url` | worker ctx | `{ purpose, prompt, width, height, key }` → `{ key, url, width, height }` |
| (agent tool) | `use_uploaded_image` | worker ctx | `{ assetId, key, width?, height?, removeBackground? }` → `{ key, url }` |

## Acceptance checklist
- [ ] Prompting "a grocery store app for online selling" produces product cards with relevant images (banana, strawberries, milk …) in preview and in Expo Go.
- [ ] Re-running the same image prompt in the same project reuses the URL (no new `project_assets` row, no new generation).
- [ ] With `IMAGEKIT_*` unset, builds still succeed with themed placeholders.
- [ ] A failed generation degrades to a placeholder, with no broken image icons.
- [ ] "Use this photo as the hero image" with an attachment puts that image in the app.
- [ ] Regenerate from the inspector swaps one image without an LLM call.
- [ ] Per-run cap enforced: the 13th `image_url` call in a build returns a tool error.
- [ ] **Security:** `IMAGEKIT_PRIVATE_KEY` appears in no client bundle, Snack payload, export or generated file (grep test on the build output and on a materialized project). Unsigned generation URLs are rejected by the ImageKit account settings (manual check, documented in the PR).
- [ ] **Security:** prompts with newlines, `../`, `?`, `#`, `,` or very long text are sanitized and encoded. A unit test covers each.
- [ ] **AuthZ:** `use_uploaded_image` rejects asset ids from another project. Regenerate requires editor.

## Claude Code prompt
```text
We are building AppBuilding. Implement Phase 12 (AI images via an ImageProvider; ImageKit
adapter first) only.

Read first:
- docs/plan/05-ai-agent-design.md (§3 tools)
- docs/plan/06-generated-app-template.md (ownership; add data/images.ts + lib/images.ts)
- docs/plan/07-preview-runtime.md (§6 CSP)
- docs/plan/phases/phase-12-ai-images.md (this phase)
- ImageKit's CURRENT docs for: AI generation URL syntax, transformation params (resize,
  smart crop, background removal), URL signing, and the "restrict unsigned URLs" setting.
  Quote the doc section you followed in the PR description. Don't guess the syntax.

Implement, one commit per phase-doc step:
1. ImageProvider interface + "none" placeholder adapter.
2. ImageKit adapter: sanitize/encode prompt, seedKey dedupe, server-side signing, warm().
3. image_url agent tool (+ per-run cap) wired into Builder and Editor; prompt guidance.
4. Template: data/images.ts convention, lib/images.ts RemoteImage, contract + validator rule,
   preview CSP img-src.
5. use_uploaded_image tool for user uploads.
6. Regenerate endpoint + deterministic data/images.ts codemod.

Rules: IMAGEKIT_PRIVATE_KEY is server/worker-only (enforce in lib/env.ts). Generated apps get
only signed URLs. Every route uses requireProjectAccess().
Do NOT build: bundling images into exports (stretch, Phase 14), billing for images (Phase 18 —
but record image generations so they can be priced later).
Definition of done = Acceptance checklist. Finish with lint, typecheck, tests, summary.
```

## Pitfalls & notes
- **Cost visibility:** each unique generation costs money on the provider side. Log every generation (`project_assets` rows with `provider_ref`), and include image counts in eval output and later in credit pricing (`11-costs-and-pricing.md`).
- **Latency:** first-fetch generation can take several seconds. Warm in parallel (`p-limit(4)`) during S2 so it overlaps with code generation.
- **Consistency across a set:** product shots look better with a shared style suffix ("studio product photo, soft shadow, white background"). Let the Builder define one `imageStyle` in S1 and reuse it.
- **Content safety:** generation prompts come from the model, which is steered by user prompts. Keep the provider's safety filters on. If the provider supports a negative prompt, use it for text and logos.
- **Copyright/brand:** tell the agent not to request real brand logos or real people. Use generic products.
- **Alternatives:** a direct text-to-image API with results stored in our bucket (more control, no URL-generation abuse surface), Cloudinary (similar transformations), or curated stock (Unsplash API; attribution rules apply). All fit the same interface.
