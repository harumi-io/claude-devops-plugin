# Security Patterns

Security patterns for cloud infrastructure. Read this when configuring security for any resource.

## Core Principles

1. **Encryption by default** — All resources with encryption support MUST enable it
2. **Private by default** — Resources in private subnets/networks unless justified
3. **Least privilege** — IAM/RBAC policies grant minimum required permissions
4. **No secrets in code** — All credentials in secrets manager (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault)
5. **Defense in depth** — Multiple security layers

## Secrets Management

### AWS

```hcl
# NEVER hardcode secrets
# Use managed credentials
resource "aws_db_instance" "this" {
  manage_master_user_password   = true
  master_user_secret_kms_key_id = var.kms_key_arn
}

# Reference existing secret
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "app-db-credentials"
}
```

### GCP

```hcl
data "google_secret_manager_secret_version" "db" {
  secret  = "db-password"
  project = var.project
}
```

### Azure

```hcl
data "azurerm_key_vault_secret" "db" {
  name         = "db-password"
  key_vault_id = var.key_vault_id
}
```

## Storage Security

### AWS S3

```hcl
module "s3_bucket" {
  source  = "cloudposse/s3-bucket/aws"

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true

  versioning_enabled = true
}
```

### GCP GCS

```hcl
resource "google_storage_bucket" "this" {
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"

  versioning {
    enabled = true
  }
}
```

### Azure Storage

```hcl
resource "azurerm_storage_account" "this" {
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
}
```

## Database Security

```hcl
# Universal patterns:
# - Always encrypt storage
# - Never make publicly accessible
# - Enable deletion protection in production
# - Use secrets manager for credentials
# - Restrict network access to application security groups only

# AWS RDS
resource "aws_db_instance" "this" {
  storage_encrypted             = true
  publicly_accessible           = false
  deletion_protection           = var.environment == "production"
  manage_master_user_password   = true
  backup_retention_period       = 7
  vpc_security_group_ids        = [aws_security_group.rds.id]
}

# Security group: only allow from application
resource "aws_security_group" "rds" {
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "PostgreSQL from application"
  }
}
```

## IAM Security

### Least privilege pattern

```hcl
# GOOD: Specific permissions and resources
data "aws_iam_policy_document" "s3_read" {
  statement {
    sid     = "ReadBucket"
    effect  = "Allow"
    actions = ["s3:GetObject", "s3:ListBucket"]
    resources = [
      "arn:aws:s3:::${var.bucket_name}",
      "arn:aws:s3:::${var.bucket_name}/*",
    ]
  }
}

# BAD: Overly permissive
actions   = ["s3:*"]
resources = ["*"]
```

### Service role pattern

Separate execution roles (pulls images, writes logs) from task roles (application permissions):

```hcl
# Execution role — platform permissions
resource "aws_iam_role" "execution" {
  name = "${var.name}-execution-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# Task role — application permissions
resource "aws_iam_role" "task" {
  name = "${var.name}-task-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

## Network Security

- **Public subnets** — Load balancers, NAT Gateways only
- **Private subnets** — Applications, databases, caches

### Security group patterns

```hcl
# Load balancer: Public-facing
resource "aws_security_group" "alb" {
  ingress { from_port = 443; to_port = 443; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 80; to_port = 80; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}

# Application: Internal only (traffic from LB)
resource "aws_security_group" "app" {
  ingress { from_port = 0; to_port = 65535; protocol = "tcp"; security_groups = [aws_security_group.alb.id] }
  egress  { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}
```

## Encryption

### KMS (AWS)

```hcl
resource "aws_kms_key" "this" {
  description             = "KMS key for ${var.naming_namespace} ${var.environment}"
  deletion_window_in_days = 30
  enable_key_rotation     = true  # ALWAYS enable
}
```

Used for: EBS, S3, RDS storage, Secrets Manager, CloudWatch Logs.

## Security Checklist

### Storage
- [ ] Public access blocked
- [ ] Encryption enabled
- [ ] Versioning enabled (data buckets)

### Databases
- [ ] Private subnet, not publicly accessible
- [ ] Storage encrypted
- [ ] Credentials in secrets manager
- [ ] Deletion protection (production)

### Cache
- [ ] Private subnet
- [ ] Encryption at rest and in transit
- [ ] Auth configured

### IAM
- [ ] Least privilege policies
- [ ] No wildcard resources unless justified
- [ ] Service accounts have specific permissions

### Network
- [ ] Databases in private subnets
- [ ] Load balancers with HTTPS
- [ ] Security groups use specific source references
