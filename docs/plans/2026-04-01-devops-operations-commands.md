# DevOps Operations Commands Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 7 slash commands to the harumi-devops-plugin that execute daily IAM, VPN, and service account operations in the Harumi infrastructure repo.

**Architecture:** Each command is a skill with a `SKILL.md` playbook and `evals/evals.json`. The bootstrap skill (`using-devops`) registers all commands with trigger rules. Commands execute operations directly (file generation, script execution, AWS CLI) and hand off destructive Terraform operations to the user.

**Tech Stack:** SKILL.md (YAML frontmatter + Markdown), JSON evals, Terraform HCL templates, Bash scripts, AWS CLI

**Spec:** `docs/specs/2026-04-01-devops-operations-commands-design.md`

---

## File Map

**New files (7 skills x 2 files each + bootstrap update):**

| File | Responsibility |
|------|---------------|
| `skills/create-iam-user/SKILL.md` | Playbook: create IAM developer/admin user |
| `skills/create-iam-user/evals/evals.json` | Eval: create user scenario |
| `skills/remove-iam-user/SKILL.md` | Playbook: remove IAM user |
| `skills/remove-iam-user/evals/evals.json` | Eval: remove user scenario |
| `skills/create-vpn-creds/SKILL.md` | Playbook: generate VPN client cert + export |
| `skills/create-vpn-creds/evals/evals.json` | Eval: create VPN creds scenario |
| `skills/revoke-vpn-creds/SKILL.md` | Playbook: revoke VPN certificate |
| `skills/revoke-vpn-creds/evals/evals.json` | Eval: revoke VPN creds scenario |
| `skills/list-vpn-users/SKILL.md` | Playbook: list VPN certificates |
| `skills/list-vpn-users/evals/evals.json` | Eval: list VPN users scenario |
| `skills/create-service-account/SKILL.md` | Playbook: create IAM service account |
| `skills/create-service-account/evals/evals.json` | Eval: create service account scenario |
| `skills/rotate-access-keys/SKILL.md` | Playbook: rotate IAM access keys |
| `skills/rotate-access-keys/evals/evals.json` | Eval: rotate keys scenario |

**Modified files:**

| File | Change |
|------|--------|
| `skills/using-devops/SKILL.md` | Add Operations Commands table and trigger rules |

---

### Task 1: Create `/create-iam-user` skill

**Files:**
- Create: `skills/create-iam-user/SKILL.md`
- Create: `skills/create-iam-user/evals/evals.json`

- [ ] **Step 1: Create SKILL.md**

Write `skills/create-iam-user/SKILL.md`:

```markdown
---
name: create-iam-user
description: "Create a new IAM developer or admin user in the Harumi infrastructure repo. Generates Terraform files, registers the module, and runs terraform plan. Use when: user wants to add a new AWS developer, admin, or contributor."
---

# Create IAM User

Create a new IAM user by generating Terraform files and registering the module.

## Inputs

Ask for these if not provided:

1. **Full name** (e.g., "João Silva")
2. **Group**: developer, admin, contributor, or a combination (e.g., "developer + admin")

## Naming Derivation

From the full name, derive:
- `user_name`: lowercase, dot-separated (e.g., `joao.silva`)
- `directory_name`: lowercase, hyphen-separated (e.g., `joao-silva`)
- `module_suffix`: lowercase, underscore-separated (e.g., `joao_silva`)
- `module_label`: same as `module_suffix` — used inside the user directory's `main.tf` (e.g., `module "joao_silva"`)

Handle accented characters by removing accents (e.g., "João" → "joao").

## Execution Steps

Follow these steps exactly. Do not skip or reorder.

### Step 1: Verify user does not already exist

Check that `iam/users/{directory_name}/` does not already exist. If it does, report the conflict and stop.

### Step 2: Create user directory

Create `iam/users/{directory_name}/` with three files:

**main.tf:**
```hcl
module "{module_label}" {
  source    = "../../../modules/iam-developer-user"
  user_name = "{user_name}"
  groups    = [{group_list}]
}
```

Where `{group_list}` depends on the group choice:
- developer: `var.developers_group_name`
- admin: `var.admin_group_name`
- contributor: `var.contributors_group_name`
- developer + admin: `var.developers_group_name, var.admin_group_name`

**variables.tf:**

Declare only the variables needed for the chosen group(s):

```hcl
variable "developers_group_name" {
  description = "Name of the developers IAM group"
  type        = string
}
```

Repeat for `admin_group_name` and/or `contributors_group_name` as needed.

**outputs.tf:**
```hcl
output "user_name" {
  value = module.{module_label}.user_name
}

output "user_arn" {
  value = module.{module_label}.user_arn
}

output "user_unique_id" {
  value = module.{module_label}.user_unique_id
}
```

### Step 3: Register module in iam/main.tf

Add a module block to `iam/main.tf` under the appropriate section comment (`## Developer Users`, `## Admin Users`, or both):

```hcl
module "iam_users_{module_suffix}" {
  source = "./users/{directory_name}"

  developers_group_name = module.iam_groups_developers.group_name
}
```

Pass only the group variables that match the chosen group(s):
- developer: `developers_group_name = module.iam_groups_developers.group_name`
- admin: `admin_group_name = module.iam_groups_admin.group_name`
- contributor: `contributors_group_name = module.iam_groups_contributors.group_name`
- Combinations pass multiple variables.

Place the module block in the correct section based on the primary group. Follow the existing ordering pattern in the file.

### Step 4: Validate and plan

```bash
cd iam && terraform validate
cd iam && terraform plan -var-file=prod.tfvars
```

### Step 5: Hand off apply

```
Configuration ready for apply!

Execute: cd iam && terraform apply -var-file=prod.tfvars
Changes: New IAM user {user_name} in {group} group
Verification: aws iam get-user --user-name {user_name}
```

Remind the user to also create VPN credentials if needed: "Run `/create-vpn-creds` to generate VPN access for this user."
```

- [ ] **Step 2: Create evals**

Write `skills/create-iam-user/evals/evals.json`:

```json
{
  "skill_name": "create-iam-user",
  "evals": [
    {
      "id": 1,
      "prompt": "Create a new developer user for João Silva.",
      "expected_output": "Creates iam/users/joao-silva/ directory with main.tf, variables.tf, outputs.tf. Adds module block to iam/main.tf with developers_group_name. Runs terraform validate and plan. Ends with apply handoff.",
      "files": [],
      "expectations": [
        "Derives user_name as joao.silva",
        "Derives directory name as joao-silva",
        "Creates main.tf referencing iam-developer-user module with groups = [var.developers_group_name]",
        "Creates variables.tf declaring developers_group_name variable",
        "Creates outputs.tf with user_name, user_arn, user_unique_id",
        "Adds module iam_users_joao_silva to iam/main.tf passing developers_group_name",
        "Runs terraform validate",
        "Runs terraform plan -var-file=prod.tfvars",
        "Does NOT run terraform apply",
        "Ends with apply handoff block"
      ]
    },
    {
      "id": 2,
      "prompt": "Add André Costa as both developer and admin.",
      "expected_output": "Creates user with both groups. variables.tf declares both developers_group_name and admin_group_name. Module block passes both group variables.",
      "files": [],
      "expectations": [
        "Derives user_name as andre.costa",
        "Creates main.tf with groups = [var.developers_group_name, var.admin_group_name]",
        "Creates variables.tf declaring both developers_group_name and admin_group_name",
        "Adds module to iam/main.tf passing both group name variables",
        "Does NOT run terraform apply"
      ]
    }
  ]
}
```

- [ ] **Step 3: Commit**

```bash
git add skills/create-iam-user/
git commit -m "feat: add create-iam-user skill"
```

---

### Task 2: Create `/remove-iam-user` skill

**Files:**
- Create: `skills/remove-iam-user/SKILL.md`
- Create: `skills/remove-iam-user/evals/evals.json`

- [ ] **Step 1: Create SKILL.md**

Write `skills/remove-iam-user/SKILL.md`:

```markdown
---
name: remove-iam-user
description: "Remove an IAM user from the Harumi infrastructure repo. Deletes Terraform files, removes the module registration, and runs terraform plan. Use when: user wants to remove, delete, or offboard an AWS user."
---

# Remove IAM User

Remove an IAM user by deleting their Terraform directory and module registration.

## Inputs

Ask for the username if not provided. Optionally list current users first.

## Execution Steps

### Step 1: List current users

Read the directories under `iam/users/` and display them:

```
Current IAM users:
- andre-koga
- erick-jesus
- italo-rocha
- leandro-bandeira
- marcel-nicolay
- miriam-koga
- rafael-chuluc
- ricardo-leao
- wagner-souza
```

Ask which user to remove (or accept from the user's original request).

### Step 2: Verify user exists

Check that `iam/users/{directory_name}/` exists. If not, report and stop.

### Step 3: Confirm removal

Ask for explicit confirmation:

```
Remove user {user_name}? This will:
- Delete iam/users/{directory_name}/ directory
- Remove the module block from iam/main.tf
- On apply: delete the IAM user from AWS

Type 'yes' to confirm.
```

Do NOT proceed without "yes".

### Step 4: Remove module from iam/main.tf

Find and remove the `module "iam_users_{module_suffix}"` block from `iam/main.tf`. Remove the entire block including all its arguments and closing brace.

### Step 5: Delete user directory

Delete the entire `iam/users/{directory_name}/` directory.

### Step 6: Plan

```bash
cd iam && terraform plan -var-file=prod.tfvars
```

### Step 7: Hand off apply

```
Configuration ready for apply!

Execute: cd iam && terraform apply -var-file=prod.tfvars
Changes: Remove IAM user {user_name}
Verification: aws iam get-user --user-name {user_name} (should return NoSuchEntity)
```

Remind: "If this user has VPN access, also run `/revoke-vpn-creds` to revoke their certificate."
```

- [ ] **Step 2: Create evals**

Write `skills/remove-iam-user/evals/evals.json`:

```json
{
  "skill_name": "remove-iam-user",
  "evals": [
    {
      "id": 1,
      "prompt": "Remove the IAM user italo-rocha. Yes, I confirm.",
      "expected_output": "Verifies user directory exists. Removes module block from iam/main.tf. Deletes iam/users/italo-rocha/ directory. Runs terraform plan. Ends with apply handoff.",
      "files": [],
      "expectations": [
        "Checks that iam/users/italo-rocha/ exists",
        "Asks for confirmation before proceeding (or accepts the pre-given confirmation)",
        "Removes the module iam_users_italo_rocha block from iam/main.tf",
        "Deletes the iam/users/italo-rocha/ directory",
        "Runs terraform plan -var-file=prod.tfvars",
        "Does NOT run terraform apply",
        "Ends with apply handoff block",
        "Reminds about revoking VPN credentials"
      ]
    }
  ]
}
```

- [ ] **Step 3: Commit**

```bash
git add skills/remove-iam-user/
git commit -m "feat: add remove-iam-user skill"
```

---

### Task 3: Create `/create-service-account` skill

**Files:**
- Create: `skills/create-service-account/SKILL.md`
- Create: `skills/create-service-account/evals/evals.json`

- [ ] **Step 1: Create SKILL.md**

Write `skills/create-service-account/SKILL.md`:

```markdown
---
name: create-service-account
description: "Create a new IAM service account in the Harumi infrastructure repo. Supports simple (no keys) and full (access keys + Secrets Manager) patterns. Use when: user wants to create a new AWS service account."
---

# Create Service Account

Create a new IAM service account. Two patterns available based on whether the account needs programmatic access keys.

## Inputs

Ask for these if not provided:

1. **Service account name** (e.g., `my-service`)
2. **Needs access keys?** (yes/no) — determines which pattern to use

## Naming Derivation

From the service account name:
- `directory_name`: the name as-is, hyphenated (e.g., `my-service`)
- `module_suffix`: underscored (e.g., `my_service`)

## Execution Steps

### Step 1: Verify account does not already exist

Check that `iam/service-accounts/{directory_name}/` does not already exist. If it does, report the conflict and stop.

### Step 2: Create service account directory

**Pattern A: No access keys (simple)**

Create `iam/service-accounts/{directory_name}/main.tf`:

```hcl
module "{module_suffix}" {
  source    = "../_base-module"
  user_name = "{name}"
}
```

Create `iam/service-accounts/{directory_name}/outputs.tf`:

```hcl
output "user_name" {
  value = module.{module_suffix}.user_name
}

output "user_arn" {
  value = module.{module_suffix}.user_arn
}
```

**Pattern B: With access keys (full)**

Create `iam/service-accounts/{directory_name}/main.tf`:

```hcl
resource "aws_iam_user" "this" {
  name = "{name}"
  path = "/service-accounts/"

  tags = {
    Project     = "harumi"
    UserType    = "ServiceAccount"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_iam_access_key" "this" {
  user = aws_iam_user.this.name
}

resource "aws_secretsmanager_secret" "credentials" {
  name                    = "service-account-{name}-credentials"
  description             = "{name} service account AWS credentials"
  recovery_window_in_days = 7

  tags = {
    Project        = "harumi"
    Environment    = var.environment
    ManagedBy      = "terraform"
    ServiceAccount = "{name}"
  }
}

resource "aws_secretsmanager_secret_version" "credentials" {
  secret_id = aws_secretsmanager_secret.credentials.id
  secret_string = jsonencode({
    access_key_id     = aws_iam_access_key.this.id
    secret_access_key = aws_iam_access_key.this.secret
    username          = aws_iam_user.this.name
    region            = var.region
  })

  lifecycle {
    ignore_changes = [secret_string]
  }
}
```

Create `iam/service-accounts/{directory_name}/variables.tf`:

```hcl
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "region" {
  description = "AWS region"
  type        = string
}
```

Create `iam/service-accounts/{directory_name}/outputs.tf`:

```hcl
output "user_name" {
  value = aws_iam_user.this.name
}

output "user_arn" {
  value = aws_iam_user.this.arn
}

output "secret_arn" {
  value = aws_secretsmanager_secret.credentials.arn
}
```

### Step 3: Register module in iam/main.tf

Add a module block under the `## Service Accounts` section:

**Pattern A (no keys):**
```hcl
module "iam_service_accounts_{module_suffix}" {
  source = "./service-accounts/{directory_name}"
}
```

**Pattern B (with keys):**
```hcl
module "iam_service_accounts_{module_suffix}" {
  source = "./service-accounts/{directory_name}"

  environment = var.environment
  region      = var.region
}
```

### Step 4: Validate and plan

```bash
cd iam && terraform validate
cd iam && terraform plan -var-file=prod.tfvars
```

### Step 5: Hand off apply

```
Configuration ready for apply!

Execute: cd iam && terraform apply -var-file=prod.tfvars
Changes: New service account {name} [with/without] access keys
Verification: aws iam get-user --user-name {name}
```

If Pattern B was used, add: "After apply, retrieve credentials from Secrets Manager: `aws secretsmanager get-secret-value --secret-id service-account-{name}-credentials`"
```

- [ ] **Step 2: Create evals**

Write `skills/create-service-account/evals/evals.json`:

```json
{
  "skill_name": "create-service-account",
  "evals": [
    {
      "id": 1,
      "prompt": "Create a service account called data-exporter. It does not need access keys.",
      "expected_output": "Creates iam/service-accounts/data-exporter/ with main.tf using _base-module pattern. Adds module to iam/main.tf. Runs terraform plan. Handoff.",
      "files": [],
      "expectations": [
        "Creates main.tf using the _base-module source pattern",
        "Creates outputs.tf with user_name and user_arn",
        "Does NOT create access keys or Secrets Manager resources",
        "Adds module iam_service_accounts_data_exporter to iam/main.tf",
        "Runs terraform validate and plan",
        "Does NOT run terraform apply",
        "Ends with apply handoff block"
      ]
    },
    {
      "id": 2,
      "prompt": "Create a service account called etl-pipeline that needs programmatic access keys.",
      "expected_output": "Creates full pattern with aws_iam_user, aws_iam_access_key, aws_secretsmanager_secret. Module block passes environment and region. Handoff.",
      "files": [],
      "expectations": [
        "Creates main.tf with aws_iam_user, aws_iam_access_key, aws_secretsmanager_secret, aws_secretsmanager_secret_version",
        "Creates variables.tf with environment and region variables",
        "Creates outputs.tf with user_name, user_arn, secret_arn",
        "Secret name follows pattern service-account-etl-pipeline-credentials",
        "Adds module to iam/main.tf passing environment and region",
        "Does NOT run terraform apply",
        "Ends with apply handoff block"
      ]
    }
  ]
}
```

- [ ] **Step 3: Commit**

```bash
git add skills/create-service-account/
git commit -m "feat: add create-service-account skill"
```

---

### Task 4: Create `/create-vpn-creds` skill

**Files:**
- Create: `skills/create-vpn-creds/SKILL.md`
- Create: `skills/create-vpn-creds/evals/evals.json`

- [ ] **Step 1: Create SKILL.md**

Write `skills/create-vpn-creds/SKILL.md`:

```markdown
---
name: create-vpn-creds
description: "Generate VPN client certificate and export .ovpn config file for a user. Use when: user wants to create VPN credentials, generate VPN access, or set up VPN for someone."
---

# Create VPN Credentials

Generate a VPN client certificate and export the `.ovpn` configuration file.

## Inputs

Ask for the username if not provided. Use the hyphenated format (e.g., `joao-silva`).

## Execution Steps

### Step 1: Generate client certificate

```bash
./scripts/generate-vpn-certs.sh client {username}
```

If the script reports the certificate already exists and asks about regeneration, relay the question to the user.

### Step 2: Export .ovpn config

```bash
./scripts/generate-vpn-certs.sh export {username}
```

### Step 3: Report

```
VPN credentials created!

Config file: .vpn-pki/client-configs/{username}.ovpn
Distribute via 1Password or encrypted channel — never send credentials over email or Slack.

The user should import the .ovpn file into the AWS VPN Client app.
```
```

- [ ] **Step 2: Create evals**

Write `skills/create-vpn-creds/evals/evals.json`:

```json
{
  "skill_name": "create-vpn-creds",
  "evals": [
    {
      "id": 1,
      "prompt": "Create VPN credentials for joao-silva.",
      "expected_output": "Runs generate-vpn-certs.sh client joao-silva, then export joao-silva. Reports the .ovpn file path and distribution instructions.",
      "files": [],
      "expectations": [
        "Runs ./scripts/generate-vpn-certs.sh client joao-silva",
        "Runs ./scripts/generate-vpn-certs.sh export joao-silva",
        "Reports the .ovpn file location at .vpn-pki/client-configs/joao-silva.ovpn",
        "Includes security reminder about distribution (1Password or encrypted channel)",
        "Does NOT send the .ovpn file contents in the chat"
      ]
    }
  ]
}
```

- [ ] **Step 3: Commit**

```bash
git add skills/create-vpn-creds/
git commit -m "feat: add create-vpn-creds skill"
```

---

### Task 5: Create `/revoke-vpn-creds` skill

**Files:**
- Create: `skills/revoke-vpn-creds/SKILL.md`
- Create: `skills/revoke-vpn-creds/evals/evals.json`

- [ ] **Step 1: Create SKILL.md**

Write `skills/revoke-vpn-creds/SKILL.md`:

```markdown
---
name: revoke-vpn-creds
description: "Revoke a VPN client certificate. Use when: user wants to revoke VPN access, remove VPN credentials, or disable VPN for someone."
---

# Revoke VPN Credentials

Revoke a VPN client certificate, removing the user's ability to connect.

## Inputs

Ask for the username if not provided.

## Execution Steps

### Step 1: List current certificates

Run the list command to show who currently has VPN access:

```bash
./scripts/generate-vpn-certs.sh list
```

Display the output to the user.

### Step 2: Confirm revocation

Ask for explicit confirmation:

```
Revoke VPN access for {username}? This will invalidate their certificate.
Type 'yes' to confirm.
```

Do NOT proceed without "yes".

### Step 3: Revoke certificate

```bash
./scripts/generate-vpn-certs.sh revoke {username}
```

### Step 4: Report

Report the result of the revocation command.
```

- [ ] **Step 2: Create evals**

Write `skills/revoke-vpn-creds/evals/evals.json`:

```json
{
  "skill_name": "revoke-vpn-creds",
  "evals": [
    {
      "id": 1,
      "prompt": "Revoke VPN access for erick-jesus. Yes, I confirm.",
      "expected_output": "Lists current certs, confirms revocation, runs revoke command.",
      "files": [],
      "expectations": [
        "Runs ./scripts/generate-vpn-certs.sh list to show current certificates",
        "Asks for confirmation before revoking (or accepts the pre-given confirmation)",
        "Runs ./scripts/generate-vpn-certs.sh revoke erick-jesus",
        "Reports the result of the revocation"
      ]
    }
  ]
}
```

- [ ] **Step 3: Commit**

```bash
git add skills/revoke-vpn-creds/
git commit -m "feat: add revoke-vpn-creds skill"
```

---

### Task 6: Create `/list-vpn-users` skill

**Files:**
- Create: `skills/list-vpn-users/SKILL.md`
- Create: `skills/list-vpn-users/evals/evals.json`

- [ ] **Step 1: Create SKILL.md**

Write `skills/list-vpn-users/SKILL.md`:

```markdown
---
name: list-vpn-users
description: "List all VPN client certificates and their status. Use when: user wants to see VPN users, list VPN certificates, or check who has VPN access."
---

# List VPN Users

List all issued VPN client certificates and their validity status.

## Execution Steps

### Step 1: List certificates

```bash
./scripts/generate-vpn-certs.sh list
```

### Step 2: Display results

Display the formatted output showing all certificates with their status and validity dates.
```

- [ ] **Step 2: Create evals**

Write `skills/list-vpn-users/evals/evals.json`:

```json
{
  "skill_name": "list-vpn-users",
  "evals": [
    {
      "id": 1,
      "prompt": "Who has VPN access?",
      "expected_output": "Runs the list command and displays certificate information.",
      "files": [],
      "expectations": [
        "Runs ./scripts/generate-vpn-certs.sh list",
        "Displays the output showing certificates and their status"
      ]
    }
  ]
}
```

- [ ] **Step 3: Commit**

```bash
git add skills/list-vpn-users/
git commit -m "feat: add list-vpn-users skill"
```

---

### Task 7: Create `/rotate-access-keys` skill

**Files:**
- Create: `skills/rotate-access-keys/SKILL.md`
- Create: `skills/rotate-access-keys/evals/evals.json`

- [ ] **Step 1: Create SKILL.md**

Write `skills/rotate-access-keys/SKILL.md`:

```markdown
---
name: rotate-access-keys
description: "Rotate IAM access keys for a user or service account. Creates new key, deactivates old key. Use when: user wants to rotate access keys, renew credentials, or replace an access key."
---

# Rotate Access Keys

Rotate IAM access keys for a user or service account. Creates a new key before deactivating the old one to avoid downtime.

## Inputs

Ask for the IAM username if not provided.

## Execution Steps

### Step 1: List current access keys

```bash
aws iam list-access-keys --user-name {username}
```

If no keys found, report: "No access keys found for {username}. Nothing to rotate." and stop.

If the user already has 2 access keys (AWS maximum), report:

```
User {username} already has 2 access keys (AWS limit).
You must delete one before creating a new one.

Keys:
- {key1_id} (Status: {status}, Created: {date})
- {key2_id} (Status: {status}, Created: {date})

Which key should I delete first?
```

Wait for the user's choice, then delete that key before proceeding:

```bash
aws iam delete-access-key --user-name {username} --access-key-id {chosen_key_id}
```

### Step 2: Create new access key

```bash
aws iam create-access-key --user-name {username}
```

### Step 3: Display new credentials

```
New access key created:

Access Key ID:     {new_access_key_id}
Secret Access Key: {new_secret_access_key}

IMPORTANT: Save these credentials now. The secret access key cannot be retrieved again.
Update all services using this account before deactivating the old key.
```

### Step 4: Confirm old key deactivation

If there was a previous key still active:

```
Ready to deactivate old key {old_key_id}? Make sure all services have been updated first.
Type 'yes' to deactivate.
```

Do NOT proceed without "yes".

### Step 5: Deactivate old key

```bash
aws iam update-access-key --user-name {username} --access-key-id {old_key_id} --status Inactive
```

### Step 6: Report

```
Old key {old_key_id} deactivated (not deleted).

To delete it permanently after verification:
aws iam delete-access-key --user-name {username} --access-key-id {old_key_id}
```
```

- [ ] **Step 2: Create evals**

Write `skills/rotate-access-keys/evals/evals.json`:

```json
{
  "skill_name": "rotate-access-keys",
  "evals": [
    {
      "id": 1,
      "prompt": "Rotate the access keys for the airbyte-user-production service account. Yes, deactivate the old key.",
      "expected_output": "Lists current keys, creates new key, displays credentials, deactivates old key. Does not delete old key.",
      "files": [],
      "expectations": [
        "Runs aws iam list-access-keys to find current keys",
        "Runs aws iam create-access-key to create new key",
        "Displays the new access key ID and secret access key",
        "Warns to save credentials and update services",
        "Asks for confirmation before deactivating old key (or accepts pre-given confirmation)",
        "Runs aws iam update-access-key with --status Inactive to deactivate old key",
        "Does NOT delete the old key — only deactivates",
        "Provides the delete command for the user to run later"
      ]
    }
  ]
}
```

- [ ] **Step 3: Commit**

```bash
git add skills/rotate-access-keys/
git commit -m "feat: add rotate-access-keys skill"
```

---

### Task 8: Update bootstrap skill with Operations Commands

**Files:**
- Modify: `skills/using-devops/SKILL.md`

- [ ] **Step 1: Add Operations Commands section**

In `skills/using-devops/SKILL.md`, add a new section after the existing `## Available Skills` table. Insert before `**Future skills**`:

```markdown
## Operations Commands

Quick-action skills for daily DevOps operations. Use the Skill tool to invoke:

| Command | Use When |
|---------|----------|
| `harumi-devops-plugin:create-iam-user` | Add a new developer, admin, or contributor user |
| `harumi-devops-plugin:remove-iam-user` | Remove / offboard an IAM user |
| `harumi-devops-plugin:create-vpn-creds` | Generate VPN certificate and .ovpn config |
| `harumi-devops-plugin:revoke-vpn-creds` | Revoke a VPN certificate |
| `harumi-devops-plugin:list-vpn-users` | List active VPN certificates |
| `harumi-devops-plugin:create-service-account` | Create a new IAM service account |
| `harumi-devops-plugin:rotate-access-keys` | Rotate IAM access keys for a user/service account |
```

- [ ] **Step 2: Add trigger rules**

Add a new trigger rules section after the existing trigger rules:

```markdown
Invoke `harumi-devops-plugin:create-iam-user` when:
- User wants to add, create, or onboard a new AWS user (developer, admin, contributor)

Invoke `harumi-devops-plugin:remove-iam-user` when:
- User wants to remove, delete, or offboard an IAM user

Invoke `harumi-devops-plugin:create-vpn-creds` when:
- User wants to create, generate, or set up VPN credentials or access

Invoke `harumi-devops-plugin:revoke-vpn-creds` when:
- User wants to revoke, remove, or disable VPN access

Invoke `harumi-devops-plugin:list-vpn-users` when:
- User wants to list VPN users, see who has VPN access, or check VPN certificates

Invoke `harumi-devops-plugin:create-service-account` when:
- User wants to create a new service account or programmatic IAM user

Invoke `harumi-devops-plugin:rotate-access-keys` when:
- User wants to rotate, renew, or replace IAM access keys
```

- [ ] **Step 3: Remove operations commands from future skills list**

Remove `security-operations` from the future skills list since its core features are now covered by the operations commands.

- [ ] **Step 4: Commit**

```bash
git add skills/using-devops/SKILL.md
git commit -m "feat: register operations commands in bootstrap skill"
```

---

### Task 9: Update CHANGELOG

**Files:**
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Add v0.3.0 entry**

Read `CHANGELOG.md` and add a new version entry at the top following the existing format. The entry should document all 7 new operations commands added in this release.

- [ ] **Step 2: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: update changelog with operations commands"
```
