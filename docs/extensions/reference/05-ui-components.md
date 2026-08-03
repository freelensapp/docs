<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# 5. UI components (`Renderer.Component`)

`Renderer.Component` is Freelens's shared React component library. Using it
(instead of your own components) gives you the app's look, theming, and
behavior for free. Prop-type interfaces are exported alongside each component
(e.g. `Renderer.Component.ButtonProps`).

Destructure what you need:

```tsx
const { Component: { KubeObjectListLayout, DrawerItem, Badge, Icon } } = Renderer;
```

Below is the catalogue grouped by purpose. Names in `code` are the exported
symbols; the parenthesized name is the props type.

## 5.1 Form controls

| Component            | Props type                | Notes |
| -------------------- | ------------------------- | ----- |
| `Button`             | `ButtonProps`             | Styled button (tooltip-aware). |
| `Input`              | `InputProps`              | Text input with sync + async validators, `iconLeft`/`iconRight`. |
| `SearchInput` / `SearchInputUrl` | `SearchInputProps` / `SearchInputUrlProps` | Search box; URL-synced variant. |
| `FileInput`          | `FileInputProps`          | File selection. |
| `DropFileInput`      | `DropFileInputProps<T>`   | Drag-and-drop file zone. |
| `Checkbox`           | `CheckboxProps`           | |
| `Radio` / `RadioGroup` | `RadioProps<T>` / `RadioGroupProps<T>` | |
| `Switch` / `Switcher` / `FormSwitch` | `SwitchProps` / `SwitcherProps` | MUI-backed toggles. |
| `Select`             | `SelectProps<…>`          | react-select based; `SelectOption<Value>`. |
| `Slider`             | `SliderProps`             | MUI slider. |
| `EditableList`       | `EditableListProps<T>`    | Add/remove list editor. |
| `AddRemoveButtons`   | `AddRemoveButtonsProps`   | +/- button pair. |

Validators: `InputValidator` and `InputValidation` are re-exported (via
`input.tsx`). The `SyncInputValidator` / `AsyncInputValidator` interfaces are
**not** re-exported through `Renderer.Component` — type your validators against
`InputValidator` instead.

## 5.2 Layout

| Component            | Props type                | Notes |
| -------------------- | ------------------------- | ----- |
| `MainLayout`         | `MainLayoutProps`         | Header/sidebar/footer shell. |
| `SettingLayout`      | `SettingLayoutProps`      | Settings-style page shell. |
| `PageLayout`         | —                         | Convenience wrapper over the above. |
| `TabLayout`          | `TabLayoutProps`, `TabLayoutRoute` | Tabbed page. |
| `WizardLayout`       | `WizardLayoutProps`       | Two-column wizard. |
| `SubTitle`           | `SubTitleProps`           | Section heading. (`SubHeader` is not re-exported.) |
| `Gutter`             | `GutterProps` (`size`)    | Spacing. |
| `HorizontalLine`     | `HorizontalLineProps`     | `<hr>` divider. |

> `Sidebar`, `SidebarItem`, `SidebarCluster`, `TopBar`, `ContextMenu`,
> `WindowControls`, `SubHeader`, and `CloseButton` exist in the source but are
> **not re-exported** through `Renderer.Component` — they are not reachable
> from an extension. Only `SubTitle` (not `SubHeader`) is exported.

## 5.3 Kube object views — the extension workhorses

| Component                 | Props type                       | Notes |
| ------------------------- | -------------------------------- | ----- |
| `KubeObjectListLayout`    | `KubeObjectListLayoutProps<K,A,D>` | **The full resource list page**: search, filters, namespace select, columns, per-row menu. Needs `store`. |
| `KubeObjectDetails`       | —                                | Host for the details drawer (usually you register a `Details` component instead). |
| `KubeObjectMenu`          | `KubeObjectMenuProps<T>`         | Context/action menu for an object. |
| `KubeObjectMeta`          | `KubeObjectMetaProps`            | Standard metadata rows (name/ns/labels/age). |
| `KubeObjectAge`           | `KubeObjectAgeProps`             | Age cell. |
| `KubeObjectConditionsList` / `KubeObjectConditionsDrawer` | … | Render `status.conditions`. |
| `LinkTo*`                 | per-file                         | Cross-links: `LinkToNamespace`, `LinkToPod`, `LinkToNode`, `LinkToSecret`, `LinkToConfigMap`, `LinkToJob`, `LinkToReplicaSet`, `LinkToRole`, `LinkToClusterRole`, `LinkToServiceAccount`, `LinkToStorageClass`, `LinkToPriorityClass`, `LinkToRuntimeClass`, `LinkToObject`, … |
| `ItemListLayout`          | `ItemListLayoutProps<Item,…>`    | Generic (non-kube) list layout. |

`KubeObjectDetailsProps<T>` (`{ className?; object: T }`) is the props type your
registered detail components receive.

Minimal list page:

```tsx
<KubeObjectListLayout<Example, ExampleApi>
  tableId="examplesTable"
  store={Example.getStore<Example>()}
  sortingCallbacks={{ name: (o) => o.getName(), age: (o) => o.getCreationTimestamp() }}
  searchFilters={[(o) => o.getSearchFields()]}
  renderHeaderTitle={Example.crd.title}
  renderTableHeader={[{ title: "Name", sortBy: "name" }, /* … */]}
  renderTableContents={(o) => [o.getName(), <KubeObjectAge object={o} key="age" />]}
/>
```

## 5.4 Tables and lists

`Table` (`TableProps<Item>`), `TableHead`, `TableRow`, `TableCell`,
`ReactTable` (`ReactTableProps<Data>`), `List` (`ListProps<T>`),
`VirtualList` (`VirtualListProps<T>`), `TreeView`/`TreeItem`/`TreeGroup`,
`Map` (`MapProps<Item>`). Sorting types: `TableSortParams`,
`TableSortCallback(s)`, `TableSortBy`, `TableOrderBy`.

## 5.5 Menus and dropdowns

`Menu` / `SubMenu` (`MenuProps`), `MenuItem` (`MenuItemProps`),
`MenuActions` (`MenuActionsProps` — kebab/toolbar action menu),
`Dropdown` (`DropdownProps`).

## 5.6 Dialogs and notifications

| Component / object   | Notes |
| -------------------- | ----- |
| `Dialog`             | Modal dialog (`DialogProps`). |
| `LogsDialog`         | Log viewer dialog. |
| `ConfirmDialog`      | Confirm modal. Statics: `ConfirmDialog.open(params)` and `ConfirmDialog.confirm(params): Promise<boolean>`. Params: `ConfirmDialogParams` (`message`, `labelOk`, `labelCancel`, `icon`, …). |
| `Notifications`      | Object with `.ok / .error / .checkedError / .info / .shortInfo`. |
| `notificationsStore` | The notifications store instance. |

```tsx
const { Notifications } = Renderer.Component;
Notifications.error("Something failed");
if (await Renderer.Component.ConfirmDialog.confirm({ message: "Delete?" })) { /* … */ }
```

## 5.7 Charts and metrics

`Chart` (the base), `BarChart`, `PieChart` (+ their prop/data types) — there is
**no** `LineChart` export. `ResourceMetrics` / `TimeRangedResourceMetrics`,
`PodCharts`, `PodDetailsList`. Use `requestMetrics` from
[`Renderer.K8sApi`](./06-kubernetes-api.md) to fetch data.

> Two symbols you might expect here are **not** reachable through
> `Renderer.Component`: `NoMetrics` (defined in `no-metrics.tsx` but not
> re-exported by the `resource-metrics` barrel) and `MetricsTab` (a **type
> alias** `"CPU" | "Memory" | "Disk" | …`, defined in `chart/options.ts`,
> which no barrel re-exports). Don't rely on either from an extension.

## 5.8 Drawers (detail-panel building blocks)

`Drawer` (`DrawerProps`, `DrawerPosition = "top"|"left"|"right"|"bottom"`),
`DrawerItem` (`DrawerItemProps`), `DrawerItemLabels`, `DrawerTitle`,
`DrawerParamToggler`. Detail panels are typically a stack of `DrawerItem`s:

```tsx
<DrawerItem name="Description">
  <MarkdownViewer markdown={object.spec.description ?? ""} />
</DrawerItem>
```

## 5.9 Misc

`Badge` (`BadgeProps`), `BadgeBoolean`, `StatusBrick`, `Avatar`, `Spinner`,
`Icon` (`IconProps` — supports `material`, `svg`, `NamedSvg`, `small`/`big`),
`Tooltip` / `WithTooltip` (`withTooltip` HOC), `Tabs`/`Tab`, `Stepper`,
`LineProgress`, `NoItems`, `Wizard`/`WizardStep`, `MarkdownViewer`,
`MonacoEditor` (full code editor: `MonacoEditorProps`, `MonacoTheme`, …),
`LocaleDate`, `ReactiveDuration`, `MaybeLink`, `RenderDelay`, `FilePicker`,
`PathPicker`.

Namespace helpers: `NamespaceSelect`, `NamespaceSelectFilter`,
`NamespaceSelectBadge`.

## 5.10 Non-component exports

`Renderer.Component` also exposes some singletons/functions:

| Export              | Purpose |
| ------------------- | ------- |
| `CommandOverlay`    | The command-palette overlay instance. |
| `createTerminalTab(...)` / `terminalStore` (`.sendCommand`) / `TerminalStore` | Open and drive dock terminal tabs. |
| `logTabStore`       | `.createPodTab` / `.createWorkloadTab` / `.renameTab` — dock log tabs. |

### Driving the dock: terminals and log tabs

The **dock** is the bottom panel that hosts terminals and log viewers. An
extension can open and drive its tabs — this is a real, first-class capability
(it's the entire basis of the `freelens-opencode-extension`), so it's worth
spelling out.

**Terminal tabs**

```tsx
const { createTerminalTab, terminalStore } = Renderer.Component;

// 1. Open a terminal tab in the dock; the returned object carries an `.id`.
const tabId = createTerminalTab({ title: "Agent Session" }).id;

// 2. Type a command into that terminal (auto-creates a tab if none; waits for
//    the shell to be ready). `enter: true` submits it.
await terminalStore.sendCommand("opencode", { tabId, enter: true });

// 3. (Undocumented-but-real) get the low-level terminal API to check readiness:
const api = (terminalStore as any).getTerminalApi?.(tabId);
if (api?.isReady) { /* … */ }
```

- `terminalStore` is the actual store instance (via `Object.assign`), so
  instance methods like `getData(tabId)` / `getTerminalApi(tabId)` are reachable
  even though they aren't in the published `.d.ts` — hence the `as any` in the
  wild. The typed surface is `sendCommand`.
- `TerminalStore` is a **shim class**: `getInstance()` / `createInstance()`
  both return the same `terminalStore`; `resetInstance()` is a no-op that warns.
- The shell env for terminals Freelens opens can be modified from the main
  process — see `terminalShellEnvModifier` in
  [Main registrations](./04-main-registrations.md#43-terminal-shell-environment).

**Log tabs**

```tsx
const { logTabStore } = Renderer.Component;
logTabStore.createPodTab({ selectedPod, selectedContainer });   // pod logs
logTabStore.createWorkloadTab({ workload });                    // workload logs
logTabStore.renameTab(tabId);                                   // rename to "Pod <name>"
```

## 5.11 Icons from SVG

Wrap the host `Icon` with a raw-imported SVG:

```tsx
import svgIcon from "./example.svg?raw";
const { Component: { Icon } } = Renderer;

export function ExampleIcon(props: Renderer.Component.IconProps) {
  return <Icon {...props} svg={svgIcon} />;
}
```
