<!--
Copyright (c) Freelens Authors. All rights reserved.
Licensed under MIT License. See LICENSE in root directory for more information.
-->

# 6. Kubernetes API

`Renderer.K8sApi` (and its main-side twin `Main.K8sApi`) is how an extension
models and talks to Kubernetes resources. The list of built-in resources,
stores, and APIs is large; this page covers the parts you actually build with:

1. Defining a **custom CRD** (the three-class pattern).
2. Using **built-in** resource stores/APIs.
3. **Cluster query functions** (`Renderer.K8s` / `Main.K8s`).
4. **Metrics**.

## 6.1 Defining a custom CRD — the three-class pattern

For a custom resource you export **three** classes from one model file: the
`KubeObject`, its `KubeApi`, and its `KubeObjectStore`.

```ts
import { Renderer } from "@freelensapp/extensions";

export interface ExampleSpec {
  title?: string;
  active?: boolean;
  description?: string;
}
export type ExampleStatus = {};

export class Example extends Renderer.K8sApi.LensExtensionKubeObject<
  Renderer.K8sApi.KubeObjectMetadata,
  ExampleStatus,
  ExampleSpec
> {
  static readonly kind = "Example";
  static readonly namespaced = true;
  static readonly apiBase = "/apis/example.freelens.app/v1alpha1/examples";
  static readonly crd: ExampleKubeObjectCRD = {
    apiVersions: ["example.freelens.app/v1alpha1"],
    plural: "examples",
    singular: "example",
    shortNames: ["ex"],
    title: "Examples",
  };

  // Accessors are STATIC and take the object as a parameter (see the rule below):
  static getTitle(object: Example): string | undefined {
    return object.spec.title;
  }
}

export class ExampleApi extends Renderer.K8sApi.KubeApi<Example> {}
export class ExampleStore extends Renderer.K8sApi.KubeObjectStore<Example, ExampleApi> {}
```

- `KubeApi` **auto-registers** with `apiManager` (unless you pass
  `autoRegister: false`).
- `KubeObjectStore` auto-wires the cluster-frame context.
- `LensExtensionKubeObject` provides statics `getApi<K,Api>()` and
  `getStore<K,Store>()` to resolve the singleton api/store by kind, e.g.
  `Example.getStore<Example>()`.

`LensExtensionKubeObjectCRD` shape:

```ts
interface LensExtensionKubeObjectCRD {
  apiVersions: string[];
  plural: string;
  singular: string;      // required
  shortNames?: string[];
}
```

`title` is **not** part of the base type — the example extension extends it with
its own `title: string` (via `ExampleKubeObjectCRD extends
LensExtensionKubeObjectCRD`). Do the same if you want extra CRD metadata.

### CRITICAL rule: `static readonly` metadata, no instance methods

The host reads class metadata **statically** and, at runtime, hands your
components **plain object copies** of the resource data — not instances of your
class. Therefore:

- **Allowed**: `object.spec?.field`, `object.status?.conditions`,
  host base-class instance methods (`object.getName()`, `object.getNs()`,
  `object.getCreationTimestamp()`, `object.getSearchFields()`), and
  **static** helpers you write that take the object as a parameter
  (`Example.getTitle(object)`).
- **Forbidden**: instance methods you define on the subclass
  (`object.getTitle()`) — they will not exist at runtime.
- **Forbidden**: `typeof (object as any).method === "function" ? …` — always
  falls through.
- **Forbidden**: `as any` — the typed `Spec`/`Status` interfaces already exist;
  use them.

Multiple CRD versions live in separate model files (e.g.
`example-v1alpha1.ts`, `example-v1alpha2.ts`), each with its own `apiBase` and
`crd.apiVersions`. To pick whichever version's CRD is installed at runtime,
try each version's `getStore()` in order and render the first that resolves.

### Registering the CRD's UI

In `src/renderer/index.tsx`:

```tsx
kubeObjectDetailItems = [{
  kind: Example.kind,
  apiVersions: Example.crd.apiVersions,
  priority: 10,
  components: { Details: (p) => <ExampleDetails {...p} extension={this} /> },
}];
clusterPages = [{ id: "example", components: { Page: ExamplesPage } }];
clusterPageMenus = [{ title: Example.crd.title, target: { pageId: "example" }, components: { Icon: ExampleIcon } }];
```

## 6.2 Built-in resources

`Renderer.K8sApi` re-exports the standard resource model classes, pre-built
API singletons, and pre-built stores, so you can read core resources without
defining your own:

- **KubeObject classes**: `ConfigMap`, `PersistentVolume`,
  `PersistentVolumeClaim`, `StorageClass`, `ResourceQuota`, `LimitRange`,
  `HorizontalPodAutoscaler`, `PriorityClass`, `PodDisruptionBudget`,
  `RoleBinding`, `ClusterRole`, `ClusterRoleBinding`, `ServiceAccount`,
  `DaemonSet`, `StatefulSet`, `ReplicaSet`, `CronJob`, `NetworkPolicy`,
  `EndpointSlice`, `CustomResourceDefinition`, `KubeEvent`, … (plus `Pod`,
  `Node`, `Deployment`, `Ingress`, `Service`, `Secret`, `Namespace`, `Job`).
- **API singletons**: `podsApi`, `nodesApi`, `deploymentApi`, `secretsApi`,
  `serviceApi`, `serviceAccountsApi`, `roleBindingApi`, `statefulSetApi`,
  `storageClassApi`, `vpaApi`, … (`*Api` type-only exports also available).
- **Stores**: `PodStore`/`podsStore`, `NodeStore`, `DeploymentStore`,
  `ConfigMapStore`, `SecretStore`, `EventStore`, `NamespaceStore`,
  `IngressStore`, `ServiceStore`, `CustomResourceStore`/`CRDResourceStore`,
  `CustomResourceDefinitionStore`/`CRDStore`, and many more.

Core classes: `KubeObject`, `KubeApi<Object,Data>`, `KubeObjectStore<K,A,D>`,
`KubeJsonApi`, `apiManager`. Metadata types: `KubeObjectMetadata`,
`KubeJsonApiData`, `KubeJsonApiDataFor`, `OwnerReference`,
`ClusterScopedMetadata`, `NamespaceScopedMetadata`, `KubeStatus`. Patch/loading
types: `JsonPatch`, `KubeObjectStoreLoadAllParams`,
`KubeObjectStoreLoadingParams`.

Mutating through a store (from the example's menu action):

```tsx
const store = Example.getStore<Example>();
await store.patch(object, { spec: { active } }, "merge");
```

## 6.3 Clusters: cluster query functions

`Renderer.K8s` (renderer, over IPC) and `Main.K8s` (main, direct) expose the
same function set for CRUD against arbitrary clusters:

| Function              | Signature                                                        |
| --------------------- | ---------------------------------------------------------------- |
| `queryCluster<T>`     | `(clusterId, query: ResourceQuery) => Promise<T[]>`              |
| `queryClusters<T>`    | `(clusterIds[], query) => Promise<Map<ClusterId, T[] \| Error>>` |
| `queryAllClusters<T>` | `(query) => Promise<Map<ClusterId, T[] \| Error>>` (connected only) |
| `getResource<T>`      | `(clusterId, query & { name }) => Promise<T \| null>` (404 → null) |
| `applyOnCluster<T extends KubeJsonApiData>` | `(clusterId, manifest) => Promise<T>` (create-or-update) |
| `deleteOnCluster`     | `(clusterId, resource & { name }) => Promise<void>` (404 ignored)|
| `patchOnCluster<T>`   | `(clusterId, resource & { name }, patch, patchType?) => Promise<T>` |

Types: `ClusterId`, `ResourceQuery`,
`KubeApiPatchType = "strategic" | "merge" | "json"` (default `"strategic"`).

> To enumerate clusters or find the active one, use **`Renderer.Catalog`**
> (`getAllClusters`, `getActiveCluster`, `activeCluster`), not `K8s` — see
> [Catalog](./08-catalog-navigation-theme.md).

## 6.4 Metrics

```ts
Renderer.K8sApi.requestMetrics(params: RequestMetricsParams): Promise<MetricData>;
```

Types: `RequestMetrics`, `RequestMetricsParams`, `MetricData`, `MetricResult`.
Feed results into the chart/metrics components in
[UI components](./05-ui-components.md#57-charts-and-metrics).

## 6.5 Object status indicators

`KubeObjectStatus` / `KubeObjectStatusLevel` let you surface status badges on
objects. (Note: there is no `kubeObjectStatusTexts` registration in this
version — see [renderer registrations](./03-renderer-registrations.md).)
