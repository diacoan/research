# Remote Repository Monitoring: Low-Level Operational Design

## Purpose

This revised document keeps the implementation intent of [low-level-design-remote-repo-monitoring.md](./low-level-design-remote-repo-monitoring.md) but reorganizes it for operational use.

It defines:

- the minimum parser and event contract
- the aggregates required for useful remote-repository monitoring
- the signal-to-capacity correlations that matter during triage
- the guardrails required before alerting or remediation

Use it together with:

- [high-level-design-remote-repo-monitoring_revized.md](./high-level-design-remote-repo-monitoring_revized.md)
- [next-steps-remote-repo-monitoring_revized.md](./next-steps-remote-repo-monitoring_revized.md)
- [artifactory-observability-quick-check.md](./artifactory-observability-quick-check.md)

## Reading Guide

Use this document in the following order:

1. Validate that source logs and parser outputs are complete.
2. Confirm the canonical event model and normalization rules.
3. Check the quick-check table for missing or misleading signals.
4. Correlate remote signals with platform capacity controls only after data quality is trusted.

## Pipeline Scope

The initial implementation stack remains:

- `Promtail` for collection and parsing
- `Loki` for log storage and query-side aggregation
- `Grafana` for dashboards and alerting

Logical mapping:

- collector/parser -> `Promtail`
- event store and aggregate computation -> `Loki`
- evaluator -> `Grafana`
- remediation orchestrator -> external workflow

## Quick Check

| Component or signal | Required field or aggregate | Risk if wrong or missing | Recommended action |
| --- | --- | --- | --- |
| `artifactory-service.log` parser | `timestamp`, `repo_key` if present, normalized `error_type`, optional `remote_host` | Network failures cannot be grouped correctly; shared-host issues stay hidden | Normalize transport failures first and keep unattributed records separate |
| `artifactory-request-out.log` parser | `timestamp`, `repo_key`, `status_code`, `duration_ms`, optional size | Success/failure ratio and latency become unreliable | Treat missing `duration_ms` as parser-quality failure, not as zero latency |
| `repo_key` attribution | `attribution_state` | Bad automation candidates and misleading repo-level dashboards | Keep unattributed events visible but exclude them from repo actions |
| `remote_host` extraction | `remote_host` | Host-wide upstream incidents appear as unrelated per-repo noise | Group by host where possible |
| Transport error normalization | `error_type` | Alerts depend on free-text parsing and become unstable | Normalize to `timeout`, `dns`, `tls`, `connect_refused`, `reset`, `other_network` |
| `401`, `403`, `404` separation | `status_code` and `status_class` | Auth, policy, and fan-out signals are mixed together | Preserve `401`, `403`, and `404` as first-class buckets |
| Latency on tiny/small objects | `response_size_bytes`, `object_size_bucket`, latency aggregate | Queueing remains hidden until hard failures appear | Build dedicated tiny/small-object latency views |
| `success_total_24h` and `last_success_timestamp` | Freshness metrics by `repo_key` | Broken repos remain hard to distinguish from low-volume healthy repos | Use event time, not ingestion time |
| Rolling windows | `5m`, `1h`, `24h` views | Short spikes and persistent patterns cannot be separated | Keep all three windows and use them for different decisions |
| Parser quality metrics | counts of unattributed events and missing duration | Operators tune the platform based on bad telemetry | Publish parser-quality panels and gate automation on them |

## Source Log Contract

### `artifactory-service.log`

Use this stream for transport and network-style failures.

Minimum fields:

- `timestamp`
- `repo_key` if present
- raw message
- normalized `error_type`
- optional `remote_host`

Output record:

- `service_error_event`

Use it for:

- `timeout`
- `dns`
- `tls`
- `connect_refused`
- `reset`
- other transport failures

### `artifactory-request-out.log`

Use this stream for outbound HTTP behavior.

Minimum fields:

- `timestamp`
- `repo_key`
- `method`
- `status_code`
- `duration_ms`
- URL or `remote_host`
- `response_size_bytes` when available

Output record:

- `request_out_event`

Use it for:

- success rate
- `4xx` and `5xx`
- latency
- request volume
- tiny/small-object performance

## Canonical Event Model

Normalized event contract:

```text
event_id
source_type
ts
repo_key
event_type
status_code
status_class
error_type
remote_host
method
duration_ms
response_size_bytes
object_size_bucket
attribution_state
raw_message
```

Rules:

- `event_id` must be unique per ingested log line
- `source_type` must be `service_log` or `request_out_log`
- `event_type` must be `success`, `http_error`, or `network_error`
- `duration_ms` is mandatory for outbound HTTP events
- `attribution_state` must be `attributed` or `unattributed`
- unattributed events remain visible for platform-wide review but are excluded from repository actions

## Normalization Rules

### Transport Errors

Normalize errors to stable buckets:

- `connect timed out`, `read timed out` -> `timeout`
- `UnknownHost`, `no such host` -> `dns`
- `SSLException`, `handshake` -> `tls`
- `Connection refused` -> `connect_refused`
- `Connection reset` -> `reset`
- remaining transport failures -> `other_network`

Reason:

- alert rules should not depend on free-text matching at evaluation time

### HTTP Status Handling

Keep these as first-class buckets:

- `401`
- `403`
- `404`
- `other_4xx`
- `5xx`

Operational meaning:

- `401` -> credentials or token lifecycle
- `403` -> access policy or proxy path
- `404` -> lookup/fan-out signal, not a direct health verdict
- `5xx` -> upstream instability or transient shared dependency issue

### Object Size Buckets

Use:

- `tiny` -> `<= 256 KB`
- `small` -> `> 256 KB` and `<= 5 MB`
- `large` -> `> 5 MB`

Why it matters:

- long latency on `tiny` and `small` objects is a stronger queueing signal than long latency on large artifacts

## Aggregate Model

Primary key:

- `repo_key`

Recommended secondary dimensions:

- `repo_key + error_type`
- `repo_key + status_class`
- `repo_key + status_code`
- `repo_key + remote_host`
- `repo_key + package_type` when available

Required windows:

- `5m`
- `1h`
- `24h`

Recommended logical views:

- `repo_health_5m`
- `repo_health_1h`
- `repo_health_24h`
- `repo_health_by_error_type`
- `repo_health_by_status_code`
- `repo_health_by_remote_host`

## Core Derived Metrics

### Health and Traffic

- `requests_total{repo_key}`
- `success_total{repo_key}`
- `http_4xx_total{repo_key}`
- `http_401_total{repo_key}`
- `http_403_total{repo_key}`
- `http_404_total{repo_key}`
- `http_4xx_other_total{repo_key}`
- `http_5xx_total{repo_key}`
- `network_error_total{repo_key}`
- `network_error_total{repo_key,error_type}`
- `timeout_total{repo_key}`

### Freshness

- `last_success_timestamp{repo_key}`
- `last_error_timestamp{repo_key}`
- `success_count_24h{repo_key}`
- `error_count_24h{repo_key}`

### Performance

- `latency_ms_sum{repo_key}`
- `latency_ms_count{repo_key}`
- `latency_ms_p95{repo_key}`
- `tiny_object_latency_ms_p95{repo_key}`
- `small_object_latency_ms_p95{repo_key}`

## Capacity Correlation

Remote monitoring is only useful if it is correlated correctly with platform capacity.

Useful working model:

```text
active_requests = progressing_requests + waiting_requests
```

and

```text
request_lifetime
  = cpu_time + db_wait + access_wait + remote_wait + storage_wait
```

When `remote_wait` rises:

- request lifetime rises
- more Tomcat threads stay occupied
- DB, Access, and outbound pools see more overlap

The most relevant controls for that interpretation are:

- `artifactory.tomcat.connector.maxThreads`
- `artifactory.database.maxOpenConnections`
- `access.tomcat.connector.maxThreads`
- `-Dartifactory.access.client.max.connections`
- `-Dartifactory.http.client.max.total.connections`
- `-Dartifactory.http.client.max.connections.per.route`

Important rule:

- do not change these values based on remote-repository dashboards until parser quality and attribution quality are trusted

## Alert Inputs and Guardrails

Recommended first rule inputs:

- `timeout_total_5m`
- `network_error_total_5m`
- `success_total_24h`
- `requests_total_24h`
- `http_401_total_1h`
- `http_403_total_1h`
- `http_404_total_24h`
- `tiny_object_latency_ms_p95`

Recommended guardrails:

- require minimum volume thresholds
- exclude extremely low-volume repos from hard actions
- treat `404` separately from transport failures
- keep automation disabled until false positives are understood

## Data Quality Gates

Before any automation or semi-automatic triage:

- unattributed event rate must be known
- missing `duration_ms` rate must be visible
- `repo_key` extraction quality must be stable
- `remote_host` extraction must be good enough to identify shared failures
- dashboards must clearly separate platform-wide noise from repo-specific candidates

## References

Local repository references:

- [low-level-design-remote-repo-monitoring.md](./low-level-design-remote-repo-monitoring.md)
- [high-level-design-remote-repo-monitoring.md](./high-level-design-remote-repo-monitoring.md)
- [next-steps-remote-repo-monitoring.md](./next-steps-remote-repo-monitoring.md)
- [artifactory-observability-quick-check.md](./artifactory-observability-quick-check.md)
- [high-level-artifactory-capacity-interdependencies.md](./high-level-artifactory-capacity-interdependencies.md)

Relevant JFrog references:

- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/runtime-platform/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/high-availability/>
- <https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/>
