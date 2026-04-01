# Infrastructure Architecture Guide

How to document and understand infrastructure architecture. Read this when working on changes that span modules or need to understand dependencies.

## What to Document

Every infrastructure project should have a clear picture of:

### State File Organization

Map which root modules manage which resources, and where their state lives:

| Root Module | State Path | Status |
|-------------|------------|--------|
| Core Infrastructure | `[backend]://[bucket]/[key]` | Active |
| IAM | `[backend]://[bucket]/[key]` | Active |
| Applications | `[backend]://[bucket]/[key]` | Active |

### Module Inventory

List active submodules with their status and purpose:

| Submodule | Path | Status | Description |
|-----------|------|--------|-------------|
| network | `./network` | ACTIVE | VPC, subnets, NAT, IGW |
| dns | `./dns` | ACTIVE | DNS zones, certificates |
| security | `./security` | ACTIVE | KMS keys, WAF rules |

Use `CONDITIONAL` status for feature-flagged modules.

### Network Configuration

Document CIDRs and subnet layout:

| Component | CIDR | Location |
|-----------|------|----------|
| VPC | 10.0.0.0/16 | core/network |
| Public Subnet A | 10.0.100.0/20 | core/network |
| Private Subnet A | 10.0.116.0/20 | core/network |

### Domain Configuration

| Domain | Purpose | Managed By |
|--------|---------|------------|
| example.com | Primary domain | core/dns |
| *.example.com | Wildcard cert | core/dns |
| api.example.com | Backend API | applications |

### Conditional Feature Flags

```hcl
# prod.tfvars
enable_feature_x = true
remove_legacy_z  = false  # ALWAYS default false for removal flags
```

## Initialization Commands

Always document the exact init commands for each module, especially backend-config requirements:

```bash
# Module with explicit backend-config
cd [module-path]
terraform init -backend-config="key=[module]/terraform.tfstate"
terraform plan -var-file=[var_file]

# Module with default backend
cd [root-path]
terraform init
terraform plan -var-file=[var_file]
```

## Cross-Module Dependencies

Document which modules reference other modules' state:

```
core-infrastructure (VPC, DNS, Security)
    |
    +--> IAM (roles reference VPC)
    |
    +--> Applications (reference VPC, DNS, IAM outputs)
```

## Example: Harumi.io Architecture

The Harumi.io project uses three independent state files:

```
infrastructure/
+-- main.tf                    # Legacy: ECS, RDS, Redis, S3, Glue, CDN
+-- core-infrastructure/       # Modern: Network, DNS, Security, ECS cluster, EKS
+-- iam/                       # Users, groups, roles, service accounts
+-- modules/                   # 27 reusable child modules
```

This demonstrates a common migration pattern from monolithic to modular Terraform.
