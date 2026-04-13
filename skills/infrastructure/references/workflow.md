# Infrastructure Workflow and Handoff

Detailed workflow phases for infrastructure changes. Read this when following the full change workflow or need downtime/cost guidance.

## Phase 2: Verify with CLI

Verify current state before changes. Terraform state may be outdated or resources may have been modified outside Terraform.

### AWS

```bash
aws ec2 describe-vpcs --vpc-ids [vpc-id]
aws ec2 describe-subnets --filters "Name=vpc-id,Values=[vpc-id]"
aws ecs describe-services --cluster [cluster-name] --services [name]
aws ecs describe-task-definition --task-definition [name]
aws rds describe-db-instances --db-instance-identifier [name]
aws elasticache describe-replication-groups --replication-group-id [name]
aws s3api get-bucket-encryption --bucket [bucket-name]
aws s3api get-public-access-block --bucket [bucket-name]
aws iam get-user --user-name [username]
aws iam get-role --role-name [role-name]
aws eks describe-cluster --name [cluster-name]
aws route53 list-resource-record-sets --hosted-zone-id [zone-id]
```

## Phase 3: Clarify Ambiguities

Common questions to ask the user:

- **Resource location**: Which module or directory should this resource live in?
- **Pattern**: Follow existing patterns or introduce a new convention?
- **Naming**: What naming pattern does this project use? (Check `harumi.yaml` naming section)

## Phase 4a: Downtime Assessment

### Downtime risk by resource type

| Resource | Downtime Risk | Data Loss Risk |
|----------|---------------|----------------|
| RDS | HIGH (10-30 min) | YES |
| ElastiCache Redis | HIGH (5-15 min) | YES (cache) |
| ECS Fargate | LOW (1-5 min) | NO |
| EKS Node Group | MEDIUM (5-15 min) | NO |
| ALB | MEDIUM (5-10 min) | NO |
| VPC | CRITICAL | YES |
| S3 | CRITICAL | YES |
| Security Group / Firewall Rule | LOW | NO |
| IAM Role / Service Account | LOW | NO |
| DNS Record | LOW (TTL dependent) | NO |

### Present alternatives template

```
This change will recreate [resource], which may cause downtime.

Option 1: Zero-downtime migration (Recommended for production)
- Create new resource -> migrate data/traffic -> verify -> remove old
- Complexity: HIGH | Risk: LOW | Downtime: ZERO

Option 2: In-place modification (If provider supports it)
- Apply change directly, provider handles update
- Complexity: LOW | Risk: MEDIUM | Downtime: Brief (~5 min)

Option 3: Maintenance window recreation
- Schedule window -> destroy + create -> restore data
- Complexity: LOW | Risk: HIGH | Downtime: 10-30 minutes

Which approach do you prefer?
```

### Zero-downtime patterns

**ECS**: Rolling updates via deployment configuration (maximum_percent = 200, minimum_healthy_percent = 100).

**EKS Node Groups**: Create new group first, cordon/drain old, then remove old using feature flags.

**Load Balancers**: Use weighted routing to shift traffic gradually.

**Databases**: Create read replica, promote, switch traffic, remove old.

## Phase 4b: Cost Assessment

### Present cost options template

```
This resource has cost implications:

Option 1: Cost-optimized (~$X/month)
- Specs: [instance type, storage]
- Best for: Dev/test, low traffic

Option 2: Balanced (~$Y/month) - Recommended
- Specs: [instance type, storage]
- Best for: Production, moderate load

Option 3: Performance (~$Z/month)
- Specs: [instance type, storage]
- Best for: High traffic, critical workloads

Which option fits your needs?
```

### AWS Pricing Reference (us-east-2, approximate)

**RDS PostgreSQL** (add ~20% for Multi-AZ):

| Instance | vCPU | Memory | Monthly |
|----------|------|--------|---------|
| db.t4g.micro | 2 | 1 GB | ~$12 |
| db.t4g.small | 2 | 2 GB | ~$24 |
| db.t4g.medium | 2 | 4 GB | ~$48 |
| db.t4g.large | 2 | 8 GB | ~$96 |
| db.m6g.large | 2 | 8 GB | ~$120 |

**ElastiCache Redis**:

| Instance | Memory | Monthly |
|----------|--------|---------|
| cache.t4g.micro | 0.5 GB | ~$12 |
| cache.t4g.small | 1.4 GB | ~$24 |
| cache.t4g.medium | 3.1 GB | ~$48 |
| cache.m6g.large | 6.4 GB | ~$110 |

**ECS Fargate** (Spot is ~70% cheaper):

| CPU | Memory | Monthly (24/7) |
|-----|--------|----------------|
| 256 | 512 MB | ~$9 |
| 512 | 2 GB | ~$27 |
| 1024 | 4 GB | ~$54 |
| 2048 | 4 GB | ~$72 |
| 4096 | 8 GB | ~$144 |

**EKS** (Spot saves 50-70%):

| Component | Monthly |
|-----------|---------|
| Control Plane | ~$73 |
| t3.large node | ~$60 |
| m5.large node | ~$70 |
| NAT Gateway (per AZ) | ~$32 + data |

## Phase 5-6: Implement and Validate

```bash
# Format check
terraform fmt -check -recursive
terraform fmt  # Fix formatting

# Validation
terraform validate
```

Common validation errors: `Undeclared resource` (check typo), `Missing required argument` (add field), `Invalid reference` (check provider docs), `Type mismatch` (fix variable type).

## Phase 7: Plan

```bash
terraform plan -var-file=[var_file from harumi.yaml]
```

Red flags in plan output:
- **Unexpected destroys**: Why is this being destroyed?
- **Force replacement (-/+)**: Will data be lost?
- **Many changes**: Does scope match intent?
- **Changes to core resources**: VPC, cluster, etc.

## Phase 8: Handoff

```
Configuration ready for apply!

Please review the plan, then execute:
cd [MODULE_PATH]
terraform apply -var-file=[var_file]

What this will do:
- Create X new resources
- Modify Y existing resources
- Destroy Z resources (if any)

Verification after apply:
terraform state list | grep [resource]
[cloud CLI verification command]

Do NOT proceed if you see unexpected deletions.
```

Risk indicators: Creates only = Safe, Modifications = Caution, Deletions/state changes = High Risk.

## Phase 9: Verify After Apply

After user confirms success, verify with cloud CLI (use service-specific commands from Phase 2).

```
Verification complete!
- [resource 1]: Created successfully
- [resource 2]: Configuration matches expected
- [resource 3]: Security settings correct
```

## Phase 10: Update Documentation

| File | Update When |
|------|-------------|
| Module README/CLAUDE.md | AI guidance or usage changed |
| Architecture docs | Structure changed |
| Naming conventions | New patterns established |

## Multi-Module Changes

Order of operations:
1. **Core Infrastructure** first (VPC, DNS, security)
2. **IAM** second (roles and policies)
3. **Databases** third (RDS, Redis, etc.)
4. **Applications** last (ECS, Lambda, etc.)

When changes span modules, apply core infrastructure first, then refresh dependent modules.

## State Management

Safe operations:
```bash
terraform state list
terraform state show '<address>'
terraform state mv '<old>' '<new>'
terraform import -var-file=[var_file] '<address>' '<id>'
```

Dangerous operations (require explicit user approval):
```bash
terraform state rm '<address>'
terraform force-unlock LOCK_ID
```

## Troubleshooting

| Error | Solution |
|-------|----------|
| `Error acquiring the state lock` | Wait for other process, or `terraform force-unlock LOCK_ID` |
| `Failed to instantiate provider` | `terraform init -upgrade` |
| `Unable to find remote state` | Check backend config key path |
