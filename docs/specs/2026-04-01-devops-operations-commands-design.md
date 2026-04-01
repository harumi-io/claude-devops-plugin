# DevOps Operations Commands — Design Spec

**Date:** 2026-04-01
**Status:** Draft

## Overview

Add 7 slash commands to the harumi-devops-plugin that execute daily DevOps operations in the Harumi infrastructure repo. Each command is implemented as a skill with a tight, imperative `SKILL.md` playbook — not open-ended guidance, but deterministic step-by-step execution.

Commands handle IAM user management, VPN certificate lifecycle, service accounts, and access key rotation. All Terraform mutations hand off `terraform apply` to the user. VPN scripts execute directly (non-destructive operations) with confirmation required for revocation.

## Non-Goals

- Config-driven / multi-repo support — commands are Harumi-specific
- Guidance mode — commands execute, not guide
- State snapshot / rollback commands — deferred
- ECS/EKS operational commands — deferred

## Command Inventory

| Command | Skill name | Input | Action |
|---------|-----------|-------|--------|
| `/create-iam-user` | `create-iam-user` | full name, group | Generate Terraform user directory, register in `iam/main.tf`, plan |
| `/remove-iam-user` | `remove-iam-user` | username | Remove Terraform user directory and module block, plan |
| `/create-vpn-creds` | `create-vpn-creds` | username | Run cert generation and export scripts |
| `/revoke-vpn-creds` | `revoke-vpn-creds` | username | Run cert revocation script (with confirmation) |
| `/list-vpn-users` | `list-vpn-users` | none | Run cert list script |
| `/create-service-account` | `create-service-account` | name, needs-access-keys | Generate Terraform service account directory, register, plan |
| `/rotate-access-keys` | `rotate-access-keys` | IAM username | Create new key, deactivate old via AWS CLI |

## Skill File Structure

Each command follows a minimal structure — no `references/` directory needed:

```
skills/{command-name}/
├── SKILL.md
└── evals/
    └── evals.json
```

## Command Specifications

### `/create-iam-user`

**Input:** Full name (e.g., "João Silva"), group choice (developer or admin).

**Execution steps:**

1. Ask for full name and group (developer / admin / both).
2. Derive identifiers:
   - `user_name`: `joao.silva` (lowercase, dot-separated)
   - `directory`: `iam/users/joao-silva/` (lowercase, hyphen-separated)
   - `module_name`: `iam_users_joao_silva` (lowercase, underscore-separated)
3. Verify `iam/users/joao-silva/` does not already exist.
4. Write `iam/users/joao-silva/main.tf`:
   ```hcl
   module "joao_silva" {
     source    = "../../../modules/iam-developer-user"
     user_name = "joao.silva"
     groups    = [var.developers_group_name]
   }
   ```
   - If admin: `groups = [var.admin_group_name]`
   - If both: `groups = [var.developers_group_name, var.admin_group_name]`
5. Write `iam/users/joao-silva/variables.tf`:
   - Declare `developers_group_name` if developer or both
   - Declare `admin_group_name` if admin or both
6. Write `iam/users/joao-silva/outputs.tf`:
   ```hcl
   output "user_name" {
     value = module.joao_silva.user_name
   }
   output "user_arn" {
     value = module.joao_silva.user_arn
   }
   output "user_unique_id" {
     value = module.joao_silva.user_unique_id
   }
   ```
7. Add module block to `iam/main.tf`:
   ```hcl
   module "iam_users_joao_silva" {
     source                = "./users/joao-silva"
     developers_group_name = module.iam_groups_developers.group_name
   }
   ```
   - Pass `admin_group_name` if admin or both.
8. Run `cd iam && terraform validate`.
9. Run `cd iam && terraform plan -var-file=prod.tfvars`.
10. Hand off:
    ```
    Configuration ready for apply!

    Execute: cd iam && terraform apply -var-file=prod.tfvars
    Changes: New IAM user joao.silva in developers group
    Verification: aws iam get-user --user-name joao.silva
    ```

### `/remove-iam-user`

**Input:** Username (or interactive selection).

**Execution steps:**

1. List current users by reading directories under `iam/users/`.
2. Ask which user to remove (or accept as argument).
3. Confirm: "Remove user {name}? This will delete the Terraform config. Type 'yes' to confirm."
4. Remove the module block for this user from `iam/main.tf`.
5. Delete the directory `iam/users/{name}/`.
6. Run `cd iam && terraform plan -var-file=prod.tfvars`.
7. Hand off:
   ```
   Configuration ready for apply!

   Execute: cd iam && terraform apply -var-file=prod.tfvars
   Changes: Remove IAM user {user_name}
   Verification: aws iam get-user --user-name {user_name} (should return NoSuchEntity)
   ```
8. Remind: also revoke VPN credentials if the user has them (`/revoke-vpn-creds`).

### `/create-vpn-creds`

**Input:** Username (e.g., `joao-silva`).

**Execution steps:**

1. Ask for username if not provided.
2. Run `./scripts/generate-vpn-certs.sh client {name}`.
3. Run `./scripts/generate-vpn-certs.sh export {name}`.
4. Report:
   ```
   VPN credentials created!

   Config file: .vpn-pki/client-configs/{name}.ovpn
   Distribute via 1Password or encrypted channel — never send credentials over email or Slack.
   ```

### `/revoke-vpn-creds`

**Input:** Username.

**Execution steps:**

1. Run `./scripts/generate-vpn-certs.sh list` to show current certificates.
2. Ask which user to revoke (or accept as argument).
3. Confirm: "Revoke VPN access for {name}? Type 'yes' to confirm."
4. Run `./scripts/generate-vpn-certs.sh revoke {name}`.
5. Report result.

### `/list-vpn-users`

**Input:** None.

**Execution steps:**

1. Run `./scripts/generate-vpn-certs.sh list`.
2. Display formatted output.

### `/create-service-account`

**Input:** Service account name, whether it needs access keys.

**Execution steps:**

1. Ask for service account name (e.g., `my-service`) and whether it needs programmatic access keys (y/n).
2. Derive identifiers:
   - `directory`: `iam/service-accounts/{name}/`
   - `module_name`: `iam_service_accounts_{name_underscored}`
3. Verify directory does not already exist.
4. **If no access keys** (simple pattern):
   - Write `iam/service-accounts/{name}/main.tf`:
     ```hcl
     module "{name_underscored}" {
       source    = "../_base-module"
       user_name = "{name}"
     }
     ```
   - Write `iam/service-accounts/{name}/outputs.tf` with user_name, user_arn.
5. **If access keys** (full pattern):
   - Write `iam/service-accounts/{name}/main.tf` with:
     - `aws_iam_user` (path `/service-accounts/`, tags: UserType=ServiceAccount)
     - `aws_iam_access_key`
     - `aws_secretsmanager_secret` (name: `service-account-{name}-credentials`)
     - `aws_secretsmanager_secret_version` (stores access_key_id, secret_access_key, username, region)
   - Write `iam/service-accounts/{name}/variables.tf` with environment, region variables.
   - Write `iam/service-accounts/{name}/outputs.tf` with user_name, user_arn, secret_arn.
6. Add module block to `iam/main.tf`.
7. Run `cd iam && terraform validate`.
8. Run `cd iam && terraform plan -var-file=prod.tfvars`.
9. Hand off `terraform apply`.

### `/rotate-access-keys`

**Input:** IAM username.

**Execution steps:**

1. Ask for IAM username if not provided.
2. Run `aws iam list-access-keys --user-name {name}` to find current key(s).
3. If no keys found, report and stop.
4. Run `aws iam create-access-key --user-name {name}`.
5. Display new credentials:
   ```
   New access key created:
   Access Key ID: AKIA...
   Secret Access Key: ...

   Update all services using this account before deactivating the old key.
   ```
6. Ask: "Ready to deactivate old key {old-key-id}? Type 'yes' to confirm."
7. Run `aws iam update-access-key --user-name {name} --access-key-id {old-id} --status Inactive`.
8. Report:
   ```
   Old key {old-id} deactivated (not deleted).
   To delete it after verification: aws iam delete-access-key --user-name {name} --access-key-id {old-id}
   ```

## Bootstrap Registration

Update `skills/using-devops/SKILL.md` to add an **Operations Commands** section with a table and trigger rules:

| Command | Trigger |
|---------|---------|
| `create-iam-user` | "add user", "new developer", "new admin", "create IAM user" |
| `remove-iam-user` | "remove user", "delete user", "offboard" |
| `create-vpn-creds` | "create vpn", "vpn credentials", "vpn access", "generate vpn" |
| `revoke-vpn-creds` | "revoke vpn", "remove vpn access" |
| `list-vpn-users` | "list vpn", "vpn users", "who has vpn" |
| `create-service-account` | "create service account", "new service account" |
| `rotate-access-keys` | "rotate keys", "rotate access key", "new access key" |

## Safety Rules

All commands inherit the universal safety rules from bootstrap. Additional rules:

- **IAM commands (`create-iam-user`, `remove-iam-user`, `create-service-account`):** Never run `terraform apply` — always hand off to user.
- **VPN `revoke-vpn-creds`:** Requires explicit "yes" confirmation before revocation.
- **`rotate-access-keys`:** Always create new key before deactivating old. Never delete old key — only deactivate. Report deletion command for the user to run after verification.
- **All commands:** Verify current state before changes (check if user/cert/key already exists).

## Implementation Order

Build in this order (each command is independent, but this order maximizes reuse of patterns):

1. `/create-iam-user` — establishes the Terraform file generation pattern
2. `/remove-iam-user` — inverse of create, validates the pattern
3. `/create-service-account` — variant of create with two sub-patterns
4. `/create-vpn-creds` — first script-execution command
5. `/revoke-vpn-creds` — adds confirmation pattern
6. `/list-vpn-users` — simplest command, validates script execution
7. `/rotate-access-keys` — AWS CLI command pattern
8. Update bootstrap (`using-devops/SKILL.md`) — register all commands
9. Write evals for all commands
