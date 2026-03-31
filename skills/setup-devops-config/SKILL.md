---
name: setup-devops-config
description: "Generate a .devops.yaml configuration file for the claude-devops-plugin by analyzing the current repository. Use when: the user asks to create, generate, or set up .devops.yaml, or when no .devops.yaml exists and the user is configuring the plugin for a new repo."
---

# Setup DevOps Config

Generate a `.devops.yaml` configuration file for this repository by analyzing existing files.

## Your Process

Follow these steps in order. Do not skip steps or ask questions before completing detection.

### Step 1: Detect stack from codebase

Use Glob and Grep tools to detect each field. Record confident values and a list of unknowns.

**Detection rules:**

| Field | How to detect | Default if not found |
|-------|--------------|----------------------|
| `cloud.provider` | Grep `**/*.tf` for `provider "aws"` / `provider "google"` / `provider "azurerm"` | ask |
| `cloud.region` | Grep `**/*.tf`, `**/*.tfvars` for `region` assignment or default | ask |
| `cloud.account_alias` | Grep `**/*.tf`, `**/*.tfvars` for `namespace` value | ask |
| `terraform.version` | Read `.terraform-version`; or Grep `**/versions.tf` for `required_version` | ask |
| `terraform.state_backend` | Grep `**/backend.tf`, `**/*.tf` for `backend "s3"` / `backend "gcs"` / `backend "azurerm"` | ask |
| `terraform.var_file` | Glob `**/*.tfvars`; prefer `prod.tfvars` → `production.tfvars` → first match | ask |
| `kubernetes.tool` | Default `kubectl`; Grep CI/CD configs for `oc ` (OpenShift) | `kubectl` |
| `kubernetes.gitops` | Glob `**/Application.yaml` with `kind: Application` (ArgoCD); `**/HelmRelease.yaml` (Flux) | `none` |
| `kubernetes.clusters` | Grep K8s manifests, CI/CD configs for cluster names or `--context` flags | ask if k8s detected, else omit |
| `cicd.platform` | Glob `.github/workflows/*.yml` → `github-actions`; `.gitlab-ci.yml` → `gitlab-ci`; `.circleci/config.yml` → `circleci` | ask |
| `observability.metrics` | Glob/Grep for `prometheus.yml`, `prometheus` in compose/K8s; `datadog` agent configs | `prometheus` |
| `observability.dashboards` | Glob for `grafana/` directory or `grafana` in compose/K8s | `grafana` |
| `observability.logs` | Grep configs for `loki`, `cloudwatch`, `datadog` | `loki` |
| `observability.traces` | Grep configs for `tempo`, `jaeger`, `xray` | `tempo` |
| `containers.runtime` | Glob `**/Dockerfile` → `docker`; Grep scripts/CI for `podman` | `docker` |
| `containers.registry` | Grep CI/CD, Dockerfiles for `*.dkr.ecr.*.amazonaws.com` → `ecr`; `gcr.io` → `gcr`; `ghcr.io` → `ghcr`; `docker.io` → `dockerhub` | ask |
| `naming.pattern` | Grep `**/*.tf` for CloudPosse `namespace`/`stage`/`name` pattern | `{namespace}-{stage}-{name}` |
| `naming.namespace` | Grep `**/*.tf`, `**/*.tfvars` for `namespace` value | ask |
| `naming.stage` | Grep `**/*.tfvars` for `stage` or `environment` value | `production` |

**Confidence rule:** If a field appears with a single clear value → use it and note the source file. If multiple conflicting values are found → treat as unknown and ask. If no evidence → use the default from the table; if default is `ask`, add to unknowns list.

**Omit the `kubernetes` section entirely** if no K8s manifests, Helm charts, or K8s-related CI/CD steps are found.

**Omit the `observability` section** if no monitoring configs are found and all defaults would apply — note this in the preview instead.

### Step 2: Ask about unknowns

For each field in the unknowns list, ask **one question at a time**. Be specific:

- "I couldn't detect your AWS region from the codebase. What region does this project use? (e.g. us-east-1)"
- "What Terraform state backend does this project use? (s3 / gcs / azurerm)"
- "What container registry does this project push images to? (ecr / gcr / ghcr / dockerhub)"

Wait for each answer before asking the next.

### Step 3: Check for existing `.devops.yaml`

Before showing the preview, check if `.devops.yaml` exists in the current working directory.

- If it **does not exist**: proceed to Step 4.
- If it **exists**: read it and show what would change:

```
An existing .devops.yaml was found. Here is what would change:

  cloud.region: us-east-1  →  us-east-2   (detected from infrastructure/prod.tfvars)
  terraform.var_file: (not set)  →  prod.tfvars  (detected from infrastructure/)
  [unchanged fields omitted]
```

### Step 4: Show preview

Print the full proposed `.devops.yaml`. Annotate each detected value with the source file as a comment. Omit comments for defaulted values.

Example:

```yaml
cloud:
  provider: aws        # detected: infrastructure/main.tf
  region: us-east-2   # detected: infrastructure/prod.tfvars
  account_alias: harumi  # detected: infrastructure/prod.tfvars

terraform:
  version: "1.5.7"    # detected: infrastructure/.terraform-version
  state_backend: s3   # detected: infrastructure/backend.tf
  var_file: prod.tfvars  # detected: infrastructure/prod.tfvars

cicd:
  platform: github-actions  # detected: .github/workflows/

containers:
  runtime: docker     # detected: Dockerfile
  registry: ecr       # detected: .github/workflows/deploy.yml

naming:
  pattern: "{namespace}-{stage}-{name}"
  namespace: harumi   # detected: infrastructure/prod.tfvars
  stage: production   # detected: infrastructure/prod.tfvars
```

Then ask:

> Write this to `.devops.yaml`? Reply **yes** to write, **edit** to paste corrections, or **cancel** to abort.

### Step 5: Handle response

**If "yes":**
- Write the file (without the source comments — clean YAML only)
- Report: "`.devops.yaml` written successfully."
- Add: "The plugin will pick up this config on your next session start. Run `/clear` to reload now."

**If "edit":**
- Ask: "Paste the corrected YAML below:"
- Accept the user's YAML, show it as the new preview, ask again: "Write this to `.devops.yaml`? (yes / cancel)"

**If "cancel":**
- Report: "Cancelled. No file was written."
- Do nothing further.

## What NOT to do

- Do not write the file without explicit "yes" confirmation
- Do not ask all unknown fields at once — one question at a time
- Do not include source file comments in the written file — preview only
- Do not invent values; if genuinely unknown and no default exists, ask
