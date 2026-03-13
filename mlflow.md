# MLflow on Databricks — Complete Reference Card

> **Version targets:** MLflow 2.x · Databricks Runtime ML 14+ · Python 3.10+

-----

## Table of Contents

1. [Environment Setup](#1-environment-setup)
1. [Core Concepts](#2-core-concepts)
1. [Tracking API — Full Method Reference](#3-tracking-api--full-method-reference)
1. [MLflow Models — Full Method Reference](#4-mlflow-models--full-method-reference)
1. [Model Registry — Full Method Reference](#5-model-registry--full-method-reference)
1. [MLflow Projects](#6-mlflow-projects)
1. [MLflow Pipelines / Recipes](#7-mlflow-pipelines--recipes)
1. [Databricks-Specific Integrations](#8-databricks-specific-integrations)
1. [Autologging Reference](#9-autologging-reference)
1. [Search & Query DSL](#10-search--query-dsl)
1. [Model Serving & Deployment](#11-model-serving--deployment)
1. [Feature Store Integration](#12-feature-store-integration)
1. [Common Patterns & Recipes](#13-common-patterns--recipes)
1. [Quick Comparison Tables](#14-quick-comparison-tables)

-----

## 1. Environment Setup

### Installation

```python
# Databricks Runtime ML — MLflow pre-installed
# Standard Databricks Runtime
%pip install mlflow[databricks]

# Extra flavor support
%pip install mlflow[sklearn,keras,pytorch,spark,pyfunc]
```

### Tracking URI Configuration

```python
import mlflow

# Databricks-hosted tracking (recommended on Databricks)
mlflow.set_tracking_uri("databricks")

# Remote MLflow server
mlflow.set_tracking_uri("http://mlflow-server:5000")

# Local filesystem
mlflow.set_tracking_uri("file:///tmp/mlruns")

# Azure ML / other backends
mlflow.set_tracking_uri("azureml://...")

# Check current URI
print(mlflow.get_tracking_uri())
```

### Authentication (Databricks)

```python
import os
# Via environment variables (CI/CD, notebooks outside Databricks)
os.environ["DATABRICKS_HOST"] = "https://<workspace>.azuredatabricks.net"
os.environ["DATABRICKS_TOKEN"] = "<personal-access-token>"

# Via Databricks CLI profile
mlflow.set_tracking_uri("databricks://my-profile")
```

-----

## 2. Core Concepts

|Concept             |Description                                                  |
|--------------------|-------------------------------------------------------------|
|**Tracking Server** |Backend store for runs, metrics, params, artifacts           |
|**Experiment**      |Named container for related runs                             |
|**Run**             |Single execution of ML code with logged metadata             |
|**Metric**          |Numeric value (optionally time-series) logged per run        |
|**Parameter**       |Key-value config logged once per run                         |
|**Tag**             |Arbitrary string metadata on runs or experiments             |
|**Artifact**        |Files/objects saved alongside a run (models, plots, data)    |
|**Model**           |Standardized artifact with `MLmodel` manifest                |
|**Flavor**          |Framework-specific model representation (sklearn, pytorch, …)|
|**Registered Model**|Named model lineage in the Model Registry                    |
|**Model Version**   |Specific run artifact promoted to registry                   |
|**Stage**           |Registry lifecycle: `None → Staging → Production → Archived` |
|**Alias**           |Mutable named pointer to a model version (MLflow 2.x)        |

-----

## 3. Tracking API — Full Method Reference

### Experiment Management

|Method                  |Signature                                                                     |Description                                     |
|------------------------|------------------------------------------------------------------------------|------------------------------------------------|
|`set_experiment`        |`(name, artifact_location=None, tags=None)`                                   |Set active experiment; creates if missing       |
|`get_experiment`        |`(experiment_id)`                                                             |Return `Experiment` object by ID                |
|`get_experiment_by_name`|`(name)`                                                                      |Return `Experiment` by name                     |
|`create_experiment`     |`(name, artifact_location=None, tags=None)`                                   |Create and return experiment ID                 |
|`delete_experiment`     |`(experiment_id)`                                                             |Soft-delete an experiment                       |
|`restore_experiment`    |`(experiment_id)`                                                             |Restore a deleted experiment                    |
|`rename_experiment`     |`(experiment_id, new_name)`                                                   |Rename an experiment                            |
|`search_experiments`    |`(filter_string=None, max_results=1000, order_by=None, view_type=ACTIVE_ONLY)`|Search experiments, returns list of `Experiment`|

```python
import mlflow

# Basic experiment setup
mlflow.set_experiment("/Users/alice@company.com/fraud-detection")

# With tags and custom artifact location
exp_id = mlflow.create_experiment(
    name="fraud-detection-v2",
    artifact_location="dbfs:/experiments/fraud-detection-v2",
    tags={"team": "risk", "project": "fraud", "env": "dev"}
)

# Search experiments
exps = mlflow.search_experiments(
    filter_string="name LIKE 'fraud%'",
    order_by=["creation_time DESC"]
)
```

### Run Lifecycle

|Method           |Signature                                                                                                               |Description                                               |
|-----------------|------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
|`start_run`      |`(run_id=None, experiment_id=None, run_name=None, nested=False, tags=None, description=None)`                           |Start or resume a run; returns `ActiveRun` context manager|
|`end_run`        |`(status='FINISHED')`                                                                                                   |End the current active run                                |
|`active_run`     |`()`                                                                                                                    |Return currently active `Run` or `None`                   |
|`last_active_run`|`()`                                                                                                                    |Return last active run                                    |
|`get_run`        |`(run_id)`                                                                                                              |Fetch a `Run` by ID                                       |
|`search_runs`    |`(experiment_ids, filter_string='', run_view_type=ACTIVE_ONLY, max_results=1000, order_by=None, output_format='pandas')`|Search runs, returns DataFrame or list                    |
|`delete_run`     |`(run_id)`                                                                                                              |Soft-delete a run                                         |

```python
# Context manager (recommended)
with mlflow.start_run(run_name="xgb-baseline", tags={"version": "1"}) as run:
    run_id = run.info.run_id
    # ... training code

# Nested runs (hyperparameter search)
with mlflow.start_run(run_name="parent-sweep") as parent:
    for lr in [0.01, 0.1]:
        with mlflow.start_run(run_name=f"child-lr-{lr}", nested=True):
            mlflow.log_param("lr", lr)

# Resume an existing run
with mlflow.start_run(run_id="abc123def456"):
    mlflow.log_metric("final_auc", 0.95)
```

### Logging — Parameters

|Method      |Signature       |Description                    |
|------------|----------------|-------------------------------|
|`log_param` |`(key, value)`  |Log a single parameter         |
|`log_params`|`(params: dict)`|Log multiple parameters at once|
|`set_tag`   |`(key, value)`  |Set a single tag               |
|`set_tags`  |`(tags: dict)`  |Set multiple tags              |

```python
mlflow.log_param("n_estimators", 100)
mlflow.log_params({
    "learning_rate": 0.05,
    "max_depth": 6,
    "subsample": 0.8,
    "colsample_bytree": 0.8,
})
mlflow.set_tag("mlflow.note.content", "Baseline XGBoost run")
mlflow.set_tags({"framework": "xgboost", "data_version": "v3"})
```

### Logging — Metrics

|Method       |Signature                                |Description                 |
|-------------|-----------------------------------------|----------------------------|
|`log_metric` |`(key, value, step=None, timestamp=None)`|Log a single metric         |
|`log_metrics`|`(metrics: dict, step=None)`             |Log multiple metrics at once|

```python
mlflow.log_metric("accuracy", 0.92)
mlflow.log_metrics({"precision": 0.89, "recall": 0.94, "f1": 0.91})

# Time-series metric (training loss curve)
for epoch, loss in enumerate(training_losses):
    mlflow.log_metric("train_loss", loss, step=epoch)
    mlflow.log_metric("val_loss",   val_losses[epoch], step=epoch)
```

### Logging — Artifacts

|Method            |Signature                         |Description                                       |
|------------------|----------------------------------|--------------------------------------------------|
|`log_artifact`    |`(local_path, artifact_path=None)`|Upload a local file                               |
|`log_artifacts`   |`(local_dir, artifact_path=None)` |Upload all files in a directory                   |
|`log_text`        |`(text, artifact_file)`           |Log a string as a text file                       |
|`log_dict`        |`(dictionary, artifact_file)`     |Log a dict as JSON/YAML                           |
|`log_image`       |`(image, artifact_file)`          |Log a PIL Image or ndarray                        |
|`log_figure`      |`(figure, artifact_file)`         |Log a matplotlib/plotly figure                    |
|`log_table`       |`(data, artifact_file)`           |Log a DataFrame or dict as JSON table             |
|`get_artifact_uri`|`(artifact_path=None)`            |Get URI of artifact directory or specific artifact|

```python
# Upload files
mlflow.log_artifact("confusion_matrix.png", "plots")
mlflow.log_artifacts("./output_dir/",       "outputs")

# Inline logging
mlflow.log_text("col1,col2\n1,2\n3,4", "sample_data.csv")
mlflow.log_dict({"threshold": 0.5, "classes": ["fraud","legit"]}, "config.json")

# Figures
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.plot(history["loss"])
mlflow.log_figure(fig, "loss_curve.png")
plt.close(fig)

# Tables (visible in Databricks UI)
import pandas as pd
mlflow.log_table(data={"pred": [0,1,1], "label": [0,1,0]}, artifact_file="predictions.json")
```

### Artifact Access

|Method                           |Signature                      |Description                    |
|---------------------------------|-------------------------------|-------------------------------|
|`MlflowClient.list_artifacts`    |`(run_id, path=None)`          |List artifact files            |
|`MlflowClient.download_artifacts`|`(run_id, path, dst_path=None)`|Download artifact to local path|

```python
from mlflow.tracking import MlflowClient
client = MlflowClient()

artifacts = client.list_artifacts(run_id, "plots")
local_path = client.download_artifacts(run_id, "model/model.pkl", "/tmp/")
```

-----

## 4. MLflow Models — Full Method Reference

### Saving & Loading — Flavor Methods

Every flavor exposes `log_model`, `save_model`, and `load_model`.

|Flavor               |Import                      |Notes                        |
|---------------------|----------------------------|-----------------------------|
|`mlflow.sklearn`     |`import mlflow.sklearn`     |scikit-learn estimators      |
|`mlflow.xgboost`     |`import mlflow.xgboost`     |XGBoost Booster/XGBClassifier|
|`mlflow.lightgbm`    |`import mlflow.lightgbm`    |LightGBM models              |
|`mlflow.catboost`    |`import mlflow.catboost`    |CatBoost models              |
|`mlflow.pytorch`     |`import mlflow.pytorch`     |PyTorch nn.Module            |
|`mlflow.tensorflow`  |`import mlflow.tensorflow`  |TF2 SavedModel / Keras       |
|`mlflow.keras`       |`import mlflow.keras`       |Keras (standalone)           |
|`mlflow.transformers`|`import mlflow.transformers`|HuggingFace Transformers     |
|`mlflow.langchain`   |`import mlflow.langchain`   |LangChain chains/agents      |
|`mlflow.openai`      |`import mlflow.openai`      |OpenAI completions           |
|`mlflow.pyfunc`      |`import mlflow.pyfunc`      |Custom Python model          |
|`mlflow.spark`       |`import mlflow.spark`       |Spark MLlib PipelineModel    |
|`mlflow.statsmodels` |`import mlflow.statsmodels` |Statsmodels results          |
|`mlflow.prophet`     |`import mlflow.prophet`     |Prophet forecasting          |
|`mlflow.pmdarima`    |`import mlflow.pmdarima`    |pmdarima ARIMA models        |

### Common Flavor Method Signatures

```python
# log_model (all flavors share this structure)
mlflow.sklearn.log_model(
    sk_model,                    # model object
    artifact_path,               # path inside run artifacts
    conda_env=None,              # conda.yaml dict or path
    code_paths=None,             # list of local code files to include
    registered_model_name=None,  # auto-register to Model Registry
    signature=None,              # ModelSignature
    input_example=None,          # sample input for schema inference
    await_registration_for=300,  # seconds to wait for registration
    pip_requirements=None,       # list of pip packages
    extra_pip_requirements=None,
    metadata=None,               # dict of custom metadata
)

# save_model — save to local directory
mlflow.sklearn.save_model(model, path="./saved_model")

# load_model — load by URI
model = mlflow.sklearn.load_model("runs:/<run_id>/model")
model = mlflow.sklearn.load_model("models:/MyModel/Production")
model = mlflow.sklearn.load_model("models:/MyModel@champion")
```

### Model Signature & Input Examples

```python
from mlflow.models import ModelSignature, infer_signature
from mlflow.types.schema import Schema, ColSpec, TensorSpec

# Auto-infer signature
signature = infer_signature(X_train, model.predict(X_train))

# Manual signature (tabular)
input_schema  = Schema([ColSpec("double","age"), ColSpec("string","category")])
output_schema = Schema([ColSpec("double","prediction")])
signature = ModelSignature(inputs=input_schema, outputs=output_schema)

# Tensor signature (images)
from mlflow.types.schema import TensorSpec
import numpy as np
signature = ModelSignature(
    inputs=Schema([TensorSpec(np.dtype("float32"), (-1, 224, 224, 3), "images")]),
    outputs=Schema([TensorSpec(np.dtype("float32"), (-1, 1000), "logits")])
)

# Log with signature and example
mlflow.sklearn.log_model(
    model, "model",
    signature=signature,
    input_example=X_train[:5]
)
```

### PyFunc — Custom Python Models

```python
import mlflow.pyfunc

# Define custom model class
class PreprocessPredict(mlflow.pyfunc.PythonModel):
    def load_context(self, context):
        import pickle
        with open(context.artifacts["scaler"], "rb") as f:
            self.scaler = pickle.load(f)
        self.model = mlflow.sklearn.load_model(context.artifacts["model"])

    def predict(self, context, model_input, params=None):
        scaled = self.scaler.transform(model_input)
        return self.model.predict(scaled)

# Log
mlflow.pyfunc.log_model(
    artifact_path="pipeline",
    python_model=PreprocessPredict(),
    artifacts={
        "scaler": "./scaler.pkl",
        "model": f"runs:/{run_id}/model"
    },
    conda_env={"dependencies": ["scikit-learn", "numpy"]},
    signature=signature,
)

# Load and predict
pyfunc_model = mlflow.pyfunc.load_model(f"runs:/{run_id}/pipeline")
preds = pyfunc_model.predict(X_test)
```

-----

## 5. Model Registry — Full Method Reference

### MlflowClient Registry Methods

|Method                          |Signature                                                 |Description                           |
|--------------------------------|----------------------------------------------------------|--------------------------------------|
|`create_registered_model`       |`(name, tags=None, description=None)`                     |Create new registered model           |
|`get_registered_model`          |`(name)`                                                  |Get model metadata                    |
|`list_registered_models`        |`(max_results=100)`                                       |List all models                       |
|`search_registered_models`      |`(filter_string='', max_results=100, order_by=None)`      |Search with filters                   |
|`update_registered_model`       |`(name, description=None)`                                |Update description                    |
|`rename_registered_model`       |`(name, new_name)`                                        |Rename a model                        |
|`delete_registered_model`       |`(name)`                                                  |Delete registered model               |
|`create_model_version`          |`(name, source, run_id=None, tags=None, description=None)`|Register a new version                |
|`get_model_version`             |`(name, version)`                                         |Get version metadata                  |
|`list_model_versions`           |`(*args)`                                                 |List versions (deprecated, use search)|
|`search_model_versions`         |`(filter_string='', max_results=200, order_by=None)`      |Search versions                       |
|`update_model_version`          |`(name, version, description=None)`                       |Update version description            |
|`transition_model_version_stage`|`(name, version, stage, archive_existing_versions=False)` |Change lifecycle stage                |
|`delete_model_version`          |`(name, version)`                                         |Delete a version                      |
|`get_model_version_download_uri`|`(name, version)`                                         |Get artifact download URI             |
|`set_registered_model_alias`    |`(name, alias, version)`                                  |Set a named alias on a version        |
|`get_model_version_by_alias`    |`(name, alias)`                                           |Get version by alias                  |
|`delete_registered_model_alias` |`(name, alias)`                                           |Remove an alias                       |
|`set_registered_model_tag`      |`(name, key, value)`                                      |Tag a registered model                |
|`set_model_version_tag`         |`(name, version, key, value)`                             |Tag a model version                   |
|`delete_registered_model_tag`   |`(name, key)`                                             |Remove a model tag                    |
|`get_latest_versions`           |`(name, stages=None)`                                     |Get latest version per stage          |

```python
from mlflow.tracking import MlflowClient
client = MlflowClient()

# Register a model from a run
mv = client.create_model_version(
    name="FraudDetector",
    source=f"runs:/{run_id}/model",
    run_id=run_id,
    description="XGBoost trained on fraud dataset v3"
)
print(f"Version: {mv.version}")

# Promote to staging
client.transition_model_version_stage(
    name="FraudDetector",
    version=mv.version,
    stage="Staging",
    archive_existing_versions=False
)

# Promote to production (archive old production)
client.transition_model_version_stage(
    name="FraudDetector",
    version=mv.version,
    stage="Production",
    archive_existing_versions=True    # archives previous Production versions
)

# Modern: use aliases instead of stages
client.set_registered_model_alias("FraudDetector", "champion", mv.version)
champion = client.get_model_version_by_alias("FraudDetector", "champion")

# Load champion model
model = mlflow.pyfunc.load_model("models:/FraudDetector@champion")

# Search versions
versions = client.search_model_versions(
    filter_string="name='FraudDetector' AND tags.env='production'"
)
```

-----

## 6. MLflow Projects

```yaml
# MLproject file
name: fraud-training

conda_env: conda.yaml
# or: docker_env: {image: "mlflow-docker-example"}

entry_points:
  main:
    parameters:
      n_estimators: {type: int, default: 100}
      learning_rate: {type: float, default: 0.05}
      data_path: {type: str}
    command: "python train.py --n-estimators {n_estimators} --lr {learning_rate} --data {data_path}"

  evaluate:
    parameters:
      model_uri: str
    command: "python evaluate.py --model-uri {model_uri}"
```

```python
# Run a project
import mlflow

# Local
mlflow.projects.run(
    uri=".",
    entry_point="main",
    parameters={"n_estimators": 200, "learning_rate": 0.01},
    experiment_name="fraud-experiment",
)

# From Git
mlflow.projects.run(
    uri="https://github.com/org/repo.git#subdir",
    version="main",
    parameters={"alpha": 0.5},
)

# On Databricks
mlflow.projects.run(
    uri=".",
    backend="databricks",
    backend_config={
        "spark_version": "14.3.x-ml-scala2.12",
        "node_type_id": "i3.xlarge",
        "num_workers": 4,
    }
)
```

-----

## 7. MLflow Pipelines / Recipes

```python
# Regression Recipe (MLflow 2.x)
# Directory structure:
# recipe.yaml, steps/ingest.py, steps/split.py, steps/transform.py,
# steps/train.py, steps/evaluate.py, steps/register.py

from mlflow.recipes import Recipe

r = Recipe(profile="local")
r.run()                          # run all steps
r.run("train")                   # run a specific step
r.inspect("train")               # inspect step output
r.get_artifact("model")          # retrieve artifact
r.get_artifact("transformed_training_data")
r.clean()                        # clean cached step outputs
```

-----

## 8. Databricks-Specific Integrations

### Unity Catalog Model Registry (UC)

```python
import mlflow
mlflow.set_registry_uri("databricks-uc")

# UC model names use 3-level namespace
mlflow.register_model(
    model_uri=f"runs:/{run_id}/model",
    name="catalog.schema.model_name"
)

# Load from UC
model = mlflow.pyfunc.load_model("models:/catalog.schema.model_name/1")
model = mlflow.pyfunc.load_model("models:/catalog.schema.model_name@champion")
```

### Databricks Experiments (Workspace UI)

```python
# Notebook-relative experiment path
mlflow.set_experiment("/Users/alice@co.com/my-notebook-experiment")

# Workspace-level shared experiment
mlflow.set_experiment("/Shared/team-experiments/my-exp")

# Get experiment from current notebook (Databricks auto-creates one)
experiment = mlflow.get_experiment_by_name(dbutils.notebook.entry_point.getDbutils().notebook().getContext().notebookPath().get())
```

### DBFS Artifact Storage

```python
# Log to DBFS (default on Databricks)
with mlflow.start_run():
    mlflow.log_artifact("./model.pkl")
    artifact_uri = mlflow.get_artifact_uri()
    # -> dbfs:/databricks/mlflow-tracking/<exp_id>/<run_id>/artifacts/

# Custom DBFS path
mlflow.create_experiment(
    "my-exp",
    artifact_location="dbfs:/mnt/mlflow-artifacts/my-exp"
)
```

### Delta Table Integration

```python
import mlflow.data
from mlflow.data.delta_dataset_source import DeltaDatasetSource

# Log dataset lineage
dataset = mlflow.data.from_spark(
    spark_df,
    table_name="catalog.schema.features",
    version=5
)

with mlflow.start_run():
    mlflow.log_input(dataset, context="training")

# Log Delta table as input dataset
delta_source = DeltaDatasetSource(
    delta_table_name="catalog.schema.train_data",
    delta_table_version=3
)
mlflow.log_input(mlflow.data.load_delta(delta_source), "training")
```

### MLflow with Spark ML

```python
import mlflow.spark
from pyspark.ml import Pipeline
from pyspark.ml.classification import RandomForestClassifier

pipeline = Pipeline(stages=[indexer, assembler, RandomForestClassifier()])
model = pipeline.fit(train_df)

with mlflow.start_run():
    mlflow.spark.log_model(
        model,
        "spark-model",
        registered_model_name="SparkFraudModel",
        dfs_tmpdir="dbfs:/tmp/spark-mlflow"
    )

# Load as Spark model
spark_model = mlflow.spark.load_model(f"runs:/{run_id}/spark-model")
predictions = spark_model.transform(test_df)

# Load as pyfunc (for single-node inference)
pyfunc_model = mlflow.pyfunc.load_model(f"runs:/{run_id}/spark-model")
```

### MLflow with Databricks AutoML

```python
from databricks import automl

# AutoML automatically logs to MLflow
summary = automl.classify(
    dataset=train_df,
    target_col="label",
    primary_metric="f1",
    timeout_minutes=30,
    experiment_dir="/Users/alice/automl-fraud"
)

# Access best run
best_run = summary.best_trial
print(best_run.mlflow_run_id)
best_model = mlflow.pyfunc.load_model(f"runs:/{best_run.mlflow_run_id}/model")
```

-----

## 9. Autologging Reference

### Enable / Disable Autologging

|Method                        |Description                 |
|------------------------------|----------------------------|
|`mlflow.autolog()`            |Enable all supported flavors|
|`mlflow.sklearn.autolog()`    |sklearn-specific autolog    |
|`mlflow.xgboost.autolog()`    |XGBoost autolog             |
|`mlflow.lightgbm.autolog()`   |LightGBM autolog            |
|`mlflow.pytorch.autolog()`    |PyTorch Lightning autolog   |
|`mlflow.tensorflow.autolog()` |TensorFlow/Keras autolog    |
|`mlflow.spark.autolog()`      |Spark ML autolog            |
|`mlflow.disable_autologging()`|Disable all autologging     |

### Autolog Parameters

```python
mlflow.autolog(
    log_input_examples=False,   # Log sample inputs
    log_model_signatures=True,  # Infer and log signatures
    log_models=True,            # Save trained models
    log_datasets=True,          # Log dataset info
    disable=False,              # Master switch
    exclusive=False,            # Disable other autologging
    disable_for_unsupported_versions=False,
    silent=False,               # Suppress warnings
    extra_tags={"team": "ml"},  # Add tags to auto-created runs
)

# Sklearn-specific
mlflow.sklearn.autolog(
    log_input_examples=True,
    log_model_signatures=True,
    log_models=True,
    log_post_training_metrics=True,   # metrics on test data
    serialization_format="cloudpickle",
    registered_model_name=None,
    pos_label=None,
    max_tuning_runs=5,                # log N best CV runs
)

# TensorFlow/Keras-specific
mlflow.tensorflow.autolog(
    every_n_iter=1,       # log metrics every N epochs
    log_models=True,
    log_input_examples=False,
    log_model_signatures=True,
    saved_model_kwargs=None,
    keras_model_kwargs=None,
)
```

### What Gets Autologged

|Framework        |Params          |Metrics            |Model      |Extra                                |
|-----------------|----------------|-------------------|-----------|-------------------------------------|
|sklearn          |Estimator params|CV scores, fit time|Pickle     |Feature importances, confusion matrix|
|XGBoost          |All params      |Training curves    |Booster    |Feature importances                  |
|LightGBM         |All params      |Training/val curves|LGBM format|Feature importances                  |
|PyTorch Lightning|Trainer params  |Epoch metrics      |TorchScript|Checkpoints                          |
|Keras            |Compile params  |Per-epoch metrics  |SavedModel |Model summary                        |
|Spark ML         |Pipeline params |—                  |Spark model|—                                    |

-----

## 10. Search & Query DSL

### Search Runs

```python
runs = mlflow.search_runs(
    experiment_ids=["1", "2"],
    filter_string="metrics.accuracy > 0.9 AND params.model_type = 'xgboost'",
    run_view_type=mlflow.entities.ViewType.ACTIVE_ONLY,
    max_results=50,
    order_by=["metrics.accuracy DESC"],
    output_format="pandas"   # or "list"
)
# Returns a pandas DataFrame with columns: run_id, status, params.*, metrics.*, tags.*
```

### Filter DSL Reference

```
# Parameters
params.learning_rate < 0.1
params.model_type = 'xgboost'
params.model_type != 'linear'

# Metrics
metrics.accuracy > 0.9
metrics.loss <= 0.05
metrics.f1 BETWEEN 0.8 AND 0.95

# Tags
tags.env = 'production'
tags.mlflow.runName LIKE 'xgb%'
tags.team IN ('risk', 'fraud')

# Run attributes
attributes.status = 'FINISHED'
attributes.run_name = 'baseline'
attributes.start_time > 1700000000000   # epoch ms

# Combine
metrics.auc > 0.85 AND params.n_estimators = '100' AND tags.env = 'prod'
```

### Search Experiments

```python
exps = mlflow.search_experiments(
    filter_string="name LIKE 'fraud%' AND tags.team = 'risk'",
    order_by=["last_update_time DESC"],
    max_results=20,
    view_type=mlflow.entities.ViewType.ACTIVE_ONLY
)
```

### Search Model Versions

```python
from mlflow.tracking import MlflowClient
client = MlflowClient()

versions = client.search_model_versions(
    filter_string="name='FraudDetector' AND version_number > 5",
    max_results=20,
    order_by=["version_number DESC"]
)
```

-----

## 11. Model Serving & Deployment

### Deploy to Databricks Model Serving

```python
# Via Databricks REST API / SDK
# (Also available via UI: Models > Serving)
import requests

response = requests.post(
    f"{DATABRICKS_HOST}/api/2.0/serving-endpoints",
    headers={"Authorization": f"Bearer {TOKEN}"},
    json={
        "name": "fraud-detector-endpoint",
        "config": {
            "served_models": [{
                "model_name": "FraudDetector",
                "model_version": "3",
                "workload_size": "Small",   # Small, Medium, Large
                "scale_to_zero_enabled": True,
            }]
        }
    }
)
```

### Batch Inference with Spark

```python
import mlflow.pyfunc

# Load model as Spark UDF
predict_udf = mlflow.pyfunc.spark_udf(
    spark,
    model_uri="models:/FraudDetector@champion",
    result_type="double"    # or ArrayType(DoubleType())
)

predictions = test_df.withColumn("prediction", predict_udf(*feature_cols))
```

### Local / Batch Deployment

```python
# Deploy to local REST server (for testing)
# mlflow models serve -m models:/MyModel/Production -p 5001

# Evaluate deployed model
import mlflow.models

result = mlflow.models.evaluate(
    model=f"runs:/{run_id}/model",
    data=test_df,
    targets="label",
    model_type="classifier",    # or "regressor"
    evaluators="default",
    evaluator_config={
        "log_model_explainability": True,
        "explainability_nsamples": 100,
    }
)
print(result.metrics)
```

### MLflow Models Evaluate

```python
from mlflow.models import evaluate

# Full evaluation API
eval_result = evaluate(
    model=f"runs:/{run_id}/model",
    data=X_test,           # DataFrame, Dataset, or URI
    targets=y_test,        # target column name or array
    model_type="classifier",
    dataset_name="holdout-set",
    feature_names=feature_cols,
    evaluators=["default"],
    extra_metrics=[
        mlflow.models.make_metric(
            eval_fn=lambda preds, targets, metrics, **kwargs: pd.Series(preds > 0.5).mean(),
            name="approval_rate",
            greater_is_better=False,
        )
    ],
    validation_thresholds={
        "accuracy":      mlflow.models.MetricThreshold(threshold=0.85, greater_is_better=True),
        "precision":     mlflow.models.MetricThreshold(threshold=0.80, greater_is_better=True),
    },
)
print(eval_result.metrics)
print(eval_result.artifacts)
```

-----

## 12. Feature Store Integration

### Databricks Feature Store

```python
from databricks.feature_store import FeatureStoreClient

fs = FeatureStoreClient()

# Write features
fs.create_table(
    name="catalog.schema.customer_features",
    primary_keys=["customer_id"],
    df=features_df,
    schema=features_df.schema,
    description="Customer aggregated features"
)
fs.write_table("catalog.schema.customer_features", features_df, mode="merge")

# Training with Feature Store (logs lineage to MLflow)
from databricks.feature_store import FeatureLookup

feature_lookups = [
    FeatureLookup(
        table_name="catalog.schema.customer_features",
        feature_names=["avg_txn_30d", "num_txns_7d"],
        lookup_key="customer_id",
    )
]

training_set = fs.create_training_set(
    df=labels_df,
    feature_lookups=feature_lookups,
    label="is_fraud",
    exclude_columns=["customer_id"]
)
training_df = training_set.load_df()

# Log model with Feature Store metadata
with mlflow.start_run():
    model.fit(training_df.drop("is_fraud"), training_df["is_fraud"])
    fs.log_model(
        model=model,
        artifact_path="model",
        flavor=mlflow.sklearn,
        training_set=training_set,
        registered_model_name="FraudDetectorFS"
    )

# Batch scoring with automatic feature lookup
predictions = fs.score_batch(
    model_uri="models:/FraudDetectorFS@champion",
    df=inference_df     # only needs primary keys
)
```

-----

## 13. Common Patterns & Recipes

### Hyperparameter Tuning with Hyperopt

```python
from hyperopt import fmin, tpe, hp, STATUS_OK, Trials
import mlflow

search_space = {
    "n_estimators":  hp.quniform("n_estimators", 50, 500, 50),
    "max_depth":     hp.quniform("max_depth", 3, 10, 1),
    "learning_rate": hp.loguniform("learning_rate", -5, 0),
    "subsample":     hp.uniform("subsample", 0.5, 1.0),
}

def objective(params):
    with mlflow.start_run(nested=True):
        params["n_estimators"] = int(params["n_estimators"])
        params["max_depth"]    = int(params["max_depth"])
        mlflow.log_params(params)

        model = XGBClassifier(**params)
        model.fit(X_train, y_train)
        auc = roc_auc_score(y_val, model.predict_proba(X_val)[:,1])
        mlflow.log_metric("val_auc", auc)
    return {"loss": -auc, "status": STATUS_OK}

with mlflow.start_run(run_name="hyperopt-sweep"):
    mlflow.set_tag("search_strategy", "TPE")
    best = fmin(objective, search_space, algo=tpe.suggest, max_evals=50, trials=Trials())
```

### Cross-Validation with MLflow

```python
from sklearn.model_selection import cross_val_score

with mlflow.start_run(run_name="cv-evaluation"):
    mlflow.log_params({"model": "RandomForest", "cv_folds": 5})

    scores = cross_val_score(model, X, y, cv=5, scoring="roc_auc")

    for fold, score in enumerate(scores):
        mlflow.log_metric("fold_auc", score, step=fold)

    mlflow.log_metrics({
        "mean_auc": scores.mean(),
        "std_auc":  scores.std(),
        "min_auc":  scores.min(),
    })
```

### CI/CD Model Promotion Pattern

```python
from mlflow.tracking import MlflowClient

def promote_if_better(candidate_run_id, model_name, metric="accuracy", threshold=0.90):
    client = MlflowClient()
    candidate_run = client.get_run(candidate_run_id)
    candidate_metric = candidate_run.data.metrics[metric]

    if candidate_metric < threshold:
        print(f"Candidate ({candidate_metric:.3f}) below threshold {threshold}")
        return False

    # Compare against current production
    prod_versions = client.get_latest_versions(model_name, stages=["Production"])
    if prod_versions:
        prod_run = client.get_run(prod_versions[0].run_id)
        prod_metric = prod_run.data.metrics.get(metric, 0)
        if candidate_metric <= prod_metric:
            print(f"Candidate ({candidate_metric:.3f}) not better than production ({prod_metric:.3f})")
            return False

    # Register and promote
    mv = client.create_model_version(
        name=model_name,
        source=f"runs:/{candidate_run_id}/model",
        run_id=candidate_run_id
    )
    client.transition_model_version_stage(
        name=model_name, version=mv.version,
        stage="Production", archive_existing_versions=True
    )
    client.set_registered_model_alias(model_name, "champion", mv.version)
    print(f"Promoted version {mv.version} to Production with {metric}={candidate_metric:.3f}")
    return True
```

### Comparing Runs & Plotting

```python
import mlflow
import pandas as pd
import matplotlib.pyplot as plt

runs = mlflow.search_runs(
    experiment_names=["fraud-detection"],
    filter_string="status = 'FINISHED'",
    order_by=["metrics.val_auc DESC"]
)

# Plot metric vs parameter
fig, ax = plt.subplots(figsize=(8, 5))
ax.scatter(
    runs["params.learning_rate"].astype(float),
    runs["metrics.val_auc"],
    alpha=0.6
)
ax.set_xlabel("Learning Rate"); ax.set_ylabel("Val AUC")
ax.set_xscale("log")
ax.set_title("Val AUC vs Learning Rate")
plt.tight_layout()
# Save to best run
with mlflow.start_run(run_id=runs.iloc[0]["run_id"]):
    mlflow.log_figure(fig, "lr_vs_auc.png")
```

### LLM / GenAI Tracing (MLflow 2.14+)

```python
import mlflow

mlflow.set_experiment("llm-eval")

# Trace a chain automatically
with mlflow.start_span(name="my-chain") as span:
    span.set_inputs({"query": "What is the capital of France?"})
    response = my_llm_chain.invoke({"query": "What is the capital of France?"})
    span.set_outputs({"response": response})

# Log evaluation results
with mlflow.start_run():
    mlflow.log_table(
        data={"input": inputs, "output": outputs, "score": scores},
        artifact_file="eval_results.json"
    )
```

-----

## 14. Quick Comparison Tables

### Logging Functions: When to Use

|Task                          |Use                                 |
|------------------------------|------------------------------------|
|Single number during training |`log_metric(key, value, step=epoch)`|
|All training metrics at once  |`log_metrics({...}, step=epoch)`    |
|Model hyperparameters         |`log_params({...})` (once per run)  |
|Config files, plots, CSVs     |`log_artifact(path)`                |
|Dictionary config             |`log_dict(d, "config.json")`        |
|Raw string output             |`log_text(s, "output.txt")`         |
|DataFrame result              |`log_table(df, "results.json")`     |
|matplotlib Figure             |`log_figure(fig, "plot.png")`       |
|All of the above automatically|`mlflow.autolog()`                  |

### Model URI Formats

|URI Format                |Example                     |Description            |
|--------------------------|----------------------------|-----------------------|
|`runs:/<run_id>/<path>`   |`runs:/abc123/model`        |By run ID              |
|`models:/<name>/<version>`|`models:/MyModel/3`         |Specific version number|
|`models:/<name>/<stage>`  |`models:/MyModel/Production`|By stage (legacy)      |
|`models:/<name>@<alias>`  |`models:/MyModel@champion`  |By alias (recommended) |
|`dbfs:/<path>`            |`dbfs:/models/myfolder`     |DBFS path (Databricks) |
|`s3://<bucket>/<path>`    |`s3://bucket/models/v1`     |S3                     |
|`azureml://...`           |`azureml://...`             |Azure ML               |

### Stage Transition Summary

```
None → Staging    (new model, under evaluation)
Staging → Production   (approved, serving production traffic)
Production → Archived  (replaced by newer model)
Any → Archived         (deprecated)
```

### Flavor Load Compatibility

|`log_model` flavor |`load_model` flavors supported      |
|-------------------|------------------------------------|
|`mlflow.sklearn`   |`mlflow.sklearn`, `mlflow.pyfunc`   |
|`mlflow.xgboost`   |`mlflow.xgboost`, `mlflow.pyfunc`   |
|`mlflow.pytorch`   |`mlflow.pytorch`, `mlflow.pyfunc`   |
|`mlflow.tensorflow`|`mlflow.tensorflow`, `mlflow.pyfunc`|
|`mlflow.spark`     |`mlflow.spark`, `mlflow.pyfunc`     |
|`mlflow.pyfunc`    |`mlflow.pyfunc` only                |

### Autolog vs Manual: Trade-offs

|Approach                 |Pros                                                             |Cons                               |
|-------------------------|-----------------------------------------------------------------|-----------------------------------|
|`mlflow.autolog()`       |Zero code, framework-aware, catches everything                   |Less control, may log excess params|
|`mlflow.log_*()`         |Full control, clean runs                                         |More verbose code                  |
|Hybrid (autolog + manual)|Best of both — autolog captures model, manual logs custom metrics|Need to ensure no duplicate keys   |

-----

## Environment & Version Reference

```python
import mlflow, databricks

print(mlflow.__version__)        # e.g. 2.14.1
print(mlflow.get_tracking_uri())
print(mlflow.get_registry_uri())

# Check active experiment
exp = mlflow.get_experiment_by_name(mlflow.get_experiment(mlflow.active_run().info.experiment_id).name)

# Databricks context
from dbruntime.databricks_repl_context import get_context
ctx = get_context()
print(ctx.notebookId, ctx.clusterId, ctx.workspaceId)
```

-----

*Generated: 2026 · MLflow 2.x · Databricks Runtime ML 14+*