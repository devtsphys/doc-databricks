# Databricks Asset Bundles (DAB) — Comprehensive Reference Card

> **Last updated:** March 2026 · CLI v0.218.0+ required · Databricks Runtime 11.3 LTS+

-----

## Table of Contents

1. [Overview & Concepts](#1-overview--concepts)
1. [Project Structure](#2-project-structure)
1. [CLI Commands](#3-cli-commands-reference)
1. [Configuration Schema (`databricks.yml`)](#4-configuration-schema-databricksyml)
1. [Top-Level Mappings Reference](#5-top-level-mappings-reference)
1. [Resources Reference](#6-resources-reference)
1. [Variables & Substitutions](#7-variables--substitutions)
1. [Targets & Deployment Modes](#8-targets--deployment-modes)
1. [Permissions](#9-permissions)
1. [Artifacts](#10-artifacts)
1. [Python-based Bundles (PyDABs)](#11-python-based-bundles-pydabs)
1. [Authentication](#12-authentication)
1. [CI/CD Patterns](#13-cicd-patterns)
1. [Design Patterns & Best Practices](#14-design-patterns--best-practices)
1. [Common Errors & Fixes](#15-common-errors--fixes)

-----

## 1. Overview & Concepts

Databricks Asset Bundles (DABs) are an **Infrastructure-as-Code (IaC)** approach for managing Databricks projects. They describe Databricks resources — jobs, pipelines, clusters, ML endpoints, dashboards — as YAML (or Python) source files stored alongside code.

|Concept             |Description                                                                           |
|--------------------|--------------------------------------------------------------------------------------|
|**Bundle**          |An end-to-end project definition: source files + resource metadata, deployed as a unit|
|**`databricks.yml`**|Root configuration file; every bundle has exactly one                                 |
|**Target**          |A named deployment environment (e.g. `dev`, `staging`, `prod`)                        |
|**Resource**        |A Databricks object managed by the bundle (job, pipeline, cluster, etc.)              |
|**Artifact**        |A built output (Python wheel, JAR) produced during `bundle deploy`                    |
|**Substitution**    |Dynamic `${...}` expression resolved at deploy/run time                               |
|**Variable**        |User-defined `${var.name}` placeholder with optional default & lookup                 |
|**Mode**            |Deployment behaviour preset: `development` or `production`                            |
|**Sync**            |File synchronisation from local to workspace during `bundle deploy`                   |

### Bundle Lifecycle

```
bundle init   →   (edit YAML / code)   →   bundle validate
                                        →   bundle deploy   →   bundle run
                                        →   bundle destroy
```

-----

## 2. Project Structure

### Recommended Layout

```
my-project/
├── databricks.yml           # Root bundle config (required)
├── resources/
│   ├── my_job.yml           # Job definitions (included via glob)
│   └── my_pipeline.yml      # Pipeline definitions
├── src/
│   ├── notebook.ipynb       # Notebooks / Python files
│   └── transform.py
├── tests/
│   └── test_transform.py
├── .databricks/
│   └── bundle/
│       └── dev/
│           └── variable-overrides.json   # Local dev variable values
└── setup.py / pyproject.toml             # For whl artifacts
```

### Minimal `databricks.yml`

```yaml
bundle:
  name: my-bundle

targets:
  dev:
    default: true
    workspace:
      host: https://adb-<id>.azuredatabricks.net
```

### Multi-file Modularisation

```yaml
# databricks.yml
bundle:
  name: my-bundle
include:
  - 'resources/*.yml'    # glob includes all yml in resources/
  - 'infra/*.yml'
```

-----

## 3. CLI Commands Reference

### All Bundle Commands

|Command                              |Description                                                |Key Flags                                                               |
|-------------------------------------|-----------------------------------------------------------|------------------------------------------------------------------------|
|`databricks bundle init`             |Initialise new bundle from template                        |`--template-dir`, `--output-dir`, `--config-file`                       |
|`databricks bundle validate`         |Validate YAML config; outputs resolved JSON                |`-t <target>`, `--output json`                                          |
|`databricks bundle deploy`           |Build artifacts + sync files + create/update resources     |`-t <target>`, `--auto-approve`, `--force-lock`, `--fail-on-active-runs`|
|`databricks bundle run <key>`        |Run a job or pipeline defined in the bundle                |`-t <target>`, `--refresh-all`, `--full-refresh-table`                  |
|`databricks bundle destroy`          |Tear down all resources in the target                      |`-t <target>`, `--auto-approve`                                         |
|`databricks bundle summary`          |Print resolved bundle config as JSON                       |`-t <target>`                                                           |
|`databricks bundle schema`           |Print JSON schema for bundle config                        |                                                                        |
|`databricks bundle generate`         |Generate YAML from existing workspace resource             |`--type job|pipeline|...`, `--existing-id <id>`                         |
|`databricks bundle sync`             |Sync local files to workspace (without deploying resources)|`-t <target>`, `--watch`                                                |
|`databricks bundle open`             |Open a resource in the browser                             |`<resource-key>`                                                        |
|`databricks bundle deployment bind`  |Bind bundle resource to existing workspace resource        |`KEY RESOURCE_ID`, `--target`                                           |
|`databricks bundle deployment unbind`|Remove binding without deleting the workspace resource     |`KEY`                                                                   |

### Common Flag Patterns

```bash
# Target selection
databricks bundle deploy -t prod
databricks bundle deploy --target prod

# Pass a variable at runtime (highest precedence)
databricks bundle deploy -t prod --var="cluster_size=Large"
databricks bundle run my_job -t prod --var="env=production"

# Use a specific auth profile
databricks bundle deploy -t prod --profile my-profile

# Force deployment lock (use only if previous deploy crashed)
databricks bundle deploy -t prod --force-lock

# Skip interactive confirmation
databricks bundle deploy -t prod --auto-approve

# Run with full refresh
databricks bundle run my_pipeline -t prod --full-refresh

# Watch for file changes (sync only)
databricks bundle sync -t dev --watch
```

-----

## 4. Configuration Schema (`databricks.yml`)

### Full Top-Level Structure

```yaml
# ── Bundle identity ─────────────────────────────────────────────────
bundle:
  name: string                        # Required. Logical bundle name
  databricks_cli_version: ">= 0.218.0"
  cluster_id: string                  # Override cluster for all tasks
  git:
    origin_url: string
    branch: string
  deployment:
    fail_on_active_runs: boolean
    lock:
      enabled: boolean
      force: boolean

# ── Extra config files to merge ─────────────────────────────────────
include:
  - 'resources/*.yml'

# ── File sync settings ───────────────────────────────────────────────
sync:
  include:
    - 'src/**'
  exclude:
    - '**/__pycache__/**'
    - '.git/**'
  paths:
    - '.'

# ── Artifact build settings ─────────────────────────────────────────
artifacts:
  my_wheel:
    type: whl                         # whl | jar
    build: python setup.py bdist_wheel
    path: .
    dynamic_version: true

# ── Custom variables ─────────────────────────────────────────────────
variables:
  my_var:
    description: "Cluster size tier"
    default: "Standard_DS3_v2"
  my_cluster:
    description: "Complex cluster config"
    type: complex                     # Only valid complex type value

# ── Resources (jobs, pipelines, clusters, etc.) ──────────────────────
resources:
  jobs:
    my_job:
      name: "My ETL Job"
      # ... job config
  pipelines:
    my_pipeline:
      name: "My DLT Pipeline"
      # ... pipeline config

# ── Global permissions ───────────────────────────────────────────────
permissions:
  - level: CAN_VIEW
    group_name: data-readers
  - level: CAN_MANAGE
    user_name: admin@example.com
  - level: CAN_RUN
    service_principal_name: "123456-abcdef"

# ── Python DAB config ────────────────────────────────────────────────
python:
  venv_path: .venv
  resources:
    - "my_project.resources:load_resources"
  mutators:
    - "my_project.mutators:add_default_cluster"

# ── Run-as identity ──────────────────────────────────────────────────
run_as:
  service_principal_name: "my-sp-app-id"

# ── Workspace settings (global defaults) ─────────────────────────────
workspace:
  host: https://adb-<id>.azuredatabricks.net
  root_path: /Workspace/Users/${workspace.current_user.userName}/.bundle/${bundle.name}/${bundle.target}

# ── Targets (per-environment overrides) ──────────────────────────────
targets:
  dev:
    default: true
    mode: development
    workspace:
      host: https://adb-dev.azuredatabricks.net
    variables:
      my_var: "Standard_DS3_v2"
    presets:
      name_prefix: "[dev] "
      trigger_pause_status: PAUSED

  prod:
    mode: production
    workspace:
      host: https://adb-prod.azuredatabricks.net
    run_as:
      service_principal_name: "prod-sp-id"
    permissions:
      - level: CAN_VIEW
        group_name: prod-viewers
    presets:
      name_prefix: ""
      trigger_pause_status: UNPAUSED
      jobs_max_concurrent_runs: 1
```

-----

## 5. Top-Level Mappings Reference

### `bundle` Mapping

|Key                             |Type   |Description                                       |
|--------------------------------|-------|--------------------------------------------------|
|`name`                          |String |**Required.** Logical name for the bundle         |
|`databricks_cli_version`        |String |Version constraint, e.g. `>= 0.218.0` or `0.218.*`|
|`cluster_id`                    |String |Cluster ID to override all task clusters          |
|`git.origin_url`                |String |Git remote URL for source tracking                |
|`git.branch`                    |String |Branch name                                       |
|`deployment.fail_on_active_runs`|Boolean|Interrupt running jobs/pipelines on deploy        |
|`deployment.lock.enabled`       |Boolean|Enable concurrent deployment lock                 |
|`deployment.lock.force`         |Boolean|Force-acquire lock (use when stale)               |
|`uuid`                          |String |Reserved. Auto-generated by `bundle init`         |

### `workspace` Mapping

|Key            |Type  |Description                                              |
|---------------|------|---------------------------------------------------------|
|`host`         |String|Workspace URL, e.g. `https://adb-xxx.azuredatabricks.net`|
|`root_path`    |String|Base remote path for bundle files                        |
|`file_path`    |String|Remote path for synced source files                      |
|`artifact_path`|String|Remote path for built artifacts                          |
|`state_path`   |String|Remote path for bundle state                             |
|`profile`      |String|`.databrickscfg` profile to use                          |

### `presets` Mapping

|Key                       |Type   |Description                                               |
|--------------------------|-------|----------------------------------------------------------|
|`name_prefix`             |String |Prefix added to all resource names                        |
|`trigger_pause_status`    |String |`PAUSED` or `UNPAUSED` — applied to all schedules/triggers|
|`jobs_max_concurrent_runs`|Integer|Max concurrent runs for all jobs                          |
|`pipelines_development`   |Boolean|Lock all pipelines to development mode                    |
|`source_linked_deployment`|Boolean|Link deployment to bundle source                          |
|`tags`                    |Map    |Tags applied to all deployed resources                    |

### `sync` Mapping

|Key      |Type    |Description                              |
|---------|--------|-----------------------------------------|
|`include`|Sequence|Glob patterns of files to include in sync|
|`exclude`|Sequence|Glob patterns of files to exclude        |
|`paths`  |Sequence|Additional local paths to sync           |

### `experimental` Mapping

|Key                   |Type   |Description                |
|----------------------|-------|---------------------------|
|`python_wheel_wrapper`|Boolean|Use Python wheel wrapper   |
|`use_legacy_run_as`   |Boolean|Use legacy run_as behaviour|
|`scripts`             |Map    |Named shell scripts to run |

-----

## 6. Resources Reference

### Supported Resource Types

|Resource Key             |Python Support                  |Description                               |
|-------------------------|--------------------------------|------------------------------------------|
|`jobs`                   |✅ `databricks.bundles.jobs`     |Lakeflow Jobs (Workflows)                 |
|`pipelines`              |✅ `databricks.bundles.pipelines`|Lakeflow Spark Declarative Pipelines (DLT)|
|`clusters`               |—                               |All-purpose or job clusters               |
|`schemas`                |✅ `databricks.bundles.schemas`  |Unity Catalog schemas                     |
|`volumes`                |✅ `databricks.bundles.volumes`  |Unity Catalog volumes                     |
|`registered_models`      |—                               |UC registered models                      |
|`experiments`            |—                               |MLflow experiments                        |
|`model_serving_endpoints`|—                               |Inference serving endpoints               |
|`dashboards`             |—                               |Lakeview dashboards                       |
|`alerts`                 |—                               |SQL alerts (v2)                           |
|`apps`                   |—                               |Databricks Apps                           |
|`sql_warehouses`         |—                               |SQL warehouses                            |
|`quality_monitors`       |—                               |Lakehouse Monitor                         |
|`secret_scopes`          |—                               |Secret scopes                             |
|`database_catalogs`      |—                               |Database catalogs                         |
|`database_instances`     |—                               |Database instances                        |
|`models`                 |—                               |Legacy Model Registry models              |


> **Tip:** Use `databricks bundle generate --type job --existing-id <job-id>` to auto-generate YAML from an existing resource.

-----

### Job Resource Example

```yaml
resources:
  jobs:
    etl_job:
      name: "ETL Job — ${bundle.target}"
      description: "Daily ETL pipeline"

      # Schedule
      schedule:
        quartz_cron_expression: "0 0 6 * * ?"
        timezone_id: "UTC"
        pause_status: UNPAUSED       # overridden per-target via presets

      # Email notifications
      email_notifications:
        on_failure:
          - "oncall@example.com"
        on_start: []

      # Health monitoring
      health:
        rules:
          - metric: RUN_DURATION_SECONDS
            op: GREATER_THAN
            value: 3600

      # Shared job clusters (no libraries here — declare in task)
      job_clusters:
        - job_cluster_key: main_cluster
          new_cluster:
            spark_version: "15.4.x-scala2.12"
            node_type_id: "${var.node_type}"
            num_workers: 4
            spark_conf:
              spark.databricks.delta.preview.enabled: "true"
            custom_tags:
              project: "etl"
              environment: "${bundle.target}"

      # Tasks
      tasks:
        - task_key: ingest
          job_cluster_key: main_cluster
          notebook_task:
            notebook_path: src/ingest.ipynb
            base_parameters:
              env: "${bundle.target}"
          libraries:
            - whl: ./dist/*.whl

        - task_key: transform
          depends_on:
            - task_key: ingest
          job_cluster_key: main_cluster
          python_wheel_task:
            package_name: my_package
            entry_point: transform
          libraries:
            - whl: ./dist/*.whl

        - task_key: validate
          depends_on:
            - task_key: transform
          serverless_task:                  # Serverless compute
            environment_key: default
          notebook_task:
            notebook_path: src/validate.ipynb

      # Serverless environment spec
      environments:
        - environment_key: default
          spec:
            client: "1"
            dependencies:
              - my_package==1.0.0

      # Run as a specific identity
      run_as:
        service_principal_name: "${var.sp_app_id}"

      # Permissions scoped to this job
      permissions:
        - level: CAN_VIEW
          group_name: data-analysts
        - level: CAN_MANAGE_RUN
          group_name: data-engineers
```

-----

### Pipeline Resource Example (Spark Declarative Pipelines / DLT)

```yaml
resources:
  pipelines:
    ingestion_pipeline:
      name: "Ingestion Pipeline — ${bundle.target}"
      catalog: main
      target: "${var.schema_name}"
      channel: CURRENT                # CURRENT | PREVIEW

      development: false              # Set true in dev via presets/mode

      # Compute
      clusters:
        - label: default
          node_type_id: "${var.node_type}"
          num_workers: 2
          custom_tags:
            team: data-engineering

      # Libraries (notebooks or Python files with @dp.table/@dp.view)
      libraries:
        - notebook:
            path: src/pipelines/bronze_layer.ipynb
        - notebook:
            path: src/pipelines/silver_layer.py

      # Pipeline configuration
      configuration:
        bundle.sourcePath: "${workspace.file_path}/src"
        pipelines.enableTrackHistory: "true"

      # Notifications
      notifications:
        - email_recipients:
            - "data-team@example.com"
          alerts:
            - on-update-failure
            - on-flow-failure

      # Permissions
      permissions:
        - level: CAN_VIEW
          group_name: analysts

      # Event log
      event_log:
        catalog: main
        schema: _dlt_event_logs
```

-----

### Cluster Resource Example

```yaml
resources:
  clusters:
    shared_cluster:
      cluster_name: "shared-${bundle.target}"
      spark_version: "15.4.x-scala2.12"
      node_type_id: Standard_DS3_v2
      autoscale:
        min_workers: 1
        max_workers: 10
      spark_conf:
        spark.databricks.cluster.profile: serverless
      data_security_mode: SINGLE_USER
      runtime_engine: PHOTON
```

-----

### Schema & Volume Example

```yaml
resources:
  schemas:
    bronze_schema:
      name: bronze
      catalog_name: main
      comment: "Raw ingestion layer"
      grants:
        - principal: data-engineers
          privileges:
            - CREATE_TABLE
            - USE_SCHEMA

  volumes:
    landing_volume:
      name: landing
      catalog_name: main
      schema_name: bronze
      volume_type: MANAGED
      comment: "Landing zone for raw files"
```

-----

### Model Serving Endpoint Example

```yaml
resources:
  model_serving_endpoints:
    fraud_model_endpoint:
      name: "fraud-detection-${bundle.target}"
      config:
        served_models:
          - model_name: fraud_model
            model_version: "1"
            scale_to_zero_enabled: true
            workload_size: Small
      permissions:
        - level: CAN_VIEW
          group_name: ml-consumers
```

-----

### Registered Model Example

```yaml
resources:
  registered_models:
    fraud_model:
      name: fraud_model
      catalog_name: main
      schema_name: ml_models
      comment: "Fraud detection model"
      grants:
        - principal: ml-engineers
          privileges:
            - EXECUTE
```

-----

### MLflow Experiment Example

```yaml
resources:
  experiments:
    fraud_experiment:
      name: /Users/${workspace.current_user.userName}/fraud_experiment
      description: "Fraud detection experiments"
      permissions:
        - level: CAN_READ
          group_name: data-scientists
```

-----

## 7. Variables & Substitutions

### Built-in Substitutions

|Substitution                         |Resolves To                             |
|-------------------------------------|----------------------------------------|
|`${bundle.name}`                     |Bundle name from `bundle.name`          |
|`${bundle.target}`                   |Current target name (e.g. `dev`, `prod`)|
|`${workspace.host}`                  |Workspace URL                           |
|`${workspace.root_path}`             |Resolved workspace root path            |
|`${workspace.file_path}`             |Path where files are synced             |
|`${workspace.artifact_path}`         |Path where artifacts are stored         |
|`${workspace.current_user.userName}` |Username of deploying identity          |
|`${workspace.current_user.shortName}`|Short username without domain           |
|`${resources.<type>.<name>.<field>}` |Any field of a named resource           |

### Custom Variable Declaration

```yaml
variables:
  # Simple string variable with default
  node_type:
    description: "VM node type for clusters"
    default: "Standard_DS3_v2"

  # Variable without default (must be set at deploy time)
  sp_app_id:
    description: "Service principal application ID"

  # Lookup variable — resolves to the ID of a named object
  my_cluster_id:
    lookup:
      cluster: "shared-dev-cluster"    # Returns cluster_id by name

  my_warehouse_id:
    lookup:
      warehouse: "Shared SQL Warehouse"

  # Complex variable (structured object)
  cluster_config:
    description: "Full cluster config"
    type: complex
    default:
      spark_version: "15.4.x-scala2.12"
      node_type_id: "Standard_DS3_v2"
      num_workers: 2
```

### Setting Variable Values (Precedence Order, High → Low)

|Method                                               |Example                                      |
|-----------------------------------------------------|---------------------------------------------|
|`--var` CLI flag                                     |`--var="node_type=Standard_DS5_v2"`          |
|Environment variable `BUNDLE_VAR_*`                  |`export BUNDLE_VAR_NODE_TYPE=Standard_DS5_v2`|
|`.databricks/bundle/<target>/variable-overrides.json`|`{ "node_type": "Standard_DS5_v2" }`         |
|`variables` in `targets` mapping                     |`variables: node_type: Standard_DS5_v2`      |
|`default` in top-level `variables`                   |`default: "Standard_DS3_v2"`                 |

### Variable Reference Syntax

```yaml
resources:
  jobs:
    my_job:
      job_clusters:
        - job_cluster_key: main
          new_cluster:
            node_type_id: ${var.node_type}      # Simple variable
            num_workers: 4
      tasks:
        - task_key: run
          existing_cluster_id: ${var.my_cluster_id}  # Lookup variable
```

### Complex Variable Override File

```json
// .databricks/bundle/dev/variable-overrides.json
{
  "cluster_config": {
    "spark_version": "15.4.x-scala2.12",
    "node_type_id": "Standard_DS3_v2",
    "num_workers": 2
  }
}
```

-----

## 8. Targets & Deployment Modes

### Mode Behaviours

|Behaviour           |`development` mode        |`production` mode           |
|--------------------|--------------------------|----------------------------|
|Resource name prefix|`[dev <user>]` prepended  |None (or your `name_prefix`)|
|Schedules/triggers  |Auto-paused               |Unchanged                   |
|Pipelines           |Forced to development mode|Full production mode        |
|Deployment lock     |Disabled                  |Enabled                     |
|`run_as` required   |No                        |Yes (recommended)           |

### Multi-Target Example

```yaml
targets:
  dev:
    default: true
    mode: development
    workspace:
      host: https://adb-dev.azuredatabricks.net
    variables:
      node_type: Standard_DS3_v2
      schema_name: bronze_dev
    presets:
      name_prefix: "[dev] "
      trigger_pause_status: PAUSED
      pipelines_development: true

  staging:
    mode: development             # Still dev mode but separate workspace
    workspace:
      host: https://adb-staging.azuredatabricks.net
    variables:
      node_type: Standard_DS4_v2
      schema_name: bronze_staging
    run_as:
      service_principal_name: "staging-sp-id"

  prod:
    mode: production
    workspace:
      host: https://adb-prod.azuredatabricks.net
    variables:
      node_type: Standard_DS5_v2
      schema_name: bronze
    run_as:
      service_principal_name: "prod-sp-id"
    permissions:
      - level: CAN_VIEW
        group_name: all-users
    presets:
      trigger_pause_status: UNPAUSED
      jobs_max_concurrent_runs: 1
      tags:
        environment: production
        cost_center: "DE-001"
```

### Per-Target Resource Overrides

```yaml
resources:
  jobs:
    my_job:
      name: "My Job"
      job_clusters:
        - job_cluster_key: main
          new_cluster:
            num_workers: 2            # dev default

targets:
  prod:
    resources:
      jobs:
        my_job:
          job_clusters:
            - job_cluster_key: main
              new_cluster:
                num_workers: 10       # production override
```

-----

## 9. Permissions

### Permission Levels by Resource

|Resource          |Valid Levels                                   |
|------------------|-----------------------------------------------|
|Jobs              |`CAN_VIEW`, `CAN_MANAGE_RUN`, `CAN_MANAGE`     |
|Pipelines         |`CAN_VIEW`, `CAN_RUN`, `CAN_MANAGE`            |
|Clusters          |`CAN_ATTACH_TO`, `CAN_RESTART`, `CAN_MANAGE`   |
|SQL Warehouses    |`CAN_USE`, `CAN_MANAGE`                        |
|ML Endpoints      |`CAN_VIEW`, `CAN_QUERY`, `CAN_MANAGE`          |
|Dashboards        |`CAN_READ`, `CAN_EDIT`, `CAN_RUN`, `CAN_MANAGE`|
|Experiments       |`CAN_READ`, `CAN_EDIT`, `CAN_MANAGE`           |
|Bundle (top-level)|`CAN_VIEW`, `CAN_RUN`, `CAN_MANAGE`            |

### Permission Configuration

```yaml
# Global permissions (all bundle resources)
permissions:
  - level: CAN_VIEW
    group_name: data-readers
  - level: CAN_RUN
    group_name: data-engineers
  - level: CAN_MANAGE
    service_principal_name: "ci-sp-id"

# Resource-scoped permissions
resources:
  jobs:
    my_job:
      permissions:
        - level: CAN_VIEW
          user_name: analyst@example.com
        - level: CAN_MANAGE_RUN
          group_name: de-team
```

-----

## 10. Artifacts

### Python Wheel Artifact

```yaml
artifacts:
  my_wheel:
    type: whl
    build: |
      python -m pytest tests/ -v
      python setup.py bdist_wheel
    path: .
    dynamic_version: true    # Auto-bumps version on each deploy

resources:
  jobs:
    my_job:
      tasks:
        - task_key: run
          python_wheel_task:
            package_name: my_package
            entry_point: main
          libraries:
            - whl: ./dist/*.whl
```

### JAR Artifact

```yaml
artifacts:
  my_jar:
    type: jar
    build: mvn package -DskipTests
    path: target/
    files:
      - source: target/my-app-1.0.jar

resources:
  jobs:
    spark_job:
      tasks:
        - task_key: run
          spark_jar_task:
            main_class_name: com.example.Main
          libraries:
            - jar: ./target/my-app-1.0.jar
```

### Poetry / Custom Build

```yaml
artifacts:
  poetry_wheel:
    type: whl
    build: poetry build
    executable: bash
    path: .
```

-----

## 11. Python-based Bundles (PyDABs)

The `databricks-bundles` Python package allows defining resources in Python instead of (or alongside) YAML.

### Installation

```bash
pip install databricks-bundles
```

### `databricks.yml` Python Integration

```yaml
bundle:
  name: my-python-bundle

python:
  venv_path: .venv
  resources:
    - "my_project.resources:load_resources"
  mutators:
    - "my_project.mutators:apply_defaults"

targets:
  dev:
    default: true
    workspace:
      host: https://adb-xxx.azuredatabricks.net
```

### Python Resource Definition

```python
# my_project/resources.py
from databricks.bundles.jobs import Job, Task, NotebookTask, JobCluster, NewCluster
from databricks.bundles.pipelines import Pipeline
from databricks.bundles.core import Resources

def load_resources() -> Resources:
    job = Job(
        name="My Python Job",
        job_clusters=[
            JobCluster(
                job_cluster_key="main",
                new_cluster=NewCluster(
                    spark_version="15.4.x-scala2.12",
                    node_type_id="Standard_DS3_v2",
                    num_workers=2,
                ),
            )
        ],
        tasks=[
            Task(
                task_key="run",
                job_cluster_key="main",
                notebook_task=NotebookTask(
                    notebook_path="src/notebook.ipynb"
                ),
            )
        ],
    )
    return Resources(jobs={"my_job": job})
```

### Python Mutators

```python
# my_project/mutators.py
from databricks.bundles.core import Resources, Bundle

def apply_defaults(bundle: Bundle, resources: Resources) -> Resources:
    """Inject default tags on all jobs."""
    for job in resources.jobs.values():
        if job.tags is None:
            job.tags = {}
        job.tags["bundle"] = bundle.name
        job.tags["target"] = bundle.target
    return resources
```

-----

## 12. Authentication

### Recommended: OAuth U2M (User to Machine)

```bash
databricks auth login --host https://adb-xxx.azuredatabricks.net
```

### Service Principal for CI/CD (M2M)

```bash
# Set environment variables
export DATABRICKS_HOST=https://adb-xxx.azuredatabricks.net
export DATABRICKS_CLIENT_ID=<sp-client-id>
export DATABRICKS_CLIENT_SECRET=<sp-client-secret>
```

Or in `databricks.yml` (workspace reference only — no secrets in YAML):

```yaml
targets:
  prod:
    workspace:
      host: https://adb-prod.azuredatabricks.net
    run_as:
      service_principal_name: "${env.SP_CLIENT_ID}"
```

### Profile-based Auth

```yaml
targets:
  prod:
    workspace:
      host: https://adb-prod.azuredatabricks.net
      profile: prod-profile     # matches .databrickscfg [prod-profile]
```

### Azure Workload Identity (OIDC)

```yaml
# In GitHub Actions — no secrets needed with federated identity
- name: Deploy bundle
  env:
    ARM_CLIENT_ID: ${{ vars.SP_CLIENT_ID }}
    ARM_TENANT_ID: ${{ vars.TENANT_ID }}
    DATABRICKS_HOST: ${{ vars.DATABRICKS_HOST }}
  run: |
    databricks bundle deploy -t prod --auto-approve
```

-----

## 13. CI/CD Patterns

### GitHub Actions — Full Pipeline

```yaml
# .github/workflows/bundle-cicd.yml
name: DAB CI/CD

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

env:
  DATABRICKS_HOST: ${{ vars.DATABRICKS_HOST }}
  DATABRICKS_CLIENT_ID: ${{ vars.SP_CLIENT_ID }}
  DATABRICKS_CLIENT_SECRET: ${{ secrets.SP_CLIENT_SECRET }}

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main
      - run: databricks bundle validate -t dev

  deploy-dev:
    needs: validate
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main
      - run: databricks bundle deploy -t dev --auto-approve

  deploy-prod:
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production       # Requires manual approval gate
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main
      - run: databricks bundle deploy -t prod --auto-approve
      - run: databricks bundle run smoke_test_job -t prod
```

### Azure DevOps Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - feature/*

variables:
  DATABRICKS_HOST: $(DATABRICKS_HOST)
  DATABRICKS_CLIENT_ID: $(SP_CLIENT_ID)
  DATABRICKS_CLIENT_SECRET: $(SP_CLIENT_SECRET)

stages:
  - stage: Validate
    jobs:
      - job: validate
        pool:
          vmImage: ubuntu-latest
        steps:
          - script: curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
            displayName: Install Databricks CLI
          - script: databricks bundle validate -t dev
            displayName: Validate bundle

  - stage: DeployDev
    condition: and(succeeded(), ne(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - job: deploy
        steps:
          - script: databricks bundle deploy -t dev --auto-approve

  - stage: DeployProd
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: deploy
        environment: production   # Requires approval gate
        strategy:
          runOnce:
            deploy:
              steps:
                - script: databricks bundle deploy -t prod --auto-approve
```

-----

## 14. Design Patterns & Best Practices

### Pattern 1 — Environment Isolation via Targets

Each target deploys to a separate workspace and UC catalog. Production writes are only possible via service principal.

```yaml
variables:
  catalog:
    default: dev_catalog

targets:
  dev:
    variables:
      catalog: dev_catalog
  prod:
    variables:
      catalog: prod_catalog
    run_as:
      service_principal_name: "prod-sp"

resources:
  pipelines:
    my_pipeline:
      catalog: ${var.catalog}
      target: bronze
```

-----

### Pattern 2 — Modular Multi-file Configuration

Split large configs by domain. Use glob includes to keep `databricks.yml` minimal.

```yaml
# databricks.yml
bundle:
  name: platform-bundle
include:
  - 'resources/jobs/*.yml'
  - 'resources/pipelines/*.yml'
  - 'resources/infra/*.yml'
```

```yaml
# resources/jobs/etl_job.yml
resources:
  jobs:
    etl_job:
      name: "ETL Job"
      # ...
```

-----

### Pattern 3 — Shared Cluster Definitions via Variables

Define cluster specs once, reference everywhere.

```yaml
variables:
  default_cluster:
    type: complex
    default:
      spark_version: "15.4.x-scala2.12"
      node_type_id: Standard_DS3_v2
      num_workers: 4
      spark_conf:
        spark.databricks.delta.preview.enabled: "true"

resources:
  jobs:
    job_a:
      job_clusters:
        - job_cluster_key: main
          new_cluster: ${var.default_cluster}
    job_b:
      job_clusters:
        - job_cluster_key: main
          new_cluster: ${var.default_cluster}
```

-----

### Pattern 4 — CI/CD-Only Production Write Model

No human has `CAN_MANAGE` in production. All prod deployments run via service principal.

```yaml
targets:
  prod:
    mode: production
    run_as:
      service_principal_name: "${env.PROD_SP_CLIENT_ID}"
    permissions:
      - level: CAN_VIEW
        group_name: all-data-users
      - level: CAN_RUN
        group_name: data-engineers
      # Note: CAN_MANAGE intentionally omitted for humans
```

-----

### Pattern 5 — Dynamic Naming to Prevent Collisions

Use bundle.target and username to isolate developer workspaces.

```yaml
bundle:
  name: my-bundle

workspace:
  root_path: /Workspace/Users/${workspace.current_user.userName}/.bundle/${bundle.name}/${bundle.target}

targets:
  dev:
    mode: development       # Prepends [dev <username>] to resource names automatically
    default: true
```

-----

### Pattern 6 — Resource Binding for Brownfield Migration

When migrating existing resources, bind them to the bundle to avoid data loss.

```bash
# Generate YAML from existing job (don't destroy data)
databricks bundle generate --type job --existing-id 12345678

# Bind existing pipeline to the bundle
databricks bundle deployment bind my_pipeline 7668611149d5-abcd --target prod
```

-----

### Pattern 7 — Lookup Variables for Cross-Resource References

Avoid hard-coding IDs; resolve names to IDs at deploy time.

```yaml
variables:
  warehouse_id:
    lookup:
      warehouse: "Shared SQL Warehouse"

  cluster_id:
    lookup:
      cluster: "shared-dev-cluster"

resources:
  jobs:
    sql_job:
      tasks:
        - task_key: run
          sql_task:
            warehouse_id: ${var.warehouse_id}
            query:
              query_id: "abc-123"
```

-----

### Pattern 8 — Artifact Dynamic Versioning

Avoid bumping `setup.py` versions on every deploy.

```yaml
artifacts:
  my_wheel:
    type: whl
    build: python setup.py bdist_wheel
    path: .
    dynamic_version: true    # Patches version with timestamp automatically
```

-----

### Pattern 9 — Parameterised Job Runs

Pass parameters at runtime without re-deploying.

```bash
# Pass job parameters
databricks bundle run my_job -t prod \
  --python-params='["--date=2026-01-01", "--mode=full"]'

# Pass notebook parameters
databricks bundle run my_job -t prod \
  --notebook-params='{"date": "2026-01-01", "mode": "full"}'
```

In YAML, use dynamic value references for context-aware parameters:

```yaml
tasks:
  - task_key: run
    notebook_task:
      notebook_path: src/notebook.ipynb
      base_parameters:
        run_id: "{{job.run_id}}"
        start_time: "{{job.start_time.iso_datetime}}"
```

-----

### Pattern 10 — CLI Version Pinning

Prevent breaking changes from CLI upgrades in CI.

```yaml
bundle:
  name: my-bundle
  databricks_cli_version: ">= 0.218.0, < 1.0.0"
```

-----

## 15. Common Errors & Fixes

|Error                                      |Cause                                              |Fix                                                               |
|-------------------------------------------|---------------------------------------------------|------------------------------------------------------------------|
|`databricks_cli_version constraint not met`|CLI version too old/new                            |Update CLI: `brew upgrade databricks` / `curl install.sh`         |
|`Error: unknown field "..."`               |Typo or unsupported YAML key                       |Run `databricks bundle validate` for exact location; check schema |
|`Deployment lock is acquired`              |Previous deploy left stale lock                    |`databricks bundle deploy --force-lock -t <target>`               |
|`Variable X has no value`                  |Variable not set for the target                    |Add to `--var`, `BUNDLE_VAR_*`, or `variable-overrides.json`      |
|`Error: 'type' must be 'complex'`          |`default` is a map but `type: complex` missing     |Add `type: complex` to the variable declaration                   |
|`Permission denied` on deploy              |SP missing `CAN_MANAGE` on workspace               |Grant `IS_ADMIN` or `workspace:CAN_MANAGE` to the SP              |
|`Object with name X not found` (lookup)    |Named resource doesn’t exist in workspace          |Verify the name; check target workspace                           |
|`File not found` for artifact              |`path` incorrect or build failed                   |Check `build` command output; ensure `dist/*.whl` exists          |
|`Active runs detected`                     |`fail_on_active_runs: true` and job is running     |Wait for runs to complete or set `fail_on_active_runs: false`     |
|Bundle validates but deploy fails          |Config valid locally but workspace rejects API call|Check REST API field constraints; run `bundle summary -t <target>`|
|`databricks bundle run` fails silently     |Resource not found by key                          |Keys are case-sensitive; match exact key in YAML                  |

-----

## Quick Reference Card

### Frequently Used Substitutions

```
${bundle.name}                        → Bundle name
${bundle.target}                      → Target name (dev/prod)
${workspace.current_user.userName}    → Deploying user email
${workspace.current_user.shortName}   → Deploying user short name
${workspace.host}                     → Workspace URL
${workspace.file_path}                → Synced files path
${workspace.artifact_path}            → Artifact storage path
${resources.jobs.my_job.id}           → Resolved job ID
${resources.pipelines.my_pipeline.id} → Resolved pipeline ID
${var.my_variable}                    → Custom variable value
```

### One-liner Cheatsheet

```bash
# Initialise from template
databricks bundle init

# Validate (with JSON output for scripting)
databricks bundle validate -t dev --output json

# Deploy to dev (default target)
databricks bundle deploy

# Deploy to prod with approval
databricks bundle deploy -t prod --auto-approve

# Run a job
databricks bundle run my_job -t prod

# Run with parameters
databricks bundle run my_job -t prod --notebook-params='{"key":"value"}'

# Sync files only (fast iteration)
databricks bundle sync -t dev --watch

# Generate YAML from existing resource
databricks bundle generate --type job --existing-id 123456

# Bind to existing resource (brownfield)
databricks bundle deployment bind my_pipeline 7668abc --target prod

# Destroy target resources
databricks bundle destroy -t dev --auto-approve

# Check CLI version
databricks --version
```

-----

*Sources: [Databricks Asset Bundles documentation](https://docs.databricks.com/aws/en/dev-tools/bundles/) · [Configuration reference](https://docs.databricks.com/aws/en/dev-tools/bundles/reference) · [Resources reference](https://docs.databricks.com/aws/en/dev-tools/bundles/resources) · [Variables & substitutions](https://docs.databricks.com/aws/en/dev-tools/bundles/variables) · [CLI commands](https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands)*