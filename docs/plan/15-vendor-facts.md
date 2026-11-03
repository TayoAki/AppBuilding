# 15 — Verified Vendor Facts (checked 2026-11-03)

Checked against primary sources: the npm registry, the `expo/snack` repo (last commit 2026-10-28), the `eas-cli` 27.3.0 package (published 2026-11-03), official docs and store listings. **Re-run this check (Phase 00b S0) if more than about 6 weeks pass before the dependent phase.** ✅ verified · 🟡 partially verified · ❓ couldn't verify.

---

## 1. Expo SDK, Expo Go, Snack: ⚠️ changes the plan

| Fact | Status |
|---|---|
| Latest Expo SDK is **58** (`expo@58.0.0` published 2026-09-29; stable announcement 2026-10-15; RN 0.88, React 19.3) | ✅ |
| Store **Expo Go is 58.0.5** (App Store 2026-10-22, Play 2026-10-19) and supports **only SDK 58** (one SDK at a time) | ✅ |
| Published `snack-sdk` is **6.6.2** (2026-04-01), which supports **SDK 50–54** (default 54). GitHub `main` (unpublished 7.1.0) lists 50–56. **No Snack supports 57/58.** | ✅ |
| Consequence: store Expo Go most likely **can't open any Snack right now** (Snack URLs request `runtime-version=exposdk:NN`) | 🟡 inferred; confirm on a phone in S3 |
| `snack-sdk` API: `updateFiles`, `updateDependencies`, `setOnline`, `saveAsync`, `getState`/`getStateAsync`, `addStateListener`, `addLogListener`, `reloadConnectedClients`, `getPreviewAsync`. **No `getUrlAsync`**: the URL is `state.url`. `state.connectedClients` lists devices (name, platform, status) | ✅ |
| Snack runtime renders `expo-router/entry` via `ExpoRoot` (code only, no docs page) | 🟡 |
| SDK 58 Expo Router defaults `web.output` to **"server"** when unset, so set `"single"` explicitly | ✅ |

**Plan changes:**
- **D7 is now SDK 58** for the template, matching store Expo Go and current EAS.
- **Phone preview defaults to a hosted Expo dev server** (Phase 11 Path B). Snack (Path A) is used only when a published `snack-sdk` supports the template SDK.
- The reference app's SDK 54 pin would force users onto an old Expo Go, which iOS users can't install from the App Store.

## 2. EAS Build

| Fact | Status |
|---|---|
| Android local credentials: `credentials.json` `{android:{keystore:{keystorePath, keystorePassword, keyAlias, keyPassword}}}` + `"credentialsSource": "local"` in the eas.json profile (default `"remote"`) | ✅ |
| **EAS generates an Android keystore in `--non-interactive` builds** when none exists (`generateRandomKeystoreAsync`). Older CI docs still suggest one interactive build first | ✅ (eas-cli 27.3.0 source) |
| iOS non-interactive: `EXPO_ASC_API_KEY_PATH`, `EXPO_ASC_KEY_ID`, `EXPO_ASC_ISSUER_ID` (omit for individual keys), `EXPO_APPLE_TEAM_ID`, `EXPO_APPLE_TEAM_TYPE`. Provisioning profiles can be created/repaired non-interactively, but **a missing distribution certificate fails** (`MissingCredentialsNonInteractiveError`). It must be created once interactively or uploaded | ✅ |
| Git: required by default. **`EAS_NO_VCS=1`** uploads the working directory without git (respects `.gitignore`/`.easignore`). `cli.requireCommit` defaults to false | ✅ |
| Flags: `--non-interactive`, `--wait`/`--no-wait`, `--json` (implies non-interactive), `--freeze-credentials`, `--auto-submit`, `--message` | ✅ |
| `appVersionSource: "remote"`: EAS stores versionCode/buildNumber; pair it with `autoIncrement` | ✅ |
| Non-interactive runs auto-create and link the EAS project when `owner` is set or the account has one owner. **Robot tokens require `owner`** | ✅ |

## 3. EAS webhooks

| Fact | Status |
|---|---|
| `eas webhook:create --event BUILD\|SUBMIT --url … --secret …` (secret ≥ 16 chars), **per project** | ✅ |
| Header **`expo-signature`**: HMAC-**SHA1** hex of the raw body, compared as `sha1=<hex>` | ✅ |
| Payload: `id, appId, platform, status (errored\|finished\|canceled), artifacts.buildUrl, buildDetailsPageUrl, metadata{appVersion, buildProfile, …}, error{message, errorCode}` | ✅ |
| Retries with exponential back-off on non-2xx/3xx responses | ✅ |

## 4. EAS Hosting ("Expo Hosting")

| Fact | Status |
|---|---|
| `npx expo export --platform web` then `eas deploy` with `--prod`, `--alias`, `--id`, `--dev-domain`, `--export-dir`, `--environment`, `--json`, `--non-interactive` | ✅ |
| Non-interactive deploys auto-pick a subdomain and **error if taken**, so pass `--dev-domain <slug>` | ✅ |
| URLs: production `https://<sub>.expo.app`; alias `https://<sub>--<alias>.expo.app`; preview `https://<sub>--<id>.expo.app` | ✅ |
| Free plan: 100k requests/mo, 1 GB storage, 5 aliases, 30-day deployment retention, no custom domain, usage stops at limits | ✅ |
| Auth via `EXPO_TOKEN` (standard eas-cli auth; not stated on the Hosting pages) | 🟡 |

## 5. ImageKit AI: ⚠️ changes pricing

| Fact | Status |
|---|---|
| Syntax: `{URL_ENDPOINT}/ik-genimg-prompt-{percent-encoded text}/{path}/{file.ext}` | ✅ |
| First request may return **200 HTML "being prepared"** with header **`is-intermediate-response: true`**; generation takes seconds to a minute | ✅ |
| The same path returns the cached image even if the prompt changes. **Change the filename** (or `?v=`) to regenerate | ✅ |
| Prompt length limit | ❓ (cap at ~200 chars ourselves) |
| Units: text-to-image **65**, `e-removedotbg` 130, `e-bgremove` 10, `e-edit` 50, `e-changebg` 60, `bg-genfill` 90, `e-genvar` 25, `e-upscale` 5, `e-retouch` 5 | ✅ |
| Plans: Free 650 units/mo (≈ 10 images, for the whole platform); Pro $89/mo includes 4,000 units, then $9 per 5,000 units (default cap 8,000) | ✅ |

**Plan change:** one generated image costs us about **$0.12** at overage rates (65 units × $9/5,000), roughly 4 credits at $0.03, not 1. See the revised image pricing in `11` §3. Build-time images are capped and charged.

## 6. Supabase (for platform integrations)

| Fact | Status |
|---|---|
| OAuth apps (org settings → OAuth Apps); `https://api.supabase.com/v1/oauth/authorize` + `/v1/oauth/token`; PKCE recommended; scopes set on the app (`scope` param deprecated) | ✅ |
| Create project: `POST /v1/projects` (scope `projects:write`; needs `db_pass`, `name`, `organization_slug`) | ✅ |
| Run SQL: `POST /v1/projects/{ref}/database/query` (scope `database:write`, **Beta**); body `{query, parameters, read_only}` | ✅ |
| Keys: `GET /v1/projects/{ref}/api-keys?reveal=` (scope `secrets:read`). New **`sb_publishable_…` / `sb_secret_…`** keys; anon/service_role are deprecated by end of 2026. Secret keys get 401 from browsers | ✅ |
| `db_pass` can't be read back, so **store it encrypted** at creation | ✅ |

**Plan change:** A15 for Supabase moves from 🔴 to 🟡 (documented; still needs the S6 run). Use publishable keys in apps and secret keys only server-side.

## 7. Anthropic: mid-conversation system messages

| Fact | Status |
|---|---|
| Supported on `claude-opus-5-5`, no beta header | ✅ |
| **Placement:** must come **right after a user turn** (or after an assistant turn ending in a server-tool result), and must be the **last entry or followed by an assistant turn**; otherwise 400 | ✅ |
| `clear_at`, mid-conversation tool changes and per-message effort need betas; not available on Sonnet 5 | ✅ |

**Plan change:** the PROJECT STATE message goes **after** the latest user message, as the last entry (`05` §3.1 fixed).
