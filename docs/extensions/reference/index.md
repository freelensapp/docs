<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# Freelens Extension Development Reference

This is a developer reference for building extensions for the Freelens
Kubernetes IDE. It answers the central question:

> **As an extension developer, what can I do — and what can I not do?**

Everything an extension can do flows through a single package,
`@freelensapp/extensions`, which exposes three namespaces:

| Namespace  | Runs in            | You use it to…                                              |
| ---------- | ------------------ | ----------------------------------------------------------- |
| `Common`   | both processes     | Persist data, read app info, subclass catalog entities, log |
| `Main`     | main process only  | Register OS/app menus, tray items, power hooks, run k8s ops directly |
| `Renderer` | renderer only      | Register UI (pages, menus, detail panels…), use React components, query k8s |

An extension is an npm package whose `package.json` names a `main` and/or a
`renderer` entry module. Each module `export default` a subclass of
`Main.LensExtension` or `Renderer.LensExtension`. You register features by
setting **class-field arrays** on that subclass (not by calling methods).

```
export default class MyExtension extends Renderer.LensExtension {
  clusterPages = [ /* … */ ];
  kubeObjectDetailItems = [ /* … */ ];
  async onActivate() { /* bootstrap */ }
}
```

## Table of contents

1. [Getting started: project layout, build, entry points](./01-getting-started.md)
2. [Extension classes and lifecycle](./02-extension-classes.md)
3. [Renderer registrations (all UI extension points)](./03-renderer-registrations.md)
4. [Main registrations (menus, tray, power, shell env)](./04-main-registrations.md)
5. [UI components (`Renderer.Component`)](./05-ui-components.md)
6. [Kubernetes API: CRDs, stores, cluster queries](./06-kubernetes-api.md)
7. [Persistence (`ExtensionStore`) and IPC](./07-persistence-and-ipc.md)
8. [Catalog, navigation, theming, app info](./08-catalog-navigation-theme.md)
9. [**Capabilities and limitations — the CAN / CANNOT list**](./09-capabilities-and-limitations.md)

## Source of truth

The API surface is defined in the Freelens monorepo under
`packages/core/src/extensions/`:

- `common-api/` → the `Common` namespace
- `main-api/` → the `Main` namespace
- `renderer-api/` → the `Renderer` namespace
- `extension-api.ts` (in `packages/extensions/src/`) → the published entry that
  re-exports the three namespaces from `@freelensapp/core`.

At runtime these symbols are **not bundled** into your extension; the host
assigns them to `globalThis.FreelensExtensionApi` at startup and your build
resolves them from host globals (see
[getting started](./01-getting-started.md)).

The canonical, working example is the `freelens-example-extension` repo, which
implements a custom CRD (`Example`) end to end: model, list page, detail panel,
object-menu action, preference, and preferences store.

> Reference version: `@freelensapp/extensions ^1.10.x`, Freelens `>= 1.8.0`,
> React 17, MobX 6.
