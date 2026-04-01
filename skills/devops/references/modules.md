# Module Patterns

Module patterns and architecture conventions. Read this when creating new modules or refactoring existing ones.

## Root Module Pattern

```
<root-module>/
+-- main.tf         # Module calls and orchestration
+-- variables.tf    # Input variables
+-- outputs.tf      # Outputs for cross-module references
+-- locals.tf       # Local values (optional)
+-- backend.tf      # State backend configuration
+-- providers.tf    # Provider configuration
+-- versions.tf     # Provider/Terraform version constraints
+-- prod.tfvars     # Production values
+-- dev.tfvars      # Development values (optional)
+-- README.md       # Module documentation
```

Each root module should use a different state key when sharing a backend:

```bash
terraform init -backend-config="key=core-infrastructure/terraform.tfstate"
terraform init -backend-config="key=iam/terraform.tfstate"
terraform init  # Root/default module
```

## Child Module Pattern

### Community modules (CloudPosse, HashiCorp, etc.)

```hcl
module "s3_bucket" {
  source  = "cloudposse/s3-bucket/aws"
  version = "4.5.0"

  namespace = var.naming_namespace
  stage     = var.environment
  name      = var.name
}
```

### Custom modules

```hcl
# modules/fargate/main.tf
resource "aws_ecs_task_definition" "this" {
  family = "${var.container_name}-${var.environment}-ecs-task"
}
```

### Module call pattern

```hcl
module "backend_api" {
  source = "./modules/fargate"

  container_name  = "${var.naming_namespace}-api"
  environment     = var.environment
}
```

## Cross-Module References

### Remote state pattern

```hcl
data "terraform_remote_state" "core" {
  backend = "s3"  # Match your state_backend
  config = {
    bucket = var.state_bucket
    key    = "core-infrastructure/terraform.tfstate"
    region = var.region
  }
}

# Usage
vpc_id = data.terraform_remote_state.core.outputs.vpc_id
```

### Expected outputs from core modules

```hcl
# Core infrastructure should expose:
output "vpc_id" {}
output "vpc_cidr_block" {}
output "public_subnet_ids" {}
output "private_subnet_ids" {}
output "route53_zone_id" {}
output "acm_certificate_arn" {}
output "kms_key_arn" {}
```

## Conditional Resource Creation

### Feature flags

```hcl
# tfvars
enable_feature_x        = true
remove_legacy_resource  = false  # ALWAYS default false for removal flags

# main.tf
module "feature_x" {
  count  = var.enable_feature_x ? 1 : 0
  source = "./feature-x"
}
```

### Zero-downtime migration pattern

```hcl
# Phase 1: Create new (enable_new=true, remove_old=false)
# Phase 2: Validate new resources
# Phase 3: Remove old (remove_old=true)

variable "remove_legacy_nodegroup" {
  type    = bool
  default = false  # NEVER default to true
}
```

## When to Create a New Module

**Create in `modules/`** when the component is reusable, has 3+ related resources, and follows a consistent pattern.

**Create as root module** when resources need isolated state, different teams manage them, or changes should not affect other infrastructure.

**Add to existing root module** when resources are tightly coupled, state isolation is not needed, or quick iteration matters more.

When in doubt, ask the user.
