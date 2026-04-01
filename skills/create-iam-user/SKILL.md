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
