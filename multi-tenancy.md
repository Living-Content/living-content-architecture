# Multi-Tenancy Model

Living Content implements a multi-tenant architecture with namespace-based
isolation, RBAC enforcement, and GitOps-driven provisioning.

## Tenancy Hierarchy

```mermaid
graph TB
    subgraph platform["Platform"]
        subgraph cluster["Cluster (uscentral1-1)"]
            subgraph infra_ns["Namespace: living-content"]
                ARGO["ArgoCD"]
                TM["Tenant Manager"]
                APPSETS["ApplicationSets"]
            end

            subgraph tenant_ns["Namespace: tenant-siriuscorp"]
                T_RES["Tenant Resources"]
                T_QUOTA["Resource Quotas"]
            end

            subgraph gaim_ns1["Namespace: gaim-marvin"]
                G1_API["API"]
                G1_WORKER["Worker"]
                G1_HUB["Hub"]
                G1_REDIS["Redis"]
                G1_MONGO["MongoDB"]
            end

            subgraph gaim_ns2["Namespace: gaim-deepthought"]
                G2_API["API"]
                G2_WORKER["Worker"]
                G2_HUB["Hub"]
                G2_REDIS["Redis"]
                G2_MONGO["MongoDB"]
            end
        end
    end

    infra_ns --> tenant_ns
    infra_ns --> gaim_ns1
    infra_ns --> gaim_ns2
```

## Key Concepts

| Concept         | Description                                                |
| --------------- | ---------------------------------------------------------- |
| **Tenant**      | Top-level organizational unit that owns and manages GAIMs  |
| **GAIM**        | Deployable AI application instance (Generative AI Manager) |
| **AppProject**  | ArgoCD resource defining deployment access boundaries      |
| **Application** | ArgoCD resource that deploys actual workloads              |
| **Namespace**   | Kubernetes isolation unit                                  |

## Naming and Identification

### Dual Identifier Strategy

Each Tenant and GAIM has two identifiers:

| Identifier | Format                  | Usage                              |
| ---------- | ----------------------- | ---------------------------------- |
| **ID**     | UUID v4                 | Internal, permanent, database keys |
| **Name**   | `^[a-z][a-z0-9]{2,11}$` | Human-friendly, URLs, DNS, K8s     |

**Valid Name Examples:** `marvin`, `babelfish`, `heartofgold`

**Invalid Names:** `2fast` (starts with number), `ab` (too short),
`verylongname123` (too long)

### Namespace Naming

| Resource Type  | Pattern                | Example             |
| -------------- | ---------------------- | ------------------- |
| Infrastructure | `living-content`       | `living-content`    |
| Tenant         | `tenant-{tenant_name}` | `tenant-siriuscorp` |
| GAIM           | `gaim-{gaim_name}`     | `gaim-marvin`       |

### Cluster Naming

Pattern: `{region}-{cluster_index}`

Examples: `uscentral1-1`, `euwest1-2`, `useast1-1`

## AppProject Hierarchy

ArgoCD AppProjects define deployment permissions and resource boundaries.

```mermaid
graph TB
    subgraph projects["ArgoCD AppProjects"]
        INFRA["living-content<br/>(Infrastructure)"]
        TENANT["tenant-{name}<br/>(Per-Tenant)"]
        GAIMS["gaims<br/>(Shared)"]
    end

    subgraph namespaces["Kubernetes Namespaces"]
        NS_INFRA["living-content"]
        NS_TENANT["tenant-siriuscorp"]
        NS_GAIM1["gaim-marvin"]
        NS_GAIM2["gaim-deepthought"]
    end

    INFRA -->|"deploys to"| NS_INFRA
    TENANT -->|"controls"| NS_TENANT
    GAIMS -->|"deploys to"| NS_GAIM1
    GAIMS -->|"deploys to"| NS_GAIM2
```

### AppProject Types

| AppProject       | Scope      | Resources                                |
| ---------------- | ---------- | ---------------------------------------- |
| `living-content` | Platform   | ArgoCD, Tenant Manager, ApplicationSets  |
| `tenant-{name}`  | Per-tenant | Tenant-specific resources                |
| `gaims`          | All GAIMs  | GAIM deployments (API, Worker, Frontend) |

## Isolation Mechanisms

### Network Isolation

NetworkPolicies restrict cross-namespace communication:

```mermaid
flowchart LR
    subgraph gaim1["gaim-marvin"]
        A1["API"] --> R1["Redis"]
        W1["Worker"] --> R1
        A1 --> M1["MongoDB"]
        W1 --> M1
    end

    subgraph gaim2["gaim-deepthought"]
        A2["API"] --> R2["Redis"]
        W2["Worker"] --> R2
    end

    gaim1 -.->|"blocked"| gaim2
```

### RBAC Isolation

AppProjects define deployment permissions:

- Which namespaces can be deployed to
- Which resources can be created
- Which source repositories are allowed

### Resource Isolation

Per-namespace limits via ResourceQuotas and LimitRanges:

| Resource | Quota                 |
| -------- | --------------------- |
| CPU      | Configurable per tier |
| Memory   | Configurable per tier |
| Pods     | Configurable per tier |
| PVCs     | Configurable per tier |

## GAIM Namespace Contents

Each GAIM namespace contains:

```mermaid
graph TB
    subgraph gaim_ns["Namespace: gaim-{name}"]
        subgraph deployments["Deployments"]
            API["api<br/>Replicas: HPA"]
            WORKER["worker<br/>Replicas: HPA"]
            HUB["hub<br/>Replicas: 1-2"]
        end

        subgraph statefulsets["StatefulSets"]
            REDIS["redis<br/>Replicas: 1"]
            MONGO["mongodb<br/>Replica Set"]
        end

        subgraph services["Services"]
            SVC_API["api-svc"]
            SVC_HUB["hub-svc"]
            SVC_REDIS["redis-svc"]
            SVC_MONGO["mongodb-svc"]
        end

        subgraph networking["Networking"]
            ROUTE_API["HTTPRoute (API)"]
            ROUTE_HUB["HTTPRoute (Hub)"]
        end

        subgraph config["Configuration"]
            CM["ConfigMaps"]
            SEC["Secrets"]
            SA["ServiceAccount"]
        end
    end

    subgraph external["External (CDN)"]
        FE["Frontend<br/>(Embeddable JS)"]
    end

    API --> SVC_REDIS
    API --> SVC_MONGO
    WORKER --> SVC_REDIS
    WORKER --> SVC_MONGO

    ROUTE_API --> SVC_API
    ROUTE_HUB --> SVC_HUB
    FE -.->|"API calls"| ROUTE_API
```

## Tenant-GAIM Association

GAIMs can be associated with multiple tenants:

```mermaid
erDiagram
    TENANT ||--o{ TENANT_GAIM : "has"
    GAIM ||--o{ TENANT_GAIM : "belongs_to"

    TENANT {
        uuid tenant_id PK
        string tenant_name UK
        string display_name
        timestamp created_at
    }

    GAIM {
        uuid gaim_id PK
        string gaim_name UK
        string display_name
        array associated_tenant_ids
        timestamp created_at
    }

    TENANT_GAIM {
        uuid tenant_id FK
        uuid gaim_id FK
    }
```

## GitOps Directory Structure

```plaintext
living-content-gitops/
└── clusters/
    └── uscentral1-1/
        ├── infrastructure/
        │   ├── argocd/
        │   ├── tenant-manager/
        │   └── applicationsets/
        │
        ├── tenants/
        │   ├── tenant-siriuscorp/
        │   │   ├── namespace.yaml
        │   │   ├── resourcequota.yaml
        │   │   └── rolebindings.yaml
        │   │
        │   └── tenant-globex/
        │       └── ...
        │
        └── gaims/
            ├── gaim-marvin/
            │   ├── namespace.yaml
            │   ├── api-deployment.yaml
            │   ├── worker-deployment.yaml
            │   ├── hub-deployment.yaml
            │   ├── redis-statefulset.yaml
            │   ├── mongodb-statefulset.yaml
            │   ├── services.yaml
            │   └── httproutes.yaml
            │
            └── gaim-deepthought/
                └── ...
```

## ApplicationSet Discovery

### Tenant ApplicationSet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: uscentral1-1-tenants
spec:
  generators:
    - git:
        repoURL: https://github.com/org/living-content-gitops
        revision: HEAD
        files:
          - path: clusters/uscentral1-1/tenants/*/namespace.yaml
  template:
    # Application template for each tenant
```

### GAIM ApplicationSet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: uscentral1-1-gaims
spec:
  generators:
    - git:
        repoURL: https://github.com/org/living-content-gitops
        revision: HEAD
        directories:
          - path: clusters/uscentral1-1/gaims/*
  template:
    # Application template for each GAIM
```

## Provisioning Flow

```mermaid
sequenceDiagram
    participant Admin as Admin CLI
    participant TM as Tenant Manager
    participant FS as Firestore
    participant Git as GitOps Repo
    participant Argo as ArgoCD
    participant K8s as Kubernetes

    Admin->>TM: Create GAIM request
    TM->>FS: Validate tenant association
    TM->>FS: Store GAIM metadata
    TM->>TM: Render templates
    TM->>Git: Push manifests
    Git-->>Argo: Webhook trigger
    Argo->>Git: Pull manifests
    Argo->>K8s: Create namespace
    Argo->>K8s: Apply resources
    K8s-->>Argo: Sync status
    Argo-->>TM: Status callback
    TM->>FS: Update sync status
```

## Multi-Cluster Considerations

- Same tenant/GAIM can exist across multiple clusters
- Firestore is shared across all clusters
- Each cluster has its own:
  - Tenant Manager instance
  - ArgoCD instance
  - ApplicationSets
  - Manifests directory (`clusters/{cluster}/`)

## Related Documentation

- [Platform Overview](platform-overview.md) - High-level architecture
- [Infrastructure Layer](infrastructure-layer.md) - Tenant Manager details
- [Networking](networking.md) - DNS and routing
- [Data Stores](data-stores.md) - Firestore collections
