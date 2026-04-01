# Setup DevOps Config Skill — Design Spec

**Date:** 2026-03-31
**Status:** Approved

## Overview

A new skill for the `harumi-devops-plugin` that generates a `.devops.yaml` configuration file by analyzing the current repository's codebase. It auto-detects the tech stack from existing files, asks targeted questions only for fields it cannot determine, shows a preview, and writes the file only after explicit user confirmation.

## Goals

- Help users get a correct `.devops.yaml` on the first try without manually reading the default config
- Auto-detect as much as possible from the codebase to minimize questions
- Never write the file without explicit user confirmation
- Handle the case where `.devops.yaml` already exists (show a diff, not just the new file)

## Non-Goals

- Validating the written `.devops.yaml` against a schema
- Auto-applying the config or reloading the session automatically
- Detecting stack from runtime environments (only static file analysis)

---

## Section 1: Skill Structure

**File:** `skills/setup-devops-config/SKILL.md`

No `references/` subdirectory — the skill is self-contained.

**Registration:** `skills/using-devops/SKILL.md` is updated to add this skill to the available skills table with trigger:

> Invoke `harumi-devops-plugin:setup-devops-config` when: the user asks to create, generate, or set up `.devops.yaml`, or when no `.devops.yaml` exists and the user is setting up the plugin for the first time.

---

## Section 2: Detection Logic

The skill instructs Claude to use its file-reading tools (`Glob`, `Grep`, `Read`) to inspect the codebase. Detection runs in one pass before any questions are asked.

| Field | Detection method |
|-------|-----------------|
| `cloud.provider` | Grep `*.tf` for `provider "aws"`, `provider "google"`, `provider "azurerm"` |
| `cloud.region` | Grep `*.tf`, `*.tfvars` for `region` variable/default value |
| `cloud.account_alias` | Grep `*.tf`, `*.tfvars` for `namespace` or top-level project name |
| `terraform.version` | Read `.terraform-version`; or grep `required_version` in `versions.tf` |
| `terraform.state_backend` | Grep `backend.tf`/`*.tf` for `backend "s3"`, `backend "gcs"`, `backend "azurerm"` |
| `terraform.var_file` | Glob `*.tfvars`; prefer `prod.tfvars` > `production.tfvars` > first match |
| `kubernetes.tool` | Default `kubectl`; grep for `oc` (OpenShift) hints |
| `kubernetes.gitops` | Glob for `Application.yaml` with `kind: Application` (ArgoCD); `HelmRelease.yaml` (Flux); default `none` |
| `kubernetes.clusters` | Grep K8s manifests or CI/CD for cluster names/contexts |
| `cicd.platform` | Glob `.github/workflows/*.yml` (GitHub Actions); `.gitlab-ci.yml` (GitLab); `.circleci/config.yml` (CircleCI) |
| `observability.metrics` | Glob/grep for `prometheus.yml`, `prometheus` in compose/K8s; `datadog` agent; default `prometheus` |
| `observability.dashboards` | Glob for `grafana` directories or configs |
| `observability.logs` | Glob/grep for `loki`, `cloudwatch`, `datadog` in configs |
| `observability.traces` | Glob/grep for `tempo`, `jaeger`, `xray` in configs |
| `containers.runtime` | Glob for `Dockerfile` (docker); grep for `podman` in scripts/CI |
| `containers.registry` | Grep CI/CD configs and Dockerfiles for ECR URL pattern (`*.ecr.*.amazonaws.com`), `gcr.io`, `ghcr.io`, `docker.io` |
| `naming.pattern` | Grep `*.tf` for CloudPosse `namespace`/`stage`/`name` pattern; default `{namespace}-{stage}-{name}` |
| `naming.namespace` | Grep `*.tf`/`*.tfvars` for `namespace` value |
| `naming.stage` | Grep `*.tfvars` for `stage` or `environment` value |

**Confidence rules:**
- If a field is found unambiguously in one place → use it, note the source in a comment
- If multiple conflicting values are found → ask the user
- If no evidence → ask the user

---

## Section 3: Output Flow

1. **Detect** — Run all detection checks from Section 2. Collect confident values and a list of unknowns.

2. **Ask for unknowns** — For each field that couldn't be determined, ask one targeted question at a time. Example: *"I couldn't detect your cloud region from the codebase. What region does this project use?"*

3. **Check for existing file** — If `.devops.yaml` exists in the working directory, read it and compute what would change. Present a diff view, not just the new file.

4. **Show preview** — Print the full proposed `.devops.yaml` with inline comments noting how each value was detected. Ask: *"Write this to `.devops.yaml`? (yes / edit first / cancel)"*

5. **Handle response:**
   - **yes** → write the file, report success
   - **edit first** → show the YAML, invite the user to paste corrections, then re-show preview
   - **cancel** → do nothing

6. **Post-write nudge** — After writing, remind the user: *"The plugin will pick up this config on your next session start (or run `/clear` to reload now)."*

---

## Section 4: Update to `using-devops` Bootstrap Skill

Add to the Available Skills table in `skills/using-devops/SKILL.md`:

| Skill | Trigger | Use When |
|-------|---------|----------|
| `harumi-devops-plugin:setup-devops-config` | User asks to create/generate `.devops.yaml`; no config exists | Setting up the plugin for a new repo |
