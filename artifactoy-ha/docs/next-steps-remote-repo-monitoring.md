# Next Steps: Remote Repository Monitoring

## Purpose

Define what the team should do after monitoring starts producing usable signals.

This document is action-oriented. It maps observed patterns in monitoring to follow-up work, ownership, and expected outcomes.

## Phase 0: Establish Observability Baseline

Before decisions are made from monitoring, confirm:

- `Promtail` is collecting both target logs
- `repo_key` extraction is reliable enough for repository-level aggregation
- `duration_ms` is present for outbound requests
- response size is present or a fallback plan exists
- `Loki` queries return stable results over `5m`, `1h`, and `24h`
- `Grafana` dashboards show per-repo success, failure, and latency views

Exit criteria:

- parser accuracy is acceptable
- unattributed event rate is known
- dashboards are usable by platform operators

## Observation-Driven Next Steps

### 1. High Transport Errors

Observed pattern:

- repeated `timeout`, `dns`, `tls`, `connect_refused`, or `reset`
- low or zero recent success

Next steps:

- identify affected `repo_key`
- confirm whether the issue is isolated or shared across multiple repos on the same upstream host
- verify upstream reachability, credentials, TLS, proxy, and firewall path
- classify the repository as `suspect`
- decide whether the repo should become an `action-candidate`

Expected outcome:

- repository health issue is confirmed
- issue is routed to the correct owner
- repository is either fixed or queued for controlled offline remediation

### 2. High `401`

Observed pattern:

- rising `http_401_total`
- low success ratio

Next steps:

- identify secret ownership
- verify token or credential expiry
- verify whether credentials are shared across repos
- rotate or fix credentials
- add repository to credential-review watchlist if repeated

Expected outcome:

- authentication issues are fixed quickly
- repeated credential failures become visible as an ownership problem, not just as noise

### 3. High `403`

Observed pattern:

- rising `http_403_total`
- possible intermittent success

Next steps:

- verify upstream access policy
- verify proxy path and proxy ACL behavior
- confirm whether the issue is repository-specific or host-wide
- check whether a credentials fix was already attempted unsuccessfully

Expected outcome:

- access-control vs connectivity-via-proxy issues are separated cleanly
- remediation goes to the correct team

### 4. High `404`

Observed pattern:

- sustained high `http_404_total`
- repository still receives useful traffic

Next steps:

- identify which teams, clients, or build sources generate the requests
- estimate `distinct_teams`
- determine whether the repository is broadly useful in the virtual repository
- if `distinct_teams < X`, contact the relevant teams
- ask those teams to call the remote separately when appropriate
- evaluate moving the remote out of the large virtual repository

Expected outcome:

- reduced fan-out
- reduced wasted outbound lookups
- narrower and healthier virtual repository design

### 5. High Tiny/Small-Object Latency

Observed pattern:

- elevated p95 latency for `tiny` or `small` objects
- examples include `pom.xml`, `maven-metadata.xml`, checksum files, package indexes

Next steps:

- compare latency growth against transport error growth
- check whether the same time window shows elevated request concurrency
- check whether many repositories are affected at once
- determine whether the signal points to:
  - outbound HTTP connection contention
  - upstream slowness
  - queueing in the request path
- correlate with platform-level thread and outbound connection saturation signals if available

Expected outcome:

- queueing issues are detected before they show up only as hard failures
- platform tuning and repository design changes can be prioritized

### 6. High Failure Cost on High-Usage Repositories

Observed pattern:

- repository has high request share
- repository also has high timeout or failure cost

Next steps:

- classify the repo as high-priority
- review its position in virtual repository resolution order
- decide whether it should be prioritized, isolated, or remediated first

Expected outcome:

- failures on important repositories are addressed before low-value noise

## Weekly Review Cadence

Every week, review:

- top repositories by timeout count
- top repositories by `403`
- top repositories by `404`
- top repositories by tiny/small-object latency
- top repositories by request share
- repositories with no success in `24h`

Review outputs:

- repository health backlog
- candidate cleanup list for virtual repositories
- candidate offline-mode list
- candidate credential fixes
- candidate proxy or ACL fixes

## Decision Backlog

The monitoring program should maintain a backlog of repository actions:

- investigate
- fix credentials
- fix proxy or ACL
- remove from virtual
- reorder in virtual
- set offline mode
- no action

Each backlog item should carry:

- `repo_key`
- observed pattern
- time window
- owner
- severity
- decision status
- next review date

## Exit Criteria for Automation

Do not automate repository changes until all are true:

- false-positive rate is understood
- repository ownership metadata exists
- policy class exists for critical vs non-critical repositories
- rollback is simple
- action candidates are reviewed consistently

## Immediate Deliverables

- finalize sample logs for both Artifactory files
- write first `Promtail` parsing rules
- define first `Loki` queries
- build first `Grafana` dashboard
- set first warning and critical thresholds
- start weekly review with repository action backlog
