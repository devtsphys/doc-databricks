# Databricks Utilities & SDKs — Reference Card

> **Scope:** `dbutils`, `mlflow` / `MlflowClient`, `WorkspaceClient` (Databricks SDK), `FeatureEngineeringClient`, `DeltaTable`, `spark` session utilities, and key supporting libraries.  
> All examples target **Databricks Runtime 14+**, **Python 3.10+**, **Unity Catalog**-enabled workspaces.

-----

## Table of Contents

1. [dbutils — Databricks Utilities](#1-dbutils--databricks-utilities)
1. [MLflow Tracking API (fluent)](#2-mlflow-tracking-api-fluent)
1. [MlflowClient (programmatic)](#3-mlflowclient-programmatic)
1. [WorkspaceClient (Databricks SDK)](#4-workspaceclient-databricks-sdk)
1. [FeatureEngineeringClient](#5-featureengineeringclient)
1. [DeltaTable API](#6-deltatable-api)
1. [Spark Session & Context Utilities](#7-spark-session--context-utilities)
1. [Unity Catalog Helpers](#8-unity-catalog-helpers)
1. [Design Patterns & Best Practices](#9-design-patterns--best-practices)
1. [Method Quick-Reference Tables](#10-method-quick-reference-tables)

-----

## 1. `dbutils` — Databricks Utilities

### 1.1 Initialisation

```python
# In a Databricks notebook — dbutils is injected automatically
# In a local / CI context (Databricks Connect 13+):
from databricks.sdk.runtime import dbutils          # SDK shim
# OR via databricks-connect:
from pyspark.dbutils import DBUtils
dbutils = DBUtils(spark)
```

-----

### 1.2 `dbutils.fs` — FileSystem Utilities

```python
# List a path
files = dbutils.fs.ls("dbfs:/mnt/raw/")           # returns list[FileInfo]
for f in files:
    print(f.path, f.name, f.size, f.modificationTime)

# Read / write text
text = dbutils.fs.head("dbfs:/mnt/raw/file.txt", maxBytes=65536)
dbutils.fs.put("dbfs:/tmp/test.txt", "hello world", overwrite=True)

# Copy, move, delete
dbutils.fs.cp("dbfs:/mnt/src/", "dbfs:/mnt/dst/", recurse=True)
dbutils.fs.mv("dbfs:/mnt/old/file.csv", "dbfs:/mnt/new/file.csv")
dbutils.fs.rm("dbfs:/mnt/tmp/", recurse=True)

# Mount (legacy — prefer Unity Catalog external locations)
dbutils.fs.mount(
    source       = "wasbs://container@account.blob.core.windows.net/",
    mount_point  = "/mnt/adls",
    extra_configs= {"fs.azure.account.key.account.blob.core.windows.net": dbutils.secrets.get("scope","key")}
)
dbutils.fs.unmount("/mnt/adls")
dbutils.fs.mounts()                                 # list all mounts

# Refresh cached mount table
dbutils.fs.refreshMounts()

# DBFS shortcuts
dbutils.fs.ls("abfss://container@account.dfs.core.windows.net/")
```

**`FileInfo` attributes:** `path`, `name`, `size`, `modificationTime`

-----

### 1.3 `dbutils.secrets` — Secret Utilities

```python
# List scopes and keys (names only — values are redacted)
dbutils.secrets.listScopes()                        # list[SecretScope]
dbutils.secrets.list("my-scope")                   # list[SecretMetadata]

# Retrieve a secret
token = dbutils.secrets.get(scope="my-scope", key="my-key")   # str
raw   = dbutils.secrets.getBytes(scope="my-scope", key="key") # bytes (for binary secrets)

# Use in connection string — value is REDACTED in notebook output
jdbc_url = f"jdbc:sqlserver://host;password={dbutils.secrets.get('scope','sql-pw')}"
```

> **Best practice:** never `print()` a secret value; Databricks redacts it automatically, but log frameworks may not.

-----

### 1.4 `dbutils.widgets` — Notebook Widgets

```python
# Create widgets
dbutils.widgets.text("env", "dev", "Environment")
dbutils.widgets.dropdown("region", "eu-west-1", ["eu-west-1","us-east-1"], "Region")
dbutils.widgets.multiselect("tables", "orders", ["orders","customers","products"], "Tables")
dbutils.widgets.combobox("model", "xgboost", ["xgboost","lgbm","rf"], "Model")

# Read values
env    = dbutils.widgets.get("env")
tables = dbutils.widgets.get("tables").split(",")

# Remove
dbutils.widgets.remove("env")
dbutils.widgets.removeAll()
```

-----

### 1.5 `dbutils.notebook` — Notebook Orchestration

```python
# Run a child notebook — returns its exit value as a string
result = dbutils.notebook.run(
    path      = "/Repos/project/notebooks/etl_job",
    timeout_seconds = 3600,
    arguments = {"env": "prod", "date": "2024-01-01"}
)

# Exit current notebook with a value (consumed by parent run call)
dbutils.notebook.exit("SUCCESS: 42 rows written")
```

-----

### 1.6 `dbutils.jobs` — Job Context

```python
# Available inside a Databricks Job run
ctx = dbutils.jobs.taskValues

# Set a task output value
ctx.set(key="row_count", value=42)
ctx.set(key="status",    value="SUCCESS")

# Get a value from an upstream task
upstream_count = ctx.get(taskKey="ingest_task", key="row_count", default=0)
upstream_count = ctx.get(taskKey="ingest_task", key="row_count", debugValue=999)  # local testing
```

-----

### 1.7 `dbutils.data` (Preview)

```python
# Quick profiling of a DataFrame in a notebook
dbutils.data.summarize(df, precise=True)           # rich HTML summary table
```

-----

## 2. MLflow Tracking API (fluent)

```python
import mlflow
import mlflow.sklearn        # or mlflow.pyfunc, mlflow.spark, etc.

mlflow.set_tracking_uri("databricks")              # automatic inside Databricks
mlflow.set_registry_uri("databricks-uc")           # Unity Catalog model registry
```

### 2.1 Experiments

```python
# Set by name (creates if absent)
mlflow.set_experiment("/Users/me@org.com/my-experiment")

# Set by ID
mlflow.set_experiment(experiment_id="123456789")

# Retrieve experiment object
exp = mlflow.get_experiment_by_name("/Users/me@org.com/my-experiment")
print(exp.experiment_id, exp.artifact_location)
```

### 2.2 Runs

```python
with mlflow.start_run(run_name="xgb-baseline") as run:
    run_id = run.info.run_id

    # Params, metrics, tags
    mlflow.log_param("learning_rate", 0.05)
    mlflow.log_params({"max_depth": 6, "n_estimators": 200})

    mlflow.log_metric("rmse", 0.123)
    mlflow.log_metric("rmse", 0.119, step=1)        # step for charts
    mlflow.log_metrics({"mae": 0.09, "r2": 0.95})

    mlflow.set_tag("team", "mlops")
    mlflow.set_tags({"framework": "xgboost", "env": "prod"})

    # Artifacts
    mlflow.log_artifact("report.html")
    mlflow.log_artifacts("outputs/", artifact_path="plots")
    mlflow.log_text("plain text content", "notes.txt")
    mlflow.log_dict({"config": {"lr": 0.05}}, "config.json")
    mlflow.log_figure(fig, "feature_importance.png")   # matplotlib/plotly
    mlflow.log_image(np_img, "sample.png")             # numpy array

    # Model logging
    mlflow.sklearn.log_model(
        model,
        artifact_path = "model",
        registered_model_name = "main.ml.my_model",    # UC: catalog.schema.name
        input_example = X_train.head(5),
        signature     = mlflow.models.infer_signature(X_train, y_pred),
    )

# Nested runs
with mlflow.start_run(run_name="parent"):
    with mlflow.start_run(run_name="child-fold-1", nested=True):
        mlflow.log_metric("cv_rmse", 0.12)
```

### 2.3 Autolog

```python
mlflow.autolog()                                   # enable for all supported flavours
mlflow.sklearn.autolog(log_models=True, log_input_examples=True)
mlflow.xgboost.autolog()
mlflow.pytorch.autolog(log_every_n_epoch=5)
mlflow.spark.autolog()

mlflow.autolog(disable=True)                       # disable
```

### 2.4 Loading Models

```python
# Load as Python function (generic)
model = mlflow.pyfunc.load_model("runs:/RUN_ID/model")
model = mlflow.pyfunc.load_model("models:/main.ml.my_model/3")          # version
model = mlflow.pyfunc.load_model("models:/main.ml.my_model@champion")   # alias

# Native flavour
model = mlflow.sklearn.load_model("runs:/RUN_ID/model")
model = mlflow.spark.load_model("models:/main.ml.my_model/Production")

# Spark UDF for batch scoring
predict_udf = mlflow.pyfunc.spark_udf(spark, "models:/main.ml.my_model@champion",
                                       result_type="double")
scored_df = df.withColumn("prediction", predict_udf(*feature_cols))
```

-----

## 3. `MlflowClient` (programmatic)

```python
from mlflow.tracking import MlflowClient

client = MlflowClient(
    tracking_uri  = "databricks",
    registry_uri  = "databricks-uc",
)
```

### 3.1 Experiment Methods

```python
exp    = client.get_experiment("123456")
exp    = client.get_experiment_by_name("/Users/me/exp")
exp_id = client.create_experiment("/Users/me/new-exp",
            artifact_location="dbfs:/mlflow/exp",
            tags={"project": "churn"})
client.rename_experiment(exp_id, "/Users/me/renamed-exp")
client.set_experiment_tag(exp_id, "status", "archived")
client.delete_experiment(exp_id)
client.restore_experiment(exp_id)
exps   = client.search_experiments(
            filter_string = "tags.project = 'churn'",
            order_by      = ["last_update_time DESC"],
            max_results   = 50)
```

### 3.2 Run Methods

```python
run    = client.get_run(run_id)
run.info.run_id
run.info.status            # RUNNING | FINISHED | FAILED | KILLED
run.data.params
run.data.metrics           # latest value per key
run.data.tags

# Create / update runs
run    = client.create_run(experiment_id=exp_id, run_name="manual-run",
                           tags={"mlflow.user": "tim"})
client.log_param(run_id, "alpha", 0.1)
client.log_metric(run_id, "auc", 0.87, step=10)
client.set_tag(run_id, "env", "prod")
client.set_terminated(run_id, status="FINISHED")   # FINISHED | FAILED | KILLED
client.delete_run(run_id)
client.restore_run(run_id)

# Search runs
runs = client.search_runs(
    experiment_ids = [exp_id],
    filter_string  = "metrics.rmse < 0.15 AND params.model = 'xgb'",
    order_by       = ["metrics.rmse ASC"],
    max_results    = 100,
    run_view_type  = mlflow.entities.ViewType.ACTIVE_ONLY,
)

# Artifacts
artifacts = client.list_artifacts(run_id, path="plots")  # list[FileInfo]
local_path = client.download_artifacts(run_id, "model/model.pkl", dst_path="/tmp/")
```

### 3.3 Registered Model Methods (Unity Catalog)

```python
# Create / get
client.create_registered_model(
    name        = "main.ml.my_model",
    tags        = {"team": "fraud"},
    description = "XGBoost fraud classifier",
)
rm = client.get_registered_model("main.ml.my_model")
client.rename_registered_model("main.ml.my_model", "main.ml.fraud_model")
client.update_registered_model("main.ml.fraud_model", description="Updated desc")
client.delete_registered_model("main.ml.fraud_model")

# List / search
models = client.search_registered_models(
    filter_string = "tags.team = 'fraud'",
    order_by      = ["name ASC"],
    max_results   = 200,
)
```

### 3.4 Model Version Methods

```python
# Create from run
mv = client.create_model_version(
    name         = "main.ml.my_model",
    source       = f"runs:/{run_id}/model",
    run_id       = run_id,
    description  = "Trained on 2024-Q1 data",
    tags         = {"data_cut": "2024-Q1"},
)
mv.version                             # "1", "2", etc.

# Get / update / delete
mv  = client.get_model_version("main.ml.my_model", "3")
client.update_model_version("main.ml.my_model", "3", description="Production ready")
client.delete_model_version("main.ml.my_model", "3")

# Aliases (UC replaces Stages)
client.set_registered_model_alias("main.ml.my_model", "champion", "3")
client.delete_registered_model_alias("main.ml.my_model", "challenger")
mv  = client.get_model_version_by_alias("main.ml.my_model", "champion")

# Tags on versions
client.set_model_version_tag("main.ml.my_model", "3", "approved", "true")
client.delete_model_version_tag("main.ml.my_model", "3", "approved")

# Download model artifacts
local = client.download_model_version_artifact(
            "main.ml.my_model", "3", artifact_path="", dst_path="/tmp/model")

# Search versions
versions = client.search_model_versions(
    filter_string = "name='main.ml.my_model' AND tags.approved='true'",
    order_by      = ["version_number DESC"],
    max_results   = 10,
)
```

-----

## 4. `WorkspaceClient` (Databricks SDK)

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service import jobs, clusters, sql, catalog, iam

# Authentication — picks up env vars, ~/.databrickscfg, or instance profile automatically
w = WorkspaceClient()

# Explicit config
w = WorkspaceClient(
    host  = "https://adb-xxxx.azuredatabricks.net",
    token = dbutils.secrets.get("scope", "pat"),
)
```

### 4.1 Jobs

```python
# List jobs
for job in w.jobs.list(name="my_job"):
    print(job.job_id, job.settings.name)

# Get
job = w.jobs.get(job_id=12345)

# Create
created = w.jobs.create(
    name    = "FeatureStore_Refresh",
    tasks   = [
        jobs.Task(
            task_key    = "ingest",
            description = "Ingest raw data",
            new_cluster = clusters.ClusterSpec(
                spark_version      = "14.3.x-scala2.12",
                node_type_id       = "Standard_DS3_v2",
                num_workers        = 4,
                spark_conf         = {"spark.databricks.delta.optimizeWrite.enabled": "true"},
            ),
            notebook_task = jobs.NotebookTask(notebook_path="/Repos/project/notebooks/ingest"),
        )
    ],
    schedule = jobs.CronSchedule(
        quartz_cron_expression = "0 0 3 * * ?",
        timezone_id            = "Europe/Berlin",
        pause_status           = jobs.PauseStatus.UNPAUSED,
    ),
)

# Run now
run = w.jobs.run_now(job_id=created.job_id,
                     notebook_params={"env": "prod"})

# Wait for completion
run_result = w.jobs.wait_get_run_job_terminated_or_skipped(run_id=run.run_id)

# Cancel / delete
w.jobs.cancel_run(run_id=run.run_id)
w.jobs.delete(job_id=created.job_id)

# List runs
for run in w.jobs.list_runs(job_id=12345, active_only=False):
    print(run.run_id, run.state.result_state)
```

### 4.2 Clusters

```python
# List
for c in w.clusters.list():
    print(c.cluster_id, c.cluster_name, c.state)

# Get / start / stop / restart
c = w.clusters.get(cluster_id="xxxx-xxxxxx-xxxxxxxx")
w.clusters.start(cluster_id=c.cluster_id)
w.clusters.wait_get_cluster_running(cluster_id=c.cluster_id)
w.clusters.restart(cluster_id=c.cluster_id)
w.clusters.terminate(cluster_id=c.cluster_id)
w.clusters.wait_get_cluster_terminated(cluster_id=c.cluster_id)

# Create all-purpose cluster
new_c = w.clusters.create(
    cluster_name   = "dev-shared",
    spark_version  = "14.3.x-scala2.12",
    node_type_id   = "Standard_DS3_v2",
    autoscale      = clusters.AutoScale(min_workers=2, max_workers=8),
    autotermination_minutes = 30,
    spark_conf     = {"spark.sql.shuffle.partitions": "auto"},
    custom_tags    = {"cost_centre": "ml-team"},
)

# Spark versions & node types
versions = w.clusters.spark_versions()
node_types = w.clusters.list_node_types()
```

### 4.3 Secrets

```python
# Scopes
w.secrets.create_scope(scope="my-scope")
w.secrets.list_scopes()
w.secrets.delete_scope(scope="old-scope")

# Secrets
w.secrets.put_secret(scope="my-scope", key="my-key", string_value="s3cr3t")
w.secrets.put_secret(scope="my-scope", key="binary-key", bytes_value=b"\x00\x01")
w.secrets.list_secrets(scope="my-scope")        # metadata only
w.secrets.delete_secret(scope="my-scope", key="my-key")

# ACLs
w.secrets.put_acl(scope="my-scope", principal="users", permission=iam.AclPermission.READ)
w.secrets.list_acls(scope="my-scope")
```

### 4.4 DBFS / Files

```python
# DBFS
w.dbfs.mkdirs(path="dbfs:/tmp/mydir")
w.dbfs.list(path="dbfs:/tmp/")
w.dbfs.move(source_path="dbfs:/tmp/a.txt", destination_path="dbfs:/tmp/b.txt")
w.dbfs.delete(path="dbfs:/tmp/old/", recursive=True)

# Files API (Unity Catalog Volumes)
w.files.upload(file_path="/Volumes/main/data/raw/file.csv",
               contents=open("local.csv","rb"))
info = w.files.get_metadata(file_path="/Volumes/main/data/raw/file.csv")
w.files.download(file_path="/Volumes/main/data/raw/file.csv")
w.files.delete(file_path="/Volumes/main/data/raw/old.csv")
```

### 4.5 Unity Catalog — Catalogs, Schemas, Tables

```python
# Catalogs
for cat in w.catalogs.list():
    print(cat.name)
w.catalogs.create(name="bronze", comment="Raw ingestion layer")
w.catalogs.get(name="bronze")
w.catalogs.update(name="bronze", comment="Updated comment")
w.catalogs.delete(name="old_catalog", force=True)

# Schemas
for schema in w.schemas.list(catalog_name="main"):
    print(schema.name)
w.schemas.create(name="ml", catalog_name="main", comment="ML features")
w.schemas.get(full_name="main.ml")

# Tables
for tbl in w.tables.list(catalog_name="main", schema_name="ml"):
    print(tbl.name, tbl.table_type)
tbl = w.tables.get(full_name="main.ml.customer_features")
w.tables.delete(full_name="main.ml.old_table")

# Table existence check
exists = w.tables.exists(full_name="main.ml.customer_features")
```

### 4.6 Workspace (Notebooks, Repos)

```python
# Notebooks / workspace objects
w.workspace.list(path="/Users/me@org.com")
w.workspace.get_status(path="/Repos/project/notebooks/ingest")
w.workspace.export(path="/Repos/project/notebooks/ingest",
                   format=workspace.ExportFormat.SOURCE)
w.workspace.import_(path="/Repos/imported", content=b64_content,
                    format=workspace.ImportFormat.AUTO,
                    language=workspace.Language.PYTHON, overwrite=True)
w.workspace.delete(path="/Repos/tmp/scratch.py")
w.workspace.mkdirs(path="/Users/me@org.com/temp")

# Repos
for repo in w.repos.list():
    print(repo.id, repo.path, repo.branch)
w.repos.create(url="https://github.com/org/project.git",
               provider="gitHub", path="/Repos/project")
w.repos.update(repo_id=123, branch="main")
w.repos.delete(repo_id=123)
```

### 4.7 SQL Warehouses

```python
for wh in w.warehouses.list():
    print(wh.id, wh.name, wh.state)

wh = w.warehouses.create(
    name            = "shared-serverless",
    cluster_size    = "Small",
    warehouse_type  = sql.CreateWarehouseRequestWarehouseType.PRO,
    auto_stop_mins  = 10,
    enable_serverless_compute = True,
)
w.warehouses.start(id=wh.id)
w.warehouses.wait_get_warehouse_running(id=wh.id)
w.warehouses.stop(id=wh.id)

# Execute SQL
stmt = w.statement_execution.execute_statement(
    warehouse_id = wh.id,
    statement    = "SELECT count(*) FROM main.ml.customer_features",
    wait_timeout = "30s",
)
result = stmt.result.data_array      # list of rows as lists
```

### 4.8 Users & Groups (IAM)

```python
# Users
for user in w.users.list():
    print(user.id, user.user_name)
u = w.users.create(
    user_name    = "alice@org.com",
    display_name = "Alice Smith",
    emails       = [iam.ComplexValue(value="alice@org.com", primary=True)],
)
w.users.update(id=u.id, display_name="Alice J. Smith")
w.users.delete(id=u.id)

# Groups
for grp in w.groups.list():
    print(grp.id, grp.display_name)
g = w.groups.create(display_name="ml-engineers")
w.groups.patch(id=g.id, operations=[iam.Patch(
    op=iam.PatchOp.ADD,
    path="members",
    value=[{"value": str(u.id)}],
)])
```

-----

## 5. `FeatureEngineeringClient`

```python
from databricks.feature_engineering import FeatureEngineeringClient, FeatureLookup, FeatureFunction

fe = FeatureEngineeringClient()
```

### 5.1 Feature Tables

```python
# Create feature table (Delta table backed by Unity Catalog)
fe.create_table(
    name            = "main.ml.customer_features",
    primary_keys    = ["customer_id"],
    timestamp_keys  = ["event_date"],          # optional — enables point-in-time lookup
    df              = features_df,
    description     = "Customer behavioural features",
    tags            = {"domain": "customer", "team": "ml"},
)

# Write features — merge by primary key
fe.write_table(
    name            = "main.ml.customer_features",
    df              = updated_df,
    mode            = "merge",                 # merge | overwrite
)

# Read feature table as Spark DataFrame
df = fe.read_table(name="main.ml.customer_features")

# Get metadata
ft = fe.get_table(name="main.ml.customer_features")
print(ft.name, ft.primary_keys, ft.timestamp_keys, ft.description)

# List feature tables in schema
tables = fe.list_tables(parent_path="main.ml")
for t in tables:
    print(t.name)

# Delete (removes Unity Catalog table)
fe.drop_table(name="main.ml.customer_features")
```

### 5.2 Creating Training Datasets

```python
from databricks.feature_engineering import FeatureLookup

# Training labels / spine
labels_df = spark.table("main.ml.churn_labels")   # must contain primary_key column(s)

feature_lookups = [
    FeatureLookup(
        table_name     = "main.ml.customer_features",
        feature_names  = ["lifetime_value", "days_since_last_order", "order_count"],
        lookup_key     = "customer_id",
        # For point-in-time joins:
        timestamp_lookup_key = "label_date",
    ),
    FeatureLookup(
        table_name     = "main.ml.product_features",
        feature_names  = None,                     # None = all features
        lookup_key     = ["customer_id", "product_id"],
        rename_outputs = {"price_bucket": "prod_price_bucket"},
    ),
]

training_set = fe.create_training_set(
    df              = labels_df,
    feature_lookups = feature_lookups,
    label           = "churned",
    exclude_columns = ["label_date"],              # columns to drop from output
)

training_df = training_set.load_df()               # returns Spark DataFrame
```

### 5.3 Logging & Loading Models with Feature Engineering

```python
import mlflow

# Log model — lineage records which feature tables are used
with mlflow.start_run():
    fe.log_model(
        model           = pipeline,
        artifact_path   = "model",
        flavor          = mlflow.sklearn,
        training_set    = training_set,
        registered_model_name = "main.ml.churn_model",
        input_example   = training_df.limit(5).toPandas(),
    )

# Batch scoring — feature values auto-fetched from feature store
batch_df = spark.table("main.ml.scoring_spine")   # must contain lookup keys

predictions = fe.score_batch(
    model_uri    = "models:/main.ml.churn_model@champion",
    df           = batch_df,                       # spine only, no need to pass feature cols
    result_type  = "double",
    env_manager  = "virtualenv",                   # conda | virtualenv | local
)
# predictions is a Spark DataFrame with a "prediction" column appended
```

### 5.4 Feature Functions (on-demand)

```python
from databricks.feature_engineering import FeatureFunction

# Register a SQL function in UC as an on-demand feature
# (SQL UDF must already exist in the catalog)
# e.g.: CREATE FUNCTION main.ml.days_since(last_date DATE) RETURNS INT ...

feature_lookups_with_fn = [
    FeatureLookup(
        table_name    = "main.ml.customer_features",
        lookup_key    = "customer_id",
        feature_names = ["last_order_date"],
    ),
    FeatureFunction(
        udf_name   = "main.ml.days_since",
        output_name= "days_since_last_order",
        input_bindings = {"last_date": "last_order_date"},
    ),
]

training_set = fe.create_training_set(
    df              = labels_df,
    feature_lookups = feature_lookups_with_fn,
    label           = "churned",
)
```

-----

## 6. `DeltaTable` API

```python
from delta.tables import DeltaTable
```

### 6.1 Reading & Creating

```python
# Load existing table
dt = DeltaTable.forName(spark, "main.ml.events")
dt = DeltaTable.forPath(spark, "abfss://container@account.dfs.core.windows.net/events")

# Check existence
DeltaTable.isDeltaTable(spark, "abfss://...")     # bool

# Create
DeltaTable.createIfNotExists(spark) \
    .tableName("main.bronze.events") \
    .addColumn("event_id",   "BIGINT",  nullable=False) \
    .addColumn("event_ts",   "TIMESTAMP") \
    .addColumn("payload",    "STRING") \
    .partitionedBy("date_col") \
    .property("delta.autoOptimize.optimizeWrite", "true") \
    .execute()
```

### 6.2 Merge (Upsert)

```python
# Standard upsert
DeltaTable.forName(spark, "main.ml.customer_features") \
    .alias("target") \
    .merge(
        source    = new_df.alias("source"),
        condition = "target.customer_id = source.customer_id"
    ) \
    .whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .whenNotMatchedBySourceDelete() \
    .execute()

# Conditional merge
dt.alias("t").merge(src.alias("s"), "t.id = s.id") \
    .whenMatchedUpdate(
        condition = "s.updated_at > t.updated_at",
        set       = {"name": "s.name", "score": "s.score", "updated_at": "s.updated_at"}
    ) \
    .whenNotMatchedInsert(
        values = {"id": "s.id", "name": "s.name", "score": "s.score",
                  "updated_at": "s.updated_at"}
    ) \
    .execute()
```

### 6.3 History & Time Travel

```python
dt = DeltaTable.forName(spark, "main.ml.events")

# History
history = dt.history()                            # DataFrame
history.select("version","timestamp","operation","operationParameters").show()
history = dt.history(limit=5)

# Time travel reads (Spark SQL / DataFrame API)
df_v5   = spark.read.option("versionAsOf", 5).table("main.ml.events")
df_ts   = spark.read.option("timestampAsOf", "2024-01-01").table("main.ml.events")

# Restore
dt.restoreToVersion(5)
dt.restoreToTimestamp("2024-01-01 00:00:00")
```

### 6.4 Maintenance

```python
# Optimize (file compaction)
dt.optimize().executeCompaction()
dt.optimize().where("date_col = '2024-01-15'").executeCompaction()
dt.optimize().executeZOrderBy("customer_id", "event_ts")

# Vacuum (remove old files)
dt.vacuum(retentionHours=168)                     # default 168h (7 days)

# Update / Delete
dt.update(
    condition = "status = 'PENDING'",
    set       = {"status": "'PROCESSED'", "updated_at": "current_timestamp()"}
)
dt.delete(condition = "event_ts < '2020-01-01'")

# Convert to Delta
DeltaTable.convertToDelta(spark, "parquet.`abfss://...`", "date_col STRING")

# Detail (metadata)
dt.detail().show()                                # single-row DataFrame with table stats
```

### 6.5 Change Data Feed

```python
# Enable on table
spark.sql("ALTER TABLE main.ml.events SET TBLPROPERTIES (delta.enableChangeDataFeed = true)")

# Read changes
changes = spark.read \
    .option("readChangeFeed", "true") \
    .option("startingVersion", 10) \
    .table("main.ml.events")
# _change_type: insert | update_preimage | update_postimage | delete

# Streaming changes
stream = spark.readStream \
    .option("readChangeFeed", "true") \
    .option("startingVersion", "latest") \
    .table("main.ml.events")
```

-----

## 7. Spark Session & Context Utilities

### 7.1 SparkSession

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("FeatureRefresh") \
    .config("spark.sql.shuffle.partitions", "auto") \
    .config("spark.databricks.delta.optimizeWrite.enabled", "true") \
    .config("spark.databricks.delta.autoCompact.enabled", "true") \
    .getOrCreate()

# Useful properties
spark.version
spark.sparkContext.applicationId
spark.sparkContext.getConf().getAll()
spark.catalog.listDatabases()
spark.catalog.listTables("main.ml")
spark.catalog.refreshTable("main.ml.customer_features")
spark.catalog.cacheTable("main.ml.small_lookup")
spark.catalog.uncacheTable("main.ml.small_lookup")
spark.catalog.isCached("main.ml.small_lookup")
spark.catalog.tableExists("main", "ml", "customer_features")

# SQL
spark.sql("USE CATALOG main")
spark.sql("USE SCHEMA ml")
result_df = spark.sql("SELECT * FROM customer_features WHERE customer_id = 42")
```

### 7.2 SparkContext

```python
sc = spark.sparkContext

sc.setJobDescription("FeatureStore write — customer_features v42")
sc.setLocalProperty("spark.scheduler.pool", "high")

# Broadcast variables
lookup_bc = sc.broadcast({"A": 1, "B": 2})
# Use in UDF: lookup_bc.value["A"]

# Accumulators
counter = sc.accumulator(0)
# Use in mapPartitions to count records processed

sc.addFile("dbfs:/tmp/utils.py")                  # add file to all workers
import site; site.addsitedir(sc.addedFiles)

# Config
spark.conf.set("spark.sql.shuffle.partitions", "200")
val = spark.conf.get("spark.sql.shuffle.partitions")
```

### 7.3 Streaming Utilities

```python
# Trigger types
from pyspark.sql.streaming import StreamingQuery

query: StreamingQuery = df.writeStream \
    .format("delta") \
    .outputMode("append")  \
    .option("checkpointLocation", "dbfs:/checkpoints/stream1") \
    .trigger(availableNow=True)      \            # micro-batch until caught up then stop
    .toTable("main.bronze.events")

query.id                                          # UUID
query.name
query.status                                      # dict with isDataAvailable etc.
query.lastProgress                                # metrics dict
query.recentProgress                              # list of recent metric dicts
query.awaitTermination(timeout=300)
query.stop()

spark.streams.active                              # list of active StreamingQuery
spark.streams.awaitAnyTermination()
```

-----

## 8. Unity Catalog Helpers

```python
# Grant / revoke via SQL (most common pattern)
spark.sql("GRANT SELECT ON TABLE main.ml.customer_features TO `ml-engineers`")
spark.sql("GRANT MODIFY ON SCHEMA main.ml TO `ml-engineers`")
spark.sql("REVOKE SELECT ON TABLE main.ml.old_table FROM `data-analysts`")
spark.sql("SHOW GRANTS ON TABLE main.ml.customer_features")

# Tag management
spark.sql("ALTER TABLE main.ml.customer_features SET TAGS ('pii' = 'false', 'domain' = 'customer')")
spark.sql("ALTER TABLE main.ml.customer_features ALTER COLUMN email SET TAGS ('pii' = 'true')")

# Information schema (SQL)
spark.sql("""
    SELECT table_name, table_type, data_source_format, comment
    FROM   main.information_schema.tables
    WHERE  table_schema = 'ml'
""")

spark.sql("""
    SELECT column_name, data_type, comment
    FROM   main.information_schema.columns
    WHERE  table_schema = 'ml'
    AND    table_name   = 'customer_features'
""")

# SDK helpers (WorkspaceClient already shown in §4.5)
# Lineage via REST (no SDK wrapper — use requests):
import requests, json
resp = requests.get(
    f"{workspace_url}/api/2.0/lineage-tracking/table-lineage",
    headers={"Authorization": f"Bearer {token}"},
    params={"table_name": "main.ml.customer_features"},
)
```

-----

## 9. Design Patterns & Best Practices

### 9.1 Secret Management Pattern

```python
# ✅ DO — centralise secret retrieval, never pass raw secrets downstream
class Config:
    def __init__(self, scope: str):
        self._scope = scope
    @property
    def storage_key(self): return dbutils.secrets.get(self._scope, "storage-key")
    @property
    def sql_password(self): return dbutils.secrets.get(self._scope, "sql-pw")

cfg = Config("proj-scope")
# ❌ DON'T
# pw = dbutils.secrets.get("scope", "key"); print(pw)  — leaked in logs
```

### 9.2 MLflow Run Context Manager + Registry Promotion

```python
def train_and_register(params: dict, training_df, label_col: str) -> str:
    """Train, log, register, and promote a model. Returns run_id."""
    mlflow.set_experiment("/Shared/experiments/churn-model")
    with mlflow.start_run(run_name=f"xgb-{params['max_depth']}d") as run:
        mlflow.log_params(params)
        model, metrics = _train(training_df, label_col, params)
        mlflow.log_metrics(metrics)
        fe.log_model(
            model=model, artifact_path="model", flavor=mlflow.sklearn,
            training_set=training_set,
            registered_model_name="main.ml.churn_model",
        )
    # Promote champion if AUC improves
    client = MlflowClient()
    latest = client.get_latest_versions("main.ml.churn_model")[0]
    champion = client.get_model_version_by_alias("main.ml.churn_model", "champion")
    if metrics["auc"] > float(champion.tags.get("auc", 0)):
        client.set_registered_model_alias("main.ml.churn_model", "champion", latest.version)
        client.set_model_version_tag("main.ml.churn_model", latest.version, "auc", str(metrics["auc"]))
    return run.info.run_id
```

### 9.3 Feature Engineering Pipeline Pattern

```python
# Layered feature refresh — raw → curated → feature table
def refresh_feature_table(env: str):
    catalog = "main" if env == "prod" else f"dev_{env}"

    # 1. Read raw data
    raw = spark.table(f"{catalog}.bronze.events")

    # 2. Transform
    features = (
        raw.groupBy("customer_id")
           .agg(
               F.count("*").alias("event_count"),
               F.max("event_ts").alias("last_event_ts"),
               F.sum("value").alias("total_value"),
           )
           .withColumn("days_since_last",
               F.datediff(F.current_date(), F.col("last_event_ts").cast("date")))
    )

    # 3. Write to Feature Store with merge
    fe.write_table(name=f"{catalog}.ml.customer_features",
                   df=features, mode="merge")
    print(f"Refreshed {catalog}.ml.customer_features — {features.count()} rows")

refresh_feature_table(env=dbutils.widgets.get("env"))
```

### 9.4 WorkspaceClient Job Retry Pattern

```python
import time
from databricks.sdk.service.jobs import RunLifeCycleState

def run_job_with_retry(job_id: int, params: dict, retries: int = 3) -> bool:
    w = WorkspaceClient()
    for attempt in range(1, retries + 1):
        run = w.jobs.run_now(job_id=job_id, notebook_params=params)
        finished = w.jobs.wait_get_run_job_terminated_or_skipped(run_id=run.run_id)
        if finished.state.result_state.name == "SUCCESS":
            return True
        print(f"Attempt {attempt} failed: {finished.state.state_message}")
        if attempt < retries:
            time.sleep(30 * attempt)               # exponential back-off
    return False
```

### 9.5 Delta Merge Idempotency Pattern

```python
def idempotent_merge(source_df, target_table: str, key_cols: list[str]):
    """Safe idempotent merge — suitable for re-runnable pipelines."""
    key_condition = " AND ".join(f"t.{k} = s.{k}" for k in key_cols)
    (
        DeltaTable.forName(spark, target_table)
            .alias("t")
            .merge(source_df.alias("s"), key_condition)
            .whenMatchedUpdateAll()
            .whenNotMatchedInsertAll()
            .execute()
    )
```

### 9.6 Schema Evolution Safe Writer

```python
def safe_write(df, table: str, mode: str = "append"):
    """Write with schema evolution and optimised file sizes."""
    df.write \
      .format("delta") \
      .mode(mode) \
      .option("mergeSchema", "true") \
      .option("optimizeWrite", "true") \
      .option("autoCompact", "true") \
      .saveAsTable(table)
```

### 9.7 Notebook Task Value Passing

```python
# Producer notebook
results = process_batch(spark, batch_df)
dbutils.jobs.taskValues.set("records_written", results["count"])
dbutils.jobs.taskValues.set("status", "SUCCESS")
dbutils.notebook.exit(json.dumps(results))

# Consumer notebook
count = dbutils.jobs.taskValues.get(
    taskKey    = "ingest_task",
    key        = "records_written",
    debugValue = 0,                                # used when run outside a Job
)
```

### 9.8 Unit Testing with `dbutils` Mock

```python
# tests/conftest.py
import pytest
from unittest.mock import MagicMock

@pytest.fixture
def mock_dbutils():
    dbutils = MagicMock()
    dbutils.secrets.get.side_effect = lambda scope, key: f"fake-{key}"
    dbutils.widgets.get.side_effect = lambda k: {"env": "test", "date": "2024-01-01"}[k]
    dbutils.jobs.taskValues.get.return_value = 0
    return dbutils
```

### 9.9 Model Scoring Pipeline Pattern

```python
def batch_score(model_alias: str, spine_table: str, output_table: str):
    """End-to-end batch scoring with the Feature Store."""
    spine = spark.table(spine_table)

    preds = fe.score_batch(
        model_uri   = f"models:/main.ml.churn_model@{model_alias}",
        df          = spine,
        result_type = "double",
    )

    (
        preds.withColumn("scored_at", F.current_timestamp())
             .write.format("delta")
             .mode("overwrite").option("overwriteSchema","true")
             .saveAsTable(output_table)
    )
    print(f"Scored {preds.count()} rows → {output_table}")
```

-----

## 10. Method Quick-Reference Tables

### `dbutils` — Complete Method Reference

|Namespace        |Method         |Signature (simplified)                |Returns               |
|-----------------|---------------|--------------------------------------|----------------------|
|`fs`             |`ls`           |`(path)`                              |`list[FileInfo]`      |
|`fs`             |`head`         |`(path, maxBytes=65536)`              |`str`                 |
|`fs`             |`put`          |`(path, content, overwrite=False)`    |`bool`                |
|`fs`             |`cp`           |`(src, dst, recurse=False)`           |`bool`                |
|`fs`             |`mv`           |`(src, dst)`                          |`bool`                |
|`fs`             |`rm`           |`(path, recurse=False)`               |`bool`                |
|`fs`             |`mkdirs`       |`(path)`                              |`bool`                |
|`fs`             |`mount`        |`(source, mount_point, extra_configs)`|`bool`                |
|`fs`             |`unmount`      |`(mount_point)`                       |`bool`                |
|`fs`             |`mounts`       |`()`                                  |`list[MountInfo]`     |
|`fs`             |`refreshMounts`|`()`                                  |`bool`                |
|`secrets`        |`get`          |`(scope, key)`                        |`str`                 |
|`secrets`        |`getBytes`     |`(scope, key)`                        |`bytes`               |
|`secrets`        |`list`         |`(scope)`                             |`list[SecretMetadata]`|
|`secrets`        |`listScopes`   |`()`                                  |`list[SecretScope]`   |
|`widgets`        |`text`         |`(name, defaultValue, label)`         |`void`                |
|`widgets`        |`dropdown`     |`(name, defaultValue, choices, label)`|`void`                |
|`widgets`        |`multiselect`  |`(name, defaultValue, choices, label)`|`void`                |
|`widgets`        |`combobox`     |`(name, defaultValue, choices, label)`|`void`                |
|`widgets`        |`get`          |`(name)`                              |`str`                 |
|`widgets`        |`remove`       |`(name)`                              |`void`                |
|`widgets`        |`removeAll`    |`()`                                  |`void`                |
|`notebook`       |`run`          |`(path, timeout_seconds, arguments)`  |`str`                 |
|`notebook`       |`exit`         |`(value)`                             |`void`                |
|`jobs.taskValues`|`set`          |`(key, value)`                        |`void`                |
|`jobs.taskValues`|`get`          |`(taskKey, key, default, debugValue)` |`any`                 |
|`data`           |`summarize`    |`(df, precise=False)`                 |`void` (HTML output)  |

-----

### MLflow Tracking — Method Reference

|Category  |Method                  |Key Parameters                                             |Notes               |
|----------|------------------------|-----------------------------------------------------------|--------------------|
|Experiment|`set_experiment`        |`name` or `experiment_id`                                  |Creates if absent   |
|Experiment|`get_experiment_by_name`|`name`                                                     |Returns `Experiment`|
|Run       |`start_run`             |`run_name, run_id, nested, tags`                           |Context manager     |
|Run       |`end_run`               |`status`                                                   |Auto-called by CM   |
|Run       |`log_param`             |`key, value`                                               |Single param        |
|Run       |`log_params`            |`params: dict`                                             |Batch params        |
|Run       |`log_metric`            |`key, value, step`                                         |Optional step       |
|Run       |`log_metrics`           |`metrics: dict, step`                                      |Batch metrics       |
|Run       |`set_tag`               |`key, value`                                               |String tag          |
|Run       |`set_tags`              |`tags: dict`                                               |Batch tags          |
|Artifact  |`log_artifact`          |`local_path, artifact_path`                                |Single file         |
|Artifact  |`log_artifacts`         |`local_dir, artifact_path`                                 |Directory           |
|Artifact  |`log_text`              |`text, artifact_file`                                      |In-memory text      |
|Artifact  |`log_dict`              |`dictionary, artifact_file`                                |JSON artifact       |
|Artifact  |`log_figure`            |`figure, artifact_file`                                    |matplotlib/plotly   |
|Artifact  |`log_image`             |`image, artifact_file`                                     |numpy array         |
|Model     |`sklearn.log_model`     |`sk_model, artifact_path, registered_model_name, signature`|Sklearn flavour     |
|Model     |`pyfunc.load_model`     |`model_uri`                                                |`runs:/` `models:/` |
|Model     |`pyfunc.spark_udf`      |`spark, model_uri, result_type`                            |Batch scoring UDF   |
|AutoLog   |`autolog`               |`disable, log_models, log_input_examples`                  |Per-flavour too     |

-----

### `MlflowClient` — Method Reference

|Category  |Method                         |Key Parameters                                        |
|----------|-------------------------------|------------------------------------------------------|
|Experiment|`create_experiment`            |`name, artifact_location, tags`                       |
|Experiment|`get_experiment`               |`experiment_id`                                       |
|Experiment|`get_experiment_by_name`       |`name`                                                |
|Experiment|`rename_experiment`            |`experiment_id, new_name`                             |
|Experiment|`delete_experiment`            |`experiment_id`                                       |
|Experiment|`restore_experiment`           |`experiment_id`                                       |
|Experiment|`search_experiments`           |`filter_string, order_by, max_results`                |
|Experiment|`set_experiment_tag`           |`experiment_id, key, value`                           |
|Run       |`get_run`                      |`run_id`                                              |
|Run       |`create_run`                   |`experiment_id, run_name, tags`                       |
|Run       |`search_runs`                  |`experiment_ids, filter_string, order_by, max_results`|
|Run       |`log_param`                    |`run_id, key, value`                                  |
|Run       |`log_metric`                   |`run_id, key, value, step`                            |
|Run       |`set_tag`                      |`run_id, key, value`                                  |
|Run       |`set_terminated`               |`run_id, status, end_time`                            |
|Run       |`delete_run`                   |`run_id`                                              |
|Run       |`restore_run`                  |`run_id`                                              |
|Artifact  |`list_artifacts`               |`run_id, path`                                        |
|Artifact  |`download_artifacts`           |`run_id, path, dst_path`                              |
|Registry  |`create_registered_model`      |`name, tags, description`                             |
|Registry  |`get_registered_model`         |`name`                                                |
|Registry  |`rename_registered_model`      |`name, new_name`                                      |
|Registry  |`update_registered_model`      |`name, description`                                   |
|Registry  |`delete_registered_model`      |`name`                                                |
|Registry  |`search_registered_models`     |`filter_string, order_by, max_results`                |
|Version   |`create_model_version`         |`name, source, run_id, description, tags`             |
|Version   |`get_model_version`            |`name, version`                                       |
|Version   |`update_model_version`         |`name, version, description`                          |
|Version   |`delete_model_version`         |`name, version`                                       |
|Version   |`set_registered_model_alias`   |`name, alias, version`                                |
|Version   |`delete_registered_model_alias`|`name, alias`                                         |
|Version   |`get_model_version_by_alias`   |`name, alias`                                         |
|Version   |`set_model_version_tag`        |`name, version, key, value`                           |
|Version   |`search_model_versions`        |`filter_string, order_by, max_results`                |

-----

### `WorkspaceClient` — Service Reference

|Service                |Key Methods                                                                                                      |
|-----------------------|-----------------------------------------------------------------------------------------------------------------|
|`w.jobs`               |`list, get, create, update, delete, run_now, cancel_run, list_runs, wait_get_run_job_terminated_or_skipped`      |
|`w.clusters`           |`list, get, create, start, restart, terminate, delete, spark_versions, list_node_types, wait_get_cluster_running`|
|`w.warehouses`         |`list, get, create, start, stop, delete, wait_get_warehouse_running`                                             |
|`w.statement_execution`|`execute_statement, get_statement, cancel_execution`                                                             |
|`w.repos`              |`list, get, create, update, delete`                                                                              |
|`w.workspace`          |`list, get_status, export, import_, delete, mkdirs`                                                              |
|`w.secrets`            |`create_scope, delete_scope, list_scopes, put_secret, delete_secret, list_secrets, put_acl, list_acls`           |
|`w.dbfs`               |`list, mkdirs, move, delete, get_status`                                                                         |
|`w.files`              |`upload, download, get_metadata, delete, list_directory_contents`                                                |
|`w.catalogs`           |`list, get, create, update, delete`                                                                              |
|`w.schemas`            |`list, get, create, update, delete`                                                                              |
|`w.tables`             |`list, get, delete, exists, list_summaries`                                                                      |
|`w.users`              |`list, get, create, update, patch, delete`                                                                       |
|`w.groups`             |`list, get, create, update, patch, delete`                                                                       |
|`w.service_principals` |`list, get, create, update, delete`                                                                              |
|`w.permissions`        |`get, set, update`                                                                                               |
|`w.pipelines`          |`list, get, create, update, start_update, stop, delete`                                                          |
|`w.model_registry`     |`(via MlflowClient — see §3)`                                                                                    |

-----

### `FeatureEngineeringClient` — Method Reference

|Method               |Key Parameters                                                     |Notes                          |
|---------------------|-------------------------------------------------------------------|-------------------------------|
|`create_table`       |`name, primary_keys, timestamp_keys, df, description, tags`        |Creates UC Delta table         |
|`write_table`        |`name, df, mode`                                                   |`mode`: merge | overwrite      |
|`read_table`         |`name`                                                             |Returns Spark DataFrame        |
|`get_table`          |`name`                                                             |Returns `FeatureTable` metadata|
|`list_tables`        |`parent_path`                                                      |`catalog.schema` path          |
|`drop_table`         |`name`                                                             |Deletes UC table               |
|`create_training_set`|`df, feature_lookups, label, exclude_columns`                      |Returns `TrainingSet`          |
|`log_model`          |`model, artifact_path, flavor, training_set, registered_model_name`|MLflow log with lineage        |
|`score_batch`        |`model_uri, df, result_type, env_manager`                          |Auto-fetches features          |

-----

### `DeltaTable` — Method Reference

|Method              |Key Parameters                      |Notes                         |
|--------------------|------------------------------------|------------------------------|
|`forName`           |`spark, tableOrViewName`            |Load by UC name               |
|`forPath`           |`spark, path`                       |Load by ABFSS/DBFS path       |
|`isDeltaTable`      |`spark, identifier`                 |Static, returns bool          |
|`createIfNotExists` |`spark` → fluent                    |Builder pattern               |
|`merge`             |`source, condition`                 |Returns `DeltaMergeBuilder`   |
|`update`            |`condition, set`                    |In-place row update           |
|`delete`            |`condition`                         |In-place row delete           |
|`history`           |`limit`                             |Returns DataFrame             |
|`restoreToVersion`  |`version`                           |In-place restore              |
|`restoreToTimestamp`|`timestamp`                         |In-place restore              |
|`optimize`          |—                                   |Returns `DeltaOptimizeBuilder`|
|`vacuum`            |`retentionHours`                    |Removes old files             |
|`detail`            |—                                   |Returns 1-row DataFrame       |
|`toDF`              |—                                   |Returns Spark DataFrame       |
|`convertToDelta`    |`spark, identifier, partitionSchema`|Static                        |

-----

## Key Imports Cheat Sheet

```python
# Core utilities
from pyspark.sql import SparkSession, functions as F, types as T, Window
from delta.tables import DeltaTable

# MLflow
import mlflow
import mlflow.sklearn, mlflow.pyfunc, mlflow.spark
from mlflow.tracking import MlflowClient
from mlflow.models import infer_signature

# Databricks SDK
from databricks.sdk import WorkspaceClient
from databricks.sdk.service import jobs, clusters, sql, catalog, iam, workspace

# Feature Engineering
from databricks.feature_engineering import (
    FeatureEngineeringClient, FeatureLookup, FeatureFunction
)

# dbutils (non-notebook)
from databricks.sdk.runtime import dbutils
```

-----

## Environment Variables (SDK Auth)

|Variable                  |Purpose                                                 |
|--------------------------|--------------------------------------------------------|
|`DATABRICKS_HOST`         |Workspace URL e.g. `https://adb-xxx.azuredatabricks.net`|
|`DATABRICKS_TOKEN`        |Personal Access Token                                   |
|`DATABRICKS_CLIENT_ID`    |Service Principal client ID (OAuth M2M)                 |
|`DATABRICKS_CLIENT_SECRET`|Service Principal secret (OAuth M2M)                    |
|`DATABRICKS_CLUSTER_ID`   |Default cluster for Databricks Connect                  |
|`MLFLOW_TRACKING_URI`     |`databricks` (auto inside cluster)                      |
|`MLFLOW_REGISTRY_URI`     |`databricks-uc` (Unity Catalog registry)                |

-----

*Generated: Databricks Runtime 14.x · Python 3.10+ · Unity Catalog · MLflow 2.x · Databricks SDK 0.20+*
