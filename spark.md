# PySpark on Databricks — Complete Reference Card

> **Databricks Runtime** | Python API (pyspark) | Delta Lake | Unity Catalog  
> Covers: Spark 3.x / DBR 13+ / Delta 2.x

-----

## Table of Contents

1. [Session & Context Setup](#1-session--context-setup)
1. [Data Ingestion & Reading](#2-data-ingestion--reading)
1. [DataFrame API — Transformations](#3-dataframe-api--transformations)
1. [Column Operations & Expressions](#4-column-operations--expressions)
1. [Built-in Functions Reference](#5-built-in-functions-reference)
1. [Aggregations & Grouping](#6-aggregations--grouping)
1. [Joins](#7-joins)
1. [Window Functions](#8-window-functions)
1. [Streaming (Structured Streaming)](#9-streaming-structured-streaming)
1. [Delta Lake Operations](#10-delta-lake-operations)
1. [Writing & Output](#11-writing--output)
1. [Schema & Type System](#12-schema--type-system)
1. [UDFs & Pandas UDFs (Arrow)](#13-udfs--pandas-udfs-arrow)
1. [SQL Integration](#14-sql-integration)
1. [Optimisation & Performance](#15-optimisation--performance)
1. [Databricks-Specific APIs](#16-databricks-specific-apis)
1. [Unity Catalog & Governance](#17-unity-catalog--governance)
1. [Design Patterns & Best Practices](#18-design-patterns--best-practices)
1. [Master Method Reference Table](#19-master-method-reference-table)

-----

## 1. Session & Context Setup

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql import types as T
from pyspark.sql.window import Window

# On Databricks, `spark` is pre-created — no need to instantiate
spark  # already available in notebooks

# Build a custom session (local dev / unit tests)
spark = (
    SparkSession.builder
    .appName("MyApp")
    .config("spark.sql.adaptive.enabled", "true")
    .config("spark.sql.shuffle.partitions", "200")
    .config("spark.databricks.delta.optimizeWrite.enabled", "true")
    .getOrCreate()
)

# Access underlying SparkContext
sc = spark.sparkContext
sc.setLogLevel("WARN")          # ERROR | WARN | INFO | DEBUG

# Useful session properties
spark.version                   # Spark version string
spark.conf.get("spark.sql.shuffle.partitions")
spark.conf.set("spark.sql.shuffle.partitions", "400")
spark.catalog.listDatabases()
spark.catalog.listTables("my_db")
spark.catalog.currentDatabase()
spark.catalog.setCurrentDatabase("my_db")
```

-----

## 2. Data Ingestion & Reading

### Common Read Patterns

```python
# Parquet
df = spark.read.parquet("/mnt/data/path/")
df = spark.read.format("parquet").load("/mnt/data/path/")

# Delta (preferred on Databricks)
df = spark.read.format("delta").load("/mnt/delta/table/")
df = spark.table("catalog.schema.table")          # Unity Catalog 3-part name

# CSV
df = (
    spark.read
    .option("header", "true")
    .option("inferSchema", "true")
    .option("sep", ",")
    .option("quote", '"')
    .option("escape", "\\")
    .option("nullValue", "NULL")
    .option("dateFormat", "yyyy-MM-dd")
    .option("timestampFormat", "yyyy-MM-dd HH:mm:ss")
    .option("multiLine", "true")
    .csv("/mnt/data/*.csv")
)

# JSON
df = spark.read.option("multiLine", "true").json("/mnt/data/")

# Avro
df = spark.read.format("avro").load("/mnt/data/")

# JDBC
df = (
    spark.read.format("jdbc")
    .option("url", "jdbc:postgresql://host:5432/db")
    .option("dbtable", "schema.table")
    .option("user", "user").option("password", "pass")
    .option("numPartitions", 8)
    .option("partitionColumn", "id")
    .option("lowerBound", 1).option("upperBound", 1000000)
    .load()
)

# Explicit schema (avoids full scan)
schema = T.StructType([
    T.StructField("id",    T.LongType(),   nullable=False),
    T.StructField("name",  T.StringType(), nullable=True),
    T.StructField("ts",    T.TimestampType()),
])
df = spark.read.schema(schema).parquet("/mnt/data/")
```

### Delta Time Travel

```python
df = spark.read.format("delta").option("versionAsOf", 5).load("/path/")
df = spark.read.format("delta").option("timestampAsOf", "2024-01-01").load("/path/")
```

-----

## 3. DataFrame API — Transformations

### Selection & Filtering

```python
# Select columns
df.select("col1", "col2")
df.select(F.col("col1"), F.col("col2").alias("c2"))
df.select(df["col1"], df.col2)
df.selectExpr("col1", "col2 * 2 as col2_double", "UPPER(name) as name_upper")

# Filter / Where (interchangeable)
df.filter(F.col("age") > 30)
df.filter("age > 30")                              # SQL string
df.where(F.col("status").isin("A", "B", "C"))
df.filter(F.col("name").isNotNull())
df.filter(~F.col("flag"))                          # NOT
df.filter((F.col("a") > 1) & (F.col("b") < 10))   # AND
df.filter((F.col("a") > 1) | (F.col("b") < 10))   # OR

# Drop columns
df.drop("col1", "col2")

# Rename
df.withColumnRenamed("old_name", "new_name")
df.toDF("c1", "c2", "c3")                         # rename all

# Add / modify column
df.withColumn("new_col", F.col("a") + F.col("b"))
df.withColumn("flag", F.when(F.col("x") > 0, True).otherwise(False))

# Distinct
df.distinct()
df.dropDuplicates(["col1", "col2"])                # subset dedupe

# Limit & sample
df.limit(100)
df.sample(fraction=0.1, seed=42)
df.sample(withReplacement=False, fraction=0.1)
```

### Sorting & Ordering

```python
df.orderBy("col1")
df.orderBy(F.col("col1").asc(), F.col("col2").desc())
df.orderBy(F.col("col1").asc_nulls_last())
df.orderBy(F.col("col1").desc_nulls_first())
df.sortWithinPartitions("col1")                    # cheaper — sort per partition
```

### Reshaping

```python
# Explode array/map
df.withColumn("item", F.explode("array_col"))
df.withColumn("item", F.explode_outer("array_col"))   # keeps NULLs
df.select(F.posexplode("array_col").alias("pos", "item"))

# Pivot
df.groupBy("year").pivot("month").agg(F.sum("revenue"))
df.groupBy("year").pivot("month", ["Jan","Feb","Mar"]).agg(F.sum("revenue"))

# Unpivot (Spark 3.4+)
df.unpivot(["id"], ["col_a", "col_b"], "variable", "value")

# Flatten nested struct
df.select("top.*")                                 # expand struct fields
df.select(F.col("struct_col.field_name"))

# Cast
df.withColumn("col", F.col("col").cast(T.DoubleType()))
df.withColumn("col", F.col("col").cast("double"))
```

-----

## 4. Column Operations & Expressions

```python
from pyspark.sql.functions import col, lit, expr

# Column construction
col("name")           # column reference
lit(42)               # literal value
expr("a + b * 2")     # SQL expression string

# Arithmetic
col("a") + col("b")
col("a") - lit(1)
col("a") * col("b")
col("a") / col("b")
col("a") % lit(2)      # modulo
col("a") ** 2          # power

# Comparison
col("a") == col("b")   # equality  (== not = )
col("a") != col("b")
col("a") > lit(0)
col("a").between(1, 10)
col("a").isin([1, 2, 3])
col("a").isNull()
col("a").isNotNull()

# String column ops
col("s").like("%pattern%")
col("s").rlike(r"^\d{4}-\d{2}")   # regex
col("s").startswith("pre")
col("s").endswith("suf")
col("s").contains("sub")

# Alias
col("revenue").alias("rev")
(col("a") + col("b")).alias("sum_ab")

# Boolean column
~col("flag")                       # NOT
col("a") & col("b")                # AND (bitwise on boolean col)
col("a") | col("b")                # OR
```

-----

## 5. Built-in Functions Reference

```python
import pyspark.sql.functions as F
```

### String Functions

|Function                                       |Description               |
|-----------------------------------------------|--------------------------|
|`F.upper(c)`                                   |Uppercase                 |
|`F.lower(c)`                                   |Lowercase                 |
|`F.trim(c)`                                    |Strip whitespace both ends|
|`F.ltrim(c)` / `F.rtrim(c)`                    |Strip left / right        |
|`F.lpad(c, n, pad)` / `F.rpad(c, n, pad)`      |Pad string                |
|`F.length(c)`                                  |String length             |
|`F.substring(c, pos, len)`                     |Substring (1-indexed)     |
|`F.split(c, pattern, limit=-1)`                |Split to array            |
|`F.regexp_replace(c, pat, rep)`                |Regex replace             |
|`F.regexp_extract(c, pat, group)`              |Regex extract group       |
|`F.concat(c1, c2, ...)`                        |Concatenate               |
|`F.concat_ws(sep, c1, c2, ...)`                |Concatenate with separator|
|`F.format_string(fmt, *cols)`                  |printf-style format       |
|`F.translate(c, match, replace)`               |Char-by-char replace      |
|`F.initcap(c)`                                 |Title case                |
|`F.instr(c, substr)`                           |Position of substring     |
|`F.locate(substr, c, pos=1)`                   |Locate substring          |
|`F.reverse(c)`                                 |Reverse string            |
|`F.repeat(c, n)`                               |Repeat string             |
|`F.soundex(c)`                                 |Soundex code              |
|`F.levenshtein(c1, c2)`                        |Edit distance             |
|`F.base64(c)` / `F.unbase64(c)`                |Base64 encode/decode      |
|`F.md5(c)` / `F.sha1(c)` / `F.sha2(c, n)`      |Hashing                   |
|`F.xxhash64(*cols)`                            |Fast hash                 |
|`F.encode(c, charset)` / `F.decode(c, charset)`|Encoding                  |

### Numeric Functions

|Function                                            |Description             |
|----------------------------------------------------|------------------------|
|`F.abs(c)`                                          |Absolute value          |
|`F.round(c, scale=0)`                               |Round                   |
|`F.bround(c, scale=0)`                              |Banker’s rounding       |
|`F.ceil(c)` / `F.floor(c)`                          |Ceiling / floor         |
|`F.sqrt(c)`                                         |Square root             |
|`F.cbrt(c)`                                         |Cube root               |
|`F.exp(c)` / `F.log(c)` / `F.log2(c)` / `F.log10(c)`|Exp / log               |
|`F.pow(c, n)`                                       |Power                   |
|`F.signum(c)`                                       |Sign (-1, 0, 1)         |
|`F.greatest(*cols)` / `F.least(*cols)`              |Row-wise max/min        |
|`F.rand(seed=None)`                                 |Random uniform [0,1)    |
|`F.randn(seed=None)`                                |Random normal           |
|`F.rint(c)`                                         |Round to nearest integer|
|`F.factorial(c)`                                    |Factorial               |
|`F.shiftleft(c, n)` / `F.shiftright(c, n)`          |Bit shift               |
|`F.bitwiseNOT(c)`                                   |Bitwise NOT             |
|`F.bin(c)` / `F.hex(c)` / `F.unhex(c)`              |Base conversion         |

### Date & Timestamp Functions

|Function                                               |Description                     |
|-------------------------------------------------------|--------------------------------|
|`F.current_date()`                                     |Today’s date                    |
|`F.current_timestamp()`                                |Now (cluster TZ)                |
|`F.now()`                                              |Alias of current_timestamp      |
|`F.to_date(c, fmt)`                                    |Parse to date                   |
|`F.to_timestamp(c, fmt)`                               |Parse to timestamp              |
|`F.date_format(c, fmt)`                                |Format date                     |
|`F.year(c)` / `F.month(c)` / `F.dayofmonth(c)`         |Extract parts                   |
|`F.hour(c)` / `F.minute(c)` / `F.second(c)`            |Extract time                    |
|`F.weekofyear(c)` / `F.dayofweek(c)` / `F.dayofyear(c)`|Calendar                        |
|`F.quarter(c)`                                         |Quarter (1-4)                   |
|`F.trunc(c, fmt)`                                      |Truncate to unit (year/month)   |
|`F.date_trunc(unit, c)`                                |Truncate (year/month/day/hour/…)|
|`F.date_add(c, n)` / `F.date_sub(c, n)`                |Add/subtract days               |
|`F.add_months(c, n)`                                   |Add months                      |
|`F.months_between(c1, c2)`                             |Months diff (decimal)           |
|`F.datediff(end, start)`                               |Day difference                  |
|`F.next_day(c, dayOfWeek)`                             |Next given weekday              |
|`F.last_day(c)`                                        |Last day of month               |
|`F.unix_timestamp(c, fmt)`                             |To Unix epoch (seconds)         |
|`F.from_unixtime(c, fmt)`                              |From Unix epoch                 |
|`F.from_utc_timestamp(c, tz)`                          |UTC to local                    |
|`F.to_utc_timestamp(c, tz)`                            |Local to UTC                    |
|`F.make_date(y, m, d)`                                 |Construct date                  |
|`F.make_timestamp(y, mo, d, h, mi, s)`                 |Construct timestamp             |
|`F.timestamp_seconds(c)`                               |Epoch seconds to timestamp      |
|`F.datepart(field, c)`                                 |Extract any field (Spark 3.4+)  |

### Null Handling

```python
F.isnull(col)                              # True if null
F.isnan(col)                               # True if NaN (float)
F.coalesce(c1, c2, c3)                     # First non-null
F.nullif(c1, c2)                           # NULL if c1 == c2
F.ifnull(c, default)                       # NULL -> default
F.nvl(c, default)                          # Alias for ifnull
F.nvl2(c, not_null_val, null_val)          # Switch on null
F.nanvl(c1, c2)                            # If NaN -> c2
df.na.fill({"col1": 0, "col2": "unknown"}) # Fill NULLs
df.na.drop(subset=["col1", "col2"])        # Drop rows with NULLs
df.na.replace({"old": "new"}, subset=["col"])
```

### Conditional / Control Flow

```python
# CASE WHEN
F.when(F.col("score") >= 90, "A") \
 .when(F.col("score") >= 80, "B") \
 .otherwise("C")

# IF
F.expr("IF(a > 0, 'positive', 'non-positive')")

# Switch (Spark 3.5+)
F.case_when([
    (F.col("x") > 10, lit("big")),
    (F.col("x") > 0,  lit("small")),
], default=lit("zero"))
```

### Array Functions

|Function                                  |Description                |
|------------------------------------------|---------------------------|
|`F.array(c1, c2, ...)`                    |Create array               |
|`F.array_contains(c, val)`                |Check membership           |
|`F.array_distinct(c)`                     |Remove duplicates          |
|`F.array_except(c1, c2)`                  |Set difference             |
|`F.array_intersect(c1, c2)`               |Set intersection           |
|`F.array_union(c1, c2)`                   |Set union                  |
|`F.array_join(c, delim, null_rep)`        |Join to string             |
|`F.array_max(c)` / `F.array_min(c)`       |Max/min element            |
|`F.array_position(c, val)`                |1-indexed position         |
|`F.array_remove(c, val)`                  |Remove all occurrences     |
|`F.array_repeat(val, count)`              |Repeat element             |
|`F.array_sort(c)` / `F.sort_array(c, asc)`|Sort array                 |
|`F.array_zip(c1, c2)`                     |Zip arrays                 |
|`F.arrays_overlap(c1, c2)`                |Any common element         |
|`F.flatten(c)`                            |Flatten nested array       |
|`F.reverse(c)`                            |Reverse array              |
|`F.sequence(start, stop, step)`           |Generate sequence          |
|`F.shuffle(c)`                            |Random shuffle             |
|`F.size(c)`                               |Array/map length           |
|`F.slice(c, start, length)`               |Subarray                   |
|`F.transform(c, func)`                    |Map over array (lambda)    |
|`F.filter(c, func)`                       |Filter array (lambda)      |
|`F.aggregate(c, init, merge, finish)`     |Reduce array               |
|`F.exists(c, func)`                       |Any element matches        |
|`F.forall(c, func)`                       |All elements match         |
|`F.zip_with(c1, c2, func)`                |Zip with transform         |
|`F.explode(c)`                            |Explode to rows            |
|`F.posexplode(c)`                         |Explode with position      |
|`F.collect_list(c)`                       |Aggregate to array         |
|`F.collect_set(c)`                        |Aggregate to distinct array|
|`F.element_at(c, idx)`                    |Element at 1-based index   |

### Map Functions

|Function                               |Description                  |
|---------------------------------------|-----------------------------|
|`F.create_map(k1,v1,k2,v2,...)`        |Create map literal           |
|`F.map_from_arrays(keys_col, vals_col)`|Map from two arrays          |
|`F.map_keys(c)`                        |Extract keys array           |
|`F.map_values(c)`                      |Extract values array         |
|`F.map_entries(c)`                     |Array of structs {key, value}|
|`F.map_from_entries(c)`                |Array of structs to map      |
|`F.map_concat(c1, c2)`                 |Merge maps                   |
|`F.map_contains_key(c, key)`           |Key existence                |
|`F.map_filter(c, func)`                |Filter map entries           |
|`F.map_zip_with(c1, c2, func)`         |Zip two maps                 |
|`F.element_at(c, key)`                 |Value by key                 |
|`F.explode(c)`                         |Explode to key/value rows    |

### Struct Functions

```python
F.struct("a", "b", "c")                     # Create struct
F.struct(F.col("a").alias("x"), F.col("b")) # Named struct
F.col("struct_col.field")                   # Access field
F.col("struct_col.*")                       # Expand all fields

# Named struct (Spark 3.4+)
F.named_struct(lit("key"), col("val"))
```

### JSON Functions

```python
F.from_json(col("json_str"), schema)             # Parse JSON string
F.to_json(col("struct_or_map"))                  # Serialize to JSON
F.get_json_object(col("json_str"), "$.key")      # JSONPath extraction
F.json_tuple(col("json_str"), "k1", "k2")        # Multiple keys to cols
F.schema_of_json(lit('{"a":1}'))                 # Infer schema string
F.json_array_length(col("json_arr"))             # Array length
```

-----

## 6. Aggregations & Grouping

```python
# Basic groupBy -> agg
df.groupBy("dept").agg(
    F.count("*").alias("cnt"),
    F.sum("salary").alias("total"),
    F.avg("salary").alias("avg_sal"),
    F.min("salary").alias("min_sal"),
    F.max("salary").alias("max_sal"),
    F.stddev("salary").alias("stddev"),
    F.variance("salary").alias("var"),
    F.countDistinct("emp_id").alias("unique_emps"),
    F.first("name", ignorenulls=True).alias("first_name"),
    F.last("name",  ignorenulls=True).alias("last_name"),
    F.collect_list("name").alias("names"),
    F.collect_set("status").alias("statuses"),
    F.approx_count_distinct("user_id", rsd=0.05).alias("approx_users"),
    F.percentile_approx("salary", [0.25, 0.5, 0.75]).alias("quartiles"),
    F.kurtosis("salary"),
    F.skewness("salary"),
    F.corr("salary", "bonus").alias("corr"),
    F.covar_samp("salary", "bonus").alias("covariance"),
)

# Rollup (hierarchical subtotals)
df.rollup("year", "month").agg(F.sum("revenue"))

# Cube (all combinations)
df.cube("year", "month", "region").agg(F.sum("revenue"))

# GROUPING SETS (SQL preferred)
spark.sql("""
  SELECT year, month, SUM(revenue)
  FROM t
  GROUP BY GROUPING SETS ((year, month), (year), ())
""")

# Pivot
df.groupBy("country").pivot("year").agg(F.sum("sales"))

# Multiple percentiles on same column
df.groupBy("dept").agg(
    *[F.percentile_approx("salary", q).alias(f"p{int(q*100)}")
      for q in [0.5, 0.75, 0.90, 0.95, 0.99]]
)

# Filter after aggregation (HAVING equivalent)
df.groupBy("dept").agg(F.count("*").alias("cnt")).filter(F.col("cnt") > 10)
```

-----

## 7. Joins

```python
# Join types: inner, left, right, full, left_semi, left_anti, cross
df1.join(df2, on="key", how="inner")
df1.join(df2, on=["k1", "k2"], how="left")
df1.join(df2, df1.id == df2.id, how="inner")        # explicit condition
df1.join(df2, (df1.a == df2.a) & (df1.b > df2.b))  # compound condition

# Avoid column ambiguity after join
df1.alias("a").join(df2.alias("b"), F.col("a.id") == F.col("b.id")) \
   .select("a.*", F.col("b.extra_col"))

# Semi-join: keep left rows that have a match (no right columns added)
df1.join(df2, on="id", how="left_semi")

# Anti-join: keep left rows with NO match
df1.join(df2, on="id", how="left_anti")

# Broadcast join (forces small table to be broadcast)
from pyspark.sql.functions import broadcast
df_large.join(broadcast(df_small), on="key")

# Cross join (Cartesian - use carefully)
df1.crossJoin(df2)

# Range join (inequality - can be slow)
df1.join(df2, (df2.start <= df1.ts) & (df1.ts < df2.end))
```

-----

## 8. Window Functions

```python
from pyspark.sql.window import Window
from pyspark.sql import functions as F

# Define window spec
w = (
    Window
    .partitionBy("dept")
    .orderBy(F.col("salary").desc())
    .rowsBetween(Window.unboundedPreceding, Window.currentRow)
)

# Range-based frame (value-based, not row-based)
w_range = Window.partitionBy("dept").orderBy("date") \
               .rangeBetween(-7, 0)   # last 7 days (numeric/date diff)

# Ranking functions
F.row_number().over(w)
F.rank().over(w)
F.dense_rank().over(w)
F.percent_rank().over(w)
F.ntile(4).over(w)                     # quartile buckets
F.cume_dist().over(w)

# Analytic / navigation
F.lag(col("val"), 1).over(w)           # previous row
F.lead(col("val"), 1).over(w)          # next row
F.first(col("val"), ignorenulls=True).over(w)
F.last(col("val"),  ignorenulls=True).over(w)
F.nth_value(col("val"), 2).over(w)

# Aggregate over window
F.sum("salary").over(w)
F.avg("salary").over(w)
F.count("*").over(w)
F.max("salary").over(w)
F.min("salary").over(w)

# Running total pattern
w_run = Window.partitionBy("dept").orderBy("ts").rowsBetween(Window.unboundedPreceding, 0)
df.withColumn("running_total", F.sum("amount").over(w_run))

# Top-N per group pattern
w_top = Window.partitionBy("category").orderBy(F.col("score").desc())
df.withColumn("rank", F.row_number().over(w_top)).filter(F.col("rank") <= 3)
```

-----

## 9. Streaming (Structured Streaming)

```python
# Read stream from Kafka
stream_df = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "topic_name")
    .option("startingOffsets", "latest")
    .load()
)
# Kafka payload: key, value (binary), topic, partition, offset, timestamp, headers

# Deserialize JSON value
schema = T.StructType([T.StructField("id", T.LongType()), T.StructField("event", T.StringType())])
parsed = stream_df.select(F.from_json(F.col("value").cast("string"), schema).alias("data")).select("data.*")

# Read stream from Delta
stream_df = spark.readStream.format("delta").option("maxFilesPerTrigger", 100).table("my_delta_table")

# Read stream from Auto Loader (Databricks - highly recommended for cloud storage)
stream_df = (
    spark.readStream.format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/checkpoints/schema/")
    .load("/mnt/raw/landing/")
)

# Watermarking for late data
windowed = (
    parsed
    .withWatermark("event_time", "10 minutes")
    .groupBy(F.window("event_time", "5 minutes", "1 minute"), "user_id")
    .agg(F.count("*").alias("events"))
)

# Output modes: append | complete | update
query = (
    windowed.writeStream
    .outputMode("append")
    .format("delta")
    .option("checkpointLocation", "/mnt/checkpoints/my_query/")
    .trigger(processingTime="1 minute")         # micro-batch
    # .trigger(availableNow=True)               # run-once (preferred)
    # .trigger(once=True)                       # legacy run-once
    .table("catalog.schema.output_table")
    .start()
)

query.awaitTermination()
query.stop()
query.status
query.lastProgress
spark.streams.active                            # list active queries

# ForeachBatch (arbitrary write logic)
def process_batch(batch_df, batch_id):
    batch_df.write.format("delta").mode("append").save("/mnt/output/")

(stream_df.writeStream
 .foreachBatch(process_batch)
 .option("checkpointLocation", "/mnt/ckpt/")
 .start())
```

-----

## 10. Delta Lake Operations

```python
from delta.tables import DeltaTable

# Access Delta table
dt = DeltaTable.forPath(spark, "/mnt/delta/my_table/")
dt = DeltaTable.forName(spark, "catalog.schema.table")

# History & Time Travel
dt.history().show()
dt.history(10).show()                              # last 10 versions

# UPSERT (MERGE)
dt.alias("target").merge(
    updates_df.alias("source"),
    "target.id = source.id"
).whenMatchedUpdate(set={"name": "source.name", "ts": "source.ts"}) \
 .whenNotMatchedInsert(values={"id": "source.id", "name": "source.name", "ts": "source.ts"}) \
 .whenNotMatchedBySourceDelete() \
 .execute()

# Delete
dt.delete("event_date < '2023-01-01'")
dt.delete(F.col("active") == False)

# Update
dt.update("status = 'inactive'", {"updated_at": "current_timestamp()"})
dt.update(F.col("score") > 100, {"score": lit(100)})

# Restore to a prior version
dt.restoreToVersion(5)
dt.restoreToTimestamp("2024-06-01 00:00:00")

# Optimize (compaction)
dt.optimize().executeCompaction()
dt.optimize().where("date >= '2024-01-01'").executeCompaction()

# Z-Order (co-locate data for query predicates)
dt.optimize().executeZOrderBy("user_id", "date")

# Vacuum (remove old files)
spark.sql("VACUUM catalog.schema.table RETAIN 168 HOURS")  # 7 days min recommended
dt.vacuum(retentionHours=168)

# Schema evolution
spark.conf.set("spark.databricks.delta.schema.autoMerge.enabled", "true")
df.write.format("delta").option("mergeSchema", "true").mode("append").save("/path/")

# Change Data Feed
spark.conf.set("spark.databricks.delta.properties.defaults.enableChangeDataFeed", "true")
cdf = (spark.read.format("delta")
       .option("readChangeFeed", "true")
       .option("startingVersion", 10)
       .table("my_table"))
# cdf includes _change_type: insert | update_preimage | update_postimage | delete
```

-----

## 11. Writing & Output

```python
# Basic write
df.write.format("delta").mode("overwrite").save("/mnt/delta/path/")
df.write.format("delta").mode("append").saveAsTable("catalog.schema.table")
df.writeTo("catalog.schema.table").append()
df.writeTo("catalog.schema.table").overwrite(F.col("date") == "2024-01-01")

# Write modes: overwrite | append | ignore | error (default)

# Parquet
df.write.mode("overwrite").parquet("/mnt/output/")

# CSV
df.write.mode("overwrite").option("header","true").csv("/mnt/output/")

# Partitioned write
df.write.partitionBy("year","month").format("delta").mode("overwrite").save("/path/")

# Control output file count
df.coalesce(1).write.csv("/mnt/output/single_file/")   # narrow (no shuffle)
df.repartition(100).write.parquet("/mnt/output/")       # shuffle repartition

# Repartition by column for partition-pruning
df.repartition(F.col("country")).write.partitionBy("country").format("delta").save("/path/")

# Dynamic partition overwrite (only overwrite touched partitions)
spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")
df.write.mode("overwrite").format("delta").partitionBy("date").save("/path/")

# Create table DDL
spark.sql("""
  CREATE TABLE IF NOT EXISTS catalog.schema.table (
    id BIGINT NOT NULL,
    name STRING,
    ts TIMESTAMP
  )
  USING DELTA
  PARTITIONED BY (date DATE)
  LOCATION '/mnt/delta/table/'
  TBLPROPERTIES ('delta.autoOptimize.optimizeWrite' = 'true',
                 'delta.autoOptimize.autoCompact'   = 'true')
""")
```

-----

## 12. Schema & Type System

### Type Mapping

|PySpark Type                     |Python  |SQL Type     |
|---------------------------------|--------|-------------|
|`T.ByteType()`                   |int     |TINYINT      |
|`T.ShortType()`                  |int     |SMALLINT     |
|`T.IntegerType()`                |int     |INT          |
|`T.LongType()`                   |int     |BIGINT       |
|`T.FloatType()`                  |float   |FLOAT        |
|`T.DoubleType()`                 |float   |DOUBLE       |
|`T.DecimalType(p,s)`             |Decimal |DECIMAL(p,s) |
|`T.StringType()`                 |str     |STRING       |
|`T.BinaryType()`                 |bytes   |BINARY       |
|`T.BooleanType()`                |bool    |BOOLEAN      |
|`T.DateType()`                   |date    |DATE         |
|`T.TimestampType()`              |datetime|TIMESTAMP    |
|`T.TimestampNTZType()`           |datetime|TIMESTAMP_NTZ|
|`T.ArrayType(element, nullable)` |list    |ARRAY        |
|`T.MapType(key, value, nullable)`|dict    |MAP          |
|`T.StructType([fields...])`      |Row     |STRUCT       |
|`T.NullType()`                   |None    |VOID         |

### Schema Construction

```python
schema = T.StructType([
    T.StructField("id",      T.LongType(),   nullable=False),
    T.StructField("name",    T.StringType(), nullable=True),
    T.StructField("score",   T.DoubleType()),
    T.StructField("tags",    T.ArrayType(T.StringType())),
    T.StructField("meta",    T.MapType(T.StringType(), T.StringType())),
    T.StructField("address", T.StructType([
        T.StructField("city",  T.StringType()),
        T.StructField("zip",   T.StringType()),
    ])),
])

# Inspect schema
df.schema                  # StructType object
df.printSchema()           # pretty print
df.dtypes                  # list of (name, type_str) tuples
df.schema.simpleString()   # DDL string
df.schema.toDDL()
```

-----

## 13. UDFs & Pandas UDFs (Arrow)

### Python UDF (row-by-row — slower)

```python
from pyspark.sql.functions import udf

@udf(returnType=T.StringType())
def clean_name(s: str) -> str:
    return s.strip().title() if s else None

df.withColumn("name_clean", clean_name("name"))

# Register for SQL
spark.udf.register("clean_name", clean_name)
spark.sql("SELECT clean_name(name) FROM t")
```

### Pandas UDF — Scalar (vectorized, Arrow-optimized)

```python
from pyspark.sql.functions import pandas_udf
import pandas as pd

@pandas_udf(T.DoubleType())
def normalize(s: pd.Series) -> pd.Series:
    return (s - s.mean()) / s.std()

df.withColumn("score_norm", normalize("score"))
```

### Pandas UDF — Grouped Map (split-apply-combine)

```python
from pyspark.sql.functions import PandasUDFType

output_schema = T.StructType([
    T.StructField("dept",       T.StringType()),
    T.StructField("emp_id",     T.LongType()),
    T.StructField("salary_pct", T.DoubleType()),
])

@pandas_udf(output_schema, PandasUDFType.GROUPED_MAP)
def compute_pct(pdf: pd.DataFrame) -> pd.DataFrame:
    pdf["salary_pct"] = pdf["salary"] / pdf["salary"].sum()
    return pdf[["dept", "emp_id", "salary_pct"]]

df.groupBy("dept").apply(compute_pct)
```

### Pandas UDF — Grouped Aggregate (Spark 3.x)

```python
@pandas_udf(T.DoubleType())
def weighted_avg(values: pd.Series, weights: pd.Series) -> float:
    return (values * weights).sum() / weights.sum()

df.groupBy("dept").agg(weighted_avg("salary", "weight"))
```

### Arrow UDF (Spark 3.5+)

```python
@udf(returnType=T.LongType(), useArrow=True)
def fast_hash(s: str) -> int:
    return hash(s) & 0x7FFFFFFF
```

### UDF Performance Hierarchy

```
Built-in F.*  >>  Pandas UDF (Arrow)  >>  Python UDF (row-by-row)
   fastest              fast                    slowest
```

-----

## 14. SQL Integration

```python
# Register temp view
df.createOrReplaceTempView("my_view")
df.createOrReplaceGlobalTempView("global_view")   # global_temp.global_view

# Run SQL
result = spark.sql("SELECT * FROM my_view WHERE score > 90")

# Parameterized SQL (Spark 3.4+)
result = spark.sql("SELECT * FROM {tbl} WHERE score > {threshold}",
                   tbl=spark.table("my_view"), threshold=90)

# Run arbitrary DDL
spark.sql("CREATE SCHEMA IF NOT EXISTS catalog.my_schema")
spark.sql("DROP TABLE IF EXISTS catalog.schema.old_table")
spark.sql("DESCRIBE DETAIL catalog.schema.table")
spark.sql("SHOW TABLES IN catalog.schema")
spark.sql("SHOW COLUMNS IN catalog.schema.table")
spark.sql("ANALYZE TABLE catalog.schema.table COMPUTE STATISTICS FOR ALL COLUMNS")

# Convert SQL results back to DataFrame
result_df = spark.sql("SELECT * FROM range(100)")
```

-----

## 15. Optimisation & Performance

### Configuration Knobs

```python
# Adaptive Query Execution (AQE) — default on in DBR
spark.conf.set("spark.sql.adaptive.enabled",                    "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled",           "true")
spark.conf.set("spark.sql.adaptive.localShuffleReader.enabled", "true")

# Shuffle partitions (AQE will auto-tune if enabled)
spark.conf.set("spark.sql.shuffle.partitions", "200")

# Broadcast threshold (bytes)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", str(50 * 1024 * 1024))  # 50 MB

# Photon (DBR compiled execution engine)
spark.conf.set("spark.databricks.photon.enabled", "true")

# Predicate pushdown
spark.conf.set("spark.sql.parquet.filterPushdown", "true")
spark.conf.set("spark.databricks.delta.stats.collect", "true")

# Caching
df.cache()                      # lazy - materialises on first action
df.persist(StorageLevel.MEMORY_AND_DISK_SER)
df.unpersist()
spark.catalog.clearCache()

# Explain plan
df.explain()                    # physical plan
df.explain("extended")          # parsed, analyzed, optimised, physical
df.explain("codegen")           # generated code
df.explain("cost")              # plan + stats
df.explain("formatted")         # human-readable tree (preferred)
```

### Partition Management

```python
# Check current partitions
df.rdd.getNumPartitions()

# Repartition (shuffle) — use for write or downstream heavy ops
df.repartition(200)
df.repartition(200, "country")            # hash-partitioned by column

# Coalesce (no shuffle) — use to reduce small files before write
df.coalesce(10)

# Partition hints in SQL
spark.sql("SELECT /*+ REPARTITION(100, dept) */ * FROM employees")
spark.sql("SELECT /*+ COALESCE(1) */ * FROM small_result")
spark.sql("SELECT /*+ REBALANCE(dept) */ * FROM employees")    # AQE rebalance

# Join hints
spark.sql("SELECT /*+ BROADCAST(small) */ * FROM big JOIN small ON big.id = small.id")
spark.sql("SELECT /*+ MERGE(a, b) */ * FROM a JOIN b ON a.id = b.id")
spark.sql("SELECT /*+ SHUFFLE_HASH(a) */ * FROM a JOIN b ON a.id = b.id")
```

### Skew Handling

```python
# Salting pattern for skewed joins
salt_factor = 20
df_skewed = df_skewed.withColumn("salt", (F.rand() * salt_factor).cast("int"))
df_small   = df_small.withColumn("salt", F.explode(F.array([F.lit(i) for i in range(salt_factor)])))
df_skewed.join(df_small, on=["key", "salt"]).drop("salt")

# AQE skew join (automatic - preferred when AQE is enabled)
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", str(256*1024*1024))
```

### Caching Patterns

```python
# Cache only when reused multiple times
df_filtered = df.filter("active = true").cache()
df_filtered.count()                    # materialize

count_by_dept = df_filtered.groupBy("dept").count()
avg_by_dept   = df_filtered.groupBy("dept").agg(F.avg("salary"))
# both reuse cache

df_filtered.unpersist()                # free memory when done

# Delta IO cache (disk-level — automatic in DBR)
spark.conf.set("spark.databricks.io.cache.enabled", "true")
```

-----

## 16. Databricks-Specific APIs

### dbutils

```python
# File system
dbutils.fs.ls("/mnt/data/")
dbutils.fs.mkdirs("/mnt/data/new_dir/")
dbutils.fs.cp("/mnt/src/file", "/mnt/dst/file", recurse=False)
dbutils.fs.mv("/mnt/src/", "/mnt/dst/", recurse=True)
dbutils.fs.rm("/mnt/data/old/", recurse=True)
dbutils.fs.put("/mnt/data/file.txt", "content", overwrite=True)
dbutils.fs.head("/mnt/data/file.txt", maxBytes=65536)
dbutils.fs.mount("s3://bucket/", "/mnt/mybucket", extra_configs={...})
dbutils.fs.unmount("/mnt/mybucket")
dbutils.fs.refreshMounts()

# Secrets
secret = dbutils.secrets.get(scope="my-scope", key="api-key")
dbutils.secrets.list("my-scope")

# Widgets (notebook parameters)
dbutils.widgets.text("env", "dev", "Environment")
dbutils.widgets.dropdown("mode", "full", ["full", "incremental"])
dbutils.widgets.combobox("table", "", ["t1", "t2"])
dbutils.widgets.multiselect("cols", "a", ["a","b","c"])
env = dbutils.widgets.get("env")
dbutils.widgets.removeAll()

# Notebook utilities
result = dbutils.notebook.run("/path/to/notebook", timeout_seconds=3600,
                              arguments={"env": "prod"})
dbutils.notebook.exit("SUCCESS")

# Jobs / Task Values
dbutils.jobs.taskValues.set(key="count", value=42)
count = dbutils.jobs.taskValues.get(taskKey="upstream_task", key="count")
```

### Display & Visualization

```python
display(df)                        # Rich table with pagination
dbutils.data.summarize(df)         # Statistical summary / profiler
displayHTML("<h2>Hello Databricks</h2>")
```

### Broadcast Variables & Accumulators

```python
# Broadcast variable (distribute read-only lookup to all workers)
bc_lookup = sc.broadcast({"US": "United States", "DE": "Germany"})
lookup_udf = udf(lambda code: bc_lookup.value.get(code), T.StringType())
df.withColumn("country_name", lookup_udf("country_code"))

# Accumulator (aggregate counters from workers)
acc = sc.accumulator(0)
# inside map: acc += 1
```

-----

## 17. Unity Catalog & Governance

```python
# 3-part naming: catalog.schema.table
spark.table("main.sales.transactions")
spark.sql("SELECT * FROM my_catalog.my_schema.my_table")

# Manage catalog objects
spark.sql("CREATE CATALOG IF NOT EXISTS my_catalog")
spark.sql("CREATE SCHEMA IF NOT EXISTS my_catalog.my_schema")
spark.sql("USE CATALOG my_catalog")
spark.sql("USE SCHEMA my_schema")

# Permissions
spark.sql("GRANT SELECT ON TABLE my_catalog.my_schema.sales TO `analysts@company.com`")
spark.sql("GRANT MODIFY ON SCHEMA my_catalog.my_schema TO `data_engineers`")
spark.sql("REVOKE SELECT ON TABLE my_catalog.my_schema.sales FROM `user@company.com`")
spark.sql("SHOW GRANTS ON TABLE my_catalog.my_schema.sales")

# Row-level security (row filter)
spark.sql("""
  CREATE OR REPLACE FUNCTION my_catalog.my_schema.row_filter(dept STRING)
  RETURNS BOOLEAN
  RETURN IS_MEMBER('dept_' || dept)
""")
spark.sql("ALTER TABLE t SET ROW FILTER my_catalog.my_schema.row_filter ON (dept)")

# Column masking
spark.sql("""
  CREATE OR REPLACE FUNCTION my_catalog.my_schema.mask_ssn(ssn STRING)
  RETURNS STRING
  RETURN IF(IS_MEMBER('pii_viewers'), ssn, '***-**-****')
""")
spark.sql("ALTER TABLE t ALTER COLUMN ssn SET MASK my_catalog.my_schema.mask_ssn")

# Tags & comments
spark.sql("ALTER TABLE t SET TAGS ('pii' = 'true', 'domain' = 'finance')")
spark.sql("ALTER TABLE t ALTER COLUMN email SET TAGS ('pii' = 'email')")
spark.sql("COMMENT ON TABLE t IS 'Core customer table - updated nightly'")
```

-----

## 18. Design Patterns & Best Practices

### Pattern 1 — Medallion Architecture

```python
# BRONZE: raw ingestion via Auto Loader
bronze = (
    spark.readStream.format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", f"{ckpt}/bronze_schema")
    .load(raw_path)
    .withColumn("_ingest_ts",   F.current_timestamp())
    .withColumn("_source_file", F.input_file_name())
)
bronze.writeStream.format("delta").option("checkpointLocation", f"{ckpt}/bronze") \
      .table("catalog.bronze.events").start()

# SILVER: cleanse & conform
silver = (
    spark.readStream.format("delta").table("catalog.bronze.events")
    .filter(F.col("event_type").isNotNull())
    .withColumn("ts", F.to_timestamp("event_time"))
    .dropDuplicates(["event_id"])
)
silver.writeStream.format("delta").option("checkpointLocation", f"{ckpt}/silver") \
      .table("catalog.silver.events").start()

# GOLD: business aggregation (batch)
(
    spark.table("catalog.silver.events")
    .groupBy(F.date_trunc("day", "ts").alias("day"), "user_id")
    .agg(F.count("*").alias("events"))
    .write.format("delta").mode("overwrite").table("catalog.gold.daily_events")
)
```

### Pattern 2 — Incremental / CDC Load

```python
def incremental_load(source_table: str, target_table: str, watermark_col: str):
    last_ts = spark.sql(f"SELECT MAX({watermark_col}) FROM {target_table}").collect()[0][0]
    new_data = spark.table(source_table).filter(F.col(watermark_col) > last_ts)

    dt = DeltaTable.forName(spark, target_table)
    dt.alias("t").merge(
        new_data.alias("s"), "t.id = s.id"
    ).whenMatchedUpdateAll() \
     .whenNotMatchedInsertAll() \
     .execute()
```

### Pattern 3 — SCD Type 2

```python
def scd2_merge(source_df, target_table: str, key_cols: list, track_cols: list):
    dt = DeltaTable.forName(spark, target_table)
    cols_changed = " OR ".join([f"t.{c} <> s.{c}" for c in track_cols])
    join_cond    = " AND ".join([f"t.{k} = s.{k}" for k in key_cols]) + " AND t.current = true"

    updates = source_df.withColumn("current",         F.lit(True)) \
                       .withColumn("effective_start", F.current_date()) \
                       .withColumn("effective_end",   F.lit(None).cast("date"))

    (dt.alias("t").merge(updates.alias("s"), join_cond)
     .whenMatchedUpdate(
         condition=cols_changed,
         set={"current": "false", "effective_end": "current_date() - INTERVAL 1 DAY"})
     .whenNotMatchedInsertAll()
     .execute())
```

### Pattern 4 — Dynamic Partition Write

```python
spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

(
    new_data
    .write.format("delta")
    .mode("overwrite")
    .partitionBy("date", "region")
    .option("replaceWhere", "date >= '2024-01-01'")  # explicit predicate
    .table("my_table")
)
```

### Pattern 5 — Composable Transform Pipeline

```python
from pyspark.sql import DataFrame
from typing import Callable, List

Transform = Callable[[DataFrame], DataFrame]

def pipeline(*transforms: Transform) -> Transform:
    def apply(df: DataFrame) -> DataFrame:
        for t in transforms:
            df = t(df)
        return df
    return apply

def add_audit_cols(df: DataFrame) -> DataFrame:
    return df.withColumn("_ts", F.current_timestamp())

def drop_nulls(cols: List[str]) -> Transform:
    return lambda df: df.na.drop(subset=cols)

def standardise_strings(*cols: str) -> Transform:
    def apply(df: DataFrame) -> DataFrame:
        for c in cols:
            df = df.withColumn(c, F.trim(F.lower(F.col(c))))
        return df
    return apply

process = pipeline(
    drop_nulls(["id", "name"]),
    standardise_strings("name", "email"),
    add_audit_cols,
)
result = process(raw_df)
```

### Pattern 6 — Dead Letter Queue (Error Handling)

```python
def safe_parse(df: DataFrame, schema) -> tuple:
    parsed = df.withColumn("_parsed", F.from_json("payload", schema))
    good   = parsed.filter(F.col("_parsed").isNotNull()).select("_parsed.*")
    bad    = parsed.filter(F.col("_parsed").isNull()) \
                   .withColumn("_error_ts", F.current_timestamp())
    return good, bad

good_df, dlq_df = safe_parse(raw_df, event_schema)
dlq_df.write.format("delta").mode("append").table("catalog.errors.dlq")
```

### Best Practices Checklist

|Category                |Practice                                                                                    |
|------------------------|--------------------------------------------------------------------------------------------|
|**Schema**              |Always provide explicit schema on read; avoid `inferSchema` in production                   |
|**Partitioning**        |Partition by low-cardinality date/region columns; aim for 128 MB–1 GB files                 |
|**Delta**               |Enable `optimizeWrite` + `autoCompact`; run `OPTIMIZE` + `ZORDER` on query predicate columns|
|**Joins**               |Broadcast tables < 50 MB; filter before joining; avoid Cartesian joins                      |
|**UDFs**                |Prefer built-in `F.*` → Pandas UDF → Python UDF (slowest)                                   |
|**Caching**             |Cache only reused DataFrames; always `unpersist()` when done                                |
|**Shuffle**             |Set `shuffle.partitions` = 2–3x cores; let AQE auto-tune                                    |
|**Streaming**           |Use Auto Loader + Delta; always set checkpoint; prefer `availableNow` trigger               |
|**Unity Catalog**       |Use 3-part names everywhere; never use `hive_metastore` in new code                         |
|**Secrets**             |Always use `dbutils.secrets`; never hardcode credentials                                    |
|**Idempotency**         |Design all writes to be re-runnable (merge over insert, dynamic partitions)                 |
|**Testing**             |Use `chispa` or `pytest` with small DataFrames; mock `spark` in unit tests                  |
|**Column reuse**        |Use `withColumns({})` instead of chaining many `withColumn()` calls                         |
|**Predicate pushdown**  |Filter as early as possible; use partition columns in predicates                            |
|**Wide transformations**|Minimise shuffles; combine multiple aggregations in a single `agg()` call                   |

-----

## 19. Master Method Reference Table

### DataFrame Methods

|Method                                   |Returns               |Description                       |
|-----------------------------------------|----------------------|----------------------------------|
|`agg(*exprs)`                            |DataFrame             |Aggregate without groupBy         |
|`alias(name)`                            |DataFrame             |Name the DataFrame for joins      |
|`cache()`                                |DataFrame             |Persist in memory (lazy)          |
|`checkpoint(eager=True)`                 |DataFrame             |Materialise and truncate lineage  |
|`coalesce(n)`                            |DataFrame             |Reduce partitions (no shuffle)    |
|`collect()`                              |list[Row]             |Bring all rows to driver          |
|`columns`                                |list[str]             |Column name list                  |
|`count()`                                |int                   |Row count (action)                |
|`createOrReplaceTempView(name)`          |None                  |Register as temp SQL view         |
|`createOrReplaceGlobalTempView(name)`    |None                  |Register as global temp view      |
|`crossJoin(other)`                       |DataFrame             |Cartesian join                    |
|`describe(*cols)`                        |DataFrame             |Summary statistics                |
|`distinct()`                             |DataFrame             |Remove duplicate rows             |
|`drop(*cols)`                            |DataFrame             |Remove columns                    |
|`dropDuplicates(subset)`                 |DataFrame             |Deduplicate by subset             |
|`dropna(how, thresh, subset)`            |DataFrame             |Drop rows with NULLs              |
|`dtypes`                                 |list[tuple]           |(name, type) pairs                |
|`explain(mode)`                          |None                  |Print execution plan              |
|`fillna(value, subset)`                  |DataFrame             |Fill NULLs                        |
|`filter(cond)` / `where(cond)`           |DataFrame             |Row filter                        |
|`first()`                                |Row                   |First row (action)                |
|`foreach(func)`                          |None                  |Apply func to each row (action)   |
|`foreachPartition(func)`                 |None                  |Apply func per partition (action) |
|`groupBy(*cols)`                         |GroupedData           |Begin aggregation                 |
|`head(n)`                                |list[Row]             |First n rows (action)             |
|`hint(name, *params)`                    |DataFrame             |Add optimizer hint                |
|`inputFiles()`                           |list[str]             |Source files                      |
|`intersect(other)`                       |DataFrame             |Set intersection                  |
|`intersectAll(other)`                    |DataFrame             |Intersection preserving duplicates|
|`isEmpty()`                              |bool                  |Check if empty                    |
|`join(other, on, how)`                   |DataFrame             |Join                              |
|`limit(n)`                               |DataFrame             |Take first n rows                 |
|`localCheckpoint()`                      |DataFrame             |Checkpoint without HDFS           |
|`na`                                     |DataFrameNaFunctions  |Null-handling methods             |
|`observe(observation, *exprs)`           |DataFrame             |Collect metrics inline            |
|`orderBy(*cols)` / `sort(*cols)`         |DataFrame             |Sort globally                     |
|`persist(storageLevel)`                  |DataFrame             |Persist with storage level        |
|`printSchema()`                          |None                  |Print schema tree                 |
|`randomSplit(weights, seed)`             |list[DataFrame]       |Split randomly                    |
|`rdd`                                    |RDD                   |Low-level RDD access              |
|`repartition(n, *cols)`                  |DataFrame             |Shuffle to n partitions           |
|`repartitionByRange(n, *cols)`           |DataFrame             |Range partition                   |
|`replace(to_replace, value, subset)`     |DataFrame             |Replace values                    |
|`rollup(*cols)`                          |GroupedData           |Rollup aggregation                |
|`sample(fraction, seed)`                 |DataFrame             |Random sample                     |
|`sampleBy(col, fractions, seed)`         |DataFrame             |Stratified sample                 |
|`schema`                                 |StructType            |Schema object                     |
|`select(*cols)`                          |DataFrame             |Project columns                   |
|`selectExpr(*exprs)`                     |DataFrame             |Project with SQL expressions      |
|`show(n, truncate, vertical)`            |None                  |Print rows (action)               |
|`sortWithinPartitions(*cols)`            |DataFrame             |Sort per partition only           |
|`stat`                                   |DataFrameStatFunctions|Statistics methods                |
|`subtract(other)`                        |DataFrame             |Set difference                    |
|`summary(*stats)`                        |DataFrame             |Extended statistics               |
|`tail(n)`                                |list[Row]             |Last n rows (action)              |
|`take(n)`                                |list[Row]             |First n rows as list (action)     |
|`toDF(*cols)`                            |DataFrame             |Rename all columns                |
|`toJSON(use_unicode)`                    |RDD                   |Rows as JSON strings              |
|`toLocalIterator()`                      |Iterator              |Row-by-row iterator               |
|`toPandas()`                             |pd.DataFrame          |Convert to Pandas                 |
|`transform(func, *args)`                 |DataFrame             |Apply a function to DataFrame     |
|`union(other)`                           |DataFrame             |Union (by position)               |
|`unionAll(other)`                        |DataFrame             |Alias for union                   |
|`unionByName(other, allowMissingColumns)`|DataFrame             |Union by column name              |
|`unpersist(blocking)`                    |DataFrame             |Release cached data               |
|`unpivot(ids, values, var, val)`         |DataFrame             |Wide-to-long (Spark 3.4+)         |
|`where(cond)`                            |DataFrame             |Alias for filter                  |
|`withColumn(name, col)`                  |DataFrame             |Add or replace column             |
|`withColumnRenamed(old, new)`            |DataFrame             |Rename column                     |
|`withColumns(colsMap)`                   |DataFrame             |Add multiple columns at once      |
|`withMetadata(col, metadata)`            |DataFrame             |Attach metadata to column         |
|`withWatermark(col, delay)`              |DataFrame             |Streaming watermark               |
|`write`                                  |DataFrameWriter       |Access write API                  |
|`writeStream`                            |DataStreamWriter      |Access streaming write API        |
|`writeTo(table)`                         |DataFrameWriterV2     |V2 write API                      |

### GroupedData Methods

|Method                       |Description               |
|-----------------------------|--------------------------|
|`agg(*exprs)`                |Aggregate with expressions|
|`apply(func)`                |Pandas UDF grouped map    |
|`applyInPandas(func, schema)`|Grouped map (Spark 3.3+)  |
|`avg(*cols)`                 |Mean                      |
|`count()`                    |Row count                 |
|`max(*cols)`                 |Maximum                   |
|`mean(*cols)`                |Mean (alias)              |
|`min(*cols)`                 |Minimum                   |
|`pivot(col, values)`         |Pivot on column           |
|`sum(*cols)`                 |Sum                       |

### DataFrameWriter Methods

|Method                   |Description                        |
|-------------------------|-----------------------------------|
|`bucketBy(n, col, *cols)`|Bucket output                      |
|`csv(path)`              |Write CSV                          |
|`format(source)`         |Set format                         |
|`insertInto(table)`      |Insert into existing table         |
|`json(path)`             |Write JSON                         |
|`mode(saveMode)`         |overwrite / append / ignore / error|
|`option(key, value)`     |Set write option                   |
|`options(**kwargs)`      |Set multiple options               |
|`orc(path)`              |Write ORC                          |
|`parquet(path)`          |Write Parquet                      |
|`partitionBy(*cols)`     |Partition output                   |
|`save(path)`             |Save to path                       |
|`saveAsTable(name)`      |Save as catalog table              |
|`sortBy(*cols)`          |Sort within buckets                |
|`text(path)`             |Write text                         |

### DeltaTable Methods

|Method                     |Description                  |
|---------------------------|-----------------------------|
|`alias(name)`              |Alias for merge operations   |
|`delete(condition)`        |Delete matching rows         |
|`detail()`                 |Table metadata DataFrame     |
|`forName(spark, name)`     |Access by table name (static)|
|`forPath(spark, path)`     |Access by path (static)      |
|`generate(mode)`           |Generate manifest etc.       |
|`history(limit)`           |Version history DataFrame    |
|`isDeltaTable(spark, path)`|Check if Delta (static)      |
|`merge(source, condition)` |Begin merge/upsert builder   |
|`optimize()`               |Begin optimize builder       |
|`restoreToTimestamp(ts)`   |Restore to timestamp         |
|`restoreToVersion(v)`      |Restore to version           |
|`toDF()`                   |Current snapshot as DataFrame|
|`update(condition, set)`   |Update matching rows         |
|`vacuum(retentionHours)`   |Remove old files             |

-----

*Reference covers Spark 3.4–3.5, Databricks Runtime 13–15, Delta Lake 2.x, and the PySpark Python API.  
Always check [docs.databricks.com](https://docs.databricks.com) and [spark.apache.org/docs](https://spark.apache.org/docs/latest/api/python/) for the most current signatures and new additions.*