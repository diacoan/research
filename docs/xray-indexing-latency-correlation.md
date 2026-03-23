# Xray Indexing Latency Correlation

This note explains the most relevant Xray chart areas to inspect when there is a long delay between:

- artifact upload into Artifactory
- and the Xray log line `indexing message received for artifact ...`

## Imported Chart

The official JFrog Xray chart is now vendored locally at:

- [xray/Chart.yaml](/Users/lilstew/Downloads/research/xray/Chart.yaml)

At import time, the chart metadata shows:

- chart version `103.137.26`
- app version `3.137.26`

This imported chart is for local correlation and may not exactly match the deployed Xray version in a live environment.

## Functional Path

Useful simplified model:

```text
artifact upload
-> Artifactory persists metadata/content
-> Xray integration path receives the event
-> Xray internal pipeline publishes/consumes queue messages
-> Xray indexer receives the indexing message
-> downstream analysis/persist work continues
```

The observed delay before `indexing message received ...` usually points more strongly to:

- Xray-side queueing
- Xray server/router handoff
- RabbitMQ pressure
- indexer readiness, replica count, or resource pressure

and less strongly to Artifactory request-thread tuning by itself.

## Most Relevant Xray Settings

### Artifactory Endpoint for Xray

Xray needs the Artifactory platform URL:

- [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L187)
- [xray/files/system.yaml](/Users/lilstew/Downloads/research/xray/files/system.yaml#L15)

This is the first thing to verify when correlation is unclear:

- correct URL
- reachable URL
- no proxy/routing mismatch

### RabbitMQ

Xray uses RabbitMQ internally for message flow:

- [xray/files/system.yaml](/Users/lilstew/Downloads/research/xray/files/system.yaml#L26)
- [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L485)

If indexing messages arrive minutes late, RabbitMQ backlog or consumer throughput is a first-class suspect.

### Indexer

Defaults in the imported chart:

- `indexer.replicaCount: 1`
- `indexer.autoscaling.enabled: false`
- queue targets include `indexRegular`, `indexRegularRetry-0`, `indexExistsContent`

References:

- [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1398)
- [xray/templates/services/indexer/indexer-deployment.yaml](/Users/lilstew/Downloads/research/xray/templates/services/indexer/indexer-deployment.yaml)

Operational reading:

- one indexer replica is a simple default, not a guarantee of low queue latency
- if indexing demand spikes, backlog can build before the indexer logs message receipt

### Analysis

Defaults in the imported chart:

- `analysis.replicaCount: 1`
- `analysis.autoscaling.enabled: false`

References:

- [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L952)

Analysis is downstream of indexing, but saturation here can still slow total pipeline turnover and contribute to queue buildup.

### Persist

Defaults in the imported chart:

- `persist.replicaCount: 1`
- `persist.autoscaling.enabled: false`

References:

- [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1506)

Persist pressure can also reduce end-to-end throughput and keep queues elevated.

### Split Deployment Mode

The chart supports separating Xray services into dedicated deployments:

- [xray/values.yaml](/Users/lilstew/Downloads/research/xray/values.yaml#L1998)

Important default:

- `splitXraytoSeparateDeployments.fullSplit: false`

Operational reading:

- without full split, service-level scaling isolation is more limited
- for indexing-specific bottlenecks, split mode can make indexer-specific tuning easier

## First Correlation Checklist

When upload-to-indexing delay is measured in minutes, check first:

1. Xray server, router, and indexer logs around the same artifact and timestamp.
2. RabbitMQ queue depth and consumer lag for index-related queues.
3. Whether `indexer` is still single-replica and non-autoscaled.
4. CPU, memory, restart, and readiness behavior on `indexer`, `analysis`, and `persist`.
5. Whether Artifactory-side delay is actually before event publication, or only after Xray receives work.

## Practical Hypothesis

For the specific symptom:

- upload happens now
- `indexing message received for artifact ...` appears minutes later

the strongest first hypothesis is:

- the event is not being consumed quickly enough by Xray's queue-driven indexing path

before assuming:

- Artifactory Tomcat thread shortage
- generic outbound HTTP pool shortage in Artifactory

Those can still matter indirectly, but they are not the first explanation for this exact symptom pattern.
