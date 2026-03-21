# Prompt for Enterprise Cloud SQL Proxy Playbook

Use this prompt to generate an enterprise-specific version of the Cloud SQL Auth Proxy sidecar playbook.

## Prompt

Generate a Markdown document for the enterprise Cloud SQL Auth Proxy sidecar used by Artifactory primary pods.

### Reference Documents

- `docs/cloudsql-proxy-sidecar-for-artifactory-playbook.md`
- `docs/cloudsql-postgres-for-artifactory-playbook.md`
- `docs/gke-cluster-resource-assessment-playbook.md`
- `docs/artifactory-observability-quick-check.md`
- `docs/high-level-artifactory-capacity-interdependencies.md`

### Required Context Files

- `docs/enterprise-doc-kit/context/00-document-profile.template.yaml`
- `docs/enterprise-doc-kit/context/01-enterprise-platform-context.template.yaml`
- `docs/enterprise-doc-kit/context/02-topology-and-deployment-context.template.yaml`
- `docs/enterprise-doc-kit/context/03-gke-cluster-context.template.yaml`
- `docs/enterprise-doc-kit/context/04-cloudsql-context.template.yaml`
- `docs/enterprise-doc-kit/context/05-cloudsql-proxy-context.template.yaml`
- `docs/enterprise-doc-kit/context/07-observability-and-slos.template.yaml`
- `docs/enterprise-doc-kit/context/08-security-network-and-org.template.yaml`
- `docs/enterprise-doc-kit/context/09-incidents-patterns-and-decisions.template.yaml`

### Output Focus

The document must explain:

- how the enterprise deploys the proxy sidecar
- which Artifactory services share each sidecar
- how to calculate configured and observed connection envelopes
- how to derive requests and limits from observed behavior
- how to distinguish proxy bottlenecks from Cloud SQL instance bottlenecks
- what enterprise constraints apply to tuning, restarts, and debug telemetry

### Required Detailed Sub-Topics

- proxy role in the end-to-end DB path
- sidecar resource sizing and scheduling
- resource allocation formulas based on Artifactory pools and observed telemetry
- proxy metrics, logs, and diagnostics
- network path and private IP behavior
- enterprise tuning order and escalation paths

### Special Instructions

- Use the enterprise proxy metrics and sidecar resources from context if available.
- If the enterprise lacks proxy telemetry, state that explicitly as a blind spot.
- Convert generic formulas into enterprise-specific examples when enough data exists.
- Include clear guidance for when to tune the sidecar, when to tune Artifactory pools, and when to escalate to DB or platform teams.
