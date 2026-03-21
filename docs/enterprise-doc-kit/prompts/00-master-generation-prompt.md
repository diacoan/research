# Master Prompt for Enterprise-Specific Markdown Generation

Use this prompt when you want to generate a new enterprise-specific Markdown document from the current repo documents plus completed context templates.

## Prompt

You are generating an enterprise-specific Markdown document for an Artifactory platform running on GKE with external Google-managed dependencies.

Use the current repo documents as conceptual and operational references, but adapt the output to the completed enterprise context files. Do not restate generic guidance when specific enterprise facts are available.

### Inputs

Reference documents:

- `docs/artifactory-observability-quick-check.md`
- `docs/high-level-artifactory-capacity-interdependencies.md`
- `docs/artifactory-jvm-parameters-guide.md`
- domain-specific playbooks from `docs/`

Enterprise context files:

- `docs/enterprise-doc-kit/context/00-document-profile.template.yaml`
- relevant completed domain context files from `docs/enterprise-doc-kit/context/`

### Output Goal

Generate one Markdown document named:

- `<environment>-<topic>-enterprise-playbook.md`

The output must preserve the section pattern used in the current repo:

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

### Requirements

- Use the target language from `document_profile.target_language`.
- Treat completed context files as enterprise facts.
- Treat repo docs as conceptual and operational references.
- Explicitly mark important statements that are operational inference rather than hard fact.
- If enterprise facts contradict repo defaults, prefer enterprise facts.
- If a required fact is missing, add it to `Open Questions and Validation Gaps`.
- Keep the document concrete and operational, not abstract.
- Avoid generic “best practices” bullets unless they are tied to the enterprise context.

### Rules for Each Detailed Sub-Topic

Each sub-topic must include:

- concept / explanation
- functionality
- metrics / parameters and what they mean
- how they correlate
- what and how to monitor or verify
- potential problems
- decision making guide based on behavior and patterns

### Enterprise-Specific Additions

Add the following wherever relevant:

- actual ownership boundaries
- change approval rules
- enterprise risk tolerance
- observability blind spots
- known incidents and recurring patterns
- constraints that cannot be changed easily

### Final Review Checklist

Before finalizing, verify that:

- the document is not just a paraphrase of the repo docs
- the enterprise topology appears explicitly
- monitoring guidance maps to the actual available telemetry
- recommendations respect enterprise constraints and ownership
- open questions are listed where the context is incomplete
