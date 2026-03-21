# Proactive Scaling Strategy: Operating Model Overview

## Purpose

Define how the platform team should operate the monitoring, triage, ownership, and change loop that turns telemetry into safe proactive scaling actions.

## Operating Principle

The operating model should optimize for correct attribution before capacity change.

In practice, the loop is:

1. maintain trusted telemetry
2. classify the dominant symptom
3. correlate across neighboring domains
4. assign the correct owner
5. make the smallest effective change
6. verify the result in both short and long windows

## Baseline Operating Stages

### Stage 1: Keep the Observability Baseline Trustworthy

Before any proactive scaling program is credible, confirm:

- metrics are present and stable
- key labels such as `repo_key`, pod, container, and node are usable
- short and long windows can be queried reliably
- parser-quality and unattributed-event rates are known

If this stage is weak, scaling decisions should remain conservative.

### Stage 2: Run Symptom-Led Triage

Start from the dominant symptom family:

- latency with low CPU
- DB pool pressure
- proxy asymmetry
- local disk pressure
- remote transport failure
- `404` fan-out waste
- node-pressure or scheduling failure

Do not start from the preferred tuning action.

### Stage 3: Correlate Neighboring Domains

Every main domain in this repo has at least one close neighbor that can create false attribution:

- app latency may actually be DB, proxy, remote, or storage wait
- Cloud SQL instance health may hide proxy-side bottlenecks
- healthy GCS may hide local cache or temp-disk pressure
- more replicas may fail because of GKE allocatable or anti-affinity constraints
- remote repository problems may create artificial concurrency demand

The operating model should require one correlation pass before material scaling.

### Stage 4: Assign the Correct Owner

Suggested owner map:

- platform engineering: app capacity, remote-repo design, edge, node-pool fit
- database or cloud team: Cloud SQL baseline, storage, failover, instance shape
- Kubernetes or infrastructure team: node pools, autoscaling, anti-affinity, scheduling constraints
- observability owner: parser quality, telemetry gaps, dashboard integrity
- repository or upstream owner: credentials, ACL, proxy path, upstream reachability

### Stage 5: Choose the Smallest Effective Change

Preferred action order:

1. remove invalid demand
2. fix broken dependency paths
3. tune local choke points
4. scale shared dependencies
5. scale app replicas or request concurrency

This order reflects the repo-wide guidance that more concurrency often increases waiting if the real bottleneck is elsewhere.

### Stage 6: Verify After the Change

Every material change should have:

- a short-window validation
- a `24h` stability check
- a `30d` or `90d` review if the change was capacity-related

Without this verification loop, the organization accumulates tuning folklore instead of operating knowledge.

## Review Cadence

Recommended cadence:

- continuous: alerts and dashboard monitoring for active symptoms
- daily: review of active high-risk patterns and asymmetric pods
- weekly: top repository issues, top failure-cost sources, top proxy or node-pressure patterns
- monthly: `30d` and `90d` capacity review for node pools, Cloud SQL, storage growth, and observability tax

## Change Classes in Scope

This repo supports proactive actions in at least these classes:

- repository cleanup or controlled offline mode
- credentials, policy, or proxy-path repair
- outbound HTTP pool tuning
- JVM or app pool tuning
- `cloud-sql-proxy` sidecar sizing
- Cloud SQL instance sizing or storage adjustments
- GCS and local cache policy tuning
- GKE node-pool resizing, autoscaling, and placement improvements
- anti-affinity and topology spread refinement
- observability-footprint reduction

## Operational Guardrails

- Treat shared dependencies as higher-risk change targets than per-pod local settings.
- Keep cluster, node pool, node, pod, and repository scopes separate until the evidence says they are linked.
- Require rollback criteria for changes that affect request concurrency, DB connections, or node-pool topology.
- Prefer reversible changes before architectural changes during active incidents.

## Optional AI-Assisted Operating Layer

The AI-agent kit in this repo can be used as an interpretation layer above monitoring and dashboards.

Recommended chain:

1. context loader
2. telemetry normalizer
3. symptom classifier
4. domain specialists
5. cross-domain correlator
6. decision and runbook agent

Use it to improve explanation quality and prioritization, not to bypass operator review.

Relevant docs:

- [../ai-agent-kit/README.md](../ai-agent-kit/README.md)
- [../ai-agent-kit/ai-agents-required-inputs-and-decision-flow.md](../ai-agent-kit/ai-agents-required-inputs-and-decision-flow.md)
- [../ai-agent-kit/architecture/artifactory-ai-agent-architecture.md](../ai-agent-kit/architecture/artifactory-ai-agent-architecture.md)

## Detailed Sources

- [../artifactory-observability-quick-check.md](../artifactory-observability-quick-check.md)
- [../high-level-artifactory-capacity-interdependencies.md](../high-level-artifactory-capacity-interdependencies.md)
- [../cloudsql-proxy-sidecar-for-artifactory-playbook.md](../cloudsql-proxy-sidecar-for-artifactory-playbook.md)
- [../cloudsql-postgres-for-artifactory-playbook.md](../cloudsql-postgres-for-artifactory-playbook.md)
- [../gcs-filestore-for-artifactory-playbook.md](../gcs-filestore-for-artifactory-playbook.md)
- [../gke-cluster-resource-assessment-playbook.md](../gke-cluster-resource-assessment-playbook.md)
- [../next-steps-remote-repo-monitoring_revized.md](../next-steps-remote-repo-monitoring_revized.md)
