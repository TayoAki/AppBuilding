# 07 — Browser Preview Runtime (Canvas + Live Preview + Inspector)

The canvas shows every screen of the generated app side by side in phone frames, and the right panel runs the app interactively. Both run **real generated code in the browser**. This is the riskiest component in the project, so Phase 07 starts with a spike against a hand-written fixture app, before any AI is involved.

---

## 1. Requirements

| # | Requirement | Source |
|---|---|---|
| R1 | Render any screen of the project in isolation (a canvas frame pinned to one route) | Canvas |
| R2 | Run the whole app interactively (navigation, state, local data) | Right-side preview |
| R3 | Hot-swap a module when a `file.written` event arrives, with no full reload | Live build |
| R4 | Instant theme switch | Theme picker |
| R5 | Hover/click to select an element and map it back to source (file + line + col) | Inline editing |
| R6 | Report runtime errors to the parent (for the "Fix with AI" button and the render smoke test) | Quality |
| R7 | Generated code can't reach the platform session, cookies or APIs | Security |
| R8 | 6–10 frames on one canvas stay usable on a laptop | Performance |

## 2. Approach

**Run react-native-web inside sandboxed iframes, with an in-browser module loader.** Real devices are a separate concern (Phase 11: hosted Expo dev server by default, Snack only when it supports the template SDK).

Alternatives considered:
- *Snack web player for everything.* One session per app, slower cold start, tied to the Snack SDK list, and hard to instrument for element selection. Rejected for the canvas.
- *Server-side Metro/Expo web bundling per edit.* Seconds of latency per change and heavy infrastructure. Rejected.
- *Sandpack (CodeSandbox) bundler.* Possible, but it doesn't understand expo-router and adds an external runtime dependency. It's a fallback if the custom loader spike fails.

## 3. Pieces

```
preview-runtime/
  src/
    vendor.ts          bundles react, react-dom, react-native-web, @expo/vector-icons (web, Ionicons
                       font as a bundled asset), react-native-safe-area-context, async-storage (web), and
                       a web build of every module in 06 §4's allowlist, nothing else; the platform's
                       remote backend client for `lib/backend` (Phase 13)
                       → exposes a module registry: window.__AB_MODULES__["react-native"] = …
    shims/
      expo-router.tsx  memory-history implementation of the subset the template uses:
                       Stack, Tabs, Slot, Link, Redirect, router, useRouter,
                       useLocalSearchParams, useSegments, usePathname, Stack.Screen options
      expo-haptics.ts (no-op), expo-image-picker.ts (<input type=file>), expo-clipboard.ts,
                       expo-location.ts (browser geolocation, permission-gated), expo-notifications.ts (no-op),
                       expo-linking.ts, expo-status-bar.ts — one shim per module in 06 §4
      async-storage.ts in-memory store; the parent persists snapshots via postMessage
                       (a sandboxed iframe without allow-same-origin has no usable
                       localStorage — touching it throws)
    transpile.ts       @babel/standalone: TS + JSX → CommonJS, plus the inspector plugin (§5)
    loader.ts          tiny CommonJS loader: resolves "./x", "@/x", file extensions, index files;
                       caches by sha256; supports invalidate(path) for hot swap
    routes.ts          builds the route table from app/** file paths (expo-router conventions:
                       (groups), [param], _layout, index)
    bridge.ts          postMessage protocol with the parent (§4)
    entry.tsx          boots: receive files → build routes → render <Root initialRoute/> 
  build.mjs            esbuild → public/preview-runtime/{runtime.js, vendor.js}
```

- **Vendor bundle** is built once at platform build time and cached by the browser. Generated code never brings its own dependencies: imports outside the registry fail with a clear error (the validator catches this first).
- **Transpile cache:** key = file sha256. The same file in 8 frames is transpiled once per frame. Optionally share via a `SharedWorker` later.
- **expo-router shim:** implement just enough for the template's patterns, not all of expo-router. Generated apps are constrained by the contract, so the surface is small and testable. Fidelity issues (e.g. tab bar look) are acceptable in preview; real fidelity comes from Expo Go (Phase 11) and EAS builds. The video notes the same.

## 4. Parent ↔ iframe protocol (`bridge.ts`)

All messages are `{ v: 1, type, ... }`. Validate them with Zod on **both** sides. Accept messages only from the known iframe `contentWindow` (opaque origins show up as `"null"`, so compare `event.source`, not just `origin`).

Parent → runtime:
| type | payload |
|---|---|
| `init` | `{ projectId, frameId, mode: "frame" \| "interactive", initialRoute, files: {path, sha, content}[], themeId, viewport }` |
| `file.update` | `{ path, sha, content }` (or `deleted: true`) |
| `theme.set` | `{ themeId }` |
| `navigate` | `{ route }` |
| `inspect.mode` | `{ enabled: boolean }` |
| `inspect.highlight` | `{ afid }` (sync selection from code view) |
| `edit.override` | `{ afid, style?, text?, imageUri? }`: apply on top of the running code (§5) |
| `reset.data` | — clears the frame's in-memory AsyncStorage (and the parent's persisted copy) |
| `storage.restore` | `{ snapshot }` — parent restores persisted AsyncStorage contents on boot |

Runtime → parent:
| type | payload |
|---|---|
| `ready` | `{ frameId }` |
| `rendered` | `{ route, ms }` |
| `route.changed` | `{ route }` |
| `element.hover` / `element.select` | `{ afid, rect, component, props: { style (resolved), text? }, isDynamic }` |
| `runtime.error` | `{ message, stack, path?, line? }` |
| `console` | `{ level, args }` (dev only) |
| `storage.snapshot` | `{ snapshot }` (debounced) — parent persists it per project in its own storage |

## 5. Element instrumentation (for inline editing)

Source files stay clean. Instrumentation is added **only at preview transpile time** by a Babel plugin:

1. For every JSX element whose name resolves to a host-ish RN component (`View`, `Text`, `Pressable`, `TouchableOpacity`, `Image`, `ScrollView`, `TextInput`, and every app component from `components/<Name>.tsx`), add `dataSet={{ afid: "<path>:<line>:<col>" }}`. If `dataSet` already exists, merge into it. (The reference app stamped `<file>:<line>:<type>` because its compiler gave no column. Two elements of the same type on one line were then AI-edit-only. Babel gives columns, so we stamp `line:col`. The "one element per line" rule (`06` §9) still applies, because it keeps splicing simple.)
2. react-native-web renders `dataSet` as `data-afid="…"` on the DOM node.
3. In inspect mode the runtime listens to `pointermove`/`click` (capture phase, `preventDefault` on click), finds the nearest `[data-afid]`, and posts `element.hover` / `element.select` with `getBoundingClientRect()` and the computed style.
4. `isDynamic`: the plugin also records whether the element's text child is a JSX literal (`<Text>Daily reminder</Text>`) or an expression (`<Text>{habit.name}</Text>`). The inspector disables text editing for dynamic values, like the video does.

Mapping `afid` → AST node on the server: re-parse the current revision and find the JSX element whose `loc.start` matches `line:col`. If the file has changed since (sha mismatch), the client refreshes the selection first.

**Live override, then persist** (pattern proven in the reference app): a manual edit is first sent to every frame as `edit.override {afid, style?, text?}`. The runtime applies it on top of the running code and re-renders only that element, so the app never restarts. Shortly after (debounced), the server splices the edit into the source and saves a revision. Frames **keep running the code they compiled**, so their `afid` stamps stay valid, until the agent changes that file or the page reloads. Each save re-applies all pending edits to that same base revision.

## 6. Isolation and security

- Iframes use `sandbox="allow-scripts"` **without** `allow-same-origin`. The result is an opaque origin: no cookies, no `localStorage` from the parent origin, no fetch with credentials to our API.
- Load the runtime HTML via `srcdoc` (or from a separate preview domain). Set a CSP that allows only our vendor/runtime script URLs, `img-src` for the image provider, our storage and data/blob, **`font-src <runtime origin> data:`** (Ionicons ships as `@font-face`; without it icons render blank, ADR-003 #15), and `connect-src` limited to the connected backend's URL (or our gateway) once Phase 13 exists:
  `default-src 'none'; script-src <runtime origin> 'unsafe-eval'; style-src 'unsafe-inline'; font-src <runtime origin> data:; img-src https: data: blob:; connect-src <backend or gateway URL>`
- `html-to-image` frame snapshots must inline fonts and images (fetch them as data URLs first), or snapshots show missing icons.
- Phase 07 acceptance includes a check that icons render under the **production CSP** in a sandboxed frame.
- The runtime never receives platform tokens. Files are pushed in by the parent via `postMessage`; the runtime has no API access of its own.
- Generated code that loops forever freezes only its own iframe. The parent shows a "frame not responding" overlay after a heartbeat timeout and offers reload.

## 7. Performance

- **Canvas frames** (`mode: "frame"`) render one route each, at a scale transform (e.g. 0.6). Frames outside the viewport are **paused**: replaced by their last screenshot (`html-to-image` snapshot posted by the runtime), and only mounted when visible.
- **Interactive preview** (`mode: "interactive"`) is a single iframe at 1:1 phone size.
- During a build, `file.update` messages are coalesced per 150 ms.
- Theme switch is just `theme.set` → set CSS variables on the frame root. `colors.*` tokens resolve to `var(--c-*)` in the preview (`06` §5), so nothing re-renders or re-transpiles.

## 8. Render smoke test (server side, Phase 08)

The **sandbox service** (no secrets, no DB access; `08` §5, ADR-003 #18) runs Playwright + Chromium. It serves the bundled `public/preview-runtime/` (built into the sandbox image, the same build as the web app's) from a local static server on `127.0.0.1`, and loads `harness.html` with the project's files, which the worker passes in as a signed storage URL. It renders each route in turn and collects `runtime.error` messages. Their count goes into `validation.result`, and errors go back to the agent's fix stage. The same harness produces **thumbnails** for the projects grid (Phase 17).

## 9. Spike plan (Phase 00b S1; the code graduates into `preview-runtime/`)

The spike runs as **S1 in Phase 00b** (ADR-003 #10). Its code is the first version of `preview-runtime/` (not throwaway); Phase 07 hardens and tests it.

1. Hand-write the **one canonical fixture** `preview-runtime/fixtures/habit-tracker/` following the template contract: `plan.json` with **sign-in + 2 tab screens + 1 stack detail screen**, **4 components**, **2 collections**, 5 themes.
2. Get it rendering in a single interactive iframe: vendor bundle, loader, router shim.
3. Add frame mode with `initialRoute`, and 4 frames on a test page.
4. Add hot swap: edit a file in a textarea → `file.update` → re-render under 300 ms.
5. Add the inspector plugin and print `afid`s on click; `edit.override` restyles one element without a restart.
6. CSS-variable theme switch across all frames; icons render under the production CSP.

**Exit criteria:** all six work in Chrome and Safari; 8 frames stay usable. If the router shim proves infeasible in about 3 days, fall back to Sandpack with a custom expo-router shim (A1 in `14`).
