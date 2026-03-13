# OpenTofu × Databricks — Deployment Reference Card

> **Scope:** OpenTofu (Terraform-compatible OSS) with the `databricks` provider — focusing on access rights, privileges, groups, and securable objects (Unity Catalog).  
> **Provider:** `databricks/databricks` — tested against `≥ 1.38.0`

-----

## Table of Contents

1. [Provider & Backend Setup](#1-provider--backend-setup)
1. [Core OpenTofu CLI Reference](#2-core-opentofu-cli-reference)
1. [Databricks Provider Resources — Full Table](#3-databricks-provider-resources--full-table)
1. [Identity & Group Management](#4-identity--group-management)
1. [Unity Catalog Securable Objects](#5-unity-catalog-securable-objects)
1. [Grants & Privileges Reference](#6-grants--privileges-reference)
1. [Row & Column Security](#7-row--column-security)
1. [Compute Privileges](#8-compute-privileges)
1. [MLflow & AI Asset Permissions](#9-mlflow--ai-asset-permissions)
1. [Design Patterns & Best Practices](#10-design-patterns--best-practices)
1. [SCIM / Entra ID Integration](#11-scim--entra-id-integration)
1. [Multi-Workspace / Multi-Environment Patterns](#12-multi-workspace--multi-environment-patterns)
1. [Drift Detection & Remediation](#13-drift-detection--remediation)
1. [Common Mistakes & Gotchas](#14-common-mistakes--gotchas)
1. [End-to-End Deployment in Practice](#15-end-to-end-deployment-in-practice)

-----

## 1. Provider & Backend Setup

### Minimum provider block

```hcl
terraform {
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.38"
    }
  }
  required_version = ">= 1.6"   # OpenTofu 1.6+ for full feature parity
}

provider "databricks" {
  host          = var.databricks_host          # e.g. https://adb-<id>.18.azuredatabricks.net
  azure_use_msi = true                         # Managed Identity (recommended for CI/CD)
  # OR: azure_client_id / azure_client_secret / azure_tenant_id for SP auth
}
```

### Account-level provider (Unity Catalog admin operations)

```hcl
provider "databricks" {
  alias      = "accounts"
  host       = "https://accounts.azuredatabricks.net"
  account_id = var.databricks_account_id
  azure_use_msi = true
}
```

### Remote state backend (Azure Blob)

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "satfstate"
    container_name       = "tofu-state"
    key                  = "databricks/prod.tfstate"
    use_azuread_auth     = true   # use Entra ID rather than storage key
  }
}
```

### Workspaces for environment isolation

```hcl
# tofu workspace new dev
# tofu workspace new staging
# tofu workspace new prod

locals {
  env = terraform.workspace   # "dev" | "staging" | "prod"
  config = {
    dev     = { host = "https://adb-dev.azuredatabricks.net",  sku = "standard" }
    staging = { host = "https://adb-stg.azuredatabricks.net",  sku = "premium"  }
    prod    = { host = "https://adb-prd.azuredatabricks.net",  sku = "premium"  }
  }
}
```

-----

## 2. Core OpenTofu CLI Reference

|Command                             |Description                     |
|------------------------------------|--------------------------------|
|`tofu init`                         |Initialize providers and backend|
|`tofu init -upgrade`                |Upgrade provider versions       |
|`tofu validate`                     |Syntax / schema check           |
|`tofu fmt -recursive`               |Auto-format all `.tf` files     |
|`tofu plan -out=tfplan`             |Preview changes, save plan      |
|`tofu apply tfplan`                 |Apply saved plan                |
|`tofu apply -target=resource.name`  |Apply single resource           |
|`tofu apply -replace=resource.name` |Force re-create a resource      |
|`tofu destroy -target=resource.name`|Destroy single resource         |
|`tofu state list`                   |List all tracked resources      |
|`tofu state show resource.name`     |Inspect state of resource       |
|`tofu state mv src dst`             |Rename / move state entry       |
|`tofu state rm resource.name`       |Remove from state (no destroy)  |
|`tofu import resource.name id`      |Import existing resource        |
|`tofu output -json`                 |Dump all outputs as JSON        |
|`tofu console`                      |REPL for expression testing     |
|`tofu force-unlock <lock-id>`       |Release stuck state lock        |
|`tofu providers lock`               |Lock provider checksums         |
|`tofu test`                         |Run `.tftest.hcl` test files    |

-----

## 3. Databricks Provider Resources — Full Table

### Identity & Access

|Resource                            |Description                |Import ID           |
|------------------------------------|---------------------------|--------------------|
|`databricks_user`                   |Workspace or account user  |`id` (internal)     |
|`databricks_service_principal`      |SP / app registration      |`application_id`    |
|`databricks_group`                  |Local or SCIM-synced group |`id`                |
|`databricks_group_member`           |Adds user/SP/group to group|`group_id|member_id`|
|`databricks_group_role`             |Assigns role to group      |`group_id|role`     |
|`databricks_user_role`              |Assigns role to user       |`user_id|role`      |
|`databricks_service_principal_role` |Assigns role to SP         |`sp_id|role`        |
|`databricks_access_control_rule_set`|Rule-set for workspace ACL |`rule_set_id`       |

### Unity Catalog Objects

|Resource                         |Description                     |Import ID              |
|---------------------------------|--------------------------------|-----------------------|
|`databricks_metastore`           |UC metastore                    |`metastore_id`         |
|`databricks_metastore_assignment`|Attach metastore to workspace   |`workspace_id`         |
|`databricks_catalog`             |UC catalog                      |`catalog_name`         |
|`databricks_schema`              |Database / schema inside catalog|`catalog.schema`       |
|`databricks_table`               |Managed or external table       |`catalog.schema.table` |
|`databricks_external_location`   |External storage reference      |`name`                 |
|`databricks_storage_credential`  |Cloud credential for storage    |`name`                 |
|`databricks_volume`              |UC volume (file storage)        |`catalog.schema.volume`|
|`databricks_registered_model`    |UC-registered MLflow model      |`catalog.schema.model` |
|`databricks_connection`          |Lakehouse Federation connection |`name`                 |
|`databricks_credential`          |Service credential (non-storage)|`name`                 |

### Grants (Privileges)

|Resource           |Description                                      |
|-------------------|-------------------------------------------------|
|`databricks_grants`|**Authoritative** — replaces ALL grants on object|
|`databricks_grant` |**Additive** — adds a single privilege set       |

### Workspace Security

|Resource                   |Description                                           |
|---------------------------|------------------------------------------------------|
|`databricks_permissions`   |ACL for workspace objects (clusters, jobs, notebooks…)|
|`databricks_secret_acl`    |ACL on a secret scope                                 |
|`databricks_ip_access_list`|IP allowlist / blocklist                              |
|`databricks_workspace_conf`|Workspace-level feature flags                         |

### Compute

|Resource                   |Description               |
|---------------------------|--------------------------|
|`databricks_cluster`       |Interactive cluster       |
|`databricks_cluster_policy`|Cluster policy definition |
|`databricks_instance_pool` |Instance pool             |
|`databricks_job`           |Databricks Job            |
|`databricks_pipeline`      |Delta Live Tables pipeline|
|`databricks_sql_endpoint`  |SQL warehouse             |
|`databricks_sql_query`     |Saved SQL query           |
|`databricks_sql_dashboard` |SQL dashboard             |

### Data Sources (read-only)

|Data Source                   |Returns                          |
|------------------------------|---------------------------------|
|`databricks_current_user`     |Currently authenticated principal|
|`databricks_user`             |Look up user by user_name        |
|`databricks_group`            |Look up group by display_name    |
|`databricks_service_principal`|Look up SP by display_name       |
|`databricks_metastore`        |Metastore details                |
|`databricks_catalogs`         |List of catalogs                 |
|`databricks_schemas`          |List of schemas in catalog       |
|`databricks_tables`           |List of tables in schema         |
|`databricks_node_type`        |Smallest matching node type      |
|`databricks_spark_version`    |Latest LTS runtime               |

-----

## 4. Identity & Group Management

### Creating groups with hierarchy

```hcl
# Top-level functional groups
resource "databricks_group" "data_engineers" {
  display_name = "data-engineers-${local.env}"
  workspace_access            = true
  databricks_sql_access       = false
  allow_cluster_create        = true
  allow_instance_pool_create  = false
}

resource "databricks_group" "data_analysts" {
  display_name          = "data-analysts-${local.env}"
  workspace_access      = true
  databricks_sql_access = true
}

resource "databricks_group" "ml_engineers" {
  display_name              = "ml-engineers-${local.env}"
  workspace_access          = true
  allow_cluster_create      = true
}

# Environment parent groups (nesting pattern)
resource "databricks_group" "env_parent" {
  display_name = "databricks-${local.env}"
}

resource "databricks_group_member" "de_in_parent" {
  group_id  = databricks_group.env_parent.id
  member_id = databricks_group.data_engineers.id
}
```

### Adding members (users, service principals, nested groups)

```hcl
# User membership
resource "databricks_group_member" "add_user" {
  for_each  = toset(var.engineer_users)
  group_id  = databricks_group.data_engineers.id
  member_id = databricks_user.users[each.key].id
}

# SP membership
resource "databricks_group_member" "add_sp" {
  group_id  = databricks_group.data_engineers.id
  member_id = databricks_service_principal.pipeline_sp.id
}
```

### 9-group AWS-to-Databricks mapping pattern

```hcl
# 3 environments × 3 functional roles = 9 groups
locals {
  environments = ["dev", "staging", "prod"]
  roles        = ["engineers", "analysts", "ml-engineers"]

  groups = {
    for combo in setproduct(local.environments, local.roles) :
    "${combo[0]}-${combo[1]}" => {
      env  = combo[0]
      role = combo[1]
    }
  }
}

resource "databricks_group" "env_role" {
  for_each     = local.groups
  display_name = "db-${each.key}"

  workspace_access      = true
  databricks_sql_access = each.value.role == "analysts" ? true : false
  allow_cluster_create  = contains(["engineers", "ml-engineers"], each.value.role)
}
```

-----

## 5. Unity Catalog Securable Objects

### Hierarchy

```
Metastore
└── Catalog
    └── Schema (Database)
        ├── Table
        ├── View
        ├── Volume
        └── Function
```

### Catalog

```hcl
resource "databricks_catalog" "main" {
  name         = "main_${local.env}"
  comment      = "Primary catalog for ${local.env}"
  properties   = { environment = local.env }

  # For external catalog (Lakehouse Federation):
  # connection_name = databricks_connection.mysql.name
  # options         = { database = "mydb" }
}
```

### Schema

```hcl
resource "databricks_schema" "bronze" {
  catalog_name = databricks_catalog.main.name
  name         = "bronze"
  comment      = "Raw ingestion layer"
  properties   = { layer = "bronze" }

  # Force drop even if non-empty (use with caution)
  force_destroy = false
}
```

### External location & storage credential

```hcl
resource "databricks_storage_credential" "adls" {
  name = "adls-credential-${local.env}"

  azure_managed_identity {
    access_connector_id = var.access_connector_id
  }

  comment = "Managed identity for ADLS Gen2"
}

resource "databricks_external_location" "datalake" {
  name            = "datalake-${local.env}"
  url             = "abfss://container@storageaccount.dfs.core.windows.net"
  credential_name = databricks_storage_credential.adls.name
  comment         = "Primary ADLS external location"

  # Validate that Databricks can access the path
  skip_validation = false
}
```

-----

## 6. Grants & Privileges Reference

### Privilege matrix by securable object

|Securable Object      |Privilege                  |Effect                            |
|----------------------|---------------------------|----------------------------------|
|**METASTORE**         |`CREATE CATALOG`           |Create new catalogs               |
|**METASTORE**         |`CREATE CONNECTION`        |Create external connections       |
|**METASTORE**         |`CREATE EXTERNAL LOCATION` |Create external locations         |
|**METASTORE**         |`CREATE STORAGE CREDENTIAL`|Create storage credentials        |
|**METASTORE**         |`MANAGE`                   |Full metastore admin              |
|**CATALOG**           |`USE CATALOG`              |Required to access anything inside|
|**CATALOG**           |`CREATE SCHEMA`            |Create schemas                    |
|**CATALOG**           |`ALL PRIVILEGES`           |Full catalog control              |
|**SCHEMA**            |`USE SCHEMA`               |Required to access objects inside |
|**SCHEMA**            |`CREATE TABLE`             |Create tables/views               |
|**SCHEMA**            |`CREATE FUNCTION`          |Create UDFs                       |
|**SCHEMA**            |`CREATE VOLUME`            |Create volumes                    |
|**SCHEMA**            |`EXECUTE`                  |Execute functions in schema       |
|**SCHEMA**            |`MODIFY`                   |INSERT/UPDATE/DELETE on all tables|
|**SCHEMA**            |`SELECT`                   |Query all tables/views            |
|**TABLE / VIEW**      |`SELECT`                   |Read data                         |
|**TABLE**             |`MODIFY`                   |Write data (INSERT/UPDATE/DELETE) |
|**TABLE**             |`ALL PRIVILEGES`           |Full table control                |
|**VOLUME**            |`READ VOLUME`              |Read files                        |
|**VOLUME**            |`WRITE VOLUME`             |Write files                       |
|**EXTERNAL LOCATION** |`READ FILES`               |Read from path                    |
|**EXTERNAL LOCATION** |`WRITE FILES`              |Write to path                     |
|**EXTERNAL LOCATION** |`CREATE EXTERNAL TABLE`    |Create external table here        |
|**STORAGE CREDENTIAL**|`READ FILES`               |Read via credential               |
|**STORAGE CREDENTIAL**|`WRITE FILES`              |Write via credential              |
|**STORAGE CREDENTIAL**|`CREATE EXTERNAL LOCATION` |Use cred for new ext locations    |
|**REGISTERED MODEL**  |`EXECUTE`                  |Load and use model                |
|**REGISTERED MODEL**  |`APPLY TAG`                |Tag a model version               |
|**REGISTERED MODEL**  |`MODIFY`                   |Update/delete model               |
|**CONNECTION**        |`USE CONNECTION`           |Use for Lakehouse Federation      |

### `databricks_grants` — Authoritative (preferred)

> Replaces ALL grants on the object. Safe for GitOps — prevents privilege drift.

```hcl
# Catalog-level grants
resource "databricks_grants" "catalog_main" {
  catalog = databricks_catalog.main.name

  grant {
    principal  = databricks_group.data_engineers.display_name
    privileges = ["USE CATALOG", "CREATE SCHEMA"]
  }

  grant {
    principal  = databricks_group.data_analysts.display_name
    privileges = ["USE CATALOG"]
  }

  grant {
    principal  = "account users"   # All account users
    privileges = ["USE CATALOG"]
  }
}

# Schema-level grants
resource "databricks_grants" "schema_bronze" {
  schema = "${databricks_catalog.main.name}.bronze"

  grant {
    principal  = databricks_group.data_engineers.display_name
    privileges = ["USE SCHEMA", "CREATE TABLE", "MODIFY", "SELECT"]
  }

  grant {
    principal  = databricks_group.data_analysts.display_name
    privileges = ["USE SCHEMA", "SELECT"]
  }
}

# Table-level grants
resource "databricks_grants" "table_orders" {
  table = "${databricks_catalog.main.name}.silver.orders"

  grant {
    principal  = databricks_group.data_analysts.display_name
    privileges = ["SELECT"]
  }
}

# External location grants
resource "databricks_grants" "ext_location" {
  external_location = databricks_external_location.datalake.name

  grant {
    principal  = databricks_group.data_engineers.display_name
    privileges = ["READ FILES", "WRITE FILES", "CREATE EXTERNAL TABLE"]
  }
}

# Storage credential grants
resource "databricks_grants" "storage_cred" {
  storage_credential = databricks_storage_credential.adls.name

  grant {
    principal  = databricks_group.data_engineers.display_name
    privileges = ["READ FILES", "WRITE FILES"]
  }
}

# Metastore-level grants
resource "databricks_grants" "metastore" {
  metastore = data.databricks_metastore.this.id

  grant {
    principal  = "data-platform-admins"
    privileges = ["CREATE CATALOG", "CREATE EXTERNAL LOCATION", "CREATE STORAGE CREDENTIAL"]
  }
}
```

### `databricks_grant` — Additive

> Adds privileges without removing others. Use for exception/overlay grants.

```hcl
resource "databricks_grant" "analyst_extra_table" {
  table     = "${var.catalog}.${var.schema}.${var.table}"
  principal = databricks_user.analyst_lead.user_name
  privileges = ["SELECT", "MODIFY"]
}
```

### Grants pattern: all schemas in a catalog via `for_each`

```hcl
locals {
  schemas = ["bronze", "silver", "gold", "sandbox"]

  schema_privileges = {
    (databricks_group.data_engineers.display_name) = ["USE SCHEMA", "CREATE TABLE", "MODIFY", "SELECT"]
    (databricks_group.data_analysts.display_name)  = ["USE SCHEMA", "SELECT"]
    (databricks_group.ml_engineers.display_name)   = ["USE SCHEMA", "SELECT"]
  }
}

resource "databricks_grants" "all_schemas" {
  for_each = toset(local.schemas)
  schema   = "${databricks_catalog.main.name}.${each.value}"

  dynamic "grant" {
    for_each = local.schema_privileges
    content {
      principal  = grant.key
      privileges = grant.value
    }
  }
}
```

-----

## 7. Row & Column Security

### Column mask (dynamic view approach)

```hcl
# Grant SELECT on masked view, not underlying table
resource "databricks_grants" "pii_view" {
  table = "${databricks_catalog.main.name}.silver.customers_masked"

  grant {
    principal  = databricks_group.data_analysts.display_name
    privileges = ["SELECT"]
  }
}

# Deny SELECT on raw table for analysts
resource "databricks_grants" "raw_customers" {
  table = "${databricks_catalog.main.name}.silver.customers"

  grant {
    principal  = databricks_group.data_engineers.display_name
    privileges = ["SELECT", "MODIFY"]
  }
  # analysts NOT listed → no access to raw table
}
```

### Row filter function (Unity Catalog)

```hcl
# Deploy the row filter UDF via notebooks/SQL or Databricks Asset Bundles,
# then reference it in grants. OpenTofu manages who can EXECUTE the function.

resource "databricks_grants" "row_filter_fn" {
  schema = "${databricks_catalog.main.name}.security"

  grant {
    principal  = "account users"
    privileges = ["EXECUTE"]
  }
}
```

-----

## 8. Compute Privileges

### Cluster ACL (workspace-level)

```hcl
resource "databricks_permissions" "cluster_access" {
  cluster_id = databricks_cluster.shared.id

  access_control {
    group_name       = databricks_group.data_engineers.display_name
    permission_level = "CAN_RESTART"
  }

  access_control {
    group_name       = databricks_group.data_analysts.display_name
    permission_level = "CAN_ATTACH_TO"
  }
}
```

### Cluster policy ACL

```hcl
resource "databricks_permissions" "policy_access" {
  cluster_policy_id = databricks_cluster_policy.standard.id

  access_control {
    group_name       = databricks_group.data_engineers.display_name
    permission_level = "CAN_USE"
  }
}
```

### SQL Warehouse ACL

```hcl
resource "databricks_permissions" "sql_warehouse" {
  sql_endpoint_id = databricks_sql_endpoint.main.id

  access_control {
    group_name       = databricks_group.data_analysts.display_name
    permission_level = "CAN_USE"
  }

  access_control {
    group_name       = databricks_group.data_engineers.display_name
    permission_level = "CAN_MANAGE"
  }
}
```

### Job permissions

```hcl
resource "databricks_permissions" "etl_job" {
  job_id = databricks_job.etl_pipeline.id

  access_control {
    group_name       = databricks_group.data_engineers.display_name
    permission_level = "CAN_MANAGE_RUN"
  }

  access_control {
    service_principal_name = databricks_service_principal.cicd_sp.application_id
    permission_level       = "IS_OWNER"
  }
}
```

### Permission level reference

|Level           |Applies to                    |Effect                        |
|----------------|------------------------------|------------------------------|
|`CAN_USE`       |Cluster policy, SQL WH, pools |Use the resource              |
|`CAN_ATTACH_TO` |Cluster                       |Attach notebooks              |
|`CAN_RESTART`   |Cluster                       |Restart cluster               |
|`CAN_MANAGE`    |Cluster, job, SQL WH, notebook|Full management               |
|`CAN_RUN`       |Job, notebook                 |Execute                       |
|`CAN_MANAGE_RUN`|Job                           |Run/cancel/view runs          |
|`CAN_VIEW`      |Job, notebook, dashboard      |Read-only view                |
|`CAN_EDIT`      |Notebook, query, dashboard    |Edit content                  |
|`CAN_READ`      |Secret scope                  |Read secrets                  |
|`CAN_WRITE`     |Secret scope                  |Write secrets                 |
|`IS_OWNER`      |Any                           |Full ownership, can change ACL|

-----

## 9. MLflow & AI Asset Permissions

### Registered model (Unity Catalog)

```hcl
resource "databricks_grants" "registered_model" {
  registered_model = "${databricks_catalog.main.name}.ml.churn_model"

  grant {
    principal  = databricks_group.ml_engineers.display_name
    privileges = ["ALL PRIVILEGES"]
  }

  grant {
    principal  = databricks_group.data_engineers.display_name
    privileges = ["EXECUTE", "APPLY TAG"]
  }

  grant {
    principal  = "account users"
    privileges = ["EXECUTE"]
  }
}
```

### MLflow experiment permissions (workspace-level)

```hcl
resource "databricks_permissions" "mlflow_experiment" {
  experiment_id = "/Users/owner@example.com/churn-experiment"

  access_control {
    group_name       = databricks_group.ml_engineers.display_name
    permission_level = "CAN_MANAGE"
  }

  access_control {
    group_name       = databricks_group.data_engineers.display_name
    permission_level = "CAN_READ"
  }
}
```

### Model serving endpoint permissions

```hcl
resource "databricks_permissions" "serving_endpoint" {
  serving_endpoint_id = databricks_model_serving.churn.id

  access_control {
    group_name       = databricks_group.ml_engineers.display_name
    permission_level = "CAN_MANAGE"
  }

  access_control {
    group_name       = "account users"
    permission_level = "CAN_QUERY"
  }
}
```

-----

## 10. Design Patterns & Best Practices

### Pattern 1: Layered grants (least-privilege by layer)

```
Metastore level  → Platform admins only (CREATE CATALOG, etc.)
Catalog level    → All groups get USE CATALOG
Schema level     → Role-specific (analysts=SELECT, engineers=MODIFY)
Table level      → Exception overrides only
```

```hcl
# Implement as three separate grants resources per layer
# Use databricks_grants (authoritative) at each layer
# Never mix additive and authoritative on same object
```

### Pattern 2: Privilege sets as locals

```hcl
locals {
  privilege_sets = {
    read_only  = ["USE CATALOG", "USE SCHEMA", "SELECT"]
    read_write = ["USE CATALOG", "USE SCHEMA", "SELECT", "MODIFY", "CREATE TABLE"]
    schema_admin = ["USE CATALOG", "USE SCHEMA", "SELECT", "MODIFY",
                    "CREATE TABLE", "CREATE FUNCTION", "CREATE VOLUME"]
  }
}

resource "databricks_grants" "gold_schema" {
  schema = "${databricks_catalog.main.name}.gold"

  grant {
    principal  = databricks_group.data_analysts.display_name
    privileges = local.privilege_sets.read_only
  }

  grant {
    principal  = databricks_group.data_engineers.display_name
    privileges = local.privilege_sets.schema_admin
  }
}
```

### Pattern 3: Environment-scoped isolation

```hcl
# Each environment gets its own catalog
resource "databricks_catalog" "env" {
  for_each = toset(["dev", "staging", "prod"])
  name     = each.value
}

# Grant all engineers FULL access to dev, limited to prod
resource "databricks_grants" "catalog_per_env" {
  for_each = toset(["dev", "staging", "prod"])
  catalog  = databricks_catalog.env[each.value].name

  dynamic "grant" {
    for_each = each.value == "dev" ? [1] : []
    content {
      principal  = databricks_group.data_engineers.display_name
      privileges = ["ALL PRIVILEGES"]
    }
  }

  dynamic "grant" {
    for_each = each.value == "prod" ? [1] : []
    content {
      principal  = databricks_group.data_engineers.display_name
      privileges = ["USE CATALOG"]
    }
  }
}
```

### Pattern 4: Service principal per pipeline

```hcl
resource "databricks_service_principal" "pipeline" {
  display_name = "sp-pipeline-${local.env}"
  active       = true
}

resource "databricks_service_principal_role" "pipeline_role" {
  service_principal_id = databricks_service_principal.pipeline.id
  role                 = "roles/databricks.user"   # workspace user
}

# Grant only what the pipeline needs
resource "databricks_grants" "pipeline_grants" {
  schema = "${databricks_catalog.main.name}.bronze"

  grant {
    principal  = databricks_service_principal.pipeline.display_name
    privileges = ["USE SCHEMA", "CREATE TABLE", "MODIFY"]
  }
}
```

### Pattern 5: Module structure for large deployments

```
modules/
  databricks-identity/          # groups, users, SPs
    main.tf
    variables.tf
    outputs.tf
  databricks-catalog/           # catalogs, schemas, locations
    main.tf
    variables.tf
    outputs.tf
  databricks-grants/            # all grants in one place
    main.tf
    variables.tf
  databricks-compute/           # clusters, jobs, SQL warehouses
    main.tf
    variables.tf

environments/
  dev/
    main.tf                     # calls modules with dev vars
    terraform.tfvars
  prod/
    main.tf
    terraform.tfvars
```

### Pattern 6: Data-driven grants via CSV / YAML input

```hcl
# grants.yaml → converted to variable via yamldecode
variable "grants_config" {
  type = map(object({
    catalog    = optional(string)
    schema     = optional(string)
    table      = optional(string)
    privileges = list(string)
  }))
}

# Load from file
locals {
  grants_raw = yamldecode(file("${path.module}/grants.yaml"))
}
```

-----

## 11. SCIM / Entra ID Integration

### SCIM provisioning (account-level)

```hcl
# Entra ID SCIM provisioning is configured in the Azure portal.
# OpenTofu manages the resulting groups/users after sync.

# Reference a SCIM-synced group by display name
data "databricks_group" "entra_de_group" {
  provider     = databricks.accounts
  display_name = "AzureAD-DataEngineers"
}

# Use in grants
resource "databricks_grants" "schema_silver" {
  schema = "${databricks_catalog.main.name}.silver"

  grant {
    principal  = data.databricks_group.entra_de_group.display_name
    privileges = ["USE SCHEMA", "SELECT", "MODIFY"]
  }
}
```

### Group assignment to workspace via account provider

```hcl
resource "databricks_mws_workspace" "this" {
  provider       = databricks.accounts
  account_id     = var.account_id
  workspace_name = "ws-${local.env}"
  # ... networking config ...
}

resource "databricks_access_control_rule_set" "workspace_rules" {
  provider = databricks.accounts
  name     = "accounts/${var.account_id}/workspaces/${databricks_mws_workspace.this.workspace_id}/ruleSets/default"

  grant_rules {
    principals = [
      "groups/${data.databricks_group.entra_de_group.display_name}"
    ]
    role = "roles/databricks.user"
  }
}
```

-----

## 12. Multi-Workspace / Multi-Environment Patterns

### Provider alias per workspace

```hcl
provider "databricks" {
  alias         = "dev"
  host          = var.dev_host
  azure_use_msi = true
}

provider "databricks" {
  alias         = "prod"
  host          = var.prod_host
  azure_use_msi = true
}

resource "databricks_group" "de_dev" {
  provider     = databricks.dev
  display_name = "data-engineers"
}

resource "databricks_group" "de_prod" {
  provider     = databricks.prod
  display_name = "data-engineers"
}
```

### Iterating workspaces with `for_each` providers (OpenTofu 1.7+)

```hcl
# OpenTofu 1.7 introduced for_each on provider blocks (experimental)
# Until stable, use alias pattern above or separate state per environment
```

### Separate state per environment (recommended)

```bash
# Directory per environment, shared modules
environments/dev/   → tofu init && tofu apply
environments/prod/  → tofu init && tofu apply

# CI/CD: pass -var-file and -backend-config per environment
tofu apply \
  -var-file=environments/prod/prod.tfvars \
  -backend-config=environments/prod/backend.hcl
```

-----

## 13. Drift Detection & Remediation

### Scheduled plan in CI (detect drift)

```yaml
# GitHub Actions — drift detection
- name: Detect drift
  run: |
    tofu plan -detailed-exitcode -out=drift.tfplan
    # exit 0 = no changes, exit 2 = drift detected
  continue-on-error: true

- name: Alert on drift
  if: steps.detect-drift.outcome == 'failure'
  run: echo "Drift detected!" | slack-notify
```

### Import existing resources

```bash
# Import existing Databricks group
tofu import databricks_group.data_engineers <group_id>

# Import existing catalog
tofu import databricks_grants.catalog_main <catalog_name>

# Generate config from existing state (OpenTofu 1.6+)
tofu plan -generate-config-out=generated.tf
```

### Prevent accidental destroy

```hcl
resource "databricks_catalog" "main" {
  name = "main_prod"

  lifecycle {
    prevent_destroy = true    # Blocks tofu destroy
  }
}

resource "databricks_grants" "catalog_main" {
  catalog = databricks_catalog.main.name

  lifecycle {
    # Re-apply grants if someone manually changes them
    ignore_changes = []  # keep empty to track all changes
  }
}
```

-----

## 14. Common Mistakes & Gotchas

|Mistake                                                       |Issue                                                          |Fix                                                 |
|--------------------------------------------------------------|---------------------------------------------------------------|----------------------------------------------------|
|Mixing `databricks_grants` + `databricks_grant` on same object|`databricks_grants` will overwrite additive grants             |Use ONE approach per object                         |
|Using `display_name` vs `user_name` as principal              |Grants require `display_name` for groups, `user_name` for users|Check principal type                                |
|Not granting `USE CATALOG` + `USE SCHEMA`                     |Child object grants fail without parent USE grants             |Always grant USE up the hierarchy                   |
|Hardcoding SP secrets in `.tf` files                          |Secret exposed in state file                                   |Use `sensitive` variables + Key Vault data source   |
|Single provider for account + workspace ops                   |Metastore assignment requires account-level provider           |Use `provider = databricks.accounts` for account ops|
|`prevent_destroy = false` in prod                             |Catalog/schema destroyed on plan accident                      |Set `prevent_destroy = true` for all prod objects   |
|Not pinning provider version                                  |Breaking changes in minor releases                             |Use `~> 1.38` (allows patch, blocks minor)          |
|Modifying grants outside Tofu                                 |State drifts; next `apply` reverts                             |Use `databricks_grants` for authoritative control   |
|Missing `metastore_assignment`                                |Workspace cannot see Unity Catalog                             |Explicitly assign metastore to workspace            |
|Granting to workspace-local group in UC                       |UC uses account-level groups                                   |Ensure groups are created at account level          |

-----

## Quick-Reference Snippets

### Lookup current user

```hcl
data "databricks_current_user" "me" {}

output "current_user" {
  value = data.databricks_current_user.me.user_name
}
```

### Sensitive variable pattern

```hcl
variable "sp_client_secret" {
  type      = string
  sensitive = true
}

# Pass via env var: TF_VAR_sp_client_secret=...
# Or: tofu apply -var="sp_client_secret=$SECRET"
```

### Tag all resources

```hcl
locals {
  common_tags = {
    environment = local.env
    managed_by  = "opentofu"
    team        = "data-platform"
  }
}

resource "databricks_catalog" "main" {
  name       = "main_${local.env}"
  properties = local.common_tags
}
```

### Full privilege revoke (authoritative empty grants)

```hcl
# Remove ALL grants from an object
resource "databricks_grants" "revoke_all" {
  schema = "main.deprecated_schema"
  # No grant blocks = authoritative empty = all privileges removed
}
```

-----

## 15. End-to-End Deployment in Practice

This section walks through the complete lifecycle: creating and configuring a Service Principal, installing OpenTofu, authenticating against an existing Databricks workspace, and running a deployment — both locally and from a remote CI/CD pipeline.

-----

### 15.1 Service Principal Setup (Azure / Databricks Account)

A Service Principal (SP) is the recommended non-human identity for all automated deployments. You need **two levels** of permission:

|Level                         |What it controls                                        |Where it is granted                 |
|------------------------------|--------------------------------------------------------|------------------------------------|
|**Azure RBAC**                |Ability to read ARM resources, use MSI, access Key Vault|Azure Portal / `az` CLI             |
|**Databricks account-level**  |Ability to manage Unity Catalog, workspaces, metastore  |Databricks Account Console          |
|**Databricks workspace-level**|Ability to deploy resources inside a specific workspace |Workspace Admin settings or OpenTofu|

#### Step 1 — Create the SP in Entra ID

```bash
# Create SP and capture credentials
az ad sp create-for-rbac \
  --name "sp-opentofu-databricks-prod" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<rg-name> \
  --output json

# Output — save these securely:
# {
#   "appId":       "<client-id>",       ← ARM_CLIENT_ID
#   "displayName": "sp-opentofu-...",
#   "password":    "<client-secret>",   ← ARM_CLIENT_SECRET
#   "tenant":      "<tenant-id>"        ← ARM_TENANT_ID
# }
```

#### Step 2 — Add the SP as a Databricks account admin

In the **Databricks Account Console** (`accounts.azuredatabricks.net`):

1. Go to **User Management → Service Principals**
1. Add the SP by its Entra ID `appId` (client ID)
1. Assign the **Account Admin** role if it needs to manage metastores/catalogs, or **Workspace Admin** via workspace assignment for workspace-only operations

#### Step 3 — Add the SP to the workspace

```hcl
# OpenTofu can manage this — or do it once manually in the Workspace Admin UI
resource "databricks_service_principal" "tofu_deployer" {
  application_id = var.sp_client_id
  display_name   = "sp-opentofu-deployer"
  active         = true
}

# Give the SP workspace admin rights so it can deploy everything
resource "databricks_service_principal_role" "tofu_admin" {
  service_principal_id = databricks_service_principal.tofu_deployer.id
  role                 = "roles/databricks.admin"
}
```

> **Principle of least privilege:** For deployments that only manage grants and UC objects (not workspace settings), `roles/databricks.user` + Unity Catalog `MANAGE` on relevant objects is sufficient. Reserve `admin` for initial bootstrapping only.

-----

### 15.2 Local Machine Setup

#### Install OpenTofu

```bash
# macOS (Homebrew)
brew install opentofu

# Linux — official install script
curl --proto '=https' --tlsv1.2 -fsSL https://get.opentofu.org/install-opentofu.sh | sh

# Windows (winget)
winget install OpenTofu.OpenTofu

# Verify
tofu version
# OpenTofu v1.9.x
```

#### Install Azure CLI (needed for `az login` / MSI auth)

```bash
# macOS
brew install azure-cli

# Linux
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verify
az version
```

#### Authenticate locally — two approaches

**Option A: Client secret via environment variables (simplest)**

```bash
# Set in your shell profile (.zshrc / .bashrc) or a .env file (never commit .env)
export ARM_CLIENT_ID="<appId>"
export ARM_CLIENT_SECRET="<password>"
export ARM_TENANT_ID="<tenant>"
export ARM_SUBSCRIPTION_ID="<subscription-id>"

# Databricks-specific (used by provider when azure_use_msi = false)
export DATABRICKS_HOST="https://adb-<id>.18.azuredatabricks.net"
export DATABRICKS_AZURE_CLIENT_ID="$ARM_CLIENT_ID"
export DATABRICKS_AZURE_CLIENT_SECRET="$ARM_CLIENT_SECRET"
export DATABRICKS_AZURE_TENANT_ID="$ARM_TENANT_ID"
```

**Option B: Azure CLI interactive login (developer convenience)**

```bash
# Log in as yourself interactively — OpenTofu picks this up automatically
az login
az account set --subscription "<subscription-id>"

# Provider block uses az CLI token automatically when no explicit auth is set
# Useful for read-only plan runs during development
```

#### Provider block for local development

```hcl
# providers.tf
provider "databricks" {
  host = var.databricks_host

  # Explicit SP auth — reads from env vars if variables not set
  azure_client_id       = var.azure_client_id       # or env DATABRICKS_AZURE_CLIENT_ID
  azure_client_secret   = var.azure_client_secret   # or env DATABRICKS_AZURE_CLIENT_SECRET
  azure_tenant_id       = var.azure_tenant_id       # or env DATABRICKS_AZURE_TENANT_ID
}
```

#### Local variable management with `terraform.tfvars`

```hcl
# terraform.tfvars  ← add to .gitignore, NEVER commit
databricks_host       = "https://adb-<id>.18.azuredatabricks.net"
azure_client_id       = "<appId>"
azure_client_secret   = "<client-secret>"
azure_tenant_id       = "<tenant-id>"
databricks_account_id = "<account-id>"
```

```bash
# .gitignore
*.tfvars
*.tfvars.json
.terraform/
.terraform.lock.hcl   # commit this one — it pins provider checksums
terraform.tfstate
terraform.tfstate.backup
*.tfplan
.env
```

#### First-time local deployment workflow

```bash
# 1. Clone your repo and navigate to the environment directory
cd environments/dev

# 2. Initialise — downloads provider, configures backend
tofu init

# 3. Validate syntax and schema
tofu validate

# 4. Preview what will be created/changed/destroyed
tofu plan -out=dev.tfplan

# 5. Review the plan output carefully, then apply
tofu apply dev.tfplan

# 6. Confirm state is clean
tofu plan   # should report "No changes"
```

-----

### 15.3 Remote CI/CD Pipeline Setup

#### Authentication in CI — Managed Identity (recommended for Azure pipelines)

When running inside Azure (Azure DevOps, GitHub Actions with Azure-hosted runners, or ACI), use a **Managed Identity** instead of a client secret — no credentials to rotate or leak.

```hcl
# Provider block for CI with MSI
provider "databricks" {
  host          = var.databricks_host
  azure_use_msi = true                      # uses VM/runner managed identity automatically
  # azure_msi_endpoint — only needed for custom MSI endpoints
}

provider "azurerm" {
  use_msi = true
  features {}
}
```

Assign the managed identity of the runner the same Entra ID / Databricks permissions as the SP described in §15.1.

#### GitHub Actions — full deployment pipeline

```yaml
# .github/workflows/deploy.yml
name: OpenTofu Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write      # Required for OIDC / Workload Identity Federation
  contents: read
  pull-requests: write

env:
  TF_VERSION: "1.9.0"
  ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
  ARM_TENANT_ID:        ${{ secrets.ARM_TENANT_ID }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}

    steps:
      # ── Auth ──────────────────────────────────────────────
      - name: Azure Login (OIDC — no stored secret)
        uses: azure/login@v2
        with:
          client-id:       ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id:       ${{ secrets.ARM_TENANT_ID }}
          subscription-id: ${{ secrets.ARM_SUBSCRIPTION_ID }}

      # ── OpenTofu ──────────────────────────────────────────
      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: ${{ env.TF_VERSION }}

      - uses: actions/checkout@v4

      - name: Tofu Init
        run: tofu init -input=false
        working-directory: environments/${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}
        env:
          ARM_CLIENT_ID:     ${{ secrets.AZURE_CLIENT_ID }}
          # Backend uses OIDC — no ARM_CLIENT_SECRET needed

      - name: Tofu Validate
        run: tofu validate

      - name: Tofu Plan
        id: plan
        run: tofu plan -input=false -out=pipeline.tfplan -no-color 2>&1 | tee plan.txt
        env:
          DATABRICKS_HOST:                  ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_AZURE_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
          DATABRICKS_AZURE_TENANT_ID:       ${{ secrets.ARM_TENANT_ID }}
          ARM_CLIENT_ID:                    ${{ secrets.AZURE_CLIENT_ID }}
          ARM_USE_OIDC:                     "true"

      - name: Post Plan to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const plan = require('fs').readFileSync('plan.txt', 'utf8').slice(0, 65000);
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `### OpenTofu Plan\n\`\`\`\n${plan}\n\`\`\``
            });

      - name: Tofu Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: tofu apply -input=false pipeline.tfplan
        env:
          DATABRICKS_HOST:                  ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_AZURE_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
          DATABRICKS_AZURE_TENANT_ID:       ${{ secrets.ARM_TENANT_ID }}
          ARM_CLIENT_ID:                    ${{ secrets.AZURE_CLIENT_ID }}
          ARM_USE_OIDC:                     "true"
```

#### Azure DevOps pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

variables:
  - group: databricks-secrets          # Variable Group in Azure DevOps Library
  - name: TF_ROOT
    value: environments/$(Build.SourceBranchName)

stages:
  - stage: Plan
    jobs:
      - job: TofuPlan
        steps:
          - task: AzureCLI@2
            displayName: Azure Login
            inputs:
              azureSubscription: sc-opentofu-prod    # Service Connection
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: echo "Authenticated via Service Connection"

          - script: |
              curl -fsSL https://get.opentofu.org/install-opentofu.sh | sh
              tofu version
            displayName: Install OpenTofu

          - script: tofu init -input=false
            displayName: Tofu Init
            workingDirectory: $(TF_ROOT)
            env:
              ARM_CLIENT_ID:     $(ARM_CLIENT_ID)
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              ARM_TENANT_ID:     $(ARM_TENANT_ID)
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)

          - script: tofu plan -input=false -out=pipeline.tfplan
            displayName: Tofu Plan
            workingDirectory: $(TF_ROOT)
            env:
              DATABRICKS_HOST:              $(DATABRICKS_HOST)
              DATABRICKS_AZURE_CLIENT_ID:   $(ARM_CLIENT_ID)
              DATABRICKS_AZURE_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              DATABRICKS_AZURE_TENANT_ID:   $(ARM_TENANT_ID)

          - publish: $(TF_ROOT)/pipeline.tfplan
            artifact: tfplan

  - stage: Apply
    dependsOn: Plan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: TofuApply
        environment: production              # requires manual approval gate in ADO
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: tfplan

                - script: tofu apply -input=false pipeline.tfplan
                  displayName: Tofu Apply
                  workingDirectory: $(TF_ROOT)
                  env:
                    DATABRICKS_HOST:              $(DATABRICKS_HOST)
                    DATABRICKS_AZURE_CLIENT_ID:   $(ARM_CLIENT_ID)
                    DATABRICKS_AZURE_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
                    DATABRICKS_AZURE_TENANT_ID:   $(ARM_TENANT_ID)
```

-----

### 15.4 Secrets Management — Best Practices

Never store credentials in `.tf` files or pipeline YAML. Use one of these approaches:

#### Azure Key Vault as secrets backend

```hcl
# Read secrets at plan/apply time — nothing stored in .tf
data "azurerm_key_vault" "tofu_kv" {
  name                = "kv-opentofu-prod"
  resource_group_name = "rg-platform"
}

data "azurerm_key_vault_secret" "sp_secret" {
  name         = "sp-opentofu-client-secret"
  key_vault_id = data.azurerm_key_vault.tofu_kv.id
}

provider "databricks" {
  host                  = var.databricks_host
  azure_client_id       = var.azure_client_id
  azure_client_secret   = data.azurerm_key_vault_secret.sp_secret.value
  azure_tenant_id       = var.azure_tenant_id
}
```

#### Environment variable precedence (provider resolution order)

The Databricks provider resolves authentication in this order — first match wins:

```
1. Explicit attributes in provider block
2. Environment variables (DATABRICKS_*)
3. ~/.databrickscfg profile
4. Azure Managed Identity (if azure_use_msi = true)
5. Azure CLI token (az login)
```

```bash
# Full set of Databricks provider env vars
DATABRICKS_HOST                       # workspace URL
DATABRICKS_TOKEN                      # PAT (avoid in prod — use SP)
DATABRICKS_AZURE_CLIENT_ID            # SP appId
DATABRICKS_AZURE_CLIENT_SECRET        # SP secret
DATABRICKS_AZURE_TENANT_ID            # Entra tenant
DATABRICKS_AZURE_SUBSCRIPTION_ID      # Azure subscription
DATABRICKS_ACCOUNT_ID                 # for account-level provider
```

#### GitHub Actions secrets checklist

```
Repository / Environment Secrets:
  ARM_SUBSCRIPTION_ID        ← Azure subscription
  ARM_TENANT_ID              ← Entra tenant ID
  AZURE_CLIENT_ID            ← SP appId (used with OIDC — no secret needed)
  DATABRICKS_HOST            ← workspace URL per environment
  DATABRICKS_ACCOUNT_ID      ← Databricks account ID

For client-secret auth (non-OIDC):
  AZURE_CLIENT_SECRET        ← SP secret — rotate every 90 days
```

-----

### 15.5 State File Security

The state file contains sensitive data (resource IDs, some attribute values). Protect it:

```hcl
# Remote state with encryption at rest and access control
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "satfstate<env>"
    container_name       = "tofu-state"
    key                  = "databricks/<env>.tfstate"
    use_azuread_auth     = true      # no storage key in config
    # State blob is encrypted with Azure Storage SSE by default
    # Enable customer-managed key (CMK) in the storage account for higher compliance
  }
}
```

```bash
# Lock storage account: only the SP and platform admins should have access
az role assignment create \
  --assignee <sp-object-id> \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub>/resourceGroups/rg-tfstate/providers/Microsoft.Storage/storageAccounts/satfstate
```

-----

### 15.6 Bootstrapping Order for a New Workspace

When a workspace exists but OpenTofu has never been run, follow this sequence to avoid dependency errors:

```
Phase 1 — Account-level (provider = databricks.accounts)
  ├── databricks_service_principal  (the deployer SP itself)
  ├── databricks_metastore          (if not already exists)
  ├── databricks_metastore_assignment
  └── Account-level group creation (SCIM groups)

Phase 2 — Storage & credentials
  ├── databricks_storage_credential
  └── databricks_external_location

Phase 3 — Catalog structure
  ├── databricks_catalog
  └── databricks_schema (per layer)

Phase 4 — Identity (workspace-level groups & members)
  ├── databricks_group
  ├── databricks_group_member
  └── databricks_service_principal_role

Phase 5 — Grants (depends on all above)
  ├── databricks_grants (metastore)
  ├── databricks_grants (catalogs)
  ├── databricks_grants (schemas)
  └── databricks_grants (tables / volumes / models)

Phase 6 — Compute permissions
  ├── databricks_permissions (clusters, SQL WH)
  └── databricks_permissions (jobs, pipelines)
```

Use `-target` to apply phases explicitly during initial bootstrap:

```bash
# Phase 1
tofu apply -target=databricks_metastore_assignment.this

# Phase 2
tofu apply -target=databricks_storage_credential.adls \
           -target=databricks_external_location.datalake

# Phase 3+
tofu apply   # full apply once foundations are in place
```

-----

### 15.7 Day-2 Operations — Ongoing Deployment Workflow

```
Developer workflow (feature branch):
  1. git checkout -b feat/add-analyst-group
  2. Edit .tf files
  3. tofu fmt && tofu validate
  4. tofu plan                          ← review locally
  5. git push → PR opens
  6. CI runs tofu plan → posts output to PR
  7. Peer review of plan output (not just code)
  8. Merge to main → CI runs tofu apply automatically

Hotfix / emergency (break-glass):
  1. tofu apply -target=databricks_grants.affected_object
  2. Immediately raise a PR to capture the change in code
  3. Confirm CI plan shows "No changes" after merge
```

#### Makefile for local convenience

```makefile
# Makefile
ENV ?= dev
ROOT = environments/$(ENV)

init:
	cd $(ROOT) && tofu init -upgrade

fmt:
	tofu fmt -recursive

validate: fmt
	cd $(ROOT) && tofu validate

plan: validate
	cd $(ROOT) && tofu plan -out=$(ENV).tfplan

apply:
	cd $(ROOT) && tofu apply $(ENV).tfplan

destroy-dry:
	cd $(ROOT) && tofu plan -destroy

# Usage:
# make plan ENV=prod
# make apply ENV=prod
```

-----

### 15.8 Authentication Decision Matrix

|Scenario                        |Recommended auth method                  |Why                                         |
|--------------------------------|-----------------------------------------|--------------------------------------------|
|Local dev, interactive          |`az login` (Azure CLI)                   |No secrets to manage; uses your own identity|
|Local dev, SP testing           |Client secret via env vars               |Quick to set up; never hardcode in `.tf`    |
|GitHub Actions                  |OIDC / Workload Identity Federation      |No stored secrets; short-lived tokens       |
|Azure DevOps                    |Service Connection (SP or MSI)           |Native ADO secret management                |
|Azure-hosted runner / ACI       |Managed Identity (`azure_use_msi = true`)|Zero credential management                  |
|Self-hosted runner in Azure VM  |Managed Identity on the VM               |Same as above                               |
|Self-hosted runner outside Azure|Client secret via pipeline secret store  |Rotate every 90 days; use Key Vault         |
|Databricks PAT                  |**Avoid in automation**                  |Non-auditable, no expiry enforcement        |

-----

*Generated for OpenTofu ≥ 1.6 · Databricks provider `databricks/databricks` ≥ 1.38 · Unity Catalog GA*