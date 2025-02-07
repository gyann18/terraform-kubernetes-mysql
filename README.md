## MySQL DB Terraform Module


<br>
This module simplifies deploying a <strong>MySQL database</strong> on Kubernetes with flexible configuration options, including storage class, volume sizes, and architecture. It supports <strong>backups</strong>, <strong>restores</strong>, and deploying exporters for <strong>Grafana metrics</strong>.<br><br> It can create namespaces and configure recovery windows for <strong>AWS Secrets Manager</strong>, <strong>Azure Key Vault</strong>, and <strong>GCP Secrets Manager</strong>. This module enables easy deployment of highly available <strong>MySQL databases</strong> on <strong>AWS EKS</strong>, <strong>Azure AKS</strong>, and <strong>GCP GKE</strong> with extensive customization options.

## Supported Versions:

|  MysqlDB Helm Chart Version    |     K8s supported version (EKS, AKS & GKE)  |  
| :-----:                       |         :---                |
| **9.2.0**                     |    **1.23,1.24,1.25,1.26,1.27**           |


## Usage Example

```hcl
locals {
  name        = "mysql"
  region      = "us-east-2"
  environment = "prod"
  additional_tags = {
    Owner      = "organization_name"
    Expires    = "Never"
    Department = "Engineering"
  }
  create_namespace                   = true
  namespace                          = "mysql"
  store_password_to_secret_manager   = false
  mysqldb_custom_credentials_enabled = true
  mysqldb_custom_credentials_config = {
    root_user            = "root"
    root_password        = "RJDRIFsYC8ZS1WQuV0ps"
    custom_username      = "admin"
    custom_user_password = "NCPFUKEMd7rrWuvMAa73"
    replication_user     = "replicator"
    replication_password = "nvAHhm1uGQNYWVw6ZyAH"
    exporter_user        = "mysqld_exporter"
    exporter_password    = "ZawhvpueAehRdKFlbjaq"
  }
  custom_user_username = "custom"
}

module "aws" {
  source                             = "saturnops/mysql/kubernetes//modules/resources/aws"
  cluster_name                       = "prod-eks"
  environment                        = "prod"
  name                               = "mysql"
  namespace                          = local.namespace
  store_password_to_secret_manager   = true
  mysqldb_custom_credentials_enabled = true
  mysqldb_custom_credentials_config  = {
    root_user            = "root"
    root_password        = "RJDRIFsYC8ZS1WQuV0ps"
    custom_username      = "admin"
    custom_user_password = "NCPFUKEMd7rrWuvMAa73"
    replication_user     = "replicator"
    replication_password = "nvAHhm1uGQNYWVw6ZyAH"
    exporter_user        = "mysqld_exporter"
    exporter_password    = "ZawhvpueAehRdKFlbjaq"
  }
  custom_user_username               = mysqldb_custom_credentials_enabled ? "" : "custome_username"
}

module "mysql" {
  source           = "saturnops/mysql/kubernetes"
  create_namespace = local.create_namespace
  namespace        = local.namespace
  mysqldb_config = {
    name                             = "mysql"
    app_version                      = "8.0.29-debian-11-r9"
    environment                      = "prod"
    values_yaml                      = ""
    architecture                     = "replication"
    custom_database                  = "test_db"
    storage_class_name               = "gp2"
    custom_user_username             = local.mysqldb_custom_credentials_enabled ? "" : local.custom_user_username
    primary_db_volume_size           = "10Gi"
    secondary_db_volume_size         = "10Gi"
    secondary_db_replica_count       = 2
    store_password_to_secret_manager = true
  }
  mysqldb_custom_credentials_enabled = local.mysqldb_custom_credentials_enabled
  mysqldb_custom_credentials_config  = local.mysqldb_custom_credentials_config
  root_password                      = local.mysqldb_custom_credentials_enabled ? "" : module.aws.root_password
  metric_exporter_pasword            = local.mysqldb_custom_credentials_enabled ? "" : module.aws.metric_exporter_pasword
  mysqldb_replication_user_password  = local.mysqldb_custom_credentials_enabled ? "" : module.aws.mysqldb_replication_user_password
  custom_user_password               = local.mysqldb_custom_credentials_enabled ? "" : module.aws.custom_user_password
  bucket_provider_type               = "s3"
  iam_role_arn_backup                = module.aws.iam_role_arn_backup
  mysqldb_backup_enabled             = true
  mysqldb_backup_config = {
    mysql_database_name  = ""
    bucket_uri           = "s3://bucket_name"
    s3_bucket_region     = ""
    cron_for_full_backup = "*/5 * * * *"
  }
  mysqldb_restore_enabled = true
  iam_role_arn_restore    = module.aws.iam_role_arn_restore
  mysqldb_restore_config = {
    bucket_uri       = "s3://bucket_name/mysqldump_20230710_120501.zip"
    file_name        = "mysqldump_20230710_120501.zip"
    s3_bucket_region = ""
  }
  mysqldb_exporter_enabled = true
}


```
- Refer [AWS examples](https://github.com/saturnops/terraform-kubernetes-mysql/tree/main/examples/complete/aws) for more details.
- Refer [Azure examples](https://github.com/saturnops/terraform-kubernetes-mysql/tree/main/examples/complete/azure) for more details.
- Refer [GCP examples](https://github.com/saturnops/terraform-kubernetes-mysql/tree/main/examples/complete/gcp) for more details.

## IAM Permissions
The required IAM permissions to create resources from this module can be found [here](https://github.com/saturnops/terraform-kubernetes-mysql/blob/main/IAM.md)

## MySQL Backup and Restore
This module provides functionality to automate the backup and restore process for MySQL databases using AWS S3 buckets. It allows users to easily schedule backups, restore databases from backups stored in S3, and manage access permissions using AWS IAM roles.
Features
### Backup
- Users can schedule full backups.
- upports specifying individual database names for backup or backing up all databases except system databases.
- Backups are stored in specified S3 buckets.
### Restore
- Users can restore MySQL databases from backups stored in S3 buckets.
- Supports specifying the backup file to restore from and the target S3 bucket region.
### IAM Role for Permissions
- Users need to provide an IAM role for the module to access the specified S3 bucket and perform backup and restore operations.
## Module Inputs
### Backup Configuration
- command using to do backup:
```
mysqldump -h$HOST -u$USER -p$PASSWORD --databases db_name > full-backup.sql
```
- mysql_database_name: The name of the MySQL database to backup. Leave blank to backup all databases except system databases.
- bucket_uri: The URI of the S3 bucket where backups will be stored.
- s3_bucket_region: The region of the S3 bucket.
- cron_for_full_backup: The cron expression for scheduling full backups.
### Restore Configuration
- mysqldb_restore_config: Configuration for restoring databases.bucket_uri: The URI of the S3 bucket containing the backup file.
- file_name: The name of the backup file to restore.
- s3_bucket_region: The region of the S3 bucket containing the backup file.
## Important Notes
  1. In order to enable the exporter, it is required to deploy Prometheus/Grafana first.
  2. The exporter is a tool that extracts metrics data from an application or system and makes it available to be scraped by Prometheus.
  3. Prometheus is a monitoring system that collects metrics data from various sources, including exporters, and stores it in a time-series database.
  4. Grafana is a data visualization and dashboard tool that works with Prometheus and other data sources to display the collected metrics in a user-friendly way.
  5. To deploy Prometheus/Grafana, please follow the installation instructions for each tool in their respective documentation.
  6. Once Prometheus and Grafana are deployed, the exporter can be configured to scrape metrics data from your application or system and send it to Prometheus.
  7. Finally, you can use Grafana to create custom dashboards and visualize the metrics data collected by Prometheus.
  8. This module is compatible with EKS, AKS & GKE which is great news for users deploying the module on an AWS, Azure & GCP cloud. Review the module's documentation, meet specific configuration requirements, and test thoroughly after deployment to ensure everything works as expected.
<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## Requirements

No requirements.

## Providers

| Name | Version |
|------|---------|
| <a name="provider_helm"></a> [helm](#provider\_helm) | n/a |
| <a name="provider_kubernetes"></a> [kubernetes](#provider\_kubernetes) | n/a |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [helm_release.mysqldb](https://registry.terraform.io/providers/hashicorp/helm/latest/docs/resources/release) | resource |
| [helm_release.mysqldb_backup](https://registry.terraform.io/providers/hashicorp/helm/latest/docs/resources/release) | resource |
| [helm_release.mysqldb_restore](https://registry.terraform.io/providers/hashicorp/helm/latest/docs/resources/release) | resource |
| [kubernetes_namespace.mysqldb](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/namespace) | resource |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_app_version"></a> [app\_version](#input\_app\_version) | Version of the MySQL application that will be deployed. | `string` | `"8.0.29-debian-11-r9"` | no |
| <a name="input_azure_container_name"></a> [azure\_container\_name](#input\_azure\_container\_name) | Azure container name | `string` | `""` | no |
| <a name="input_azure_storage_account_key"></a> [azure\_storage\_account\_key](#input\_azure\_storage\_account\_key) | Azure storage account key | `string` | `""` | no |
| <a name="input_azure_storage_account_name"></a> [azure\_storage\_account\_name](#input\_azure\_storage\_account\_name) | Azure storage account name | `string` | `""` | no |
| <a name="input_bucket_provider_type"></a> [bucket\_provider\_type](#input\_bucket\_provider\_type) | Choose what type of provider you want (s3, gcs) | `string` | `"gcs"` | no |
| <a name="input_chart_version"></a> [chart\_version](#input\_chart\_version) | Version of the Mysql chart that will be used to deploy MySQL application. | `string` | `"9.2.0"` | no |
| <a name="input_cluster_name"></a> [cluster\_name](#input\_cluster\_name) | Specifies the name of the EKS cluster to deploy the MySQL application on. | `string` | `""` | no |
| <a name="input_create_namespace"></a> [create\_namespace](#input\_create\_namespace) | Specify whether or not to create the namespace if it does not already exist. Set it to true to create the namespace. | `string` | `true` | no |
| <a name="input_custom_user_password"></a> [custom\_user\_password](#input\_custom\_user\_password) | custom user password for MongoDB | `string` | `""` | no |
| <a name="input_iam_role_arn_backup"></a> [iam\_role\_arn\_backup](#input\_iam\_role\_arn\_backup) | IAM role ARN for backup (AWS) | `string` | `""` | no |
| <a name="input_iam_role_arn_restore"></a> [iam\_role\_arn\_restore](#input\_iam\_role\_arn\_restore) | IAM role ARN for restore (AWS) | `string` | `""` | no |
| <a name="input_metric_exporter_pasword"></a> [metric\_exporter\_pasword](#input\_metric\_exporter\_pasword) | Metric exporter password for MongoDB | `string` | `""` | no |
| <a name="input_mysqldb_backup_config"></a> [mysqldb\_backup\_config](#input\_mysqldb\_backup\_config) | configuration options for MySQL database backups. It includes properties such as the S3 bucket URI, the S3 bucket region, cron expression for full backups and the database name to take backup of particular database or if send empty it backup whole database | `any` | <pre>{<br>  "bucket_uri": "",<br>  "cron_for_full_backup": "",<br>  "mysql_database_name": "",<br>  "s3_bucket_region": ""<br>}</pre> | no |
| <a name="input_mysqldb_backup_enabled"></a> [mysqldb\_backup\_enabled](#input\_mysqldb\_backup\_enabled) | Specifies whether to enable backups for MySQL database. | `bool` | `false` | no |
| <a name="input_mysqldb_config"></a> [mysqldb\_config](#input\_mysqldb\_config) | Specify the configuration settings for MySQL, including the name, environment, storage options, replication settings, and custom YAML values. | `any` | <pre>{<br>  "architecture": "",<br>  "custom_database": "",<br>  "custom_user_username": "",<br>  "environment": "",<br>  "name": "",<br>  "primary_db_volume_size": "",<br>  "secondary_db_replica_count": 1,<br>  "secondary_db_volume_size": "",<br>  "storage_class_name": "",<br>  "store_password_to_secret_manager": true,<br>  "values_yaml": ""<br>}</pre> | no |
| <a name="input_mysqldb_custom_credentials_config"></a> [mysqldb\_custom\_credentials\_config](#input\_mysqldb\_custom\_credentials\_config) | Specify the configuration settings for MySQL to pass custom credentials during creation | `any` | <pre>{<br>  "custom_user_password": "",<br>  "custom_username": "",<br>  "exporter_password": "",<br>  "exporter_user": "",<br>  "replication_password": "",<br>  "replication_user": "",<br>  "root_password": "",<br>  "root_user": ""<br>}</pre> | no |
| <a name="input_mysqldb_custom_credentials_enabled"></a> [mysqldb\_custom\_credentials\_enabled](#input\_mysqldb\_custom\_credentials\_enabled) | Specifies whether to enable custom credentials for MySQL database. | `bool` | `false` | no |
| <a name="input_mysqldb_exporter_enabled"></a> [mysqldb\_exporter\_enabled](#input\_mysqldb\_exporter\_enabled) | Specify whether or not to deploy Mysql exporter to collect Mysql metrics for monitoring in Grafana. | `bool` | `false` | no |
| <a name="input_mysqldb_replication_user_password"></a> [mysqldb\_replication\_user\_password](#input\_mysqldb\_replication\_user\_password) | Replicator password for MongoDB | `string` | `""` | no |
| <a name="input_mysqldb_restore_config"></a> [mysqldb\_restore\_config](#input\_mysqldb\_restore\_config) | Configuration options for restoring dump to the MySQL database. | `any` | <pre>{<br>  "bucket_uri": "",<br>  "file_name": "",<br>  "s3_bucket_region": ""<br>}</pre> | no |
| <a name="input_mysqldb_restore_enabled"></a> [mysqldb\_restore\_enabled](#input\_mysqldb\_restore\_enabled) | Specifies whether to enable restoring dump to the MySQL database. | `bool` | `false` | no |
| <a name="input_namespace"></a> [namespace](#input\_namespace) | Name of the Kubernetes namespace where the MYSQL deployment will be deployed. | `string` | `"mysqldb"` | no |
| <a name="input_project_id"></a> [project\_id](#input\_project\_id) | Google Cloud project ID | `string` | `""` | no |
| <a name="input_recovery_window_aws_secret"></a> [recovery\_window\_aws\_secret](#input\_recovery\_window\_aws\_secret) | Number of days that AWS Secrets Manager will wait before deleting a secret. This value can be set to 0 to force immediate deletion, or to a value between 7 and 30 days to allow for recovery. | `number` | `0` | no |
| <a name="input_resource_group_location"></a> [resource\_group\_location](#input\_resource\_group\_location) | Azure region | `string` | `"East US"` | no |
| <a name="input_resource_group_name"></a> [resource\_group\_name](#input\_resource\_group\_name) | Azure Resource Group name | `string` | `""` | no |
| <a name="input_root_password"></a> [root\_password](#input\_root\_password) | Root password for MongoDB | `string` | `""` | no |
| <a name="input_service_account_backup"></a> [service\_account\_backup](#input\_service\_account\_backup) | Service account for backup (GCP) | `string` | `""` | no |
| <a name="input_service_account_restore"></a> [service\_account\_restore](#input\_service\_account\_restore) | Service account for restore (GCP) | `string` | `""` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_mysqldb_credential"></a> [mysqldb\_credential](#output\_mysqldb\_credential) | MySQL credentials used for accessing the MySQL database. |
| <a name="output_mysqldb_endpoints"></a> [mysqldb\_endpoints](#output\_mysqldb\_endpoints) | MySQL endpoints in the Kubernetes cluster. |
<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->






##           





Please give our GitHub repository a ⭐️ to show your support and increase its visibility.





