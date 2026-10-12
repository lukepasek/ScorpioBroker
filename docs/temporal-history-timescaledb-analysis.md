# TimescaleDB for Temporal History Storage

Status: feasibility analysis  
Date: 2026-09-23

## Executive summary

TimescaleDB can be used for ScorpioBroker's temporal attribute history, and the
existing row-per-attribute-instance model is a reasonable starting point. The
clearest immediate benefit is retention: TimescaleDB can remove complete time
chunks instead of issuing large row-by-row `DELETE` operations.

This is not a drop-in extension. The current table cannot be converted directly
because its timestamp columns are nullable and mutable, while its primary and
unique keys do not include a time partition column. A safe implementation should
be introduced as an opt-in storage mode and validated with production-sized data.

## Current persistence model

Temporal entities and their attribute instances are stored in two tables defined
by `HistoryEntityManager/src/main/resources/db/migration/V20190604.3__temporal.sql`:

- `temporalentity` stores the entity identity, types, and entity-level timestamps.
- `temporalentityattrinstance` stores one JSONB document for each historical
  attribute instance.
- Attribute rows contain extracted `createdat`, `modifiedat`, `observedat`, and
  `deletedat` timestamps, plus geometry and identifying columns.
- `internalid BIGSERIAL` is the primary key.
- `(temporalentity_id, attributeid, instanceid)` has a unique index.
- `temporalentity_id` references `temporalentity(id)` with cascading update and
  deletion.

The trigger currently defined by
`HistoryEntityManager/src/main/resources/db/migration/V20231024.1__tempattrsfix.sql`
extracts timestamps and geometry from the JSONB document whenever data is inserted
or changed.

The history query manager:

- Applies temporal predicates directly to `teai.<timeproperty>` in
  `Commons/src/main/java/eu/neclab/ngsildbroker/commons/datatypes/terms/TemporalQueryTerm.java`.
- Supports `observedat`, `createdat`, `modifiedat`, and `deletedat` as temporal
  query properties.
- Orders attribute values with `array_agg(... ORDER BY teai.<timeproperty>)` in
  `HistoryQueryManager/src/main/java/eu/neclab/ngsildbroker/historyquerymanager/repository/HistoryDAO.java`.
- Has covering indexes for ascending and descending observed, created, and
  modified timestamps in
  `HistoryEntityManager/src/main/resources/db/migration/V20250731.1__entitymap.sql`.

No automatic retention policy currently removes old temporal attribute history.

## Compatibility findings

### Non-null partition key

A TimescaleDB time dimension cannot contain null values. None of the existing
semantic timestamps is universally non-null. For example, an attribute deletion
marker may contain `deletedat` without `observedat`.

Using `observedat` directly would therefore require a fallback value and a
backfill policy.

### Unique constraints

Every unique index on a hypertable must include all partition columns. The
current primary key on `internalid` and unique index on
`(temporalentity_id, attributeid, instanceid)` do not include time, so TimescaleDB
will reject a direct conversion.

Adding the partition timestamp to the business key weakens the existing global
instance-ID uniqueness: the same instance ID could then occur in separate chunks.
If that invariant must remain database-enforced, a regular lookup table is needed.

### Mutable timestamps

`HistoryEntityManager/src/main/java/eu/neclab/ngsildbroker/historyentitymanager/repository/HistoryDAO.java`
updates an existing instance by merging JSONB into `data`. The extraction trigger
can consequently change an existing semantic timestamp.

TimescaleDB does not support an update that moves a row from one time chunk to
another. The partition value must therefore be immutable, or such updates must be
implemented as delete-plus-insert operations.

### Foreign keys

Current TimescaleDB supports a foreign key from a hypertable to a regular
PostgreSQL table. The existing reference from `temporalentityattrinstance` to
`temporalentity`, including its cascade behavior, is not by itself a blocker.

Foreign keys between two hypertables have additional restrictions, but
`temporalentity` does not need to become a hypertable.

### Parent-row cleanup

A retention policy drops old attribute chunks but does not remove corresponding
rows from `temporalentity`. A separate cleanup operation is required for entity
rows that no longer have any retained attribute instances.

### Multi-tenancy

ScorpioBroker creates a separate PostgreSQL database for each tenant and applies
Flyway migrations through
`Commons/src/main/java/eu/neclab/ngsildbroker/commons/storage/ConnectionManager.java`.

TimescaleDB must be installed at the PostgreSQL server level and enabled and
migrated in:

- The default `ngb` database.
- Every existing tenant database.
- Every tenant database created after the feature is enabled.

### Deployment image

The Compose definitions currently use `postgis/postgis`. TimescaleDB support
requires a PostgreSQL image containing compatible PostgreSQL, PostGIS, and
TimescaleDB versions, or a maintained custom image that installs both extensions.

## Recommended design

### Partition timestamp

Add an immutable physical ingestion timestamp:

```sql
recordedat TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
```

Use `recordedat` as the hypertable partition dimension and never recalculate it
when an attribute instance is updated. This makes retention mean "time since
ScorpioBroker stored the observation" and avoids cross-chunk updates.

If retention must instead use semantic event age, add an immutable `historytime`
populated once at insertion:

```sql
COALESCE(observedat, modifiedat, createdat, deletedat, clock_timestamp())
```

That value must remain unchanged even if the JSON-LD temporal properties are
subsequently updated.

### Keys and lookup

A candidate hypertable layout is:

```sql
PRIMARY KEY (internalid, recordedat)

CREATE UNIQUE INDEX ... ON temporalentityattrinstance (
    temporalentity_id,
    attributeid,
    instanceid,
    recordedat
);
```

To preserve uniqueness and efficiently locate mutable instances without already
knowing their chunk, add a regular lookup table keyed by
`(temporalentity_id, attributeid, instanceid)` that stores `internalid` and
`recordedat`.

### Retention

After conversion, retention can be configured with a policy such as:

```sql
SELECT add_retention_policy(
    'temporalentityattrinstance',
    INTERVAL '365 days'
);
```

TimescaleDB drops a chunk only when the complete chunk falls outside the retention
window. The chunk interval therefore determines both deletion granularity and the
number of physical partitions. It should be selected from actual ingestion volume;
one to seven days is a reasonable starting range for measurement, not a universal
default.

Add a separate scheduled cleanup for `temporalentity` rows with no remaining
attribute instances.

### Indexes and columnstore

Initially keep the existing secondary indexes needed by NGSI-LD queries and add
the hypertable without columnstore conversion. Reassess the index set after query
plans are measured because every index is materialized across chunks.

Columnstore can significantly reduce storage and accelerate scans, but it should
be a second phase because ScorpioBroker permits updates and deletes of historical
instances and uses JSONB, GIN, and PostGIS/GiST access paths.

## Expected impact

### Retention and deletion

This is the strongest use case. Dropping complete chunks is substantially cheaper
than deleting millions of rows and avoids the table bloat and vacuum work caused
by large PostgreSQL deletes.

### Query performance

Hypertable partition pruning only applies when a query constrains the partition
column. Partitioning by `recordedat` does not automatically accelerate a query
that only filters on `observedat`.

Because NGSI-LD supports several time properties, no single hypertable dimension
optimizes every temporal query. Existing indexes remain important. Chunk skipping,
columnstore ordering, and a semantic partition key should be considered only after
representative query plans have been benchmarked.

### Aggregation

TimescaleDB time bucketing and continuous aggregates may eventually simplify or
accelerate temporal aggregations. They should be evaluated separately so the
existing NGSI-LD aggregation semantics and JSON-LD response shape remain unchanged.

## Proposed rollout

1. Build or select a pinned PostgreSQL image containing compatible PostGIS and
   TimescaleDB releases.
2. Create a shadow hypertable rather than converting the production table in
   place.
3. Backfill representative production-scale temporal history.
4. Compare `EXPLAIN (ANALYZE, BUFFERS)` for before, after, between, `lastN`,
   aggregation, geospatial, and dataset-ID queries.
5. Test instance updates, attribute deletion, entity cascade deletion, late-arriving
   observations, retention, and parent cleanup across chunk boundaries.
6. Test startup and tenant creation with and without TimescaleDB enabled.
7. Introduce an opt-in Flyway migration and configuration only after compatibility
   and performance targets are met.
8. Consider columnstore and continuous aggregates as later, independently measured
   improvements.

Converting a populated table with `migrate_data => true` can take substantial locks
and migration time. A shadow-table migration with controlled cutover is preferable
for large deployments.

## Proof-of-concept acceptance criteria

- Existing temporal API behavior and ordering remain unchanged.
- Instance uniqueness is preserved across chunks.
- Updates cannot accidentally move rows between chunks.
- Range-query plans demonstrate chunk pruning or another measurable improvement.
- Retention removes expired chunks without deleting data inside the configured
  window.
- Empty `temporalentity` rows are cleaned up safely.
- PostGIS temporal queries return identical results.
- New tenant databases receive the same TimescaleDB schema and policies.
- Plain PostgreSQL operation remains available when the feature is disabled.

## References

- [TimescaleDB hypertable limitations](https://www.tigerdata.com/docs/use-timescale/latest/limitations)
- [TimescaleDB unique indexes](https://www.tigerdata.com/docs/use-timescale/latest/hypertables/hypertables-and-unique-indexes)
- [TimescaleDB constraints and foreign keys](https://www.tigerdata.com/docs/use-timescale/latest/schema-management/about-constraints)
- [TimescaleDB retention policies](https://www.tigerdata.com/docs/use-timescale/latest/data-retention/create-a-retention-policy)
- [TimescaleDB hypertable API](https://www.tigerdata.com/docs/api/latest/hypertable/create_hypertable)
