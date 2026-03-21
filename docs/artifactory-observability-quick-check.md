# Artifactory HA Observability Quick Check and Tuning

## Purpose

Document what the metrics in `log-analytics-prometheus-master` measure, which `artifactoy-ha` Helm parameters influence them, how those parameters relate to each other, and which patterns usually indicate bottlenecks in a self-managed Artifactory HA deployment.

This document is intentionally operational. It is meant to support:

- quick triage
- platform tuning
- correlation between Prometheus signals and Helm values
- GCP-oriented fine tuning for:
  - `google-storage-v2-direct`
  - Cloud SQL PostgreSQL
  - cloud-native full-primary HA

## Reading Guide

Use this document in the following order:

1. Start with the quick-check table.
2. Identify the dominant symptom.
3. Read the corresponding logical flow section.
4. Only then change Helm values.

## Telemetry Path

There are two metric paths in the repository:

1. Native OpenMetrics endpoints exposed by Artifactory and Observability.
2. Log-derived metrics parsed from `artifactory-metrics.log`, `observability-metrics.log`, and Xray metric logs.

In the current example integration, Prometheus discovery is built mainly around the native product endpoints and `ServiceMonitor` objects:

- [log-analytics-prometheus-master/helm/artifactory-values.yaml](../log-analytics-prometheus-master/helm/artifactory-values.yaml)
- [log-analytics-prometheus-master/helm/xray-values.yaml](../log-analytics-prometheus-master/helm/xray-values.yaml)

The Fluentd metric configs are still useful because they show the canonical metric names and the log-derived fallback path:

- [log-analytics-prometheus-master/fluent_metrics.conf.rt](../log-analytics-prometheus-master/fluent_metrics.conf.rt)
- [log-analytics-prometheus-master/fluent_metrics.conf.xray](../log-analytics-prometheus-master/fluent_metrics.conf.xray)

Operational consequence:

- Prometheus and Grafana usually perform the real aggregation.
- Fluentd mostly transforms metric log lines into Prometheus-compatible series.
- If dashboards show metric names from both native and log-derived sources, treat label sets carefully and avoid double-counting.

## Quick Check

| Metric or pattern | Primary Helm parameter(s) | Risk signal | Recommended action |
| --- | --- | --- | --- |
| `jfrt_runtime_heap_freememory_bytes` falling while `jfrt_runtime_heap_totalmemory_bytes` approaches `jfrt_runtime_heap_maxmemory_bytes` | `artifactory.primary.javaOpts.xms`, `artifactory.primary.javaOpts.xmx`, `artifactory.primary.resources.memory` | JVM heap pressure, GC churn, eventual OOM or long pauses | Increase memory and heap together; do not raise `xmx` without checking container limit and non-heap overhead |
| `jfrt_db_connections_active_total` near `jfrt_db_connections_max_active_total`, `idle` near zero | `artifactory.database.maxOpenConnections`, `access.database.maxOpenConnections`, DB server capacity | DB pool saturation, request blocking, rising latency with low CPU | Validate DB headroom first, then raise pool values conservatively; do not increase Tomcat threads first |
| High active DB connections after raising request concurrency | `artifactory.tomcat.connector.maxThreads`, `access.tomcat.connector.maxThreads`, `artifactory.primary.replicaCount` | Concurrency moved contention into the DB tier | Rebalance threads and DB pools together; check Cloud SQL limits and actual DB wait |
| High latency, high active requests, low CPU | `artifactory.tomcat.connector.maxThreads`, outbound HTTP pool values in `javaOpts.other`, storage type | Waiting, not useful progress; likely remote, DB, Access, or storage wait | Find the dominant wait source first; avoid treating it as a CPU-only problem |
| `app_disk_free_bytes` falling fast while `jfrt_storage_current_total_size_bytes` is stable | `artifactory.persistence.size`, `artifactory.persistence.maxCacheSize`, `artifactory.loggers`, `filebeat.enabled` | Local cache, temp, or log pressure rather than final filestore growth | Review cache sizing, local PVC size, sidecar log volume, and retention |
| `jfrt_storage_current_total_size_bytes` growing steadily | `artifactory.persistence.type`, external filestore capacity, lifecycle/retention policy | Filestore growth, future storage exhaustion | Validate object storage growth policy and repository cleanup before adding more request concurrency |
| `sys_cpu_ratio` high with healthy DB and enough disk | `artifactory.primary.resources.cpu`, `artifactory.primary.replicaCount`, `nginx.replicaCount` | Real compute saturation | Scale compute horizontally or vertically only after verifying wait sources are not dominant |
| `sys_memory_free_bytes` low across multiple pods | `artifactory.primary.resources`, `access.resources`, `metadata.resources`, `observability.resources` | Shared node or pod memory pressure | Reduce noisy sidecar overhead and rebalance requests/limits before changing only heap |
| Nginx-side latency or queueing symptoms with healthy Artifactory internals | `nginx.replicaCount`, `nginx.resources`, `nginx.disableProxyBuffering`, ingress architecture | Front-door bottleneck | Scale Nginx or move to external ingress if appropriate; keep anti-affinity and dedicated capacity |
| High latency after enabling extra loggers, Filebeat, or aggressive observability | `artifactory.loggers`, `nginx.loggers`, `filebeat.enabled`, `observability.resources` | Telemetry overhead consuming platform headroom | Cut unnecessary log volume, limit sidecar resources, and keep only diagnosis-critical streams |
| `db_connection_pool_in_use_total / db_connection_pool_max_open_total` high in Xray | Xray chart values, not `artifactoy-ha` | Xray DB saturation | Tune in the Xray chart; do not assume Artifactory HA values will help |
| Healthy app metrics but poor user experience after scaling replicas | `artifactory.primary.replicaCount`, shared DB, shared storage, anti-affinity | Shared backend bottleneck or placement issue | Check DB/storage before adding more replicas; verify zone spread and node pool isolation |

## Metric Families and What They Mean

### Heap and JVM Runtime

Relevant metrics:

- `jfrt_runtime_heap_freememory_bytes`
- `jfrt_runtime_heap_totalmemory_bytes`
- `jfrt_runtime_heap_maxmemory_bytes`
- `jfrt_runtime_heap_processors_total`

Relevant values:

- `artifactory.primary.javaOpts.xms`
- `artifactory.primary.javaOpts.xmx`
- `artifactory.primary.resources`
- `artifactory.primary.replicaCount`

Sources:

- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- [artifactoy-ha/files/system.yaml](../artifactoy-ha/files/system.yaml)
- [log-analytics-prometheus-master/fluent_metrics.conf.rt](../log-analytics-prometheus-master/fluent_metrics.conf.rt)

What these metrics tell you:

- `free` is currently unused heap inside the JVM.
- `total` is heap currently committed by the JVM.
- `max` is the upper heap ceiling.
- `processors_total` is how many CPUs the runtime sees.

Practical interpretation:

- `free` low by itself is not enough to call a problem.
- `free` low and `total ~= max` for sustained periods usually means genuine heap pressure.
- High CPU together with low free heap can point to GC churn.
- Low CPU together with low free heap often means memory pressure is present, but it is not yet the dominant wait source.

Fine point:

- The chart lets you set `xms/xmx`, but container memory must still cover heap, metaspace, direct memory, thread stacks, native buffers, sidecars, and filesystem cache.
- The sizing presets already include non-heap Java tuning in `javaOpts.other`, for example in:
  - [artifactoy-ha/sizing/artifactory-small-extra-config.yaml](../artifactoy-ha/sizing/artifactory-small-extra-config.yaml)
  - [artifactoy-ha/sizing/artifactory-large-extra-config.yaml](../artifactoy-ha/sizing/artifactory-large-extra-config.yaml)

### Database Pool Pressure

Relevant metrics:

- `jfrt_db_connections_active_total`
- `jfrt_db_connections_idle_total`
- `jfrt_db_connections_max_active_total`
- `jfrt_db_connections_min_idle_total`

Relevant values:

- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`
- `metadata.database.maxOpenConnections`
- `artifactory.tomcat.connector.maxThreads`
- `access.tomcat.connector.maxThreads`
- `postgresql.postgresqlExtendedConf.maxConnections` when using the in-chart PostgreSQL

Sources:

- [artifactoy-ha/files/system.yaml](../artifactoy-ha/files/system.yaml)
- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- [artifactoy-ha/charts/postgresql/values.yaml](../artifactoy-ha/charts/postgresql/values.yaml)

What these metrics tell you:

- `active` is current DB concurrency in the Artifactory pool.
- `idle` is available cushion.
- `max_active` is the configured ceiling for that pool.

Practical interpretation:

- `active` repeatedly close to `max_active` with `idle ~= 0` is a strong saturation signal.
- If response times increase while CPU remains moderate, the blocked work may be sitting behind the DB pool.
- If you add Tomcat threads before relieving DB pressure, you often increase waiting requests rather than useful throughput.

Official guidance aligned with this document:

- JFrog recommends PostgreSQL for platform services.
- Database connection configuration is part of `system.yaml`.
- DB HA remains active-passive with failover at the database tier.

Reference:

- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/database/>

### Storage and Local Disk

Relevant metrics:

- `jfrt_storage_current_total_size_bytes`
- `app_disk_used_bytes`
- `app_disk_free_bytes`

Relevant values:

- `artifactory.persistence.type`
- `artifactory.persistence.size`
- `artifactory.persistence.maxCacheSize`
- `artifactory.persistence.googleStorage.*`
- `artifactory.persistence.nfs.*`

Sources:

- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- [artifactoy-ha/files/binarystore.xml](../artifactoy-ha/files/binarystore.xml)

What these metrics tell you:

- `jfrt_storage_current_total_size_bytes` is logical Artifactory storage usage.
- `app_disk_*` is application-local disk pressure.

Interpretation by storage type:

- `file-system`: local disk pressure and final filestore growth are closely coupled.
- `nfs`: local disk usually reflects cache/temp/logs, while final data sits remotely; latency can still dominate.
- `google-storage-v2-direct`: final binaries are in GCS, while local disk still matters for `cache-fs`, temp work, logs, and transient operational overhead.

Important JFrog guidance:

- Object storage is the recommended final filestore.
- NFS is not recommended for high-load environments.
- Local SSD performance still matters for cache and temp work even with object storage.

Reference:

- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/>

### System CPU and Memory

Relevant metrics:

- `sys_cpu_ratio`
- `sys_memory_used_bytes`
- `sys_memory_free_bytes`

Relevant values:

- `artifactory.primary.resources`
- `access.resources`
- `metadata.resources`
- `observability.resources`
- `nginx.resources`

Sources:

- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- [artifactoy-ha/templates/artifactory-primary-statefulset.yaml](../artifactoy-ha/templates/artifactory-primary-statefulset.yaml)
- [artifactoy-ha/templates/artifactory-node-statefulset.yaml](../artifactoy-ha/templates/artifactory-node-statefulset.yaml)

Important detail:

- The Observability service is deployed as a sidecar in Artifactory pods.
- Filebeat and extra loggers are also sidecars when enabled.
- This means observability can consume the same CPU, memory, disk I/O, and network headroom that operators are trying to diagnose.

Engineering implication:

- If observability is over-configured, it can worsen the exact symptoms it is supposed to explain.
- This is especially relevant during incidents with high log volume or aggressive sidecar collection.

Reference:

- <https://jfrog.com/reference-architecture/observability/>

## Logical Flow for Triage

### 1. Edge and Ingress

Start here when users report:

- intermittent slowness at login or UI load
- slow downloads before any internal saturation signal is obvious
- inconsistent behavior across replicas

Primary values:

- `nginx.replicaCount`
- `nginx.resources`
- `nginx.disableProxyBuffering`
- `nginx.service.externalTrafficPolicy`
- `nginx.logs.*`

Notes:

- `nginx.disableProxyBuffering=true` disables request and response buffering in the bundled Nginx config.
- This can reduce latency amplification for streaming-like behavior, but it also changes memory and connection behavior at the edge.
- JFrog sizing guidance treats Nginx as its own capacity domain and recommends separate node pool isolation for ingress/Nginx in serious deployments.

Reference:

- <https://jfrog.com/reference-architecture/self-managed/deployment/sizing/aws/>
- <https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/>

### 2. Artifactory Request Concurrency

Primary values:

- `artifactory.tomcat.connector.maxThreads`
- `artifactory.tomcat.connector.extraConfig`
- `artifactory.primary.replicaCount`

Interpretation:

- `maxThreads` controls active request-handling concurrency per pod.
- `acceptCount` in `extraConfig` is backlog depth, not productive throughput.
- Increasing `maxThreads` in isolation can enlarge the number of blocked requests.

Use this when:

- active requests are high
- latency is rising
- CPU is not fully saturated

This usually means waiting is accumulating somewhere else: DB, Access, remote upstream, or storage.

### 3. Access and Authorization Flow

Primary values:

- `access.tomcat.connector.maxThreads`
- `access.database.maxOpenConnections`
- `access.resources`

Hidden but important relationship:

- The chart renders `-Dartifactory.access.client.max.connections={{ .Values.access.tomcat.connector.maxThreads }}` into `shared.extraJavaOpts`.
- That means Access server concurrency and Artifactory client concurrency toward Access are coupled by default.

Operational consequence:

- Raising Artifactory request concurrency may require revisiting Access thread capacity.
- If permission-heavy requests slow down while core artifact traffic is only moderate, Access is a plausible choke point.

### 4. Shared Database

Primary values:

- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`
- `metadata.database.maxOpenConnections`
- external DB capacity or in-chart PostgreSQL capacity

Interpretation:

- More replicas increase shared DB demand.
- More Tomcat threads increase shared DB overlap.
- More remote wait can indirectly increase DB overlap because requests live longer.

This is why DB tuning must be treated as a shared dependency problem, not as a per-pod-only problem.

### 5. Filestore and Local Cache

Primary values:

- `artifactory.persistence.type`
- `artifactory.persistence.maxCacheSize`
- `artifactory.persistence.size`

Interpretation:

- Final binary storage and local cache are different capacity domains.
- Object storage fixes some scalability problems, but not local SSD pressure, cache size, temp work, or log growth.

When disk-related metrics mislead operators:

- If final binaries live in GCS, `app_disk_free_bytes` does not tell you bucket headroom.
- It tells you node-local or PVC-local headroom.
- This is still critical because local disk exhaustion can degrade or destabilize the pod even when GCS is healthy.

## Parameter Relationships That Matter Most

### `artifactory.tomcat.connector.maxThreads` vs DB Pools

This is the most common tuning mistake.

If `maxThreads` rises materially, review at least:

- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`
- actual DB CPU, wait, and connection limits

Do not assume that more threads means more throughput.

### `artifactory.primary.resources.memory` vs JVM heap

If you increase:

- `artifactory.primary.javaOpts.xmx`

you should also review:

- `artifactory.primary.resources.requests.memory`
- `artifactory.primary.resources.limits.memory`
- sidecar resource limits

Otherwise you create a larger JVM inside the same pod memory envelope.

### `artifactory.primary.replicaCount` vs Shared Backends

Cloud-native HA improves availability and horizontal compute scale, but:

- storage remains shared
- database remains shared

So scaling replicas can simply move the bottleneck:

- from CPU to DB
- from DB to storage
- from a single busy pod to a saturated shared service

### `artifactory.persistence.type` vs Metric Interpretation

Use the storage type before interpreting disk signals:

- `file-system`: disk metrics often reflect final data growth
- `google-storage-v2-direct`: disk metrics mostly reflect cache/temp/log behavior
- `nfs`: disk pressure may stay moderate while latency is still poor

## Fine Tuning for GCS `google-storage-v2-direct`

### Recommended Baseline

Relevant values exposed directly by the standard chart:

- `artifactory.persistence.type=google-storage-v2-direct`
- `artifactory.persistence.googleStorage.bucketName`
- `artifactory.persistence.googleStorage.path`
- `artifactory.persistence.googleStorage.useInstanceCredentials`
- `artifactory.persistence.googleStorage.enableSignedUrlRedirect`
- `artifactory.persistence.maxCacheSize`

Relevant advanced values supported by the product but not exposed by the standard GCS Helm section:

- `maxConnections`
- `connectionTimeout`

Chart evidence:

- The chart renders a direct chain for `google-storage-v2-direct` where `cache-fs` wraps `google-storage-v2`.
- It does not use the eventual-cluster path used by the older Google storage modes.
- In the standard `googleStorage` Helm values, the chart exposes bucket, path, endpoint, auth mode, and signed URL redirect settings, but not `maxConnections` or `connectionTimeout` for GCS.

Source:

- [artifactoy-ha/files/binarystore.xml](../artifactoy-ha/files/binarystore.xml)
- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- <https://jfrog.com/help/r/jfrog-installation-setup-documentation/google-storage-v2-direct-template-configuration-recommended>

Why it matters:

- This is the cleanest GCS mode in this chart for modern production use.
- It reduces the older eventual-upload style complexity.
- It still depends on local cache performance.
- JFrog's official `google-storage-v2-direct` template includes `maxConnections` and `connectionTimeout`, which means these are valid product-level knobs even though the local chart does not surface them as first-class Helm values.

Practical recommendations:

- Prefer `google-storage-v2-direct` over legacy Google storage modes.
- Prefer workload identity or equivalent instance credentials over long-lived static key material when the platform design allows it.
- Keep `cache-fs` meaningful; object storage does not eliminate the need for fast local disk.
- Size local PVC or node-local storage for cache, temp work, and logs, not only for the misconception that "GCS stores everything anyway".
- If you need to tune GCS client concurrency or connect behavior, use a custom `binarystore.xml` path instead of expecting the stock `googleStorage` Helm values to carry those settings.
- In this chart, the usual entry points for that are:
  - `artifactory.persistence.customBinarystoreXmlSecret`
  - a fully customized `binarystore.xml`

Interpretation of the advanced knobs:

- `maxConnections` controls the Artifactory provider-side concurrency toward GCS.
- `connectionTimeout` controls how long Artifactory waits while establishing the connection to GCS.

When to consider them:

- Consider `maxConnections` only if the platform is healthy on heap, DB, and CPU, and storage concurrency toward GCS is the suspected limiter.
- Consider `connectionTimeout` only when the issue is clearly on connection establishment or network path behavior, not when transfers are already established and simply slow.

Engineering inference:

- If small-object download latency is high while GCS itself is healthy, investigate local cache behavior before changing GCS settings.
- If `app_disk_free_bytes` falls while bucket growth is stable, local operational disk pressure is likely the immediate issue.
- If throughput remains poor after scaling Artifactory pods, do not assume the next move is higher `maxConnections`; first verify whether the real wait source is DB, Access, Nginx, or local cache.

## Fine Tuning for Cloud SQL PostgreSQL

### Recommended Baseline

JFrog guidance for production-grade deployments is to use an external PostgreSQL rather than the in-chart dependency.

Relevant values:

- `postgresql.enabled=false`
- `database.type=postgresql`
- `database.driver=org.postgresql.Driver`
- `database.url`
- `database.user`
- `database.password` or `database.secrets.*`
- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`
- `metadata.database.maxOpenConnections`

Sources:

- [artifactoy-ha/README.md](../artifactoy-ha/README.md)
- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/database/>

Operational implications:

- `postgresql.postgresqlExtendedConf.maxConnections` only helps if you are using the in-chart PostgreSQL.
- For Cloud SQL, the DB-side max connections and HA behavior are managed in Cloud SQL, not inside the Helm chart.
- Helm still controls the application-side pool pressure.

Recommended operating model:

- Keep application pool sizes below proven Cloud SQL headroom.
- Treat DB connections as a shared budget across Artifactory, Access, and Metadata.
- Prefer private connectivity and low-latency network placement between GKE and Cloud SQL.

Engineering inference:

- If Cloud SQL CPU, lock wait, or connection pressure is already visible, do not raise `artifactory.database.maxOpenConnections` first.
- If application latency is high but DB connection use is low, the bottleneck is probably elsewhere and more DB pool will not help.

### Cloud SQL Quick Rules

- Increase app pools only after validating Cloud SQL headroom.
- Revisit pool sizes whenever `artifactory.primary.replicaCount` changes.
- Keep HA thinking separate:
  - Artifactory HA is horizontal application scale.
  - Cloud SQL HA is database failover behavior.

## Fine Tuning for Full-Primary HA

### What It Means in This Chart

The chart defaults now support cloud-native HA:

- `artifactory.primary.replicaCount: 3`
- `artifactory.node.replicaCount: 0`

Sources:

- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- [artifactoy-ha/CHANGELOG.md](../artifactoy-ha/CHANGELOG.md)
- [artifactoy-ha/README.md](../artifactoy-ha/README.md)

Meaning:

- All active Artifactory pods are primary-role pods from the chart's perspective.
- Resource and Java tuning should therefore be applied under `artifactory.primary.*`.
- Older mental models based on one primary plus member nodes are no longer the safe default for this chart version.

### What to Tune First

Primary values:

- `artifactory.primary.replicaCount`
- `artifactory.primary.resources`
- `artifactory.primary.javaOpts`
- `artifactory.primary.podAntiAffinity`
- `artifactory.topologySpreadConstraints`
- `nginx.replicaCount`

Operational rules:

- Keep anti-affinity meaningful. Soft anti-affinity is better than nothing; hard anti-affinity is usually better for strict HA.
- Spread Artifactory pods across nodes and ideally zones.
- Do not scale replicas without reviewing shared DB and shared filestore headroom.
- Treat Nginx or external ingress as a separate scale decision.

Official guidance aligned here:

- JFrog recommends clustered deployment for highest availability.
- In Kubernetes, JFrog guidance calls for three Artifactory instances.
- Shared storage and shared DB remain common dependencies.

Reference:

- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/high-availability/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/runtime-platform/>

## Observability Overhead Review

Review observability settings when:

- log volume spikes during incidents
- sidecars compete for pod memory
- local disk shrinks faster than business data grows
- dashboards become noisy or duplicate similar metrics

Relevant values:

- `observability.enabled`
- `observability.resources`
- `artifactory.loggers`
- `artifactory.loggersResources`
- `nginx.loggers`
- `nginx.loggersResources`
- `filebeat.enabled`
- `filebeat.resources`

Rule of thumb:

- Keep enough signal to identify the dominant wait source.
- Do not collect so much telemetry that the telemetry itself becomes a capacity problem.

## Xray Note

The repository also contains Xray metric mappings:

- `sys_cpu_ratio`
- `sys_memory_*`
- `app_disk_*`
- `go_memstats_heap_*`
- `db_connection_pool_*`

Those belong to the Xray deployment path, not to `artifactoy-ha`.

If the next step is Xray tuning, the same quick-check model should be repeated against the Xray chart and its DB, indexing, and analysis services.

## Source Notes

Primary local sources:

- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- [artifactoy-ha/files/system.yaml](../artifactoy-ha/files/system.yaml)
- [artifactoy-ha/files/binarystore.xml](../artifactoy-ha/files/binarystore.xml)
- [artifactoy-ha/charts/postgresql/values.yaml](../artifactoy-ha/charts/postgresql/values.yaml)
- [log-analytics-prometheus-master/fluent_metrics.conf.rt](../log-analytics-prometheus-master/fluent_metrics.conf.rt)
- [log-analytics-prometheus-master/fluent_metrics.conf.xray](../log-analytics-prometheus-master/fluent_metrics.conf.xray)
- [log-analytics-prometheus-master/helm/artifactory-values.yaml](../log-analytics-prometheus-master/helm/artifactory-values.yaml)

JFrog official references used for interpretation:

- <https://jfrog.com/reference-architecture/observability/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/database/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/high-availability/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/runtime-platform/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/sizing/aws/>
- <https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/>

## Follow-Up Topics

Good next candidates for the same style of appendix:

- ingress controller external to the bundled Nginx
- Xray DB and analysis/indexer capacity
- GKE node pool layout and autoscaling policy
- Prometheus alert rules for the quick-check signals

Related appendix already added:

- [outbound-http-pool-tuning-for-remote-repositories.md](./outbound-http-pool-tuning-for-remote-repositories.md)
