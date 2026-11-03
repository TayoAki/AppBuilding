# Phase 15 — Native Builds (Android APK/AAB, iOS IPA via EAS Build)

| | |
|---|---|
| Video chapter(s) | Ch.17 "Build Android & iOS App Files" (02:23:32) |
| Branch | `feat/15-native-builds` |
| Depends on | Phase 05 (jobs), Phase 08 (materialize), Phase 14 (encrypted connections pattern) |
| Effort (solo dev + Claude Code) | ~5–7 days (Android ~3, iOS ~2–4 depending on Apple account access) |
| New env vars | `EAS_WEBHOOK_SECRET` |

## Goal
From **Publish → App Store**, a user picks an artifact (Android **APK** for testing, Android **AAB** for Google Play, iOS **IPA** for App Store/TestFlight). The platform starts an **EAS Build** on the user's own Expo account and shows live status. When the build finishes, the user downloads the file. Signing credentials are created and stored safely, so later builds can update the same store listing.

## What we do differently from the video (and why)
- **EAS Build from the start, not GitHub Actions.** The video first tried GitHub Actions. Workflows can only be dispatched from a repo's default branch; it needed either a central builder repo or a workflow file in every user repo; macOS runners are needed for iOS; and keystores in CI are fiddly. It failed and switched to Expo's build service. EAS already handles Android/iOS toolchains, signing, artifact hosting and build webhooks for Expo projects, so we skip the detour.
- **We generate the Android keystore ourselves.** Non-interactive EAS builds can't create a new keystore (the video hit this). The worker generates one with `keytool`, encrypts it into `project_signing`, backs it up, and passes it to EAS via `credentials.json` with `credentialsSource: "local"`.
- **The user's Expo account, the user's token.** The user pastes an Expo personal access token once. It's stored encrypted as `oauth_connections(provider = "expo")`. Builds count against **their** EAS quota. No platform-wide Expo token exists, so one leaked token can't touch every user's builds.
- **Honest iOS story.** IPA builds need a paid Apple Developer account and signing credentials. We support the App Store Connect API key path and say up front that iOS needs more setup. The video glosses over this.
- **Builds treat project code as untrusted.** The materialized project runs through `eas-cli` in an **ephemeral container/temp dir** with only the user's Expo token in env, no platform secrets, and no install scripts (`npm ci --ignore-scripts` if dependencies are needed at all). The actual compilation happens on Expo's servers.

## Steps

### 1. Expo token connect
- Settings / Publish dialog: "Connect Expo" → instructions (expo.dev → Account settings → Access tokens → create) → paste token.
- `POST /api/expo/token` → validate by calling the Expo account API (`eas whoami` equivalent) in the worker → store encrypted in `oauth_connections` with `external_login` = Expo username.
- Show the connected account + "Disconnect".

### 2. EAS project linking
- On the first build for a project, the worker runs the EAS project init non-interactively under the user's account (e.g. `eas init --non-interactive --force` / `eas project:init`; **verify flags for the installed `eas-cli` version**). Store the resulting EAS project id in `projects.eas_project_id` and render it into `app.json` (`extra.eas.projectId`) at materialization.
- Slug, bundle identifier and Android package come from project metadata: `com.<expoUsername>.<slug>` by default, editable in the dialog before the first build, then **locked** (changing them breaks store updates).

### 3. `app.json` + `eas.json` generation
Platform-owned templates (see `06-generated-app-template.md` §3, §10):
```jsonc
// eas.json (rendered)
{
  "cli": { "appVersionSource": "remote" },
  "build": {
    "preview":    { "android": { "buildType": "apk", "credentialsSource": "local" }, "distribution": "internal" },
    "production": { "android": { "buildType": "app-bundle", "credentialsSource": "local" },
                    "ios": { "credentialsSource": "remote" }, "autoIncrement": true }
  }
}
```
- `appVersionSource: "remote"` + `autoIncrement` lets EAS manage `versionCode`/`buildNumber`. The user-facing `version` (e.g. `1.0.0`) is editable in the dialog. This is the "automatically detects the version number" behaviour from the video.
- No `app.config.js` ever: static JSON only, so the CLI doesn't execute project-provided JS when reading config.

### 4. Android keystore
- Worker `lib/providers/build/android-keystore.ts`: `keytool -genkeypair -v -storetype JKS -keyalg RSA -keysize 2048 -validity 10000 -alias <alias> …` with random passwords. It runs once per project, on the first Android build.
- Store base64 keystore + passwords encrypted in `project_signing`; copy to the backup bucket/key (see `phase-19-production-deploy.md`).
- At build time write `credentials.json` + the `.jks` file into the temp dir only, and delete the temp dir afterwards.
- "Download signing key" (owner only, re-auth required) lets the user take the key with them if they leave the platform.

### 5. iOS credentials
- Needs: Apple Developer Program membership, team id, and an **App Store Connect API key** (`.p8`, key id, issuer id) that EAS uses to manage certificates/profiles non-interactively. **Verify current EAS docs** for the exact env vars/flags.
- Store the key encrypted (add `ios_asc_key_enc`, `ios_asc_key_id`, `ios_asc_issuer_id`, `apple_team_id` to `project_signing`, or per-user if shared across projects).
- If the user has none: show the checklist and disable the IPA option. Internal-distribution IPAs also need registered device UDIDs. Document this rather than hide it.

### 6. Build job `build.native`
Worker `worker/handlers/build-native.ts`:
1. Gate: `gate.canNativeBuild(user)` (stub returns true; real check in Phase 18).
2. Materialize the current snapshot into `mkdtemp()`; secret scan (Phase 14).
3. Write `app.json`, `eas.json`, signing files.
4. Run `eas build --platform <android|ios> --profile <preview|production> --non-interactive --no-wait --json` with `EXPO_TOKEN=<user token>` as the **only** secret in the child env (verify flags against the pinned `eas-cli`).
5. Parse the JSON for the EAS build id → insert/update `native_builds` (`status = 'queued'`, `eas_build_id`, `logs_url`).
6. Clean up the temp dir.

### 7. Status updates: webhook + polling fallback
- EAS webhooks are configured **per Expo account**. On Expo connect, the worker creates a BUILD webhook on the user's account pointing at `/api/webhooks/eas` with secret `EAS_WEBHOOK_SECRET` (`eas webhook:create …`, non-interactive; verify flags).
- `POST /api/webhooks/eas`: verify the HMAC signature header over the **raw body** (see EAS docs for header name/algorithm), look up `native_builds` by `eas_build_id`, update `status` / `artifact_url` / `error`, append a `run_events` row so open workspaces update.
- Fallback: a pg-boss scheduled job polls active builds (`eas build:view <id> --json`) every 60 s, for when webhook creation failed or events were missed.

### 8. Publish dialog → App Store tab
- Choices: Android APK (testing), Android AAB (Play Store), iOS IPA (App Store). Version field; bundle id/package (locked after first build).
- Build history list: platform, version, status chip, started/finished, **Download** (artifact URL), **Logs**.
- Builds continue when the dialog is closed; the header shows "Building Android…".

## Files to create / change
- `lib/providers/build/{types.ts,eas.ts,android-keystore.ts,ios-credentials.ts}`
- `worker/handlers/{build-native.ts,eas-poll.ts,expo-connect.ts}`
- `app/api/expo/token/route.ts`, `app/api/projects/[projectId]/builds/route.ts`, `app/api/webhooks/eas/route.ts`
- `templates/expo-app/{app.json.hbs,eas.json.hbs}`
- `lib/vfs/materialize.ts` (signing files excluded from exports; `extra.eas.projectId`)
- `lib/credits/gate.ts` (`canNativeBuild` stub)
- `components/workspace/publish/{PublishDialog.tsx,AppStoreTab.tsx,BuildHistory.tsx,ConnectExpo.tsx}`
- Worker Docker image: Node 22 + `eas-cli` (pinned) + JDK (for `keytool`)

## Database changes
- `oauth_connections`: rows with `provider = 'expo'`.
- `native_builds`: `id`, `project_id`, `platform`, `artifact`, `version`, `build_number`, `eas_build_id`, `status`, `artifact_url`, `logs_url`, `error`.
- `project_signing`: Android keystore fields; add iOS ASC key fields (see Step 5).
- **New column, not yet in 03-data-model.md:** `projects.eas_project_id text null`, and `projects.bundle_id` / `projects.android_package` (locked after first store build).
- New `runs.kind` value in use: `build_native`.

## API surface
| Method | Path | Auth | Request → Response |
|---|---|---|---|
| POST | `/api/expo/token` | session | `{ token }` → `{ connected: true, username }` |
| DELETE | `/api/expo/token` | session | → `{ ok }` |
| GET | `/api/projects/:id/builds` | viewer | → `{ builds: NativeBuild[] }` (no credentials) |
| POST | `/api/projects/:id/builds` | owner (or editor) + plan gate | `{ platform, artifact, version }` → `{ runId, buildId }` |
| GET | `/api/projects/:id/signing/android` | owner + recent re-auth | → keystore download (audit-logged) |
| POST | `/api/webhooks/eas` | HMAC signature | EAS payload → 200 |

## Acceptance checklist
- [ ] Connecting an invalid Expo token shows an error; a valid one shows the username.
- [ ] The first Android APK build creates the EAS project, generates and stores a keystore (encrypted), queues the build, and the status updates to finished with a working download link.
- [ ] A second build reuses the same keystore (same SHA-1 fingerprint) and auto-increments `versionCode`.
- [ ] An AAB build succeeds with the same keystore.
- [ ] Webhook with a bad signature → 401; a valid one updates `native_builds` and the open workspace.
- [ ] With the webhook disabled, the polling fallback still completes the status.
- [ ] The child process env for `eas build` contains only `EXPO_TOKEN` (+ PATH/HOME); test asserts no platform secrets are passed.
- [ ] IPA option is disabled with a clear checklist when no Apple credentials are present.
- [ ] Non-owner can't download signing keys; viewers can't start builds.
- [ ] Temp dirs are deleted after success and failure.

## Claude Code prompt
```text
We're implementing Phase 15 (native builds via EAS Build) of AppBuilding.

Read first:
- docs/plan/phases/phase-15-native-builds.md (scope + acceptance checklist)
- docs/plan/06-generated-app-template.md §3 and §10 (platform-owned files, materialize)
- docs/plan/03-data-model.md §7 (oauth_connections, native_builds, project_signing)
- docs/plan/08-security.md (untrusted code, secrets, webhooks)

Before coding, check the installed/pinned eas-cli version's docs for: non-interactive project init,
`eas build --non-interactive --no-wait --json`, credentials.json format with credentialsSource "local",
appVersionSource "remote" + autoIncrement, `eas webhook:create`, webhook signature verification, and
iOS App Store Connect API key env vars. Write what you verified into lib/providers/build/README.md.

Implement:
1. Expo token connect (validate, encrypt, store as oauth_connections provider 'expo').
2. app.json/eas.json templates; bundle id/package defaults, locked after the first build.
3. Android keystore generation with keytool (once per project), encrypted in project_signing.
4. build.native worker job: materialize into mkdtemp, secret scan, write configs + credentials,
   run eas-cli with ONLY EXPO_TOKEN as secret env, record native_builds, always clean up.
5. EAS webhook route with signature verification + scheduled polling fallback.
6. Publish dialog App Store tab + build history; IPA disabled with checklist if no Apple creds.
7. gate.canNativeBuild stub (real plan check comes in Phase 18).

Definition of done = Acceptance checklist. Tests: webhook signature, child-env allowlist,
keystore reuse, authz on every route. Use a fake BuildProvider in tests; never call EAS in CI.

Do NOT build: web publish (Phase 16), billing (Phase 18), store submission (eas submit).
Finish by running lint, typecheck, tests; summarize and list the manual steps (Expo token, Apple account).
```

## Pitfalls & notes
- **Keystore loss is permanent.** If you lose the Android upload key for an app published on Play (without Play App Signing), you can never update it. Encrypt it, back it up separately from the DB, and test restore. Encourage users to enroll in Play App Signing.
- **Bundle id changes break updates.** Lock them after the first production build; changing them means a new store listing.
- **EAS free-tier queues.** Free accounts can wait many minutes. The video saw a ~12.7 min queue on the web build. Set expectations in the UI ("Queued on Expo — free tier queues can take 10+ minutes") and keep showing status after the dialog closes.
- **Snack/Expo Go vs native build fidelity.** Builds use the real expo-router and native tab bars. Small UI differences from the browser preview are expected (the video notes this on device).
- **Child process hygiene.** Spawn with an explicit `env` object (not `process.env`), a timeout, and a max output size. Never log the token. Mask it in captured CLI output.
- **Credential sources.** Android uses `local` (our keystore); iOS uses `remote` (EAS-managed via the ASC key). Mixing these up per profile is a common failure. Keep the rendered `eas.json` in a snapshot test.
