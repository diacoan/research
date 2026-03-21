# Proactive Scaling Strategy: Executive Abstract

## Purpose

Provide a compact strategic entry point for proactive scaling of Artifactory as a platform, based on the detailed playbooks already present in `docs/` and the recent chat-context summaries in the repository root.

## Executive Summary

In this repo, proactive scaling should not be read as "add more replicas when users complain".

It should be read as a closed loop:

1. detect early stress signals
2. identify the dominant bottleneck and its failure domain
3. remove invalid demand before adding capacity
4. scale the correct layer with explicit verification

The main operating model already established across the repo is that Artifactory throughput is constrained by the slowest active layer on the request path, not by one isolated Helm parameter.

Useful high-level model:

```text
effective_throughput
  ~= min(
       request_execution_capacity,
       db_progress_capacity,
       access_progress_capacity,
       outbound_progress_capacity,
       storage_progress_capacity,
       gke_allocatable_capacity,
       proxy_sidecar_path_capacity
     )
```

This is an engineering model for operations and decision making. It is not a vendor formula.

## What Proactive Scaling Means in This Repository

The detailed documents in `docs/` show that platform degradation usually appears first as one of these patterns:

- rising request lifetime with low useful progress
- DB pool pressure before obvious DB-instance alarms
- asymmetric slowdown caused by per-pod `cloud-sql-proxy` sidecars
- remote-repository fan-out waste that consumes outbound capacity
- local cache or node pressure even when GCS looks healthy
- GKE packing, node-pressure, or autoscaler limits that block otherwise valid scaling

Therefore, proactive scaling in this workspace means:

- monitor leading indicators before hard failures
- reason per failure domain: repository, pod, sidecar, node, node pool, shared service
- separate invalid demand from true business growth
- choose the safest scale action that actually removes the dominant wait source

## Strategic Objectives

1. Protect stability by detecting bottlenecks before they become `OOMKilled`, `FailedScheduling`, or hard user-facing incidents.
2. Protect performance by reducing queueing and wait amplification, not only by increasing concurrency.
3. Protect functionality by keeping shared dependencies healthy enough that additional replicas can provide useful throughput.
4. Make scaling decisions explainable, reversible, and tied to telemetry windows such as `5m`, `1h`, `24h`, `30d`, and `90d`.

## Core Guardrails

- Do not raise `maxThreads` first when the dominant symptom is DB wait, remote wait, or storage wait.
- Do not scale Artifactory primaries without accounting for Cloud SQL, `cloud-sql-proxy`, GCS or local cache, and GKE node headroom.
- Do not treat `401`, `403`, and broad `404` fan-out as capacity problems until credentials, policy, and repository design are reviewed.
- Do not treat instance-level Cloud SQL health as proof that the per-pod proxy path is healthy.
- Do not ignore observability overhead, because sidecars and extra log volume can consume the same headroom operators are trying to protect.

## Recommended Reading Order

1. [01-monitoring-strategy-overview.md](./01-monitoring-strategy-overview.md)
2. [02-operating-model-overview.md](./02-operating-model-overview.md)
3. [03-decision-framework-overview.md](./03-decision-framework-overview.md)

Then go deeper into the detailed playbooks:

- [../artifactory-observability-quick-check.md](../artifactory-observability-quick-check.md)
- [../high-level-artifactory-capacity-interdependencies.md](../high-level-artifactory-capacity-interdependencies.md)
- [../cloudsql-proxy-sidecar-for-artifactory-playbook.md](../cloudsql-proxy-sidecar-for-artifactory-playbook.md)
- [../cloudsql-postgres-for-artifactory-playbook.md](../cloudsql-postgres-for-artifactory-playbook.md)
- [../gcs-filestore-for-artifactory-playbook.md](../gcs-filestore-for-artifactory-playbook.md)
- [../gke-cluster-resource-assessment-playbook.md](../gke-cluster-resource-assessment-playbook.md)
- [../high-level-design-remote-repo-monitoring_revized.md](../high-level-design-remote-repo-monitoring_revized.md)
- [../next-steps-remote-repo-monitoring_revized.md](../next-steps-remote-repo-monitoring_revized.md)
- [../outbound-http-pool-tuning-for-remote-repositories.md](../outbound-http-pool-tuning-for-remote-repositories.md)

## Optional AI-Assisted Layer

The repo also now contains an AI-agent operating model that can sit above the monitoring and decision loop, but should not replace it:

- [../ai-agent-kit/README.md](../ai-agent-kit/README.md)
- [../ai-agent-kit/ai-agents-required-inputs-and-decision-flow.md](../ai-agent-kit/ai-agents-required-inputs-and-decision-flow.md)
- [../ai-agent-kit/architecture/artifactory-ai-agent-architecture.md](../ai-agent-kit/architecture/artifactory-ai-agent-architecture.md)

Recommended role of this layer:

- normalize telemetry
- correlate cross-domain symptoms
- rank bottleneck candidates
- suggest safe next actions

Human approval should remain mandatory for material scale changes.

## Source Basis

This abstract is synthesized from:

- `current-chat-context.md`
- `chat-context-summary-2026-03-21.md`
- the Artifactory, Cloud SQL, GCS, GKE, remote-repository, and AI-agent documents under `docs/`
