# GKE Cluster Resource Assessment Worksheet Guide

## Purpose

Explain how to populate [gke-cluster-resource-assessment-worksheet.csv](./gke-cluster-resource-assessment-worksheet.csv) and how to make sizing and risk decisions from the collected data.

Use this guide together with:

- [gke-cluster-resource-assessment-playbook.md](./gke-cluster-resource-assessment-playbook.md)

## What the CSV Is For

The worksheet is a structured assessment surface for four different questions:

1. What capacity is installed in the cluster?
2. What capacity is formally reserved by workloads?
3. What capacity is actually used over time?
4. What failures or pressure signals prove that the cluster is too tight, badly packed, or missing guardrails?

The CSV is intentionally multi-level. It is not only a node sheet and not only a workload sheet.

It supports these row types:

- `cluster`
- `node_pool`
- `node`
- `namespace`
- `workload_family`
- `sidecar`
- `daemonset`
- `future_consumer`
- `decision_summary`

## Recommended Population Order

Populate the file in this order:

1. Fill one `cluster` row.
2. Fill one row per `node_pool`.
3. Fill one row per critical `node`.
4. Fill one row per major `namespace`.
5. Fill one row per `workload_family`.
6. Fill one row per repeated `sidecar`.
7. Fill one row per `daemonset`.
8. Fill one row per planned `future_consumer`.
9. Fill one or more `decision_summary` rows.

Why this order matters:

- the `cluster` and `node_pool` rows define the supply side
- the workload, sidecar, and daemonset rows define the demand side
- the `decision_summary` row is where the final conclusion is written

## How to Read a Row

Each row represents one assessment scope.

Examples:

- one `node_pool` row = summary for that whole node pool
- one `workload_family` row = summary for all Artifactory primaries, or all Xray servers, or all Prometheus components
- one `sidecar` row = summary for all `cloudsql-proxy` containers or all future `promtail` sidecars
- one `daemonset` row = summary for one DaemonSet repeated on every node

Do not try to fill every column for every row type. Some columns are intentionally left blank when they do not apply.

## Column Groups

The header is large, but it breaks into stable groups.

### 1. Identity and Scope

Columns:

- `section`
- `scope`
- `scope_type`
- `cluster_name`
- `region`
- `node_pool`
- `node_name`
- `namespace`
- `workload_name`
- `workload_type`
- `container_name`

What they mean:

- `section`: worksheet row category such as `node_pool` or `workload_family`
- `scope`: human-readable grouping label
- `scope_type`: concrete object class such as `cluster`, `node_pool`, `container`, `daemonset`, `planned_workload`
- `cluster_name`, `region`: cluster identity
- `node_pool`, `node_name`: infrastructure scope
- `namespace`, `workload_name`, `workload_type`, `container_name`: workload scope

How to fill:

- Always fill `section`, `scope`, and `scope_type`.
- Fill only the identifiers that apply to that row.

Example:

- for a `daemonset` row, `container_name` may be filled but `node_name` should usually be blank
- for a `node` row, `node_name` is filled and `workload_name` is usually blank

### 2. Node Shape and Supply-Side Capacity

Columns:

- `machine_type`
- `node_count_current`
- `node_count_min`
- `node_count_max`
- `cpu_capacity_cores_raw`
- `memory_capacity_gib_raw`
- `cpu_allocatable_cores`
- `memory_allocatable_gib`
- `pods_allocatable`

What they mean:

- `machine_type`: for example `n4-highmem-32`
- `node_count_current`: current node count in the pool
- `node_count_min`, `node_count_max`: autoscaling boundaries
- `cpu_capacity_cores_raw`, `memory_capacity_gib_raw`: VM size before kube/system reservation
- `cpu_allocatable_cores`, `memory_allocatable_gib`: workload-usable capacity
- `pods_allocatable`: max Pods schedulable on a node

How to fill:

- Use `gcloud container node-pools describe` for `machine_type` and autoscaling bounds.
- Use `kubectl get nodes` or `kubectl describe node` for allocatable.
- Use one representative value per node pool if all nodes in the pool are homogeneous.

Decision use:

- this group defines the real supply envelope
- all later demand and usage numbers are compared against `allocatable`, not raw VM size

### 3. Declared Scheduling Envelope

Columns:

- `requests_cpu_cores`
- `requests_memory_gib`
- `known_limits_cpu_cores`
- `known_limits_memory_gib`
- `containers_without_cpu_request`
- `containers_without_memory_request`
- `containers_without_cpu_limit`
- `containers_without_memory_limit`

What they mean:

- `requests_*`: sum of declared requests in the assessment scope
- `known_limits_*`: sum of declared limits where they exist
- `containers_without_*`: count of containers missing those declarations

How to fill:

- Use `kubectl describe node` for per-node allocated resource summaries.
- Use `kubectl get pods -A -o json` plus local parsing for missing requests/limits counts.
- For workload or sidecar rows, aggregate at workload-family level.

Decision use:

- high requests vs allocatable = scheduling pressure
- high counts without limits = weaker predictability
- high counts without requests = scheduler sees less demand than the workload may actually use

### 4. Current Runtime Usage

Columns:

- `current_cpu_usage_cores`
- `current_memory_usage_gib`

What they mean:

- snapshot runtime usage at the time of assessment

How to fill:

- Use `kubectl top node`
- Use `kubectl top pod -A --containers`
- For workload-family rows, aggregate the current usage of the relevant replicas

Decision use:

- useful for hotspot identification
- not sufficient for final sizing decisions on its own

### 5. Historical Usage and Peak Profile

Columns:

- `peak_cpu_30d_cores`
- `peak_cpu_90d_cores`
- `peak_memory_30d_gib`
- `peak_memory_90d_gib`
- `p95_cpu_30d_cores`
- `p95_cpu_90d_cores`
- `p95_memory_30d_gib`
- `p95_memory_90d_gib`
- `node_cpu_allocatable_util_max_30d`
- `node_cpu_allocatable_util_max_90d`
- `node_memory_allocatable_util_max_30d`
- `node_memory_allocatable_util_max_90d`

What they mean:

- `peak_*`: highest observed value in the time window
- `p95_*`: typical high-load value, less spike-sensitive than `max`
- `node_*_allocatable_util_*`: percentage of allocatable consumed at node level

How to fill:

- Use GKE Observability and Metrics Explorer.
- Use `max` aligner for memory and utilization.
- Use `rate` for CPU cumulative metrics, then extract `max` and `p95`.

Decision use:

- `peak` supports safety and incident-risk decisions
- `p95` supports steady-state sizing
- allocatable utilization shows whether risk exists even when requests look low

### 6. Pressure and Failure Signals

Columns:

- `oom_events_30d`
- `oom_events_90d`
- `evicted_events_30d`
- `evicted_events_90d`
- `failed_scheduling_events_30d`
- `failed_scheduling_events_90d`
- `autoscaler_out_of_resources_30d`
- `autoscaler_out_of_resources_90d`

What they mean:

- count of hard evidence events, not just high usage

How to fill:

- Use Logs Explorer or `gcloud logging read`
- Use `30d` for default log retention unless you have extended retention
- Use `90d` only when the log bucket or sink retains that period

Decision use:

- if these are non-zero, the cluster has already expressed real pressure
- these columns carry more weight than averages

### 7. Structural Overhead and Future Deltas

Columns:

- `daemonset_tax_cpu_cores_current`
- `daemonset_tax_memory_gib_current`
- `sidecar_tax_cpu_cores_current`
- `sidecar_tax_memory_gib_current`
- `future_delta_cpu_cores`
- `future_delta_memory_gib`

What they mean:

- `daemonset_tax_*`: repeated cost per node from DaemonSets
- `sidecar_tax_*`: repeated cost per replica from sidecars
- `future_delta_*`: expected new load not yet in production

How to fill:

- Use `kubectl top pod -A --containers` for current baseline
- Use historical metrics where already deployed
- For planned workloads such as future `promtail` or future Hugging Face proxy NGINX, fill estimated demand and label it clearly in `notes`

Decision use:

- these fields explain why raw node utilization can rise before business traffic rises
- they are critical for comparing larger nodes vs more nodes

### 8. Network, Ephemeral Storage, and Secondary Signals

Columns:

- `network_egress_peak_30d_mbps`
- `network_egress_peak_90d_mbps`
- `ephemeral_storage_peak_30d_gib`
- `ephemeral_storage_peak_90d_gib`
- `storage_pressure_signal`
- `restart_signal`

What they mean:

- egress peaks are useful for proxy, exporter, and remote-fetch workloads
- ephemeral storage peaks matter for Prometheus, logs, and temporary files
- `storage_pressure_signal` and `restart_signal` are qualitative summaries from the operator

How to fill:

- use Cloud Monitoring where metrics exist
- use `kubectl describe node`, workload observability, and logs when only indirect evidence exists
- for qualitative fields, use controlled labels such as `none`, `low`, `moderate`, `high`

Decision use:

- use these when CPU and memory alone do not explain instability
- very relevant for future NGINX proxying and observability-heavy nodes

### 9. Decision Columns

Columns:

- `observed_risk_level`
- `main_bottleneck_hypothesis`
- `decision_recommendation`
- `residual_risk`
- `notes`
- `data_source`
- `last_updated`

What they mean:

- `observed_risk_level`: operator classification such as `low`, `moderate`, `high`, `critical`
- `main_bottleneck_hypothesis`: short statement of the primary risk driver
- `decision_recommendation`: what should be done
- `residual_risk`: what remains risky even after the proposed action
- `notes`: free-form context
- `data_source`: for example `kubectl top`, `Metrics Explorer`, `Logs Explorer`
- `last_updated`: assessment date

How to fill:

- fill these only after technical columns are populated
- prefer short, explicit, testable statements

Example:

- `observed_risk_level=high`
- `main_bottleneck_hypothesis=node memory pressure amplified by unbounded sidecars`
- `decision_recommendation=prefer n4-highmem-64 or add stronger limits/isolation`

## Which Columns Apply to Which Row Type

### `cluster` row

Fill mainly:

- cluster identity
- current total node count
- high-level pressure signals
- aggregated historical peaks
- decision columns

### `node_pool` row

Fill mainly:

- machine type
- node counts
- allocatable
- requests / limits summary
- allocatable utilization peaks
- event evidence linked to that pool if available

### `node` row

Fill mainly:

- node identity
- allocatable
- current usage
- per-node allocated resources
- node-level pressure signals

### `namespace` row

Fill mainly:

- namespace
- request/limit coverage
- current usage
- historical peaks
- restart and pressure patterns

### `workload_family` row

Fill mainly:

- namespace
- workload name
- workload type
- request/limit coverage
- current usage
- historical peaks
- sidecar relationship if relevant

Recommended workload families for your cluster:

- Artifactory primaries
- Artifactory NGINX
- Artifactory `cloudsql-proxy`
- Xray server
- RabbitMQ
- Distribution
- Prometheus
- Grafana
- exporters
- JetBrains License Vault

### `sidecar` row

Fill mainly:

- container name
- repeated current usage
- repeated historical usage
- sidecar tax
- future delta if not yet deployed

Recommended sidecar rows for your cluster:

- `cloudsql-proxy`
- future `promtail`

### `daemonset` row

Fill mainly:

- daemonset identity
- current per-node tax
- historical tax
- notes about whether it scales with node count

### `future_consumer` row

Fill mainly:

- planned workload name
- estimated CPU and memory delta
- expected network/storage characteristics
- recommendation impact

Recommended future consumers:

- `promtail`
- `nginx-huggingface-proxy`

### `decision_summary` row

Fill mainly:

- final risk level
- final bottleneck hypothesis
- final recommendation
- residual risk
- source and date

This row is the executive summary for the assessment.

## How to Populate the CSV in Practice

### Step 1: Fill Supply-Side Rows

Start with:

- `cluster`
- `node_pool`
- critical `node`

These rows define:

- available capacity
- autoscaling envelope
- allocatable headroom

### Step 2: Fill Demand-Side Rows

Populate:

- `namespace`
- `workload_family`
- `sidecar`
- `daemonset`

These rows define:

- who consumes resources
- whether usage is repeated or incidental
- whether risk comes from the business workload or from platform overhead

### Step 3: Fill Historical and Event Evidence

Add:

- `peak_*`
- `p95_*`
- allocatable utilization peaks
- OOM / Evicted / FailedScheduling / autoscaler counts

This step separates “busy but healthy” from “already unstable”.

### Step 4: Fill Planned Changes

Populate:

- `future_consumer`
- `future_delta_*`

Do not leave planned additions outside the assessment. That would make the worksheet backward-looking only.

### Step 5: Write Summary and Recommendation

Finish with:

- `decision_summary`

Use one summary row per major scenario if needed:

- current state on `n4-highmem-32`
- current state on `n4-highmem-64`
- future state after adding `promtail`
- future state after adding Hugging Face proxy NGINX

## How Decisions Should Be Taken

Use the worksheet in this order:

1. Check `allocatable` and node count.
2. Check request pressure.
3. Check missing limits.
4. Check observed peaks and `p95`.
5. Check failure evidence.
6. Check repeated overhead.
7. Add future deltas.
8. Only then decide machine type or node-pool changes.

## Decision Rules

### Rule 1: Use `allocatable`, not raw machine size

If `cpu_capacity_cores_raw` and `memory_capacity_gib_raw` look comfortable but `cpu_allocatable_cores` and `memory_allocatable_gib` are materially lower, size the cluster based on `allocatable`.

### Rule 2: Requests drive placement, not real safety

If `requests_*` are low but `node_*_allocatable_util_max_*` is high, the cluster is probably relying on bursting or underdeclared workloads.

Decision implication:

- fix requests or isolate workloads before trusting node efficiency numbers

### Rule 3: Missing limits increase uncertainty

If `containers_without_memory_limit` is high:

- do not use declared limits as the main risk model
- use `peak_memory_*`, node utilization, OOM and eviction evidence instead

Decision implication:

- if critical services are unbounded, bias toward more headroom or stricter limits

### Rule 4: Peaks and `p95` must be read together

If `peak_memory_90d_gib` is much higher than `p95_memory_90d_gib`:

- the cluster has spike behavior
- decide whether to absorb spikes with node headroom, more nodes, or better autoscaling

Decision implication:

- do not size only for `p95` if spike failure is unacceptable

### Rule 5: Failure signals override optimistic averages

If any of these are non-zero:

- `oom_events_*`
- `evicted_events_*`
- `failed_scheduling_events_*`
- `autoscaler_out_of_resources_*`

then the cluster has already expressed instability or placement risk.

Decision implication:

- treat risk as real even if average utilization looks acceptable

### Rule 6: Repeated tax changes node-size economics

If `daemonset_tax_*` is high:

- larger nodes may be more efficient because the fixed per-node overhead is amortized across more allocatable capacity

If `sidecar_tax_*` is high:

- workload replica count may dominate more than node count

Decision implication:

- choose node size based on repeated overhead structure, not only on app usage

### Rule 7: Planned additions belong in the same model

If `future_delta_*` is not included:

- the worksheet is incomplete for sizing

Decision implication:

- include future `promtail` and future Hugging Face proxy before making a node-size recommendation

## Practical Decision Patterns

### Pattern A: Low requests, high allocatable utilization, many missing limits

Meaning:

- the cluster likely relies on burst behavior

Typical decision:

- increase observability of real peaks
- tighten requests/limits
- avoid aggressive consolidation onto smaller nodes

### Pattern B: High memory peaks, eviction evidence, moderate CPU

Meaning:

- memory is the primary risk, not CPU

Typical decision:

- favor more memory headroom
- consider stronger memory limits and isolation
- bias toward `n4-highmem-64` only if packing and blast radius still make sense

### Pattern C: High DaemonSet tax, low business workload density per node

Meaning:

- too much capacity is lost to per-node fixed overhead

Typical decision:

- larger nodes can be more efficient
- but validate blast radius and scheduling behavior first

### Pattern D: High sidecar tax on every Artifactory primary and Xray replica

Meaning:

- repeated per-replica overhead is material

Typical decision:

- budget sidecars as first-class workload components
- do not size only for the main app container

### Pattern E: `FailedScheduling` with moderate usage

Meaning:

- the issue may be shape, anti-affinity, taints, or PVC placement, not raw shortage

Typical decision:

- investigate placement constraints before changing node size

### Pattern F: Autoscaler out-of-resources

Meaning:

- Kubernetes wanted more nodes, but Google Cloud could not satisfy the request due to quota, stock, or shape limitations

Typical decision:

- review quotas, region, node-pool strategy, and machine-family fallback options

## How to Write the Final Recommendation

The `decision_summary` row should answer five things:

1. What is the dominant risk?
2. What evidence supports it?
3. What should change?
4. What risk remains after the change?
5. What assumption still needs validation?

Good example:

- `observed_risk_level=high`
- `main_bottleneck_hypothesis=node memory pressure amplified by unbounded sidecars and daemonset tax`
- `decision_recommendation=prefer n4-highmem-64 for primary pool or enforce limits before consolidating`
- `residual_risk=autoscaler out-of-resources still possible if regional capacity is tight`

Weak example:

- `decision_recommendation=need more resources`

The good version is actionable and testable. The weak version is not.

## Minimal Completion Standard

Do not consider the worksheet complete unless all are true:

- `cluster` row is filled
- every active `node_pool` has a row
- every critical workload family has a row
- repeated sidecars and DaemonSets are represented
- `30d` peak data exists
- pressure-event columns are reviewed
- planned future consumers are included
- at least one `decision_summary` row exists

## Recommended Next Step After Completion

Once the worksheet is filled:

1. duplicate the final `decision_summary` row into scenario variants
2. compare `n4-highmem-32` and `n4-highmem-64`
3. record assumptions explicitly
4. mark which conclusions are based on:
   - declarative config
   - observed metrics
   - log evidence
   - forecast for future consumers

## References

- [gke-cluster-resource-assessment-playbook.md](./gke-cluster-resource-assessment-playbook.md)
- [gke-cluster-resource-assessment-worksheet.csv](./gke-cluster-resource-assessment-worksheet.csv)
