# Databricks references

- [Databricks utilities](#Databricks-Utilities)
- [Databricks CLI](#Databricks-CLI)
- [Databricks Asset Bundles](#Databricks-Asset-Bundles)
- [Databricks Unity Catalog](#Databricks-Unity-Catalog)


# Databricks Utilities

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


# Databricks CLI

## Installation & Setup

### Installation

#### Method 1: Using Homebrew (macOS/Linux) - Recommended
```bash
# Install via Homebrew
brew tap databricks/tap
brew install databricks

# Upgrade
brew upgrade databricks

# Verify installation
databricks --version
```

#### Method 2: Direct Binary Installation (Bash/Shell)
```bash
# Download and install Databricks CLI binary (Linux/macOS)
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

# Or using wget
wget -O - https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

# Manual download and install (Linux)
DATABRICKS_CLI_VERSION="0.230.0"  # Replace with desired version
curl -fsSL "https://github.com/databricks/cli/releases/download/v${DATABRICKS_CLI_VERSION}/databricks_cli_${DATABRICKS_CLI_VERSION}_linux_amd64.tar.gz" | tar -xz -C /tmp
sudo mv /tmp/databricks /usr/local/bin/
chmod +x /usr/local/bin/databricks

# Manual download and install (macOS Intel)
curl -fsSL "https://github.com/databricks/cli/releases/download/v${DATABRICKS_CLI_VERSION}/databricks_cli_${DATABRICKS_CLI_VERSION}_darwin_amd64.tar.gz" | tar -xz -C /tmp
sudo mv /tmp/databricks /usr/local/bin/
chmod +x /usr/local/bin/databricks

# macOS (Apple Silicon - M1/M2/M3)
curl -fsSL "https://github.com/databricks/cli/releases/download/v${DATABRICKS_CLI_VERSION}/databricks_cli_${DATABRICKS_CLI_VERSION}_darwin_arm64.tar.gz" | tar -xz -C /tmp
sudo mv /tmp/databricks /usr/local/bin/
chmod +x /usr/local/bin/databricks

# Verify installation
databricks --version
which databricks
```

#### Method 3: Using Windows Package Managers
```bash
# Using winget
winget install Databricks.DatabricksCLI

# Using Chocolatey (Experimental)
choco install databricks-cli

# Verify
databricks --version
```

#### Method 4: Using Snap (Linux)
```bash
# Install via Snap
sudo snap install databricks --classic

# Upgrade
sudo snap refresh databricks
```

#### Method 5: Using pip (Python-based - Legacy CLI)
```bash
# Note: This installs the legacy CLI (v0.18 and below) - NOT RECOMMENDED
pip install databricks-cli

# For the modern CLI, use one of the methods above
```

#### Verify Installation
```bash
# Check version (should be 0.205 or above)
databricks --version

# Check help
databricks --help

# List command groups
databricks -h
```

---

### Authentication & Configuration

#### OAuth Authentication (Recommended)
```bash
# Configure with OAuth (opens browser for authentication)
databricks auth login --host https://<workspace-url>.cloud.databricks.com

# With specific profile name
databricks auth login --host https://<workspace-url>.cloud.databricks.com --profile my-profile

# List existing profiles
databricks auth profiles

# View current profile settings
databricks auth env --profile <profile-name>

# View OAuth token info
databricks auth token --host https://<workspace-url>.cloud.databricks.com -p <profile>
```

#### Personal Access Token Authentication
```bash
# Configure using environment variables
export DATABRICKS_HOST="https://<workspace>.cloud.databricks.com"
export DATABRICKS_TOKEN="<your-token>"

# Or configure interactively (creates profile in ~/.databrickscfg)
databricks configure --token

# Test connection
databricks workspace list /
```

#### Configuration File Structure
Location: `~/.databrickscfg` (Linux/Mac) or `%USERPROFILE%\.databrickscfg` (Windows)

```ini
[DEFAULT]
host = https://workspace.cloud.databricks.com
token = dapi1234567890abcdef

[production]
host = https://prod-workspace.cloud.databricks.com
token = dapi0987654321fedcba

[development]
host = https://dev-workspace.cloud.databricks.com
token = dapi1122334455667788
```

#### Using Profiles
```bash
# Use specific profile
databricks --profile production clusters list

# Or set as environment variable
export DATABRICKS_CONFIG_PROFILE="production"
databricks clusters list
```

---

## Core Command Categories

### 1. Workspace Commands

#### List Workspace Items
```bash
# List root workspace
databricks workspace list /

# List specific path
databricks workspace list /Users/user@example.com

# List with output format
databricks workspace list /Shared --output json
```

#### Export Notebooks
```bash
# Export single notebook as source
databricks workspace export /path/to/notebook notebook.py

# Export as different formats
databricks workspace export /path/to/notebook notebook.html --format HTML
databricks workspace export /path/to/notebook notebook.ipynb --format JUPYTER
databricks workspace export /path/to/notebook notebook.dbc --format DBC

# Export entire directory
databricks workspace export-dir /Workspace/MyProject ./local_backup
```

#### Import Notebooks
```bash
# Import notebook
databricks workspace import notebook.py /Workspace/path/to/notebook --language PYTHON

# Import from file
databricks workspace import notebook.scala /path/to/notebook --language SCALA

# Import directory
databricks workspace import-dir ./local_project /Workspace/MyProject
```

#### Workspace Management
```bash
# Create directory (mkdirs creates parent directories)
databricks workspace mkdirs /Workspace/new/folder/path

# Delete item
databricks workspace delete /path/to/file

# Delete recursively
databricks workspace delete /path/to/folder --recursive

# Get workspace status
databricks workspace get-status /path/to/item
```

---

### 2. Clusters Commands

#### List & Get Cluster Info
```bash
# List all clusters
databricks clusters list

# List with JSON output
databricks clusters list --output json

# Get cluster details
databricks clusters get <cluster-id>

# Example with specific cluster
databricks clusters get 1234-567890-abcde123

# List cluster events
databricks clusters events <cluster-id>

# Events with filters
databricks clusters events <cluster-id> \
  --start-time 1640000000000 \
  --end-time 1640086400000 \
  --order DESC \
  --limit 100
```

#### Cluster Lifecycle
```bash
# Create cluster from JSON file
databricks clusters create --json-file cluster-config.json

# Create cluster with inline JSON (Linux/Mac)
databricks clusters create --json '{
  "cluster_name": "my-cluster",
  "spark_version": "13.3.x-scala2.12",
  "node_type_id": "i3.xlarge",
  "num_workers": 2
}'

# Start cluster
databricks clusters start <cluster-id>

# Restart cluster
databricks clusters restart <cluster-id>

# Terminate cluster
databricks clusters delete <cluster-id>

# Permanent delete (removes configuration)
databricks clusters permanent-delete <cluster-id>
```

#### Cluster Configuration Example (JSON)
```json
{
  "cluster_name": "production-cluster",
  "spark_version": "14.3.x-scala2.12",
  "node_type_id": "i3.xlarge",
  "num_workers": 4,
  "autoscale": {
    "min_workers": 2,
    "max_workers": 8
  },
  "spark_conf": {
    "spark.speculation": "true",
    "spark.sql.adaptive.enabled": "true"
  },
  "spark_env_vars": {
    "PYSPARK_PYTHON": "/databricks/python3/bin/python3"
  },
  "auto_termination_minutes": 120,
  "enable_elastic_disk": true,
  "cluster_source": "API"
}
```

#### Advanced Cluster Operations
```bash
# Edit cluster configuration
databricks clusters edit --json-file updated-config.json

# Resize cluster (must be running)
databricks clusters resize <cluster-id> --num-workers 8

# List available node types
databricks clusters list-node-types

# List Spark versions
databricks clusters spark-versions

# Get cluster permissions
databricks clusters get-permission-levels <cluster-id>

# Change cluster owner (requires admin)
databricks clusters change-owner <cluster-id> new-owner@example.com
```

---

### 3. Jobs Commands

#### List & Get Jobs
```bash
# List all jobs
databricks jobs list

# List with pagination
databricks jobs list --limit 50 --offset 0

# Get job details
databricks jobs get <job-id>

# Example
databricks jobs get 123456789

# List runs for a job
databricks jobs list-runs --job-id <job-id>

# Get specific run details
databricks jobs get-run <run-id>

# Get run output
databricks jobs get-run-output <run-id>
```

#### Job Management
```bash
# Create job from JSON file
databricks jobs create --json-file job-config.json

# Create job with inline JSON (Linux/Mac)
databricks jobs create --json '{
  "name": "Daily ETL",
  "tasks": [{
    "task_key": "main_task",
    "notebook_task": {
      "notebook_path": "/Workspace/ETL/main",
      "source": "WORKSPACE"
    },
    "existing_cluster_id": "cluster-123"
  }]
}'

# Update entire job configuration
databricks jobs reset <job-id> --json-file new-config.json

# Update specific job settings
databricks jobs update <job-id> --json-file partial-update.json

# Delete job
databricks jobs delete <job-id>

# Run job now
databricks jobs run-now <job-id>

# Run with parameters (Linux/Mac)
databricks jobs run-now <job-id> --json '{
  "notebook_params": {"date": "2024-01-01", "env": "prod"}
}'
```

#### Job Run Operations
```bash
# Submit one-time run
databricks jobs submit --json-file one-time-job.json

# Cancel run
databricks jobs cancel-run <run-id>

# Cancel all runs for a job
databricks jobs cancel-all-runs <job-id>

# Repair run (retry failed tasks)
databricks jobs repair-run <run-id>

# Export run results
databricks jobs export-run <run-id>

# List active runs
databricks jobs list-runs --active-only
```

#### Job Configuration Example
```json
{
  "name": "Multi-Task ETL Pipeline",
  "tasks": [
    {
      "task_key": "extract_data",
      "notebook_task": {
        "notebook_path": "/Workspace/ETL/extract",
        "source": "WORKSPACE",
        "base_parameters": {
          "environment": "production"
        }
      },
      "new_cluster": {
        "spark_version": "14.3.x-scala2.12",
        "node_type_id": "i3.xlarge",
        "num_workers": 2
      }
    },
    {
      "task_key": "transform_data",
      "depends_on": [{"task_key": "extract_data"}],
      "notebook_task": {
        "notebook_path": "/Workspace/ETL/transform",
        "source": "WORKSPACE"
      },
      "existing_cluster_id": "cluster-id"
    }
  ],
  "schedule": {
    "quartz_cron_expression": "0 0 8 * * ?",
    "timezone_id": "America/Los_Angeles",
    "pause_status": "UNPAUSED"
  },
  "email_notifications": {
    "on_failure": ["team@example.com"],
    "on_success": ["manager@example.com"]
  },
  "max_concurrent_runs": 1,
  "timeout_seconds": 3600
}
```

---

### 4. Files (DBFS & Workspace) Commands

#### Basic File Operations
```bash
# List files in DBFS
databricks fs ls dbfs:/path/to/directory

# List files in workspace volumes (Unity Catalog)
databricks fs ls /Volumes/catalog/schema/volume

# List with details
databricks fs ls -l dbfs:/

# Copy file to DBFS
databricks fs cp local_file.csv dbfs:/data/file.csv

# Copy from DBFS to local
databricks fs cp dbfs:/data/file.csv ./local_file.csv

# Copy recursively
databricks fs cp -r ./local_dir dbfs:/data/dir

# Copy with overwrite
databricks fs cp --overwrite local.csv dbfs:/data/file.csv
```

#### File Management
```bash
# Remove file
databricks fs rm dbfs:/path/to/file

# Remove directory recursively
databricks fs rm -r dbfs:/path/to/directory

# Create directory
databricks fs mkdirs dbfs:/new/directory

# Move/rename
databricks fs mv dbfs:/old/path dbfs:/new/path

# View file contents (cat)
databricks fs cat dbfs:/path/to/file.txt

# View with head (first N bytes)
databricks fs head dbfs:/logs/app.log
```

---

### 5. Secrets Management

#### Scope Management
```bash
# Create secret scope
databricks secrets create-scope <scope-name>

# Create scope with backend type
databricks secrets create-scope <scope-name> --scope-backend-type DATABRICKS

# List scopes
databricks secrets list-scopes

# Delete scope
databricks secrets delete-scope <scope-name>
```

#### Secret Operations
```bash
# Put secret (prompts for value)
databricks secrets put-secret <scope-name> <key-name>

# Put secret with string value
databricks secrets put-secret <scope-name> <key-name> --string-value "my-secret"

# Put secret from file
databricks secrets put-secret <scope-name> <key-name> --binary-file ./secret.key

# List secrets in scope
databricks secrets list-secrets <scope-name>

# Delete secret
databricks secrets delete-secret <scope-name> <key-name>
```

#### Access Control Lists (ACLs)
```bash
# Grant permission
databricks secrets put-acl <scope-name> <principal> READ

# Available permissions: READ, WRITE, MANAGE
databricks secrets put-acl <scope-name> user@example.com WRITE

# List ACLs for scope
databricks secrets list-acls <scope-name>

# Get ACL for specific principal
databricks secrets get-acl <scope-name> <principal>

# Delete ACL
databricks secrets delete-acl <scope-name> <principal>
```

---

### 6. Libraries Management

#### Cluster Libraries
```bash
# List cluster libraries
databricks libraries cluster-status <cluster-id>

# Install JAR library
databricks libraries install <cluster-id> --jar dbfs:/jars/mylib.jar

# Install PyPI package
databricks libraries install <cluster-id> --pypi pandas==2.0.0

# Install Maven package
databricks libraries install <cluster-id> \
  --maven '{"coordinates": "com.example:package:1.0.0"}'

# Install wheel file
databricks libraries install <cluster-id> --whl dbfs:/wheels/mypackage-1.0-py3-none-any.whl

# Install multiple libraries from JSON
databricks libraries install <cluster-id> --json-file libraries.json

# Uninstall library
databricks libraries uninstall <cluster-id> --pypi pandas

# Get library status
databricks libraries all-cluster-statuses
```

#### Library Configuration Example (JSON)
```json
{
  "cluster_id": "cluster-id",
  "libraries": [
    {"pypi": {"package": "pandas==2.0.0"}},
    {"pypi": {"package": "scikit-learn>=1.3.0"}},
    {"jar": "dbfs:/jars/custom-lib-1.0.jar"},
    {"maven": {"coordinates": "com.databricks:spark-xml_2.12:0.16.0"}},
    {"whl": "dbfs:/wheels/mypackage-1.0-py3-none-any.whl"}
  ]
}
```

---

### 7. Instance Pools

```bash
# List instance pools
databricks instance-pools list

# Get pool details
databricks instance-pools get <pool-id>

# Create instance pool
databricks instance-pools create --json-file pool-config.json

# Edit instance pool
databricks instance-pools edit <pool-id> --json-file updated-pool.json

# Delete instance pool
databricks instance-pools delete <pool-id>
```

#### Pool Configuration Example
```json
{
  "instance_pool_name": "shared-pool",
  "min_idle_instances": 2,
  "max_capacity": 10,
  "node_type_id": "i3.xlarge",
  "idle_instance_autotermination_minutes": 15,
  "enable_elastic_disk": true,
  "preloaded_spark_versions": [
    "14.3.x-scala2.12"
  ]
}
```

---

### 8. Groups & Users (Identity Management)

#### Group Management
```bash
# List groups
databricks groups list

# Create group
databricks groups create <group-name>

# Delete group
databricks groups delete <group-name>

# Get group details
databricks groups get <group-name>

# Add user to group
databricks groups add-member <group-name> --user-name user@example.com

# Add service principal to group
databricks groups add-member <group-name> --service-principal-name app-id

# Remove member
databricks groups remove-member <group-name> --user-name user@example.com

# List group members
databricks groups list-members <group-name>
```

#### User Management
```bash
# List users
databricks users list

# Get user details
databricks users get <user-id>

# Create user (requires admin)
databricks users create --user-name newuser@example.com

# Update user
databricks users update <user-id> --json '{"active": false}'

# Delete user
databricks users delete <user-id>
```

#### Service Principal Management
```bash
# List service principals
databricks service-principals list

# Get service principal
databricks service-principals get <sp-id>

# Create service principal
databricks service-principals create --json-file sp-config.json

# Delete service principal
databricks service-principals delete <sp-id>
```

---

### 9. Tokens Management

```bash
# Create token
databricks tokens create --comment "CLI automation token" --lifetime-seconds 7776000

# Create token with JSON
databricks tokens create --json '{
  "comment": "Production API token",
  "lifetime_seconds": 7776000
}'

# List tokens
databricks tokens list

# Get token info
databricks tokens get <token-id>

# Revoke token
databricks tokens delete <token-id>

# List tokens for other users (admin only)
databricks token-management list --created-by-username user@example.com
```

---

### 10. Repos (Git Integration)

```bash
# List repos
databricks repos list

# Get repo details
databricks repos get <repo-id>

# Get repo by path
databricks repos get --path /Repos/user@example.com/my-repo

# Create repo
databricks repos create --url https://github.com/user/repo --provider github

# Create with specific path
databricks repos create \
  --url https://github.com/user/repo \
  --provider github \
  --path /Repos/user@example.com/my-repo

# Update repo (pull latest changes)
databricks repos update <repo-id> --branch main

# Switch branch
databricks repos update <repo-id> --branch feature-branch

# Delete repo
databricks repos delete <repo-id>

# Supported providers: github, bitbucketCloud, gitLab, azureDevOpsServices
```

---

### 11. Unity Catalog Commands

#### Catalogs
```bash
# List catalogs
databricks catalogs list

# Get catalog details
databricks catalogs get <catalog-name>

# Create catalog
databricks catalogs create <catalog-name>

# Update catalog
databricks catalogs update <catalog-name> --comment "Production catalog"

# Delete catalog
databricks catalogs delete <catalog-name>
```

#### Schemas
```bash
# List schemas
databricks schemas list <catalog-name>

# Get schema
databricks schemas get <catalog-name>.<schema-name>

# Create schema
databricks schemas create <catalog-name>.<schema-name>

# Delete schema
databricks schemas delete <catalog-name>.<schema-name>
```

#### Tables
```bash
# List tables
databricks tables list <catalog-name>.<schema-name>

# Get table
databricks tables get <catalog-name>.<schema-name>.<table-name>

# Delete table
databricks tables delete <catalog-name>.<schema-name>.<table-name>
```

#### Volumes
```bash
# List volumes
databricks volumes list <catalog-name>.<schema-name>

# Get volume
databricks volumes get <catalog-name>.<schema-name>.<volume-name>

# Create volume
databricks volumes create <catalog-name>.<schema-name>.<volume-name> --json '{
  "volume_type": "MANAGED"
}'

# Delete volume
databricks volumes delete <catalog-name>.<schema-name>.<volume-name>
```

#### Grants
```bash
# Grant permissions
databricks grants update <securable-type> <full-name> \
  --principal user@example.com \
  --privilege SELECT

# Example: Grant SELECT on table
databricks grants update table catalog.schema.table \
  --principal user@example.com \
  --privilege SELECT

# Get grants
databricks grants get <securable-type> <full-name>

# Revoke grant
databricks grants revoke <securable-type> <full-name> \
  --principal user@example.com \
  --privilege SELECT
```

---

### 12. SQL Warehouses

```bash
# List SQL warehouses
databricks sql-warehouses list

# Get warehouse details
databricks sql-warehouses get <warehouse-id>

# Create SQL warehouse
databricks sql-warehouses create --json-file warehouse-config.json

# Start warehouse
databricks sql-warehouses start <warehouse-id>

# Stop warehouse
databricks sql-warehouses stop <warehouse-id>

# Edit warehouse
databricks sql-warehouses edit <warehouse-id> --json-file config.json

# Delete warehouse
databricks sql-warehouses delete <warehouse-id>
```

---

### 13. Bundles (Databricks Asset Bundles)

```bash
# Initialize new bundle
databricks bundle init

# Initialize with template
databricks bundle init --template default-python

# Validate bundle
databricks bundle validate

# Deploy bundle
databricks bundle deploy

# Deploy to specific target
databricks bundle deploy --target production

# Run bundle workflow
databricks bundle run <workflow-name>

# Destroy bundle resources
databricks bundle destroy
```

---

## Complete Command Reference Table

| Category | Command | Description |
|----------|---------|-------------|
| **Auth** | `auth login` | Authenticate via OAuth |
| | `auth token` | Get OAuth token info |
| | `auth profiles` | List authentication profiles |
| | `auth env` | View profile configuration |
| **Workspace** | `workspace list` | List workspace items |
| | `workspace export` | Export notebook/file |
| | `workspace export-dir` | Export directory |
| | `workspace import` | Import notebook/file |
| | `workspace import-dir` | Import directory |
| | `workspace mkdirs` | Create directory |
| | `workspace delete` | Delete item |
| | `workspace get-status` | Get item metadata |
| **Clusters** | `clusters list` | List all clusters |
| | `clusters get` | Get cluster details |
| | `clusters create` | Create new cluster |
| | `clusters edit` | Edit cluster config |
| | `clusters start` | Start cluster |
| | `clusters restart` | Restart cluster |
| | `clusters delete` | Terminate cluster |
| | `clusters permanent-delete` | Permanently delete cluster |
| | `clusters resize` | Resize cluster |
| | `clusters events` | Get cluster events |
| | `clusters list-node-types` | List node types |
| | `clusters spark-versions` | List Spark versions |
| | `clusters change-owner` | Change cluster owner |
| **Jobs** | `jobs list` | List all jobs |
| | `jobs get` | Get job details |
| | `jobs create` | Create new job |
| | `jobs reset` | Replace job config |
| | `jobs update` | Update job settings |
| | `jobs delete` | Delete job |
| | `jobs run-now` | Trigger job run |
| | `jobs submit` | Submit one-time run |
| | `jobs list-runs` | List job runs |
| | `jobs get-run` | Get run details |
| | `jobs get-run-output` | Get run output |
| | `jobs cancel-run` | Cancel run |
| | `jobs cancel-all-runs` | Cancel all runs |
| | `jobs repair-run` | Retry failed tasks |
| **Files** | `fs ls` | List files |
| | `fs cp` | Copy files |
| | `fs rm` | Remove files |
| | `fs mkdirs` | Create directory |
| | `fs mv` | Move/rename |
| | `fs cat` | View file contents |
| | `fs head` | View file head |
| **Secrets** | `secrets create-scope` | Create secret scope |
| | `secrets list-scopes` | List scopes |
| | `secrets delete-scope` | Delete scope |
| | `secrets put-secret` | Store secret |
| | `secrets list-secrets` | List secrets |
| | `secrets delete-secret` | Delete secret |
| | `secrets put-acl` | Grant permission |
| | `secrets list-acls` | List permissions |
| | `secrets get-acl` | Get ACL |
| | `secrets delete-acl` | Remove permission |
| **Libraries** | `libraries cluster-status` | List cluster libraries |
| | `libraries install` | Install library |
| | `libraries uninstall` | Uninstall library |
| | `libraries all-cluster-statuses` | Get all statuses |
| **Pools** | `instance-pools list` | List instance pools |
| | `instance-pools get` | Get pool details |
| | `instance-pools create` | Create pool |
| | `instance-pools edit` | Edit pool |
| | `instance-pools delete` | Delete pool |
| **Groups** | `groups list` | List groups |
| | `groups get` | Get group details |
| | `groups create` | Create group |
| | `groups delete` | Delete group |
| | `groups add-member` | Add member |
| | `groups remove-member` | Remove member |
| | `groups list-members` | List members |
| **Users** | `users list` | List users |
| | `users get` | Get user details |
| | `users create` | Create user |
| | `users update` | Update user |
| | `users delete` | Delete user |
| **Service Principals** | `service-principals list` | List service principals |
| | `service-principals get` | Get details |
| | `service-principals create` | Create SP |
| | `service-principals delete` | Delete SP |
| **Tokens** | `tokens create` | Create access token |
| | `tokens list` | List tokens |
| | `tokens get` | Get token info |
| | `tokens delete` | Revoke token |
| | `token-management list` | List all tokens (admin) |
| **Repos** | `repos list` | List repos |
| | `repos get` | Get repo details |
| | `repos create` | Create repo |
| | `repos update` | Update/pull repo |
| | `repos delete` | Delete repo |
| **Unity Catalog** | `catalogs list` | List catalogs |
| | `catalogs get` | Get catalog |
| | `catalogs create` | Create catalog |
| | `catalogs delete` | Delete catalog |
| | `schemas list` | List schemas |
| | `schemas get` | Get schema |
| | `schemas create` | Create schema |
| | `schemas delete` | Delete schema |
| | `tables list` | List tables |
| | `tables get` | Get table |
| | `tables delete` | Delete table |
| | `volumes list` | List volumes |
| | `volumes get` | Get volume |
| | `volumes create` | Create volume |
| | `volumes delete` | Delete volume |
| | `grants update` | Grant permissions |
| | `grants get` | Get grants |
| | `grants revoke` | Revoke permissions |
| **SQL** | `sql-warehouses list` | List warehouses |
| | `sql-warehouses get` | Get warehouse |
| | `sql-warehouses create` | Create warehouse |
| | `sql-warehouses start` | Start warehouse |
| | `sql-warehouses stop` | Stop warehouse |
| | `sql-warehouses edit` | Edit warehouse |
| | `sql-warehouses delete` | Delete warehouse |
| **Bundles** | `bundle init` | Initialize bundle |
| | `bundle validate` | Validate bundle |
| | `bundle deploy` | Deploy bundle |
| | `bundle run` | Run workflow |
| | `bundle destroy` | Destroy resources |
| **API** | `api` | Make REST API requests |
| **System** | `configure` | Configure CLI |
| | `version` | Show CLI version |
| | `completion` | Generate shell completion |

---

## Advanced Techniques

### 1. Scripting & Automation

#### Bash Script Example - Cluster Management
```bash
#!/bin/bash
set -e

PROFILE="production"
CLUSTER_ID="1234-567890-cluster"

# Function to check cluster state
check_cluster_state() {
    databricks --profile "$PROFILE" clusters get "$CLUSTER_ID" \
        --output json | jq -r '.state'
}

# Start cluster if not running
STATE=$(check_cluster_state)
if [ "$STATE" != "RUNNING" ]; then
    echo "Starting cluster..."
    databricks --profile "$PROFILE" clusters start "$CLUSTER_ID"
    
    # Wait for cluster to start
    while [ "$(check_cluster_state)" != "RUNNING" ]; do
        echo "Waiting for cluster to start..."
        sleep 30
    done
    echo "Cluster is running"
fi

# Run job
echo "Running job..."
RUN_ID=$(databricks --profile "$PROFILE" jobs run-now 456 --output json | jq -r '.run_id')
echo "Job run ID: $RUN_ID"

# Monitor job status
while true; do
    STATUS=$(databricks --profile "$PROFILE" jobs get-run "$RUN_ID" --output json | jq -r '.state.life_cycle_state')
    echo "Job status: $STATUS"
    
    if [ "$STATUS" = "TERMINATED" ]; then
        RESULT=$(databricks --profile "$PROFILE" jobs get-run "$RUN_ID" --output json | jq -r '.state.result_state')
        echo "Job completed with result: $RESULT"
        
        if [ "$RESULT" = "SUCCESS" ]; then
            exit 0
        else
            echo "Job failed!"
            exit 1
        fi
    fi
    
    sleep 30
done
```

#### Python Script Example - Workspace Backup
```python
#!/usr/bin/env python3
import subprocess
import json
import os
from datetime import datetime

PROFILE = "production"
BACKUP_DIR = f"./backup_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

def run_command(cmd):
    """Execute databricks CLI command and return output"""
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    if result.returncode != 0:
        raise Exception(f"Command failed: {result.stderr}")
    return result.stdout

def backup_notebooks():
    """Backup all notebooks from workspace"""
    print(f"Creating backup directory: {BACKUP_DIR}")
    os.makedirs(BACKUP_DIR, exist_ok=True)
    
    # Export workspace
    cmd = f"databricks --profile {PROFILE} workspace export-dir / {BACKUP_DIR}"
    run_command(cmd)
    print(f"Notebooks backed up to {BACKUP_DIR}")

def backup_jobs():
    """Backup all job configurations"""
    jobs_dir = os.path.join(BACKUP_DIR, "jobs")
    os.makedirs(jobs_dir, exist_ok=True)
    
    # List all jobs
    cmd = f"databricks --profile {PROFILE} jobs list --output json"
    output = run_command(cmd)
    jobs = json.loads(output).get('jobs', [])
    
    # Export each job configuration
    for job in jobs:
        job_id = job['job_id']
        job_name = job['settings']['name'].replace('/', '_')
        
        cmd = f"databricks --profile {PROFILE} jobs get {job_id} --output json"
        job_config = run_command(cmd)
        
        with open(os.path.join(jobs_dir, f"{job_id}_{job_name}.json"), 'w') as f:
            f.write(job_config)
    
    print(f"Backed up {len(jobs)} job configurations")

def backup_clusters():
    """Backup all cluster configurations"""
    clusters_dir = os.path.join(BACKUP_DIR, "clusters")
    os.makedirs(clusters_dir, exist_ok=True)
    
    cmd = f"databricks --profile {PROFILE} clusters list --output json"
    output = run_command(cmd)
    clusters = json.loads(output).get('clusters', [])
    
    for cluster in clusters:
        cluster_id = cluster['cluster_id']
        cluster_name = cluster['cluster_name'].replace('/', '_')
        
        cmd = f"databricks --profile {PROFILE} clusters get {cluster_id} --output json"
        cluster_config = run_command(cmd)
        
        with open(os.path.join(clusters_dir, f"{cluster_name}.json"), 'w') as f:
            f.write(cluster_config)
    
    print(f"Backed up {len(clusters)} cluster configurations")

if __name__ == "__main__":
    try:
        backup_notebooks()
        backup_jobs()
        backup_clusters()
        print(f"\nBackup completed successfully: {BACKUP_DIR}")
    except Exception as e:
        print(f"Backup failed: {e}")
        exit(1)
```

### 2. JSON Output Processing with jq

```bash
# Get all running clusters with their names and IDs
databricks clusters list --output json | \
  jq '.clusters[] | select(.state=="RUNNING") | {name: .cluster_name, id: .cluster_id}'

# Get all jobs scheduled to run
databricks jobs list --output json | \
  jq '.jobs[] | select(.settings.schedule != null) | {name: .settings.name, schedule: .settings.schedule}'

# Find clusters by tag
databricks clusters list --output json | \
  jq '.clusters[] | select(.custom_tags.Environment == "production")'

# Get total number of workers across all running clusters
databricks clusters list --output json | \
  jq '[.clusters[] | select(.state=="RUNNING") | .num_workers] | add'

# Export all notebooks in a directory
databricks workspace list /Users/user@example.com --output json | \
  jq -r '.objects[] | select(.object_type=="NOTEBOOK") | .path' | \
  while read path; do
    filename=$(basename "$path")
    databricks workspace export "$path" "./${filename}.py"
    echo "Exported: $path"
  done

# List all failed job runs from today
databricks jobs list-runs --output json | \
  jq '.runs[] | select(.state.result_state == "FAILED") | {job_id, run_id, start_time}'

# Get cluster costs (if cost tags are set)
databricks clusters list --output json | \
  jq '.clusters[] | {name: .cluster_name, cost: .custom_tags.cost_center}'
```

### 3. Bulk Operations

#### Delete All Terminated Clusters
```bash
#!/bin/bash
databricks clusters list --output json | \
  jq -r '.clusters[] | select(.state == "TERMINATED") | .cluster_id' | \
  while read cluster_id; do
    echo "Permanently deleting cluster: $cluster_id"
    databricks clusters permanent-delete "$cluster_id"
  done
```

#### Export All Job Configurations
```bash
#!/bin/bash
BACKUP_DIR="./jobs_backup_$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

databricks jobs list --output json | \
  jq -r '.jobs[] | "\(.job_id)|\(.settings.name)"' | \
  while IFS='|' read -r job_id job_name; do
    safe_name=$(echo "$job_name" | tr '/' '_' | tr ' ' '_')
    echo "Exporting job: $job_name (ID: $job_id)"
    databricks jobs get "$job_id" --output json > "$BACKUP_DIR/${job_id}_${safe_name}.json"
  done

echo "All jobs exported to $BACKUP_DIR"
```

#### Cancel All Running Jobs
```bash
#!/bin/bash
echo "Finding all active runs..."
databricks jobs list-runs --active-only --output json | \
  jq -r '.runs[] | "\(.run_id)|\(.run_name)"' | \
  while IFS='|' read -r run_id run_name; do
    echo "Cancelling run: $run_name (ID: $run_id)"
    databricks jobs cancel-run "$run_id"
  done
```

#### Install Library on Multiple Clusters
```bash
#!/bin/bash
LIBRARY="pandas==2.1.0"

# Get all running clusters
databricks clusters list --output json | \
  jq -r '.clusters[] | select(.state == "RUNNING") | .cluster_id' | \
  while read cluster_id; do
    echo "Installing $LIBRARY on cluster $cluster_id"
    databricks libraries install "$cluster_id" --pypi "$LIBRARY"
  done
```

### 4. Environment-Specific Configurations

#### Multi-Environment Deployment Script
```bash
#!/bin/bash

deploy_to_env() {
    local ENV=$1
    local PROFILE=$2
    
    echo "Deploying to $ENV environment..."
    
    # Import notebooks
    databricks --profile "$PROFILE" workspace import-dir \
        ./notebooks "/Production/${ENV}"
    
    # Update job with environment-specific config
    databricks --profile "$PROFILE" jobs reset 123 \
        --json-file "configs/job_${ENV}.json"
    
    # Trigger job
    RUN_ID=$(databricks --profile "$PROFILE" jobs run-now 123 \
        --json "{\"notebook_params\": {\"env\": \"${ENV}\"}}" \
        --output json | jq -r '.run_id')
    
    echo "Deployment to $ENV completed. Run ID: $RUN_ID"
}

# Deploy to each environment
deploy_to_env "dev" "development"
deploy_to_env "staging" "staging"
deploy_to_env "prod" "production"
```

#### Configuration Management
```bash
# Development
export DATABRICKS_CONFIG_PROFILE="dev"
databricks jobs run-now 123

# Staging
export DATABRICKS_CONFIG_PROFILE="staging"
databricks jobs run-now 456

# Production
export DATABRICKS_CONFIG_PROFILE="prod"
databricks jobs run-now 789

# Or use inline profile specification
databricks --profile dev jobs run-now 123
databricks --profile staging jobs run-now 456
databricks --profile prod jobs run-now 789
```

### 5. Error Handling & Retry Logic

#### Robust Script with Error Handling
```bash
#!/bin/bash
set -euo pipefail

# Configuration
MAX_RETRIES=3
RETRY_DELAY=10
CLUSTER_ID="cluster-123"

# Function to retry commands
retry() {
    local retries=$1
    shift
    local cmd="$@"
    local count=0
    
    until $cmd; do
        exit_code=$?
        count=$((count + 1))
        
        if [ $count -lt $retries ]; then
            echo "Command failed with exit code $exit_code. Retry $count/$retries..."
            sleep $RETRY_DELAY
        else
            echo "Command failed after $retries retries"
            return $exit_code
        fi
    done
    
    return 0
}

# Function to start cluster with retry
start_cluster() {
    echo "Starting cluster $CLUSTER_ID..."
    
    if retry $MAX_RETRIES databricks clusters start "$CLUSTER_ID"; then
        echo "Cluster started successfully"
        return 0
    else
        echo "Failed to start cluster" >&2
        return 1
    fi
}

# Function to wait for cluster
wait_for_cluster() {
    echo "Waiting for cluster to be ready..."
    
    local timeout=600  # 10 minutes
    local elapsed=0
    
    while [ $elapsed -lt $timeout ]; do
        STATE=$(databricks clusters get "$CLUSTER_ID" --output json | jq -r '.state')
        
        if [ "$STATE" = "RUNNING" ]; then
            echo "Cluster is ready"
            return 0
        elif [ "$STATE" = "ERROR" ] || [ "$STATE" = "TERMINATED" ]; then
            echo "Cluster failed to start: $STATE" >&2
            return 1
        fi
        
        echo "Cluster state: $STATE (waiting...)"
        sleep 10
        elapsed=$((elapsed + 10))
    done
    
    echo "Timeout waiting for cluster" >&2
    return 1
}

# Main execution
if start_cluster && wait_for_cluster; then
    echo "Proceeding with job execution..."
    databricks jobs run-now 456
else
    echo "Cluster setup failed" >&2
    exit 1
fi
```

### 6. CI/CD Integration Examples

#### GitHub Actions Workflow
```yaml
name: Deploy to Databricks

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Databricks CLI
        run: |
          curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
          echo "$HOME/.databricks/bin" >> $GITHUB_PATH
      
      - name: Configure Databricks CLI
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: |
          databricks configure --token <<EOF
          $DATABRICKS_HOST
          $DATABRICKS_TOKEN
          EOF
      
      - name: Deploy notebooks
        run: |
          databricks workspace import-dir ./notebooks /Production/notebooks
      
      - name: Update job configuration
        run: |
          databricks jobs reset 123 --json-file ./configs/job.json
      
      - name: Run tests
        run: |
          RUN_ID=$(databricks jobs run-now 456 --output json | jq -r '.run_id')
          echo "Test run ID: $RUN_ID"
          
          # Wait for completion
          while true; do
            STATE=$(databricks jobs get-run "$RUN_ID" --output json | jq -r '.state.life_cycle_state')
            if [ "$STATE" = "TERMINATED" ]; then
              RESULT=$(databricks jobs get-run "$RUN_ID" --output json | jq -r '.state.result_state')
              if [ "$RESULT" != "SUCCESS" ]; then
                echo "Tests failed!"
                exit 1
              fi
              break
            fi
            sleep 30
          done
```

#### GitLab CI Pipeline
```yaml
stages:
  - validate
  - deploy
  - test

variables:
  DATABRICKS_HOST: "https://your-workspace.cloud.databricks.com"

validate:
  stage: validate
  image: python:3.10
  script:
    - curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
    - export PATH="$HOME/.databricks/bin:$PATH"
    - databricks --version
    - databricks bundle validate
  only:
    - merge_requests

deploy_staging:
  stage: deploy
  image: python:3.10
  script:
    - curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
    - export PATH="$HOME/.databricks/bin:$PATH"
    - export DATABRICKS_TOKEN=$STAGING_TOKEN
    - databricks workspace import-dir ./notebooks /Staging
    - databricks jobs reset 123 --json-file ./configs/staging-job.json
  only:
    - staging
  environment:
    name: staging

deploy_production:
  stage: deploy
  image: python:3.10
  script:
    - curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
    - export PATH="$HOME/.databricks/bin:$PATH"
    - export DATABRICKS_TOKEN=$PROD_TOKEN
    - databricks workspace import-dir ./notebooks /Production
    - databricks jobs reset 789 --json-file ./configs/prod-job.json
  only:
    - main
  environment:
    name: production
  when: manual
```

### 7. Monitoring & Alerting

#### Cluster Cost Monitor
```bash
#!/bin/bash
# Monitor cluster costs and send alerts

COST_THRESHOLD=1000
ALERT_EMAIL="ops@example.com"

# Get all running clusters
CLUSTERS=$(databricks clusters list --output json | \
  jq -r '.clusters[] | select(.state == "RUNNING") | {
    name: .cluster_name,
    id: .cluster_id,
    workers: .num_workers,
    type: .node_type_id,
    uptime: .start_time
  }')

# Calculate estimated costs (example logic)
TOTAL_COST=0

echo "$CLUSTERS" | jq -c '.' | while read cluster; do
    NAME=$(echo "$cluster" | jq -r '.name')
    WORKERS=$(echo "$cluster" | jq -r '.workers')
    
    # Add your cost calculation logic here
    ESTIMATED_COST=$((WORKERS * 10))  # Example: $10 per worker
    TOTAL_COST=$((TOTAL_COST + ESTIMATED_COST))
    
    echo "Cluster: $NAME - Estimated cost: \$ESTIMATED_COST"
done

# Send alert if threshold exceeded
if [ $TOTAL_COST -gt $COST_THRESHOLD ]; then
    echo "ALERT: Total cluster cost \$TOTAL_COST exceeds threshold \$COST_THRESHOLD"
    # Send email or notification
fi
```

#### Job Failure Monitor
```bash
#!/bin/bash
# Monitor recent job failures and report

HOURS_AGO=24
SINCE_TIMESTAMP=$(($(date +%s) - (HOURS_AGO * 3600)))

echo "Checking for failed jobs in the last $HOURS_AGO hours..."

databricks jobs list-runs --output json | \
  jq --arg since "$SINCE_TIMESTAMP" '.runs[] | 
    select(.start_time > ($since | tonumber) and 
           .state.result_state == "FAILED") | 
    {
      job_id,
      run_id,
      run_name,
      error: .state.state_message
    }' | \
  jq -s '.' > failed_jobs.json

FAILURE_COUNT=$(jq 'length' failed_jobs.json)

if [ "$FAILURE_COUNT" -gt 0 ]; then
    echo "Found $FAILURE_COUNT failed jobs:"
    jq -r '.[] | "Job: \(.run_name) (ID: \(.job_id))\nRun ID: \(.run_id)\nError: \(.error)\n"' failed_jobs.json
else
    echo "No failed jobs found"
fi
```

---

## Common Use Cases & Workflows

### 1. Daily ETL Pipeline Deployment

```bash
#!/bin/bash
# Complete ETL pipeline deployment workflow

set -e

PROFILE="production"
CLUSTER_ID="etl-cluster-123"
JOB_ID="789"

echo "=== Starting ETL Pipeline Deployment ==="

# Step 1: Upload new data files
echo "1. Uploading data files..."
databricks --profile "$PROFILE" fs cp ./data/input.csv dbfs:/mnt/data/input.csv --overwrite

# Step 2: Import updated notebooks
echo "2. Importing notebooks..."
databricks --profile "$PROFILE" workspace import-dir \
    ./notebooks /Production/ETL --overwrite

# Step 3: Ensure cluster is running
echo "3. Starting cluster..."
databricks --profile "$PROFILE" clusters start "$CLUSTER_ID"

# Wait for cluster
echo "Waiting for cluster to be ready..."
sleep 60

# Step 4: Run ETL job
echo "4. Running ETL job..."
RUN_ID=$(databricks --profile "$PROFILE" jobs run-now "$JOB_ID" \
    --json '{"notebook_params": {"date": "'$(date +%Y-%m-%d)'"}}' \
    --output json | jq -r '.run_id')

echo "ETL job started with run ID: $RUN_ID"

# Step 5: Monitor execution
echo "5. Monitoring job execution..."
while true; do
    STATUS=$(databricks --profile "$PROFILE" jobs get-run "$RUN_ID" \
        --output json | jq -r '.state.life_cycle_state')
    
    echo "Current status: $STATUS"
    
    if [ "$STATUS" = "TERMINATED" ]; then
        RESULT=$(databricks --profile "$PROFILE" jobs get-run "$RUN_ID" \
            --output json | jq -r '.state.result_state')
        
        if [ "$RESULT" = "SUCCESS" ]; then
            echo "✓ ETL pipeline completed successfully"
            
            # Archive input file
            databricks --profile "$PROFILE" fs mv \
                dbfs:/mnt/data/input.csv \
                dbfs:/mnt/archive/input_$(date +%Y%m%d).csv
            
            exit 0
        else
            echo "✗ ETL pipeline failed"
            databricks --profile "$PROFILE" jobs get-run-output "$RUN_ID"
            exit 1
        fi
    fi
    
    sleep 30
done
```

### 2. Disaster Recovery & Backup

```bash
#!/bin/bash
# Complete workspace backup script

PROFILE="production"
BACKUP_ROOT="./backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_ROOT"

echo "=== Starting Databricks Backup ==="
echo "Backup location: $BACKUP_ROOT"

# Backup workspace
echo "1. Backing up workspace..."
databricks --profile "$PROFILE" workspace export-dir / "$BACKUP_ROOT/workspace"

# Backup all job configurations
echo "2. Backing up jobs..."
mkdir -p "$BACKUP_ROOT/jobs"
databricks --profile "$PROFILE" jobs list --output json | \
  jq -r '.jobs[] | .job_id' | \
  while read job_id; do
    databricks --profile "$PROFILE" jobs get "$job_id" \
        --output json > "$BACKUP_ROOT/jobs/job_${job_id}.json"
  done

# Backup cluster configurations
echo "3. Backing up clusters..."
mkdir -p "$BACKUP_ROOT/clusters"
databricks --profile "$PROFILE" clusters list --output json | \
  jq -r '.clusters[] | .cluster_id' | \
  while read cluster_id; do
    databricks --profile "$PROFILE" clusters get "$cluster_id" \
        --output json > "$BACKUP_ROOT/clusters/cluster_${cluster_id}.json"
  done

# Backup secrets (metadata only, not values)
echo "4. Backing up secret scopes..."
mkdir -p "$BACKUP_ROOT/secrets"
databricks --profile "$PROFILE" secrets list-scopes --output json | \
  jq -r '.scopes[] | .name' | \
  while read scope; do
    databricks --profile "$PROFILE" secrets list-secrets "$scope" \
        --output json > "$BACKUP_ROOT/secrets/scope_${scope}.json"
  done

# Compress backup
echo "5. Compressing backup..."
tar -czf "${BACKUP_ROOT}.tar.gz" -C "$(dirname $BACKUP_ROOT)" "$(basename $BACKUP_ROOT)"
rm -rf "$BACKUP_ROOT"

echo "✓ Backup completed: ${BACKUP_ROOT}.tar.gz"
```

### 3. Development to Production Promotion

```bash
#!/bin/bash
# Promote code from dev to production

set -e

DEV_PROFILE="development"
PROD_PROFILE="production"
TEMP_DIR="./promotion_$(date +%Y%m%d_%H%M%S)"

mkdir -p "$TEMP_DIR"

echo "=== Promoting to Production ==="

# Step 1: Export from development
echo "1. Exporting from development..."
databricks --profile "$DEV_PROFILE" workspace export-dir \
    /Development/Project "$TEMP_DIR/notebooks"

# Step 2: Export job configuration
echo "2. Exporting job configuration..."
databricks --profile "$DEV_PROFILE" jobs get 123 \
    --output json > "$TEMP_DIR/job_dev.json"

# Step 3: Transform config for production
echo "3. Transforming configuration..."
jq '.settings.name = "PROD - " + .settings.name |
    .settings.tasks[].existing_cluster_id = "prod-cluster-456" |
    .settings.schedule.pause_status = "PAUSED"' \
    "$TEMP_DIR/job_dev.json" > "$TEMP_DIR/job_prod.json"

# Step 4: Import to production
echo "4. Importing to production..."
databricks --profile "$PROD_PROFILE" workspace import-dir \
    "$TEMP_DIR/notebooks" /Production/Project

# Step 5: Update production job
echo "5. Updating production job..."
databricks --profile "$PROD_PROFILE" jobs reset 789 \
    --json-file "$TEMP_DIR/job_prod.json"

# Cleanup
rm -rf "$TEMP_DIR"

echo "✓ Promotion completed successfully"
echo "Note: Production job is PAUSED. Unpause when ready."
```

### 4. Cluster Auto-Scaling Analysis

```bash
#!/bin/bash
# Analyze cluster auto-scaling patterns

PROFILE="production"
CLUSTER_ID="$1"
DAYS=7

if [ -z "$CLUSTER_ID" ]; then
    echo "Usage: $0 <cluster-id>"
    exit 1
fi

echo "=== Cluster Auto-Scaling Analysis ==="
echo "Cluster ID: $CLUSTER_ID"
echo "Analysis period: Last $DAYS days"

# Get cluster events
START_TIME=$(($(date +%s%3N) - (DAYS * 24 * 60 * 60 * 1000)))

databricks --profile "$PROFILE" clusters events "$CLUSTER_ID" \
    --start-time "$START_TIME" \
    --output json | \
    jq '.events[] | select(.type | contains("RESIZED")) | {
        timestamp: (.timestamp / 1000 | strftime("%Y-%m-%d %H:%M:%S")),
        type,
        details: .details
    }' > cluster_resize_events.json

# Count scaling events
SCALE_UP=$(jq '[.[] | select(.type == "UPSIZE_COMPLETED")] | length' cluster_resize_events.json)
SCALE_DOWN=$(jq '[.[] | select(.type == "DOWNSIZE_COMPLETED")] | length' cluster_resize_events.json)

echo ""
echo "Scaling events:"
echo "  Scale up events: $SCALE_UP"
echo "  Scale down events: $SCALE_DOWN"
echo ""
echo "Recommendations:"

if [ "$SCALE_UP" -gt "$SCALE_DOWN" ]; then
    echo "  - Cluster frequently scales up. Consider increasing min_workers."
elif [ "$SCALE_DOWN" -gt "$SCALE_UP" ]; then
    echo "  - Cluster frequently scales down. Consider decreasing max_workers."
else
    echo "  - Auto-scaling configuration appears balanced."
fi
```

---

## Tips & Best Practices

### 1. Use Profiles for Environment Management
```bash
# Set up multiple profiles in ~/.databrickscfg
[dev]
host = https://dev-workspace.cloud.databricks.com
token = dapi...

[staging]
host = https://staging-workspace.cloud.databricks.com
token = dapi...

[prod]
host = https://prod-workspace.cloud.databricks.com
token = dapi...

# Use profiles in commands
databricks --profile dev clusters list
databricks --profile prod jobs run-now 123
```

### 2. JSON Configuration Files for Repeatability
```bash
# Store configurations in version control
git add configs/prod-cluster.json
git add configs/etl-job.json

# Apply configurations
databricks clusters create --json-file configs/prod-cluster.json
databricks jobs create --json-file configs/etl-job.json
```

### 3. Output Format for Automation
```bash
# JSON output for parsing
databricks clusters list --output json | jq '.clusters[]'

# Text output for human reading
databricks clusters list --output text

# Set default output in profile
[DEFAULT]
host = https://workspace.cloud.databricks.com
token = dapi...
output = json
```

### 4. Environment Variables
```bash
# Set environment variables to avoid profile flags
export DATABRICKS_HOST="https://workspace.cloud.databricks.com"
export DATABRICKS_TOKEN="dapi..."
export DATABRICKS_CONFIG_PROFILE="production"

# Now run commands without --profile flag
databricks clusters list
databricks jobs run-now 123
```

### 5. Version Control Integration
```bash
# Keep configurations in Git
configs/
  ├── clusters/
  │   ├── prod-cluster.json
  │   └── dev-cluster.json
  ├── jobs/
  │   ├── etl-job.json
  │   └── ml-job.json
  └── notebooks/
      ├── ETL/
      └── ML/

# Deployment script
#!/bin/bash
git pull origin main
databricks workspace import-dir ./configs/notebooks /Production
databricks jobs reset 123 --json-file ./configs/jobs/etl-job.json
```

### 6. Testing Before Production
```bash
# Validate JSON configurations
cat job-config.json | jq empty

# Test in development first
databricks --profile dev jobs create --json-file job-config.json

# Validate job runs successfully
RUN_ID=$(databricks --profile dev jobs run-now <job-id> --output json | jq -r '.run_id')

# Only promote to production after successful test
if [ "$(get_run_status $RUN_ID)" = "SUCCESS" ]; then
    databricks --profile prod jobs create --json-file job-config.json
fi
```

### 7. Logging & Auditing
```bash
# Enable debug logging
databricks --debug clusters list

# Log to file
databricks --log-file ./databricks.log clusters list

# JSON logging for parsing
databricks --log-level info --log-format json clusters list
```

### 8. Resource Tagging
```json
{
  "cluster_name": "prod-etl-cluster",
  "custom_tags": {
    "Environment": "production",
    "Project": "ETL",
    "CostCenter": "Engineering",
    "Owner": "data-team@example.com",
    "ManagedBy": "terraform"
  }
}
```

---

## Troubleshooting Guide

### Common Issues & Solutions

#### 1. Authentication Errors
```bash
# Problem: "Error: Authentication is not configured"
# Solution: Configure authentication
databricks configure --token

# Or use OAuth
databricks auth login --host https://workspace.cloud.databricks.com

# Verify configuration
cat ~/.databrickscfg

# Test connection
databricks workspace list /
```

#### 2. Token Expiration
```bash
# Problem: "Error: Invalid token"
# Solution: Create new token via UI, then update config
databricks configure --token

# Or create programmatically (if you have valid token)
databricks tokens create --comment "New CLI token" --lifetime-seconds 7776000
```

#### 3. Permission Denied
```bash
# Problem: "Error: Permission denied"
# Solution: Check your permissions
databricks current-user me --output json

# List your groups
databricks groups list --output json | jq '.[] | select(.members[].userName == "your@email.com")'

# Contact workspace admin for proper permissions
```

#### 4. Cluster Not Found
```bash
# Problem: "Error: Cluster <id> does not exist"
# Solution: List all clusters to find correct ID
databricks clusters list --output json | jq '.clusters[] | {id: .cluster_id, name: .cluster_name}'

# Check if cluster was terminated
databricks clusters list --output json | jq '.clusters[] | select(.state == "TERMINATED")'
```

#### 5. Job Run Failures
```bash
# Get detailed error information
databricks jobs get-run-output <run-id>

# Check job run logs
databricks jobs get-run <run-id> --output json | jq '.state'

# View cluster logs if cluster-related
databricks clusters events <cluster-id>
```

#### 6. File Not Found (DBFS)
```bash
# Problem: "Error: File not found"
# Solution: List directory contents
databricks fs ls dbfs:/path/to/directory

# Check file permissions
databricks fs ls -l dbfs:/path/to/file

# Verify correct path format (dbfs:/ prefix)
databricks fs ls dbfs:/mnt/data/
```

#### 7. Rate Limiting




# Databricks Asset Bundles

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


# Databricks Unity Catalog

## Overview

Unity Catalog is Databricks’ unified governance solution for data and AI assets. It provides centralized access control, auditing, lineage, and data discovery across all workspaces.

## Core Concepts & Hierarchy

### Three-Level Namespace

```
catalog.schema.table
catalog.schema.volume
catalog.schema.function
```

### Object Types

- **Catalogs**: Top-level containers for organizing data
- **Schemas**: Logical groupings within catalogs
- **Tables**: Structured data (managed/external)
- **Views**: Virtual tables based on queries
- **Functions**: Reusable SQL/Python functions
- **Volumes**: File storage locations
- **Models**: ML models (MLflow integration)

## Catalog Management

### Creating Catalogs

```sql
-- Create a new catalog
CREATE CATALOG IF NOT EXISTS my_catalog
COMMENT 'Production data catalog';

-- Set properties
ALTER CATALOG my_catalog SET PROPERTIES ('department' = 'analytics');

-- Show catalogs
SHOW CATALOGS;

-- Describe catalog
DESCRIBE CATALOG EXTENDED my_catalog;
```

### Setting Current Catalog

```sql
USE CATALOG my_catalog;
SELECT current_catalog();
```

## Schema Management

### Creating Schemas

```sql
-- Create schema
CREATE SCHEMA IF NOT EXISTS my_catalog.sales_data
COMMENT 'Sales analytics schema'
LOCATION 's3://my-bucket/sales/';

-- Create schema with properties
CREATE SCHEMA my_catalog.hr_data
WITH PROPERTIES (
  'team' = 'people-analytics',
  'env' = 'prod'
);

-- Show schemas
SHOW SCHEMAS IN my_catalog;

-- Set current schema
USE my_catalog.sales_data;
```

## Table Management

### Creating Tables

```sql
-- Managed table (Delta format by default)
CREATE TABLE my_catalog.sales_data.transactions (
  id BIGINT GENERATED ALWAYS AS IDENTITY,
  customer_id STRING,
  amount DECIMAL(10,2),
  transaction_date DATE,
  created_at TIMESTAMP DEFAULT current_timestamp()
) 
COMMENT 'Customer transaction records'
TBLPROPERTIES (
  'delta.autoOptimize.optimizeWrite' = 'true',
  'delta.autoOptimize.autoCompact' = 'true'
);

-- External table
CREATE TABLE my_catalog.sales_data.external_data
USING DELTA
LOCATION 's3://my-bucket/external-data/'
AS SELECT * FROM temp_view;

-- Create table as select (CTAS)
CREATE TABLE my_catalog.sales_data.monthly_summary
AS SELECT 
  DATE_TRUNC('month', transaction_date) as month,
  SUM(amount) as total_amount,
  COUNT(*) as transaction_count
FROM my_catalog.sales_data.transactions
GROUP BY DATE_TRUNC('month', transaction_date);
```

### Table Operations

```sql
-- Show tables
SHOW TABLES IN my_catalog.sales_data;

-- Describe table
DESCRIBE TABLE EXTENDED my_catalog.sales_data.transactions;

-- Table history (Delta feature)
DESCRIBE HISTORY my_catalog.sales_data.transactions;

-- Optimize table
OPTIMIZE my_catalog.sales_data.transactions
ZORDER BY (customer_id, transaction_date);

-- Vacuum old files
VACUUM my_catalog.sales_data.transactions RETAIN 168 HOURS;

-- Clone table
CREATE TABLE my_catalog.sales_data.transactions_backup
DEEP CLONE my_catalog.sales_data.transactions;
```

## Views Management

### Creating Views

```sql
-- Standard view
CREATE VIEW my_catalog.sales_data.high_value_customers AS
SELECT customer_id, SUM(amount) as total_spent
FROM my_catalog.sales_data.transactions
WHERE amount > 1000
GROUP BY customer_id;

-- Materialized view
CREATE MATERIALIZED VIEW my_catalog.sales_data.daily_sales AS
SELECT 
  transaction_date,
  COUNT(*) as transaction_count,
  SUM(amount) as daily_revenue
FROM my_catalog.sales_data.transactions
GROUP BY transaction_date;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW my_catalog.sales_data.daily_sales;
```

## Functions

### Creating Functions

```sql
-- SQL function
CREATE FUNCTION my_catalog.sales_data.calculate_tax(amount DOUBLE, rate DOUBLE)
RETURNS DOUBLE
LANGUAGE SQL
DETERMINISTIC
CONTAINS SQL
COMMENT 'Calculate tax amount'
RETURN amount * rate;

-- Python function
CREATE FUNCTION my_catalog.sales_data.clean_email(email STRING)
RETURNS STRING
LANGUAGE PYTHON
COMMENT 'Clean and validate email addresses'
AS $$
  import re
  if email and re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', email):
    return email.lower().strip()
  return None
$$;

-- Use functions
SELECT 
  customer_id,
  amount,
  my_catalog.sales_data.calculate_tax(amount, 0.08) as tax_amount
FROM my_catalog.sales_data.transactions;
```

## Volumes (File Storage)

### Creating Volumes

```sql
-- Managed volume
CREATE VOLUME IF NOT EXISTS my_catalog.sales_data.data_files
COMMENT 'Storage for CSV and JSON files';

-- External volume
CREATE VOLUME my_catalog.sales_data.external_files
LOCATION 's3://my-bucket/files/'
COMMENT 'External file storage';

-- Show volumes
SHOW VOLUMES IN my_catalog.sales_data;
```

### Working with Volumes

```python
# Python - Upload files to volume
dbutils.fs.cp("/FileStore/shared_uploads/data.csv", 
              "/Volumes/my_catalog/sales_data/data_files/")

# List files in volume
dbutils.fs.ls("/Volumes/my_catalog/sales_data/data_files/")

# Read from volume
df = spark.read.csv("/Volumes/my_catalog/sales_data/data_files/data.csv", 
                    header=True, inferSchema=True)
```

## Access Control & Permissions

### Grant Permissions

```sql
-- Catalog level permissions
GRANT USE CATALOG ON CATALOG my_catalog TO `analysts@company.com`;
GRANT CREATE SCHEMA ON CATALOG my_catalog TO `data-engineers@company.com`;

-- Schema level permissions
GRANT USE SCHEMA ON SCHEMA my_catalog.sales_data TO `analysts@company.com`;
GRANT CREATE TABLE ON SCHEMA my_catalog.sales_data TO `data-engineers@company.com`;

-- Table level permissions
GRANT SELECT ON TABLE my_catalog.sales_data.transactions TO `analysts@company.com`;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE my_catalog.sales_data.transactions 
TO `data-engineers@company.com`;

-- Function permissions
GRANT EXECUTE ON FUNCTION my_catalog.sales_data.calculate_tax TO `analysts@company.com`;

-- Volume permissions
GRANT READ FILES ON VOLUME my_catalog.sales_data.data_files TO `analysts@company.com`;
GRANT WRITE FILES ON VOLUME my_catalog.sales_data.data_files TO `data-engineers@company.com`;
```

### Revoke Permissions

```sql
REVOKE SELECT ON TABLE my_catalog.sales_data.transactions FROM `user@company.com`;
```

### Check Permissions

```sql
-- Show grants for a user
SHOW GRANTS ON TABLE my_catalog.sales_data.transactions;

-- Check current user's permissions
SELECT current_user();
DESCRIBE TABLE my_catalog.sales_data.transactions;
```

## Row-Level Security & Column Masking

### Row-Level Security

```sql
-- Create row filter function
CREATE FUNCTION my_catalog.sales_data.customer_filter(customer_region STRING)
RETURNS BOOLEAN
RETURN 
  CASE 
    WHEN is_member('regional-managers') THEN true
    WHEN is_member('us-analysts') AND customer_region = 'US' THEN true
    WHEN is_member('eu-analysts') AND customer_region = 'EU' THEN true
    ELSE false
  END;

-- Apply row filter to table
ALTER TABLE my_catalog.sales_data.transactions 
SET ROW FILTER my_catalog.sales_data.customer_filter(region);
```

### Column Masking

```sql
-- Create masking function
CREATE FUNCTION my_catalog.sales_data.mask_email(email STRING)
RETURNS STRING
RETURN 
  CASE 
    WHEN is_member('privacy-admins') THEN email
    ELSE regexp_replace(email, '(.{2})(.+)(@.+)', '$1***$3')
  END;

-- Apply column mask
ALTER TABLE my_catalog.sales_data.customers 
ALTER COLUMN email SET MASK my_catalog.sales_data.mask_email;
```

## Data Lineage & Discovery

### Information Schema Queries

```sql
-- Find all tables in catalog
SELECT * FROM system.information_schema.tables 
WHERE table_catalog = 'my_catalog';

-- Find columns containing PII
SELECT table_catalog, table_schema, table_name, column_name
FROM system.information_schema.columns
WHERE column_name ILIKE '%email%' OR column_name ILIKE '%ssn%';

-- Table usage statistics
SELECT * FROM system.information_schema.table_usage
WHERE table_catalog = 'my_catalog'
ORDER BY last_used DESC;
```

### System Tables for Governance

```sql
-- Audit access events
SELECT * FROM system.access.audit
WHERE service_name = 'unityCatalog'
AND action_name = 'SELECT'
ORDER BY event_time DESC;

-- Monitor data access
SELECT user_identity.email, request_params.full_name_arg
FROM system.access.audit
WHERE service_name = 'unityCatalog'
AND action_name = 'SELECT';
```

## Data Quality & Constraints

### Table Constraints

```sql
-- Add constraints to table
ALTER TABLE my_catalog.sales_data.transactions
ADD CONSTRAINT positive_amount CHECK (amount > 0);

ALTER TABLE my_catalog.sales_data.transactions
ADD CONSTRAINT valid_date CHECK (transaction_date >= '2020-01-01');

-- Primary key (informational only in Delta)
ALTER TABLE my_catalog.sales_data.transactions
ADD CONSTRAINT pk_transactions PRIMARY KEY (id);

-- Foreign key (informational only in Delta)
ALTER TABLE my_catalog.sales_data.transactions
ADD CONSTRAINT fk_customer 
FOREIGN KEY (customer_id) REFERENCES my_catalog.sales_data.customers(id);
```

## Delta Lake Integration

### Time Travel

```sql
-- Query table at specific timestamp
SELECT * FROM my_catalog.sales_data.transactions 
TIMESTAMP AS OF '2024-01-01 00:00:00';

-- Query table at specific version
SELECT * FROM my_catalog.sales_data.transactions 
VERSION AS OF 5;

-- Restore table to previous state
RESTORE TABLE my_catalog.sales_data.transactions TO VERSION AS OF 5;
```

### Change Data Feed

```sql
-- Enable change data feed
ALTER TABLE my_catalog.sales_data.transactions 
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);

-- Query changes
SELECT * FROM table_changes('my_catalog.sales_data.transactions', 2, 5);
```

## ML Integration

### Model Registry

```python
# Python - Register model in Unity Catalog
import mlflow

mlflow.set_registry_uri("databricks-uc")

# Log and register model
with mlflow.start_run():
    # Train your model here
    model = train_model()
    
    # Log model
    mlflow.sklearn.log_model(
        model, 
        "model",
        registered_model_name="my_catalog.ml_models.customer_churn"
    )
```

```sql
-- SQL - Grant permissions on models
GRANT EXECUTE ON MODEL my_catalog.ml_models.customer_churn TO `ml-engineers@company.com`;
```

## Python/PySpark Integration

### Working with Unity Catalog in Python

```python
# Set current catalog and schema
spark.sql("USE CATALOG my_catalog")
spark.sql("USE SCHEMA sales_data")

# Read from Unity Catalog table
df = spark.table("my_catalog.sales_data.transactions")

# Write to Unity Catalog table
df.write \
  .mode("overwrite") \
  .option("overwriteSchema", "true") \
  .saveAsTable("my_catalog.sales_data.processed_transactions")

# Dynamic SQL with Unity Catalog
table_name = "my_catalog.sales_data.transactions"
df = spark.sql(f"SELECT * FROM {table_name} WHERE amount > 100")

# Create temporary view for complex operations
df.createOrReplaceTempView("temp_high_value")
result = spark.sql("""
    SELECT customer_id, AVG(amount) as avg_amount
    FROM temp_high_value
    GROUP BY customer_id
""")
```

## Best Practices

### Naming Conventions

```sql
-- Environment-based catalogs
CREATE CATALOG dev_analytics;
CREATE CATALOG staging_analytics;
CREATE CATALOG prod_analytics;

-- Domain-based schemas
CREATE SCHEMA prod_analytics.customer_data;
CREATE SCHEMA prod_analytics.financial_data;
CREATE SCHEMA prod_analytics.marketing_data;

-- Descriptive table names
CREATE TABLE prod_analytics.customer_data.customer_profiles;
CREATE TABLE prod_analytics.customer_data.customer_transactions_daily;
```

### Security Best Practices

```sql
-- Use service principals for automated jobs
GRANT SELECT ON CATALOG prod_analytics TO `service-principal://etl-job-sp`;

-- Implement least privilege access
GRANT USE CATALOG ON CATALOG prod_analytics TO `all-analysts`;
GRANT SELECT ON SCHEMA prod_analytics.public_data TO `all-analysts`;
-- Grant specific table access as needed

-- Use row-level security for multi-tenant scenarios
CREATE FUNCTION prod_analytics.security.tenant_filter(tenant_id STRING)
RETURNS BOOLEAN
RETURN current_user() LIKE '%' || tenant_id || '%' OR is_member('admin');
```

### Performance Optimization

```sql
-- Use Z-ordering for query performance
OPTIMIZE my_catalog.sales_data.transactions
ZORDER BY (customer_id, transaction_date);

-- Partition large tables
CREATE TABLE my_catalog.sales_data.transactions_partitioned (
  id BIGINT,
  customer_id STRING,
  amount DECIMAL(10,2),
  transaction_date DATE
)
PARTITIONED BY (DATE_FORMAT(transaction_date, 'yyyy-MM'));

-- Use liquid clustering for evolving access patterns
CREATE TABLE my_catalog.sales_data.transactions_clustered (
  id BIGINT,
  customer_id STRING,
  amount DECIMAL(10,2),
  transaction_date DATE
)
CLUSTER BY (customer_id, transaction_date);
```

## Troubleshooting Common Issues

### Permission Errors

```sql
-- Check current permissions
SHOW GRANTS ON TABLE my_catalog.sales_data.transactions;

-- Verify current user
SELECT current_user();

-- Check catalog access
SHOW CATALOGS;
```

### Metadata Issues

```sql
-- Refresh table metadata
REFRESH TABLE my_catalog.sales_data.transactions;

-- Repair table (for external tables)
MSCK REPAIR TABLE my_catalog.sales_data.external_table;

-- Check table properties
SHOW TBLPROPERTIES my_catalog.sales_data.transactions;
```

### Common Error Messages

- **“PERMISSION_DENIED”**: User lacks required permissions
- **“SCHEMA_NOT_FOUND”**: Schema doesn’t exist or no access
- **“TABLE_OR_VIEW_NOT_FOUND”**: Table doesn’t exist or no SELECT permission
- **“INVALID_PARAMETER_VALUE.CATALOG_NAME”**: Catalog name format incorrect

## Useful System Functions

```sql
-- Identity and access functions
SELECT current_user();
SELECT current_catalog();
SELECT current_schema();
SELECT is_member('group-name');

-- Metadata functions
SELECT current_timestamp();
SELECT current_date();
SELECT uuid();

-- Information schema queries
SELECT * FROM system.information_schema.catalogs;
SELECT * FROM system.information_schema.schemata;
SELECT * FROM system.information_schema.tables;
SELECT * FROM system.information_schema.columns;
SELECT * FROM system.information_schema.functions;
```

## Monitoring and Observability

### Query History

```sql
-- Query usage patterns
SELECT 
  query_start_time,
  user_name,
  query_text,
  total_duration_ms
FROM system.query.history
WHERE query_text ILIKE '%my_catalog%'
ORDER BY query_start_time DESC;
```

### Billing and Usage

```python
# Python - Monitor compute usage
usage_df = spark.sql("""
    SELECT 
        usage_date,
        workspace_id,
        sku_name,
        usage_quantity,
        dollar_amount
    FROM system.billing.usage
    WHERE usage_date >= '2024-01-01'
    ORDER BY usage_date DESC
""")

display(usage_df)
```

This reference card covers the essential Unity Catalog concepts, syntax, and best practices. Keep it handy for quick reference when working with Databricks Unity Catalog!
