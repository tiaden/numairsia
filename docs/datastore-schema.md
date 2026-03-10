# Numairsia Authoritative Schema

**Governing principle:**
> A stream is the stable logical time series. A device is the movable and replaceable hardware. Bindings connect them over time.

---

## Technology Stack

| Layer | Choice | Rationale |
| --- | --- | --- |
| Table format | Apache Iceberg | Open standard, S3-native, schema evolution, partition evolution, multi-engine |
| Object storage | Ceph S3 | First-class Iceberg backend; no separate database for data |
| Catalog | Apache Polaris (Iceberg REST Catalog), backed by PostgreSQL | Open standard HTTP catalog API; every engine (PyIceberg, DuckDB, Trino, StarRocks) connects to the same endpoint identically |
| Primary write path | PyIceberg (Python) | Runs on central server; direct Arrow → Parquet → Iceberg |
| Ad-hoc queries | DuckDB | Zero-server, reads Iceberg natively |
| Distributed queries | Trino or StarRocks | When query concurrency or BI dashboards are required |

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
| Ingestion daemon (Python) | `TZ=UTC` in process environment |
| Governance CLI (Python) | `TZ=UTC` in process environment |
| Maintenance jobs (Python) | `TZ=UTC` in process environment |
| PostgreSQL catalog connection | `SET TIME ZONE 'UTC'` on every connection |
| DuckDB query sessions | `SET TimeZone = 'UTC'` before any timestamp comparison |
| Trino query sessions | `SET TIME ZONE 'UTC'` |
| StarRocks query sessions | `SET time_zone = '+00:00'` at session start |

Any writer or query session that operates in a local timezone can produce incorrect partition pruning and incorrect time-range query results

---

## Logical Model

```text
measurement_types   stations
      │                │
      └──────┬──────── ┘
             │
          streams ──────── stream_bindings ──── devices (sensors)
             │
    ┌────────┴─────────┐
    │                  │
obs_numeric_scalar   obs_numeric_vector   obs_text (optional)
```

`streams` is the central entity. Observations always belong to a stream, never directly to a device.

---

## Tables

### `measurement_types`

Defines what is being measured. Immutable in meaning ... if semantics change materially, create a new record.

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
device_output      string                   -- nullable; distinguishes outputs on multi-output hardware;
valid_from         timestamptz   NOT NULL   -- UTC microseconds; inclusive
valid_to           timestamptz              -- UTC microseconds; exclusive; NULL = currently active
replacement_reason string                   -- descriptive, not normative
properties         map<string, string>
```

**Rules:**

- `device_output` determines which physical output channel on a device is producing the data on a multi-output hardware.
- **Primary invariant (analytically critical):** For a given `stream_id`, binding intervals must not overlap. A stream must have at most one active producer at any time (violating this makes observations ambiguous). Enforced by ingestion validation.
- **Secondary invariant:** For a given `(device_id, device_output)`, binding intervals should not overlap unless deliberate fan-out to multiple streams is intended and explicitly documented. Enforced by governance tooling.
- `valid_to = NULL` means the binding is open (device is currently assigned).
- The active binding for an observation at time T: `valid_from <= T AND (valid_to IS NULL OR valid_to > T)`

---

### `obs_numeric_scalar`

Fact table for scalar numeric observations.

**Partitioning:** `days(observed_at)`
**Sort order within partition:** `stream_id ASC, observed_at ASC`

```text
observed_at        timestamptz   NOT NULL   -- UTC microseconds; event time, not ingestion time
stream_id          string        NOT NULL   -- FK → streams (value_kind must be "numeric_scalar")
value              double        NOT NULL
quality_code       string                   -- e.g. "good", "suspect", "bad", "corrected"; vocabulary is operational
source_record_id   string                   -- stable external ID for idempotency and replay; nullable
stream_binding_id  string                   -- non-authoritative convenience annotation; see policy below
ingest_batch_id    string                   -- FK → ingest_batches
ingested_at        timestamptz              -- UTC microseconds; when this row was written
```

Do **not** declare identifier fields on this table or the other obs tables. Observations have no enforced unique row identity (clock jitter can produce two readings at the same microsecond for the same stream). Identifier fields on billion-row append tables would be operationally misleading.

**`stream_binding_id` policy:** Written once at ingestion time as a hint to avoid temporal joins in the common case. **Never repaired after the fact.** If a binding interval is later corrected, rows in this table retain their original stale value. Any query requiring accurate provenance must join `stream_bindings` on `(stream_id, observed_at)`.

---

### `obs_numeric_vector`

Fact table for ordered numeric vector observations (e.g. GPS, accelerometer, wind vector).

**Partitioning:** `days(observed_at)`
**Sort order within partition:** `stream_id ASC, observed_at ASC`

```text
observed_at        timestamptz   NOT NULL   -- UTC microseconds
stream_id          string        NOT NULL   -- FK → streams (value_kind must be "numeric_vector")
values             list<double>  NOT NULL   -- ordered per measurement_type.dimensions[].ordinal
quality_code       string
source_record_id   string
stream_binding_id  string                   -- non-authoritative; see policy in obs_numeric_scalar
ingest_batch_id    string
ingested_at        timestamptz
```

**Rules:**

- `cardinality(values)` must equal `measurement_type.dimension_count` for the stream's measurement type.
- Dimension semantics (which index means what) live in `measurement_types.dimensions`, not in this table.

---

### `obs_text` (optional)

Fact table for categorical or string observations.

**Partitioning:** `days(observed_at)`
**Sort order within partition:** `stream_id ASC, observed_at ASC`

```text
observed_at        timestamptz   NOT NULL
stream_id          string        NOT NULL   -- FK → streams (value_kind must be "text")
value              string        NOT NULL
quality_code       string
source_record_id   string
stream_binding_id  string                   -- non-authoritative; see policy in obs_numeric_scalar
ingest_batch_id    string
ingested_at        timestamptz
```

---

### `ingest_batches`

Audit table for ingestion runs. Retained indefinitely ... it is the permanent audit log.

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

## Iceberg Table Properties

These properties are set at table creation and govern file layout, compression, and pruning effectiveness. They apply to all three obs tables. Metadata tables (stations, devices, streams, etc.) use default properties.They are small enough that tuning is unnecessary.

```text
write.target-file-size-bytes          = 134217728   -- 128 MB; sized for daily partitions at this scale
write.parquet.compression-codec       = zstd         -- default; stated explicitly to prevent writer default drift
write.metadata.metrics.column.stream_id = full       -- UUID strings need full min/max for row-group pruning
```

**Why 128 MB and not the Iceberg default of 512 MB:** At ~1-10M rows/day compressed with zstd, a daily partition produces roughly 100 to 500 MB of data. A 512 MB target would produce one under-sized file per day after compaction. A 128 MB target produces 1-4 well-sized files per day, enabling meaningful parallelism in Trino/StarRocks without creating too many small files for DuckDB.

**Why only `stream_id` needs full metrics:** `observed_at` is `timestamptz`, stored as INT64. Iceberg already applies full min/max metrics to numeric types by default. The `truncate(16)` default applies only to string and binary columns. `stream_id` is a 36-character UUID string. With `truncate(16)`, only the first 16 characters are stored for statistics, which reduces row-group skip effectiveness. The `full` override stores the complete UUID, enabling precise range-based pruning.

---

## Partitioning Rationale

`days(observed_at)` is the right granularity for this workload:

- At ~1-10M rows/day across all streams, one day fits in a few 128 MB Parquet files well-sized for Iceberg.
- Aligns with the dominant query pattern ("show me data for this stream over N days").
- `hours()` creates too many small files overnight. `months()` forces full-month scans for single-day queries.

Sort order `(stream_id ASC, observed_at ASC)` within each partition improves the effectiveness of Parquet row-group pruning: when writers respect this order and row groups are appropriately sized, a query `WHERE stream_id = X AND observed_at BETWEEN t1 AND t2` can skip most row groups in the daily partition via min/max statistics. This avoids per-stream sub-partitioning, which would create thousands of tiny files per day. The benefit depends on writers honoring the sort order and on row-group size. it is not a hard guarantee.

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

The "never reuse a removed name" rule is critical: downstream code (scripts, dashboards, views) often references column names by string. Reusing a name silently changes the meaning of all such references.

---

## Null, NaN, and Infinity Policy (case of ingestion via jsonl format)

The JSON → Arrow → Parquet → SQL pipeline does not treat IEEE 754 special floating-point values consistently:

- JSON (strict spec) does not define NaN or Infinity as valid number literals.
- Arrow `float64` supports NaN, +Inf, and -Inf.
- Parquet DOUBLE stores IEEE 754 natively, so NaN is a valid value in a Parquet file.
- SQL engines diverge: DuckDB supports NaN in queries; Trino and StarRocks have inconsistent NaN propagation in aggregations.

**Policy:**

- NaN, +Infinity, and -Infinity are **rejected at ingestion**. The ingestion daemon must validate every numeric value and refuse to write the row entirely if a special float is present.
- A sensor reading that cannot produce a valid finite double is **dropped**, not stored. The associated `ingest_batch` record accounts for the dropped row count. Persistent drop patterns should be investigated at the source.
- `value` is `NOT NULL` on all numeric fact tables. There is no partial-row storage for numeric observations.
- `quality_code` is for valid finite readings with data quality concerns (e.g. `"suspect"`, `"out_of_range"` for in-range but physically implausible values) not a substitute for a missing value.
- Negative zero (`-0.0`) is accepted and treated as zero.

---

## Late-Data and Correction Policy

### Late observations

Late observations are accepted without a time limit. Iceberg's append model naturally handles out-of-order data. A late record appends to the correct daily partition regardless of how old it is. Nightly compaction restores sort order within the partition. `ingested_at` records when the row arrived, enabling audit queries that measure ingestion latency.

### Duplicate detection

If `source_record_id` is present, the ingestion daemon checks for an existing row with the same `source_record_id` in the same day partition before writing. Rows without `source_record_id` are not deduplicated. This check is best-effort at ingestion time; it does not prevent duplicates from concurrent writers or from replayed batches that omit `source_record_id`.

### Corrections

The design is **append-first**. This is not a default ! it is a deliberate commitment.

- **Corrected values** are appended as new rows with `quality_code = "corrected"`. The original erroneous row is not modified.
- **If the erroneous row must be removed,** the correction mechanism is a **batch partition rewrite** of the affected daily partition ... not Iceberg row-level delete files (positional or equality deletes). Row-level deletes add read-time overhead and operational complexity without proportional benefit at this scale and correction frequency.
- **Binding corrections** (fixing a `stream_bindings` interval) are applied directly to the small governance table. Fact rows are not retroactively updated. The `stream_binding_id` convenience column may become stale, per the documented policy.
- **No retroactive updates are applied to fact rows under any circumstance.** If accurate provenance is required after a binding correction, join `stream_bindings` on `(stream_id, observed_at)`.

---

## Identifier Fields

Iceberg supports declaring identifier fields on tables. These must be `NOT NULL` and signal to tooling (e.g. merge operations) what constitutes row identity. They are **not** enforced uniqueness constraints. Iceberg is not a relational database !

**Set identifier fields on all small metadata tables:**

| Table | Identifier field |
| --- | --- |
| `measurement_types` | `measurement_type_id` |
| `stations` | `station_id` |
| `devices` | `device_id` |
| `streams` | `stream_id` |
| `stream_bindings` | `stream_binding_id` |
| `ingest_batches` | `ingest_batch_id` |

**Do not set identifier fields on obs tables** (`obs_numeric_scalar`, `obs_numeric_vector`, `obs_text`). Observations have no enforced unique row identity. Two readings can arrive for the same stream at the same microsecond due to clock jitter or replay. Declaring identifier fields on billion-row append tables would be semantically misleading and provide no operational benefit under an append-first model.

---

## Iceberg Limitations (Non-Enforced Rules)

Iceberg does not enforce relational constraints. The following rules must be enforced by ingestion validation code and periodic audit jobs:

1. Foreign key correctness across tables
2. `stream_code` uniqueness within a station
3. Non-overlap of `stream_bindings` intervals for a given `stream_id` (primary invariant)
4. Non-overlap of `stream_bindings` intervals for a given `(device_id, device_output)` (secondary invariant)
5. Consistency between a stream's `value_kind` and the fact table used for its observations
6. `cardinality(values) = dimension_count` for vector observations
7. Immutability of stream `station_id` and `measurement_type_id` after creation
8. Immutability of measurement type semantics after creation

---

## Business Rules Summary

| Rule | Invariant class | Enforced by |
| --- | --- | --- |
| Stream belongs to one station for life | Primary | Ingestion guard |
| Stream has one measurement type for life | Primary | Ingestion guard |
| Binding intervals for a stream do not overlap | Primary | Ingestion guard |
| `obs_numeric_scalar` rows only for `numeric_scalar` streams | Primary | Ingestion guard |
| `obs_numeric_vector` rows only for `numeric_vector` streams | Primary | Ingestion guard |
| `cardinality(values) = dimension_count` | Primary | Ingestion guard |
| NaN, ±Infinity rejected at ingestion | Primary | Ingestion guard |
| Binding intervals for a `(device_id, device_output)` do not overlap | Secondary | Governance tooling |
| `stream_code` unique within station | Secondary | Ingestion guard |
| `measurement_types.code` globally unique | Secondary | Ingestion guard |
| `source_record_id` stable when present | Convention | - |
| `stream_binding_id` on fact rows is never repaired | Policy | Documentation |
| All system components operate in UTC | Operational | Deployment config |

---

## Ingestion Architecture

``` text
Central Server (other team's - ingestion daemon runs here)
        │
        │  [input interface - OPEN QUESTION: see below]
        ▼
  Ingestion Daemon (Python / PyIceberg)
  - validates stream_id exists and is actively bound
  - determines target fact table from stream's value_kind
  - rejects NaN, ±Infinity; drops the row and increments dropped-row counters
  - checks source_record_id for duplicates (if present)
  - buffers rows into Arrow RecordBatches
  - flushes every ~30 seconds or ~10,000 rows (whichever first)
  - writes stream_binding_id at ingestion time (point-in-time lookup)
  - writes ingest_batch record on completion
  - compresses ingested data and writes it to an archive for daily upload to an object store.
        │
        │  S3 API (network)
        ▼
Your Server
  - Polaris REST Catalog service (HTTP :8181)
      └── backed by PostgreSQL (catalog state only)
  - Ceph S3 (Parquet data + Iceberg metadata)
```

**Catalog connection (PyIceberg):** The ingestion daemon connects to Polaris using `type="rest"` with the Polaris HTTP endpoint, not `type="sql"`. DuckDB, Trino, and StarRocks each have native REST catalog connectors and connect to the same endpoint. PostgreSQL is internal to Polaris... no consumer talks to it directly.

**Open question:**: How does sensor data arrive at the central server's local interface?
Options: directory watch (inotify), Unix socket, localhost TCP, local MQTT broker, named pipe.
This determines only the input adapter of the ingestion daemon. All downstream logic is independent of the answer.

---

## Metadata Management

Streams, bindings, and reference tables are managed by a governance CLI (Python), not by the ingestion daemon. Operators:

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
| File compaction | Nightly | Merge small append files into large sorted Parquet files per partition | PyIceberg rewrite API if sufficient; minimal local PySpark script as fallback ... verify against current PyIceberg release before relying on it | Target: 128 MB files post-compaction |
| Snapshot expiry (obs tables) | Weekly | Expire snapshots beyond retention window | PyIceberg `expire_snapshots()` | Keep snapshots ≤ 7 days old; always retain at least the most recent snapshot |
| Snapshot expiry (metadata tables) | Weekly | Expire old metadata snapshots | PyIceberg `expire_snapshots()` | Keep snapshots ≤ 30 days old; always retain at least the most recent snapshot |
| Month-end snapshot pinning | Monthly, before expiry runs | Pin a named tag for the last snapshot of each month | PyIceberg tag API | Retained indefinitely for time-travel audit |
| Orphan file cleanup | Monthly | Remove unreferenced Parquet files from failed or aborted writes | PyIceberg or custom S3 diff script | Remove files unreferenced for > 3 days (buffer for in-progress writes) |
| Binding audit | Weekly | Check for overlapping binding intervals (stream-side and device-output-side) | Custom Python audit job | Alert on first violation |
| Type consistency audit | Weekly | Check that each stream's observations land in the correct fact table | Custom Python audit job | Alert on first violation |

**Note on snapshot volume:** At a 30-second flush interval, each obs table produces ~2,880 snapshots per day. After 7 days that is ~20,000 snapshots per table. Weekly snapshot expiry is not optional. It must run reliably or catalog metadata will grow without bound.

**Note on compaction:** File compaction is the most operationally significant maintenance job. PyIceberg's rewrite/compaction API maturity should be verified before relying on it exclusively. If it is insufficient for the file volume produced at this ingestion rate, a minimal PySpark job (local, not a cluster) is the fallback.

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
    ├── ingest_batches/
    └── obs_numeric_scalar/
        ├── metadata/
        │   ├── v1.metadata.json
        │   └── snap-*.avro
        └── data/
            ├── observed_at_day=2025-01-01/
            │   ├── 00000-compact.parquet   ← compacted, sorted
            │   └── 00001-append.parquet    ← recent append, pre-compaction
            ├── observed_at_day=2025-01-02/
            └── ...
    (obs_numeric_vector and obs_text follow the same structure)
```

---

## Common Query Patterns

> SQL examples are semantic illustrations. Exact timestamp literal syntax and time-zone functions vary by engine. All production query sessions must be configured to run in UTC before executing any timestamp comparison.

### One stream over a time range (most common)

```sql
SELECT observed_at, value
FROM obs_numeric_scalar
WHERE stream_id = '<uuid>'
  AND observed_at >= TIMESTAMPTZ '2025-01-01 00:00:00 UTC'
  AND observed_at <  TIMESTAMPTZ '2025-02-01 00:00:00 UTC'
ORDER BY observed_at;
```

### All streams at a station for a day

```sql
SELECT o.observed_at, s.stream_code, o.value
FROM obs_numeric_scalar o
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
FROM obs_numeric_scalar
WHERE observed_at >= TIMESTAMPTZ '2025-01-01 00:00:00 UTC'
  AND observed_at <  TIMESTAMPTZ '2025-01-08 00:00:00 UTC'
GROUP BY stream_id, date_trunc('hour', observed_at)
ORDER BY stream_id, hour;
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

### GPS vector retrieval (named dimensions from measurement_types)

```sql
SELECT o.observed_at, o.values,
       mt.dimensions[1].name AS dim1,  -- "latitude"
       mt.dimensions[2].name AS dim2,  -- "longitude"
       mt.dimensions[3].name AS dim3   -- "elevation"
FROM obs_numeric_vector o
JOIN streams s ON o.stream_id = s.stream_id
JOIN measurement_types mt ON s.measurement_type_id = mt.measurement_type_id
WHERE o.stream_id = '<uuid>'
  AND o.observed_at >= TIMESTAMPTZ '2025-01-01 00:00:00 UTC'
  AND o.observed_at <  TIMESTAMPTZ '2025-01-02 00:00:00 UTC'
ORDER BY o.observed_at;
```
