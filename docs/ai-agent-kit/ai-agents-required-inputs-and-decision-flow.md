# AI Agents: Required Inputs and Decision Flow

## Purpose

Define what AI agents need as input, where those inputs should come from, and how the agents should move from symptoms to operationally safe conclusions for Artifactory as a platform.

This document translates the working notes in `notes.md` into an operational model that is reusable across:

- triage
- performance analysis
- bottleneck isolation
- capacity planning
- enterprise-specific decision support

Use it together with:

- `architecture/artifactory-ai-agent-architecture.md`
- `schemas/canonical-agent-output.schema.yaml`
- `../enterprise-doc-kit/context/`
- the existing operational playbooks under `docs/`

## Problem Statement

An AI agent is only useful if it is grounded in the right combination of:

- vendor documentation
- environment-specific configuration
- runtime telemetry
- operational decision logic

If one of these layers is missing, the result becomes weak:

- docs without runtime data produce generic advice
- runtime data without topology and ownership produces shallow conclusions
- metrics without playbooks produce correlations but not decisions
- Helm values without real environment facts produce false certainty

## Stability, Performance, and Functionality Impact

Poor input quality or poor decision flow can degrade operations directly:

- Stability impact:
  - unsafe tuning during incidents
  - missed node, pod, or sidecar failure domains
  - repeated misclassification of root cause
- Performance impact:
  - concurrency increases that amplify wait instead of reducing it
  - confusion between compute saturation and dependency wait
  - failure to detect the real bottleneck in shared paths
- Functionality impact:
  - user-visible slowness remains unexplained
  - asymmetric degradation across Artifactory primaries is missed
  - platform teams do not know which owner should act next

## Core Model

The required input model has four layers:

1. grounding and static context
2. dynamic runtime evidence
3. decision playbooks and correlation logic
4. structured decision output

The AI layer should reason over all four, not only over metrics and logs.

## Layer 1: Grounding and Static Context

### Purpose

Give the agents trusted reference material and environment facts before they interpret live telemetry.

### What Belongs Here

#### Vendor grounding

- JFrog official documentation
- Google Cloud official documentation

#### Environment-specific configuration grounding

- installed JFrog Helm charts and effective values
- overlays, injected sidecars, and non-chart manifests
- Google Cloud services in use, their sizing, and their actual configuration

#### Enterprise operating context

- topology
- ownership boundaries
- SLOs and SLIs
- maintenance and approval model
- accepted tradeoffs
- known blind spots
- known incident patterns

### Important Rule

Helm charts are configuration references, not complete runtime truth.

Agents should treat:

- Helm and repo docs as configuration and conceptual grounding
- completed enterprise context YAML files as environment facts
- runtime telemetry as evidence of actual behavior

### Practical Sources in This Repo

- `docs/high-level-artifactory-capacity-interdependencies.md`
- `docs/artifactory-observability-quick-check.md`
- `docs/cloudsql-proxy-sidecar-for-artifactory-playbook.md`
- `docs/cloudsql-postgres-for-artifactory-playbook.md`
- `docs/gcs-filestore-for-artifactory-playbook.md`
- `docs/gke-cluster-resource-assessment-playbook.md`
- `docs/outbound-http-pool-tuning-for-remote-repositories.md`
- `docs/jfrog-log-parsing-standard-loki.md`
- `docs/enterprise-doc-kit/context/`
- `current-chat-context.md`
- `chat-context-summary-2026-03-21.md`

## Layer 2: Dynamic Runtime Evidence

### Purpose

Give the agents direct evidence of what the platform is doing now and how it behaved over time.

### Required Dynamic Inputs

#### Metrics

- metrics collected from REST API or OpenMetrics endpoints
- Prometheus aggregates
- cloud service metrics from Google Cloud Monitoring

Examples:

- JVM heap and runtime metrics
- DB pool metrics
- Cloud SQL metrics
- `cloud-sql-proxy` telemetry
- GCS bucket or storage-path metrics
- GKE node, pod, and container resource metrics

#### Logs

- Artifactory service logs
- `request.log`
- `request-out.log`
- router logs
- `cloud-sql-proxy` logs
- Kubernetes events and workload logs
- Google Cloud service logs where relevant

#### Runtime events and state signals

- restarts
- `OOMKilled`
- `FailedScheduling`
- evictions
- autoscaler failures
- rollout events
- maintenance or failover events

#### Historical windows

Agents should read runtime evidence across explicit time windows:

- `5m`, `15m`, `1h` for short-term triage
- `24h` for incident context
- `30d`, `90d` for capacity and behavior patterns

### Important Rule

Dynamic data does not mean only metrics and log analytics.

For Artifactory platform operations, runtime evidence must also include:

- Kubernetes scheduling and health events
- Google Cloud service telemetry
- proxy-side behavior
- per-pod and per-node asymmetry

## Layer 3: Decision Playbooks and Correlation Logic

### Purpose

Turn raw or normalized evidence into repeatable operational reasoning.

### What Belongs Here

- use-case-based decision playbooks
- symptom classifiers
- domain correlation rules
- guardrails that block unsafe tuning

### Required Reasoning Pattern

The agents should not jump directly from one signal to one action.

Recommended pattern:

1. identify the primary symptom
2. classify the symptom family
3. correlate with neighboring domains
4. rank bottleneck candidates
5. exclude weak alternatives where possible
6. propose only safe next actions

### Minimum Statement Types

Every important statement should be explicitly labeled as one of:

- `observed_fact`
- `documented_fact`
- `operational_inference`
- `assumption`

This prevents the agent from mixing telemetry and guesswork into the same conclusion.

## Layer 4: Structured Decision Output

### Purpose

Ensure the agents produce operationally useful output rather than free-form explanations only.

### Required Output Elements

At minimum, the agent output should include:

- summary
- observations
- hypotheses
- bottleneck assessment
- impact on:
  - operability
  - functionality
  - performance
- recommended actions:
  - safe now
  - validate next
  - low-risk changes
  - avoid for now
- open questions
- references used

Use:

- `schemas/canonical-agent-output.schema.yaml`

## Source Map by Input Category

| Input category | What the agent needs | Typical sources |
| --- | --- | --- |
| Vendor grounding | correct semantics and product constraints | JFrog official docs, Google Cloud official docs |
| JFrog config grounding | effective Artifactory, Access, Metadata, Xray, Nginx settings | Helm chart, values, overlays, rendered `system.yaml`, sidecar manifests |
| Cloud service grounding | real DB, storage, and cluster shape | Cloud SQL config, GCS bucket config, GKE cluster and node-pool config |
| Enterprise context | topology, owners, SLOs, constraints, incidents | `docs/enterprise-doc-kit/context/`, chat context files, operator notes |
| Runtime metrics | current and historical behavior | Prometheus, OpenMetrics endpoints, Google Cloud Monitoring |
| Logs and event evidence | failures, latencies, restarts, scheduling issues | Loki, Promtail, Kubernetes events, Cloud Logging |
| Decision logic | what to correlate and what to avoid changing early | existing playbooks under `docs/`, incident patterns, runbooks |

## Example Use Case: User Slowness

### Purpose

Show the expected decision flow for a common operational symptom.

### Symptom

Users report slowness in Artifactory.

### Correct Decision Flow

#### Step 1: Check primary signals

Start with the most direct indicators:

- request latency
- error rate
- active requests
- CPU and memory usage
- JVM heap pressure
- DB pool activity
- Cloud SQL instance signals
- `cloud-sql-proxy` sidecar signals
- outbound HTTP wait
- local disk and storage signals
- node pressure and scheduling signals

#### Step 2: Classify the symptom family

Possible families:

- application-side concurrency or heap pressure
- DB pool saturation
- Cloud SQL instance stress
- `cloud-sql-proxy` bottleneck
- outbound remote wait
- virtual-repository fan-out waste
- storage latency or local disk pressure
- GKE node or scheduling pressure

#### Step 3: Correlate across domains

Examples:

- if latency is high and CPU is low, correlate with DB wait, proxy wait, remote wait, and storage wait before calling it a compute problem
- if one or two primaries are slow while Cloud SQL looks normal, compare per-pod `cloud-sql-proxy` behavior before resizing the database
- if high `404` exists together with useful success on remote traffic, correlate with virtual-repository design before treating it as an upstream outage
- if pod memory pressure exists, correlate JVM heap, direct memory, sidecars, and node pressure before changing only `-Xmx`

#### Step 4: Produce a bottleneck assessment

The agent should return:

- dominant bottleneck candidate
- secondary stress signals
- plausible alternatives that are less likely
- confidence level

#### Step 5: Produce safe operational actions

Example action classes:

- `safe_now`
  - verify missing telemetry
  - compare affected vs healthy primaries
  - inspect per-pod asymmetry
- `validate_next`
  - correlate DB pool metrics with proxy open connections
  - correlate latency spikes with node-pressure windows
  - correlate remote errors by `remote_host`
- `low_risk_changes`
  - only when the dominant bottleneck is sufficiently supported
- `avoid_for_now`
  - avoid increasing `maxThreads`, DB pools, or replica count before the wait source is proven

### Short Form Decision Pattern

The working pattern from the notes is correct after one refinement:

1. check `x`, `y`, `z`
2. if `X`, correlate with `m` and `n`; if `Y`, correlate with `a`, `b`, and `c`
3. if `correlation_1`, produce an `operational_inference`
4. before action, state:
   - confidence
   - missing evidence
   - actions safe now
   - actions to avoid for now

The missing piece in the original note is step `4`.

## What the Agents Should Not Do

- treat Helm values as proof of runtime behavior
- recommend major tuning based on one metric alone
- collapse cluster-wide and per-pod behavior into the same conclusion
- ignore missing telemetry
- output actions without confidence and rationale
- confuse documentation facts with live observations

## Cross-Topic Interdependencies

The decision flow must stay aligned with the cross-domain capacity model already established in this repo.

Important examples:

- `maxThreads` is not useful throughput by itself
- DB pool pressure can dominate even when CPU looks healthy
- `cloud-sql-proxy` can bottleneck before Cloud SQL instance metrics look stressed
- remote-repository issues can manifest as app slowness through outbound wait and thread retention
- GCS bucket health does not prove local cache or temp-disk health
- missing `limits` in Kubernetes makes historical usage and failure events more important than manifest intent

## Recommended Minimum Input Set for a First Useful Deployment

If you want the smallest viable AI-agent deployment, start with:

1. enterprise context files:
   - topology
   - GKE
   - Cloud SQL proxy
   - observability and SLOs
   - incidents and patterns
2. core playbooks from `docs/`
3. Prometheus or equivalent metric source
4. Loki or equivalent log source
5. Kubernetes events
6. one canonical structured output schema

This is enough for a read-only copilot model that explains likely bottlenecks without automating changes.

## Decision Output Standard

A good final answer from the agents should be able to answer all of these:

- what is happening
- where it is happening
- why it is likely happening there
- what other explanations were considered
- what evidence is still missing
- what action is safe right now
- what change should be avoided until more proof exists
- which owner should take the next step

## References Used

- `notes.md`
- `current-chat-context.md`
- `chat-context-summary-2026-03-21.md`
- `ai-agent-kit/architecture/artifactory-ai-agent-architecture.md`
- `ai-agent-kit/schemas/canonical-agent-output.schema.yaml`
- `artifactory-observability-quick-check.md`
- `high-level-artifactory-capacity-interdependencies.md`
- `cloudsql-proxy-sidecar-for-artifactory-playbook.md`
- `cloudsql-postgres-for-artifactory-playbook.md`
- `gcs-filestore-for-artifactory-playbook.md`
- `gke-cluster-resource-assessment-playbook.md`
- `outbound-http-pool-tuning-for-remote-repositories.md`
- `jfrog-log-parsing-standard-loki.md`
