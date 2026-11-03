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
- **The Android keystore is created before the first build.** Non-interactive EAS builds won't create a new keystore (the video and the reference app both hit this). There are two ways to do it (step 4). We recommend letting **EAS generate and hold it** if the pinned `eas-cli`/API can do that non-interactively, and generating it ourselves with `keytool` otherwise. Either way, an app that has a keystore is never given a new one.
- **The user's Expo account, the user's token.** The user pastes an Expo personal access token once. It's stored encrypted as `oauth_connections(provider = "expo")`. Builds count against **their** EAS quota. No platform-wide Expo token exists, so one leaked token can't touch every user's builds.
- **Honest iOS story.** IPA builds need a paid Apple Developer account and signing credentials. We support the App Store Connect API key path and say up front that iOS needs more setup. The video glosses over this.
- **Reference app notes** (`13-reference-implementation.md`):
  - **R18:** identity (slug, `android.package`, `ios.bundleIdentifier`) is set **once** at the first build with a project-id suffix, because names change and repeat. Export and push use it too, along with `eas.json`. `appVersionSource: remote` lets EAS own `versionCode`/`buildNumber`.
  - The uploaded project **must be a git repo**, or EAS prompts "Would you like us to run git init…" and fails non-interactively ("stdin is not readable"). We `git init` and commit inside the temp dir (step 6).
  - iOS: if EAS has no Apple credentials for the bundle id, the dialog shows **setup steps instead of starting a build**.
  - The reference app dispatches **EAS workflows** through the undocumented GraphQL API behind `eas workflow:run` (uploading a tarball, no git ref), and **polls build status only when someone opens the dialog** (no webhook). We prefer the documented `eas-cli` in our worker plus EAS webhooks, with polling as a fallback, so builds progress with the tab closed and nothing depends on an undocumented API (`13` §3). If `eas-cli` can't do something non-interactively, isolate any GraphQL fallback in `lib/providers/build/eas.ts`, as the reference app does in `lib/publish/expo.ts`.
- **Builds treat project code as untrusted.** The materialized project runs through `eas-cli` in an **ephemeral container/temp dir** with only the user's Expo token in env, no platform secrets, and no install scripts (`npm ci --ignore-scripts` if dependencies are needed at all). The actual compilation happens on Expo's servers.

## Steps

### 1. Expo token connect
- Settings / Publish dialog: "Connect Expo" → instructions (expo.dev → Account settings → Access tokens → create) → paste token.
- `POST /api/expo/token` → validate by calling the Expo account API (`eas whoami` equivalent) in the worker → store encrypted in `oauth_connections` with `external_login` = Expo username.
- Show the connected account + "Disconnect".

### 2. EAS project linking
- On the first build for a project, the worker runs the EAS project init non-interactively under the user's account (e.g. `eas init --non-interactive --force` / `eas project:init`; **verify flags for the installed `eas-cli` version**). Store the resulting EAS project id in `projects.eas_project_id` and render it into `app.json` (`extra.eas.projectId`) at materialization.
- **Identity, set once:** on the first build or web publish, compute `projects.slug` = `<kebab app name>-<6 chars of project id>` (e.g. `pennywise-k3x9qa`), `android_package` = `com.<expoUsername>.<slug without dashes>`, and `bundle_id` = the same. Show them in the dialog so the user can edit them **before** the first build. After it they're **locked**, because changing them breaks store updates. `materialize({ target: "eas" | "export" | "web" })` reads them (`06` §10), so zip/GitHub exports carry the same ids.

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

### 4. Android keystore (`ensureAndroidKeystore`, once per project)
Pick one option during implementation, after checking the pinned `eas-cli` docs.

| | **A. EAS-managed (recommended if available)** | **B. We generate it** |
|---|---|---|
| How | Have EAS generate a keystore and register it as the app's **remote** Android credentials before the first build. The reference app's `ensureAndroidKeystore` does this. Check first whether a documented non-interactive path exists (credentials commands/API); otherwise isolate the call. | `keytool -genkeypair -v -storetype JKS -keyalg RSA -keysize 2048 -validity 10000 -alias <alias> …` with random passwords; pass it via `credentials.json` + `credentialsSource: "local"` |
| `eas.json` | `android.credentialsSource: "remote"` | `android.credentialsSource: "local"` |
| Secrets we hold | none | keystore + passwords (encrypted `project_signing`, separate backup) |
| Portability | user downloads it from expo.dev | "Download signing key" (owner + re-auth, audit-logged) |
| Risk | depends on an EAS path that may be undocumented | key loss is on us, so backups are mandatory (Phase 19) |

Rules for both: check for an existing keystore first and **never replace one**. Record which option a project uses (`project_signing.android_keystore_enc` null ⇒ remote). For option B, write `credentials.json` + `.jks` only into the temp dir and delete it afterwards.

### 5. iOS credentials
- Needs: Apple Developer Program membership, team id, and an **App Store Connect API key** (`.p8`, key id, issuer id) that EAS uses to manage certificates/profiles non-interactively. **Verify current EAS docs** for the exact env vars/flags.
- Store the key encrypted (add `ios_asc_key_enc`, `ios_asc_key_id`, `ios_asc_issuer_id`, `apple_team_id` to `project_signing`, or per-user if shared across projects).
- Before an iOS build, check whether EAS already has distribution credentials for `bundle_id`, or we hold an ASC key. If neither, the dialog shows **setup steps instead of starting a build** (enroll in Apple Developer Program → create an ASC API key → paste it here, *or* run `eas credentials` once locally). The IPA option stays disabled until this is resolved. Internal-distribution IPAs also need registered device UDIDs. Document this rather than hide it.

### 6. Build job `build.native`
Worker `worker/handlers/build-native.ts`:
1. Gate: `gate.canNativeBuild(user)` (stub returns true; Phase 18 makes it Pro-only).
2. `materialize({ target: "eas" })` of the current snapshot into `mkdtemp()` (static `app.json` with identity + `extra.eas.projectId`, `package.json` from `bundledNativeModules.json`); secret scan (Phase 14).
3. Write `eas.json` and signing files (option B only).
3b. **Make it a git repo:** `git init -q && git add -A && git -c user.name=AppBuilding -c user.email=builds@<domain> commit -qm "Build <version>"`. Without it EAS prompts for `git init` and fails non-interactively. (Check whether the pinned `eas-cli` supports a no-VCS mode such as `EAS_NO_VCS=1`; a committed repo works either way.)
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
- Uses `projects.slug`, `projects.bundle_id`, `projects.android_package` (set once, then locked) and `projects.eas_project_id` (all in `03-data-model.md` §2).
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
- [ ] The first Android APK build sets the identity once (slug with project-id suffix; package/bundle id shown, editable, then locked), creates the EAS project, ensures exactly one keystore (option A or B), queues the build, and the status updates to finished with a working download link, **with the browser tab closed during the build**.
- [ ] The uploaded/temp project is a git repo; a build never stalls on an EAS `git init` prompt.
- [ ] A second build reuses the same keystore (same SHA-1 fingerprint) and auto-increments `versionCode`.
- [ ] An AAB build succeeds with the same keystore.
- [ ] Webhook with a bad signature → 401; a valid one updates `native_builds` and the open workspace.
- [ ] With the webhook disabled, the polling fallback still completes the status.
- [ ] The child process env for `eas build` contains only `EXPO_TOKEN` (+ PATH/HOME); test asserts no platform secrets are passed.
- [ ] With no Apple credentials in EAS (and no ASC key), choosing IPA shows setup steps instead of starting a build.
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
2. Identity set once (projects.slug = name + project-id suffix, android_package, bundle_id; editable
   before the first build, then locked) + eas.json template (appVersionSource remote, autoIncrement).
3. ensureAndroidKeystore: prefer EAS-managed remote credentials if a non-interactive path exists in the
   pinned eas-cli; otherwise keytool + credentials.json (local), encrypted in project_signing. Never replace
   an existing keystore. Tell me which option you chose and why.
4. build.native worker job: materialize({target:"eas"}) into mkdtemp, secret scan, write configs
   (+ credentials for option B), git init + commit in the temp dir, run eas-cli with ONLY EXPO_TOKEN as
   secret env, record native_builds, always clean up.
5. EAS webhook route with signature verification + scheduled polling fallback.
6. Publish dialog App Store tab + build history; iOS shows setup steps instead of building when EAS has
   no Apple credentials for the bundle id. Status survives closing the tab.
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
- **Credential sources.** Android uses `remote` (option A) or `local` (option B); iOS uses `remote` (EAS-managed via the ASC key). Mixing these up per profile is a common failure. Keep the rendered `eas.json` in a snapshot test for both options.
- **"Experience … does not exist" (EXPERIENCE_NOT_FOUND).** The reference app hit this when the EAS project didn't exist yet under the user's account. Always run project init (step 2) and store `eas_project_id` before the first build.
