# Enterprise Documentation Adaptation Kit

## Purpose

This directory contains reusable inputs for turning the generic playbooks in this repo into enterprise-specific Markdown documents.

The kit is designed for cases where:

- the Helm chart explains only part of the runtime reality
- the final documents must reflect actual enterprise topology, constraints, and operating model
- additional facts must be supplied from GKE, Cloud SQL, GCS, IAM, networking, observability, and operational history

## High-Level Structure

This kit has three layers:

1. `context/`
   Fill-in templates for enterprise facts that cannot be derived from the Artifactory Helm chart alone.

2. `prompts/`
   Reusable prompt templates for generating enterprise-specific Markdown documents while using the current repo documents as conceptual and operational references.

3. `generated/`
   Suggested output location for the final enterprise-specific documents created from the prompts and context files.

## What Cannot Be Reliably Extracted from the Helm Chart

The Helm chart can tell you a lot about:

- Artifactory JVM settings
- pool sizes
- replica counts
- storage provider type
- some observability and service-level settings

But it cannot reliably tell you:

- the actual GKE node pools, autoscaling, and DaemonSet tax
- the real Cloud SQL instance shape, HA mode, flags, maintenance, and quotas
- the actual Cloud SQL Auth Proxy sidecar manifest and runtime resources
- the real GCS bucket lifecycle, retention, logging, and IAM posture
- traffic mix, peak windows, and enterprise workload patterns
- SLOs, incident history, change management rules, and business criticality
- enterprise network path, firewalls, NAT, proxies, PSC, or VPC design
- who owns each operational decision and what risk is acceptable

## Recommended Workflow

1. Fill `context/00-document-profile.template.yaml`.
2. Fill only the domain-specific context templates that are relevant.
3. Choose the prompt from `prompts/` that matches the target document.
4. Load the referenced docs from `../` plus the completed context files.
5. Generate the enterprise-specific Markdown doc into `generated/`.
6. Review open questions and unresolved assumptions before treating the output as final guidance.

## Output Structure for Generated Documents

Each generated enterprise document should keep this structure:

1. `Purpose`
2. `Problem Statement`
3. `Stability, Performance, and Functionality Impact`
4. `Enterprise Context`
5. `Quick Check`
6. Detailed sub-topics
7. `Cross-Topic Interdependencies`
8. `Decision Matrix`
9. `Open Questions and Validation Gaps`
10. `References Used`

Each detailed sub-topic should include:

- concept / explanation
- functionality
- metrics / parameters and what they represent
- how they correlate
- what and how to monitor or verify
- potential problems
- decision making guide based on observed patterns

## Reference Map

Use these repo documents as the main references:

- Artifactory app and JVM:
  - [../artifactory-jvm-parameters-guide.md](../artifactory-jvm-parameters-guide.md)
  - [../high-level-artifactory-capacity-interdependencies.md](../high-level-artifactory-capacity-interdependencies.md)
- Cloud SQL:
  - [../cloudsql-postgres-for-artifactory-playbook.md](../cloudsql-postgres-for-artifactory-playbook.md)
- Cloud SQL proxy:
  - [../cloudsql-proxy-sidecar-for-artifactory-playbook.md](../cloudsql-proxy-sidecar-for-artifactory-playbook.md)
- GCS filestore:
  - [../gcs-filestore-for-artifactory-playbook.md](../gcs-filestore-for-artifactory-playbook.md)
- GKE sizing and monitoring:
  - [../gke-cluster-resource-assessment-playbook.md](../gke-cluster-resource-assessment-playbook.md)
  - [../gke-cluster-resource-assessment-worksheet-guide.md](../gke-cluster-resource-assessment-worksheet-guide.md)
  - [../gke-cluster-resource-assessment-worksheet.csv](../gke-cluster-resource-assessment-worksheet.csv)
- Cross-domain operational quick checks:
  - [../artifactory-observability-quick-check.md](../artifactory-observability-quick-check.md)
- Remote repository specifics:
  - [../outbound-http-pool-tuning-for-remote-repositories.md](../outbound-http-pool-tuning-for-remote-repositories.md)
  - [../high-level-design-remote-repo-monitoring_revized.md](../high-level-design-remote-repo-monitoring_revized.md)
  - [../low-level-design-remote-repo-monitoring_revized.md](../low-level-design-remote-repo-monitoring_revized.md)

## Suggested Naming

Suggested output file naming:

- `<environment>-enterprise-cloudsql-playbook.md`
- `<environment>-enterprise-cloudsql-proxy-playbook.md`
- `<environment>-enterprise-gcs-filestore-playbook.md`
- `<environment>-enterprise-gke-sizing-playbook.md`
- `<environment>-enterprise-artifactory-app-sizing-playbook.md`
- `<environment>-enterprise-cross-domain-decision-matrix.md`

## Quality Rules

The prompts in this kit assume that the generated document should:

- preserve the current repo style and sectioning
- replace generic statements with enterprise facts when available
- explicitly flag unknowns as open questions
- distinguish operational inference from stated fact
- avoid pretending that Helm values alone describe runtime behavior
