

# Terraform code for  RDS instance resource


```terraform
resource "aws_db_instance" "default" {
  allocated_storage    = 10
  db_name              = "mydb"
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  username             = "foo"
  password             = "foobarbaz"
  parameter_group_name = "default.mysql8.0"
  skip_final_snapshot  = true
}
```

### RDS Custom for Oracle Usage with Replica

```terraform
# Lookup the available instance classes for the custom engine for the region being operated in
data "aws_rds_orderable_db_instance" "custom-oracle" {
  engine                     = "custom-oracle-ee" # CEV engine to be used
  engine_version             = "19.c.ee.002"      # CEV engine version to be used
  license_model              = "bring-your-own-license"
  storage_type               = "gp3"
  preferred_instance_classes = ["db.r5.xlarge", "db.r5.2xlarge", "db.r5.4xlarge"]
}

# The RDS instance resource requires an ARN. Look up the ARN of the KMS key associated with the CEV.
data "aws_kms_key" "by_id" {
  key_id = "example-ef278353ceba4a5a97de6784565b9f78" # KMS key associated with the CEV
}

resource "aws_db_instance" "default" {
  allocated_storage           = 50
  auto_minor_version_upgrade  = false                         # Custom for Oracle does not support minor version upgrades
  custom_iam_instance_profile = "AWSRDSCustomInstanceProfile" # Instance profile is required for Custom for Oracle. See: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/custom-setup-orcl.html#custom-setup-orcl.iam-vpc
  backup_retention_period     = 7
  db_subnet_group_name        = local.db_subnet_group_name
  engine                      = data.aws_rds_orderable_db_instance.custom-oracle.engine
  engine_version              = data.aws_rds_orderable_db_instance.custom-oracle.engine_version
  identifier                  = "ee-instance-demo"
  instance_class              = data.aws_rds_orderable_db_instance.custom-oracle.instance_class
  kms_key_id                  = data.aws_kms_key.by_id.arn
  license_model               = data.aws_rds_orderable_db_instance.custom-oracle.license_model
  multi_az                    = false # Custom for Oracle does not support multi-az
  password                    = "avoid-plaintext-passwords"
  username                    = "test"
  storage_encrypted           = true

  timeouts {
    create = "3h"
    delete = "3h"
    update = "3h"
  }
}

resource "aws_db_instance" "test-replica" {
  replicate_source_db         = aws_db_instance.default.identifier
  replica_mode                = "mounted"
  auto_minor_version_upgrade  = false
  custom_iam_instance_profile = "AWSRDSCustomInstanceProfile" # Instance profile is required for Custom for Oracle. See: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/custom-setup-orcl.html#custom-setup-orcl.iam-vpc
  backup_retention_period     = 7
  identifier                  = "ee-instance-replica"
  instance_class              = data.aws_rds_orderable_db_instance.custom-oracle.instance_class
  kms_key_id                  = data.aws_kms_key.by_id.arn
  multi_az                    = false # Custom for Oracle does not support multi-az
  skip_final_snapshot         = true
  storage_encrypted           = true

  timeouts {
    create = "3h"
    delete = "3h"
    update = "3h"
  }
}
```

### RDS Custom for SQL Server

```terraform
# Lookup the available instance classes for the custom engine for the region being operated in
data "aws_rds_orderable_db_instance" "custom-sqlserver" {
  engine                     = "custom-sqlserver-se" # CEV engine to be used
  engine_version             = "15.00.4249.2.v1"     # CEV engine version to be used
  storage_type               = "gp3"
  preferred_instance_classes = ["db.r5.xlarge", "db.r5.2xlarge", "db.r5.4xlarge"]
}

# The RDS instance resource requires an ARN. Look up the ARN of the KMS key.
data "aws_kms_key" "by_id" {
  key_id = "example-ef278353ceba4a5a97de6784565b9f78" # KMS key
}

resource "aws_db_instance" "example" {
  allocated_storage           = 500
  auto_minor_version_upgrade  = false                                  # Custom for SQL Server does not support minor version upgrades
  custom_iam_instance_profile = "AWSRDSCustomSQLServerInstanceProfile" # Instance profile is required for Custom for SQL Server. See: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/custom-setup-sqlserver.html#custom-setup-sqlserver.iam
  backup_retention_period     = 7
  db_subnet_group_name        = local.db_subnet_group_name # Copy the subnet group from the RDS Console
  engine                      = data.aws_rds_orderable_db_instance.custom-sqlserver.engine
  engine_version              = data.aws_rds_orderable_db_instance.custom-sqlserver.engine_version
  identifier                  = "sql-instance-demo"
  instance_class              = data.aws_rds_orderable_db_instance.custom-sqlserver.instance_class
  kms_key_id                  = data.aws_kms_key.by_id.arn
  multi_az                    = false # Custom for SQL Server does support multi-az
  password                    = "avoid-plaintext-passwords"
  storage_encrypted           = true
  username                    = "test"

  timeouts {
    create = "3h"
    delete = "3h"
    update = "3h"
  }
}
```

### RDS Db2 Usage

```terraform
# Lookup the default version for the engine. Db2 Standard Edition is `db2-se`, Db2 Advanced Edition is `db2-ae`.
data "aws_rds_engine_version" "default" {
  engine = "db2-se" #Standard Edition
}

# Lookup the available instance classes for the engine in the region being operated in
data "aws_rds_orderable_db_instance" "example" {
  engine                     = data.aws_rds_engine_version.default.engine
  engine_version             = data.aws_rds_engine_version.default.version
  license_model              = "bring-your-own-license"
  storage_type               = "gp3"
  preferred_instance_classes = ["db.t3.small", "db.r6i.large", "db.m6i.large"]
}

# The RDS Db2 instance resource requires licensing information. Create a new parameter group using the default paramater group as a source, and set license information.
resource "aws_db_parameter_group" "example" {
  name   = "db-db2-params"
  family = data.aws_rds_engine_version.default.parameter_group_family

  parameter {
    apply_method = "immediate"
    name         = "rds.ibm_customer_id"
    value        = 0000000000
  }
  parameter {
    apply_method = "immediate"
    name         = "rds.ibm_site_id"
    value        = 0000000000
  }
}

# Create the RDS Db2 instance, use the data sources defined to set attributes
resource "aws_db_instance" "example" {
  allocated_storage       = 100
  backup_retention_period = 7
  db_name                 = "test"
  engine                  = data.aws_rds_orderable_db_instance.example.engine
  engine_version          = data.aws_rds_orderable_db_instance.example.engine_version
  identifier              = "db2-instance-demo"
  instance_class          = data.aws_rds_orderable_db_instance.example.instance_class
  parameter_group_name    = aws_db_parameter_group.example.name
  password                = "avoid-plaintext-passwords"
  username                = "test"
}
```

### Storage Autoscaling

To enable Storage Autoscaling with instances that support the feature, define the `max_allocated_storage` argument higher than the `allocated_storage` argument. Terraform will automatically hide differences with the `allocated_storage` argument value if autoscaling occurs.

```terraform
resource "aws_db_instance" "example" {
  # ... other configuration ...

  allocated_storage     = 50
  max_allocated_storage = 100
}
```

### Managed Master Passwords via Secrets Manager, default KMS Key

-> More information about RDS/Aurora Aurora integrates with Secrets Manager to manage master user passwords for your DB clusters can be found in the [RDS User Guide](https://aws.amazon.com/about-aws/whats-new/2022/12/amazon-rds-integration-aws-secrets-manager/) and [Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-secrets-manager.html).

You can specify the `manage_master_user_password` attribute to enable managing the master password with Secrets Manager. You can also update an existing cluster to use Secrets Manager by specify the `manage_master_user_password` attribute and removing the `password` attribute (removal is required).

```terraform
resource "aws_db_instance" "default" {
  allocated_storage           = 10
  db_name                     = "mydb"
  engine                      = "mysql"
  engine_version              = "8.0"
  instance_class              = "db.t3.micro"
  manage_master_user_password = true
  username                    = "foo"
  parameter_group_name        = "default.mysql8.0"
}
```

### Managed Master Passwords via Secrets Manager, specific KMS Key

-> More information about RDS/Aurora Aurora integrates with Secrets Manager to manage master user passwords for your DB clusters can be found in the [RDS User Guide](https://aws.amazon.com/about-aws/whats-new/2022/12/amazon-rds-integration-aws-secrets-manager/) and [Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-secrets-manager.html).

You can specify the `master_user_secret_kms_key_id` attribute to specify a specific KMS Key.

```terraform
resource "aws_kms_key" "example" {
  description = "Example KMS Key"
}

resource "aws_db_instance" "default" {
  allocated_storage             = 10
  db_name                       = "mydb"
  engine                        = "mysql"
  engine_version                = "8.0"
  instance_class                = "db.t3.micro"
  manage_master_user_password   = true
  master_user_secret_kms_key_id = aws_kms_key.example.key_id
  username                      = "foo"
  parameter_group_name          = "default.mysql8.0"
}
```

