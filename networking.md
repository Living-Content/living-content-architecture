# Networking

Living Content uses Google Cloud's Gateway API with a Global Load Balancer for
traffic routing, TLS termination, and host-based routing to services across
clusters.

## Traffic Architecture

```mermaid
flowchart TB
    subgraph internet["Internet"]
        CLIENT((Client))
    end

    subgraph gcp["Google Cloud Platform"]
        subgraph glb["Global Load Balancer"]
            CERT["Certificate Manager<br/>Wildcard Certs"]
            URLMAP["URL Maps<br/>Host-based Routing"]
        end

        subgraph clusters["GKE Clusters"]
            subgraph cluster1["Cluster: uscentral1-1"]
                NEG1["Network Endpoint Groups"]

                subgraph services1["Services"]
                    TM1["Tenant Manager"]
                    ARGO1["ArgoCD"]
                    GAIM1_API["gaim-marvin API"]
                    GAIM1_HUB["gaim-marvin Hub"]
                end
            end
        end

        AUTH_CR["Auth Service<br/>(Cloud Run)"]
    end

    CLIENT -->|"HTTPS"| glb
    glb -->|"TLS termination"| CERT
    URLMAP -->|"HTTP"| NEG1
    NEG1 --> services1
    glb -->|"HTTPS"| AUTH_CR
```

## DNS Patterns

### Nested Subdomain Structure

```mermaid
graph TB
    subgraph root["livingcontent.co"]
        subgraph api["*.api.livingcontent.co"]
            GAIM_API["{gaim}.api"]
        end

        subgraph hub["*.hub.livingcontent.co"]
            GAIM_HUB["{gaim}.hub"]
        end

        subgraph assets["assets.livingcontent.co"]
            CDN["Frontend JS (CDN)"]
        end

        subgraph tm["*.tm.livingcontent.co"]
            TM["{cluster}.tm"]
        end

        subgraph argocd["*.argocd.livingcontent.co"]
            ARGO["{cluster}.argocd"]
        end

        subgraph service["*.service.livingcontent.co"]
            AUTH["auth.service"]
        end
    end
```

### Domain Patterns

| Service Type   | Pattern                             | Example                                |
| -------------- | ----------------------------------- | -------------------------------------- |
| GAIM API       | `{gaim}.api.livingcontent.co`       | `marvin.api.livingcontent.co`          |
| GAIM Hub       | `{gaim}.hub.livingcontent.co`       | `marvin.hub.livingcontent.co`          |
| Frontend JS    | `assets.livingcontent.co`           | CDN-hosted embeddable JS               |
| Tenant Manager | `{cluster}.tm.livingcontent.co`     | `uscentral1-1.tm.livingcontent.co`     |
| ArgoCD         | `{cluster}.argocd.livingcontent.co` | `uscentral1-1.argocd.livingcontent.co` |
| Auth Service   | `auth.service.livingcontent.co`     | -                                      |

## Wildcard Certificates

Managed via Google Certificate Manager with DNS authorization.

| Certificate                  | Domain                      | Purpose          |
| ---------------------------- | --------------------------- | ---------------- |
| `living-content-cert`        | `*.livingcontent.co`        | Root wildcard    |
| `living-content-api-cert`    | `*.api.livingcontent.co`    | GAIM APIs        |
| `living-content-hub-cert`    | `*.hub.livingcontent.co`    | GAIM Hubs        |
| `living-content-assets-cert` | `assets.livingcontent.co`   | Frontend CDN     |
| `living-content-tm-cert`     | `*.tm.livingcontent.co`     | Tenant Managers  |
| `living-content-argocd-cert` | `*.argocd.livingcontent.co` | ArgoCD instances |

## Gateway API Configuration

### Gateway Resource

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: external-gateway
  namespace: living-content
spec:
  gatewayClassName: gke-l7-global-external-managed
  addresses:
    - type: NamedAddress
      value: living-content-global-ip
  listeners:
    - name: https-api
      hostname: '*.api.livingcontent.co'
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: living-content-api-cert
    - name: https-hub
      hostname: '*.hub.livingcontent.co'
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: living-content-hub-cert
```

### HTTPRoute Resource (GAIM API)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gaim-marvin-api
  namespace: gaim-marvin
spec:
  parentRefs:
    - name: external-gateway
      namespace: living-content
      sectionName: https-api
  hostnames:
    - 'marvin.api.livingcontent.co'
  rules:
    - backendRefs:
        - name: api-svc
          port: 8000
```

## Traffic Flow

### Request Path

```mermaid
sequenceDiagram
    participant Client
    participant DNS
    participant GLB as Global Load Balancer
    participant NEG as Network Endpoint Group
    participant Pod as Service Pod

    Client->>DNS: Resolve marvin.api.livingcontent.co
    DNS-->>Client: GLB IP address

    Client->>GLB: HTTPS request
    Note over GLB: TLS termination<br/>Certificate validation

    GLB->>GLB: URL Map routing<br/>(host-based)

    GLB->>NEG: HTTP request
    Note over NEG: Health check<br/>Load balancing

    NEG->>Pod: Forward to healthy pod
    Pod-->>NEG: Response
    NEG-->>GLB: Response
    GLB-->>Client: HTTPS response
```

### Internal Traffic

Within the cluster, traffic flows over HTTP (TLS terminates at GLB):

```mermaid
flowchart LR
    subgraph namespace["gaim-marvin namespace"]
        API["API Pod"] -->|"HTTP"| REDIS["Redis Service"]
        API -->|"HTTP"| MONGO["MongoDB Service"]
        WORKER["Worker Pod"] -->|"HTTP"| REDIS
        WORKER -->|"HTTP"| MONGO
    end
```

## Load Balancing

### Global Load Balancer Features

| Feature          | Configuration                     |
| ---------------- | --------------------------------- |
| Protocol         | HTTPS (external), HTTP (internal) |
| SSL Policy       | Modern ciphers only               |
| HTTP/2           | Enabled                           |
| Session Affinity | None (stateless)                  |
| Health Checks    | HTTP `/health` endpoint           |

### Network Endpoint Groups (NEGs)

NEGs provide direct pod-level load balancing:

- Auto-created by GKE Gateway controller
- Pod IP addresses registered directly
- Bypass kube-proxy for reduced latency
- Health checks per-pod

### Health Checks

| Endpoint      | Interval   | Threshold |
| ------------- | ---------- | --------- |
| API `/health` | Configured | Threshold |
| Hub `/`       | Configured | Threshold |
| TM `/health`  | Configured | Threshold |

## Rate Limiting

Implemented via Cloud Armor security policies at the GLB.

### Default Limits

| Endpoint        | Rate Limit      |
| --------------- | --------------- |
| Auth endpoints  | Restricted      |
| Query endpoints | Per-user limits |
| Default         | Standard        |

### Cloud Armor Policy

```yaml
securityPolicy:
  rules:
    - action: rate_based_ban
      match:
        expr:
          expression: "request.path.matches('/auth/init')"
      rateLimitOptions:
        rateLimitThreshold:
          count: 5
          intervalSec: 1
```

## HTTP to HTTPS Redirect

All HTTP traffic is redirected to HTTPS:

```mermaid
sequenceDiagram
    participant Client
    participant GLB as Load Balancer

    Client->>GLB: HTTP request (port 80)
    GLB-->>Client: 301 Redirect to HTTPS
    Client->>GLB: HTTPS request (port 443)
    GLB-->>Client: Response
```

## Multi-Cluster Routing

```mermaid
flowchart TB
    subgraph dns["DNS"]
        WILDCARD["*.api.livingcontent.co"]
    end

    subgraph glb["Global Load Balancer"]
        URLMAP["URL Maps"]
    end

    subgraph clusters["Clusters"]
        C1["uscentral1-1"]
        C2["useast1-1"]
        C3["euwest1-1"]
    end

    WILDCARD --> glb
    URLMAP -->|"marvin.api"| C1
    URLMAP -->|"deepthought.api"| C2
    URLMAP -->|"zaphod.api"| C3
```

Each GAIM's HTTPRoute specifies which cluster handles its traffic. Future
multi-region GAIMs could use weighted routing.

## WebSocket Support

WebSocket connections for real-time streaming:

```mermaid
sequenceDiagram
    participant Client
    participant GLB
    participant API as API Service

    Client->>GLB: HTTP Upgrade request
    GLB->>API: Forward Upgrade
    API-->>GLB: 101 Switching Protocols
    GLB-->>Client: WebSocket established

    loop Streaming
        API->>GLB: WebSocket frame
        GLB->>Client: Forward frame
    end
```

### WebSocket Configuration

| Parameter      | Value                  |
| -------------- | ---------------------- |
| Timeout        | Extended for streaming |
| Ping Interval  | Configured             |
| Max Frame Size | 64KB                   |

## Related Documentation

- [Platform Overview](platform-overview.md) - Architecture context
- [Infrastructure Layer](infrastructure-layer.md) - GLB setup
- [Application Layer](application-layer.md) - Service endpoints
- [Authentication](authentication.md) - Auth service routing
