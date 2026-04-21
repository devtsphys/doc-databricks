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





