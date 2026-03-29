# Azure Databricks SQL — Comprehensive Reference Card

> **Platform:** Azure Databricks (Databricks Runtime 13+, Unity Catalog, Delta Lake)  
> **Dialect:** ANSI SQL + Databricks SQL Extensions  
> **Last Updated:** 2026

-----

## Table of Contents

1. [SQL Execution Contexts](#1-sql-execution-contexts)
1. [DDL — Data Definition Language](#2-ddl--data-definition-language)
1. [DML — Data Manipulation Language](#3-dml--data-manipulation-language)
1. [Query Fundamentals](#4-query-fundamentals)
1. [Scalar Functions — Complete Reference Table](#5-scalar-functions--complete-reference-table)
1. [Aggregate Functions](#6-aggregate-functions)
1. [Window Functions](#7-window-functions)
1. [Higher-Order Functions](#8-higher-order-functions)
1. [Table-Valued Functions (TVFs)](#9-table-valued-functions-tvfs)
1. [Semi-Structured Data (JSON, Arrays, Structs, Maps)](#10-semi-structured-data-json-arrays-structs-maps)
1. [Delta Lake SQL Extensions](#11-delta-lake-sql-extensions)
1. [Unity Catalog SQL](#12-unity-catalog-sql)
1. [Streaming & Lakeflow SQL](#13-streaming--lakeflow-sql)
1. [AI & ML SQL Functions](#14-ai--ml-sql-functions)
1. [Parameterized Queries & Variables](#15-parameterized-queries--variables)
1. [CTEs, Subqueries & Query Composition](#16-ctes-subqueries--query-composition)
1. [Performance Tuning & Optimization](#17-performance-tuning--optimization)
1. [Design Patterns](#18-design-patterns)
1. [Best Practices](#19-best-practices)
1. [Quick-Reference Cheat Snippets](#20-quick-reference-cheat-snippets)

-----

## 1. SQL Execution Contexts

|Context                        |Where                         |Notes                                         |
|-------------------------------|------------------------------|----------------------------------------------|
|**Databricks SQL Warehouse**   |SQL Editor / BI tools         |Serverless or Classic; optimized for analytics|
|**Notebook (SQL cell)**        |`%sql` magic or `spark.sql()` |Interactive + batch; full Spark SQL           |
|**spark.sql() in Python**      |PySpark driver                |Returns DataFrame; full SQL support           |
|**Delta Live Tables (DLT)**    |`CREATE OR REFRESH LIVE TABLE`|Declarative pipeline SQL                      |
|**Spark Declarative Pipelines**|`pyspark.pipelines` dp module |New SDP API (Runtime 16+)                     |
|**JDBC/ODBC**                  |External BI tools             |Connects to SQL Warehouse                     |

```sql
-- Notebook execution
%sql
SELECT current_catalog(), current_database(), current_user();

-- PySpark
df = spark.sql("SELECT * FROM catalog.schema.table WHERE dt = '2025-01-01'")
```

-----

## 2. DDL — Data Definition Language

### 2.1 Catalog & Schema

```sql
-- Unity Catalog three-level namespace
CREATE CATALOG IF NOT EXISTS my_catalog
  COMMENT 'Production catalog';

CREATE SCHEMA IF NOT EXISTS my_catalog.analytics
  MANAGED LOCATION 'abfss://container@storage.dfs.core.windows.net/schemas/analytics'
  COMMENT 'Analytics layer';

ALTER SCHEMA my_catalog.analytics SET OWNER TO `data-engineers`;

DROP SCHEMA my_catalog.analytics CASCADE;  -- drops all objects
```

### 2.2 Tables

```sql
-- Managed Delta table (preferred)
CREATE TABLE IF NOT EXISTS my_catalog.analytics.sales (
  sale_id     BIGINT        NOT NULL,
  customer_id BIGINT        NOT NULL,
  product_id  BIGINT,
  amount      DECIMAL(18,2) NOT NULL,
  sale_date   DATE          NOT NULL,
  region      STRING,
  created_at  TIMESTAMP     DEFAULT current_timestamp()
)
USING DELTA
PARTITIONED BY (sale_date)
CLUSTER BY (customer_id)              -- Liquid Clustering (recommended over partitioning)
TBLPROPERTIES (
  'delta.autoOptimize.optimizeWrite' = 'true',
  'delta.autoOptimize.autoCompact'   = 'true',
  'delta.columnMapping.mode'         = 'name',
  'delta.enableDeletionVectors'      = 'true'
)
COMMENT 'Fact table: individual sales transactions';

-- External table
CREATE TABLE my_catalog.raw.events
USING PARQUET
LOCATION 'abfss://raw@storage.dfs.core.windows.net/events/'
OPTIONS (mergeSchema 'true');

-- CTAS (Create Table As Select)
CREATE OR REPLACE TABLE my_catalog.gold.daily_sales
USING DELTA
PARTITIONED BY (sale_date)
AS
SELECT
  sale_date,
  region,
  SUM(amount) AS total_sales,
  COUNT(*)    AS transaction_count
FROM my_catalog.analytics.sales
GROUP BY sale_date, region;

-- View
CREATE OR REPLACE VIEW my_catalog.analytics.vw_active_customers AS
SELECT c.customer_id, c.name, MAX(s.sale_date) AS last_purchase
FROM   my_catalog.analytics.customers c
JOIN   my_catalog.analytics.sales s USING (customer_id)
GROUP  BY c.customer_id, c.name;

-- Materialized View (Databricks SQL only)
CREATE MATERIALIZED VIEW my_catalog.analytics.mv_monthly_revenue AS
SELECT DATE_TRUNC('month', sale_date) AS month, SUM(amount) AS revenue
FROM   my_catalog.analytics.sales
GROUP  BY 1;

-- Dynamic View (row/column-level security)
CREATE OR REPLACE VIEW my_catalog.analytics.vw_sales_secure AS
SELECT
  sale_id,
  customer_id,
  CASE WHEN is_account_group_member('finance') THEN amount ELSE NULL END AS amount,
  sale_date,
  region
FROM my_catalog.analytics.sales
WHERE region = current_user_region();  -- custom UDF for row filtering
```

### 2.3 ALTER TABLE

```sql
ALTER TABLE my_catalog.analytics.sales ADD COLUMNS (loyalty_points INT AFTER region);
ALTER TABLE my_catalog.analytics.sales DROP COLUMN loyalty_points;
ALTER TABLE my_catalog.analytics.sales RENAME COLUMN region TO sales_region;
ALTER TABLE my_catalog.analytics.sales CHANGE COLUMN amount amount DECIMAL(20,2) COMMENT 'Sale amount in EUR';
ALTER TABLE my_catalog.analytics.sales SET TBLPROPERTIES ('delta.logRetentionDuration' = 'interval 60 days');
ALTER TABLE my_catalog.analytics.sales CLUSTER BY (customer_id, sale_date); -- change clustering keys
ALTER TABLE my_catalog.analytics.sales ADD CONSTRAINT pk_sale_id PRIMARY KEY (sale_id);
ALTER TABLE my_catalog.analytics.sales ADD CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
```

### 2.4 Table Constraints

```sql
ALTER TABLE orders ADD CONSTRAINT chk_amount CHECK (amount > 0);
ALTER TABLE orders ADD CONSTRAINT uq_order_ref UNIQUE (order_ref);
ALTER TABLE orders DROP CONSTRAINT chk_amount;
```

-----

## 3. DML — Data Manipulation Language

### 3.1 INSERT

```sql
-- Simple insert
INSERT INTO my_catalog.analytics.sales VALUES (1, 100, 200, 99.99, '2025-01-15', 'EMEA', current_timestamp());

-- Insert from query
INSERT INTO my_catalog.analytics.sales
SELECT * FROM staging.raw_sales WHERE processed = FALSE;

-- INSERT OVERWRITE (replaces matching partition)
INSERT OVERWRITE my_catalog.analytics.sales
PARTITION (sale_date = '2025-01-01')
SELECT * FROM staging.sales_20250101;
```

### 3.2 MERGE (Upsert)

```sql
-- Standard SCD Type 1 upsert
MERGE INTO my_catalog.analytics.customers AS tgt
USING (
  SELECT customer_id, name, email, phone, updated_at
  FROM   staging.customer_updates
) AS src
ON tgt.customer_id = src.customer_id
WHEN MATCHED AND src.updated_at > tgt.updated_at THEN
  UPDATE SET
    tgt.name       = src.name,
    tgt.email      = src.email,
    tgt.phone      = src.phone,
    tgt.updated_at = src.updated_at
WHEN NOT MATCHED THEN
  INSERT (customer_id, name, email, phone, updated_at)
  VALUES (src.customer_id, src.name, src.email, src.phone, src.updated_at)
WHEN NOT MATCHED BY SOURCE THEN
  DELETE;   -- removes rows not in source (full replace semantics)

-- SCD Type 2 — historical tracking with MERGE
MERGE INTO dim_customer AS tgt
USING (
  SELECT customer_id, name, email, current_timestamp() AS eff_start, TRUE AS is_current
  FROM   staging.customer_updates
) AS src
ON tgt.customer_id = src.customer_id AND tgt.is_current = TRUE
WHEN MATCHED AND (tgt.name <> src.name OR tgt.email <> src.email) THEN
  UPDATE SET tgt.is_current = FALSE, tgt.eff_end = current_timestamp()
WHEN NOT MATCHED THEN
  INSERT *;
-- Then insert new current records separately
```

### 3.3 UPDATE & DELETE

```sql
UPDATE my_catalog.analytics.sales
SET    region = 'EMEA-W'
WHERE  region = 'EMEA' AND sale_date >= '2025-01-01';

DELETE FROM my_catalog.analytics.sales
WHERE  sale_date < '2020-01-01';   -- GDPR / retention policy

-- Conditional delete with subquery
DELETE FROM orders
WHERE  customer_id IN (SELECT customer_id FROM churned_customers);
```

### 3.4 COPY INTO (Incremental Ingest)

```sql
COPY INTO my_catalog.raw.events
FROM  'abfss://landing@storage.dfs.core.windows.net/events/'
FILEFORMAT = JSON
FORMAT_OPTIONS (
  'multiLine'        = 'false',
  'timestampFormat'  = 'yyyy-MM-dd HH:mm:ss'
)
COPY_OPTIONS (
  'mergeSchema' = 'true',
  'force'       = 'false'   -- incremental: skips already-loaded files
);
```

-----

## 4. Query Fundamentals

### 4.1 SELECT Clause Features

```sql
-- Lateral column alias (Databricks SQL)
SELECT
  amount * 1.19 AS amount_with_vat,
  amount_with_vat * 0.1 AS vat_portion   -- references prior alias
FROM sales;

-- EXCEPT / REPLACE in SELECT *
SELECT * EXCEPT (ssn, credit_card_number) FROM customers;
SELECT * REPLACE (UPPER(name) AS name, amount * 1.1 AS amount) FROM sales;

-- QUALIFY (filter on window functions without subquery)
SELECT customer_id, sale_date, amount,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY sale_date DESC) AS rn
FROM   sales
QUALIFY rn = 1;   -- most recent sale per customer

-- PIVOT
SELECT * FROM (
  SELECT region, product_category, amount FROM sales
)
PIVOT (
  SUM(amount) FOR product_category IN ('Electronics', 'Clothing', 'Food')
);

-- UNPIVOT
SELECT customer_id, quarter, revenue
FROM quarterly_revenue
UNPIVOT (revenue FOR quarter IN (q1, q2, q3, q4));
```

### 4.2 JOIN Types

```sql
-- Standard joins
INNER JOIN, LEFT JOIN, RIGHT JOIN, FULL OUTER JOIN, CROSS JOIN

-- ANTI JOIN (rows with no match)
SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM shipments s WHERE s.order_id = o.order_id);
-- or
SELECT o.* FROM orders o LEFT JOIN shipments s ON o.order_id = s.order_id WHERE s.order_id IS NULL;

-- SEMI JOIN
SELECT * FROM customers WHERE customer_id IN (SELECT customer_id FROM orders);

-- LATERAL JOIN (correlated table subquery)
SELECT c.customer_id, c.name, latest.sale_date, latest.amount
FROM   customers c
JOIN   LATERAL (
         SELECT sale_date, amount FROM sales
         WHERE  customer_id = c.customer_id
         ORDER  BY sale_date DESC LIMIT 1
       ) AS latest ON TRUE;

-- ASOF JOIN (time-series — Databricks SQL 14+)
SELECT o.order_id, o.order_date, p.price
FROM   orders o
ASOF JOIN prices p ON o.product_id = p.product_id
MATCH_CONDITION (o.order_date >= p.effective_date);
```

### 4.3 Set Operations

```sql
UNION ALL    -- all rows (fastest, no dedup)
UNION        -- distinct rows
INTERSECT    -- rows in both
INTERSECT ALL
EXCEPT       -- rows in left not in right
EXCEPT ALL
```

### 4.4 TABLESAMPLE

```sql
SELECT * FROM sales TABLESAMPLE (10 PERCENT);       -- approximate
SELECT * FROM sales TABLESAMPLE (1000 ROWS);         -- exact row count
SELECT * FROM sales TABLESAMPLE (BUCKET 3 OUT OF 10 ON customer_id);
```

-----

## 5. Scalar Functions — Complete Reference Table

### 5.1 String Functions

|Function                     |Signature                                                       |Description                               |Example                                   |
|-----------------------------|----------------------------------------------------------------|------------------------------------------|------------------------------------------|
|`LENGTH`                     |`LENGTH(str)`                                                   |Character count                           |`LENGTH('hello')` → `5`                   |
|`CHAR_LENGTH`                |`CHAR_LENGTH(str)`                                              |Same as LENGTH                            |                                          |
|`UPPER`                      |`UPPER(str)`                                                    |Uppercase                                 |`UPPER('hello')` → `'HELLO'`              |
|`LOWER`                      |`LOWER(str)`                                                    |Lowercase                                 |                                          |
|`TRIM`                       |`TRIM([LEADING|TRAILING|BOTH] [chars] FROM str)`                |Strip chars                               |`TRIM('  hi  ')` → `'hi'`                 |
|`LTRIM` / `RTRIM`            |`LTRIM(str[, chars])`                                           |Left/right trim                           |                                          |
|`LPAD` / `RPAD`              |`LPAD(str, len, pad)`                                           |Pad string                                |`LPAD('5', 3, '0')` → `'005'`             |
|`SUBSTR` / `SUBSTRING`       |`SUBSTR(str, pos[, len])`                                       |Extract substring                         |`SUBSTR('hello', 2, 3)` → `'ell'`         |
|`LEFT` / `RIGHT`             |`LEFT(str, n)`                                                  |First/last n chars                        |`LEFT('hello', 3)` → `'hel'`              |
|`INSTR`                      |`INSTR(str, substr)`                                            |Position of substr                        |`INSTR('hello', 'l')` → `3`               |
|`LOCATE`                     |`LOCATE(substr, str[, pos])`                                    |Position from pos                         |                                          |
|`REPLACE`                    |`REPLACE(str, search, replace)`                                 |Replace all occurrences                   |`REPLACE('aabaa','a','x')` → `'xxbxx'`    |
|`TRANSLATE`                  |`TRANSLATE(str, from, to)`                                      |Char-by-char replace                      |                                          |
|`REGEXP_REPLACE`             |`REGEXP_REPLACE(str, pattern, replace[, pos[, occur[, flags]]])`|Regex replace                             |                                          |
|`REGEXP_EXTRACT`             |`REGEXP_EXTRACT(str, pattern[, idx])`                           |Extract capture group                     |                                          |
|`REGEXP_EXTRACT_ALL`         |`REGEXP_EXTRACT_ALL(str, pattern[, idx])`                       |All matches as array                      |                                          |
|`REGEXP_LIKE` / `RLIKE`      |`REGEXP_LIKE(str, pattern)`                                     |Regex match boolean                       |                                          |
|`REGEXP_COUNT`               |`REGEXP_COUNT(str, pattern)`                                    |Count pattern matches                     |                                          |
|`REGEXP_SUBSTR`              |`REGEXP_SUBSTR(str, pattern[, pos[, occur[, flags[, idx]]]])`   |Substring by regex                        |                                          |
|`SPLIT`                      |`SPLIT(str, delimiter[, limit])`                                |Split to array                            |`SPLIT('a,b,c', ',')` → `['a','b','c']`   |
|`SPLIT_PART`                 |`SPLIT_PART(str, delimiter, partNum)`                           |Nth part of split                         |`SPLIT_PART('a.b.c', '.', 2)` → `'b'`     |
|`CONCAT`                     |`CONCAT(str1, str2, ...)`                                       |Concatenate                               |                                          |
|`CONCAT_WS`                  |`CONCAT_WS(sep, str1, str2, ...)`                               |Concat with separator, nulls skipped      |                                          |
|`FORMAT_STRING`              |`FORMAT_STRING(fmt, args...)`                                   |printf-style format                       |`FORMAT_STRING('%05d', 42)` → `'00042'`   |
|`PRINTF`                     |`PRINTF(fmt, args...)`                                          |Alias for FORMAT_STRING                   |                                          |
|`REPEAT`                     |`REPEAT(str, n)`                                                |Repeat string                             |`REPEAT('ab', 3)` → `'ababab'`            |
|`REVERSE`                    |`REVERSE(str)`                                                  |Reverse string                            |                                          |
|`INITCAP`                    |`INITCAP(str)`                                                  |Title case                                |`INITCAP('hello world')` → `'Hello World'`|
|`ASCII`                      |`ASCII(str)`                                                    |ASCII code of first char                  |                                          |
|`CHR`                        |`CHR(n)`                                                        |Char from ASCII code                      |`CHR(65)` → `'A'`                         |
|`BASE64` / `UNBASE64`        |`BASE64(binary)`                                                |Encode/decode base64                      |                                          |
|`HEX` / `UNHEX`              |`HEX(binary|int)`                                               |Hex encode/decode                         |                                          |
|`MD5`                        |`MD5(str)`                                                      |MD5 hash                                  |                                          |
|`SHA1` / `SHA2`              |`SHA2(str, bits)`                                               |SHA hash                                  |`SHA2('data', 256)`                       |
|`CRC32`                      |`CRC32(str)`                                                    |CRC32 checksum                            |                                          |
|`SOUNDEX`                    |`SOUNDEX(str)`                                                  |Phonetic encoding                         |                                          |
|`LEVENSHTEIN`                |`LEVENSHTEIN(str1, str2[, threshold])`                          |Edit distance                             |                                          |
|`LIKE` / `ILIKE`             |`str LIKE pattern`                                              |Pattern match (case-sensitive/insensitive)|`'ABC' ILIKE 'abc'` → `TRUE`              |
|`STARTSWITH`                 |`STARTSWITH(str, prefix)`                                       |Prefix check                              |                                          |
|`ENDSWITH`                   |`ENDSWITH(str, suffix)`                                         |Suffix check                              |                                          |
|`CONTAINS`                   |`CONTAINS(str, substr)`                                         |Substring check                           |                                          |
|`OVERLAY`                    |`OVERLAY(str PLACING repl FROM pos [FOR len])`                  |Replace portion                           |                                          |
|`SPACE`                      |`SPACE(n)`                                                      |n spaces                                  |                                          |
|`BIT_LENGTH` / `OCTET_LENGTH`|`BIT_LENGTH(str)`                                               |Bit/byte length                           |                                          |
|`TO_UTF8` / `FROM_UTF8`      |`TO_UTF8(str)`                                                  |String <-> binary                         |                                          |
|`DECODE`                     |`DECODE(binary, charset)`                                       |Binary to string                          |                                          |
|`ENCODE`                     |`ENCODE(str, charset)`                                          |String to binary                          |                                          |
|`SENTENCES`                  |`SENTENCES(str[, lang, country])`                               |Split into sentences array                |                                          |
|`WORDS`                      |n/a                                                             |use `SPLIT` + tokenize                    |                                          |

### 5.2 Numeric Functions

|Function                                |Signature                       |Description                |Example                   |
|----------------------------------------|--------------------------------|---------------------------|--------------------------|
|`ABS`                                   |`ABS(n)`                        |Absolute value             |`ABS(-5)` → `5`           |
|`CEIL` / `CEILING`                      |`CEIL(n)`                       |Round up                   |`CEIL(4.2)` → `5`         |
|`FLOOR`                                 |`FLOOR(n)`                      |Round down                 |`FLOOR(4.9)` → `4`        |
|`ROUND`                                 |`ROUND(n[, scale])`             |Round to scale             |`ROUND(3.456, 2)` → `3.46`|
|`BROUND`                                |`BROUND(n[, scale])`            |Banker’s rounding          |                          |
|`TRUNC` / `TRUNCATE`                    |`TRUNC(n[, scale])`             |Truncate to scale          |`TRUNC(3.999, 1)` → `3.9` |
|`MOD` / `%`                             |`MOD(n, d)`                     |Modulo                     |`MOD(10, 3)` → `1`        |
|`POWER` / `POW`                         |`POWER(base, exp)`              |Exponentiation             |`POWER(2, 10)` → `1024`   |
|`SQRT`                                  |`SQRT(n)`                       |Square root                |                          |
|`CBRT`                                  |`CBRT(n)`                       |Cube root                  |                          |
|`EXP`                                   |`EXP(n)`                        |e^n                        |                          |
|`LN`                                    |`LN(n)`                         |Natural log                |                          |
|`LOG`                                   |`LOG(base, n)`                  |Logarithm                  |`LOG(10, 100)` → `2`      |
|`LOG2` / `LOG10`                        |`LOG2(n)`                       |Base-2/10 log              |                          |
|`SIN`, `COS`, `TAN`                     |`SIN(n)`                        |Trig functions (radians)   |                          |
|`ASIN`, `ACOS`, `ATAN`, `ATAN2`         |`ATAN2(y, x)`                   |Inverse trig               |                          |
|`SINH`, `COSH`, `TANH`                  |                                |Hyperbolic trig            |                          |
|`DEGREES` / `RADIANS`                   |`DEGREES(n)`                    |Convert angles             |                          |
|`PI`                                    |`PI()`                          |π constant                 |`PI()` → `3.14159...`     |
|`E`                                     |`E()`                           |Euler’s number             |                          |
|`SIGN`                                  |`SIGN(n)`                       |-1, 0, or 1                |                          |
|`SIGNUM`                                |`SIGNUM(n)`                     |Alias for SIGN             |                          |
|`FACTORIAL`                             |`FACTORIAL(n)`                  |n!                         |                          |
|`GREATEST`                              |`GREATEST(v1, v2, ...)`         |Max across columns         |                          |
|`LEAST`                                 |`LEAST(v1, v2, ...)`            |Min across columns         |                          |
|`RAND`                                  |`RAND([seed])`                  |Uniform [0,1)              |                          |
|`RANDN`                                 |`RANDN([seed])`                 |Standard normal            |                          |
|`UNIFORM`                               |`UNIFORM(min, max, seed)`       |Uniform int/float          |                          |
|`NORMAL`                                |`NORMAL(mean, stddev, seed)`    |Normal distribution        |                          |
|`WIDTH_BUCKET`                          |`WIDTH_BUCKET(val, min, max, n)`|Histogram bucket           |                          |
|`SHIFTLEFT` / `SHIFTRIGHT`              |`SHIFTLEFT(n, bits)`            |Bit shift                  |                          |
|`BITNOT` / `BITAND` / `BITOR` / `BITXOR`|`BITAND(a, b)`                  |Bitwise ops                |                          |
|`BIT_COUNT`                             |`BIT_COUNT(n)`                  |Count set bits             |                          |
|`INT2IP` / `IP2INT`                     |                                |IP/int conversion          |                          |
|`TYPEOF`                                |`TYPEOF(expr)`                  |Returns type name as string|                          |
|`TRY_*` prefix                          |`TRY_CAST`, `TRY_ADD`, etc.     |Returns NULL on error      |                          |

### 5.3 Date & Timestamp Functions

|Function                          |Signature                                |Description                  |Example                                           |
|----------------------------------|-----------------------------------------|-----------------------------|--------------------------------------------------|
|`CURRENT_DATE`                    |`CURRENT_DATE()`                         |Today’s date                 |                                                  |
|`CURRENT_TIMESTAMP`               |`CURRENT_TIMESTAMP()`                    |Now (session timezone)       |                                                  |
|`NOW`                             |`NOW()`                                  |Alias for CURRENT_TIMESTAMP  |                                                  |
|`CURRENT_TIMEZONE`                |`CURRENT_TIMEZONE()`                     |Session timezone name        |                                                  |
|`DATE`                            |`DATE(expr)`                             |Cast to DATE                 |                                                  |
|`TIMESTAMP`                       |`TIMESTAMP(expr)`                        |Cast to TIMESTAMP            |                                                  |
|`TO_DATE`                         |`TO_DATE(str, fmt)`                      |Parse date from string       |`TO_DATE('2025-01-15', 'yyyy-MM-dd')`             |
|`TO_TIMESTAMP`                    |`TO_TIMESTAMP(str, fmt)`                 |Parse timestamp              |                                                  |
|`TO_UNIX_TIMESTAMP`               |`TO_UNIX_TIMESTAMP(ts[, fmt])`           |Epoch seconds                |                                                  |
|`FROM_UNIXTIME`                   |`FROM_UNIXTIME(epoch[, fmt])`            |Epoch to timestamp           |                                                  |
|`UNIX_TIMESTAMP`                  |`UNIX_TIMESTAMP([ts[, fmt]])`            |Current or specific epoch    |                                                  |
|`DATE_FORMAT`                     |`DATE_FORMAT(ts, fmt)`                   |Format to string             |`DATE_FORMAT(NOW(), 'yyyy-MM')`                   |
|`DATE_ADD`                        |`DATE_ADD(date, days)`                   |Add days                     |                                                  |
|`DATE_SUB`                        |`DATE_SUB(date, days)`                   |Subtract days                |                                                  |
|`DATEADD`                         |`DATEADD(unit, n, date)`                 |Add n units                  |`DATEADD(MONTH, 3, '2025-01-01')`                 |
|`DATEDIFF`                        |`DATEDIFF(end, start)`                   |Day difference               |                                                  |
|`TIMESTAMPDIFF`                   |`TIMESTAMPDIFF(unit, start, end)`        |Difference in units          |                                                  |
|`DATE_TRUNC`                      |`DATE_TRUNC(unit, date)`                 |Truncate to unit             |`DATE_TRUNC('month', '2025-03-15')` → `2025-03-01`|
|`DATE_PART` / `EXTRACT`           |`DATE_PART('year', date)`                |Extract field                |`EXTRACT(DOW FROM sale_date)`                     |
|`YEAR` / `MONTH` / `DAY`          |`YEAR(date)`                             |Extract year/month/day       |                                                  |
|`HOUR` / `MINUTE` / `SECOND`      |`HOUR(ts)`                               |Extract time parts           |                                                  |
|`DAYOFWEEK`                       |`DAYOFWEEK(date)`                        |1=Sunday … 7=Saturday        |                                                  |
|`DAYOFYEAR`                       |`DAYOFYEAR(date)`                        |1–366                        |                                                  |
|`WEEKOFYEAR`                      |`WEEKOFYEAR(date)`                       |ISO week number              |                                                  |
|`QUARTER`                         |`QUARTER(date)`                          |1–4                          |                                                  |
|`MONTHS_BETWEEN`                  |`MONTHS_BETWEEN(d1, d2)`                 |Fractional months            |                                                  |
|`ADD_MONTHS`                      |`ADD_MONTHS(date, n)`                    |Add n months                 |                                                  |
|`LAST_DAY`                        |`LAST_DAY(date)`                         |Last day of month            |                                                  |
|`NEXT_DAY`                        |`NEXT_DAY(date, dayname)`                |Next weekday                 |`NEXT_DAY('2025-01-01', 'Mon')`                   |
|`TRUNC` (date)                    |`TRUNC(date, fmt)`                       |Alias for DATE_TRUNC         |                                                  |
|`CONVERT_TIMEZONE`                |`CONVERT_TIMEZONE(src_tz, tgt_tz, ts)`   |Timezone conversion          |                                                  |
|`FROM_UTC_TIMESTAMP`              |`FROM_UTC_TIMESTAMP(ts, tz)`             |UTC to local                 |                                                  |
|`TO_UTC_TIMESTAMP`                |`TO_UTC_TIMESTAMP(ts, tz)`               |Local to UTC                 |                                                  |
|`TIMESTAMP_SECONDS`               |`TIMESTAMP_SECONDS(n)`                   |Epoch → TIMESTAMP_LTZ        |                                                  |
|`TIMESTAMP_MILLIS`                |`TIMESTAMP_MILLIS(n)`                    |Epoch ms → TIMESTAMP         |                                                  |
|`TIMESTAMP_MICROS`                |`TIMESTAMP_MICROS(n)`                    |Epoch µs → TIMESTAMP         |                                                  |
|`DATE_FROM_PARTS`                 |`DATE_FROM_PARTS(y, m, d)`               |Build date from parts        |                                                  |
|`TIMESTAMP_FROM_PARTS`            |`TIMESTAMP_FROM_PARTS(y,m,d,h,mi,s[,ns])`|Build timestamp              |                                                  |
|`MAKE_DATE`                       |`MAKE_DATE(y, m, d)`                     |Alias                        |                                                  |
|`MAKE_TIMESTAMP`                  |`MAKE_TIMESTAMP(y, m, d, h, mi, s)`      |Alias                        |                                                  |
|`MAKE_INTERVAL`                   |`MAKE_INTERVAL(y,mo,w,d,h,mi,s)`         |Build interval               |                                                  |
|`INTERVAL`                        |`INTERVAL '1' YEAR`                      |Interval literal             |                                                  |
|`TRY_TO_DATE` / `TRY_TO_TIMESTAMP`|                                         |Returns NULL on parse failure|                                                  |

### 5.4 Conditional & Null Functions

|Function            |Signature                          |Description         |Example                       |
|--------------------|-----------------------------------|--------------------|------------------------------|
|`CASE WHEN`         |`CASE WHEN c THEN v ... ELSE d END`|Conditional         |                              |
|`IF`                |`IF(cond, true_val, false_val)`    |Inline if           |`IF(amount > 0, 'pos', 'neg')`|
|`IFF`               |`IFF(cond, t, f)`                  |Alias for IF        |                              |
|`IFNULL`            |`IFNULL(expr, replace)`            |NULL replacement    |                              |
|`NULLIF`            |`NULLIF(e1, e2)`                   |NULL if equal       |`NULLIF(divisor, 0)`          |
|`NVL`               |`NVL(e1, e2)`                      |Oracle-style IFNULL |                              |
|`NVL2`              |`NVL2(e, not_null_val, null_val)`  |NULL branch         |                              |
|`COALESCE`          |`COALESCE(e1, e2, ...)`            |First non-NULL      |                              |
|`ISNULL`            |`ISNULL(expr)`                     |TRUE if NULL        |                              |
|`ISNOTNULL`         |`ISNOTNULL(expr)`                  |TRUE if not NULL    |                              |
|`ISNAN`             |`ISNAN(expr)`                      |TRUE if NaN         |                              |
|`NANVL`             |`NANVL(e, replace)`                |Replace NaN         |                              |
|`DECODE`            |`DECODE(e, s1,r1, s2,r2, ..., def)`|CASE shorthand      |                              |
|`ZAP_SPECIAL_FLOATS`|internal                           |Used in aggregations|                              |
|`ASSERT_TRUE`       |`ASSERT_TRUE(cond[, msg])`         |Raise error if false|                              |
|`RAISE_ERROR`       |`RAISE_ERROR(msg)`                 |Always raises       |                              |

### 5.5 Type Conversion Functions

|Function                                                           |Signature                      |Notes                              |
|-------------------------------------------------------------------|-------------------------------|-----------------------------------|
|`CAST`                                                             |`CAST(expr AS type)`           |Strict; error on failure           |
|`TRY_CAST`                                                         |`TRY_CAST(expr AS type)`       |NULL on failure                    |
|`BIGINT`, `INT`, `DOUBLE`, `STRING`, `BOOLEAN`, `DATE`, `TIMESTAMP`|`BIGINT(expr)`                 |Shorthand casts                    |
|`TO_BINARY`                                                        |`TO_BINARY(str[, fmt])`        |String to BINARY (hex/base64/utf-8)|
|`FROM_JSON`                                                        |`FROM_JSON(json_str, schema)`  |JSON string to struct              |
|`TO_JSON`                                                          |`TO_JSON(struct_or_map)`       |Struct/map to JSON string          |
|`PARSE_JSON`                                                       |`PARSE_JSON(str)`              |Returns VARIANT (Databricks SQL)   |
|`SCHEMA_OF_JSON`                                                   |`SCHEMA_OF_JSON(str)`          |Infer schema from JSON sample      |
|`SCHEMA_OF_CSV`                                                    |`SCHEMA_OF_CSV(str[, opts])`   |Infer schema from CSV              |
|`FROM_CSV`                                                         |`FROM_CSV(str, schema[, opts])`|CSV string to struct               |
|`TO_CSV`                                                           |`TO_CSV(struct)`               |Struct to CSV string               |

### 5.6 Collection / Semi-Structured Functions

> Covered in detail in [Section 10](#10-semi-structured-data-json-arrays-structs-maps)

|Function                                                                                                                                                                                                                                                                                                        |Description                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------|
|`ARRAY`, `ARRAY_AGG`, `ARRAY_APPEND`, `ARRAY_COMPACT`, `ARRAY_CONTAINS`, `ARRAY_DISTINCT`, `ARRAY_EXCEPT`, `ARRAY_INTERSECT`, `ARRAY_JOIN`, `ARRAY_MAX`, `ARRAY_MIN`, `ARRAY_POSITION`, `ARRAY_PREPEND`, `ARRAY_REMOVE`, `ARRAY_REPEAT`, `ARRAY_REVERSE`, `ARRAY_SIZE`, `ARRAY_SORT`, `ARRAY_UNION`, `ARRAY_ZIP`|Array construction & manipulation   |
|`MAP`, `MAP_CONCAT`, `MAP_CONTAINS_KEY`, `MAP_ENTRIES`, `MAP_FROM_ARRAYS`, `MAP_FROM_ENTRIES`, `MAP_KEYS`, `MAP_VALUES`, `MAP_FILTER`, `MAP_TRANSFORM_KEYS`, `MAP_TRANSFORM_VALUES`, `MAP_ZIP_WITH`                                                                                                             |Map construction & manipulation     |
|`STRUCT`, `NAMED_STRUCT`, `STRUCT_EXPLODE`                                                                                                                                                                                                                                                                      |Struct construction                 |
|`EXPLODE`, `EXPLODE_OUTER`, `POSEXPLODE`, `POSEXPLODE_OUTER`, `INLINE`, `INLINE_OUTER`                                                                                                                                                                                                                          |Array/map to rows                   |
|`FLATTEN`                                                                                                                                                                                                                                                                                                       |Flatten nested arrays               |
|`ZIP_WITH`, `TRANSFORM`, `FILTER`, `AGGREGATE`, `FORALL`, `EXISTS`, `REDUCE`                                                                                                                                                                                                                                    |Higher-order (see Section 8)        |
|`SEQUENCE`                                                                                                                                                                                                                                                                                                      |Generate array of values            |
|`GET` / `ELEMENT_AT`                                                                                                                                                                                                                                                                                            |Element by index (0-based / 1-based)|
|`SLICE`                                                                                                                                                                                                                                                                                                         |Subarray                            |
|`SORT_ARRAY`                                                                                                                                                                                                                                                                                                    |Sort array                          |

### 5.7 Hash & ID Functions

|Function                            |Signature                      |Description                      |
|------------------------------------|-------------------------------|---------------------------------|
|`HASH`                              |`HASH(expr, ...)`              |Spark murmur3 hash               |
|`XXHASH64`                          |`XXHASH64(expr, ...)`          |64-bit xxHash                    |
|`MD5`                               |`MD5(str)`                     |MD5 hex string                   |
|`SHA1`                              |`SHA1(str)`                    |SHA-1 hex string                 |
|`SHA2`                              |`SHA2(str, bits)`              |SHA-2 (224/256/384/512)          |
|`UUID`                              |`UUID()`                       |Random UUID v4                   |
|`MONOTONICALLY_INCREASING_ID`       |`MONOTONICALLY_INCREASING_ID()`|Unique (not sequential) 64-bit ID|
|`SPARK_PARTITION_ID`                |`SPARK_PARTITION_ID()`         |Current partition number         |
|`INPUT_FILE_NAME`                   |`INPUT_FILE_NAME()`            |Source file path                 |
|`INPUT_FILE_BLOCK_START` / `_LENGTH`|                               |File block metadata              |

### 5.8 Metadata & System Functions

|Function                                 |Description                   |
|-----------------------------------------|------------------------------|
|`CURRENT_USER()`                         |Current user/service principal|
|`SESSION_USER()`                         |Alias                         |
|`IS_ACCOUNT_GROUP_MEMBER(group)`         |Group membership check        |
|`CURRENT_CATALOG()`                      |Active catalog                |
|`CURRENT_DATABASE()` / `CURRENT_SCHEMA()`|Active schema                 |
|`CURRENT_METASTORE()`                    |Active Unity Catalog metastore|
|`VERSION()`                              |Databricks Runtime version    |
|`SPARK_VERSION()`                        |Spark version string          |
|`INPUT_FILE_NAME()`                      |Used in COPY INTO / Autoloader|
|`INPUT_FILE_MODIFICATION_TIME()`         |Source file mtime             |
|`TYPEOF(expr)`                           |Data type of expr             |

-----

## 6. Aggregate Functions

|Function                           |Signature                                         |Notes                          |
|-----------------------------------|--------------------------------------------------|-------------------------------|
|`COUNT`                            |`COUNT(*)`, `COUNT(col)`, `COUNT(DISTINCT col)`   |Null-ignoring except `COUNT(*)`|
|`SUM`                              |`SUM([DISTINCT] col)`                             |                               |
|`AVG` / `MEAN`                     |`AVG(col)`                                        |                               |
|`MIN` / `MAX`                      |`MIN(col)`                                        |Works on strings, dates        |
|`STDDEV` / `STDDEV_SAMP`           |                                                  |Sample std deviation           |
|`STDDEV_POP`                       |                                                  |Population std deviation       |
|`VARIANCE` / `VAR_SAMP` / `VAR_POP`|                                                  |Variance                       |
|`SKEWNESS`                         |                                                  |Skewness                       |
|`KURTOSIS`                         |                                                  |Excess kurtosis                |
|`CORR`                             |`CORR(col1, col2)`                                |Pearson correlation            |
|`COVAR_SAMP` / `COVAR_POP`         |                                                  |Covariance                     |
|`REGR_SLOPE` / `REGR_INTERCEPT`    |                                                  |Linear regression              |
|`FIRST` / `LAST`                   |`FIRST(col[, ignoreNulls])`                       |First/last value in group      |
|`FIRST_VALUE` / `LAST_VALUE`       |Window variant                                    |                               |
|`COLLECT_LIST`                     |`COLLECT_LIST(col)`                               |Group to array (with dups)     |
|`COLLECT_SET`                      |`COLLECT_SET(col)`                                |Group to array (distinct)      |
|`ARRAY_AGG`                        |`ARRAY_AGG(col)`                                  |ANSI alias for COLLECT_LIST    |
|`STRING_AGG` / `LISTAGG`           |`STRING_AGG(col, sep)`                            |Concatenate strings            |
|`GROUP_CONCAT`                     |`GROUP_CONCAT(col [ORDER BY ...] [SEPARATOR sep])`|MySQL-style                    |
|`APPROX_COUNT_DISTINCT`            |`APPROX_COUNT_DISTINCT(col[, rsd])`               |Fast approx distinct count     |
|`APPROX_PERCENTILE`                |`APPROX_PERCENTILE(col, p[, accuracy])`           |Approx quantile                |
|`PERCENTILE`                       |`PERCENTILE(col, p)`                              |Exact percentile (expensive)   |
|`PERCENTILE_APPROX`                |Alias                                             |                               |
|`MEDIAN`                           |`MEDIAN(col)`                                     |Exact median                   |
|`MODE`                             |`MODE(col)`                                       |Most frequent value            |
|`MAX_BY`                           |`MAX_BY(value_col, order_col)`                    |Value at max of order_col      |
|`MIN_BY`                           |`MIN_BY(value_col, order_col)`                    |Value at min                   |
|`ANY_VALUE`                        |`ANY_VALUE(col)`                                  |Arbitrary value from group     |
|`BOOL_AND` / `EVERY`               |                                                  |All TRUE?                      |
|`BOOL_OR` / `SOME` / `ANY`         |                                                  |Any TRUE?                      |
|`BIT_AND` / `BIT_OR` / `BIT_XOR`   |                                                  |Bitwise aggregates             |
|`HISTOGRAM_NUMERIC`                |`HISTOGRAM_NUMERIC(col, n)`                       |Returns n histogram structs    |
|`HLLSKETCH_AGG`                    |`HLL_SKETCH_AGG(col)`                             |HyperLogLog sketch             |
|`HLL_SKETCH_ESTIMATE`              |`HLL_SKETCH_ESTIMATE(sketch)`                     |Estimate from sketch           |
|`MERGE_AGG`                        |                                                  |Merge VARIANT values           |

### GROUPING SETS

```sql
-- All grouping combos in one pass
SELECT region, product_category, SUM(amount) AS total
FROM   sales
GROUP  BY GROUPING SETS (
  (region, product_category),  -- subtotals by region+category
  (region),                    -- subtotals by region
  (product_category),          -- subtotals by category
  ()                           -- grand total
);

-- ROLLUP (hierarchical totals)
SELECT year, quarter, month, SUM(revenue) AS revenue
FROM   sales
GROUP  BY ROLLUP (year, quarter, month);

-- CUBE (all combinations)
SELECT region, product, SUM(sales) FROM sales GROUP BY CUBE (region, product);

-- GROUPING() function (identify aggregate rows)
SELECT
  region,
  product_category,
  SUM(amount),
  GROUPING(region)           AS is_region_total,
  GROUPING(product_category) AS is_category_total,
  GROUPING_ID(region, product_category) AS group_level
FROM sales
GROUP BY ROLLUP (region, product_category);
```

### FILTER clause

```sql
SELECT
  COUNT(*) FILTER (WHERE status = 'COMPLETED')  AS completed_count,
  SUM(amount) FILTER (WHERE region = 'EMEA')    AS emea_total,
  AVG(score) FILTER (WHERE score IS NOT NULL)   AS avg_score
FROM orders;
```

-----

## 7. Window Functions

```sql
-- Anatomy
function_name([args]) OVER (
  [PARTITION BY col1, col2, ...]
  [ORDER BY col3 [ASC|DESC] [NULLS FIRST|LAST], ...]
  [ROWS|RANGE BETWEEN frame_start AND frame_end]
)
```

### Frame Boundaries

|Boundary             |Meaning                    |
|---------------------|---------------------------|
|`UNBOUNDED PRECEDING`|Start of partition         |
|`n PRECEDING`        |n rows/range before current|
|`CURRENT ROW`        |Current row                |
|`n FOLLOWING`        |n rows/range after current |
|`UNBOUNDED FOLLOWING`|End of partition           |

### Ranking Functions

|Function        |Description                            |
|----------------|---------------------------------------|
|`ROW_NUMBER()`  |Unique sequential rank (1,2,3…)        |
|`RANK()`        |Same value → same rank, gaps after ties|
|`DENSE_RANK()`  |Same value → same rank, no gaps        |
|`PERCENT_RANK()`|`(rank-1)/(total-1)`                   |
|`CUME_DIST()`   |`(rows ≤ current)/(total rows)`        |
|`NTILE(n)`      |Divide into n buckets                  |

### Offset Functions

|Function     |Signature                         |Description         |
|-------------|----------------------------------|--------------------|
|`LAG`        |`LAG(col[, offset[, default]])`   |Previous row value  |
|`LEAD`       |`LEAD(col[, offset[, default]])`  |Next row value      |
|`FIRST_VALUE`|`FIRST_VALUE(col[, ignoreNulls])` |First value in frame|
|`LAST_VALUE` |`LAST_VALUE(col[, ignoreNulls])`  |Last value in frame |
|`NTH_VALUE`  |`NTH_VALUE(col, n[, ignoreNulls])`|nth value in frame  |

### Analytical Functions

|Function                           |Description               |
|-----------------------------------|--------------------------|
|`SUM`, `AVG`, `MIN`, `MAX`, `COUNT`|Running/rolling aggregates|
|`STDDEV`, `VARIANCE`, `CORR`       |Rolling stats             |

```sql
-- Examples
SELECT
  customer_id,
  sale_date,
  amount,

  -- Ranking
  ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY sale_date)        AS txn_seq,
  DENSE_RANK() OVER (ORDER BY amount DESC)                               AS amount_rank,

  -- Running totals
  SUM(amount) OVER (PARTITION BY customer_id ORDER BY sale_date
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)    AS running_total,

  -- Moving average (30-day window by date range)
  AVG(amount) OVER (PARTITION BY customer_id ORDER BY CAST(sale_date AS BIGINT)
                    RANGE BETWEEN 2592000 PRECEDING AND CURRENT ROW)     AS ma_30d,

  -- Lag/lead
  LAG(amount, 1, 0) OVER (PARTITION BY customer_id ORDER BY sale_date)  AS prev_amount,
  LEAD(sale_date)   OVER (PARTITION BY customer_id ORDER BY sale_date)  AS next_sale_date,

  -- Session analysis: gap > 30 days = new session
  SUM(CASE WHEN DATEDIFF(sale_date,
            LAG(sale_date) OVER (PARTITION BY customer_id ORDER BY sale_date)) > 30
       THEN 1 ELSE 0 END)
  OVER (PARTITION BY customer_id ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)                AS session_id
FROM sales;
```

-----

## 8. Higher-Order Functions

Higher-order functions accept lambda expressions `x -> expr` or `(x, y) -> expr`.

|Function              |Signature                                           |Description           |
|----------------------|----------------------------------------------------|----------------------|
|`TRANSFORM`           |`TRANSFORM(array, x -> expr)`                       |Map over array        |
|`FILTER`              |`FILTER(array, x -> bool_expr)`                     |Keep matching elements|
|`AGGREGATE`           |`AGGREGATE(array, init, (acc, x) -> expr[, finish])`|Left fold (reduce)    |
|`REDUCE`              |alias for AGGREGATE                                 |                      |
|`EXISTS`              |`EXISTS(array, x -> bool)`                          |Any element matches   |
|`FORALL`              |`FORALL(array, x -> bool)`                          |All elements match    |
|`ZIP_WITH`            |`ZIP_WITH(arr1, arr2, (x, y) -> expr)`              |Element-wise combine  |
|`ARRAY_SORT`          |`ARRAY_SORT(array[, (x, y) -> comparator])`         |Custom sort           |
|`MAP_FILTER`          |`MAP_FILTER(map, (k, v) -> bool)`                   |Filter map entries    |
|`MAP_TRANSFORM_KEYS`  |`MAP_TRANSFORM_KEYS(map, (k, v) -> new_k)`          |Transform keys        |
|`MAP_TRANSFORM_VALUES`|`MAP_TRANSFORM_VALUES(map, (k, v) -> new_v)`        |Transform values      |
|`MAP_ZIP_WITH`        |`MAP_ZIP_WITH(map1, map2, (k, v1, v2) -> expr)`     |Combine maps by key   |

```sql
-- Transform: convert cents to dollars
SELECT TRANSFORM(prices, p -> p / 100.0) AS prices_eur FROM products;

-- Filter: keep only positives
SELECT FILTER(scores, s -> s > 0) AS positive_scores FROM surveys;

-- Aggregate: sum of array
SELECT AGGREGATE(amounts, 0L, (acc, x) -> acc + x) AS total FROM order_lines;

-- Exists: any item out of stock
SELECT product_id, EXISTS(inventory, i -> i.stock = 0) AS has_oos_variant FROM products;

-- Forall: all reviews ≥ 4 stars
SELECT product_id, FORALL(reviews, r -> r.rating >= 4) AS all_positive FROM products;

-- ZipWith: combine parallel arrays
SELECT ZIP_WITH(ARRAY(1,2,3), ARRAY('a','b','c'), (n, c) -> CONCAT(n, c)) AS zipped;
-- Result: ['1a', '2b', '3c']

-- Array sort with custom comparator
SELECT ARRAY_SORT(ARRAY(3,1,2), (x, y) -> CASE WHEN x < y THEN -1 WHEN x > y THEN 1 ELSE 0 END);

-- Map transform
SELECT MAP_TRANSFORM_VALUES(price_map, (k, v) -> ROUND(v * 1.19, 2)) AS prices_with_vat;
```

-----

## 9. Table-Valued Functions (TVFs)

```sql
-- EXPLODE arrays/maps to rows
SELECT order_id, item
FROM   orders
LATERAL VIEW EXPLODE(items) t AS item;

-- POSEXPLODE (include position index)
SELECT order_id, pos, item
FROM   orders
LATERAL VIEW POSEXPLODE(items) t AS pos, item;

-- EXPLODE_OUTER (keep nulls/empties)
SELECT order_id, item
FROM   orders
LATERAL VIEW OUTER EXPLODE(items) t AS item;

-- INLINE (explode array of structs)
SELECT order_id, item_id, qty
FROM   orders
LATERAL VIEW INLINE(line_items) t AS item_id, qty;

-- RANGE (generate row series)
SELECT * FROM RANGE(1, 10, 2);  -- 1,3,5,7,9

-- User-defined TVFs (Python)
-- Defined in Python with @udf return type ArrayType; called in SQL similarly
```

-----

## 10. Semi-Structured Data (JSON, Arrays, Structs, Maps)

### 10.1 JSON Parsing

```sql
-- Parse JSON column
SELECT
  json_col:customer_id::BIGINT               AS customer_id,   -- colon-path + cast (Databricks SQL)
  json_col:address.city::STRING              AS city,
  json_col:orders[0].amount::DECIMAL(18,2)   AS first_order,
  GET_JSON_OBJECT(json_col, '$.address.zip') AS zip            -- SparkSQL JSONPath
FROM raw_events;

-- Typed extraction
SELECT FROM_JSON(json_col, 'STRUCT<id:BIGINT, name:STRING, tags:ARRAY<STRING>>') AS parsed
FROM raw_events;

-- Schema inference
SELECT SCHEMA_OF_JSON('{"id":1,"name":"Tim","active":true}');

-- Explode JSON array field
SELECT e.*, item.product_id, item.qty
FROM   events e
LATERAL VIEW EXPLODE(FROM_JSON(e.json_col, 'ARRAY<STRUCT<product_id:BIGINT,qty:INT>>')) t AS item;

-- VARIANT type (Databricks SQL native semi-structured)
SELECT
  data:id::INT,
  data:tags[0]::STRING,
  VARIANT_GET(data, '$.address.city', 'STRING') AS city
FROM semi_table;
```

### 10.2 Struct Operations

```sql
-- Access struct fields with dot notation
SELECT address.city, address.zip FROM customers;

-- Construct struct
SELECT NAMED_STRUCT('first', first_name, 'last', last_name) AS full_name FROM employees;

-- Star-expand struct
SELECT address.* FROM customers;
```

### 10.3 Array Operations

```sql
-- Construction
SELECT ARRAY(1, 2, 3, 4, 5) AS nums;
SELECT SEQUENCE(1, 10) AS seq;
SELECT SEQUENCE(CURRENT_DATE, DATE_ADD(CURRENT_DATE, 30), INTERVAL '1' DAY) AS date_range;

-- Access
SELECT items[0] AS first_item FROM orders;           -- 0-based in arrays
SELECT ELEMENT_AT(items, 1) AS first_item FROM orders; -- 1-based

-- Manipulation
SELECT ARRAY_APPEND(tags, 'new_tag')   AS tags FROM articles;
SELECT ARRAY_PREPEND('x', arr)         AS arr  FROM t;
SELECT ARRAY_REMOVE(tags, 'obsolete')  AS tags FROM articles;
SELECT ARRAY_DISTINCT(tags)            AS unique_tags FROM articles;
SELECT ARRAY_SORT(scores)              AS sorted  FROM tests;
SELECT ARRAY_REVERSE(items)            AS reversed FROM lists;
SELECT ARRAY_CONTAINS(tags, 'promo')   AS has_promo FROM articles;
SELECT ARRAY_POSITION(items, 'target') AS idx FROM t;  -- 1-based, 0 if not found
SELECT ARRAY_SIZE(items)               AS n FROM orders;
SELECT SIZE(items)                     AS n FROM orders;  -- alias
SELECT FLATTEN(nested_arrays)          AS flat FROM t;
SELECT SLICE(arr, 2, 4)               AS sub FROM t;  -- start=2 (1-based), length=4
SELECT ARRAY_JOIN(tags, ', ')          AS tag_str FROM articles;
SELECT ARRAY_UNION(a, b)              AS union_arr FROM t;
SELECT ARRAY_INTERSECT(a, b)          AS common FROM t;
SELECT ARRAY_EXCEPT(a, b)             AS diff FROM t;
SELECT ARRAYS_ZIP(a, b)               AS zipped FROM t;  -- array of structs
SELECT ARRAY_COMPACT(arr)             AS no_nulls FROM t;
SELECT ARRAY_MAX(scores)              AS top FROM tests;
SELECT ARRAY_MIN(scores)              AS low FROM tests;
```

### 10.4 Map Operations

```sql
SELECT MAP('key1', 1, 'key2', 2) AS m;
SELECT MAP_FROM_ARRAYS(ARRAY('a','b'), ARRAY(1,2)) AS m;
SELECT MAP_FROM_ENTRIES(ARRAY(STRUCT('a',1), STRUCT('b',2))) AS m;

SELECT m['key1'] AS val FROM t;                   -- access by key
SELECT ELEMENT_AT(m, 'key1') AS val FROM t;

SELECT MAP_KEYS(m) AS keys FROM t;
SELECT MAP_VALUES(m) AS vals FROM t;
SELECT MAP_ENTRIES(m) AS entries FROM t;           -- array of structs
SELECT MAP_CONTAINS_KEY(m, 'key') AS has_key FROM t;
SELECT MAP_CONCAT(m1, m2) AS merged FROM t;        -- m2 wins on collision
SELECT CARDINALITY(m) AS size FROM t;              -- number of entries
SELECT SIZE(m) AS size FROM t;                     -- alias
```

-----

## 11. Delta Lake SQL Extensions

### 11.1 Time Travel

```sql
-- By version
SELECT * FROM my_catalog.analytics.sales VERSION AS OF 42;

-- By timestamp
SELECT * FROM my_catalog.analytics.sales TIMESTAMP AS OF '2025-03-01 00:00:00';

-- With @ shorthand
SELECT * FROM my_catalog.analytics.sales@v42;
SELECT * FROM my_catalog.analytics.sales@20250301000000000;

-- Restore
RESTORE TABLE my_catalog.analytics.sales TO VERSION AS OF 42;
RESTORE TABLE my_catalog.analytics.sales TO TIMESTAMP AS OF '2025-03-01';

-- Audit history
DESCRIBE HISTORY my_catalog.analytics.sales;
DESCRIBE HISTORY my_catalog.analytics.sales LIMIT 10;

-- Diff between versions
SELECT * FROM my_catalog.analytics.sales VERSION AS OF 10
EXCEPT ALL
SELECT * FROM my_catalog.analytics.sales VERSION AS OF 9;
```

### 11.2 Table Maintenance

```sql
-- Optimize (compaction + Z-Ordering)
OPTIMIZE my_catalog.analytics.sales;
OPTIMIZE my_catalog.analytics.sales WHERE sale_date >= '2025-01-01';
OPTIMIZE my_catalog.analytics.sales ZORDER BY (customer_id, sale_date);

-- Vacuum (remove old files)
VACUUM my_catalog.analytics.sales RETAIN 168 HOURS;   -- 7 days (default minimum)
VACUUM my_catalog.analytics.sales DRY RUN;             -- preview only

-- Analyze (update statistics)
ANALYZE TABLE my_catalog.analytics.sales COMPUTE STATISTICS;
ANALYZE TABLE my_catalog.analytics.sales COMPUTE STATISTICS FOR COLUMNS amount, sale_date;
ANALYZE TABLE my_catalog.analytics.sales COMPUTE DELTA STATISTICS;

-- Reorg (predictive optimization prerequisite)
REORG TABLE my_catalog.analytics.sales APPLY (PURGE);  -- purge deletion vectors
```

### 11.3 Table Properties & Details

```sql
DESCRIBE TABLE EXTENDED my_catalog.analytics.sales;
DESCRIBE DETAIL my_catalog.analytics.sales;   -- Delta-specific stats
SHOW TBLPROPERTIES my_catalog.analytics.sales;
SHOW PARTITIONS my_catalog.analytics.sales;
SHOW CREATE TABLE my_catalog.analytics.sales;

-- Change data feed
ALTER TABLE sales SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');
SELECT * FROM table_changes('my_catalog.analytics.sales', 1, 20)
WHERE _change_type IN ('insert', 'update_postimage', 'delete');
```

### 11.4 Generated & Identity Columns

```sql
CREATE TABLE orders (
  order_id   BIGINT GENERATED ALWAYS AS IDENTITY (START WITH 1 INCREMENT BY 1),
  order_date DATE   NOT NULL,
  year_month STRING GENERATED ALWAYS AS (DATE_FORMAT(order_date, 'yyyy-MM')),
  amount     DECIMAL(18,2),
  amount_eur DECIMAL(18,2) GENERATED ALWAYS AS (amount * 1.1)
)
USING DELTA;
```

-----

## 12. Unity Catalog SQL

### 12.1 Privileges

```sql
-- Grant privileges
GRANT USE CATALOG ON CATALOG my_catalog TO `data-readers`;
GRANT USE SCHEMA  ON SCHEMA  my_catalog.analytics TO `data-readers`;
GRANT SELECT      ON TABLE   my_catalog.analytics.sales TO `data-readers`;
GRANT MODIFY      ON TABLE   my_catalog.analytics.sales TO `data-engineers`;
GRANT ALL PRIVILEGES ON SCHEMA my_catalog.analytics TO `data-engineers`;

-- Revoke
REVOKE SELECT ON TABLE my_catalog.analytics.sales FROM `data-readers`;

-- Show privileges
SHOW GRANTS ON TABLE my_catalog.analytics.sales;
SHOW GRANTS TO `data-engineers`;

-- Row filters (Unity Catalog)
CREATE ROW FILTER region_filter ON my_catalog.analytics.sales (region STRING)
RETURN is_account_group_member('emea-team') OR region = 'GLOBAL';

ALTER TABLE my_catalog.analytics.sales SET ROW FILTER region_filter ON (sales_region);
ALTER TABLE my_catalog.analytics.sales DROP ROW FILTER;

-- Column masks
CREATE FUNCTION mask_pii(val STRING) RETURNS STRING
RETURN CASE WHEN is_account_group_member('pii-access') THEN val ELSE SHA2(val, 256) END;

ALTER TABLE customers ALTER COLUMN email SET MASK mask_pii;
ALTER TABLE customers ALTER COLUMN email DROP MASK;
```

### 12.2 Tags & Lineage

```sql
-- Apply tags
ALTER TABLE my_catalog.analytics.sales SET TAGS ('env' = 'prod', 'pii' = 'false');
ALTER TABLE customers ALTER COLUMN email SET TAGS ('pii' = 'true', 'gdpr_class' = 'personal');

-- Remove tags
ALTER TABLE sales UNSET TAGS ('env');

-- View tags
SELECT * FROM system.information_schema.table_tags WHERE table_name = 'sales';
```

### 12.3 Information Schema

```sql
-- List all tables in catalog
SELECT table_catalog, table_schema, table_name, table_type
FROM   my_catalog.information_schema.tables;

-- Column metadata
SELECT *
FROM   my_catalog.information_schema.columns
WHERE  table_name = 'sales' ORDER BY ordinal_position;

-- Constraints
SELECT * FROM my_catalog.information_schema.table_constraints;

-- Lineage (System tables — requires audit log)
SELECT * FROM system.access.audit LIMIT 100;
SELECT * FROM system.lineage.table_lineage WHERE target_table_full_name = 'my_catalog.analytics.sales';
```

### 12.4 External Locations & Credentials

```sql
CREATE STORAGE CREDENTIAL my_cred
WITH AZURE_MANAGED_IDENTITY (CONNECTOR_ID = '/subscriptions/.../connectors/...') ;

CREATE EXTERNAL LOCATION my_ext_loc
URL 'abfss://container@storage.dfs.core.windows.net/'
WITH (STORAGE CREDENTIAL my_cred);

SHOW EXTERNAL LOCATIONS;
VALIDATE STORAGE CREDENTIAL my_cred;
```

-----

## 13. Streaming & Lakeflow SQL

### 13.1 Structured Streaming in SQL

```sql
-- Read streaming source (Auto Loader)
CREATE STREAMING TABLE raw_events AS
SELECT * FROM cloud_files(
  'abfss://landing@storage.dfs.core.windows.net/events/',
  'json',
  map(
    'cloudFiles.inferColumnTypes', 'true',
    'cloudFiles.schemaLocation', '/checkpoints/raw_events_schema'
  )
);

-- Streaming aggregation with watermark
CREATE STREAMING TABLE hourly_event_counts AS
SELECT
  window(event_time, '1 hour').start AS window_start,
  event_type,
  COUNT(*) AS event_count
FROM STREAM(my_catalog.raw.events)
WHERE event_time > CURRENT_TIMESTAMP() - INTERVAL '7' DAYS
GROUP BY window(event_time, '1 hour'), event_type;
```

### 13.2 Delta Live Tables (DLT) SQL

```sql
-- Bronze: raw ingest
CREATE OR REFRESH STREAMING TABLE bronze_orders
COMMENT 'Raw orders from ADLS landing zone'
TBLPROPERTIES ('quality' = 'bronze')
AS
SELECT
  *,
  current_timestamp() AS _ingest_ts,
  INPUT_FILE_NAME()   AS _source_file
FROM cloud_files('abfss://landing@storage.dfs.core.windows.net/orders/', 'json',
     map('cloudFiles.inferColumnTypes', 'true'));

-- Silver: cleaned with expectations
CREATE OR REFRESH STREAMING TABLE silver_orders (
  CONSTRAINT valid_amount   EXPECT (amount > 0)        ON VIOLATION DROP ROW,
  CONSTRAINT not_null_id    EXPECT (order_id IS NOT NULL) ON VIOLATION FAIL UPDATE,
  CONSTRAINT valid_status   EXPECT (status IN ('PENDING','SHIPPED','DELIVERED','CANCELLED'))
)
COMMENT 'Validated, cleansed orders'
AS
SELECT
  CAST(order_id AS BIGINT)   AS order_id,
  CAST(customer_id AS BIGINT) AS customer_id,
  TO_DATE(order_date)         AS order_date,
  ROUND(CAST(amount AS DECIMAL(18,2)), 2) AS amount,
  UPPER(TRIM(status))        AS status,
  _ingest_ts
FROM STREAM(LIVE.bronze_orders);

-- Gold: business aggregate (batch materialized view)
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_revenue
COMMENT 'Daily revenue by region'
AS
SELECT
  order_date,
  region,
  SUM(amount)   AS total_revenue,
  COUNT(*)      AS order_count,
  AVG(amount)   AS avg_order_value
FROM LIVE.silver_orders s
JOIN LIVE.dim_region r USING (region_code)
GROUP BY order_date, region;

-- Apply changes (CDC / APPLY CHANGES INTO)
CREATE OR REFRESH STREAMING TABLE silver_customers;
APPLY CHANGES INTO LIVE.silver_customers
FROM   STREAM(LIVE.bronze_customers_cdc)
KEYS   (customer_id)
SEQUENCE BY updated_at
COLUMNS * EXCEPT (_rescued_data)
STORED AS SCD TYPE 2
TRACK HISTORY ON * EXCEPT (updated_at, _ingest_ts);
```

-----

## 14. AI & ML SQL Functions

### 14.1 Built-in AI Functions (Databricks SQL)

```sql
-- Invoke Foundation Models directly in SQL
SELECT
  order_id,
  ai_classify(review_text, ARRAY('positive', 'neutral', 'negative')) AS sentiment,
  ai_extract(review_text, ARRAY('product', 'issue', 'suggestion'))   AS extracted,
  ai_fix_grammar(review_text)                                        AS cleaned,
  ai_gen('Summarize in 20 words: ' || review_text)                   AS summary,
  ai_mask(customer_email, ARRAY('EMAIL', 'PHONE'))                   AS masked_email,
  ai_similarity(title, search_query)                                 AS relevance_score,
  ai_translate(review_text, 'DE')                                    AS review_de,
  ai_analyze_sentiment(review_text)                                  AS sentiment_score
FROM product_reviews;

-- Vector Search
SELECT
  product_id,
  title,
  VECTOR_SEARCH(
    index  => 'my_catalog.analytics.product_embeddings_index',
    query  => 'wireless noise cancelling headphones',
    num_results => 10
  ) AS search_results
FROM (VALUES (1)) t;

-- Model serving endpoint inference
SELECT
  customer_id,
  ai_query(
    'voucher-propensity-model-endpoint',
    NAMED_STRUCT(
      'recency',   recency_days,
      'frequency', purchase_count,
      'monetary',  total_spend
    )
  ):probability::DOUBLE AS churn_probability
FROM customer_features;
```

### 14.2 MLflow SQL Integration

```sql
-- Query MLflow experiment runs
SELECT * FROM mlflow.experiments WHERE name LIKE '%voucher%';
SELECT * FROM mlflow.runs WHERE experiment_id = '123456';

-- Feature Store (read from feature table)
SELECT * FROM my_catalog.feature_store.customer_rfm_features WHERE snapshot_date = CURRENT_DATE;
```

-----

## 15. Parameterized Queries & Variables

```sql
-- Session variables (Databricks SQL)
DECLARE VARIABLE v_start_date DATE DEFAULT '2025-01-01';
DECLARE VARIABLE v_region STRING;
SET VAR v_region = 'EMEA';

SELECT * FROM sales WHERE sale_date >= v_start_date AND region = v_region;

-- Widget parameters (Databricks Notebooks)
CREATE WIDGET TEXT region DEFAULT 'EMEA';
CREATE WIDGET DROPDOWN status DEFAULT 'ACTIVE' CHOICES SELECT DISTINCT status FROM customers;
CREATE WIDGET COMBOBOX start_date DEFAULT '2025-01-01';

SELECT * FROM sales WHERE region = getArgument('region');

-- Named parameters (API)
SELECT * FROM sales WHERE sale_date >= :start_date AND region = :region;

-- Positional parameters (JDBC)
SELECT * FROM sales WHERE sale_date >= ? AND region = ?;
```

-----

## 16. CTEs, Subqueries & Query Composition

```sql
-- Standard CTE
WITH
  base AS (
    SELECT customer_id, sale_date, amount, region
    FROM   sales
    WHERE  sale_date >= '2025-01-01'
  ),
  aggregated AS (
    SELECT customer_id, SUM(amount) AS total, COUNT(*) AS txn_count
    FROM   base
    GROUP  BY customer_id
  ),
  ranked AS (
    SELECT *, DENSE_RANK() OVER (ORDER BY total DESC) AS rank
    FROM   aggregated
  )
SELECT r.rank, r.customer_id, r.total, r.txn_count, c.name, c.tier
FROM   ranked r
JOIN   customers c USING (customer_id)
WHERE  r.rank <= 100;

-- Recursive CTE (hierarchical/graph traversal)
WITH RECURSIVE org_hierarchy AS (
  -- Anchor: root nodes
  SELECT employee_id, manager_id, name, 0 AS depth, CAST(name AS STRING) AS path
  FROM   employees WHERE manager_id IS NULL

  UNION ALL

  -- Recursive: children
  SELECT e.employee_id, e.manager_id, e.name, h.depth + 1,
         CONCAT(h.path, ' > ', e.name)
  FROM   employees e
  JOIN   org_hierarchy h ON e.manager_id = h.employee_id
  WHERE  h.depth < 10  -- guard against cycles
)
SELECT * FROM org_hierarchy ORDER BY path;

-- Correlated subquery
SELECT o.order_id, o.amount,
  (SELECT AVG(amount) FROM orders o2 WHERE o2.customer_id = o.customer_id) AS cust_avg
FROM orders o;

-- Scalar subquery in SELECT
SELECT customer_id,
  (SELECT MAX(sale_date) FROM sales s WHERE s.customer_id = c.customer_id) AS last_sale
FROM customers c;

-- EXISTS / NOT EXISTS
SELECT c.* FROM customers c
WHERE  EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id AND o.amount > 1000);
```

-----

## 17. Performance Tuning & Optimization

### 17.1 Explain & Analysis

```sql
EXPLAIN SELECT * FROM sales WHERE sale_date = '2025-01-01';
EXPLAIN FORMATTED SELECT * FROM sales JOIN customers USING (customer_id);
EXPLAIN COST SELECT * FROM sales;  -- with cardinality estimates

-- Execution metrics
SET spark.sql.statistics.histogram.enabled = TRUE;
ANALYZE TABLE sales COMPUTE STATISTICS FOR ALL COLUMNS;
```

### 17.2 Hints

```sql
-- Broadcast hint (force small table broadcast)
SELECT /*+ BROADCAST(c) */ s.*, c.name
FROM   sales s JOIN customers c USING (customer_id);

-- Merge hint (Sort-Merge Join)
SELECT /*+ MERGE(s, o) */ * FROM sales s JOIN orders o USING (customer_id);

-- Shuffle hash hint
SELECT /*+ SHUFFLE_HASH(a) */ * FROM big_table a JOIN medium_table b USING (id);

-- Repartition hint
SELECT /*+ REPARTITION(200) */ * FROM sales;
SELECT /*+ REPARTITION(50, sale_date) */ * FROM sales;

-- Coalesce hint
SELECT /*+ COALESCE(10) */ * FROM small_result;

-- Rebalance hint (eliminate skew in output)
SELECT /*+ REBALANCE */ * FROM sales;
SELECT /*+ REBALANCE(customer_id) */ * FROM sales;

-- Skip hints (disable specific optimizations)
SELECT /*+ NO_BROADCAST(large) */ * FROM large JOIN small USING (id);
```

### 17.3 Liquid Clustering vs Partitioning

```sql
-- Liquid Clustering (recommended for most tables, Runtime 13.3+)
CREATE TABLE sales (...) USING DELTA CLUSTER BY (customer_id, sale_date);

-- Change clustering keys without rewrite
ALTER TABLE sales CLUSTER BY (region, sale_date);

-- Trigger clustering (async)
OPTIMIZE sales;

-- Traditional partitioning (only for very high-cardinality time dimensions)
CREATE TABLE sales_partitioned (...) USING DELTA PARTITIONED BY (sale_date);

-- Z-Ordering (on partitioned tables without Liquid Clustering)
OPTIMIZE sales_partitioned ZORDER BY (customer_id, product_id);
```

### 17.4 Configuration

```sql
-- Session-level config
SET spark.sql.shuffle.partitions = 200;
SET spark.databricks.adaptive.autoOptimizeShuffle.enabled = TRUE;
SET spark.sql.adaptive.enabled = TRUE;   -- Adaptive Query Execution
SET spark.sql.adaptive.coalescePartitions.enabled = TRUE;
SET spark.sql.adaptive.skewJoin.enabled = TRUE;
SET spark.sql.autoBroadcastJoinThreshold = 52428800;  -- 50MB

-- Table-level properties
ALTER TABLE sales SET TBLPROPERTIES (
  'delta.autoOptimize.optimizeWrite' = 'true',
  'delta.autoOptimize.autoCompact'   = 'true',
  'delta.enableDeletionVectors'      = 'true',
  'delta.targetFileSize'             = '134217728',  -- 128MB
  'delta.tuneFileSizesForRewrites'   = 'true'
);
```

-----

## 18. Design Patterns

### 18.1 Medallion Architecture (Bronze → Silver → Gold)

```sql
-- Bronze: raw, schema-on-read
CREATE TABLE bronze.raw_orders USING DELTA
TBLPROPERTIES ('quality' = 'bronze', 'delta.enableChangeDataFeed' = 'true')
AS SELECT *, current_timestamp() AS _ingest_ts, INPUT_FILE_NAME() AS _source_file
FROM json.`abfss://landing@storage.dfs.core.windows.net/orders/`;

-- Silver: typed, deduplicated, validated
CREATE OR REPLACE TABLE silver.orders USING DELTA AS
WITH deduped AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY _ingest_ts DESC) AS rn
  FROM bronze.raw_orders
)
SELECT
  CAST(order_id AS BIGINT)    AS order_id,
  CAST(customer_id AS BIGINT) AS customer_id,
  TO_DATE(order_date)         AS order_date,
  CAST(amount AS DECIMAL(18,2)) AS amount,
  UPPER(TRIM(status))         AS status
FROM deduped WHERE rn = 1 AND order_id IS NOT NULL AND amount > 0;

-- Gold: business-ready aggregates
CREATE OR REPLACE TABLE gold.monthly_revenue USING DELTA AS
SELECT
  DATE_TRUNC('month', order_date) AS month,
  region,
  SUM(amount)  AS revenue,
  COUNT(*)     AS order_count,
  COUNT(DISTINCT customer_id) AS unique_customers
FROM silver.orders o JOIN silver.customers c USING (customer_id)
GROUP BY 1, 2;
```

### 18.2 Incremental Load Pattern

```sql
-- Watermark-based incremental load
WITH new_records AS (
  SELECT * FROM source_table
  WHERE updated_at > (
    SELECT COALESCE(MAX(updated_at), '1900-01-01') FROM target_table
  )
)
MERGE INTO target_table AS tgt
USING new_records AS src ON tgt.id = src.id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

### 18.3 Data Vault Pattern

```sql
-- Hub (unique business keys)
CREATE TABLE dv.hub_customer (
  hub_customer_hk STRING NOT NULL,  -- HASH of customer_bk
  customer_bk     STRING NOT NULL,  -- business key
  load_date       TIMESTAMP NOT NULL DEFAULT current_timestamp(),
  record_source   STRING NOT NULL
) USING DELTA;

-- Link (relationships)
CREATE TABLE dv.link_order_customer (
  link_hk         STRING NOT NULL,  -- HASH(order_bk||customer_bk)
  hub_order_hk    STRING NOT NULL,
  hub_customer_hk STRING NOT NULL,
  load_date       TIMESTAMP NOT NULL DEFAULT current_timestamp(),
  record_source   STRING NOT NULL
) USING DELTA;

-- Satellite (descriptive attributes + history)
CREATE TABLE dv.sat_customer_details (
  hub_customer_hk STRING NOT NULL,
  load_date       TIMESTAMP NOT NULL,
  load_end_date   TIMESTAMP,
  hash_diff       STRING NOT NULL,  -- HASH of all tracked columns
  name            STRING,
  email           STRING,
  tier            STRING,
  record_source   STRING NOT NULL
) USING DELTA CLUSTER BY (hub_customer_hk);

-- Load Hub (idempotent)
MERGE INTO dv.hub_customer AS tgt
USING (
  SELECT DISTINCT
    SHA2(UPPER(TRIM(customer_id)), 256) AS hub_customer_hk,
    customer_id AS customer_bk,
    'CRM' AS record_source
  FROM staging.customer_updates
) AS src ON tgt.hub_customer_hk = src.hub_customer_hk
WHEN NOT MATCHED THEN INSERT *;
```

### 18.4 SCD Type 2 Pattern

```sql
-- Close existing records
UPDATE dim_customer
SET    eff_end_date = current_timestamp(), is_current = FALSE
WHERE  is_current = TRUE
AND    customer_id IN (
  SELECT customer_id FROM staging.customer_updates s
  JOIN   dim_customer d USING (customer_id)
  WHERE  d.is_current = TRUE AND (d.name <> s.name OR d.email <> s.email)
);

-- Insert new versions
INSERT INTO dim_customer
SELECT
  UUID()                AS surrogate_key,
  customer_id,
  name, email, tier,
  current_timestamp()   AS eff_start_date,
  NULL                  AS eff_end_date,
  TRUE                  AS is_current
FROM staging.customer_updates s
WHERE EXISTS (
  SELECT 1 FROM dim_customer d
  WHERE d.customer_id = s.customer_id AND d.is_current = TRUE
    AND (d.name <> s.name OR d.email <> s.email)
)
OR NOT EXISTS (SELECT 1 FROM dim_customer WHERE customer_id = s.customer_id);
```

### 18.5 Slowly Changing Dimension with MERGE

```sql
-- Efficient SCD2 with a single MERGE (requires pre-processing)
WITH changes AS (
  SELECT s.customer_id, s.name, s.email, s.tier,
         MD5(CONCAT_WS('|', s.name, s.email, s.tier)) AS hash_diff
  FROM staging.customer_updates s
),
to_expire AS (
  SELECT d.surrogate_key, c.customer_id, c.name, c.email, c.tier
  FROM dim_customer d JOIN changes c USING (customer_id)
  WHERE d.is_current AND d.hash_diff <> c.hash_diff
)
MERGE INTO dim_customer AS tgt
USING (
  SELECT surrogate_key, NULL AS customer_id, NULL AS name, NULL AS email,
         NULL AS tier, NULL AS hash_diff, FALSE AS is_current, current_timestamp() AS eff_end
  FROM to_expire
  UNION ALL
  SELECT UUID(), customer_id, name, email, tier, hash_diff, TRUE, NULL
  FROM to_expire
  UNION ALL
  SELECT UUID(), customer_id, name, email, tier, hash_diff, TRUE, NULL
  FROM changes c WHERE NOT EXISTS (SELECT 1 FROM dim_customer d WHERE d.customer_id = c.customer_id)
) AS src (surrogate_key, customer_id, name, email, tier, hash_diff, is_current, eff_end)
ON tgt.surrogate_key = src.surrogate_key
WHEN MATCHED THEN UPDATE SET tgt.is_current = FALSE, tgt.eff_end_date = src.eff_end
WHEN NOT MATCHED THEN INSERT (surrogate_key, customer_id, name, email, tier, hash_diff, is_current, eff_start_date, eff_end_date)
  VALUES (src.surrogate_key, src.customer_id, src.name, src.email, src.tier, src.hash_diff, TRUE, current_timestamp(), NULL);
```

### 18.6 Session Analysis Pattern

```sql
WITH sessions AS (
  SELECT
    customer_id,
    event_time,
    event_type,
    -- New session if gap > 30 min from previous event
    CASE
      WHEN TIMESTAMPDIFF(MINUTE,
             LAG(event_time) OVER (PARTITION BY customer_id ORDER BY event_time),
             event_time) > 30
        OR LAG(event_time) OVER (PARTITION BY customer_id ORDER BY event_time) IS NULL
      THEN 1 ELSE 0
    END AS new_session_flag
  FROM website_events
),
session_ids AS (
  SELECT *,
    SUM(new_session_flag) OVER (PARTITION BY customer_id ORDER BY event_time
                                 ROWS UNBOUNDED PRECEDING) AS session_id
  FROM sessions
)
SELECT
  customer_id,
  session_id,
  MIN(event_time)  AS session_start,
  MAX(event_time)  AS session_end,
  TIMESTAMPDIFF(MINUTE, MIN(event_time), MAX(event_time)) AS duration_min,
  COUNT(*)         AS event_count,
  COUNT(DISTINCT event_type) AS unique_event_types
FROM session_ids
GROUP BY customer_id, session_id;
```

-----

## 19. Best Practices

### Query Writing

|Practice                                                  |Detail                                          |
|----------------------------------------------------------|------------------------------------------------|
|**Use three-level namespacing**                           |Always `catalog.schema.table` in production code|
|**Prefer MERGE over DELETE + INSERT**                     |Atomic, supports concurrency, Delta-native      |
|**Avoid `SELECT *` in production**                        |Explicit columns → schema evolution safety      |
|**Use `QUALIFY` instead of subqueries for window filters**|More readable, same performance                 |
|**Use `LATERAL VIEW OUTER EXPLODE`**                      |Keeps rows with empty/null arrays               |
|**Use `TRY_CAST` for untrusted data**                     |Prevents job failure on bad data                |
|**Prefer `COALESCE` over `IFNULL`**                       |ANSI standard; works with multiple args         |
|**Filter before JOIN**                                    |Push predicates down; reduces shuffle           |
|**Use CTEs instead of nested subqueries**                 |Readability and plan reuse                      |
|**Avoid UDFs in hot paths**                               |They break Photon optimization; use built-ins   |

### Delta Lake

|Practice                                          |Detail                                          |
|--------------------------------------------------|------------------------------------------------|
|**Enable Deletion Vectors**                       |Faster deletes/updates; avoids full file rewrite|
|**Use Liquid Clustering over manual partitioning**|Auto-rebalances; no over-partitioning           |
|**Run OPTIMIZE regularly**                        |Compact small files; schedule via Workflows     |
|**Set `autoOptimize` properties**                 |Reduces manual maintenance burden               |
|**Always VACUUM**                                 |Prevents unbounded storage growth; 7-day default|
|**Enable Change Data Feed**                       |Required for efficient incremental processing   |
|**Use generated columns for derived partitions**  |e.g., `year_month` from `sale_date`             |

### Performance

|Practice                               |Detail                                                     |
|---------------------------------------|-----------------------------------------------------------|
|**Enable AQE**                         |`spark.sql.adaptive.enabled = true` (default on Runtime 9+)|
|**Broadcast small tables explicitly**  |Use `/*+ BROADCAST(t) */` when optimizer misses            |
|**Use Photon-compatible SQL**          |Avoid Python UDFs; use built-in SQL functions              |
|**Predicate pushdown on partitions**   |Filter on partition columns = partition pruning            |
|**Use `APPROX_COUNT_DISTINCT`**        |Dramatically faster than `COUNT(DISTINCT)` at scale        |
|**Avoid `collect_list` on huge arrays**|OOM risk; consider `ARRAY_AGG` with limits                 |
|**Use `REBALANCE` hint**               |Fixes output skew without full repartition                 |
|**Cache hot reference tables**         |`CACHE TABLE dim_product;` for repeated joins              |

### Security

|Practice                          |Detail                                           |
|----------------------------------|-------------------------------------------------|
|**Always use Unity Catalog**      |Single governance plane across workspaces        |
|**Principle of least privilege**  |Grant minimum required privileges                |
|**Use row filters + column masks**|Dynamic security; no view proliferation          |
|**Tag PII columns**               |`ALTER COLUMN email SET TAGS ('pii' = 'true')`   |
|**Never hardcode credentials**    |Use `dbutils.secrets` / Unity Catalog credentials|
|**Audit via system tables**       |`system.access.audit` for compliance             |

-----

## 20. Quick-Reference Cheat Snippets

```sql
-- ── LAST N DAYS ──────────────────────────────────────────────────────────────
WHERE sale_date >= CURRENT_DATE - INTERVAL '30' DAY

-- ── CURRENT MONTH ────────────────────────────────────────────────────────────
WHERE DATE_TRUNC('month', sale_date) = DATE_TRUNC('month', CURRENT_DATE)

-- ── DEDUPLICATE (keep latest) ─────────────────────────────────────────────────
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) AS rn FROM t
) WHERE rn = 1

-- ── TOP N PER GROUP ────────────────────────────────────────────────────────────
SELECT * FROM (
  SELECT *, DENSE_RANK() OVER (PARTITION BY region ORDER BY amount DESC) AS rnk FROM sales
) WHERE rnk <= 3

-- ── RUNNING TOTAL ─────────────────────────────────────────────────────────────
SUM(amount) OVER (PARTITION BY customer_id ORDER BY sale_date ROWS UNBOUNDED PRECEDING)

-- ── YOY GROWTH ────────────────────────────────────────────────────────────────
SELECT year, revenue,
       LAG(revenue) OVER (ORDER BY year) AS prev_year,
       ROUND((revenue - LAG(revenue) OVER (ORDER BY year)) /
              LAG(revenue) OVER (ORDER BY year) * 100, 2) AS yoy_pct
FROM annual_revenue

-- ── PIVOT WITHOUT PIVOT KEYWORD ───────────────────────────────────────────────
SELECT
  region,
  SUM(CASE WHEN quarter = 'Q1' THEN revenue END) AS Q1,
  SUM(CASE WHEN quarter = 'Q2' THEN revenue END) AS Q2,
  SUM(CASE WHEN quarter = 'Q3' THEN revenue END) AS Q3,
  SUM(CASE WHEN quarter = 'Q4' THEN revenue END) AS Q4
FROM revenue GROUP BY region

-- ── SAFE DIVIDE ───────────────────────────────────────────────────────────────
COALESCE(numerator / NULLIF(denominator, 0), 0)

-- ── STRING SPLIT TO ROWS ──────────────────────────────────────────────────────
SELECT id, tag
FROM t
LATERAL VIEW EXPLODE(SPLIT(tags_str, ',')) v AS tag

-- ── PARSE NESTED JSON INLINE ──────────────────────────────────────────────────
SELECT json_col:address.city::STRING AS city,
       json_col:orders[0].amount::DOUBLE AS first_order_amount
FROM raw_events

-- ── GENERATE DATE SPINE ───────────────────────────────────────────────────────
SELECT EXPLODE(SEQUENCE(
  TO_DATE('2025-01-01'),
  CURRENT_DATE,
  INTERVAL '1' DAY
)) AS dt

-- ── CONDITIONAL AGGREGATION ───────────────────────────────────────────────────
SELECT
  customer_id,
  COUNT(*) FILTER (WHERE status = 'COMPLETED')  AS completed,
  COUNT(*) FILTER (WHERE status = 'CANCELLED')  AS cancelled,
  SUM(amount) FILTER (WHERE region = 'EMEA')    AS emea_revenue
FROM orders GROUP BY customer_id

-- ── ARRAY TO STRING ───────────────────────────────────────────────────────────
ARRAY_JOIN(tags, ', ')
CONCAT_WS(', ', tag1, tag2, tag3)  -- skips NULLs

-- ── FLATTEN NESTED ARRAY ──────────────────────────────────────────────────────
SELECT FLATTEN(ARRAY(ARRAY(1,2), ARRAY(3,4)))  -- → [1,2,3,4]

-- ── MAP FROM TWO ARRAYS ───────────────────────────────────────────────────────
SELECT MAP_FROM_ARRAYS(ARRAY('a','b','c'), ARRAY(1,2,3))

-- ── HASH BUSINESS KEY ─────────────────────────────────────────────────────────
SHA2(UPPER(TRIM(CONCAT_WS('|', col1, col2))), 256) AS hash_key

-- ── UPSERT SHORTHAND ──────────────────────────────────────────────────────────
MERGE INTO tgt USING src ON tgt.id = src.id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *

-- ── TIME TRAVEL DIFF ──────────────────────────────────────────────────────────
SELECT * FROM t VERSION AS OF 10
EXCEPT ALL
SELECT * FROM t VERSION AS OF 9

-- ── CHANGE DATA FEED ──────────────────────────────────────────────────────────
SELECT * FROM table_changes('catalog.schema.table', 5, 10)
WHERE _change_type = 'update_postimage'

-- ── AI CLASSIFY IN BATCH ──────────────────────────────────────────────────────
SELECT id, text,
  ai_classify(text, ARRAY('complaint','praise','question','other')) AS category
FROM support_tickets

-- ── SCHEMA INFERENCE FROM JSON ────────────────────────────────────────────────
SELECT SCHEMA_OF_JSON(raw_col) FROM events LIMIT 1

-- ── CHECK TABLE HEALTH ────────────────────────────────────────────────────────
DESCRIBE DETAIL catalog.schema.table
-- → numFiles, sizeInBytes, numRowGroups, partitionColumns, clusteringColumns
```

-----

## Data Types Quick Reference

|Type             |Description                     |Example Literal                 |
|-----------------|--------------------------------|--------------------------------|
|`TINYINT`        |1-byte signed integer           |`1Y`                            |
|`SMALLINT`       |2-byte signed integer           |`1S`                            |
|`INT` / `INTEGER`|4-byte signed integer           |`42`                            |
|`BIGINT`         |8-byte signed integer           |`9999999999L`                   |
|`FLOAT` / `REAL` |4-byte float                    |`3.14F`                         |
|`DOUBLE`         |8-byte float                    |`3.14`                          |
|`DECIMAL(p, s)`  |Exact numeric                   |`DECIMAL(18,2)`                 |
|`STRING`         |Variable-length UTF-8           |`'hello'`                       |
|`CHAR(n)`        |Fixed-length string             |                                |
|`VARCHAR(n)`     |Max-length string               |                                |
|`BINARY`         |Byte sequence                   |`X'DEADBEEF'`                   |
|`BOOLEAN`        |True/false                      |`TRUE`, `FALSE`                 |
|`DATE`           |Calendar date                   |`DATE'2025-01-15'`              |
|`TIMESTAMP`      |Date + time (µs)                |`TIMESTAMP'2025-01-15 12:00:00'`|
|`TIMESTAMP_NTZ`  |No timezone                     |                                |
|`INTERVAL`       |Duration                        |`INTERVAL '1' YEAR`             |
|`ARRAY<T>`       |Ordered collection              |`ARRAY(1, 2, 3)`                |
|`MAP<K,V>`       |Key-value pairs                 |`MAP('a', 1)`                   |
|`STRUCT<f:T,...>`|Named fields                    |`NAMED_STRUCT('x', 1)`          |
|`VARIANT`        |Semi-structured (Databricks SQL)|`PARSE_JSON('{}')`              |
|`VOID`           |Null-only type                  |                                |

-----

*Reference: Databricks Documentation, Delta Lake OSS, Unity Catalog, Databricks SQL Reference, Spark SQL Reference — 2026*
