# Naming Conventions

Naming standards for infrastructure resources. Read this when naming new resources or resolving naming ambiguities.

The naming pattern is configured in `harumi.yaml` under `naming`. Defaults:

```yaml
naming:
  pattern: "{namespace}-{stage}-{name}"
  namespace: harumi
  stage: production
```

## Resource Naming

### CloudPosse pattern: namespace-stage-name

```hcl
module "s3_bucket" {
  source    = "cloudposse/s3-bucket/aws"
  namespace = var.naming_namespace    # from config: naming.namespace
  stage     = var.environment         # from config: naming.stage
  name      = var.name                # resource purpose
}
# Result: harumi-production-data-lake
```

### Common patterns

| Pattern | Example |
|---------|---------|
| CloudPosse: {namespace}-{stage}-{name} | harumi-production-data-lake |
| Native: {app}-{env}-{resource} | myapp-prod-vpc |

## IAM Naming

### Users

| Element | Pattern | Example |
|---------|---------|---------|
| Directory | firstname-lastname | italo-rocha |
| Module name | firstname_lastname | italo_rocha |
| Username | firstname.lastname | italo.rocha |

### Service accounts

```hcl
user_name = "airbyte-service"    # purpose-specific
path      = "/service-accounts/"
```

### Groups

```hcl
group_name = "developers-production"  # {purpose}-{environment}
group_name = "admin"                  # Global groups: no environment suffix
```

### Roles

```hcl
name = "developer-role"       # {purpose}-role
name = "ecs-task-role"
name = "ecs-execution-role"
```

## Module and Directory Naming

```
modules/
+-- fargate/              # Lowercase, hyphenated
+-- s3/                   # Simple names for simple resources
+-- iam-developer-user/   # Hyphenated for compound names
+-- data-pipeline-glue/   # Hyphenated for features
```

### Module call naming

```hcl
# Pattern: {layer}_{purpose} or {resource_type}_{name}
module "backend_api" {}
module "frontend_platform" {}
```

## File Naming

### Standard module files

| File | Purpose |
|------|---------|
| main.tf | Primary resource definitions |
| variables.tf | Input variables |
| outputs.tf | Output values |
| versions.tf | Provider/Terraform versions |
| locals.tf | Local values (optional) |
| data.tf | Data sources (optional) |

## Variable Naming

```hcl
variable "environment" {
  description = "Deployment environment (production, staging, development)"
  type        = string
  default     = "production"
}

variable "region" {
  type    = string
}

# Boolean enable flags
variable "enable_encryption" { type = bool; default = true }

# Removal flags: ALWAYS default false
variable "remove_legacy_resource" { type = bool; default = false }
```

## Tag Naming

All resources MUST have:

```hcl
tags = {
  Environment = var.environment
  Project     = var.naming_namespace
  ManagedBy   = "terraform"
}
```

## Output Naming

Pattern: `resource_attribute`

```hcl
output "vpc_id" {}
output "vpc_cidr_block" {}
output "private_subnet_ids" {}
```
