Configuration driven feature engineering

```yaml
feature_groups:
  - name: "customer_activity_features"
    description: "Features basierend auf Kundeninteraktionen im Info-Mart"
    primary_keys: ["customer_id"]
    timestamp_column: "event_time"
    target_table: "ml_catalog.feature_store.customer_metrics"
    source_query: |
      SELECT 
        customer_id,
        current_timestamp() as event_time,
        count(order_id) as total_orders_90d,
        sum(total_amount) as total_revenue_90d
      FROM 
        info_mart.sales_data
      WHERE 
        order_date >= date_sub(current_date(), 90)
      GROUP BY 
        customer_id

  - name: "product_inventory_features"
    primary_keys: ["product_id"]
    target_table: "ml_catalog.feature_store.product_stats"
    source_query: "SELECT product_id, stock_level, last_restock_date FROM info_mart.inventory"
```

Using the feature generation
```python
import yaml
from databricks.feature_engineering import FeatureEngineeringClient

# 1. YAML laden
with open("features.yml", "r") as f:
    config = yaml.safe_load(f)

fe = FeatureEngineeringClient()

# 2. Über Feature-Gruppen iterieren und Tabellen aktualisieren
for group in config['feature_groups']:
    print(f"Processing: {group['name']}")
    
    # SQL aus YAML ausführen
    df = spark.sql(group['source_query'])
    
    # In Feature Store schreiben (Upsert/Create)
    fe.write_table(
        name=group['target_table'],
        df=df,
        mode="merge"
    )

```python
Feature validation
```python
import yaml
import sys

def validate_feature_config(config_path):
    with open(config_path, "r") as f:
        config = yaml.safe_load(f)

    errors = []

    for group in config.get('feature_groups', []):
        group_name = group.get('name', 'Unknown')
        print(f"--- Validating Group: {group_name} ---")

        # 1. Pflichtfelder prüfen
        required_fields = ['name', 'primary_keys', 'target_table', 'source_query']
        for field in required_fields:
            if field not in group:
                errors.append(f"[{group_name}] Missing field: {field}")

        # 2. SQL-Syntax Prüfung (Spark EXPLAIN)
        query = group.get('source_query')
        if query:
            try:
                # EXPLAIN validiert die Syntax und Tabellenexistenz, ohne Daten zu lesen
                plan = spark.sql(f"EXPLAIN {query}")
                
                # 3. Schema-Abgleich: Sind PKs in der Query enthalten?
                df_schema = spark.sql(query).limit(0) # Nur Header laden
                columns = df_schema.columns
                for pk in group.get('primary_keys', []):
                    if pk not in columns:
                        errors.append(f"[{group_name}] Primary Key '{pk}' not found in SQL columns: {columns}")
                
            except Exception as e:
                errors.append(f"[{group_name}] SQL Syntax Error: {str(e)}")

    if errors:
        print("\n❌ Validation Failed:")
        for err in errors:
            print(f"  - {err}")
        sys.exit(1) # Job/Pipeline abbrechen
    else:
        print("\n✅ All features validated successfully.")

# Aufruf im Databricks Notebook
validate_feature_config("features.yml")
```

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.catalog import MonitorTimeSeries, MonitorConfig

w = WorkspaceClient()

# Monitor für die neue Feature-Tabelle erstellen
w.monitors.create(
    full_table_name="main.feature_store.customer_behavior_metrics",
    assets_dir="/Shared/monitoring/customer_metrics",
    config=MonitorConfig(
        time_series=MonitorTimeSeries(
            timestamp_col="event_time", 
            granularities=["1 day"] # Statistiken pro Tag berechnen
        )
    )
)
```

```python
def validate_data_quality(df, expectations):
    """
    Prüft den Dataframe gegen die in der YAML definierten Regeln.
    """
    for check in expectations:
        col = check['column']
        cond = check['condition']
        action = check['action']
        
        # Zähle Zeilen, die die Bedingung NICHT erfüllen
        invalid_count = df.filter(f"NOT ({col} {cond})").count()
        
        if invalid_count > 0:
            msg = f"❌ DQ Alert: {invalid_count} Zeilen verletzen '{col} {cond}'"
            if action == "FAIL":
                raise ValueError(msg) # Job bricht sofort ab
            else:
                print(f"⚠️ {msg}") # Nur Warnung im Log
    
    print("✅ Datenqualitätsprüfung bestanden.")

# Integration im Haupt-Skript:
# ... nach df_features = spark.sql(query)
validate_data_quality(df_features, group.get('expectations', []))
# ... vor fe.write_table(...)
```

```
feature_groups:
  - name: "customer_behavior_metrics"
    primary_keys: ["customer_id"]
    # ... (Rest wie vorher)
    expectations:
      - { column: "customer_id", condition: "IS NOT NULL", action: "FAIL" }
      - { column: "spend_sum_1m", condition: ">= 0", action: "WARN" }
      - { column: "event_time", condition: "> '2024-01-01'", action: "FAIL" }
```

```yaml
resources:
  jobs:
    daily_feature_generation:
      name: "Daily_Feature_Store_Refresh"
      tasks:
        - task_key: "compute_info_mart_features"
          job_cluster_key: "shared_cluster"
          spark_python_task:
            python_file: "../src/run_feature_engineering.py"
          # Optional: Übergib die Umgebung (Dev/Prod) als Parameter
          parameters: ["--env", "${bundle.target}"]

      job_clusters:
        - job_cluster_key: "shared_cluster"
          new_cluster:
            spark_version: "14.3.x-scala2.12"
            node_type_id: "Standard_DS3_v2"
            num_workers: 2
```

```python
import yaml
from databricks.feature_engineering import FeatureEngineeringClient
from pyspark.sql import SparkSession

def run_job():
    spark = SparkSession.builder.getOrCreate()
    fe = FeatureEngineeringClient()

    # 1. YAML Konfiguration laden
    with open("features.yml", "r") as f:
        config = yaml.safe_load(f)

    for group in config['feature_groups']:
        print(f"🚀 Starte Feature-Erstellung für: {group['name']}")
        
        # 2. Dynamisches SQL generieren (Code aus der vorherigen Antwort)
        query = generate_feature_sql(group) # Deine SQL-Builder Funktion
        df_features = spark.sql(query)

        target_table = f"main.feature_store.{group['name']}"

        # 3. Feature Tabelle erstellen oder aktualisieren
        # Falls die Tabelle nicht existiert, wird sie mit PKs angelegt
        try:
            fe.get_table(target_table)
            print(f"Table {target_table} exists. Performing UPSERT (merge)...")
            fe.write_table(
                name=target_table,
                df=df_features,
                mode="merge" # Wichtig für tägliche Updates ohne Duplikate
            )
        except:
            print(f"Table {target_table} not found. Creating new...")
            fe.create_table(
                name=target_table,
                primary_keys=group['primary_keys'],
                timeseries_columns=["event_time"], # Für Point-in-Time Joins
                df=df_features,
                description=f"Automatisierte Features für {group['name']}"
            )

if __name__ == "__main__":
    run_job()


def generate_feature_sql(group_cfg):
    select_clauses = []
    
    for metric in group_cfg['metrics']:
        col_name = metric['col']
        alias = metric['alias']
        
        for func in metric['functions']:
            for window in group_cfg['windows']:
                # Erzeugt Namen wie: spend_sum_3m, orders_count_1m
                feature_name = f"{alias}_{func.lower()}_{window['suffix']}"
                
                # SQL Fragment mit bedingter Aggregation
                clause = f"""
                {func}(CASE 
                  WHEN {group_cfg['timestamp_col']} >= DATE_SUB(current_date(), {window['days']}) 
                  THEN {col_name} 
                END) AS {feature_name}
                """
                select_clauses.append(clause.strip())

    # Finale Query zusammenbauen
    pks = ", ".join(group_cfg['primary_keys'])
    query = f"""
    SELECT 
      {pks},
      current_timestamp() as event_time,
      {", ".join(select_clauses)}
    FROM {group_cfg['source_table']}
    GROUP BY {pks}
    """
    return query

# Testlauf
# print(generate_feature_sql(config['feature_groups'][0]))
```

```yaml
feature_groups:
  - name: "customer_behavior_metrics"
    source_table: "info_mart.orders"
    primary_keys: ["customer_id"]
    timestamp_col: "order_date"
    # Definition der Metriken
    metrics:
      - { col: "amount", functions: ["SUM", "AVG", "MAX"], alias: "spend" }
      - { col: "order_id", functions: ["COUNT"], alias: "orders" }
    # Definition der Zeiträume
    windows:
      - { suffix: "1m", days: 30 }
      - { suffix: "3m", days: 90 }
      - { suffix: "6m", days: 180 }

# resources/jobs.yml
resources:
  jobs:
    feature_pipeline:
      name: "Feature Engineering with Validation"
      tasks:
        - task_key: "validate_config"
          notebook_task:
            notebook_path: "./scripts/validate_features.py"
        
        - task_key: "run_feature_extraction"
          depends_on:
            - task_key: "validate_config" # Startet nur, wenn Validierung OK
          notebook_task:
            notebook_path: "./scripts/run_extraction.py"
```

```python
import numpy as np

def filter_redundant_features(df, correlation_threshold=0.98):
    """
    Entfernt Features mit Varianz 0 oder extrem hoher Korrelation.
    """
    # 1. Entferne Features ohne Varianz (nur ein Wert vorhanden)
    # (In Spark über describe oder summary prüfbar, hier als Logik-Konzept)
    
    # 2. Korrelationsmatrix für numerische Features (Stichprobe ziehen für Performance)
    sample_df = df.sample(0.1).toPandas()
    corr_matrix = sample_df.corr().abs()
    
    upper = corr_matrix.where(np.triu(np.ones(corr_matrix.shape), k=1).astype(bool))
    to_drop = [column for column in upper.columns if any(upper[column] > correlation_threshold)]
    
    print(f"Entferne {len(to_drop)} redundante Features: {to_drop}")
    return df.drop(*to_drop)
```

Kategorie	Beschreibung	Beispiel	Empfohlene Aggregationen
Zustands-Flags	Beschreiben, was ein Kunde ist. Ändern sich selten.	is_business, is_active, has_newsletter	AVG (Anteil), MAX (Präsenz), LAST (Aktuell)
Ereignis-Flags	Beschreiben, ob etwas passiert ist (Trigger).	had_complaint, did_login, payment_failed	SUM (Häufigkeit), MAX (Vorkommen), MIN (Lücken)
Trend-Flags	Zeigen Richtungswechsel an.	is_churn_risk, is_premium_candidate	STDDEV (Instabilität), FIRST vs LAST (Veränderung)

```python
from databricks.feature_engineering import FeatureEngineeringClient
from pyspark.sql import functions as F

class FeatureRunner:
    def __init__(self, spark_session, yaml_config):
        self.spark = spark_session
        self.config = yaml_config
        self.fe = FeatureEngineeringClient() # UC Feature Engineering Client

    def run_and_register(self, df_base, catalog, schema):
        # 1. Basis-Features berechnen (wie bisher)
        df_final = self._calculate_all_features(df_base)
        
        table_name = f"{catalog}.{schema}.customer_features_gold"
        
        # 2. Delta Tabelle im Unity Catalog erstellen/überschreiben
        # WICHTIG: Primary Key ist für Feature-Tabellen im UC zwingend erforderlich
        df_final.write.mode("overwrite").option("overwriteSchema", "true").saveAsTable(table_name)
        
        # 3. Tabelle als Feature-Tabelle registrieren (falls noch nicht geschehen)
        # Dies verknüpft die Tabelle offiziell mit dem Feature Store
        try:
            self.fe.get_table(table_name)
            print(f"Tabelle {table_name} ist bereits registriert.")
        except:
            print(f"Registriere {table_name} im Feature Store...")
            self.fe.register_table(
                delta_table=table_name,
                primary_keys=["customer_hk", "snapshot_date"], # Deine PKs aus dem Data Vault
                description="Zentrale Feature-Tabelle aus Info Mart Snapshots"
            )
            
        return df_final
```

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

class FeatureRunner:
    def __init__(self, spark_session, yaml_config):
        self.spark = spark_session
        self.config = yaml_config
        self.seconds_per_day = 86400

    def run_and_save(self, df_base, target_catalog, target_schema):
        # Wir speichern die DataFrames in einem Dictionary { '30d': df, '90d': df, ... }
        # plus einen für 'static' Features
        window_outputs = {}
        
        # 1. Statische Features (immer in allen Tabellen dabei)
        static_features = [f for f in self.config['features'] if f['type'] == "sql_expr"]
        df_static = df_base
        for f in static_features:
            df_static = df_static.withColumn(f['name'], F.expr(f['expression']))
        
        # 2. Windowed Features nach Zeitfenstern gruppieren
        windowed_configs = [f for f in self.config['features'] if f['type'] == "windowed_agg"]
        
        # Sammle alle vorkommenden Zeitfenster
        all_windows = set()
        for f in windowed_configs:
            all_windows.update(f['windows'])
            
        for days in all_windows:
            df_window = df_static # Basis inkl. statischer Features
            
            for f in windowed_configs:
                if days in f['windows']:
                    source_col = f['source_column']
                    window_spec = Window.partitionBy("customer_hk") \
                                        .orderBy(F.col("snapshot_date").cast("long")) \
                                        .rangeBetween(-days * self.seconds_per_day, 0)
                    
                    for method in f['methods']:
                        col_name = f"{f['name']}_{method}_{days}d"
                        # Dynamischer Aufruf der Spark-Funktion
                        agg_func = getattr(F, method.lower())
                        df_window = df_window.withColumn(col_name, agg_func(source_col).over(window_spec))
            
            # Speichern der Tabelle pro Zeitfenster
            table_name = f"{target_catalog}.{target_schema}.customer_features_{days}d"
            print(f"Speichere Tabelle: {table_name}")
            df_window.write.mode("overwrite").saveAsTable(table_name)
            
            window_outputs[f"{days}d"] = df_window
            
        return window_outputs

# Anwendung
# runner = FeatureRunner(spark, config)
# runner.run_and_save(df_mart, "main", "gold_features")
```






