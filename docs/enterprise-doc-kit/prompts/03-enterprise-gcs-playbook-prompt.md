# Prompt for Enterprise GCS Filestore Playbook

Use this prompt to generate an enterprise-specific version of the GCS filestore playbook.

## Prompt

Generate a Markdown document for the enterprise GCS filestore used by Artifactory.

### Reference Documents

- `docs/gcs-filestore-for-artifactory-playbook.md`
- `docs/artifactory-observability-quick-check.md`
- `docs/high-level-artifactory-capacity-interdependencies.md`
- `docs/gke-cluster-resource-assessment-playbook.md`

### Required Context Files

- `docs/enterprise-doc-kit/context/00-document-profile.template.yaml`
- `docs/enterprise-doc-kit/context/01-enterprise-platform-context.template.yaml`
- `docs/enterprise-doc-kit/context/02-topology-and-deployment-context.template.yaml`
- `docs/enterprise-doc-kit/context/06-gcs-filestore-context.template.yaml`
- `docs/enterprise-doc-kit/context/07-observability-and-slos.template.yaml`
- `docs/enterprise-doc-kit/context/08-security-network-and-org.template.yaml`
- `docs/enterprise-doc-kit/context/09-incidents-patterns-and-decisions.template.yaml`

### Output Focus

The document must explain the enterprise GCS design, not just the generic `google-storage-v2-direct` model.

Cover at minimum:

- actual bucket location, protection model, and lifecycle
- real IAM and access path
- Artifactory cache-fs interaction with local disk
- observed request patterns and large vs small object behavior
- logging, audit, retention, and deletion protection posture
- enterprise operational ownership and change rules

### Required Detailed Sub-Topics

- bucket identity and lifecycle protection
- Artifactory provider settings and cache-fs behavior
- request rate and performance scaling behavior
- local disk pressure and cache interactions
- observability, audit, and access logging
- tuning and decision rules for provider settings and bucket controls

### Special Instructions

- If advanced provider settings such as `maxConnections` or `connectionTimeout` are set through custom `binarystore.xml`, explain that operationally.
- If the enterprise has no bucket access logs or metrics, call that out as a monitoring blind spot.
- Tie every recommendation to actual observed usage patterns or to open questions if the data is missing.
