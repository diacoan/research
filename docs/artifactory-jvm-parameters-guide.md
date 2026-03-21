# Artifactory JVM Parameters Guide

## Purpose

Explain the JVM-related parameters used in the local `artifactoy-ha` Artifactory configuration and answer, for each parameter:

- what it means in generic JVM or Java process terms
- how it influences JVM or Java application behavior
- what generic tuning guidance usually applies
- what it means specifically for Artifactory as a platform

## Scope

This document covers the parameters that are actually present in the local Artifactory chart configuration:

- [artifactoy-ha/files/system.yaml](../artifactoy-ha/files/system.yaml)
- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- [artifactoy-ha/sizing/artifactory-xsmall-extra-config.yaml](../artifactoy-ha/sizing/artifactory-xsmall-extra-config.yaml)
- [artifactoy-ha/sizing/artifactory-small-extra-config.yaml](../artifactoy-ha/sizing/artifactory-small-extra-config.yaml)
- [artifactoy-ha/sizing/artifactory-medium-extra-config.yaml](../artifactoy-ha/sizing/artifactory-medium-extra-config.yaml)
- [artifactoy-ha/sizing/artifactory-large-extra-config.yaml](../artifactoy-ha/sizing/artifactory-large-extra-config.yaml)
- [artifactoy-ha/sizing/artifactory-xlarge-extra-config.yaml](../artifactoy-ha/sizing/artifactory-xlarge-extra-config.yaml)
- [artifactoy-ha/sizing/artifactory-2xlarge-extra-config.yaml](../artifactoy-ha/sizing/artifactory-2xlarge-extra-config.yaml)

It focuses on the Artifactory JVM process, not on the Access JVM tuning block under `access.javaOpts`.

## Important Distinction

Not every parameter below is a "JVM tuning flag" in the strict sense.

There are three categories:

1. Standard JVM heap and native-memory flags:
   - `-Xms`
   - `-Xmx`
   - `-XX:InitialRAMPercentage`
   - `-XX:MaxRAMPercentage`
   - `-XX:MaxMetaspaceSize`
   - `-XX:CompressedClassSpaceSize`
   - `-XX:MaxDirectMemorySize`
   - `-Djdk.nio.maxCachedBufferSize`
2. Standard Java management and JMX properties:
   - `-Dcom.sun.management.*`
   - `-Djava.rmi.server.hostname`
3. Artifactory-specific Java system properties:
   - `-Dartifactory.*`

For category 3, the JVM only passes the property into the Java process. The JVM itself does not interpret the property. Artifactory does.

## Source Confidence

This document uses three evidence levels:

- `Official JVM/JMX documentation`: Oracle Java documentation or OpenJDK primary sources
- `Official JFrog documentation`: JFrog blog, reference architecture, or public best-practice material
- `Operational inference`: interpretation from the chart, parameter name, and platform behavior when no public product documentation was found

Whenever a property is Artifactory-specific and not clearly documented in public JFrog sources, it is marked as `Operational inference`.

## Parameter Inventory in This Repo

### Parameters Rendered Directly by `system.yaml`

- `-Dartifactory.graceful.shutdown.max.request.duration.millis`
- `-Dartifactory.access.client.max.connections`
- `-Dartifactory.async.corePoolSize`
- `-Xms`
- `-Xmx`
- `-Dcom.sun.management.jmxremote`
- `-Dcom.sun.management.jmxremote.port`
- `-Dcom.sun.management.jmxremote.rmi.port`
- `-Dcom.sun.management.jmxremote.ssl`
- `-Djava.rmi.server.hostname`
- `-Dcom.sun.management.jmxremote.authenticate`
- `-Dcom.sun.management.jmxremote.access.file`
- `-Dcom.sun.management.jmxremote.password.file`

### Parameters Added by Sizing Overlays

- `-XX:InitialRAMPercentage`
- `-XX:MaxRAMPercentage`
- `-Dartifactory.async.poolMaxQueueSize`
- `-Dartifactory.http.client.max.total.connections`
- `-Dartifactory.http.client.max.connections.per.route`
- `-Dartifactory.metadata.event.operator.threads`
- `-XX:MaxMetaspaceSize`
- `-XX:CompressedClassSpaceSize` for `xsmall`
- `-Djdk.nio.maxCachedBufferSize`
- `-XX:MaxDirectMemorySize`

## Quick Interpretation

Use the parameters in the following logical groups:

- heap sizing:
  - `-Xms`
  - `-Xmx`
  - `-XX:InitialRAMPercentage`
  - `-XX:MaxRAMPercentage`
- native memory protection:
  - `-XX:MaxMetaspaceSize`
  - `-XX:CompressedClassSpaceSize`
  - `-XX:MaxDirectMemorySize`
  - `-Djdk.nio.maxCachedBufferSize`
- observability and remote JVM access:
  - JMX and RMI properties
- Artifactory concurrency and back-pressure:
  - `-Dartifactory.async.corePoolSize`
  - `-Dartifactory.async.poolMaxQueueSize`
  - `-Dartifactory.http.client.max.total.connections`
  - `-Dartifactory.http.client.max.connections.per.route`
  - `-Dartifactory.access.client.max.connections`
  - `-Dartifactory.metadata.event.operator.threads`
- lifecycle behavior:
  - `-Dartifactory.graceful.shutdown.max.request.duration.millis`

## Generic JVM Heap Parameters

### `-Xms`

#### Generic JVM Meaning

`-Xms` sets the minimum and initial Java heap size.

Official reference:

- Oracle `java` command:
  - <https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html>

#### JVM / Java Application Impact

- startup heap size
- lower bound for heap growth
- affects how quickly the JVM needs to resize the heap under load

#### Generic Tuning Recommendations

Generic recommendation, based on JVM behavior:

- use `-Xms` when you want predictable startup and more stable GC behavior
- in server applications, setting `-Xms` close to `-Xmx` reduces heap resizing during runtime
- do not set it so high that it steals memory needed for native memory, direct buffers, threads, or filesystem cache

This recommendation is operational inference based on the Oracle semantics of `-Xms` and common server tuning practice.

#### Meaning for Artifactory

For Artifactory, `-Xms` matters when:

- startup performance matters
- the node is expected to run at stable sustained load
- you want fewer heap-resize transitions under traffic spikes

Operationally:

- too low -> more heap growth during startup and early workload ramp-up
- too high -> less container headroom for metaspace, direct memory, NIO caches, thread stacks, and sidecars

### `-Xmx`

#### Generic JVM Meaning

`-Xmx` sets the maximum Java heap size.

Official reference:

- Oracle `java` command:
  - <https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html>

#### JVM / Java Application Impact

- upper bound for Java object heap
- large influence on GC frequency and pause behavior
- too low can produce heap pressure and `OutOfMemoryError: Java heap space`

#### Generic Tuning Recommendations

Generic recommendation:

- set `-Xmx` based on live-set size, allocation rate, GC behavior, and container memory
- leave non-heap headroom for metaspace, direct memory, code cache, thread stacks, native libs, and file cache
- do not equate container memory with safe `-Xmx`

This is operational inference grounded in standard JVM memory layout and Oracle heap semantics.

#### Meaning for Artifactory

For Artifactory, `-Xmx` is one of the primary capacity controls because the platform is:

- long-lived
- metadata-heavy
- cache-heavy
- often under sustained concurrent load

Operationally:

- too low -> GC churn, low free heap, latency inflation, eventual heap OOM
- too high -> container-level memory pressure, especially when combined with sidecars and direct buffer usage

## Container-Aware Heap Percentage Parameters

### `-XX:InitialRAMPercentage`

#### Generic JVM Meaning

Sets the initial Java heap as a percentage of the maximum memory available to the JVM process.

Official reference:

- Oracle `java` command:
  - <https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html>

#### JVM / Java Application Impact

- gives the JVM a container-aware way to choose initial heap size
- useful when the same image runs under different memory limits

#### Generic Tuning Recommendations

Generic recommendation:

- use RAM-percentage flags when you want one image to adapt to multiple container sizes
- keep them aligned with actual pod memory limits
- avoid mixing percentage-based and explicit fixed-heap strategies unless the result is intentional and tested

The last point is operational inference. The JVM docs define the flags, but not a deployment policy for mixed sizing models.

#### Meaning for Artifactory

In this chart, sizing overlays use `InitialRAMPercentage` instead of mandatory fixed `-Xms`.

That means Artifactory startup heap is intended to scale with pod memory shape rather than one static number.

Operationally:

- good for standardized sizing presets across multiple environments
- risky if pod memory limits are loose, inconsistent, or absent

### `-XX:MaxRAMPercentage`

#### Generic JVM Meaning

Sets the maximum Java heap as a percentage of the maximum memory available to the JVM process.

Official reference:

- Oracle `java` command:
  - <https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html>

#### JVM / Java Application Impact

- container-aware heap ceiling
- affects how much process memory is allocated to heap before the JVM applies ergonomics

#### Generic Tuning Recommendations

Generic recommendation:

- this is useful in containerized deployments when fixed `-Xmx` is not preferred
- do not treat it as the whole memory budget; you still need headroom for non-heap and native memory
- if the application uses significant direct buffers or metaspace, the percentage should be conservative enough to leave room

#### Meaning for Artifactory

For Artifactory, this is especially important because the process also needs room for:

- metaspace
- direct buffers
- NIO caches
- threads
- internal HTTP pools
- sidecars in the same pod

Operationally:

- too high -> container OOM risk even when heap itself looks healthy
- too low -> unnecessary GC pressure and reduced throughput headroom

## Native Memory and Metadata Parameters

### `-XX:MaxMetaspaceSize`

#### Generic JVM Meaning

Sets an upper bound for native memory used for class metadata.

Official reference:

- Oracle GC tuning guide:
  - <https://docs.oracle.com/en/java/javase/11/gctuning/other-considerations.html>

#### JVM / Java Application Impact

- caps metadata memory for loaded classes and related structures
- too low can trigger class-metadata pressure and eventually `OutOfMemoryError: Metaspace`

#### Generic Tuning Recommendations

Generic recommendation:

- leave it unset unless you want a hard safety bound or you know the application’s class-loading profile
- if you set it, monitor for metaspace growth and class-loader churn
- too aggressive a cap can force more GC pressure around class unloading or fail under plugin-heavy or dynamically loaded applications

#### Meaning for Artifactory

For Artifactory, this mostly acts as a guardrail:

- it protects the pod from unlimited metaspace expansion
- it forces a conscious non-heap budget

Operationally:

- too low -> startup or runtime failure if metadata and loaded class footprint exceeds the cap
- sensible cap -> better predictability in a container budget

### `-XX:CompressedClassSpaceSize`

#### Generic JVM Meaning

Sets the reserved space used for compressed class pointers when compressed class pointers are enabled.

Official reference:

- Oracle GC tuning guide:
  - <https://docs.oracle.com/en/java/javase/11/gctuning/other-considerations.html>

#### JVM / Java Application Impact

- affects one sub-region related to class metadata
- not usually the first tuning lever in ordinary server tuning

#### Generic Tuning Recommendations

Generic recommendation:

- usually leave the default unless you have a very constrained memory budget or a specific metadata profile
- tune only together with understanding of metaspace and class-loading behavior

#### Meaning for Artifactory

In this repo, `CompressedClassSpaceSize` appears only in the `xsmall` sizing overlay.

That suggests a defensive low-footprint preset rather than a primary throughput tuning strategy.

Operationally:

- if class metadata footprint is normal, this may be harmless
- if the environment is unusually class-heavy, too small a value can create avoidable metadata pressure

### `-XX:MaxDirectMemorySize`

#### Generic JVM Meaning

Sets the maximum total size of `java.nio` direct-buffer allocations.

Official reference:

- Oracle `java` command:
  - <https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html>

#### JVM / Java Application Impact

- caps one major category of native memory outside the heap
- too low can limit I/O efficiency or cause direct-buffer allocation failure
- too high can let the process consume too much native memory

#### Generic Tuning Recommendations

Generic recommendation:

- tune this when the application uses significant NIO or off-heap I/O buffers
- consider it together with heap, metaspace, and container memory limit
- if you see direct-memory pressure, raise it only if the pod still has native-memory headroom

#### Meaning for Artifactory

This is highly relevant for Artifactory because the platform handles:

- HTTP traffic
- streaming
- repository I/O
- remote repository access

Operationally:

- too low -> direct-buffer pressure, possible off-heap bottlenecks, reduced I/O efficiency
- too high -> native memory steals headroom from the rest of the pod

### `-Djdk.nio.maxCachedBufferSize`

#### Generic JVM Meaning

This system property limits the size of temporary direct buffers that can be retained in the NIO temporary buffer cache.

Primary reference:

- OpenJDK release note:
  - <https://bugs.openjdk.org/browse/JDK-8175230>

#### JVM / Java Application Impact

- reduces the chance that one unusually large I/O operation leaves a very large direct buffer cached
- if set too low, may increase allocation/free churn for larger transient buffers

#### Generic Tuning Recommendations

Generic recommendation:

- useful for I/O-heavy server applications where rare large buffers would otherwise stay cached
- use it when you want tighter control of off-heap retention, not as the first lever for normal heap tuning

#### Meaning for Artifactory

For Artifactory, this is an off-heap hygiene control:

- it helps bound temporary direct-buffer cache retention
- it complements `MaxDirectMemorySize`

Operationally:

- lower values favor tighter off-heap control
- higher values favor reuse and lower allocation churn

## JMX and Management Parameters

### `-Dcom.sun.management.jmxremote`

#### Generic JVM Meaning

Enables the ready-to-use JMX management agent.

Official reference:

- Oracle monitoring and management guide:
  - <https://docs.oracle.com/en/java/javase/24/management/monitoring-and-management-using-jmx-technology.html>

#### JVM / Java Application Impact

- exposes JVM and application MBeans
- enables monitoring and management tooling

#### Generic Tuning Recommendations

- enable only when you need remote or structured JVM observability
- in production, pair it with secure authentication and SSL

#### Meaning for Artifactory

For Artifactory, JMX is useful when:

- you need JVM internals
- you need HTTP pool MBeans or GC/MBeans diagnostics
- external monitoring tools integrate via JMX

### `-Dcom.sun.management.jmxremote.port`

#### Generic JVM Meaning

Defines the JMX remote port.

Official reference:

- Oracle monitoring and management guide:
  - <https://docs.oracle.com/en/java/javase/24/management/monitoring-and-management-using-jmx-technology.html>

#### JVM / Java Application Impact

- creates a network endpoint for JMX RMI connections

#### Generic Tuning Recommendations

- use an unused port
- control access with firewalling, authentication, and SSL

#### Meaning for Artifactory

Operationally:

- required if you want remote JMX into the Artifactory JVM
- must be treated as an admin surface, not a public service endpoint

### `-Dcom.sun.management.jmxremote.rmi.port`

#### Generic JVM Meaning

Defines the RMI port used by the JMX connector.

Official reference:

- Oracle monitoring and management guide:
  - <https://docs.oracle.com/en/java/javase/24/management/monitoring-and-management-using-jmx-technology.html>

#### JVM / Java Application Impact

- stabilizes the second port used by JMX RMI
- useful for firewall and NAT traversal

#### Generic Tuning Recommendations

- set it explicitly when remote JMX crosses firewalls, Kubernetes networking controls, or NAT boundaries

#### Meaning for Artifactory

For Kubernetes-operated Artifactory, explicit RMI port configuration avoids the usual remote-JMX networking surprises.

### `-Dcom.sun.management.jmxremote.ssl`

#### Generic JVM Meaning

Controls SSL for remote JMX.

Official reference:

- Oracle monitoring and management guide:
  - <https://docs.oracle.com/en/java/javase/24/management/monitoring-and-management-using-jmx-technology.html>

#### JVM / Java Application Impact

- enables or disables SSL on the remote JMX channel

#### Generic Tuning Recommendations

- production recommendation is to keep remote JMX secured
- disabling SSL is acceptable only in tightly controlled internal environments

#### Meaning for Artifactory

If Artifactory JMX is exposed outside a local trusted path, `ssl=false` is a security risk, not a tuning feature.

### `-Djava.rmi.server.hostname`

#### Generic JVM Meaning

Defines the host or interface address used in exported RMI stubs.

Official reference:

- Oracle monitoring and management guide:
  - <https://docs.oracle.com/en/java/javase/24/management/monitoring-and-management-using-jmx-technology.html>

#### JVM / Java Application Impact

- determines how clients are told to connect back to the RMI endpoint

#### Generic Tuning Recommendations

- set it explicitly when the JVM is behind NAT, load balancers, or multi-interface environments

#### Meaning for Artifactory

In Kubernetes, this matters because:

- pod DNS, service DNS, and external access paths are different
- wrong hostname breaks remote JMX even when the port is open

### `-Dcom.sun.management.jmxremote.authenticate`

#### Generic JVM Meaning

Controls password authentication for remote JMX.

Official reference:

- Oracle monitoring and management guide:
  - <https://docs.oracle.com/en/java/javase/24/management/monitoring-and-management-using-jmx-technology.html>

#### JVM / Java Application Impact

- if `false`, remote monitoring can be unauthenticated

#### Generic Tuning Recommendations

- keep authentication enabled in production
- use `false` only for local or tightly isolated development diagnostics

#### Meaning for Artifactory

For Artifactory in production, `authenticate=false` should be treated as a deliberate exception, not a normal baseline.

### `-Dcom.sun.management.jmxremote.access.file`

#### Generic JVM Meaning

Specifies the JMX access file path.

Official reference:

- Oracle monitoring and management guide:
  - <https://docs.oracle.com/en/java/javase/24/management/monitoring-and-management-using-jmx-technology.html>

#### JVM / Java Application Impact

- controls which authenticated roles may monitor or control the JVM

#### Generic Tuning Recommendations

- use explicit role scoping
- ensure file permissions are correct

#### Meaning for Artifactory

This is part of hardening remote JMX for operational use.

### `-Dcom.sun.management.jmxremote.password.file`

#### Generic JVM Meaning

Specifies the JMX password file path.

Official reference:

- Oracle monitoring and management guide:
  - <https://docs.oracle.com/en/java/javase/24/management/monitoring-and-management-using-jmx-technology.html>

#### JVM / Java Application Impact

- supplies credentials for file-based JMX authentication

#### Generic Tuning Recommendations

- use only with secure file permissions
- avoid weak operational hygiene because the file-based approach stores credentials in clear text

#### Meaning for Artifactory

Operationally useful for controlled JMX access, but it should be treated as a security configuration concern, not a performance tuning knob.

## Artifactory-Specific Java System Properties

### `-Dartifactory.graceful.shutdown.max.request.duration.millis`

#### Generic JVM Meaning

In generic JVM terms, this is just a Java system property passed at startup. The JVM does not interpret it.

Official generic `-D` reference:

- Oracle `java` command:
  - <https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html>

#### JVM / Java Application Impact

- none at pure JVM level
- only the Java application changes behavior if it reads the property

#### Generic Tuning Recommendations

- generic JVM guidance does not apply
- tune only according to the application lifecycle semantics

#### Meaning for Artifactory

`Operational inference`

In this chart, the property is rendered from `artifactory.terminationGracePeriodSeconds`, so it represents how long Artifactory may allow in-flight requests to continue during shutdown.

Operationally:

- too low -> downloads/uploads or long requests can be cut during rollout or pod eviction
- too high -> slower pod turnover and longer maintenance windows

### `-Dartifactory.access.client.max.connections`

#### Generic JVM Meaning

Generic JVM meaning: startup system property only.

#### JVM / Java Application Impact

- JVM does not interpret it
- application uses it if coded to do so

#### Generic Tuning Recommendations

- no generic JVM tuning guidance
- tune only with product behavior in mind

#### Meaning for Artifactory

Official JFrog guidance exists that this controls the internal HTTP connection pool Artifactory uses to interact with Access.

Sources:

- JFrog database best-practices whitepaper
- local chart render path in [artifactoy-ha/files/system.yaml](../artifactoy-ha/files/system.yaml)

Operationally:

- too low -> authn/authz and permission-heavy flows can queue behind internal Access calls
- too high -> more concurrency against Access and its downstream DB

Important local relationship:

- this chart renders it from `access.tomcat.connector.maxThreads`

### `-Dartifactory.async.corePoolSize`

#### Generic JVM Meaning

Generic JVM meaning: startup system property only.

#### JVM / Java Application Impact

- none at JVM level
- application-defined worker-pool behavior changes if Artifactory reads it

#### Generic Tuning Recommendations

- generic JVM guidance does not exist
- in Java applications generally, larger async pools increase concurrency but also contention, context switching, and backend pressure

#### Meaning for Artifactory

`Operational inference`

The property name strongly suggests core worker-pool concurrency for internal asynchronous Artifactory tasks.

Operationally:

- too low -> async backlogs, slower background progress, delayed internal work
- too high -> more concurrent pressure on CPU, DB, storage, and other internal dependencies

### `-Dartifactory.async.poolMaxQueueSize`

#### Generic JVM Meaning

Generic JVM meaning: startup system property only.

#### JVM / Java Application Impact

- none at JVM level
- queue sizing only matters if the application uses the property

#### Generic Tuning Recommendations

- in generic Java concurrency design, a larger queue trades rejection for latency and memory retention
- too large a queue can hide overload and increase memory use
- too small a queue can create back-pressure too early

#### Meaning for Artifactory

`Operational inference`

This likely caps the queue behind Artifactory async workers.

Operationally:

- too low -> background work may reject or stall sooner
- too high -> backlog can accumulate silently and consume memory while hiding saturation

### `-Dartifactory.http.client.max.total.connections`

#### Generic JVM Meaning

Generic JVM meaning: startup system property only.

#### JVM / Java Application Impact

- no JVM behavior change by itself
- application HTTP client behavior changes if Artifactory reads it

#### Generic Tuning Recommendations

- in generic Java HTTP client pooling, total connection caps should reflect real upstream capacity and the application’s concurrency profile
- increasing the cap helps only if upstream systems and the rest of the app can use the extra concurrency productively

#### Meaning for Artifactory

JFrog performance guidance documents this as an Artifactory HTTP connection tuning area for remote-dependent work.

Operationally for Artifactory:

- too low -> remote waits, queued outbound work, longer request lifetime
- too high -> amplified waste against broken remotes or overloaded proxies/upstreams

This property is central for remote repositories and outbound integrations.

### `-Dartifactory.http.client.max.connections.per.route`

#### Generic JVM Meaning

Generic JVM meaning: startup system property only.

#### JVM / Java Application Impact

- no JVM behavior change directly
- application per-route HTTP pooling changes if Artifactory reads it

#### Generic Tuning Recommendations

- in generic client pooling, per-route caps matter when many requests converge on one target host or proxy
- tune it together with total connection cap, not in isolation

#### Meaning for Artifactory

For Artifactory, this is especially relevant when:

- many remote repositories point at the same upstream host
- a shared enterprise proxy sits in front of many remotes

Operationally:

- too low -> one hot upstream becomes a bottleneck even if total outbound capacity looks sufficient
- too high -> one upstream can absorb too much concurrency and create localized overload

### `-Dartifactory.metadata.event.operator.threads`

#### Generic JVM Meaning

Generic JVM meaning: startup system property only.

#### JVM / Java Application Impact

- no direct JVM effect

#### Generic Tuning Recommendations

- generic JVM guidance does not exist
- for application worker-thread counts, more threads increase concurrency only if downstream dependencies can keep up

#### Meaning for Artifactory

`Operational inference`

The property name strongly suggests concurrency for metadata-event processing.

Operationally:

- too low -> slower event processing, backlog in metadata-related internal flows
- too high -> extra DB or internal coordination pressure

This likely matters more in metadata-heavy or event-heavy environments than in simple binary-serving workloads.

## Parameter Relationships That Matter Most

### Heap vs Native Memory

The following must be budgeted together:

- `-Xmx` or `-XX:MaxRAMPercentage`
- `-XX:MaxMetaspaceSize`
- `-XX:MaxDirectMemorySize`
- `-Djdk.nio.maxCachedBufferSize`

Practical rule:

- if you increase heap aggressively without adjusting the native-memory budget, the pod can still OOM outside the heap

### Fixed Heap vs Percentage Heap

The chart supports both:

- explicit heap sizing through `-Xms` / `-Xmx`
- adaptive container-relative sizing through `InitialRAMPercentage` / `MaxRAMPercentage`

Practical rule:

- choose one primary sizing model per deployment and keep it intentional

### Artifactory Request Concurrency vs Internal Pools

These should be thought about together:

- `artifactory.tomcat.connector.maxThreads`
- `-Dartifactory.access.client.max.connections`
- `-Dartifactory.http.client.max.total.connections`
- `-Dartifactory.http.client.max.connections.per.route`
- `-Dartifactory.async.corePoolSize`
- `-Dartifactory.metadata.event.operator.threads`

Practical rule:

- raising one concurrency dimension alone often just moves the bottleneck elsewhere

## Generic Tuning Strategy for Artifactory

Use this order:

1. decide heap model:
   - fixed `-Xms` / `-Xmx`
   - or container-relative `InitialRAMPercentage` / `MaxRAMPercentage`
2. reserve native-memory headroom:
   - metaspace
   - direct memory
   - temporary NIO cache
3. secure or disable JMX intentionally
4. tune Artifactory-specific concurrency only after identifying the dominant bottleneck:
   - Access
   - outbound HTTP
   - background async work
   - metadata event processing

## What Is Publicly Documented vs Inferred

Publicly documented in official JVM/JMX docs:

- `-Xms`
- `-Xmx`
- `-XX:InitialRAMPercentage`
- `-XX:MaxRAMPercentage`
- `-XX:MaxMetaspaceSize`
- `-XX:CompressedClassSpaceSize`
- `-XX:MaxDirectMemorySize`
- `-Dcom.sun.management.*`
- `-Djava.rmi.server.hostname`
- generic `-Dproperty=value`

Documented in primary OpenJDK sources:

- `-Djdk.nio.maxCachedBufferSize`

Documented by JFrog or strongly corroborated in JFrog public material:

- `-Dartifactory.http.client.max.total.connections`
- `-Dartifactory.access.client.max.connections`

Operational inference from local chart and parameter names:

- `-Dartifactory.graceful.shutdown.max.request.duration.millis`
- `-Dartifactory.async.corePoolSize`
- `-Dartifactory.async.poolMaxQueueSize`
- `-Dartifactory.http.client.max.connections.per.route`
- `-Dartifactory.metadata.event.operator.threads`

## References

Local repo references:

- [artifactoy-ha/files/system.yaml](../artifactoy-ha/files/system.yaml)
- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)
- [artifactoy-ha/sizing/artifactory-xsmall-extra-config.yaml](../artifactoy-ha/sizing/artifactory-xsmall-extra-config.yaml)
- [artifactoy-ha/sizing/artifactory-large-extra-config.yaml](../artifactoy-ha/sizing/artifactory-large-extra-config.yaml)

Official JVM/JMX references:

- Oracle `java` command:
  - <https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html>
- Oracle monitoring and management guide:
  - <https://docs.oracle.com/en/java/javase/24/management/monitoring-and-management-using-jmx-technology.html>
- Oracle GC tuning guide, class metadata:
  - <https://docs.oracle.com/en/java/javase/11/gctuning/other-considerations.html>
- OpenJDK release note for `jdk.nio.maxCachedBufferSize`:
  - <https://bugs.openjdk.org/browse/JDK-8175230>

Official / primary JFrog references:

- monitoring and optimizing Artifactory performance:
  - <https://jfrog.com/blog/monitoring-and-optimizing-artifactory-performance/>
- Artifactory database best practices:
  - <https://media.jfrog.com/wp-content/uploads/2020/03/07194222/Best-Practices-for-Managing-Your-Artifactory-Database-JFrog.pdf>
