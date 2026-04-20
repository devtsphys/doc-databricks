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

```



