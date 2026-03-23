# Xray Troubleshooting Checklist

This note is a practical troubleshooting guide for Xray event intake and indexing latency, especially for symptoms such as:

- artifact upload completes in Artifactory
- Xray logs `indexing message received for artifact ...` only minutes later

This is written for local correlation with the vendored Xray chart at [xray/Chart.yaml](/Users/lilstew/Downloads/research/xray/Chart.yaml) and the local log/metrics pipeline under [log-analytics-prometheus-master](/Users/lilstew/Downloads/research/log-analytics-prometheus-master).

## Scope

Useful simplified path:

```text
artifact upload
-> Artifactory request completes
-> Xray receives the event or request
-> RabbitMQ queues the work
-> indexer consumes the message
-> analysis and persist continue downstream
```

The goal is to identify which interval is actually slow:

- `t1 - t0`: Artifactory upload completed, but Xray does not appear to receive work yet
- `t2 - t1`: Xray has received work, but indexing does not start promptly
- `t3 - t2`: indexing started, but downstream processing remains slow

Suggested anchors:

- `t0`: upload completion in Artifactory
- `t1`: first Xray router/server receive signal
- `t2`: `indexing message received for artifact ...`
- `t3`: downstream analysis or persist completion if needed

## Relevant Chart Defaults

Important defaults in the imported chart:

- `xray.jfrogUrl` is mandatory in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L187) and rendered in [xray/files/system.yaml](/Users/lilstew/Downloads/research/xray/files/system.yaml#L15)
- `rabbitmq.enabled: true` and `rabbitmq.replicaCount: 1` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L485)
- `analysis.replicaCount: 1` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L952)
- `indexer.replicaCount: 1` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1398)
- `persist.replicaCount: 1` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1506)
- `server.replicaCount: 2` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1614)
- `splitXraytoSeparateDeployments.fullSplit: false` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1998)

Operational meaning:

- with `fullSplit: false`, service-level bottlenecks are less isolated
- with `indexer.replicaCount: 1` and autoscaling off, queue buildup can translate directly into delayed indexing start
- with a single RabbitMQ replica, queue durability exists, but queue throughput and maintenance headroom are more limited than a larger HA layout

## Logs To Collect

### Artifactory

Start with these files:

- `artifactory-request.log`
- `router-request.log`
- optionally `artifactory-service.log`
- optionally `artifactory-event.log`

These are present in the local logger list at [artifactoy-ha/ci/loggers-values.yaml](/Users/lilstew/Downloads/research/artifactoy-ha/ci/loggers-values.yaml#L28).

Parsing references:

- request log structure in [jfrog-log-parsing-standard-loki.md](/Users/lilstew/Downloads/research/docs/jfrog-log-parsing-standard-loki.md#L109)
- router request JSON fields in [jfrog-log-parsing-standard-loki.md](/Users/lilstew/Downloads/research/docs/jfrog-log-parsing-standard-loki.md#L207)
- Artifactory request tail in [fluent_metrics.conf.rt](/Users/lilstew/Downloads/research/log-analytics-prometheus-master/fluent_metrics.conf.rt#L337)
- Artifactory router request tail in [fluent_metrics.conf.rt](/Users/lilstew/Downloads/research/log-analytics-prometheus-master/fluent_metrics.conf.rt#L407)

Use these fields first:

- `timestamp`
- `trace_id`
- `request_url`
- `request_method`
- `return_status`
- `request_duration`
- `StartUTC`
- `Duration`
- `request_Uber-Trace-Id`

### Xray

Start with these files:

- `router-request.log`
- `router-service.log`
- `xray-request.log`
- `xray-server-service.log`
- `xray-indexer-service.log`
- optionally `xray-analysis-service.log`
- optionally `xray-persist-service.log`

These are listed in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L219) and tailed in [fluent_metrics.conf.xray](/Users/lilstew/Downloads/research/log-analytics-prometheus-master/fluent_metrics.conf.xray#L212).

Use these fields first:

- `timestamp`
- `trace_id`
- `request_url`
- `request_method`
- `return_status`
- `request_duration`
- service-log `message`

## Metrics To Collect

### RabbitMQ Queue Backlog

The most relevant queues for indexing are:

- `indexRegular`
- `indexRegularRetry-0`
- `indexExistsContent`

These appear in the chart at [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1455).

The local Grafana dashboard already expects:

- `queue_messages_total{service="xray-auth-metrics", queue_name!~".*(R|r)etry.*"}`
- `queue_messages_total{service="xray-auth-metrics"}`

References:

- [XrayMetrics.json](/Users/lilstew/Downloads/research/log-analytics-prometheus-master/grafana/XrayMetrics.json#L752)
- [XrayMetrics.json](/Users/lilstew/Downloads/research/log-analytics-prometheus-master/grafana/XrayMetrics.json#L859)

Important distinction:

- these are queue depth or message count signals
- KEDA thresholds in the chart use `mode: QueueLength` in [xray/templates/services/_common-config.tpl](/Users/lilstew/Downloads/research/xray/templates/services/_common-config.tpl#L521)
- the `value: "30"` entries are autoscaling thresholds, not a hard intake limit

### Xray Runtime Health

Useful Xray runtime metrics already normalized locally:

- `sys_cpu_ratio`
- `sys_memory_used_bytes`
- `sys_memory_free_bytes`
- `app_disk_free_bytes`
- `app_disk_used_bytes`
- `db_connection_pool_in_use_total`
- `db_connection_pool_idle_total`
- `db_connection_pool_max_open_total`

References:

- [fluent_metrics.conf.xray](/Users/lilstew/Downloads/research/log-analytics-prometheus-master/fluent_metrics.conf.xray#L63)
- [fluent_metrics.conf.xray](/Users/lilstew/Downloads/research/log-analytics-prometheus-master/fluent_metrics.conf.xray#L171)

These help answer:

- is Xray CPU-starved
- is Xray memory pressured
- is Xray disk nearly full
- is Xray DB pool saturated

### Pod Health

Collect these from Kubernetes at the same time window:

- pod restarts
- readiness failures
- liveness failures
- CPU throttling if available
- OOMKill history if available

Focus on:

- `xray`
- `xray-server`
- `xray-indexer`
- `xray-analysis`
- `xray-persist`
- `rabbitmq`

Which pods exist depends on whether `fullSplit` is enabled.

## Step-By-Step Correlation

### 1. Confirm Upload Completion In Artifactory

Find the request in `artifactory-request.log` first. Match by:

- artifact path
- checksum
- client
- trace ID if available

What to record:

- upload completion timestamp
- HTTP status
- request duration
- trace ID

Use `router-request.log` if you need both request start and request end:

- `StartUTC` is request start
- `time` is completion time
- `Duration` is in nanoseconds

Expected result:

- you can identify a stable `t0`
- the request is successful and not itself delayed by minutes

If `t0` is already slow:

- the problem is still on the Artifactory side
- do not jump to Xray first

### 2. Confirm Xray Router Or Server Receive

Look in Xray:

- `router-request.log`
- `xray-request.log`
- `router-service.log`
- `xray-server-service.log`

Try to correlate by:

- artifact name or path
- trace ID
- matching time window

Expected result:

- `t1` is the first Xray-side signal that the event or request was received

If `t1 - t0` is large:

- verify `xray.jfrogUrl` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L187)
- verify the rendered URL in [xray/files/system.yaml](/Users/lilstew/Downloads/research/xray/files/system.yaml#L15)
- check routing, DNS, proxy, TLS, or network issues between Artifactory and Xray
- check Xray router and server pod readiness and restarts

### 3. Check RabbitMQ Backlog

Once Xray receive is confirmed, inspect queue depth for:

- `indexRegular`
- `indexRegularRetry-0`
- `indexExistsContent`

Optional downstream queues:

- `analysisRegular`
- `persistRegular`

Expected result:

- backlog should either stay low or drain quickly after receive

Signs of trouble:

- queue depth rises after `t1`
- queue depth stays elevated for minutes
- retry queues grow alongside primary queues

Interpretation:

- high queue depth with slow drain means consumer throughput is insufficient
- low queue depth with no `t2` suggests the problem is earlier than consumer backlog or inside service availability

### 4. Confirm Indexer Consumption

Search `xray-indexer-service.log` for:

- `indexing message received for artifact`

That gives `t2`.

Also inspect:

- surrounding warnings or errors
- restarts or readiness flaps around the same time
- CPU, memory, disk, and DB pool pressure

If `t2 - t1` is large and queues are high:

- indexer throughput is the first suspect

If `t2 - t1` is large and queues are low:

- look for indexer availability issues
- look for router or server delays before handoff

### 5. Check Downstream Analysis And Persist

If indexing starts quickly but final results remain delayed, inspect:

- `xray-analysis-service.log`
- `xray-persist-service.log`
- queue depth for `analysisRegular` and `persistRegular`
- Xray DB pool usage

At this point, the bottleneck is no longer event intake. It is downstream processing turnover.

## Fast Interpretation Matrix

| Symptom | First place to look | Most likely interpretation |
| --- | --- | --- |
| Upload itself is slow | Artifactory request and router logs | not an Xray-first issue |
| `t1 - t0` is large | Xray router/server receive path | routing, network, URL, or Xray ingress issue |
| `t2 - t1` is large and queues are high | RabbitMQ plus indexer | backlog or insufficient consumer throughput |
| `t2 - t1` is large and queues are low | indexer health plus Xray server handoff | availability issue, not pure backlog |
| `t2` is quick but final scan result is slow | analysis and persist | downstream processing bottleneck |
| retry queues grow | RabbitMQ plus service logs | transient failures or poisoned work items |
| DB pool ratio is high in Xray | Xray DB metrics | Xray DB saturation is limiting throughput |

## First Checks Before Tuning

Before changing replica counts or autoscaling, verify:

1. `xray.jfrogUrl` is correct and reachable.
2. Xray pods are healthy and not restarting.
3. RabbitMQ queues show whether backlog is real.
4. `indexer`, `analysis`, and `persist` are not CPU, memory, disk, or DB bound.
5. Delay is not already present in the Artifactory upload path.

## Tuning Direction After Diagnosis

Only tune after locating the slow interval.

If the problem is pre-receive:

- fix routing, network, or endpoint configuration first

If the problem is queue backlog:

- review `indexer.replicaCount` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1398)
- review autoscaling queue thresholds in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1455)
- consider whether `fullSplit` should be enabled in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1998)

If the problem is downstream:

- review `analysis.replicaCount` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L952)
- review `persist.replicaCount` in [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1506)
- check Xray DB capacity and queue drain behavior

## Minimal Data Set For One Incident

For one problematic artifact, capture:

1. Artifact path, repository, and approximate upload time.
2. `t0` from Artifactory request or router logs.
3. `t1` from Xray router or server logs.
4. queue depth over the same time window for `indexRegular` and `indexRegularRetry-0`.
5. `t2` from `xray-indexer-service.log`.
6. pod restarts, readiness changes, CPU, memory, disk, and DB pool signals during the same interval.

That is usually enough to determine whether the delay is:

- before Xray receive
- inside queueing
- at indexer start
- or after indexing in downstream processing

## Related Notes

- [xray-indexing-latency-correlation.md](/Users/lilstew/Downloads/research/docs/xray-indexing-latency-correlation.md)
- [artifactory-observability-quick-check.md](/Users/lilstew/Downloads/research/docs/artifactory-observability-quick-check.md)
