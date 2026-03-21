# GKE Cluster Resource Assessment Playbook

## Purpose

Provide a complete operational playbook for assessing resource usage, sizing risk, and future capacity needs in a GKE Standard cluster that hosts:

- Artifactory full-primary HA
- Xray
- Distribution
- Prometheus, Grafana, and exporters
- supporting platform services
- cluster-wide DaemonSets
- future `promtail` sidecars
- future NGINX egress proxy for `huggingface.co`

This playbook is designed to answer these questions:

- What is the real workload capacity of the current cluster?
- How much is formally reserved by `requests` and `limits`?
- How much is actually used in practice over `30d` and `90d`?
- How should unbounded containers be assessed when `limits` are missing?
- What node-level usage patterns indicate the right choice between `n4-highmem-32` and `n4-highmem-64`?

## Problem Statement

In this cluster shape, resource pressure can come from multiple layers at once:

- stateful JVM-heavy workloads
- sidecars such as `cloudsql-proxy`
- observability overhead
- node-level DaemonSet overhead
- future log-shipping sidecars
- network-heavy external proxying

If the assessment is based only on current `kubectl top` output, it misses:

- long-term peaks
- packing inefficiency
- lack of `limits`
- node pressure and evictions
- autoscaler failure patterns
- fixed per-node overhead

If the assessment is based only on manifests, it misses:

- actual usage
- bursting behavior
- unbounded memory consumption
- sidecar and DaemonSet tax
- cluster-wide operational noise

This playbook combines declarative configuration, current measurements, historical metrics, and log-based pressure signals into one decision framework.

## Stability, Performance, and Functionality Impact

Poor assessment quality leads directly to platform risk:

- Stability impact:
  - OOM kills
  - node-pressure eviction
  - failed scheduling
  - autoscaler saturation
- Performance impact:
  - latency spikes
  - CPU throttling
  - degraded JVM behavior
  - noisy-neighbor effects
- Functionality impact:
  - failed deployments
  - unstable stateful services
  - incomplete observability during incidents
  - inability to onboard future consumers safely

## Target Topology Under Assessment

The assessment target for this document is a GKE Standard cluster with:

- Artifactory full-primary HA:
  - `5` primary pods
  - `4` NGINX pods
  - `splitServicesToContainers`
  - `cloudsql-proxy` sidecars attached to each Artifactory primary pod
- Xray:
  - `4` `xray-server` pods
  - `4` RabbitMQ pods
- Distribution:
  - `1` pod
- Monitoring and observability:
  - Grafana
  - Prometheus server
  - `5` to `10` exporters
- Additional applications:
  - JetBrains License Vault
- Platform overhead:
  - increased DaemonSets in multiple namespaces for monitoring, auditing, and cluster operations
- Planned additions:
  - `promtail` sidecars on Artifactory and Xray pods
  - dedicated NGINX instance proxying Integrity and Curation traffic to `huggingface.co`

Candidate machine types:

- `n4-highmem-32`
- `n4-highmem-64`

Official machine-size reference:

- <https://docs.cloud.google.com/compute/docs/general-purpose-machines>

## Working Principles

Use the following model throughout the playbook:

- raw VM capacity is not the same as Kubernetes workload capacity
- `allocatable` is the effective upper bound for workload placement on a node
- `requests` are scheduling guarantees, not observed usage
- `limits` are workload caps only when they exist
- without `limits`, observed peak usage plus node-pressure signals matter more than manifest intent
- logs usually tell you why the cluster failed
- metrics usually tell you how close the cluster was to failing

## Assessment Data Sources

Use four source classes together:

1. Cluster inventory:
   - `gcloud container ...`
   - `kubectl get`
   - `kubectl describe`
2. Current usage:
   - `kubectl top`
3. Historical metrics:
   - GKE Observability tab
   - Metrics Explorer
   - Monitoring API
4. Pressure and failure evidence:
   - Logs Explorer
   - `gcloud logging read`

Important retention note:

- Google Cloud metrics are retained long enough for `30d` and `90d` analysis.
- Cloud Logging `_Default` bucket retention is `30` days unless customized.
- If you want `90d` log-based analysis for events such as `FailedScheduling` or `Evicted`, you need longer bucket retention or a dedicated sink.

Official references:

- metrics retention and latency:
  - <https://docs.cloud.google.com/monitoring/api/v3/latency-n-retention>
- logs retention:
  - <https://docs.cloud.google.com/logging/quotas>
  - <https://docs.cloud.google.com/logging/docs/store-log-entries>

## Sub-Topic 1: Capacity Inventory and Node Pool Baseline

### Purpose

Establish the installed compute baseline of the cluster and the structure of node pools before looking at usage.

### Problem Statement

Sizing discussions often start from machine type names instead of the actual cluster shape. That misses:

- per-node allocatable capacity
- autoscaling boundaries
- node pool asymmetry
- node image differences

### Stability, Performance, and Functionality Impact

If you misread the baseline:

- you can overestimate safe workload headroom
- you can miss node-pool fragmentation
- you can choose a machine size that looks correct on paper but fails in real packing behavior

### Concept / Explanation

Inventory answers:

- how many node pools exist
- how many nodes each pool has
- which machine types are in use
- whether autoscaling exists and what its boundaries are
- what the node-level allocatable capacity is

### Functionality

This step creates the static baseline against which all later measurements are interpreted.

### Metrics / Parameters

Key fields:

- cluster node pool count
- node pool `machineType`
- node pool autoscaling `min/max`
- node `capacity`
- node `allocatable`

Meaning:

- `capacity`: total node resources from the VM and kubelet view
- `allocatable`: workload-usable portion after system reservations

Correlation:

- `allocatable < capacity` by design in GKE
- all request, limit, and real-usage analysis must be compared to `allocatable`, not only to VM size

### What / How to Verify

Official commands:

```bash
gcloud container clusters get-credentials CLUSTER_NAME \
  --region=REGION \
  --project=PROJECT_ID

gcloud container clusters describe CLUSTER_NAME \
  --region=REGION \
  --project=PROJECT_ID \
  --format="yaml(name,location,currentNodeCount,nodePools)"

gcloud container node-pools list \
  --cluster=CLUSTER_NAME \
  --region=REGION \
  --project=PROJECT_ID

gcloud container node-pools describe NODEPOOL_NAME \
  --cluster=CLUSTER_NAME \
  --region=REGION \
  --project=PROJECT_ID \
  --format="yaml(name,config.machineType,autoscaling,management)"

kubectl get nodes -L cloud.google.com/gke-nodepool \
  -o custom-columns=NODE:.metadata.name,POOL:.metadata.labels.cloud\\.google\\.com/gke-nodepool,CPU_ALLOC:.status.allocatable.cpu,MEM_ALLOC:.status.allocatable.memory,PODS_ALLOC:.status.allocatable.pods
```

Validation note:

- `gcloud` commands above are official Google Cloud CLI commands.
- `kubectl get ... -o custom-columns` is official `kubectl get` output formatting with local presentation customization.

### Potential Problems

- assuming `n4-highmem-32` means all `256 GB` are available to workloads
- comparing node pools only by machine type and ignoring autoscaling bounds
- forgetting that DaemonSet overhead scales with node count

### Decision Making Guide

- If pools are heterogeneous, analyze them separately.
- If autoscaling max is low, treat that as a hard expansion constraint.
- If `allocatable` is significantly lower than expected after adding DaemonSets, factor that into node-size decisions immediately.

### Official References

- `gcloud container` command group:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/container>
- `gcloud container clusters get-credentials`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/container/clusters/get-credentials>
- `gcloud container clusters describe`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/container/clusters/describe>
- `gcloud container node-pools describe`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/container/node-pools/describe>
- node sizing and allocatable:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/concepts/plan-node-sizes>

## Sub-Topic 2: Allocatable vs Capacity vs Requests vs Limits

### Purpose

Separate what the cluster can theoretically host from what workloads have reserved and from what they may actually consume.

### Problem Statement

Most capacity misunderstandings come from mixing:

- VM size
- node allocatable
- Pod `requests`
- Pod `limits`
- real usage

### Stability, Performance, and Functionality Impact

Confusing these layers leads to:

- oversubscription without noticing it
- false confidence in stability
- incorrect node-size choice
- wrong interpretation of cluster autoscaler behavior

### Concept / Explanation

Definitions:

- `capacity`: full node resource total
- `allocatable`: node resource available for Pods
- `requests`: scheduler reservation and minimum guaranteed share
- `limits`: cap enforced for a resource when configured
- observed usage: real runtime consumption

### Functionality

This section tells you which number is valid for which decision:

- scheduling -> `requests` vs `allocatable`
- hard capping -> `limits` when present
- performance -> observed usage
- node-risk assessment -> observed usage plus pressure signals

### Metrics / Parameters

Relevant metrics and fields:

- `.status.capacity`
- `.status.allocatable`
- container `resources.requests.cpu`
- container `resources.requests.memory`
- container `resources.limits.cpu`
- container `resources.limits.memory`
- `kubernetes.io/container/cpu/request_cores`
- `kubernetes.io/container/cpu/limit_cores`
- `kubernetes.io/container/memory/request_bytes`
- `kubernetes.io/container/memory/limit_bytes`

Meaning:

- request metrics reflect declared resource requests
- limit metrics reflect declared caps
- if no limit exists, the corresponding limit metric may be absent or not useful for that container

Correlation:

- `sum(requests)` drives placement and bin-packing
- `sum(limits)` is not the same as peak real usage
- no memory limit means the node, not the container, often becomes the effective safety boundary

### What / How to Verify

Current per-node scheduled allocation:

```bash
for n in $(kubectl get nodes -o name); do
  echo "### $n"
  kubectl describe "$n" | sed -n '/Allocated resources:/,/Events:/p'
done
```

List containers missing requests or limits:

```bash
kubectl get pods -A -o json | jq -r '
.items[] as $p
| ($p.spec.containers[]?, $p.spec.initContainers[]?) as $c
| select(
    ($c.resources.requests.cpu|not) or
    ($c.resources.requests.memory|not) or
    ($c.resources.limits.cpu|not) or
    ($c.resources.limits.memory|not)
  )
| [$p.metadata.namespace,$p.metadata.name,$c.name,
   ($c.resources.requests.cpu // "-"),
   ($c.resources.requests.memory // "-"),
   ($c.resources.limits.cpu // "-"),
   ($c.resources.limits.memory // "-")] | @tsv'
```

Historical declarative metrics in Cloud Monitoring:

- `kubernetes.io/container/cpu/request_cores`
- `kubernetes.io/container/cpu/limit_cores`
- `kubernetes.io/container/memory/request_bytes`
- `kubernetes.io/container/memory/limit_bytes`

### Potential Problems

- assuming all workloads have usable `limit` data
- using `sum(limits)` as a proxy for real peak
- ignoring node pressure when limits are missing

### Decision Making Guide

- If limits exist and are sane, use them as declared caps.
- If limits are missing, use observed `max/p95` usage plus eviction/OOM evidence.
- If requests are far below observed steady-state usage, expect noisy-neighbor and packing instability.
- If requests are far above observed steady-state usage, expect poor node utilization and possible oversizing.

### Official References

- node allocatable:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/concepts/plan-node-sizes>
- pod bursting and missing limits in Standard:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/pod-bursting-gke>
- GKE metrics:
  - <https://docs.cloud.google.com/monitoring/api/metrics_kubernetes>

## Sub-Topic 3: Current Usage Snapshot

### Purpose

Measure what is happening right now across nodes, pods, and containers.

### Problem Statement

Current usage is useful for incident triage, but not sufficient for sizing by itself.

### Stability, Performance, and Functionality Impact

Ignoring current usage during an incident can hide:

- hot nodes
- runaway sidecars
- recent packing imbalance
- noisy DaemonSets

### Concept / Explanation

`kubectl top` provides near-real-time CPU and memory usage from the Metrics API. It is the fastest way to locate current hotspots.

### Functionality

Use it to answer:

- which nodes are currently hottest
- which pods or containers are driving usage
- whether sidecars are materially contributing to the current footprint

### Metrics / Parameters

Current runtime values:

- node CPU usage
- node memory usage
- pod CPU usage
- pod memory usage
- container CPU usage
- container memory usage

Correlation:

- high pod usage with low node headroom means immediate risk
- high sidecar usage can distort application-level conclusions
- in this cluster, `cloudsql-proxy`, future `promtail`, and exporters must be evaluated independently

### What / How to Verify

Official commands:

```bash
kubectl top node --sort-by=cpu
kubectl top node --sort-by=memory
kubectl top pod -A --containers
```

Recommended supporting inventory:

```bash
kubectl get pods -A \
  -o custom-columns=NS:.metadata.namespace,POD:.metadata.name,NODE:.spec.nodeName \
  --no-headers | sort -k3,3
```

### Potential Problems

- treating one healthy low-traffic hour as representative
- looking only at pod totals and ignoring sidecars
- missing imbalanced scheduling across nodes

### Decision Making Guide

- If node usage is uneven, investigate scheduling and pool spread before resizing nodes.
- If sidecars are significant, budget them explicitly in capacity calculations.
- If current usage looks fine but historical peaks are high, trust historical analysis more for sizing.

### Official References

- `kubectl top`:
  - <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/>
- `kubectl top node`:
  - <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/kubectl_top_node/>
- `kubectl top pod`:
  - <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/kubectl_top_pod/>

## Sub-Topic 4: Historical Usage Patterns over 30d and 90d

### Purpose

Identify real cluster behavior over time instead of relying on current snapshots.

### Problem Statement

Sizing decisions based only on current usage miss:

- traffic bursts
- maintenance windows
- backup or indexing spikes
- release-day anomalies
- delayed autoscaler reactions

### Stability, Performance, and Functionality Impact

Without long-range history:

- you can undersize node pools
- you can miss recurring high-water marks
- you can choose a machine type that fits daily median but fails on weekly or monthly spikes

### Concept / Explanation

Use Cloud Monitoring metrics over `30d` and `90d` windows to extract:

- `max`
- `p95`
- recurring spike patterns
- hot-node concentration

### Functionality

Historical metrics tell you:

- peak observed usage
- steady-state consumption
- burst amplitude
- whether pressure is global or localized

### Metrics / Parameters

Recommended node metrics:

- `kubernetes.io/node/cpu/allocatable_utilization`
- `kubernetes.io/node/memory/allocatable_utilization`
- `kubernetes.io/node/cpu/core_usage_time`
- `kubernetes.io/node/memory/used_bytes`

Recommended container metrics:

- `kubernetes.io/container/cpu/core_usage_time`
- `kubernetes.io/container/memory/used_bytes`
- `kubernetes.io/container/cpu/request_cores`
- `kubernetes.io/container/memory/request_bytes`
- `kubernetes.io/container/cpu/limit_cores`
- `kubernetes.io/container/memory/limit_bytes`

Meaning:

- allocatable utilization shows how much usable node capacity is consumed
- container used metrics show real application and sidecar behavior
- request and limit metrics show declared scheduling envelope

Correlation:

- high node allocatable utilization plus moderate requests often means bursting or underdeclared requests
- high requests with low actual usage often means waste and poor packing efficiency

### What / How to Verify

Console path:

- `Kubernetes Engine > Clusters > <cluster> > Observability`
- `Kubernetes Engine > Workloads > <workload> > Observability`
- `Monitoring > Metrics Explorer`

Recommended chart settings:

- time range: `Last 30 days`, then `Last 90 days`
- aligners:
  - CPU cumulative usage -> `rate`
  - memory usage -> `max`
  - allocatable utilization -> `max` and optionally `p95`

Monitoring API example:

```bash
TOKEN=$(gcloud auth print-access-token)
START=$(date -u -d '90 days ago' +%FT%TZ)
END=$(date -u +%FT%TZ)

curl -s -G \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode 'filter=metric.type="kubernetes.io/node/memory/allocatable_utilization" AND resource.type="k8s_node" AND resource.labels.cluster_name="CLUSTER_NAME"' \
  --data-urlencode "interval.startTime=$START" \
  --data-urlencode "interval.endTime=$END" \
  --data-urlencode 'aggregation.alignmentPeriod=3600s' \
  --data-urlencode 'aggregation.perSeriesAligner=ALIGN_MAX' \
  "https://monitoring.googleapis.com/v3/projects/PROJECT_ID/timeSeries"
```

Validation note:

- the API path and `timeSeries.list` method are official.
- the exact `curl` wrapper is an operational example over the official REST method.

### Potential Problems

- using only average values and missing spikes
- forgetting metric latency and downsampling effects
- ignoring that log retention and metric retention are different

### Decision Making Guide

- For machine-size selection, prefer `p95` plus peak-event review, not only mean usage.
- If `90d max` is much higher than `30d max`, recent history may be too optimistic.
- If the cluster shows rare but real high peaks, consider whether they should be absorbed by larger nodes, more nodes, or better autoscaling.

### Official References

- view observability metrics:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/view-observability-metrics>
- GKE system metrics:
  - <https://docs.cloud.google.com/monitoring/api/metrics_kubernetes>
- Monitoring API:
  - <https://docs.cloud.google.com/monitoring/api/v3>
  - <https://docs.cloud.google.com/monitoring/api/ref_v3/rest/v3/projects.timeSeries>
  - <https://docs.cloud.google.com/monitoring/api/ref_v3/rest/v3/organizations.timeSeries/list>
- metric retention and latency:
  - <https://docs.cloud.google.com/monitoring/api/v3/latency-n-retention>

## Sub-Topic 5: Unbounded Containers, Bursting, and Missing Limits

### Purpose

Handle the common case where not all containers have `resources.limits` configured.

### Problem Statement

If a container has no limit:

- there is no manifest-defined hard ceiling
- CPU can burst into available capacity
- memory can grow until node pressure or OOM/eviction occurs

### Stability, Performance, and Functionality Impact

This is one of the biggest hidden risk sources in Standard clusters:

- node-memory exhaustion
- unpredictable burst behavior
- unstable packing
- difficult sizing conclusions if only declarative configuration is used

### Concept / Explanation

Google documents that GKE Standard supports Pod bursting. If limits are unset, Pods can use unused node capacity.

Practical implication:

- without limits, the effective upper bound is often node allocatable capacity, not the manifest

### Functionality

This section tells you how to estimate risk when no declared hard cap exists.

### Metrics / Parameters

Key fields and metrics:

- container `requests`
- container `limits` when present
- `kubernetes.io/container/memory/used_bytes`
- `kubernetes.io/container/cpu/core_usage_time`
- node allocatable utilization
- OOM and eviction events

Correlation:

- no limit + high observed memory peaks + eviction evidence = real sizing risk
- low requests + no limits + high node allocatable utilization = packing risk

### What / How to Verify

Manifest-level gap detection:

```bash
kubectl get pods -A -o json | jq -r '
.items[] as $p
| ($p.spec.containers[]?) as $c
| select(($c.resources.limits.memory|not) or ($c.resources.limits.cpu|not))
| [$p.metadata.namespace,$p.metadata.name,$c.name] | @tsv'
```

Historical usage for unbounded containers:

- chart `kubernetes.io/container/memory/used_bytes` by `container_name`
- chart `kubernetes.io/container/cpu/core_usage_time` with `rate`

### Potential Problems

- pretending there is a valid cluster-wide “max usable” number for all workloads
- using CPU burst behavior as if it were memory-safe behavior
- ignoring that memory pressure tends to fail at node level, not at container level, when limits are absent

### Decision Making Guide

- For containers without memory limits, use observed `max` and node-pressure evidence.
- If unbounded containers belong to critical services, strongly prefer explicit limits or stronger workload isolation.
- If unbounded sidecars exist on all replicas, budget them as repeated per-pod tax, not incidental noise.

### Official References

- pod bursting in GKE:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/pod-bursting-gke>
- node sizing:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/concepts/plan-node-sizes>

## Sub-Topic 6: Node Pressure, OOM, Eviction, and Failed Scheduling

### Purpose

Capture evidence that the cluster has already encountered capacity stress.

### Problem Statement

Metrics can show high usage, but logs and events show actual failure modes.

### Stability, Performance, and Functionality Impact

These are direct service-risk indicators:

- `OOMKilled` -> container memory exhaustion
- `Evicted` -> node pressure and reclaim behavior
- `FailedScheduling` -> capacity or placement failure
- cluster autoscaler out-of-resources -> infrastructure expansion failure

### Concept / Explanation

Pressure events are the best proof that the current cluster shape is already too tight or badly packed for some workloads.

### Functionality

Use this section to answer:

- did workloads already fail due to resource pressure
- was the failure local to one pod or systemic to the cluster
- was autoscaling able to help

### Metrics / Parameters

Main signals:

- Kubernetes events
- Cloud Logging event entries
- cluster autoscaler visibility logs
- restart count trends

Correlation:

- `FailedScheduling` without high usage may indicate affinity, anti-affinity, PVC, or shape mismatch
- `Evicted` with high node memory utilization indicates real node pressure
- autoscaler `scale.up.error.out.of.resources` means cloud-side capacity or quota issues, not only Kubernetes packing

### What / How to Verify

Logs Explorer query for unschedulable Pods, directly aligned with Google guidance:

```text
logName="projects/PROJECT_ID/logs/events"
jsonPayload.source.component="default-scheduler"
jsonPayload.reason="FailedScheduling"
resource.labels.cluster_name="CLUSTER_NAME"
```

Logs Explorer query for autoscaler out-of-resources, directly aligned with Google guidance:

```text
resource.type="k8s_cluster"
log_id("container.googleapis.com/cluster-autoscaler-visibility")
resource.labels.cluster_name="CLUSTER_NAME"
jsonPayload.resultInfo.results.errorMsg.messageId="scale.up.error.out.of.resources"
```

`gcloud logging read` equivalent:

```bash
gcloud logging read --project PROJECT_ID --freshness=30d \
'logName="projects/PROJECT_ID/logs/events"
 jsonPayload.source.component="default-scheduler"
 jsonPayload.reason="FailedScheduling"
 resource.labels.cluster_name="CLUSTER_NAME"'
```

OOM and eviction review:

- Google documents dedicated OOM troubleshooting guidance.
- Query `events` and node logs for `OOMKilled`, `Evicted`, and memory-pressure evidence.

### Potential Problems

- looking only at metrics and missing historical failures
- assuming every scheduling failure means more nodes are required
- ignoring cloud quota and regional capacity signals from autoscaler logs

### Decision Making Guide

- If you see `FailedScheduling`, check placement constraints before resizing nodes.
- If you see `Evicted`, prioritize memory-risk analysis over raw CPU sizing.
- If autoscaler reports out-of-resources, review quotas, regional capacity, and node-pool strategy before changing workload requests.

### Official References

- cluster autoscaler scale-up troubleshooting:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/troubleshooting/cluster-autoscaler-scale-up>
- logging query language:
  - <https://cloud.google.com/logging/docs/view/logging-query-language>
- `gcloud logging read`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/logging/read>
- OOM troubleshooting:
  - <https://cloud.google.com/kubernetes-engine/docs/troubleshooting/oom-events>

## Sub-Topic 7: DaemonSet Tax and Sidecar Tax

### Purpose

Quantify recurring overhead that is multiplied by node count or replica count.

### Problem Statement

In this cluster, a large share of non-business overhead is structural:

- monitoring DaemonSets
- auditing DaemonSets
- operational DaemonSets
- `cloudsql-proxy` sidecars
- future `promtail` sidecars
- exporters

Ignoring this overhead is a common sizing error.

### Stability, Performance, and Functionality Impact

If fixed tax is underestimated:

- allocatable headroom is overestimated
- node memory pressure arrives earlier than expected
- scaling stateful services becomes less predictable

### Concept / Explanation

Use two categories:

- DaemonSet tax:
  - repeated once per node
- Sidecar tax:
  - repeated once per pod replica

### Functionality

This section answers:

- how much baseline resource is consumed before the main services even start doing useful work
- whether larger or smaller nodes reduce the percentage cost of repeated overhead

### Metrics / Parameters

Key measurements:

- container usage by DaemonSet
- container usage by sidecar
- node allocatable utilization
- node count per pool
- pod count per service

Correlation:

- more nodes -> more DaemonSet tax
- more replicas -> more sidecar tax
- large nodes reduce percentage DaemonSet tax but increase blast radius

### What / How to Verify

List DaemonSets:

```bash
kubectl get daemonsets -A
```

Current container-level view:

```bash
kubectl top pod -A --containers
```

Historical container metrics:

- `kubernetes.io/container/cpu/core_usage_time`
- `kubernetes.io/container/memory/used_bytes`

Recommended grouping:

- namespace
- workload type
- sidecar container name

### Potential Problems

- treating DaemonSets as negligible
- treating `cloudsql-proxy` as free because it is colocated
- adding `promtail` without budgeting per-pod memory and CPU overhead

### Decision Making Guide

- If DaemonSet tax is high, larger nodes may be more efficient.
- If sidecar tax is high and repeated across many replicas, total per-service expansion cost may dominate the sizing decision.
- If future `promtail` is planned, include its expected steady-state and peak footprint before finalizing machine size.

### Official References

- workload observability and dashboards:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/view-observability-metrics>
- GKE metrics:
  - <https://docs.cloud.google.com/monitoring/api/metrics_kubernetes>

## Sub-Topic 8: Application Group Assessment for This Cluster

### Purpose

Evaluate the main workload families according to their typical resource and failure profiles.

### Problem Statement

Not all workloads stress nodes in the same way:

- Artifactory is JVM-heavy and stateful
- Xray and RabbitMQ add memory and I/O pressure
- Prometheus adds memory and disk pressure
- exporters and sidecars add constant overhead
- future NGINX proxy adds network and connection pressure

### Stability, Performance, and Functionality Impact

If workload families are not separated conceptually:

- CPU and memory patterns are misread
- network-heavy components get sized like JVMs
- stateful workloads share nodes with noisy system components

### Concept / Explanation

Assess at least these families independently:

- Artifactory primaries
- Artifactory NGINX
- Artifactory `cloudsql-proxy`
- Xray servers
- RabbitMQ
- Distribution
- Prometheus
- Grafana
- exporters
- DaemonSets
- future `promtail`
- future Hugging Face proxy NGINX

### Functionality

This section helps assign each observed hotspot to the right workload family.

### Metrics / Parameters

Focus per family:

- JVM-heavy services:
  - CPU rate
  - memory used
  - restart count
- observability stack:
  - memory used
  - storage growth
- proxies and exporters:
  - CPU rate
  - network throughput
  - connection-oriented scaling patterns

Correlation:

- JVM and RabbitMQ pressure often bias the choice toward more memory headroom
- Prometheus pressure can become dominant during long retention or high-cardinality ingestion
- future NGINX egress proxy may increase network and TLS overhead rather than only CPU

### What / How to Verify

Use namespace and label filters in Metrics Explorer and Observability tabs for each family.

Current distribution:

```bash
kubectl get pods -A -o wide
kubectl top pod -A --containers
```

### Potential Problems

- grouping Artifactory app containers and sidecars into one undifferentiated usage bucket
- missing Prometheus memory pressure
- forgetting that future consumers alter the steady-state baseline before any business traffic growth

### Decision Making Guide

- If one family dominates memory, optimize placement and memory headroom first.
- If future additions are proxy/logging heavy, reassess network, CPU, and per-node repeated overhead.
- If observability itself is large, do not assume the business workloads are the only node-sizing driver.

## Sub-Topic 9: Metrics Explorer and GKE Observability Usage Guide

### Purpose

Define exactly how to verify the cluster using official Google Cloud monitoring surfaces.

### Problem Statement

Without a standard chart set, different operators compare different views and reach inconsistent sizing conclusions.

### Stability, Performance, and Functionality Impact

Inconsistent observability leads to:

- incorrect headroom calculations
- missed hot nodes
- no shared operational baseline

### Concept / Explanation

Use GKE Observability tabs for fast curated views and Metrics Explorer for precise custom charts.

### Functionality

Required standard chart pack:

- node CPU allocatable utilization
- node memory allocatable utilization
- node CPU rate
- node memory used bytes
- container CPU rate
- container memory used bytes
- request metrics
- limit metrics

### Metrics / Parameters

Recommended node charts:

- `kubernetes.io/node/cpu/allocatable_utilization`
- `kubernetes.io/node/memory/allocatable_utilization`
- `kubernetes.io/node/cpu/core_usage_time`
- `kubernetes.io/node/memory/used_bytes`

Recommended container charts:

- `kubernetes.io/container/cpu/core_usage_time`
- `kubernetes.io/container/memory/used_bytes`
- `kubernetes.io/container/cpu/request_cores`
- `kubernetes.io/container/memory/request_bytes`
- `kubernetes.io/container/cpu/limit_cores`
- `kubernetes.io/container/memory/limit_bytes`

### What / How to Verify

Console workflow:

1. Open `Kubernetes Engine > Clusters`.
2. Select the cluster.
3. Open `Observability`.
4. Review the predefined dashboards for `30d` and `90d`.
5. Open `Monitoring > Metrics Explorer`.
6. Rebuild the standard chart pack with explicit aligners.

Recommended aligners:

- CPU cumulative metrics -> `rate`
- memory usage -> `max`
- utilization -> `max` and `p95`

### Potential Problems

- using `mean` where `max` is required for risk analysis
- comparing charts with different grouping or alignment
- forgetting that Google Cloud metrics can have visibility latency after sampling

### Decision Making Guide

- Use `max` for failure-risk and node-size safety checks.
- Use `p95` for typical high-load behavior.
- Use workload-specific grouping to separate business apps from DaemonSets and sidecars.

### Official References

- view observability metrics:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/view-observability-metrics>
- proactive monitoring:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/troubleshooting/introduction-monitoring>
- GKE metrics:
  - <https://docs.cloud.google.com/monitoring/api/metrics_kubernetes>
- Metrics Explorer:
  - <https://docs.cloud.google.com/monitoring/charts/metrics-explorer>

## Sub-Topic 10: Logs Explorer and Event Review

### Purpose

Use logs as the evidence layer for failures, pressure, and autoscaling constraints.

### Problem Statement

Metrics tell you "how much". Logs tell you "why it failed".

### Stability, Performance, and Functionality Impact

Without log review:

- a cluster can look merely busy when it is already unstable
- node-size decisions can be based on the wrong bottleneck

### Concept / Explanation

Review at least:

- scheduler failures
- autoscaler failures
- OOM and eviction indicators
- restart-heavy workloads

### Functionality

This section gives you the incident and pressure evidence required to justify node-size changes.

### Metrics / Parameters

Primary evidence:

- `FailedScheduling`
- autoscaler `scale.up.error.out.of.resources`
- OOM-related entries
- `Evicted` events

### What / How to Verify

Logs Explorer queries:

Unschedulable pods:

```text
logName="projects/PROJECT_ID/logs/events"
jsonPayload.source.component="default-scheduler"
jsonPayload.reason="FailedScheduling"
resource.labels.cluster_name="CLUSTER_NAME"
```

Autoscaler scale-up resource failure:

```text
resource.type="k8s_cluster"
log_id("container.googleapis.com/cluster-autoscaler-visibility")
resource.labels.cluster_name="CLUSTER_NAME"
jsonPayload.resultInfo.results.errorMsg.messageId="scale.up.error.out.of.resources"
```

Suggested operational query for evictions:

```text
logName="projects/PROJECT_ID/logs/events"
resource.labels.cluster_name="CLUSTER_NAME"
jsonPayload.reason="Evicted"
```

Suggested operational query for OOM investigation:

```text
resource.labels.cluster_name="CLUSTER_NAME"
("OOMKilled" OR "TaskOOM event" OR "ContainerDied")
```

CLI access:

```bash
gcloud logging read --project PROJECT_ID --freshness=30d 'LOG_QUERY_HERE'
```

### Potential Problems

- expecting `90d` log evidence when the `_Default` log bucket still retains only `30d`
- searching only one project when using cross-project monitoring
- reading autoscaler logs without considering regional cloud capacity or quota limits

### Decision Making Guide

- If logs show `FailedScheduling`, check whether the problem is resource shape, quota, affinity, PVC, or taints.
- If logs show frequent evictions, prioritize memory-risk reduction.
- If logs show autoscaler out-of-resources, validate quota and regional capacity before redesigning requests.

### Official References

- `gcloud logging read`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/logging/read>
- Cloud Logging CLI usage:
  - <https://docs.cloud.google.com/logging/docs/reference/tools/gcloud-logging>
- logging query language:
  - <https://cloud.google.com/logging/docs/view/logging-query-language>
- cluster autoscaler troubleshooting:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/troubleshooting/cluster-autoscaler-scale-up>
- logs retention:
  - <https://docs.cloud.google.com/logging/quotas>

## Sub-Topic 11: Decision Framework for `n4-highmem-32` vs `n4-highmem-64`

### Purpose

Translate the assessment into a node-size decision.

### Problem Statement

Machine-size discussions often get stuck in raw capacity numbers and ignore:

- per-node DaemonSet tax
- blast radius
- packing flexibility
- stateful workload placement
- unbounded memory consumers

### Stability, Performance, and Functionality Impact

Choosing the wrong size can cause:

- poor bin-packing
- higher eviction risk
- inefficient fixed overhead
- larger incident blast radius

### Concept / Explanation

Use these machine baselines from Google:

- `n4-highmem-32` -> `32` vCPU, `256 GB`
- `n4-highmem-64` -> `64` vCPU, `512 GB`

But size selection must be based on:

- node allocatable
- fixed per-node overhead
- observed peak memory
- scheduling behavior
- failure evidence

### Functionality

This section converts measurements into a recommendation path.

### Metrics / Parameters

Decision inputs:

- per-node `allocatable` CPU and memory
- node allocatable utilization `max` and `p95`
- DaemonSet tax per node
- sidecar tax per replica
- scheduling failures
- eviction history
- autoscaler evidence

Correlation:

- more nodes -> higher repeated DaemonSet tax, lower blast radius
- fewer larger nodes -> lower repeated DaemonSet tax percentage, higher blast radius
- memory-heavy stateful services often benefit from larger headroom, but only if placement and failure domains remain acceptable

### What / How to Verify

Build a worksheet with these columns:

- node pool
- machine type
- allocatable CPU
- allocatable memory
- current requested CPU
- current requested memory
- known limited CPU
- known limited memory
- observed peak CPU `30d`
- observed peak CPU `90d`
- observed peak memory `30d`
- observed peak memory `90d`
- count of containers without limits
- DaemonSet tax
- sidecar tax
- OOM / Evicted / FailedScheduling evidence

### Potential Problems

- choosing large nodes because averages look low
- choosing small nodes while ignoring high repeated per-node overhead
- ignoring that full-primary HA plus sidecars increases repeated memory tax

### Decision Making Guide

- Prefer `n4-highmem-32` when:
  - blast radius matters more
  - workloads pack reasonably on smaller nodes
  - node-level fragmentation is not the main constraint
  - autoscaling agility is more important than DaemonSet overhead efficiency

- Prefer `n4-highmem-64` when:
  - DaemonSet tax is significant
  - memory-heavy workloads need larger contiguous headroom
  - repeated sidecar and observability tax makes smaller nodes inefficient
  - scheduling failures show shape mismatch rather than raw cluster shortage

- Do not decide only from averages.

- If unbounded memory consumers exist on critical workloads, bias the decision toward stronger headroom or stronger isolation, not only toward nominal utilization efficiency.

### Official References

- machine families:
  - <https://docs.cloud.google.com/compute/docs/general-purpose-machines>
- GKE node sizing:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/concepts/plan-node-sizes>

## Interdependencies Between Sub-Topics

The sub-topics above are not independent.

Main dependency chain:

```text
machine type
-> node allocatable
-> scheduling headroom
-> requests packing
-> bursting behavior
-> real usage
-> node pressure
-> autoscaler response
-> observed stability
```

Important interdependencies:

- `allocatable` is the base for every later interpretation.
- `requests` explain placement but not real saturation.
- missing `limits` make historical usage and event evidence more important.
- DaemonSet tax depends on node count, so it changes with node-size strategy.
- sidecar tax depends on replica count, so it changes with service topology.
- historical peaks matter more when workloads are burstable or unbounded.
- logs explain whether usage actually became instability.

Specific examples for this cluster:

- more Artifactory primaries increase sidecar tax because each primary also carries a `cloudsql-proxy`
- adding `promtail` increases repeated sidecar overhead on both Artifactory and Xray
- more nodes increase the cost of cluster-wide monitoring and auditing DaemonSets
- a dedicated NGINX proxy for `huggingface.co` adds network and CPU pressure that may not correlate with JVM-heavy workload patterns

## Complete Execution Flow

Use the playbook in this order:

1. Inventory the cluster and node pools.
2. Record node allocatable values.
3. Capture current node and container usage.
4. Extract requests and limits coverage.
5. Identify containers without limits.
6. Build `30d` and `90d` historical usage charts.
7. Review `FailedScheduling`, `Evicted`, OOM, and autoscaler logs.
8. Quantify DaemonSet tax and sidecar tax.
9. Separate workload families and planned future additions.
10. Build the sizing worksheet.
11. Compare `n4-highmem-32` and `n4-highmem-64` against:
    - headroom
    - packing
    - blast radius
    - repeated overhead
    - historical peaks
    - pressure evidence
12. Make the node-size decision only after all six dimensions agree.

## Recommended Final Deliverables

At the end of the assessment, produce:

1. Node-pool inventory table
2. Requests/limits coverage report
3. `30d` and `90d` peak usage report
4. Pressure-event report
5. DaemonSet and sidecar overhead report
6. Final sizing recommendation with assumptions and residual risk

## Source Validation Notes

Validated directly against official documentation:

- `gcloud container clusters get-credentials`
- `gcloud container clusters describe`
- `gcloud container node-pools describe`
- `gcloud logging read`
- `kubectl top`
- GKE Observability tab workflow
- GKE system metrics names
- Cloud Monitoring `timeSeries.list`
- GKE node sizing / allocatable
- GKE Pod bursting behavior
- cluster autoscaler troubleshooting queries
- Cloud Logging retention constraints
- N4 machine family definitions

Operational examples included in this document:

- `jq`, `sed`, and `custom-columns` post-processing
- suggested Logs Explorer queries for `Evicted` and OOM investigation
- sample `curl` wrapper around the Monitoring API

These examples are built on top of official commands and APIs, but the formatting logic itself is local operator tooling.

## References

- GKE observability metrics:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/view-observability-metrics>
- GKE proactive monitoring:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/troubleshooting/introduction-monitoring>
- GKE metrics list:
  - <https://docs.cloud.google.com/monitoring/api/metrics_kubernetes>
- GKE node sizing:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/concepts/plan-node-sizes>
- GKE Pod bursting:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/pod-bursting-gke>
- GKE autoscaler troubleshooting:
  - <https://docs.cloud.google.com/kubernetes-engine/docs/troubleshooting/cluster-autoscaler-scale-up>
- GKE OOM troubleshooting:
  - <https://cloud.google.com/kubernetes-engine/docs/troubleshooting/oom-events>
- Cloud Monitoring API overview:
  - <https://docs.cloud.google.com/monitoring/api/v3>
- Cloud Monitoring `projects.timeSeries`:
  - <https://docs.cloud.google.com/monitoring/api/ref_v3/rest/v3/projects.timeSeries>
- Cloud Monitoring latency and retention:
  - <https://docs.cloud.google.com/monitoring/api/v3/latency-n-retention>
- Metrics Explorer:
  - <https://docs.cloud.google.com/monitoring/charts/metrics-explorer>
- Cloud Logging query language:
  - <https://cloud.google.com/logging/docs/view/logging-query-language>
- Cloud Logging CLI:
  - <https://docs.cloud.google.com/logging/docs/reference/tools/gcloud-logging>
- `gcloud logging read`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/logging/read>
- Cloud Logging retention:
  - <https://docs.cloud.google.com/logging/quotas>
  - <https://docs.cloud.google.com/logging/docs/store-log-entries>
- `gcloud container`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/container>
- `gcloud container clusters get-credentials`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/container/clusters/get-credentials>
- `gcloud container clusters describe`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/container/clusters/describe>
- `gcloud container node-pools describe`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/container/node-pools/describe>
- `kubectl top`:
  - <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/>
- `kubectl top node`:
  - <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/kubectl_top_node/>
- `kubectl top pod`:
  - <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_top/kubectl_top_pod/>
- Node status and `kubectl describe node`:
  - <https://kubernetes.io/docs/reference/node/node-status/>
- Compute Engine general-purpose machine families:
  - <https://docs.cloud.google.com/compute/docs/general-purpose-machines>
