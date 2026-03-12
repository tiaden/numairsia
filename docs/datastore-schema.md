# Numairsia Authoritative Schema

Last update : March 2026

**Governing principle:**
> A stream is the stable logical time series. A device is the movable and replaceable hardware. Bindings connect them over time.

---

## Technology Stack

| Layer | Choice | Rationale |
| --- | --- | --- |
| Table format | Apache Iceberg | Open standard, S3-native, schema evolution, partition evolution, multi-engine |
| Object storage | Ceph S3 | First-class Iceberg backend; no separate database for data |
| Catalog | Apache Polaris (Iceberg REST Catalog), backed by PostgreSQL | Open standard HTTP catalog API; every engine (PyIceberg, DuckDB, Trino, StarRocks) connects to the same endpoint identically |
| Primary write path | PyIceberg (Python) | Runs on a server; direct Arrow → Parquet → Iceberg |
| Ad-hoc queries | DuckDB | Zero-server, reads Iceberg natively |
| Distributed queries | Trino / StarRocks / Spark / Dremio | When query concurrency or BI dashboards are required |
| Large correction campaigns | PyIceberg or Spark on Iceberg branches | Governed correction workflows run on isolated branches and are validated before promotion |

---

## Timestamp Convention and UTC Operations Rule

All temporal fields are stored as Iceberg `timestamptz` (timestamp with time zone).
Precision: **microseconds since Unix epoch** (INT64 in Parquet).
All values are UTC.

`timestamptz` is the correct choice for two reasons:

1. **Semantic correctness.** Sensor observations are UTC instants, not wall-clock readings. The type must express what the data means. `timestamp` (no tz) means "a local date-time with no timezone" which is not what these values are.
2. **Engine safety.** `timestamptz` preserves instant semantics at the storage level, but rendering, literal interpretation, and time-binning can still depend on session time zone in some engines. `timestamptz` is therefore the correct type, but it does not eliminate the need for explicit UTC session configuration which is why the UTC-only operations rule below is mandatory.

`timestamptz` also supports native Iceberg partition transforms (`days()`, `hours()`) without an extra derived column.

**UTC-only operations rule.** All system components must operate in UTC without exception:

| Component | Requirement |
| --- | --- |
| Ingestion process (Python) | `TZ=UTC` in process environment |
| Governance CLI (Python) | `TZ=UTC` in process environment |
| Maintenance jobs (Python) | `TZ=UTC` in process environment |
| PostgreSQL catalog connection | `SET TIME ZONE 'UTC'` on every connection |
| DuckDB query sessions | `SET TimeZone = 'UTC'` before any timestamp comparison |
| Trino query sessions | `SET TIME ZONE 'UTC'` |
| StarRocks query sessions | `SET time_zone = '+00:00'` at session start |

Any writer or query session that operates in a local timezone can produce incorrect partition pruning and incorrect time-range query results.

---

## Logical Model

![tables-schema](img/tables-schema.svg)

The observation layer is split into two families of fact tables:

1. **Current tables** (`obs_*_current`) hold exactly one row per logical observation. These are the default query targets for analytics, dashboards, and user-facing retrieval.
2. **History tables** (`obs_*_history`) hold every accepted version of every logical observation. These are the audit and correction-lineage tables.

This split is deliberate. The dominant query pattern is a time-series scan by `stream_id` and `observed_at`. The dominant correction pattern is point mutation of the latest accepted version of a logical observation. One physical table should not be forced to optimize both workloads simultaneously.

---

## Observation Identity Model

Fact tables use three distinct identity concepts. These must not be conflated.

| Concept | Meaning | Stability |
| --- | --- | --- |
| `ingress_record_id` | Identity of the exact upstream record / message / file-row used for replay protection | Stable across retries/replays of the same upstream record |
| `observation_id` | Identity of the logical observation across corrections | Stable across all accepted versions of the same logical observation |
| `observation_version_id` | Identity of a specific stored version of the observation | New for every accepted version |

**Operational rule:** one column must not do multiple jobs.

- `ingress_record_id` is for idempotency and raw provenance.
- `observation_id` is for logical observation identity.
- `observation_version_id` is for version lineage.

Iceberg v3 row lineage does not change this rule. `_row_id` and `_last_updated_sequence_number` are hidden Iceberg lineage fields, not business keys. They may be absent in pre-upgrade history and are therefore not authoritative for domain semantics.

The full operational rules governing how these identity fields interact during ingestion, replay, and correction are defined in Observation Lifecycle.

---

## Tables

### `measurement_types`

Defines what is being measured. Immutable in meaning; if semantics change materially, create a new record.

```text
measurement_type_id  string        NOT NULL   -- UUID, primary key; identifier field
code                 string        NOT NULL   -- stable semantic code, e.g. "air_temperature", "gps_position"
name                 string
value_kind           string        NOT NULL   -- enum: "numeric_scalar" | "numeric_vector" | "text"
dimension_count      int           NOT NULL   -- 1 for scalar, N for vector
dimensions           list<struct<
                       ordinal: int,
                       name: string,          -- e.g. "latitude", "temperature"
                       unit: string           -- nullable (e.g. GPS has no single unit)
                     >>
properties           map<string, string>      -- escape hatch for low-value metadata
```

**Rules:**

- `value_kind` determines which observation fact table to use.
- `dimension_count` must equal `cardinality(dimensions)`.
- `code` must be globally unique.
- For `numeric_scalar`, `dimension_count = 1`.

---

### `stations`

Physical or logical deployment sites.

```text
station_id     string        NOT NULL   -- UUID, primary key; identifier field
station_code   string                   -- short operator-facing code
name           string
description    string
installed_at   timestamptz              -- UTC microseconds
retired_at     timestamptz              -- NULL = active
properties     map<string, string>
```

---

### `devices`

Physical hardware items. Identity survives movement and redeployment.

```text
device_id        string        NOT NULL   -- UUID, primary key; identifier field
manufacturer     string
model            string
serial_number    string                   -- STRING, not numeric; serial numbers can have leading zeros or letters
manufactured_at  timestamptz              -- UTC microseconds
retired_at       timestamptz              -- NULL = not retired
properties       map<string, string>
```

---

### `streams`

Stable logical time series. A stream belongs to exactly one station and one measurement type for its entire life. If either changes, retire the stream and create a new one.

```text
stream_id            string        NOT NULL   -- UUID, primary key; identifier field
station_id           string        NOT NULL   -- FK → stations
measurement_type_id  string        NOT NULL   -- FK → measurement_types
stream_code          string        NOT NULL   -- stable operator-facing code, unique within a station; e.g. "rooftop_north_outdoor_temp"
name                 string                   -- friendly display label
created_at           timestamptz              -- UTC microseconds
retired_at           timestamptz              -- NULL = active
properties           map<string, string>
```

**Rules:**

- `stream_code` must be unique within a `station_id`.
- A stream's `station_id` and `measurement_type_id` are immutable after creation.

---

### `stream_bindings`

Links a device or sensor to a stream for a validity interval. This is the mechanism that models device replacement and movement.

```text
stream_binding_id  string        NOT NULL   -- UUID, primary key; identifier field
stream_id          string        NOT NULL   -- FK → streams
device_id          string        NOT NULL   -- FK → devices
device_output      string                   -- nullable; distinguishes outputs on multi-output hardware
valid_from         timestamptz   NOT NULL   -- UTC microseconds; inclusive
valid_to           timestamptz              -- UTC microseconds; exclusive; NULL = currently active
replacement_reason string                   -- descriptive, not normative
properties         map<string, string>
```

**Rules:**

- `device_output` determines which physical output channel on a device is producing the data on multi-output hardware.
- **Primary invariant (analytically critical):** For a given `stream_id`, binding intervals must not overlap. A stream must have at most one active producer at any time (violating this makes observations ambiguous). Enforced by ingestion validation.
- **Secondary invariant:** For a given `(device_id, device_output)`, binding intervals should not overlap unless deliberate fan-out to multiple streams is intended and explicitly documented. Enforced by governance tooling.
- `valid_to = NULL` means the binding is open (device is currently assigned).
- The active binding for an observation at time T: `valid_from <= T AND (valid_to IS NULL OR valid_to > T)`

---

### `obs_numeric_scalar_current`

Canonical fact table for the latest accepted scalar numeric observations.

**Partitioning:** `days(observed_at)`
**Sort order within partition:** `stream_id ASC, observed_at ASC`

```text
observation_id          string        NOT NULL   -- stable logical identity; identifier field
observation_version_id  string        NOT NULL   -- latest accepted version currently in force
observed_at             timestamptz   NOT NULL   -- UTC microseconds; event time, not ingestion time
stream_id               string        NOT NULL   -- FK → streams (value_kind must be "numeric_scalar")
value                   double        NOT NULL
quality_code            string                   -- e.g. "good", "suspect", "bad", "corrected"
source_system           string                   -- namespace for upstream identities, e.g. "mqtt_gateway_a", "csv_import", "jsonl_import"
ingress_record_id       string                   -- exact upstream record/message/file-row identity; same value on replay
stream_binding_id       string                   -- non-authoritative convenience annotation; see policy below
ingest_batch_id         string                   -- FK → ingest_batches
ingested_at             timestamptz              -- UTC microseconds; when this current row version was written
correction_event_id     string                   -- nullable FK → correction_events (optional, can be removed if corrections tables is absent)
```

**`stream_binding_id` policy:** Written once at ingestion time as a hint to avoid temporal joins in the common case. **Never repaired after the fact.** If a binding interval is later corrected, rows in this table retain their original stale value. Any query requiring accurate provenance must join `stream_bindings` on `(stream_id, observed_at)`.

---

### `obs_numeric_scalar_history`

Append-only history table for scalar numeric observations. Holds every accepted version.

**Partitioning:** `days(observed_at)`
**Sort order within partition:** `stream_id ASC, observed_at ASC`

```text
observation_version_id  string        NOT NULL   -- unique stored version identity
observation_id          string        NOT NULL   -- stable logical identity across corrections
supersedes_version_id   string                   -- nullable; points to the immediately previous accepted version
observed_at             timestamptz   NOT NULL   -- UTC microseconds; event time, not ingestion time
stream_id               string        NOT NULL   -- FK → streams (value_kind must be "numeric_scalar")
value                   double        NOT NULL
quality_code            string
source_system           string
ingress_record_id       string
stream_binding_id       string                   -- non-authoritative convenience annotation; see policy in current table
ingest_batch_id         string
ingested_at             timestamptz
correction_reason       string                   -- nullable; human-readable explanation for replacement/correction
correction_event_id     string                   -- nullable FK → correction_events (optional, can be removed if corrections tables is absent)
```

Do **not** declare Iceberg identifier fields on this table or the other history tables. These are large append-only audit tables. Even though `observation_version_id` is unique by policy, Iceberg identifier fields are not uniqueness constraints, and declaring them here provides little operational value.

---

### `obs_numeric_vector_current`

Canonical fact table for the latest accepted ordered numeric vector observations (e.g. GPS, accelerometer, wind vector).

**Partitioning:** `days(observed_at)`
**Sort order within partition:** `stream_id ASC, observed_at ASC`

```text
observation_id          string        NOT NULL   -- stable logical identity; identifier field
observation_version_id  string        NOT NULL   -- latest accepted version currently in force
observed_at             timestamptz   NOT NULL
stream_id               string        NOT NULL   -- FK → streams (value_kind must be "numeric_vector")
values                  list<double>  NOT NULL   -- ordered per measurement_type.dimensions[].ordinal
quality_code            string
source_system           string
ingress_record_id       string
stream_binding_id       string                   -- non-authoritative; see policy in obs_numeric_scalar_current
ingest_batch_id         string
ingested_at             timestamptz
correction_event_id     string                   -- nullable FK → correction_events (optional, can be removed if corrections tables is absent)
```

---

### `obs_numeric_vector_history`

Append-only history table for ordered numeric vector observations.

**Partitioning:** `days(observed_at)`
**Sort order within partition:** `stream_id ASC, observed_at ASC`

```text
observation_version_id  string        NOT NULL
observation_id          string        NOT NULL
supersedes_version_id   string
observed_at             timestamptz   NOT NULL
stream_id               string        NOT NULL   -- FK → streams (value_kind must be "numeric_vector")
values                  list<double>  NOT NULL   -- ordered per measurement_type.dimensions[].ordinal
quality_code            string
source_system           string
ingress_record_id       string
stream_binding_id       string
ingest_batch_id         string
ingested_at             timestamptz
correction_reason       string
correction_event_id     string                   -- nullable FK → correction_events (optional, can be removed if corrections tables is absent)
```

**Rules:**

- `cardinality(values)` must equal `measurement_type.dimension_count` for the stream's measurement type.
- Dimension semantics (which index means what) live in `measurement_types.dimensions`, not in these fact tables.

---

### `obs_text_current` (optional)

Canonical fact table for the latest accepted text or categorical observations.

**Partitioning:** `days(observed_at)`
**Sort order within partition:** `stream_id ASC, observed_at ASC`

```text
observation_id          string        NOT NULL   -- stable logical identity; identifier field
observation_version_id  string        NOT NULL   -- latest accepted version currently in force
observed_at             timestamptz   NOT NULL
stream_id               string        NOT NULL   -- FK → streams (value_kind must be "text")
value                   string        NOT NULL
quality_code            string
source_system           string
ingress_record_id       string
stream_binding_id       string                   -- non-authoritative; see policy in obs_numeric_scalar_current
ingest_batch_id         string
ingested_at             timestamptz
correction_event_id     string                   -- nullable FK → correction_events (optional, can be removed if corrections tables is absent)
```

---

### `obs_text_history` (optional)

Append-only history table for text or categorical observations.

**Partitioning:** `days(observed_at)`
**Sort order within partition:** `stream_id ASC, observed_at ASC`

```text
observation_version_id  string        NOT NULL
observation_id          string        NOT NULL
supersedes_version_id   string
observed_at             timestamptz   NOT NULL
stream_id               string        NOT NULL   -- FK → streams (value_kind must be "text")
value                   string        NOT NULL
quality_code            string
source_system           string
ingress_record_id       string
stream_binding_id       string
ingest_batch_id         string
ingested_at             timestamptz
correction_reason       string
correction_event_id     string                   -- nullable FK → correction_events (optional, can be removed if corrections tables is absent)
```

---

### `correction_events` (optional)

Audit table for manual corrections, recalibration campaigns, vendor re-issues, and other governed replacement workflows.

```text
correction_event_id   string        NOT NULL   -- UUID, primary key; identifier field
created_at            timestamptz   NOT NULL   -- UTC microseconds
created_by            string                   -- operator, service account, or workflow name
reason                string                   -- human-readable description
method                string                   -- e.g. "manual", "vendor_reissue", "recalibration", "algorithm_backfill"
reference             string                   -- job ID, or external reference
properties            map<string, string>
```

---

### `ingest_batches`

Audit table for ingestion runs. Retained indefinitely; it is the permanent audit log.

```text
ingest_batch_id      string        NOT NULL   -- UUID, primary key; identifier field
started_at           timestamptz              -- UTC microseconds
completed_at         timestamptz              -- NULL if still running or failed
source_description   string                   -- human-readable description of the data source
status               string                   -- "running" | "completed" | "failed"
scalar_row_count     long
vector_row_count     long
text_row_count       long
dropped_row_count    long
properties           map<string, string>
```

---

## Observation Lifecycle: Versioning, Replay, and Corrections

These rules apply to all three observation families (numeric scalar, numeric vector, text).

### Versioning rules

1. `observation_id` identifies the logical observation. It is stable across corrections.
2. `observation_version_id` identifies one accepted stored version. It is new for every accepted version.
3. The current tables contain exactly one row per `observation_id`. They are the controlled mutable serving layer and the correct place for row replacement. Under Iceberg v3, current-table corrections may be implemented using copy-on-write rewrites or merge-on-read row-level changes, depending on engine support and configured write mode. This is an implementation detail. It does not change the logical rule that the current table holds exactly one accepted row per `observation_id`.
4. The history tables are append-only. They contain every accepted version for an `observation_id`. Every accepted version is written once. No prior history row is modified to mark it stale. The correction lineage is expressed through `observation_id`, `observation_version_id`, and `supersedes_version_id`. Iceberg v3 deletion vectors improve the efficiency of row-level maintenance for mutable tables but do not remove the need for a stable append-only audit layer.
5. The first accepted version for an observation has `supersedes_version_id = NULL` in the corresponding history table.
6. `source_system` is required whenever `ingress_record_id` is populated.
7. Rows without `ingress_record_id` cannot be replay-deduplicated by default and must be handled by source-specific policy.

### Observation identity assignment

`observation_id` is the stable identity of the logical observation across corrections. The assignment rule is:

1. **Use a source-provided logical observation key** when the upstream system has one and it is trustworthy.
2. Otherwise use a **governed deterministic key** derived from stable business fields.
3. If neither exists, mint a **platform-assigned `observation_id`** when the observation is first accepted, and require later correction workflows to reference it explicitly.

### Replay detection

Replay protection uses the pair:

```text
(source_system, ingress_record_id)
```

If a source record arrives again with the same `(source_system, ingress_record_id)`, it is treated as a replay of the same upstream record. A replay must not create a new history version and must not alter the current table.

**Important:** this pair identifies the upstream ingress event, not the logical observation.

### Corrections

Corrections create a new accepted version of an existing logical observation. A correction:

- reuses the same `observation_id`
- gets a new `observation_version_id`
- points `supersedes_version_id` at the immediately prior accepted version in the history table
- normally carries a new `(source_system, ingress_record_id)`

After the new version is appended to history, the corresponding current-table row for that `observation_id` is replaced with the new version.

If an upstream system models corrections as resends of the same ingress record, that source requires a source-specific handler; pure replay dedupe is insufficient.

### Replay vs. correction

This distinction is mandatory:

- **Replay:** same upstream ingress event again → same `(source_system, ingress_record_id)` → no new row
- **Correction:** new accepted version of the same logical observation → same `observation_id`, new `observation_version_id`

These are different things and must not be modeled with the same identifier.

### Late observations

Late observations are accepted without a time limit. Iceberg's append model naturally handles out-of-order data. A late record writes to the correct daily partition based on `observed_at`, regardless of when it arrives. `ingested_at` records when the row arrived, enabling audit queries that measure ingestion latency.

### Large correction campaigns

Large correction campaigns (e.g. mass sensor recalibration, or algorithmic backfill) must not be run as ad hoc direct writes to the main production branch.

The operational pattern is:

1. Create a dedicated Iceberg branch from `main` for each affected observation table.
2. Run the correction job against the branch, not against `main`.
3. Validate counts, spot-check values, and run audit queries on the branch.
4. Create a permanent tag on the pre-campaign production snapshot for rollback and audit reference.
5. Promote the validated branch state to production using the engine's supported branch publication mechanism.
6. Record a `correction_event` describing the campaign.

This workflow contains blast radius, preserves a named recovery point, and keeps the campaign auditable. Branch publication is an operational workflow, not an implicit side effect of ordinary ingestion. PyIceberg is suitable for routine governed correction campaigns because it supports branch-scoped writes and snapshot reference management. Spark remains the preferred engine for the largest or highest-risk campaigns, especially when the correction affects many partitions, requires heavy rewrite/compaction work, or depends on the most mature documented branch publication workflow.

### Binding corrections

Binding corrections (fixing a `stream_bindings` interval) are applied directly to the small governance table. Fact-table `stream_binding_id` values are not repaired (see policy in `obs_numeric_scalar_current`).

---

## Iceberg Table Properties

These properties are set at table creation and govern file layout, compression, and pruning effectiveness.

### Observation tables (current and history)

These properties apply to all `obs_*_current` and `obs_*_history` tables.

```text
write.target-file-size-bytes                  = 134217728   -- 128 MB; sized for daily partitions at this scale
write.parquet.compression-codec               = zstd        -- default; stated explicitly to prevent writer default drift
write.metadata.metrics.column.stream_id       = full        -- UUID strings need full min/max for row-group pruning
write.metadata.metrics.column.observation_id  = full
```

### Additional: current-table row-level write mode

The default operational policy for current tables is:

```text
write.delete.mode = copy-on-write
write.update.mode = copy-on-write
write.merge.mode  = copy-on-write
```

This is the conservative default.

When all production engines used by this system support Iceberg v3 deletion vectors well enough for normal operations, the current tables may be switched to:

```text
write.delete.mode = merge-on-read
write.update.mode = merge-on-read
write.merge.mode  = merge-on-read
```

This is an operational tuning change, not a schema change. The purpose of the change is to reduce write amplification on current-table corrections. It does not alter table semantics.

History tables are append-only (see Observation Lifecycle). No row-level write-mode policy is required for normal history-table operation.

Metadata tables (`stations`, `devices`, `streams`, etc.) use default properties. They are small enough that tuning is unnecessary.

**Why 128 MB and not the Iceberg default of 512 MB:** At ~1-10M rows/day compressed with zstd, a daily partition produces roughly 100 to 500 MB of data. A 512 MB target would produce one under-sized file per day after compaction. A 128 MB target produces 1-4 well-sized files per day, enabling meaningful parallelism in Trino/StarRocks without creating too many small files for DuckDB.

**Why only `stream_id` and `observation_id` need full metrics:** Both are UUID-like strings. The `truncate(16)` default applies to string and binary columns. With truncated statistics, row-group skip effectiveness is reduced. The `full` override stores complete min/max bounds, enabling more precise pruning for point lookups and mixed scan + lookup workloads.

---

## Partitioning Rationale

`days(observed_at)` is the right granularity for both current and history tables:

- At ~1-10M rows/day across all streams, one day fits in a few 128 MB Parquet files well-sized for Iceberg.
- Aligns with the dominant query pattern ("show me data for this stream over N days").
- `hours()` creates too many small files overnight. `months()` forces full-month scans for single-day queries.

Sort order `(stream_id ASC, observed_at ASC)` within each partition improves the effectiveness of Parquet row-group pruning: when writers respect this order and row groups are appropriately sized, a query `WHERE stream_id = X AND observed_at BETWEEN t1 AND t2` can skip most row groups in the daily partition via min/max statistics. This avoids per-stream sub-partitioning, which would create thousands of tiny files per day. The benefit depends on writers honoring the sort order and on row-group size; it is not a hard guarantee.

The current tables and the history tables intentionally share the same partitioning and sort strategy. This keeps query semantics consistent and allows operational tooling to reason about both layers the same way.

---

## Schema Evolution Policy

Iceberg tracks columns by **field ID**, not by name. Adding a column assigns a new field ID; renaming preserves the existing field ID. This is why Iceberg schema evolution is safe at the storage level. The following policy governs how this capability is used operationally.

| Operation | Policy |
| -- | --- |
| Add a column | Allowed at any time. Old files expose the new column as NULL. No migration required. |
| Rename a column | Allowed only for wording clarity, not semantic change. Field ID is preserved; old files are unaffected. |
| Remove a column | Allowed, but the name is permanently retired. Never reuse a removed column name with a different meaning. |
| Promote a type | Only Iceberg-permitted promotions (e.g. `int → long`, `float → double`). Others require a migration plan. |
| Change measurement semantics | Create a new `measurement_type` record. Do not mutate the existing one. |
| Change stream identity | Retire the stream and create a new one. Do not mutate `station_id` or `measurement_type_id`. |
| Change observation identity policy | Allowed only by introducing new columns or a migration plan. Do not silently repurpose `ingress_record_id`, `observation_id`, or `observation_version_id`. |
| Change current/history split | Requires an explicit migration plan. The serving-table and audit-table roles are part of the authoritative model. |

The "never reuse a removed name" rule is critical: downstream code (scripts, dashboards, views) often references column names by string. Reusing a name silently changes the meaning of all such references.

---

## Null, NaN, and Infinity Policy (case of ingestion via JSON/JSONL format)

The JSON → Arrow → Parquet → SQL pipeline does not treat IEEE 754 special floating-point values consistently:

- JSON (strict spec) does not define NaN or Infinity as valid number literals.
- Arrow `float64` supports NaN, +Inf, and -Inf.
- Parquet DOUBLE stores IEEE 754 natively, so NaN is a valid value in a Parquet file.
- SQL engines diverge: DuckDB supports NaN in queries; Trino and StarRocks have inconsistent NaN propagation in aggregations.

**Policy:**

- NaN, +Infinity, and -Infinity are **rejected at ingestion**. The ingestion process must validate every numeric value and refuse to write the row entirely if a special float is present.
- A sensor reading that cannot produce a valid finite double is **dropped**, not stored. The associated `ingest_batch` record accounts for the dropped row count. Persistent drop patterns should be investigated at the source.
- `value` is `NOT NULL` on all numeric fact tables. There is no partial-row storage for numeric observations.
- `quality_code` is for valid finite readings with data quality concerns (e.g. `"suspect"`, `"out_of_range"` for in-range but physically implausible values), not a substitute for a missing value.
- Negative zero (`-0.0`) is accepted and treated as zero.

---

## Identifier Fields

Iceberg supports declaring identifier fields on tables. These must be `NOT NULL` and signal to tooling what constitutes row identity. They are **not** enforced uniqueness constraints. Iceberg is not a relational database.

**Set identifier fields on all small metadata tables:**

| Table | Identifier field |
| --- | --- |
| `measurement_types` | `measurement_type_id` |
| `stations` | `station_id` |
| `devices` | `device_id` |
| `streams` | `stream_id` |
| `stream_bindings` | `stream_binding_id` |
| `correction_events` | `correction_event_id` |
| `ingest_batches` | `ingest_batch_id` |

**Set identifier fields on the current tables:**

| Table | Identifier field |
| --- | --- |
| `obs_numeric_scalar_current` | `observation_id` |
| `obs_numeric_vector_current` | `observation_id` |
| `obs_text_current` | `observation_id` |

The current tables truly have one row per logical observation. `observation_id` is therefore the correct row identity signal.

**Do not set identifier fields on the history tables** (`obs_numeric_scalar_history`, `obs_numeric_vector_history`, `obs_text_history`). These are large append-only audit tables. Their correctness depends on ingestion validation and maintenance audits, not on Iceberg identifier declarations.

**Iceberg v3 row lineage note:** Iceberg v3 row lineage fields do not replace these business keys (see Observation Identity Model).

---

## Iceberg Limitations (Non-Enforced Rules)

Iceberg does not enforce relational constraints. The following rules must be enforced by ingestion validation code and periodic audit jobs:

1. Foreign key correctness across tables
2. `stream_code` uniqueness within a station
3. Non-overlap of `stream_bindings` intervals for a given `stream_id` (primary invariant)
4. Non-overlap of `stream_bindings` intervals for a given `(device_id, device_output)` (secondary invariant)
5. Consistency between a stream's `value_kind` and the fact-table family used for its observations
6. `cardinality(values) = dimension_count` for vector observations
7. Immutability of stream `station_id` and `measurement_type_id` after creation
8. Immutability of measurement type semantics after creation
9. Uniqueness of `observation_id` within each current table
10. Uniqueness of `observation_version_id` within each history table
11. Valid `supersedes_version_id` lineage within each history table
12. Consistency of `(source_system, ingress_record_id)` replay handling when those fields are populated
13. Consistency between each current-table row and the latest accepted version in the corresponding history table

---

## Business Rules Summary

> This table is a cross-reference index. The authoritative definition of each rule lives in the section indicated by the "Enforced by" column and in the relevant table DDL. It is not a second source of truth.

| Rule | Invariant class | Enforced by |
| --- | --- | --- |
| Stream belongs to one station for life | Primary | Ingestion guard |
| Stream has one measurement type for life | Primary | Ingestion guard |
| Binding intervals for a stream do not overlap | Primary | Ingestion guard |
| `obs_numeric_scalar_*` rows only for `numeric_scalar` streams | Primary | Ingestion guard |
| `obs_numeric_vector_*` rows only for `numeric_vector` streams | Primary | Ingestion guard |
| `cardinality(values) = dimension_count` | Primary | Ingestion guard |
| NaN, ±Infinity rejected at ingestion | Primary | Ingestion guard |
| One current row per `observation_id` in current tables | Primary | Ingestion + maintenance audit |
| `observation_version_id` unique in history tables | Primary | Ingestion guard |
| Replay dedupe uses `(source_system, ingress_record_id)` when present | Primary | Ingestion guard |
| Current table matches latest accepted history version | Primary | Ingestion + maintenance audit |
| Binding intervals for a `(device_id, device_output)` do not overlap | Secondary | Governance tooling |
| `stream_code` unique within station | Secondary | Ingestion guard |
| `measurement_types.code` globally unique | Secondary | Ingestion guard |
| `stream_binding_id` on fact rows is never repaired | Policy | Documentation |
| All system components operate in UTC | Operational | Deployment config |

---

## Ingestion Architecture

```text
Ingestion and Coordination Server (sensors team's - ingestion process runs here)
        │
        │  [input interface - OPEN QUESTION: see below]
        ▼
  Ingestion Process (Python / PyIceberg)
  - receives incoming data as discrete batches (files, message groups, or transfer payloads)
  - validates stream_id exists and is actively bound
  - determines target fact-table family from stream's value_kind
  - rejects NaN, ±Infinity; drops the row and increments dropped-row counters
  - validates source_system whenever ingress_record_id is present
  - checks (source_system, ingress_record_id) for duplicate replay (if present)
  - determines observation_id:
      * use trusted source logical key when available
      * otherwise derive governed deterministic key
      * otherwise mint platform-assigned observation_id
  - allocates a new observation_version_id for each accepted version
  - accumulates validated rows into Arrow RecordBatches
  - commits each batch as an atomic Iceberg operation (batch size is a tunable parameter)
  - writes stream_binding_id at ingestion time (point-in-time lookup)
  - appends the accepted version to the appropriate history table
  - upserts the corresponding row in the appropriate current table
  - writes ingest_batch record on completion
  - compresses ingested data and writes it to an archive for daily upload to an object store
        │
        │  S3 API (network)
        ▼
Server (Compute Canada)
  - Polaris REST Catalog service (HTTP :8181)
      └── backed by PostgreSQL (catalog state only)
  - Ceph S3 (Parquet data + Iceberg metadata)
```

**Batch commit policy:** Each incoming data batch (a file, a transfer payload, or a configurable row-count threshold) is processed and committed as a single Iceberg operation. The commit frequency is a tunable operational parameter. Higher frequency reduces the data-at-risk window (rows buffered but not yet committed) at the cost of more Iceberg snapshots and metadata overhead. Lower frequency produces fewer, larger commits but increases the data-at-risk window per batch. The maintenance schedule's snapshot expiry job must be calibrated to the chosen commit frequency.

**Catalog connection (PyIceberg):** The ingestion process connects to Polaris using `type="rest"` with the Polaris HTTP endpoint, not `type="sql"`. DuckDB, Trino, and StarRocks each have native REST catalog connectors and connect to the same endpoint. PostgreSQL is internal to Polaris; no consumer talks to it directly.

**Open question:** How does sensor data arrive at the ingestion server's local interface?

Options under consideration, in order of alignment with the batch ingestion model: directory watch on incoming files (inotify), local message queue or broker, Unix socket, localhost TCP, named pipe.  
The preferred approach is file-based delivery (stations write files locally, files are transferred to the ingestion server periodically), which naturally aligns with batch processing and simplifies crash recovery (unprocessed files can be re-read on restart). This determines only the input adapter of the ingestion process. All downstream logic is independent of the answer.

---

## Metadata Management

Streams, bindings, and reference tables are managed by a governance CLI (Python), not by the ingestion process. Operators:

1. Register a `measurement_type` when a new sensor category is introduced
2. Register a `station` when a new site is deployed
3. Register a `device` when hardware is provisioned
4. Create a `stream` to declare the logical series at a station
5. Create a `stream_binding` to connect the device to the stream (with overlap validation)

When a device is replaced: close the existing binding (`valid_to = now`) and open a new one for the replacement device. The stream and all its historical observations remain untouched.

---

## Maintenance Schedule

| Job | Frequency | Operation | Tooling | Retention rule |
| --- | --- | --- | --- | --- |
| File compaction (current tables) | Nightly | Merge small files into large sorted Parquet files per partition | PyIceberg rewrite API if sufficient; minimal local PySpark script as fallback; verify against current PyIceberg release before relying on it | Target: 128 MB files post-compaction |
| File compaction (history tables) | Nightly | Merge recent append files into large sorted Parquet files per partition | PyIceberg rewrite API if sufficient; minimal local PySpark script as fallback | Target: 128 MB files post-compaction |
| Snapshot expiry (current tables) | Weekly | Expire snapshots beyond retention window | PyIceberg `expire_snapshots()` | Keep snapshots ≤ 7 days old; always retain at least the most recent snapshot |
| Snapshot expiry (history tables) | Weekly | Expire snapshots beyond retention window | PyIceberg `expire_snapshots()` | Keep snapshots ≤ 7 days old; always retain at least the most recent snapshot |
| Snapshot expiry (metadata tables) | Weekly | Expire old metadata snapshots | PyIceberg `expire_snapshots()` | Keep snapshots ≤ 30 days old; always retain at least the most recent snapshot |
| Month-end snapshot pinning | Monthly, before expiry runs | Pin a named tag for the last snapshot of each month on production branches | Iceberg tag API | Retained indefinitely for time-travel audit |
| Orphan file cleanup | Monthly | Remove unreferenced Parquet files from failed or aborted writes | PyIceberg or custom S3 diff script | Remove files unreferenced for > 3 days (buffer for in-progress writes) |
| Binding audit | Weekly | Check for overlapping binding intervals (stream-side and device-output-side) | Custom Python audit job | Alert on first violation |
| Type consistency audit | Weekly | Check that each stream's observations land in the correct fact-table family | Custom Python audit job | Alert on first violation |
| Version consistency audit | Weekly | Check current/history consistency, version uniqueness, and valid supersession chains | Custom Python audit job | Alert on first violation |
| Replay identity audit | Weekly | Check for inconsistent reuse of `(source_system, ingress_record_id)` | Custom Python audit job | Alert on first violation |

**Note on snapshot volume:** Each batch commit produces one snapshot per table written. The total daily snapshot count depends on the chosen commit frequency and the number of active table families. For example, if the ingestion process commits once per minute to both the history and current tables of a single observation family, that alone produces ~2,880 snapshots per day. Multiple active families multiply accordingly. Weekly snapshot expiry is not optional. It must run reliably or catalog metadata will grow without bound. The chosen commit frequency should be documented, and the snapshot volume estimate should be derived from it before deployment.

**Note on metadata cleanup:** Each write in Iceberg produces a new metadata file. Tables with frequent commits should enable regular metadata cleanup and orphan-file cleanup. This matters regardless of whether row-level changes are implemented through file rewrites or through v3 row-level delete machinery.

**Note on compaction:** File compaction remains the most operationally significant maintenance job. PyIceberg's rewrite/compaction API maturity should be verified before relying on it exclusively. If it is insufficient for the file volume produced at this ingestion rate, a minimal PySpark job (local, not a cluster) is the fallback.

**Current-table rewrites and row-level maintenance:** In Iceberg v3, current-table corrections may be implemented more efficiently than before, but they still create new snapshots and may still require later maintenance. If current tables eventually adopt merge-on-read row-level writes, periodic rewrite/compaction of affected partitions remains part of the operating model to keep read performance bounded.

**History tables:** Append-only (see Observation Lifecycle). They should not accumulate row-level correction metadata under normal operation.

**`ingest_batches` rows:** Retained indefinitely. The table is small and serves as the permanent audit log.

---

## S3 Layout

```text
s3://numairs-warehouse/
└── numairs.db/
    ├── measurement_types/
    │   ├── metadata/
    │   └── data/
    ├── stations/
    ├── devices/
    ├── streams/
    ├── stream_bindings/
    ├── correction_events/
    ├── ingest_batches/
    ├── obs_numeric_scalar_current/
    │   ├── metadata/
    │   └── data/
    │       ├── observed_at_day=2025-01-01/
    │       ├── observed_at_day=2025-01-02/
    │       └── ...
    ├── obs_numeric_scalar_history/
    │   ├── metadata/
    │   └── data/
    │       ├── observed_at_day=2025-01-01/
    │       │   ├── 00000-compact.parquet   ← compacted, sorted
    │       │   └── 00001-append.parquet    ← recent append, pre-compaction
    │       ├── observed_at_day=2025-01-02/
    │       └── ...
    └── ...
    (vector and text current/history tables follow the same structure)
```

---

## Common Query Patterns

> SQL examples are semantic illustrations. Exact timestamp literal syntax and time-zone functions vary by engine. All production query sessions must be configured to run in UTC before executing any timestamp comparison.

### One stream over a time range (most common)

```sql
SELECT observed_at, value
FROM obs_numeric_scalar_current
WHERE stream_id = '<uuid>'
  AND observed_at >= TIMESTAMPTZ '2025-01-01 00:00:00 UTC'
  AND observed_at <  TIMESTAMPTZ '2025-02-01 00:00:00 UTC'
ORDER BY observed_at;
```

### All streams at a station for a day

```sql
SELECT o.observed_at, s.stream_code, o.value
FROM obs_numeric_scalar_current o
JOIN streams s ON o.stream_id = s.stream_id
WHERE s.station_id = '<uuid>'
  AND o.observed_at >= TIMESTAMPTZ '2025-01-01 00:00:00 UTC'
  AND o.observed_at <  TIMESTAMPTZ '2025-01-02 00:00:00 UTC'
ORDER BY o.observed_at, s.stream_code;
```

### Hourly aggregates

```sql
SELECT
  stream_id,
  date_trunc('hour', observed_at) AS hour,
  avg(value), min(value), max(value)
FROM obs_numeric_scalar_current
WHERE observed_at >= TIMESTAMPTZ '2025-01-01 00:00:00 UTC'
  AND observed_at <  TIMESTAMPTZ '2025-01-08 00:00:00 UTC'
GROUP BY stream_id, date_trunc('hour', observed_at)
ORDER BY stream_id, hour;
```

### Full correction history for one logical observation

```sql
SELECT observation_id,
       observation_version_id,
       supersedes_version_id,
       quality_code,
       correction_reason,
       correction_event_id,
       ingested_at,
       value
FROM obs_numeric_scalar_history
WHERE observation_id = '<observation-id>'
ORDER BY ingested_at;
```

### Which device produced a stream during a time range

```sql
SELECT b.device_id, b.valid_from, b.valid_to,
       d.manufacturer, d.model, d.serial_number
FROM stream_bindings b
JOIN devices d ON b.device_id = d.device_id
WHERE b.stream_id = '<uuid>'
  AND b.valid_from  < TIMESTAMPTZ '2025-02-01 00:00:00 UTC'
  AND coalesce(b.valid_to, TIMESTAMPTZ '9999-12-31 00:00:00 UTC') > TIMESTAMPTZ '2025-01-01 00:00:00 UTC'
ORDER BY b.valid_from;
```

### Full device history across moves and replacements

```sql
SELECT b.valid_from, b.valid_to, s.stream_code, s.station_id, s.measurement_type_id
FROM stream_bindings b
JOIN streams s ON b.stream_id = s.stream_id
WHERE b.device_id = '<uuid>'
ORDER BY b.valid_from;
```

### GPS vector retrieval (named dimensions from `measurement_types`)

```sql
SELECT o.observed_at, o.values,
       mt.dimensions[1].name AS dim1,  -- "latitude"
       mt.dimensions[2].name AS dim2,  -- "longitude"
       mt.dimensions[3].name AS dim3   -- "elevation"
FROM obs_numeric_vector_current o
JOIN streams s ON o.stream_id = s.stream_id
JOIN measurement_types mt ON s.measurement_type_id = mt.measurement_type_id
WHERE o.stream_id = '<uuid>'
  AND o.observed_at >= TIMESTAMPTZ '2025-01-01 00:00:00 UTC'
  AND o.observed_at <  TIMESTAMPTZ '2025-01-02 00:00:00 UTC'
ORDER BY o.observed_at;
```

### Audit: all accepted versions written from one upstream ingress record

```sql
SELECT observation_id,
       observation_version_id,
       stream_id,
       observed_at,
       quality_code,
       value,
       ingested_at
FROM obs_numeric_scalar_history
WHERE source_system = 'mqtt_gateway_a'
  AND ingress_record_id = 'abc123'
ORDER BY ingested_at;
```

### Audit: current row must match latest accepted history version

```sql
WITH ranked_history AS (
  SELECT
    observation_id,
    observation_version_id,
    row_number() OVER (
      PARTITION BY observation_id
      ORDER BY ingested_at DESC, observation_version_id DESC
    ) AS rn
  FROM obs_numeric_scalar_history
)
SELECT c.observation_id,
       c.observation_version_id AS current_version_id,
       h.observation_version_id AS latest_history_version_id
FROM obs_numeric_scalar_current c
JOIN ranked_history h
  ON c.observation_id = h.observation_id
WHERE h.rn = 1
  AND c.observation_version_id <> h.observation_version_id;
```

---

## Open Questions and Pre-Deployment Validations

This section tracks design decisions that are not yet resolved, assumptions that need validation before deployment, and operational gaps that should be addressed as the system matures. Items are grouped by urgency.

---

### Must resolve before first production ingestion

**OQ-1. Ingestion process crash recovery.**
The ingestion process buffers rows in memory until it commits the current batch. If the process crashes mid-batch, uncommitted rows are lost unless the input adapter supports replay. If the input is file-based (the preferred model), crash recovery is straightforward: re-process unacknowledged files on restart. If the input is a non-persistent channel (e.g. bare TCP socket), data loss is possible. The choice of input adapter must be made with crash recovery semantics as a primary selection criterion, not an afterthought.

**OQ-2. Network failure between ingestion server and Compute Canada.**
The ingestion process runs on the sensors team's server and writes to Ceph S3 on Compute Canada over a network link. If the link goes down, the process must either buffer locally and retry, or stop accepting input until the link recovers. Questions to answer: Is there a disk-backed queue? How large can the local backlog grow? What happens if the backlog exceeds available local disk? For a permanent-retention project, this failure mode needs an explicit policy.

**OQ-3. The archive path.**
The ingestion architecture mentions that the ingestion process "compresses ingested data and writes it to an archive for daily upload to an object store." This line appears once and is not elaborated. Questions to answer: Is this the raw input data before validation, or a second copy of validated data? Is the destination the same Ceph S3 or a separate store? Is this intended as a disaster recovery mechanism? If so, there must be a documented rebuild procedure that can reconstruct Iceberg table state from archived data. If it is not intended as a recovery path, its role should be clarified so operators don't rely on it incorrectly.

**OQ-4. Replay deduplication implementation.**
The schema defines the dedup rule (`source_system, ingress_record_id` → reject) but not the mechanism. At the project's ingestion rate (~100-1,000 events/sec), a simple approach is sufficient: maintain an in-memory set of recently seen pairs, bounded by a configurable time window, with periodic persistence to survive restarts. On process restart, re-scan recent history rows (filtered by `source_system` and a time window) to rebuild the dedup set. Document the chosen mechanism, the window size, and the restart rebuild procedure.

**OQ-5. DuckDB + Polaris REST catalog compatibility.**
The architecture commits to Apache Polaris as the single catalog endpoint and DuckDB as the primary ad-hoc query engine. DuckDB's Iceberg extension and its REST catalog support should be tested against the specific Polaris version deployed. The same applies to PyIceberg's REST catalog client. Confirm working integration (with version numbers) before deployment, or document a fallback if REST catalog support is incomplete.

---

### Must resolve before sustained production operation

**OQ-6. Current-table upsert atomicity.**
The ingestion process appends to the history table and then upserts the current table. Are these one Iceberg commit or two? If two, there is a brief window where the current table is stale relative to history. At typical batch commit frequencies the window is likely tolerable, but the consistency guarantee should be stated explicitly. If they are separate commits, each batch commit produces two snapshots per active table family, which affects the snapshot volume estimate (see OQ-7).

**OQ-7. Snapshot volume arithmetic.**
The daily snapshot count depends on the commit frequency and the number of active table families. If history and current writes are separate commits, each batch commit produces two snapshots per family. Before deployment, calculate the expected daily snapshot volume from the chosen commit frequency, confirm that the Polaris/PostgreSQL catalog can handle it, and verify that weekly snapshot expiry keeps metadata growth bounded.

**OQ-8. `quality_code` controlled vocabulary.**
`quality_code` is currently a free-form string. At ~1,000 streams ingesting from heterogeneous field stations, values will diverge quickly ("good", "Good", "GOOD", "ok", "valid", etc.). Define a canonical vocabulary (suggested starting point: `good`, `suspect`, `bad`, `corrected`, `missing_source`) and enforce it at ingestion. Add at least one quality-filtered query to Common Query Patterns, since in practice nearly every analytical query on environmental data excludes suspect or bad readings.

**OQ-9. Operational surface area tiering.**
The schema defines 13 tables, nightly compaction, weekly snapshot expiry, 4 weekly audit jobs, monthly orphan cleanup, monthly snapshot pinning, and a governance CLI. For a small team on research infrastructure, prioritize which components are required from day one vs. which can be deferred. Suggested day-one minimum: scalar current/history tables, metadata tables, ingest_batches, compaction, snapshot expiry. Suggested deferred: text tables, vector tables (if only a few streams need them), correction_events, branched correction campaigns, the 4 separate audit jobs (start with a single monthly sweep and split later).

**OQ-10. Catalog disaster recovery.**
The Polaris catalog is backed by PostgreSQL on Compute Canada. If that PostgreSQL instance is lost, the Iceberg tables become unqueryable — the Parquet files still exist on Ceph, but no engine knows how to assemble them into tables. Add a PostgreSQL backup policy to the maintenance schedule (suggested: daily pg_dump, retained for 30 days). Document a catalog rebuild procedure. For a project where "migration is copying files" is a selling point, the catalog's recoverability is part of that promise.

**OQ-11. Ingestion process monitoring.**
The `ingest_batches` table captures post-hoc audit data, but nothing detects a stalled or degraded ingestion process in real time. At minimum, implement: a heartbeat or last-successful-commit timestamp, a rows-ingested-per-batch metric, and an alert on abnormal rejection rates. This is especially important given the network topology (ingestion process on one server, storage on another).

---

### Should resolve before the schema stabilizes

**OQ-12. `properties` map governance.**
The `properties map<string, string>` field appears on 7 tables with no usage policy. Without guidance, it becomes a dumping ground for unstructured metadata that no query engine can efficiently filter on (Parquet map columns do not support predicate pushdown in most engines). Proposed policy: "The `properties` map is for operator-managed metadata that is not used in query predicates, join conditions, or analytical logic. Any field that influences query behavior or appears in WHERE clauses must be promoted to a named column via schema evolution."

**OQ-13. `stream_binding_id` lookup semantics for late data.**
When the ingestion process handles a late observation (e.g. `observed_at` is three days in the past), it performs a binding lookup to populate the convenience `stream_binding_id` field. The document should state whether this lookup resolves against the binding state at ingestion time or against the binding interval valid at `observed_at`. Given the policy that `stream_binding_id` is never repaired after the fact, getting it right on the initial write matters. Late data is a routine condition for field stations with intermittent connectivity, not an edge case.

**OQ-14. Nested struct query ergonomics for `measurement_types.dimensions`.**
The `dimensions` field uses `list<struct<ordinal, name, unit>>`. The GPS vector query example uses `mt.dimensions[1].name`, which works in some engines but not all. DuckDB, Trino, and StarRocks each have different syntax for accessing nested struct fields within lists. Since this table is small reference data, this is not a performance issue, but it could create friction for ad-hoc users. Consider adding engine-specific syntax notes to the GPS query example, or document the tested engines and versions.

**OQ-15. Vector `list<double>` query cost.**
Storing vectors as `list<double>` is clean and flexible, but Parquet stores lists as nested groups. No engine can push down predicates on individual dimensions (e.g. "all GPS readings where latitude > 45" must deserialize the entire list for every row). At the project's likely scale (a small number of vector streams), this is probably acceptable. But if vector streams grow to dominate the workload, a materialized view or denormalized projection with named scalar columns per dimension may become necessary. Acknowledge this tradeoff so future operators know when to revisit the decision.

**OQ-16. Concurrent writer policy.**
What happens if two ingestion process instances run simultaneously (e.g. during a deployment rollover or accidental double-start)? Iceberg handles this with optimistic concurrency (commit retries on snapshot conflicts), but the upsert logic for current tables may not be safe under concurrent writers without explicit conflict handling. State whether single-writer is a hard architectural requirement (enforced by PID file, lock, or deployment constraint) or whether the ingestion process is designed to handle commit conflicts gracefully.
