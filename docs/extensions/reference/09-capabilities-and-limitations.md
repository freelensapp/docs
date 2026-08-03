<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# 9. Capabilities and limitations

The direct answer to *"what can I do, and what can I not do, from an
extension?"*

## 9.1 What you CAN do

### Add UI (renderer)

- **Pages**: full pages inside a cluster (`clusterPages`) or in the root frame
  (`globalPages`), each with its own URL and params.
- **Sidebar entries**: cluster sidebar items and nested sub-items
  (`clusterPageMenus`).
- **Top bar items** (`topBarItems`) and **status bar items** (`statusBarItems`).
- **Kube object detail panels** for any `kind`/`apiVersions`
  (`kubeObjectDetailItems`).
- **Kube object menu actions** and dynamic context-menu items
  (`kubeObjectMenuItems`, `kubeObjectHandlers`).
- **Workloads Overview sections** (`kubeWorkloadsOverviewItems`).
- **Preferences**: preference control blocks and whole new preference tabs
  (`appPreferences`, `appPreferenceTabs`).
- **Entity settings panes** (`entitySettings`).
- **Catalog UI**: extra columns and replacement views for catalog categories
  (`additionalCategoryColumns`, `customCategoryViews`), and catalog-entity
  detail panels (`catalogEntityDetailItems`).
- **Command-palette commands** (`commands`) and **Welcome-page actions**
  (`welcomeMenus`).
- **Always-mounted overlays** in the cluster frame (`clusterFrameComponents`).
- Use the full **shared React component library** (`Renderer.Component`):
  layouts, tables, forms, dialogs, notifications, charts, drawers, the Monaco
  editor, icons, and the ready-made `KubeObjectListLayout`.

### Work with Kubernetes

- **Model custom resources (CRDs)** with the `KubeObject` / `KubeApi` /
  `KubeObjectStore` three-class pattern, including multiple API versions.
- **Read/watch resources** through built-in and custom stores (reactive via
  MobX).
- **CRUD against any cluster** with the `K8s` functions: `queryCluster`,
  `queryClusters`, `queryAllClusters`, `getResource`, `applyOnCluster`,
  `deleteOnCluster`, `patchOnCluster` — from the renderer (over IPC) or the
  main process (direct).
- **Request metrics** and render them with the chart/metrics components.
- **Drive dock tabs**: open terminals (`createTerminalTab`, `terminalStore`)
  and log tabs (`logTabStore`).

### OS / app integration (main)

- Add **application menu** items (`appMenus`) and **system-tray** items
  (`trayMenus`).
- **Modify terminal shell environment** (`terminalShellEnvModifier`).
- Hook **power/lifecycle events** (`Main.Power`: suspend/resume/shutdown).
- Handle **`freelens://` deep links** (`protocolHandlers`).

### State, catalog, and infrastructure

- **Persist data** with `Common.Store.ExtensionStore` — auto-synced across
  processes, MobX-reactive.
- **Publish catalog entities** and **define catalog categories/entity kinds**
  (`addCatalogSource`, `Common.Catalog` base classes).
- **Read the catalog** and the active cluster (`Renderer.Catalog`).
- **Communicate main ↔ renderer** via per-extension IPC (`Main.Ipc` /
  `Renderer.Ipc`) or raw Electron IPC.
- **Navigate** the app (`Renderer.Navigation`).
- **React to the active theme** (`Renderer.Theme.activeTheme`).
- **Read app info** (`Common.App`), **log** (`Common.logger`), use the
  **event bus** (`Common.EventBus`), resolve the **system proxy**
  (`Common.Proxy`), and use bundled **utilities** (`Common.Util`).
- **Enable/disable per cluster** with `isEnabledForCluster(cluster)` (now
  `@deprecated` — prefer per-registration `enabled`/`visible` + `activeCluster`).
- Write to a **per-extension file folder** (`getExtensionFileFolder()`).

## 9.2 What you CANNOT do (or must not rely on)

### Registration surface limits

- **No arbitrary UI injection.** You can only render where a registration
  extension point exists (the lists in [chapter 3](./03-renderer-registrations.md)).
  There is no generic "inject a component at CSS selector X" API.
- **`kubeObjectStatusTexts` and `kubeObjectListLayoutColumns` do not exist** in
  this Freelens version. To customize a resource list, render your own
  `KubeObjectListLayout` inside a `clusterPage` instead.
- **No standalone `getActiveTheme()`** — read `Renderer.Theme.activeTheme.get()`.

### Process boundaries

- **`Main` is `undefined` in the renderer, and `Renderer` is `undefined` in the
  main process.** Only touch the namespace for the process you're running in.
  Cross-process work goes through IPC or an `ExtensionStore`.
- **The main process has no active cluster.** `ClusterInfo.isActive` is always
  `false` there; active-cluster APIs are renderer-only.

### The KubeObject runtime model

- **No instance methods on `KubeObject` subclasses.** The host hands your
  components plain object copies, not class instances. Instance methods you
  define will not exist at runtime. Use `static` helpers that take the object
  as a parameter, direct `spec`/`status` access, or host base-class methods
  (`getName()`, `getNs()`, `getCreationTimestamp()`, `getSearchFields()`).
- **Don't `as any` around this** — the typed `Spec`/`Status` interfaces exist;
  use them.

### Build / packaging constraints

- **Do not bundle host-provided globals** (`@freelensapp/extensions`, `react`,
  `react-dom`, `react/jsx-runtime`, `mobx`, `mobx-react`, `react-router-dom`).
  Bundling them yields duplicate React/MobX and broken hooks/observables.
  Externalize them to `global.*` (the `globalExternals` shim).
- **You are locked to the host's React 17 and MobX 6** for anything that
  crosses the UI boundary.
- **Output must be CommonJS** for Freelens 1.x hosts.
- **Compatibility is gated** by `engines.freelens` (MAJOR.MINOR, coerced to
  `^MAJOR.MINOR`). An extension outside the range is marked incompatible by the
  host. This is the **only** version negotiation — there is **no**
  inter-extension dependency mechanism or extension-to-extension API-version
  contract. You cannot declare "requires extension X"; each extension is
  independent.
- **The host pnpm-installs your `dependencies` at install time** and loads from
  a `.tgz` (production) or the extension folder. Keep host globals in
  `devDependencies`; everything else is either bundled by your build or
  pnpm-installed. See
  [Getting started §1.7](./01-getting-started.md#17-how-the-host-installs-and-discovers-your-extension).

### IPC caveats

- The published `Main.Ipc` / `Renderer.Ipc` are **abstract classes with no
  concrete instance accessor**, so the intended `Class.method(...)` usage does
  not typecheck cleanly; you may have to fall back to raw Electron IPC (with a
  self-chosen channel prefix and manual cleanup). See
  [Persistence and IPC](./07-persistence-and-ipc.md#the-typecheck-gotcha-important).
- IPC payloads are **sanitized (`toJS`)** — send plain data, not live
  observables or class instances.

### State/store constraints

- `ExtensionStore.fromStore` / `toJSON` **must be synchronous**.
- A store subclass is a **singleton** — create it via the static
  `createInstance` / `getInstanceOrCreate`, and call `loadExtension(this)`
  once from `onActivate` to start syncing.

## 9.3 Rules of thumb

- **Register, don't inject.** Find the right extension point; if none exists,
  the capability isn't exposed.
- **Right process, right namespace.** `Renderer` for UI, `Main` for
  OS/app/direct-k8s, `Common` for shared state and info.
- **Static + data over instances.** For KubeObjects, never rely on instance
  methods; the runtime only has the data.
- **Externalize the host globals; bundle everything else.**
- **Persist through `ExtensionStore`; cross processes through IPC or the store.**
