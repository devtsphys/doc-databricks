# doc-databricks

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