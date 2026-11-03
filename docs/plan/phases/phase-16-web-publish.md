# Phase 16 — Publish to the Web (Expo Web Export + Hosting)

| | |
|---|---|
| Video chapter(s) | Ch.18 "Publish App to the Web" (02:30:02) |
| Branch | `feat/16-web-publish` |
| Depends on | Phase 05 (jobs), Phase 08 (materialize), Phase 15 (Expo token connect) |
| Effort (solo dev + Claude Code) | ~3 days |
| New env vars | none (uses the user's Expo token); optional `PUBLISH_HOSTING_PROVIDER` if you add a non-EAS host. If you add it, also add it to `04-tech-stack.md`'s env list. |

## Goal
**Publish → Web** turns the generated Expo app into a website, using react-native-web via `expo export -p web`, and deploys it to a public URL anyone can open and share. Republishing updates the same URL. The deploy runs in the background and its progress stays visible after the dialog is closed. Web publishing is available on every plan, as in the video.

## What we do differently from the video (and why)
- **Export in our worker, then upload.** The video's flow waited on Expo's build queue. Their screenshot showed a ~12.7 minute wait on the free plan, and the first attempt failed with "no dist folder found" because the job didn't export first. We run `npx expo export -p web` ourselves (about a minute, no queue) and then `eas deploy` only uploads the static `dist/` to EAS Hosting.
- **Cached dependencies per template version.** `package.json` is platform-owned and identical for every project on a template version (see `06-generated-app-template.md` §3). The worker image ships a pre-installed `node_modules` per template version, so each export copies or symlinks it instead of running `npm install`. Fast, and no install scripts from project code.
- **Hosting behind `HostingProvider`.** EAS Hosting is the default because the user already connected Expo in Phase 15. A provider interface keeps the option to host on Cloudflare Pages or our own static bucket later.
- **We record every deployment** (`web_deployments`), so the dialog can show history, the live URL, and "republish".

## Steps

### 1. `HostingProvider` interface
```ts
// lib/providers/hosting/types.ts
export interface HostingProvider {
  id: "eas_hosting" | "static_bucket" | "cloudflare_pages";
  deploy(ctx: HostCtx, input: { distDir: string; projectSlug: string; production: boolean }):
    Promise<{ url: string; deploymentId: string; alias?: string }>;
  status?(ctx: HostCtx, deploymentId: string): Promise<"live" | "failed" | "pending">;
}
```
Default: `eas_hosting` (uses the user's Expo token and the project's EAS project id from Phase 15).

### 2. Web config in the template
- `app.json` (rendered): `"web": { "output": "static" | "single", "bundler": "metro", "favicon": … }`. Pick `single` (SPA) unless static rendering works with all generated routes. Test with the fixture app.
- Make sure every allowlisted module has a working web implementation (already required by the preview runtime, `07-preview-runtime.md`).
- Platform-specific code must be guarded with `Platform.OS === "web"` checks. Add a validator warning for known native-only APIs (haptics, etc.) used without guards.

### 3. Worker job `publish.web`
`worker/handlers/publish-web.ts`:
1. Ensure the Expo connection + EAS project exist (reuse Phase 15 linking).
2. Materialize the current snapshot into `mkdtemp()`; secret scan.
3. Link the cached `node_modules` for `project.template_version`.
4. `npx expo export -p web --output-dir dist` with a minimal env (no platform secrets), timeout 5 min, captured logs → `run_events` (`publish.step`: exporting / uploading / deploying).
5. Verify `dist/index.html` exists, otherwise fail with the export log tail.
6. `hosting.deploy({ distDir, production: true })` → for EAS: `eas deploy --prod --non-interactive --json` (or the equivalent flags for the pinned `eas-cli`; **verify at implementation**), `EXPO_TOKEN` in env.
7. Insert a `web_deployments` row (`status = 'live'`, `url`, `alias`, `snapshot_id`); `runs.output = { url }`.
8. Clean up the temp dir.

### 4. Publish dialog → Web tab
- Initial state: description + **Publish to web** button (requires Expo connected; otherwise show the Connect Expo step from Phase 15).
- In progress: step list (Exporting → Uploading → Deploying) driven by run events. Closing the dialog doesn't cancel; the header shows "Publishing web…".
- Done: public URL with Copy / Open buttons, "Last published 3 min ago from snapshot …", **Publish update** button, small history list.
- If the project has a connected backend (Phase 13), remind the user that the web app uses **live** data and that auth redirect URLs must include the published URL (link to the social-auth checklist).

### 5. Re-publish + aliases
- Republish creates a new deployment and moves the production alias/URL to it. The URL stays the same.
- Keep the last N deployments listed; "Roll back" re-promotes an earlier deployment if the host supports it (EAS Hosting supports promoting deployments; verify).

### 6. Abuse and terms
- With EAS Hosting the site lives on the **user's** Expo account and Expo's terms apply. Our ToS still needs to forbid phishing/illegal content generated with the platform.
- If we ever host ourselves (`static_bucket`): use a separate registrable domain for user content (e.g. `*.apps-<brand>usercontent.com`) so user sites can't read platform cookies; add a "Report this app" link and a takedown runbook; rate-limit publishes per user; scan for credential-form phishing patterns before going live.

## Files to create / change
- `lib/providers/hosting/{types.ts,eas-hosting.ts,index.ts}`
- `worker/handlers/publish-web.ts`
- `worker/Dockerfile` (pre-installed `node_modules` per template version under `/opt/template-deps/<version>`)
- `templates/expo-app/app.json.hbs` (web section)
- `lib/validate/platform-guards.ts` (warning rule)
- `app/api/projects/[projectId]/publish/web/route.ts`
- `components/workspace/publish/WebTab.tsx`, header status chip

## Database changes
- `web_deployments`: `id`, `project_id`, `provider`, `url`, `alias`, `status`, `snapshot_id`. Add `log_tail text` for failures.
- New `runs.kind` value in use: `publish_web`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/projects/:id/publish/web` | editor (owner by default) | `{}` → `{ runId }` |
| GET | `/api/projects/:id/publish/web` | viewer | → `{ current: {url, publishedAt, snapshotId} \| null, history: WebDeployment[] }` |
| POST | `/api/projects/:id/publish/web/:deploymentId/promote` | owner | → `{ url }` |

## Acceptance checklist
- [ ] Publishing the fixture app yields a public URL that renders the app on desktop and mobile browsers, with working navigation.
- [ ] A theme change + "Publish update" shows the new theme at the same URL.
- [ ] Closing the dialog mid-publish doesn't stop it; reopening shows the current step; the header chip shows progress.
- [ ] Export failure shows the last ~50 log lines and a "Fix with AI" action (feeds the log to the Editor).
- [ ] Export runs without network installs (cached deps) and without platform secrets in the child env (tested).
- [ ] Viewers can see the URL but can't publish; non-members get 403/404.
- [ ] Without Expo connected, the dialog guides the user to connect instead of failing.

## Claude Code prompt
```text
We're implementing Phase 16 (publish generated app to the web) of AppBuilding.

Read first:
- docs/plan/phases/phase-16-web-publish.md (scope + acceptance checklist)
- docs/plan/phases/phase-15-native-builds.md (Expo token + EAS project linking — reuse it)
- docs/plan/06-generated-app-template.md §3, §4, §10
- docs/plan/08-security.md (untrusted code execution, user-content hosting)

Verify against the pinned expo / eas-cli versions: `npx expo export -p web` flags and output dir,
app.json web.output options, `eas deploy` non-interactive production deploy + JSON output, aliases/promote.
Record findings in lib/providers/hosting/README.md.

Implement:
1. HostingProvider interface with an eas_hosting implementation.
2. Worker image: pre-installed node_modules per template version; publish.web job that materializes
   into a temp dir, links cached deps, exports web, checks dist/index.html, deploys, records
   web_deployments, streams step events, always cleans up. Child env: EXPO_TOKEN only (+PATH/HOME).
3. platform-guards validator warning for unguarded native-only APIs.
4. Publish dialog Web tab: publish, progress, URL copy/open, history, publish update, promote.

Definition of done = Acceptance checklist; tests for env allowlist, missing-dist failure path, authz.
Mock the HostingProvider in tests.

Do NOT build: custom domains for user apps, self-hosted static hosting, billing gates.
Finish by running lint, typecheck, tests; summarize changes and any manual setup.
```

## Pitfalls & notes
- **"No dist folder found"** means the deploy step ran without an export. Always export first and check the output before uploading.
- **SPA routing.** With `output: "single"`, deep links like `/habit/123` must fall back to `index.html`. Check that the host handles it (EAS Hosting does for single-page output; verify), or use `static` output.
- **Expo free-tier limits** apply to EAS Hosting on the user's account (deployments, bandwidth). Surface Expo's error message verbatim.
- **Fonts and images.** Web export needs assets reachable at build time. Phase 12 image URLs are remote, which is fine. Bundled fonts must be in `assets/`.
- **Backend auth on web.** OAuth redirect URIs for the published URL must be added in the connected backend (Phase 13 checklist), or web sign-in fails while native works.
- **Template upgrades.** Cached `node_modules` are keyed by template version. Projects on old versions keep working until they're migrated.
