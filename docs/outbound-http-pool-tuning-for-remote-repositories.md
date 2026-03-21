# Outbound HTTP Pool Tuning for Remote Repositories

## Purpose

This appendix explains how to reason about outbound HTTP pool tuning for remote-repository traffic in Artifactory HA.

It focuses on:

- `-Dartifactory.http.client.max.total.connections`
- `-Dartifactory.http.client.max.connections.per.route`
- their interaction with remote-repository fan-out, request lifetime, and Tomcat thread pressure

Use it together with:

- [high-level-artifactory-capacity-interdependencies.md](./high-level-artifactory-capacity-interdependencies.md)
- [low-level-design-remote-repo-monitoring.md](./low-level-design-remote-repo-monitoring.md)
- [artifactory-observability-quick-check.md](./artifactory-observability-quick-check.md)

## Quick Check

| Signal or pattern | Likely meaning | First action | Tuning implication |
| --- | --- | --- | --- |
| High latency, high active requests, low CPU, low remote success ratio | Requests are waiting on remote work, not progressing | Check remote health and fan-out first | Do not raise Tomcat threads first |
| High tiny/small-object latency across many repos using the same upstream host or proxy | Per-route outbound concurrency is likely the practical choke point | Group by `remote_host` and verify shared upstream behavior | Review `connections.per.route` before changing `maxThreads` |
| High latency with many different healthy remotes and no clear single-host concentration | Aggregate outbound concurrency may be the limiting factor | Verify remote health and upstream capacity | Review `max.total.connections` |
| High `401`, `403`, or transport failures with little success | Invalid demand or broken upstreams | Fix credentials, policy, proxy path, or upstream availability | Do not increase outbound pools to mask the problem |
| High `404` with useful success still present | Virtual-repository fan-out waste | Reduce unnecessary remote lookups | Clean up repository design before pool tuning |
| Throughput does not improve after increasing Tomcat threads | More waiting requests were created, not more useful work | Re-check outbound wait and remote health | Rebalance request concurrency and outbound capacity together |

## What the Parameters Mean

### `-Dartifactory.http.client.max.total.connections`

Operational meaning:

- aggregate outbound HTTP concurrency available to Artifactory for remote-dependent work

Use it to reason about:

- many remote lookups across many upstream hosts
- cluster nodes spending time waiting for outbound slots
- remote-heavy workloads where the aggregate outbound path is the limiting layer

### `-Dartifactory.http.client.max.connections.per.route`

Operational meaning:

- concurrency guard for a single upstream route

Operational inference:

- in practice this matters most when many requests converge on the same upstream host or the same enterprise proxy
- if many logical remote repositories exit through one proxy or one host, `per.route` may become the real bottleneck before `max.total`

### Relationship Between Them

Use this simple rule:

- `max.total.connections` caps aggregate outbound concurrency
- `connections.per.route` caps concentration against one upstream path

Practical interpretation:

- many healthy remotes across many hosts -> `max.total` is more likely to bind first
- many requests to the same host or proxy -> `per.route` is more likely to bind first

This distinction is not explained explicitly in the existing repo docs and should be treated as operational reasoning from the parameter names, workload shape, and chart presets.

## How This Maps to the Helm Chart in This Repo

This chart does not expose dedicated top-level Helm keys for these two HTTP client properties.

In this repo, they are passed as Java options, typically through sizing overlays:

- [artifactory-xsmall-extra-config.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/sizing/artifactory-xsmall-extra-config.yaml#L13)
- [artifactory-small-extra-config.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/sizing/artifactory-small-extra-config.yaml#L13)
- [artifactory-medium-extra-config.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/sizing/artifactory-medium-extra-config.yaml#L13)
- [artifactory-large-extra-config.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/sizing/artifactory-large-extra-config.yaml#L13)
- [artifactory-xlarge-extra-config.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/sizing/artifactory-xlarge-extra-config.yaml#L13)
- [artifactory-2xlarge-extra-config.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/sizing/artifactory-2xlarge-extra-config.yaml#L13)

The generated `system.yaml` pulls Java options from:

- [system.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/files/system.yaml#L13)

Relevant override paths:

- [values.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/values.yaml#L69) for `systemYamlOverride`
- [values.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/values.yaml#L667) for `artifactory.extraSystemYaml`

Practical note:

- the sizing files are examples, not magic defaults
- if `systemYamlOverride` replaces the generated file, sizing-overlay assumptions may no longer apply

## Existing Repo Baselines

The repo currently ships example outbound pool values such as:

- `xsmall`: `20 / 20`
- `small`: `50 / 50`
- `medium`: `50 / 50`
- `large`: `100 / 100`
- `xlarge`: `150 / 150`
- `2xlarge`: `150 / 150`

Operational implication:

- this repo keeps `max.total` and `per.route` equal in the supplied overlays
- that is a reasonable simple baseline, but it does not distinguish between:
  - many upstream hosts
  - one shared proxy or one hot upstream

## How to Measure Before Tuning

### Direct Signal

JFrog’s performance guidance calls out the `HTTPConnectionPool` MBean as the direct place to inspect Artifactory HTTP connection usage.

Use it when available to see whether usage is persistently close to the configured limit.

Relevant JFrog source:

- <https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/>

### Indirect Signals Already Modeled in This Repo

If direct pool telemetry is not already wired into Prometheus, use:

- `tiny_object_latency_ms_p95`
- `small_object_latency_ms_p95`
- low remote success ratio
- rising timeout rate
- high active requests with low CPU
- grouping failures by `remote_host`

These are weaker than direct pool measurements, but still operationally useful.

## Tuning Order

1. Confirm the dominant symptom is remote wait, not DB, Access, storage, or credentials.
2. Remove invalid demand first:
   - broken remotes
   - bad credentials
   - `403` policy issues
   - excessive `404` fan-out
3. Determine whether the likely limit is aggregate or per-route:
   - one shared host or proxy -> review `connections.per.route`
   - many healthy hosts -> review `max.total.connections`
4. Verify that upstream systems and proxies can sustain more concurrency.
5. Increase conservatively and observe latency, success ratio, and direct pool usage if available.
6. Only after remote wait decreases should you revisit Tomcat thread counts.

## Cluster-Level Fine Point

In full-primary HA, these limits apply per Artifactory node.

Operational inference:

- increasing outbound pool size on one node increases cluster-wide outbound pressure only in proportion to active nodes
- a `3`-node full-primary cluster with `50` outbound connections per node can create materially more upstream pressure than a single-node deployment with the same per-node value

This matters especially when:

- many nodes target the same upstream host
- all traffic exits through one proxy
- upstream rate limits or connection quotas exist

## Anti-Patterns

Avoid:

- increasing outbound pool capacity while broken remotes remain in the virtual path
- increasing Tomcat threads first when remote wait is dominant
- assuming all latency is CPU starvation
- assuming all `404` is repo failure
- increasing per-node outbound capacity without thinking about cluster-wide upstream pressure

## Short Operational Rule

Increase outbound HTTP pool capacity only when all are true:

- the workload is legitimately remote-heavy
- the remotes are mostly healthy
- fan-out waste has already been reduced
- upstream systems can absorb more concurrency
- the current bottleneck is actually outbound wait

## References

Local references:

- [high-level-artifactory-capacity-interdependencies.md](./high-level-artifactory-capacity-interdependencies.md)
- [low-level-design-remote-repo-monitoring.md](./low-level-design-remote-repo-monitoring.md)
- [artifactory-observability-quick-check.md](./artifactory-observability-quick-check.md)
- [system.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/files/system.yaml#L13)
- [values.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/values.yaml#L69)
- [values.yaml](/Users/lilstew/Downloads/artifactory-ha/artifactoy-ha/values.yaml#L667)

JFrog references:

- <https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/sizing/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/high-availability/>
