<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# 8. Catalog, navigation, theming, app info

This page collects the remaining renderer/common APIs: reading and extending
the catalog, navigating the UI, reacting to the active theme, and app-level
info and utilities.

## 8.1 Catalog

The **catalog** is Freelens's registry of entities (clusters, weblinks, and
whatever extensions add). You can both **read** it and **extend** it.

### Reading (`Renderer.Catalog`)

| Export                         | Purpose |
| ------------------------------ | ------- |
| `catalogEntities`              | Entity registry: `.activeEntity`, `.entities` (Map), `.getById(id)`, `.getItemsForApiKind<T>(apiVersion, kind)`, `.getItemsForCategory<T>(category)`, `.addOnBeforeRun(hook): Disposer`. |
| `catalogCategories`            | The category registry (same object as `Main.Catalog.catalogCategories`); register your `CatalogCategory` subclass here. |
| `activeCluster`                | The active `KubernetesCluster` entity. |
| `getAllClusters()`             | `() => ClusterInfo[]`. |
| `getClusterById(id)`           | `(ClusterId) => ClusterInfo \| undefined`. |
| `getActiveCluster()`           | `() => ClusterInfo \| undefined`. |

`ClusterInfo` (`Common.Clusters`):

```ts
interface ClusterInfo {
  id: ClusterId;
  name: string;
  kubeConfigPath: string;
  contextName: string;
  status: ClusterConnectionStatus;   // CONNECTING | CONNECTED | DISCONNECTED | DISCONNECTING
  labels: Record<string, string>;
  isActive: boolean;
  metadata?: ClusterMetadata;
}

interface ClusterMetadata {          // extended, may be absent
  distribution?: string;             // "eks" | "gke" | "aks" | "minikube" | …
  kubernetesVersion?: string;        // "1.28.0"
  nodeCount?: number;
  lastConnected?: Date;
  connectionError?: string;
}
```

### Extending (`Common.Catalog`)

Subclass the abstract base classes to publish new entity kinds:

```ts
abstract class CatalogEntity<Metadata, Status, Spec> {
  abstract apiVersion: string;
  abstract kind: string;
  metadata; status; spec;              // observable
  getId(); getName(); getSource(); isEnabled();
  onRun?; onContextMenuOpen?; onSettingsOpen?;   // optional behavior hooks
}

abstract class CatalogCategory {
  abstract apiVersion; kind; metadata; spec;
  getId(); getName(); getBadge();
  addMenuFilter(fn): Disposer;
}
```

Concrete built-ins are re-exported: `KubernetesCluster`, `GeneralEntity`,
`WebLink`, plus `kubernetesClusterCategory`. Register a category via the
`catalogCategories` registry — it is exposed on **both** namespaces
(`Main.Catalog.catalogCategories` and `Renderer.Catalog.catalogCategories`) —
then publish entities with `Main.LensExtension.addCatalogSource(...)`
(see [Main registrations](./04-main-registrations.md)).

To actually subclass `CatalogEntity` you also need `CatalogEntityData`
(the constructor argument: `{ metadata, status, spec }`) and, for the
category, the `categoryVersion(name, EntityClass)` helper — both re-exported
from `Common.Catalog`.

## 8.2 Navigation (`Renderer.Navigation`)

| Export                | Purpose |
| --------------------- | ------- |
| `navigate`            | Navigate to a URL/route. |
| `createPageParam`     | Build a typed URL page parameter (`PageParam`/`PageParamInit`). |
| `getDetailsUrl`       | URL for a kube-object details drawer. |
| `getMaybeDetailsUrl`  | Same, nullable input. |
| `showDetails`         | Open a kube-object details drawer. |
| `hideDetails`         | Close it. |
| `showEntityDetails` / `hideEntityDetails` | Catalog-entity details drawer. |
| `isActiveRoute`       | Whether a route is currently active. |

Types: `URLParams`, `PageParam`, `PageParamInit`.

## 8.3 Theming (`Renderer.Theme`)

Only two symbols:

```ts
Renderer.Theme.activeTheme;   // IComputedValue<ReadonlyDeep<LensTheme>>

interface LensTheme {
  name: string;
  type: "dark" | "light";
  colors: Record<LensColorName, string>;              // finite union of theme color keys
  terminalColors: Partial<Record<TerminalColorName, string>>;  // required (xterm ITheme keys)
  description: string;
  author: string;
  monacoTheme: MonacoTheme;
  isDefault?: boolean;
}
```

Read the current theme reactively with `activeTheme.get()`. There is **no**
`getActiveTheme()` function. To adapt your UI to the theme, prefer the theme
CSS custom properties (`var(--…)`) over reading the object.

## 8.4 App info (`Common.App`)

| Member                        | Type / value | Meaning |
| ----------------------------- | ------------ | ------- |
| `App.version`                 | `string`     | App/build version. |
| `App.appName`                 | `string`     | App name. |
| `App.isWindows / isMac / isLinux` | `boolean` | Current OS. |
| `App.isFlatpak / isSnap`      | `boolean`    | Packaging format. |
| `App.lensBuildEnvironment`    | `string`     | `production` / `development`. |
| `App.issuesTrackerUrl`        | `string`     | Issue tracker URL. |
| `App.getEnabledExtensions()`  | `InstalledExtension[]` | Currently enabled extensions. |
| `App.Preferences.getKubectlPath()` | `string?` | Configured kubectl path. |

## 8.5 Utilities (`Common.Util`)

`Common.Util` spreads all of `@freelensapp/utilities` plus a few helpers:

- `openExternal(url)` / `openBrowser(url)` — open a link in the OS browser.
- `getAppVersion()`.
- URL builders (`buildURL`, `urlBuilderFor`), `WrappedAbortController` /
  `isAbortError`, `HashSet` / `ObservableHashSet`, `Result` / `AsyncResult`,
  and other general utilities.

## 8.6 Logging (`Common.logger`)

```ts
import { Common } from "@freelensapp/extensions";
Common.logger.info("…");
Common.logger.warn("…");
Common.logger.error("…", err);
Common.logger.debug("…");
```

## 8.7 Event bus (`Common.EventBus`)

`Common.EventBus.appEventBus` is an app-wide `EventEmitter<AppEvent>` you can
emit to and subscribe on. Types: `AppEvent`, `EventEmitter`,
`EventEmitterCallback`, `EventEmitterOptions`.

## 8.8 Proxy (`Common.Proxy`)

`Common.Proxy.resolveSystemProxy(url)` resolves the system proxy for a URL
(wraps Electron's `session.resolveProxy`).
