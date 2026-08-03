<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# 4. Main registrations

The main process has a much smaller extension surface than the renderer,
because it has no UI. What it offers is OS/app integration and direct
(no-IPC) Kubernetes access.

Everything here is set on your `Main.LensExtension` subclass. The whole `Main`
namespace is `undefined` in the renderer — only touch it from your main entry.

## 4.1 Application menu

```ts
type MenuRegistration = {
  parentId: string;                         // menu bar section to attach under
  visible?: IComputedValue<boolean> | boolean;
} & Omit<Electron.MenuItemConstructorOptions, "visible">;
```

Because it extends Electron's `MenuItemConstructorOptions`, you get
`label`, `click`, `submenu`, `accelerator`, `type`, `role`, etc. Assign an
array (or a MobX computed array) to `appMenus`:

```ts
appMenus = [{
  parentId: "help",
  label: "My Extension Docs",
  click: () => { /* … */ },
}];
```

## 4.2 System tray menu

```ts
interface TrayMenuRegistration {
  id?: string;
  label?: string | IComputedValue<string>;
  toolTip?: string;
  type?: "normal" | "separator" | "submenu";
  enabled?: boolean | IComputedValue<boolean>;
  visible?: IComputedValue<boolean>;
  click?: (menuItem: TrayMenuRegistration) => void;
  submenu?: TrayMenuRegistration[];
}
```

Assign to `trayMenus`.

## 4.3 Terminal shell environment

```ts
interface ShellEnvContext {
  catalogEntity: CatalogEntity;
}

type ShellEnvModifier = (
  ctx: ShellEnvContext,
  env: Record<string, string | undefined>,
) => Record<string, string | undefined>;
```

Set `terminalShellEnvModifier` to inject/modify environment variables for
terminals Freelens opens (e.g. per-cluster tooling). **Return** the (possibly
mutated) `env`.

## 4.4 Catalog sources

Publish your own catalog entities from the main process:

```ts
addCatalogSource(
  id: string,
  source: IObservableArray<CatalogEntity> | IComputedValue<CatalogEntity[]>,
): void;

removeCatalogSource(id: string): void;
```

To define the entity/category types themselves, subclass `Common.Catalog`
base classes — see [Catalog](./08-catalog-navigation-theme.md).

## 4.5 Navigation

```ts
await this.navigate(pageId?, params?, frameId?);
```

Navigates a renderer frame to one of your extension pages. The `Main.Navigation`
namespace also exposes a single low-level `navigate(url: string)` function.

## 4.6 Power / lifecycle events (`Main.Power`)

Register callbacks for OS power and app lifecycle events:

| Export             | Purpose                                    |
| ------------------ | ------------------------------------------ |
| `onSuspend`        | System is going to sleep.                  |
| `onResume`         | System woke from sleep.                    |
| `onShutdown`       | System reboot/shutdown.                    |

Each takes a listener and returns a disposer.

## 4.7 Direct Kubernetes access (`Main.K8s`)

The main process talks to clusters directly (no IPC round-trip). The functions
mirror `Renderer.K8s` — see [Kubernetes API](./06-kubernetes-api.md#63-clusters-cluster-query-functions):
`queryCluster`, `queryClusters`, `queryAllClusters`, `getResource`,
`applyOnCluster`, `deleteOnCluster`, `patchOnCluster`.

`Main.K8sApi` re-exports the same `KubeApi`/`KubeObjectStore`/
`LensExtensionKubeObject` classes as the renderer.

## 4.8 Main-side catalog reads (`Main.Catalog`)

```ts
Main.Catalog.getAllClusters(): ClusterInfo[];
Main.Catalog.getClusterById(id): ClusterInfo | undefined;
Main.Catalog.catalogEntities.getItemsForApiKind(apiVersion, kind): CatalogEntity[];
Main.Catalog.catalogCategories;   // category registry — register your CatalogCategory subclass
                                  // (also available as Renderer.Catalog.catalogCategories)
```

> The main process does **not** track an "active" cluster — `isActive` on
> `ClusterInfo` is always `false` there. Active-cluster APIs live in the
> renderer (`Renderer.Catalog`).
