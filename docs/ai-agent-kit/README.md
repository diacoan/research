# AI Agent Kit for Artifactory Operations

## Purpose

This directory contains a reusable kit for applying AI agents to issue detection, bottleneck analysis, and operational data interpretation for Artifactory as a platform.

The kit is designed to work with the existing repo documents, especially:

- `docs/artifactory-observability-quick-check.md`
- `docs/high-level-artifactory-capacity-interdependencies.md`
- `docs/cloudsql-proxy-sidecar-for-artifactory-playbook.md`
- `docs/cloudsql-postgres-for-artifactory-playbook.md`
- `docs/gcs-filestore-for-artifactory-playbook.md`
- `docs/gke-cluster-resource-assessment-playbook.md`
- `docs/outbound-http-pool-tuning-for-remote-repositories.md`
- `docs/jfrog-log-parsing-standard-loki.md`

Use it together with:

- `docs/enterprise-doc-kit/context/`

The AI agents should treat the enterprise context files as environment facts and the existing Markdown playbooks as conceptual and operational references.

## Core Principle

The AI layer should not replace telemetry, alerting, or runbooks.

It should:

- normalize and explain signals
- correlate symptoms across domains
- separate likely bottlenecks from nearby but non-dominant noise
- express confidence and uncertainty clearly
- produce operator-safe actions with explicit guardrails

It should not:

- invent runtime state from Helm values alone
- hide missing telemetry
- recommend aggressive tuning without proving the dominant bottleneck first

## Directory Layout

1. `architecture/`
   Reference architecture for a multi-agent operational analysis flow.

2. `prompts/`
   Prompt templates for the main agent roles.

3. `schemas/`
   Canonical output schema for agent analysis results.

4. reference documents in the kit root
   including:
   - `ai-agents-required-inputs-and-decision-flow.md`

## Recommended Agent Chain

1. `Context loader`
   Load completed context YAML files, repo docs, and recent chat summaries.

2. `Telemetry normalizer`
   Convert metrics, logs, events, and config fragments into normalized facts.

3. `Symptom classifier`
   Group normalized facts into symptom families such as DB wait, heap pressure, proxy saturation, remote fan-out waste, or node-pressure risk.

4. `Domain specialist`
   Analyze one domain at a time:
   `app-jvm`, `db-proxy`, `storage-gcs`, `gke-capacity`, `remote-repos`, `xray`.

5. `Cross-domain correlator`
   Determine the most likely active bottleneck on the end-to-end request path.

6. `Decision and runbook agent`
   Produce safe next steps, decision guidance, ownership, and escalation paths.

## Output Contract

All agents should emit YAML that conforms to:

- `schemas/canonical-agent-output.schema.yaml`

The schema is intentionally broad enough for:

- context loading
- telemetry normalization
- triage analysis
- cross-domain bottleneck correlation
- decision support

## Suggested Workflow

1. Load current enterprise context from `docs/enterprise-doc-kit/context/`.
2. Load the latest chat summaries or operator notes that describe the current focus.
3. Normalize telemetry into explicit facts.
4. Classify symptoms before proposing tuning.
5. Correlate across domains to avoid false attribution.
6. Produce recommendations ordered by safety:
   - validate first
   - change conservatively
   - avoid actions that only increase concurrency without relieving the bottleneck

## Initial High-Value Use Cases

- Cloud SQL instance bottleneck vs `cloud-sql-proxy` sidecar bottleneck
- JVM heap pressure vs pod memory pressure vs node memory pressure
- remote repository upstream failure vs virtual-repository fan-out waste
- storage latency vs local `cache-fs` or temp-disk pressure
- true GKE headroom vs misleading manifest-level `requests` or missing `limits`

## Included Reference Document

- `ai-agents-required-inputs-and-decision-flow.md`
  Defines the minimum input model, source map, and symptom-to-decision flow for the agents.
