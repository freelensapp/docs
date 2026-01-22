# Catalog API

!!! info "Available in Freelens 1.7+"

The Catalog API provides extensions with access to cluster information across all registered Kubernetes clusters. This enables building multi-cluster extensions that can enumerate clusters, react to cluster changes, and access cluster metadata.

## Renderer API

```typescript
import { Catalog } from "@freelensapp/core/renderer";
```

### getAllClusters()

Returns all registered clusters.

```typescript
const clusters = Catalog.getAllClusters();

for (const cluster of clusters) {
  console.log(`${cluster.name}: ${cluster.status}`);
}
```

### getClusterById(id)

Returns a specific cluster by ID, or `undefined` if not found.

```typescript
const cluster = Catalog.getClusterById("my-cluster-id");

if (cluster) {
  console.log(`Found: ${cluster.name}`);
}
```

### getActiveCluster()

Returns the currently active (selected) cluster.

```typescript
const active = Catalog.getActiveCluster();
console.log(`Viewing: ${active?.name}`);
```

## Main API

```typescript
import { Catalog } from "@freelensapp/core/main";
```

The main process API provides:

- `Catalog.getAllClusters()` - Get all clusters
- `Catalog.getClusterById(id)` - Get cluster by ID

!!! note
    `getActiveCluster()` is only available in the renderer process since the "active" cluster is a UI concept.

## Types

### ClusterId

A type alias for cluster identifiers. Use this with the [K8s API](./k8s-api.md) to perform operations on clusters:

```typescript
import { Catalog, K8s, ClusterId } from "@freelensapp/core/renderer";

const cluster = Catalog.getActiveCluster();
const clusterId: ClusterId = cluster.id;

// Use with K8s API
const pods = await K8s.queryCluster(clusterId, { apiVersion: "v1", kind: "Pod" });
```

### ClusterInfo

```typescript
interface ClusterInfo {
  readonly id: ClusterId;  // Use with K8s API
  readonly name: string;
  readonly kubeConfigPath: string;
  readonly contextName: string;
  readonly status: ClusterConnectionStatus;
  readonly labels: Record<string, string>;
  readonly isActive: boolean;
  readonly metadata?: ClusterMetadata;
}
```

### ClusterConnectionStatus

```typescript
enum ClusterConnectionStatus {
  CONNECTING = "connecting",
  CONNECTED = "connected",
  DISCONNECTED = "disconnected",
  DISCONNECTING = "disconnecting",
}
```

### ClusterMetadata

Optional metadata when available:

```typescript
interface ClusterMetadata {
  readonly distribution?: string;       // "eks", "gke", "minikube", etc.
  readonly kubernetesVersion?: string;  // "1.28.0"
  readonly nodeCount?: number;
}
```

## Reactivity

The API is MobX-reactive. Use `reaction()` to respond to changes:

```typescript
import { reaction } from "mobx";
import { Catalog } from "@freelensapp/core/renderer";

// React when active cluster changes
const dispose = reaction(
  () => Catalog.getActiveCluster(),
  (cluster, prev) => {
    console.log(`Switched from ${prev?.name} to ${cluster?.name}`);
  }
);

// Clean up when done
dispose();
```

!!! tip
    Wrap React components with `observer()` from `mobx-react` to automatically re-render when cluster data changes.

!!! warning
    Always call the dispose function returned by `reaction()` when your extension deactivates to prevent memory leaks.
