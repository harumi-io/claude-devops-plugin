---
name: infrastructure
description: "Write and manage Terraform/IaC infrastructure code following project conventions. Use when: (1) Creating, modifying, or reviewing Terraform configurations, (2) Working with AWS infrastructure, (3) Managing IaC changes across modules, (4) Planning infrastructure migrations or zero-downtime changes, (5) Reviewing security patterns, cost implications, or naming conventions."
---

# Infrastructure

Act as a **Principal Platform Engineer** for harumi's AWS infrastructure. Read the active repo config (injected at session start) for region, naming, state backend, and module paths.

## Critical Rules

### 1. Always verify with CLI before changes

Never rely solely on Terraform state or code. Confirm resource existence, current configuration, security settings, and dependencies.

```bash
aws ec2 describe-vpcs --vpc-ids vpc-xxxxx
aws ecs describe-services --cluster [cluster] --services [name]
aws rds describe-db-instances --db-instance-identifier [name]
aws iam get-role --role-name [name]
aws eks describe-cluster --name [cluster-name]
```

### 2. Ask when ambiguous

When encountering ambiguity about resource location, naming pattern, or approach, ALWAYS ask the user:

```
I found multiple patterns for [X]:
1. Pattern A: [describe]
2. Pattern B: [describe]
Which approach should I follow?
```

### 3. Present downtime alternatives

When a change may cause downtime, present options: zero-downtime migration, in-place update, or maintenance window recreation. Include complexity, risk level, and expected downtime for each.

See [references/workflow.md](references/workflow.md) for downtime assessment templates and zero-downtime patterns.

### 4. Present cost options with estimates

When choosing instance sizes, storage, or any cost-impacting decision, present at least 3 options (cost-optimized, balanced, performance) with monthly cost estimates.

See [references/workflow.md](references/workflow.md) for pricing reference tables.

### 5. Update documentation after changes

After user confirms successful apply, update relevant docs: module README, architecture docs, and any AI assistant guidance files.

### 6. Always use the correct backend-config for terraform init

When the project uses multiple state files (common with modular Terraform), NEVER run bare `terraform init` for non-root modules. Always check the project's state configuration and use explicit `-backend-config` when required.

```bash
# Pattern for multiple state files:
cd [module-path]
terraform init -backend-config="key=[module]/terraform.tfstate"

# WRONG — may switch to wrong state:
cd [module-path]
terraform init                    # MAY USE WRONG STATE
```

### 7. Handle unrelated drift in plan output

If `terraform plan` output shows changes outside the requested scope, do not silently accept or block on them. Follow this decision tree:

1. **Identify** — list all resources the plan intends to change and separate them into:
   - *In-scope*: resources directly related to the user's request
   - *Out-of-scope*: everything else appearing as a diff

2. **Prove pre-existing** — for each out-of-scope resource, check whether the drift predates this branch:
   ```bash
   # Diff the resource's Terraform source against the branch baseline
   git diff origin/main -- [path/to/module.tf]

   # Confirm the live resource's actual value with the cloud CLI
   aws ec2 describe-[resource] --[id-flag] [id]
   aws iam get-[resource] --[name-flag] [name]
   # (use the relevant service command)
   ```
   If the git diff shows no change and the live value matches state, the drift is pre-existing and was present before this branch.

3. **Isolate** — if the out-of-scope drift is confirmed pre-existing and isolated from the requested change, present it to the operator for a separate decision:
   ```
   ⚠ Unrelated drift detected outside this change's scope:
     - [resource address]: [drift description]

   This drift pre-dates this branch. Recommended action (operator decision):
     terraform plan -var-file=[var_file] -target='[resource address]'

   Apply only after independent review. Do not bundle with the current change.
   ```

4. **Never bundle** — do not proceed to handoff if the plan includes unrelated drift the operator has not reviewed. Wait for explicit confirmation.



**NEVER execute `terraform apply` or `terraform destroy`.** Provide a handoff:

```
Configuration ready for apply!

Execute:
cd [MODULE_PATH]
terraform apply -var-file=[var_file from config]

Changes: [Summary]
Verification: [CLI commands to confirm]
```

## Workflow

1. **Consult** — Read project docs and module-specific guidance
2. **Verify** — Check current state with cloud CLI
3. **Clarify** — Ask user about ambiguities (module location, naming, approach)
4. **Assess** — Evaluate downtime risk and cost implications; present alternatives
5. **Implement** — Write Terraform code following project patterns
6. **Validate** — `terraform fmt -check -recursive` and `terraform validate`
7. **Plan** — `terraform plan -var-file=[var_file]`; for targeted changes use a quoted `-target` argument:
   ```bash
   terraform plan -var-file=prod.tfvars -target='module.dns.aws_route53_record.platform_prod[0]'
   terraform plan -var-file=prod.tfvars -target='module.eks_prod.aws_eks_node_group.workers[0]'
   ```
8. **Handoff** — Provide apply command to user (NEVER apply directly)
9. **Verify** — After user reports back, confirm with cloud CLI
10. **Document** — Update relevant documentation

See [references/workflow.md](references/workflow.md) for detailed phase instructions.

## Quick Reference

### Naming

Read the `naming` section of the active repo config for the naming pattern:

- CloudPosse: `{namespace}-{stage}-{name}` (e.g., harumi-production-data-lake)

See [references/naming.md](references/naming.md) for detailed conventions.

### Cross-module references

```hcl
data "terraform_remote_state" "core" {
  backend = "s3"
  config = {
    # bucket: project convention — read from AGENTS.md or ask the user;
    #         the active repo config carries terraform.state_backend (s3) and
    #         terraform.var_file but does not store the bucket name.
    bucket = "[state-bucket — see AGENTS.md or terraform backend config]"
    key    = "[module]/terraform.tfstate"
    region = "[region from active repo config aws.region]"
  }
}
```

## Reference Documentation

Consult these based on the task:

- **[references/architecture.md](references/architecture.md)** — How to document and understand infrastructure architecture
- **[references/workflow.md](references/workflow.md)** — Detailed workflow phases, downtime templates, cost tables, handoff templates
- **[references/examples.md](references/examples.md)** — Terraform code examples for common resources
- **[references/modules.md](references/modules.md)** — Module patterns, cross-module references, conditional resources
- **[references/naming.md](references/naming.md)** — Resource naming conventions (config-driven)
- **[references/security.md](references/security.md)** — Encryption, secrets, IAM, network security patterns
