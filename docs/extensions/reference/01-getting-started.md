<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# 1. Getting started

An extension is a normal npm package. It is built into CJS bundles, packed into
a `.tgz`, and installed into Freelens (Preferences → Extensions, or by dropping
the tarball in the extensions folder). This page covers the project layout,
`package.json` manifest, and the build configuration that makes it all work.

## 1.1 Project layout

The example extension uses this layout (yours can differ, but this is the
idiomatic shape):

```
src/
  main/index.ts            # main-process entry (CJS)  → out/main/index.js
  renderer/index.tsx       # renderer-process entry (CJS) → out/renderer/index.js
  renderer/api/<crd>/*.ts  # KubeObject/Api/Store model classes, one per CRD version
  renderer/pages/*.tsx     # cluster page components
  renderer/details/*.tsx   # detail-panel components
  renderer/menus/*.tsx     # object-menu action components
  renderer/icons/*.tsx     # SVG icon wrappers
  renderer/preferences/*   # preference input/hint components
  common/store/*           # shared ExtensionStore subclass + model
  common/utils.ts          # shared helpers
```

Build output goes to `out/`. Only `out/**/*` is shipped (`files` field).

## 1.2 The manifest (`package.json`)

Your `package.json` **is** the extension manifest. The host parses it into a
`LensExtensionManifest` (`installed-extension.ts`).

```jsonc
{
  "name": "@myorg/my-extension",
  "version": "0.1.0",
  "description": "What it does",
  "main": "out/main/index.js",        // main-process entry (optional)
  "renderer": "out/renderer/index.js", // renderer-process entry (optional)
  "files": ["out/**/*"],
  "engines": {
    "node": ">= 22.0.0",
    "freelens": "^1.8.0"              // compatibility gate — MAJOR.MINOR only
  },
  "storeName": "my-extension",         // optional: persisted-data folder name
  "publishConfig": { "access": "public", "registry": "https://registry.npmjs.org/" }
}
```

Manifest fields the host reads (`LensExtensionManifest` in
`installed-extension.ts`):

| Field                | Type                 | Meaning                                                              |
| -------------------- | -------------------- | ------------------------------------------------------------------- |
| `name`               | `string`             | Extension id/name.                                                  |
| `version`            | `string`             | Extension version.                                                  |
| `description`        | `string?`            | Shown in the extensions list.                                       |
| `main`               | `string?`            | Path to the compiled main-process bundle. Omit for no main code.    |
| `renderer`           | `string?`            | Path to the compiled renderer bundle. Omit for no renderer code.    |
| `engines.freelens`   | `string`             | Semver range; only MAJOR.MINOR is considered for `isCompatible`.    |
| `storeName`          | `string?`            | Folder name for persisted data; defaults to `name`. Set it so a rename doesn't orphan stored data. |
| `publishConfig`      | `Partial<Record<string,string>>?` | npm publish config; carried through into the parsed manifest. |

Each entry module must `export default` the appropriate extension subclass.

## 1.3 Host-provided globals (do not bundle these)

These packages are provided by the Freelens host at runtime as globals. You
must **externalize** them, never bundle them, or you get duplicate React/MobX
instances and broken hooks/observables:

| Package                  | Host global               | Available in |
| ------------------------ | ------------------------- | ------------ |
| `@freelensapp/extensions`| `global.LensExtensions`   | main + renderer |
| `mobx`                   | `global.Mobx`             | main + renderer |
| `react`                  | `global.React`            | renderer     |
| `react/jsx-runtime`      | `global.ReactJsxRuntime`  | renderer     |
| `react-dom`              | `global.ReactDom`         | renderer     |
| `mobx-react`             | `global.MobxReact`        | renderer     |
| `react-router-dom`       | `global.ReactRouterDom`   | renderer     |

Because they are never bundled, declare them under **`devDependencies`** (the
example extension has an empty `dependencies` — everything host-provided is
dev-only). Any other dependency you use **is** bundled into `out/`.

## 1.4 Build configuration (`electron-vite`)

The example builds with `electron-vite` (Vite + Rollup/Rolldown). Key points:

- **Output format is CommonJS** for Freelens 1.x hosts.
- Decorators must be enabled for MobX `@observable`: the config turns on both
  `oxc.decorator { legacy: true, emitDecoratorMetadata: true }` and the Babel
  `@babel/plugin-proposal-decorators` (`version: "2023-05"`).
- **`globalExternals` shim** — a small custom Rollup plugin rewrites imports of
  the host packages above into ESM shims that read from the corresponding
  `global.*`, emitting explicit named exports:

  ```js
  const __m = global.React;
  export default __m;
  export const forwardRef = __m.forwardRef;   // …and every other named export
  ```

  The plugin discovers export names at config time by `require`-ing the real
  module (falling back to source parsing for browser-only packages like
  `@freelensapp/extensions`). This replaces the older `module.exports =
  global.React` shim, which only exposed `default` and caused intermittent
  `[MISSING_EXPORT]` failures under Rolldown.
- `VITE_PRESERVE_MODULES=false` produces a single-file production bundle
  (`build:production`); the default dev build preserves modules.
- SCSS modules generate `*.module.d.scss.ts` typing files via
  `vite-plugin-sass-dts`. Clean them with `pnpm clean:dts` when SCSS changes.

## 1.5 Common commands (from the example extension)

```bash
pnpm type:check       # tsc --noEmit
pnpm biome:check      # TS/TSX/JS/JSON/CSS/SCSS/HTML
pnpm trunk:check      # Markdown/YAML/etc.
pnpm test:unit        # vitest
pnpm build            # type-check + electron-vite build
pnpm build:production # single-file bundle (no preserveModules)
pnpm pack:dev         # bump prerelease, build, produce .tgz for install
pnpm clean:all        # dts + node_modules + out + tgz
```

## 1.6 Minimal entries

**Main** (`src/main/index.ts`) — often just bootstraps a store:

```ts
import { Main } from "@freelensapp/extensions";
import { MyStore } from "../common/store";

export default class MyMain extends Main.LensExtension {
  async onActivate() {
    await MyStore.getInstanceOrCreate().loadExtension(this);
  }
}
```

**Renderer** (`src/renderer/index.tsx`) — registers UI:

```tsx
import { Renderer } from "@freelensapp/extensions";

export default class MyRenderer extends Renderer.LensExtension {
  clusterPages = [ /* … */ ];
  clusterPageMenus = [ /* … */ ];
  kubeObjectDetailItems = [ /* … */ ];
  async onActivate() { /* bootstrap */ }
}
```

## 1.7 How the host installs and discovers your extension

Understanding this explains *why* the build produces a `.tgz`, why declared
`dependencies` behave differently from bundled code, and where to look when an
extension "won't load."

- **Discovery folder.** The host watches `~/.freelens/extensions`
  (`extension-discovery.ts` → `localFolderPath`). Any subdirectory containing a
  `package.json` is treated as an extension. Installing via Preferences →
  Extensions (or dropping a tarball) lands here.
- **Dependency install.** When an extension appears, the host runs **pnpm** to
  install that extension's own npm `dependencies` into a shared `node_modules`
  (`installExtension` → `forkPnpm("install", "--prefer-offline", "--prod",
  "--force")`); uninstall runs `forkPnpm("uninstall", "--force", <name>)`. This
  is the key reason the split matters:
  - **Host-provided globals** (react, mobx, `@freelensapp/extensions`, …) →
    `devDependencies`, externalized, never installed or bundled.
  - **Your other runtime deps** → they are **bundled into `out/`** by the
    build, so in practice the example extension keeps `dependencies` empty.
    Anything you *do* leave in `dependencies` will be pnpm-installed at
    install time.
- **Folder vs `.tgz`.** In a **production** host, if a
  `<name>-<version>.tgz` sits next to the manifest it is loaded from the
  tarball (`absolutePath = npmPackage`); otherwise the folder is used
  (`loadExtensionFromFolder`). This is why `pnpm pack:dev` produces a `.tgz`.
- **Compatibility gate.** `isCompatibleExtension` coerces
  `engines.freelens` to `^MAJOR.MINOR` and compares against the host version.
  This is the **only** version negotiation — there is **no** inter-extension
  dependency or API-version mechanism. An extension outside the range is marked
  incompatible and not activated.

Next: [Extension classes and lifecycle](./02-extension-classes.md).
