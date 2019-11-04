# Azure SQL
[![Changelog](https://img.shields.io/badge/changelog-release-green.svg)](CHANGELOG.md) [![Notice](https://img.shields.io/badge/notice-copyright-yellow.svg)](NOTICE) [![Apache V2 License](http://img.shields.io/badge/license-Apache%20V2-blue.svg)](LICENSE)

This Terraform module creates an [Azure SQL Server](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-servers) 
and associated databases in an [SQL Elastic Pool](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-elastic-pool) 
with [DTU purchasing model](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-service-tiers-dtu) 
only along with [Firewall rules](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-firewall-configure) 
and [Diagnostic settings](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-metrics-diag-logging) 
enabled.

The [vCore-based model](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-service-tiers-vcore)
is not available.

## Requirements

* [Terraform](https://www.terraform.io/downloads.html) >= 0.12
* [AzureRM Terraform provider](https://www.terraform.io/docs/providers/azurerm/) >= 1.31
* [PowerShell with Az.Sql module](https://docs.microsoft.com/en-us/powershell/module/az.sql/) >= 1.3 is mandatory and is used for backup retention configuration
* [`SqlServer` Powershell module](https://docs.microsoft.com/en-us/sql/powershell/sql-server-powershell) is needed for databases users creation

## Limitations

* The long term backup retention configuration is done asynchronously since the command lasts a long time.
  Command result is never fetch, only a check on Azure allows to know if configuration went fine.

## Usage

You can use this module by including it this way:
```hcl
module "az-region" {
  source = "git::ssh://git@git.fr.clara.net/claranet/cloudnative/projects/cloud/azure/terraform/modules/regions.git?ref=vX.X.X"

  azure_region = var.azure_region
}

module "rg" {
  source = "git::ssh://git@git.fr.clara.net/claranet/cloudnative/projects/cloud/azure/terraform/modules/rg.git?ref=vX.X.X"

  location    = module.az-region.location
  client_name = var.client_name
  environment = var.environment
  stack       = var.stack
}

module "sql" {
  source = "git::ssh://git@git.fr.clara.net/claranet/cloudnative/projects/cloud/azure/terraform/features/db-sql.git?ref=vX.X.X"

  client_name         = var.client_name
  environment         = var.environment
  location            = module.az-region.location
  location_short      = module.az-region.location_short
  resource_group_name = module.rg.resource_group_name
  stack               = var.stack

  databases_names = ["users", "documents"]

  administrator_login    = "claranet"
  administrator_password = var.sql_admin_password

  sku = {
    tier     = "Standard"
    capacity = "100"
  }

  elastic_pool_max_size = "50"

  # This can costs you money https://docs.microsoft.com/en-us/azure/sql-database/sql-database-advanced-data-security
  enable_advanced_data_security = "true"

  enable_logs_to_storage  = "true"
  logs_storage_account_id = data.terraform_remote_state.run.logs_storage_account_id

  enable_logs_to_log_analytics    = "true"
  logs_log_analytics_workspace_id = data.terraform_remote_state.run.log_analytics_id
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| administrator\_login | Administrator login for SQL Server | string | n/a | yes |
| administrator\_password | Administrator password for SQL Server | string | n/a | yes |
| advanced\_data\_security\_additional\_emails | List of addiional email addresses for Advanced Data Security alerts. | list(string) | `<list>` | no |
| allowed\_cidr\_list | Allowed IP addresses to access the server in CIDR format. Default to all Azure services | list(string) | `<list>` | no |
| client\_name |  | string | n/a | yes |
| create\_databases\_users | True to create a user named <db>_user per database with generated password and role db_owner. | string | `"true"` | no |
| daily\_backup\_retention | Retention in days for the databases backup. Value can be 7, 14, 21, 28 or 35. | number | `"35"` | no |
| database\_max\_dtu\_capacity | The maximum capacity any one database can consume in the Elastic Pool. Default to the max Elastic Pool capacity. | string | `""` | no |
| database\_min\_dtu\_capacity | The minimum capacity all databases are guaranteed in the Elastic Pool. Defaults to 0. | string | `"0"` | no |
| databases\_collation | SQL Collation for the databases | string | `"SQL_LATIN1_GENERAL_CP1_CI_AS"` | no |
| databases\_extra\_tags | Extra tags to add on the SQL databases | map(string) | `<map>` | no |
| databases\_names | Names of the databases to create for this server | list(string) | n/a | yes |
| elastic\_pool\_custom\_name | Name of the SQL Elastic Pool, generated if not set. | string | `""` | no |
| elastic\_pool\_extra\_tags | Extra tags to add on the SQL Elastic Pool | map(string) | `<map>` | no |
| elastic\_pool\_max\_size | Maximum size of the Elastic Pool in gigabytes | string | n/a | yes |
| enable\_advanced\_data\_security | Boolean flag to enable Advanced Data Security. The cost of ADS is aligned with Azure Security Center standard tier pricing. See https://docs.microsoft.com/en-us/azure/sql-database/sql-database-advanced-data-security | string | `"false"` | no |
| enable\_advanced\_data\_security\_admin\_emails | Boolean flag to define if account administrators should be emailed with Advanced Data Security alerts. | string | `"false"` | no |
| enable\_logging | Boolean flag to specify whether logging is enabled | bool | `"true"` | no |
| environment |  | string | n/a | yes |
| extra\_tags | Extra tags to add | map(string) | `<map>` | no |
| location | Azure location for SQL Server. | string | n/a | yes |
| location\_short | Short string for Azure location. | string | n/a | yes |
| logs\_log\_analytics\_workspace\_id | Log Analytics Workspace id for logs | string | `"null"` | no |
| logs\_storage\_account\_id | Storage Account id for logs | string | `"null"` | no |
| logs\_storage\_retention | Retention in days for logs on Storage Account | string | `"30"` | no |
| monthly\_backup\_retention | Retention in months for the monthly databases backup. | number | `"3"` | no |
| name\_prefix | Optional prefix for the generated name | string | `""` | no |
| resource\_group\_name |  | string | n/a | yes |
| server\_custom\_name | Name of the SQL Server, generated if not set. | string | `""` | no |
| server\_extra\_tags | Extra tags to add on SQL Server | map(string) | `<map>` | no |
| server\_version | Version of the SQL Server. Valid values are: 2.0 (for v11 server) and 12.0 (for v12 server). See https://www.terraform.io/docs/providers/azurerm/r/sql_server.html#version | string | `"12.0"` | no |
| sku | SKU for the Elastic Pool with tier and eDTUs capacity. Premium tier with zone redundancy is mandatory for high availability.     Possible values for tier are "Basic", "Standard", or "Premium". Example {tier="Standard", capacity="50"}.     See https://docs.microsoft.com/en-us/azure/sql-database/sql-database-dtu-resource-limits-elastic-pools" | map(string) | n/a | yes |
| stack |  | string | n/a | yes |
| weekly\_backup\_retention | Retention in weeks for the weekly databases backup. | number | `"0"` | no |
| yearly\_backup\_retention | Retention in years for the yearly backup. | number | `"0"` | no |
| yearly\_backup\_time | Week number taken in account for the yearly backup retention. | number | `"52"` | no |
| zone\_redundant | Whether or not the Elastic Pool is zone redundant, SKU tier must be Premium to use it. This is mandatory for high availability. | string | `"false"` | no |

## Outputs

| Name | Description |
|------|-------------|
| databases\_users | List of usernames of created users corresponding to input databases names. |
| databases\_users\_passwords | List of passwords of created users corresponding to input databases names. |
| default\_administrator\_databases\_connection\_strings | Connection strings of the SQL Databases with administrator credentials |
| sql\_databases\_creation\_date | Creation date of the SQL Databases |
| sql\_databases\_default\_secondary\_location | The default secondary location of the SQL Databases |
| sql\_databases\_id | Id of the SQL Databases |
| sql\_elastic\_pool\_id | Id of the SQL Elastic Pool |
| sql\_server\_fqdn | Fully qualified domain name of the SQL Server |
| sql\_server\_id | Id of the SQL Server |

## Related documentation

Terraform SQL Server documentation: [https://www.terraform.io/docs/providers/azurerm/r/sql_server.html]

Terraform SQL Database documentation: [https://www.terraform.io/docs/providers/azurerm/r/sql_database.html]

Terraform SQL Elastic Pool documentation: [https://www.terraform.io/docs/providers/azurerm/r/mssql_elasticpool.html]

Microsoft Azure root documentation: [https://docs.microsoft.com/en-us/azure/sql-database/]
