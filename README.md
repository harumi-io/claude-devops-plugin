# Harumi DevOps Plugin

DevOps skills for [Claude Code](https://claude.ai/code), [Cursor](https://cursor.com), and GitHub Copilot. Provides infrastructure, Kubernetes, CI/CD, and cloud operations guidance through an extensible skill system.

## Installation

### Claude Code

Register the marketplace and install the plugin:

```bash
/plugin marketplace add git@github.com:harumi-io/harumi-devops-plugin.git
/plugin install harumi-devops-plugin@harumi-devops-marketplace
```

For local development, you can register directly from a cloned copy:

```bash
/plugin marketplace add /path/to/harumi-devops-plugin
/plugin install harumi-devops-plugin@harumi-devops-marketplace
```

### Cursor

Clone the repository and register the plugin in Cursor Agent chat:

```text
/add-plugin /path/to/harumi-devops-plugin
```

### GitHub Copilot

Clone the repository. The session-start hook auto-detects the Copilot environment.

## Configuration

Create a `.devops.yaml` in your repository root to configure the plugin for your stack:

```yaml
cloud:
  provider: aws          # aws | gcp | azure
  region: us-east-1

terraform:
  version: "1.5.7"
  state_backend: s3
  var_file: prod.tfvars

naming:
  pattern: "{namespace}-{stage}-{name}"
  namespace: mycompany
  stage: production
```

See `config/default.devops.yaml` for all available options.

If no `.devops.yaml` is found, the plugin uses its built-in defaults.

## Skills

### Available (MVP)

| Skill | Description |
|-------|-------------|
| `devops` | Terraform/IaC management with multi-provider support (AWS, GCP, Azure) |

### Planned

| Skill | Description |
|-------|-------------|
| `kubernetes` | K8s manifest management, Helm, ArgoCD/Flux |
| `cicd` | CI/CD pipeline authoring and deployment patterns |
| `cost-optimization` | Resource right-sizing and cost analysis |
| `observability` | Monitoring, alerting, and dashboard management |
| `security-operations` | IAM audit, secrets rotation, compliance |
| `containers` | Dockerfile optimization, image management |

## How It Works

1. **Session start** — The hook loads the bootstrap skill and merges your `.devops.yaml` config
2. **Skill triggering** — The bootstrap skill tells Claude when to invoke domain-specific skills
3. **Safety rules** — Destructive operations (apply, destroy, delete) always require user confirmation via handoff

## Relationship to Superpowers

This plugin is **domain-oriented** (DevOps knowledge). [Superpowers](https://github.com/obra/superpowers) is **process-oriented** (TDD, debugging, planning). They work together — use superpowers for workflow, devops-plugin for domain expertise.

## License

MIT
