# Prompt for Enterprise GKE Sizing and Tuning Playbook

Use this prompt to generate an enterprise-specific version of the GKE sizing and tuning playbook.

## Prompt

Generate a Markdown document for the enterprise GKE cluster sizing, tuning, and operational decision model used by Artifactory and related JFrog products.

### Reference Documents

- `docs/gke-cluster-resource-assessment-playbook.md`
- `docs/gke-cluster-resource-assessment-worksheet-guide.md`
- `docs/cloudsql-proxy-sidecar-for-artifactory-playbook.md`
- `docs/cloudsql-postgres-for-artifactory-playbook.md`
- `docs/gcs-filestore-for-artifactory-playbook.md`
- `docs/artifactory-observability-quick-check.md`

### Required Context Files

- `docs/enterprise-doc-kit/context/00-document-profile.template.yaml`
- `docs/enterprise-doc-kit/context/01-enterprise-platform-context.template.yaml`
- `docs/enterprise-doc-kit/context/02-topology-and-deployment-context.template.yaml`
- `docs/enterprise-doc-kit/context/03-gke-cluster-context.template.yaml`
- `docs/enterprise-doc-kit/context/05-cloudsql-proxy-context.template.yaml`
- `docs/enterprise-doc-kit/context/07-observability-and-slos.template.yaml`
- `docs/enterprise-doc-kit/context/08-security-network-and-org.template.yaml`
- `docs/enterprise-doc-kit/context/09-incidents-patterns-and-decisions.template.yaml`

### Output Focus

The document must explain:

- the real enterprise node pool strategy
- current and future sidecar tax
- DaemonSet overhead
- workload placement and isolation
- how cluster-level behavior affects Artifactory, proxy, DB, and storage performance
- how sizing decisions are actually made under enterprise constraints

### Required Detailed Sub-Topics

- cluster shape and node pool purpose
- allocatable vs requests vs limits vs observed usage
- sidecar and DaemonSet tax
- scheduling and failure-domain behavior
- monitoring and logs for capacity diagnosis
- decision model for machine type, node count, and workload isolation

### Special Instructions

- Use actual enterprise node pool names and machine types from context.
- Explain where runtime usage data exists and where the organization is still relying on estimates.
- Tie GKE recommendations back to Artifactory primaries, Cloud SQL proxies, and observability overhead.
