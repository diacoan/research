# Low Level Design: Remote Repository Health Monitoring and Control

## Implementation Stack

The initial implementation stack is:

- `Promtail` for log collection and parsing
- `Loki` for storage and query-based aggregation
- `Grafana` for dashboards and alerting

Logical component mapping:

- collector/parser -> `Promtail`
- aggregator/query layer -> `Loki`
- evaluator -> `Grafana`
- remediation orchestrator -> external workflow, implemented later

## Pipeline Stages

1. `Promtail` ingests raw logs.
2. `Promtail` parses raw fields.
3. `Promtail` enriches events with extracted fields.
4. Events are normalized into the canonical schema.
5. Normalized logs are written to `Loki`.
6. `Loki` queries compute rolling aggregates.
7. `Grafana` evaluates alert rules and produces action candidates.

## Source Logs

### `artifactory-service.log`

Required extracted fields:

- timestamp
- logger or source file
- repository key if present
- error message
- optional upstream host

Output record type:

- `service_error_event`

If `repo_key` cannot be extracted, mark the event as `unattributed` and keep it out of repository-level automation.

### `artifactory-request-out.log`

Required extracted fields:

- timestamp
- repository key
- method
- URL or upstream host
- status code
- duration
- content length or response size if available

Output record type:

- `request_out_event`

Rules:

- records without `repo_key` may still be retained for platform-wide volume analysis, but are excluded from repository actions
- records without duration are invalid for this design and must be counted by a parser quality metric
- response size should be captured when available for small-object bucketing

## Normalized Event Model

Canonical event contract:

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

Field rules:

- `event_id` must be unique per ingested log line
- `source_type` must be `service_log` or `request_out_log`
- `event_type` must be `success`, `http_error`, or `network_error`
- `status_code` and `status_class` are nullable for network-only failures
- `error_type` is nullable for successful HTTP requests
- `duration_ms` is mandatory for outbound HTTP request events
- `response_size_bytes` is optional but strongly recommended for request-out events
- `object_size_bucket` must be derived when size is available
- `attribution_state` must be `attributed` or `unattributed`

## Error Normalization

Normalization mapping:

- `connect timed out`, `read timed out` -> `timeout`
- `UnknownHost`, `no such host` -> `dns`
- `SSLException`, `handshake` -> `tls`
- `Connection refused` -> `connect_refused`
- `Connection reset` -> `reset`
- other transport failures -> `other_network`

Normalization rules:

- if multiple patterns match, use the most specific category
- if no pattern matches, emit `other_network`
- keep the raw message for investigation

## HTTP Status Handling

`4xx` responses must be split at least as follows:

- `401`
- `403`
- `404`
- `other_4xx`

Interpretation:

- `401` -> expired or invalid credentials
- `403` -> access-policy issue or upstream proxy behavior
- `404` -> fan-out and virtual repository design signal
- `other_4xx` -> retained as a grouped bucket unless a future pattern justifies more splitting

## Small-Object Classification

Object size buckets:

- `tiny` object: response size `<= 256 KB`
- `small` object: response size `> 256 KB` and `<= 5 MB`
- `large` object: response size `> 5 MB`

Functional meaning:

- `tiny` and `small` objects represent metadata, descriptor, index, checksum, manifest, and other control-plane style artifacts
- examples include `pom.xml`, `maven-metadata.xml`, checksum files, package indexes, and similar lookup artifacts
- long request times for `tiny` and `small` objects are stronger indicators of queueing or connection contention than long times for large artifact downloads

## Aggregation Strategy

Primary key:

- `repo_key`

Secondary dimensions:

- `repo_key + error_type`
- `repo_key + status_class`
- `repo_key + status_code`
- `repo_key + remote_host`
- `repo_key + package_type` if repository metadata is available

Time windows:

- `5m`
- `1h`
- `24h`

Logical aggregate views:

- `repo_health_5m`
- `repo_health_1h`
- `repo_health_24h`

Each aggregate row is keyed by:

```text
window_start
window_end
repo_key
```

## Derived Metrics

### Traffic

- `requests_total{repo_key}`
- `success_total{repo_key}`
- `http_4xx_total{repo_key}`
- `http_4xx_total{repo_key,status_code}`
- `http_5xx_total{repo_key}`
- `http_401_total{repo_key}`
- `http_403_total{repo_key}`
- `http_404_total{repo_key}`
- `http_4xx_other_total{repo_key}`

Rules:

- `requests_total` counts all attributed outbound request events
- `success_total` includes `2xx` and `3xx`
- `http_4xx_other_total` excludes `401`, `403`, and `404`

### Network Failures

- `network_error_total{repo_key}`
- `network_error_total{repo_key,error_type}`
- `timeout_total{repo_key}`

Rule:

- `timeout_total` must be derived from `error_type=timeout`, not from raw string matching at alert time

### Freshness

- `last_success_timestamp{repo_key}`
- `last_error_timestamp{repo_key}`
- `success_count_24h{repo_key}`
- `error_count_24h{repo_key}`

Rule:

- freshness must use event time, not ingestion time

### Latency

- `latency_ms_sum{repo_key}`
- `latency_ms_count{repo_key}`
- `latency_ms_p95{repo_key}`
- `tiny_object_latency_ms_p95{repo_key}`
- `small_object_latency_ms_p95{repo_key}`

Rules:

- latency metrics must be computed from valid attributed outbound request events
- missing duration must never be treated as zero
- tiny/small object latency must be a first-class signal

## Capacity Correlation Model

There is no single exact formula for Artifactory throughput because request paths differ, but the system should be reasoned about as a constrained chain.

Working model:

```text
effective_throughput
  ~= min(
       tomcat_request_capacity,
       db_progress_capacity,
       access_progress_capacity,
       outbound_remote_progress_capacity
     )
```

Where:

- `tomcat_request_capacity` is shaped mainly by `artifactory.tomcat.connector.maxThreads`
- `db_progress_capacity` is shaped mainly by `artifactory.database.maxOpenConnections` and DB server health
- `access_progress_capacity` is shaped mainly by `access.tomcat.connector.maxThreads` and `Artifactory.access.client.max.connections`
- `outbound_remote_progress_capacity` is shaped mainly by `Artifactory.http.client.max.total.connections` and `Artifactory.http.client.max.connections.per.route`

Useful mental model:

```text
active_requests = progressing_requests + waiting_requests
```

and

```text
request_lifetime = cpu_time + db_wait + access_wait + remote_wait + storage_wait
```

If any wait component grows, request lifetime grows. When request lifetime grows, Tomcat threads stay occupied longer. That increases concurrency pressure on all other shared pools.

This creates a feedback loop:

```text
higher_wait_time
-> longer_request_lifetime
-> more_active_requests
-> more_pool_contention
-> even_higher_wait_time
```

For remote-heavy workloads, a useful approximation is:

```text
remote_pressure_index
  ~= request_rate
   * remote_lookup_ratio
   * avg_remote_attempts_per_request
   * avg_remote_wait_time
```

Interpretation:

- if `remote_lookup_ratio` rises, more requests depend on outbound HTTP pools
- if `avg_remote_attempts_per_request` rises, large virtual repositories create more fan-out
- if `avg_remote_wait_time` rises, threads are retained longer and platform concurrency pressure increases

## Relative Sizing Guidance

These are correlation rules, not hard vendor formulas.

- `artifactory.tomcat.connector.maxThreads` should not be increased in isolation.
- `artifactory.database.maxOpenConnections` should be validated whenever request concurrency rises materially.
- `access.tomcat.connector.maxThreads` and `Artifactory.access.client.max.connections` should be reviewed when increasing Artifactory request concurrency.
- `Artifactory.http.client.max.total.connections` and `Artifactory.http.client.max.connections.per.route` should be reviewed when the workload is remote-heavy.

Practical reading:

- high Tomcat threads with low outbound HTTP capacity usually creates more waiting, not more useful remote throughput
- high outbound HTTP capacity with unhealthy remotes may amplify wasted remote work
- high DB pool values without DB server headroom move contention into the database tier
- high Access capacity without enough Artifactory concurrency provides little benefit

## Symptom-Driven Adjustment Guide

### Symptom: high request latency, high active request count, low remote success rate

Likely bottleneck:

- outbound HTTP pools
- unhealthy or slow remote repositories

Check first:

- `Artifactory.http.client.max.total.connections`
- `Artifactory.http.client.max.connections.per.route`
- timeout rate
- tiny/small-object latency
- remote repo error rate

Adjustments:

- remove or isolate unhealthy remotes from large virtual repositories
- reduce fan-out before increasing connection pools
- increase outbound HTTP pool capacity only if remotes are healthy and upstream can sustain more load

### Symptom: high request latency, DB wait or connection exhaustion patterns, moderate remote latency

Likely bottleneck:

- DB pool or DB tier

Check first:

- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`
- DB server saturation
- long transaction or connection wait indicators

Adjustments:

- increase DB pool only if the database tier has headroom
- reduce request lifetime by addressing remote latency and retries
- avoid increasing Tomcat threads before DB pressure is understood

### Symptom: auth-related slowness, Access timeouts, permission-heavy requests slowing down

Likely bottleneck:

- Access pool or Access service concurrency

Check first:

- `access.tomcat.connector.maxThreads`
- `Artifactory.access.client.max.connections`
- Access response time and error rate

Adjustments:

- increase Access-side concurrency if Access is the clear choke point
- verify Access resources and DB capacity
- avoid increasing only Artifactory request threads if Access is already saturated

### Symptom: many active Tomcat threads, low CPU, high latency, backlog growth

Likely bottleneck:

- waiting on external dependencies rather than CPU

Check first:

- remote latency
- DB wait
- Access wait
- connector backlog
- small-object latency

Adjustments:

- do not treat this as a pure Tomcat scaling problem
- increasing `maxThreads` may only increase the number of blocked requests
- find the dominant wait source and relieve that first

### Symptom: high `401`

Likely bottleneck:

- credential validity, not capacity

Check first:

- repository credentials
- secret expiry or rotation status

Adjustments:

- fix credentials before tuning any pool or thread value

### Symptom: high `403`

Likely bottleneck:

- ACL or proxy path issue, not raw concurrency

Check first:

- upstream access policy
- proxy routing behavior
- repository configuration

Adjustments:

- fix access path first
- capacity tuning is usually the wrong first move

### Symptom: high `404` with useful success still present

Likely bottleneck:

- virtual repository fan-out inefficiency

Check first:

- `http_404_total`
- distinct team usage
- top consumers
- repository usefulness in the virtual repository

Adjustments:

- move low-value remotes out of large virtual repositories
- ask small consumer groups to call the remote directly when appropriate
- prioritize repository design cleanup before increasing remote connection capacity

### Symptom: high tiny/small-object latency with low transfer size

Likely bottleneck:

- connection contention
- queueing
- upstream slowness

Check first:

- `tiny_object_latency_ms_p95`
- `small_object_latency_ms_p95`
- outbound HTTP usage
- remote timeout rate

Adjustments:

- reduce fan-out and remove broken remotes
- if remotes are healthy but saturated, consider increasing outbound HTTP capacity carefully
- review per-route limits if many requests target the same upstream host or proxy

## Tuning Order of Operations

Recommended order:

1. validate symptom type
2. identify dominant wait source
3. remove invalid demand such as broken remotes, bad credentials, or poor virtual-repo design
4. verify downstream dependency headroom
5. only then increase concurrency-related settings

Anti-patterns:

- increasing `maxThreads` first when the actual issue is remote wait
- increasing outbound HTTP pools while keeping unhealthy remotes in the virtual path
- increasing DB pools without verifying DB server capacity
- treating all latency as CPU starvation

## Repository State Model

Repository states:

- `healthy`
- `degraded`
- `suspect`
- `fanout-heavy`
- `action-candidate`

Example transitions:

- repeated transport failures + no success -> `suspect`
- sustained high `404` + useful success still present -> `fanout-heavy`
- sustained failures + policy allows action -> `action-candidate`

## High 404 Analysis Record

For each repository with elevated `404`:

```text
repo_key
window
http_404_total
distinct_clients
distinct_teams
top_virtual_repos
top_user_agents
top_source_ips_or_tokens
recommended_action
```

Decision support rules:

- if `distinct_teams < X`, flag for targeted outreach
- if `distinct_teams >= X`, treat as shared behavior and prioritize virtual repo optimization
- if consumers cannot be mapped, hold in review state rather than acting automatically

## Alerting

Recommended rule inputs per `repo_key`:

- `timeout_total_5m`
- `network_error_total_5m`
- `success_total_24h`
- `requests_total_24h`
- `http_401_total_1h`
- `http_403_total_1h`
- `http_404_total_24h`
- `tiny_object_latency_ms_p95_5m`
- `small_object_latency_ms_p95_5m`

Rule output contract:

```text
alert_name
severity
repo_key
reason
supporting_metrics
recommended_runbook
```

Routing guidance:

- `401` -> credentials or secret ownership path
- `403` -> proxy, upstream ACL, or repository config path
- transport failures -> platform or network operations path
- high `404` -> repository design optimization path
- high tiny/small-object latency -> outbound connection contention, queueing, or upstream slowness path

Alert timing guidance:

- alerting is near-real-time, not strict real-time
- expected delay includes log write time, `Promtail` collection delay, `Loki` ingestion delay, query window duration, and `Grafana` evaluation interval

## Repository Summary Object

Preferred dashboard and automation summary:

```text
repo_key
window
requests_total
success_total
http_401_total
http_403_total
http_404_total
http_4xx_other_total
http_5xx_total
network_error_total
timeout_total
last_success_timestamp
last_error_timestamp
latency_ms_p95
tiny_object_latency_ms_p95
small_object_latency_ms_p95
state
```

## Automation Contract

Define an `action_candidate` object:

```text
repo_key
candidate_type
candidate_reason
source_alert
policy_class
last_success_timestamp
failure_window
requires_human_approval
proposed_change
```

Allowed initial `candidate_type` values:

- `set_offline_mode`
- `review_virtual_membership`
- `review_credentials`
- `review_proxy_or_acl`

Automation must consume only `action_candidate` objects, never raw log events.

## Operational Rollout

### Phase 1

- implement `Promtail` scrape and pipeline stages for the two Artifactory log files
- validate field extraction on sample logs
- publish normalized event schema
- build first `Loki` queries and `Grafana` dashboards

### Phase 2

- add repository state derivation
- deploy warning and critical rules
- compare alerts with known failures and false positives

### Phase 3

- add policy metadata store
- generate `action_candidate` records
- integrate PR creation workflow with review gates

## Risks and Mitigations

- variable log format -> version parser rules and keep parser tests
- low-volume noise -> minimum sample thresholds
- high `404` noise -> separate `404` analysis path
- missing attribution -> exclude unattributed events from automation
- unsafe automation -> require policy metadata and human approval by default

## Required Next Deliverables

- sample log corpus
- parser test cases
- first normalized schema draft
- first aggregate query set
- first alert threshold proposal
