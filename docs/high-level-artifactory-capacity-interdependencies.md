# High Level Design: Artifactory Capacity Interdependencies

## Purpose

Describe the high-level relationship between the main capacity controls in Artifactory HA and explain how they influence each other under load.

This document is intentionally broader than remote repository monitoring. Remote repositories are one important workload category, but not the only one.

References:

- [JFrog Platform Reference Architecture](/Users/lilstew/Downloads/artifactory-ha/docs/high-level-artifactory-capacity-interdependencies.md)

## Scope

This document focuses on the interaction between:

- Tomcat request threads
- database connection pools
- Access service capacity
- outbound HTTP connection pools
- request lifetime and queue formation

This document does not focus on:

- a single package type such as Maven
- only remote repositories
- logging or observability implementation details

References:

- [High Availability](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/high-availability/)
- [Runtime Platform](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/runtime-platform/)

## Core Principle

Artifactory throughput is not determined by a single configuration parameter.

Real throughput is bounded by the slowest constrained resource on the active request path.

At a high level:

```text
effective_throughput
  ~= min(
       request_execution_capacity,
       db_progress_capacity,
       access_progress_capacity,
       outbound_progress_capacity,
       storage_progress_capacity
     )
```

This is not a vendor formula. It is a system model for reasoning about bottlenecks.

This model is aligned with JFrog’s official guidance that:

- HA capacity is achieved by clustered platform services with horizontal scalability
- the storage and database deployments are shared across cluster members
- performance depends heavily on Tomcat thread handling, database connections, and storage behavior

The formula above remains an engineering approximation, not an official JFrog sizing equation.

References:

- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)
- [High Availability](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/high-availability/)

## Main Capacity Domains

### 1. Request Execution Capacity

The first execution boundary is the number of active request-handling threads.

Representative setting:

- `artifactory.tomcat.connector.maxThreads`

What it controls:

- how many incoming HTTP requests can be actively processed by a node at the same time

What it does not guarantee:

- that all active requests are making useful progress

Under load, a request thread may be:

- executing application logic
- waiting on DB
- waiting on Access
- waiting on remote I/O
- waiting on storage or network I/O

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)
- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)

### 2. Database Progress Capacity

The second boundary is the ability to progress metadata and state operations.

Representative settings:

- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`

What it controls:

- concurrent DB work for Artifactory and Access

What it influences:

- metadata lookups
- repository state
- locks
- permission-related state access
- internal service workflows

References:

- [Storage](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/)
- [Best Practices for Managing Your Artifactory Database](https://jfrog.com/fr/whitepaper/best-practices-for-managing-your-artifactory-database/)
- [Security](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/security/)

### 3. Access Service Capacity

Access is a dependency service for authentication, authorization, and platform-level coordination.

Representative settings:

- `access.tomcat.connector.maxThreads`
- `-Dartifactory.access.client.max.connections`

What it controls:

- how much authn/authz-related work can progress concurrently

What it influences:

- permission-heavy requests
- token-related flows
- platform coordination paths that depend on Access

References:

- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)
- [Security](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/security/)

### 4. Outbound Progress Capacity

Some requests need outbound I/O. Remote repositories are one example, but not the only one.

Representative settings:

- `-Dartifactory.http.client.max.total.connections`
- `-Dartifactory.http.client.max.connections.per.route`

What it controls:

- how much outbound HTTP-dependent work can progress concurrently

This matters strongly when workloads depend on external systems, upstream repositories, or high-latency network paths.

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)
- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)

### 5. Storage Progress Capacity

The request path is also influenced by underlying storage performance.

Examples:

- local disk IOPS
- network storage latency
- object storage latency
- cache efficiency

JFrog’s reference architecture explicitly states that:

- Artifactory manages binaries in a filestore and their metadata in a relational database
- high-performance local storage is essential for `cache-fs`
- object storage is the recommended filestore option
- NFS is not recommended for high-load environments

Even when threads and pools are sized generously, storage latency can still dominate request lifetime.

References:

- [Storage](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/)
- [AWS Deployment Guidance](https://jfrog.com/reference-architecture/self-managed/deployment/aws/)
- [GCP Deployment Guidance](https://jfrog.com/reference-architecture/self-managed/deployment/gcp/)
- [Best Practices for Managing Your Artifactory Filestore](https://jfrog.com/de/whitepaper/best-practices-for-managing-your-artifactory-filestore-2/)

### 6. Auxiliary and Platform Services

In addition to the main request path resources, several platform services influence overall capacity and stability:

- Frontend
- Metadata
- Mission Control
- Observability

These services do not all participate equally in every request path.

The strongest vendor-supported capacity model remains centered on:

- Artifactory nodes
- shared database
- shared filestore
- cluster horizontal scaling

The service notes below should therefore be read as operational reasoning for self-managed platform deployments, not as official JFrog sizing formulas.

References:

- [JFrog Platform Reference Architecture](https://jfrog.com/reference-architecture/)
- [Observability](https://jfrog.com/reference-architecture/observability/)

## Request Lifetime Model

A useful high-level model is:

```text
request_lifetime
  = cpu_time
  + db_wait
  + access_wait
  + outbound_wait
  + storage_wait
```

If any wait component rises, the total request lifetime rises.

When request lifetime rises:

- active requests accumulate
- request threads remain occupied longer
- queue pressure increases
- downstream pools see more overlap and contention

This is the main feedback loop in Artifactory under load.

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)
- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)

## Queue Formation Model

Another useful approximation is:

```text
active_requests = progressing_requests + waiting_requests
```

In a healthy system:

- a large share of active requests are progressing

In a degraded system:

- a large share of active requests are only waiting on constrained dependencies

That is why high concurrency does not automatically mean high throughput.

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)
- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)

## Interdependencies

### Tomcat Threads vs DB Pool

If request concurrency rises but DB progress capacity does not:

- more requests remain active while waiting for DB work
- latency rises
- more threads are occupied by waiting rather than progress

Increasing only request threads can therefore increase pressure without improving throughput.

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)
- [Best Practices for Managing Your Artifactory Database](https://jfrog.com/fr/whitepaper/best-practices-for-managing-your-artifactory-database/)

### Tomcat Threads vs Access Capacity

If request concurrency rises but Access cannot keep up:

- permission and token-related paths become slower
- request lifetime increases
- the node appears slower even if CPU is not saturated

References:

- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)
- [Security](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/security/)

### Tomcat Threads vs Outbound Capacity

If requests increasingly depend on outbound I/O:

- only a subset can make outbound progress at once
- other requests stay active but wait longer
- request lifetime rises

This is why high request-thread count alone does not imply high external-resolution throughput.

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)
- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)

### DB Pool vs Outbound or Access Delays

Even when DB is not the first bottleneck, long request lifetime increases overlap between requests.

That means:

- more requests are alive at the same time
- more DB connections are demanded over overlapping windows
- DB pressure rises indirectly

References:

- [Best Practices for Managing Your Artifactory Database](https://jfrog.com/fr/whitepaper/best-practices-for-managing-your-artifactory-database/)
- [Storage](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/)

### Access vs Outbound or DB Delays

The same indirect amplification happens for Access:

- longer request lifetime increases concurrent dependency on Access
- Access saturation can appear as part of a wider platform slowdown

References:

- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)
- [High Availability](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/high-availability/)

### Metadata vs DB and Request Lifetime

JFrog documents that Artifactory stores binary metadata in the relational database, and also describes metadata as a core Artifactory capability. That supports the conservative conclusion that metadata-heavy flows increase dependence on DB progress and internal processing capacity.

If metadata work becomes slower or heavier:

- DB pressure rises
- request lifetime may increase for metadata-sensitive operations
- overlap between in-flight requests increases

This makes metadata-related work an amplifier of platform load, especially in environments with large artifact volumes or metadata-heavy workflows.

References:

- [Storage](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/)
- [Understanding Your Artifact Repository Ecosystem](https://jfrog.com/blog/navigating-the-artifact-jungle/)

### Frontend vs Request Execution Capacity

In platform deployments where a separate frontend component is present, it should generally be treated as a user-facing layer that depends on healthy downstream services.

If Artifactory, Access, or router paths are slow:

- UI actions become slower
- users experience the platform as degraded even if the bottleneck is elsewhere

Frontend should therefore usually be treated as a consumer and reflector of platform degradation rather than the first cause of artifact throughput issues.

References:

- [JFrog Platform Reference Architecture](https://jfrog.com/reference-architecture/)
- [Runtime Platform](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/runtime-platform/)

### Mission Control vs Control Plane Stability

Mission Control should not be treated as part of the core modern capacity model for the JFrog Platform.

JFrog has publicly announced that Mission Control is sunset and not supported after the end of Q2 2025.

If an environment still uses Mission Control for legacy operational reasons, it is more relevant to control-plane and administrative workflows than to the core data path of every artifact request.

Under platform stress:

- administrative operations may become slower
- management visibility may degrade

Mission Control should therefore be considered optional legacy context, not a primary artifact throughput tuning lever.

References:

- [Mission Control Sunset FAQ](https://jfrog.com/blog/mission-control-sunset-faq/)
- [JFrog DevOps Platform Announcement with Sunset Update](https://jfrog.com/blog/jfrog-devops-platform/)

### Observability vs Shared Resource Headroom

JFrog’s observability guidance for self-hosted deployments explicitly recommends:

- application metrics collection
- integration with Prometheus and Grafana
- log aggregation using tools such as Loki, Fluentd, or Filebeat

Observability does not usually sit directly in the business request path, but it consumes shared resources:

- CPU
- memory
- disk I/O
- network I/O

If observability is over-configured:

- log volume increases
- metric emission overhead increases
- platform headroom decreases

If observability is under-configured:

- bottlenecks become harder to diagnose
- tuning becomes less precise

Observability must therefore be treated as a balancing act: enough signal to explain platform behavior, but not so much overhead that it materially worsens the same behavior.

References:

- [Observability](https://jfrog.com/reference-architecture/observability/)

## Practical Tuning Rule

Do not tune one layer in isolation unless you are certain it is the dominant bottleneck.

A safer tuning sequence is:

1. identify the dominant source of wait
2. remove invalid or wasteful demand
3. verify downstream headroom
4. only then raise concurrency-related settings

References:

- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)
- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)

## Symptom Interpretation

### Symptom: high concurrency, low CPU, high latency

Likely meaning:

- requests are waiting, not progressing

Possible sources:

- DB contention
- Access contention
- outbound waits
- storage waits

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)

### Symptom: high latency with backlog growth

Likely meaning:

- request lifetime is increasing faster than requests can complete

Possible sources:

- under-sized pools
- slow dependencies
- retries amplifying demand

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)

### Symptom: increasing active requests with no throughput gain

Likely meaning:

- concurrency has been increased beyond real downstream capacity

Possible sources:

- too many request threads relative to DB, Access, storage, or outbound limits

References:

- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)

### Symptom: UI feels slow but artifact throughput is the main concern

Likely meaning:

- Frontend is reflecting degradation in Artifactory, Access, router, DB, or outbound paths

Possible sources:

- request thread saturation
- Access slowness
- DB slowness
- storage or outbound waits

References:

- [JFrog Platform Reference Architecture](https://jfrog.com/reference-architecture/)

### Symptom: metadata-heavy workflows become slow before large downloads do

Likely meaning:

- metadata and DB-related work is becoming the limiting factor

Possible sources:

- DB contention
- metadata processing pressure
- request lifetime inflation in metadata-sensitive flows

References:

- [Storage](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/)
- [Understanding Your Artifact Repository Ecosystem](https://jfrog.com/blog/navigating-the-artifact-jungle/)

### Symptom: platform monitoring itself consumes visible headroom

Likely meaning:

- observability overhead is too high for the current platform shape

Possible sources:

- excessive log volume
- excessive cardinality
- too frequent collection or shipping

References:

- [Observability](https://jfrog.com/reference-architecture/observability/)

## High-Level Adjustment Guidance

### When to increase request threads

Only when:

- the system is CPU-capable
- downstream dependencies have headroom
- requests are not dominated by waiting

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)
- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)

### When to increase DB pool capacity

Only when:

- DB wait is a real bottleneck
- the DB tier can sustain additional concurrency

References:

- [Best Practices for Managing Your Artifactory Database](https://jfrog.com/fr/whitepaper/best-practices-for-managing-your-artifactory-database/)

### When to increase Access capacity

Only when:

- Access is clearly the bottleneck
- Artifactory-side concurrency is already justified

References:

- [Sizing](https://jfrog.com/reference-architecture/self-managed/deployment/sizing/)

### When to increase outbound capacity

Only when:

- outbound-dependent workloads are legitimate and healthy
- remote or upstream endpoints can sustain more concurrency
- the current issue is not poor workload design or wasteful fan-out

References:

- [Monitoring and Optimizing Artifactory Performance](https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/)

### When not to increase anything

Do not increase concurrency-related settings first when the issue is caused by:

- invalid credentials
- broken upstreams
- poor repository design
- high retry behavior
- high storage latency

References:

- [Storage](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/)
- [Observability](https://jfrog.com/reference-architecture/observability/)

### When to review Metadata service capacity

Review Metadata behavior when:

- metadata-heavy workflows slow down
- DB pressure rises without a matching increase in large file transfer volume
- state or metadata operations dominate request time

References:

- [Storage](https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/)
- [Understanding Your Artifact Repository Ecosystem](https://jfrog.com/blog/navigating-the-artifact-jungle/)

### When to review Frontend capacity

Review Frontend primarily when:

- user-facing UI responsiveness matters operationally
- interactive traffic is materially contributing to request concurrency

Do not treat Frontend as the first throughput tuning target unless evidence shows that UI or API gateway behavior is the limiting layer.

References:

- [JFrog Platform Reference Architecture](https://jfrog.com/reference-architecture/)

### When to review Mission Control

Review Mission Control only when:

- the main issue is platform management, topology awareness, or administrative control-plane behavior

Do not treat Mission Control as a primary artifact throughput tuning lever, and note that it is sunset in current JFrog platform direction.

References:

- [Mission Control Sunset FAQ](https://jfrog.com/blog/mission-control-sunset-faq/)

### When to review Observability settings

Review observability settings when:

- logging or metrics overhead is measurable
- high-cardinality labels or excessive log volume are consuming headroom
- platform diagnosis requires more signal than current telemetry provides

References:

- [Observability](https://jfrog.com/reference-architecture/observability/)

## Operational Conclusion

Artifactory should be viewed as a system of coupled queues and constrained dependencies.

The most important high-level relationship is:

- more wait causes longer request lifetime
- longer request lifetime causes more active requests
- more active requests increase contention on every shared pool

This is why tuning should focus on reducing wait and invalid demand first, and increasing concurrency second.

## Related Documents

- [High Level Design: Remote Repository Health Monitoring and Control](/Users/lilstew/Downloads/artifactory-ha/docs/high-level-design-remote-repo-monitoring.md)
- [Low Level Design: Remote Repository Health Monitoring and Control](/Users/lilstew/Downloads/artifactory-ha/docs/low-level-design-remote-repo-monitoring.md)
- [Next Steps: Remote Repository Monitoring](/Users/lilstew/Downloads/artifactory-ha/docs/next-steps-remote-repo-monitoring.md)

## Reference Notes

This document was adjusted against official JFrog sources covering:

- JFrog Platform reference architecture and HA guidance
- storage considerations and sizing guidance
- Artifactory performance guidance for Tomcat threads and DB connections
- Access service role
- self-hosted observability guidance
- Mission Control sunset status
