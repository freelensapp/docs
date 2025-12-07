# K8s API

!!! info "Available in Freelens 1.8+"

The K8s API provides extensions with the ability to perform Kubernetes operations on any cluster by ID, without switching the UI context. This enables building multi-cluster extensions that can query, create, update, and delete resources across clusters.

## Quick Start

```typescript
import { K8s, Catalog } from "@freelensapp/core/renderer";

// Get pods from a specific cluster
const cluster = Catalog.getActiveCluster();
const pods = await K8s.queryCluster(cluster.id, {
  apiVersion: "v1",
  kind: "Pod",
  namespace: "default",
});

console.log(`Found ${pods.length} pods`);
```

## Renderer API

```typescript
import { K8s } from "@freelensapp/core/renderer";
```

### queryCluster()

List resources from a specific cluster.

```typescript
const pods = await K8s.queryCluster<Pod>(clusterId, {
  apiVersion: "v1",
  kind: "Pod",
  namespace: "default",
  labelSelector: "app=nginx", // optional
});
```

### queryClusters()

Query multiple clusters in parallel.

```typescript
const results = await K8s.queryClusters<Pod>([clusterId1, clusterId2], {
  apiVersion: "v1",
  kind: "Pod",
  namespace: "default",
});

for (const [clusterId, data] of results) {
  if (data instanceof Error) {
    console.error(`Cluster ${clusterId} failed:`, data.message);
  } else {
    console.log(`Cluster ${clusterId}: ${data.length} pods`);
  }
}
```

### queryAllClusters()

Query all connected clusters.

```typescript
const results = await K8s.queryAllClusters<Node>({
  apiVersion: "v1",
  kind: "Node",
});

for (const [clusterId, data] of results) {
  if (!(data instanceof Error)) {
    console.log(`Cluster ${clusterId}: ${data.length} nodes`);
  }
}
```

### getResource()

Get a single resource by name.

```typescript
const pod = await K8s.getResource<Pod>(clusterId, {
  apiVersion: "v1",
  kind: "Pod",
  namespace: "default",
  name: "my-pod",
});

if (pod) {
  console.log(`Pod status: ${pod.status?.phase}`);
}
```

### applyOnCluster()

Create or update a resource.

```typescript
await K8s.applyOnCluster(clusterId, {
  apiVersion: "v1",
  kind: "ConfigMap",
  metadata: { name: "my-config", namespace: "default" },
  data: { key: "value" },
});
```

### deleteOnCluster()

Delete a resource.

```typescript
await K8s.deleteOnCluster(clusterId, {
  apiVersion: "v1",
  kind: "Pod",
  namespace: "default",
  name: "my-pod",
});
```

### patchOnCluster()

Patch a resource using strategic merge, JSON merge, or JSON patch.

```typescript
// Strategic merge patch (default)
await K8s.patchOnCluster(
  clusterId,
  {
    apiVersion: "apps/v1",
    kind: "Deployment",
    namespace: "default",
    name: "my-app",
  },
  { spec: { replicas: 3 } },
  "strategic",
);

// JSON merge patch
await K8s.patchOnCluster(
  clusterId,
  {
    apiVersion: "v1",
    kind: "ConfigMap",
    namespace: "default",
    name: "my-config",
  },
  { data: { newKey: "newValue" } },
  "merge",
);

// JSON patch
await K8s.patchOnCluster(
  clusterId,
  { apiVersion: "v1", kind: "Pod", namespace: "default", name: "my-pod" },
  [{ op: "add", path: "/metadata/labels/patched", value: "true" }],
  "json",
);
```

## Main API

```typescript
import { K8s } from "@freelensapp/core/main";
```

The main process API provides all the same functions:

- `K8s.queryCluster()`
- `K8s.queryClusters()`
- `K8s.queryAllClusters()`
- `K8s.getResource()`
- `K8s.applyOnCluster()`
- `K8s.deleteOnCluster()`
- `K8s.patchOnCluster()`

## Types

### ClusterId

All functions accept a `ClusterId` type for the cluster identifier. Import it from `K8s` and get values from the Catalog API:

```typescript
import { K8s, Catalog } from "@freelensapp/core/renderer";
import type { ClusterId } from "@freelensapp/core/renderer";

const cluster = Catalog.getActiveCluster();
const clusterId: ClusterId = cluster.id; // ClusterInfo.id is typed as ClusterId
```

Or more simply, since `ClusterInfo.id` is already typed as `ClusterId`:

```typescript
import { K8s, Catalog } from "@freelensapp/core/renderer";

const cluster = Catalog.getActiveCluster();
const pods = await K8s.queryCluster(cluster.id, { ... }); // cluster.id is ClusterId
```

### ResourceQuery

```typescript
interface ResourceQuery {
  apiVersion: string; // e.g., "v1", "apps/v1"
  kind: string; // e.g., "Pod", "Deployment"
  namespace?: string; // required for namespaced resources
  name?: string; // for single-resource operations
  labelSelector?: string; // e.g., "app=nginx,env=prod"
  fieldSelector?: string; // e.g., "status.phase=Running"
}
```

### KubeApiPatchType

```typescript
type KubeApiPatchType = "strategic" | "merge" | "json";
```

## Error Handling

Single-cluster operations throw errors on failure:

```typescript
try {
  await K8s.queryCluster(clusterId, query);
} catch (error) {
  console.error("Query failed:", error.message);
}
```

Multi-cluster operations (`queryClusters`, `queryAllClusters`) return per-cluster results:

```typescript
const results = await K8s.queryClusters(clusterIds, query);

for (const [id, data] of results) {
  if (data instanceof Error) {
    console.error(`Cluster ${id}:`, data.message);
  } else {
    console.log(`Cluster ${id}: ${data.length} items`);
  }
}
```

!!! warning "Cluster must be connected"
    Operations will fail with an error if the target cluster is not connected. Check `cluster.status === ClusterConnectionStatus.CONNECTED` before making API calls.

!!! tip "Namespace is required for namespaced resources"
    Always include `namespace` in your query for namespaced resources like Pods, Deployments, and ConfigMaps. Cluster-scoped resources like Nodes and Namespaces don't require it.

