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
