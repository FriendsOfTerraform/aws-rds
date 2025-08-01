# Relational Database Service Module

This module will build and configure an [RDS](https://aws.amazon.com/rds/) instance or [Aurora](https://aws.amazon.com/rds/aurora/) cluster with additional readers

**This repository is a READ-ONLY sub-tree split**. See https://github.com/FriendsOfTerraform/modules to create issues or submit pull requests.

## Table of Contents

- [Example Usage](#example-usage)
    - [Basic Usage](#basic-usage)
    - [Multi-AZ Instance](#multi-az-instance)
    - [Multi-AZ Cluster](#multi-az-cluster)
    - [Aurora Regional Cluster](#aurora-regional-cluster)
    - [Aurora Global Cluster](#aurora-global-cluster)
    - [RDS Proxies](#rds-proxies)
    - [Cloudwatch Alarms](#cloudwatch-alarms)
- [Argument Reference](#argument-reference)
    - [Mandatory](#mandatory)
    - [Optional](#optional)
- [Outputs](#outputs)

## Example Usage

### Basic Usage

```terraform
module "rds_demo" {
  source = "github.com/FriendsOfTerraform/aws-rds.git?ref=v2.1.0"

  engine = {
    type    = "mysql"
    version = "8.0.34"
  }

  name = "singleinstance"

  authentication_config = {
    db_master_account = {
      username                           = "admin"
      manage_password_in_secrets_manager = true
    }
  }

  instance_class = "db.m5d.large"

  storage_config = {
    type              = "gp3"
    allocated_storage = 200
  }

  networking_config = {
    db_subnet_group_name = "db-subnet-group"
    security_group_ids   = ["sg-00ce17012345abcde"]
  }

  db_name = "demo"
}
```

### Multi-AZ Instance

```terraform
module "multiazinstance_demo" {
  source = "github.com/FriendsOfTerraform/aws-rds.git?ref=v2.1.0"

  engine = {
    type    = "mysql"
    version = "8.0.34"
  }

  deployment_option = "MultiAZInstance"
  name              = "multiazinstance"

  authentication_config = {
    db_master_account = {
      username                           = "admin"
      manage_password_in_secrets_manager = true
    }
  }

  instance_class = "db.m5d.large"

  storage_config = {
    type                  = "gp3"
    allocated_storage     = 2000
    max_allocated_storage = 10000
    provisioned_iops      = 12000
    storage_throughput    = 1000
  }

  networking_config = {
    db_subnet_group_name = "db-subnet-group"
    security_group_ids   = ["sg-00ce17012345abcde"]
  }

  monitoring_config = {
    enable_enhanced_monitoring = {
      interval = 60
    }

    enable_performance_insight = {
      retention_period = 7
    }
  }

  db_name = "demo"

  enable_automated_backup = {
    retention_period      = 7
    window                = "00:00-06:00" #PST 1700-2300
    copy_tags_to_snapshot = true
  }

  cloudwatch_log_exports = ["audit", "error", "general", "slowquery"]

  maintenance_config = {
    enable_auto_minor_version_upgrade = true
    window                            = "sat:07:00-sat:15:00" #PST saturday 0000 - 0800
  }
}
```

### Multi-AZ Cluster

```terraform
module "multiazcluster_demo" {
  source = "github.com/FriendsOfTerraform/aws-rds.git?ref=v2.1.0"

  engine = {
    type    = "mysql"
    version = "8.0.34"
  }

  deployment_option = "MultiAZCluster"
  name              = "multiazcluster-demo"

  authentication_config = {
    db_master_account = {
      username                           = "admin"
      manage_password_in_secrets_manager = true
    }
  }

  instance_class = "db.m5d.large"

  # Multi-AZ cluster only supports provisioned IOPS storage
  storage_config = {
    type              = "io1"
    allocated_storage = 400
    provisioned_iops  = 3000
  }

  networking_config = {
    db_subnet_group_name = "test-subnet-group"
    security_group_ids   = ["sg-00ce17012345abcde"]
  }
}
```

### Aurora Regional Cluster

```terraform
module "aurora_regional_demo" {
  source = "github.com/FriendsOfTerraform/aws-rds.git?ref=v2.1.0"

  engine = {
    type    = "aurora-mysql"
    version = "8.0.mysql_aurora.3.04.0"
  }

  name = "aurora-regional-demo"

  authentication_config = {
    db_master_account = {
      username                           = "admin"
      manage_password_in_secrets_manager = true
    }

    iam_database_authentication = {
      enabled = true

      # Creates IAM policies to allow connection to this RDS cluster
      # The name of the db users must already existed in the DB
      # IAM policies must be attached to an IAM principal
      create_iam_policies_for_db_users = ["peter", "jane"]
    }
  }

  # Manages multiple auto scaling policies
  auto_scaling_policies = {
    # The keys of the map are the policy names
    scale_by_cpu = {
      target_metric = {
        average_cpu_utilization_of_aurora_replicas = 60
      }
    }
    scale_by_number_of_connections = {
      target_metric = {
        average_connections_of_aurora_replicas = 100
      }
    }
  }

  instance_class = "db.t3.medium"

  networking_config = {
    db_subnet_group_name = "db-subnet-group"
    security_group_ids   = ["sg-00ce17012345abcde"]
  }

  db_name = "demo"

  enable_automated_backup = {
    retention_period      = 7
    window                = "00:00-06:00" #PST 1700-2300
    copy_tags_to_snapshot = true
  }

  cloudwatch_log_exports = ["audit", "error", "general", "slowquery"]

  maintenance_config = {
    window = "sat:07:00-sat:15:00" #PST saturday 0000 - 0800
  }

  cluster_instances = {
    # The key of the map will be the instance's name
    primary = {}
    secondary = {
      networking_config = { availability_zone = "us-east-1b" }
    }
  }
}
```

### Aurora Global Cluster

```terraform
module "aurora_global_demo" {
  source = "github.com/FriendsOfTerraform/aws-rds.git?ref=v2.1.0"

  # Creates a new global cluster
  aurora_global_cluster = {
    name = "global-cluster-demo"
  }

  engine = {
    type    = "aurora-mysql"
    version = "8.0.mysql_aurora.3.04.0"
  }

  name = "us-east-1-cluster"

  authentication_config = {
    db_master_account = {
      username                           = "admin"
      manage_password_in_secrets_manager = true
    }
  }

  instance_class = "db.serverless"

  serverless_capacity = {
    max_acus = 50
    min_acus = 20
  }

  networking_config = {
    db_subnet_group_name = "db-subnet-group"
    security_group_ids   = ["sg-00ce17012345abcde"]
  }

  db_name = "demo"

  enable_automated_backup = {
    retention_period      = 7
    window                = "00:00-06:00" #PST 1700-2300
    copy_tags_to_snapshot = true
  }

  cloudwatch_log_exports = ["audit", "error", "general", "slowquery"]

  maintenance_config = {
    window = "sat:07:00-sat:15:00" #PST saturday 0000 - 0800
  }

  cluster_instances = {
    # The key of the map will be the instance's name
    primary   = {}
    secondary = {}
  }
}
```

### RDS Proxies

```terraform
module "rds_proxies" {
  source = "github.com/FriendsOfTerraform/aws-rds.git?ref=v2.1.0"

  name           = "demo-db"
  instance_class = "db.t4g.medium"

  engine = {
    type    = "postgres"
    version = "14.17"
  }

  authentication_config = {
    db_master_account = {
      username                           = "postgres"
      manage_password_in_secrets_manager = true
    }
  }

  networking_config = {
    db_subnet_group_name = "default"
    security_group_ids   = ["sg-01230e2abcdef"]
    ca_cert_identifier   = "rds-ca-rsa2048-g1"
  }

  proxies = {
    # The keys of the map are the proxies' name
    demo-proxy = {
      security_group_ids = ["sg-04e232731f6abcdef"]
      subnet_ids         = ["subnet-abcdef012345", "subnet-543210fedcba"]

      # Manages multiple authentications
      authentications = {
        # The keys of the map are secrets manager arn for the DB users
        "arn:aws:secretsmanager:us-east-1:111122223333:secret:demo-db-user" = { client_authentication_type = "POSTGRES_SCRAM_SHA_256" }
      }

      # You can create multiple additional endpoints beside the default one
      additional_endpoints = {
        # The keys of the map are the endpoints' name
        demo-proxy-read-only = { target_role = "READ_ONLY" }
      }
    }
  }
}
```

### Cloudwatch Alarms

```terraform
module "cloudwatch_alarms" {
  source = "github.com/FriendsOfTerraform/aws-rds.git?ref=v2.1.0"

  name                = "aurora-demo"
  db_name             = "demo"
  instance_class      = "db.t3.medium"
  skip_final_snapshot = true

  engine = {
    type    = "aurora-postgresql"
    version = "14.15"
  }

  cluster_instances = {
    primary = {
      failover_priority = 0

      monitoring_config = {
        cloudwatch_alarms = {
          # The key of the map are the alarms' name
          freeable-memory-anomaly = {
            metric_name            = "FreeableMemory"
            expression             = "average < 1000000000"
            notification_sns_topic = "arn:aws:sns:us-east-1:111122223333:email-admin"
          }

          cpu-utilization = {
            metric_name            = "CPUUtilization"
            expression             = "average >= 85"
            notification_sns_topic = "arn:aws:sns:us-east-1:111122223333:email-admin"
          }
        }
      }
    }

    secondary = {
      failover_priority = 1

      monitoring_config = {
        cloudwatch_alarms = {
          cpu-utilization = {
            metric_name            = "CPUUtilization"
            expression             = "average >= 85"
            notification_sns_topic = "arn:aws:sns:us-east-1:111122223333:email-admin"
          }
        }
      }
    }
  }

  authentication_config = {
    db_master_account = {
      username                           = "postgres"
      manage_password_in_secrets_manager = true
    }
  }

  networking_config = {
    db_subnet_group_name = "default"
    security_group_ids   = ["sg-01234593f7eabcdef"]
  }
}
```

## Argument Reference

### Mandatory

- (object) **`authentication_config`** _[since v1.0.0]_

    Configures RDS authentication methods

    - (object) **`db_master_account`** _[since v1.0.0]_

        Manages the DB master account

        - (string) **`username`** _[since v1.0.0]_

            Username for the master DB user

        - (string) **`customer_kms_key_id = null`** _[since v1.0.0]_

            Specify the KMS key to encrypt the master password in secrets manager. If not specified, the default KMS key for your AWS account is used. Used when `manage_password_in_secrets_manager = true`

        - (bool) **`manage_password_in_secrets_manager = false`** _[since v1.0.0]_

            Set to true to allow RDS to [manage the master user password in Secrets Manager][manage-password-in-secrets-manager]. Mutually exclusive with `password`. This feature does not support Aurora global cluster.

        - (string) **`password = null`** _[since v1.0.0]_

            Password for the master DB user. Mutually exclusive with `manage_password_in_secrets_manager`

    - (object) **`iam_database_authentication = null`** _[since v1.0.0]_

        Configures [AWS Identity and Access Management (IAM) accounts to database accounts][rds-iam-db-authentication]. Cannot be used when `deployment_option = "MultiAZCluster"`. Plesae refer to the following documentations for instruction to each DB engine.

        - [MySQL, MariaDB](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Connecting.AWSCLI.html)
        - [PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Connecting.AWSCLI.PostgreSQL.html)

      - (bool) **`enabled = true`** _[since v1.0.0]_

          Specify whether IAM DB authentication is enabled. [See example](#aurora-regional-cluster)

      - (list(string)) **`create_iam_policies_for_db_users = []`** _[since v1.0.0]_

          Specify a list of DB user names to create IAM policies for RDS IAM Authentication. This will allow an IAM principal such as an IAM role to request authentication token for the specific DB user. Please refer to [this documentation][rds-iam-authentication-policy] for more information.

- (object) **`engine`** _[since v1.0.0]_

    Configures RDS engine options

    - (string) **`type`** _[since v1.0.0]_

        Specify the engine type, This module currently supports: `"aurora-mysql"`, `"aurora-postgresql"`, `"mysql"`, `"postgres"`, `"mariadb`

    - (string) **`version`** _[since v1.0.0]_

        Specify the engine version. You can get a list of engine version with `aws rds describe-db-engine-versions --engine aurora-mysql --query DBEngineVersions[].[EngineVersion]`

- (string) **`instance_class`** _[since v1.0.0]_

    The compute and memory capacity of the DB instance, for example `"db.m5.large"`. For the full list of DB instance classes, please refer to [DB instance class][db-instance-class] and [Aurora DB instance class][aurora-db-instance-class]

- (string) **`name`** _[since v1.0.0]_

    Specify the name of the RDS instance or the RDS cluster

- (object) **`networking_config`** _[since v1.0.0]_

    Configures RDS connectivity options

    - (string) **`db_subnet_group_name`** _[since v1.0.0]_

        Name of DB subnet group. DB instance will be created in the VPC associated with the DB subnet group. A DB subnet group with at least three AZs must be specified if `deployment_option = "MultiAZCluster"`

    - (list(string)) **`security_group_ids`** _[since v1.0.0]_

        List of VPC security groups to associate to the RDS instance or cluster

    - (string) **`availability_zone = null`** _[since v1.0.0]_

        The availability zone to deploy the RDS instance in

    - (string) **`ca_cert_identifier = null`** _[since v1.0.0]_

        The certificate authority (CA) is the certificate that identifies the root CA at the top of the certificate chain. The CA signs the DB server certificate, which is installed on each DB instance. The DB server certificate identifies the DB instance as a trusted server. Please refer to [this documentation][rds-ca] for valid values. Defaults to `"rds-ca-2019"`. Refers to the following documentations for requirements to connect to each DB engine with SSL.

        - [MariaDB](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/ssl-certificate-rotation-mariadb.html)
        - [MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/ssl-certificate-rotation-mysql.html)
        - [PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Concepts.General.SSL.html)

    - (bool) **`enable_ipv6 = false`** _[since v1.0.0]_

        Specify whether the RDS instance or cluster supports IPv6

    - (bool) **`enable_public_access = false`** _[since v1.0.0]_

        Specify whether the RDS instance or cluster is publicly accessible

    - (string) **`port = null`** _[since v1.0.0]_

        Specify the port on which the DB accepts connections.

### Optional

- (map(string)) **`additional_tags = {}`** _[since v1.0.0]_

    Additional tags for the RDS instance or cluster

- (map(string)) **`additional_tags_all = {}`** _[since v1.0.0]_

    Additional tags for all resources deployed with this module

- (bool) **`apply_immediately = null`** _[since v1.0.0]_

    Specifies whether any database modifications are applied immediately, or during the next maintenance window. Using `apply_immediately` can result in a brief downtime as the server reboots.

- (object) **`aurora_global_cluster = null`** _[since v1.0.0]_

    Creates new or join existing Aurora Global cluster. Must be used with an `"aurora-*"` engine type

    - (string) **`join_existing_global_cluster = null`** _[since v1.0.0]_

        The name of an existing global Aurora cluster to join. Cannot be used with `name`

    - (string) **`name = null`** _[since v1.0.0]_

        Specify the name of the global cluster to be created. Cannot be used with `join_existing_global_cluster`

- (map(object)) **`auto_scaling_policies = {}`** _[since v2.0.0]_

    Manages multiple auto scaling policies. Only applicable to Aurora clusters. Please see [example](#aurora-regional-cluster)

    - (object) **`target_metric`** _[since v2.0.0]_

        The cloudwatch metric to monitor for scaling. Must specify one of the following.

      - (number) **`average_cpu_utilization_of_aurora_replicas = null`** _[since v2.0.0]_

          The average value of the CPUUtilization metric in CloudWatch across all Aurora Replicas in the Aurora DB cluster.

      - (number) **`average_connections_of_aurora_replicas = null`** _[since v2.0.0]_

          The average value of the DatabaseConnections metric in CloudWatch across all Aurora Replicas in the Aurora DB cluster.

    - (bool) **`enable_scale_in = true`** _[since v2.0.0]_

        Allow this Auto Scaling policy to remove Aurora Replicas. Aurora Replicas created by you are not removed by Auto Scaling.

    - (number) **`maximum_capacity = 15`** _[since v2.0.0]_

        Specify the maximum number of Aurora Replicas to maintain. Up to 15 Aurora Replicas are supported.

    - (number) **`minimum_capacity = 1`** _[since v2.0.0]_

        Specify the minimum number of Aurora Replicas to maintain.

    - (string) **`scale_in_cooldown_period = "5 minutes"`** _[since v2.0.0]_

        Specify the number of seconds to wait between scale-in actions.

    - (string) **`scale_out_cooldown_period = "5 minutes"`** _[since v2.0.0]_

        Specify the number of seconds to wait between scale-out actions.

- (list(string)) **`cloudwatch_log_exports = null`** _[since v1.0.0]_

    Set of log types to enable for exporting to CloudWatch logs. If omitted, no logs will be exported. Valid values (depending on engine). MySQL and MariaDB: `"audit"`, `"error"`, `"general"`, `"slowquery"`. PostgreSQL: `"postgresql"`.

- (map(object)) **`cluster_instances = {}`** _[since v1.0.0]_

    Manages multiple instances for an Aurora cluster. Must be used with an `"aurora-*"` engine type. See [example](#aurora-regional-cluster)

    - (map(string)) **`additional_tags = {}`** _[since v1.0.0]_

        Additional tags for the individual cluster instance

    - (string) **`db_parameter_group = null`** _[since v1.0.0]_

        Specify the name of the DB parameter group to be associated to the instance.

    - (number) **`failover_priority = null`** _[since v1.0.0]_

        Default 0. [Failover Priority][aurora-failover-priority] setting on instance level. The reader who has lower tier has higher priority to get promoted to writer.

    - (string) **`instance_class = null`** _[since v1.0.0]_

        Specify the DB instance class for the individual instance. Do not use for serverless cluster. See [example](#aurora-global-cluster)

    - (object) **`maintenance_config = {}`** _[since v2.0.0]_

        Configures RDS maintenance options. If not specified, the cluster level options will be used.

        - (string) **`window = null`** _[since v2.0.0]_

            Window to perform maintenance in (in UTC). Syntax: `"ddd:hh24:mi-ddd:hh24:mi"`. For example `"Mon:00:00-Mon:03:00"`.

        - (bool) **`enable_auto_minor_version_upgrade = null`** _[since v2.0.0]_

            Indicates that minor engine upgrades will be applied automatically to the DB instance during the maintenance window

    - (object) **`monitoring_config = {}`** _[since v2.0.0]_

        Configures RDS monitoring options for individual cluster instances

        - (map(object)) **`cloudwatch_alarms = {}`** _[since v2.0.0]_

            Configures multiple Cloudwatch alarms. Please see [example](#cloudwatch-alarms)

          - (string) **`metric_name`** _[since v2.0.0]_

              The metric to monitor. Please refer to [this document][aurora-cloudwatch-metrics] for more information

          - (string) **`expression`** _[since v2.0.0]_

            The expression in `<statistic> <operator> <unit>` format. For example: `"Average < 50"`

          - (string) **`notification_sns_topic`** _[since v2.0.0]_

            The SNS topic where notification will be sent

          - (string) **`description = null`** _[since v2.0.0]_

            The description of the alarm

          - (number) **`evaluation_periods = 1`** _[since v2.0.0]_

            The number of periods over which data is compared to the specified threshold.

          - (string) **`period = "1 minute"`** _[since v2.0.0]_

            The period in seconds over which the specified statistic is applied. Valid values: `"1 minute"` - `"6 hours"`

        - (object) **`enable_enhanced_monitoring = null`** _[since v2.0.0]_

            Enables [RDS enhanced monitoring][rds-enhanced-monitoring].

            - (number) **`interval`** _[since v2.0.0]_

                Interval, in seconds, between points when Enhanced Monitoring metrics are collected for the DB instance. To disable collecting Enhanced Monitoring metrics, specify 0. Valid Values: `0`, `1`, `5`, `10`, `15`, `30`, `60`.

            - (string) **`iam_role_arn = null`** _[since v2.0.0]_

                ARN for the IAM role that permits RDS to send enhanced monitoring metrics to CloudWatch Logs. Please refer to [this documentation][rds-enhanced-monitoring-iam-requirement] for information of the required IAM permissions. One will be created if not specified.

        - (object) **`enable_performance_insight = null`** _[since v2.0.0]_

            Enables [RDS performance insight][rds-performance-insight]

            - (number) **`retention_period`** _[since v2.0.0]_

                Amount of time in days to retain Performance Insights data. Valid values are `7`, `731` (2 years) or a `multiple of 31`.

            - (string) **`kms_key_id = null`** _[since v2.0.0]_

                ARN for the KMS key to encrypt Performance Insights data.

    - (object) **`networking_config = null`** _[since v1.0.0]_

        Configures connectivity options for the individual instance

        - (string) **`availability_zone = null`** _[since v1.0.0]_

            The availability zone to deploy the RDS instance in

        - (bool) **`enable_public_access = null`** _[since v1.0.0]_

            Specify whether the RDS instance is publicly accessible

- (string) **`database_insights = "standard"`** _[since v1.0.0]_

    The mode of Database Insights that is enabled for the cluster or the instance. Valid values: `"standard"`, `"advanced"`

- (string) **`db_name = null`** _[since v1.0.0]_

    The name of the database to create when the DB instance or cluster is created. If this parameter is not specified, no database is created.

- (string) **`db_cluster_parameter_group = null`** _[since v1.0.0]_

    Specify the name of the DB parameter group to be attached to all instances in the cluster

- (string) **`db_parameter_group = null`** _[since v1.0.0]_

    Specify the name of the DB parameter group to be attached to the instance

- (bool) **`delete_protection_enabled = false`** _[since v1.0.0]_

    Prevent the instance or cluster from deletion when this value is set to `true`

- (string) **`deployment_option = SingleInstance`** _[since v1.0.0]_

    Specify the option for non-aurora deployment. Valid values are: `"SingleInstance"`, ["MultiAZInstance"][rds-multi-az-instance], ["MultiAZCluster"][rds-multi-az-cluster]. `MultiAZInstance` and `MultiAZCluster` only support the `"mysql"` and `"postgres"` engine type.

- (object) **`enable_automated_backup = null`** _[since v1.0.0]_

    Configures RDS automated backup

    - (number) **`retention_period`** _[since v1.0.0]_

        The number of days (1-35) for which automatic backups are kept.

    - (bool) **`copy_tags_to_snapshot = true`** _[since v1.0.0]_

        Indicates whether to copy all of the user-defined tags from the DB instance to snapshots of the DB instance

    - (string) **`window = null`** _[since v1.0.0]_

        Daily time range (in UTC) during which automated backups are created. In the `"hh24:mi-hh24:mi"` format. For example `"04:00-09:00"`

- (object) **`enable_encryption = null`** _[since v1.0.0]_

    Enables [RDS DB encryption][rds-db-encryption] to encrypt the DB instance's underlying storage

    - (string) **`kms_key_alias`** _[since v1.0.0]_

        The KMS CMK used to encrypt the DB and storage

- (object) **`maintenance_config = null`** _[since v1.0.0]_

    Configures RDS maintenance options

    - (string) **`window`** _[since v1.0.0]_

        Window to perform maintenance in (in UTC). Syntax: `"ddd:hh24:mi-ddd:hh24:mi"`. For example `"Mon:00:00-Mon:03:00"`.

    - (bool) **`enable_auto_minor_version_upgrade = true`** _[since v1.0.0]_

        Indicates that minor engine upgrades will be applied automatically to the DB instance during the maintenance window

- (object) **`monitoring_config = {}`** _[since v1.0.0]_

    Configures RDS monitoring options

    - (map(object)) **`cloudwatch_alarms = {}`** _[since v2.0.0]_

        Configures multiple Cloudwatch alarms. Please see [example](#cloudwatch-alarms)

      - (string) **`metric_name`** _[since v2.0.0]_

          The metric to monitor. Please refer to [this document][rds-cloudwatch-metrics] for more information

      - (string) **`expression`** _[since v2.0.0]_

        The expression in `<statistic> <operator> <unit>` format. For example: `"Average < 50"`

      - (string) **`notification_sns_topic`** _[since v2.0.0]_

        The SNS topic where notification will be sent

      - (string) **`description = null`** _[since v2.0.0]_

        The description of the alarm

      - (number) **`evaluation_periods = 1`** _[since v2.0.0]_

        The number of periods over which data is compared to the specified threshold.

      - (string) **`period = "1 minute"`** _[since v2.0.0]_

        The period in seconds over which the specified statistic is applied. Valid values: `"1 minute"` - `"6 hours"`


    - (object) **`enable_enhanced_monitoring = null`** _[since v1.0.0]_

        Enables [RDS enhanced monitoring][rds-enhanced-monitoring]. If this is enabled when using a cluster setup, you can no longer enable enhanced monitoring in each individual cluster instances.

        - (number) **`interval`** _[since v1.0.0]_

            Interval, in seconds, between points when Enhanced Monitoring metrics are collected for the DB instance. To disable collecting Enhanced Monitoring metrics, specify 0. Valid Values: `0`, `1`, `5`, `10`, `15`, `30`, `60`.

        - (string) **`iam_role_arn = null`** _[since v1.0.0]_

            ARN for the IAM role that permits RDS to send enhanced monitoring metrics to CloudWatch Logs. Please refer to [this documentation][rds-enhanced-monitoring-iam-requirement] for information of the required IAM permissions. One will be created if not specified.

    - (object) **`enable_performance_insight = null`** _[since v1.0.0]_

        Enables [RDS performance insight][rds-performance-insight]

        - (number) **`retention_period`** _[since v1.0.0]_

            Amount of time in days to retain Performance Insights data. Valid values are `7`, `731` (2 years) or a `multiple of 31`.

        - (string) **`kms_key_id = null`** _[since v1.0.0]_

            ARN for the KMS key to encrypt Performance Insights data.

- (string) **`option_group = null`** _[since v1.0.0]_

    Specify the name of the [option group][rds-option-group] to be attached to the instance

- (map(object)) **`proxies = {}`** _[since v1.1.0]_

    Manages multiple RDS proxies that are associated to the DB cluster or instance. Please see [example](#rds-proxies)

    - (map(object)) **`authentications`** _[since v1.1.0]_

        Managers multiple authentication configurations. The key of the map will be the Secrets Manager secrets representing the credentials for database user accounts that the proxy can use.

      - (string) **`client_authentication_type`** _[since v1.1.0]_

          The method that the proxy uses to authenticate connections from clients. Valid values: `"MYSQL_CACHING_SHA2_PASSWORD"`, `"MYSQL_NATIVE_PASSWORD"`, `"POSTGRES_SCRAM_SHA_256"`, `"POSTGRES_MD5"`

      - (bool) **`allow_iam_authentication = false`** _[since v1.1.0]_

          Whether to require or disallow Amazon Web Services Identity and Access Management (IAM) authentication for connections to the proxy

    - (list(string)) **`security_group_ids`** _[since v1.1.0]_

        One or more RDS security groups to allow access to your proxy

    - (list(string)) **`subnet_ids`** _[since v1.1.0]_

        List of subnets the database can use in the VPC that you selected. A minimum of 2 subnets in different Availability Zones is required for the proxy.

    - (map(object)) **`additional_endpoints = null`** _[since v1.1.0]_

        Manages additional endpoints beside the default

      - (list(string)) **`security_group_ids = null`** _[since v1.1.0]_

          One or more RDS security groups to allow access to your proxy. If not specified, the security_group_ids of the proxy will be used.

      - (list(string)) **`subnet_ids = null`** _[since v1.1.0]_

          List of subnets the database can use in the VPC that you selected. A minimum of 2 subnets in different Availability Zones is required for the proxy. If not specified, the subnet_ids of the proxy will be used.

      - (string) **`target_role = "READ_WRITE"`** _[since v1.1.0]_

          Defines how the workload for this proxy endpoint will be used. Valid values: `"READ_WRITE"`, `"READ_ONLY"`

    - (bool) **`activate_enhanced_logging = false`** _[since v1.1.0]_

        With enhanced logging, details of queries processed by the proxy are logged and published to CloudWatch Logs.

    - (map(object)) **`additional_tags = {}`** _[since v1.1.0]_

        Additional tags that are attached to the proxy

    - (string) **`iam_role_arn = null`** _[since v1.1.0]_

        ARN of the IAM role the proxy will use to access the AWS Secrets Manager secrets specified in `authentications`. If unspecified, an IAM role will be created with read permissions to all the secrets specified in `authentications`.

    - (string) **`idle_client_connection_timeout = "30 minutes"`** _[since v1.1.0]_

        Idle connection from your application are closed after the specified time. Valid value: `"1 minute" - "8 hours"`

    - (bool) **`require_transport_layer_security = false`** _[since v1.1.0]_

        whether Transport Layer Security (TLS) encryption is required for connections to the proxy

    - (object) **`target_group_config = {}`** _[since v1.1.0]_

        Manages the default target group's configuration

      - (string) **`connection_borrow_timeout = "2 minutes"`** _[since v1.1.0]_

          Timeout for borrowing DB connection from the pool. Valid values: `"1 second" - "5 minutes"`

      - (number) **`connection_pool_maximum_connections = 100`** _[since v1.1.0]_

          Specify the maximum allowed connections, as a percentage of the maximum connection limit of your database. For example, if you have set the maximum connections to 5,000 connections, specifying `50` allows your proxy to create up to 2,500 connections to the database.

      - (string) **`initalization_query = null`** _[since v1.1.0]_

          Specify one or more SQL statements to set up the initial session state for each connection. Separate statements with semicolons.

      - (number) **`max_idle_connections_percent = 50`** _[since v1.1.0]_

          Controls how actively the proxy closes idle database connections in the connection pool. A high value enables the proxy to leave a high percentage of idle connections open. A low value causes the proxy to close idle client connections and return the underlying database connections to the connection pool. For Aurora MySQL, it is expressed as a percentage of the max_connections setting for the RDS DB instance or Aurora DB cluster used by the target group.

      - (list(string)) **`session_pinning_filters = null`** _[since v1.1.0]_

          Each item in the list represents a class of SQL operations that normally cause all later statements in a session using a proxy to be pinned to the same underlying database connection. Including an item in the list exempts that class of SQL operations from the pinning behavior. This setting is only supported for MySQL engine family databases. Valid values: `"EXCLUDE_VARIABLE_SETS"`

- (object) **`restore = {}`** _[since v2.1.0]_

    Restore RDS cluster or instance from a particular source.

    - (string) **`from_snapshot = null`** _[since v2.1.0]_

        The snapshot ARN from which RDS restored

- (object) **`serverless_capacity = null`** _[since v1.0.0]_

    Specify the capacity range of the serverless instance. Must be used with `instance_class = "db.serverless"` and an `"aurora-*"` engine type, [see example](#aurora-global-cluster). Refer to [this documentation][aurora-capacity-unit] for more details.

    - (number) **`min_acus`** _[since v1.0.0]_

        Specify the minimum Aurora capacity unit. Each ACU corresponds to approximately 2 GiB of memory

    - (number) **`max_acus = null`** _[since v1.0.0]_

        Specify the maximum Aurora capacity unit. Each ACU corresponds to approximately 2 GiB of memory. Must be greater than `min_acus`, if unspecified, the value of `min_acus` will be used.

- (bool) **`skip_final_snapshot = null`** _[since v1.0.0]_

    Determines whether a final DB snapshot is created before the DB cluster is deleted

- (object) **`storage_config = null`** _[since v1.0.0]_

    Configures RDS storage options

    - (number) **`allocated_storage`** _[since v1.0.0]_

        The allocated storage in gibibytes

    - (string) **`type`** _[since v1.0.0]_

        Specify the storage type. Valid values are: `"gp3"` and `"io1"`

    - (number) **`max_allocated_storage = null`** _[since v1.0.0]_

        When configured, the upper limit to which Amazon RDS can automatically scale the storage of the DB instance. Configuring this will automatically ignore differences to `allocated_storage`. Must be greater than or equal to allocated_storage or `0` to disable Storage Autoscaling

    - (number) **`provisioned_iops = null`** _[since v1.0.0]_

        The amount of provisioned IOPS. Can only be set when `type` is `"io1"` or `"gp3"`. Please refer to [this documentation][rds-provisioned-iops] for more details.

    - (number) **`storage_throughput = null`** _[since v1.0.0]_

        The storage throughput value for the DB instance. Can only be set when `type = "gp3"`. Please refer to [this documentation][rds-storage-throughput] for more details.

## Outputs

- (string) **`aurora_cluster_endpoint`** _[since v1.0.0]_

    DNS address of the Writer instance

- (list(string)) **`aurora_cluster_members`** _[since v1.0.0]_

    List of RDS Instances that are a part of this Aurora cluster

- (string) **`aurora_cluster_reader_endpoint`** _[since v1.0.0]_

    Read-only endpoint for the Aurora cluster, automatically load-balanced across replicas

- (string) **`aurora_global_cluster_arn`** _[since v1.0.0]_

    The ARN of the Aurora global cluster created by this module

- (string) **`aurora_global_cluster_identifier`** _[since v1.0.0]_

    The name of the Aurora global cluster created by this module

- (string) **`cluster_arn`** _[since v1.0.0]_

    The ARN of the RDS cluster. Only applicable if deploying an `Aurora cluster` or a `Multi-AZ Cluster`

- (string) **`cluster_identifier`** _[since v1.0.0]_

    The name of the RDS cluster. Only applicable if deploying an `Aurora cluster` or a `Multi-AZ Cluster`

- (object) **`master_user_secret`** _[since v1.0.0]_

    Retrive master user secret. Only available when `authentication_config.db_master_account.manage_password_in_secrets_manager = true`

    - (string) **`kms_key_id`** _[since v1.0.0]_

        Amazon Web Services KMS key identifier that is used to encrypt the secret.

    - (string) **`secret_arn`** _[since v1.0.0]_

        Amazon Resource Name (ARN) of the secret.

    - (string) **`secret_status`** _[since v1.0.0]_

        Status of the secret. Value can be: `"creating"`, `"active"`, `"rotating"`, or `"impaired"`.

- (map(string)) **`rds_connect_iam_policy_arns`** _[since v1.0.0]_

    The map of IAM policy ARNs for RDS connect. Only available when `authentication_config.iam_database_authentication.enabled = true`

[aurora-capacity-unit]:https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html
[aurora-cloudwatch-metrics]:https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html
[aurora-db-instance-class]:https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.DBInstanceClass.html
[aurora-failover-priority]:https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html#Aurora.Managing.FaultTolerance
[db-instance-class]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.DBInstanceClass.html
[manage-password-in-secrets-manager]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-secrets-manager.html
[rds-ca]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html#UsingWithRDS.SSL.RegionCertificateAuthorities
[rds-cluster-parameter-group]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithDBClusterParamGroups.html
[rds-instance-parameter-group]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithDBInstanceParamGroups.html
[rds-db-encryption]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html
[rds-enhanced-monitoring]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_Monitoring.OS.overview.html
[rds-enhanced-monitoring-iam-requirement]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_Monitoring.OS.Enabling.html#USER_Monitoring.OS.Enabling.Prerequisites
[rds-iam-authentication-policy]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.IAMPolicy.html
[rds-iam-db-authentication]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html
[rds-cloudwatch-metrics]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-metrics.html
[rds-multi-az-instance]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html
[rds-multi-az-cluster]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html
[rds-performance-insight]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.Overview.html
[rds-provisioned-iops]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html#gp3-storage
[rds-option-group]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithOptionGroups.html
[rds-storage-throughput]:https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html#gp3-storage
