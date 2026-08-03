<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# 3. Renderer registrations

Every UI extension point is a field on your `Renderer.LensExtension` subclass.
This page documents the shape of each registration type and where it renders.
All `components.*` fields take `React.ComponentType`.

Common conventions:

- `visible?: IComputedValue<boolean>` — MobX-reactive show/hide.
- `priority?: number` / `orderNumber?: number` — ordering (default 50 where used).
- `apiVersions: string[]` + `kind: string` — match against a k8s object/entity.
- `id?: string` — usually optional; auto-derived when omitted.

---

## 3.1 Pages

```ts
interface PageRegistration {
  id?: string;                    // unique per extension; part of the URL
  params?: PageParams;            // typed page URL params
  components: { Page: React.ComponentType<any> };
  enabled?: IComputedValue<boolean>;
}

interface PageTarget {
  extensionId?: string;
  pageId?: string;
  params?: PageParams;
}
```

- `globalPages` render in the **root frame** (no cluster context).
- `clusterPages` render **inside a cluster frame**, alongside the cluster sidebar.

A page is reached via a sidebar menu entry (`clusterPageMenus`) whose `target`
points at the page `id`, or programmatically via navigation.

## 3.2 Cluster sidebar menu

```ts
interface ClusterPageMenuRegistration {
  id?: string;
  parentId?: string;              // nest under another menu item to make a sub-item
  target?: PageTarget;            // page to navigate to on click
  title: React.ReactNode;         // source type is StrictReactNode, not plain string
  components: { Icon?: React.ComponentType<IconProps> | null };
  visible?: IComputedValue<boolean>;
  orderNumber?: number;
}
```

Example (from the example extension):

```tsx
clusterPageMenus = [{
  id: "example",
  title: Example.crd.title,
  target: { pageId: "example" },
  components: { Icon: ExampleIcon },
}];
```

## 3.3 Always-mounted cluster components

```ts
interface ClusterFrameChildComponent {
  id: string;
  Component: React.ElementType;
  shouldRender: IComputedValue<boolean>;
}
```

Use for dialogs/overlays that must exist regardless of the current page.

## 3.4 Preferences

```ts
interface AppPreferenceRegistration {
  title: string;
  id?: string;
  showInPreferencesTab?: string;   // target a tab by id
  components: {
    Hint: React.ComponentType<any>;
    Input: React.ComponentType<any>;
  };
}

interface AppPreferenceTabRegistration {
  title: string;
  id: string;
  orderNumber?: number;
  visible?: IComputedValue<boolean>;
}
```

`appPreferences` adds a control block; `appPreferenceTabs` adds a whole tab to
the Preferences window. The `Input` component typically binds to an
[`ExtensionStore`](./07-persistence-and-ipc.md) observable.

## 3.5 Entity settings

```ts
interface EntitySettingRegistration {
  apiVersions: string[];
  kind: string;
  title: string;
  components: { View: React.ComponentType<{ entity: CatalogEntity }> };
  source?: string;
  id?: string;                     // default btoa(title)
  priority?: number;               // default 50
  group?: string;                  // default "Extensions"
}
```

Adds a settings pane for a catalog entity — e.g. a per-cluster settings page.

## 3.6 Status bar

```ts
interface StatusBarRegistration {
  components?: {
    Item: React.ComponentType<StatusBarItemProps>;   // StatusBarItemProps = {}
    position?: "left" | "right";                      // default "right"
  };
  visible?: IComputedValue<boolean>;
  item?: React.ReactNode | (() => React.ReactNode);   // @deprecated, use components
}
```

## 3.7 Kube object detail panels

```ts
interface KubeObjectDetailRegistration<T extends KubeObject = KubeObject> {
  kind: string;
  apiVersions: string[];
  components: { Details: React.ComponentType<KubeObjectDetailsProps<T>> };
  priority?: number;
  visible?: IComputedValue<boolean>;
}

interface KubeObjectDetailsProps<T extends KubeObject> {
  className?: string;
  object: T;
}
```

Adds a panel to the details drawer of any k8s object matching `kind` +
`apiVersions`. Your `Details` component receives the object as `props.object`.
Example:

```tsx
kubeObjectDetailItems = [{
  kind: Example.kind,
  apiVersions: Example.crd.apiVersions,
  priority: 10,
  components: {
    Details: (props: Renderer.Component.KubeObjectDetailsProps<any>) =>
      <ExampleDetails {...props} extension={this} />,
  },
}];
```

## 3.8 Kube object menu items

```ts
interface KubeObjectMenuRegistration<Props extends KubeObjectMenuItemProps = any> {
  kind: string;
  apiVersions: string[];
  components: { MenuItem: React.ComponentType<Props> };
  visible?: IComputedValue<boolean>;
}

interface KubeObjectMenuItemProps<Object extends KubeObject = KubeObject> {
  object: Object;
  toolbar?: boolean;
}
```

Adds action items to an object's context/action menu.

## 3.9 Kube object context-menu handlers

For dynamic menu items (computed when the menu opens):

```ts
type KubeObjectHandlerRegistration = { apiVersions: string[]; kind: string }
  & RequireAtLeastOne<{ onContextMenuOpen: KubeObjectOnContextMenuOpen }>;

// KubeObjectOnContextMenuOpen is the FUNCTION type; its argument is typed
// KubeObjectOnContextMenuOpenContext:
type KubeObjectOnContextMenuOpen = (ctx: KubeObjectOnContextMenuOpenContext) => void;

interface KubeObjectOnContextMenuOpenContext {
  readonly menuItems: IObservableArray<KubeObjectContextMenuItem>;
  navigate: (location: string) => void;
}

interface KubeObjectContextMenuItem {
  id?: string;
  icon: string | BaseIconProps;   // string = Material icon name shorthand
  title: string;
  onClick: (obj: KubeObject) => void;
}
```

Push items into `ctx.menuItems` when the menu opens.

## 3.10 Workloads Overview sections

```ts
interface WorkloadsOverviewDetailRegistration {
  components: { Details: React.ComponentType<{}> };
  priority?: number;
  visible?: IComputedValue<boolean>;
}
```

## 3.11 Command palette

```ts
interface CommandRegistration {
  id: string;                                        // globally unique
  title: string | ((ctx: CommandContext) => string);
  action: (ctx: CommandActionContext) => void;
  isActive?: (ctx: CommandContext) => boolean;       // default () => true
  scope?: "global" | "entity";                       // @deprecated, use isActive
}

interface CommandContext { readonly entity: CatalogEntity }
interface CommandActionContext extends CommandContext {
  navigate: (url: string, opts?: { forceRootFrame?: boolean }) => void;
}
```

## 3.12 Welcome menu

```ts
interface WelcomeMenuRegistration {
  title: string | (() => string);
  icon: string;
  click: () => void | Promise<void>;
}
```

## 3.13 Catalog entity detail panels

```ts
interface CatalogEntityDetailRegistration<T extends CatalogEntity> {
  kind: string;
  apiVersions: string[];
  components: { Details: React.ComponentType<{ entity: T }> };
  priority?: number;
}
```

## 3.14 Top bar

```ts
interface TopBarRegistration {
  components: { Item: React.ComponentType };
}
```

## 3.15 Catalog category columns and views

```ts
interface CategoryColumnRegistration {
  priority?: number;                                    // default 50
  id: string;                                           // unique to your extension
  renderCell: (entity: CatalogEntity) => React.ReactNode;
  titleProps: { className?: string; title: React.ReactNode; "data-testid"?: string };
  sortCallback?: (entity: CatalogEntity) => string | number | (string | number)[];
  searchFilter?: (entity: CatalogEntity) => string | string[];
}

interface AdditionalCategoryColumnRegistration extends CategoryColumnRegistration {
  kind: string;    // e.g. "KubernetesCluster"
  group: string;   // e.g. "entity.k8slens.dev"
}

interface CustomCategoryViewRegistration {
  kind: string;
  group: string;
  priority?: number;                                    // default 50
  components: { View: React.ComponentType<{ category: CatalogCategory }> };
}
```

`additionalCategoryColumns` adds columns to an existing category list;
`customCategoryViews` replaces the whole view for a category.

---

## Note: two registrations that do NOT exist

Older Lens tutorials mention `kubeObjectStatusTexts` and
`kubeObjectListLayoutColumns`. **These are not present in this Freelens
version** — do not use them. To add columns to a resource list, render your own
`KubeObjectListLayout` inside a `clusterPage`
(see [Kubernetes API](./06-kubernetes-api.md)).
