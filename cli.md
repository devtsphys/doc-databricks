# Databricks CLI Cheat Sheet

> CLI v0.205+ (New Go-based CLI) · Last updated March 2026

-----

## Table of Contents

1. [Installation & Upgrade](#installation--upgrade)
1. [Authentication](#authentication)
1. [Configuration Profiles](#configuration-profiles)
1. [Global Flags](#global-flags)
1. [Command Groups Reference](#command-groups-reference)
- [Workspace Commands](#workspace-commands)
- [Compute Commands](#compute-commands)
- [Jobs Commands](#jobs-commands)
- [Pipelines Commands](#pipelines-commands)
- [Machine Learning Commands](#machine-learning-commands)
- [Serving Endpoints Commands](#serving-endpoints-commands)
- [SQL Commands](#sql-commands)
- [Unity Catalog Commands](#unity-catalog-commands)
- [Identity & Access Commands](#identity--access-commands)
- [Account-Level Commands](#account-level-commands)
1. [Bundle (DABs) Commands](#bundle-declarative-automation-bundles)
1. [Filesystem (fs) Commands](#filesystem-fs-commands)
1. [Output Formatting & Filtering](#output-formatting--filtering)
1. [Secrets Management](#secrets-management)
1. [CI/CD Integration Patterns](#cicd-integration-patterns)
1. [Design Patterns & Best Practices](#design-patterns--best-practices)
1. [Quick Reference: All Command Groups](#quick-reference-all-command-groups)

-----

## Installation & Upgrade

### Install (Linux / macOS via curl)

```bash
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
```

### Install via Homebrew (macOS/Linux)

```bash
brew tap databricks/tap
brew install databricks
```

### Install via winget (Windows)

```bash
winget install Databricks.DatabricksCLI
```

### Install via Docker

```bash
docker pull ghcr.io/databricks/cli:latest
docker run -e DATABRICKS_HOST=$HOST -e DATABRICKS_TOKEN=$TOKEN \
  ghcr.io/databricks/cli:latest current-user me
```

### Verify & Update

```bash
databricks version              # check installed version
databricks update               # self-update to latest
databricks -v                   # short version flag
```

> **Note:** The new Go-based CLI (v0.200+) is a complete rewrite of the legacy Python-based CLI (v0.17 and below). The legacy CLI is no longer actively developed. Migrate via: `databricks migrate`.

-----

## Authentication

### OAuth U2M (recommended for interactive use)

```bash
databricks auth login --host https://<workspace-url>.azuredatabricks.net
# Opens browser for OAuth flow; tokens auto-refresh
```

### OAuth M2M (Service Principal — recommended for CI/CD)

```bash
# Set environment variables
export DATABRICKS_HOST=https://<workspace>.azuredatabricks.net
export DATABRICKS_CLIENT_ID=<sp-client-id>
export DATABRICKS_CLIENT_SECRET=<sp-client-secret>
```

### PAT (Personal Access Token)

```bash
databricks configure --token
# Prompts for host + token, writes to ~/.databrickscfg
```

### Azure MSI / Workload Identity (Azure)

```bash
export ARM_USE_MSI=true
export DATABRICKS_HOST=https://<workspace>.azuredatabricks.net
```

### Environment Variable Auth

```bash
export DATABRICKS_HOST=https://<workspace>.azuredatabricks.net
export DATABRICKS_TOKEN=<pat-or-token>
```

### Check Current Auth

```bash
databricks auth describe                # show current auth details
databricks auth env                     # show auth as env vars
databricks auth profiles                # list all configured profiles
databricks auth token                   # print current token
databricks current-user me              # validate auth works
```

-----

## Configuration Profiles

Profiles are stored in `~/.databrickscfg`.

### Configure a Named Profile

```bash
databricks configure --profile dev --token
databricks configure --profile prod --token
```

### Example `~/.databrickscfg`

```ini
[DEFAULT]
host  = https://adb-xxx.azuredatabricks.net
token = dapiXXXXXXXXXXX

[dev]
host  = https://adb-dev.azuredatabricks.net
token = dapiDEV

[prod]
host  = https://adb-prod.azuredatabricks.net
token = dapiPROD
```

### Use a Profile

```bash
databricks clusters list --profile prod
databricks jobs list -p dev
```

-----

## Global Flags

These flags are available on every command:

|Flag                |Short|Description                                                     |
|--------------------|-----|----------------------------------------------------------------|
|`--profile <name>`  |`-p` |Use named profile from `~/.databrickscfg`                       |
|`--output <format>` |`-o` |Output format: `text` (default) or `json`                       |
|`--log-file <path>` |     |Write logs to file (default: stderr)                            |
|`--log-format <fmt>`|     |Log format: `text` (default) or `json`                          |
|`--log-level <lvl>` |     |Log level: `disabled`, `trace`, `debug`, `info`, `warn`, `error`|
|`--help`            |`-h` |Show help for any command                                       |
|`--version`         |`-v` |Show CLI version                                                |

```bash
# Examples
databricks clusters list --output json
databricks jobs list --profile prod --output json
databricks clusters get <id> --log-level debug
```

-----

## Command Groups Reference

### Workspace Commands

#### `workspace` — Notebooks & Folders

```bash
databricks workspace list /path                      # list contents
databricks workspace list /path --output json        # JSON output
databricks workspace get-status /path/notebook       # stat a file
databricks workspace mkdirs /path/new-folder         # create directory
databricks workspace export /Shared/notebook.py ./notebook.py   # download
databricks workspace export /Shared/nb --format SOURCE          # export as source
databricks workspace import ./local.py /Workspace/Users/u@e.com/remote.py
databricks workspace import-dir ./local-dir /Workspace/dest     # recursive upload
databricks workspace export-dir /Workspace/src ./local-dest     # recursive download
databricks workspace delete /path/notebook          # delete single item
databricks workspace delete /path/folder --recursive # recursive delete

# Permissions
databricks workspace get-permissions /path
databricks workspace set-permissions /path --json '{"access_control_list": [...]}'
```

#### `repos` — Git Repos

```bash
databricks repos list
databricks repos create --url https://github.com/org/repo --provider gitHub
databricks repos get <repo-id>
databricks repos update <repo-id> --branch main
databricks repos delete <repo-id>
```

#### `git-credentials` — Git Credentials

```bash
databricks git-credentials list
databricks git-credentials create --git-provider gitHub \
  --git-username myuser --personal-access-token mytoken
databricks git-credentials delete <credential-id>
```

#### `fs` — File System (DBFS / Volumes)

```bash
databricks fs ls dbfs:/path                        # list DBFS
databricks fs ls dbfs:/ --output json
databricks fs cp ./local.csv dbfs:/FileStore/      # upload
databricks fs cp dbfs:/FileStore/data.csv ./       # download
databricks fs cp -r ./folder dbfs:/dest/           # recursive upload
databricks fs mkdir dbfs:/new-path
databricks fs rm dbfs:/path/file
databricks fs rm -r dbfs:/path/folder              # recursive delete
databricks fs cat dbfs:/path/file.txt              # print contents
```

> **Volumes path syntax:** `dbfs:/Volumes/<catalog>/<schema>/<volume>/path`

-----

### Compute Commands

#### `clusters` — Clusters

```bash
# List & Get
databricks clusters list
databricks clusters list --output json | jq '.[] | {id:.cluster_id, name:.cluster_name}'
databricks clusters get <cluster-id>
databricks clusters events <cluster-id>

# Lifecycle
databricks clusters start <cluster-id>
databricks clusters restart <cluster-id>
databricks clusters delete <cluster-id>          # terminate
databricks clusters permanent-delete <cluster-id>

# Create (inline JSON)
databricks clusters create --json '{
  "cluster_name": "my-cluster",
  "spark_version": "15.4.x-scala2.12",
  "node_type_id": "Standard_DS3_v2",
  "num_workers": 2,
  "autotermination_minutes": 120
}'

# Create from file
databricks clusters create --json @cluster-config.json

# Edit
databricks clusters edit <cluster-id> --json @updated-config.json

# Resize
databricks clusters resize <cluster-id> --num-workers 4

# Pin / Unpin (prevent auto-purge)
databricks clusters pin <cluster-id>
databricks clusters unpin <cluster-id>

# Reference data
databricks clusters spark-versions               # list available runtimes
databricks clusters list-node-types              # list available node types
databricks clusters list-zones                   # list available zones

# Permissions
databricks clusters get-permissions <cluster-id>
databricks clusters set-permissions <cluster-id> --json '{
  "access_control_list": [
    {"group_name": "data-engineers", "permission_level": "CAN_RESTART"}
  ]
}'
```

#### `cluster-policies` — Cluster Policies

```bash
databricks cluster-policies list
databricks cluster-policies get <policy-id>
databricks cluster-policies create --json @policy.json
databricks cluster-policies edit <policy-id> --json @updated-policy.json
databricks cluster-policies delete <policy-id>
databricks cluster-policies get-permissions <policy-id>
databricks cluster-policies set-permissions <policy-id> --json '...'
```

#### `instance-pools` — Instance Pools

```bash
databricks instance-pools list
databricks instance-pools get <pool-id>
databricks instance-pools create --json '{
  "instance_pool_name": "my-pool",
  "node_type_id": "Standard_DS3_v2",
  "min_idle_instances": 2,
  "max_capacity": 10
}'
databricks instance-pools edit <pool-id> --json @updated-pool.json
databricks instance-pools delete <pool-id>
```

#### `libraries` — Cluster Libraries

```bash
databricks libraries cluster-status <cluster-id>
databricks libraries all-cluster-statuses
databricks libraries install <cluster-id> --json '{
  "libraries": [{"pypi": {"package": "pandas==2.0.0"}}]
}'
databricks libraries uninstall <cluster-id> --json '{
  "libraries": [{"pypi": {"package": "pandas"}}]
}'
```

#### `global-init-scripts` — Init Scripts

```bash
databricks global-init-scripts list
databricks global-init-scripts get <script-id>
databricks global-init-scripts create --name "my-init" \
  --script "$(base64 < init.sh)" --enabled true
databricks global-init-scripts update <script-id> --json @update.json
databricks global-init-scripts delete <script-id>
```

-----

### Jobs Commands

#### `jobs` — Jobs (Lakeflow Jobs)

```bash
# List & Get
databricks jobs list
databricks jobs list --output json
databricks jobs get <job-id>

# Create
databricks jobs create --json @job-definition.json
databricks jobs create --json '{
  "name": "My Job",
  "tasks": [{
    "task_key": "etl",
    "notebook_task": {"notebook_path": "/Shared/etl"},
    "new_cluster": {
      "spark_version": "15.4.x-scala2.12",
      "node_type_id": "Standard_DS3_v2",
      "num_workers": 2
    }
  }],
  "schedule": {
    "quartz_cron_expression": "0 0 8 * * ?",
    "timezone_id": "UTC"
  }
}'

# Update / Reset
databricks jobs update <job-id> --json @partial-update.json   # partial update
databricks jobs reset <job-id> --json @full-definition.json   # full replace

# Delete
databricks jobs delete <job-id>

# Run Now
databricks jobs run-now <job-id>
databricks jobs run-now <job-id> --job-parameters '{"env": "prod"}'

# List / Get Runs
databricks jobs list-runs --job-id <job-id>
databricks jobs list-runs --active-only
databricks jobs list-runs --completed-only --limit 10
databricks jobs get-run <run-id>
databricks jobs get-run-output <run-id>
databricks jobs export-run <run-id>

# Cancel Runs
databricks jobs cancel-run <run-id>
databricks jobs cancel-all-runs <job-id>

# Repair a Failed Run
databricks jobs repair-run <run-id> --rerun-tasks task1,task2

# Submit One-Time Run (no job definition saved)
databricks jobs submit --json @one-time-run.json

# Delete Run
databricks jobs delete-run <run-id>

# Permissions
databricks jobs get-permissions <job-id>
databricks jobs set-permissions <job-id> --json '...'
```

-----

### Pipelines Commands

#### `pipelines` — Lakeflow Spark Declarative Pipelines (DLT)

```bash
# List & Get
databricks pipelines list-pipelines
databricks pipelines get <pipeline-id>

# Create / Update
databricks pipelines create --json @pipeline.json
databricks pipelines update <pipeline-id> --json @update.json

# Run
databricks pipelines start-update <pipeline-id>
databricks pipelines start-update <pipeline-id> --full-refresh          # full refresh
databricks pipelines start-update <pipeline-id> --full-refresh-selection "table1,table2"

# Stop
databricks pipelines stop <pipeline-id>

# Updates History
databricks pipelines list-updates <pipeline-id>
databricks pipelines get-update <pipeline-id> <update-id>

# Events / Logs
databricks pipelines list-pipeline-events <pipeline-id>
databricks pipelines list-pipeline-events <pipeline-id> --filter "level='ERROR'"

# Delete
databricks pipelines delete <pipeline-id>

# Permissions
databricks pipelines get-permissions <pipeline-id>
databricks pipelines set-permissions <pipeline-id> --json '...'
```

**Pipeline JSON skeleton:**

```json
{
  "name": "my-pipeline",
  "target": "my_schema",
  "catalog": "main",
  "libraries": [
    {"notebook": {"path": "/Shared/pipeline-notebook"}}
  ],
  "clusters": [{"label": "default", "num_workers": 2}],
  "continuous": false,
  "channel": "CURRENT"
}
```

-----

### Machine Learning Commands

#### `experiments` — MLflow Experiments

```bash
databricks experiments list-experiments
databricks experiments get-experiment <experiment-id>
databricks experiments get-by-name --experiment-name "/path/to/exp"
databricks experiments create-experiment --name "/Users/u@e.com/my-exp"
databricks experiments delete-experiment <experiment-id>
databricks experiments restore-experiment <experiment-id>

# Runs
databricks experiments search-runs --json '{"experiment_ids": ["123"]}'
databricks experiments get-run <run-id>
databricks experiments create-run --experiment-id <id>
databricks experiments log-metric --run-id <id> --key accuracy --value 0.95 --step 1
databricks experiments log-param --run-id <id> --key lr --value 0.001
databricks experiments set-tag --run-id <id> --key env --value prod
databricks experiments delete-run <run-id>

# Artifacts
databricks experiments list-artifacts --run-id <id>
databricks experiments list-artifacts --run-id <id> --path model

# Permissions
databricks experiments get-permissions <experiment-id>
databricks experiments set-permissions <experiment-id> --json '...'
```

#### `model-registry` — MLflow Model Registry

```bash
databricks model-registry list-models
databricks model-registry get-model <model-name>
databricks model-registry create-model --name my-model
databricks model-registry create-model-version --name my-model \
  --source "runs:/<run-id>/model" --run-id <run-id>
databricks model-registry get-model-version --name my-model --version 3
databricks model-registry transition-stage --name my-model --version 3 \
  --stage Production --archive-existing-versions
databricks model-registry delete-model <model-name>
databricks model-registry delete-model-version --name my-model --version 1
```

-----

### Serving Endpoints Commands

#### `serving-endpoints` — Model Serving

```bash
databricks serving-endpoints list
databricks serving-endpoints get <endpoint-name>
databricks serving-endpoints create --json '{
  "name": "my-endpoint",
  "config": {
    "served_models": [{
      "name": "my-model-v1",
      "model_name": "my-model",
      "model_version": "3",
      "workload_size": "Small",
      "scale_to_zero_enabled": true
    }]
  }
}'
databricks serving-endpoints update-config <endpoint-name> --json @config.json
databricks serving-endpoints patch <endpoint-name> --json '{"tags": [...]}'
databricks serving-endpoints delete <endpoint-name>

# Monitoring
databricks serving-endpoints logs <endpoint-name> <served-model-name>
databricks serving-endpoints build-logs <endpoint-name> <served-model-name>
databricks serving-endpoints export-metrics <endpoint-name>

# Query
databricks serving-endpoints query <endpoint-name> --json '{"inputs": [[1,2,3]]}'

# Permissions
databricks serving-endpoints get-permissions <endpoint-name>
databricks serving-endpoints set-permissions <endpoint-name> --json '...'
```

-----

### SQL Commands

#### `warehouses` — SQL Warehouses

```bash
databricks warehouses list
databricks warehouses get <warehouse-id>
databricks warehouses create --json '{
  "name": "my-warehouse",
  "cluster_size": "Small",
  "auto_stop_mins": 10,
  "warehouse_type": "PRO"
}'
databricks warehouses start <warehouse-id>
databricks warehouses stop <warehouse-id>
databricks warehouses edit <warehouse-id> --json @update.json
databricks warehouses delete <warehouse-id>
databricks warehouses get-workspace-warehouse-config
databricks warehouses set-workspace-warehouse-config --json @config.json
```

#### `queries` — SQL Queries

```bash
databricks queries list
databricks queries get <query-id>
databricks queries create --json '{"query": "SELECT 1", "display_name": "Test"}'
databricks queries update <query-id> --json @update.json
databricks queries delete <query-id>
databricks queries restore <query-id>
```

#### `alerts` — SQL Alerts

```bash
databricks alerts list
databricks alerts get <alert-id>
databricks alerts create --json @alert.json
databricks alerts update <alert-id> --json @update.json
databricks alerts delete <alert-id>
```

#### `query-history` — Query History

```bash
databricks query-history list
databricks query-history list --json '{
  "filter_by": {
    "warehouse_ids": ["<id>"],
    "statuses": ["FAILED"]
  },
  "max_results": 50
}'
```

-----

### Unity Catalog Commands

#### `catalogs`

```bash
databricks catalogs list
databricks catalogs get <catalog-name>
databricks catalogs create --name my-catalog --comment "Production catalog"
databricks catalogs update <catalog-name> --comment "Updated"
databricks catalogs delete <catalog-name>
databricks catalogs delete <catalog-name> --force   # force-delete non-empty
```

#### `schemas`

```bash
databricks schemas list --catalog-name main
databricks schemas get main.my_schema
databricks schemas create --catalog-name main --name my_schema
databricks schemas update main.my_schema --comment "Updated"
databricks schemas delete main.my_schema
```

#### `tables`

```bash
databricks tables list --catalog-name main --schema-name default
databricks tables get main.default.my_table
databricks tables delete main.default.my_table
databricks tables list-summaries --catalog-name main
databricks tables get-table-constraints main.default.my_table
```

#### `volumes`

```bash
databricks volumes list --catalog-name main --schema-name default
databricks volumes get main.default.my_volume
databricks volumes create --catalog-name main --schema-name default \
  --name my_volume --volume-type MANAGED
databricks volumes delete main.default.my_volume
```

#### `external-locations`

```bash
databricks external-locations list
databricks external-locations get <location-name>
databricks external-locations create --name my-ext-loc \
  --url "abfss://container@account.dfs.core.windows.net/path" \
  --credential-name my-credential
databricks external-locations delete <location-name>
databricks external-locations validate --json @validate.json
```

#### `storage-credentials`

```bash
databricks storage-credentials list
databricks storage-credentials get <credential-name>
databricks storage-credentials create --json @storage-cred.json
databricks storage-credentials delete <credential-name>
```

#### `credentials` (UC Service Credentials)

```bash
databricks credentials list-credentials
databricks credentials get-credential <credential-name>
databricks credentials create-credential --json @cred.json
databricks credentials delete-credential <credential-name>
databricks credentials validate-credential --json @validate.json
```

#### `metastores`

```bash
databricks metastores list
databricks metastores get <metastore-id>
databricks metastores summary                  # get current workspace metastore
databricks metastores create --name my-metastore --storage-root "abfss://..."
databricks metastores assign <metastore-id> --workspace-id <ws-id>
databricks metastores delete <metastore-id>
```

#### `grants` — UC Privilege Grants

```bash
# Get grants on an object
databricks grants get TABLE main.default.my_table
databricks grants get SCHEMA main.default
databricks grants get CATALOG main

# Update grants
databricks grants update TABLE main.default.my_table --json '{
  "changes": [
    {
      "principal": "data-engineers",
      "add": ["SELECT", "MODIFY"]
    }
  ]
}'

# Get effective permissions
databricks grants get-effective TABLE main.default.my_table
```

#### `connections` — Lakehouse Federation

```bash
databricks connections list
databricks connections get <connection-name>
databricks connections create --json '{
  "name": "my-postgres",
  "connection_type": "POSTGRESQL",
  "options": {"host": "host", "port": "5432", "database": "mydb"},
  "credential_type": "USERNAME_PASSWORD",
  "credential": {"username": "user", "password": "pass"}
}'
databricks connections delete <connection-name>
```

#### `data-quality` — Lakehouse Monitoring

```bash
databricks data-quality create-monitor --table-name main.default.my_table \
  --json @monitor-config.json
databricks data-quality get-monitor --table-name main.default.my_table
databricks data-quality list-monitor
databricks data-quality update-monitor --table-name main.default.my_table --json @update.json
databricks data-quality delete-monitor --table-name main.default.my_table
databricks data-quality create-refresh --table-name main.default.my_table
databricks data-quality get-refresh --table-name main.default.my_table --refresh-id <id>
databricks data-quality list-refresh --table-name main.default.my_table
databricks data-quality cancel-refresh --table-name main.default.my_table --refresh-id <id>
```

-----

### Identity & Access Commands

#### `users`

```bash
databricks users list
databricks users list --filter 'displayName co "Alice"'
databricks users get <user-id>
databricks users create --json '{"userName": "user@example.com", "displayName": "Alice"}'
databricks users patch <user-id> --json '{"Operations": [{"op": "add", "path": "roles", "value": [...]}]}'
databricks users delete <user-id>
```

#### `groups`

```bash
databricks groups list
databricks groups get <group-id>
databricks groups create --json '{"displayName": "data-engineers"}'
databricks groups patch <group-id> --json '{
  "Operations": [{
    "op": "add",
    "path": "members",
    "value": [{"value": "<user-id>"}]
  }]
}'
databricks groups delete <group-id>
```

#### `service-principals`

```bash
databricks service-principals list
databricks service-principals get <sp-id>
databricks service-principals create --json '{"displayName": "my-sp"}'
databricks service-principals patch <sp-id> --json '...'
databricks service-principals delete <sp-id>
```

#### `permissions`

```bash
databricks permissions get <object-type> <object-id>
databricks permissions set <object-type> <object-id> --json @acl.json
databricks permissions update <object-type> <object-id> --json @acl.json
databricks permissions get-permission-levels <object-type> <object-id>
# Object types: clusters, jobs, notebooks, directories, registered-models,
#               sql/warehouses, sql/queries, sql/alerts, pipelines, ...
```

-----

### Account-Level Commands

```bash
# Switch to account-level context
export DATABRICKS_HOST=https://accounts.azuredatabricks.net
export DATABRICKS_ACCOUNT_ID=<your-account-id>

databricks account workspaces list
databricks account workspaces get <workspace-id>
databricks account metastores list
databricks account metastore-assignments list
databricks account service-principals list
databricks account groups list
databricks account users list
databricks account budgets list
databricks account billable-usage download --start-month 2025-01 --end-month 2025-12
databricks account log-delivery list
databricks account ip-access-lists list
databricks account network-connectivity list
```

-----

## Bundle (Declarative Automation Bundles)

Bundles (formerly DABs) are the IaC mechanism for deploying Databricks resources. They use `databricks.yml` as the root configuration file.

### Bundle Lifecycle Commands

|Command                              |Description                                        |
|-------------------------------------|---------------------------------------------------|
|`databricks bundle init`             |Initialize new bundle from template                |
|`databricks bundle validate`         |Validate bundle configuration YAML                 |
|`databricks bundle deploy`           |Deploy bundle to remote workspace                  |
|`databricks bundle run <resource>`   |Run a deployed bundle resource                     |
|`databricks bundle destroy`          |Destroy all resources created by bundle            |
|`databricks bundle summary`          |Show bundle summary and resource URLs              |
|`databricks bundle generate`         |Generate bundle config from existing resource      |
|`databricks bundle deployment bind`  |Bind bundle resource to existing workspace resource|
|`databricks bundle deployment unbind`|Unbind a bundle-managed resource                   |

```bash
# Init from built-in template
databricks bundle init                                     # interactive
databricks bundle init default-python                      # Python job template
databricks bundle init default-sql                         # SQL template
databricks bundle init mlops-stacks                        # MLOps template

# Init from custom Git template
databricks bundle init https://github.com/org/my-template

# Validate
databricks bundle validate
databricks bundle validate -t dev                          # target-specific

# Deploy
databricks bundle deploy
databricks bundle deploy -t dev                            # to dev target
databricks bundle deploy -t prod                           # to prod target
databricks bundle deploy --force-lock                      # skip state lock check
databricks bundle deploy --auto-approve                    # skip confirmation

# Run
databricks bundle run my_job
databricks bundle run my_pipeline
databricks bundle run my_job -t prod
databricks bundle run my_job --python-params '["--env", "prod"]'
databricks bundle run my_job --notebook-params '{"param": "value"}'

# Summary & open in browser
databricks bundle summary
databricks bundle summary -t prod
databricks bundle open my_job                              # open resource in browser

# Destroy
databricks bundle destroy
databricks bundle destroy -t dev --auto-approve

# Generate config from existing resource
databricks bundle generate job --existing-job-id <id>
databricks bundle generate pipeline --existing-pipeline-id <id>
databricks bundle generate app --existing-app-name <name>

# Bind / Unbind
databricks bundle deployment bind my_pipeline <pipeline-id> -t prod
databricks bundle deployment unbind my_pipeline -t prod
```

### `databricks.yml` Structure

```yaml
bundle:
  name: my-project                          # Required
  databricks_cli_version: ">=0.218.0"       # Min CLI version

# Additional config files to merge
include:
  - resources/*.yml
  - environments/*.yml

# Variables (parameterize configs)
variables:
  cluster_size:
    description: "Cluster node type"
    default: "Standard_DS3_v2"
  env_tag:
    description: "Deployment environment"

# Artifacts (local code to upload)
artifacts:
  my_wheel:
    type: whl
    path: ./dist/my_package-*.whl

# Resource definitions
resources:
  jobs:
    my_etl_job:
      name: "My ETL Job [${bundle.target}]"
      tasks:
        - task_key: etl_task
          notebook_task:
            notebook_path: ./notebooks/etl.py
            source: WORKSPACE
          job_cluster_key: main_cluster
      job_clusters:
        - job_cluster_key: main_cluster
          new_cluster:
            spark_version: "15.4.x-scala2.12"
            node_type_id: ${var.cluster_size}
            num_workers: 2
      schedule:
        quartz_cron_expression: "0 0 6 * * ?"
        timezone_id: "UTC"
      email_notifications:
        on_failure:
          - "oncall@company.com"

  pipelines:
    my_dlt_pipeline:
      name: "My DLT Pipeline [${bundle.target}]"
      target: "${bundle.target}_catalog.my_schema"
      libraries:
        - notebook:
            path: ./pipelines/ingest.py
      clusters:
        - label: default
          num_workers: 2

  # Run as specific identity
run_as:
  service_principal_name: "my-ci-sp"

# Permissions applied to all resources
permissions:
  - level: CAN_MANAGE
    group_name: "data-platform-admins"
  - level: CAN_VIEW
    group_name: "data-consumers"

# Target environments
targets:
  dev:
    mode: development                       # adds [dev <user>] prefix, pauses schedules
    default: true
    workspace:
      host: https://adb-dev.azuredatabricks.net
    variables:
      cluster_size: "Standard_DS2_v2"

  staging:
    workspace:
      host: https://adb-staging.azuredatabricks.net

  prod:
    mode: production                        # enables schedules, strict permissions
    workspace:
      host: https://adb-prod.azuredatabricks.net
    run_as:
      service_principal_name: "prod-deploy-sp"
    variables:
      cluster_size: "Standard_DS3_v2"
```

### Bundle Variable Substitution

```yaml
# Built-in variables
${bundle.name}                     # bundle name
${bundle.target}                   # current target name
${workspace.host}                  # workspace URL
${workspace.current_user.userName} # current user email
${workspace.file_path}             # /Workspace/Users/<user>/.bundle/<name>/<target>/files
${workspace.root_path}             # bundle root in workspace

# Custom variable reference
${var.my_variable}
```

### Deployment Modes

|Mode         |Behavior                                                                                                        |
|-------------|----------------------------------------------------------------------------------------------------------------|
|`development`|Prefixes resource names with `[dev <username>]`, pauses all schedules/triggers, enables source-linked deployment|
|`production` |Enforces `run_as`, validates no personal identities own resources, all schedules active                         |
|*(none)*     |No automatic modifications                                                                                      |

-----

## Filesystem (fs) Commands

### Complete Method Table

|Command   |Arguments    |Flags                          |Description           |
|----------|-------------|-------------------------------|----------------------|
|`fs ls`   |`<path>`     |`--output json`                |List files/directories|
|`fs cp`   |`<src> <dst>`|`-r` (recursive), `--overwrite`|Copy files            |
|`fs mv`   |`<src> <dst>`|                               |Move/rename           |
|`fs rm`   |`<path>`     |`-r` (recursive)               |Delete                |
|`fs mkdir`|`<path>`     |                               |Create directory      |
|`fs cat`  |`<path>`     |                               |Print file contents   |

### Path Prefixes

|Prefix                             |Target                       |
|-----------------------------------|-----------------------------|
|`dbfs:/`                           |Databricks File System (DBFS)|
|`dbfs:/Volumes/catalog/schema/vol/`|Unity Catalog Volume         |
|`/Workspace/`                      |Workspace filesystem         |

-----

## Output Formatting & Filtering

```bash
# JSON output for scripting
databricks clusters list --output json

# Pipe to jq for filtering
databricks clusters list --output json | jq '.[] | select(.state == "RUNNING") | .cluster_id'
databricks jobs list --output json | jq '[.[] | {id: .job_id, name: .settings.name}]'

# Get a single field
databricks clusters list --output json | jq -r '.[0].cluster_id'

# Paginated listing (all pages)
databricks jobs list --output json | jq '.jobs[]'

# Grep text output
databricks clusters list | grep RUNNING

# Save JSON to file
databricks jobs get <job-id> --output json > job-backup.json

# Use saved JSON to re-create
databricks jobs create --json @job-backup.json
```

-----

## Secrets Management

```bash
# Scopes
databricks secrets list-scopes
databricks secrets create-scope my-scope
databricks secrets create-scope my-scope --initial-manage-principal users
databricks secrets delete-scope my-scope

# Secrets
databricks secrets list-secrets my-scope
databricks secrets put-secret my-scope my-key --string-value "my-secret-value"
databricks secrets put-secret my-scope my-key --bytes-value "$(base64 < keyfile.pem)"
databricks secrets get-secret my-scope my-key        # returns base64-encoded value
databricks secrets delete-secret my-scope my-key

# ACLs
databricks secrets list-acls my-scope
databricks secrets get-acl my-scope my-group
databricks secrets put-acl my-scope my-group MANAGE  # MANAGE | READ | WRITE
databricks secrets delete-acl my-scope my-group

# Read a secret in Python notebook (cannot be read back via CLI after write)
# dbutils.secrets.get(scope="my-scope", key="my-key")
```

-----

## CI/CD Integration Patterns

### GitHub Actions — Bundle Deploy

```yaml
name: Deploy to Databricks
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Databricks CLI
        run: curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

      - name: Deploy bundle
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_CLIENT_ID: ${{ secrets.DATABRICKS_CLIENT_ID }}
          DATABRICKS_CLIENT_SECRET: ${{ secrets.DATABRICKS_CLIENT_SECRET }}
        run: |
          databricks bundle validate -t prod
          databricks bundle deploy -t prod --auto-approve
```

### Azure DevOps Pipeline — Bundle Deploy

```yaml
trigger:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

steps:
  - script: |
      curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
    displayName: Install Databricks CLI

  - script: |
      databricks bundle validate -t prod
      databricks bundle deploy -t prod --auto-approve
    displayName: Deploy Bundle
    env:
      DATABRICKS_HOST: $(DATABRICKS_HOST)
      DATABRICKS_CLIENT_ID: $(DATABRICKS_CLIENT_ID)
      DATABRICKS_CLIENT_SECRET: $(DATABRICKS_CLIENT_SECRET)
```

### OpenTofu / Terraform — Run CLI as `local-exec`

```hcl
resource "null_resource" "deploy_bundle" {
  triggers = { always = timestamp() }
  provisioner "local-exec" {
    command = "databricks bundle deploy -t ${var.environment} --auto-approve"
    environment = {
      DATABRICKS_HOST          = var.workspace_url
      DATABRICKS_CLIENT_ID     = var.sp_client_id
      DATABRICKS_CLIENT_SECRET = var.sp_client_secret
    }
  }
}
```

### Script: Wait for Job Run to Complete

```bash
#!/usr/bin/env bash
set -euo pipefail

JOB_ID=$1
PROFILE=${2:-DEFAULT}

# Trigger job
RUN_ID=$(databricks jobs run-now "$JOB_ID" --profile "$PROFILE" \
  --output json | jq -r '.run_id')
echo "Started run: $RUN_ID"

# Poll until terminal state
while true; do
  STATE=$(databricks jobs get-run "$RUN_ID" --profile "$PROFILE" \
    --output json | jq -r '.state.life_cycle_state')
  echo "  State: $STATE"
  case "$STATE" in
    TERMINATED|SKIPPED|INTERNAL_ERROR) break ;;
    *) sleep 15 ;;
  esac
done

RESULT=$(databricks jobs get-run "$RUN_ID" --profile "$PROFILE" \
  --output json | jq -r '.state.result_state')
echo "Result: $RESULT"
[[ "$RESULT" == "SUCCESS" ]] && exit 0 || exit 1
```

### Script: Promote Notebooks (Dev → Prod)

```bash
#!/usr/bin/env bash
set -euo pipefail

SRC_PROFILE=dev
DST_PROFILE=prod
SRC_PATH="/Workspace/Users/dev@company.com/notebooks"
DST_PATH="/Workspace/Shared/production-notebooks"

databricks workspace export-dir "$SRC_PATH" ./tmp-notebooks -p "$SRC_PROFILE"
databricks workspace import-dir ./tmp-notebooks "$DST_PATH" -p "$DST_PROFILE" --overwrite
rm -rf ./tmp-notebooks
echo "Promoted notebooks from dev → prod"
```

-----

## Design Patterns & Best Practices

### 1. Authentication

- **Prefer OAuth U2M** for interactive/developer use — tokens auto-refresh, no token rotation needed.
- **Prefer OAuth M2M (Service Principal)** for CI/CD — use environment variables, never hardcode credentials.
- **Avoid PATs** where possible; they don’t auto-rotate and are tied to individual users.
- Store secrets in your CI/CD vault (GitHub Secrets, Azure Key Vault, etc.), not in `.databrickscfg` for service accounts.

### 2. Bundle (DABs) Design

- **One bundle per bounded-context project** (ETL domain, ML project, etc.) — avoid mega-bundles.
- **Use `mode: development`** in dev targets to auto-prefix resources, pause schedules, and prevent conflicts between developers.
- **Use `mode: production`** in prod targets to enforce `run_as` service principal and validate resource ownership.
- **Use `include:` to split resources** across files (`resources/jobs.yml`, `resources/pipelines.yml`) for maintainability.
- **Pin CLI version** with `databricks_cli_version: ">=0.218.0"` in `databricks.yml`.
- **Commit `databricks.yml` to Git** — treat it as IaC. Never manually edit resources managed by a bundle.
- **Use `bundle deployment bind`** to adopt existing workspace resources into bundle management without re-creating them.

### 3. JSON Input Patterns

```bash
# Inline JSON (simple cases)
databricks clusters create --json '{"cluster_name": "test", ...}'

# File reference (complex configs, version-controlled)
databricks clusters create --json @cluster-config.json

# Pipe from command (dynamic)
echo '{"cluster_name": "dynamic"}' | databricks clusters create --json -

# Generate a skeleton by getting existing resource
databricks jobs get <id> --output json > job-template.json
# Edit, then re-create
databricks jobs create --json @job-template.json
```

### 4. Scripting & Automation

- **Always use `--output json`** when parsing CLI output in scripts.
- **Combine with `jq`** for field extraction and filtering.
- **Use `set -euo pipefail`** in bash scripts to fail fast on errors.
- **Check job run result state**, not just life cycle state — `TERMINATED` with `FAILED` result still means failure.
- **Use `--auto-approve`** flag on bundle deploy/destroy in unattended CI pipelines.

### 5. Multi-Environment Workflow

```
[Dev Workspace]                [Staging]                [Prod]
databricks bundle deploy -t dev → databricks bundle deploy -t staging → databricks bundle deploy -t prod
     ↑ developer machines            ↑ CI on PR merge                     ↑ CI on main branch tag
```

- Each target maps to a different workspace or uses different `root_path` in the same workspace.
- Use Git branch / tag events to gate which target receives a deployment.

### 6. Cluster Management

- **Pin long-lived clusters** with `databricks clusters pin` to prevent 30-day auto-deletion of terminated clusters.
- **Use cluster policies** to standardize node types, autoscaling limits, and cost tags — enforce through CLI or bundles.
- **Use instance pools** to reduce cluster startup time for short-lived job clusters.
- **Set `autotermination_minutes`** on interactive clusters to prevent cost runaway.

### 7. Unity Catalog Governance via CLI

- **Automate privilege grants** with `databricks grants update` in CI/CD pipelines post-deployment.
- **Validate external locations** before creating storage credentials to catch misconfigurations early.
- **Script catalog/schema creation** as part of workspace bootstrapping — keep it idempotent using `|| true` guards or bundle `presets`.

### 8. Secrets Best Practices

- **Scope per environment** — use `dev-secrets`, `prod-secrets` scopes rather than one shared scope.
- **Scope ACLs** — grant only `READ` to compute/job service principals; keep `MANAGE` for admins only.
- **Rotate secrets** by re-running `put-secret` — it overwrites the existing value without changing consumer configuration.
- **Never log secret values** — `get-secret` returns base64-encoded bytes; decoding in logs exposes the value.

-----

## Quick Reference: All Command Groups

|Group                |Category     |Key Operations                                                                    |
|---------------------|-------------|----------------------------------------------------------------------------------|
|`alerts`             |SQL          |create, delete, get, list, update                                                 |
|`apps`               |Apps         |create, delete, deploy, get, list, logs, run-local, start, stop                   |
|`artifact-allowlists`|Unity Catalog|get, update                                                                       |
|`auth`               |Auth         |describe, env, login, profiles, token                                             |
|`bundle`             |DABs         |deploy, destroy, generate, init, open, run, summary, validate                     |
|`catalogs`           |Unity Catalog|create, delete, get, list, update                                                 |
|`cluster-policies`   |Compute      |create, delete, edit, get, list + permissions                                     |
|`clusters`           |Compute      |create, delete, edit, events, get, list, pin, resize, restart, start + permissions|
|`connections`        |Unity Catalog|create, delete, get, list, update                                                 |
|`credentials`        |Unity Catalog|create, delete, get, list, update, validate                                       |
|`current-user`       |Auth         |me                                                                                |
|`data-quality`       |Unity Catalog|create/delete/get/list/update monitor+refresh                                     |
|`experiments`        |ML           |create, delete, get, list, log-metric, log-param, search-runs                     |
|`external-locations` |Unity Catalog|create, delete, get, list, update, validate                                       |
|`feature-engineering`|ML           |create, delete, get, list, update                                                 |
|`fs`                 |Workspace    |cat, cp, ls, mkdir, mv, rm                                                        |
|`git-credentials`    |Workspace    |create, delete, get, list, update                                                 |
|`global-init-scripts`|Compute      |create, delete, get, list, update                                                 |
|`grants`             |Unity Catalog|get, get-effective, update                                                        |
|`groups`             |IAM          |create, delete, get, list, patch, update                                          |
|`instance-pools`     |Compute      |create, delete, edit, get, list + permissions                                     |
|`instance-profiles`  |Compute (AWS)|add, edit, list, remove                                                           |
|`jobs`               |Jobs         |cancel-run, create, delete, get, list, repair-run, run-now, submit + permissions  |
|`libraries`          |Compute      |all-cluster-statuses, cluster-status, install, uninstall                          |
|`metastores`         |Unity Catalog|assign, create, delete, get, list, summary                                        |
|`model-registry`     |ML           |create-model, create-model-version, list, transition-stage                        |
|`permissions`        |IAM          |get, get-permission-levels, set, update                                           |
|`pipelines`          |Pipelines    |create, delete, get, list, start-update, stop, update + permissions               |
|`queries`            |SQL          |create, delete, get, list, restore, update                                        |
|`query-history`      |SQL          |list                                                                              |
|`repos`              |Workspace    |create, delete, get, list, update + permissions                                   |
|`schemas`            |Unity Catalog|create, delete, get, list, update                                                 |
|`secrets`            |Workspace    |create-scope, delete-scope, list-scopes, list-secrets, put-acl, put-secret        |
|`service-principals` |IAM          |create, delete, get, list, patch, update                                          |
|`serving-endpoints`  |ML Serving   |build-logs, create, delete, get, list, logs, query, update-config + permissions   |
|`storage-credentials`|Unity Catalog|create, delete, get, list, update                                                 |
|`tables`             |Unity Catalog|delete, get, list, list-summaries                                                 |
|`users`              |IAM          |create, delete, get, list, patch, update                                          |
|`volumes`            |Unity Catalog|create, delete, get, list, update                                                 |
|`warehouses`         |SQL          |create, delete, edit, get, list, start, stop + permissions                        |
|`workspace`          |Workspace    |delete, export, export-dir, import, import-dir, list, mkdirs + permissions        |
|`account`            |Account-level|workspaces, metastores, users, groups, budgets, billing, networking               |

-----

## Useful One-Liners

```bash
# Find all RUNNING clusters
databricks clusters list --output json | jq -r '.[] | select(.state=="RUNNING") | "\(.cluster_id)\t\(.cluster_name)"'

# Get all failed job runs in last N days
databricks jobs list-runs --output json | jq '.runs[] | select(.state.result_state=="FAILED") | {run_id,job_id,start_time}'

# List all tables in a catalog
databricks tables list-summaries --catalog-name main --output json | jq -r '.tables[].full_name'

# Export all notebooks from a folder
databricks workspace export-dir /Workspace/Shared ./backup-$(date +%Y%m%d)

# List all job schedules
databricks jobs list --output json | jq -r '.jobs[] | select(.settings.schedule != null) | "\(.job_id)\t\(.settings.name)\t\(.settings.schedule.quartz_cron_expression)"'

# Get cluster events filtered to errors
databricks clusters events <cluster-id> --output json | jq '.events[] | select(.type=="DRIVER_NOT_FOUND" or .type=="SPARK_EXCEPTION")'

# List all secrets scopes and their keys
for scope in $(databricks secrets list-scopes --output json | jq -r '.scopes[].name'); do
  echo "=== $scope ==="; databricks secrets list-secrets "$scope"
done

# Create a secret from a file
databricks secrets put-secret my-scope my-cert --bytes-value "$(base64 -w0 < cert.pem)"

# Bulk-delete old job runs (keep last 10)
databricks jobs list-runs --job-id <id> --output json | \
  jq -r '.runs[10:] | .[].run_id' | \
  xargs -I{} databricks jobs delete-run {}

# Check current user and workspace
databricks current-user me && databricks auth describe
```

-----

*Reference: Databricks CLI v0.205+ · https://docs.databricks.com/dev-tools/cli/ · https://github.com/databricks/cli*
