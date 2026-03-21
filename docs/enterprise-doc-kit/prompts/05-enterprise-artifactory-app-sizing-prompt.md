# Prompt for Enterprise Artifactory App Sizing and Tuning Playbook

Use this prompt to generate an enterprise-specific Artifactory application sizing and tuning document.

## Prompt

Generate a Markdown document for the enterprise Artifactory application layer, focusing on how JVM settings, Tomcat concurrency, DB pools, remote repository behavior, and storage dependencies interact in the actual runtime environment.

### Reference Documents

- `docs/artifactory-jvm-parameters-guide.md`
- `docs/artifactory-observability-quick-check.md`
- `docs/high-level-artifactory-capacity-interdependencies.md`
- `docs/outbound-http-pool-tuning-for-remote-repositories.md`
- `docs/cloudsql-postgres-for-artifactory-playbook.md`
- `docs/cloudsql-proxy-sidecar-for-artifactory-playbook.md`
- `docs/gcs-filestore-for-artifactory-playbook.md`

### Required Context Files

- `docs/enterprise-doc-kit/context/00-document-profile.template.yaml`
- `docs/enterprise-doc-kit/context/01-enterprise-platform-context.template.yaml`
- `docs/enterprise-doc-kit/context/02-topology-and-deployment-context.template.yaml`
- `docs/enterprise-doc-kit/context/04-cloudsql-context.template.yaml`
- `docs/enterprise-doc-kit/context/05-cloudsql-proxy-context.template.yaml`
- `docs/enterprise-doc-kit/context/06-gcs-filestore-context.template.yaml`
- `docs/enterprise-doc-kit/context/07-observability-and-slos.template.yaml`
- `docs/enterprise-doc-kit/context/09-incidents-patterns-and-decisions.template.yaml`

### Output Focus

The document must explain how the actual enterprise deployment should size and tune Artifactory as an application platform.

Cover at minimum:

- JVM heap and runtime behavior
- Tomcat request concurrency
- DB pool sizes by service
- remote repository outbound tuning if relevant
- dependency bottlenecks in Cloud SQL, proxy, GCS, and GKE
- enterprise risk, ownership, and change sequencing

### Special Instructions

- Do not present JVM tuning in isolation.
- Always tie JVM and pool guidance to the observed enterprise bottleneck patterns and dependency limits.
- If the enterprise has no good metrics for a section, say that explicitly and add it to validation gaps.
