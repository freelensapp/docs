<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# 2. Extension classes and lifecycle

All three namespaces expose a `LensExtension` class. You subclass one per
process:

- `Main.LensExtension` (= `LensMainExtension`) — main-process code.
- `Renderer.LensExtension` (= `LensRendererExtension`) — renderer/UI code.
- `Common.LensExtension` — **type only**; the shared base. You never subclass
  this directly.

Both concrete classes extend the shared base `LensExtension`
(`lens-extension.ts`).

## 2.1 Shared base: identity and lifecycle

Available on both `Main.LensExtension` and `Renderer.LensExtension`:

### Identity (read-only getters)

| Member                 | Type                | Meaning                                                    |
| ---------------------- | ------------------- | ---------------------------------------------------------- |
| `id`                   | `string`            | Unique extension id.                                       |
| `name`                 | `string`            | `manifest.name`.                                           |
| `version`              | `string`            | `manifest.version`.                                        |
| `description`          | `string?`           | `manifest.description`.                                    |
| `storeName`            | `string`            | `manifest.storeName ?? name`; the persisted-data folder.   |
| `manifest`             | `LensExtensionManifest` | Parsed `package.json`.                                  |
| `manifestPath`         | `string`            | Path to the (symlinked) manifest.                          |
| `sanitizedExtensionId` | `string`            | `name` with `@`/`/` stripped; used for tab/DOM ids.        |
| `isEnabled`            | `boolean` (computed)| Whether the extension is currently enabled.                |

> There is **no** `isBundled` on the instance. Bundled status is tracked by the
> loader, not the extension object.

### Lifecycle methods

| Method / hook          | Signature                              | Notes                                                                 |
| ---------------------- | -------------------------------------- | --------------------------------------------------------------------- |
| `onActivate()`         | `protected (): Promise<void> \| void`  | **Override this.** Called when the extension activates. Bootstrap stores, register catalog sources, etc. May be `async`. |
| `onDeactivate()`       | `protected (): Promise<void> \| void`  | **Override this.** Called on disable/teardown. Undo side effects. May be `async`. |
| `getExtensionFileFolder()` | `(): Promise<string>`             | A per-extension writable folder on disk (obfuscated path).            |
| `enable()` / `disable()` / `activate()` | host-driven           | Managed by the host; you generally don't call these.                  |

`disable()` runs a set of registered disposers automatically, so anything you
register through the built-in APIs (e.g. `Ipc.listen`, catalog sources via the
provided helpers) is cleaned up for you.

### Protocol handlers (`freelens://` URLs)

Both classes expose:

```ts
protocolHandlers: ProtocolHandlerRegistration[];

interface ProtocolHandlerRegistration {
  pathSchema: string;                       // e.g. "/settings/:section"
  handler: (params: {
    search: Record<string, string | undefined>;
    pathname: Record<string, string | undefined>;
    tail?: string;
  }) => void;
}
```

Push handlers here to respond to deep links like
`freelens://extension/<name>/…`.

## 2.2 `Main.LensExtension` (main process)

Registration fields (arrays or MobX computed arrays), and helper methods:

| Member                    | Type                                                             | Purpose |
| ------------------------- | ---------------------------------------------------------------- | ------- |
| `appMenus`                | `MenuRegistration[] \| IComputedValue<MenuRegistration[]>`       | Items in the application menu bar. |
| `trayMenus`               | `TrayMenuRegistration[] \| IComputedValue<TrayMenuRegistration[]>` | Items in the OS system-tray menu. |
| `terminalShellEnvModifier?` | `ShellEnvModifier`                                             | Mutate env vars for terminals opened by Freelens. |
| `navigate(pageId?, params?, frameId?)` | `async`                                              | Navigate a renderer frame to an extension page. |
| `addCatalogSource(id, source)` | `(string, IObservableArray<CatalogEntity> \| IComputedValue<CatalogEntity[]>)` | Publish catalog entities. |
| `removeCatalogSource(id)` | `(string)`                                                       | Remove a previously-added source. |

See [Main registrations](./04-main-registrations.md) for the type shapes.

## 2.3 `Renderer.LensExtension` (renderer process)

This is the big one — nearly all UI extension points live here. Every field is
an array you assign as a class field (defaults to `[]`):

| Field                        | Registration type                          | Adds… |
| ---------------------------- | ------------------------------------------ | ----- |
| `globalPages`                | `PageRegistration[]`                       | Full pages outside cluster context (root frame). |
| `clusterPages`               | `PageRegistration[]`                       | Full pages inside a cluster (with cluster sidebar). |
| `clusterPageMenus`           | `ClusterPageMenuRegistration[]`            | Sidebar entries (and sub-items) in a cluster. |
| `clusterFrameComponents`     | `ClusterFrameChildComponent[]`             | Always-mounted components in the cluster frame (overlays/dialogs). |
| `appPreferences`             | `AppPreferenceRegistration[]`              | Preference control blocks. |
| `appPreferenceTabs`          | `AppPreferenceTabRegistration[]`           | New tabs in the Preferences window. |
| `entitySettings`             | `EntitySettingRegistration[]`              | Settings panes for a catalog entity (e.g. per-cluster). |
| `statusBarItems`             | `StatusBarRegistration[]`                  | Items in the bottom status bar. |
| `kubeObjectDetailItems`      | `KubeObjectDetailRegistration[]`           | Panels in a k8s object's details drawer. |
| `kubeObjectMenuItems`        | `KubeObjectMenuRegistration[]`             | Actions in a k8s object's context/action menu. |
| `kubeObjectHandlers`         | `KubeObjectHandlerRegistration[]`          | Hooks fired when a k8s object context menu opens. |
| `kubeWorkloadsOverviewItems` | `WorkloadsOverviewDetailRegistration[]`    | Sections on the Workloads Overview page. |
| `commands`                   | `CommandRegistration[]`                    | Command-palette commands. |
| `welcomeMenus`               | `WelcomeMenuRegistration[]`                | Actions on the Welcome page. |
| `catalogEntityDetailItems`   | `CatalogEntityDetailRegistration[]`        | Detail panels in the catalog entity detail view. |
| `topBarItems`                | `TopBarRegistration[]`                     | Items in the application top bar. |
| `additionalCategoryColumns`  | `AdditionalCategoryColumnRegistration[]`   | Columns in a catalog category list table. |
| `customCategoryViews`        | `CustomCategoryViewRegistration[]`         | Replacement view for a whole catalog category. |

Plus helper methods on the renderer subclass:

```ts
navigate(pageId?: string, params?: object): Promise<void>;   // go to one of THIS extension's pages
navigateToPreferences(): void;                                // open this extension's preferences tab
addCatalogFilter(fn: EntityFilter): Disposer;                 // filter which catalog entities show
addCatalogCategoryFilter(fn: CategoryFilter): Disposer;       // filter which catalog categories show
isEnabledForCluster(cluster: KubernetesCluster): Promise<boolean> | boolean;
```

- `navigate` resolves the target against this extension's own
  `globalPages`/`clusterPages` and is a no-op if the `pageId` doesn't match one
  of them. (Note: the renderer signature is `(pageId?, params?)` — it has no
  `frameId` argument, unlike `Main.navigate`.)
- `navigateToPreferences()` always opens **this** extension's preferences tab,
  never the generic or another extension's preferences.
- `addCatalogFilter` / `addCatalogCategoryFilter` register filter functions and
  return a `Disposer`; the filter is also auto-removed when the extension is
  disabled.

The last one, `isEnabledForCluster(cluster)`, returns `false` to disable this
extension's cluster UI for a given cluster.

> **Deprecated.** In the current source `isEnabledForCluster` is marked
> `@deprecated` — prefer the per-registration `enabled` / `visible` properties
> combined with the active cluster (`Renderer.Catalog.activeCluster`). It still
> works, but new code should gate on those instead.

See [Renderer registrations](./03-renderer-registrations.md) for every type
shape and where each renders.

## 2.4 Namespace → class mapping

| Author import                   | Concrete class          |
| ------------------------------- | ----------------------- |
| `Main.LensExtension`            | `LensMainExtension`     |
| `Renderer.LensExtension`        | `LensRendererExtension` |
| `Common.LensExtension` (type)   | `LensExtension` (base)  |
