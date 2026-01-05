# Platform Overview

## System Architecture

Living Content is a GitOps-driven multi-tenant AI infrastructure platform
providing automated provisioning, deployment, and management for isolated tenant
environments.

```mermaid
flowchart TB
    subgraph external["External"]
        GH[("GitHub<br/>GitOps Repos")]
        USER((Users))
    end

    subgraph gcp["Google Cloud Platform"]
        subgraph cloudrun["Cloud Run Services"]
            AUTH["Auth Service<br/>auth.service.livingcontent.co"]
            REC["Reconciler Service"]
        end

        subgraph gke["GKE Cluster"]
            subgraph infra["Infrastructure Namespace"]
                TM["Tenant Manager"]
                ARGO["ArgoCD"]
                APPSETS["ApplicationSets"]
            end

            subgraph tenant["Tenant Namespace"]
                TRES["Tenant Resources"]
            end

            subgraph gaim["GAIM Namespace"]
                API["API Service<br/>(FastAPI)"]
                WORKER["Worker<br/>(Taskiq)"]
                FE["Frontend<br/>(React)"]
                REDIS[("Redis")]
                MONGO[("MongoDB")]
            end
        end

        GLB["Global Load Balancer<br/>+ Certificate Manager"]
        FS[("Firestore")]
        FSTORE[("Filestore NFS")]
        SM["Secret Manager"]
        AR["Artifact Registry"]
    end

    USER --> GLB
    GLB --> AUTH
    GLB --> TM
    GLB --> API
    GLB --> FE

    GH <--> ARGO
    TM --> GH
    TM --> FS
    TM --> ARGO

    ARGO --> tenant
    ARGO --> gaim

    API --> REDIS
    API --> MONGO
    API --> FS
    WORKER --> REDIS
    WORKER --> MONGO
    WORKER --> FSTORE

    REC --> FS
    REC --> ARGO

    AUTH --> FS
```

## Component Hierarchy

```mermaid
graph TB
    subgraph platform["LIVING CONTENT PLATFORM"]
        subgraph infra["INFRASTRUCTURE LAYER (living-content-admin)"]
            direction TB
            subgraph platform_svc["Platform Services (Cloud Run)"]
                AUTH_SVC["Auth Service"]
                REC_SVC["Reconciler"]
            end

            subgraph cluster_svc["Cluster Services (GKE)"]
                TM_SVC["Tenant Manager"]
                ARGO_SVC["ArgoCD"]
            end

            subgraph gcp_infra["GCP Infrastructure"]
                GKE["GKE Clusters"]
                GLB_I["Global Load Balancer"]
                CERT["Certificate Manager"]
                FS_I["Firestore"]
                FSTORE_I["Filestore"]
                KMS["Cloud KMS"]
                SM_I["Secret Manager"]
                AR_I["Artifact Registry"]
                CB["Cloud Build"]
                IP["Identity Platform"]
            end

            subgraph gitops["GitOps System"]
                TEMPLATES["Templates Repo"]
                MANIFESTS["Manifests Repo"]
                APPSETS_I["ApplicationSets"]
                APPPROJ["AppProjects"]
            end
        end

        subgraph app["APPLICATION LAYER (living-content-gaim)"]
            direction TB
            subgraph runtime["Runtime Services (per GAIM)"]
                API_RT["API (FastAPI)"]
                WORKER_RT["Worker (Taskiq)"]
                FE_RT["Frontend (React)"]
            end

            subgraph core["Core Components"]
                SELECTOR["Tool Selector"]
                KNOWLEDGE["Knowledge System"]
                REGISTRY["Tool Registry"]
            end

            subgraph tools["External Tools"]
                IMG["Image Generator"]
                AUD["Audio Generator"]
                VID["Video Generator"]
                SPEECH["Speech Services"]
            end

            subgraph gaim_infra["Infrastructure (per GAIM)"]
                REDIS_G[("Redis")]
                MONGO_G[("MongoDB")]
                LANCE[("LanceDB")]
            end
        end
    end

    platform_svc --> cluster_svc
    cluster_svc --> gcp_infra
    gitops --> cluster_svc

    runtime --> core
    core --> tools
    runtime --> gaim_infra
```

## Data Flow

### GitOps Deployment Flow

```mermaid
sequenceDiagram
    participant CLI as lco-admin CLI
    participant TM as Tenant Manager
    participant FS as Firestore
    participant Templates as Templates Repo
    participant Manifests as Manifests Repo
    participant ArgoCD
    participant GKE

    CLI->>TM: Create GAIM request
    TM->>FS: Store GAIM metadata
    TM->>Templates: Read Jinja2 templates
    TM->>TM: Render templates
    TM->>Manifests: Push generated manifests
    Manifests-->>ArgoCD: Webhook notification
    ArgoCD->>Manifests: Pull manifests
    ArgoCD->>GKE: Apply resources
    GKE-->>ArgoCD: Sync status
    ArgoCD-->>TM: Status update
    TM->>FS: Update sync status
```

### Query Processing Flow

```mermaid
sequenceDiagram
    participant Client
    participant API as API Service
    participant Redis as Redis Streams
    participant Worker as Worker Service
    participant Tools as Tool Registry
    participant WS as WebSocket

    Client->>API: POST /queries
    API->>API: Validate request
    API->>Redis: Enqueue job
    API-->>Client: {requestId, streaming: true}

    Redis->>Worker: Dequeue job
    Worker->>Tools: Select tool (vector search)
    Worker->>Tools: Execute tool

    loop Streaming Response
        Tools-->>Worker: Chunk
        Worker->>Redis: Pub/Sub publish
        Redis->>WS: Forward to client
        WS-->>Client: stream_chunk
    end

    Worker->>Redis: Pub/Sub complete
    Redis->>WS: Forward completion
    WS-->>Client: stream_complete
```

## Technology Stack

| Layer                       | Component            | Technology                           |
| --------------------------- | -------------------- | ------------------------------------ |
| **Cloud Platform**          | Infrastructure       | Google Cloud Platform                |
| **Container Orchestration** | Runtime              | Google Kubernetes Engine (GKE)       |
| **Load Balancing**          | Traffic              | Gateway API + Global Load Balancer   |
| **GitOps**                  | Deployment           | ArgoCD + ApplicationSets             |
| **Serverless**              | Platform Services    | Cloud Run                            |
| **Database**                | Metadata/Config      | Firestore                            |
| **Hub**                     | Tenant Management UI | Vanilla JS (GKE)                     |
| **Frontend**                | User-facing UI       | Vanilla JS (Embeddable, Event-based) |
| **Database**                | Application Data     | MongoDB                              |
| **Cache/Queue**             | Messaging            | Redis (Streams + Pub/Sub)            |
| **Vector Store**            | Embeddings           | LanceDB                              |
| **File Storage**            | RAG Data             | Filestore (NFS)                      |
| **Secrets**                 | Credentials          | Secret Manager + KMS                 |
| **Auth**                    | Identity             | Identity Platform                    |
| **Registry**                | Images               | Artifact Registry                    |
| **CI/CD**                   | Builds               | Cloud Build                          |

## Design Principles

1. **GitOps-First**: All infrastructure changes flow through Git → ArgoCD
2. **Multi-Tenant Isolation**: Namespace-based with RBAC enforcement
3. **UUID/Name Duality**: Permanent IDs with human-friendly names
4. **Eventual Consistency**: Reconciler handles sync convergence
5. **Level-Triggered Controllers**: Kubernetes-style control loops
6. **Fail-Safe Defaults**: Rate limiting, backoff, manual intervention
7. **Security-First**: Workload Identity, Secret Manager, encrypted storage
8. **Observability**: Comprehensive logging, metrics, and status tracking

## Related Documentation

- [Infrastructure Layer](infrastructure-layer.md) - Platform management details
- [Application Layer](application-layer.md) - GAIM runtime components
- [Multi-Tenancy](multi-tenancy.md) - Isolation and namespace model
- [Networking](networking.md) - Traffic routing and DNS
