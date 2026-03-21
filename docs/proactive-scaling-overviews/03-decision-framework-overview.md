# Proactive Scaling Strategy: Decision Framework Overview

## Purpose

Define the decision logic for when to scale, what to scale, and what should be fixed first instead of scaled.

## Core Decision Rule

Scale only after the dominant bottleneck is identified with enough confidence that the chosen action is likely to reduce wait, not only redistribute it.

In this repo, the main decision mistake to avoid is increasing concurrency against a dependency layer that is already the real choke point.

## Decision Sequence

### 1. Confirm the Symptom and Time Window

Record:

- what is slow or unstable
- which users or workloads are affected
- whether the pattern is current, recurring, or long-term
- which window matters most: `5m`, `24h`, `30d`, or `90d`

### 2. Identify the Failure Domain

Start by deciding where the problem is most likely centered:

- repository or upstream host
- pod or sidecar
- shared service such as Cloud SQL or GCS
- node or node pool
- cluster-wide placement or autoscaling envelope

### 3. Separate Invalid Demand from Valid Growth

Ask this before any scale action:

- is the platform under real business load growth
- or is it spending capacity on broken remotes, bad credentials, policy failures, repeated `404` fan-out, or noisy observability

If the answer is "invalid demand", fix that first.

### 4. Determine the Bottleneck Type

Use the repo-wide domains:

- request execution
- DB progress
- Access or platform coordination
- proxy-side path
- outbound HTTP progress
- storage progress
- GKE allocatable and placement

### 5. Choose the Safest Effective Action Class

Use this action order:

1. repair or remove waste
2. tune the local choke point
3. scale the shared dependency
4. scale replicas or request concurrency

## Preferred Decision Patterns

| Observed pattern | Prefer this decision first | Avoid this decision first |
| --- | --- | --- |
| High `401` or `403` against remotes | fix credentials, ACL, or proxy path | raising pools or threads |
| High `404` with continued useful success | reduce virtual-repo fan-out and low-value remotes | increasing outbound pool size |
| High latency, low CPU, DB pool active near max | validate DB headroom and pool alignment | raising `maxThreads` |
| One or two primaries degraded, Cloud SQL instance normal | inspect `cloud-sql-proxy` sidecars and node-local conditions | scaling Cloud SQL blindly |
| Bucket healthy but local disk filling | tune cache, temp, logs, or local PVC shape | assuming GCS scale solves the issue |
| Scaling blocked by `FailedScheduling` or autoscaler limits | revise node-pool shape, max nodes, or placement constraints | increasing pod replicas only |
| Healthy remotes but high tiny/small-object latency on shared host | review outbound per-route and upstream-path limits | raising Tomcat threads first |
| High observability footprint during incidents | reduce telemetry tax to recover headroom | adding more logging or sidecars indiscriminately |

## What Usually Counts as a True Scaling Decision

The repository material supports these as true proactive scaling decisions:

- increasing Artifactory replica count when shared dependencies have proven headroom
- increasing pod CPU or memory when the app itself is the constrained layer
- increasing `cloud-sql-proxy` sidecar resources when the proxy path is the bottleneck
- resizing Cloud SQL compute or storage when the instance itself is the progress gate
- resizing node pools or changing machine family when GKE headroom or packing is the dominant limit
- increasing outbound HTTP capacity only when remotes are healthy and outbound wait is the real bottleneck

## What Usually Counts as a Non-Scaling Fix

These are often higher-value than scaling:

- cleaning virtual repositories
- fixing remote credentials or access policy
- correcting proxy or network path issues
- reducing observability overhead
- improving anti-affinity or topology spread
- fixing local cache, temp-disk, or retention policy problems

## Validation Rules Before Approval

Before approving a proactive scaling action, require:

- explicit statement of the dominant bottleneck
- evidence from at least one neighboring domain showing the alternative explanation is weaker
- target metrics for post-change validation
- rollback criteria
- ownership for both execution and review

## Optional AI-Assisted Decision Support

The AI-agent kit can help structure the decision path by separating:

- observed facts
- documented facts
- operational inferences
- assumptions

Use it to improve confidence and traceability, not to auto-approve scale changes.

Relevant docs:

- [../ai-agent-kit/ai-agents-required-inputs-and-decision-flow.md](../ai-agent-kit/ai-agents-required-inputs-and-decision-flow.md)
- [../ai-agent-kit/architecture/artifactory-ai-agent-architecture.md](../ai-agent-kit/architecture/artifactory-ai-agent-architecture.md)

## Detailed Sources

- [../high-level-artifactory-capacity-interdependencies.md](../high-level-artifactory-capacity-interdependencies.md)
- [../artifactory-observability-quick-check.md](../artifactory-observability-quick-check.md)
- [../cloudsql-proxy-sidecar-for-artifactory-playbook.md](../cloudsql-proxy-sidecar-for-artifactory-playbook.md)
- [../cloudsql-postgres-for-artifactory-playbook.md](../cloudsql-postgres-for-artifactory-playbook.md)
- [../gcs-filestore-for-artifactory-playbook.md](../gcs-filestore-for-artifactory-playbook.md)
- [../gke-cluster-resource-assessment-playbook.md](../gke-cluster-resource-assessment-playbook.md)
- [../high-level-design-remote-repo-monitoring_revized.md](../high-level-design-remote-repo-monitoring_revized.md)
- [../outbound-http-pool-tuning-for-remote-repositories.md](../outbound-http-pool-tuning-for-remote-repositories.md)
