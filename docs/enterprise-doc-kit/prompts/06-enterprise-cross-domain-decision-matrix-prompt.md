# Prompt for Cross-Domain Enterprise Decision Matrix

Use this prompt to generate a synthesis document that connects GKE, Artifactory app sizing, Cloud SQL, Cloud SQL proxy, and GCS filestore.

## Prompt

Generate a Markdown decision matrix for the enterprise Artifactory platform.

The goal is not to restate each domain playbook, but to connect them into one operator-facing decision model.

### Reference Documents

- `docs/artifactory-observability-quick-check.md`
- `docs/high-level-artifactory-capacity-interdependencies.md`
- `docs/cloudsql-postgres-for-artifactory-playbook.md`
- `docs/cloudsql-proxy-sidecar-for-artifactory-playbook.md`
- `docs/gcs-filestore-for-artifactory-playbook.md`
- `docs/gke-cluster-resource-assessment-playbook.md`
- any already generated enterprise-specific domain playbooks

### Required Context Files

- `docs/enterprise-doc-kit/context/00-document-profile.template.yaml`
- `docs/enterprise-doc-kit/context/01-enterprise-platform-context.template.yaml`
- `docs/enterprise-doc-kit/context/02-topology-and-deployment-context.template.yaml`
- `docs/enterprise-doc-kit/context/03-gke-cluster-context.template.yaml`
- `docs/enterprise-doc-kit/context/04-cloudsql-context.template.yaml`
- `docs/enterprise-doc-kit/context/05-cloudsql-proxy-context.template.yaml`
- `docs/enterprise-doc-kit/context/06-gcs-filestore-context.template.yaml`
- `docs/enterprise-doc-kit/context/07-observability-and-slos.template.yaml`
- `docs/enterprise-doc-kit/context/08-security-network-and-org.template.yaml`
- `docs/enterprise-doc-kit/context/09-incidents-patterns-and-decisions.template.yaml`

### Output Structure

The document must include:

1. `Purpose`
2. `Problem Statement`
3. `Stability, Performance, and Functionality Impact`
4. `Enterprise Context`
5. `Cross-Domain Quick Check`
6. `Interdependency Model`
7. `Behavior-to-Decision Matrix`
8. `Change Sequencing Guide`
9. `Open Questions and Validation Gaps`
10. `References Used`

### What the Matrix Must Do

- map symptoms to the most likely bottleneck domain
- separate primary bottleneck from secondary amplification effects
- show what team owns the next validation step
- show what can be tuned safely and what needs cross-team approval
- explain which changes should happen first, second, and never in parallel
