<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# 7. Persistence and IPC

## 7.1 Persisting data — `Common.Store.ExtensionStore`

`Common.Store.ExtensionStore<T>` (implemented by `BaseExtensionStore`) is the
supported way to persist extension data. It writes JSON to disk under the
extension's data folder and **syncs across the main and renderer processes**
automatically (backed by `conf` + a MobX reaction over messaging channels).

Subclass it, hold observable state, and implement `fromStore`/`toJSON`:

```ts
import { Common } from "@freelensapp/extensions";
import { makeObservable, observable } from "mobx";

interface Model {
  enabled: boolean;
}

export class MyPreferencesStore extends Common.Store.ExtensionStore<Model> {
  @observable enabled = false;

  constructor() {
    super({
      configName: "my-preferences-store",   // file name
      defaults: { enabled: false },
    });
    makeObservable(this);
  }

  fromStore(model: Partial<Model>): void {   // deserialize persisted JSON → state
    this.enabled = model.enabled ?? false;
  }

  toJSON(): Model {                          // serialize state → JSON
    return { enabled: this.enabled };
  }
}
```

Lifecycle (singleton pattern inherited from the base):

```ts
// In your extension's onActivate (both processes bootstrap the same store):
await MyPreferencesStore.getInstanceOrCreate().loadExtension(this);

// Anywhere else, read the singleton:
const prefs = MyPreferencesStore.getInstance<MyPreferencesStore>();
prefs.enabled;   // reactive
```

Key methods:

| Method                              | Purpose                                                    |
| ----------------------------------- | ---------------------------------------------------------- |
| `static createInstance(...)`        | Create the singleton.                                      |
| `static getInstance(strict?)`       | Get it (throws if missing when `strict`).                  |
| `static getInstanceOrCreate(...)`   | Get or create — use in `onActivate`.                       |
| `static resetInstance()`            | Tear it down.                                              |
| `loadExtension(extension)`          | **Start syncing** — call once, passing the extension (`this`). |
| `fromStore(data)` / `toJSON()`      | You implement these (must be synchronous).                 |
| `cwd()`                             | Override this **method** to change the on-disk folder (defaults to `<userData>/extension-store/<storeName>`). |

`ExtensionStoreParams<T>` = `configName` (required), optional `defaults`,
`migrations` (conf-style), `syncOptions`. Note on `cwd`: `loadExtension()`
ignores the `cwd` **constructor param** it is given (kept for backwards
compatibility), but the default `cwd()` method still reads `rawParams.cwd` —
so a `cwd` you pass to the constructor *does* take effect unless you override
the `cwd()` method. Overriding the method is the unambiguous way to relocate
the folder.

Bind a preference `Input` component to the store's observable, and it becomes
a persisted, cross-process, reactive setting.

## 7.2 Inter-process communication (IPC)

Your main and renderer code run in separate processes. Freelens provides
per-extension IPC through `Main.Ipc` (`IpcMain`) and `Renderer.Ipc`
(`IpcRenderer`), both extending `IpcRegistrar`.

### How it's meant to work

- Each subclass is a **singleton**; construct it via
  `MyIpc.createInstance(extension)`, then call instance methods.
- Channels are **auto-namespaced** per extension:
  `extensions@<sha256(extension.id)>:<channel>`, so extensions can't collide.
- Listeners registered through it are **auto-disposed** on disable/uninstall.

Methods:

```ts
// Main.Ipc (IpcMain)
listen(channel, (event, ...args) => any): Disposer;   // subscribe to broadcasts
handle(channel, (event, ...args) => any): void;        // expose an RPC (renderer invokes)
broadcast(channel, ...args): void;                     // fan out to main + all renderer frames

// Renderer.Ipc (IpcRenderer)
listen(channel, (event, ...args) => any): Disposer;    // subscribe to broadcasts
invoke(channel, ...args): Promise<any>;                // call a main-process handle()
broadcast(channel, ...args): void;
```

`broadcast` reaches **both** processes for the same extension and sanitizes
payloads with `toJS`/`sanitizePayload` (so send plain data, not live
observables/class instances).

### The typecheck gotcha (important)

The published `@freelensapp/extensions` type declarations expose `IpcMain` /
`IpcRenderer` as **abstract classes**. There is no published concrete instance
accessor, so writing `Main.Ipc.handle(...)` (a call on the class) **does not
typecheck**, and the intended `MyIpc.createInstance(extension).handle(...)`
path is awkward to satisfy against the shipped `.d.ts`.

Because of this, some extensions (e.g. the `freelens-opencode-extension`)
**bypass the abstraction** and use Electron's raw `electron.ipcMain` /
`electron.ipcRenderer` directly, with a self-chosen channel prefix:

```ts
// main
import { ipcMain } from "electron";
const CHANNEL_PREFIX = "my-extension:";
ipcMain.handle(`${CHANNEL_PREFIX}do-thing`, async (_event, arg) => { /* … */ });

// renderer
import { ipcRenderer } from "electron";
const result = await ipcRenderer.invoke("my-extension:do-thing", arg);
```

Both approaches sit on the same underlying Electron `ipcMain`/`ipcRenderer`.
The trade-off:

| | `Main.Ipc` / `Renderer.Ipc` | Raw `electron.ipcMain` / `ipcRenderer` |
| --- | --- | --- |
| Channel namespacing | automatic (`extensions@<hash>:`) | you must pick a unique prefix |
| Cleanup on disable/uninstall | automatic (disposers) | you manage it manually |
| Typechecks against shipped d.ts | problematic | fine |

**Recommendation:** prefer the built-in abstraction if you can get it to
typecheck for your version; otherwise use raw Electron IPC with a
collision-safe prefix (e.g. your extension name) and remove your handlers in
`onDeactivate`.
