# AI Agent Architecture for Artifactory Operational Analysis

## Purpose

Define a practical multi-agent architecture for easier issue detection, bottleneck identification, and data interpretation for Artifactory as a platform.

This architecture is intended for environments where:

- Artifactory runs in HA on Kubernetes
- the request path crosses multiple shared dependencies
- useful diagnosis requires both telemetry and deployment context
- operators need explanations, not only alerts

## Problem Statement

Artifactory platform issues are often misdiagnosed because the visible symptom and the dominant bottleneck are not the same thing.

Typical examples:

- high latency with low CPU that is actually DB wait, remote wait, or proxy wait
- healthy Cloud SQL instance metrics while one Artifactory primary is slow because of a per-pod `cloud-sql-proxy`
- storage complaints that are really local cache or temp-disk pressure
- apparent compute shortage that is actually queueing behind shared pools

A single general-purpose assistant tends to flatten these distinctions unless the analysis is explicitly staged.

## Stability, Performance, and Functionality Impact

Poor AI architecture can make operations worse:

- Stability impact:
  - wrong changes during incidents
  - increased blast radius from blind tuning
  - missing per-pod or per-node failure domains
- Performance impact:
  - raising concurrency instead of removing the actual wait source
  - masking queueing until harder failures appear
  - treating throughput and latency as the same problem
- Functionality impact:
  - degraded user flows remain unexplained
  - asymmetry across primaries is missed
  - runbooks are not matched to the right dependency layer

## Core Principle

Use AI agents to explain and correlate facts, not to replace observability systems.

Recommended split:

- rules, queries, and dashboards detect signals
- AI agents normalize, correlate, prioritize, and explain
- humans approve material changes

## Reference Capacity Domains

The agent system should reason over the same capacity domains already established in this repo:

1. request execution capacity
2. database progress capacity
3. Access and platform coordination capacity
4. outbound HTTP progress capacity
5. storage progress capacity
6. GKE allocatable and placement capacity
7. proxy-side path capacity where per-pod sidecars are present

This keeps the AI layer aligned with:

- `artifactory-observability-quick-check.md`
- `high-level-artifactory-capacity-interdependencies.md`
- domain playbooks under `docs/`

## Agent Roles

### 1. Context Loader Agent

Responsibilities:

- load enterprise context YAML files
- load local playbooks and recent chat context files
- extract ownership, topology, SLOs, blind spots, and accepted tradeoffs
- publish only environment facts and documented constraints

Should not:

- infer runtime state from Helm values alone
- declare active bottlenecks

## 2. Telemetry Normalizer Agent

Responsibilities:

- normalize metrics, logs, events, and config snapshots into explicit observations
- preserve source, time window, and confidence
- separate:
  - observed fact
  - documented fact
  - operational inference
  - assumption

Should not:

- tune anything
- collapse multiple signal types into one synthetic statement without traceability

## 3. Symptom Classifier Agent

Responsibilities:

- map normalized observations into symptom families such as:
  - heap pressure
  - DB pool saturation
  - proxy dial stress
  - remote transport failure
  - fan-out waste
  - local disk pressure
  - scheduling or node-pressure risk
- rank symptom severity and likely domain

Should not:

- claim the dominant bottleneck without cross-domain correlation

## 4. Domain Specialist Agents

Recommended domain agents:

- `app-jvm`
- `db-proxy`
- `storage-gcs`
- `gke-capacity`
- `remote-repos`
- `xray`

Responsibilities:

- analyze one domain deeply
- explain what the signals mean inside that domain
- identify likely local constraints and non-constraints
- return domain-scoped hypotheses, not global certainty

## 5. Cross-Domain Correlator Agent

Responsibilities:

- compare domain hypotheses on the same request path
- determine where wait is most likely accumulating
- distinguish:
  - dominant bottleneck
  - secondary stress
  - coincidental noise
- explain why close alternatives are less likely

This agent produces the most important judgment in the chain.

## 6. Decision and Runbook Agent

Responsibilities:

- convert likely bottlenecks into safe actions
- order actions by safety and reversibility
- map recommendations to owners and verification steps
- call out what not to change yet

This agent should speak in operational language, not model-centric language.

## 7. Human Approval Gate

Keep a human gate for:

- scaling changes
- memory and thread-pool increases
- Cloud SQL tier changes
- GKE scheduling or anti-affinity changes
- any automation that can affect user-facing traffic

## Canonical Analysis Flow

```text
enterprise context
  + playbooks
  + chat context
  + telemetry
  -> context loader
  -> telemetry normalizer
  -> symptom classifier
  -> domain specialists
  -> cross-domain correlator
  -> decision and runbook agent
  -> operator
```

## Data Flow Rules

1. Every important statement must cite at least one source.
2. Every inferred statement must be labeled as `operational_inference`.
3. Time windows must be explicit:
   - short-term triage: `5m`, `15m`, `1h`
   - incident trend: `24h`
   - capacity planning: `30d`, `90d`
4. Environment facts from completed context files override generic defaults.
5. Missing telemetry is itself a finding, not a silent omission.

## Recommended Output Shape

All agents should emit the same high-level output contract:

- metadata
- input scope
- observations
- hypotheses
- bottleneck assessment
- impact assessment
- recommended actions
- open questions
- references used

Use:

- `../schemas/canonical-agent-output.schema.yaml`

## Guardrails

### Guardrail 1: Detection Should Stay Rules-First

Use AI for interpretation after:

- PromQL
- LogQL
- alert rules
- monitoring dashboards
- config extraction

### Guardrail 2: Never Tune Concurrency Blindly

The AI layer should explicitly block recommendations such as:

- raising `maxThreads`
- raising DB pools
- raising outbound pools

unless the current bottleneck analysis justifies them.

### Guardrail 3: Preserve Failure-Domain Granularity

The system must keep separate scopes where relevant:

- cluster
- node pool
- node
- pod
- container
- service
- repository
- remote host

### Guardrail 4: Prefer Explanations With Exclusions

A good result should say:

- what is likely
- what is less likely
- what evidence is missing

not only what sounds plausible.

## Suggested Initial Deployment Pattern

Start with a read-only copilot pattern:

1. operators ask for analysis
2. agents return structured findings
3. humans execute changes

Only after repeated validation should you consider automation for:

- issue classification
- ticket drafting
- dashboard annotations
- change suggestions

## High-Value First Scenarios

1. Distinguish Cloud SQL instance saturation from `cloud-sql-proxy` sidecar saturation.
2. Distinguish heap pressure from broader pod or node memory pressure.
3. Distinguish remote-upstream failure from virtual-repository fan-out waste.
4. Distinguish real cluster shortage from poor packing, `DaemonSet` tax, or missing `limits`.
5. Distinguish bucket health from local `cache-fs`, temp, and log-disk pressure.

## References

- `../README.md`
- `../../artifactory-observability-quick-check.md`
- `../../high-level-artifactory-capacity-interdependencies.md`
- `../../cloudsql-proxy-sidecar-for-artifactory-playbook.md`
- `../../cloudsql-postgres-for-artifactory-playbook.md`
- `../../gcs-filestore-for-artifactory-playbook.md`
- `../../gke-cluster-resource-assessment-playbook.md`
- `../../outbound-http-pool-tuning-for-remote-repositories.md`
- `../../enterprise-doc-kit/context/`

