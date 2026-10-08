# Monitoring and Observability

This repository adds monitoring and observability to the ML inference case study using Flux, Prometheus, Grafana, Loki, and Grafana Alloy.

## Monitoring strategy

I focused on the signals that help answer three practical questions:

1. Are the services available?
2. Are requests succeeding and completing within an acceptable time?
3. If requests fail, is the problem in the application, PostgreSQL, or Kubernetes?

| Component | Purpose |
|---|---|
| **Prometheus** | Collects application and Kubernetes metrics and evaluates alert rules |
| **ServiceMonitors** | Configure scraping for the ML API and Backend API `/metrics` endpoints |
| **Grafana** | Provides dashboards and log exploration |
| **Alloy** | Collects Kubernetes container logs |
| **Loki** | Stores and queries logs |
| **PrometheusRules** | Define GitOps-managed alerting rules |
| **Alertmanager** | Receives and routes firing alerts; external notification receivers are not configured yet |

The monitoring resources are reconciled by Flux from Git. Application ServiceMonitors and alert rules live with their application manifests, while the shared monitoring stack and Grafana dashboards live under `infrastructure/`.

## Dashboards

| Dashboard | Purpose |
|---|---|
| **ML API Overview** | Inference request volume, prediction throughput, request status, average latency, P95 latency, and 5xx rate |
| **Backend API Overview** | Document-processing traffic, request status, processing latency, 5xx rate, database connections, and database query failures |
| **PostgreSQL Overview** | Database connectivity, active connections, transaction activity, rollback rate, cache hit ratio, database size, deadlocks, and temporary data written |
| **Application Resource Health** | CPU and memory compared with requests and limits, OOM events, restarts, pod readiness, and application logs |
| **Application Alerts** | Current firing and pending application alerts grouped by service and severity |

The dashboards are provisioned from JSON files through labelled ConfigMaps, so the Git version is the source of truth. UI edits to provisioned dashboards are not expected to persist across reconciliation.

## Metrics monitored

| Metric | Why it matters | Dashboard |
|---|---|---|
| `ml_api_requests_total` | Request volume and HTTP failures by endpoint | ML API Overview |
| `ml_api_request_duration_seconds` | Prediction latency and tail latency | ML API Overview |
| `ml_api_predictions_total` | Inference throughput | ML API Overview |
| `ml_api_memory_bytes` | Application-level memory signal | ML API Overview |
| `backend_api_requests_total` | Document-processing traffic and HTTP 500 responses | Backend API Overview |
| `backend_api_request_duration_seconds` | `/process` latency and P95 latency | Backend API Overview |
| `backend_api_db_connections_active` | Active database connections reported by the Backend API | Backend API Overview |
| `backend_api_db_queries_total` | Database query volume and failures | Backend API Overview |
| `pg_up` | Whether the PostgreSQL exporter can connect to the database | PostgreSQL Overview |
| `pg_stat_database_numbackends` | Current active database connections | PostgreSQL Overview |
| `pg_stat_database_xact_commit` / `xact_rollback` | Commit and rollback activity | PostgreSQL Overview |
| `pg_stat_database_blks_hit` / `blks_read` | Buffer cache hit ratio and disk-read pressure | PostgreSQL Overview |
| `pg_database_size_bytes` | Database size and growth | PostgreSQL Overview |
| `pg_stat_database_deadlocks` | Deadlocks that can abort transactions | PostgreSQL Overview |
| `pg_stat_database_temp_bytes` | Temporary data written by large sorts or joins | PostgreSQL Overview |
| `container_memory_working_set_bytes` | Kubernetes container memory usage | Application Resource Health |
| `container_cpu_usage_seconds_total` | Kubernetes container CPU usage | Application Resource Health |
| `kube_pod_container_resource_requests` / `limits` | Resource allocation context | Application Resource Health |
| `container_oom_events_total` | Container OOM events during the selected time range | Application Resource Health |
| `kube_pod_container_status_restarts_total` | Repeated container restarts | Application Resource Health |
| `kube_pod_status_ready` | Ready and unready pod counts | Application Resource Health |
| `ALERTS` | Current pending and firing alert state | Application Alerts |

Dashboard resource values are averaged across matching replicas so usage, requests, and limits represent a typical replica. Alert expressions use the maximum individual pod usage where appropriate so one overloaded replica is not hidden by healthy replicas.

## Alerting approach

Prometheus evaluates the rules every 30 seconds. The `for` duration prevents a short-lived spike from immediately becoming an incident. The rules are intentionally scoped to the relevant namespace and container.

| Alert | Condition | Duration | Severity |
|---|---|---:|---|
| `MLAPIHighErrorRate` | ML API 5xx rate is above 5% | 5m | Critical |
| `MLAPIHighPredictionLatency` | P95 `/predict` latency is above 1 second | 10m | Warning |
| `MLAPIDeploymentReplicasUnavailable` | One or more ML API replicas are unavailable | 5m | Critical |
| `MLAPIContainerRestarts` | More than two ML API restarts in 15 minutes | 5m | Warning |
| `MLAPIHighMemoryUsage` | A replica is above 80% of its memory limit | 10m | Warning |
| `BackendAPIHighErrorRate` | Backend API 5xx rate is above 5% | 5m | Critical |
| `BackendAPIHighProcessingLatency` | P95 `/process` latency is above 1 second | 10m | Warning |
| `BackendAPIDeploymentReplicasUnavailable` | One or more Backend API replicas are unavailable | 5m | Critical |
| `BackendAPIHighDatabaseErrorRate` | More than 1% of reported DB queries fail | 5m | Critical |
| `BackendAPIContainerRestarts` | More than two Backend API restarts in 15 minutes | 5m | Warning |
| `BackendAPIHighMemoryUsage` | A replica is above 80% of its memory limit | 10m | Warning |
| `PostgreSQLUnavailable` | PostgreSQL has no available replica | 5m | Critical |
| `PostgreSQLContainerRestarts` | PostgreSQL restarts more than twice in 15 minutes | 5m | Warning |
| `PostgreSQLHighRollbackRate` | More than 5% of transactions roll back | 5m | Warning |
| `PostgreSQLDeadlocksDetected` | One or more deadlocks occur in 15 minutes | 5m | Warning |

At present, Alertmanager receives firing alerts but routes them to its default `null` receiver. Alert state is therefore visible in Prometheus, Alertmanager, and the Application Alerts dashboard, but no external notification is sent yet.

## Example incident

The current demo exposes a useful Backend API failure scenario: PostgreSQL is reachable, but the expected `documents` table is not initialized.

```text
POST /process
    -> Backend API inserts into documents
    -> PostgreSQL reports: relation "documents" does not exist
    -> Backend API returns HTTP 500
```

The monitoring signals make the failure easy to follow:

1. PostgreSQL logs show the missing relation.
2. `backend_api_db_queries_total{status="error"}` increases.
3. `backend_api_requests_total{endpoint="/process",status="500"}` increases.
4. `BackendAPIHighDatabaseErrorRate` fires.
5. `BackendAPIHighErrorRate` fires.
6. Grafana logs, metrics, and the Application Alerts dashboard point to the same dependency failure.

This demonstrates why both user-facing alerts and dependency-specific alerts are useful: one shows impact, while the other helps identify the likely cause.

## Tradeoffs and future improvements

- **k3d and `local-path` storage:** Simple for local development, but not suitable for highly available production storage.
- **Single-binary Loki with filesystem storage:** Appropriate for this small cluster; production would use object storage and a scalable Loki deployment.
- **Application-owned ServiceMonitors and PrometheusRules:** Keeps monitoring changes close to the application and makes them reviewable in Git.
- **Git-managed Grafana dashboards:** Provides reproducible dashboards, but UI edits to provisioned dashboards are not authoritative.
- **No external Alertmanager receiver yet:** This keeps the local setup self-contained. Mailpit, Slack, or PagerDuty could be added later.
- **No automated PostgreSQL schema migration:** A migration or initialization Job would create the `documents` table declaratively.
- **Alert thresholds are initial working values:** They should be tuned against production baselines and service-level objectives.

## Verification

```bash
kubectl get pods -A
flux get all
kubectl get servicemonitor -A
kubectl get prometheusrule -A
```

To inspect alert state in Prometheus:

```promql
ALERTS{alertstate="pending"}
```

```promql
ALERTS{alertstate="firing"}
```
