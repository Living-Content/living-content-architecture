# Observability

Living Content implements comprehensive observability through Prometheus
metrics, structured logging, health checks, and resilience patterns.

## Observability Architecture

```mermaid
flowchart TB
    subgraph services["Services"]
        API["API Service"]
        WORKER["Worker Service"]
        TM["Tenant Manager"]
        REC["Reconciler"]
    end

    subgraph collection["Collection"]
        PROM["GCP Managed Prometheus"]
        LOGS["Cloud Logging"]
    end

    subgraph monitoring["Monitoring"]
        DASH["Dashboards"]
        ALERTS["Alerting"]
    end

    API -->|"/metrics"| PROM
    WORKER -->|"/metrics"| PROM
    TM -->|"/metrics"| PROM

    services -->|"structured logs"| LOGS

    PROM --> DASH
    PROM --> ALERTS
    LOGS --> DASH
```

## Prometheus Metrics

### Metric Collection

PodMonitor resources enable automatic metric scraping.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: gaim-api
  namespace: gaim-marvin
spec:
  selector:
    matchLabels:
      app: api
  podMetricsEndpoints:
    - port: metrics
      interval: { configured }
```

### Metric Categories

```mermaid
graph TB
    subgraph http["HTTP Metrics"]
        REQ["gaim_http_requests_total"]
        DUR["gaim_http_duration_seconds"]
    end

    subgraph tasks["Task Metrics"]
        TASK_TOTAL["gaim_tasks_total"]
        TASK_DUR["gaim_task_duration_seconds"]
        TASK_FLIGHT["gaim_tasks_in_flight"]
        QUEUE["gaim_task_queue_depth"]
    end

    subgraph llm["LLM Metrics"]
        LLM_CALLS["gaim_llm_calls_total"]
        LLM_TOKENS["gaim_llm_tokens_total"]
    end

    subgraph ws["WebSocket Metrics"]
        WS_CONN["gaim_websocket_connections_active"]
    end
```

### Metric Reference

| Metric                              | Type      | Labels               | Description                  |
| ----------------------------------- | --------- | -------------------- | ---------------------------- |
| `gaim_http_requests_total`          | Counter   | method, path, status | HTTP request count           |
| `gaim_http_duration_seconds`        | Histogram | method, path         | Request latency              |
| `gaim_tasks_total`                  | Counter   | task_name, status    | Task execution count         |
| `gaim_task_duration_seconds`        | Histogram | task_name            | Task latency                 |
| `gaim_tasks_in_flight`              | Gauge     | task_name            | Currently executing tasks    |
| `gaim_task_queue_depth`             | Gauge     | -                    | Pending tasks in queue       |
| `gaim_llm_calls_total`              | Counter   | model, provider      | LLM API calls                |
| `gaim_llm_tokens_total`             | Counter   | model, direction     | Token usage (input/output)   |
| `gaim_websocket_connections_active` | Gauge     | -                    | Active WebSocket connections |

### Auto-Added Labels

GCP Managed Prometheus automatically adds:

- `namespace`
- `pod`
- `cluster`
- `location`

## Health Checks

### Endpoint Structure

```mermaid
flowchart LR
    subgraph probes["Kubernetes Probes"]
        LIVE["Liveness<br/>/health"]
        READY["Readiness<br/>/health"]
    end

    subgraph checks["Health Checks"]
        REDIS["Redis connectivity"]
        MONGO["MongoDB connectivity"]
        CONFIG["Config loaded"]
    end

    probes --> checks
```

### API Health Endpoint

```json
GET /health

{
  "status": "healthy",
  "checks": {
    "redis": "ok",
    "mongodb": "ok",
    "config": "loaded"
  },
  "version": "1.2.3"
}
```

### Worker Health Check

File-based heartbeat for worker processes:

```mermaid
sequenceDiagram
    participant Thread as Heartbeat Thread
    participant File as /tmp/healthy
    participant Probe as K8s Probe

    loop Periodic check
        Thread->>File: Write timestamp
    end

    Probe->>File: Check file exists
    Probe->>File: Check timestamp freshness
    alt Healthy
        Probe-->>K8s: Success
    else Unhealthy
        Probe-->>K8s: Failure
    end
```

### Probe Configuration

| Probe            | Endpoint   | Type |
| ---------------- | ---------- | ---- |
| API Liveness     | `/health`  | HTTP |
| API Readiness    | `/health`  | HTTP |
| Worker Liveness  | File check | File |
| Worker Readiness | File check | File |

## Resilience Patterns

### Pattern Overview

```mermaid
graph TB
    subgraph patterns["Resilience Patterns"]
        REC["Reconciler<br/>GitOps retry"]
        DLQ["Dead Letter Queue<br/>Failed tasks"]
        CB["Circuit Breakers<br/>Cascading failure protection"]
        BP["Backpressure<br/>Overload protection"]
        CANCEL["Cancellation<br/>Cooperative task stop"]
        BACKOFF["Exponential Backoff<br/>Retry delays"]
    end
```

### Reconciler

Automatic retry of failed GitOps syncs.

```mermaid
stateDiagram-v2
    [*] --> Synced
    Synced --> Failed: Sync error
    Failed --> Retrying: Retry scheduled
    Retrying --> Synced: Success
    Retrying --> Failed: Error persists
    Failed --> Manual: Max retries exceeded
```

**Retry Schedule:**

Uses exponential backoff with jitter, capped at a maximum delay.

### Dead Letter Queue

Failed tasks after maximum retries.

```mermaid
flowchart LR
    TASK["Task"] --> EXEC["Execute"]
    EXEC -->|"success"| DONE["Complete"]
    EXEC -->|"failure"| RETRY["Retry"]
    RETRY -->|"< max"| EXEC
    RETRY -->|"max reached"| DLQ["Dead Letter Queue"]
```

**DLQ Entry:**

```json
{
  "task_name": "process_query",
  "task_id": "task-123",
  "args": { ... },
  "error": "Tool execution timeout",
  "timestamp": "2024-01-22T16:35:00Z",
  "retry_count": 3
}
```

### Circuit Breakers

Redis-backed distributed circuit breaker.

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: Failure threshold
    Open --> HalfOpen: Timeout
    HalfOpen --> Closed: Success
    HalfOpen --> Open: Failure
```

| State         | Behavior                         |
| ------------- | -------------------------------- |
| **Closed**    | Normal operation, count failures |
| **Open**      | Reject requests immediately      |
| **Half-Open** | Allow limited requests to test   |

### Backpressure Control

Queue depth monitoring with request rejection.

```mermaid
flowchart TB
    REQ["New Request"] --> CHECK["Check Queue Depth"]
    CHECK -->|"low"| ACCEPT["Accept"]
    CHECK -->|"medium"| WARN["Accept + Warn"]
    CHECK -->|"high"| REJECT["Reject (503)"]
    REJECT --> RETRY["Retry-After: {delay}"]
```

| Threshold | Action                        |
| --------- | ----------------------------- |
| Low       | Normal processing             |
| Medium    | Accept with warning log       |
| High      | Reject with 503 + Retry-After |

### Cooperative Cancellation

Task cancellation via CancellationToken.

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Redis
    participant Worker

    Client->>API: DELETE /queries/{id}
    API->>Redis: Set cancellation flag
    API-->>Client: 202 Accepted

    loop Every 10 chunks
        Worker->>Redis: Check cancellation
        alt Cancelled
            Worker->>Worker: Stop processing
            Worker->>Redis: Publish cancelled status
        end
    end
```

### Backoff Tracker

Infrastructure failure handling with adaptive backoff.

```mermaid
flowchart TB
    ERROR["Error Occurred"] --> CLASSIFY["Classify Error"]
    CLASSIFY -->|"infrastructure"| INFRA["Infra Backoff"]
    CLASSIFY -->|"application"| APP["App Counter"]

    INFRA --> WAIT["Wait 2^n seconds<br/>(capped)"]
    WAIT --> RETRY["Retry Forever"]

    APP --> COUNT["Increment Counter"]
    COUNT -->|"below max"| RETRY2["Retry"]
    COUNT -->|"max reached"| FAIL["Give Up"]
```

| Error Type     | Behavior                           |
| -------------- | ---------------------------------- |
| Infrastructure | Exponential backoff, never give up |
| Application    | Count failures, give up after 10   |

## Logging

### Structured Logging

JSON-formatted logs for Cloud Logging integration.

```json
{
  "severity": "INFO",
  "message": "Query processed",
  "timestamp": "2024-01-22T16:30:00.123Z",
  "trace": "projects/xxx/traces/dontpanic42",
  "labels": {
    "gaim_id": "660e8400-...",
    "user_id": "dent42",
    "request_id": "req-456"
  },
  "httpRequest": {
    "requestMethod": "POST",
    "requestUrl": "/queries",
    "status": 200,
    "latency": "0.234s"
  }
}
```

### Log Levels

| Level    | Usage                        |
| -------- | ---------------------------- |
| DEBUG    | Detailed debugging info      |
| INFO     | Normal operations            |
| WARNING  | Potential issues             |
| ERROR    | Failures requiring attention |
| CRITICAL | System-level failures        |

### Correlation

Request tracing across services:

```mermaid
flowchart LR
    REQ["Request"] --> API["API<br/>trace: dontpanic42"]
    API --> REDIS["Redis<br/>trace: dontpanic42"]
    REDIS --> WORKER["Worker<br/>trace: dontpanic42"]
    WORKER --> TOOL["Tool<br/>trace: dontpanic42"]
```

## Infrastructure Health Monitor

Coordinates health checking across worker tasks.

```mermaid
flowchart TB
    subgraph tasks["Worker Tasks"]
        T1["Task 1"]
        T2["Task 2"]
        T3["Task 3"]
    end

    subgraph monitor["InfraHealthMonitor"]
        CHECK["Health Checks"]
        LOG["Coordinated Logging"]
    end

    tasks --> monitor
    monitor --> REDIS["Redis"]
    monitor --> MONGO["MongoDB"]
```

| Feature              | Description                       |
| -------------------- | --------------------------------- |
| Deduplication        | One log per failure, not per task |
| Recovery Detection   | Log when service recovers         |
| Backoff Coordination | Shared backoff state              |

## Queue Monitoring

### Queue Status Endpoint

```json
GET /system/queue

{
  "status": "healthy",
  "depth": 42,
  "consumers": 5,
  "oldest_message_age_seconds": 2.3,
  "processed_last_minute": 156,
  "failed_last_minute": 2
}
```

### Queue Alerts

| Condition            | Alert    |
| -------------------- | -------- |
| Depth elevated       | Warning  |
| Depth high           | Critical |
| Message age warning  | Warning  |
| Message age critical | Critical |
| No consumers         | Critical |

## Dashboard Recommendations

### Key Panels

1. **Request Rate** - `rate(gaim_http_requests_total[5m])`
2. **Error Rate** - `rate(gaim_http_requests_total{status=~"5.."}[5m])`
3. **Latency P95** - `histogram_quantile(0.95, gaim_http_duration_seconds)`
4. **Queue Depth** - `gaim_task_queue_depth`
5. **Active Connections** - `gaim_websocket_connections_active`
6. **LLM Token Usage** - `rate(gaim_llm_tokens_total[1h])`

### SLO Targets

| Metric       | Target             |
| ------------ | ------------------ |
| Availability | 99.9%              |
| Latency P95  | Within SLO targets |
| Error Rate   | < 0.1%             |
| Queue Depth  | Within threshold   |

## Related Documentation

- [Platform Overview](platform-overview.md) - Architecture context
- [Application Layer](application-layer.md) - Service metrics
- [Infrastructure Layer](infrastructure-layer.md) - Reconciler details
- [Data Stores](data-stores.md) - Redis queue details
