# Cloud SQL PostgreSQL for Artifactory Playbook

## Purpose

Provide a complete playbook for sizing, tuning, monitoring, and decision making for a Google Cloud SQL for PostgreSQL instance used by Artifactory.

This playbook is intended for environments where:

- Artifactory runs on Kubernetes or VM infrastructure
- PostgreSQL is external to the Helm chart
- Cloud SQL is the metadata and transactional state backend
- Artifactory replicas share the same database tier

Use it together with:

- [artifactory-observability-quick-check.md](./artifactory-observability-quick-check.md)
- [cloudsql-proxy-sidecar-for-artifactory-playbook.md](./cloudsql-proxy-sidecar-for-artifactory-playbook.md)
- [high-level-artifactory-capacity-interdependencies.md](./high-level-artifactory-capacity-interdependencies.md)
- [gke-cluster-resource-assessment-playbook.md](./gke-cluster-resource-assessment-playbook.md)

## Problem Statement

For Artifactory, Cloud SQL is not just a persistence layer. It is one of the main progress gates for the platform.

If Cloud SQL is undersized, misconfigured, or poorly monitored:

- request latency rises
- active requests accumulate
- DB pools saturate
- replica scaling stops helping
- Access and metadata-sensitive flows become slower

At the same time, a Cloud SQL instance can look healthy at a coarse level while still being the dominant bottleneck through:

- connection saturation
- lock waits
- memory pressure
- storage growth
- query-level contention
- failover or maintenance behavior

## Stability, Performance, and Functionality Impact

Cloud SQL problems affect Artifactory in three distinct ways:

- Stability impact:
  - instance restart
  - connection failures
  - failover disruption
  - write unavailability if storage is exhausted
- Performance impact:
  - higher DB wait
  - slower metadata operations
  - queueing behind app-side connection pools
  - latency amplification across the platform
- Functionality impact:
  - login and permission flows degrade
  - artifact operations slow down
  - background work falls behind
  - HA replicas remain up but fail to improve useful throughput

## Artifactory Context

Artifactory stores binaries in the filestore and metadata in the relational database.

For Cloud SQL PostgreSQL, that means:

- object payloads are not the main database content
- metadata, repository state, permissions, properties, indexing state, and platform coordination remain DB-sensitive
- scaling Artifactory replicas increases the concurrency presented to the same database backend

Official JFrog storage and database guidance:

- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/database/>
- <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/>

## Local Chart Mapping

Relevant local chart values:

- external DB mode:
  - `postgresql.enabled=false`
  - `database.type=postgresql`
  - `database.driver=org.postgresql.Driver`
  - `database.url`
  - `database.user`
  - `database.password`
- Artifactory-side pool:
  - `artifactory.database.maxOpenConnections`
- Access-side pool:
  - `access.database.maxOpenConnections`
- Metadata-side pool:
  - `metadata.database.maxOpenConnections`
- request concurrency that influences DB demand:
  - `artifactory.tomcat.connector.maxThreads`
  - `access.tomcat.connector.maxThreads`
  - `artifactory.primary.replicaCount`

Relevant local sources:

- [artifactoy-ha/files/system.yaml](../artifactoy-ha/files/system.yaml)
- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)

## Working Principles

Use these principles throughout the playbook:

- Cloud SQL is a shared dependency across Artifactory replicas
- DB pool size is not the same as DB capacity
- more app replicas can increase DB contention without increasing useful throughput
- memory, connections, storage, and query behavior must be read together
- application-side connection pooling and Cloud SQL-side connection limits must be aligned

## Sub-Topic 1: Cloud SQL Instance Shape and HA Baseline

### Purpose

Define the database service shape before tuning application pools or reading metrics.

### Problem Statement

If you do not know the Cloud SQL edition, machine type, storage type, storage size, and HA mode, later tuning conclusions will be weak or misleading.

### Stability, Performance, and Functionality Impact

If the baseline is misunderstood:

- storage headroom is misread
- failover expectations are wrong
- memory and CPU limits are misestimated
- app-side pool tuning may exceed real DB capability

### Concept / Explanation

At minimum, record:

- edition
- machine series / machine type
- vCPU and memory
- storage type
- storage size
- automatic storage increase
- regional HA or standalone
- read replicas if any

### Functionality

This defines the supply-side envelope of the database tier.

### Metrics / Parameters

Important instance settings:

- machine type
- memory
- cores
- storage capacity
- storage auto increase
- region and zone placement
- HA enabled or not

Meaning:

- machine and memory define primary compute headroom
- storage type and size define I/O and growth envelope
- HA defines failover behavior, not free additional write throughput

### What / How to Verify

Official commands:

```bash
gcloud sql instances list --project=PROJECT_ID

gcloud sql instances describe INSTANCE_NAME \
  --project=PROJECT_ID

gcloud sql operations list \
  --instance=INSTANCE_NAME \
  --project=PROJECT_ID \
  --limit=20
```

Key things to read from `describe`:

- `settings.tier`
- `settings.dataDiskType`
- `settings.dataDiskSizeGb`
- `settings.storageAutoResize`
- `settings.availabilityType`
- `settings.databaseFlags`
- `region`

### Potential Problems

- assuming HA means zero-impact failover
- assuming storage growth is reversible
- tuning app pools without knowing whether the instance is memory-first or CPU-first constrained

### Decision Making Guide

- If the instance is standalone, failure risk is higher and app-side concurrency increases should be more conservative.
- If storage auto increase is disabled, disk utilization thresholds deserve higher priority.
- If the instance is HA, treat failover as recovery protection, not as a throughput multiplier.

### Official References

- instance settings:
  - <https://cloud.google.com/sql/docs/postgres/instance-settings>
- create instances:
  - <https://cloud.google.com/sql/docs/postgres/create-instance>
- high availability:
  - <https://cloud.google.com/sql/docs/postgres/high-availability>
- `gcloud sql instances describe`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/sql/instances/describe>
- `gcloud sql instances list`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/sql/instances/list>
- `gcloud sql operations list`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/sql/operations/list>

## Sub-Topic 2: Application Pool Sizing vs Cloud SQL Connection Capacity

### Purpose

Align Artifactory connection pools with actual Cloud SQL connection headroom.

### Problem Statement

Many performance problems are created by increasing app-side connection pools or request concurrency before validating DB-side headroom.

### Stability, Performance, and Functionality Impact

If the app asks for more connections than Cloud SQL can serve efficiently:

- DB waits increase
- CPU may remain moderate while latency rises
- connections accumulate in waiting states
- lock contention can worsen

### Concept / Explanation

There are three layers to correlate:

1. Artifactory-side DB pools
2. Cloud SQL `max_connections` and actual backend usage
3. request concurrency that generates DB demand

### Functionality

This section determines whether the DB is a hard concurrency bottleneck.

### Metrics / Parameters

Artifactory-side parameters:

- `artifactory.database.maxOpenConnections`
- `access.database.maxOpenConnections`
- `metadata.database.maxOpenConnections`
- `artifactory.tomcat.connector.maxThreads`
- `access.tomcat.connector.maxThreads`

Cloud SQL-side parameter:

- PostgreSQL `max_connections`

Cloud SQL monitoring metrics:

- `cloudsql.googleapis.com/database/postgresql/num_backends`
- `cloudsql.googleapis.com/database/postgresql/num_backends_by_application`
- `cloudsql.googleapis.com/database/postgresql/num_backends_by_state`
- `cloudsql.googleapis.com/database/network/connection_attempt_count`
- `cloudsql.googleapis.com/database/postgresql/new_connection_count`

Meaning:

- `num_backends` is actual PostgreSQL connection count
- `num_backends_by_state` shows whether connections are active, idle, or idle in transaction
- `new_connection_count` shows connection churn
- connection attempts show whether clients are repeatedly reconnecting or failing

Correlation:

- high `num_backends` plus app-pool saturation usually means DB concurrency pressure is real
- high `idle_in_transaction` is a strong sign of application or query-pattern inefficiency
- high connection churn with moderate pool size suggests unstable connection lifecycle or reconnect storms

### What / How to Verify

Read Cloud SQL flags:

```bash
gcloud sql instances describe INSTANCE_NAME \
  --project=PROJECT_ID \
  --format="yaml(settings.databaseFlags)"
```

If needed, list PostgreSQL settings in-session:

```sql
SELECT name, setting
FROM pg_settings
WHERE name IN ('max_connections');
```

In Metrics Explorer, chart:

- `cloudsql.googleapis.com/database/postgresql/num_backends`
- `cloudsql.googleapis.com/database/postgresql/num_backends_by_state`
- `cloudsql.googleapis.com/database/postgresql/new_connection_count`

### Potential Problems

- raising `maxOpenConnections` in the app because requests are slow
- high idle sessions wasting memory
- high connection churn due to poor pool behavior

### Decision Making Guide

- If app pools are close to their ceiling but Cloud SQL still has connection headroom, app-side pool tuning may help.
- If Cloud SQL `num_backends` is already high and memory is tight, do not raise app pools first.
- If idle or idle-in-transaction sessions are high, fix connection lifecycle or query behavior before resizing the instance.

### Official References

- manage database connections:
  - <https://cloud.google.com/sql/docs/postgres/manage-connections>
- configure flags:
  - <https://cloud.google.com/sql/docs/postgres/flags>
- Cloud SQL metrics:
  - <https://cloud.google.com/sql/docs/postgres/admin-api/metrics>

## Sub-Topic 3: CPU, Memory, and Query Pressure

### Purpose

Determine whether Cloud SQL is limited by compute, memory, or query behavior.

### Problem Statement

Cloud SQL can look slow for very different reasons:

- high CPU from query work
- high memory pressure
- high lock wait
- high I/O time
- poor cache hit behavior

### Stability, Performance, and Functionality Impact

These are primary throughput and latency drivers for Artifactory metadata operations.

### Concept / Explanation

Use Cloud SQL metrics and Query Insights together:

- system-level metrics explain instance pressure
- query-level metrics explain who causes it

### Functionality

This section helps distinguish instance underprovisioning from query inefficiency.

### Metrics / Parameters

Core Cloud SQL metrics:

- `cloudsql.googleapis.com/database/cpu/utilization`
- `cloudsql.googleapis.com/database/cpu/usage_time`
- `cloudsql.googleapis.com/database/memory/utilization`
- `cloudsql.googleapis.com/database/memory/total_usage`
- `cloudsql.googleapis.com/database/memory/components`
- `cloudsql.googleapis.com/database/postgresql/backends_in_wait`

Query Insights metrics:

- `cloudsql.googleapis.com/database/postgresql/insights/aggregate/execution_time`
- `cloudsql.googleapis.com/database/postgresql/insights/aggregate/latencies`
- `cloudsql.googleapis.com/database/postgresql/insights/aggregate/io_time`
- `cloudsql.googleapis.com/database/postgresql/insights/aggregate/lock_time`
- `cloudsql.googleapis.com/database/postgresql/insights/perquery/execution_time`
- `cloudsql.googleapis.com/database/postgresql/insights/perquery/latencies`
- `cloudsql.googleapis.com/database/postgresql/insights/perquery/io_time`
- `cloudsql.googleapis.com/database/postgresql/insights/perquery/lock_time`

Meaning:

- CPU utilization shows compute saturation
- memory utilization shows RAM pressure
- memory components split usage, cache, and free percentages
- query insights metrics show where execution, lock, and I/O time accumulate

Correlation:

- high CPU + high execution time -> compute-bound queries or insufficient cores
- high lock time + moderate CPU -> contention, not compute shortage
- high IO time + disk activity -> storage-sensitive queries or flush pressure
- high memory utilization with low free/cache safety margin -> OOM risk or memory underprovisioning

### What / How to Verify

In Metrics Explorer, chart:

- `database/cpu/utilization`
- `database/memory/utilization`
- `database/memory/components`
- `database/postgresql/backends_in_wait`
- `database/postgresql/insights/aggregate/latencies`
- `database/postgresql/insights/perquery/latencies`

Use Query Insights in console for normalized query analysis.

### Potential Problems

- high memory with high `work_mem` and high connection count
- high lock waits from concurrent metadata-heavy behavior
- high CPU from unindexed or inefficient queries

### Decision Making Guide

- If CPU is consistently high, scale cores or reduce expensive query work.
- If memory utilization is high and free/cache safety margin is low, prioritize memory or flag tuning before raising connection counts.
- If lock time dominates, instance resizing alone may not solve the issue.

### Official References

- Cloud SQL metrics:
  - <https://cloud.google.com/sql/docs/postgres/admin-api/metrics>
- Query Insights:
  - <https://cloud.google.com/sql/docs/postgres/using-query-insights>
- optimize high memory usage:
  - <https://cloud.google.com/sql/docs/postgres/optimize-high-memory-usage>
- best practices for memory usage:
  - <https://cloud.google.com/sql/docs/postgres/manage-memory-usage-best-practices>

## Sub-Topic 4: Storage Utilization, WAL, and Temporary Files

### Purpose

Track disk consumption and I/O pressure in the database tier.

### Problem Statement

Cloud SQL disk exhaustion and temp-file growth can destabilize Artifactory even when CPU and memory look acceptable.

### Stability, Performance, and Functionality Impact

- storage full can take the instance offline
- temp-file growth often signals memory or query planning issues
- WAL growth affects recovery, replication, and maintenance behavior

### Concept / Explanation

For Artifactory, storage pressure in Cloud SQL usually reflects:

- metadata growth
- temp work from queries
- WAL activity
- backups or maintenance side effects

### Functionality

This section determines whether disk is the real constraint or merely a symptom.

### Metrics / Parameters

Primary metrics:

- `cloudsql.googleapis.com/database/disk/bytes_used`
- `cloudsql.googleapis.com/database/disk/bytes_used_by_data_type`
- `cloudsql.googleapis.com/database/disk/utilization`
- `cloudsql.googleapis.com/database/disk/read_ops_count`
- `cloudsql.googleapis.com/database/disk/write_ops_count`
- `cloudsql.googleapis.com/database/postgresql/temp_bytes_written_count`
- `cloudsql.googleapis.com/database/postgresql/temp_files_written_count`
- `cloudsql.googleapis.com/database/postgresql/write_ahead_log/flushed_bytes_count`
- `cloudsql.googleapis.com/database/postgresql/write_ahead_log/inserted_bytes_count`
- `cloudsql.googleapis.com/database/postgresql/write_ahead_log/redo_size`

Meaning:

- `disk/utilization` shows how close the instance is to its disk quota
- temp metrics show spill behavior and memory-sensitive query pressure
- WAL metrics show write volume and recovery-related backlog

Correlation:

- high temp bytes with high memory pressure often points to insufficient memory or expensive sorts/hashes
- high WAL volume with heavy write load can amplify replica lag and maintenance time
- high disk utilization with auto-resize disabled is an immediate risk

### What / How to Verify

In Metrics Explorer, chart:

- `database/disk/utilization`
- `database/disk/bytes_used_by_data_type`
- `database/postgresql/temp_bytes_written_count`
- `database/postgresql/temp_files_written_count`
- `database/postgresql/write_ahead_log/inserted_bytes_count`

Review instance settings:

- storage size
- storage auto increase
- storage type

### Potential Problems

- storage auto increase disabled on a fast-growing instance
- temp files growing because query memory settings are too aggressive or too low
- WAL growth associated with long-running replication or recovery backlog

### Decision Making Guide

- If disk utilization is high, act before changing app concurrency.
- If temp-file metrics are high, investigate query shape and memory tuning before only adding storage.
- If WAL metrics are high and replicas lag, review write load, replica health, and maintenance events.

### Official References

- instance settings:
  - <https://cloud.google.com/sql/docs/postgres/instance-settings>
- diagnose issues:
  - <https://cloud.google.com/sql/docs/postgres/diagnose-issues>
- Cloud SQL metrics:
  - <https://cloud.google.com/sql/docs/postgres/admin-api/metrics>

## Sub-Topic 5: Failover, Replication, and Maintenance Behavior

### Purpose

Understand how Cloud SQL HA and replica behavior affect Artifactory availability.

### Problem Statement

Database HA is often misunderstood as a performance feature. For Artifactory, it is primarily an availability and recovery characteristic.

### Stability, Performance, and Functionality Impact

- failover closes active connections
- replica lag can make read scaling misleading
- maintenance or failover behavior can look like application instability

### Concept / Explanation

Cloud SQL HA uses a standby and failover model.

For Artifactory:

- a failover can cause short disruption
- the application must reconnect cleanly
- existing in-flight DB sessions are lost

### Functionality

This section tells you whether platform symptoms are related to DB lifecycle events rather than steady-state sizing.

### Metrics / Parameters

Relevant metrics:

- `cloudsql.googleapis.com/database/available_for_failover`
- `cloudsql.googleapis.com/database/replication/network_lag`
- `cloudsql.googleapis.com/database/replication/replica_lag`
- `cloudsql.googleapis.com/database/postgresql/replication/replica_byte_lag`

Relevant operational sources:

- Cloud SQL operations log
- Cloud Logging

Meaning:

- failover availability tells you whether failover can currently proceed
- replication lag shows whether read replicas or standby synchronization are behind

### What / How to Verify

Commands:

```bash
gcloud sql operations list \
  --instance=INSTANCE_NAME \
  --project=PROJECT_ID \
  --limit=50

gcloud logging read \
  'resource.type="cloudsql_database" AND resource.labels.database_id="INSTANCE_NAME"' \
  --project=PROJECT_ID \
  --limit=50
```

Logs Explorer:

- resource type: `Cloud SQL Database`
- log name examples:
  - `cloudsql.googleapis.com/postgres.log`

### Potential Problems

- interpreting restart or failover symptoms as app-side pool saturation
- assuming HA eliminates all DB downtime
- ignoring replica lag in read-sensitive architectures

### Decision Making Guide

- If failover or restart events line up with incidents, fix lifecycle stability first.
- If replica lag is high, do not assume replicas provide reliable read scaling.
- If failover is unavailable, treat HA posture as degraded.

### Official References

- high availability:
  - <https://cloud.google.com/sql/docs/postgres/high-availability>
- replication lag:
  - <https://cloud.google.com/sql/docs/postgres/replication/replication-lag>
- view instance logs:
  - <https://docs.cloud.google.com/sql/docs/postgres/logging>
- audit logging:
  - <https://cloud.google.com/sql/docs/postgres/audit-logging>

## Sub-Topic 6: Logs, Operations, and SQL-Level Diagnostics

### Purpose

Use logs and SQL introspection to explain why Cloud SQL is slow or unstable.

### Problem Statement

Metrics show that something is wrong; logs and SQL views usually tell you what kind of wrong it is.

### Stability, Performance, and Functionality Impact

Without diagnostic evidence:

- you can scale the instance for the wrong reason
- you can miss connection leaks, lock contention, or query-path errors

### Concept / Explanation

Use three layers:

1. Cloud SQL operations and instance logs
2. Logs Explorer filters
3. SQL-level inspection using PostgreSQL catalog views

### Functionality

This section supports root-cause investigation and evidence-based tuning.

### Metrics / Parameters

Useful SQL views and queries:

- `pg_stat_activity`
- `pg_settings`

Meaning:

- `pg_stat_activity` shows current sessions and states
- `pg_settings` shows active PostgreSQL configuration values

### What / How to Verify

Official command:

```bash
gcloud logging read "resource.type=cloudsql_database" \
  --project=PROJECT_ID \
  --limit=10 \
  --format=json
```

Useful SQL:

```sql
SELECT * FROM pg_stat_activity;
```

```sql
SELECT name, setting FROM pg_settings;
```

### Potential Problems

- enabling too much statement logging and increasing Cloud SQL overhead
- chasing app-side errors while DB logs already show resource exhaustion
- missing idle-in-transaction sessions

### Decision Making Guide

- If logs show connection churn or auth failures, fix connectivity and pool behavior before scaling the instance.
- If `pg_stat_activity` shows many idle or idle-in-transaction sessions, address connection hygiene first.
- If logs show memory pressure or resource exhaustion, prioritize memory and query behavior review.

### Official References

- view logs:
  - <https://docs.cloud.google.com/sql/docs/postgres/logging>
- debug connectivity:
  - <https://cloud.google.com/sql/docs/postgres/debugging-connectivity>
- flags:
  - <https://docs.cloud.google.com/sql/docs/postgres/flags>

## Sub-Topic 7: Decision Framework for Cloud SQL Used by Artifactory

### Purpose

Turn all observations into a structured sizing and tuning decision.

### Problem Statement

Cloud SQL can be slow because of:

- app pool pressure
- instance underprovisioning
- lock contention
- memory-sensitive queries
- disk pressure
- failover or maintenance events

### Stability, Performance, and Functionality Impact

The wrong fix can:

- increase cost without solving latency
- amplify connection pressure
- move the bottleneck into Cloud SQL faster

### Concept / Explanation

Use the following decision order:

1. confirm Cloud SQL instance shape and HA mode
2. compare app pools with actual backend usage
3. inspect CPU, memory, wait, and query insights
4. inspect disk, temp files, and WAL behavior
5. inspect logs and operations
6. only then change instance size, flags, or app-side pools

### Decision Making Guide

- Increase Artifactory DB pools only when Cloud SQL still has proven headroom.
- Increase Cloud SQL compute when CPU and execution-time evidence show compute saturation.
- Increase memory or reduce query memory pressure when memory usage is persistently high or OOM risk is visible.
- Increase storage or enable storage auto increase before the instance approaches disk exhaustion.
- Prefer query or connection cleanup over raw instance upsizing when lock waits, temp spills, or idle transactions dominate.
- If HA events explain the incident, fix availability posture before throughput tuning.

## Interdependencies with Other Artifactory Sub-Topics

Cloud SQL does not operate in isolation.

Key interdependencies:

- more Artifactory primaries -> more DB concurrency pressure
- more Tomcat threads -> more overlapping DB demand
- more Access concurrency -> more authn/authz-related DB load
- remote repository wait can indirectly increase DB overlap by keeping requests alive longer
- GCS filestore health can be good while Cloud SQL remains the dominant metadata bottleneck

Practical rule:

- do not interpret healthy GCS as healthy Artifactory if Cloud SQL is saturated
- do not interpret healthy JVM heap as healthy platform if DB wait dominates

## Complete Execution Flow

1. Record Cloud SQL instance shape and HA mode.
2. Record Artifactory-side DB pool settings.
3. Chart connection counts and connection states.
4. Chart CPU, memory, waits, and Query Insights metrics.
5. Chart disk, temp-file, and WAL metrics.
6. Review logs and operation history.
7. Decide whether the issue is:
   - pool sizing
   - compute sizing
   - memory sizing
   - storage growth
   - query inefficiency
   - lifecycle event
8. Only then apply Cloud SQL or Artifactory-side changes.

## References

Local repo references:

- [artifactory-observability-quick-check.md](./artifactory-observability-quick-check.md)
- [high-level-artifactory-capacity-interdependencies.md](./high-level-artifactory-capacity-interdependencies.md)
- [artifactoy-ha/files/system.yaml](../artifactoy-ha/files/system.yaml)
- [artifactoy-ha/values.yaml](../artifactoy-ha/values.yaml)

Google Cloud references:

- instance settings:
  - <https://cloud.google.com/sql/docs/postgres/instance-settings>
- create instances:
  - <https://cloud.google.com/sql/docs/postgres/create-instance>
- high availability:
  - <https://cloud.google.com/sql/docs/postgres/high-availability>
- diagnose issues:
  - <https://cloud.google.com/sql/docs/postgres/diagnose-issues>
- debug connectivity:
  - <https://cloud.google.com/sql/docs/postgres/debugging-connectivity>
- manage connections:
  - <https://cloud.google.com/sql/docs/postgres/manage-connections>
- configure flags:
  - <https://docs.cloud.google.com/sql/docs/postgres/flags>
- metrics:
  - <https://cloud.google.com/sql/docs/postgres/admin-api/metrics>
- Query Insights:
  - <https://cloud.google.com/sql/docs/postgres/using-query-insights>
- optimize high memory usage:
  - <https://cloud.google.com/sql/docs/postgres/optimize-high-memory-usage>
- operational guidelines:
  - <https://cloud.google.com/sql/docs/postgres/operational-guidelines>
- view instance logs:
  - <https://docs.cloud.google.com/sql/docs/postgres/logging>
- audit logging:
  - <https://cloud.google.com/sql/docs/postgres/audit-logging>
- `gcloud sql instances describe`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/sql/instances/describe>
- `gcloud sql operations list`:
  - <https://docs.cloud.google.com/sdk/gcloud/reference/sql/operations/list>

JFrog references:

- database considerations:
  - <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/database/>
- storage considerations:
  - <https://jfrog.com/reference-architecture/self-managed/deployment/considerations/storage/>
