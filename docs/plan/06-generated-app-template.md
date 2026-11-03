# 06 — Generated App Template (the "contract")

Every app the platform generates is an Expo app built **inside one fixed template**. The agent fills in screens, components, theme and data. The platform owns everything else.

This one decision is what makes browser preview, Expo Go (QR), zip export, GitHub push, EAS builds and web publish all work from the same files. It also keeps the agent from adding dependencies or scripts that would run arbitrary code on our build workers.

Template location in the platform repo: `templates/expo-app/`. It's versioned (`templates/expo-app/VERSION`), and each project stores `template_version`.

---

## 1. Expo SDK pinning (decide in Phase 07)

- Pin **one** Expo SDK version that is supported by **Expo Snack** (for QR/device preview) and the **current Expo Go** app in the stores. Check the Snack SDK list and Expo Go release notes at implementation time.
- The video hit exactly this problem: generated code targeted a newer SDK than Snack supported, which broke the QR flow. Pick the version deliberately and write it into `templates/expo-app/package.json`.
- Moving to a new SDK means a new template version plus a migration script for existing projects. Never do it silently.

## 2. Folder layout of a generated app

```
app/                         ← agent-owned: Expo Router routes
  _layout.tsx                  root Stack + ThemeProvider + SafeAreaProvider
  (tabs)/_layout.tsx           ← platform-written: Tabs, regenerated deterministically from
                                 the `screens` list + plan.navigation (S1, add/remove screen)
  (tabs)/index.tsx             first tab screen
  (tabs)/<screen>.tsx
  <stack-screen>.tsx           e.g. habit/[id].tsx, add-habit.tsx (modal)
components/
  ui/                        ← platform-owned component library (read-only to agent)
  <screenId>/                ← agent-owned screen-local components
  shared/                    ← agent-owned app-wide components
theme/
  tokens.ts                  ← platform-owned: Theme type + helpers (contrast, alpha)
  themes.ts                  ← agent-owned: the 5 theme packs
  active.ts                  ← platform-owned writer: export const ACTIVE_THEME_ID = "…"
  ThemeProvider.tsx, useTheme.ts   ← platform-owned
data/
  types.ts                   ← agent-owned: entity types (from plan.entities)
  seed.ts                    ← agent-owned: realistic sample data
  images.ts                  ← agent-owned (Phase 12): image registry { key: {uri, width, height} }
lib/
  store/                     ← agent-owned: one collection per entity, via createCollection()
  data/                      ← platform-owned: adapter interface + LocalAdapter + RemoteAdapter
    config.ts                ← platform-written: which adapter + public backend config
  images.ts                  ← platform-owned (Phase 12): image URL helper + <RemoteImage>
backend/                     ← agent-owned (Phase 13)
  migrations/NNN_*.sql
assets/                      ← platform-managed: icon, splash, uploaded images
package.json, app.json, eas.json, babel.config.js, tsconfig.json, metro.config.js
                             ← platform-owned, generated at materialization
README.md                    ← platform-generated at export
```

## 3. Ownership rules (enforced by `lib/vfs` and validators)

| Path | Agent may write | User code editor may write | Notes |
|---|---|---|---|
| `app/**` (except `app/(tabs)/_layout.tsx`) | ✅ | ✅ | |
| `app/(tabs)/_layout.tsx` | ❌ (platform regenerates from `screens`) | ❌ | Keeps tab order consistent (Phase 09) |
| `components/<screenId>/**`, `components/shared/**` | ✅ | ✅ | |
| `components/ui/**` | ❌ | ❌ (v1) | The library is versioned with the template |
| `theme/themes.ts` | ✅ | ✅ | |
| `theme/active.ts` | ❌ (platform writes on theme switch) | ❌ | |
| `data/**`, `lib/store/**` | ✅ | ✅ | |
| `lib/data/**` | ❌ | ❌ | Rewritten by backend connect |
| `backend/migrations/**` | ✅ (Phase 13) | ✅ | Applied by provisioning job, never by the agent |
| `package.json`, `app.json`, `eas.json`, configs | ❌ | ❌ | Generated from project metadata at materialization |

Use a **static `app.json` only**, never `app.config.js`/`app.config.ts`. A JS config would execute code during `expo export`/EAS builds.

Platform-owned files are **not stored per project**. They're rendered from the template plus project metadata (name, slug, bundle id, icon, theme colors for splash) every time we materialize. So a template patch fixes every project at once.

## 4. Allowed imports (`templates/expo-app/ALLOWED_IMPORTS.json`)

Only modules that ship in Expo Go **and** work on react-native-web:

```json
[
  "react", "react-native",
  "expo-router", "expo-status-bar", "expo-image", "expo-linear-gradient",
  "expo-haptics", "expo-constants", "expo-linking",
  "@expo/vector-icons",
  "react-native-safe-area-context",
  "react-native-svg",
  "@react-native-async-storage/async-storage",
  "date-fns"
]
```

Add modules only when the preview runtime has a working web shim **and** the Snack/Expo Go SDK includes them. Each addition is a template version bump.

**Expected addition in Phase 13:** the chosen backend's JS client (e.g. `@supabase/supabase-js`). It gets imported only by platform-owned `lib/data/remote*.ts`, never by agent code. It's added to the preview runtime's vendor bundle, and the backend URL is added to the runtime CSP `connect-src` (`07` §6).

## 5. Theme system

```ts
// theme/tokens.ts (platform-owned)
export type Theme = {
  id: string; name: string; mode: "light" | "dark";
  colors: {
    background: string; surface: string; surfaceAlt: string; border: string;
    primary: string; onPrimary: string; secondary: string; onSecondary: string;
    text: string; textMuted: string; success: string; warning: string; danger: string;
  };
  radius: { sm: number; md: number; lg: number; pill: number };
  spacing: (n: number) => number;          // 4-pt scale
  font: { family?: string; sizes: { xs: number; sm: number; md: number; lg: number; xl: number; xxl: number } };
};
```

- The agent writes **5 themes** in `theme/themes.ts`. The first matches `plan.themeSeed`, the rest are variations (e.g. "Mint Leisure", "Harbor Night" in the video).
- When `themes.ts` is written, the platform **post-processes** it with a deterministic contrast fix: `onPrimary`/`text` are adjusted until WCAG AA contrast (≥ 4.5:1) holds. "Text colors adjust automatically to stay readable" in the video is this step.
- The theme picker writes `theme/active.ts` only. The preview hot-swaps it instantly, with no LLM call and no credits.
- **Contract rule:** build theme-dependent styles **inside** the component, `const styles = useMemo(() => makeStyles(theme), [theme])`, never in a module-level `StyleSheet.create` that can't see the theme. Instant theme switching and the Phase 10 inline token edits on shared styles both depend on it.

## 6. Data layer

```ts
// lib/data/adapter.ts (platform-owned)
export interface DataAdapter {
  list<T>(collection: string, opts?: { orderBy?: string }): Promise<T[]>;
  get<T>(collection: string, id: string): Promise<T | null>;
  insert<T>(collection: string, value: Omit<T, "id">): Promise<T>;
  update<T>(collection: string, id: string, patch: Partial<T>): Promise<T>;
  remove(collection: string, id: string): Promise<void>;
  subscribe?(collection: string, cb: () => void): () => void;
}
// LocalAdapter: AsyncStorage, seeded from data/seed.ts on first run.
// RemoteAdapter: thin client for the connected backend provider (Phase 13).
```

```ts
// lib/store/habits.ts (agent-written, uses platform helper)
export const habits = createCollection<Habit>("habits", { seed: seedHabits });
// → habits.useAll(), habits.useOne(id), habits.add(v), habits.update(id, p), habits.remove(id)
```

Screens may only touch data through `lib/store/*`. When the user connects a backend, the platform swaps the adapter in `lib/data/config.ts`, and the agent writes migrations for one table per entity. Screens don't change.

## 7. Auth hooks (Phase 13)

`lib/data/auth.ts` (platform-owned) exposes `useSession()`, `signIn`, `signUp`, `signOut`, `signInWithOAuth(provider)`. With the LocalAdapter it's a fake single-user session, so auth screens work in preview before any backend exists.

## 8. Component library (`components/ui`)

Ship a polished, theme-aware library so generated apps look consistent and professional:

`Screen` (safe area + scroll + background), `Header`, `Text` (variants), `Button` (primary/secondary/ghost/destructive, sizes), `IconButton`, `Card`, `ListItem`, `TextField`, `NumberField`, `Select`, `Switch`, `Chip`, `SegmentedControl`, `Badge`, `Avatar`, `StatCard`, `ProgressBar`, `ProgressRing` (svg), `EmptyState`, `Sheet` (modal), `FAB`, `Divider`, `Skeleton`, `Toast`.

Every component:
- reads `useTheme()` only (no literal colors),
- forwards `testID` and `dataSet` props, which the inspector needs (`07-preview-runtime.md` §5),
- has a one-paragraph doc block. These are concatenated into the **COMPONENT API DOCS** prompt block.

## 9. `CONTRACT.md`

`templates/expo-app/CONTRACT.md` is the human-readable version of §2–§8. It's injected verbatim into the agent's system prompt (cached). Keep it **under ~4k tokens**, concrete and example-driven. Update it whenever the template changes. The eval suite catches regressions.

## 10. Materialization (`lib/vfs/materialize.ts`)

```
materialize(projectId, snapshotId?) → { files: Map<path, string> }
  1. render platform-owned files from templates/expo-app (Handlebars-style placeholders:
     {{appName}}, {{slug}}, {{bundleId}}, {{scheme}}, {{version}}, {{buildNumber}}, {{splashBg}})
  2. overlay project_files (at snapshot or current)
  3. write lib/data/config.ts from project_backends.public_config (or local)
  4. add assets (icon/splash/images) from storage
  5. add README.md (how to run: npm install && npx expo start)
```

Zip export, GitHub push, Snack sync, EAS build and web publish all call `materialize()`.
