# High Level Design: Remote Repository Health Monitoring and Control

## Purpose

Define a monitoring and control approach for unhealthy remote repositories in Artifactory, with emphasis on high-traffic virtual repositories used by Maven and Python workloads.

The goal is to reduce wasted outbound HTTP connections, reduce request latency amplification, and prevent degraded remote repositories from harming overall platform throughput.

### Low-Level Design

- Produce one versioned design artifact for the log-to-metric pipeline.
- Assign one canonical identifier for each remote repository: `repo_key`.
- Use a single normalized outbound event contract across all downstream stages.
- Keep parsing, aggregation, alerting, and remediation as separate logical components so they can evolve independently.

## Problem Statement

Large virtual repositories can fan out requests across many remote repositories. When some remotes are slow, broken, or unreachable, Artifactory keeps spending outbound HTTP connections and request-processing time on low-value remote attempts.

This is especially damaging for request-hungry ecosystems such as Maven and Python, where a single build can trigger a large number of metadata and artifact resolution calls.

Observed platform risks:

- Increased consumption of outbound HTTP connection pool capacity.
- Higher request latency for healthy repositories due to retries and fan-out.
- Tomcat request threads blocked longer on remote I/O.
- Increased pressure on DB, Access, and internal async execution due to longer-lived requests.
- Larger blast radius from a small number of unhealthy remote repositories.

### Low-Level Design

- Model the main failure path as `client request -> virtual repo resolution -> remote repo fan-out -> outbound HTTP wait`.
- Record where waste occurs:
  - outbound connection slot consumption
  - request thread occupancy
  - repeated upstream failures
- Treat one remote repository as the smallest control boundary for detection and remediation.

## Goals

- Detect unhealthy remote repositories from logs with low operational ambiguity.
- Aggregate health signals per remote repository over short and long time windows.
- Alert on repositories that are likely harming platform performance.
- Build a clean path toward controlled remediation, including offline mode automation.
- Identify high-traffic remote repositories and support priority resolution decisions.

### Low-Level Design

- For each goal, define one measurable output:
  - detection -> normalized outbound events
  - aggregation -> per-repo time-window counters
  - alerting -> warning and critical rule evaluations
  - remediation -> repository action candidates
  - prioritization -> ranked repository usage reports

## Non-Goals

- Real-time root cause analysis for all upstream failures.
- Replacing Artifactory’s native repository health mechanisms.
- Immediate automated repository state changes without guardrails.
- Full implementation details of log pipeline tooling.

### Low-Level Design

- Do not couple this design to one specific backend such as Loki, Elasticsearch, or Splunk.
- Do not require changes to Artifactory runtime behavior in phase 1.
- Do not treat raw logs as operator-facing output; only normalized events and aggregates are first-class outputs.

## Scope

Initial scope covers:

- `artifactory-service.log`
- `artifactory-request-out.log`
- Per-remote-repository aggregation
- High-level alerting inputs
- Future automation hooks for changing repository state

Out of scope for the first iteration:

- UI/dashboard design
- Terraform implementation details
- Exact parser syntax for a specific logging backend
- Per-client or per-virtual-repository attribution if not already present in logs

### Low-Level Design

- In-scope pipeline stages:
  - ingestion
  - parsing
  - normalization
  - aggregation
  - alert evaluation
- Out-of-scope capabilities must not block initial implementation; they should be modeled as optional enrichments.

## Rationale

Sending requests to broken remote repositories, especially in large virtual repositories, consumes outbound HTTP capacity without delivering useful resolution results.

The intended control model is:

1. Observe outbound success and failure behavior per remote repository.
2. Detect persistent unhealthy repositories.
3. Alert when unhealthy repositories are likely causing platform waste.
4. Optionally trigger controlled remediation, such as switching low-value remotes to offline mode.
5. Use traffic distribution to prioritize healthy and frequently used remotes in virtual repository resolution order.

### Low-Level Design

- Represent the control model as four logical services:
  - parser
  - aggregator
  - evaluator
  - remediation orchestrator
- Pass data between services using stable records keyed by `repo_key` and time window.
- Store enough history to support both short-term alerting and long-term optimization decisions.

## Data Sources

### 1. `artifactory-service.log`

Primary source for remote repository error events, including:

- timeouts
- connection failures
- DNS failures
- TLS failures
- socket resets

This log should be used to classify network and transport-level failures.

#### Low-Level Design

- Required extracted fields:
  - timestamp
  - logger or source file
  - repository key if present
  - error message
  - optional upstream host
- Parser output record type: `service_error_event`.
- If `repo_key` cannot be extracted, mark the event as `unattributed` and keep it out of repository-level automation.

### 2. `artifactory-request-out.log`

Primary source for outbound HTTP response monitoring, including:

- outbound request count
- HTTP response code class
- successful requests
- 5xx responses
- request duration, which is mandatory for this design

This log should be used to quantify successful and failed outbound traffic per remote repository.

#### Low-Level Design

- Required extracted fields:
  - timestamp
  - repository key
  - method
  - URL or upstream host
  - status code
  - duration
- Parser output record type: `request_out_event`.
- Records without `repo_key` should still be retained for platform-wide volume analysis but excluded from repository actions.
- Records without duration are invalid for this design and must be counted by a parser quality metric.
- Duration is required because long request times for small files may indicate queueing or saturation in outbound HTTP connection usage.

## Normalized Event Model

Both log streams should be normalized into a shared outbound event schema.

Suggested fields:

```text
ts
repo_key
event_type
status_code
status_class
error_type
remote_host
method
duration_ms
raw_message
```

Field meanings:

- `repo_key`: Artifactory remote repository key.
- `event_type`: one of `success`, `http_error`, `network_error`.
- `status_code`: outbound HTTP status code if available.
- `status_class`: one of `2xx`, `3xx`, `4xx`, `5xx`, or null. For `4xx`, aggregation must also preserve `status_code` so that `401`, `403`, `404`, and all other `4xx` responses can be analyzed separately.
- `error_type`: normalized failure category for network errors.
- `remote_host`: upstream host if extractable.
- `duration_ms`: request latency in milliseconds. This field is mandatory for `request_out_event`.

### Low-Level Design

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
attribution_state
raw_message
```

Contract rules:

- `event_id` must be unique per ingested log line.
- `source_type` must be `service_log` or `request_out_log`.
- `attribution_state` must be `attributed` or `unattributed`.
- `status_code` and `status_class` are nullable for network-only failures.
- `error_type` is nullable for successful HTTP requests.
- `duration_ms` is mandatory for outbound HTTP request events and optional only for service-log-derived network errors.

## Error Normalization

Errors from `artifactory-service.log` should be normalized into stable categories.

Suggested mapping:

- `connect timed out`, `read timed out` -> `timeout`
- `UnknownHost`, `no such host` -> `dns`
- `SSLException`, `handshake` -> `tls`
- `Connection refused` -> `connect_refused`
- `Connection reset` -> `reset`
- other transport failures -> `other_network`

This normalization is important because alerting should operate on small, stable labels rather than raw free-text messages.

### Low-Level Design

Normalization function contract:

- input: raw log message plus optional parsed exception text
- output: one normalized `error_type`

Fallback behavior:

- if multiple patterns match, use the most specific category
- if no pattern matches, emit `other_network`
- store the raw matched token in a secondary field if supported by the pipeline

## Aggregation Strategy

The primary aggregation key is:

- `repo_key`

Useful secondary dimensions:

- `repo_key + error_type`
- `repo_key + status_class`
- `repo_key + remote_host`
- `repo_key + package_type` if available from repository metadata

Recommended time windows:

- `5m` for rapid detection
- `1h` for short-term trend
- `24h` for persistent unhealthy state and remediation gating

### Low-Level Design

Define three aggregation tables or equivalent logical views:

- `repo_health_5m`
- `repo_health_1h`
- `repo_health_24h`

Each aggregate row should be keyed by:

```text
window_start
window_end
repo_key
```

Optional dimension tables:

- `repo_health_by_error_type`
- `repo_health_by_status_code`
- `repo_health_by_remote_host`

## Core Derived Metrics

### Traffic Metrics

- `requests_total{repo_key}`
- `success_total{repo_key}`
- `http_4xx_total{repo_key}`
- `http_4xx_total{repo_key,status_code}`
- `http_5xx_total{repo_key}`
- `http_401_total{repo_key}`
- `http_403_total{repo_key}`
- `http_404_total{repo_key}`
- `http_4xx_other_total{repo_key}`

#### Low-Level Design

- `requests_total` must count all attributed outbound request events.
- `success_total` must include `2xx` and `3xx`.
- `http_4xx_total` must be derivable from `status_code` without double-counting the more specific 401/403/404 breakdown.
- `http_4xx_other_total` must exclude `401`, `403`, and `404`.

### Network Failure Metrics

- `network_error_total{repo_key}`
- `network_error_total{repo_key,error_type}`
- `timeout_total{repo_key}`

#### Low-Level Design

- `network_error_total` must count only normalized `network_error` events.
- `timeout_total` must be derived from `error_type=timeout`, not from generic string search at query time.

### Freshness / Availability Metrics

- `last_success_timestamp{repo_key}`
- `last_error_timestamp{repo_key}`
- `success_count_24h{repo_key}`
- `error_count_24h{repo_key}`

#### Low-Level Design

- `last_success_timestamp` must come from the latest successful attributed `request_out_event`.
- `last_error_timestamp` must consider both `http_error` and `network_error`.
- freshness calculations should use event time, not ingestion time.

### Optional Performance Metrics

- `latency_ms_sum{repo_key}`
- `latency_ms_count{repo_key}`
- `latency_ms_p95{repo_key}` if supported by the log pipeline

#### Low-Level Design

- latency metrics should be computed from all valid attributed outbound request events.
- p95 must be scoped per `repo_key` and time window.
- missing duration must not be treated as zero.
- a dedicated small-object latency view should be created so the system can detect unexpectedly long request times for small files.
- the small-object latency view should be used as a leading indicator for queueing, connection pool exhaustion, or upstream slowness.

## Interpretation Rules

Signals should not be treated equally.

Strong unhealthy signals:

- timeouts
- DNS failures
- TLS failures
- connection refused
- repeated 5xx from upstream
- persistent error bursts with no recent success

Weak or noisy signals:

- isolated 404s
- low-volume repositories with rare traffic

Important note for `4xx` handling:

- `401` should be aggregated separately because it may indicate expired credentials or invalid credentials for the remote repository.
- `403` should be aggregated separately because it may indicate access-policy issues, or may indicate connectivity behavior through an upstream proxy layer.
- `404` should be aggregated separately and treated as a fan-out indicator rather than a direct health signal.
- all remaining `4xx` responses should stay grouped under a generic `other_4xx` bucket unless a repeated pattern justifies further splitting.

Important note for Maven:

- `404` must not be treated as a direct indicator of repository health.
- Many Maven resolution flows generate expected 404s while searching for metadata or artifacts across multiple remotes.
- A high level of `404` for a remote repository should trigger usage analysis rather than direct remediation.

### Low-Level Design

Define a repository state machine:

- `healthy`
- `degraded`
- `suspect`
- `fanout-heavy`
- `action-candidate`

Example transitions:

- repeated transport failures + no success -> `suspect`
- sustained high 404 + useful success still present -> `fanout-heavy`
- sustained failures + policy allows action -> `action-candidate`

## High 404 Analysis Model

Repositories with elevated `404` counts should be analyzed differently from repositories with transport failures.

Target questions:

- which teams are generating the requests
- whether the remote repository is broadly useful through the virtual repository
- whether the remote repository should stay inside the virtual repository

Recommended analysis flow:

1. Identify repositories with sustained high `404` volume.
2. Correlate usage to teams, clients, or build sources if the available telemetry supports it.
3. Estimate how many distinct teams are actively using the repository through the virtual repository path.
4. If fewer than a defined threshold `X` teams are using the repository, contact the team or teams.
5. Ask those teams to call the remote repository directly rather than through the large virtual repository when appropriate.
6. Consider moving the remote repository out of the virtual repository to reduce fan-out for all other consumers.

This flow is intended to reduce unnecessary virtual repository breadth and lower outbound request waste without breaking valid team-specific dependency resolution patterns.

### Low-Level Design

For each repository with high `404`, produce an analysis record:

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
- if `distinct_teams >= X`, treat as shared behavior and prioritize virtual repo optimization instead
- if top consumers cannot be mapped, hold the repository in review state rather than acting automatically

## Initial Query Model

The first implementation should produce per-repository aggregates from both logs.

Minimum useful outputs:

- successes per `repo_key`
- 5xx per `repo_key`
- network errors per `repo_key,error_type`
- last success time per `repo_key`

High-level query logic:

1. Parse `artifactory-service.log` into network error events.
2. Parse `artifactory-request-out.log` into outbound response events.
3. Normalize both into the shared schema.
4. Aggregate by `repo_key` over `5m`, `1h`, and `24h`.

### Low-Level Design

Pipeline stages:

1. ingest raw logs
2. parse raw fields
3. enrich with `repo_key`, `status_code`, `remote_host`, and optional team metadata
4. normalize to canonical event schema
5. write events to a searchable store
6. compute rolling aggregates
7. evaluate alert rules and produce action candidates

Required parser outputs:

- `service_error_event`
- `request_out_event`
- `normalized_outbound_event`

## Alerting Model

Alerting should combine volume, persistence, and lack of success.

Recommended first alert condition:

- high number of `timeout` or network errors for a remote repository
- zero successful outbound requests for the same repository in the last `24h`
- recent evidence that the repository is still being used

Recommended safeguards:

- require a minimum failure count threshold
- do not alert on extremely low-volume repositories
- treat `404` separately from transport failures

Example alert classes:

- Warning: elevated transport errors with intermittent success
- Critical: persistent transport errors and no success in the last `24h`

### Low-Level Design

Recommended rule inputs per `repo_key`:

- `timeout_total_5m`
- `network_error_total_5m`
- `success_total_24h`
- `requests_total_24h`
- `http_403_total_1h`
- `http_401_total_1h`
- `http_404_total_24h`

Recommended rule output:

```text
alert_name
severity
repo_key
reason
supporting_metrics
recommended_runbook
```

Alert routing guidance:

- `401` -> credentials or secret ownership path
- `403` -> proxy, upstream ACL, or repository config path
- transport failures -> platform or network operations path
- high `404` -> repository design optimization path

## Proposed Aggregation Outputs

For each remote repository, maintain:

- recent success count
- recent 5xx count
- recent timeout/network error count
- last successful request timestamp
- last error timestamp
- optional success ratio
- optional p95 latency

This supports both dashboards and automation decisions.

### Low-Level Design

Materialize one repository summary object per time window:

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
state
```

This summary object is the preferred input for dashboards, alerts, and remediation scoring.

## Automation Path

Automation is intentionally deferred until the signal quality is proven.

Target future flow:

1. Grafana alert detects unhealthy remote repository behavior.
2. Incident or validation workflow confirms signal quality.
3. Automation opens a PR against repository configuration.
4. PR changes the remote repository to offline mode if policy allows.
5. Change is reviewed or auto-approved according to guardrails.

Required guardrails:

- repository is not classified as critical and unique
- there is evidence of persistent failure, not a transient issue
- there has been no recent success
- rollback path is simple and documented

### Low-Level Design

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

## Priority Resolution Guidance

Monitoring alone does not solve the root problem when virtual repositories contain many low-value remotes.

The design should support follow-up analysis for:

- most frequently used remote repositories per technology
- remote repositories with the highest failure volume
- remote repositories with high latency and low success value

These outputs should be used to:

- reorder remotes in virtual repositories
- enable priority resolution for high-value remotes
- remove or isolate low-value unhealthy remotes
- reduce unnecessary fan-out for Maven and Python ecosystems

### Low-Level Design

Produce a ranked report per technology:

```text
technology
repo_key
request_share
success_share
404_share
timeout_share
latency_rank
priority_recommendation
```

Ranking inputs should favor:

- high request volume
- high success utility
- low failure cost

Repositories with low utility and high failure or fan-out cost should move toward lower priority or removal from large virtual repositories.

## Operational Rollout

### Phase 1

- Parse and normalize both log sources.
- Produce per-repository metrics.
- Build dashboards for error counts, success counts, and last success time.

#### Low-Level Design

- implement parsers
- validate field extraction on sample logs
- publish normalized event schema
- backfill short retention aggregates

### Phase 2

- Add alerting with minimum-volume thresholds.
- Validate signal quality against real incidents.

#### Low-Level Design

- add repository state derivation
- deploy warning and critical rules
- compare alerts with known failures and false positives

### Phase 3

- Add repository classification and allowlist/blocklist for automation.
- Introduce controlled PR-based offline mode remediation.

#### Low-Level Design

- add policy metadata store
- generate `action_candidate` records
- integrate PR creation workflow with review gates

## Risks

- Log formats may vary between Artifactory versions and environments.
- Some repositories may be rarely used, producing misleading low-volume signals.
- `404` noise may be high for Maven and similar ecosystems.
- Missing `repo_key` in some log lines may reduce attribution quality.
- Automation without repository criticality metadata would be unsafe.

### Low-Level Design

For each risk, define a mitigation:

- variable log format -> version parser rules and keep parser tests
- low-volume noise -> minimum sample thresholds
- high `404` noise -> separate `404` analysis path
- missing attribution -> exclude unattributed events from automation
- unsafe automation -> require policy metadata and human approval by default

## Open Questions

- What exact log formats are present in the target environment?
- Can `repo_key` always be extracted reliably from both logs?
- Is per-technology classification already available from repository metadata?
- Which repositories are considered automation-safe for offline mode?
- What is the desired threshold policy for warning vs critical alerts?

### Low-Level Design

Track open questions as implementation blockers with owners:

```text
question_id
question
owner
target_date
blocking_area
status
```

## Recommended Next Step

Collect a representative sample from:

- `artifactory-service.log`
- `artifactory-request-out.log`

Then define parser rules and exact queries for Loki.

### Low-Level Design

Required next-step deliverables:

- sample log corpus
- parser test cases
- first normalized schema draft
- first aggregate query set
- first alert threshold proposal
