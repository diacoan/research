# Proactive Scaling Strategy: Monitoring Overview

## Purpose

Define the monitoring strategy that should feed proactive scaling decisions for Artifactory, its shared dependencies, and the surrounding GKE platform.

## Monitoring Objective

The monitoring goal is not only to detect failure.

It is to detect the approach to saturation early enough that operators can:

- distinguish valid load growth from waste
- identify the active bottleneck domain
- scale or tune the correct layer before user-visible degradation becomes widespread

## Monitoring Layers

The repo material supports seven monitoring layers that should be read together:

1. application request execution
2. DB progress and pool behavior
3. Cloud SQL Auth Proxy sidecar path
4. storage and local cache behavior
5. outbound remote-repository behavior
6. GKE node, pod, and node-pool capacity
7. observability quality and telemetry overhead

## Primary Signal Families

| Layer | Leading indicators | Confirmation indicators | Hard-failure indicators |
| --- | --- | --- | --- |
| App JVM and request path | rising latency with low CPU, heap free falling, active requests growing | DB wait, remote wait, or storage wait correlation | OOM, repeated long pauses, pod restart |
| DB pool and Cloud SQL | active connections near max, idle near zero, latency rising before CPU saturation | backend growth, lock or wait signals, query contention | connection failures, failover disruption, storage exhaustion |
| `cloud-sql-proxy` sidecar | rising `dial_latency`, sidecar CPU or memory growth, open connections rising | per-pod asymmetry against healthy instance-level metrics | restarts, `OOMKilled`, dial failures, refresh failures |
| GCS and local storage | bucket growth trend, local disk free falling, cache pressure | latency trend and cache inefficiency aligned with request load | write failures, temp-disk exhaustion, auth or policy failures |
| Remote repositories | high tiny/small-object latency, success ratio degradation, host concentration | repeated transport failures, high `404`, high failure cost on high-usage repos | persistent transport errors with no useful success |
| GKE capacity | allocatable headroom shrinking, requests growing, historical peaks tightening | node-pressure signals, throttling, packing inefficiency, autoscaler events | `FailedScheduling`, evictions, autoscaler out-of-resources |
| Telemetry quality | parser quality drift, unattributed events, duplicate series risk | missing labels, inconsistent windows, noisy or expensive logging | blind spots during incidents |

## Required Time Windows

The monitoring model in this repository is explicitly multi-window:

- `5m`, `15m`, `1h` for near-real-time symptom detection
- `24h` for incident context and trend confirmation
- `30d`, `90d` for headroom, growth pattern, and scaling decisions

Practical rule:

- short windows tell you what is hot now
- long windows tell you whether the platform shape is still appropriate

## Domain-Specific Priorities

### 1. Application and JVM

Monitor:

- heap pressure
- active DB pool usage
- request latency and concurrency
- CPU and memory utilization

Use this layer to tell whether requests are progressing or mostly waiting.

### 2. Cloud SQL and Proxy Path

Monitor both:

- shared Cloud SQL instance health
- per-pod `cloud-sql-proxy` behavior

This separation is mandatory because some bottlenecks are pod-local and invisible in coarse Cloud SQL dashboards.

### 3. Storage

Monitor both:

- final filestore behavior in GCS
- local cache, temp, and log disk behavior

GCS health alone is insufficient for Artifactory storage triage.

### 4. Remote Repositories

Monitor:

- transport failures
- `401`, `403`, and `404` separately
- tiny/small-object latency
- grouping by `repo_key` and `remote_host`

This layer is essential because invalid outbound demand can look like a scaling problem while actually being a repository-health or virtual-repository design problem.

### 5. GKE Capacity

Monitor:

- allocatable capacity
- requests and limits where present
- observed usage and historical peaks
- node-pressure evidence
- autoscaler failures and delayed scale behavior

This layer decides whether a scale action can actually land successfully in the cluster.

## Signal Hierarchy for Proactive Scaling

Use the following order when designing dashboards and alerts:

1. leading indicators
   - latency shift
   - active-vs-max pool pressure
   - rising asymmetry by pod
   - decreasing node headroom
2. confirmation indicators
   - cross-domain correlation
   - historical-window comparison
   - shared-host or shared-node concentration
3. hard-failure indicators
   - restarts
   - evictions
   - failed scheduling
   - repeated transport or dial failures

The goal is to act before stage 3 dominates.

## Monitoring Outputs That Must Exist

- quick-check dashboards for operators
- domain dashboards for DB, proxy, storage, remote repos, and GKE
- alert rules for repeated and high-cost failure patterns
- a backlog or review queue of repositories, services, or node pools that need action
- explicit parser-quality and telemetry-quality checks

## Monitoring Guardrails

- Avoid double counting when native and log-derived metrics coexist.
- Track missing attribution as its own operational problem.
- Keep repository health, proxy saturation, DB saturation, and GKE pressure as separate views before correlating them.
- Treat observability overhead as part of the monitored system, not as a free control plane.

## Detailed Sources

- [../artifactory-observability-quick-check.md](../artifactory-observability-quick-check.md)
- [../cloudsql-proxy-sidecar-for-artifactory-playbook.md](../cloudsql-proxy-sidecar-for-artifactory-playbook.md)
- [../cloudsql-postgres-for-artifactory-playbook.md](../cloudsql-postgres-for-artifactory-playbook.md)
- [../gcs-filestore-for-artifactory-playbook.md](../gcs-filestore-for-artifactory-playbook.md)
- [../gke-cluster-resource-assessment-playbook.md](../gke-cluster-resource-assessment-playbook.md)
- [../high-level-design-remote-repo-monitoring_revized.md](../high-level-design-remote-repo-monitoring_revized.md)
- [../next-steps-remote-repo-monitoring_revized.md](../next-steps-remote-repo-monitoring_revized.md)
- [../outbound-http-pool-tuning-for-remote-repositories.md](../outbound-http-pool-tuning-for-remote-repositories.md)
