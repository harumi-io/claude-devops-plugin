---
name: using-devops
description: "Bootstrap skill for harumi-devops-plugin. Injected at session start. Announces available DevOps skills, loads repo config, defines trigger rules, and enforces safety rules for infrastructure operations."
---

# DevOps Plugin

You have the **harumi-devops-plugin** installed. This plugin provides DevOps-oriented skills for infrastructure, Kubernetes, CI/CD, and cloud operations.

## Available Skills

Use the Skill tool to invoke these when triggered:

| Skill | Trigger | Use When |
|-------|---------|----------|
| `harumi-devops-plugin:devops` | `.tf` files, Terraform, AWS/GCP/Azure infra | Creating, modifying, or reviewing Terraform/IaC configurations |
| `harumi-devops-plugin:setup-devops-config` | User asks to create/set up `.devops.yaml`; no config exists | Setting up the plugin for a new repo |

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

**Future skills** (not yet available):
- `kubernetes` — K8s manifests, Helm charts, ArgoCD/Flux
- `cicd` — CI/CD pipeline configs, deployment workflows
- `cost-optimization` — Resource sizing, cost analysis
- `observability` — Monitoring, alerting, dashboards
- `containers` — Dockerfiles, image builds, registries

## Trigger Rules

Invoke `harumi-devops-plugin:devops` when you encounter ANY of:
- `.tf` files or Terraform discussions
- AWS, GCP, or Azure infrastructure tasks
- IaC changes, module creation, state management
- Infrastructure migrations or zero-downtime changes
- Cost or security review of cloud resources

Invoke `harumi-devops-plugin:setup-devops-config` when you encounter ANY of:
- User asks to create, generate, or set up `.devops.yaml`
- User says "configure the plugin" or "set up devops config"
- No `.devops.yaml` exists and the user expresses intent to configure or set up the plugin

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

## Universal Safety Rules (NON-NEGOTIABLE)

These apply to ALL DevOps skills:

1. **Never run `terraform apply` or `terraform destroy`** — Always provide a handoff with the exact command for the user to execute
2. **Never `kubectl delete` in production without explicit user confirmation**
3. **Never push images to production registries without confirmation**
4. **Always verify current state before making changes** — Use CLI commands (aws, gcloud, az, kubectl) to confirm resource existence and configuration
5. **Always present the handoff pattern for destructive actions:**

```
Configuration ready for apply!

Execute: cd [path] && terraform apply -var-file=[tfvars]
Changes: [summary]
Verification: [CLI commands to confirm]
```

## Configuration

The active `.devops.yaml` config (loaded at session start) tells you:
- **Cloud provider** — which CLI commands to use (aws, gcloud, az)
- **Terraform settings** — version, state backend, var file
- **Naming pattern** — how resources are named
- **Stack details** — K8s tool, CI/CD platform, observability stack, container runtime

Read config values to adapt your guidance to the user's specific stack.

## Relationship to Superpowers

This plugin is **domain-oriented** (how to do DevOps). Superpowers is **process-oriented** (how to work: TDD, debugging, planning). They complement each other. Use superpowers skills for workflow (brainstorming, planning, debugging) and devops skills for domain knowledge (Terraform patterns, cloud architecture, security).
