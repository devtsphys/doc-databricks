# Databricks Jobs, Workflows & Orchestration — Reference Card

> **Covers:** Databricks Jobs · Workflows · Lakeflow Spark Declarative Pipelines (SDP / formerly DLT) · AutoLoader · Structured Streaming · Ingestion · Orchestration Patterns · Best Practices
> **Runtime:** Databricks Runtime 14.x+ / Unity Catalog compatible · **Updated:** Lakeflow GA (2025/2026)

-----

## Table of Contents

1. [Databricks Jobs](#1-databricks-jobs)
1. [Workflows (Multi-Task Jobs)](#2-workflows-multi-task-jobs)
1. [Delta Live Tables — Declarative Pipelines (Legacy `dlt` API)](#3-delta-live-tables--declarative-pipelines-legacy-dlt-api)
1. [Lakeflow Spark Declarative Pipelines (SDP — `dp` API)](#4-lakeflow-spark-declarative-pipelines-sdp--dp-api)
1. [AutoLoader (cloudFiles)](#5-autoloader-cloudfiles)
1. [Structured Streaming](#6-structured-streaming)
1. [Ingestion Patterns](#7-ingestion-patterns)
1. [Workflow Orchestration Design Patterns](#8-workflow-orchestration-design-patterns)
1. [Job & Task Configuration Reference](#9-job--task-configuration-reference)
1. [Method & API Reference Tables](#10-method--api-reference-tables)
1. [Best Practices](#11-best-practices)
1. [Quick Examples](#12-quick-examples)

-----

## 1. Databricks Jobs

### Core Concepts

|Concept        |Description                                                                                                 |
|---------------|------------------------------------------------------------------------------------------------------------|
|**Job**        |A non-interactive way to run code (notebooks, JARs, Python scripts, SQL, dbt, etc.) on a schedule or trigger|
|**Run**        |A single execution instance of a job                                                                        |
|**Task**       |A unit of work within a job (jobs can be single- or multi-task)                                             |
|**Cluster**    |Compute attached to a task — new job cluster, existing cluster, or serverless                               |
|**Trigger**    |What starts the job: schedule (cron), file arrival, continuous, manual, or API                              |
|**Job Cluster**|Ephemeral cluster created for the job and terminated after; cheapest for non-interactive workloads          |

### Job Trigger Types

|Trigger       |Use Case                            |Config Key                       |
|--------------|------------------------------------|---------------------------------|
|`CRON`        |Scheduled, time-based               |`schedule.quartz_cron_expression`|
|`CONTINUOUS`  |Re-runs immediately after completion|`continuous.pause_status`        |
|`FILE_ARRIVAL`|Fires when new files land in a path |`trigger.file_arrival.url`       |
|`TABLE`       |Fires when a Delta table is updated |`trigger.table.table_names`      |
|`MANUAL`      |Via UI / API / CLI only             |*(no trigger block)*             |

### Job Cluster vs All-Purpose Cluster

|              |Job Cluster              |All-Purpose        |
|--------------|-------------------------|-------------------|
|**Cost**      |Lower (DBU rate)         |Higher             |
|**Startup**   |~5–8 min (cold)          |Instant if warm    |
|**State**     |Ephemeral                |Persistent         |
|**Best for**  |Scheduled production jobs|Development, ad-hoc|
|**Serverless**|✅ Instant start          |✅                  |

### Retry & Error Handling

```yaml
# Job-level retry settings
max_retries: 3
min_retry_interval_millis: 60000   # 1 minute
retry_on_timeout: true
timeout_seconds: 3600

# Task-level override
task:
  max_retries: 1
  run_if: "ALL_SUCCESS"            # ALL_SUCCESS | ALL_DONE | AT_LEAST_ONE_SUCCESS | AT_LEAST_ONE_FAILED | NONE_FAILED
```

### Run Conditions (`run_if`)

|Value                   |Meaning                                   |
|------------------------|------------------------------------------|
|`ALL_SUCCESS`           |All upstream tasks succeeded (default)    |
|`ALL_DONE`              |All upstream tasks finished (any result)  |
|`AT_LEAST_ONE_SUCCESS`  |≥1 upstream succeeded                     |
|`AT_LEAST_ONE_FAILED`   |≥1 upstream failed (error-handler pattern)|
|`NONE_FAILED`           |All succeeded or were skipped             |
|`NONE_FAILED_OR_SKIPPED`|All succeeded                             |

-----

## 2. Workflows (Multi-Task Jobs)

### DAG Structure

```
Task A ──┬──► Task C ──► Task E  (final)
         │
Task B ──┘──► Task D
```

Tasks form a **Directed Acyclic Graph (DAG)**. Dependencies are declared with `depends_on`.

### Task Types

|Task Type            |Runtime         |Key Config                               |
|---------------------|----------------|-----------------------------------------|
|**Notebook**         |Any DBR         |`notebook_task.notebook_path`            |
|**Python Script**    |DBR / Serverless|`python_wheel_task` / `spark_python_task`|
|**Python Wheel**     |DBR             |`python_wheel_task.package_name`         |
|**JAR**              |DBR             |`spark_jar_task.main_class_name`         |
|**SQL**              |SQL Warehouse   |`sql_task.query`                         |
|**dbt**              |SQL Warehouse   |`dbt_task.project_directory`             |
|**Delta Live Tables**|DLT Runtime     |`pipeline_task.pipeline_id`              |
|**Run Job**          |—               |`run_job_task.job_id`                    |
|**Condition**        |—               |`condition_task.op` (logic gate)         |
|**For Each**         |—               |`for_each_task` (loop over inputs)       |

### Condition Task (Logic Gate)

```json
{
  "task_key": "check_row_count",
  "condition_task": {
    "left": "{{tasks.ingest.values.row_count}}",
    "op": "GREATER_THAN",
    "right": "0"
  }
}
```

Operators: `EQUAL_TO` · `NOT_EQUAL` · `GREATER_THAN` · `GREATER_THAN_OR_EQUAL` · `LESS_THAN` · `LESS_THAN_OR_EQUAL`

### For Each Task (Dynamic Loops)

```json
{
  "task_key": "process_each_file",
  "for_each_task": {
    "inputs": "{{tasks.list_files.values.files}}",
    "concurrency": 5,
    "task": {
      "task_key": "process_file",
      "notebook_task": {
        "notebook_path": "/pipelines/process_single_file",
        "base_parameters": {"file": "{{input}}"}
      }
    }
  }
}
```

### Task Values (Cross-Task Communication)

```python
# Producer task — set a value
dbutils.jobs.taskValues.set(key="row_count", value=df.count())
dbutils.jobs.taskValues.set(key="status",    value="LOADED")

# Consumer task — read upstream value
count = dbutils.jobs.taskValues.get(
    taskKey="ingest_task",
    key="row_count",
    default=0,
    debugValue=999   # used in interactive runs
)
```

### Job Parameters

```python
# Retrieve job-level parameters in notebook
dbutils.widgets.text("env", "dev")
env = dbutils.widgets.get("env")

# Or via Widgets API
dbutils.widgets.dropdown("mode", "full", ["full", "incremental"])
```

### Repair & Re-run

|Action          |When to use                                                       |
|----------------|------------------------------------------------------------------|
|**Repair Run**  |Re-run only failed/skipped tasks without rerunning succeeded tasks|
|**Full Re-run** |Start from scratch                                                |
|**Repair + Fix**|Fix root cause, repair run — preserves lineage                    |

-----

## 3. Delta Live Tables — Declarative Pipelines (Legacy `dlt` API)

> **Note:** DLT is now **Lakeflow Spark Declarative Pipelines (SDP)**. The `dlt` module still works with full backward compatibility, but Databricks recommends migrating to the new `pyspark.pipelines` (`dp`) API. See [Section 4](#4-lakeflow-spark-declarative-pipelines-sdp--dp-api) for the current API.

### Core Concepts

|Concept              |Description                                                   |
|---------------------|--------------------------------------------------------------|
|**Pipeline**         |A DLT graph of datasets; runs on managed DLT clusters         |
|**Streaming Table**  |Processes new data incrementally from a stream                |
|**Materialized View**|Recomputed from full source on each update                    |
|**View**             |Logical alias; not materialized, used within the pipeline only|
|**Expectation**      |Data quality rule; controls what happens on violation         |
|**Target Schema**    |Unity Catalog schema where tables are published               |

### Dataset Decorators (Python)

```python
import dlt
from pyspark.sql.functions import *

# Streaming Table — append-only ingestion
@dlt.table(
    name="bronze_events",
    comment="Raw events from S3",
    table_properties={"pipelines.reset.allowed": "true"},
    partition_cols=["event_date"]
)
def bronze_events():
    return (
        spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.schemaLocation", "/checkpoints/bronze_events_schema")
            .load("s3://bucket/raw/events/")
    )

# Materialized View — recomputed each run
@dlt.table(comment="Daily event aggregates")
def silver_event_agg():
    return (
        dlt.read("bronze_events")
           .groupBy("event_date", "event_type")
           .agg(count("*").alias("cnt"))
    )

# Logical View — only visible within pipeline
@dlt.view
def cleaned_events():
    return dlt.read("bronze_events").filter(col("event_type").isNotNull())
```

### DLT SQL Syntax

```sql
-- Streaming Table
CREATE OR REFRESH STREAMING TABLE bronze_events
COMMENT "Raw events from cloud storage"
AS SELECT * FROM cloud_files(
  "s3://bucket/raw/events/",
  "json",
  map("cloudFiles.inferColumnTypes", "true")
);

-- Materialized View
CREATE OR REFRESH MATERIALIZED VIEW silver_event_agg AS
SELECT event_date, event_type, COUNT(*) AS cnt
FROM LIVE.bronze_events
GROUP BY event_date, event_type;

-- Logical View
CREATE LIVE VIEW cleaned_events AS
SELECT * FROM LIVE.bronze_events WHERE event_type IS NOT NULL;
```

### DLT Expectations (Data Quality)

```python
# Warn on violation (record retained, metric tracked)
@dlt.expect("valid_event_type", "event_type IS NOT NULL")

# Drop violating records
@dlt.expect_or_drop("non_null_user", "user_id IS NOT NULL")

# Fail the pipeline on violation
@dlt.expect_or_fail("critical_check", "amount > 0")

# Multiple expectations
@dlt.expect_all({
    "valid_date":  "event_date >= '2020-01-01'",
    "valid_event": "event_type IN ('click','view','purchase')"
})

@dlt.expect_all_or_drop({
    "non_null_id":  "user_id IS NOT NULL",
    "positive_amt": "amount > 0"
})

@dlt.expect_all_or_fail({
    "schema_check": "user_id IS NOT NULL AND event_type IS NOT NULL"
})
```

### DLT Table Properties

|Property                        |Effect                          |
|--------------------------------|--------------------------------|
|`pipelines.reset.allowed`       |Allow full refresh of this table|
|`pipelines.autoOptimize.managed`|Auto-optimize enabled           |
|`delta.targetFileSize`          |Target file size in bytes       |
|`quality`                       |Tag: `bronze`, `silver`, `gold` |

### DLT Pipeline Modes

|Mode                    |Description                                                  |
|------------------------|-------------------------------------------------------------|
|**Triggered**           |Runs once, processes all new data, then stops                |
|**Continuous**          |Runs indefinitely, processes data as it arrives (low latency)|
|**Enhanced Autoscaling**|Automatically scales workers during triggered runs           |

### DLT `apply_changes` (CDC / SCD)

```python
# CDC with APPLY CHANGES (Type 1 — Upsert)
dlt.create_streaming_table("silver_customers")

dlt.apply_changes(
    target="silver_customers",
    source="bronze_cdc_feed",
    keys=["customer_id"],
    sequence_by=col("_commit_timestamp"),
    apply_as_deletes=expr("op = 'DELETE'"),
    apply_as_truncates=expr("op = 'TRUNCATE'"),
    except_column_list=["op", "_commit_timestamp"],
    stored_as_scd_type=1  # or 2
)

# SCD Type 2 — keeps history
dlt.apply_changes(
    target="silver_customers_history",
    source="bronze_cdc_feed",
    keys=["customer_id"],
    sequence_by="_commit_timestamp",
    stored_as_scd_type=2,
    track_history_except_column_list=["_metadata"]
)
```

-----

## 4. Lakeflow Spark Declarative Pipelines (SDP — `dp` API)

> **What is SDP?** Lakeflow Spark Declarative Pipelines (SDP) is the evolution of Delta Live Tables, now built on open-source **Apache Spark Declarative Pipelines** (Spark 4.1+). It is Generally Available as of 2025. The core Python API moved from `import dlt` → `from pyspark import pipelines as dp`. All existing DLT code continues to work without changes.

### Lakeflow vs DLT vs Apache Spark — Name Mapping

|Concept                |Old DLT syntax                        |New SDP syntax (`dp`)                       |In Apache Spark OSS|
|-----------------------|--------------------------------------|--------------------------------------------|-------------------|
|Import                 |`import dlt`                          |`from pyspark import pipelines as dp`       |✅ Same             |
|Streaming Table        |`@dlt.table` + `readStream`           |`@dp.table()`                               |✅                  |
|Materialized View      |`@dlt.table` + `read`                 |`@dp.materialized_view()`                   |✅                  |
|Temporary View         |`@dlt.view`                           |`@dp.temporary_view()`                      |✅                  |
|Append Flow            |`@dlt.append_flow`                    |`@dp.append_flow()`                         |✅                  |
|CDC / AUTO CDC         |`dlt.apply_changes(...)`              |`dp.create_auto_cdc_flow(...)`              |❌ Databricks only  |
|Snapshot CDC           |`dlt.apply_changes_from_snapshot(...)`|`dp.create_auto_cdc_from_snapshot_flow(...)`|❌ Databricks only  |
|Expectations           |`@dlt.expect(...)`                    |`@dp.expect(...)`                           |❌ Databricks only  |
|Sink                   |`dlt.create_sink(...)`                |`dp.create_sink(...)`                       |✅                  |
|Event log              |`spark.read.table("event_log")`       |`spark.read.table("event_log")`             |❌ Databricks only  |
|SQL — Streaming Table  |`CREATE OR REFRESH STREAMING TABLE`   |`CREATE STREAMING TABLE`                    |✅                  |
|SQL — Materialized View|`CREATE OR REFRESH MATERIALIZED VIEW` |`CREATE MATERIALIZED VIEW`                  |✅                  |
|SQL — Flow             |*(not supported)*                     |`CREATE FLOW ... INSERT INTO`               |✅                  |
|Reference dataset      |`dlt.read("table")`                   |`spark.read.table("table")` (in-pipeline)   |✅                  |


> **Key rule:** `@dp.table()` = streaming table (uses `readStream`). `@dp.materialized_view()` = batch table (uses `read`). The read type determines the dataset type — not a separate config.

-----

### `dp` Decorator Reference

#### `@dp.table()` — Streaming Table

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, current_timestamp, to_date

@dp.table(
    name="bronze_events",            # optional: defaults to function name
    comment="Raw events from S3",
    partition_cols=["event_date"],   # optional partition columns
    table_properties={               # optional Delta/pipeline properties
        "pipelines.reset.allowed": "true",
        "quality": "bronze"
    },
    path="s3://bucket/tables/bronze_events",  # optional external location
    schema="event_id STRING, user_id STRING, event_ts TIMESTAMP"  # optional explicit schema
)
def bronze_events():
    return (
        spark.readStream                    # readStream → Streaming Table
            .format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.schemaLocation", "/dlt/schema/events")
            .load("s3://bucket/landing/events/")
            .withColumn("event_date", to_date(col("event_ts")))
            .withColumn("_ingest_ts", current_timestamp())
    )
```

#### `@dp.materialized_view()` — Materialized View

```python
@dp.materialized_view(
    name="gold_daily_counts",        # optional: defaults to function name
    comment="Daily event counts",
    partition_cols=["event_date"],
    table_properties={"quality": "gold"}
)
def gold_daily_counts():
    return (
        spark.read.table("silver_events")  # spark.read (not readStream) → Materialized View
             .groupBy("event_date", "event_type")
             .count()
    )
```

#### `@dp.temporary_view()` — Temporary View

```python
# Visible only within the pipeline; not persisted to Unity Catalog
@dp.temporary_view(name="cleaned_events")
def cleaned_events():
    return (
        spark.readStream.table("bronze_events")
             .filter(col("event_type").isNotNull())
    )
```

#### `@dp.append_flow()` — Explicit Append Flow

```python
# Use when multiple sources feed into a single target streaming table
# First: declare the target table (no flow defined inline)
@dp.table(name="all_events")
def all_events():
    return spark.readStream.table("events_us")  # default/primary flow

# Then: add additional flows into the same target
@dp.append_flow(target="all_events", name="flow_eu_events")
def append_eu_events():
    return spark.readStream.table("events_eu")

@dp.append_flow(target="all_events", name="flow_apac_events")
def append_apac_events():
    return spark.readStream.table("events_apac")
```

-----

### Data Quality Expectations (`dp`)

```python
# Warn — record retained, metric tracked in pipeline event log
@dp.expect("valid_event_type", "event_type IS NOT NULL")
@dp.table()
def silver_events():
    return spark.readStream.table("bronze_events")

# Drop — violating records silently dropped
@dp.expect_or_drop("valid_user", "user_id IS NOT NULL")
@dp.table()
def silver_events_clean():
    return spark.readStream.table("bronze_events")

# Fail — pipeline fails on any violation
@dp.expect_or_fail("critical_schema", "event_id IS NOT NULL")
@dp.table()
def bronze_events_validated():
    return spark.readStream.table("raw_events")

# Multiple expectations (dict)
@dp.expect_all({
    "valid_date":   "event_date >= '2020-01-01'",
    "valid_type":   "event_type IN ('click','view','purchase')"
})
@dp.table()
def validated_events():
    return spark.readStream.table("bronze_events")

@dp.expect_all_or_drop({
    "non_null_id": "user_id IS NOT NULL",
    "positive_amt": "amount > 0"
})
@dp.table()
def clean_transactions():
    return spark.readStream.table("bronze_transactions")
```

-----

### AUTO CDC — Change Data Capture (`dp`)

```python
# AUTO CDC replaces dlt.apply_changes()
# Step 1: Declare the target streaming table
@dp.table(name="silver_customers")
def silver_customers():
    pass  # target declared; AUTO CDC flow writes into it

# Step 2: Define the CDC flow
dp.create_auto_cdc_flow(
    target="silver_customers",
    source="bronze_cdc_feed",
    keys=["customer_id"],
    sequence_by="_commit_timestamp",       # ordering column
    apply_as_deletes="op = 'DELETE'",      # expression for delete rows
    apply_as_truncates="op = 'TRUNCATE'",  # expression for truncate
    except_column_list=["op", "_commit_timestamp"],
    stored_as_scd_type=1                   # 1 (upsert) or 2 (history)
)

# SCD Type 2 — keeps full history with __START_AT / __END_AT columns
dp.create_auto_cdc_flow(
    target="silver_customers_history",
    source="bronze_cdc_feed",
    keys=["customer_id"],
    sequence_by="_commit_timestamp",
    stored_as_scd_type=2,
    track_history_except_column_list=["_metadata"]
)

# Multiple CDC flows into one target (2025.30+)
dp.create_auto_cdc_flow(
    name="main_feed_cdc",               # named flow — allows multiples per target
    target="silver_customers",
    source="bronze_main_feed",
    keys=["customer_id"],
    sequence_by="_ts"
)
dp.create_auto_cdc_flow(
    name="correction_feed_cdc",
    target="silver_customers",          # same target — new in 2025.30
    source="bronze_corrections_feed",
    keys=["customer_id"],
    sequence_by="_ts"
)
```

### AUTO CDC from Snapshot

```python
# For sources that deliver full snapshots (not CDC deltas)
@dp.table(name="silver_products")
def silver_products():
    pass

dp.create_auto_cdc_from_snapshot_flow(
    target="silver_products",
    source="bronze_product_snapshot",
    keys=["product_id"],
    stored_as_scd_type=2
)
```

-----

### Sinks — Writing to External Systems

```python
# Write stream to Kafka (not a Unity Catalog table)
dp.create_sink(
    name="kafka_orders_sink",
    format="kafka",
    options={
        "kafka.bootstrap.servers": "broker:9092",
        "topic": "silver-orders"
    }
)

@dp.append_flow(target="kafka_orders_sink")
def write_orders_to_kafka():
    return (
        spark.readStream.table("silver_orders")
             .select(col("order_id").cast("string").alias("key"),
                     to_json(struct("*")).alias("value"))
    )

# Delta sink (external Delta table not managed by the pipeline)
dp.create_sink(
    name="external_delta_sink",
    format="delta",
    options={"path": "s3://external/delta/orders"}
)
```

-----

### SDP SQL Syntax

```sql
-- Streaming Table (ingestion from AutoLoader)
CREATE STREAMING TABLE bronze_orders
COMMENT "Raw orders from landing zone"
TBLPROPERTIES ('quality' = 'bronze', 'pipelines.reset.allowed' = 'true')
AS SELECT *, current_timestamp() AS _ingest_ts
FROM STREAM(cloud_files(
    "s3://bucket/orders/",
    "json",
    map("cloudFiles.inferColumnTypes", "true")
));

-- Streaming Table (from another pipeline table)
CREATE STREAMING TABLE silver_customers
AS SELECT customer_id, name, email
FROM STREAM(bronze_customers)
WHERE customer_id IS NOT NULL;

-- Materialized View
CREATE MATERIALIZED VIEW gold_order_summary
COMMENT "Aggregated order metrics"
TBLPROPERTIES ('quality' = 'gold')
AS SELECT
    order_date,
    COUNT(*)        AS total_orders,
    SUM(amount)     AS total_revenue
FROM silver_orders
GROUP BY order_date;

-- Explicit Append Flow (multi-source fan-in)
CREATE STREAMING TABLE all_events;

CREATE FLOW us_events_flow
INSERT INTO all_events
AS SELECT * FROM STREAM(events_us);

CREATE FLOW eu_events_flow
INSERT INTO all_events
AS SELECT * FROM STREAM(events_eu);

-- One-time backfill flow
CREATE FLOW backfill_historical
INSERT INTO all_events ONCE
AS SELECT * FROM historical_events_archive;

-- AUTO CDC (SCD Type 1)
CREATE STREAMING TABLE silver_customers;
APPLY CHANGES INTO silver_customers
FROM STREAM(bronze_cdc)
KEYS (customer_id)
SEQUENCE BY _commit_timestamp
DELETE WHEN op = 'DELETE';

-- AUTO CDC (SCD Type 2)
CREATE STREAMING TABLE silver_customers_history;
APPLY CHANGES INTO silver_customers_history
FROM STREAM(bronze_cdc)
KEYS (customer_id)
SEQUENCE BY _commit_timestamp
STORED AS SCD TYPE 2;
```

-----

### `dp` API Reference Table

|Function / Decorator                     |Signature                                                                                                                                                  |Notes                                                            |
|-----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
|`@dp.table()`                            |`(name, comment, partition_cols, table_properties, path, schema, temporary)`                                                                               |Creates a Streaming Table; function must return `readStream` DF  |
|`@dp.materialized_view()`                |`(name, comment, partition_cols, table_properties, path, schema)`                                                                                          |Creates a Materialized View; function must return batch `read` DF|
|`@dp.temporary_view()`                   |`(name, comment)`                                                                                                                                          |Temporary/logical view; not published to UC                      |
|`@dp.append_flow()`                      |`(target, name, comment, spark_conf, libraries)`                                                                                                           |Explicit append flow; for multi-source fan-in                    |
|`@dp.expect()`                           |`(name, constraint)`                                                                                                                                       |Warn on violation; record retained                               |
|`@dp.expect_or_drop()`                   |`(name, constraint)`                                                                                                                                       |Drop violating records                                           |
|`@dp.expect_or_fail()`                   |`(name, constraint)`                                                                                                                                       |Fail pipeline on violation                                       |
|`@dp.expect_all()`                       |`({name: constraint, ...})`                                                                                                                                |Multiple warn expectations                                       |
|`@dp.expect_all_or_drop()`               |`({name: constraint, ...})`                                                                                                                                |Multiple drop expectations                                       |
|`@dp.expect_all_or_fail()`               |`({name: constraint, ...})`                                                                                                                                |Multiple fail expectations                                       |
|`dp.create_auto_cdc_flow()`              |`(target, source, keys, sequence_by, apply_as_deletes, apply_as_truncates, except_column_list, stored_as_scd_type, track_history_except_column_list, name)`|CDC flow; Databricks only                                        |
|`dp.create_auto_cdc_from_snapshot_flow()`|`(target, source, keys, stored_as_scd_type)`                                                                                                               |Snapshot CDC; Databricks only                                    |
|`dp.create_sink()`                       |`(name, format, options)`                                                                                                                                  |External streaming sink (Kafka, Delta)                           |
|`dp.create_streaming_table()`            |`(name, comment, schema, partition_cols, path, table_properties, expect_all, expect_all_or_drop, expect_all_or_fail)`                                      |Programmatic table declaration                                   |

-----

### SDP ETL Pipeline — Full Bronze → Silver → Gold Example

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, current_timestamp, to_date, count, sum as _sum

# ── BRONZE: raw AutoLoader ingest ───────────────────────────────────────────
@dp.table(
    name="bronze_orders",
    comment="Raw orders — AutoLoader JSON ingest",
    table_properties={"quality": "bronze", "pipelines.reset.allowed": "true"},
    partition_cols=["order_date"]
)
def bronze_orders():
    return (
        spark.readStream
            .format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.schemaLocation", "/dlt/schema/orders")
            .option("cloudFiles.inferColumnTypes", "true")
            .option("cloudFiles.schemaEvolutionMode", "rescue")
            .load("s3://data-lake/landing/orders/")
            .withColumn("order_date", to_date(col("order_ts")))
            .withColumn("_ingest_ts", current_timestamp())
            .withColumn("_source_file", col("_metadata.file_path"))
    )

# ── SILVER: clean + validate + deduplicate ──────────────────────────────────
@dp.expect_or_drop("valid_order_id", "order_id IS NOT NULL")
@dp.expect_or_drop("valid_customer",  "customer_id IS NOT NULL")
@dp.expect("positive_amount",         "amount > 0")
@dp.table(
    name="silver_orders",
    comment="Cleaned and validated orders",
    partition_cols=["order_date"],
    table_properties={"quality": "silver"}
)
def silver_orders():
    return (
        spark.readStream.table("bronze_orders")
             .dropDuplicates(["order_id"])
             .select(
                 "order_id", "customer_id",
                 col("amount").cast("double"),
                 col("order_ts").cast("timestamp"),
                 "order_date", "status",
                 "_ingest_ts"
             )
    )

# ── GOLD: aggregated business metrics ──────────────────────────────────────
@dp.materialized_view(
    name="gold_daily_revenue",
    comment="Daily revenue and order counts",
    table_properties={"quality": "gold"}
)
def gold_daily_revenue():
    return (
        spark.read.table("silver_orders")       # batch read → Materialized View
             .filter(col("status") == "COMPLETED")
             .groupBy("order_date")
             .agg(
                 count("order_id").alias("order_count"),
                 _sum("amount").alias("total_revenue")
             )
    )
```

-----

### SDP Pipeline Configuration Modes

|Mode                      |Description                                    |Use Case                          |
|--------------------------|-----------------------------------------------|----------------------------------|
|**Triggered**             |Processes all new data once, then stops        |Scheduled batch ETL, job-triggered|
|**Continuous**            |Runs indefinitely, low latency                 |Near-real-time streaming          |
|**Serverless Standard**   |Serverless triggered; 26% better TCO vs classic|Most production workloads         |
|**Serverless Performance**|Serverless triggered; optimized for tight SLAs |Latency-sensitive workloads       |
|**Enhanced Autoscaling**  |Auto-scales workers during triggered runs      |Variable/bursty data volumes      |

### Key SDP Differences vs Manual Spark/Streaming

|                      |Manual Spark + Streaming |Lakeflow SDP                        |
|----------------------|-------------------------|------------------------------------|
|Retry logic           |Manual try/catch         |Auto: task → flow → pipeline        |
|Ordering / parallelism|Manual DAG               |Automatic from dependency graph     |
|CDC / MERGE           |Custom foreachBatch MERGE|`create_auto_cdc_flow`              |
|Data quality          |Custom assertions        |`@dp.expect*` with event log        |
|Incremental MV        |Manual watermarks        |Built-in incremental engine         |
|Checkpoint mgmt       |Manual, per-stream       |Managed by pipeline                 |
|Schema evolution      |Manual schema hints      |Automatic with `schemaEvolutionMode`|

-----

## 5. AutoLoader (`cloudFiles`)

### Basic AutoLoader Read

```python
df = (
    spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")          # json | csv | parquet | avro | orc | text | binaryFile
        .option("cloudFiles.schemaLocation", "/checkpoints/schema/")
        .option("cloudFiles.inferColumnTypes", "true")
        .option("cloudFiles.schemaEvolutionMode", "addNewColumns")  # default
        .load("s3://bucket/landing/")
)
```

### AutoLoader Options Reference

|Option                           |Default        |Description                                                |
|---------------------------------|---------------|-----------------------------------------------------------|
|`cloudFiles.format`              |*(required)*   |Source file format                                         |
|`cloudFiles.schemaLocation`      |*(required)*   |Path to persist inferred schema                            |
|`cloudFiles.inferColumnTypes`    |`false`        |Infer exact types (not all-string)                         |
|`cloudFiles.schemaEvolutionMode` |`addNewColumns`|`addNewColumns` · `rescue` · `failOnNewColumns` · `none`   |
|`cloudFiles.schemaHints`         |—              |Override specific column types: `"id LONG, ts TIMESTAMP"`  |
|`cloudFiles.maxFilesPerTrigger`  |`1000`         |Limit files processed per micro-batch                      |
|`cloudFiles.maxBytesPerTrigger`  |unlimited      |Limit bytes per micro-batch                                |
|`cloudFiles.useNotifications`    |`false`        |Use cloud event notifications instead of listing (scalable)|
|`cloudFiles.includeExistingFiles`|`true`         |Process files already present at start                     |
|`cloudFiles.backfillInterval`    |—              |Periodic re-scan interval (e.g., `1 day`)                  |
|`cloudFiles.allowOverwrites`     |`false`        |Reprocess updated files                                    |
|`cloudFiles.fileNamePattern`     |—              |Glob filter: `*.json`                                      |
|`cloudFiles.pathRewrites`        |—              |Path prefix mapping                                        |
|`cloudFiles.resourceTags`        |—              |Cloud notification tags                                    |
|`recursiveFileLookup`            |`false`        |Traverse subdirectories                                    |

### Schema Evolution Modes

|Mode              |New Column Behavior                   |Missing Column|
|------------------|--------------------------------------|--------------|
|`addNewColumns`   |Added to schema automatically         |NULL          |
|`rescue`          |Rescued to `_rescued_data` JSON column|NULL          |
|`failOnNewColumns`|Stream fails                          |NULL          |
|`none`            |Ignored silently                      |NULL          |

### AutoLoader with Schema Hints

```python
df = (
    spark.readStream.format("cloudFiles")
        .option("cloudFiles.format", "csv")
        .option("cloudFiles.schemaLocation", "/chk/schema")
        .option("cloudFiles.schemaHints", "amount DOUBLE, event_ts TIMESTAMP")
        .option("header", "true")
        .load("s3://bucket/csv-landing/")
)
```

### AutoLoader Rescue Column

```python
# Rescued data contains rows/columns that didn't fit the schema
df = spark.readStream.format("cloudFiles") \
    .option("cloudFiles.schemaEvolutionMode", "rescue") \
    .option("cloudFiles.schemaLocation", "/chk/schema") \
    .option("cloudFiles.format", "json") \
    .load("s3://bucket/events/")

# Inspect rescued records
df.filter(col("_rescued_data").isNotNull()).display()
```

### AutoLoader Metadata Columns

|Column                            |Description             |
|----------------------------------|------------------------|
|`_metadata.file_path`             |Full path of source file|
|`_metadata.file_name`             |File name only          |
|`_metadata.file_size`             |File size in bytes      |
|`_metadata.file_modification_time`|Last modified timestamp |
|`_metadata.file_block_start`      |Block start offset      |
|`_metadata.file_block_length`     |Block length            |

```python
df = df.withColumn("source_file", col("_metadata.file_path")) \
       .withColumn("ingest_ts",   current_timestamp())
```

-----

## 6. Structured Streaming

### Core Stream Operations

```python
# Read stream
stream_df = spark.readStream.format("delta").table("raw.events")

# Transformations (same as batch)
processed = stream_df \
    .filter(col("event_type").isNotNull()) \
    .withColumn("hour", date_trunc("hour", col("event_ts")))

# Write stream
query = processed.writeStream \
    .format("delta") \
    .outputMode("append")  \       # append | complete | update
    .option("checkpointLocation", "/checkpoints/events_silver/") \
    .option("mergeSchema", "true") \
    .trigger(availableNow=True) \  # see trigger types below
    .toTable("silver.events")
```

### Trigger Types

|Trigger                        |Code                                   |Behavior                                                |
|-------------------------------|---------------------------------------|--------------------------------------------------------|
|**Default** (micro-batch)      |`.trigger()` omitted                   |Runs as fast as possible                                |
|**Fixed interval**             |`.trigger(processingTime="30 seconds")`|Runs every N seconds                                    |
|**Once** *(deprecated)*        |`.trigger(once=True)`                  |Processes all available, stops                          |
|**Available Now** *(preferred)*|`.trigger(availableNow=True)`          |Processes all available in multiple micro-batches, stops|
|**Continuous**                 |`.trigger(continuous="1 second")`      |Low-latency, epoch-based (~ms latency)                  |

### Output Modes

|Mode      |When to use                                             |
|----------|--------------------------------------------------------|
|`append`  |New rows only; no updates to existing rows              |
|`complete`|Full result table rewritten each batch (aggregations)   |
|`update`  |Only changed rows output (aggregations, supported sinks)|

### Watermarking & Late Data

```python
# Watermark: tolerate late events up to 10 minutes
windowed = stream_df \
    .withWatermark("event_ts", "10 minutes") \
    .groupBy(
        window(col("event_ts"), "5 minutes"),
        col("event_type")
    ) \
    .count()
```

### Streaming Joins

```python
# Stream-Stream join (requires watermark)
impressions = spark.readStream.table("bronze.impressions") \
    .withWatermark("imp_ts", "2 hours")

clicks = spark.readStream.table("bronze.clicks") \
    .withWatermark("click_ts", "3 hours")

joined = impressions.join(
    clicks,
    expr("""
        imp_id = click_imp_id AND
        click_ts BETWEEN imp_ts AND imp_ts + INTERVAL 1 HOUR
    """),
    "leftOuter"
)

# Stream-Static join (no watermark needed on static)
static_users = spark.read.table("silver.users")
enriched = stream_df.join(static_users, "user_id", "left")
```

### `foreachBatch` — Arbitrary Sink Logic

```python
def process_batch(batch_df, batch_id):
    batch_df.persist()
    
    # Write to multiple targets
    batch_df.write.format("delta").mode("append").saveAsTable("silver.events")
    
    batch_df.filter(col("event_type") == "error") \
            .write.format("delta").mode("append").saveAsTable("silver.errors")
    
    # Upsert via MERGE
    batch_df.createOrReplaceTempView("updates")
    batch_df.sparkSession.sql("""
        MERGE INTO silver.customers t
        USING updates s ON t.id = s.id
        WHEN MATCHED THEN UPDATE SET *
        WHEN NOT MATCHED THEN INSERT *
    """)
    
    batch_df.unpersist()

query = stream_df.writeStream \
    .foreachBatch(process_batch) \
    .option("checkpointLocation", "/chk/events/") \
    .trigger(availableNow=True) \
    .start()
```

### Stream Checkpoint Management

```python
# Force checkpoint reset (use sparingly)
dbutils.fs.rm("/checkpoints/my_stream/", recurse=True)

# Query progress
query.lastProgress       # dict with metrics
query.status             # current status
query.recentProgress     # last N progress reports
query.stop()             # gracefully stop

# Wait for termination
query.awaitTermination()
query.awaitTermination(timeout=300)  # 5 min timeout
```

-----

## 7. Ingestion Patterns

### Medallion Architecture

```
[Source] → Bronze (raw, append-only) → Silver (cleaned, deduplicated) → Gold (aggregated, business logic)
```

|Layer     |Format              |Expectations    |Retention   |
|----------|--------------------|----------------|------------|
|**Bronze**|Delta, append-only  |None / warn     |Long (years)|
|**Silver**|Delta, Type 1 upsert|Drop/warn       |Medium      |
|**Gold**  |Delta, aggregate    |Fail on critical|Short-medium|

### COPY INTO (Batch Ingestion)

```sql
-- Idempotent batch load; tracks loaded files in Delta log
COPY INTO silver.events
FROM 's3://bucket/landing/events/'
FILEFORMAT = JSON
FORMAT_OPTIONS ('inferSchema' = 'true', 'mergeSchema' = 'true')
COPY_OPTIONS ('mergeSchema' = 'true', 'force' = 'false');  -- force=true reloads all
```

### MERGE (Upsert Pattern)

```python
from delta.tables import DeltaTable

target = DeltaTable.forName(spark, "silver.customers")

target.alias("t").merge(
    source=updates_df.alias("s"),
    condition="t.customer_id = s.customer_id"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .whenNotMatchedBySourceDelete() \  # optional: delete orphans
 .execute()
```

### Change Data Capture (CDC) with MERGE

```python
def upsert_to_delta(micro_batch_df, batch_id):
    target = DeltaTable.forName(spark, "silver.orders")
    
    # Deduplicate within batch (keep latest by sequence)
    deduped = micro_batch_df \
        .withColumn("rank", rank().over(
            Window.partitionBy("order_id").orderBy(desc("_commit_ts"))
        )).filter(col("rank") == 1).drop("rank")
    
    target.alias("t").merge(
        deduped.alias("s"),
        "t.order_id = s.order_id"
    ).whenMatchedUpdate(
        condition="s.op != 'D'",
        set={"status": "s.status", "updated_at": "s._commit_ts"}
    ).whenMatchedDelete(
        condition="s.op = 'D'"
    ).whenNotMatchedInsert(
        condition="s.op != 'D'",
        values={"order_id": "s.order_id", "status": "s.status"}
    ).execute()
```

### Streaming from Kafka

```python
kafka_df = (
    spark.readStream
        .format("kafka")
        .option("kafka.bootstrap.servers", "broker:9092")
        .option("subscribe", "topic1,topic2")           # or subscribePattern
        .option("startingOffsets", "earliest")          # earliest | latest | json offsets
        .option("maxOffsetsPerTrigger", 100000)
        .option("kafka.security.protocol", "SASL_SSL")
        .load()
)

# Deserialize value
parsed = kafka_df.select(
    col("key").cast("string"),
    from_json(col("value").cast("string"), schema).alias("data"),
    col("topic"),
    col("partition"),
    col("offset"),
    col("timestamp")
).select("key", "data.*", "topic", "timestamp")
```

### Delta Live Tables Ingestion (Recommended)

```python
# Bronze — AutoLoader in DLT
@dlt.table
def bronze_orders():
    return (
        spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.schemaLocation", "/dlt/schema/orders")
            .load("s3://bucket/orders/")
    )

# Silver — clean + deduplicate
@dlt.table
@dlt.expect_or_drop("valid_order_id", "order_id IS NOT NULL")
def silver_orders():
    return (
        dlt.read_stream("bronze_orders")
           .dropDuplicates(["order_id", "updated_at"])
           .select("order_id", "customer_id", "amount", "status",
                   to_timestamp("updated_at").alias("updated_at"))
    )
```

-----

## 8. Workflow Orchestration Design Patterns

### Pattern 1: Fan-Out / Fan-In

```
               ┌──► task_region_A ──┐
ingest_task ───┼──► task_region_B ──┼──► aggregate_task
               └──► task_region_C ──┘
```

```json
{"task_key": "task_region_A", "depends_on": [{"task_key": "ingest_task"}]},
{"task_key": "aggregate_task", "depends_on": [
    {"task_key": "task_region_A"},
    {"task_key": "task_region_B"},
    {"task_key": "task_region_C"}
]}
```

### Pattern 2: Conditional Branching

```
ingest ──► check_quality ──[SUCCESS]──► transform
                         └─[FAILURE]──► notify_failure
```

```json
{"task_key": "transform",      "run_if": "ALL_SUCCESS", "depends_on": [{"task_key": "check_quality"}]},
{"task_key": "notify_failure", "run_if": "AT_LEAST_ONE_FAILED", "depends_on": [{"task_key": "check_quality"}]}
```

### Pattern 3: Dynamic Parallel Processing (For Each)

```python
# List task outputs a JSON array; For Each task processes each element in parallel
# Use case: process N files, N partitions, N customers
{
  "task_key": "for_each_customer",
  "for_each_task": {
    "inputs": "{{tasks.list_customers.values.customer_ids}}",
    "concurrency": 10,
    "task": { ... }
  }
}
```

### Pattern 4: ELT with Repair

```
extract ──► load ──► transform ──► validate ──► notify
             │                          │
             └─────── repair ◄──────────┘  (run_if: AT_LEAST_ONE_FAILED)
```

### Pattern 5: Trigger Downstream Job

```python
# From within a notebook: trigger another job and wait
import requests

token  = dbutils.secrets.get("my-scope", "job-token")
job_id = 12345

response = requests.post(
    "https://<workspace>.azuredatabricks.net/api/2.1/jobs/run-now",
    headers={"Authorization": f"Bearer {token}"},
    json={"job_id": job_id, "notebook_params": {"env": "prod"}}
)
run_id = response.json()["run_id"]
```

### Pattern 6: DLT Pipeline as Workflow Task

```json
{
  "task_key": "run_dlt_pipeline",
  "pipeline_task": {
    "pipeline_id": "abc-123-def",
    "full_refresh": false
  }
}
```

### Pattern 7: Job-as-a-Service (Run Job Task)

```json
{
  "task_key": "call_shared_utility",
  "run_job_task": {
    "job_id": 67890,
    "job_run_parameters": {
      "named_parameters": {"input_table": "silver.events", "date": "{{job.start_time.iso_date}}"}
    }
  }
}
```

### Pattern 8: File Arrival Trigger with Idempotency

```yaml
trigger:
  file_arrival:
    url: "s3://bucket/landing/"
    min_time_between_triggers_seconds: 300
    wait_after_last_change_seconds: 60

# Notebook uses COPY INTO for idempotent load
# Track processing in a control table
```

-----

## 9. Job & Task Configuration Reference

### Job Parameters & Dynamic Values

|Expression                         |Resolves to                  |
|-----------------------------------|-----------------------------|
|`{{job.id}}`                       |Job ID                       |
|`{{job.run_id}}`                   |Current run ID               |
|`{{job.start_time}}`               |Run start time (epoch ms)    |
|`{{job.start_time.iso_date}}`      |`YYYY-MM-DD`                 |
|`{{job.start_time.iso_datetime}}`  |`YYYY-MM-DDTHH:mm:ss`        |
|`{{tasks.<task_key>.values.<key>}}`|Task value from upstream task|
|`{{tasks.<task_key>.run_id}}`      |Run ID of a specific task    |

### Notification Settings

```json
{
  "email_notifications": {
    "on_start":   ["oncall@company.com"],
    "on_success": [],
    "on_failure": ["team@company.com", "oncall@company.com"],
    "no_alert_for_skipped_runs": true
  },
  "webhook_notifications": {
    "on_failure": [{"id": "webhook-uuid"}]
  }
}
```

### Cluster Policy & Permissions

```json
{
  "new_cluster": {
    "spark_version":    "14.3.x-scala2.12",
    "node_type_id":     "Standard_DS3_v2",
    "num_workers":       4,
    "policy_id":        "cluster-policy-abc",
    "autoscale": {"min_workers": 2, "max_workers": 10},
    "spark_conf": {
      "spark.databricks.delta.optimizeWrite.enabled": "true",
      "spark.databricks.delta.autoCompact.enabled":   "true"
    },
    "azure_attributes": {
      "availability":      "SPOT_WITH_FALLBACK_AZURE",
      "spot_bid_max_price": -1
    }
  }
}
```

### Serverless Task Configuration

```json
{
  "task_key":        "serverless_notebook",
  "notebook_task":   {"notebook_path": "/jobs/my_notebook"},
  "environment_key": "Default",
  "new_cluster":     null
}
```

### Libraries

```json
{
  "libraries": [
    {"pypi":  {"package": "great-expectations==0.18.0"}},
    {"maven": {"coordinates": "io.delta:delta-core_2.12:2.4.0"}},
    {"whl":   "dbfs:/libs/my_package-1.0-py3-none-any.whl"},
    {"jar":   "dbfs:/libs/my_lib.jar"},
    {"notebook": {"path": "/Shared/utils"}}
  ]
}
```

-----

## 10. Method & API Reference Tables

### Jobs REST API 2.1

|Method  |Endpoint                         |Description                |
|--------|---------------------------------|---------------------------|
|`POST`  |`/api/2.1/jobs/create`           |Create a new job           |
|`GET`   |`/api/2.1/jobs/get?job_id=X`     |Get job definition         |
|`POST`  |`/api/2.1/jobs/update`           |Partial update job         |
|`POST`  |`/api/2.1/jobs/reset`            |Full replace job definition|
|`DELETE`|`/api/2.1/jobs/delete`           |Delete job                 |
|`GET`   |`/api/2.1/jobs/list`             |List all jobs              |
|`POST`  |`/api/2.1/jobs/run-now`          |Trigger a run              |
|`GET`   |`/api/2.1/jobs/runs/list`        |List runs                  |
|`GET`   |`/api/2.1/jobs/runs/get?run_id=X`|Get run details            |
|`POST`  |`/api/2.1/jobs/runs/cancel`      |Cancel a run               |
|`POST`  |`/api/2.1/jobs/runs/repair`      |Repair failed run          |
|`GET`   |`/api/2.1/jobs/runs/get-output`  |Get task output            |
|`DELETE`|`/api/2.1/jobs/runs/delete`      |Delete run record          |

### Pipelines REST API (DLT)

|Method  |Endpoint                               |Description          |
|--------|---------------------------------------|---------------------|
|`POST`  |`/api/2.0/pipelines`                   |Create pipeline      |
|`PUT`   |`/api/2.0/pipelines/{id}`              |Update pipeline      |
|`DELETE`|`/api/2.0/pipelines/{id}`              |Delete pipeline      |
|`GET`   |`/api/2.0/pipelines/{id}`              |Get pipeline spec    |
|`POST`  |`/api/2.0/pipelines/{id}/updates`      |Start an update (run)|
|`GET`   |`/api/2.0/pipelines/{id}/updates/{uid}`|Get update status    |
|`GET`   |`/api/2.0/pipelines/{id}/events`       |Get pipeline events  |
|`GET`   |`/api/2.0/pipelines`                   |List pipelines       |

### DLT Python API

|Function                         |Signature                                                                             |Description                                      |
|---------------------------------|--------------------------------------------------------------------------------------|-------------------------------------------------|
|`dlt.table`                      |`@dlt.table(name, comment, table_properties, partition_cols, path, schema, temporary)`|Define a materialized table or streaming table   |
|`dlt.view`                       |`@dlt.view(name, comment)`                                                            |Define a temporary view                          |
|`dlt.expect`                     |`dlt.expect(name, constraint)`                                                        |Warn on violation                                |
|`dlt.expect_or_drop`             |`dlt.expect_or_drop(name, constraint)`                                                |Drop violating rows                              |
|`dlt.expect_or_fail`             |`dlt.expect_or_fail(name, constraint)`                                                |Fail pipeline on violation                       |
|`dlt.expect_all`                 |`dlt.expect_all({name: constraint, ...})`                                             |Multiple warn expectations                       |
|`dlt.expect_all_or_drop`         |`dlt.expect_all_or_drop({...})`                                                       |Multiple drop expectations                       |
|`dlt.expect_all_or_fail`         |`dlt.expect_all_or_fail({...})`                                                       |Multiple fail expectations                       |
|`dlt.read`                       |`dlt.read("table_name")`                                                              |Read a DLT dataset (batch)                       |
|`dlt.read_stream`                |`dlt.read_stream("table_name")`                                                       |Read a DLT dataset (stream)                      |
|`dlt.create_streaming_table`     |`dlt.create_streaming_table(name, ...)`                                               |Programmatically declare target for apply_changes|
|`dlt.apply_changes`              |`dlt.apply_changes(target, source, keys, sequence_by, ...)`                           |CDC processing                                   |
|`dlt.apply_changes_from_snapshot`|`dlt.apply_changes_from_snapshot(...)`                                                |Snapshot-based CDC                               |

### Delta Table Python API (DeltaTable)

|Method                                 |Description                     |
|---------------------------------------|--------------------------------|
|`DeltaTable.forName(spark, "db.table")`|Reference by name               |
|`DeltaTable.forPath(spark, "/path")`   |Reference by path               |
|`DeltaTable.create(spark)`             |Create via builder              |
|`DeltaTable.createIfNotExists(spark)`  |Idempotent create               |
|`.alias(name)`                         |Alias for merge                 |
|`.merge(source, condition)`            |Initiate MERGE                  |
|`.whenMatchedUpdate(set=...)`          |Update on match                 |
|`.whenMatchedUpdateAll()`              |Update all cols on match        |
|`.whenMatchedDelete()`                 |Delete on match                 |
|`.whenNotMatchedInsert(values=...)`    |Insert when not matched         |
|`.whenNotMatchedInsertAll()`           |Insert all cols when not matched|
|`.whenNotMatchedBySourceDelete()`      |Delete target rows not in source|
|`.execute()`                           |Run the MERGE                   |
|`.toDF()`                              |Read as DataFrame               |
|`.detail()`                            |Table metadata                  |
|`.history(n)`                          |Transaction log history         |
|`.vacuum(retentionHours)`              |Remove old files                |
|`.optimize()`                          |Trigger OPTIMIZE                |
|`.restoreToVersion(n)`                 |Time travel restore             |
|`.restoreToTimestamp(ts)`              |Time travel restore             |
|`.generate("symlink_format_manifest")` |Generate manifest               |

### `dbutils` Reference

|Method                                                          |Description           |
|----------------------------------------------------------------|----------------------|
|`dbutils.widgets.text(name, default)`                           |Text input widget     |
|`dbutils.widgets.dropdown(name, default, choices)`              |Dropdown widget       |
|`dbutils.widgets.get(name)`                                     |Get widget value      |
|`dbutils.widgets.remove(name)`                                  |Remove widget         |
|`dbutils.widgets.removeAll()`                                   |Remove all widgets    |
|`dbutils.jobs.taskValues.set(key, value)`                       |Set cross-task value  |
|`dbutils.jobs.taskValues.get(taskKey, key, default, debugValue)`|Get cross-task value  |
|`dbutils.notebook.run(path, timeout, args)`                     |Run child notebook    |
|`dbutils.notebook.exit(value)`                                  |Exit with return value|
|`dbutils.secrets.get(scope, key)`                               |Get secret            |
|`dbutils.secrets.list(scope)`                                   |List secrets          |
|`dbutils.fs.ls(path)`                                           |List files            |
|`dbutils.fs.cp(from, to, recurse)`                              |Copy file(s)          |
|`dbutils.fs.mv(from, to)`                                       |Move/rename           |
|`dbutils.fs.rm(path, recurse)`                                  |Delete                |
|`dbutils.fs.mkdirs(path)`                                       |Create directory      |

### Structured Streaming Methods

|Method                               |Description               |
|-------------------------------------|--------------------------|
|`spark.readStream.format(...)`       |Create streaming source   |
|`.option(key, value)`                |Set source option         |
|`.schema(schema)`                    |Set explicit schema       |
|`.table("name")`                     |Read Delta table as stream|
|`.load(path)`                        |Load from path            |
|`df.writeStream.format(...)`         |Create streaming sink     |
|`.outputMode(mode)`                  |Set output mode           |
|`.option("checkpointLocation", path)`|Set checkpoint            |
|`.trigger(...)`                      |Set trigger               |
|`.partitionBy(cols)`                 |Partition output          |
|`.toTable("name")`                   |Write to Delta table      |
|`.start(path)`                       |Start to path             |
|`.foreachBatch(func)`                |Custom batch processing   |
|`.foreach(writer)`                   |Row-level custom sink     |
|`.queryName(name)`                   |Name the query            |
|`query.stop()`                       |Stop query                |
|`query.awaitTermination()`           |Block until stopped       |
|`query.lastProgress`                 |Latest progress dict      |
|`query.status`                       |Current status            |
|`spark.streams.active`               |List active queries       |
|`spark.streams.awaitAnyTermination()`|Wait for any query to stop|

-----

## 11. Best Practices

### Jobs & Workflows

- **Use Job Clusters** for production jobs to minimize cost and avoid interference with interactive workloads.
- **Use Serverless** for short-lived, bursty tasks — instant start, no cluster management.
- **Set timeouts** on every task to prevent runaway jobs consuming DBUs.
- **Parameterize everything** — use job parameters and task values instead of hardcoded paths/dates.
- **Use Repair Run** after failures to avoid re-running already-succeeded tasks.
- **Name tasks descriptively** — `ingest_orders`, not `task_1`.
- **Separate concerns** — one notebook per logical step; avoid monolithic notebooks.
- **Store secrets in Databricks Secrets** — never in notebooks or job configs.
- **Tag jobs** with `team`, `env`, `project` for cost attribution.
- **Use `run_if: ALL_DONE`** on notification tasks so they always fire.
- **Idempotency** — design every task to be safely re-runnable.
- **For Each concurrency** — tune `concurrency` to avoid overwhelming downstream systems.

### Delta Live Tables

- **Use DLT for complex multi-hop pipelines** — it handles restarts, checkpoints, and ordering automatically.
- **Prefer Streaming Tables over Materialized Views** for large, append-only sources — incremental processing is far cheaper.
- **Use `@dlt.expect_or_drop`** at Silver layer to keep data clean without failing pipelines.
- **Use `@dlt.expect_or_fail`** only for truly critical checks (e.g., schema integrity, PII compliance).
- **Avoid `LIVE.` references in SQL** unless necessary — prefer `dlt.read()` in Python for refactorability.
- **Use `full_refresh=False`** in production triggers — only full-refresh during major schema changes.
- **Use `apply_changes`** for CDC rather than custom MERGE logic — it handles ordering and compaction.
- **Declare `table_properties`** explicitly for lifecycle management.
- **Use `temporary=True`** for intermediate tables not needed downstream.

### AutoLoader

- **Always set `schemaLocation`** — this prevents schema re-inference on restart.
- **Use `cloudFiles.useNotifications=true`** for high-volume buckets (thousands of files/hour) — avoids listing overhead.
- **Set `cloudFiles.maxFilesPerTrigger`** to control micro-batch size and cluster load.
- **Use `rescue` mode** in early development; switch to `addNewColumns` in production.
- **Keep schema hints minimal** — over-specifying causes failures on unexpected data.
- **Store checkpoints close to data** — use cloud storage, not DBFS `dbfs:/` root for production.
- **Use `_metadata` columns** to track provenance end-to-end.

### Structured Streaming

- **Never delete checkpoints** in production — use `restoreToVersion` on the Delta table instead.
- **Use `availableNow=True` trigger** instead of deprecated `once=True`.
- **Use `foreachBatch`** for complex sinks (multiple outputs, upserts, external calls).
- **Persist `batch_df`** in `foreachBatch` if you read it multiple times.
- **Watermarks are required** for stream-stream joins and stateful operations.
- **Monitor `query.lastProgress`** — track `inputRowsPerSecond`, `processedRowsPerSecond`, `numInputRows`.
- **Use `spark.streams.active`** to manage multiple concurrent queries.
- **Separate checkpoint per query** — never share checkpoint directories.

### Ingestion & Data Quality

- **Use COPY INTO** for one-time or infrequent batch loads — idempotent by default.
- **Use AutoLoader** for continuous or near-real-time ingestion — scalable and managed.
- **Deduplicate at Silver** using `dropDuplicates` or window functions in `foreachBatch`.
- **Partition Bronze by date** (`event_date`) for efficient downstream reads and deletion.
- **Use Z-ORDER** on frequently filtered columns at Silver/Gold (`OPTIMIZE ... ZORDER BY (customer_id)`).
- **Enable Auto Optimize** on high-write tables: `delta.autoOptimize.optimizeWrite=true`, `autoCompact=true`.
- **Use `VACUUM`** regularly — default 7-day retention; never reduce below streaming query lag.

-----

## 12. Quick Examples

### Example 1: Full Bronze → Silver Pipeline with AutoLoader + DLT

```python
import dlt
from pyspark.sql.functions import col, current_timestamp, to_date

# ── Bronze: raw ingest ──────────────────────────────────────────────────────
@dlt.table(
    name="bronze_clickstream",
    comment="Raw clickstream events from S3 landing zone",
    table_properties={"quality": "bronze", "pipelines.reset.allowed": "true"}
)
def bronze_clickstream():
    return (
        spark.readStream
            .format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.schemaLocation", "/dlt/schema/clickstream")
            .option("cloudFiles.inferColumnTypes", "true")
            .option("cloudFiles.schemaEvolutionMode", "rescue")
            .load("s3://data-lake/landing/clickstream/")
            .withColumn("_ingest_ts",    current_timestamp())
            .withColumn("_source_file",  col("_metadata.file_path"))
            .withColumn("event_date",    to_date(col("event_ts")))
    )

# ── Silver: clean + validate ────────────────────────────────────────────────
@dlt.table(
    name="silver_clickstream",
    comment="Cleaned and validated clickstream events",
    partition_cols=["event_date"],
    table_properties={"quality": "silver"}
)
@dlt.expect("valid_event_ts",   "event_ts IS NOT NULL")
@dlt.expect_or_drop("valid_user", "user_id IS NOT NULL AND user_id != ''")
@dlt.expect_or_drop("valid_type", "event_type IN ('click','view','search','purchase','add_to_cart')")
def silver_clickstream():
    return (
        dlt.read_stream("bronze_clickstream")
           .dropDuplicates(["event_id"])
           .select(
               "event_id", "user_id", "session_id",
               "event_type", "page_url", "product_id",
               col("event_ts").cast("timestamp"),
               "event_date", "_ingest_ts"
           )
    )

# ── Gold: daily aggregates ──────────────────────────────────────────────────
@dlt.table(
    name="gold_daily_event_counts",
    comment="Daily event type counts",
    table_properties={"quality": "gold"}
)
def gold_daily_event_counts():
    return (
        dlt.read("silver_clickstream")
           .groupBy("event_date", "event_type")
           .count()
           .withColumnRenamed("count", "event_count")
    )
```

### Example 2: Incremental Streaming Job with foreachBatch MERGE

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col, rank, current_timestamp
from pyspark.sql.window import Window

def upsert_customers(batch_df, batch_id):
    """Deduplicate within batch and MERGE into silver table."""
    if batch_df.isEmpty():
        return
    
    batch_df.persist()
    
    # Keep latest record per customer in this batch
    w = Window.partitionBy("customer_id").orderBy(col("updated_at").desc())
    deduped = (
        batch_df
            .withColumn("rn", rank().over(w))
            .filter(col("rn") == 1)
            .drop("rn")
            .withColumn("_silver_ts", current_timestamp())
    )
    
    DeltaTable.forName(spark, "silver.customers") \
        .alias("t") \
        .merge(deduped.alias("s"), "t.customer_id = s.customer_id") \
        .whenMatchedUpdateAll() \
        .whenNotMatchedInsertAll() \
        .execute()
    
    batch_df.unpersist()


query = (
    spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "parquet")
        .option("cloudFiles.schemaLocation", "/chk/schema/customers")
        .load("s3://bucket/cdc/customers/")
        .writeStream
        .foreachBatch(upsert_customers)
        .option("checkpointLocation", "/chk/silver_customers/")
        .trigger(availableNow=True)
        .start()
)

query.awaitTermination()
print(f"Processed in {query.lastProgress.get('batchDuration', 0)}ms")
```

### Example 3: Multi-Task Workflow YAML (Databricks Asset Bundle)

```yaml
# databricks.yml (DAB resource definition)
resources:
  jobs:
    etl_pipeline:
      name: "ETL Pipeline — Orders"
      
      schedule:
        quartz_cron_expression: "0 0 6 * * ?"   # 06:00 UTC daily
        timezone_id: "UTC"
        pause_status: UNPAUSED

      email_notifications:
        on_failure: ["data-team@company.com"]
      
      job_clusters:
        - job_cluster_key: main_cluster
          new_cluster:
            spark_version: "14.3.x-scala2.12"
            node_type_id:  "Standard_DS3_v2"
            autoscale:
              min_workers: 2
              max_workers: 8
      
      tasks:
        - task_key: ingest_orders
          job_cluster_key: main_cluster
          notebook_task:
            notebook_path: /jobs/ingest_orders
            base_parameters:
              env:  "${var.env}"
              date: "{{job.start_time.iso_date}}"

        - task_key: validate_quality
          depends_on: [{task_key: ingest_orders}]
          job_cluster_key: main_cluster
          notebook_task:
            notebook_path: /jobs/validate_orders
          max_retries: 0

        - task_key: transform_silver
          depends_on: [{task_key: validate_quality}]
          run_if: ALL_SUCCESS
          job_cluster_key: main_cluster
          notebook_task:
            notebook_path: /jobs/transform_orders

        - task_key: handle_failure
          depends_on:
            - task_key: validate_quality
            - task_key: transform_silver
          run_if: AT_LEAST_ONE_FAILED
          job_cluster_key: main_cluster
          notebook_task:
            notebook_path: /jobs/alert_on_failure
          
        - task_key: run_dlt_gold
          depends_on: [{task_key: transform_silver}]
          run_if: ALL_SUCCESS
          pipeline_task:
            pipeline_id: "${var.dlt_pipeline_id}"
            full_refresh: false
```

### Example 4: Control Table Pattern (Idempotent Processing)

```python
# Control table tracks which files/partitions have been processed
# Useful when not using DLT or AutoLoader checkpoint

def get_unprocessed_partitions(spark, source_table, control_table):
    available = spark.sql(f"""
        SELECT DISTINCT date_partition FROM {source_table}
    """)
    processed = spark.sql(f"""
        SELECT date_partition FROM {control_table} WHERE status = 'SUCCESS'
    """)
    return available.subtract(processed)

def mark_processed(spark, control_table, partition, status, rows):
    spark.sql(f"""
        INSERT INTO {control_table} VALUES
        ('{partition}', '{status}', {rows}, current_timestamp())
    """)

# Main loop
unprocessed = get_unprocessed_partitions(spark, "bronze.events", "meta.processing_log")

for row in unprocessed.collect():
    partition = row["date_partition"]
    try:
        df = spark.table("bronze.events").filter(f"date_partition = '{partition}'")
        row_count = df.count()
        # ... transform and write ...
        mark_processed(spark, "meta.processing_log", partition, "SUCCESS", row_count)
    except Exception as e:
        mark_processed(spark, "meta.processing_log", partition, "FAILED", 0)
        raise
```

### Example 5: Programmatic Job Creation via Python SDK

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import (
    Task, NotebookTask, JobCluster, ClusterSpec,
    CronSchedule, JobEmailNotifications
)

w = WorkspaceClient()  # uses DATABRICKS_HOST + DATABRICKS_TOKEN env vars

job = w.jobs.create(
    name="My ETL Job",
    job_clusters=[
        JobCluster(
            job_cluster_key="main",
            new_cluster=ClusterSpec(
                spark_version="14.3.x-scala2.12",
                node_type_id="Standard_DS3_v2",
                num_workers=4
            )
        )
    ],
    tasks=[
        Task(
            task_key="ingest",
            job_cluster_key="main",
            notebook_task=NotebookTask(
                notebook_path="/jobs/ingest",
                base_parameters={"env": "prod"}
            )
        )
    ],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 8 * * ?",
        timezone_id="Europe/Berlin"
    ),
    email_notifications=JobEmailNotifications(
        on_failure=["team@company.com"]
    )
)

print(f"Created job: {job.job_id}")

# Trigger a run
run = w.jobs.run_now(job_id=job.job_id)
print(f"Run ID: {run.run_id}")
```

-----

## Appendix: Key Delta Table SQL Commands

```sql
-- Optimize & compact
OPTIMIZE silver.events;
OPTIMIZE silver.events ZORDER BY (customer_id, event_date);

-- Vacuum (remove old files, min 7 days for streaming safety)
VACUUM silver.events RETAIN 168 HOURS;

-- Time travel
SELECT * FROM silver.events VERSION AS OF 5;
SELECT * FROM silver.events TIMESTAMP AS OF '2024-06-01';
RESTORE TABLE silver.events TO VERSION AS OF 5;

-- Table history
DESCRIBE HISTORY silver.events;

-- Table details
DESCRIBE DETAIL silver.events;

-- Auto-optimize settings
ALTER TABLE silver.events SET TBLPROPERTIES (
  'delta.autoOptimize.optimizeWrite' = 'true',
  'delta.autoOptimize.autoCompact'   = 'true'
);

-- Partitioning (set at creation)
CREATE TABLE silver.events (
  event_id STRING, user_id STRING,
  event_ts TIMESTAMP, event_date DATE
)
USING DELTA
PARTITIONED BY (event_date)
LOCATION 's3://bucket/silver/events/';

-- Clone (shallow for testing, deep for backup)
CREATE TABLE dev.events_test SHALLOW CLONE silver.events;
CREATE TABLE backup.events_snap DEEP CLONE silver.events;

-- Column mapping (rename/drop without rewrite)
ALTER TABLE silver.events SET TBLPROPERTIES ('delta.columnMapping.mode' = 'name');
ALTER TABLE silver.events RENAME COLUMN user_id TO customer_id;
ALTER TABLE silver.events DROP COLUMN deprecated_col;
```

-----

*Reference card compiled for Databricks Runtime 14.x+ with Unity Catalog. API versions: Jobs API 2.1, Pipelines API 2.0, Delta 3.x.*