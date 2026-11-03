# 06 — Generated App Template (the "contract")

Every generated app is an Expo app inside **one fixed structure**. The generators write screens and shared components. The **platform** writes everything derived from data (themes, schema, seed data, tab layout, backend config) and every config file.

This is what lets one set of files run in the browser preview, on Expo Snack/Expo Go, in zip/GitHub exports, in EAS builds and on the web. It also stops the agent from adding dependencies or scripts.

> **Revised after the reference implementation** (`13-reference-implementation.md`). Changes from v1: the agent no longer writes data stores (generic hooks from `@/lib/backend` instead); there's no fixed platform UI library (each app gets a generated shared-component library); themes are imported as a `colors` object, with CSS variables in the preview; the platform writes schema, seed and themes from JSON.

Template source: `templates/expo-app/` (versioned; `projects.template_version`).

---

## 1. Expo SDK pinning

- Pin the newest SDK that **the published `snack-sdk` supports**, and check that store **Expo Go** opens it. The reference app is pinned to **SDK 54**, because `snack-sdk` didn't go past 54 and newer Expo Go builds refuse older SDKs. Re-check both at Phase 07 start (decision D7).
- Dependency versions for export/builds come from `expo/bundledNativeModules.json` of the pinned `expo` version. Moving SDKs means a new template version, new dependency versions, and the Snack SDK constant, together.
- The Planner/generator prompts say "Expo SDK `<pinned>`, Expo Router".

## 2. Layout of a generated app

```
app/
  _layout.tsx                ← platform: root Stack, providers, first-screen redirect, per-screen
                               options from the plan (header shown/hidden, `presentation: "modal"` for add/edit screens)
  (tabs)/_layout.tsx         ← platform: shared tab bar from plan screens with `tab` (tab apps only)
  (tabs)/<screen-id>.tsx     ← Screen Builder: tab screens (the group doesn't change routes)
  <screen-id>.tsx            ← Screen Builder: stack screens (sign-in, details, …)
components/
  <Name>.tsx                 ← Component Designer/Builder: shared components (flat, PascalCase)
theme/
  index.ts                   ← platform-owned: exports `colors` (active theme) + `useThemeId`
  themes.ts                  ← platform-written from Foundation JSON: themes[] + activeThemeId
data/
  schema.ts                  ← platform-written: TS types from plan.collections
  seed.ts                    ← platform-written: sample records from plan.collections
lib/backend/                 ← platform-owned (never written by the agent)
  index.ts                     hooks: useCollection, useStoredValue, useAuth, uploadFile, sendEmailToMe
  provider.ts                  BackendProvider interface
  local.ts                     device storage (AsyncStorage) seeded from data/seed.ts
  <provider>.ts                remote provider client (Phase 13)
  config.ts                  ← platform-written: which provider + public config only
assets/                      ← icon, splash (platform)
package.json, app.json (static), eas.json, babel.config.js, tsconfig.json, README.md
                             ← platform-generated at materialization
```

Routes: `screenRoute(id)` maps a plan screen to `/(tabs)/<id>` or `/<id>`. The first screen (`plan.firstScreenId`) is the redirect target in `app/_layout.tsx`.

## 3. Ownership (enforced by `lib/vfs` + validators)

| Path | Generators | Inline editor / user | Notes |
|---|---|---|---|
| `app/(tabs)/<id>.tsx`, `app/<id>.tsx` | Screen Builder | ✅ (splice edits) | |
| `components/<Name>.tsx` | Component Designer/Builder | ✅ | |
| `app/_layout.tsx`, `app/(tabs)/_layout.tsx` | ❌ | ❌ | Regenerated from the plan on every change. Adding or renaming a tab rewrites no screen. |
| `theme/themes.ts` | ❌ (Foundation returns JSON) | via theme picker / `set_theme` | |
| `data/schema.ts`, `data/seed.ts` | ❌ (Foundation/Data return JSON) | ❌ | |
| `theme/index.ts`, `lib/backend/**` | ❌ | ❌ | Template code |
| configs | ❌ | ❌ | **Static `app.json` only**, never `app.config.js` (that would execute code during export/builds) |

Only agent- and user-authored files live in `project_files`. Everything else is rendered at materialization from the template + `projects.plan`, so a template fix reaches every project.

## 4. Allowed imports (`templates/expo-app/ALLOWED_IMPORTS.json`)

Must ship in Expo Go for the pinned SDK **and** have a working web shim in the preview runtime:

```
react, react-native, expo-router, react-native-safe-area-context, @expo/vector-icons,
expo-haptics, expo-image-picker, expo-clipboard, expo-linking, expo-location,
expo-notifications, expo-status-bar
```
plus the app's own `@/theme`, `@/lib/backend`, `@/data/schema` and `@/components/<Name>` (only components listed **before** the importing one, for components).

Phase 13 adds nothing to this list: the remote provider client is imported only by platform-owned `lib/backend/<provider>.ts`. The preview runtime bundles it, and the CSP `connect-src` allows the backend URL (`07` §6).

## 5. Theme system

**Tokens** (every theme defines all of them as `#rrggbb`):
`background, surface, surfaceAlt, border, text, textMuted, primary, onPrimary, primarySoft, accent, onAccent, accentSoft, success, successSoft, warning, warningSoft, danger, dangerSoft, overlay, shadow`

The Foundation Designer returns the base tokens. The platform derives soft tints and `on*` colors deterministically and fixes contrast (WCAG AA ≥ 4.5:1 for text on background, surface and primary). It also tints neutrals toward the primary hue if the model returned pure grays.

**Usage in code:** `import { colors } from "@/theme"`, which is usable inside `StyleSheet.create`. There's no hook and no `makeStyles`, because the static `StyleSheet` keeps every element on one line with its own named style (§9).

**How switching works without regenerating:**
- **Preview runtime:** `colors.primary` resolves to `"var(--c-primary)"` and so on. react-native-web passes these through to CSS, so a theme switch only sets CSS variables on the frame's root. It's instant, with no re-render or re-transpile.
- **Native (Snack, builds):** `theme/index.ts` resolves `colors` from `themes.ts` `activeThemeId` at load. A switch rewrites `activeThemeId`, and Snack live-reloads.
- `set_theme` can also **add** a theme (e.g. "make it purple"), which goes through the same derivation and contrast pipeline.

**Person-chosen colors:** an inline edit or an explicit "make this button #2563EB" writes a literal hex on that element. It's allowed by the validator (it was already in the file or named in the request) and **doesn't follow theme switches**. The inspector says so.

## 6. Data layer

The Foundation Designer outputs `collections`:

```ts
type Collection = {
  name: string;          // "habits" (camelCase plural)
  typeName: string;      // "Habit"
  fields: { name: string; type: "string" | "number" | "boolean" | "string[]"; references: string | null }[];
  items: Record<string, unknown>[];   // 5–8 realistic records, ids like "habit-1"
  shared: boolean;       // everyone sees the same records (catalog/feed) vs private per person
};
```

Generated screens use only `@/lib/backend`:

```ts
const { items, add, update, remove, loading } = useCollection<Habit>("habits");
const [units, setUnits] = useStoredValue("units", "metric");      // single saved values / settings
const { user, loading, signIn, signUp, signOut } = useAuth();      // local fake session until a backend exists
const { url } = await uploadFile(uri);
```

- `local.ts` stores collections in AsyncStorage on device. In the preview, the parent workspace persists the data (`07` §3), and seeds from `data/seed.ts` on first run.
- `useCollection` is a live list shared across screens (an external store), so a change on one screen shows on the others.
- Writes are validated against the collection's fields and types, on device too. That keeps behaviour identical once the remote provider refuses bad writes (Phase 13).
- Connecting a backend swaps `lib/backend/config.ts`, and **no screen changes**. Tables are derived from `collections` by the platform, never written by the agent (Phase 13).
- The Data Designer may only **add** collections, fields and records. Each new field needs a value on existing records, and screens must handle a missing field on older records.

## 7. Images

- When `IMAGEKIT_ENABLED=true`, generators write image URLs as **one plain string** of the form `<IMAGEKIT_URL_ENDPOINT>/ik-genimg-prompt-<description with %20, no commas/slashes/colons/quotes>/<short-name>.jpg` (verify the current syntax in ImageKit docs, Q2).
- Before saving a step's code or data, the server finds those URLs (at most 8 per step, placeholders past that). It waits for each image to finish generating, because a phone can't render ImageKit's "still preparing" page, then copies it into the project's folder and rewrites the URL to the stored copy. See Phase 12.
- Otherwise, placeholders: `https://picsum.photos/seed/<word>/400/300`. No local assets in generated code.

## 8. Shared components

- Generated per app by the Component Designer: 5–10 components, simple ones first, each importing only earlier ones. Always a button, a screen header and an empty state; **never a tab bar**.
- Props-only: records, copy and callbacks come in as props. No data hooks, no navigation. Export the `<Name>Props` type first, default-export the component, 20–80 lines, and give the outer element a `style` prop.
- Screens see only each component's **props type + purpose** (extracted by regex/AST), not its code.
- Editing a component keeps every existing prop working. New props are optional.

## 9. Editability rules (enforced by prompt + validator)

- Each visible element is **its own JSX element on its own line with its own named style** in `StyleSheet.create`.
- Fixed copy is written directly in JSX. `.map` only for real lists. No config-object rendering.
- These rules are what make element selection and source splicing reliable (`07` §5, Phase 10).

## 10. Materialization (`lib/vfs/materialize.ts`)

```
materialize(projectId, { target: "preview" | "snack" | "export" | "eas" | "web", snapshotId? })
  1. render platform files from template + projects.plan (themes, schema, seed, layouts, config)
  2. overlay project_files (screens, components)
  3. target-specific transforms:
     - snack:  path aliases → relative imports; App.js entry replaces app/_layout.tsx;
               "expo-router" → small shim on React Navigation (bottom-tabs for tab apps); the generated
               App.js reproduces the root layout: providers, `firstScreenId` as initial route, screen options  (Phase 11)
     - export/eas/web: package.json with versions from bundledNativeModules.json; eas.json;
               identity (`projects.slug`, `android_package`, `bundle_id`, set once at first build) (Phase 15);
               app.json web.output = "single" (SPA: screens never render in Node)
     - export: README (npm install && npx expo start)
  4. assets
```
