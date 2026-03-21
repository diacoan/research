# Cloud SQL Auth Proxy Sidecar for Artifactory Playbook

## Purpose

Provide a complete playbook for identifying, monitoring, and tuning situations where Artifactory primary pods are bottlenecked by the per-pod Cloud SQL Auth Proxy sidecar rather than by the Cloud SQL PostgreSQL instance itself.

This playbook is intended for environments where:

- Artifactory runs in Kubernetes
- Cloud SQL for PostgreSQL is external to the Helm chart
- each Artifactory primary pod has a colocated Cloud SQL Auth Proxy sidecar
- operators need to distinguish instance-side database saturation from pod-local proxy saturation

Use it together with:

- [cloudsql-postgres-for-artifactory-playbook.md](./cloudsql-postgres-for-artifactory-playbook.md)
- [artifactory-observability-quick-check.md](./artifactory-observability-quick-check.md)
- [high-level-artifactory-capacity-interdependencies.md](./high-level-artifactory-capacity-interdependencies.md)
- [gke-cluster-resource-assessment-playbook.md](./gke-cluster-resource-assessment-playbook.md)

## Problem Statement

In a full-primary HA topology, Cloud SQL can look healthy at the instance level while some Artifactory primaries still show DB-related latency.

That can happen because the request path is:

`Artifactory JVM -> localhost TCP -> Cloud SQL Auth Proxy sidecar -> network path -> Cloud SQL`

If the bottleneck is inside the proxy sidecar or on the pod or node hosting it:

- Cloud SQL CPU, memory, and backend counts may remain moderate
- only one or a subset of primaries may degrade
- Artifactory-side DB waits can rise without a clear instance-level Cloud SQL alarm
- increasing app pools or replica count can make the pattern worse

## Stability, Performance, and Functionality Impact

Proxy-side bottlenecks affect Artifactory in three ways:

- Stability impact:
  - proxy restart or `OOMKilled`
  - connection setup failures
  - intermittent readiness or startup failures
- Performance impact:
  - slower DB connection establishment
  - higher request latency on affected primaries
  - uneven behavior across Artifactory primaries
  - amplified queueing behind application-side connection pools
- Functionality impact:
  - login, metadata, and repository operations become inconsistent by pod
  - HA replicas remain available but do not contribute equally
  - background jobs and maintenance flows can lag on specific primaries

## Artifactory Context

For Artifactory, the proxy is not the database itself and it is not a DB connection pooler.

It is a secure gateway that:

- accepts local TCP connections from the Java process
- authenticates to Cloud SQL using the pod's IAM identity or credentials
- encrypts traffic to Cloud SQL
- performs connection establishment, certificate refresh, and related control-plane operations

Important context for Artifactory:

- Artifactory is a Java application and Google documents that Unix sockets are not supported for Java applications in this connection model, so the practical local path is TCP to `127.0.0.1:<DB_PORT>`.
- In this repo, the standard `artifactoy-ha` chart defines Artifactory DB pools and Tomcat concurrency, but it does not natively template a Cloud SQL proxy sidecar. Inference: the sidecar is added through cluster-specific manifests, overlays, or an admission/injection workflow outside the standard chart.
- More Artifactory primaries mean more proxy sidecars and more total connection demand presented to Cloud SQL.

Relevant Artifactory-side values in this repo:

- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`
- `metadata.database.maxOpenConnections`
- `artifactory.tomcat.connector.maxThreads`
- `access.tomcat.connector.maxThreads`
- `artifactory.primary.replicaCount`

Relevant local sources:

- [artifactoy-ha/files/system.yaml](../artifactoy-ha/files/system.yaml)
- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)

## Quick Check

| Observed behavior | Likely meaning | What to verify next | Typical action |
| --- | --- | --- | --- |
| One or two primaries are slow, but Cloud SQL instance CPU and memory look normal | Localized proxy-side or node-local bottleneck | Compare per-primary proxy CPU, memory, restarts, and proxy telemetry | Tune sidecar resources first; do not increase Cloud SQL blindly |
| Proxy CPU usage is high and Artifactory DB wait rises on the same pod | Proxy is IO-path constrained | Check sidecar requests, limits, and CPU throttling signals | Increase sidecar CPU request and limit; review node pressure |
| Proxy memory usage grows with active DB connections | Sidecar memory is scaling with connection count | Compare sidecar memory to active connections and Artifactory pool settings | Increase memory or reduce pool/concurrency on affected primaries |
| `dial_latency` rises and `dial_failure_count` or `refresh_failure_count` increases while Cloud SQL instance looks healthy | Bottleneck is in proxy connection establishment, network path, IAM, or SQL Admin API interaction | Check proxy logs, network path, IAM, and Admin API quota patterns | Fix path or quota issue before changing DB instance size |
| App-side DB pools are busy but Cloud SQL `num_backends` is still moderate | App requests are queueing before reaching the instance efficiently | Compare Artifactory DB pool metrics with proxy `open_connections` by pod | Investigate proxy saturation or connection churn |
| Proxy logs show request quota or refresh errors | Large deployment or connection churn is stressing proxy control-plane calls | Review number of proxies, startup churn, and connection churn | Keep pools stable, reduce churn, and evaluate pooling patterns if needed |
| Proxy container restarts or is `OOMKilled` | Sidecar is undersized or unstable | `kubectl describe pod`, pod events, restart count, memory peaks | Raise memory and stabilize before tuning DB pools |

## Working Principles

Use these principles throughout the playbook:

- the proxy is a per-pod failure domain
- the proxy is not a substitute for app-side connection pooling
- proxy CPU scales with IO between app and database
- proxy memory scales with active connections
- Cloud SQL instance metrics alone are insufficient for diagnosing proxy-side bottlenecks
- proxy-side issues are often asymmetric across primaries
- scaling Artifactory replicas also scales proxy count and control-plane activity

## Sub-Topic 1: End-to-End Path and Failure Domain

### Purpose

Identify where proxy-side bottlenecks live in the request path.

### Problem Statement

If the request path is treated as a single opaque DB dependency, operators tend to misclassify pod-local proxy issues as Cloud SQL instance issues.

### Stability, Performance, and Functionality Impact

Misclassification leads to:

- wrong scaling actions
- wasted Cloud SQL upsizing
- overgrowth of app-side pools
- persistent asymmetry across primaries

### Concept / Explanation

For each Artifactory primary pod, the path is:

1. the Artifactory JVM asks its local JDBC pool for a connection
2. the application opens or reuses a local TCP connection to `127.0.0.1:<DB_PORT>`
3. the Cloud SQL Auth Proxy sidecar authenticates and dials Cloud SQL
4. Cloud SQL accepts the backend connection

This means there are three separate observation planes:

- application plane: Artifactory DB pool and request latency
- proxy plane: sidecar CPU, memory, telemetry, and logs
- database plane: Cloud SQL instance metrics and PostgreSQL-level state

### Functionality

This model lets you isolate whether contention is:

- before the database instance
- inside the database instance
- or across both planes

### Metrics / Parameters

Application-side:

- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`
- `metadata.database.maxOpenConnections`
- Artifactory DB pool metrics such as active, idle, and max-active

Proxy-side:

- sidecar CPU and memory usage
- restart count
- `cloudsqlconn/open_connections`
- `cloudsqlconn/dial_latency`
- `cloudsqlconn/dial_failure_count`
- `cloudsqlconn/refresh_success_count`
- `cloudsqlconn/refresh_failure_count`

Cloud SQL-side:

- `cloudsql.googleapis.com/database/postgresql/num_backends`
- `cloudsql.googleapis.com/database/cpu/utilization`
- `cloudsql.googleapis.com/database/memory/utilization`
- query wait and lock signals from Query Insights or PostgreSQL views

Correlation rules:

- high app wait + high proxy stress + normal Cloud SQL instance = proxy-side bottleneck is likely
- high app wait + high Cloud SQL stress across all primaries = instance-side bottleneck is more likely
- high app wait only on some primaries = compare proxy and node-level signals before touching Cloud SQL size

### What / How to Verify

Audit the live proxy definition on a pod:

```bash
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='
{range .spec.initContainers[?(@.name=="cloud-sql-proxy")]}
name={.name}{"\n"}image={.image}{"\n"}args={.args}{"\n"}resources={.resources}{"\n\n"}
{end}
{range .spec.containers[?(@.name=="cloud-sql-proxy")]}
name={.name}{"\n"}image={.image}{"\n"}args={.args}{"\n"}resources={.resources}{"\n"}
{end}'
```

Compare primaries:

```bash
kubectl top pod -n NAMESPACE --containers | rg 'artifactory|cloud-sql-proxy'
```

Read proxy logs:

```bash
kubectl logs -n NAMESPACE POD_NAME -c cloud-sql-proxy --since=30m
```

### Potential Problems

- assuming Cloud SQL instance metrics tell the whole story
- assuming all primaries behave equally
- treating the proxy as a connection pooler with its own useful concurrency cap

### Decision Making Guide

- If only selected primaries are affected, compare proxy sidecar behavior by pod before changing Cloud SQL instance size.
- If proxy-side symptoms track node-local pressure, investigate node placement and shared DaemonSet overhead.
- If both proxy-side and Cloud SQL-side signals are stressed, solve the dominant bottleneck first rather than tuning both blindly.

## Sub-Topic 2: Resource Sizing and Scheduling of the Proxy Sidecar

### Purpose

Size the sidecar so it is not the local choke point for a healthy Cloud SQL instance.

### Problem Statement

A proxy sidecar can be CPU-throttled, memory-constrained, or repeatedly restarted even when the database instance still has headroom.

### Stability, Performance, and Functionality Impact

If the sidecar is undersized:

- dials take longer
- connection churn becomes more expensive
- pod-local instability appears first on busy primaries
- HA becomes uneven because not all primaries can drive the same useful throughput

### Concept / Explanation

Google's GKE guidance for the Cloud SQL Auth Proxy sidecar states:

- proxy memory use scales linearly with the number of active connections
- proxy CPU use scales linearly with the amount of IO between the database and the application
- resource requests and limits should be set and adjusted to application needs

That means there is no single universal CPU or memory setting for Artifactory.

The correct sizing depends on:

- active DB connections per primary
- metadata intensity of the workload
- amount of network IO between primary and Cloud SQL
- sidecar density and node pressure

### Functionality

Resource sizing determines whether the proxy can:

- keep up with ongoing DB traffic
- establish new connections without excessive delay
- refresh connection metadata and certificates reliably

### Metrics / Parameters

Key Kubernetes parameters:

- sidecar `resources.requests.cpu`
- sidecar `resources.requests.memory`
- sidecar `resources.limits.cpu`
- sidecar `resources.limits.memory`
- pod placement and node pool characteristics

Key runtime indicators:

- current sidecar CPU
- current sidecar memory
- restart count
- OOM events
- if available in your observability stack, CPU throttling signals
- proxy `open_connections`

Correlation rules:

- memory growth should broadly track active connections
- CPU growth should broadly track database IO and request intensity
- repeated restarts invalidate any higher-level DB tuning conclusion

### Resource Allocation Model Based on Artifactory DB Pools

The most useful starting point is to separate:

- configured connection envelope
- observed steady-state connection envelope
- observed connection churn

The proxy sidecar should not be sized from configuration alone.

Configuration gives the maximum theoretical concurrency the pod could try to sustain. Runtime telemetry shows whether the pod is actually memory-bound by many long-lived connections or CPU-bound by connection churn and traffic.

#### Step 1: Calculate the configured connection envelope for the pod

For one Artifactory primary pod:

```text
C_cfg(pod) = Σ maxOpenConnections(service_i)
```

Where `service_i` includes only the Artifactory microservices that:

- run in that same pod
- connect through that same `cloud-sql-proxy` sidecar

Do not add pools from other pods just because they connect to the same Cloud SQL instance.

In this repo, the relevant pool knobs are:

- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`
- `metadata.database.maxOpenConnections`
- optionally `mc.database.maxOpenConnections`
- optionally `mc.idgenerator.maxOpenConnections`

Example using the default values present in this repo:

- Artifactory: `80`
- Access: `80`
- Metadata: `80`

If all three run in the same pod and share the same proxy:

```text
C_cfg = 80 + 80 + 80 = 240
```

If only the main Artifactory service uses that sidecar in the primary pod:

```text
C_cfg = 80
```

This is the upper bound from configuration, not the expected runtime value.

#### Step 2: Measure the observed connection envelope

Use proxy telemetry to derive:

```text
C_open_avg  = avg(open_connections)
C_open_p95  = p95(open_connections)
C_open_peak = max(open_connections)
```

These values tell you how many backend connections stay open in practice.

Interpretation:

- `C_open_p95` is the best basis for `requests.memory`
- `C_open_peak` is the best basis for `limits.memory`
- `C_cfg` is only a safety ceiling and sanity check

#### Step 3: Interpret connection residency

The effect of "how long connections stay open" is best represented by steady-state occupancy rather than by a standalone timer.

Use:

```text
R_residency = C_open_p95 / C_cfg
```

Interpretation:

- high `R_residency` means many configured connections remain open for long periods
- low `R_residency` means the pool cap is large relative to the number of connections actually kept open

Operational inference:

- `R_residency >= 0.6` and stable usually indicates memory-driven sizing matters more
- `R_residency <= 0.3` with high CPU or dial latency usually indicates churn or traffic-driven sizing matters more

These thresholds are heuristics, not official Google limits.

#### Step 4: Interpret connection churn

Long-lived open connections stress proxy memory more than CPU.

Short-lived connections created frequently stress:

- proxy CPU
- dial latency
- refresh activity
- sometimes SQL Admin API interaction during larger-scale churn or rollout events

You usually infer churn from a combination of:

- high proxy CPU
- rising `cloudsqlconn/dial_latency`
- rising `cloudsqlconn/dial_failure_count`
- Cloud SQL `new_connection_count`
- low or moderate `open_connections` relative to CPU cost

This means two pods with the same `C_cfg` can need very different CPU:

- Pod A keeps 100 connections open steadily and does moderate IO
- Pod B keeps only 20 to 30 open, but creates and tears them down aggressively and drives more CPU in the proxy

#### Step 5: Convert observations into requests and limits

Because Google documents linear scaling but not a universal per-connection memory or CPU constant, the safest method is empirical sizing per workload family.

Define:

```text
M_base     = proxy memory when open_connections is very low
M_per_conn = (M2 - M1) / (C2 - C1)
```

Where:

- `M1`, `M2` are observed proxy memory values
- `C1`, `C2` are corresponding observed `open_connections` values

Then use:

```text
memory_request = (M_base + M_per_conn * C_open_p95)  * 1.20
memory_limit   = (M_base + M_per_conn * C_open_peak) * 1.30
```

For CPU, use usage directly rather than connection count alone:

```text
cpu_request = CPU_p95  * 1.20
cpu_limit   = CPU_peak * 1.20
```

If your environment shows strong throttling or bursty dial behavior, keep extra headroom above the simple formula.

The `20%` to `30%` headroom values above are operational guidance, not official vendor defaults.

#### Step 6: Practical rule of thumb

Use this sequence:

1. Calculate `C_cfg` from the Artifactory pool settings for services that share the pod-local proxy.
2. Measure `C_open_p95` and `C_open_peak` from proxy telemetry.
3. Size memory from observed open connections, not from `C_cfg` alone.
4. Size CPU from observed CPU and dial behavior, not from connection count alone.
5. If `open_connections` is low but CPU and `dial_latency` are high, treat the problem as churn-driven rather than memory-driven.
6. If `open_connections` is persistently high and memory tracks it upward, review memory or reduce app-side pool sizes.

#### What the formulas mean for Artifactory

For Artifactory as a platform:

- `maxOpenConnections` defines how much DB concurrency a service is allowed to create
- it does not guarantee that the service will keep all those connections open
- the longer connections stay open, the more the sidecar behaves like a memory-sensitive local gateway
- the more frequently connections are opened and closed, the more the sidecar behaves like a CPU-sensitive dialer

This is why you should read these together:

- Artifactory DB pool active and idle counts
- proxy `open_connections`
- proxy CPU and memory
- proxy dial telemetry
- Cloud SQL `num_backends` and `new_connection_count`

### What / How to Verify

Check the actual sidecar resources:

```bash
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='
{range .spec.containers[?(@.name=="cloud-sql-proxy")]}
requests.cpu={.resources.requests.cpu}{"\n"}
requests.memory={.resources.requests.memory}{"\n"}
limits.cpu={.resources.limits.cpu}{"\n"}
limits.memory={.resources.limits.memory}{"\n"}
{end}'
```

Check current usage:

```bash
kubectl top pod POD_NAME -n NAMESPACE --containers
```

Check restart and event history:

```bash
kubectl describe pod POD_NAME -n NAMESPACE
```

Check pod-local historical trends in GKE Observability or Metrics Explorer with:

- `kubernetes.io/container/cpu/core_usage_time`
- `kubernetes.io/container/memory/used_bytes`
- `kubernetes.io/container/cpu/request_cores`
- `kubernetes.io/container/memory/request_bytes`
- restart and OOM events from Kubernetes events and logs

### Potential Problems

- no requests set for the proxy
- low CPU request leading to chronic throttling under sustained metadata load
- memory limit too tight for peak active connection count
- tuning app pools upward without scaling sidecar resources

### Decision Making Guide

- If proxy CPU is repeatedly high on busy primaries and `dial_latency` rises, increase CPU request first and limit second.
- If proxy memory tracks connection growth and approaches limit, either raise memory or reduce app-side maximum open connections.
- If `C_cfg` is large but `C_open_p95` is much lower, do not size memory directly from the configured cap.
- If `C_open_p95 / C_cfg` stays high for long periods, size memory conservatively and review whether app-side pools are oversized.
- If `open_connections` remains moderate but CPU and dial costs are high, focus on churn reduction and CPU headroom rather than larger memory limits.
- If the proxy is unstable, stabilize its resources before changing Artifactory or Cloud SQL settings.
- If primaries are uneven because some nodes are busier, solve scheduling or node-pool pressure before increasing Cloud SQL size.

## Sub-Topic 3: Proxy Telemetry and Correlation with Artifactory and Cloud SQL

### Purpose

Collect direct proxy telemetry so bottlenecks are visible before they are inferred from failures.

### Problem Statement

Without proxy-native metrics, operators only see symptoms in Artifactory or Cloud SQL and are forced to guess where the bottleneck lives.

### Stability, Performance, and Functionality Impact

Poor observability causes:

- delayed diagnosis
- oversized infrastructure changes
- repeated tuning loops without confidence

### Concept / Explanation

The Cloud SQL Auth Proxy supports:

- Cloud Monitoring telemetry
- Cloud Trace telemetry
- Prometheus metrics
- localhost admin endpoints for diagnostics
- structured logs

For an always-on Artifactory deployment, the minimum useful baseline is:

- structured logs
- sidecar resource metrics
- proxy connection telemetry

### Functionality

Proxy telemetry tells you whether:

- open connections are climbing normally
- new dials are slow or failing
- refresh operations are succeeding
- problems are local to selected primaries

### Metrics / Parameters

Official proxy telemetry families:

- `cloudsqlconn/dial_latency`
- `cloudsqlconn/open_connections`
- `cloudsqlconn/dial_failure_count`
- `cloudsqlconn/refresh_success_count`
- `cloudsqlconn/refresh_failure_count`

Useful Artifactory correlation metrics:

- DB pool active, idle, max-active
- request latency by primary pod
- JVM CPU and heap on affected primaries

Useful Cloud SQL correlation metrics:

- `num_backends`
- instance CPU and memory utilization
- Query Insights latency and lock metrics

Interpretation patterns:

- high Artifactory DB wait + rising `dial_latency` + normal instance CPU = proxy or path bottleneck
- high `refresh_failure_count` = certificate refresh, IAM, Admin API, or network-path issue
- low or flat `open_connections` on a stressed primary can indicate the primary cannot create or sustain enough backend connections efficiently

### What / How to Verify

Recommended proxy flags:

- `--structured-logs`
- `--prometheus` if you have a safe scrape pattern for pod-local metrics
- `--telemetry-project=PROJECT_ID` if you want telemetry in Cloud Monitoring and Cloud Trace
- optionally `--telemetry-prefix` to control metric naming in Cloud Monitoring

Practical monitoring paths:

1. Kubernetes metrics for sidecar CPU, memory, restarts
2. proxy telemetry via Cloud Monitoring or Prometheus
3. Cloud Logging on the `cloud-sql-proxy` container
4. Artifactory DB pool metrics per pod
5. Cloud SQL instance metrics

Logs Explorer starting query:

```text
resource.type="k8s_container"
resource.labels.cluster_name="CLUSTER_NAME"
resource.labels.namespace_name="NAMESPACE"
resource.labels.container_name="cloud-sql-proxy"
```

Temporary local inspection via port-forward:

```bash
kubectl port-forward -n NAMESPACE pod/POD_NAME 9091:9091
```

Use that only when the admin server is enabled and only for controlled diagnostics.

### Potential Problems

- enabling no proxy telemetry and then inferring everything from Cloud SQL instance graphs
- enabling Prometheus metrics without a real pod-local scrape plan
- leaving debug endpoints or verbose diagnostics enabled permanently

### Decision Making Guide

- Prefer direct proxy telemetry over indirect inference whenever possible.
- If your monitoring stack cannot scrape pod-local Prometheus endpoints safely, prefer `--telemetry-project` and structured logs.
- Use debug and profiling flags temporarily for diagnosis, not as permanent baseline settings.

## Sub-Topic 4: Logs, Errors, and What They Usually Mean

### Purpose

Turn proxy logs into actionable failure categories.

### Problem Statement

Many proxy failures look like generic database errors from the application perspective.

### Stability, Performance, and Functionality Impact

If proxy-side errors are not categorized correctly:

- teams tune the wrong layer
- intermittent failures remain unresolved
- rollout risk increases during upgrades or scale events

### Concept / Explanation

Structured logs and startup checks make the proxy easier to operate.

Useful runtime flags:

- `--structured-logs`
- `--run-connection-test` for startup validation
- `--debug-logs` temporarily, when refresh or dial behavior is unclear

Google's proxy documentation also recommends:

- using the latest version and updating on a regular schedule
- pinning a specific container image tag

### Functionality

Logs help you distinguish:

- authentication or IAM failures
- network path failures
- instance reachability failures
- Admin API or refresh failures
- local startup misconfiguration

### Metrics / Parameters

Primary log categories to watch:

- startup success and listener binding
- connection test failures
- refresh failures
- dial failures
- quota or API-related errors

Correlation rules:

- dial failures without Cloud SQL server stress usually point to path, IAM, proxy health, or control-plane issues
- startup failures with `--run-connection-test` catch bad deployments earlier
- repeated refresh failures are more serious than isolated dial retries

### What / How to Verify

Read recent logs:

```bash
kubectl logs -n NAMESPACE POD_NAME -c cloud-sql-proxy --since=1h
```

If needed, stream logs during load:

```bash
kubectl logs -n NAMESPACE POD_NAME -c cloud-sql-proxy -f
```

Patterns worth separating:

- IAM or permission errors
- DNS or network reachability errors
- certificate refresh errors
- SQL Admin API quota-related errors
- readiness and startup failures

### Potential Problems

- running outdated proxy versions
- using floating tags instead of pinned versions
- enabling `--debug-logs` permanently and creating noisy logs with little long-term value

### Decision Making Guide

- If the logs show startup validation failures, fix configuration first and do not tune performance knobs yet.
- If the logs show quota-style or refresh-style failures, reduce connection churn and inspect control-plane usage before increasing Cloud SQL size.
- If only one proxy sidecar logs repeated failures, focus on pod-local and node-local conditions first.

## Sub-Topic 5: Network Path, Connectivity Mode, and Local Binding

### Purpose

Make sure the proxy is using the intended network path and only the intended local exposure.

### Problem Statement

A healthy sidecar can still behave poorly if it uses the wrong IP path, runs on a noisy node, or is exposed too broadly.

### Stability, Performance, and Functionality Impact

Bad network-path choices can create:

- added latency
- intermittent reachability failures
- unnecessary security exposure
- misleading performance comparisons between primaries

### Concept / Explanation

Relevant proxy flags include:

- `--private-ip`
- `--port=<DB_PORT>`
- default localhost binding behavior
- `--address` only if you intentionally need a non-local listener

Official proxy guidance warns against binding the proxy to an external interface without necessity.

### Functionality

Network-path configuration determines:

- whether traffic uses private or public connectivity
- whether the local listener is pod-local only
- whether selected primaries have different effective network conditions

### Metrics / Parameters

Important configuration and signals:

- `--private-ip` when the cluster and instance support it
- DB port alignment with Artifactory JDBC settings
- pod-to-instance network path
- node egress health and latency

Interpretation patterns:

- one node pool shows worse sidecar behavior than another -> inspect node and network conditions
- all primaries using public path with avoidable complexity -> evaluate private IP path
- local listener bound broadly -> security risk, not a performance optimization

### What / How to Verify

Inspect sidecar args:

```bash
kubectl get pod POD_NAME -n NAMESPACE -o jsonpath='
{range .spec.containers[?(@.name=="cloud-sql-proxy")]}{.args}{"\n"}{end}
{range .spec.initContainers[?(@.name=="cloud-sql-proxy")]}{.args}{"\n"}{end}'
```

Verify the Cloud SQL instance connection name:

```bash
gcloud sql instances describe INSTANCE_NAME \
  --project=PROJECT_ID \
  --format='value(connectionName)'
```

### Potential Problems

- proxy uses public path when private path is expected
- local listener port mismatch with application settings
- exposing the proxy listener outside the pod without a real need

### Decision Making Guide

- If private IP is available and intended, confirm the flag and path before diagnosing performance.
- If listener binding was broadened, revert to localhost unless there is a justified operational requirement.
- If only some primaries are slow, compare node and path conditions, not just database graphs.

## Sub-Topic 6: Config Knobs and Tuning Order

### Purpose

Tune the sidecar in the right order and avoid changing the wrong knobs first.

### Problem Statement

Operators often try to fix proxy-side bottlenecks by increasing Cloud SQL size or Artifactory DB pools first.

### Stability, Performance, and Functionality Impact

Wrong tuning order can:

- increase connection churn
- increase memory use on the proxy
- amplify unevenness across primaries
- move the bottleneck instead of removing it

### Concept / Explanation

The most relevant sidecar tuning knobs are usually:

- image version
- resource requests and limits
- `--structured-logs`
- `--private-ip`
- `--port`
- `--run-connection-test`
- `--telemetry-project`
- `--telemetry-prefix`
- `--prometheus`
- `--debug-logs` for temporary diagnostics
- `--debug` and `--admin-port` for short-lived deep diagnostics

Important operational point:

- In the official documentation cited here, the Cloud SQL Auth Proxy is documented as a secure gateway and telemetry source, not as a DB-pool-style component with a primary tuning knob for maximum open connections. Inference: open connection growth should be controlled mainly through Artifactory pool settings and workload behavior, not by expecting the proxy to solve connection management on its own.

For very large deployments with high connection churn or SQL Admin API quota pressure, the official proxy guidance recommends evaluating a connection pooler such as PgBouncer. Cloud SQL also has Managed Connection Pooling as a distinct product capability. These are architectural options, not first-line fixes for an undersized sidecar.

### Functionality

This section decides whether to:

- scale proxy resources
- improve telemetry
- improve startup validation
- change network path
- keep pools stable
- escalate to connection pooling architecture

### Metrics / Parameters

Tune against these groups together:

- sidecar resource metrics
- proxy telemetry metrics
- Artifactory DB pool utilization
- Cloud SQL backend and query metrics
- rollout stability and restart history

### What / How to Verify

Recommended tuning order:

1. Verify sidecar version, args, and resources on live pods.
2. Enable structured logs and one direct telemetry path.
3. Stabilize restarts and memory pressure first.
4. Fix CPU pressure or throttling if present.
5. Confirm intended network mode and local binding.
6. Only then review Artifactory pool sizes and concurrency.
7. If large-scale connection churn remains the main issue, evaluate pooling architecture separately.

Temporary deep-diagnostic flags:

- `--debug-logs`
- `--debug`
- `--admin-port`

Use them only for short diagnostic windows.

### Potential Problems

- increasing `artifactory.database.maxOpenConnections` first
- increasing Artifactory primaries first
- leaving verbose debugging enabled permanently
- assuming a larger Cloud SQL instance will fix localized proxy saturation
- using `--lazy-refresh` by default on always-on GKE primaries even though the documented target use case is CPU-throttled serverless environments

### Decision Making Guide

- If the main problem is sidecar CPU or memory, tune sidecar resources first.
- If the main problem is path or refresh failure, fix connectivity and control-plane issues first.
- If the main problem is app-side connection churn across many proxies, keep the sidecar healthy but move the architectural discussion toward connection management rather than only vertical scaling.

## Interdependencies with Other Topics

This topic is tightly coupled with:

- Cloud SQL instance sizing:
  - the instance can be healthy while the proxy is the local bottleneck
  - the reverse is also possible
- Artifactory app sizing:
  - more `maxOpenConnections` and more Tomcat threads increase proxy load
- GKE node sizing:
  - sidecar CPU, memory, and repeated per-pod overhead consume node headroom
- full-primary HA topology:
  - more primaries improve resilience but also multiply sidecars, connection demand, and operational variance

Practical dependency rules:

- do not read proxy metrics without Artifactory pool metrics
- do not read Cloud SQL instance metrics without proxy metrics when primaries are uneven
- do not scale primaries without accounting for sidecar tax and DB connection demand

## Decision Matrix

Use the following patterns:

- Pattern: Cloud SQL instance looks normal, one primary is slow, proxy CPU high on that pod.
  - Decision: tune or reschedule the proxy sidecar before changing Cloud SQL instance size.
- Pattern: all primaries are slow, proxy metrics are calm, Cloud SQL backends and query waits are high.
  - Decision: investigate Cloud SQL capacity or query behavior first.
- Pattern: proxy memory grows with active connections and restarts begin.
  - Decision: raise proxy memory and review Artifactory connection pool sizes.
- Pattern: `dial_latency` and `dial_failure_count` rise, but DB CPU is still moderate.
  - Decision: investigate proxy path, IAM, Admin API interaction, and connection churn before scaling DB.
- Pattern: logs show quota-style or refresh-style failures during scale events.
  - Decision: reduce connection churn, stage rollouts more carefully, and evaluate whether connection pooling architecture is needed.

## Official References

- Cloud SQL Auth Proxy for GKE:
  - <https://cloud.google.com/sql/docs/postgres/connect-kubernetes-engine>
- Cloud SQL Auth Proxy connection guide:
  - <https://docs.cloud.google.com/sql/docs/postgres/connect-auth-proxy>
- Cloud SQL Auth Proxy official repository and README:
  - <https://github.com/GoogleCloudPlatform/cloud-sql-proxy>
- Manage database connections:
  - <https://docs.cloud.google.com/sql/docs/postgres/manage-connections>
- Managed Connection Pooling overview:
  - <https://docs.cloud.google.com/sql/docs/postgres/managed-connection-pooling>
- Configure Managed Connection Pooling:
  - <https://docs.cloud.google.com/sql/docs/postgres/configure-mcp>
- Cloud SQL monitoring metrics:
  - <https://docs.cloud.google.com/sql/docs/postgres/admin-api/metrics>
- GKE observability metrics:
  - <https://docs.cloud.google.com/monitoring/api/metrics_kubernetes>
- `kubectl top` reference:
  - <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/>
- `kubectl logs` reference:
  - <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/>
- `kubectl describe` reference:
  - <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/>
- JFrog database considerations:
  - <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/database/>
- JFrog runtime platform considerations:
  - <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/runtime-platform/>
