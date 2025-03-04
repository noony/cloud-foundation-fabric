# Cloud SQL instance with read replicas

This module manages the creation of Cloud SQL instances with potential read replicas in other regions. It can also create an initial set of users and databases via the `users` and `databases` parameters.

Note that this module assumes that some options are the same for both the primary instance and all the replicas (e.g. tier, disks, labels, flags, etc).

*Warning:* if you use the `users` field, you terraform state will contain each user's password in plain text.

## Simple example

This example shows how to setup a project, VPC and a standalone Cloud SQL instance.

```hcl
module "project" {
  source          = "./fabric/modules/project"
  billing_account = var.billing_account_id
  parent          = var.organization_id
  name            = "my-db-project"
  services = [
    "servicenetworking.googleapis.com"
  ]
}

module "vpc" {
  source     = "./fabric/modules/net-vpc"
  project_id = module.project.project_id
  name       = "my-network"
  psa_config = {
    ranges = { cloud-sql = "10.60.0.0/16" }
  }
}

module "db" {
  source     = "./fabric/modules/cloudsql-instance"
  project_id = module.project.project_id
  network_config = {
    connectivity = {
      psa_config = {
        private_network = module.vpc.self_link
      }
    }
  }
  name             = "db"
  region           = "europe-west1"
  database_version = "POSTGRES_13"
  tier             = "db-g1-small"
}
# tftest modules=3 resources=11 inventory=simple.yaml
```

## Cross-regional read replica

```hcl
module "db" {
  source     = "./fabric/modules/cloudsql-instance"
  project_id = var.project_id
  network_config = {
    connectivity = {
      psa_config = {
        private_network = var.vpc.self_link
      }
    }
  }
  prefix           = "myprefix"
  name             = "db"
  region           = "europe-west1"
  database_version = "POSTGRES_13"
  tier             = "db-g1-small"

  replicas = {
    replica1 = { region = "europe-west3", encryption_key_name = null }
    replica2 = { region = "us-central1", encryption_key_name = null }
  }
}
# tftest modules=1 resources=3 inventory=replicas.yaml
```

## Custom flags, databases and users

```hcl
module "db" {
  source     = "./fabric/modules/cloudsql-instance"
  project_id = var.project_id
  network_config = {
    connectivity = {
      psa_config = {
        private_network = var.vpc.self_link
      }
    }
  }
  name             = "db"
  region           = "europe-west1"
  database_version = "MYSQL_8_0"
  tier             = "db-g1-small"

  flags = {
    disconnect_on_expired_password = "on"
  }

  databases = [
    "people",
    "departments"
  ]

  users = {
    # generatea password for user1
    user1 = {
      password = null
    }
    # assign a password to user2
    user2 = {
      password = "mypassword"
    }
  }
}
# tftest modules=1 resources=6 inventory=custom.yaml
```

### CMEK encryption
```hcl

module "project" {
  source          = "./fabric/modules/project"
  billing_account = var.billing_account_id
  parent          = var.organization_id
  name            = "my-db-project"
  services = [
    "servicenetworking.googleapis.com",
    "sqladmin.googleapis.com",
  ]
}

module "kms" {
  source     = "./fabric/modules/kms"
  project_id = module.project.project_id
  keyring = {
    name     = "keyring"
    location = var.region
  }
  keys = {
    key-sql = {
      iam = {
        "roles/cloudkms.cryptoKeyEncrypterDecrypter" = [
          "serviceAccount:${module.project.service_accounts.robots.sqladmin}"
        ]
      }
    }
  }
}

module "db" {
  source              = "./fabric/modules/cloudsql-instance"
  project_id          = module.project.project_id
  encryption_key_name = module.kms.keys["key-sql"].id
  network_config = {
    connectivity = {
      psa_config = {
        private_network = var.vpc.self_link
      }
    }
  }
  name             = "db"
  region           = var.region
  database_version = "POSTGRES_13"
  tier             = "db-g1-small"
}

# tftest modules=3 resources=10
```

### Instance with PSC enabled

```hcl
module "db" {
  source     = "./fabric/modules/cloudsql-instance"
  project_id = var.project_id
  network_config = {
    connectivity = {
      psc_allowed_consumer_projects = ["my-project-id"]
    }
  }
  prefix            = "myprefix"
  name              = "db"
  region            = "europe-west1"
  availability_type = "REGIONAL"
  database_version  = "POSTGRES_13"
  tier              = "db-g1-small"
}
# tftest modules=1 resources=1
```

### Enable public IP

Use `ipv_enabled` to create instances with a public IP.

```hcl
module "db" {
  source     = "./fabric/modules/cloudsql-instance"
  project_id = var.project_id
  network_config = {
    connectivity = {
      public_ipv4 = true
      psa_config = {
        private_network = var.vpc.self_link
      }
    }
  }
  name             = "db"
  region           = "europe-west1"
  tier             = "db-g1-small"
  database_version = "MYSQL_8_0"
  replicas = {
    replica1 = { region = "europe-west3", encryption_key_name = null }
  }
}
# tftest modules=1 resources=2 inventory=public-ip.yaml
```

### Query Insights

Provide `insights_config` (can be just empty `{}`) to enable [Query Insights](https://cloud.google.com/sql/docs/postgres/using-query-insights)

```hcl
module "db" {
  source     = "./fabric/modules/cloudsql-instance"
  project_id = var.project_id
  network_config = {
    connectivity = {
      psa_config = {
        private_network = var.vpc.self_link
      }
    }
  }
  name             = "db"
  region           = "europe-west1"
  database_version = "POSTGRES_13"
  tier             = "db-g1-small"

  insights_config = {
    query_string_length = 2048
  }
}
# tftest modules=1 resources=1 inventory=insights.yaml
```
<!-- BEGIN TFDOC -->
## Variables

| name | description | type | required | default |
|---|---|:---:|:---:|:---:|
| [database_version](variables.tf#L68) | Database type and version to create. | <code>string</code> | ✓ |  |
| [name](variables.tf#L146) | Name of primary instance. | <code>string</code> | ✓ |  |
| [network_config](variables.tf#L151) | Network configuration for the instance. Only one between private_network and psc_config can be used. | <code title="object&#40;&#123;&#10;  authorized_networks &#61; optional&#40;map&#40;string&#41;&#41;&#10;  require_ssl         &#61; optional&#40;bool&#41;&#10;  connectivity &#61; object&#40;&#123;&#10;    public_ipv4 &#61; optional&#40;bool, false&#41;&#10;    psa_config &#61; optional&#40;object&#40;&#123;&#10;      private_network &#61; string&#10;      allocated_ip_ranges &#61; optional&#40;object&#40;&#123;&#10;        primary &#61; optional&#40;string&#41;&#10;        replica &#61; optional&#40;string&#41;&#10;      &#125;&#41;&#41;&#10;    &#125;&#41;&#41;&#10;    psc_allowed_consumer_projects &#61; optional&#40;list&#40;string&#41;&#41;&#10;  &#125;&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> | ✓ |  |
| [project_id](variables.tf#L190) | The ID of the project where this instances will be created. | <code>string</code> | ✓ |  |
| [region](variables.tf#L195) | Region of the primary instance. | <code>string</code> | ✓ |  |
| [tier](variables.tf#L215) | The machine type to use for the instances. | <code>string</code> | ✓ |  |
| [activation_policy](variables.tf#L16) | This variable specifies when the instance should be active. Can be either ALWAYS, NEVER or ON_DEMAND. Default is ALWAYS. | <code>string</code> |  | <code>&#34;ALWAYS&#34;</code> |
| [availability_type](variables.tf#L27) | Availability type for the primary replica. Either `ZONAL` or `REGIONAL`. | <code>string</code> |  | <code>&#34;ZONAL&#34;</code> |
| [backup_configuration](variables.tf#L33) | Backup settings for primary instance. Will be automatically enabled if using MySQL with one or more replicas. | <code title="object&#40;&#123;&#10;  enabled                        &#61; optional&#40;bool, false&#41;&#10;  binary_log_enabled             &#61; optional&#40;bool, false&#41;&#10;  start_time                     &#61; optional&#40;string, &#34;23:00&#34;&#41;&#10;  location                       &#61; optional&#40;string&#41;&#10;  log_retention_days             &#61; optional&#40;number, 7&#41;&#10;  point_in_time_recovery_enabled &#61; optional&#40;bool&#41;&#10;  retention_count                &#61; optional&#40;number, 7&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  enabled                        &#61; false&#10;  binary_log_enabled             &#61; false&#10;  start_time                     &#61; &#34;23:00&#34;&#10;  location                       &#61; null&#10;  log_retention_days             &#61; 7&#10;  point_in_time_recovery_enabled &#61; null&#10;  retention_count                &#61; 7&#10;&#125;">&#123;&#8230;&#125;</code> |
| [collation](variables.tf#L56) | The name of server instance collation. | <code>string</code> |  | <code>null</code> |
| [connector_enforcement](variables.tf#L62) | Specifies if connections must use Cloud SQL connectors. | <code>string</code> |  | <code>null</code> |
| [databases](variables.tf#L73) | Databases to create once the primary instance is created. | <code>list&#40;string&#41;</code> |  | <code>null</code> |
| [deletion_protection](variables.tf#L79) | Prevent terraform from deleting instances. | <code>bool</code> |  | <code>true</code> |
| [deletion_protection_enabled](variables.tf#L86) | Set Google's deletion protection attribute which applies across all surfaces (UI, API, & Terraform). | <code>bool</code> |  | <code>true</code> |
| [disk_autoresize_limit](variables.tf#L93) | The maximum size to which storage capacity can be automatically increased. The default value is 0, which specifies that there is no limit. | <code>number</code> |  | <code>0</code> |
| [disk_size](variables.tf#L99) | Disk size in GB. Set to null to enable autoresize. | <code>number</code> |  | <code>null</code> |
| [disk_type](variables.tf#L105) | The type of data disk: `PD_SSD` or `PD_HDD`. | <code>string</code> |  | <code>&#34;PD_SSD&#34;</code> |
| [edition](variables.tf#L111) | The edition of the instance, can be ENTERPRISE or ENTERPRISE_PLUS. | <code>string</code> |  | <code>&#34;ENTERPRISE&#34;</code> |
| [encryption_key_name](variables.tf#L117) | The full path to the encryption key used for the CMEK disk encryption of the primary instance. | <code>string</code> |  | <code>null</code> |
| [flags](variables.tf#L123) | Map FLAG_NAME=>VALUE for database-specific tuning. | <code>map&#40;string&#41;</code> |  | <code>null</code> |
| [insights_config](variables.tf#L129) | Query Insights configuration. Defaults to null which disables Query Insights. | <code title="object&#40;&#123;&#10;  query_string_length     &#61; optional&#40;number, 1024&#41;&#10;  record_application_tags &#61; optional&#40;bool, false&#41;&#10;  record_client_address   &#61; optional&#40;bool, false&#41;&#10;  query_plans_per_minute  &#61; optional&#40;number, 5&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [labels](variables.tf#L140) | Labels to be attached to all instances. | <code>map&#40;string&#41;</code> |  | <code>null</code> |
| [postgres_client_certificates](variables.tf#L174) | Map of cert keys connect to the application(s) using public IP. | <code>list&#40;string&#41;</code> |  | <code>null</code> |
| [prefix](variables.tf#L180) | Optional prefix used to generate instance names. | <code>string</code> |  | <code>null</code> |
| [replicas](variables.tf#L200) | Map of NAME=> {REGION, KMS_KEY} for additional read replicas. Set to null to disable replica creation. | <code title="map&#40;object&#40;&#123;&#10;  region              &#61; string&#10;  encryption_key_name &#61; string&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [root_password](variables.tf#L209) | Root password of the Cloud SQL instance. Required for MS SQL Server. | <code>string</code> |  | <code>null</code> |
| [users](variables.tf#L220) | Map of users to create in the primary instance (and replicated to other replicas). For MySQL, anything afterr the first `@` (if persent) will be used as the user's host. Set PASSWORD to null if you want to get an autogenerated password. The user types available are: 'BUILT_IN', 'CLOUD_IAM_USER' or 'CLOUD_IAM_SERVICE_ACCOUNT'. | <code title="map&#40;object&#40;&#123;&#10;  password &#61; optional&#40;string&#41;&#10;  type     &#61; optional&#40;string&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>null</code> |

## Outputs

| name | description | sensitive |
|---|---|:---:|
| [connection_name](outputs.tf#L24) | Connection name of the primary instance. |  |
| [connection_names](outputs.tf#L29) | Connection names of all instances. |  |
| [dns_name](outputs.tf#L37) | The dns name of the instance. |  |
| [dns_names](outputs.tf#L42) | Dns names of all instances. |  |
| [id](outputs.tf#L50) | Fully qualified primary instance id. |  |
| [ids](outputs.tf#L55) | Fully qualified ids of all instances. |  |
| [instances](outputs.tf#L63) | Cloud SQL instance resources. | ✓ |
| [ip](outputs.tf#L69) | IP address of the primary instance. |  |
| [ips](outputs.tf#L74) | IP addresses of all instances. |  |
| [name](outputs.tf#L82) | Name of the primary instance. |  |
| [names](outputs.tf#L87) | Names of all instances. |  |
| [postgres_client_certificates](outputs.tf#L95) | The CA Certificate used to connect to the SQL Instance via SSL. | ✓ |
| [psc_service_attachment_link](outputs.tf#L101) | The link to service attachment of PSC instance. |  |
| [psc_service_attachment_links](outputs.tf#L106) | Links to service attachment of PSC instances. |  |
| [self_link](outputs.tf#L114) | Self link of the primary instance. |  |
| [self_links](outputs.tf#L119) | Self links of all instances. |  |
| [user_passwords](outputs.tf#L127) | Map of containing the password of all users created through terraform. | ✓ |
<!-- END TFDOC -->
