# Databricks Lakehouse Monitoring — Reference Card

> **Scope:** Unity Catalog Lakehouse Monitoring (LHM) · `databricks.sdk` · `databricks-sdk` Python · SQL API · REST API  
> **Applies to:** Databricks Runtime 14.3 LTS+ · Unity Catalog enabled workspaces  
> **Last updated:** 2025

-----

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
1. [Core Concepts](#2-core-concepts)
1. [Monitor Types Quick Reference](#3-monitor-types-quick-reference)
1. [Python SDK — Full API Reference](#4-python-sdk--full-api-reference)
1. [REST API Reference](#5-rest-api-reference)
1. [SQL System Tables & Metric Tables](#6-sql-system-tables--metric-tables)
1. [Drift Detection — Methods & Algorithms](#7-drift-detection--methods--algorithms)
1. [Feature Store & AI Asset Monitoring](#8-feature-store--ai-asset-monitoring)
1. [Model Performance Monitoring](#9-model-performance-monitoring)
1. [Custom Metrics](#10-custom-metrics)
1. [Alerting & Notifications](#11-alerting--notifications)
1. [Design Patterns & Best Practices](#12-design-patterns--best-practices)
1. [Common Recipes & Full Examples](#13-common-recipes--full-examples)
1. [Troubleshooting](#14-troubleshooting)

-----

## 1. Overview & Architecture

Databricks Lakehouse Monitoring continuously profiles Unity Catalog tables and ML model serving endpoints, producing two auto-managed Delta tables per monitor — a **profile metrics table** and a **drift metrics table** — stored alongside the monitored asset.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Monitored Asset (UC Table)                      │
│  catalog.schema.my_table                                            │
└────────────────────────┬────────────────────────────────────────────┘
                         │  Monitor attached via API / UI
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Lakehouse Monitor                                  │
│  Type: TimeSeries | InferenceLog | Snapshot                         │
│  Cadence: scheduled refresh (cron) or on-demand                     │
└──────────────┬──────────────────────────┬───────────────────────────┘
               │                          │
               ▼                          ▼
┌──────────────────────────┐  ┌──────────────────────────────────────┐
│   Profile Metrics Table  │  │         Drift Metrics Table          │
│  _profile_metrics        │  │         _drift_metrics               │
│  (statistical summary,   │  │  (KS, PSI, chi², JS divergence,      │
│   percentiles, nulls,    │  │   Wasserstein, L-infinity, custom)   │
│   distinct counts, etc.) │  │                                      │
└──────────────────────────┘  └──────────────────────────────────────┘
               │                          │
               └──────────┬───────────────┘
                          ▼
              ┌────────────────────────┐
              │  Databricks Dashboards │
              │  / SQL Queries         │
              │  / Alerts              │
              └────────────────────────┘
```

### Key architectural facts

- Monitors are **UC-native**: they live as metadata on a UC table.
- Metric tables are **regular Delta tables** — query them with SQL, PySpark, pandas.
- Monitors run as **serverless SQL warehouses** (no cluster management).
- Baseline can be a separate UC table or a **time window** of the primary table.
- One monitor per table; multiple **slicing expressions** per monitor.

-----

## 2. Core Concepts

|Concept                    |Description                                                                        |
|---------------------------|-----------------------------------------------------------------------------------|
|**Monitor**                |Metadata + refresh schedule attached to one UC table                               |
|**Profile metrics table**  |Auto-generated `<table>_profile_metrics` — statistical summary per window/slice    |
|**Drift metrics table**    |Auto-generated `<table>_drift_metrics` — comparison to baseline per column         |
|**Monitor type**           |`TimeSeries`, `InferenceLog`, or `Snapshot` — governs windowing logic              |
|**Baseline**               |Reference dataset for drift: a UC table or time-bounded window                     |
|**Granularity**            |Time-window sizes for aggregation: `"1 day"`, `"1 hour"`, `"1 week"`               |
|**Slicing expression**     |SQL expression that partitions profiles — e.g., `"region"`, `"model_version"`      |
|**Inference log**          |Structured model serving log: includes features, predictions, labels, model version|
|**Custom metrics**         |User-defined SQL/Python aggregates appended to the profile table                   |
|**Problem type**           |`CLASSIFICATION` or `REGRESSION` — unlocks ML-specific metrics                     |
|**Label column**           |Ground-truth column for supervised model quality metrics                           |
|**Prediction column**      |Model output column                                                                |
|**Prediction proba column**|Probability/score column for binary classifiers                                    |

-----

## 3. Monitor Types Quick Reference

|Type          |Use When                                                   |Required Columns                                 |Window Logic                             |
|--------------|-----------------------------------------------------------|-------------------------------------------------|-----------------------------------------|
|`TimeSeries`  |Any time-stamped data table (features, events, ETL outputs)|`timestamp_col` (timestamp)                      |Aggregates by time granularity           |
|`InferenceLog`|Model serving logs with predictions & optional labels      |`timestamp_col`, `model_id_col`, `prediction_col`|Aggregates by granularity + model version|
|`Snapshot`    |Static or slowly changing tables without timestamps        |None required                                    |Full-table scan per refresh              |

-----

## 4. Python SDK — Full API Reference

### 4.1 Installation & Client Initialization

```python
# Install
# pip install databricks-sdk

from databricks.sdk import WorkspaceClient
from databricks.sdk.service.catalog import (
    MonitorTimeSeries,
    MonitorInferenceLog,
    MonitorInferenceLogProblemType,
    MonitorSnapshot,
    MonitorMetric,
    MonitorMetricType,
    MonitorCronSchedule,
    MonitorNotifications,
    MonitorDestination,
    CreateMonitor,
    UpdateMonitor,
)

w = WorkspaceClient()   # uses env vars DATABRICKS_HOST + DATABRICKS_TOKEN
# or:
w = WorkspaceClient(host="https://<workspace>.azuredatabricks.net", token="<pat>")
```

### 4.2 `quality_monitors` — All Methods

|Method          |Signature                                                         |Description                                   |
|----------------|------------------------------------------------------------------|----------------------------------------------|
|`create`        |`create(table_name, *, assets_dir, output_schema_name, [options])`|Create a new monitor on a UC table            |
|`get`           |`get(table_name)`                                                 |Retrieve monitor metadata                     |
|`update`        |`update(table_name, [options])`                                   |Update monitor configuration                  |
|`delete`        |`delete(table_name)`                                              |Remove monitor (does NOT delete metric tables)|
|`run_refresh`   |`run_refresh(table_name)`                                         |Trigger an on-demand refresh run              |
|`get_refresh`   |`get_refresh(table_name, refresh_id)`                             |Poll a specific refresh run                   |
|`list_refreshes`|`list_refreshes(table_name)`                                      |List all refresh runs                         |
|`cancel_refresh`|`cancel_refresh(table_name, refresh_id)`                          |Cancel a running refresh                      |

### 4.3 `create` — Full Parameter Reference

```python
monitor = w.quality_monitors.create(
    table_name="catalog.schema.table",        # str — fully qualified UC table name
    assets_dir="/Shared/monitors/my_monitor", # str — Workspace path for dashboard assets
    output_schema_name="catalog.schema",      # str — UC schema for metric tables

    # ── Choose ONE monitor type ────────────────────────────────────────
    time_series=MonitorTimeSeries(
        timestamp_col="event_time",           # str — timestamp column name
        granularities=["1 day", "1 hour"],    # List[str] — aggregation window sizes
    ),

    # ── OR ──
    inference_log=MonitorInferenceLog(
        timestamp_col="request_time",
        granularities=["1 day"],
        model_id_col="model_version",         # str — column identifying model version
        prediction_col="prediction",          # str — predicted label / value
        problem_type=MonitorInferenceLogProblemType.CLASSIFICATION,  # or REGRESSION
        label_col="ground_truth",             # str (optional) — actual label
        prediction_proba_col="score",         # str (optional) — confidence score
    ),

    # ── OR ──
    snapshot=MonitorSnapshot(),               # no params needed

    # ── Baseline ───────────────────────────────────────────────────────
    baseline_table_name="catalog.schema.baseline_table",  # str (optional)

    # ── Slicing ────────────────────────────────────────────────────────
    slicing_exprs=["region", "model_version", "CASE WHEN age > 40 THEN 'senior' ELSE 'junior' END"],

    # ── Custom metrics ─────────────────────────────────────────────────
    custom_metrics=[
        MonitorMetric(
            type=MonitorMetricType.AGGREGATE,
            name="pct_high_confidence",
            input_columns=[":table"],          # :table = whole-row context
            definition="avg(CASE WHEN score > 0.9 THEN 1 ELSE 0 END)",
        ),
        MonitorMetric(
            type=MonitorMetricType.DRIFT,
            name="custom_psi",
            input_columns=["feature_x"],
            definition="{{model}}.custom_psi({{input_column}}, {{base_column}})",
        ),
    ],

    # ── Schedule ───────────────────────────────────────────────────────
    schedule=MonitorCronSchedule(
        quartz_cron_expression="0 0 8 * * ?",  # str — every day at 08:00
        timezone_id="Europe/Berlin",
    ),

    # ── Notifications ──────────────────────────────────────────────────
    notifications=MonitorNotifications(
        on_new_classification_tag_detected=MonitorDestination(
            email_addresses=["mlops@company.com"]
        ),
        on_failure=MonitorDestination(
            email_addresses=["oncall@company.com"]
        ),
    ),

    # ── Skip baseline on first run ─────────────────────────────────────
    skip_builtin_baseline_metrics=False,       # bool — default False
    inference_log=...,                         # (repeated for clarity)
)
```

### 4.4 `update` — Mutable Fields

```python
w.quality_monitors.update(
    table_name="catalog.schema.table",
    slicing_exprs=["region"],                  # replace entire list
    schedule=MonitorCronSchedule(
        quartz_cron_expression="0 0 6 * * ?",
        timezone_id="UTC",
    ),
    notifications=MonitorNotifications(
        on_failure=MonitorDestination(email_addresses=["new-oncall@company.com"])
    ),
    custom_metrics=[...],                      # replaces all custom metrics
    baseline_table_name="catalog.schema.new_baseline",
)
```

> **Note:** `time_series`, `inference_log`, `snapshot` are **immutable** after creation. Delete and recreate to change monitor type.

### 4.5 Run Refresh & Poll

```python
# Trigger on-demand refresh
run = w.quality_monitors.run_refresh(table_name="catalog.schema.table")
print(run.refresh_id, run.state)

# Poll until complete
import time
while True:
    r = w.quality_monitors.get_refresh("catalog.schema.table", run.refresh_id)
    print(r.state)   # PENDING | RUNNING | SUCCESS | FAILED | CANCELED
    if r.state.value in ("SUCCESS", "FAILED", "CANCELED"):
        break
    time.sleep(15)

# List all refreshes (returns iterator)
for r in w.quality_monitors.list_refreshes("catalog.schema.table"):
    print(r.refresh_id, r.state, r.start_time_ms, r.end_time_ms)

# Cancel
w.quality_monitors.cancel_refresh("catalog.schema.table", run.refresh_id)
```

-----

## 5. REST API Reference

|Method  |Endpoint                                                                   |Description       |
|--------|---------------------------------------------------------------------------|------------------|
|`POST`  |`/api/2.1/unity-catalog/tables/{table_name}/monitor`                       |Create monitor    |
|`GET`   |`/api/2.1/unity-catalog/tables/{table_name}/monitor`                       |Get monitor       |
|`PUT`   |`/api/2.1/unity-catalog/tables/{table_name}/monitor`                       |Update monitor    |
|`DELETE`|`/api/2.1/unity-catalog/tables/{table_name}/monitor`                       |Delete monitor    |
|`POST`  |`/api/2.1/unity-catalog/tables/{table_name}/monitor/refresh`               |Trigger refresh   |
|`GET`   |`/api/2.1/unity-catalog/tables/{table_name}/monitor/refreshes`             |List refreshes    |
|`GET`   |`/api/2.1/unity-catalog/tables/{table_name}/monitor/refreshes/{refresh_id}`|Get refresh status|
|`DELETE`|`/api/2.1/unity-catalog/tables/{table_name}/monitor/refreshes/{refresh_id}`|Cancel refresh    |

```bash
# Example: create via REST
curl -X POST "https://<host>/api/2.1/unity-catalog/tables/catalog.schema.table/monitor" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "assets_dir": "/Shared/monitors/my_monitor",
    "output_schema_name": "catalog.monitoring",
    "time_series": {
      "timestamp_col": "event_time",
      "granularities": ["1 day"]
    },
    "slicing_exprs": ["region"]
  }'
```

-----

## 6. SQL System Tables & Metric Tables

### 6.1 Profile Metrics Table Schema

Auto-named: `<output_schema>.<table_name>_profile_metrics`

|Column                          |Type            |Description                                                         |
|--------------------------------|----------------|--------------------------------------------------------------------|
|`window`                        |struct          |`{start: timestamp, end: timestamp}` — time window boundaries       |
|`window_start_time`             |timestamp       |Start of aggregation window                                         |
|`granularity`                   |string          |Window size: `"1 day"`, `"1 hour"`, etc.                            |
|`log_type`                      |string          |`INPUT`, `OUTPUT`, `LABEL`, or null for non-inference               |
|`model_id`                      |string          |Model version (InferenceLog only)                                   |
|`slice_key`                     |string          |Slicing expression name                                             |
|`slice_value`                   |string          |Sliced partition value                                              |
|`column_name`                   |string          |Profiled column name (null = table-level row)                       |
|`column_type`                   |string          |`BOOLEAN`, `NUMERICAL`, `CATEGORICAL`, `TEXT`, `DATETIME`, `COMPLEX`|
|`num_rows`                      |long            |Row count in window                                                 |
|`num_nulls`                     |long            |Null count                                                          |
|`percent_null`                  |double          |Fraction of nulls                                                   |
|`num_distinct`                  |long            |Approximate distinct count                                          |
|`percent_distinct`              |double          |Fraction distinct                                                   |
|`min`                           |double          |Minimum value (numerical)                                           |
|`max`                           |double          |Maximum value (numerical)                                           |
|`mean`                          |double          |Mean (numerical)                                                    |
|`std`                           |double          |Standard deviation (numerical)                                      |
|`percentile_25`                 |double          |25th percentile                                                     |
|`percentile_50`                 |double          |Median                                                              |
|`percentile_75`                 |double          |75th percentile                                                     |
|`percent_zeros`                 |double          |Fraction of zero values                                             |
|`num_zeros`                     |long            |Count of zero values                                                |
|`freq_items`                    |map<string,long>|Top-N frequent values (categorical)                                 |
|`accuracy`                      |double          |Classification accuracy (InferenceLog + label)                      |
|`f1_score`                      |double          |F1 score (classification)                                           |
|`precision`                     |double          |Precision (classification)                                          |
|`recall`                        |double          |Recall (classification)                                             |
|`log_loss`                      |double          |Log loss (binary classification)                                    |
|`roc_auc`                       |double          |Area under ROC curve                                                |
|`mae`                           |double          |Mean absolute error (regression)                                    |
|`mse`                           |double          |Mean squared error                                                  |
|`rmse`                          |double          |Root mean squared error                                             |
|`r2_score`                      |double          |R² coefficient of determination                                     |
|`mean_absolute_percentage_error`|double          |MAPE                                                                |
|`<custom_metric_name>`          |any             |Your custom aggregated metrics                                      |

### 6.2 Drift Metrics Table Schema

Auto-named: `<output_schema>.<table_name>_drift_metrics`

|Column                 |Type     |Description                                                                                     |
|-----------------------|---------|------------------------------------------------------------------------------------------------|
|`window`               |struct   |Current window boundaries                                                                       |
|`window_start_time`    |timestamp|Current window start                                                                            |
|`granularity`          |string   |Window size                                                                                     |
|`model_id`             |string   |Model version (InferenceLog)                                                                    |
|`slice_key`            |string   |Slicing expression name                                                                         |
|`slice_value`          |string   |Slice value                                                                                     |
|`column_name`          |string   |Column being tested                                                                             |
|`column_type`          |string   |Column data type category                                                                       |
|`drift_type`           |string   |Algorithm: `KS`, `PSI`, `CHI_SQUARED`, `JS`, `WASSERSTEIN`, `LINF`, `TWO_SAMPLE_TTEST`, `CUSTOM`|
|`drift_score`          |double   |Test statistic or divergence value                                                              |
|`drift_detected`       |boolean  |Whether drift exceeded threshold                                                                |
|`baseline_window_start`|timestamp|Baseline window start                                                                           |
|`baseline_window_end`  |timestamp|Baseline window end                                                                             |

### 6.3 Useful SQL Queries

```sql
-- Latest profile: all column stats for today
SELECT *
FROM catalog.monitoring.my_table_profile_metrics
WHERE window_start_time >= current_date() - INTERVAL 1 DAY
ORDER BY column_name;

-- Null rate trend over 30 days for a specific column
SELECT window_start_time, percent_null
FROM catalog.monitoring.my_table_profile_metrics
WHERE column_name = 'feature_x'
  AND granularity = '1 day'
ORDER BY window_start_time;

-- Columns with drift detected in last run
SELECT column_name, drift_type, drift_score
FROM catalog.monitoring.my_table_drift_metrics
WHERE drift_detected = true
  AND window_start_time = (
    SELECT max(window_start_time) FROM catalog.monitoring.my_table_drift_metrics
  )
ORDER BY drift_score DESC;

-- Model accuracy over time (InferenceLog)
SELECT window_start_time, model_id, accuracy, f1_score, roc_auc
FROM catalog.monitoring.inference_log_profile_metrics
WHERE column_name IS NULL   -- table-level row
  AND log_type = 'OUTPUT'
ORDER BY window_start_time DESC;

-- Feature distribution shift: mean & std trend
SELECT window_start_time, column_name, mean, std,
       percentile_25, percentile_50, percentile_75
FROM catalog.monitoring.my_table_profile_metrics
WHERE column_name IN ('age', 'income', 'credit_score')
  AND granularity = '1 day'
ORDER BY window_start_time, column_name;

-- Identify top drifting features by PSI
SELECT column_name, avg(drift_score) AS avg_psi
FROM catalog.monitoring.my_table_drift_metrics
WHERE drift_type = 'PSI'
  AND window_start_time >= current_date() - INTERVAL 7 DAY
GROUP BY column_name
ORDER BY avg_psi DESC
LIMIT 10;

-- Categorical value frequency shift
SELECT window_start_time, column_name, freq_items
FROM catalog.monitoring.my_table_profile_metrics
WHERE column_type = 'CATEGORICAL'
  AND column_name = 'product_category'
ORDER BY window_start_time;

-- Prediction distribution by model version
SELECT model_id, window_start_time, mean AS avg_prediction, std AS std_prediction
FROM catalog.monitoring.inference_table_profile_metrics
WHERE column_name = 'prediction'
ORDER BY window_start_time, model_id;
```

-----

## 7. Drift Detection — Methods & Algorithms

### 7.1 Built-in Drift Tests

|Test                          |`drift_type`      |Column Type            |Measures                                 |Threshold Guidance                             |
|------------------------------|------------------|-----------------------|-----------------------------------------|-----------------------------------------------|
|**Kolmogorov-Smirnov**        |`KS`              |Numerical              |Max absolute difference between CDFs     |>0.1 mild, >0.2 severe                         |
|**Population Stability Index**|`PSI`             |Numerical / Categorical|Relative distribution change             |<0.1 stable, 0.1–0.2 moderate, >0.2 significant|
|**Chi-Squared**               |`CHI_SQUARED`     |Categorical            |Statistical independence of distributions|p < 0.05 drift detected                        |
|**Jensen-Shannon Divergence** |`JS`              |Numerical / Categorical|Symmetric KL divergence, bounded [0,1]   |>0.1 noteworthy                                |
|**Wasserstein Distance**      |`WASSERSTEIN`     |Numerical              |Earth Mover’s Distance — mean shift      |Domain-dependent                               |
|**L-Infinity Distance**       |`LINF`            |Numerical / Categorical|Max bin-wise probability difference      |>0.1 noteworthy                                |
|**Two-Sample t-Test**         |`TWO_SAMPLE_TTEST`|Numerical              |Mean shift significance test             |p-value threshold                              |

### 7.2 Drift Algorithm Details

```
KS (Kolmogorov-Smirnov):
  D = max|F_current(x) - F_baseline(x)|
  Best for: continuous distributions, sensitive to shape changes
  Limitation: less sensitive to tail differences

PSI (Population Stability Index):
  PSI = Σ (P_current - P_baseline) × ln(P_current / P_baseline)
  Best for: production monitoring, interpretable thresholds
  Buckets: typically 10 equal-frequency bins

Chi-Squared:
  χ² = Σ (O - E)² / E
  Best for: categorical features, low cardinality
  Requires: sufficient counts per bucket (≥5 recommended)

Jensen-Shannon:
  JS(P||Q) = ½ KL(P||M) + ½ KL(Q||M), M = ½(P+Q)
  Best for: robust comparison, handles zero probabilities
  Bounded: [0, 1] (or [0, log(2)] depending on base)

Wasserstein:
  W₁(P,Q) = ∫|F_P(x) - F_Q(x)| dx
  Best for: detecting mean/median shifts
  Interpretable in original units
```

### 7.3 Automatic Test Assignment

|Column Type                     |Auto-assigned Drift Test |
|--------------------------------|-------------------------|
|`NUMERICAL` (continuous)        |KS + Wasserstein + PSI   |
|`CATEGORICAL` (low cardinality) |Chi-Squared + PSI        |
|`CATEGORICAL` (high cardinality)|PSI only                 |
|`BOOLEAN`                       |Chi-Squared              |
|`DATETIME`                      |KS                       |
|`TEXT`                          |PSI on token distribution|

-----

## 8. Feature Store & AI Asset Monitoring

### 8.1 Monitoring Feature Tables

Feature tables in Unity Catalog are standard Delta tables — attach a `TimeSeries` or `Snapshot` monitor directly.

```python
# Monitor a UC feature table
w.quality_monitors.create(
    table_name="catalog.feature_store.customer_features",
    assets_dir="/Shared/monitors/customer_features",
    output_schema_name="catalog.monitoring",
    time_series=MonitorTimeSeries(
        timestamp_col="feature_timestamp",
        granularities=["1 day", "1 hour"],
    ),
    slicing_exprs=["customer_segment", "country"],
    baseline_table_name="catalog.feature_store.customer_features_baseline",
    schedule=MonitorCronSchedule(
        quartz_cron_expression="0 30 6 * * ?",  # daily 06:30
        timezone_id="UTC",
    ),
)
```

### 8.2 Linking Feature Serving to Inference Logs

```python
# Pattern: log feature values alongside predictions to a UC table
# Then monitor that table as InferenceLog

# In your serving layer / batch inference:
from pyspark.sql import functions as F
import datetime

predictions_df = (
    model_predictions
    .withColumn("request_time", F.current_timestamp())
    .withColumn("model_version", F.lit("v3.2"))
    # include all input features + prediction
)
predictions_df.write.mode("append").saveAsTable("catalog.monitoring.inference_log")

# Monitor the log table
w.quality_monitors.create(
    table_name="catalog.monitoring.inference_log",
    assets_dir="/Shared/monitors/inference_log",
    output_schema_name="catalog.monitoring",
    inference_log=MonitorInferenceLog(
        timestamp_col="request_time",
        granularities=["1 day"],
        model_id_col="model_version",
        prediction_col="prediction",
        problem_type=MonitorInferenceLogProblemType.CLASSIFICATION,
        label_col="ground_truth",           # join labels later via merge
        prediction_proba_col="probability",
    ),
    slicing_exprs=["customer_segment"],
    baseline_table_name="catalog.monitoring.inference_log_baseline",
)
```

### 8.3 Joining Ground-Truth Labels (Delayed Labels Pattern)

```python
# Common pattern: predictions arrive before labels
# Use Delta MERGE to backfill labels when they arrive

from delta.tables import DeltaTable

labels_df = spark.read.table("catalog.prod.ground_truth_labels")

inference_log = DeltaTable.forName(spark, "catalog.monitoring.inference_log")

inference_log.alias("log").merge(
    labels_df.alias("labels"),
    "log.request_id = labels.request_id"
).whenMatchedUpdate(set={"log.ground_truth": "labels.label"}).execute()

# Then trigger monitor refresh to pick up new labels
w.quality_monitors.run_refresh("catalog.monitoring.inference_log")
```

-----

## 9. Model Performance Monitoring

### 9.1 Classification Metrics

|Metric              |Column     |Requires                            |
|--------------------|-----------|------------------------------------|
|Accuracy            |`accuracy` |`label_col`                         |
|F1 Score (weighted) |`f1_score` |`label_col`                         |
|Precision (weighted)|`precision`|`label_col`                         |
|Recall (weighted)   |`recall`   |`label_col`                         |
|Log Loss            |`log_loss` |`label_col` + `prediction_proba_col`|
|ROC AUC             |`roc_auc`  |`label_col` + `prediction_proba_col`|

### 9.2 Regression Metrics

|Metric|Column                          |Description                 |
|------|--------------------------------|----------------------------|
|MAE   |`mae`                           |Mean absolute error         |
|MSE   |`mse`                           |Mean squared error          |
|RMSE  |`rmse`                          |Root mean squared error     |
|R²    |`r2_score`                      |Coefficient of determination|
|MAPE  |`mean_absolute_percentage_error`|% error relative to actuals |

### 9.3 Querying Model Quality Over Time

```sql
-- Accuracy degradation alert query
SELECT
    window_start_time,
    model_id,
    accuracy,
    roc_auc,
    log_loss,
    LAG(accuracy) OVER (PARTITION BY model_id ORDER BY window_start_time) AS prev_accuracy,
    accuracy - LAG(accuracy) OVER (PARTITION BY model_id ORDER BY window_start_time) AS accuracy_delta
FROM catalog.monitoring.inference_log_profile_metrics
WHERE column_name IS NULL
  AND log_type = 'OUTPUT'
ORDER BY window_start_time DESC;

-- Comparing model versions (A/B or champion/challenger)
SELECT model_id, window_start_time, accuracy, f1_score, roc_auc, num_rows
FROM catalog.monitoring.inference_log_profile_metrics
WHERE column_name IS NULL
  AND log_type = 'OUTPUT'
  AND model_id IN ('v2.0', 'v3.0')   -- challenger comparison
ORDER BY window_start_time, model_id;

-- Prediction distribution shift (output drift)
SELECT window_start_time, model_id, drift_score, drift_detected
FROM catalog.monitoring.inference_log_drift_metrics
WHERE column_name = 'prediction'
ORDER BY window_start_time DESC;
```

### 9.4 MLflow Integration

```python
import mlflow
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Log monitor refresh results back to MLflow for experiment tracking
def log_monitor_metrics_to_mlflow(
    table_name: str,
    run_id: str,
    metric_table: str,
):
    df = spark.sql(f"""
        SELECT * FROM {metric_table}
        WHERE window_start_time = (SELECT max(window_start_time) FROM {metric_table})
          AND column_name IS NULL
    """).toPandas()

    with mlflow.start_run(run_id=run_id):
        for _, row in df.iterrows():
            mlflow.log_metric("accuracy", row["accuracy"])
            mlflow.log_metric("roc_auc", row["roc_auc"])
            mlflow.log_metric("f1_score", row["f1_score"])

# Tag model version when drift detected
def check_and_tag_drift(model_name: str, version: str, drift_table: str):
    drift_df = spark.sql(f"""
        SELECT count(*) AS drifted_cols
        FROM {drift_table}
        WHERE drift_detected = true
          AND window_start_time >= current_date() - INTERVAL 1 DAY
    """).collect()[0]

    if drift_df["drifted_cols"] > 0:
        client.set_model_version_tag(model_name, version, "data_drift", "detected")
```

-----

## 10. Custom Metrics

### 10.1 `MonitorMetricType` Values

|Type       |When Used                                           |Input Columns                     |
|-----------|----------------------------------------------------|----------------------------------|
|`AGGREGATE`|Custom statistical summary appended to profile table|Any column(s) or `:table`         |
|`DRIFT`    |Custom drift statistic appended to drift table      |Column pair (current vs. baseline)|
|`DERIVED`  |Computed from other named metrics in the same run   |Named metric references           |

### 10.2 Custom Metric Definition Syntax

```python
# AGGREGATE metric — runs over current window rows
MonitorMetric(
    type=MonitorMetricType.AGGREGATE,
    name="pct_high_value_predictions",
    input_columns=[":table"],               # :table = access to all columns
    definition="avg(CASE WHEN prediction > 0.8 THEN 1.0 ELSE 0.0 END)",
)

# AGGREGATE with specific column
MonitorMetric(
    type=MonitorMetricType.AGGREGATE,
    name="iqr_feature_x",
    input_columns=["feature_x"],
    definition="percentile(feature_x, 0.75) - percentile(feature_x, 0.25)",
)

# DERIVED — references other metrics by name
MonitorMetric(
    type=MonitorMetricType.DERIVED,
    name="cv_feature_x",                   # coefficient of variation
    input_columns=["feature_x"],
    definition=":std / :mean",             # references built-in mean & std
)

# DRIFT metric — template syntax for current vs. baseline
# {{input_column}} = current window values
# {{base_column}}  = baseline values
# {{model}}        = built-in function registry (for referencing UDFs)
MonitorMetric(
    type=MonitorMetricType.DRIFT,
    name="mean_shift_pct",
    input_columns=["feature_x"],
    definition="abs(avg({{input_column}}) - avg({{base_column}})) / abs(avg({{base_column}}))",
)
```

### 10.3 Custom Metric Examples

```python
custom_metrics = [
    # Business metric: revenue at risk
    MonitorMetric(
        type=MonitorMetricType.AGGREGATE,
        name="revenue_at_risk",
        input_columns=[":table"],
        definition="sum(CASE WHEN prediction = 'churn' THEN expected_revenue ELSE 0 END)",
    ),

    # Calibration: average predicted vs actual positive rate
    MonitorMetric(
        type=MonitorMetricType.AGGREGATE,
        name="calibration_gap",
        input_columns=[":table"],
        definition="avg(score) - avg(CASE WHEN ground_truth = 1 THEN 1.0 ELSE 0.0 END)",
    ),

    # Custom PSI with 20 bins
    MonitorMetric(
        type=MonitorMetricType.DRIFT,
        name="psi_20bins",
        input_columns=["feature_income"],
        definition="""
            sum(CASE
                WHEN pct_curr > 0 AND pct_base > 0
                THEN (pct_curr - pct_base) * ln(pct_curr / pct_base)
                ELSE 0
            END)
        """,  # simplified — actual PSI UDF registration required
    ),

    # Null rate change (drift metric)
    MonitorMetric(
        type=MonitorMetricType.DRIFT,
        name="null_rate_delta",
        input_columns=["feature_x"],
        definition="avg(CASE WHEN {{input_column}} IS NULL THEN 1.0 ELSE 0.0 END) - "
                   "avg(CASE WHEN {{base_column}} IS NULL THEN 1.0 ELSE 0.0 END)",
    ),
]
```

-----

## 11. Alerting & Notifications

### 11.1 Built-in Notification Events

|Event Key                           |Trigger                                     |
|------------------------------------|--------------------------------------------|
|`on_new_classification_tag_detected`|New data quality tag from automated analysis|
|`on_failure`                        |Monitor refresh job failed                  |

```python
notifications=MonitorNotifications(
    on_new_classification_tag_detected=MonitorDestination(
        email_addresses=["data-quality@company.com", "ml-team@company.com"]
    ),
    on_failure=MonitorDestination(
        email_addresses=["oncall@company.com"]
    ),
)
```

### 11.2 SQL-Based Alerts (Databricks SQL Alerts)

```sql
-- Alert query: drift score above PSI threshold
SELECT count(*) AS num_drifted_columns
FROM catalog.monitoring.my_table_drift_metrics
WHERE drift_type = 'PSI'
  AND drift_score > 0.2
  AND window_start_time >= current_date() - INTERVAL 1 DAY;
-- Alert condition: num_drifted_columns > 0
```

```sql
-- Alert query: model accuracy below threshold
SELECT accuracy
FROM catalog.monitoring.inference_log_profile_metrics
WHERE column_name IS NULL
  AND log_type = 'OUTPUT'
  AND window_start_time = (
    SELECT max(window_start_time)
    FROM catalog.monitoring.inference_log_profile_metrics
    WHERE column_name IS NULL
  )
LIMIT 1;
-- Alert condition: accuracy < 0.85
```

### 11.3 Webhook / PagerDuty Pattern

```python
# Use Databricks SQL Alert with webhook destination
# or implement post-refresh callback:

def post_refresh_callback(table_name: str, drift_table: str, threshold: float = 0.2):
    """Check drift after refresh and fire webhook if needed."""
    import requests, json

    drifted = spark.sql(f"""
        SELECT column_name, drift_score
        FROM {drift_table}
        WHERE drift_detected = true
          AND drift_score > {threshold}
          AND window_start_time = (SELECT max(window_start_time) FROM {drift_table})
    """).collect()

    if drifted:
        payload = {
            "text": f"🚨 Drift detected in {table_name}: "
                    + ", ".join(f"{r.column_name} (PSI={r.drift_score:.3f})" for r in drifted)
        }
        requests.post(SLACK_WEBHOOK_URL, data=json.dumps(payload))
```

-----

## 12. Design Patterns & Best Practices

### 12.1 Architecture Patterns

```
Pattern 1: Medallion Monitoring
────────────────────────────────
Bronze table → TimeSeries monitor (data quality, schema drift)
Silver table → TimeSeries monitor (transformation quality, null rates)
Gold table   → TimeSeries monitor (business metric drift)
Inference log→ InferenceLog monitor (model quality + feature drift)

Pattern 2: Champion/Challenger Monitoring
──────────────────────────────────────────
Single inference log table containing BOTH model versions
Slice by model_id_col → automatic per-version profiles
Compare versions using sliced profile metrics SQL

Pattern 3: Baseline Rotation
──────────────────────────────
Use a trailing-window (e.g., rolling 30-day) baseline
Update baseline table monthly via DLT pipeline or job
Prevents concept drift masking in long-running monitors

Pattern 4: Label-Delayed Monitoring
────────────────────────────────────
Log predictions immediately → null label_col
MERGE labels when available (T+1 day, T+7 days)
Refresh monitor after label merge
Track metric table's label coverage as custom metric

Pattern 5: Feature Attribution Monitoring
──────────────────────────────────────────
Log SHAP values alongside predictions in inference table
Monitor SHAP column distributions as features
Detect explanability drift independent of accuracy
```

### 12.2 Scheduling Best Practices

|Scenario                      |Recommended Granularity|Schedule                                       |
|------------------------------|-----------------------|-----------------------------------------------|
|High-traffic real-time serving|`1 hour`               |Every hour                                     |
|Daily batch inference         |`1 day`                |After batch job completes (event-based via job)|
|Weekly reporting              |`1 week`               |Weekly                                         |
|Feature table health          |`1 day`                |Pre-training pipeline                          |
|Data ingestion quality        |`1 day`                |Post-ingestion job                             |

```python
# Event-driven refresh: trigger after upstream job completes
# In Databricks Jobs YAML / SDK, add a task dependency:
# task: run_monitor_refresh
# depends_on: [upstream_ingestion_task]
# notebook_task:
#   source: INLINE
#   content: |
#     w.quality_monitors.run_refresh("catalog.schema.table")
```

### 12.3 Slicing Strategy

```python
# Good slicing expressions:
slicing_exprs = [
    # Low cardinality categorical
    "country",
    "product_category",
    "model_version",

    # Derived bucket from continuous
    "CASE WHEN age < 25 THEN 'young' WHEN age < 60 THEN 'middle' ELSE 'senior' END",

    # Date-based seasonality
    "date_format(event_time, 'EEEE')",   # day of week

    # Business segment
    "customer_tier",
]

# Avoid:
# - High-cardinality columns (user_id, session_id) → too many slices, slow/expensive
# - Expressions that produce NULL for large fractions → misleading profiles
# - More than 5-6 slicing expressions → combinatorial explosion
```

### 12.4 Baseline Selection

```python
# Option A: Separate static baseline table (recommended for production)
# Freeze a representative dataset at model training time
baseline_df = training_df.sample(fraction=0.2, seed=42)
baseline_df.write.mode("overwrite").saveAsTable("catalog.monitoring.my_model_baseline")

# Option B: Time-window baseline (no separate table needed)
# Set baseline_table_name=None; LHM uses first window as implicit baseline
# Good for: continuous data without a clear "golden" reference

# Option C: Rolling baseline — update monthly
# Refreshes baseline to avoid concept drift masking
# Implemented as a scheduled job that overwrites baseline table
```

### 12.5 Performance & Cost Optimization

```python
# 1. Use output_schema_name in a dedicated monitoring schema
#    to avoid cluttering feature/model schemas
output_schema_name="catalog.monitoring"

# 2. Set appropriate granularities — fewer = cheaper
#    Only add "1 hour" if you have SLAs requiring it
granularities=["1 day"]  # default; add "1 hour" only if needed

# 3. Limit slicing expressions to business-critical dimensions
#    Each slice multiplies compute cost

# 4. Partition metric tables by window_start_time
#    (done automatically — always filter on window_start_time in queries)

# 5. Use skip_builtin_baseline_metrics=True for tables without meaningful baseline
#    (pure ingestion quality checks where drift is not relevant)

# 6. For very large tables (>1B rows), consider monitoring a sampled view
CREATE OR REPLACE VIEW catalog.schema.table_sampled AS
SELECT * FROM catalog.schema.table TABLESAMPLE (10 PERCENT);
-- Then attach monitor to the view
```

### 12.6 Naming Conventions

```
Metric tables (auto-generated — follow UC naming):
  catalog.monitoring.<source_table_name>_profile_metrics
  catalog.monitoring.<source_table_name>_drift_metrics

Dashboard assets path:
  /Shared/monitors/<team>/<table_name>/

Baseline tables:
  catalog.monitoring.<table_name>_baseline

Custom metric names:
  <noun>_<aggregation>   e.g., revenue_sum, null_pct, iqr_age
  Keep lowercase, snake_case, no spaces
```

-----

## 13. Common Recipes & Full Examples

### 13.1 Full TimeSeries Monitor — Data Quality

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.catalog import (
    MonitorTimeSeries, MonitorMetric, MonitorMetricType,
    MonitorCronSchedule, MonitorNotifications, MonitorDestination,
)

w = WorkspaceClient()

monitor = w.quality_monitors.create(
    table_name="catalog.silver.customer_events",
    assets_dir="/Shared/monitors/data-platform/customer_events",
    output_schema_name="catalog.monitoring",
    time_series=MonitorTimeSeries(
        timestamp_col="event_timestamp",
        granularities=["1 day", "1 hour"],
    ),
    baseline_table_name="catalog.monitoring.customer_events_baseline",
    slicing_exprs=["event_type", "country", "platform"],
    custom_metrics=[
        MonitorMetric(
            type=MonitorMetricType.AGGREGATE,
            name="pct_valid_user_id",
            input_columns=[":table"],
            definition="avg(CASE WHEN user_id IS NOT NULL AND length(user_id) > 0 THEN 1.0 ELSE 0.0 END)",
        ),
        MonitorMetric(
            type=MonitorMetricType.AGGREGATE,
            name="duplicate_event_rate",
            input_columns=[":table"],
            definition="1.0 - (count(DISTINCT event_id) / count(*))",
        ),
    ],
    schedule=MonitorCronSchedule(
        quartz_cron_expression="0 0 7 * * ?",
        timezone_id="UTC",
    ),
    notifications=MonitorNotifications(
        on_failure=MonitorDestination(email_addresses=["data-eng@company.com"])
    ),
)
print(f"Monitor created: {monitor.monitor_version}")
```

### 13.2 Full InferenceLog Monitor — Classification

```python
monitor = w.quality_monitors.create(
    table_name="catalog.ml.churn_model_inference_log",
    assets_dir="/Shared/monitors/ml/churn_model",
    output_schema_name="catalog.monitoring",
    inference_log=MonitorInferenceLog(
        timestamp_col="scored_at",
        granularities=["1 day"],
        model_id_col="model_version",
        prediction_col="predicted_label",
        problem_type=MonitorInferenceLogProblemType.CLASSIFICATION,
        label_col="actual_label",
        prediction_proba_col="churn_probability",
    ),
    baseline_table_name="catalog.monitoring.churn_model_baseline",
    slicing_exprs=["customer_segment", "region"],
    custom_metrics=[
        # Business impact metric
        MonitorMetric(
            type=MonitorMetricType.AGGREGATE,
            name="high_risk_volume",
            input_columns=[":table"],
            definition="sum(CASE WHEN churn_probability > 0.7 THEN 1 ELSE 0 END)",
        ),
        # Calibration error
        MonitorMetric(
            type=MonitorMetricType.AGGREGATE,
            name="calibration_error",
            input_columns=[":table"],
            definition="abs(avg(churn_probability) - avg(CAST(actual_label = 'churned' AS DOUBLE)))",
        ),
    ],
    schedule=MonitorCronSchedule(
        quartz_cron_expression="0 0 5 * * ?",
        timezone_id="UTC",
    ),
    notifications=MonitorNotifications(
        on_new_classification_tag_detected=MonitorDestination(
            email_addresses=["ml-team@company.com"]
        ),
        on_failure=MonitorDestination(
            email_addresses=["ml-oncall@company.com"]
        ),
    ),
)
```

### 13.3 Full InferenceLog Monitor — Regression

```python
monitor = w.quality_monitors.create(
    table_name="catalog.ml.pricing_model_log",
    assets_dir="/Shared/monitors/ml/pricing_model",
    output_schema_name="catalog.monitoring",
    inference_log=MonitorInferenceLog(
        timestamp_col="inference_timestamp",
        granularities=["1 day"],
        model_id_col="model_version",
        prediction_col="predicted_price",
        problem_type=MonitorInferenceLogProblemType.REGRESSION,
        label_col="actual_price",
    ),
    slicing_exprs=["product_category", "market"],
    custom_metrics=[
        MonitorMetric(
            type=MonitorMetricType.AGGREGATE,
            name="median_absolute_error",
            input_columns=[":table"],
            definition="percentile(abs(predicted_price - actual_price), 0.5)",
        ),
        MonitorMetric(
            type=MonitorMetricType.AGGREGATE,
            name="pct_within_10pct",
            input_columns=[":table"],
            definition="""avg(CASE WHEN abs(predicted_price - actual_price) / actual_price < 0.10
                              THEN 1.0 ELSE 0.0 END)""",
        ),
    ],
)
```

### 13.4 Snapshot Monitor — Reference/Lookup Table

```python
monitor = w.quality_monitors.create(
    table_name="catalog.gold.product_catalog",
    assets_dir="/Shared/monitors/data-platform/product_catalog",
    output_schema_name="catalog.monitoring",
    snapshot=MonitorSnapshot(),
    slicing_exprs=["category", "brand"],
    custom_metrics=[
        MonitorMetric(
            type=MonitorMetricType.AGGREGATE,
            name="pct_active_products",
            input_columns=[":table"],
            definition="avg(CASE WHEN status = 'active' THEN 1.0 ELSE 0.0 END)",
        ),
    ],
    schedule=MonitorCronSchedule(
        quartz_cron_expression="0 0 4 ? * MON",  # weekly Monday 04:00
        timezone_id="UTC",
    ),
)
```

### 13.5 Programmatic Drift Report

```python
def generate_drift_report(table_name: str, output_schema: str, days: int = 7):
    """Print a drift summary for the last N days."""
    base = f"{output_schema}.{table_name.split('.')[-1]}"

    drift_df = spark.sql(f"""
        SELECT
            column_name,
            drift_type,
            round(avg(drift_score), 4) AS avg_drift,
            round(max(drift_score), 4) AS max_drift,
            sum(CASE WHEN drift_detected THEN 1 ELSE 0 END) AS days_with_drift,
            count(*) AS total_days
        FROM {base}_drift_metrics
        WHERE window_start_time >= current_date() - INTERVAL {days} DAY
        GROUP BY column_name, drift_type
        HAVING avg_drift > 0.05
        ORDER BY avg_drift DESC
    """)

    print(f"\n=== Drift Report: {table_name} (last {days} days) ===")
    drift_df.show(50, truncate=False)
    return drift_df
```

### 13.6 Automated Baseline Refresh

```python
def rotate_baseline(
    source_table: str,
    baseline_table: str,
    monitor_table: str,
    sample_fraction: float = 0.20,
    seed: int = 42,
):
    """Rotate baseline to most recent month's data."""
    # Sample recent data as new baseline
    new_baseline = spark.sql(f"""
        SELECT * FROM {source_table}
        WHERE event_time >= current_date() - INTERVAL 30 DAY
    """).sample(fraction=sample_fraction, seed=seed)

    # Overwrite baseline table
    new_baseline.write.mode("overwrite").option("overwriteSchema", "true") \
        .saveAsTable(baseline_table)

    # Update monitor to point at new baseline (no-op if same name)
    w.quality_monitors.update(
        table_name=monitor_table,
        baseline_table_name=baseline_table,
    )

    # Trigger immediate refresh with new baseline
    run = w.quality_monitors.run_refresh(monitor_table)
    print(f"Baseline rotated. Refresh started: {run.refresh_id}")
```

-----

## 14. Troubleshooting

|Symptom                                   |Cause                                           |Fix                                                                            |
|------------------------------------------|------------------------------------------------|-------------------------------------------------------------------------------|
|`RESOURCE_DOES_NOT_EXIST` on create       |Unity Catalog not enabled or table doesn’t exist|Enable UC; ensure table exists in UC                                           |
|Monitor refresh `FAILED` immediately      |Invalid column names in config                  |Verify `timestamp_col`, `prediction_col` etc. exist in table schema            |
|No rows in metric tables                  |Monitor created but never refreshed             |Call `run_refresh` or wait for schedule                                        |
|Drift table empty                         |No baseline specified + first window only       |Add `baseline_table_name` or wait for second time window                       |
|All drift scores = 0                      |Baseline same as current data                   |Use separate static baseline from a different time period                      |
|`InvalidParameterValue` on `slicing_exprs`|SQL syntax error in expression                  |Test expression with `SELECT <expr> FROM <table> LIMIT 1` first                |
|Custom metric returns NULL                |Division by zero or data type mismatch          |Add CASE guards; cast types explicitly                                         |
|Very slow refresh                         |Too many slicing expressions or large table     |Reduce slices; use TABLESAMPLE view; increase warehouse size                   |
|Monitor schedule not firing               |Cron expression incorrect or timezone mismatch  |Validate cron with online parser; use UTC                                      |
|`PERMISSION_DENIED`                       |Missing UC privileges                           |Grant `SELECT` on source table + `USE SCHEMA` + `CREATE TABLE` on output schema|
|Metric table auto-deleted                 |Output schema dropped                           |Recreate output schema; re-run monitor                                         |
|Granularity window too small              |Insufficient data per window                    |Increase granularity; or use Snapshot type                                     |

### Required Unity Catalog Privileges

```sql
-- Minimum grants for monitor creator
GRANT USE CATALOG ON CATALOG catalog_name TO principal;
GRANT USE SCHEMA ON SCHEMA catalog.schema TO principal;
GRANT SELECT ON TABLE catalog.schema.monitored_table TO principal;
GRANT USE SCHEMA ON SCHEMA catalog.monitoring TO principal;
GRANT CREATE TABLE ON SCHEMA catalog.monitoring TO principal;

-- For reading metric tables
GRANT SELECT ON TABLE catalog.monitoring.my_table_profile_metrics TO analyst;
GRANT SELECT ON TABLE catalog.monitoring.my_table_drift_metrics TO analyst;
```

-----

## Quick Reference: MonitorCronSchedule Expressions

|Schedule                 |Quartz Expression|
|-------------------------|-----------------|
|Every day at midnight UTC|`0 0 0 * * ?`    |
|Every day at 06:00       |`0 0 6 * * ?`    |
|Every hour               |`0 0 * * * ?`    |
|Every 6 hours            |`0 0 0/6 * * ?`  |
|Every Monday at 04:00    |`0 0 4 ? * MON`  |
|First of month at 00:00  |`0 0 0 1 * ?`    |

-----

## Quick Reference: Granularity Strings

|String        |Bucket Size                |
|--------------|---------------------------|
|`"5 minutes"` |5-minute windows           |
|`"30 minutes"`|30-minute windows          |
|`"1 hour"`    |Hourly windows             |
|`"1 day"`     |Daily windows (most common)|
|`"1 week"`    |Weekly windows             |


> Multiple granularities are supported simultaneously: `["1 hour", "1 day"]`  
> Each granularity produces independent rows in the metric tables.

-----

*Databricks Lakehouse Monitoring — Unity Catalog · Python SDK `databricks-sdk` · REST API v2.1*
