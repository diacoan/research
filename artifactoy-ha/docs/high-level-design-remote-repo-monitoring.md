# High Level Design: Remote Repository Health Monitoring and Control

## Purpose

Define a monitoring and control approach for unhealthy remote repositories in Artifactory, with emphasis on high-traffic virtual repositories used by Maven and Python workloads.

The goal is to reduce wasted outbound HTTP connections, reduce request latency amplification, and prevent degraded remote repositories from harming overall platform throughput.

## Problem Statement

Large virtual repositories can fan out requests across many remote repositories. When some remotes are slow, broken, or unreachable, Artifactory keeps spending outbound HTTP connections and request-processing time on low-value remote attempts.

This is especially damaging for request-hungry ecosystems such as Maven and Python, where a single build can trigger a large number of metadata and artifact resolution calls.

Observed platform risks:

- Increased consumption of outbound HTTP connection pool capacity.
- Higher request latency for healthy repositories due to retries and fan-out.
- Tomcat request threads blocked longer on remote I/O.
- Increased pressure on DB, Access, and internal async execution due to longer-lived requests.
- Larger blast radius from a small number of unhealthy remote repositories.

## Goals

- Detect unhealthy remote repositories from logs with low operational ambiguity.
- Aggregate health signals per remote repository over short and long time windows.
- Alert on repositories that are likely harming platform performance.
- Build a clean path toward controlled remediation, including offline mode automation.
- Identify high-traffic remote repositories and support priority resolution decisions.

## Non-Goals

- Near-real-time root cause analysis for all upstream failures.
- Replacing Artifactory’s native repository health mechanisms.
- Immediate automated repository state changes without guardrails.
- Full implementation details of log pipeline tooling.

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
- Exact final parser syntax beyond the initial `Promtail` implementation
- Per-client or per-virtual-repository attribution if not already present in logs

## Rationale

Sending requests to broken remote repositories, especially in large virtual repositories, consumes outbound HTTP capacity without delivering useful resolution results.

The intended control model is:

1. Observe outbound success and failure behavior per remote repository.
2. Detect persistent unhealthy repositories.
3. Alert when unhealthy repositories are likely causing platform waste.
4. Optionally trigger controlled remediation, such as switching low-value remotes to offline mode.
5. Use traffic distribution to prioritize healthy and frequently used remotes in virtual repository resolution order.

## Target Architecture

The initial implementation stack is:

- `Promtail` for collection and parsing
- `Loki` for storage and query-based aggregation
- `Grafana` for dashboards and alerting

Detection and alerting are near-real-time, not strict real-time, because signals depend on log shipping, ingestion, query windows, and alert evaluation intervals.

## Data Sources

### 1. `artifactory-service.log`

Primary source for remote repository error events, including:

- timeouts
- connection failures
- DNS failures
- TLS failures
- socket resets

### 2. `artifactory-request-out.log`

Primary source for outbound HTTP response monitoring, including:

- outbound request count
- HTTP response codes
- successful requests
- 4xx and 5xx responses
- request duration
- response size, if available

## Signal Model

Both log streams are normalized into a shared outbound-event model keyed by `repo_key`.

Primary signals:

- success volume
- transport error volume
- `401` volume
- `403` volume
- `404` volume
- `5xx` volume
- last success time
- last error time
- latency on `tiny` and `small` objects

Small-object latency is important because long request times for metadata and similar low-transfer files can indicate queueing, outbound HTTP pool contention, or upstream slowness.

## Interpretation Model

Signals are not equally meaningful:

- transport failures and sustained `5xx` are strong health signals
- `401` usually points to expired or invalid credentials
- `403` may point to access-policy issues or upstream proxy behavior
- `404` is usually a fan-out and virtual-repository design signal, not a direct remote health signal

For Maven, high `404` rates should trigger usage analysis and repository design review rather than direct remediation.

## High 404 Analysis

Repositories with elevated `404` counts should be reviewed to determine:

- which teams are generating the requests
- whether the repository is broadly useful through the virtual repository
- whether the repository should remain inside the virtual repository

If a repository is used by fewer than a defined threshold `X` teams, the preferred action is to contact those teams, ask them to call the remote directly when appropriate, and evaluate removing that remote from the large virtual repository.

## Alerting Model

Alerting should combine:

- error volume
- persistence
- lack of successful requests
- evidence that the repository is still in active use

Expected alert classes:

- Warning: elevated failures with intermittent success
- Critical: persistent failures and no success in the last `24h`

## Automation Path

Automation is deferred until signal quality is proven.

Target future flow:

1. Grafana alert detects unhealthy remote repository behavior.
2. Incident or validation workflow confirms signal quality.
3. Automation opens a PR against repository configuration.
4. PR changes the remote repository to offline mode if policy allows.
5. Change is reviewed or auto-approved according to guardrails.

## Priority Resolution Guidance

Monitoring should also support design optimization of large virtual repositories.

Outputs should support:

- identifying the most used remote repositories per technology
- identifying repositories with the highest timeout and failure cost
- identifying repositories with high `404` fan-out cost
- ranking repositories for priority resolution

## Risks

- Log formats may vary between Artifactory versions and environments.
- Some repositories may be rarely used, producing misleading low-volume signals.
- `404` noise may be high for Maven and similar ecosystems.
- Missing `repo_key` in some log lines may reduce attribution quality.
- Automation without repository criticality metadata would be unsafe.

## Open Questions

- What exact log formats are present in the target environment?
- Can `repo_key` always be extracted reliably from both logs?
- Is per-technology classification already available from repository metadata?
- Which repositories are considered automation-safe for offline mode?
- What is the desired threshold policy for warning vs critical alerts?

## Related Documents

- [Low Level Design: Remote Repository Health Monitoring and Control](/Users/lilstew/Downloads/artifactory-ha/docs/low-level-design-remote-repo-monitoring.md)
- [Next Steps: Remote Repository Monitoring](/Users/lilstew/Downloads/artifactory-ha/docs/next-steps-remote-repo-monitoring.md)
