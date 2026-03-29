# Azure Databricks — Lakeflow Spark Declarative Pipelines (SDP)

## Complete Reference Card & Cheat Sheet — `pyspark.pipelines` API

> **Scope:** Lakeflow Spark Declarative Pipelines (SDP) · `pyspark.pipelines` module · Apache Spark 4.1+ · Auto Loader · Flows · CDC · Sinks · Jobs · Design Patterns · Best Practices
> **Runtime:** Databricks Runtime 15.4+ · Lakeflow Release 2025.x · Apache Spark 4.1
> **Note:** The legacy `dlt` module still works but is superseded by `pyspark.pipelines`. Migrate with `import dlt` → `from pyspark import pipelines as dp`.

-----

## Table of Contents

1. [Module Overview & Migration from DLT](#1-module-overview--migration-from-dlt)
1. [Pipeline Architecture & Concepts](#2-pipeline-architecture--concepts)
1. [Python API — Decorators & Functions](#3-python-api--decorators--functions)
1. [Pipeline SQL API — All Directives](#4-pipeline-sql-api--all-directives)
1. [Auto Loader (cloudFiles)](#5-auto-loader-cloudfiles)
1. [Data Quality — Expectations](#6-data-quality--expectations)
1. [Flows — Append Flow & Multi-Source](#7-flows--append-flow--multi-source)
1. [Change Data Capture — AUTO CDC APIs](#8-change-data-capture--auto-cdc-apis)
1. [Sinks — External Write Targets](#9-sinks--external-write-targets)
1. [Event Hooks — Custom Monitoring](#10-event-hooks--custom-monitoring)
1. [Pipeline Configuration Reference](#11-pipeline-configuration-reference)
1. [Table Properties & Storage](#12-table-properties--storage)
1. [Parameterization & Runtime Variables](#13-parameterization--runtime-variables)
1. [Design Patterns](#14-design-patterns)
1. [Databricks Jobs — Full Reference](#15-databricks-jobs--full-reference)
1. [Job Task Types & Parameters](#16-job-task-types--parameters)
1. [Job Orchestration Patterns](#17-job-orchestration-patterns)
1. [Monitoring, Observability & Alerts](#18-monitoring-observability--alerts)
1. [Method & Function Quick-Reference Tables](#19-method--function-quick-reference-tables)
1. [Best Practices](#20-best-practices)
1. [Common Pitfalls & Fixes](#21-common-pitfalls--fixes)

-----

## 1. Module Overview & Migration from DLT

### New Import

```python
# NEW — recommended
from pyspark import pipelines as dp

# OLD — still works but deprecated
import dlt
```

### DLT → SDP API Mapping Table

|Area                        |DLT (`import dlt`)                    |SDP (`from pyspark import pipelines as dp`)  |In Apache Spark 4.1?|
|----------------------------|--------------------------------------|---------------------------------------------|--------------------|
|Streaming table             |`@dlt.table` + streaming read         |`@dp.table` + streaming read                 |✅ Yes               |
|Materialized view           |`@dlt.table` + batch read             |`@dp.materialized_view`                      |✅ Yes               |
|Temporary view              |`@dlt.view`                           |`@dp.temporary_view`                         |✅ Yes               |
|Append flow                 |`@dlt.append_flow`                    |`@dp.append_flow`                            |✅ Yes               |
|Sink                        |`@dlt.create_sink`                    |`dp.create_sink(...)`                        |✅ Yes               |
|Streaming table (imperative)|`dlt.create_streaming_table(...)`     |`dp.create_streaming_table(...)`             |✅ Yes               |
|CDC (apply changes)         |`dlt.apply_changes(...)`              |`dp.create_auto_cdc_flow(...)`               |❌ Databricks only   |
|Snapshot CDC                |`dlt.apply_changes_from_snapshot(...)`|`dp.create_auto_cdc_from_snapshot_flow(...)` |❌ Databricks only   |
|Expectations                |`@dlt.expect(...)`                    |`@dp.expect(...)`                            |❌ Databricks only   |
|Event hooks                 |`@dlt.on_event_hook`                  |`@dp.on_event_hook`                          |❌ Databricks only   |
|SQL – streaming             |`CREATE STREAMING TABLE`              |`CREATE OR REFRESH STREAMING TABLE`          |✅ Yes               |
|SQL – materialized          |`CREATE MATERIALIZED VIEW`            |`CREATE OR REFRESH MATERIALIZED VIEW`        |✅ Yes               |
|SQL – flow                  |`CREATE FLOW`                         |`CREATE FLOW ... AS INSERT INTO`             |✅ Yes               |
|SQL source ref              |`STREAM(LIVE.<table>)`                |`STREAM(<table>)` or `STREAM read_files(...)`|✅ Yes               |
|Event log                   |`spark.read.table("event_log")`       |`spark.read.table("event_log")`              |❌ Databricks only   |

### Key Breaking Changes vs. DLT

- `@dp.table` is **now exclusively for streaming tables** (batch reads should use `@dp.materialized_view`)
- `@dp.temporary_view` replaces `@dlt.view` (the name `view` is gone)
- `dp.create_auto_cdc_flow()` replaces `dlt.apply_changes()` (new name, same parameters + `name` + `once`)
- `dp.create_auto_cdc_from_snapshot_flow()` replaces `dlt.apply_changes_from_snapshot()`
- `LIVE.<table>` SQL prefix is no longer needed — reference tables by name directly
- `spark_conf` parameter added to `@dp.table` and `@dp.materialized_view`
- `cluster_by_auto` parameter added to all table decorators
- `private` parameter added to keep datasets out of Unity Catalog
- `refresh_policy` parameter added to `@dp.materialized_view`

-----

## 2. Pipeline Architecture & Concepts

### Medallion Architecture with SDP

```
Source (Cloud Storage / Kafka / CDC / JDBC)
        │
        ▼
[ Bronze — Streaming Table ]   ← @dp.table + Auto Loader / read_files
        │
        ▼
[ Silver — Streaming Table ]   ← @dp.table + expectations + CDC
        │
        ▼
[ Gold — Materialized View ]   ← @dp.materialized_view + aggregations
        │
   ┌────┴────┐
   ▼         ▼
[Sink]   [Serving Table]   ← dp.create_sink (Kafka / Delta external)
```

### Key Concepts

|Concept              |Description                                                                   |
|---------------------|------------------------------------------------------------------------------|
|**Streaming Table**  |Incremental, append-based; defined with `@dp.table` + `spark.readStream`      |
|**Materialized View**|Pre-computed batch result; defined with `@dp.materialized_view` + `spark.read`|
|**Temporary View**   |In-pipeline only, not published to Unity Catalog; `@dp.temporary_view`        |
|**Flow**             |The unit of computation that writes to a dataset; each table has at least one |
|**Append Flow**      |Flow that appends new records; supports multiple flows to one target          |
|**Auto CDC Flow**    |Flow that processes CDC / SCD changes; `dp.create_auto_cdc_flow()`            |
|**Sink**             |External write target (Kafka, Delta external); `dp.create_sink()`             |
|**Update**           |One pipeline execution cycle                                                  |
|**Pipeline**         |Logical container: datasets + flows + compute config + schedule               |
|**Event Hook**       |Python function triggered on pipeline events; `@dp.on_event_hook`             |

### Pipeline Modes

|Mode         |Behaviour                                 |Use Case                   |
|-------------|------------------------------------------|---------------------------|
|`triggered`  |Runs once, cluster shuts down after update|Batch / scheduled workloads|
|`continuous` |Cluster stays alive, perpetual micro-batch|Near-real-time streaming   |
|`development`|Reuses cluster, no retries, fast iteration|Authoring & debugging      |

### Dataset Type Decision Guide

```
Does the source support streaming reads?
  ├── YES → Use @dp.table (Streaming Table)
  │         • Auto Loader, Kafka, Delta CDF, streaming tables
  └── NO  → Use @dp.materialized_view (Materialized View)
            • Batch reads, JDBC, CSV/Parquet files, dimension tables

Is the result an intermediate/temporary step?
  └── YES → Use @dp.temporary_view (not published to UC)

Do you need to write to external systems (Kafka, external Delta)?
  └── YES → Use dp.create_sink() + @dp.append_flow
```

-----

## 3. Python API — Decorators & Functions

### Import

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F
from pyspark.sql.types import *
```

-----

### `@dp.table` — Define a Streaming Table

```python
@dp.table(
    name="bronze_rides",                          # Optional: override function name
    comment="Raw ride events from Auto Loader",   # UC comment
    spark_conf={                                  # NEW: per-dataset Spark config
        "spark.sql.shuffle.partitions": "200"
    },
    schema=StructType([...]),                     # Optional: enforce schema (StructType or DDL string)
    partition_cols=["event_date"],                # Static partitioning
    cluster_by=["customer_id", "event_date"],     # Liquid Clustering (explicit columns)
    cluster_by_auto=True,                         # NEW: auto liquid clustering (let Databricks decide)
    table_properties={
        "delta.autoOptimize.optimizeWrite": "true",
        "pipelines.reset.allowed": "false",
        "quality": "bronze"
    },
    path="abfss://container@sa.dfs.core.windows.net/bronze/rides",  # External storage path
    row_filter="region = 'EU'",                   # NEW: row-level security filter
    private=False,                                # NEW: True = not published to UC
)
@dp.expect("non_null_id", "ride_id IS NOT NULL")  # Expectations stack above @dp.table
def bronze_rides():
    return (
        spark.readStream
          .format("cloudFiles")
          .option("cloudFiles.format", "json")
          .option("cloudFiles.schemaLocation", "/mnt/schema/rides")
          .load(spark.conf.get("source.rides.path"))
          .withColumn("ingested_at", F.current_timestamp())
          .withColumn("source_file", F.col("_metadata.file_path"))
    )
```

-----

### `@dp.materialized_view` — Define a Materialized View (Batch)

```python
@dp.materialized_view(
    name="gold_daily_revenue",
    comment="Daily revenue aggregation by city",
    spark_conf={"spark.sql.adaptive.enabled": "true"},
    schema="""
        event_date DATE,
        city STRING,
        total_revenue DOUBLE,
        total_rides BIGINT,
        avg_fare DOUBLE
    """,
    partition_cols=["event_date"],
    cluster_by=["city"],
    cluster_by_auto=False,
    table_properties={"quality": "gold"},
    path=None,                                    # Optional: external path
    refresh_policy=None,                          # NEW: refresh schedule policy
    row_filter=None,                              # Row-level security
    private=False,
)
def gold_daily_revenue():
    return (
        spark.read.table("silver_rides")
          .groupBy("event_date", "city")
          .agg(
              F.sum("fare_usd").alias("total_revenue"),
              F.count("ride_id").alias("total_rides"),
              F.avg("fare_usd").alias("avg_fare")
          )
    )
```

-----

### `@dp.temporary_view` — In-Pipeline Temporary View

```python
@dp.temporary_view(
    name="v_active_rides",
    comment="Active rides only"
)
def active_rides():
    return spark.read.table("silver_rides").filter(F.col("status") != "test")
```

-----

### `dp.create_streaming_table()` — Imperative Streaming Table (CDC Target)

```python
dp.create_streaming_table(
    name="dim_customers",
    comment="Customer dimension with SCD2",
    spark_conf={"spark.sql.shuffle.partitions": "100"},
    table_properties={"delta.enableChangeDataFeed": "true"},
    partition_cols=["region"],
    cluster_by=["customer_id"],
    cluster_by_auto=False,
    schema="""
        customer_id BIGINT NOT NULL,
        name STRING,
        email STRING,
        region STRING,
        __start_at TIMESTAMP,
        __end_at TIMESTAMP
    """,
    expect_all={"valid_id": "customer_id IS NOT NULL"},          # DQ on target
    expect_all_or_drop={"valid_email": "email LIKE '%@%'"},
    expect_all_or_fail={"valid_region": "region IS NOT NULL"},
    row_filter="region = 'EU'",
    path=None,
)
```

-----

### `@dp.append_flow` — Explicit Append Flow (Multi-Source or Decoupled)

```python
# Decoupled: create target then attach flows separately
dp.create_streaming_table("customers_unified")

@dp.append_flow(
    target="customers_unified",
    name="flow_from_west",                        # Flow name (checkpoint key)
    spark_conf={"spark.sql.shuffle.partitions": "50"},
    once=False                                    # True = run once per update (backfill)
)
def append_west():
    return spark.readStream.table("customers_us_west")

@dp.append_flow(target="customers_unified", name="flow_from_east")
def append_east():
    return spark.readStream.table("customers_us_east")
```

-----

### `dp.create_auto_cdc_flow()` — CDC / SCD Processing (replaces `dlt.apply_changes`)

```python
dp.create_streaming_table("dim_customers_scd1")

dp.create_auto_cdc_flow(
    target="dim_customers_scd1",
    source="bronze_customers_cdc",               # Pipeline dataset name or UC table
    keys=["customer_id"],                        # Primary key column(s)
    sequence_by=F.col("updated_at"),             # Ordering column (latest wins)
    ignore_null_updates=False,                   # If True, null values don't overwrite
    apply_as_deletes=F.expr("op = 'DELETE'"),    # Optional: delete condition
    apply_as_truncates=F.expr("op = 'TRUNCATE'"),# Optional: truncate condition
    column_list=None,                            # Optional: only these columns
    except_column_list=["_metadata", "op"],      # Optional: exclude these columns
    stored_as_scd_type=1,                        # 1 = upsert, 2 = full history
    track_history_column_list=None,              # SCD2: track only these cols
    track_history_except_column_list=None,       # SCD2: exclude from tracking
    name="cdc_flow_customers",                   # NEW: explicit flow name
    once=False,                                  # NEW: True = run once per update
)
```

-----

### `dp.create_auto_cdc_from_snapshot_flow()` — Snapshot-Based CDC

```python
@dp.temporary_view(name="jdbc_snapshot")
def jdbc_snapshot():
    return (
        spark.read.format("jdbc")
          .option("url", spark.conf.get("jdbc.url"))
          .option("dbtable", "customers")
          .load()
    )

dp.create_streaming_table("dim_customers_snapshot")

dp.create_auto_cdc_from_snapshot_flow(
    target="dim_customers_snapshot",
    source="jdbc_snapshot",                      # View or function returning next snapshot
    keys=["customer_id"],
    stored_as_scd_type=2,
    track_history_column_list=["email", "segment"],
    track_history_except_column_list=None,
)
```

-----

### `dp.create_sink()` — External Sink (Kafka, Delta External)

```python
# Kafka sink
dp.create_sink(
    name="kafka_events_sink",
    format="kafka",
    options={
        "databricks.serviceCredential": "my-service-credential",  # UC service credential
        "kafka.bootstrap.servers": "myhub.servicebus.windows.net:9093",
        "topic": "processed-events"
    }
)

# External Delta table sink
dp.create_sink(
    name="external_delta_sink",
    format="delta",
    options={
        "tableName": "main.reporting.processed_events"  # Must be fully qualified
    }
)
```

-----

### `@dp.on_event_hook` — Custom Event Monitoring

```python
import requests

API_TOKEN = dbutils.secrets.get(scope="myScope", key="slackToken")

@dp.on_event_hook(
    max_allowable_consecutive_failures=3   # Disable hook after 3 consecutive failures
)
def notify_on_failure(event):
    if (event["event_type"] == "update_progress"
            and event["details"]["update_progress"]["state"] == "FAILED"):
        requests.post(
            "https://slack.com/api/chat.postMessage",
            headers={"Authorization": f"Bearer {API_TOKEN}"},
            json={"channel": "C123ABC", "text": f"Pipeline failed: {event}"}
        )
```

-----

## 4. Pipeline SQL API — All Directives

### CREATE OR REFRESH STREAMING TABLE

```sql
-- Basic streaming table (replaces CREATE STREAMING TABLE)
CREATE OR REFRESH STREAMING TABLE bronze_rides
  COMMENT "Raw ride events from Auto Loader"
  PARTITIONED BY (event_date)
  CLUSTER BY (customer_id)                       -- Liquid clustering
  TBLPROPERTIES ('quality' = 'bronze', 'pipelines.reset.allowed' = 'false')
AS SELECT
  *,
  _metadata.file_path AS source_file,
  current_timestamp() AS ingested_at
FROM STREAM read_files(                          -- NEW: read_files replaces cloud_files
  '${source.rides.path}',
  format => 'json',
  schemaLocation => '/mnt/schema/rides',
  schemaEvolutionMode => 'addNewColumns'
);
```

### CREATE OR REFRESH MATERIALIZED VIEW

```sql
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_revenue
  COMMENT "Daily revenue aggregation"
  CLUSTER BY AUTO                                 -- Auto liquid clustering
  TBLPROPERTIES ('quality' = 'gold')
AS
SELECT
  event_date,
  city,
  SUM(fare_usd)   AS total_revenue,
  COUNT(ride_id)  AS total_rides,
  AVG(fare_usd)   AS avg_fare
FROM silver_rides                                -- No LIVE. prefix needed in SDP
GROUP BY event_date, city;
```

### CREATE FLOW (Explicit Append Flow)

```sql
-- Step 1: Create target table
CREATE OR REFRESH STREAMING TABLE customers_unified;

-- Step 2: Add flows from multiple sources
CREATE FLOW flow_from_west
AS INSERT INTO customers_unified BY NAME
   SELECT * FROM STREAM(customers_us_west);

CREATE FLOW flow_from_east
AS INSERT INTO customers_unified BY NAME
   SELECT * FROM STREAM(customers_us_east);
```

### AUTO CDC (formerly APPLY CHANGES INTO)

```sql
CREATE OR REFRESH STREAMING TABLE dim_customers;

CREATE FLOW apply_cdc_customers
AS AUTO CDC INTO dim_customers
FROM STREAM(bronze_customers_cdc)
KEYS (customer_id)
APPLY AS DELETE WHEN op = 'DELETE'
SEQUENCE BY updated_at
COLUMNS * EXCEPT (op, _metadata)
STORED AS SCD TYPE 2;
```

### CONSTRAINT (Expectations in SQL)

```sql
CREATE OR REFRESH STREAMING TABLE silver_rides (
  CONSTRAINT valid_fare    EXPECT (fare_amount > 0),                          -- Warn (default)
  CONSTRAINT non_null_id   EXPECT (ride_id IS NOT NULL) ON VIOLATION DROP ROW, -- Drop
  CONSTRAINT valid_status  EXPECT (status IN ('completed','cancelled'))
                           ON VIOLATION FAIL UPDATE                           -- Fail
)
AS SELECT * FROM STREAM(bronze_rides);
```

### PRIVATE Tables (no UC metadata)

```sql
CREATE OR REFRESH PRIVATE STREAMING TABLE staging_temp
AS SELECT * FROM STREAM(bronze_events);
```

### SET for Pipeline Parameters in SQL

```sql
-- Reference pipeline config parameters in SQL
SET startDate = '2025-01-01';

CREATE OR REFRESH MATERIALIZED VIEW filtered_rides AS
SELECT * FROM silver_rides
WHERE event_date > '${startDate}'
  AND catalog_name = '${source_catalog}';
```

### SQL Keywords Quick Reference

```
CREATE OR REFRESH STREAMING TABLE        -- Streaming incremental target
CREATE OR REFRESH MATERIALIZED VIEW      -- Batch pre-computed result
CREATE OR REFRESH PRIVATE STREAMING TABLE -- Not published to UC
CREATE FLOW <name> AS INSERT INTO        -- Explicit append flow
AUTO CDC INTO <target> FROM STREAM(...)  -- CDC flow (replaces APPLY CHANGES)
STREAM(<table>)                          -- Read table as streaming source
STREAM read_files(path, format=>...)     -- Auto Loader streaming ingest
read_files(path, format=>...)            -- Auto Loader batch ingest
KEYS (col1, col2)                        -- CDC key columns
SEQUENCE BY <col>                        -- CDC ordering column
STORED AS SCD TYPE 1 | 2               -- CDC SCD mode
APPLY AS DELETE WHEN <expr>             -- CDC delete condition
CLUSTER BY (col1, col2)                 -- Liquid clustering
CLUSTER BY AUTO                         -- Auto liquid clustering
PARTITIONED BY (col)                    -- Static partitioning
TBLPROPERTIES (...)                     -- Table properties
CONSTRAINT <n> EXPECT (<sql_expr>)      -- DQ expect (warn)
ON VIOLATION DROP ROW                    -- DQ drop
ON VIOLATION FAIL UPDATE                 -- DQ fail
```

-----

## 5. Auto Loader (cloudFiles)

### Python — `spark.readStream.format("cloudFiles")`

```python
@dp.table(comment="Bronze: raw JSON rides")
def bronze_rides():
    return (
        spark.readStream
          .format("cloudFiles")
          .option("cloudFiles.format", "json")
          .option("cloudFiles.schemaLocation", spark.conf.get("source.schema_location"))
          .option("cloudFiles.inferColumnTypes", "true")
          .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
          .option("cloudFiles.maxFilesPerTrigger", "1000")
          .option("cloudFiles.backfillInterval", "1 week")
          .option("cloudFiles.pathGlobFilter", "*.json")
          .load(spark.conf.get("source.rides.path"))
          .select(
              "*",
              F.col("_metadata.file_path").alias("source_file"),
              F.col("_metadata.file_modification_time").alias("file_ts"),
              F.current_timestamp().alias("ingested_at")
          )
    )
```

### SQL — `read_files()` Function (Preferred in SDP SQL)

```sql
-- Streaming ingest with read_files (SDP SQL preferred approach)
CREATE OR REFRESH STREAMING TABLE bronze_rides AS
SELECT
  *,
  _metadata.file_path AS source_file,
  current_timestamp() AS ingested_at
FROM STREAM read_files(
  '${source.rides.path}',
  format => 'json',
  schemaLocation => '/mnt/schema/rides',
  inferColumnTypes => true,
  schemaEvolutionMode => 'addNewColumns',
  pathGlobFilter => '*.json'
);

-- Batch ingest with read_files
CREATE OR REFRESH MATERIALIZED VIEW dim_products AS
SELECT * FROM read_files(
  '/volumes/main/default/products/',
  format => 'parquet'
);
```

### Schema Evolution Modes

|Mode              |Behaviour                                           |
|------------------|----------------------------------------------------|
|`addNewColumns`   |New columns added to schema automatically (default) |
|`failOnNewColumns`|Pipeline fails if schema change detected            |
|`rescue`          |Unknown columns saved to `_rescued_data` JSON column|
|`none`            |New columns ignored / cast to null                  |

### Auto Loader — All Options Table

|Option                           |Default        |Description                                                 |
|---------------------------------|---------------|------------------------------------------------------------|
|`cloudFiles.format`              |—              |File format: json, csv, parquet, avro, orc, text, binaryFile|
|`cloudFiles.schemaLocation`      |—              |Path to persist inferred schema (required for JSON/CSV)     |
|`cloudFiles.inferColumnTypes`    |`false`        |Infer primitive types instead of all-string                 |
|`cloudFiles.schemaEvolutionMode` |`addNewColumns`|How schema changes are handled                              |
|`cloudFiles.schemaHints`         |—              |Force type hints, e.g. `"id BIGINT, ts TIMESTAMP"`          |
|`cloudFiles.maxFilesPerTrigger`  |`1000`         |Max files per micro-batch                                   |
|`cloudFiles.maxBytesPerTrigger`  |unbounded      |Max bytes per micro-batch                                   |
|`cloudFiles.includeExistingFiles`|`true`         |Process existing files on first run                         |
|`cloudFiles.useNotifications`    |`false`        |Event-driven file detection (Azure EventGrid)               |
|`cloudFiles.pathGlobFilter`      |—              |Glob pattern to filter file names                           |
|`cloudFiles.modifiedAfter`       |—              |Only files modified after this timestamp                    |
|`cloudFiles.modifiedBefore`      |—              |Only files modified before this timestamp                   |
|`cloudFiles.allowOverwrites`     |`false`        |Reprocess overwritten files                                 |
|`cloudFiles.backfillInterval`    |—              |Periodic full directory scan (safety net)                   |
|`cloudFiles.fetchParallelism`    |`4`            |Parallel file listing threads                               |
|`cloudFiles.resourceTags`        |—              |Tags for cloud notification resources                       |

-----

## 6. Data Quality — Expectations

### Python Expectation Decorators

```python
# Single expectation — warn (default)
@dp.expect("valid_fare", "fare_amount > 0")

# Single expectation — drop row on violation
@dp.expect_or_drop("non_null_id", "ride_id IS NOT NULL")

# Single expectation — fail pipeline on violation
@dp.expect_or_fail("valid_status", "status IN ('completed','cancelled','in_progress')")

# Stack multiple expectations on a single dataset
@dp.expect("valid_fare", "fare_amount > 0")
@dp.expect_or_drop("non_null_id", "ride_id IS NOT NULL")
@dp.table
def silver_rides():
    return spark.readStream.table("bronze_rides")
```

### Bulk Expectations (Dict-Based)

```python
rules = {
    "valid_fare":    "fare_amount > 0",
    "non_null_id":   "ride_id IS NOT NULL",
    "valid_status":  "status IN ('completed','cancelled','in_progress')",
}

@dp.expect_all(rules)             # Warn on any violation
@dp.expect_all_or_drop(rules)     # Drop row if any rule fails
@dp.expect_all_or_fail(rules)     # Fail pipeline if any row fails any rule

@dp.table
def silver_rides():
    return spark.readStream.table("bronze_rides")
```

### Expectations on `create_streaming_table()`

```python
dp.create_streaming_table(
    name="dim_customers",
    expect_all={"valid_id": "customer_id IS NOT NULL"},
    expect_all_or_drop={"valid_email": "email IS NOT NULL"},
    expect_all_or_fail={"valid_region": "region IN ('EU','US','APAC')"},
)
```

### Expectation Decorator Reference

|Decorator                           |On Violation                  |Metric Tracked                    |
|------------------------------------|------------------------------|----------------------------------|
|`@dp.expect(name, expr)`            |Keep row, log warning         |`passed_records`, `failed_records`|
|`@dp.expect_or_drop(name, expr)`    |Drop row silently             |`dropped_records`                 |
|`@dp.expect_or_fail(name, expr)`    |Fail entire pipeline update   |—                                 |
|`@dp.expect_all(rules_dict)`        |Keep rows, warn per rule      |Per-rule metrics                  |
|`@dp.expect_all_or_drop(rules_dict)`|Drop row if any rule fails    |`dropped_records`                 |
|`@dp.expect_all_or_fail(rules_dict)`|Fail if any row fails any rule|—                                 |

### Quarantine Pattern

```python
rules = {
    "valid_id":  "ride_id IS NOT NULL",
    "valid_amt": "fare_amount > 0"
}
quarantine_expr = "NOT({})".format(" AND ".join(rules.values()))

@dp.expect_all_or_drop(rules)
@dp.table(name="silver_clean")
def silver_clean():
    return spark.readStream.table("bronze_rides")

@dp.table(name="silver_quarantine")
def silver_quarantine():
    return (
        spark.readStream.table("bronze_rides")
          .filter(F.expr(quarantine_expr))
          .withColumn("quarantine_reason",
              F.when(F.col("ride_id").isNull(), "null_ride_id")
               .when(F.col("fare_amount") <= 0, "invalid_fare")
               .otherwise("unknown"))
    )
```

-----

## 7. Flows — Append Flow & Multi-Source

### Default Flow (Implicit)

```python
# A @dp.table definition implicitly creates a default append flow
@dp.table
def bronze_events():
    return spark.readStream.format("cloudFiles") \
        .option("cloudFiles.format", "json") \
        .load("/volumes/raw/events/")
```

### Explicit Multi-Source Append Flow (Python)

```python
# Create the target table independently
dp.create_streaming_table(
    name="rides_unified",
    schema="ride_id STRING, fare DOUBLE, city STRING, region STRING",
)

# Flow 1 — EU source
@dp.append_flow(target="rides_unified", name="flow_eu")
def flow_eu():
    return (
        spark.readStream.table("bronze_rides_eu")
          .withColumn("region", F.lit("EU"))
    )

# Flow 2 — US source (added later without full refresh)
@dp.append_flow(target="rides_unified", name="flow_us")
def flow_us():
    return (
        spark.readStream.table("bronze_rides_us")
          .withColumn("region", F.lit("US"))
    )
```

### Append Flow with `once=True` (Backfill Pattern)

```python
@dp.append_flow(target="rides_unified", name="backfill_historical", once=True)
def backfill():
    return spark.read.table("historical_rides_archive")  # Batch source
```

### Flow Types Summary

|Flow Type      |Decorator / Function                     |Source Type       |Target Type            |
|---------------|-----------------------------------------|------------------|-----------------------|
|Default append |`@dp.table` (implicit)                   |Streaming         |Streaming table        |
|Explicit append|`@dp.append_flow`                        |Streaming or batch|Streaming table or sink|
|CDC / SCD      |`dp.create_auto_cdc_flow()`              |Streaming CDC     |Streaming table        |
|Snapshot CDC   |`dp.create_auto_cdc_from_snapshot_flow()`|Batch snapshot    |Streaming table        |
|Sink write     |`@dp.append_flow` targeting sink         |Streaming         |External sink          |

-----

## 8. Change Data Capture — AUTO CDC APIs

### SCD Type 1 (Upsert — Latest Value Wins)

```python
dp.create_streaming_table("dim_customers_scd1")

dp.create_auto_cdc_flow(
    target="dim_customers_scd1",
    source="bronze_customers_cdc",
    keys=["customer_id"],
    sequence_by=F.col("updated_at"),
    stored_as_scd_type=1,
    except_column_list=["op", "_metadata"],
    apply_as_deletes=F.expr("op = 'DELETE'"),
)
```

### SCD Type 2 (Full History)

```python
dp.create_streaming_table(
    "dim_customers_scd2",
    table_properties={"delta.enableChangeDataFeed": "true"}
)

dp.create_auto_cdc_flow(
    target="dim_customers_scd2",
    source="bronze_customers_cdc",
    keys=["customer_id"],
    sequence_by=F.col("updated_at"),
    stored_as_scd_type=2,
    track_history_column_list=["email", "segment", "tier"],  # Only track these cols
    apply_as_deletes=F.expr("op = 'DELETE'"),
    except_column_list=["op", "updated_at"],
)
# Generates __START_AT and __END_AT columns automatically
# Query current: WHERE __end_at IS NULL
```

### CDC with Composite Keys

```python
dp.create_auto_cdc_flow(
    target="fact_order_lines",
    source="bronze_order_cdc",
    keys=["order_id", "line_item_id"],            # Composite key
    sequence_by="updated_at",
    stored_as_scd_type=1,
)
```

### `dp.create_auto_cdc_flow()` — Full Parameter Reference

|Parameter                         |Type         |Description                                 |
|----------------------------------|-------------|--------------------------------------------|
|`target`                          |str          |Target streaming table name                 |
|`source`                          |str          |Source dataset or UC table                  |
|`keys`                            |list[str]    |Primary key column(s)                       |
|`sequence_by`                     |Column or str|Ordering column (latest row wins)           |
|`ignore_null_updates`             |bool         |Skip updates where all non-key cols are null|
|`apply_as_deletes`                |Column expr  |Condition to treat row as DELETE            |
|`apply_as_truncates`              |Column expr  |Condition to treat row as TRUNCATE          |
|`column_list`                     |list[str]    |Explicit list of columns to process         |
|`except_column_list`              |list[str]    |Columns to exclude from processing          |
|`stored_as_scd_type`              |int or str   |`1` = upsert, `2` = history                 |
|`track_history_column_list`       |list[str]    |SCD2: only track these columns              |
|`track_history_except_column_list`|list[str]    |SCD2: exclude from tracking                 |
|`name`                            |str          |NEW: explicit flow name (checkpoint key)    |
|`once`                            |bool         |NEW: run this flow once per update          |

-----

## 9. Sinks — External Write Targets

### Supported Sink Formats

|Format |Description                                       |
|-------|--------------------------------------------------|
|`kafka`|Kafka / Azure Event Hubs                          |
|`delta`|External Delta table (UC managed or external path)|
|Custom |Any PySpark custom streaming data source          |

### Kafka / Event Hubs Sink

```python
dp.create_sink(
    name="eventhub_sink",
    format="kafka",
    options={
        "databricks.serviceCredential": "my-uc-service-credential",
        "kafka.bootstrap.servers": "myhub.servicebus.windows.net:9093",
        "topic": "processed-rides"
    }
)

@dp.append_flow(target="eventhub_sink", name="kafka_flow")
def kafka_flow():
    return (
        spark.readStream.table("silver_rides")
          .selectExpr(
              "CAST(ride_id AS STRING) AS key",
              "to_json(struct(*)) AS value"
          )
    )
```

### Delta External Table Sink

```python
dp.create_sink(
    name="reporting_sink",
    format="delta",
    options={"tableName": "main.reporting.silver_rides"}   # Fully qualified
)

@dp.append_flow(target="reporting_sink", name="reporting_flow")
def reporting_flow():
    return spark.readStream.table("silver_rides")
```

### Sink Rules & Limitations

- Only `@dp.append_flow` can write to sinks — CDC flows and materialized views cannot
- Delta sink table names must be fully qualified (`catalog.schema.table`)
- Full refresh does NOT clear sink data — reprocessed data is appended
- Expectations are not supported on sinks
- SQL interface not supported for sinks (Python only)

-----

## 10. Event Hooks — Custom Monitoring

### Basic Event Hook

```python
from pyspark import pipelines as dp

@dp.on_event_hook
def log_all_events(event):
    print(f"[EventHook] {event['event_type']}: {event}")
```

### Filter by Event Type

```python
@dp.on_event_hook
def alert_on_failure(event):
    if (event["event_type"] == "update_progress"
            and event["details"]["update_progress"]["state"] == "FAILED"):
        # Send to monitoring system
        send_alert(event)
```

### With Failure Tolerance

```python
@dp.on_event_hook(max_allowable_consecutive_failures=3)
def webhook_hook(event):
    # Disabled automatically after 3 consecutive failures
    requests.post("https://my-webhook.company.com/events", json=event)
```

### Key Event Types

|Event Type            |Description                                                         |
|----------------------|--------------------------------------------------------------------|
|`update_progress`     |Pipeline update state changes (RUNNING, STOPPING, FAILED, COMPLETED)|
|`flow_progress`       |Individual flow state + DQ metrics                                  |
|`create_update`       |New pipeline update initiated                                       |
|`maintenance_progress`|Auto-optimize / maintenance operations                              |
|`hook_progress`       |Event hook execution status (not re-triggered)                      |
|`error`               |Error details on failure                                            |

-----

## 11. Pipeline Configuration Reference

### Pipeline Settings (Declarative Automation Bundles YAML)

```yaml
resources:
  pipelines:
    mobility_pipeline:
      name: mobility_platform_pipeline
      description: "Lakeflow SDP pipeline — mobility platform"

      # Unity Catalog target
      catalog: main
      target: silver                             # schema name

      channel: CURRENT
      edition: ADVANCED                          # CORE | PRO | ADVANCED
      photon: true
      serverless: true                           # Recommended for new pipelines

      # Trigger mode
      continuous: false                          # true = continuous mode
      development: false

      # Source files
      libraries:
        - notebook: {path: /pipelines/bronze_ingestion}
        - notebook: {path: /pipelines/silver_transforms}
        - notebook: {path: /pipelines/gold_aggregates}
        - file:     {path: /pipelines/utils/helpers.py}

      # Cluster config (skipped if serverless: true)
      clusters:
        - label: default
          autoscale:
            min_workers: 2
            max_workers: 8
            mode: ENHANCED
          node_type_id: Standard_D8ds_v4
          spark_conf:
            "spark.databricks.delta.optimizeWrite.enabled": "true"

      # Pipeline parameters
      configuration:
        source.rides.path: "abfss://raw@sa.dfs.core.windows.net/rides/"
        source.schema_location: "/mnt/schema/rides"
        mypipeline.env: "${bundle.target}"
        source_catalog: "main"

      # Notifications
      notifications:
        - email_recipients: ["data-team@company.com"]
          alerts:
            - on-update-failure
            - on-flow-failure
            - on-update-success
```

### Pipeline Editions

|Edition   |Features                                                      |
|----------|--------------------------------------------------------------|
|`CORE`    |Streaming & batch tables, basic DQ                            |
|`PRO`     |+ SCD Type 1 & 2, enhanced DQ metrics                         |
|`ADVANCED`|+ Event log, row filters, column masks, enhanced observability|

-----

## 12. Table Properties & Storage

### Important Delta / SDP Table Properties

|Property                          |Value               |Effect                                   |
|----------------------------------|--------------------|-----------------------------------------|
|`delta.autoOptimize.optimizeWrite`|`true`              |Auto-compact small files on write        |
|`delta.autoOptimize.autoCompact`  |`true`              |Background compaction                    |
|`delta.enableChangeDataFeed`      |`true`              |Enable CDF for downstream CDC            |
|`delta.columnMapping.mode`        |`name`              |Enable column rename/drop without rewrite|
|`delta.targetFileSize`            |`134217728`         |Target file size bytes (128 MB)          |
|`delta.tuneFileSizesForRewrites`  |`true`              |Optimize for merge-heavy (CDC) workloads |
|`pipelines.reset.allowed`         |`false`             |Prevent accidental full refresh          |
|`pipelines.autoOptimize.managed`  |`true`              |Let SDP manage optimization              |
|`quality`                         |`bronze/silver/gold`|Logical medallion tier tag               |

### Liquid Clustering

```python
# Explicit clustering columns
@dp.table(cluster_by=["customer_id", "event_date"])
def silver_rides():
    return spark.readStream.table("bronze_rides")

# Auto clustering — Databricks selects optimal columns
@dp.materialized_view(cluster_by_auto=True)
def gold_daily_revenue():
    return spark.read.table("silver_rides").groupBy(...).agg(...)
```

```sql
CLUSTER BY (customer_id, event_date)       -- Explicit
CLUSTER BY AUTO                             -- Auto (Databricks decides)
```

-----

## 13. Parameterization & Runtime Variables

### Reading Pipeline Config in Python

```python
# Read pipeline configuration (set in pipeline settings or DAB config)
source_path     = spark.conf.get("source.rides.path")
env             = spark.conf.get("mypipeline.env", "dev")      # with default
source_catalog  = spark.conf.get("source_catalog")

@dp.materialized_view
def transaction_summary():
    src_catalog = spark.conf.get("source_catalog")
    return (
        spark.read.table(f"{src_catalog}.sales.transactions")
          .groupBy("account_id")
          .agg(F.count("txn_id").alias("txn_count"))
    )
```

### Reading in SQL

```sql
-- Reference pipeline config as ${key}
CREATE OR REFRESH MATERIALIZED VIEW filtered_rides AS
SELECT * FROM silver_rides
WHERE event_date > '${startDate}'
  AND source_catalog = '${source_catalog}';

-- Use SET for local SQL variables
SET startDate = '2025-01-01';
CREATE OR REFRESH MATERIALIZED VIEW recent AS
SELECT * FROM silver_rides WHERE event_date > '${startDate}';
```

### Parameterized Dynamic Datasets (For Loop Pattern)

```python
# CORRECT: use closure or default arg to capture loop variable
def create_region_bronze(region):
    @dp.table(name=f"bronze_rides_{region.lower()}")
    def bronze_table(r=region):
        return (
            spark.readStream.format("cloudFiles")
              .option("cloudFiles.format", "json")
              .option("cloudFiles.schemaLocation", f"/mnt/schema/rides_{r}")
              .load(f"abfss://raw@sa.dfs.core.windows.net/{r}/")
        )

for region in ["EU", "US", "APAC"]:
    create_region_bronze(region)

# INCORRECT — all tables get the last region's value:
# for region in ["EU","US","APAC"]:
#     @dp.table(name=f"bronze_{region}")
#     def bronze_table():
#         return spark.readStream.load(f".../{region}/")  ← DON'T DO THIS
```

-----

## 14. Design Patterns

### Full Medallion Architecture

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

# ── BRONZE ──────────────────────────────────────────────────────────────────
@dp.table(
    name="bronze_rides",
    comment="Raw rides from Auto Loader",
    table_properties={"quality": "bronze", "pipelines.reset.allowed": "false"},
)
def bronze_rides():
    src = spark.conf.get("source.rides.path")
    schema_loc = spark.conf.get("source.schema_location")
    return (
        spark.readStream.format("cloudFiles")
          .option("cloudFiles.format", "json")
          .option("cloudFiles.schemaLocation", schema_loc)
          .option("cloudFiles.schemaEvolutionMode", "rescue")
          .option("cloudFiles.backfillInterval", "1 week")
          .load(src)
          .withColumn("ingested_at", F.current_timestamp())
          .withColumn("source_file", F.col("_metadata.file_path"))
    )

# ── SILVER ───────────────────────────────────────────────────────────────────
rules = {
    "valid_ride_id":  "ride_id IS NOT NULL",
    "positive_fare":  "fare_amount > 0",
    "valid_status":   "status IN ('completed','cancelled','in_progress')"
}

@dp.expect_all_or_drop(rules)
@dp.table(
    name="silver_rides",
    comment="Cleansed rides",
    partition_cols=["event_date"],
    cluster_by=["customer_id"],
    table_properties={"quality": "silver", "delta.autoOptimize.optimizeWrite": "true"},
)
def silver_rides():
    return (
        spark.readStream.table("bronze_rides")
          .filter(F.col("_rescued_data").isNull())     # Drop rescued rows
          .withColumn("event_date", F.to_date("event_ts"))
          .withColumn("fare_usd", F.round("fare_amount", 2))
    )

# ── GOLD ─────────────────────────────────────────────────────────────────────
@dp.materialized_view(
    name="gold_daily_revenue",
    comment="Daily revenue aggregation",
    cluster_by_auto=True,
    table_properties={"quality": "gold"},
)
def gold_daily_revenue():
    return (
        spark.read.table("silver_rides")
          .groupBy("event_date", "city")
          .agg(
              F.sum("fare_usd").alias("total_revenue"),
              F.count("ride_id").alias("total_rides"),
              F.avg("fare_usd").alias("avg_fare")
          )
    )
```

### Fan-Out (One Source → Multiple Targets)

```python
# One bronze → fan out by event type
@dp.table(name="bronze_events")
def bronze_events():
    return spark.readStream.format("cloudFiles") \
        .option("cloudFiles.format", "json") \
        .load("/volumes/raw/events/")

def create_event_table(event_type):
    @dp.table(name=f"silver_{event_type}s")
    def filtered(et=event_type):
        return spark.readStream.table("bronze_events") \
               .filter(F.col("type") == et)

for et in ["click", "purchase", "view"]:
    create_event_table(et)
```

### Multi-Source Fan-In (Multiple Sources → One Target)

```python
dp.create_streaming_table("rides_global")

@dp.append_flow(target="rides_global", name="eu_flow")
def eu(): return spark.readStream.table("bronze_rides_eu")

@dp.append_flow(target="rides_global", name="us_flow")
def us(): return spark.readStream.table("bronze_rides_us")

@dp.append_flow(target="rides_global", name="apac_flow")
def apac(): return spark.readStream.table("bronze_rides_apac")
```

### Late-Arriving Data with Watermark

```python
@dp.table(name="silver_events_windowed")
def silver_events_windowed():
    return (
        spark.readStream.table("bronze_events")
          .withWatermark("event_ts", "30 minutes")
          .groupBy(F.window("event_ts", "1 hour"), "customer_id")
          .agg(F.count("*").alias("event_count"))
    )
```

### Deduplication

```python
@dp.table(name="silver_deduped")
def silver_deduped():
    return (
        spark.readStream.table("bronze_events")
          .withWatermark("event_ts", "1 hour")
          .dropDuplicatesWithinWatermark(["event_id", "customer_id"])
    )
```

### SCD2 Full Pattern

```python
# Step 1: Target table
dp.create_streaming_table(
    "dim_customers",
    table_properties={"delta.enableChangeDataFeed": "true"},
    schema="""
        customer_id BIGINT NOT NULL,
        name STRING, email STRING, segment STRING, tier STRING,
        __start_at TIMESTAMP, __end_at TIMESTAMP
    """
)

# Step 2: Apply CDC
dp.create_auto_cdc_flow(
    target="dim_customers",
    source="bronze_customers_cdc",
    keys=["customer_id"],
    sequence_by="cdc_timestamp",
    apply_as_deletes=F.expr("op = 'D'"),
    stored_as_scd_type=2,
    track_history_column_list=["email", "segment", "tier"],
    except_column_list=["op", "cdc_timestamp"],
    name="cdc_dim_customers"
)

# Query current rows
# spark.read.table("main.silver.dim_customers").filter("__end_at IS NULL")
```

### Shared Utility Functions

```python
# utils.py (file in pipeline project)
from pyspark.sql import functions as F

def add_audit_cols(df):
    return (
        df.withColumn("processed_at", F.current_timestamp())
          .withColumn("pipeline_env",
              F.lit(spark.conf.get("mypipeline.env", "dev")))
    )

# pipeline notebook
from pyspark import pipelines as dp
from utils import add_audit_cols

@dp.table
def silver_rides():
    return add_audit_cols(spark.readStream.table("bronze_rides"))
```

### Event Hook — DQ Alerting Pattern

```python
from pyspark import pipelines as dp
import requests

@dp.on_event_hook(max_allowable_consecutive_failures=3)
def dq_alert_hook(event):
    if event["event_type"] != "flow_progress":
        return
    details = event.get("details", {}).get("flow_progress", {})
    dq = details.get("data_quality", {})
    dropped = dq.get("dropped_records", 0)
    if dropped and dropped > 1000:
        requests.post(
            "https://hooks.slack.com/services/...",
            json={"text": f"⚠️ DQ Alert: {dropped} rows dropped in {details.get('name')}"}
        )
```

-----

## 15. Databricks Jobs — Full Reference

### Job YAML (Declarative Automation Bundles)

```yaml
resources:
  jobs:
    mobility_etl_job:
      name: mobility_etl_job
      description: "End-to-end mobility ETL"
      tags:
        team: data-engineering
        env: "${bundle.target}"

      schedule:
        quartz_cron_expression: "0 0 6 * * ?"
        timezone_id: "Europe/Berlin"
        pause_status: UNPAUSED

      max_concurrent_runs: 1

      email_notifications:
        on_failure: ["data-team@company.com"]
        on_success: []
        on_start: []
        no_alert_for_skipped_runs: true

      timeout_seconds: 7200

      health:
        rules:
          - metric: RUN_DURATION_SECONDS
            op: GREATER_THAN
            value: 3600

      job_clusters:
        - job_cluster_key: default_cluster
          new_cluster:
            spark_version: "15.4.x-scala2.12"
            node_type_id: Standard_D8ds_v4
            autoscale:
              min_workers: 2
              max_workers: 8
            azure_attributes:
              availability: SPOT_WITH_FALLBACK_AZURE

      tasks:
        - task_key: ingest_raw
          job_cluster_key: default_cluster
          notebook_task:
            notebook_path: /jobs/01_ingest_raw
            base_parameters:
              env: "${bundle.target}"
              date: "{{job.start_time.iso_date}}"
          max_retries: 2
          min_retry_interval_millis: 60000
          timeout_seconds: 1800

        - task_key: run_sdp_pipeline
          depends_on:
            - task_key: ingest_raw
          pipeline_task:
            pipeline_id: "{{resources.pipelines.mobility_pipeline.id}}"
            full_refresh: false

        - task_key: validate_silver
          depends_on:
            - task_key: run_sdp_pipeline
          job_cluster_key: default_cluster
          python_wheel_task:
            package_name: mobility_dq
            entry_point: validate
            parameters: ["--env", "${bundle.target}"]

        - task_key: aggregate_gold
          depends_on:
            - task_key: run_sdp_pipeline
          job_cluster_key: default_cluster
          sql_task:
            query: {query_id: "uuid-of-query"}
            warehouse_id: "abc123"

        - task_key: notify_downstream
          depends_on:
            - task_key: validate_silver
            - task_key: aggregate_gold
          notebook_task:
            notebook_path: /jobs/99_notify
```

-----

## 16. Job Task Types & Parameters

### Task Types Summary

|Task Type    |YAML Key           |Use Case                             |
|-------------|-------------------|-------------------------------------|
|Notebook     |`notebook_task`    |Interactive notebooks with parameters|
|Python Script|`spark_python_task`|`.py` script in DBFS/workspace       |
|Python Wheel |`python_wheel_task`|Packaged Python (`.whl`)             |
|SDP Pipeline |`pipeline_task`    |Trigger a Declarative Pipeline       |
|SQL          |`sql_task`         |SQL query, dashboard, alert          |
|dbt          |`dbt_task`         |dbt project run                      |
|JAR          |`spark_jar_task`   |Scala/Java Spark JAR                 |
|Run Job      |`run_job_task`     |Trigger another Job                  |
|Condition    |`condition_task`   |Branch logic (if/else)               |
|For Each     |`for_each_task`    |Loop over array of inputs            |

### Pipeline Task (SDP)

```yaml
pipeline_task:
  pipeline_id: "abc-123-def-456"
  full_refresh: false         # true = full pipeline recompute
```

### For Each Task (Parallel Fan-Out)

```yaml
- task_key: process_regions
  for_each_task:
    inputs: "[\"EU\",\"US\",\"APAC\"]"
    concurrency: 3
    task:
      task_key: process_single_region
      job_cluster_key: default_cluster
      notebook_task:
        notebook_path: /jobs/process_region
        base_parameters:
          region: "{{input}}"
```

### Condition Task (Branching)

```yaml
- task_key: check_data_available
  condition_task:
    left: "{{tasks.ingest_raw.values.row_count}}"
    op: GREATER_THAN
    right: "0"

- task_key: process_if_data
  depends_on:
    - task_key: check_data_available
      outcome: "true"
```

### Task Values (Inter-Task Communication)

```python
# Task A — produce
dbutils.jobs.taskValues.set(key="row_count", value=df.count())
dbutils.jobs.taskValues.set(key="status",    value="success")

# Task B — consume
row_count = dbutils.jobs.taskValues.get(
    taskKey="ingest_raw",
    key="row_count",
    default=0,
    debugValue=100
)
```

### Dynamic System Variables

|Variable                         |Description                     |
|---------------------------------|--------------------------------|
|`{{job.id}}`                     |Job ID                          |
|`{{job.run_id}}`                 |Current run ID                  |
|`{{job.start_time.iso_date}}`    |Run start date (YYYY-MM-DD)     |
|`{{job.start_time.iso_datetime}}`|Run start datetime              |
|`{{tasks.<key>.values.<n>}}`     |Task value from upstream task   |
|`{{parent_run_id}}`              |Parent run ID (repair runs)     |
|`{{input}}`                      |Current For Each iteration value|

-----

## 17. Job Orchestration Patterns

### Diamond (Parallel → Join)

```
              ┌── validate_silver ──┐
pipeline ─────┤                     ├──── notify
              └── aggregate_gold  ──┘
```

### Conditional Branching

```
ingest ──── check_row_count ──── [>0] ──── process ──── publish
                              └── [=0] ──── log_empty_run
```

### Databricks SDK — Trigger & Monitor

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Trigger job
run = w.jobs.run_now(
    job_id=123456,
    notebook_params={"env": "prd", "date": "2025-06-15"},
    idempotency_token="run_2025-06-15_prd"      # Prevent duplicate runs
)

# Wait for completion
outcome = w.jobs.wait_get_run_job_terminated_or_skipped(run_id=run.run_id)
print(outcome.state.result_state)

# Trigger SDP pipeline directly
update = w.pipelines.start_update(
    pipeline_id="abc-123-def-456",
    full_refresh=False
)
```

-----

## 18. Monitoring, Observability & Alerts

### Pipeline Event Log

```python
# Access via UC (ADVANCED edition)
events = spark.read.table("event_log")  # Available in pipeline context

# Or by path
events = spark.read.format("delta").load("/pipelines/<pipeline_id>/system/events")

# Useful event types: flow_progress, update_progress, create_update, error

# Query DQ metrics
dq_metrics = (
    events
      .filter(F.col("event_type") == "flow_progress")
      .select(
          "timestamp",
          "origin.flow_name",
          F.col("details:flow_progress:data_quality:dropped_records").alias("dropped"),
          F.col("details:flow_progress:data_quality:expectations").alias("expectations")
      )
      .orderBy("timestamp", ascending=False)
)
```

### Lakehouse Monitoring on SDP Tables

```python
from databricks.sdk.service.catalog import MonitorTimeSeries

w = WorkspaceClient()
w.quality_monitors.create(
    table_name="main.silver.silver_rides",
    assets_dir="/Shared/monitors/silver_rides",
    output_schema_name="main.monitoring",
    time_series=MonitorTimeSeries(
        timestamp_col="event_ts",
        granularities=["1 day", "1 week"]
    )
)
```

### Alert via Event Hook (Production Pattern)

```python
from pyspark import pipelines as dp
import requests

@dp.on_event_hook(max_allowable_consecutive_failures=3)
def pipeline_monitor(event):
    et = event.get("event_type")
    if et == "update_progress":
        state = event["details"]["update_progress"]["state"]
        if state in ("FAILED", "CANCELED"):
            requests.post(
                "https://hooks.slack.com/services/xxx",
                json={"text": f"❌ Pipeline {state}: {event['origin']['pipeline_name']}"}
            )
    elif et == "flow_progress":
        dropped = (event.get("details", {})
                       .get("flow_progress", {})
                       .get("data_quality", {})
                       .get("dropped_records", 0))
        if dropped > 500:
            requests.post(
                "https://hooks.slack.com/services/xxx",
                json={"text": f"⚠️ {dropped} rows dropped in {event['origin']['flow_name']}"}
            )
```

-----

## 19. Method & Function Quick-Reference Tables

### `pyspark.pipelines` (dp) — Full API

|Function / Decorator                   |Signature                                                                                                                                                                                                                     |Open Source (Spark 4.1)?|Description               |
|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------|--------------------------|
|`@dp.table`                            |`(name, comment, spark_conf, schema, partition_cols, cluster_by, cluster_by_auto, table_properties, path, row_filter, private)`                                                                                               |✅                       |Define streaming table    |
|`@dp.materialized_view`                |`(name, comment, spark_conf, schema, partition_cols, cluster_by, cluster_by_auto, table_properties, path, refresh_policy, row_filter, private)`                                                                               |✅                       |Define materialized view  |
|`@dp.temporary_view`                   |`(name, comment)`                                                                                                                                                                                                             |✅                       |In-pipeline temporary view|
|`@dp.append_flow`                      |`(target, name, spark_conf, once)`                                                                                                                                                                                            |✅                       |Explicit append flow      |
|`@dp.expect`                           |`(name: str, constraint: str)`                                                                                                                                                                                                |❌                       |Warn on violation         |
|`@dp.expect_or_drop`                   |`(name: str, constraint: str)`                                                                                                                                                                                                |❌                       |Drop row on violation     |
|`@dp.expect_or_fail`                   |`(name: str, constraint: str)`                                                                                                                                                                                                |❌                       |Fail pipeline on violation|
|`@dp.expect_all`                       |`(rules: dict)`                                                                                                                                                                                                               |❌                       |Warn on any violation     |
|`@dp.expect_all_or_drop`               |`(rules: dict)`                                                                                                                                                                                                               |❌                       |Drop row on any violation |
|`@dp.expect_all_or_fail`               |`(rules: dict)`                                                                                                                                                                                                               |❌                       |Fail on any violation     |
|`dp.create_streaming_table`            |`(name, comment, spark_conf, schema, partition_cols, cluster_by, cluster_by_auto, table_properties, path, expect_all, expect_all_or_drop, expect_all_or_fail, row_filter)`                                                    |✅                       |Imperative streaming table|
|`dp.create_auto_cdc_flow`              |`(target, source, keys, sequence_by, ignore_null_updates, apply_as_deletes, apply_as_truncates, column_list, except_column_list, stored_as_scd_type, track_history_column_list, track_history_except_column_list, name, once)`|❌                       |CDC stream processing     |
|`dp.create_auto_cdc_from_snapshot_flow`|`(target, source, keys, stored_as_scd_type, track_history_column_list, track_history_except_column_list)`                                                                                                                     |❌                       |Snapshot-based CDC        |
|`dp.create_sink`                       |`(name: str, format: str, options: dict)`                                                                                                                                                                                     |✅                       |External write target     |
|`@dp.on_event_hook`                    |`(max_allowable_consecutive_failures=None)`                                                                                                                                                                                   |❌                       |Custom event handler      |

### Auto Loader Key Options

|Option                           |Default        |Description                                     |
|---------------------------------|---------------|------------------------------------------------|
|`cloudFiles.format`              |—              |json, csv, parquet, avro, orc, text, binaryFile |
|`cloudFiles.schemaLocation`      |—              |Schema checkpoint path (required for JSON/CSV)  |
|`cloudFiles.inferColumnTypes`    |`false`        |Infer primitive types                           |
|`cloudFiles.schemaEvolutionMode` |`addNewColumns`|addNewColumns / failOnNewColumns / rescue / none|
|`cloudFiles.schemaHints`         |—              |Type hints: `"id BIGINT, ts TIMESTAMP"`         |
|`cloudFiles.maxFilesPerTrigger`  |`1000`         |Max files per micro-batch                       |
|`cloudFiles.maxBytesPerTrigger`  |unbounded      |Max bytes per micro-batch                       |
|`cloudFiles.includeExistingFiles`|`true`         |Backfill existing files on first run            |
|`cloudFiles.useNotifications`    |`false`        |Event-driven file detection                     |
|`cloudFiles.pathGlobFilter`      |—              |Glob file filter                                |
|`cloudFiles.backfillInterval`    |—              |Periodic rescan interval                        |
|`cloudFiles.allowOverwrites`     |`false`        |Reprocess overwritten files                     |

### `dbutils` Job Methods

|Method                                                          |Description                    |
|----------------------------------------------------------------|-------------------------------|
|`dbutils.widgets.text(name, default)`                           |Declare text parameter         |
|`dbutils.widgets.dropdown(name, default, choices)`              |Declare dropdown parameter     |
|`dbutils.widgets.get(name)`                                     |Get parameter value            |
|`dbutils.widgets.getAll()`                                      |Get all parameters as dict     |
|`dbutils.jobs.taskValues.set(key, value)`                       |Set inter-task value           |
|`dbutils.jobs.taskValues.get(taskKey, key, default, debugValue)`|Get value from upstream task   |
|`dbutils.notebook.exit(value)`                                  |Exit notebook with return value|
|`dbutils.notebook.run(path, timeout, arguments)`                |Run child notebook             |

### Databricks SDK — Pipelines & Jobs Methods

|Method                                                 |Description             |
|-------------------------------------------------------|------------------------|
|`w.jobs.create(**settings)`                            |Create a job            |
|`w.jobs.run_now(job_id, **params)`                     |Trigger job run         |
|`w.jobs.get_run(run_id)`                               |Get run details         |
|`w.jobs.list_runs(job_id, limit)`                      |List runs               |
|`w.jobs.cancel_run(run_id)`                            |Cancel active run       |
|`w.jobs.repair_run(run_id, rerun_tasks)`               |Repair failed run       |
|`w.jobs.wait_get_run_job_terminated_or_skipped(run_id)`|Poll until completion   |
|`w.pipelines.create(**settings)`                       |Create SDP pipeline     |
|`w.pipelines.start_update(pipeline_id, full_refresh)`  |Trigger pipeline update |
|`w.pipelines.get_update(pipeline_id, update_id)`       |Get update status       |
|`w.pipelines.stop(pipeline_id)`                        |Stop continuous pipeline|
|`w.pipelines.delete(pipeline_id)`                      |Delete pipeline         |

-----

## 20. Best Practices

### New API Adoption

- **Replace `import dlt` with `from pyspark import pipelines as dp`** — the old module still works but is deprecated and won’t receive new features.
- **Use `@dp.materialized_view` for all batch reads** — using `@dp.table` with a batch read still works but is ambiguous and not recommended.
- **Rename `@dlt.view` → `@dp.temporary_view`** — `@dp.view` does not exist in the new module.
- **Replace `dlt.apply_changes()` → `dp.create_auto_cdc_flow()`** — the new name, plus the `name` and `once` parameters.

### Pipeline Design

- **Set `pipelines.reset.allowed = false`** on Silver/Gold to prevent accidental full refreshes in production.
- **Prefer `CLUSTER BY AUTO`** for Gold materialized views unless you have well-defined query patterns.
- **Use `private=True`** for intermediate staging tables that add unnecessary clutter in Unity Catalog.
- **Separate ingestion (Bronze) and transformation (Silver/Gold)** into different DAGs for independent scheduling and failure isolation.
- **Never call `collect()`, `count()`, `toPandas()`, or `saveAsTable()`** inside dataset-defining functions — SDP evaluates code multiple times during planning.
- **Avoid side effects** in `@dp.table` / `@dp.materialized_view` functions — use event hooks for monitoring instead.

### Auto Loader

- **Always set `cloudFiles.schemaLocation`** for JSON/CSV — without it schema is re-inferred each run.
- **Use `cloudFiles.schemaEvolutionMode = rescue`** for untrusted sources and monitor `_rescued_data`.
- **Set `cloudFiles.backfillInterval`** (e.g., `1 week`) as a safety net for any missed files.
- **Add `_metadata` columns** (file path, modification time) to Bronze tables for lineage and debugging.
- **Prefer `read_files()` in SQL** over `cloud_files()` — it’s the new standard syntax.

### Data Quality

- **Quarantine over drop** in production — use a parallel quarantine table to preserve bad records for investigation.
- **Name expectations descriptively** — names appear verbatim in DQ dashboards and the event log.
- **Use `expect_or_fail` only for critical failures** — e.g., primary key is null, not for minor data issues.
- **Define DQ rules as a dict** (`expect_all_*`) for reusability and consistent enforcement across tables.
- **Use `@dp.on_event_hook`** to trigger alerts when dropped record counts exceed thresholds.

### Flows & CDC

- **Use `@dp.append_flow` over `dbutils.notebook.run` loops** for fan-in patterns — it handles checkpointing correctly.
- **Give every flow a unique `name`** — this is the checkpoint key; renaming breaks continuity.
- **Use `once=True` on append flows for one-time backfills** — avoids triggering checkpoints on historical data repeatedly.
- **Always specify `except_column_list`** in CDC flows to exclude CDC metadata columns (`op`, `ts_ms`) from the target.

### Jobs & Orchestration

- **Set `max_concurrent_runs: 1`** to prevent overlapping runs on stateful pipelines.
- **Use `idempotency_token`** in API calls to prevent duplicate runs on retry.
- **Pass dates as parameters** — never hardcode dates in notebooks.
- **Use For Each tasks** for fan-out patterns; avoid `dbutils.notebook.run` loops.
- **Configure `health` rules** (`RUN_DURATION_SECONDS`) to alert on unexpectedly slow jobs.

### Compute & Cost

- **Use serverless compute** for new SDP pipelines — no cluster configuration, auto-scaling, lowest latency.
- **Enable Photon** for classic cluster pipelines with heavy aggregations.
- **Use `SPOT_WITH_FALLBACK`** nodes for job clusters to reduce costs.
- **Use `CLUSTER BY AUTO`** to reduce manual tuning effort on Gold tables.

-----

## 21. Common Pitfalls & Fixes

|Problem                                                |Root Cause                                  |Fix                                                            |
|-------------------------------------------------------|--------------------------------------------|---------------------------------------------------------------|
|`AttributeError: module 'dlt' has no attribute 'table'`|Using old `dlt` import pattern              |Use `from pyspark import pipelines as dp`                      |
|`@dp.table` on batch read produces unexpected results  |`@dp.table` is for streaming only in new API|Use `@dp.materialized_view` for batch reads                    |
|`@dp.view` not found                                   |`view` was renamed                          |Use `@dp.temporary_view`                                       |
|Schema inference fails every run                       |Missing `cloudFiles.schemaLocation`         |Always set schema location path                                |
|Full refresh wipes production Silver                   |`pipelines.reset.allowed` not set           |Set to `false` on Silver/Gold                                  |
|Duplicate rows in streaming target                     |No deduplication                            |Add `.dropDuplicatesWithinWatermark(["id"])`                   |
|Late data missing from aggregation                     |No watermark on streaming join              |Add `.withWatermark("ts", "30 minutes")`                       |
|SCD2 shows multiple current rows                       |Missing filter on current records           |Query with `WHERE __end_at IS NULL`                            |
|`apply_changes` not found                              |Old DLT function name                       |Replace with `dp.create_auto_cdc_flow()`                       |
|Sink not receiving data                                |Using CDC flow to write to sink             |Only `@dp.append_flow` can target sinks                        |
|Delta sink table not found                             |Unqualified table name in sink options      |Always use `catalog.schema.table` 3-part name                  |
|For loop tables all read last value                    |Late binding in Python loop                 |Use closure or default argument: `def fn(x=x)`                 |
|Event hook fires on `hook_progress` events             |Circular dependency risk                    |SDP prevents this automatically                                |
|Hook silently disabled                                 |Too many consecutive failures               |Set `max_allowable_consecutive_failures` higher or fix the hook|
|Pipeline stuck in initializing                         |Bad cluster config or init script           |Check cluster event log in pipeline UI                         |
|Job triggered twice on API retry                       |No idempotency token                        |Pass `idempotency_token` in `run_now()` call                   |
|Task value `KeyError`                                  |Upstream task didn’t set value              |Use `default=` and `debugValue=` in `taskValues.get`           |
|DLT table not visible in UC                            |`catalog` / `target` not set                |Set both in pipeline settings / DAB config                     |

-----

## API Quick-Reference: DLT → SDP One-Liner

```python
# FIND & REPLACE CHEAT SHEET
import dlt                         →  from pyspark import pipelines as dp
@dlt.table(...)                    →  @dp.table(...)                # streaming only
@dlt.table(...) + batch read       →  @dp.materialized_view(...)
@dlt.view(...)                     →  @dp.temporary_view(...)
@dlt.append_flow(...)              →  @dp.append_flow(...)
@dlt.expect(...)                   →  @dp.expect(...)               (same)
@dlt.expect_or_drop(...)           →  @dp.expect_or_drop(...)       (same)
@dlt.expect_or_fail(...)           →  @dp.expect_or_fail(...)       (same)
@dlt.expect_all(...)               →  @dp.expect_all(...)           (same)
@dlt.expect_all_or_drop(...)       →  @dp.expect_all_or_drop(...)   (same)
@dlt.expect_all_or_fail(...)       →  @dp.expect_all_or_fail(...)   (same)
dlt.read(...)                      →  spark.read.table(...)
dlt.read_stream(...)               →  spark.readStream.table(...)
dlt.create_streaming_table(...)    →  dp.create_streaming_table(...)
dlt.apply_changes(...)             →  dp.create_auto_cdc_flow(...)
dlt.apply_changes_from_snapshot(.) →  dp.create_auto_cdc_from_snapshot_flow(...)
dlt.create_sink(...)               →  dp.create_sink(...)
LIVE.<table>   (SQL)               →  <table>   (no prefix needed)
STREAM(LIVE.<t>)  (SQL)            →  STREAM(<t>)
cloud_files(...) (SQL)             →  read_files(...)  or  STREAM read_files(...)
```

-----

*Reference card covers: Lakeflow Spark Declarative Pipelines (SDP) · `pyspark.pipelines` module · Apache Spark 4.1 Open Source API · Databricks Runtime 15.4+ · Lakeflow Release 2025.x · Declarative Automation Bundles (DABs) · Unity Catalog*
