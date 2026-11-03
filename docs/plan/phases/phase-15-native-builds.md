# Phase 15 — Native Builds (Android APK/AAB, iOS IPA via EAS Build)

| | |
|---|---|
| Video chapter(s) | Ch.17 "Build Android & iOS App Files" (02:23:32) |
| Branch | `feat/15-native-builds` |
| Depends on | Phase 16 (Expo connect, EAS project init, identity: built there for the MVP and reused here), Phase 08 (materialize), Phase 14 (secret scanner), sandbox service (Phase 08) |
| Effort (solo dev + Claude Code) | ~4–6 days (Android ~2–3, iOS ~2–3 depending on Apple account access). Expo connect, EAS init and identity moved to Phase 16 (ADR-003 #8) |
| New env vars | none (per-user Expo tokens and per-project webhook secrets live encrypted in the DB; the global `EAS_WEBHOOK_SECRET` is dropped, ADR-003 #17) |

## Goal
From **Publish → App Store**, the **project owner** picks an artifact: Android **APK** for testing, Android **AAB** for Google Play, or iOS **IPA** for App Store/TestFlight. The platform starts an **EAS Build** on the owner's Expo account and shows live status. When the build finishes, the owner downloads the file. Signing credentials are created once and reused, so later builds can update the same store listing.

## What we do differently from the video (and why)
- **EAS Build from the start, not GitHub Actions.** The video first tried GitHub Actions, which needs a default-branch workflow per repo, macOS runners for iOS and fiddly keystores in CI, and then switched to Expo. We skip the detour.
- **Owner-only, owner's Expo account** (ADR-003 #3). One `projects.eas_project_id` lives on the owner's account, and editors can't start builds. The `08` §2 matrix is authoritative.
- **Verified non-interactive facts** (`15-vendor-facts.md` §2–§3, eas-cli 27.3.0):
  - EAS **generates an Android keystore in `--non-interactive` builds** when none exists.
  - **`EAS_NO_VCS=1`** uploads without git, instead of `git init`.
  - iOS needs a **distribution certificate that already exists**; a missing one fails with `MissingCredentialsNonInteractiveError`.
  - Webhooks are **per EAS project**, signed with **HMAC-SHA1** in `expo-signature: sha1=<hex>`.
- **Webhook payloads are hints.** Each project has its own encrypted webhook secret. Every payload triggers a **re-fetch of the build with the owner's token** before anything is written, so a leaked or forged payload can't inject an artifact URL.
- **Untrusted project code runs in the sandbox service** (ADR-003 #18). `eas-cli` reads the materialized project there, with the owner's `EXPO_TOKEN` passed per job, used only for the command's lifetime and never stored in the sandbox. Compilation happens on Expo's servers.

### Reference app notes
- **R18:** identity (slug, `android.package`, `ios.bundleIdentifier`) is set **once** with a project-id suffix; `appVersionSource: remote`. Both now live in Phase 16 for the MVP and are reused here.
- **R18:** "EAS requires a git repository… stdin is not readable." The reference app hit this. We set `EAS_NO_VCS=1` (verified), so there's no `git init`.
- **R18:** if EAS has no Apple credentials for the bundle id, the dialog shows **setup steps instead of starting a build**. Kept, and narrowed by the verified fact that it's specifically the distribution certificate.
- The reference app's `ensureAndroidKeystore` had Expo generate the keystore. Verified: non-interactive builds do this themselves.
- The reference app dispatched EAS workflows through an **undocumented GraphQL API** and **polled only while the dialog was open**. We use the documented `eas-cli` + per-project webhooks + a polling fallback (`13` §3), so builds progress with the tab closed.

## Steps

### 1. Prerequisites from Phase 16 (already built for the MVP) + the webhook
- Expo token connect (`oauth_connections.provider = "expo"`, owner's token, validated, encrypted).
- `ensureEasProject(projectId)`: links or creates the EAS project non-interactively under the owner's account. `app.json` carries `owner` (required for robot tokens) and `extra.eas.projectId`.
- **New in this phase:** `ensureEasWebhook(projectId)`, run once per EAS project before its first build. The sandbox runs `eas webhook:create --event BUILD --url <APP>/api/webhooks/eas/<projectId> --secret <random ≥ 32 chars>` (≥ 16 required) with the owner's token, and the secret is stored encrypted in `projects.eas_webhook_secret_enc` (plus `eas_webhook_id`).
- Identity set once: `projects.slug`, `android_package`, `bundle_id`, editable until the first store build and then locked.

### 2. `eas.json` (platform-rendered)
```jsonc
{
  "cli": { "appVersionSource": "remote" },
  "build": {
    "preview":    { "android": { "buildType": "apk" }, "distribution": "internal" },
    "production": { "android": { "buildType": "app-bundle" }, "autoIncrement": true }
  }
}
```
- `appVersionSource: "remote"` + `autoIncrement` lets EAS own `versionCode`/`buildNumber`. The user-facing `version` (e.g. `1.0.0`) is editable in the dialog.
- Static `app.json` only (no `app.config.js`), so the CLI never executes project-provided config code.

### 3. Android signing (`ensureAndroidKeystore`)
| | **A. EAS-generated (recommended, verified)** | **B. We generate it (fallback)** |
|---|---|---|
| How | Credentials stay `remote`. On the first non-interactive build, EAS generates the keystore itself (`generateRandomKeystoreAsync`). | `keytool -genkeypair -v -storetype JKS -keyalg RSA -keysize 2048 -validity 10000 …`, passed via `credentials.json` + `credentialsSource: "local"` |
| When to use | Default | Only if S4 shows A failing for the pinned eas-cli |
| Secrets we hold | none | keystore + passwords (encrypted `project_signing`, separate backup) |
| Portability | owner downloads it from expo.dev | "Download signing key" (owner + re-auth, audit-logged) |

Rule for both: never replace an existing keystore. After the first build, record the keystore's SHA-1 fingerprint (from `eas credentials` JSON or the build metadata) in `project_signing.android_fingerprint`, and fail loudly if a later build reports a different one.

### 4. iOS credentials
- Needs Apple Developer Program membership and an **App Store Connect API key** (`.p8`, key id, issuer id), stored encrypted in `project_signing` (`ios_asc_key_enc`, `ios_asc_key_id`, `ios_asc_issuer_id`, `apple_team_id`, `apple_team_type`).
- For the build, write the key to a temp file in the sandbox and set `EXPO_ASC_API_KEY_PATH`, `EXPO_ASC_KEY_ID`, `EXPO_ASC_ISSUER_ID` (omit for individual keys), `EXPO_APPLE_TEAM_ID`, `EXPO_APPLE_TEAM_TYPE` (`INDIVIDUAL` / `COMPANY_OR_ORGANIZATION` / `IN_HOUSE`). With these, EAS creates or repairs **provisioning profiles** non-interactively.
- The **distribution certificate isn't created non-interactively.** Before allowing IPA, check whether EAS has one for the team (`eas credentials` JSON / build dry check). If not, the dialog shows **setup steps instead of starting a build**: run `eas credentials -p ios` once on their machine (we show the exact command with their project id), or upload an existing `.p12` + password via expo.dev. The IPA option stays disabled until it's resolved.
- Internal-distribution IPAs also need registered device UDIDs. Document this rather than hide it.

### 5. Build job `build.native` (single-shot wrapper; the run ends at submit)
Worker `worker/handlers/build-native.ts`:
1. Gate: owner check + `gate.canNativeBuild(owner)` (Pro only from Phase 18).
2. `materialize({ target: "eas" })` of the current snapshot → tarball in storage (signed URL), with a secret scan (Phase 14).
3. Sandbox `POST /jobs/eas-build` (HMAC per-job token) with `{tarballUrl, platform, profile, expoToken, ascKey?}`. The sandbox unpacks to a temp dir and runs:
   `EAS_NO_VCS=1 EXPO_TOKEN=… eas build --platform <android|ios> --profile <preview|production> --non-interactive --no-wait --json`
   with an explicit env allowlist (PATH, HOME, `EAS_NO_VCS`, `EXPO_TOKEN`, iOS ASC vars). It returns the parsed build id, then deletes the temp dir and key files.
4. Insert `native_builds` (`status = queued`, `eas_build_id`, `logs_url`). **The run succeeds here** (`runs.output = {buildId}`); build progress lives in `native_builds`, not in the run (ADR-003 #2, #17).

### 6. Status updates: per-project webhook + polling fallback
- `POST /api/webhooks/eas/:projectId`:
  - verify `expo-signature` = `sha1=` + HMAC-SHA1(raw body, project's secret), with a constant-time compare (401 on mismatch);
  - dedupe through `processed_webhook_events(provider='eas', event_id = payload.id + status)`;
  - **re-fetch** the build with the owner's token (`eas build:view <id> --json` via the sandbox, or the EAS API), and only then update `native_builds` (`status` building/finished/errored/canceled, `artifact_url` from the re-fetched `artifacts.buildUrl`, `error`);
  - emit `build.status` on the project-scoped stream;
  - return 2xx fast (EAS retries non-2xx with back-off).
- Fallback: a pg-boss scheduled job polls active builds every 60 s (re-fetch only), for missed webhooks.

### 7. Publish dialog → App Store tab
- Choices: Android APK (testing), Android AAB (Play Store), iOS IPA (App Store). Version field; package and bundle id (locked after the first store build).
- Build history: platform, version, status chip, started/finished, **Download** (artifact URL), **Logs**.
- Builds continue when the dialog is closed; the header shows "Building Android…". Free-tier EAS queues are shown honestly ("Queued on Expo — free tier queues can take 10+ minutes").

## Files to create / change
- `lib/providers/build/{types.ts,eas.ts,ios-credentials.ts,android-signing.ts}` (+ `README.md` with the verified facts)
- `worker/handlers/{build-native.ts,eas-poll.ts}`
- `sandbox/jobs/eas-build.ts`, `sandbox/jobs/eas-view.ts`
- `app/api/projects/[projectId]/builds/route.ts`, `app/api/webhooks/eas/[projectId]/route.ts`
- `templates/expo-app/eas.json.hbs`
- `lib/credits/gate.ts` (`canNativeBuild`)
- `components/workspace/publish/{AppStoreTab.tsx,BuildHistory.tsx,IosSetupSteps.tsx}`
- Sandbox image: Node 22 + pinned `eas-cli` (+ JDK only if option B is used)

## Database changes
- `native_builds`: `id`, `project_id`, `platform`, `artifact`, `version`, `build_number`, `eas_build_id`, `status` (`queued` · `building` · `finished` · `errored` · `canceled`), `artifact_url`, `logs_url`, `error`, `requested_by`.
- `project_signing`: Android option-B fields; `android_fingerprint`; iOS ASC fields + `apple_team_type`.
- `projects.eas_webhook_id`, `projects.eas_webhook_secret_enc` (created by `ensureEasWebhook` in this phase). **Add to `03`.**
- `processed_webhook_events` (ADR-003 #6). `runs.kind = build_native`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| GET | `/api/projects/:id/builds` | viewer | → `{ builds: NativeBuild[] }` (no credentials) |
| POST | `/api/projects/:id/builds` | **owner** + plan gate | `{ platform, artifact, version }` → `{ runId }` |
| POST | `/api/projects/:id/signing/ios` | owner | `{ ascKey(p8), keyId, issuerId?, teamId, teamType }` → `{ ok, distributionCertFound }` |
| GET | `/api/projects/:id/signing/android` | owner + recent re-auth | → keystore download (option B; audit-logged) |
| POST | `/api/webhooks/eas/:projectId` | per-project HMAC-SHA1 | EAS payload → 2xx |

## Acceptance checklist
- [ ] The first Android APK build on a fresh test Expo account succeeds non-interactively (EAS generates the keystore), and status updates to finished with a working download link, **with the browser tab closed**.
- [ ] The build upload never prompts for git (`EAS_NO_VCS=1`); no `git init` code exists.
- [ ] A second build has the same keystore fingerprint and an auto-incremented `versionCode`. An AAB build succeeds with the same keystore.
- [ ] Webhook: a bad signature → 401. A valid signature with a forged `artifacts.buildUrl` is ignored, because the artifact URL comes only from the re-fetch. A duplicate delivery is a no-op.
- [ ] With the webhook disabled, the polling fallback completes the status.
- [ ] The sandbox child env contains only the allowlisted vars; a test asserts no platform secrets are passed. Temp dirs and key files are deleted after success and failure.
- [ ] With no distribution certificate on EAS, choosing IPA shows setup steps (with the exact `eas credentials -p ios` command) instead of starting a build. With one, the IPA build succeeds using the ASC key.
- [ ] **AuthZ:** editors and viewers can't start builds (403); non-members get 404; only the owner can download signing keys.

## Claude Code prompt
```text
We're implementing Phase 15 (native builds via EAS Build) of AppBuilding.

Read first:
- docs/plan/phases/phase-15-native-builds.md (scope + acceptance checklist)
- docs/plan/15-vendor-facts.md §2–§3 (verified eas-cli 27.3.0 facts) and ADR-002/ADR-003 (#2, #3, #17, #18)
- docs/plan/phases/phase-16-web-publish.md (Expo connect, ensureEasProject, ensureEasWebhook, identity: reuse)
- docs/plan/06-generated-app-template.md §10 (materialize "eas" target)
- docs/plan/08-security.md (§2 matrix, §5 untrusted code, §9 webhooks)

Re-check the pinned eas-cli's flags against these facts and note differences in
lib/providers/build/README.md.

Implement:
1. eas.json template (appVersionSource remote, autoIncrement, apk/app-bundle profiles).
2. Android signing: option A (EAS generates the keystore in non-interactive builds) by default;
   option B (keytool + credentials.json local) behind a flag; record the fingerprint; never replace.
3. iOS: ASC key storage + env vars; detect a missing distribution certificate and show setup steps
   instead of building.
4. build.native (single-shot; the run ends at submit): materialize → tarball → sandbox job runs
   `EAS_NO_VCS=1 eas build --non-interactive --no-wait --json` with an env allowlist → native_builds row.
5. Per-project webhook route: HMAC-SHA1 `sha1=<hex>` over the raw body, dedupe via
   processed_webhook_events, re-fetch the build with the owner's token before writing, emit
   build.status; plus the 60 s polling fallback.
6. App Store tab + build history; owner-only start.
Tests: signature verification, forged-payload ignored, env allowlist, keystore fingerprint reuse,
authz. Use a fake BuildProvider in tests; never call EAS in CI.
Do NOT build: store submission (eas submit), billing (Phase 18 adds the Pro gate).
Finish with lint, typecheck, tests; summarize, and list the manual steps (Apple account, test Expo account).
```

## Pitfalls & notes
- **Keystore loss is permanent** for apps on Play without Play App Signing. With option A, EAS holds it (the owner can download it from expo.dev). With option B, encrypt it, back it up separately from the DB, and test restore (Phase 19a). Encourage Play App Signing.
- **Bundle id changes break updates.** Lock them after the first store build.
- **Robot/personal tokens and `owner`.** Non-interactive linking needs `owner` in `app.json` when the token is a robot token or the account has several owners (`15` §2).
- **EXPERIENCE_NOT_FOUND.** The reference app hit this when the EAS project didn't exist under the account yet. `ensureEasProject` (Phase 16) must run before the first build.
- **Fidelity:** native builds use the real expo-router and native tab bars. Small differences from the browser preview are expected.
- **Child process hygiene:** use an explicit `env` object (never `process.env`), a timeout and a max output size. Mask the token in captured output.
