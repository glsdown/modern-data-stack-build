# Warehouses

On this page, you will:

- [x] Build the `snowflake_warehouse` module
- [x] Create shared warehouses for different workloads
- [x] Configure auto-suspend and sizing for cost efficiency

## Why Dedicated Warehouses?

Snowflake warehouses are virtual compute clusters that run your queries. Unlike traditional databases where compute is tied to storage, Snowflake lets you create multiple warehouses that all access the same data.

Dedicated warehouses for different workloads provide:

- **Cost tracking**: See exactly how much each workload costs
- **Resource isolation**: Heavy transformations don't slow down reports
- **Right-sizing**: Size each warehouse for its specific workload
- **Independent scaling**: Scale up reporting without affecting loading

## The Warehouse Module

Navigate to your Snowflake Terraform directory and create the warehouse module:

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/snowflake
mkdir -p modules/snowflake_warehouse
```

### main.tf

Create `modules/snowflake_warehouse/main.tf`:

```hcl
terraform {
  required_providers {
    snowflake = {
      source                = "Snowflake-Labs/snowflake"
      version               = "~> 0.99"
      configuration_aliases = [snowflake.sys_admin, snowflake.security_admin]
    }
  }
}

# -----------------------------------------------------------------------------
# Warehouse
# -----------------------------------------------------------------------------
resource "snowflake_warehouse" "this" {
  provider = snowflake.sys_admin
  name     = upper(var.warehouse_name)
  comment  = var.warehouse_comment

  # Start suspended to avoid immediate costs
  initially_suspended = true

  # Size and scaling
  warehouse_size    = var.warehouse_size
  min_cluster_count = var.warehouse_min_cluster_count == 1 ? null : var.warehouse_min_cluster_count
  max_cluster_count = var.warehouse_max_cluster_count == 1 ? null : var.warehouse_max_cluster_count
  scaling_policy    = var.warehouse_max_cluster_count > 1 ? var.warehouse_scaling_policy : null

  # Cost control
  auto_suspend = var.warehouse_auto_suspend
  auto_resume  = true

  # Query acceleration (Enterprise edition)
  enable_query_acceleration           = var.warehouse_enable_query_acceleration
  query_acceleration_max_scale_factor = var.warehouse_enable_query_acceleration ? var.warehouse_query_acceleration_max_scale_factor : null

  lifecycle {
    ignore_changes = [
      initially_suspended  # Prevents imported warehouses from being toggled
    ]
  }
}

# -----------------------------------------------------------------------------
# Grants
# -----------------------------------------------------------------------------
resource "snowflake_grant_privileges_to_account_role" "usage" {
  provider   = snowflake.security_admin
  for_each   = toset(var.warehouse_usage_roles)
  privileges = ["MONITOR", "OPERATE", "USAGE"]

  account_role_name = upper(each.value)

  on_account_object {
    object_type = "WAREHOUSE"
    object_name = snowflake_warehouse.this.name
  }
}
```

This module:

- Creates the warehouse using `sys_admin` (SYSADMIN role)
- Grants usage permissions using `security_admin` (SECURITYADMIN role)
- Starts warehouses suspended to avoid immediate charges
- Handles multi-cluster warehouse settings conditionally

### variables.tf

Create `modules/snowflake_warehouse/variables.tf`:

```hcl
variable "warehouse_name" {
  description = "The name of the warehouse (will be uppercased)"
  type        = string
}

variable "warehouse_comment" {
  description = "Description of the warehouse's purpose"
  type        = string
}

variable "warehouse_size" {
  description = "The size of the warehouse"
  type        = string
  default     = "X-Small"

  validation {
    condition = contains([
      "X-Small", "Small", "Medium", "Large", "X-Large",
      "2X-Large", "3X-Large", "4X-Large", "5X-Large", "6X-Large"
    ], var.warehouse_size)
    error_message = "Invalid warehouse size. Must be X-Small through 6X-Large."
  }
}

variable "warehouse_auto_suspend" {
  description = "Seconds of inactivity before auto-suspend (minimum 60)"
  type        = number
  default     = 60

  validation {
    condition     = var.warehouse_auto_suspend >= 60
    error_message = "Auto-suspend must be at least 60 seconds."
  }
}

variable "warehouse_min_cluster_count" {
  description = "Minimum clusters for multi-cluster warehouse (Enterprise edition)"
  type        = number
  default     = 1

  validation {
    condition     = var.warehouse_min_cluster_count >= 1 && var.warehouse_min_cluster_count <= 10
    error_message = "Cluster count must be between 1 and 10."
  }
}

variable "warehouse_max_cluster_count" {
  description = "Maximum clusters for multi-cluster warehouse (Enterprise edition)"
  type        = number
  default     = 1

  validation {
    condition     = var.warehouse_max_cluster_count >= 1 && var.warehouse_max_cluster_count <= 10
    error_message = "Cluster count must be between 1 and 10."
  }
}

variable "warehouse_scaling_policy" {
  description = "Scaling policy for multi-cluster warehouses: STANDARD or ECONOMY"
  type        = string
  default     = "STANDARD"

  validation {
    condition     = contains(["STANDARD", "ECONOMY"], var.warehouse_scaling_policy)
    error_message = "Scaling policy must be STANDARD or ECONOMY."
  }
}

variable "warehouse_enable_query_acceleration" {
  description = "Enable query acceleration (Enterprise edition)"
  type        = bool
  default     = false
}

variable "warehouse_query_acceleration_max_scale_factor" {
  description = "Max scale factor for query acceleration (1-100)"
  type        = number
  default     = 8

  validation {
    condition     = var.warehouse_query_acceleration_max_scale_factor >= 1 && var.warehouse_query_acceleration_max_scale_factor <= 100
    error_message = "Query acceleration scale factor must be between 1 and 100."
  }
}

variable "warehouse_usage_roles" {
  description = "Account roles to grant USAGE, OPERATE, and MONITOR privileges"
  type        = list(string)
  default     = []
}
```

### outputs.tf

Create `modules/snowflake_warehouse/outputs.tf`:

```hcl
output "warehouse_name" {
  description = "The name of the warehouse"
  value       = snowflake_warehouse.this.name
}

output "warehouse_size" {
  description = "The size of the warehouse"
  value       = snowflake_warehouse.this.warehouse_size
}
```

## Create Shared Warehouses

Now use the module to create warehouses for different workloads. Create `warehouses.tf` in your root Snowflake directory:

```hcl
# =============================================================================
# Shared Warehouses
# =============================================================================
# Dedicated warehouses for different workload types.
# Each warehouse can be sized and configured independently.

# -----------------------------------------------------------------------------
# Loading Warehouse
# -----------------------------------------------------------------------------
# Used by data ingestion tools (Airbyte, dlt, Fivetran)
module "warehouse_loading" {
  source = "./modules/snowflake_warehouse"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  warehouse_name    = "LOADING"
  warehouse_comment = "Data ingestion from external sources."
  warehouse_size    = "X-Small"

  # Loaders run periodically, suspend quickly between loads
  warehouse_auto_suspend = 60

  warehouse_usage_roles = [
    # Add loader service accounts here as you create them
    # "SVC_AIRBYTE",
    # "SVC_DLT",
  ]
}

# -----------------------------------------------------------------------------
# Transforming Warehouse
# -----------------------------------------------------------------------------
# Used by dbt for transformations
module "warehouse_transforming" {
  source = "./modules/snowflake_warehouse"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  warehouse_name    = "TRANSFORMING"
  warehouse_comment = "dbt transformations and data modelling."
  warehouse_size    = "Small"

  # Transformations run in bursts, suspend between runs
  warehouse_auto_suspend = 60

  warehouse_usage_roles = [
    # Add transformer service accounts here
    # "SVC_DBT",
  ]
}

# -----------------------------------------------------------------------------
# Reporting Warehouse
# -----------------------------------------------------------------------------
# Used by BI tools (Metabase, Looker, Tableau)
module "warehouse_reporting" {
  source = "./modules/snowflake_warehouse"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  warehouse_name    = "REPORTING"
  warehouse_comment = "Business intelligence and reporting queries."
  warehouse_size    = "Small"

  # BI tools have sporadic usage, but keep warm for responsiveness
  warehouse_auto_suspend = 300

  warehouse_usage_roles = [
    # Add reporter roles here
    # "ANALYTICS_REPORTER",
    # "SVC_METABASE",
  ]
}

# -----------------------------------------------------------------------------
# Developer Warehouse
# -----------------------------------------------------------------------------
# Used by data team members for ad-hoc queries
module "warehouse_developer" {
  source = "./modules/snowflake_warehouse"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  warehouse_name    = "DEVELOPER"
  warehouse_comment = "Ad-hoc queries and development work."
  warehouse_size    = "X-Small"

  # Developers often pause between queries
  warehouse_auto_suspend = 120

  warehouse_usage_roles = [
    # Add developer roles here
    # "ANALYTICS_DEVELOPER",
  ]
}
```

!!! info "Roles Coming Later"
    The `warehouse_usage_roles` are commented out because we haven't created the functional roles yet. You'll uncomment these in the [Functional Roles](4-functional-roles.md) section.

## Warehouse Sizing Guide

Warehouse size affects both performance and cost. Each size doubles the compute power (and cost) of the previous:

| Size | Credits/Hour | Best For |
|------|--------------|----------|
| X-Small | 1 | Development, light loads, testing |
| Small | 2 | Small datasets, basic transformations |
| Medium | 4 | Medium datasets, complex joins |
| Large | 8 | Large datasets, heavy aggregations |
| X-Large+ | 16+ | Very large datasets, intensive workloads |

!!! tip "Start Small, Scale Up"
    Always start with X-Small or Small. Monitor query performance and scale up only when queries are consistently slow. Snowflake makes it easy to resize without downtime.

## Cost Control Settings

The module includes several cost control features:

### Auto-Suspend

Warehouses automatically suspend after a period of inactivity:

```hcl
warehouse_auto_suspend = 60  # Suspend after 60 seconds
```

- **60 seconds**: Aggressive, best for batch workloads (loading, transforming)
- **300 seconds**: Moderate, good for interactive use (reporting)
- **600+ seconds**: Conservative, for very sporadic usage

### Auto-Resume

Warehouses automatically resume when a query arrives:

```hcl
auto_resume = true  # Always enabled in our module
```

This is always enabled so queries don't fail when the warehouse is suspended.

### Initially Suspended

New warehouses start in a suspended state:

```hcl
initially_suspended = true
```

This prevents charges from accumulating immediately after creation.

## Commit and Deploy

Commit your changes and push to trigger the CI/CD pipeline:

```sh
git add terraform/snowflake/
git commit -m "Add snowflake_warehouse module and shared warehouses"
git push
```

The pipeline will run `terraform plan` on the pull request. Review the plan to confirm it will create four warehouses:

```
Plan: 4 to add, 0 to change, 0 to destroy.

  + module.warehouse_developer.snowflake_warehouse.this
  + module.warehouse_loading.snowflake_warehouse.this
  + module.warehouse_reporting.snowflake_warehouse.this
  + module.warehouse_transforming.snowflake_warehouse.this
```

Once merged, the pipeline applies the changes automatically.

## Verify in Snowflake

After the pipeline completes, connect to Snowflake and verify the warehouses were created:

```sql
SHOW WAREHOUSES;
```

You should see your four new warehouses, all initially suspended.

## Summary

You've created the warehouse infrastructure for your data platform:

- [x] Built the `snowflake_warehouse` module
- [x] Created LOADING, TRANSFORMING, REPORTING, and DEVELOPER warehouses
- [x] Configured appropriate sizes and auto-suspend settings for cost efficiency

## What's Next

With compute resources in place, you need somewhere to store data. In the next section, you'll create the `snowflake_database` module and set up databases with reader/writer roles.

Continue to [Databases](3-databases.md) →
