# Prompt for Enterprise Cloud SQL Playbook

Use this prompt to generate an enterprise-specific version of the Cloud SQL PostgreSQL playbook.

## Prompt

Generate a Markdown document for the enterprise Cloud SQL for PostgreSQL layer used by Artifactory.

### Reference Documents

- `docs/cloudsql-postgres-for-artifactory-playbook.md`
- `docs/cloudsql-proxy-sidecar-for-artifactory-playbook.md`
- `docs/artifactory-observability-quick-check.md`
- `docs/high-level-artifactory-capacity-interdependencies.md`
- `docs/gke-cluster-resource-assessment-playbook.md`

### Required Context Files

- `docs/enterprise-doc-kit/context/00-document-profile.template.yaml`
- `docs/enterprise-doc-kit/context/01-enterprise-platform-context.template.yaml`
- `docs/enterprise-doc-kit/context/02-topology-and-deployment-context.template.yaml`
- `docs/enterprise-doc-kit/context/04-cloudsql-context.template.yaml`
- `docs/enterprise-doc-kit/context/07-observability-and-slos.template.yaml`
- `docs/enterprise-doc-kit/context/08-security-network-and-org.template.yaml`
- `docs/enterprise-doc-kit/context/09-incidents-patterns-and-decisions.template.yaml`

### Output Focus

The document must explain the Cloud SQL layer in the enterprise context, not in the generic repo context.

Cover at minimum:

- real Cloud SQL instance shape and HA model
- actual connection budget presented by Artifactory
- interaction with Cloud SQL Auth Proxy sidecars
- storage, flags, maintenance, failover, and Query Insights
- enterprise monitoring and alerting posture
- ownership and approval boundaries for DB changes

### Section Requirements

Keep the standard structure:

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

### Detailed Sub-Topics to Include

- instance shape and HA baseline
- connection capacity and Artifactory pool budget
- query pressure, waits, and contention
- storage growth, WAL, and disk pressure
- failover, maintenance, and operational risk
- enterprise monitoring and operational ownership

### Decision Rules

- distinguish instance-level bottlenecks from proxy-side bottlenecks
- distinguish DB capacity problems from query-behavior problems
- distinguish safe resizing actions from actions that require DBA review or CAB approval
- explicitly state when the enterprise lacks enough telemetry for confident tuning
