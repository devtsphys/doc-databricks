# Databricks Feature Store — Python API Reference Card

> **SDK**: `databricks.feature_store` (Legacy) · `databricks.feature_engineering` (Unity Catalog, v0.6+)  
> **Install**: `pip install databricks-feature-engineering`  
> **Docs**: [docs.databricks.com/en/machine-learning/feature-store](https://docs.databricks.com/en/machine-learning/feature-store/index.html)

-----

## 1. Client Initialization

```python
# Legacy Feature Store (Hive Metastore)
from databricks.feature_store import FeatureStoreClient
fs = FeatureStoreClient()

# Unity Catalog Feature Engineering (recommended, v0.6+)
from databricks.feature_engineering import FeatureEngineeringClient
fe = FeatureEngineeringClient()
```

-----

## 2. Complete Method Reference Table

### FeatureEngineeringClient (Unity Catalog)

|Method                 |Signature                                                                                                              |Description                                                 |
|-----------------------|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
|`create_table`         |`create_table(name, primary_keys, df=None, schema=None, partition_columns=None, description=None, tags=None, **kwargs)`|Create a new feature table                                  |
|`write_table`          |`write_table(name, df, mode="merge")`                                                                                  |Write features to a table (`merge` or `overwrite`)          |
|`read_table`           |`read_table(name)`                                                                                                     |Read all features from a table as a Spark DataFrame         |
|`get_table`            |`get_table(name)`                                                                                                      |Return metadata/FeatureTable object                         |
|`drop_table`           |`drop_table(name)`                                                                                                     |Permanently delete a feature table                          |
|`list_tables`          |`list_tables(catalog_name, db_name)`                                                                                   |List all feature tables in a database                       |
|`publish_table`        |`publish_table(name, online_store, ...)`                                                                               |Publish features to an online store                         |
|`log_model`            |`log_model(model, artifact_path, flavor, training_set, ...)`                                                           |Log a model packaged with feature lookup metadata           |
|`score_batch`          |`score_batch(model_uri, df, result_type=None)`                                                                         |Batch score using logged model + auto feature retrieval     |
|`create_training_set`  |`create_training_set(df, feature_lookups, label, exclude_columns=None)`                                                |Create a training dataset from point-in-time correct lookups|
|`evaluate_model`       |`evaluate_model(model_uri, dataset, ...)`                                                                              |Evaluate a logged model                                     |
|`add_tag`              |`add_tag(name, key, value)`                                                                                            |Add metadata tag to a feature table                         |
|`delete_tag`           |`delete_tag(name, key)`                                                                                                |Delete a tag from a feature table                           |
|`set_feature_table_tag`|`set_feature_table_tag(table_name, key, value)`                                                                        |Alias for add_tag                                           |
|`update_table`         |`update_table(name, description=None)`                                                                                 |Update table metadata                                       |

### FeatureStoreClient (Legacy / Hive Metastore)

|Method                |Signature                                                                                                               |Description                                        |
|----------------------|------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
|`create_table`        |`create_table(name, primary_keys, df=None, schema=None, partition_columns=None, description=None, path=None, tags=None)`|Create a new feature table                         |
|`register_table`      |`register_table(delta_table, primary_keys, description=None)`                                                           |Register an existing Delta table as a feature table|
|`write_table`         |`write_table(name, df, mode="merge")`                                                                                   |Write features (`merge` / `overwrite`)             |
|`read_table`          |`read_table(name)`                                                                                                      |Read feature table as DataFrame                    |
|`get_table`           |`get_table(name)`                                                                                                       |Get feature table metadata                         |
|`drop_table`          |`drop_table(name)`                                                                                                      |Delete feature table                               |
|`list_tables`         |`list_tables(db_name)`                                                                                                  |List tables in database                            |
|`publish_table`       |`publish_table(name, online_store, filter_condition=None, mode="merge")`                                                |Publish to online store                            |
|`log_model`           |`log_model(model, artifact_path, flavor, training_set, registered_model_name=None)`                                     |Log model with feature lineage                     |
|`score_batch`         |`score_batch(model_uri, df, result_type=None)`                                                                          |Batch scoring with feature retrieval               |
|`create_training_set` |`create_training_set(df, feature_lookups, label, exclude_columns=None)`                                                 |Build a training set                               |
|`add_tag`             |`add_tag(table_name, key, value)`                                                                                       |Add a tag                                          |
|`delete_tag`          |`delete_tag(table_name, key)`                                                                                           |Delete a tag                                       |
|`update_feature_table`|`update_feature_table(name, description)`                                                                               |Update description                                 |

-----

## 3. Core Constructs

### FeatureLookup

```python
from databricks.feature_engineering import FeatureLookup
# from databricks.feature_store import FeatureLookup  # legacy

FeatureLookup(
    table_name="catalog.schema.customer_features",  # fully-qualified (UC) or db.table (legacy)
    feature_names=["age", "lifetime_value", "churn_score"],  # None = all features
    lookup_key="customer_id",                        # join key in training df
    timestamp_lookup_key=None,                       # for point-in-time lookups
    rename_outputs={"churn_score": "target_score"},  # optional rename
)
```

### FeatureFunction (on-demand features)

```python
from databricks.feature_engineering import FeatureFunction

FeatureFunction(
    udf_name="catalog.schema.calculate_age_in_days",  # registered Python/SQL UDF
    output_name="age_in_days",
    input_bindings={"birthday": "customer_birthday"},  # map UDF params → df columns
)
```

### TrainingSet

```python
training_set = fe.create_training_set(
    df=label_df,                    # DataFrame with keys + label
    feature_lookups=[
        FeatureLookup(table_name="...", feature_names=["f1","f2"], lookup_key="id"),
        FeatureFunction(udf_name="...", output_name="computed_feat", input_bindings={...}),
    ],
    label="churn",                  # target column name
    exclude_columns=["event_ts"],   # drop columns not needed in training
)
training_df = training_set.load_df()  # returns Spark DataFrame
```

-----

## 4. End-to-End Workflow Examples

### 4.1 Create & Write a Feature Table

```python
from databricks.feature_engineering import FeatureEngineeringClient
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()
fe = FeatureEngineeringClient()

# Compute features
customer_features_df = spark.sql("""
    SELECT
        customer_id,
        COUNT(*) AS num_purchases,
        SUM(amount) AS total_spend,
        AVG(amount) AS avg_order_value,
        DATEDIFF(current_date(), MAX(order_date)) AS days_since_last_purchase
    FROM orders
    GROUP BY customer_id
""")

# Create table (first time)
fe.create_table(
    name="catalog.ml.customer_features",
    primary_keys=["customer_id"],
    df=customer_features_df,
    partition_columns=["signup_year"],  # optional
    description="Customer-level aggregated purchase features",
    tags={"team": "data-science", "domain": "ecommerce"},
)

# Subsequent updates — merge (upsert) by primary key
fe.write_table(
    name="catalog.ml.customer_features",
    df=customer_features_df,
    mode="merge",       # or "overwrite"
)
```

### 4.2 Point-in-Time Correct Training Set

```python
# Label DataFrame — must include timestamp column
label_df = spark.sql("""
    SELECT customer_id, churn_label, event_timestamp
    FROM churn_labels
""")

training_set = fe.create_training_set(
    df=label_df,
    feature_lookups=[
        FeatureLookup(
            table_name="catalog.ml.customer_features",
            feature_names=["num_purchases", "total_spend", "avg_order_value"],
            lookup_key="customer_id",
            timestamp_lookup_key="event_timestamp",  # point-in-time join
        )
    ],
    label="churn_label",
    exclude_columns=["event_timestamp"],
)

train_df = training_set.load_df().toPandas()
```

### 4.3 Train & Log a Model with Feature Lineage

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier

X = train_df.drop(columns=["churn_label", "customer_id"])
y = train_df["churn_label"]

model = GradientBoostingClassifier(n_estimators=100)
model.fit(X, y)

with mlflow.start_run():
    fe.log_model(
        model=model,
        artifact_path="churn_model",
        flavor=mlflow.sklearn,
        training_set=training_set,
        registered_model_name="catalog.ml.churn_model_v2",
    )
```

### 4.4 Batch Scoring with Automatic Feature Retrieval

```python
# Only needs the lookup keys — features fetched automatically
inference_df = spark.createDataFrame(
    [(1001,), (1002,), (1003,)], ["customer_id"]
)

predictions = fe.score_batch(
    model_uri="models:/catalog.ml.churn_model_v2/Production",
    df=inference_df,
    result_type="float",
)
# predictions contains customer_id + prediction column
display(predictions)
```

### 4.5 Register an Existing Delta Table

```python
# Legacy / FeatureStoreClient
fs = FeatureStoreClient()
fs.register_table(
    delta_table="ml.existing_delta_features",
    primary_keys=["user_id"],
    description="Pre-existing Delta table registered as feature table",
)
```

### 4.6 Publish to Online Store (DynamoDB)

```python
from databricks.feature_store.online_store_spec import AmazonDynamoDBSpec

online_store = AmazonDynamoDBSpec(
    region="us-east-1",
    write_secret_prefix="feature-store/dynamo",
    read_secret_prefix="feature-store/dynamo",
    table_name="customer_features_online",
)

fs.publish_table(
    name="ml.customer_features",
    online_store=online_store,
    mode="merge",
)
```

-----

## 5. Online Store Specs

```python
# Amazon DynamoDB
from databricks.feature_store.online_store_spec import AmazonDynamoDBSpec
spec = AmazonDynamoDBSpec(region, write_secret_prefix, read_secret_prefix, table_name)

# Amazon RDS MySQL
from databricks.feature_store.online_store_spec import AmazonRdsMySqlSpec
spec = AmazonRdsMySqlSpec(hostname, port, read_secret_prefix, write_secret_prefix, db_name, table_name)

# Azure Cosmos DB
from databricks.feature_store.online_store_spec import AzureCosmosDBSpec
spec = AzureCosmosDBSpec(account_uri, read_secret_prefix, write_secret_prefix, database_name, container_name)

# Azure SQL / MySQL
from databricks.feature_store.online_store_spec import AzureMySqlSpec
spec = AzureMySqlSpec(hostname, port, read_secret_prefix, write_secret_prefix, db_name, table_name)
```

-----

## 6. On-Demand (Real-Time) Features

```python
# 1. Register a Python UDF in Unity Catalog
spark.sql("""
    CREATE OR REPLACE FUNCTION catalog.ml.haversine_distance(
        lat1 DOUBLE, lon1 DOUBLE, lat2 DOUBLE, lon2 DOUBLE
    )
    RETURNS DOUBLE
    LANGUAGE PYTHON
    AS $$
        from math import radians, cos, sin, asin, sqrt
        R = 6371
        lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
        dlat = lat2 - lat1
        dlon = lon2 - lon1
        a = sin(dlat/2)**2 + cos(lat1)*cos(lat2)*sin(dlon/2)**2
        return 2 * R * asin(sqrt(a))
    $$
""")

# 2. Use it in a FeatureLookup pipeline
training_set = fe.create_training_set(
    df=label_df,
    feature_lookups=[
        FeatureFunction(
            udf_name="catalog.ml.haversine_distance",
            output_name="distance_to_store_km",
            input_bindings={
                "lat1": "user_lat", "lon1": "user_lon",
                "lat2": "store_lat", "lon2": "store_lon",
            },
        )
    ],
    label="purchased",
)
```

-----

## 7. Metadata & Governance

```python
# Tags
fe.add_tag(name="catalog.schema.my_table", key="pii", value="true")
fe.delete_tag(name="catalog.schema.my_table", key="pii")

# Read metadata
table_meta = fe.get_table("catalog.schema.my_table")
print(table_meta.name)
print(table_meta.primary_keys)
print(table_meta.description)
print(table_meta.tags)
print(table_meta.features)         # list of Feature objects
print(table_meta.creation_timestamp)
print(table_meta.partition_columns)

# List tables
tables = fe.list_tables(catalog_name="catalog", db_name="ml")
for t in tables:
    print(t.name, t.description)

# Update description
fe.update_table(name="catalog.schema.my_table", description="Updated description")

# Drop
fe.drop_table(name="catalog.schema.my_table")
```

-----

## 8. Design Patterns

### Pattern 1 — Feature Reuse Across Teams

```python
# Team A writes marketing features
fe.write_table("catalog.shared.marketing_features", marketing_df, mode="merge")

# Team B reuses them in their training pipeline
FeatureLookup(
    table_name="catalog.shared.marketing_features",
    feature_names=["email_open_rate", "ad_clicks_30d"],
    lookup_key="user_id",
)
```

### Pattern 2 — Incremental / Streaming Feature Updates

```python
(
    spark.readStream
        .format("delta")
        .table("bronze.events")
        .groupBy("user_id", window("event_time", "1 hour"))
        .agg(count("*").alias("events_per_hour"))
        .writeStream
        .format("delta")
        .outputMode("complete")
        .option("checkpointLocation", "/tmp/chk/user_hourly")
        .toTable("catalog.ml.user_activity_features")
)
```

### Pattern 3 — Feature Versioning via Tags

```python
fe.add_tag("catalog.ml.customer_features", "version", "v3")
fe.add_tag("catalog.ml.customer_features", "status", "production")
fe.add_tag("catalog.ml.customer_features", "deprecated_by", "v4")
```

### Pattern 4 — Multi-Key Lookups

```python
FeatureLookup(
    table_name="catalog.ml.product_user_features",
    feature_names=["affinity_score"],
    lookup_key=["user_id", "product_category"],  # composite key
)
```

### Pattern 5 — Exclude Raw Keys from Training

```python
training_set = fe.create_training_set(
    df=label_df,
    feature_lookups=[...],
    label="target",
    exclude_columns=["user_id", "event_ts", "session_id"],  # avoid leakage
)
```

-----

## 9. Best Practices

|Category              |Recommendation                                                                       |
|----------------------|-------------------------------------------------------------------------------------|
|**Primary Keys**      |Use stable, non-nullable, business-meaningful keys (e.g., `user_id`, not row hashes) |
|**Write Mode**        |Prefer `merge` for incremental updates; use `overwrite` only for full refreshes      |
|**Point-in-Time**     |Always use `timestamp_lookup_key` when training data has temporal context            |
|**Naming**            |Use `catalog.schema.feature_table` fully-qualified names in Unity Catalog            |
|**On-Demand Features**|Use `FeatureFunction` for features computed at inference time to avoid staleness     |
|**Partition Columns** |Partition on low-cardinality columns (date, region) to improve read performance      |
|**Avoid Data Leakage**|Always `exclude_columns` for keys and timestamps in `create_training_set`            |
|**Model Logging**     |Always use `fe.log_model` (not `mlflow.log_model`) to preserve feature lineage       |
|**Batch Scoring**     |Use `fe.score_batch` — it auto-joins features and prevents training-serving skew     |
|**Tags**              |Tag tables with `team`, `domain`, `pii`, `version`, `status` for governance          |
|**Schema Evolution**  |Add nullable columns only; avoid removing or renaming existing columns               |
|**Freshness SLAs**    |Use Delta Lake `OPTIMIZE` + `ZORDER` on lookup keys for fast online reads            |
|**Testing**           |Validate feature distributions at write time with Great Expectations or custom checks|
|**Access Control**    |Grant `SELECT` on feature tables via Unity Catalog GRANT statements                  |

-----

## 10. Common Errors & Solutions

|Error                             |Cause                                  |Fix                                                    |
|----------------------------------|---------------------------------------|-------------------------------------------------------|
|`FeatureTableNotFoundException`   |Table doesn’t exist or wrong name      |Verify with `fe.list_tables()`                         |
|`PrimaryKeyNotFoundException`     |Missing primary key column in DataFrame|Ensure all `primary_keys` exist in the DataFrame       |
|`MergeKeyConflictException`       |Duplicate rows for same primary key    |Deduplicate df before `write_table`                    |
|`SchemaEvolutionException`        |New column has incompatible type       |Match schema or use `overwrite` mode for schema changes|
|`TimestampKeyNotInDFException`    |`timestamp_lookup_key` column missing  |Add the column to your label DataFrame                 |
|`FeatureFunctionNotFoundException`|UDF not registered in Unity Catalog    |Run `CREATE FUNCTION` SQL first                        |
|Model serves stale features       |Online store not refreshed             |Schedule a `publish_table` job after each `write_table`|

-----

## 11. Spark SQL — Feature Table DDL

```sql
-- Read a feature table like any Delta table
SELECT * FROM catalog.ml.customer_features WHERE customer_id = 12345;

-- Grant access
GRANT SELECT ON TABLE catalog.ml.customer_features TO `data-scientists`;
GRANT MODIFY ON TABLE catalog.ml.customer_features TO `etl-jobs`;

-- Optimize for lookup performance
OPTIMIZE catalog.ml.customer_features ZORDER BY (customer_id);

-- Check table history
DESCRIBE HISTORY catalog.ml.customer_features;
```

-----

## 12. Quick Reference: write_table Modes

|Mode       |Behavior                                                       |Use When                          |
|-----------|---------------------------------------------------------------|----------------------------------|
|`merge`    |Upsert by primary key; new rows inserted, existing rows updated|Incremental / streaming updates   |
|`overwrite`|Replace entire table                                           |Full daily refresh, schema changes|

-----

## 13. Client API Comparison

|Feature                    |`FeatureStoreClient` (Legacy)|`FeatureEngineeringClient` (UC)|
|---------------------------|-----------------------------|-------------------------------|
|Metastore                  |Hive / workspace             |Unity Catalog                  |
|Table names                |`db.table`                   |`catalog.schema.table`         |
|On-demand features         |❌                            |✅ `FeatureFunction`            |
|Cross-workspace sharing    |❌                            |✅                              |
|Fine-grained access control|Limited                      |✅ UC RBAC                      |
|Recommended                |No (legacy)                  |✅ Yes                          |

-----

*Last updated: 2025 · Databricks Runtime 14.x+ · Feature Engineering Client 0.6+*