# Unity Catalog on Databricks — Complete Reference Card

> **Version:** Databricks Runtime 13.x+ / Unity Catalog GA  
> **Scope:** SQL, Python (PySpark), Terraform/OpenTofu, REST API

-----

## Table of Contents

1. [Architecture & Object Model](#1-architecture--object-model)
1. [Namespace & Naming Conventions](#2-namespace--naming-conventions)
1. [Catalog Management](#3-catalog-management)
1. [Schema (Database) Management](#4-schema-database-management)
1. [Table Management](#5-table-management)
1. [Views & Materialized Views](#6-views--materialized-views)
1. [Volumes](#7-volumes)
1. [Functions (UDFs)](#8-functions-udfs)
1. [Models (ML Model Registry)](#9-models-ml-model-registry)
1. [Privilege & Access Control (ACLs)](#10-privilege--access-control-acls)
1. [Data Lineage & Auditing](#11-data-lineage--auditing)
1. [Delta Sharing](#12-delta-sharing)
1. [External Locations & Storage Credentials](#13-external-locations--storage-credentials)
1. [Connections (Lakehouse Federation)](#14-connections-lakehouse-federation)
1. [Information Schema](#15-information-schema)
1. [Python SDK (databricks-sdk)](#16-python-sdk-databricks-sdk)
1. [Terraform / OpenTofu Resources](#17-terraform--opentofu-resources)
1. [Design Patterns & Best Practices](#18-design-patterns--best-practices)
1. [Complete Methods Reference Table](#19-complete-methods-reference-table)

-----

## 1. Architecture & Object Model

```
Metastore (1 per region, attached to workspace)
└── Catalog
    └── Schema (Database)
        ├── Table (Managed | External | Foreign)
        ├── View
        ├── Materialized View
        ├── Volume (Managed | External)
        ├── Function (SQL UDF | Python UDF | Pandas UDF)
        └── Model (MLflow Registered Model)
```

### Key Concepts

|Concept               |Description                                                                                                                   |
|----------------------|------------------------------------------------------------------------------------------------------------------------------|
|**Metastore**         |Top-level container. One per Databricks account region. Owns storage credentials and lineage.                                 |
|**Catalog**           |Namespace boundary. Maps 1:1 to a logical data domain or environment (dev/tst/prd).                                           |
|**Schema**            |Groups related objects. Owns default managed storage path.                                                                    |
|**Managed Table**     |Delta table whose lifecycle (create/drop) is fully managed by Unity Catalog. Data lives under the schema/catalog storage root.|
|**External Table**    |Registered table pointing to data in an external location. Dropping the table does **not** delete the underlying files.       |
|**Foreign Table**     |Read-only table proxied via Lakehouse Federation from an external system.                                                     |
|**Volume**            |Non-tabular file storage within Unity Catalog governance. Replaces raw DBFS usage.                                            |
|**External Location** |Registered cloud path (S3/ADLS/GCS) + Storage Credential. Gateway for external tables and volumes.                            |
|**Storage Credential**|IAM Role / Service Principal / GCS Service Account used to access cloud storage.                                              |
|**Delta Sharing**     |Open protocol for sharing live data with external recipients without data copy.                                               |

-----

## 2. Namespace & Naming Conventions

```sql
-- Three-part name: catalog.schema.table
SELECT * FROM prod_catalog.sales.orders;

-- Two-part name uses current catalog
USE CATALOG prod_catalog;
SELECT * FROM sales.orders;

-- One-part name uses current catalog + schema
USE CATALOG prod_catalog;
USE SCHEMA sales;
SELECT * FROM orders;
```

### Identifier Rules

- Lowercase alphanumeric and underscores recommended
- Backtick-escape special characters: ``my-catalog``
- Max length: 255 characters per name segment
- Reserved: `system`, `__databricks_internal`

### Environment Pattern (recommended)

```
<domain>_dev    →  analytics_dev.finance.transactions
<domain>_tst    →  analytics_tst.finance.transactions
<domain>_prd    →  analytics_prd.finance.transactions
```

-----

## 3. Catalog Management

### SQL

```sql
-- Create catalog
CREATE CATALOG IF NOT EXISTS my_catalog
  COMMENT 'Analytics domain catalog'
  MANAGED LOCATION 'abfss://container@account.dfs.core.windows.net/uc/my_catalog';

-- Create catalog with provider (Delta Sharing)
CREATE CATALOG IF NOT EXISTS shared_catalog
  USING SHARE provider_name.share_name;

-- Alter catalog
ALTER CATALOG my_catalog SET OWNER TO `data-engineers`;
ALTER CATALOG my_catalog ENABLE PREDICTIVE OPTIMIZATION;

-- Use catalog (sets current catalog context)
USE CATALOG my_catalog;

-- List catalogs
SHOW CATALOGS;
SHOW CATALOGS LIKE 'analytics*';

-- Describe catalog
DESCRIBE CATALOG EXTENDED my_catalog;

-- Drop catalog
DROP CATALOG IF EXISTS my_catalog CASCADE;  -- CASCADE drops all schemas/tables
```

### Python

```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()

# Create
w.catalogs.create(name="my_catalog", comment="Analytics domain")

# List
for cat in w.catalogs.list():
    print(cat.name)

# Update
w.catalogs.update(name="my_catalog", owner="data-engineers")

# Delete
w.catalogs.delete(name="my_catalog", force=True)
```

### PySpark SQL

```python
spark.sql("USE CATALOG my_catalog")
spark.catalog.currentCatalog()          # returns current catalog name
spark.catalog.listCatalogs()            # returns list of CatalogMetadata
```

-----

## 4. Schema (Database) Management

### SQL

```sql
-- Create schema
CREATE SCHEMA IF NOT EXISTS my_catalog.finance
  COMMENT 'Finance domain tables'
  MANAGED LOCATION 'abfss://container@account.dfs.core.windows.net/uc/finance';

-- Alter schema
ALTER SCHEMA my_catalog.finance SET OWNER TO `finance-team`;
ALTER SCHEMA my_catalog.finance ENABLE PREDICTIVE OPTIMIZATION;

-- Use schema
USE SCHEMA my_catalog.finance;
USE my_catalog.finance;  -- shorthand

-- List schemas
SHOW SCHEMAS IN my_catalog;
SHOW DATABASES IN my_catalog LIKE 'fin*';

-- Describe schema
DESCRIBE SCHEMA EXTENDED my_catalog.finance;

-- Drop schema
DROP SCHEMA IF EXISTS my_catalog.finance CASCADE;
DROP SCHEMA IF EXISTS my_catalog.finance RESTRICT;  -- fails if not empty (default)
```

### Python (SDK)

```python
w.schemas.create(name="finance", catalog_name="my_catalog", comment="Finance tables")
w.schemas.list(catalog_name="my_catalog")
w.schemas.update(full_name="my_catalog.finance", owner="finance-team")
w.schemas.delete(full_name="my_catalog.finance", force=True)
```

-----

## 5. Table Management

### Managed Tables

```sql
-- Create managed Delta table
CREATE TABLE IF NOT EXISTS my_catalog.finance.invoices (
  invoice_id   BIGINT     NOT NULL,
  customer_id  BIGINT,
  amount       DECIMAL(18, 2),
  invoice_date DATE,
  status       STRING     DEFAULT 'PENDING',
  created_at   TIMESTAMP  DEFAULT current_timestamp()
)
USING DELTA
COMMENT 'Invoice records'
TBLPROPERTIES (
  'delta.enableChangeDataFeed' = 'true',
  'delta.autoOptimize.optimizeWrite' = 'true',
  'delta.autoOptimize.autoCompact' = 'true'
)
CLUSTER BY (invoice_date, customer_id);   -- Liquid Clustering (DBR 13.3+)

-- Create from SELECT (CTAS)
CREATE TABLE my_catalog.finance.high_value_invoices
AS SELECT * FROM my_catalog.finance.invoices WHERE amount > 10000;

-- Partitioned table (classic)
CREATE TABLE my_catalog.finance.invoices_partitioned (
  invoice_id  BIGINT,
  amount      DECIMAL(18, 2),
  year        INT,
  month       INT
)
USING DELTA
PARTITIONED BY (year, month);
```

### External Tables

```sql
-- Requires an External Location to be defined first
CREATE TABLE IF NOT EXISTS my_catalog.raw.sensor_data
USING DELTA
LOCATION 'abfss://raw@account.dfs.core.windows.net/sensors/'
COMMENT 'Sensor data — external';

-- External table from Parquet
CREATE TABLE my_catalog.raw.csv_landing
USING CSV
OPTIONS (header = 'true', inferSchema = 'true')
LOCATION 'abfss://landing@account.dfs.core.windows.net/csv/';
```

### Alter Table

```sql
-- Rename table
ALTER TABLE my_catalog.finance.invoices RENAME TO my_catalog.finance.invoice_records;

-- Add column
ALTER TABLE my_catalog.finance.invoices ADD COLUMN notes STRING AFTER status;
ALTER TABLE my_catalog.finance.invoices ADD COLUMN (tax DECIMAL(10,2), discount DECIMAL(10,2));

-- Change column
ALTER TABLE my_catalog.finance.invoices ALTER COLUMN amount TYPE DECIMAL(20, 2);
ALTER TABLE my_catalog.finance.invoices ALTER COLUMN notes SET NOT NULL;
ALTER TABLE my_catalog.finance.invoices ALTER COLUMN notes DROP NOT NULL;
ALTER TABLE my_catalog.finance.invoices ALTER COLUMN notes COMMENT 'Free-form notes';

-- Rename column
ALTER TABLE my_catalog.finance.invoices RENAME COLUMN notes TO remarks;

-- Drop column
ALTER TABLE my_catalog.finance.invoices DROP COLUMN remarks;

-- Table properties
ALTER TABLE my_catalog.finance.invoices SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');
ALTER TABLE my_catalog.finance.invoices UNSET TBLPROPERTIES ('delta.enableChangeDataFeed');

-- Owner
ALTER TABLE my_catalog.finance.invoices SET OWNER TO `finance-team`;

-- Add/drop constraint
ALTER TABLE my_catalog.finance.invoices ADD CONSTRAINT invoices_pk PRIMARY KEY (invoice_id);
ALTER TABLE my_catalog.finance.invoices ADD CONSTRAINT fk_customer
  FOREIGN KEY (customer_id) REFERENCES my_catalog.crm.customers(customer_id);
ALTER TABLE my_catalog.finance.invoices DROP CONSTRAINT invoices_pk;

-- Change comment
COMMENT ON TABLE my_catalog.finance.invoices IS 'Updated invoice table';
COMMENT ON COLUMN my_catalog.finance.invoices.amount IS 'Net invoice amount in USD';
```

### Tags

```sql
-- Table tags
ALTER TABLE my_catalog.finance.invoices SET TAGS ('pii' = 'true', 'domain' = 'finance');
ALTER TABLE my_catalog.finance.invoices UNSET TAGS ('pii');

-- Column tags
ALTER TABLE my_catalog.finance.invoices
  ALTER COLUMN customer_id SET TAGS ('pii' = 'true', 'classification' = 'confidential');
ALTER TABLE my_catalog.finance.invoices
  ALTER COLUMN customer_id UNSET TAGS ('pii');
```

### Delta Operations

```sql
-- Optimize (file compaction)
OPTIMIZE my_catalog.finance.invoices;
OPTIMIZE my_catalog.finance.invoices WHERE invoice_date >= '2024-01-01';
OPTIMIZE my_catalog.finance.invoices ZORDER BY (customer_id, invoice_date);

-- Vacuum (remove old files)
VACUUM my_catalog.finance.invoices RETAIN 168 HOURS;  -- 7 days
VACUUM my_catalog.finance.invoices DRY RUN;

-- History
DESCRIBE HISTORY my_catalog.finance.invoices;
DESCRIBE HISTORY my_catalog.finance.invoices LIMIT 5;

-- Time travel
SELECT * FROM my_catalog.finance.invoices VERSION AS OF 10;
SELECT * FROM my_catalog.finance.invoices TIMESTAMP AS OF '2024-06-01 00:00:00';

-- Restore
RESTORE TABLE my_catalog.finance.invoices TO VERSION AS OF 5;
RESTORE TABLE my_catalog.finance.invoices TO TIMESTAMP AS OF '2024-06-01';

-- Clone
CREATE TABLE my_catalog.finance.invoices_backup CLONE my_catalog.finance.invoices;
CREATE OR REPLACE TABLE my_catalog.finance.invoices_shallow SHALLOW CLONE my_catalog.finance.invoices;
```

### Describe / Show

```sql
DESCRIBE TABLE my_catalog.finance.invoices;
DESCRIBE TABLE EXTENDED my_catalog.finance.invoices;
DESCRIBE DETAIL my_catalog.finance.invoices;

SHOW TABLES IN my_catalog.finance;
SHOW TABLES IN my_catalog.finance LIKE 'inv*';
SHOW TABLE EXTENDED IN my_catalog.finance LIKE 'invoices';
SHOW TBLPROPERTIES my_catalog.finance.invoices;
SHOW COLUMNS IN my_catalog.finance.invoices;
SHOW CREATE TABLE my_catalog.finance.invoices;

-- Partitions (classic partitioned tables only)
SHOW PARTITIONS my_catalog.finance.invoices_partitioned;
```

### Python (PySpark)

```python
# Create
spark.sql("""
  CREATE TABLE IF NOT EXISTS my_catalog.finance.invoices (
    invoice_id BIGINT, amount DECIMAL(18,2)
  ) USING DELTA
""")

# Write DataFrame
df.write.format("delta") \
  .mode("overwrite") \
  .option("overwriteSchema", "true") \
  .saveAsTable("my_catalog.finance.invoices")

# Append
df.write.format("delta").mode("append").saveAsTable("my_catalog.finance.invoices")

# Read
df = spark.table("my_catalog.finance.invoices")
df = spark.read.table("my_catalog.finance.invoices")

# MERGE (upsert)
from delta.tables import DeltaTable
target = DeltaTable.forName(spark, "my_catalog.finance.invoices")
target.alias("t").merge(
    updates_df.alias("s"),
    "t.invoice_id = s.invoice_id"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .whenNotMatchedBySourceDelete() \
 .execute()

# Schema evolution
df.write.format("delta") \
  .mode("append") \
  .option("mergeSchema", "true") \
  .saveAsTable("my_catalog.finance.invoices")

# Catalog API
spark.catalog.listTables("my_catalog.finance")
spark.catalog.tableExists("my_catalog.finance.invoices")
spark.catalog.recoverPartitions("my_catalog.finance.invoices_partitioned")
spark.catalog.refreshTable("my_catalog.finance.invoices")
spark.catalog.clearCache()
```

-----

## 6. Views & Materialized Views

### Standard Views

```sql
-- Create view
CREATE VIEW IF NOT EXISTS my_catalog.finance.v_pending_invoices
  COMMENT 'Pending invoices only'
AS
  SELECT invoice_id, customer_id, amount, invoice_date
  FROM my_catalog.finance.invoices
  WHERE status = 'PENDING';

-- Replace view
CREATE OR REPLACE VIEW my_catalog.finance.v_pending_invoices AS
  SELECT * FROM my_catalog.finance.invoices WHERE status = 'PENDING';

-- Dynamic view (row-level security)
CREATE VIEW my_catalog.finance.v_invoices_secured AS
  SELECT *
  FROM my_catalog.finance.invoices
  WHERE
    is_account_group_member('finance-admins')
    OR customer_id IN (
      SELECT customer_id
      FROM my_catalog.security.user_customer_map
      WHERE user_name = current_user()
    );

-- Dynamic view (column masking)
CREATE VIEW my_catalog.finance.v_invoices_masked AS
  SELECT
    invoice_id,
    CASE WHEN is_account_group_member('finance-admins')
         THEN customer_id
         ELSE -1
    END AS customer_id,
    amount
  FROM my_catalog.finance.invoices;

ALTER VIEW my_catalog.finance.v_pending_invoices SET OWNER TO `finance-team`;
DROP VIEW IF EXISTS my_catalog.finance.v_pending_invoices;
SHOW VIEWS IN my_catalog.finance;
```

### Materialized Views (DLT)

```sql
-- Only supported inside Delta Live Tables pipelines
CREATE MATERIALIZED VIEW my_catalog.finance.mv_daily_revenue AS
  SELECT
    invoice_date,
    SUM(amount) AS total_revenue,
    COUNT(*) AS invoice_count
  FROM LIVE.invoices
  WHERE status = 'PAID'
  GROUP BY invoice_date;
```

-----

## 7. Volumes

Volumes provide governed access to files (CSV, JSON, images, models, etc.) without raw DBFS.

```sql
-- Managed volume (data under catalog/schema managed storage)
CREATE VOLUME IF NOT EXISTS my_catalog.raw.landing
  COMMENT 'Landing zone for raw files';

-- External volume (points to existing cloud path)
CREATE EXTERNAL VOLUME IF NOT EXISTS my_catalog.raw.archive
  LOCATION 'abfss://archive@account.dfs.core.windows.net/data/'
  COMMENT 'Long-term archive';

-- Paths (use /Volumes/ prefix)
-- /Volumes/<catalog>/<schema>/<volume>/path/to/file.csv

-- List
SHOW VOLUMES IN my_catalog.raw;
DESCRIBE VOLUME my_catalog.raw.landing;

-- Drop
DROP VOLUME IF EXISTS my_catalog.raw.landing;
```

### Python / Shell

```python
# Read file from volume
df = spark.read.csv("/Volumes/my_catalog/raw/landing/data.csv", header=True)

# Write to volume
df.write.csv("/Volumes/my_catalog/raw/landing/output/")

# dbutils
dbutils.fs.ls("/Volumes/my_catalog/raw/landing/")
dbutils.fs.cp("dbfs:/FileStore/file.csv", "/Volumes/my_catalog/raw/landing/file.csv")

# Python file I/O (DBR 13.2+)
with open("/Volumes/my_catalog/raw/landing/config.json") as f:
    config = json.load(f)
```

-----

## 8. Functions (UDFs)

### SQL Functions

```sql
-- Scalar SQL UDF
CREATE OR REPLACE FUNCTION my_catalog.utils.mask_email(email STRING)
RETURNS STRING
LANGUAGE SQL
COMMENT 'Masks email address for PII protection'
AS (
  CONCAT(LEFT(email, 2), '***@', SPLIT(email, '@')[1])
);

-- Table-valued function (TVF)
CREATE OR REPLACE FUNCTION my_catalog.utils.get_recent_invoices(days INT)
RETURNS TABLE (invoice_id BIGINT, amount DECIMAL(18,2), invoice_date DATE)
LANGUAGE SQL
AS (
  SELECT invoice_id, amount, invoice_date
  FROM my_catalog.finance.invoices
  WHERE invoice_date >= CURRENT_DATE - MAKE_INTERVAL(0, 0, days)
);

-- Call TVF
SELECT * FROM my_catalog.utils.get_recent_invoices(30);
```

### Python UDFs

```python
# Register Python UDF via SQL
spark.sql("""
  CREATE OR REPLACE FUNCTION my_catalog.utils.to_title_case(s STRING)
  RETURNS STRING
  LANGUAGE PYTHON
  AS $$
    return s.title() if s else None
  $$
""")

# Pandas UDF (vectorized) — register via API
import pandas as pd
from pyspark.sql.functions import pandas_udf
from pyspark.sql.types import StringType

@pandas_udf(StringType())
def to_title_case_udf(s: pd.Series) -> pd.Series:
    return s.str.title()

spark.udf.register("to_title_case", to_title_case_udf)

# Apply UDF
df = spark.table("my_catalog.finance.invoices")
df.withColumn("name_title", spark.sql("my_catalog.utils.to_title_case(name)"))
```

### Manage Functions

```sql
SHOW FUNCTIONS IN my_catalog.utils;
SHOW FUNCTIONS IN my_catalog.utils LIKE 'mask*';
DESCRIBE FUNCTION my_catalog.utils.mask_email;
DESCRIBE FUNCTION EXTENDED my_catalog.utils.mask_email;
DROP FUNCTION IF EXISTS my_catalog.utils.mask_email;
ALTER FUNCTION my_catalog.utils.mask_email SET OWNER TO `data-engineers`;
```

-----

## 9. Models (ML Model Registry)

Unity Catalog hosts MLflow Registered Models using three-level namespace.

```python
import mlflow
mlflow.set_registry_uri("databricks-uc")

# Log and register model
with mlflow.start_run():
    mlflow.sklearn.log_model(
        model,
        artifact_path="model",
        registered_model_name="my_catalog.ml.fraud_detector"
    )

# Load model
model = mlflow.pyfunc.load_model("models:/my_catalog.ml.fraud_detector/1")

# Champion/challenger aliases
client = mlflow.MlflowClient()
client.set_registered_model_alias(
    name="my_catalog.ml.fraud_detector",
    alias="champion",
    version="3"
)
client.set_registered_model_alias(
    name="my_catalog.ml.fraud_detector",
    alias="challenger",
    version="4"
)

# Load by alias
model = mlflow.pyfunc.load_model("models:/my_catalog.ml.fraud_detector@champion")

# Transition (UC uses aliases, not stages)
client.delete_registered_model_alias("my_catalog.ml.fraud_detector", "champion")

# List versions
client.search_model_versions(filter_string="name='my_catalog.ml.fraud_detector'")
```

### SQL

```sql
SHOW MODELS IN my_catalog.ml;
DESCRIBE MODEL my_catalog.ml.fraud_detector;
DROP MODEL IF EXISTS my_catalog.ml.fraud_detector;
```

-----

## 10. Privilege & Access Control (ACLs)

### Privilege Hierarchy

```
Metastore → Catalog → Schema → Table/View/Volume/Function/Model
```

Privileges flow downward. You must `USE CATALOG` and `USE SCHEMA` before accessing objects.

### Core Privileges by Object

|Securable         |Privilege                  |Description                                  |
|------------------|---------------------------|---------------------------------------------|
|Metastore         |`CREATE CATALOG`           |Create top-level catalogs                    |
|Metastore         |`CREATE CONNECTION`        |Create Lakehouse Federation connections      |
|Metastore         |`CREATE EXTERNAL LOCATION` |Register external paths                      |
|Metastore         |`CREATE STORAGE CREDENTIAL`|Register cloud credentials                   |
|Metastore         |`MANAGE`                   |Full metastore admin                         |
|Catalog           |`USE CATALOG`              |Required to access any object in catalog     |
|Catalog           |`CREATE SCHEMA`            |Create schemas in catalog                    |
|Catalog           |`CREATE TABLE`             |Create tables (when granted at catalog level)|
|Catalog           |`ALL PRIVILEGES`           |All catalog-level privileges                 |
|Schema            |`USE SCHEMA`               |Required to access any object in schema      |
|Schema            |`CREATE TABLE`             |Create tables in schema                      |
|Schema            |`CREATE VIEW`              |Create views in schema                       |
|Schema            |`CREATE FUNCTION`          |Create UDFs in schema                        |
|Schema            |`CREATE VOLUME`            |Create volumes in schema                     |
|Schema            |`CREATE MODEL`             |Register ML models in schema                 |
|Table             |`SELECT`                   |Read table/view data                         |
|Table             |`MODIFY`                   |INSERT, UPDATE, DELETE, MERGE                |
|Table             |`ALL PRIVILEGES`           |SELECT + MODIFY + ownership operations       |
|Volume            |`READ VOLUME`              |Read files from volume                       |
|Volume            |`WRITE VOLUME`             |Write files to volume                        |
|Function          |`EXECUTE`                  |Call UDF                                     |
|Model             |`EXECUTE`                  |Load/invoke model                            |
|External Location |`READ FILES`               |Read from cloud path                         |
|External Location |`WRITE FILES`              |Write to cloud path                          |
|External Location |`CREATE EXTERNAL TABLE`    |Register external tables on path             |
|External Location |`CREATE MANAGED STORAGE`   |Create managed storage on path               |
|Storage Credential|`CREATE EXTERNAL LOCATION` |Use credential for external locations        |
|Storage Credential|`CREATE EXTERNAL TABLE`    |Use credential directly for tables           |
|Connection        |`CREATE FOREIGN CATALOG`   |Create foreign catalog from connection       |

### GRANT / REVOKE

```sql
-- Grant on catalog
GRANT USE CATALOG ON CATALOG my_catalog TO `analysts`;
GRANT CREATE SCHEMA ON CATALOG my_catalog TO `data-engineers`;

-- Grant on schema
GRANT USE SCHEMA, SELECT ON SCHEMA my_catalog.finance TO `analysts`;
GRANT CREATE TABLE, CREATE VIEW ON SCHEMA my_catalog.finance TO `data-engineers`;

-- Grant on table
GRANT SELECT ON TABLE my_catalog.finance.invoices TO `analysts`;
GRANT MODIFY  ON TABLE my_catalog.finance.invoices TO `data-engineers`;
GRANT ALL PRIVILEGES ON TABLE my_catalog.finance.invoices TO `finance-team`;

-- Grant on view
GRANT SELECT ON VIEW my_catalog.finance.v_pending_invoices TO `analysts`;

-- Grant on volume
GRANT READ VOLUME  ON VOLUME my_catalog.raw.landing TO `data-engineers`;
GRANT WRITE VOLUME ON VOLUME my_catalog.raw.landing TO `etl-service-principal`;

-- Grant on function
GRANT EXECUTE ON FUNCTION my_catalog.utils.mask_email TO `analysts`;

-- Grant on external location
GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION my_ext_location TO `data-engineers`;
GRANT CREATE EXTERNAL TABLE   ON EXTERNAL LOCATION my_ext_location TO `data-engineers`;

-- Grant on storage credential
GRANT CREATE EXTERNAL LOCATION ON STORAGE CREDENTIAL my_cred TO `data-engineers`;

-- Revoke
REVOKE SELECT ON TABLE my_catalog.finance.invoices FROM `analysts`;

-- Show grants
SHOW GRANTS ON TABLE my_catalog.finance.invoices;
SHOW GRANTS ON CATALOG my_catalog;
SHOW GRANTS ON SCHEMA my_catalog.finance;
SHOW GRANTS TO `analysts`;          -- what a principal has been granted
SHOW GRANTS ON VOLUME my_catalog.raw.landing;
SHOW GRANTS ON EXTERNAL LOCATION my_ext_location;

-- Change owner
ALTER TABLE  my_catalog.finance.invoices    SET OWNER TO `finance-team`;
ALTER SCHEMA my_catalog.finance             SET OWNER TO `finance-team`;
ALTER CATALOG my_catalog                    SET OWNER TO `platform-team`;
```

### Column-Level Security (Column Masks)

```sql
-- Define masking policy as a function
CREATE OR REPLACE FUNCTION my_catalog.security.mask_ssn(ssn STRING)
RETURNS STRING
RETURN CASE
  WHEN is_account_group_member('hr-admins') THEN ssn
  ELSE CONCAT('***-**-', RIGHT(ssn, 4))
END;

-- Apply column mask to table column
ALTER TABLE my_catalog.hr.employees
  ALTER COLUMN ssn SET MASK my_catalog.security.mask_ssn;

-- Remove column mask
ALTER TABLE my_catalog.hr.employees
  ALTER COLUMN ssn DROP MASK;
```

### Row Filters

```sql
-- Define row filter function
CREATE OR REPLACE FUNCTION my_catalog.security.invoices_row_filter(region STRING)
RETURNS BOOLEAN
RETURN is_account_group_member('finance-global')
       OR region = current_user_region();  -- custom lookup UDF

-- Apply row filter to table
ALTER TABLE my_catalog.finance.invoices
  SET ROW FILTER my_catalog.security.invoices_row_filter ON (region);

-- Remove row filter
ALTER TABLE my_catalog.finance.invoices DROP ROW FILTER;
```

-----

## 11. Data Lineage & Auditing

### Lineage (automatic)

Unity Catalog automatically captures column-level lineage for SQL and DataFrame operations.

```python
# Lineage is captured automatically — no code required.
# View in Catalog Explorer UI: Catalog → Table → Lineage tab

# Or query system tables
spark.sql("""
  SELECT *
  FROM system.access.table_lineage
  WHERE target_table_full_name = 'my_catalog.finance.invoices'
  ORDER BY event_time DESC
  LIMIT 100
""")

spark.sql("""
  SELECT *
  FROM system.access.column_lineage
  WHERE target_table_full_name = 'my_catalog.finance.invoices'
    AND target_column_name = 'amount'
""")
```

### Audit Logs (System Tables)

```sql
-- All audit events (requires system catalog access)
SELECT * FROM system.access.audit
WHERE service_name = 'unityCatalog'
  AND action_name IN ('getTable', 'createTable', 'deleteTable')
  AND event_date >= CURRENT_DATE - 7
ORDER BY event_time DESC;

-- Data access events
SELECT
  user_identity.email,
  action_name,
  request_params.full_name_arg,
  event_time
FROM system.access.audit
WHERE action_name = 'commandSubmit'
  AND contains(request_params.commandText, 'my_catalog.finance')
  AND event_date = CURRENT_DATE;

-- Table usage summary
SELECT
  request_params.full_name_arg AS table_name,
  COUNT(*) AS access_count,
  COUNT(DISTINCT user_identity.email) AS unique_users
FROM system.access.audit
WHERE action_name = 'getTable'
  AND event_date >= CURRENT_DATE - 30
GROUP BY 1
ORDER BY 2 DESC;
```

-----

## 12. Delta Sharing

### Provider Side

```sql
-- Create share
CREATE SHARE IF NOT EXISTS finance_share
  COMMENT 'Finance data shared with partners';

-- Add table to share
ALTER SHARE finance_share
  ADD TABLE my_catalog.finance.invoices
  COMMENT 'Invoice summary data';

-- Add table with partition filter
ALTER SHARE finance_share
  ADD TABLE my_catalog.finance.invoices
  PARTITION (year = '2024');

-- Add table with column selection
ALTER SHARE finance_share
  ADD TABLE my_catalog.finance.invoices
  SELECT invoice_id, amount, invoice_date;  -- column subset

-- Add schema to share (all current + future tables)
ALTER SHARE finance_share ADD SCHEMA my_catalog.finance;

-- Add volume
ALTER SHARE finance_share ADD VOLUME my_catalog.raw.landing;

-- Create recipient
CREATE RECIPIENT IF NOT EXISTS partner_recipient
  COMMENT 'External partner A';

-- Grant access
GRANT SELECT ON SHARE finance_share TO RECIPIENT partner_recipient;

-- Show activation link
DESCRIBE RECIPIENT partner_recipient;

-- Manage
SHOW SHARES;
DESCRIBE SHARE finance_share;
SHOW ALL IN SHARE finance_share;
SHOW RECIPIENTS;
DROP SHARE IF EXISTS finance_share;
DROP RECIPIENT IF EXISTS partner_recipient;
```

### Consumer Side

```sql
-- Register provider
CREATE PROVIDER IF NOT EXISTS our_data_provider
  DELTA SHARING RECIPIENT TOKEN '<activation_token>';

-- List available shares
SHOW SHARES IN PROVIDER our_data_provider;

-- Create catalog from share
CREATE CATALOG IF NOT EXISTS shared_finance
  USING SHARE our_data_provider.finance_share;

-- Query shared tables (read-only)
SELECT * FROM shared_finance.finance.invoices;
```

-----

## 13. External Locations & Storage Credentials

### Storage Credentials

```sql
-- Azure: Service Principal
CREATE STORAGE CREDENTIAL IF NOT EXISTS adls_cred
  AZURE SERVICE CREDENTIAL (
    DIRECTORY_ID   '<tenant_id>',
    APPLICATION_ID '<client_id>',
    CLIENT_SECRET  '<client_secret>'
  )
  COMMENT 'ADLS Gen2 credential';

-- Azure: Managed Identity
CREATE STORAGE CREDENTIAL IF NOT EXISTS adls_mi_cred
  AZURE MANAGED IDENTITY (
    CREDENTIAL_ID '<managed-identity-resource-id>'
  );

-- AWS: IAM Role
CREATE STORAGE CREDENTIAL IF NOT EXISTS s3_cred
  AWS IAM ROLE ARN 'arn:aws:iam::123456789:role/DatabricksUCRole';

-- Validate
VALIDATE STORAGE CREDENTIAL adls_cred;

-- Manage
SHOW STORAGE CREDENTIALS;
DESCRIBE STORAGE CREDENTIAL adls_cred;
ALTER STORAGE CREDENTIAL adls_cred SET OWNER TO `platform-team`;
DROP STORAGE CREDENTIAL IF EXISTS adls_cred;
```

### External Locations

```sql
-- Create external location
CREATE EXTERNAL LOCATION IF NOT EXISTS raw_data
  URL 'abfss://raw@myaccount.dfs.core.windows.net/'
  WITH (STORAGE CREDENTIAL adls_cred)
  COMMENT 'Raw zone ADLS path';

-- S3 example
CREATE EXTERNAL LOCATION IF NOT EXISTS s3_raw
  URL 's3://my-bucket/raw/'
  WITH (STORAGE CREDENTIAL s3_cred);

-- Validate
VALIDATE EXTERNAL LOCATION raw_data;

-- Manage
SHOW EXTERNAL LOCATIONS;
DESCRIBE EXTERNAL LOCATION raw_data;
ALTER EXTERNAL LOCATION raw_data SET OWNER TO `platform-team`;
DROP EXTERNAL LOCATION IF EXISTS raw_data;
```

-----

## 14. Connections (Lakehouse Federation)

```sql
-- PostgreSQL connection
CREATE CONNECTION IF NOT EXISTS postgres_prod
  TYPE POSTGRESQL
  OPTIONS (
    host       'pg-prod.company.com',
    port       '5432',
    user       'databricks_ro',
    password   secret('scope', 'pg_password')
  )
  COMMENT 'Production PostgreSQL';

-- Supported types: POSTGRESQL, MYSQL, SNOWFLAKE, REDSHIFT, SQLDATABASES (SQL Server), BIGQUERY

-- Create foreign catalog
CREATE FOREIGN CATALOG IF NOT EXISTS pg_prod_catalog
  USING CONNECTION postgres_prod
  OPTIONS (database 'prod_db');

-- Query foreign table (read-only, pushed-down)
SELECT * FROM pg_prod_catalog.public.customers LIMIT 100;

-- Manage
SHOW CONNECTIONS;
DESCRIBE CONNECTION postgres_prod;
DROP CONNECTION IF EXISTS postgres_prod;
```

-----

## 15. Information Schema

Every catalog exposes `information_schema` — ANSI SQL standard.

```sql
-- Tables
SELECT table_catalog, table_schema, table_name, table_type, created
FROM my_catalog.information_schema.tables
WHERE table_schema = 'finance';

-- Columns
SELECT table_name, column_name, data_type, is_nullable, column_default
FROM my_catalog.information_schema.columns
WHERE table_schema = 'finance'
  AND table_name = 'invoices';

-- Table privileges
SELECT grantee, privilege_type, is_grantable
FROM my_catalog.information_schema.table_privileges
WHERE table_schema = 'finance';

-- Schema privileges
SELECT * FROM my_catalog.information_schema.schema_privileges
WHERE schema_name = 'finance';

-- Views
SELECT * FROM my_catalog.information_schema.views
WHERE table_schema = 'finance';

-- Routines (UDFs)
SELECT routine_name, routine_type, data_type
FROM my_catalog.information_schema.routines
WHERE routine_schema = 'utils';

-- Constraints
SELECT * FROM my_catalog.information_schema.table_constraints
WHERE table_schema = 'finance';

-- Column constraints
SELECT * FROM my_catalog.information_schema.constraint_column_usage
WHERE table_schema = 'finance';

-- Tags
SELECT * FROM my_catalog.information_schema.table_tags
WHERE table_schema = 'finance';

SELECT * FROM my_catalog.information_schema.column_tags
WHERE table_name = 'invoices';
```

### System Catalog (`system.*`)

```sql
-- All tables across all catalogs
SELECT * FROM system.information_schema.tables;

-- Billable usage
SELECT * FROM system.billing.usage WHERE usage_date >= CURRENT_DATE - 30;

-- Query history
SELECT * FROM system.query.history
WHERE user_name = 'user@company.com'
  AND start_time >= CURRENT_DATE - 7;

-- Cluster events
SELECT * FROM system.compute.clusters;

-- Marketplace listings
SELECT * FROM system.marketplace.listing_access_events;
```

-----

## 16. Python SDK (databricks-sdk)

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.catalog import (
    CatalogType, TableType, DataSourceFormat,
    SecurableType, Privilege, PrivilegeAssignment
)

w = WorkspaceClient()  # uses DATABRICKS_HOST + DATABRICKS_TOKEN env vars

# --- Catalogs ---
cat = w.catalogs.create(name="my_catalog", comment="Test")
w.catalogs.update(name="my_catalog", owner="admins")
[c.name for c in w.catalogs.list()]
w.catalogs.delete(name="my_catalog", force=True)

# --- Schemas ---
sch = w.schemas.create(name="finance", catalog_name="my_catalog")
w.schemas.update(full_name="my_catalog.finance", comment="Finance tables")
[s.name for s in w.schemas.list(catalog_name="my_catalog")]
w.schemas.delete(full_name="my_catalog.finance", force=True)

# --- Tables ---
tbl = w.tables.get(full_name="my_catalog.finance.invoices")
[t.name for t in w.tables.list(catalog_name="my_catalog", schema_name="finance")]
w.tables.delete(full_name="my_catalog.finance.old_table")
w.tables.update(full_name="my_catalog.finance.invoices", owner="finance-team")

# --- Grants ---
w.grants.update(
    securable_type=SecurableType.TABLE,
    full_name="my_catalog.finance.invoices",
    changes=[
        PermissionsChange(
            add=[Privilege.SELECT],
            principal="analysts"
        )
    ]
)
grants = w.grants.get(securable_type=SecurableType.TABLE, full_name="my_catalog.finance.invoices")
eff = w.grants.get_effective(securable_type=SecurableType.TABLE, full_name="my_catalog.finance.invoices")

# --- External Locations ---
w.external_locations.create(
    name="raw_data",
    url="abfss://raw@account.dfs.core.windows.net/",
    credential_name="adls_cred",
    comment="Raw zone"
)

# --- Storage Credentials ---
w.storage_credentials.list()
w.storage_credentials.validate(name="adls_cred")

# --- Volumes ---
w.volumes.create(catalog_name="my_catalog", schema_name="raw", name="landing",
                 volume_type=VolumeType.MANAGED)
[v.name for v in w.volumes.list(catalog_name="my_catalog", schema_name="raw")]

# --- Metastore ---
w.metastores.current()
w.metastores.list()
w.metastores.assign(workspace_id=123456, metastore_id="abc-123", default_catalog_name="my_catalog")
```

-----

## 17. Terraform / OpenTofu Resources

```hcl
# --- Provider ---
terraform {
  required_providers {
    databricks = { source = "databricks/databricks" }
  }
}

provider "databricks" {
  host  = var.databricks_host
  token = var.databricks_token
}

# --- Storage Credential ---
resource "databricks_storage_credential" "adls" {
  name    = "adls_service_principal"
  comment = "ADLS Gen2 access"
  azure_service_principal {
    directory_id   = var.tenant_id
    application_id = var.client_id
    client_secret  = var.client_secret
  }
}

# --- External Location ---
resource "databricks_external_location" "raw" {
  name            = "raw_data_location"
  url             = "abfss://raw@${var.storage_account}.dfs.core.windows.net/"
  credential_name = databricks_storage_credential.adls.name
  comment         = "Raw zone"
}

# --- Catalog ---
resource "databricks_catalog" "analytics" {
  name           = "analytics_prd"
  comment        = "Production analytics catalog"
  storage_root   = "${databricks_external_location.raw.url}catalogs/analytics_prd"
  properties     = { environment = "production", domain = "analytics" }
}

# --- Schema ---
resource "databricks_schema" "finance" {
  catalog_name   = databricks_catalog.analytics.name
  name           = "finance"
  comment        = "Finance data"
  storage_root   = "${databricks_external_location.raw.url}schemas/finance"
}

# --- Table (external) ---
resource "databricks_table" "invoices" {
  catalog_name   = databricks_catalog.analytics.name
  schema_name    = databricks_schema.finance.name
  name           = "invoices"
  table_type     = "EXTERNAL"
  data_source_format = "DELTA"
  storage_location = "${databricks_external_location.raw.url}data/invoices"
  comment        = "Invoice records"

  column {
    name     = "invoice_id"
    type_text = "bigint"
    type_name = "LONG"
    position  = 0
    nullable  = false
  }
  column {
    name     = "amount"
    type_text = "decimal(18,2)"
    type_name = "DECIMAL"
    type_precision = 18
    type_scale    = 2
    position      = 1
  }
}

# --- Grants ---
resource "databricks_grants" "catalog_grants" {
  catalog = databricks_catalog.analytics.name
  grant {
    principal  = "analysts"
    privileges = ["USE_CATALOG"]
  }
  grant {
    principal  = "data-engineers"
    privileges = ["USE_CATALOG", "CREATE_SCHEMA"]
  }
}

resource "databricks_grants" "schema_grants" {
  schema = "${databricks_catalog.analytics.name}.${databricks_schema.finance.name}"
  grant {
    principal  = "analysts"
    privileges = ["USE_SCHEMA", "SELECT"]
  }
  grant {
    principal  = "data-engineers"
    privileges = ["USE_SCHEMA", "CREATE_TABLE", "CREATE_VIEW", "MODIFY"]
  }
}

resource "databricks_grants" "table_grants" {
  table = "${databricks_catalog.analytics.name}.${databricks_schema.finance.name}.invoices"
  grant {
    principal  = "analysts"
    privileges = ["SELECT"]
  }
}

resource "databricks_grants" "ext_location_grants" {
  external_location = databricks_external_location.raw.name
  grant {
    principal  = "data-engineers"
    privileges = ["READ_FILES", "WRITE_FILES", "CREATE_EXTERNAL_TABLE"]
  }
}

resource "databricks_grants" "storage_credential_grants" {
  storage_credential = databricks_storage_credential.adls.name
  grant {
    principal  = "data-engineers"
    privileges = ["CREATE_EXTERNAL_LOCATION"]
  }
}

# --- Volume ---
resource "databricks_volume" "landing" {
  catalog_name     = databricks_catalog.analytics.name
  schema_name      = databricks_schema.finance.name
  name             = "landing"
  volume_type      = "MANAGED"
  comment          = "Landing zone"
}

# --- Unity Catalog Connection (Lakehouse Federation) ---
resource "databricks_connection" "postgres" {
  name            = "postgres_prod"
  connection_type = "POSTGRESQL"
  comment         = "Production PostgreSQL"
  options = {
    host     = var.pg_host
    port     = "5432"
    user     = var.pg_user
    password = var.pg_password
  }
}
```

-----

## 18. Design Patterns & Best Practices

### Pattern 1: Environment Isolation via Catalogs

```
<domain>_dev  →  development environment, all data engineers have MODIFY
<domain>_tst  →  test/UAT, restricted write access
<domain>_prd  →  production, no direct human write access (CI/CD only)
```

```sql
-- CI/CD pipeline promotes via SP, not humans
-- Service principal owns schema and tables in prd
GRANT ALL PRIVILEGES ON SCHEMA analytics_prd.finance TO `sp-etl-pipeline`;
REVOKE MODIFY ON TABLE analytics_prd.finance.invoices FROM `data-engineers`;
GRANT SELECT  ON TABLE analytics_prd.finance.invoices TO `data-engineers`;
```

### Pattern 2: Layered Medallion Architecture

```
raw_catalog.landing.*       →  External tables, Parquet/CSV, untouched
bronze_catalog.ingest.*     →  Delta, schema enforcement, no transforms
silver_catalog.curated.*    →  Delta, cleaned, deduped, joined
gold_catalog.serving.*      →  Delta, aggregated, business-ready views
```

```sql
-- Each layer uses its own catalog and storage root
-- Cross-layer access: read from silver, write to gold only in pipelines
GRANT SELECT ON CATALOG silver_catalog TO `dlt-pipeline-sp`;
GRANT CREATE TABLE, MODIFY ON SCHEMA gold_catalog.serving TO `dlt-pipeline-sp`;
```

### Pattern 3: Minimum Privilege Group Model

```
Group                  →  Privileges
───────────────────────────────────────────────────────
analysts               →  USE CATALOG, USE SCHEMA, SELECT on views
data-engineers         →  + CREATE TABLE, CREATE VIEW, MODIFY in dev/tst
etl-service-principals →  + MODIFY in prd (no human members)
data-stewards          →  Owner of schemas, can manage grants
platform-admins        →  Metastore admins
```

### Pattern 4: Dynamic Views for Row-Level Security

```sql
-- Central security mapping table
CREATE TABLE my_catalog.security.user_region_map (
  user_email STRING, region STRING
) USING DELTA;

-- Dynamic view checks membership at query time
CREATE VIEW my_catalog.finance.v_invoices_by_region AS
SELECT t.*
FROM my_catalog.finance.invoices t
INNER JOIN my_catalog.security.user_region_map m
  ON m.user_email = current_user()
  AND m.region = t.region
WHERE is_account_group_member('finance-global')  -- bypass for admins
   OR t.region IN (
       SELECT region FROM my_catalog.security.user_region_map
       WHERE user_email = current_user()
   );

-- Analysts get SELECT on the view, NOT on the base table
REVOKE SELECT ON TABLE my_catalog.finance.invoices FROM `analysts`;
GRANT  SELECT ON VIEW  my_catalog.finance.v_invoices_by_region TO `analysts`;
```

### Pattern 5: Column Masking for PII

```sql
-- Tiered masking: full PII visible to HR-admins, last-4 to others
CREATE OR REPLACE FUNCTION my_catalog.security.mask_national_id(id STRING)
RETURNS STRING
RETURN CASE
  WHEN is_account_group_member('hr-admins')   THEN id
  WHEN is_account_group_member('hr-standard') THEN CONCAT('***-', RIGHT(id, 4))
  ELSE '***REDACTED***'
END;

ALTER TABLE my_catalog.hr.employees
  ALTER COLUMN national_id SET MASK my_catalog.security.mask_national_id;
```

### Pattern 6: External Location Hierarchy

```
Storage Account / S3 Bucket
└── /uc/                          ← External Location: top_level
    ├── /catalogs/analytics_dev/  ← catalog managed storage root
    ├── /catalogs/analytics_prd/
    └── /external/
        ├── /raw/                 ← External Location: raw_landing
        └── /archive/             ← External Location: archive
```

**Rule:** Define the narrowest external location possible. Avoid registering root of storage account.

### Pattern 7: Tagging Strategy

```sql
-- Apply consistent business tags at table creation
ALTER TABLE my_catalog.finance.invoices SET TAGS (
  'domain'          = 'finance',
  'pii'             = 'false',
  'data_class'      = 'confidential',
  'owner_team'      = 'finance-engineering',
  'sla_tier'        = 'gold',
  'retention_days'  = '2555'
);

-- Column-level PII tags
ALTER TABLE my_catalog.hr.employees
  ALTER COLUMN email      SET TAGS ('pii' = 'true', 'pii_type' = 'email');
ALTER TABLE my_catalog.hr.employees
  ALTER COLUMN ssn        SET TAGS ('pii' = 'true', 'pii_type' = 'government_id');
ALTER TABLE my_catalog.hr.employees
  ALTER COLUMN phone      SET TAGS ('pii' = 'true', 'pii_type' = 'phone');
```

### Pattern 8: Volumes Over DBFS

```python
# AVOID (legacy DBFS, no governance)
df = spark.read.csv("dbfs:/FileStore/raw/data.csv")

# PREFER (governed, lineage captured, ACL enforced)
df = spark.read.csv("/Volumes/my_catalog/raw/landing/data.csv")

# Service principal only needs WRITE VOLUME, not DBFS admin
```

### Pattern 9: Predictive Optimization

```sql
-- Enable at catalog level (applies to all schemas/tables)
ALTER CATALOG analytics_prd ENABLE PREDICTIVE OPTIMIZATION;

-- Or at schema level
ALTER SCHEMA analytics_prd.finance ENABLE PREDICTIVE OPTIMIZATION;

-- Disable for specific table if needed
ALTER TABLE analytics_prd.finance.invoices DISABLE PREDICTIVE OPTIMIZATION;
```

### Pattern 10: Liquid Clustering (DBR 13.3+)

```sql
-- Use instead of ZORDER for large, frequently updated tables
CREATE TABLE my_catalog.finance.invoices (
  invoice_id BIGINT, amount DECIMAL(18,2), invoice_date DATE, region STRING
)
USING DELTA
CLUSTER BY (invoice_date, region);  -- pick 1-4 high-cardinality, frequently filtered columns

-- Change clustering columns (no rewrite needed immediately)
ALTER TABLE my_catalog.finance.invoices CLUSTER BY (region, customer_id);

-- Remove clustering
ALTER TABLE my_catalog.finance.invoices CLUSTER BY NONE;

-- Incremental OPTIMIZE (only re-clusters changed files)
OPTIMIZE my_catalog.finance.invoices;
```

-----

## 19. Complete Methods Reference Table

### SQL DDL Methods

|Statement                        |Scope        |Description                                         |
|---------------------------------|-------------|----------------------------------------------------|
|`CREATE CATALOG`                 |Catalog      |Create new catalog                                  |
|`DROP CATALOG`                   |Catalog      |Remove catalog (CASCADE to drop all contents)       |
|`ALTER CATALOG`                  |Catalog      |Modify properties, owner, optimization              |
|`USE CATALOG`                    |Catalog      |Set current catalog context                         |
|`SHOW CATALOGS`                  |Catalog      |List available catalogs                             |
|`DESCRIBE CATALOG`               |Catalog      |Show catalog metadata                               |
|`CREATE SCHEMA`                  |Schema       |Create schema within catalog                        |
|`DROP SCHEMA`                    |Schema       |Remove schema (RESTRICT or CASCADE)                 |
|`ALTER SCHEMA`                   |Schema       |Modify properties, owner                            |
|`USE SCHEMA` / `USE DATABASE`    |Schema       |Set current schema context                          |
|`SHOW SCHEMAS` / `SHOW DATABASES`|Schema       |List schemas in catalog                             |
|`DESCRIBE SCHEMA`                |Schema       |Show schema metadata                                |
|`CREATE TABLE`                   |Table        |Create managed or external table                    |
|`DROP TABLE`                     |Table        |Remove table registration (managed: deletes data)   |
|`ALTER TABLE`                    |Table        |Modify columns, properties, constraints, owner, tags|
|`TRUNCATE TABLE`                 |Table        |Remove all rows (managed Delta only)                |
|`SHOW TABLES`                    |Table        |List tables in schema                               |
|`DESCRIBE TABLE`                 |Table        |Show column definitions                             |
|`DESCRIBE DETAIL`                |Table        |Show Delta metadata (size, partitions, format)      |
|`DESCRIBE HISTORY`               |Table        |Show Delta transaction log                          |
|`SHOW TBLPROPERTIES`             |Table        |List table properties                               |
|`SHOW COLUMNS`                   |Table        |List columns                                        |
|`SHOW CREATE TABLE`              |Table        |Show DDL used to create table                       |
|`OPTIMIZE`                       |Table        |Compact small files, optionally ZORDER              |
|`VACUUM`                         |Table        |Remove old Delta files                              |
|`RESTORE TABLE`                  |Table        |Revert table to prior version                       |
|`CREATE VIEW`                    |View         |Create logical view over tables                     |
|`DROP VIEW`                      |View         |Remove view                                         |
|`ALTER VIEW`                     |View         |Modify owner or redefine                            |
|`SHOW VIEWS`                     |View         |List views in schema                                |
|`CREATE VOLUME`                  |Volume       |Create managed or external volume                   |
|`DROP VOLUME`                    |Volume       |Remove volume                                       |
|`ALTER VOLUME`                   |Volume       |Modify owner, comment                               |
|`SHOW VOLUMES`                   |Volume       |List volumes in schema                              |
|`DESCRIBE VOLUME`                |Volume       |Show volume metadata                                |
|`CREATE FUNCTION`                |Function     |Define SQL or Python UDF                            |
|`DROP FUNCTION`                  |Function     |Remove UDF                                          |
|`ALTER FUNCTION`                 |Function     |Modify owner                                        |
|`SHOW FUNCTIONS`                 |Function     |List UDFs in schema                                 |
|`DESCRIBE FUNCTION`              |Function     |Show function signature                             |
|`CREATE STORAGE CREDENTIAL`      |Security     |Register cloud credential                           |
|`DROP STORAGE CREDENTIAL`        |Security     |Remove credential                                   |
|`ALTER STORAGE CREDENTIAL`       |Security     |Modify owner or properties                          |
|`VALIDATE STORAGE CREDENTIAL`    |Security     |Test connectivity                                   |
|`SHOW STORAGE CREDENTIALS`       |Security     |List credentials                                    |
|`DESCRIBE STORAGE CREDENTIAL`    |Security     |Show credential details                             |
|`CREATE EXTERNAL LOCATION`       |Security     |Register governed cloud path                        |
|`DROP EXTERNAL LOCATION`         |Security     |Remove external location                            |
|`ALTER EXTERNAL LOCATION`        |Security     |Modify owner, URL, credential                       |
|`VALIDATE EXTERNAL LOCATION`     |Security     |Test read/write access                              |
|`SHOW EXTERNAL LOCATIONS`        |Security     |List external locations                             |
|`DESCRIBE EXTERNAL LOCATION`     |Security     |Show location details                               |
|`CREATE CONNECTION`              |Federation   |Register external database connection               |
|`DROP CONNECTION`                |Federation   |Remove connection                                   |
|`SHOW CONNECTIONS`               |Federation   |List connections                                    |
|`DESCRIBE CONNECTION`            |Federation   |Show connection details                             |
|`CREATE SHARE`                   |Delta Sharing|Create data share                                   |
|`DROP SHARE`                     |Delta Sharing|Remove share                                        |
|`ALTER SHARE`                    |Delta Sharing|Add/remove tables, schemas, volumes                 |
|`SHOW SHARES`                    |Delta Sharing|List shares                                         |
|`SHOW ALL IN SHARE`              |Delta Sharing|List objects in share                               |
|`DESCRIBE SHARE`                 |Delta Sharing|Show share details                                  |
|`CREATE RECIPIENT`               |Delta Sharing|Register sharing recipient                          |
|`DROP RECIPIENT`                 |Delta Sharing|Remove recipient                                    |
|`SHOW RECIPIENTS`                |Delta Sharing|List recipients                                     |
|`DESCRIBE RECIPIENT`             |Delta Sharing|Show activation link and details                    |
|`CREATE PROVIDER`                |Delta Sharing|Register provider (consumer side)                   |
|`SHOW SHARES IN PROVIDER`        |Delta Sharing|List provider’s shares                              |
|`GRANT`                          |ACL          |Grant privilege on securable                        |
|`REVOKE`                         |ACL          |Revoke privilege                                    |
|`SHOW GRANTS`                    |ACL          |List grants on securable or principal               |
|`COMMENT ON TABLE`               |Metadata     |Set table comment                                   |
|`COMMENT ON COLUMN`              |Metadata     |Set column comment                                  |

### PySpark Catalog API Methods

|Method                                      |Description                    |
|--------------------------------------------|-------------------------------|
|`spark.catalog.currentCatalog()`            |Get current catalog name       |
|`spark.catalog.setCurrentCatalog(name)`     |Set current catalog            |
|`spark.catalog.listCatalogs()`              |List available catalogs        |
|`spark.catalog.currentDatabase()`           |Get current schema             |
|`spark.catalog.setCurrentDatabase(name)`    |Set current schema             |
|`spark.catalog.listDatabases()`             |List schemas in current catalog|
|`spark.catalog.listTables(dbName)`          |List tables in schema          |
|`spark.catalog.listColumns(tableName)`      |List columns of table          |
|`spark.catalog.listFunctions(dbName)`       |List UDFs in schema            |
|`spark.catalog.tableExists(tableName)`      |Check if table exists          |
|`spark.catalog.databaseExists(dbName)`      |Check if schema exists         |
|`spark.catalog.functionExists(funcName)`    |Check if function exists       |
|`spark.catalog.getTable(tableName)`         |Get Table object metadata      |
|`spark.catalog.getDatabase(dbName)`         |Get Database object metadata   |
|`spark.catalog.getFunction(funcName)`       |Get Function object metadata   |
|`spark.catalog.refreshTable(tableName)`     |Invalidate cache for table     |
|`spark.catalog.refreshByPath(path)`         |Refresh tables at path         |
|`spark.catalog.recoverPartitions(tableName)`|Sync partition metadata        |
|`spark.catalog.clearCache()`                |Clear all cached tables        |
|`spark.catalog.isCached(tableName)`         |Check if table is cached       |
|`spark.catalog.cacheTable(tableName)`       |Cache table in memory          |
|`spark.catalog.uncacheTable(tableName)`     |Remove table from cache        |
|`spark.catalog.createTable(tableName, ...)` |Create table programmatically  |
|`spark.catalog.createExternalTable(...)`    |Create external table          |
|`spark.catalog.dropGlobalTempView(name)`    |Drop global temp view          |
|`spark.catalog.dropTempView(name)`          |Drop local temp view           |
|`spark.catalog.registerFunction(name, f)`   |Register Python UDF            |

### Delta Table Python API (delta-spark)

|Method                                         |Description                      |
|-----------------------------------------------|---------------------------------|
|`DeltaTable.forName(spark, name)`              |Get DeltaTable by three-part name|
|`DeltaTable.forPath(spark, path)`              |Get DeltaTable by path           |
|`DeltaTable.isDeltaTable(spark, path)`         |Check if path is a Delta table   |
|`DeltaTable.create(spark)`                     |Start table builder (managed)    |
|`DeltaTable.createOrReplace(spark)`            |Start table builder (replace)    |
|`DeltaTable.createIfNotExists(spark)`          |Start table builder (idempotent) |
|`.alias(name)`                                 |Alias for merge operations       |
|`.merge(source, condition)`                    |Start merge builder              |
|`.whenMatchedUpdate(condition, set)`           |Update matched rows              |
|`.whenMatchedUpdateAll(condition)`             |Update all matched columns       |
|`.whenMatchedDelete(condition)`                |Delete matched rows              |
|`.whenNotMatchedInsert(condition, values)`     |Insert non-matched rows          |
|`.whenNotMatchedInsertAll(condition)`          |Insert all non-matched           |
|`.whenNotMatchedBySourceUpdate(condition, set)`|Update rows not in source        |
|`.whenNotMatchedBySourceDelete(condition)`     |Delete rows not in source        |
|`.execute()`                                   |Execute merge                    |
|`.update(condition, set)`                      |Update rows matching condition   |
|`.delete(condition)`                           |Delete rows matching condition   |
|`.history(limit)`                              |Get transaction history DataFrame|
|`.detail()`                                    |Get table details DataFrame      |
|`.vacuum(retentionHours)`                      |Remove old files                 |
|`.optimize().executeCompaction()`              |Compact small files              |
|`.optimize().executeZOrderBy(*cols)`           |Compact + Z-order                |
|`.restoreToVersion(version)`                   |Restore to version               |
|`.restoreToTimestamp(ts)`                      |Restore to timestamp             |
|`.toDF()`                                      |Get table as DataFrame           |
|`.generate("symlink_format_manifest")`         |Generate Hive manifest           |

### Databricks SDK (Python) Catalog Methods

|Client                 |Method                                         |Description                     |
|-----------------------|-----------------------------------------------|--------------------------------|
|`w.catalogs`           |`.create(name, ...)`                           |Create catalog                  |
|`w.catalogs`           |`.get(name)`                                   |Get catalog details             |
|`w.catalogs`           |`.list()`                                      |List catalogs                   |
|`w.catalogs`           |`.update(name, ...)`                           |Update catalog                  |
|`w.catalogs`           |`.delete(name, force)`                         |Delete catalog                  |
|`w.schemas`            |`.create(name, catalog_name, ...)`             |Create schema                   |
|`w.schemas`            |`.get(full_name)`                              |Get schema details              |
|`w.schemas`            |`.list(catalog_name)`                          |List schemas                    |
|`w.schemas`            |`.update(full_name, ...)`                      |Update schema                   |
|`w.schemas`            |`.delete(full_name, force)`                    |Delete schema                   |
|`w.tables`             |`.get(full_name)`                              |Get table details               |
|`w.tables`             |`.list(catalog_name, schema_name)`             |List tables                     |
|`w.tables`             |`.delete(full_name)`                           |Delete table                    |
|`w.tables`             |`.update(full_name, ...)`                      |Update table metadata           |
|`w.tables`             |`.exists(full_name)`                           |Check existence                 |
|`w.tables`             |`.summarize(catalog_name)`                     |Summary stats per catalog       |
|`w.grants`             |`.get(securable_type, full_name)`              |Get grants                      |
|`w.grants`             |`.get_effective(securable_type, full_name)`    |Get effective (inherited) grants|
|`w.grants`             |`.update(securable_type, full_name, changes)`  |Update grants                   |
|`w.volumes`            |`.create(catalog_name, schema_name, name, ...)`|Create volume                   |
|`w.volumes`            |`.read(name)`                                  |Get volume details              |
|`w.volumes`            |`.list(catalog_name, schema_name)`             |List volumes                    |
|`w.volumes`            |`.update(name, ...)`                           |Update volume                   |
|`w.volumes`            |`.delete(name)`                                |Delete volume                   |
|`w.external_locations` |`.create(name, url, credential_name, ...)`     |Create external location        |
|`w.external_locations` |`.get(name)`                                   |Get details                     |
|`w.external_locations` |`.list()`                                      |List all                        |
|`w.external_locations` |`.update(name, ...)`                           |Update                          |
|`w.external_locations` |`.delete(name, force)`                         |Delete                          |
|`w.external_locations` |`.validate(name, ...)`                         |Test access                     |
|`w.storage_credentials`|`.create(name, ...)`                           |Create credential               |
|`w.storage_credentials`|`.get(name)`                                   |Get details                     |
|`w.storage_credentials`|`.list()`                                      |List all                        |
|`w.storage_credentials`|`.update(name, ...)`                           |Update                          |
|`w.storage_credentials`|`.delete(name, force)`                         |Delete                          |
|`w.storage_credentials`|`.validate(name, ...)`                         |Test credential                 |
|`w.connections`        |`.create(name, connection_type, ...)`          |Create connection               |
|`w.connections`        |`.get(name)`                                   |Get details                     |
|`w.connections`        |`.list()`                                      |List connections                |
|`w.connections`        |`.delete(name, force)`                         |Delete connection               |
|`w.metastores`         |`.current()`                                   |Get assigned metastore          |
|`w.metastores`         |`.list()`                                      |List metastores in account      |
|`w.metastores`         |`.assign(workspace_id, ...)`                   |Assign metastore to workspace   |
|`w.metastores`         |`.get(id)`                                     |Get metastore by ID             |
|`w.metastores`         |`.update(id, ...)`                             |Update metastore                |

-----

## Quick Reference: Common Privilege Grants

```sql
-- Minimum for read-only analyst access
GRANT USE CATALOG ON CATALOG analytics_prd TO `analysts`;
GRANT USE SCHEMA  ON SCHEMA  analytics_prd.finance TO `analysts`;
GRANT SELECT      ON TABLE   analytics_prd.finance.invoices TO `analysts`;

-- ETL service principal: full schema access
GRANT USE CATALOG, CREATE SCHEMA         ON CATALOG analytics_prd TO `sp-etl`;
GRANT USE SCHEMA, CREATE TABLE,
      CREATE VIEW, MODIFY, CREATE VOLUME ON SCHEMA  analytics_prd.finance TO `sp-etl`;
GRANT READ FILES, WRITE FILES,
      CREATE EXTERNAL TABLE              ON EXTERNAL LOCATION raw_data TO `sp-etl`;

-- Data steward: can manage grants within a schema
GRANT USE CATALOG ON CATALOG analytics_prd TO `stewards`;
ALTER SCHEMA analytics_prd.finance SET OWNER TO `stewards`;

-- Cross-catalog read (for federation pattern)
GRANT USE CATALOG ON CATALOG silver_catalog TO `gold-pipeline-sp`;
GRANT USE SCHEMA, SELECT ON ALL TABLES IN SCHEMA silver_catalog.curated TO `gold-pipeline-sp`;
```

-----

## Quick Reference: System Functions

|Function                                 |Description                                |
|-----------------------------------------|-------------------------------------------|
|`current_user()`                         |Email of the executing user                |
|`current_date()`                         |Current date                               |
|`current_timestamp()`                    |Current datetime                           |
|`is_account_group_member(group)`         |True if user is in Databricks account group|
|`is_member(group)`                       |True if user is in workspace-local group   |
|`current_metastore()`                    |Name of the current metastore              |
|`current_catalog()`                      |Name of the current catalog                |
|`current_schema()` / `current_database()`|Name of the current schema                 |

-----

*Last updated: 2026 · Databricks Runtime 15.x / Unity Catalog GA*