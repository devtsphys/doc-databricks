# Databricks references

# Databricks Utilities (dbutils) Cheat Sheet

## Overview

Databricks utilities (`dbutils`) provide a set of helper functions to perform common tasks in Databricks environments. These utilities are only available within the Databricks environment and make various operations easier for data engineers and data scientists.

## Basic Components

### Accessing dbutils

```python
# Access dbutils in Python
dbutils

# Display available utilities
dbutils.help()

# Get help on a specific utility
dbutils.fs.help()
```

### File System Operations (fs)

| Function | Description | Example |
|----------|-------------|---------|
| `fs.ls(path)` | List files in directory | `dbutils.fs.ls("/databricks-datasets/")` |
| `fs.mkdirs(path)` | Create directory recursively | `dbutils.fs.mkdirs("/tmp/new_directory/")` |
| `fs.rm(path, recurse)` | Remove file or directory | `dbutils.fs.rm("/tmp/old_file.txt")` |
| `fs.cp(from, to, recurse)` | Copy files | `dbutils.fs.cp("/source/file.csv", "/destination/file.csv")` |
| `fs.mv(from, to)` | Move files | `dbutils.fs.mv("/source/file.csv", "/destination/file.csv")` |
| `fs.head(file, maxBytes)` | View first few bytes of a file | `dbutils.fs.head("/tmp/data.csv", 1000)` |
| `fs.put(file, contents, overwrite)` | Write string to a file | `dbutils.fs.put("/tmp/test.txt", "Hello World", True)` |

### Widget Operations (widgets)

| Function | Description | Example |
|----------|-------------|---------|
| `widgets.text(name, defaultValue, label)` | Create text input widget | `dbutils.widgets.text("name", "default", "Enter name:")` |
| `widgets.dropdown(name, defaultValue, choices, label)` | Create dropdown widget | `dbutils.widgets.dropdown("state", "CA", ["CA", "NY", "TX"], "Select state:")` |
| `widgets.combobox(name, defaultValue, choices, label)` | Create combobox widget | `dbutils.widgets.combobox("fruit", "apple", ["apple", "orange", "banana"], "Select fruit:")` |
| `widgets.multiselect(name, defaultValue, choices, label)` | Create multiselect widget | `dbutils.widgets.multiselect("colors", "blue", ["red", "blue", "green"], "Select colors:")` |
| `widgets.get(name)` | Get widget value | `state = dbutils.widgets.get("state")` |
| `widgets.remove(name)` | Remove a widget | `dbutils.widgets.remove("name")` |
| `widgets.removeAll()` | Remove all widgets | `dbutils.widgets.removeAll()` |

## Advanced Components

### Secrets Management (secrets)

| Function | Description | Example |
|----------|-------------|---------|
| `secrets.listScopes()` | List all secret scopes | `dbutils.secrets.listScopes()` |
| `secrets.list(scope)` | List secrets in a scope | `dbutils.secrets.list("my_scope")` |
| `secrets.get(scope, key)` | Get a secret value | `password = dbutils.secrets.get("my_scope", "db_password")` |

### Notebook Operations (notebook)

| Function | Description | Example |
|----------|-------------|---------|
| `notebook.run(path, timeout, params)` | Run a notebook | `result = dbutils.notebook.run("/path/to/notebook", 600, {"param1": "value1"})` |
| `notebook.exit(value)` | Exit notebook with return value | `dbutils.notebook.exit("Completed successfully")` |

### Library Operations (library)

| Function | Description | Example |
|----------|-------------|---------|
| `library.installPyPI(package, repo, version)` | Install PyPI package | `dbutils.library.installPyPI("scikit-learn", version="1.0.2")` |
| `library.restartPython()` | Restart Python interpreter | `dbutils.library.restartPython()` |

### Jobs Operations (jobs)

| Function | Description | Example |
|----------|-------------|---------|
| `jobs.taskValues.help()` | Get help on task values | `dbutils.jobs.taskValues.help()` |
| `jobs.taskValues.get(taskKey, key)` | Get task value | `value = dbutils.jobs.taskValues.get("task1", "result")` |
| `jobs.taskValues.set(key, value)` | Set task value | `dbutils.jobs.taskValues.set("result", "success")` |

## Common Techniques and Best Practices

### Working with DBFS (Databricks File System)

```python
# Writing data to DBFS
df = spark.createDataFrame([(1, "John"), (2, "Jane")], ["id", "name"])
df.write.format("parquet").save("/dbfs/tmp/people.parquet")

# Reading data from DBFS
df = spark.read.format("parquet").load("/dbfs/tmp/people.parquet")

# Using dbutils to manage DBFS files
files = dbutils.fs.ls("/tmp/")
for file in files:
    print(file.name, file.size)
```

### Managing Secrets for Secure Access

```python
# Store connection credentials in Databricks secrets
# Access securely in your code
jdbc_url = "jdbc:postgresql://hostname:port/database"
connection_properties = {
    "user": "username",
    "password": dbutils.secrets.get(scope="my_scope", key="postgres_password"),
    "driver": "org.postgresql.Driver"
}

# Use in DataFrame operations
df = spark.read.jdbc(url=jdbc_url, table="table_name", properties=connection_properties)
```

### Parameterizing Notebooks with Widgets

```python
# Create widgets
dbutils.widgets.text("date", "", "Enter date (YYYY-MM-DD)")
dbutils.widgets.dropdown("environment", "dev", ["dev", "test", "prod"], "Select environment")

# Access widget values
date_param = dbutils.widgets.get("date")
env = dbutils.widgets.get("environment")

# Use values in data processing
if env == "prod":
    path = f"/data/prod/{date_param}/"
else:
    path = f"/data/{env}/{date_param}/"

df = spark.read.parquet(path)
```

### Notebook Workflows

```python
# Execute a child notebook and get its return value
# Parent notebook
result = dbutils.notebook.run("./data_preparation", 600, {"date": "2023-01-01"})
print(f"Data preparation completed with status: {result}")

# In child notebook (data_preparation)
# Process based on parameters
date = dbutils.widgets.get("date")
# ... processing logic ...
dbutils.notebook.exit("Success")
```

## Full Reference of dbutils Functions

| Category | Function | Description |
|----------|----------|-------------|
| **fs** | `cp(from, to, recurse=False)` | Copy files |
| | `head(file, maxBytes=65536)` | View beginning of file |
| | `ls(path)` | List directory contents |
| | `mkdirs(path)` | Create directories recursively |
| | `mv(from, to)` | Move files |
| | `put(file, contents, overwrite=False)` | Write string to file |
| | `rm(path, recurse=False)` | Delete file or directory |
| | `mount(source, mountPoint, extraConfigs)` | Mount storage |
| | `unmount(mountPoint)` | Unmount storage |
| | `refreshMounts()` | Refresh all mounts |
| | `mounts()` | List all mounts |
| **widgets** | `combobox(name, defaultValue, choices, label)` | Create combobox widget |
| | `dropdown(name, defaultValue, choices, label)` | Create dropdown widget |
| | `get(name)` | Get widget value |
| | `multiselect(name, defaultValue, choices, label)` | Create multiselect widget |
| | `remove(name)` | Remove widget |
| | `removeAll()` | Remove all widgets |
| | `text(name, defaultValue, label)` | Create text widget |
| **notebook** | `exit(value)` | Exit notebook with value |
| | `run(path, timeout, params)` | Run notebook |
| **secrets** | `get(scope, key)` | Get secret value |
| | `list(scope)` | List secrets in scope |
| | `listScopes()` | List all secret scopes |
| **library** | `installPyPI(package, repo, version)` | Install PyPI package |
| | `restartPython()` | Restart Python interpreter |
| **jobs** | `taskValues.get(taskKey, key)` | Get task value |
| | `taskValues.set(key, value)` | Set task value |

## Advanced Mounting Techniques

### Mount Azure Blob Storage

```python
configs = {
  "fs.azure.account.key.<storage-account-name>.blob.core.windows.net": dbutils.secrets.get(scope="<scope-name>", key="<key-name>")
}

dbutils.fs.mount(
  source = "wasbs://<container-name>@<storage-account-name>.blob.core.windows.net",
  mount_point = "/mnt/<mount-name>",
  extra_configs = configs
)
```

### Mount AWS S3

```python
configs = {
  "fs.s3a.access.key": dbutils.secrets.get(scope="<scope-name>", key="<access-key-name>"),
  "fs.s3a.secret.key": dbutils.secrets.get(scope="<scope-name>", key="<secret-key-name>")
}

dbutils.fs.mount(
  source = "s3a://<bucket-name>",
  mount_point = "/mnt/<mount-name>",
  extra_configs = configs
)
```

### Mount Google Cloud Storage

```python
configs = {
  "fs.gs.auth.service.account.email": "<service-account-email>",
  "fs.gs.auth.service.account.private.key": dbutils.secrets.get(scope="<scope-name>", key="<key-name>"),
  "fs.gs.project.id": "<project-id>"
}

dbutils.fs.mount(
  source = "gs://<bucket-name>",
  mount_point = "/mnt/<mount-name>",
  extra_configs = configs
)
```

## Troubleshooting dbutils

### Common Issues and Solutions

1. **Permission Errors**
   ```
   Error: java.io.IOException: Permission denied
   ```
   - Check access permissions on storage account
   - Verify secret scopes and values are correct
   - Ensure cluster has correct IAM roles

2. **Mount Points Already Exists**
   ```
   Error: java.lang.RuntimeException: Mount point already exists
   ```
   - Unmount first: `dbutils.fs.unmount("/mnt/mount-name")`
   - Or use `dbutils.fs.refreshMounts()`

3. **Timeout Issues**
   ```
   Error: java.util.concurrent.TimeoutException
   ```
   - Increase timeout value in `notebook.run()`
   - Check network connectivity
   - Check resource constraints

4. **Widget Value Errors**
   ```
   Error: java.lang.IllegalArgumentException: Widget not found
   ```
   - Check widget name for typos
   - Ensure widget is created before accessing it
   - Remember widgets are notebook-specific


# Databricks CLI Command Reference

## Installation & Setup

```bash
# Installation
pip install databricks-cli

# Configure with access token
databricks configure --token
# Enter your workspace URL and access token when prompted

# Configure with username/password
databricks configure
# Enter your workspace URL, username, and password when prompted

# Configure with profile
databricks configure --profile my-profile --token
```

## Core Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks -h` | Show help for CLI | `databricks -h` |
| `databricks --version` | Show version | `databricks --version` |
| `databricks configure` | Configure authentication | `databricks configure --token --profile prod` |
| `databricks configure --help` | Show configure options | `databricks configure --help` |

## Workspace Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks workspace ls` | List workspace contents | `databricks workspace ls /Users/me` |
| `databricks workspace mkdirs` | Create directory | `databricks workspace mkdirs /Users/me/project` |
| `databricks workspace import` | Import file to workspace | `databricks workspace import example.py /Users/me/example -l PYTHON` |
| `databricks workspace export` | Export from workspace | `databricks workspace export /Users/me/notebook.py notebook.py` |
| `databricks workspace export-dir` | Export directory contents | `databricks workspace export-dir /Users/me/project ./local-dir` |
| `databricks workspace import-dir` | Import directory | `databricks workspace import-dir ./local-dir /Users/me/project` |
| `databricks workspace delete` | Delete workspace object | `databricks workspace delete /Users/me/old-notebook` |
| `databricks workspace get-status` | Get workspace item status | `databricks workspace get-status /Users/me/notebook` |

## DBFS Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks fs ls` | List DBFS contents | `databricks fs ls dbfs:/FileStore/` |
| `databricks fs mkdirs` | Create DBFS directory | `databricks fs mkdirs dbfs:/FileStore/my-dir` |
| `databricks fs cp` | Copy to/from DBFS | `databricks fs cp ./local-file dbfs:/FileStore/my-file` |
| `databricks fs mv` | Move DBFS file | `databricks fs mv dbfs:/FileStore/old dbfs:/FileStore/new` |
| `databricks fs rm` | Remove from DBFS | `databricks fs rm dbfs:/FileStore/old-file` |
| `databricks fs cat` | View DBFS file content | `databricks fs cat dbfs:/FileStore/my-file` |
| `databricks fs head` | View beginning of file | `databricks fs head -n 10 dbfs:/FileStore/my-file` |
| `databricks fs put` | Upload local file to DBFS | `databricks fs put ./local-file dbfs:/FileStore/target-file` |

## Cluster Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks clusters list` | List all clusters | `databricks clusters list` |
| `databricks clusters create` | Create a cluster | `databricks clusters create --json-file cluster-config.json` |
| `databricks clusters edit` | Edit cluster configuration | `databricks clusters edit 0123-456789-abcdefg --json-file new-config.json` |
| `databricks clusters start` | Start a cluster | `databricks clusters start 0123-456789-abcdefg` |
| `databricks clusters restart` | Restart a cluster | `databricks clusters restart 0123-456789-abcdefg` |
| `databricks clusters terminate` | Stop a cluster | `databricks clusters terminate 0123-456789-abcdefg` |
| `databricks clusters permanent-delete` | Delete a cluster | `databricks clusters permanent-delete 0123-456789-abcdefg` |
| `databricks clusters get` | Get cluster info | `databricks clusters get 0123-456789-abcdefg` |
| `databricks clusters spark-versions` | List spark versions | `databricks clusters spark-versions` |
| `databricks clusters node-types` | List node types | `databricks clusters node-types` |

## Jobs Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks jobs list` | List all jobs | `databricks jobs list` |
| `databricks jobs create` | Create a job | `databricks jobs create --json-file job-config.json` |
| `databricks jobs get` | Get job details | `databricks jobs get 12345` |
| `databricks jobs delete` | Delete a job | `databricks jobs delete 12345` |
| `databricks jobs reset` | Reset a job | `databricks jobs reset 12345 --json-file new-config.json` |
| `databricks jobs run-now` | Run a job now | `databricks jobs run-now 12345` |
| `databricks jobs runs list` | List job runs | `databricks jobs runs list --job-id 12345` |
| `databricks jobs runs get` | Get run details | `databricks jobs runs get 67890` |
| `databricks jobs runs cancel` | Cancel a run | `databricks jobs runs cancel 67890` |
| `databricks jobs runs submit` | Submit one-time run | `databricks jobs runs submit --json-file run-config.json` |

## Instance Pool Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks instance-pools list` | List all pools | `databricks instance-pools list` |
| `databricks instance-pools create` | Create pool | `databricks instance-pools create --json-file pool-config.json` |
| `databricks instance-pools edit` | Edit pool | `databricks instance-pools edit pool-id --json-file new-config.json` |
| `databricks instance-pools get` | Get pool info | `databricks instance-pools get pool-id` |
| `databricks instance-pools delete` | Delete pool | `databricks instance-pools delete pool-id` |

## Library Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks libraries list` | List cluster libraries | `databricks libraries list --cluster-id 0123-456789-abcdefg` |
| `databricks libraries install` | Install libraries | `databricks libraries install --cluster-id 0123-456789-abcdefg --jar dbfs:/FileStore/my.jar` |
| `databricks libraries uninstall` | Uninstall libraries | `databricks libraries uninstall --cluster-id 0123-456789-abcdefg --jar dbfs:/FileStore/my.jar` |
| `databricks libraries all-cluster-statuses` | Get all library statuses | `databricks libraries all-cluster-statuses` |

## Secret Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks secrets list-scopes` | List secret scopes | `databricks secrets list-scopes` |
| `databricks secrets create-scope` | Create scope | `databricks secrets create-scope --scope my-scope` |
| `databricks secrets delete-scope` | Delete scope | `databricks secrets delete-scope --scope my-scope` |
| `databricks secrets list` | List secrets in scope | `databricks secrets list --scope my-scope` |
| `databricks secrets put` | Add/update secret | `databricks secrets put --scope my-scope --key my-key --string-value "password"` |
| `databricks secrets delete` | Delete secret | `databricks secrets delete --scope my-scope --key my-key` |

## Token Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks tokens list` | List tokens | `databricks tokens list` |
| `databricks tokens create` | Create token | `databricks tokens create --comment "Jenkins automation" --lifetime-seconds 7776000` |
| `databricks tokens revoke` | Revoke token | `databricks tokens revoke --token-id abcd1234` |

## Groups and Users Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks groups list` | List all groups | `databricks groups list` |
| `databricks groups create` | Create group | `databricks groups create --group-name engineers` |
| `databricks groups get` | Get group details | `databricks groups get --group-name engineers` |
| `databricks groups delete` | Delete group | `databricks groups delete --group-name old-team` |
| `databricks groups list-members` | List group members | `databricks groups list-members --group-name engineers` |
| `databricks groups add-member` | Add group member | `databricks groups add-member --parent-name engineers --user-name john.doe@example.com` |
| `databricks groups remove-member` | Remove member | `databricks groups remove-member --parent-name engineers --user-name john.doe@example.com` |
| `databricks users list` | List users | `databricks users list` |
| `databricks users create` | Create user | `databricks users create --user-name john.doe@example.com` |
| `databricks users delete` | Delete user | `databricks users delete --user-name john.doe@example.com` |

## Workspace Conf Commands

| Command | Description | Example |
|---------|-------------|---------|
| `databricks workspace-conf get-status` | Get conf settings | `databricks workspace-conf get-status` |
| `databricks workspace-conf set-status` | Update settings | `databricks workspace-conf set-status --json-file settings.json` |

## Common Configuration Patterns

### Cluster Configuration Example

```json
{
  "cluster_name": "my-cluster",
  "spark_version": "7.3.x-scala2.12",
  "node_type_id": "i3.xlarge",
  "spark_conf": {
    "spark.speculation": true
  },
  "num_workers": 2,
  "autotermination_minutes": 30
}
```

### Job Configuration Example

```json
{
  "name": "Daily ETL Job",
  "new_cluster": {
    "spark_version": "7.3.x-scala2.12",
    "node_type_id": "i3.xlarge",
    "num_workers": 2
  },
  "notebook_task": {
    "notebook_path": "/Users/me/my-notebook"
  },
  "schedule": {
    "quartz_cron_expression": "0 0 7 * * ?",
    "timezone_id": "America/Los_Angeles"
  }
}
```

## Tips & Tricks

1. **Using Profiles**:
   ```bash
   databricks --profile prod fs ls dbfs:/
   ```

2. **Format Output as JSON**:
   ```bash
   databricks clusters list --output JSON
   ```

3. **Save Output to File**:
   ```bash
   databricks clusters list --output JSON > clusters.json
   ```

4. **Use Environment Variables for Authentication**:
   ```bash
   export DATABRICKS_HOST=https://your-workspace.cloud.databricks.com
   export DATABRICKS_TOKEN=your-token
   databricks fs ls dbfs:/
   ```

5. **Debug API Calls**:
   ```bash
   databricks --debug fs ls dbfs:/
   ```

6. **Work with Job Runs**:
   ```bash
   # Get the 10 most recent job runs
   databricks jobs runs list --limit 10
   
   # Get details about a specific run
   databricks jobs runs get --run-id 12345
   ```

7. **Export/Import Workspace Recursively**:
   ```bash
   # Export entire directory
   databricks workspace export-dir /Users/me/project ./local-backup
   
   # Import back
   databricks workspace import-dir ./local-backup /Users/me/project-restored
   ```

# Databricks Asset Bundles Reference Card

## Overview

Databricks Asset Bundles (DABs) provide a declarative way to define, deploy, and manage Databricks assets as code. They enable version control, CI/CD integration, and environment management for your data and ML workflows.

## Core Concepts

### Bundle Structure

```
my-bundle/
├── databricks.yml          # Main configuration file
├── src/                   # Source code directory
│   ├── notebook.py
│   └── job_script.py
├── resources/             # Resource definitions
│   ├── jobs.yml
│   └── workflows.yml
└── environments/          # Environment-specific configs
    ├── dev.yml
    └── prod.yml
```

### Key Components

- **Bundle Configuration**: `databricks.yml` - defines bundle metadata and structure
- **Resources**: Jobs, workflows, clusters, notebooks, etc.
- **Targets**: Environment-specific configurations (dev, staging, prod)
- **Variables**: Parameterization for different environments
- **Artifacts**: Code files that get uploaded and deployed

## Basic Configuration

### databricks.yml Structure

```yaml
bundle:
  name: my-data-pipeline
  
include:
  - resources/*.yml
  
variables:
  catalog:
    description: "Unity Catalog name"
    default: "dev_catalog"
  
targets:
  dev:
    variables:
      catalog: "dev_catalog"
    workspace:
      host: "https://your-workspace.databricks.com"
      
  prod:
    variables:
      catalog: "prod_catalog"
    workspace:
      host: "https://prod-workspace.databricks.com"
```

## Resource Types & Configuration

### Jobs

```yaml
resources:
  jobs:
    etl_pipeline:
      name: "ETL Pipeline - ${var.environment}"
      job_clusters:
        - job_cluster_key: "main_cluster"
          new_cluster:
            spark_version: "13.3.x-scala2.12"
            node_type_id: "i3.xlarge"
            num_workers: 2
            
      tasks:
        - task_key: "extract"
          job_cluster_key: "main_cluster"
          notebook_task:
            notebook_path: "./src/extract_data.py"
            base_parameters:
              catalog: "${var.catalog}"
              
        - task_key: "transform"
          depends_on:
            - task_key: "extract"
          job_cluster_key: "main_cluster"
          spark_python_task:
            python_file: "./src/transform.py"
            parameters: ["--catalog", "${var.catalog}"]
```

### Workflows (Delta Live Tables)

```yaml
resources:
  pipelines:
    dlt_pipeline:
      name: "DLT Pipeline - ${var.environment}"
      catalog: "${var.catalog}"
      target: "${var.schema}"
      libraries:
        - notebook:
            path: "./src/dlt_bronze.py"
        - notebook:
            path: "./src/dlt_silver.py"
      configuration:
        "pipeline.environment": "${var.environment}"
      clusters:
        - label: "default"
          num_workers: 2
          node_type_id: "i3.xlarge"
```

### Model Serving Endpoints

```yaml
resources:
  model_serving_endpoints:
    recommendation_model:
      name: "recommendation-model-${var.environment}"
      config:
        served_models:
          - model_name: "recommendation_model"
            model_version: "1"
            workload_size: "Small"
            scale_to_zero_enabled: true
```

### Experiments

```yaml
resources:
  experiments:
    ml_experiment:
      name: "/Shared/ml-experiment-${var.environment}"
      artifact_location: "s3://my-bucket/experiments/${var.environment}/"
```

### Clusters

```yaml
resources:
  clusters:
    analytics_cluster:
      cluster_name: "Analytics Cluster - ${var.environment}"
      spark_version: "13.3.x-scala2.12"
      node_type_id: "i3.xlarge"
      num_workers: 2
      autotermination_minutes: 30
      spark_conf:
        "spark.sql.adaptive.enabled": "true"
        "spark.sql.adaptive.coalescePartitions.enabled": "true"
```

## Advanced Components

### Variables and Templating

```yaml
variables:
  environment:
    description: "Environment name"
    type: "string"
    
  worker_count:
    description: "Number of workers"
    type: "number"
    default: 2
    
  feature_flags:
    description: "Feature toggles"
    type: "complex"
    default:
      enable_monitoring: true
      use_photon: false

# Usage in resources
resources:
  jobs:
    my_job:
      name: "Job-${var.environment}"
      job_clusters:
        - new_cluster:
            num_workers: ${var.worker_count}
            runtime_engine: |
              ${var.feature_flags.use_photon ? "PHOTON" : "STANDARD"}
```

### Conditional Resources

```yaml
resources:
  jobs:
    # Only create in production
    prod_job:
      name: "Production Only Job"
      # ... job configuration
      
targets:
  dev:
    resources:
      jobs:
        prod_job: null  # Exclude from dev
        
  prod:
    # Will include prod_job
```

### Custom Artifacts and Libraries

```yaml
artifacts:
  my_wheel:
    type: "whl"
    path: "./dist/my_package-0.1.0-py3-none-any.whl"
    
  custom_jar:
    type: "jar"
    path: "./target/my-library.jar"

resources:
  jobs:
    job_with_custom_libs:
      name: "Job with Custom Libraries"
      tasks:
        - task_key: "main"
          libraries:
            - whl: "${artifacts.my_wheel.path}"
            - jar: "${artifacts.custom_jar.path}"
```

## Functions and Elements Reference

|Element      |Type  |Description                   |Example                          |
|-------------|------|------------------------------|---------------------------------|
|`bundle.name`|string|Bundle identifier             |`my-data-pipeline`               |
|`bundle.git` |object|Git repository info           |`url`, `branch`, `commit`        |
|`include`    |array |Include additional YAML files |`- resources/*.yml`              |
|`variables`  |object|Define parameterizable values |See variables section            |
|`targets`    |object|Environment-specific configs  |`dev`, `staging`, `prod`         |
|`workspace`  |object|Databricks workspace config   |`host`, `profile`, `auth_type`   |
|`artifacts`  |object|Build artifacts to upload     |`whl`, `jar`, `file`             |
|`resources`  |object|Databricks resources to deploy|Jobs, clusters, experiments, etc.|

### Resource-Specific Elements

#### Jobs

|Element              |Description           |Example                       |
|---------------------|----------------------|------------------------------|
|`job_clusters`       |Cluster configurations|Reusable cluster specs        |
|`tasks`              |Job task definitions  |Notebook, Python, JAR tasks   |
|`schedule`           |Job scheduling        |Cron expressions, dependencies|
|`email_notifications`|Alert configurations  |Success/failure notifications |
|`timeout_seconds`    |Job timeout           |Maximum execution time        |
|`max_concurrent_runs`|Concurrency limit     |Parallel execution control    |

#### Delta Live Tables

|Element        |Description            |Example                 |
|---------------|-----------------------|------------------------|
|`libraries`    |DLT notebook/file paths|Source definitions      |
|`configuration`|Pipeline settings      |Environment variables   |
|`clusters`     |Compute configurations |Worker specifications   |
|`continuous`   |Streaming mode toggle  |`true` for continuous   |
|`development`  |Development mode       |Enhanced error reporting|

#### Model Serving

|Element                |Description            |Example                    |
|-----------------------|-----------------------|---------------------------|
|`served_models`        |Model versions to serve|Model name, version, config|
|`traffic_config`       |Traffic routing        |A/B testing, canary        |
|`workload_size`        |Compute size           |Small, Medium, Large       |
|`scale_to_zero_enabled`|Auto-scaling           |Cost optimization          |

## CLI Commands

### Basic Operations

```bash
# Initialize new bundle
databricks bundle init

# Validate bundle configuration
databricks bundle validate

# Deploy to target environment
databricks bundle deploy --target dev

# Run a job from bundle
databricks bundle run my_job --target dev

# Destroy bundle resources
databricks bundle destroy --target dev

# Generate bundle documentation
databricks bundle generate docs
```

### Advanced Commands

```bash
# Deploy with variable override
databricks bundle deploy --target prod --var="worker_count=10"

# Dry-run deployment
databricks bundle deploy --target dev --dry-run

# Deploy specific resources only
databricks bundle deploy --target dev --resource jobs.etl_pipeline

# Validate with specific target
databricks bundle validate --target prod

# View deployed resources
databricks bundle summary --target dev
```

## Best Practices

### Project Structure

```
project/
├── databricks.yml
├── src/
│   ├── common/           # Shared utilities
│   ├── jobs/            # Job-specific code
│   ├── dlt/             # DLT pipeline code
│   └── ml/              # ML training code
├── resources/
│   ├── jobs.yml
│   ├── workflows.yml
│   └── ml.yml
├── environments/
│   ├── dev.yml
│   ├── staging.yml
│   └── prod.yml
├── tests/               # Unit tests
└── docs/               # Documentation
```

### Environment Management

```yaml
# Use environment-specific variables
targets:
  dev:
    variables:
      catalog: "dev_catalog"
      cluster_size: "small"
      worker_count: 1
      
  prod:
    variables:
      catalog: "prod_catalog"
      cluster_size: "large"
      worker_count: 10
    workspace:
      host: "https://prod.databricks.com"
```

### Security and Secrets

```yaml
# Reference secrets in jobs
resources:
  jobs:
    secure_job:
      tasks:
        - task_key: "main"
          notebook_task:
            notebook_path: "./src/secure_notebook.py"
            base_parameters:
              api_key: "{{secrets/my-scope/api-key}}"
```

### Resource Naming Conventions

```yaml
# Consistent naming with environment prefixes
resources:
  jobs:
    data_ingestion_job:
      name: "${var.environment}_data_ingestion"
      
  clusters:
    analytics_cluster:
      cluster_name: "${var.environment}_analytics_cluster"
```

## Troubleshooting

### Common Issues

1. **Validation Errors**: Check YAML syntax and required fields
1. **Permission Errors**: Verify workspace permissions and authentication
1. **Resource Conflicts**: Ensure unique resource names across environments
1. **Path Issues**: Use relative paths from bundle root
1. **Variable Resolution**: Check variable scoping and default values

### Debug Commands

```bash
# Verbose output
databricks bundle deploy --target dev --verbose

# Check bundle configuration
databricks bundle validate --target dev --output json

# View generated Terraform
databricks bundle deploy --target dev --dry-run --output terraform
```

## Advanced Techniques

### Multi-Environment Deployment

```yaml
# Use matrix deployments for multiple environments
targets:
  dev:
    variables: { environment: "dev", scale: 1 }
  staging:
    variables: { environment: "staging", scale: 3 }
  prod:
    variables: { environment: "prod", scale: 10 }
```

### Dynamic Resource Generation

```yaml
# Generate multiple similar jobs
variables:
  regions:
    default: ["us-east-1", "us-west-2", "eu-west-1"]

# Use loops in templates (advanced usage)
resources:
  jobs:
    # This would require custom templating logic
    data_sync_${region}:
      name: "Data Sync - ${region}"
      # ... configuration per region
```

### Integration with CI/CD

```yaml
# GitHub Actions example
name: Deploy Bundle
on:
  push:
    branches: [main]
    
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Production
        run: |
          databricks bundle deploy --target prod
        env:
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
```

# Azure Databricks File Handling & Mounting Reference Card

## Table of Contents

1. [File System Overview](#file-system-overview)
1. [DBFS (Databricks File System)](#dbfs-databricks-file-system)
1. [Azure Storage Integration](#azure-storage-integration)
1. [Mounting Azure Storage](#mounting-azure-storage)
1. [File Operations](#file-operations)
1. [Best Practices](#best-practices)
1. [Troubleshooting](#troubleshooting)

## File System Overview

### Available File Systems

- **DBFS** - Databricks File System (default)
- **Azure Blob Storage** - Object storage service
- **Azure Data Lake Storage Gen2** - Hierarchical namespace storage
- **Azure Files** - SMB file shares

### Path Formats

```python
# DBFS paths
dbfs:/path/to/file
/dbfs/path/to/file

# Mounted storage paths
/mnt/storage-name/path/to/file

# Direct Azure storage paths
abfss://container@storageaccount.dfs.core.windows.net/path/to/file
wasbs://container@storageaccount.blob.core.windows.net/path/to/file
```

## DBFS (Databricks File System)

### Basic DBFS Operations

#### List Files

```python
# Using dbutils
dbutils.fs.ls("/")
dbutils.fs.ls("dbfs:/FileStore/")

# Using Python os module
import os
os.listdir("/dbfs/")
```

#### Create Directory

```python
dbutils.fs.mkdirs("/mnt/data/new_folder")
```

#### Copy Files

```python
# Copy single file
dbutils.fs.cp("source_path", "destination_path")

# Copy recursively
dbutils.fs.cp("source_folder", "destination_folder", recurse=True)
```

#### Remove Files

```python
# Remove single file
dbutils.fs.rm("path/to/file")

# Remove directory recursively
dbutils.fs.rm("path/to/directory", recurse=True)
```

#### File Information

```python
# Get file info
file_info = dbutils.fs.ls("path/to/file")[0]
print(f"Name: {file_info.name}")
print(f"Size: {file_info.size}")
print(f"Path: {file_info.path}")
```

## Azure Storage Integration

### Authentication Methods

#### 1. Service Principal (Recommended for Production)

```python
# Set service principal credentials
spark.conf.set("fs.azure.account.auth.type.<storage-account>.dfs.core.windows.net", "OAuth")
spark.conf.set("fs.azure.account.oauth.provider.type.<storage-account>.dfs.core.windows.net", "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider")
spark.conf.set("fs.azure.account.oauth2.client.id.<storage-account>.dfs.core.windows.net", "<application-id>")
spark.conf.set("fs.azure.account.oauth2.client.secret.<storage-account>.dfs.core.windows.net", "<client-secret>")
spark.conf.set("fs.azure.account.oauth2.client.endpoint.<storage-account>.dfs.core.windows.net", "https://login.microsoftonline.com/<tenant-id>/oauth2/token")
```

#### 2. Access Key

```python
spark.conf.set(
    "fs.azure.account.key.<storage-account>.dfs.core.windows.net",
    "<access-key>"
)
```

#### 3. SAS Token

```python
spark.conf.set(
    "fs.azure.sas.<container>.<storage-account>.dfs.core.windows.net",
    "<sas-token>"
)
```

## Mounting Azure Storage

### Mount Azure Data Lake Storage Gen2

#### Using Service Principal

```python
def mount_adls_gen2(storage_account, container, mount_point, client_id, client_secret, tenant_id):
    """
    Mount ADLS Gen2 container using service principal authentication
    """
    try:
        # Check if already mounted
        if any(mount.mountPoint == mount_point for mount in dbutils.fs.mounts()):
            print(f"Mount point {mount_point} already exists")
            return
        
        # Configuration for mounting
        configs = {
            "fs.azure.account.auth.type": "OAuth",
            "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
            "fs.azure.account.oauth2.client.id": client_id,
            "fs.azure.account.oauth2.client.secret": client_secret,
            "fs.azure.account.oauth2.client.endpoint": f"https://login.microsoftonline.com/{tenant_id}/oauth2/token"
        }
        
        # Mount the storage
        dbutils.fs.mount(
            source=f"abfss://{container}@{storage_account}.dfs.core.windows.net/",
            mount_point=mount_point,
            extra_configs=configs
        )
        
        print(f"Successfully mounted {container} to {mount_point}")
        
    except Exception as e:
        print(f"Error mounting storage: {str(e)}")

# Example usage
mount_adls_gen2(
    storage_account="mystorageaccount",
    container="mycontainer",
    mount_point="/mnt/datalake",
    client_id="your-client-id",
    client_secret="your-client-secret",
    tenant_id="your-tenant-id"
)
```

#### Using Access Key

```python
def mount_with_access_key(storage_account, container, mount_point, access_key):
    """
    Mount storage using access key
    """
    try:
        if any(mount.mountPoint == mount_point for mount in dbutils.fs.mounts()):
            print(f"Mount point {mount_point} already exists")
            return
            
        dbutils.fs.mount(
            source=f"abfss://{container}@{storage_account}.dfs.core.windows.net/",
            mount_point=mount_point,
            extra_configs={f"fs.azure.account.key.{storage_account}.dfs.core.windows.net": access_key}
        )
        
        print(f"Successfully mounted {container} to {mount_point}")
        
    except Exception as e:
        print(f"Error mounting storage: {str(e)}")
```

### Mount Azure Blob Storage

```python
def mount_blob_storage(storage_account, container, mount_point, access_key):
    """
    Mount Azure Blob Storage
    """
    try:
        if any(mount.mountPoint == mount_point for mount in dbutils.fs.mounts()):
            print(f"Mount point {mount_point} already exists")
            return
            
        dbutils.fs.mount(
            source=f"wasbs://{container}@{storage_account}.blob.core.windows.net",
            mount_point=mount_point,
            extra_configs={f"fs.azure.account.key.{storage_account}.blob.core.windows.net": access_key}
        )
        
        print(f"Successfully mounted blob container {container} to {mount_point}")
        
    except Exception as e:
        print(f"Error mounting blob storage: {str(e)}")
```

### Mount Management

#### List All Mounts

```python
# Display all current mounts
display(dbutils.fs.mounts())

# Get mount information programmatically
mounts = dbutils.fs.mounts()
for mount in mounts:
    print(f"Mount Point: {mount.mountPoint}")
    print(f"Source: {mount.source}")
    print("---")
```

#### Unmount Storage

```python
def unmount_storage(mount_point):
    """
    Safely unmount storage
    """
    try:
        dbutils.fs.unmount(mount_point)
        print(f"Successfully unmounted {mount_point}")
    except Exception as e:
        print(f"Error unmounting {mount_point}: {str(e)}")

# Example
unmount_storage("/mnt/datalake")
```

#### Refresh Mount

```python
# Refresh mount if needed
dbutils.fs.refreshMounts()
```

## File Operations

### Reading Files

#### Text Files

```python
# Read text file
with open("/dbfs/mnt/datalake/file.txt", "r") as f:
    content = f.read()

# Using dbutils
content = dbutils.fs.head("dbfs:/mnt/datalake/file.txt")
```

#### CSV Files with Spark

```python
# Read CSV file
df = spark.read.option("header", "true").option("inferSchema", "true").csv("/mnt/datalake/data.csv")

# Read with specific schema
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("id", IntegerType(), True),
    StructField("name", StringType(), True),
    StructField("value", StringType(), True)
])

df = spark.read.schema(schema).option("header", "true").csv("/mnt/datalake/data.csv")
```

#### Parquet Files

```python
# Read Parquet file
df = spark.read.parquet("/mnt/datalake/data.parquet")

# Read multiple Parquet files
df = spark.read.parquet("/mnt/datalake/year=2023/month=*/")
```

#### JSON Files

```python
# Read JSON file
df = spark.read.json("/mnt/datalake/data.json")

# Read multiline JSON
df = spark.read.option("multiline", "true").json("/mnt/datalake/data.json")
```

### Writing Files

#### Write DataFrame to Different Formats

```python
# Write as Parquet (recommended for analytics)
df.write.mode("overwrite").parquet("/mnt/datalake/output/data.parquet")

# Write as CSV
df.write.mode("overwrite").option("header", "true").csv("/mnt/datalake/output/data.csv")

# Write as JSON
df.write.mode("overwrite").json("/mnt/datalake/output/data.json")

# Write with partitioning
df.write.mode("overwrite").partitionBy("year", "month").parquet("/mnt/datalake/partitioned_data/")
```

#### Write Modes

```python
# Overwrite existing data
df.write.mode("overwrite").parquet("/mnt/datalake/data/")

# Append to existing data
df.write.mode("append").parquet("/mnt/datalake/data/")

# Error if data exists (default)
df.write.mode("error").parquet("/mnt/datalake/data/")

# Ignore if data exists
df.write.mode("ignore").parquet("/mnt/datalake/data/")
```

### File Upload/Download

#### Upload Files

```python
# Upload file to DBFS
dbutils.fs.put("/mnt/datalake/uploaded_file.txt", "File content here", overwrite=True)

# Copy local file to mounted storage
dbutils.fs.cp("file:/databricks/driver/local_file.txt", "/mnt/datalake/")
```

#### Download Files

```python
# Copy from mounted storage to local
dbutils.fs.cp("/mnt/datalake/file.txt", "file:/databricks/driver/downloaded_file.txt")
```

## Best Practices

### Security

1. **Use Service Principal Authentication** for production environments
1. **Store secrets in Azure Key Vault** and reference them in Databricks
1. **Use managed identity** when possible
1. **Rotate access keys** regularly
1. **Apply least privilege principle** for storage access

### Performance

1. **Use appropriate file formats**:
- Parquet for analytics workloads
- Delta Lake for ACID transactions
- ORC for Hive compatibility
1. **Optimize file sizes**:
- Target 100MB-1GB per file
- Avoid small files (< 10MB)
- Use `coalesce()` or `repartition()` when writing
1. **Use partitioning** for large datasets:

```python
# Good partitioning strategy
df.write.partitionBy("year", "month").parquet("/mnt/datalake/partitioned/")

# Avoid over-partitioning (too many small partitions)
# Avoid under-partitioning (too few large partitions)
```

### Mounting Strategy

```python
# Centralized mount function
def setup_storage_mounts():
    """
    Set up all required storage mounts for the workspace
    """
    mounts_config = [
        {
            "storage_account": "rawdata",
            "container": "landing",
            "mount_point": "/mnt/raw"
        },
        {
            "storage_account": "processeddata", 
            "container": "curated",
            "mount_point": "/mnt/processed"
        }
    ]
    
    for config in mounts_config:
        mount_adls_gen2(**config, 
                       client_id=dbutils.secrets.get("keyvault", "client-id"),
                       client_secret=dbutils.secrets.get("keyvault", "client-secret"),
                       tenant_id=dbutils.secrets.get("keyvault", "tenant-id"))

# Call at the beginning of notebooks
setup_storage_mounts()
```

### Error Handling

```python
def safe_file_operation(operation, *args, **kwargs):
    """
    Wrapper for safe file operations with retry logic
    """
    import time
    max_retries = 3
    retry_delay = 5
    
    for attempt in range(max_retries):
        try:
            return operation(*args, **kwargs)
        except Exception as e:
            if attempt == max_retries - 1:
                raise e
            print(f"Attempt {attempt + 1} failed: {str(e)}")
            print(f"Retrying in {retry_delay} seconds...")
            time.sleep(retry_delay)

# Example usage
def read_with_retry(path):
    return safe_file_operation(spark.read.parquet, path)

df = read_with_retry("/mnt/datalake/data.parquet")
```

## Troubleshooting

### Common Issues and Solutions

#### Mount Failures

```python
# Check if mount point is already in use
existing_mounts = [mount.mountPoint for mount in dbutils.fs.mounts()]
if "/mnt/datalake" in existing_mounts:
    dbutils.fs.unmount("/mnt/datalake")

# Verify credentials
try:
    dbutils.fs.ls("/mnt/datalake")
    print("Mount successful and accessible")
except Exception as e:
    print(f"Mount issue: {str(e)}")
```

#### Permission Issues

```python
# Test write permissions
try:
    dbutils.fs.put("/mnt/datalake/test_write.txt", "test content", overwrite=True)
    dbutils.fs.rm("/mnt/datalake/test_write.txt")
    print("Write permissions confirmed")
except Exception as e:
    print(f"Write permission issue: {str(e)}")
```

#### File Not Found Errors

```python
def file_exists(path):
    """
    Check if file or directory exists
    """
    try:
        dbutils.fs.ls(path)
        return True
    except:
        return False

# Usage
if file_exists("/mnt/datalake/data.parquet"):
    df = spark.read.parquet("/mnt/datalake/data.parquet")
else:
    print("File not found")
```

### Debugging Commands

```python
# Check Spark configuration
spark.conf.get("fs.azure.account.auth.type")

# List all Spark configurations
configs = spark.sparkContext.getConf().getAll()
for key, value in configs:
    if "azure" in key.lower():
        print(f"{key}: {value}")

# Check current working directory
import os
print(f"Current working directory: {os.getcwd()}")

# Test connectivity
dbutils.fs.ls("abfss://container@storageaccount.dfs.core.windows.net/")
```

### Performance Monitoring

```python
# Monitor file sizes
def analyze_directory(path):
    """
    Analyze files in a directory for size distribution
    """
    files = dbutils.fs.ls(path)
    sizes = [f.size for f in files if not f.isDir()]
    
    if sizes:
        import statistics
        print(f"Total files: {len(sizes)}")
        print(f"Total size: {sum(sizes):,} bytes")
        print(f"Average size: {statistics.mean(sizes):,.0f} bytes")
        print(f"Median size: {statistics.median(sizes):,.0f} bytes")
        print(f"Min size: {min(sizes):,} bytes")
        print(f"Max size: {max(sizes):,} bytes")
        
        # Identify small files (< 10MB)
        small_files = [s for s in sizes if s < 10 * 1024 * 1024]
        if small_files:
            print(f"Small files (< 10MB): {len(small_files)} ({len(small_files)/len(sizes)*100:.1f}%)")

# Usage
analyze_directory("/mnt/datalake/data/")
```

## Quick Reference Commands

```python
# Essential dbutils.fs commands
dbutils.fs.ls(path)                    # List directory contents
dbutils.fs.mkdirs(path)               # Create directory
dbutils.fs.cp(src, dst, recurse=True) # Copy files/directories  
dbutils.fs.rm(path, recurse=True)     # Remove files/directories
dbutils.fs.mv(src, dst)               # Move/rename files
dbutils.fs.head(path, max_bytes=65536) # Read file head
dbutils.fs.put(path, contents, overwrite=False) # Write string to file

# Mount operations
dbutils.fs.mount(source, mount_point, extra_configs) # Mount storage
dbutils.fs.unmount(mount_point)       # Unmount storage
dbutils.fs.mounts()                   # List all mounts
dbutils.fs.refreshMounts()            # Refresh mount cache
```
