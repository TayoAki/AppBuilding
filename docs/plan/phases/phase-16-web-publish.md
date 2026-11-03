# Phase 16 — Publish to the Web (Expo Web Export + EAS Hosting)

| | |
|---|---|
| Video chapter(s) | Ch.18 "Publish App to the Web" (02:30:02) |
| Branch | `feat/16-web-publish` |
| Depends on | Phase 05 (jobs, outbox, project stream), Phase 08 (materialize, sandbox service), Phase 14 (secret scanner). **Self-sufficient for the MVP:** it builds the Expo connect, EAS project init and identity steps that Phase 15 reuses later (ADR-003 #8) |
| Effort (solo dev + Claude Code) | ~4–4.5 days (includes ~1–1.5 d of Expo connect, EAS init and identity moved here from Phase 15) |
| New env vars | none (uses the owner's Expo token); optional `PUBLISH_HOSTING_PROVIDER` (default `eas`) |

## Goal
**Publish → Web** turns the generated Expo app into a website, using react-native-web via `expo export -p web`, and deploys it to a public URL anyone can open and share. Republishing updates the same URL, and a failed update leaves the previous version online. The deploy runs in the background and its progress stays visible after the dialog is closed. The live URL also appears in the App Preview tab. Web publishing is available on every plan, and **only the project owner** can publish (`08` §2, ADR-003 #3).

## What we do differently from the video (and why)
- **Export in our sandbox, then upload.** The video waited on Expo's build queue (~12.7 min on the free plan) and first failed with "no dist folder found". We run `npx expo export -p web` in the **sandbox service** (ADR-003 #18: untrusted generated code, no platform secrets), which takes about a minute with no queue. Then `eas deploy` only uploads `dist/`.
- **Cached dependencies per template version.** `package.json` is platform-owned, so the sandbox links a prebuilt `node_modules` per template version. That makes it fast, and no install scripts from project code ever run.
- **Verified facts** (`15-vendor-facts.md` §1, §4):
  - **Set `web.output: "single"` explicitly.** SDK 58 Expo Router defaults to `"server"`, which renders HTML on each request and wouldn't fit the free hosting tier's 10 CPU-ms per request.
  - Pass **`--dev-domain <slug>`**: non-interactive deploys auto-pick a subdomain and error if it's taken.
  - EAS Hosting's free plan has 100k requests/month, 1 GB, 5 aliases, 30-day deployment retention and no custom domain, and usage stops at the limit.
- **Hosting behind `HostingProvider`**, so Cloudflare Pages or our own static hosting can be added later.
- **One `web_deployments` row per project** (R19). `url` is the **live URL** and changes only when a publish **succeeds**.

### Reference app notes
- **R19:** one row per project; `live_url` changes only on success; the hosting name is fixed on first publish (our `--dev-domain`); `web.output: "single"`; the app's `apiUrl` comes from `NEXT_PUBLIC_APP_URL`, so publish from a deployed platform if the app has a backend.
- The reference app ran export + deploy as an **EAS workflow `deploy` job** on Expo's machines, through an undocumented API. Trade-off:

| | Export in our sandbox (default) | EAS workflow deploy job (reference) |
|---|---|---|
| Speed | ~1 min, no queue | EAS queue (~12.7 min seen on free tier) |
| Where generated code runs | our sandbox service | Expo's machines |
| Our infra | sandbox needs cached deps + disk | nothing beyond an API call |
| API surface | documented CLI (`expo export`, `eas deploy`) | workflow dispatch (undocumented in the reference app) |

## Steps

### 1. Expo connect (shared with Phase 15)
- Settings / Publish dialog: **Connect Expo** → instructions (expo.dev → Account settings → Access tokens → create) → paste the token.
- `POST /api/expo/token` (session) → the sandbox validates it (`eas whoami --json` with the token only) → stored encrypted as `oauth_connections(provider = "expo")`, with `external_login` = Expo username.
- Show the connected account and **Disconnect** (deletes our row; tell the user to revoke the token on expo.dev).

### 2. EAS project + identity, set once (shared with Phase 15)
- `ensureIdentity(projectId)` on the first publish or build: `projects.slug` = `<kebab app name>-<6 chars of project id>`, `android_package` = `com.<expoUsername>.<slug without dashes>`, `bundle_id` = the same. They're shown in the dialog and editable until the first store build (Phase 15 locks package and bundle id; `slug` is locked after the first publish too, because it's the hosting name).
- `ensureEasProject(projectId)`: the sandbox runs `eas init --non-interactive --force` (verify flags) with the **owner's** token and `owner` set in `app.json` (required for robot tokens, `15` §2). Store `projects.eas_project_id`; it's rendered into `app.json` `extra.eas.projectId`.
- Phase 15 adds `ensureEasWebhook`. Web publish doesn't need webhooks.

### 3. `HostingProvider` interface
```ts
export interface HostingProvider {
  id: "eas_hosting" | "static_bucket" | "cloudflare_pages";
  deploy(ctx: HostCtx, input: { distTarballUrl: string; devDomain: string; production: boolean }):
    Promise<{ url: string; deploymentId: string; alias?: string }>;
  promote?(ctx: HostCtx, deploymentId: string): Promise<{ url: string }>;
}
```

### 4. Job `publish.web` (single-shot wrapper)
`worker/handlers/publish-web.ts`:
1. Owner check; `ensureIdentity`; `ensureEasProject`.
2. If the project has a connected backend and `NEXT_PUBLIC_APP_URL` is a localhost/private URL, fail fast: "Publish from your deployed AppBuilding. This app's data API isn't reachable from the internet."
3. `materialize({ target: "web" })` (static `app.json` with **`web.output: "single"`**) → tarball → signed URL; secret scan.
4. Sandbox `POST /jobs/web-publish` (HMAC per-job token) `{tarballUrl, templateVersion, devDomain: slug, expoToken}` →
   - unpack, link cached `node_modules`;
   - `npx expo export --platform web --output-dir dist` (env allowlist, no platform secrets, 5 min timeout); check that `dist/index.html` exists;
   - `EXPO_TOKEN=… eas deploy --prod --non-interactive --json --export-dir dist --dev-domain <slug>`. If the dev domain is taken, return `DEV_DOMAIN_TAKEN`; the worker then retries once with `<slug>-<4 chars>` and stores that as the slug;
   - stream step logs back. The worker turns them into `publish.step` events (exporting / uploading / deploying) on the **project-scoped stream**.
5. Upsert the project's single `web_deployments` row:
   - on success: `status = live`, `url`, `alias`, `snapshot_id`, `deployment_id`, clear `log_tail`;
   - on failure: `status = failed` + `log_tail`, leaving `url`/`snapshot_id` **untouched**, so the old version stays online.
   - `runs.output = { url }`.

### 5. Publish dialog → Web tab
- Initial state: description + **Publish to web** (owner only; editors see "Ask the owner to publish"). Without an Expo connection, show the Connect Expo step.
- In progress: step list driven by `publish.step` events. Closing the dialog doesn't cancel; the header shows "Publishing web…".
- Done: public URL with Copy / Open, "Last published 3 min ago", **Publish update**, history, and **Roll back** (re-promote an earlier deployment within EAS Hosting's 30-day retention; verify the promote command/flags in S5).
- The live URL is also shown in the App Preview tab.
- Free-tier notice: "Hosted on your Expo account's free plan: 100k requests/month; the site stops serving when the limit is hit."
- Backend-connected apps: remind the owner that the web app uses **live** data, and that auth redirect URLs must include the published URL (Phase 13 checklist).

### 6. Abuse and terms
- With EAS Hosting the site lives on the **owner's** Expo account and Expo's terms apply. Our ToS still forbids phishing and illegal content.
- If we ever host ourselves (`static_bucket`): use a separate registrable domain for user content, add a report link and a takedown runbook, rate-limit publishes, and scan for credential-form phishing.

## Files to create / change
- `lib/providers/hosting/{types.ts,eas-hosting.ts,index.ts,README.md}`
- `lib/providers/expo/{connect.ts,identity.ts,eas-project.ts}` (shared with Phase 15)
- `worker/handlers/publish-web.ts`
- `sandbox/jobs/{web-publish.ts,eas-whoami.ts,eas-init.ts}`, sandbox image with `/opt/template-deps/<version>`
- `templates/expo-app/app.json.hbs` (web section, `owner`, `extra.eas.projectId`)
- `lib/validate/platform-guards.ts` (warning for unguarded native-only APIs)
- `app/api/expo/token/route.ts`, `app/api/projects/[projectId]/publish/web/route.ts`
- `components/workspace/publish/{PublishDialog.tsx,WebTab.tsx,ConnectExpo.tsx,IdentityFields.tsx}`, header status chip

## Database changes
- `oauth_connections` rows with `provider = 'expo'`.
- `projects.slug`, `android_package`, `bundle_id`, `eas_project_id` (`03` §2).
- `web_deployments`: one row per project (**unique `project_id`**): `id`, `project_id`, `provider`, `url`, `alias`, `deployment_id`, `status` (latest attempt), `snapshot_id` (live version), `log_tail`.
- `runs.kind = publish_web`, with `created_by` / `billed_user_id`, created through the `job_outbox` (ADR-003 #7).

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/expo/token` | session | `{ token }` → `{ connected: true, username }` |
| DELETE | `/api/expo/token` | session | → `{ ok }` |
| GET/PATCH | `/api/projects/:id/identity` | viewer / owner | → `{ slug, androidPackage, bundleId, locked }` / edit before lock |
| POST | `/api/projects/:id/publish/web` | **owner** | `{}` → `{ runId }` |
| GET | `/api/projects/:id/publish/web` | viewer | → `{ live: {url, publishedAt, snapshotId} \| null, latestAttempt, history }` |
| POST | `/api/projects/:id/publish/web/:deploymentId/promote` | owner | → `{ url }` |

## Acceptance checklist
- [ ] On a fresh test Expo account: connect → first publish sets the identity, inits the EAS project and deploys to `https://<slug>.expo.app`, with no interactive prompts.
- [ ] A theme change + **Publish update** shows the new theme at the same URL.
- [ ] A deliberately broken update fails with the log tail, and the previous version is still served at the unchanged URL.
- [ ] A taken dev domain is handled (retry with a suffix, slug updated before lock).
- [ ] Rendered `app.json` has `web.output: "single"` (test), and deep links fall back to `index.html` on the host.
- [ ] Publishing a backend-connected app from a localhost platform is refused with a clear message.
- [ ] Closing the dialog mid-publish doesn't stop it. Reopening shows the current step; another open tab sees the steps on the project stream.
- [ ] An export failure shows the last ~50 log lines and **Fix with AI**, which sends the log to the Planner as a chat message.
- [ ] Export runs in the sandbox without network installs and with only allowlisted env vars (tested).
- [ ] **AuthZ:** editors and viewers can see the URL but can't publish (403); non-members get 404.

## Claude Code prompt
```text
We're implementing Phase 16 (publish generated app to the web) of AppBuilding. For the MVP this phase
also builds Expo connect, EAS project init and identity, which Phase 15 reuses.

Read first:
- docs/plan/phases/phase-16-web-publish.md (scope + acceptance checklist)
- docs/plan/15-vendor-facts.md §1 and §4 (web.output default "server" on SDK 58, --dev-domain, limits)
- docs/plan/12-decisions-and-open-questions.md ADR-003 (#3 owner-only, #7 outbox, #8, #18 sandbox)
- docs/plan/06-generated-app-template.md §3, §4, §10
- docs/plan/08-security.md (§2 matrix, §5 untrusted code, user-content hosting)

Verify against the pinned expo / eas-cli: `expo export --platform web` flags, `eas init --non-interactive`,
`eas deploy --prod --non-interactive --json --export-dir --dev-domain`, promote/rollback. Record the
findings in lib/providers/hosting/README.md.

Implement:
1. Expo connect (validate in the sandbox, encrypt, oauth_connections 'expo').
2. ensureIdentity (slug with project-id suffix; package/bundle id editable until locked) +
   ensureEasProject (owner set in app.json).
3. HostingProvider + eas_hosting.
4. publish.web: materialize("web", web.output "single") → tarball → sandbox job (cached deps,
   expo export, dist check, eas deploy with --dev-domain) → the ONE web_deployments row (url changes
   only on success) → publish.step events on the project stream. Refuse localhost + backend.
5. Web tab UI (owner-only publish, progress, URL, history, roll back, free-tier notice).
Tests: env allowlist, missing-dist failure, dev-domain-taken retry, web.output "single" in the rendered
app.json, authz. Mock the HostingProvider and the sandbox in tests.
Do NOT build: native builds (Phase 15), custom domains, self-hosted static hosting, billing gates.
Finish with lint, typecheck, tests; summarize and list manual setup (test Expo account).
```

## Pitfalls & notes
- **"No dist folder found"** means the deploy ran without an export. Always export first and check `dist/index.html`.
- **SDK 58's `web.output` default is `"server"`.** Forgetting to set `"single"` produces server-rendered output that the free tier can't run. Snapshot-test the rendered `app.json`.
- **SPA deep links** like `/habit/123` must fall back to `index.html` (S5 verifies this on EAS Hosting).
- **Expo free-tier limits** stop the site at 100k requests/month on the owner's account. Surface Expo's message verbatim.
- **Backend auth on web:** the published URL must be in the backend's OAuth redirect list (Phase 13).
- **Template upgrades:** cached `node_modules` are keyed by template version (Phase 19b handles upgrades).
