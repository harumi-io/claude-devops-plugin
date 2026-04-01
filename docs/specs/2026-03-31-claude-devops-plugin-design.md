# Harumi DevOps Plugin — Design Spec

**Date:** 2026-03-31
**Status:** Approved
**Repository:** `harumi-io/harumi-devops-plugin` (new standalone repo)

## Overview

A Claude Code plugin that provides DevOps-oriented skills for infrastructure, Kubernetes, CI/CD, cost optimization, observability, security operations, and container management. Inspired by the [superpowers](https://github.com/obra/superpowers) plugin architecture. Generalizable with Harumi.io defaults.

## Goals

- Mirror the superpowers plugin structure (manifests, hooks, `SKILL.md` + `references/` pattern)
- Support 3 platforms: Claude Code, Cursor, GitHub Copilot
- Per-repo configuration via `.devops.yaml` with plugin-level defaults
- MVP: plugin scaffold + ported infrastructure skill only; add skills incrementally

## Non-Goals

- Incident response skill (excluded from scope)
- Full multi-platform support (Codex, Gemini, OpenCode deferred)
- Auto-discovery of stack from codebase (users configure explicitly via `.devops.yaml`)

---

## Section 1: Repository Layout

```
harumi-devops-plugin/
├── .claude-plugin/
│   ├── plugin.json              # Claude Code manifest
│   └── marketplace.json         # Plugin marketplace metadata
├── .cursor-plugin/
│   └── plugin.json              # Cursor manifest (skills, agents, hooks)
├── hooks/
│   ├── session-start            # Bash bootstrap script
│   ├── hooks-cursor.json        # Cursor/Claude Code hooks config
│   └── hooks.json               # Codex/Copilot hooks config
├── skills/
│   ├── using-devops/            # Meta skill — bootstrap, skill discovery
│   │   └── SKILL.md
│   └── infrastructure/          # MVP: ported from existing devops skill
│       ├── SKILL.md
│       └── references/
│           ├── architecture.md
│           ├── workflow.md
│           ├── examples.md
│           ├── modules.md
│           ├── naming.md
│           └── security.md
├── agents/                      # Subagent templates (empty for MVP)
├── config/
│   └── default.devops.yaml      # Default config (Harumi stack as example)
├── README.md
├── LICENSE
├── CHANGELOG.md
└── package.json
```

**Future skills** (post-MVP) are added as new directories under `skills/`:

```
skills/
├── kubernetes/
├── cicd/
├── cost-optimization/
├── observability/
├── security-operations/
└── containers/
```

---

## Section 2: Configuration System

### Per-repo config: `.devops.yaml`

Lives in the repo root. Users override per-repo; plugin provides defaults.

```yaml
cloud:
  provider: aws                    # aws | gcp | azure
  region: us-east-2
  account_alias: harumi

terraform:
  version: "1.5.7"
  state_backend: s3
  var_file: prod.tfvars

kubernetes:
  tool: kubectl                    # kubectl | oc (openshift)
  gitops: argocd                   # argocd | flux | none
  clusters:
    - name: eks-dev
      context: eks-dev
    - name: eks-prod
      context: eks-prod

cicd:
  platform: github-actions         # github-actions | gitlab-ci | circleci

observability:
  metrics: prometheus              # prometheus | datadog | cloudwatch
  dashboards: grafana
  logs: loki                       # loki | cloudwatch | datadog
  traces: tempo                    # tempo | jaeger | xray

containers:
  runtime: docker                  # docker | podman | nerdctl
  registry: ecr                    # ecr | gcr | dockerhub | ghcr

naming:
  pattern: "{namespace}-{stage}-{name}"
  namespace: harumi
  stage: production
```

### Config resolution

1. Session-start hook checks for `.devops.yaml` in working directory
2. If found, merges with `config/default.devops.yaml` (repo config wins)
3. Merged config is injected as context alongside the bootstrap skill
4. Skills read from config to adapt guidance (CLI commands, naming, provider-specific patterns)

---

## Section 3: Bootstrap Skill (`using-devops`)

The meta skill injected at session start. Equivalent to superpowers' `using-superpowers`.

### Responsibilities

1. **Announce available skills** with trigger descriptions
2. **Load merged repo config** so Claude knows the stack
3. **Define trigger rules** for each skill:
   - `.tf` files or infrastructure discussions → `infrastructure`
   - K8s manifests, Helm charts, ArgoCD → `kubernetes`
   - `.github/workflows/` → `cicd`
   - Cost discussions, resource sizing → `cost-optimization`
   - Prometheus rules, Grafana dashboards, alerts → `observability`
   - IAM policies, security groups, secrets → `security-operations`
   - Dockerfiles, compose files, image builds → `containers`
4. **Enforce universal safety rules:**
   - Never run `terraform apply/destroy`
   - Never `kubectl delete` in production without confirmation
   - Never push images to production registries without confirmation
   - Always verify current state before making changes
5. **Provide handoff pattern** for destructive actions

### Relationship to superpowers

Superpowers is process-oriented (how to work). This plugin is domain-oriented (how to do DevOps). They complement each other and can be installed side by side.

---

## Section 4: Infrastructure Skill (MVP)

Ported from the existing devops skill at `harumi-io/infrastructure/.claude/skills/devops/`.

### Changes from current skill

1. **Config-driven** — reads cloud provider, region, naming pattern, state backend from `.devops.yaml` instead of hardcoded Harumi values

2. **Provider-conditional CLI commands:**
   - `aws` → `aws cli` verification
   - `gcp` → `gcloud` commands
   - `azure` → `az cli` commands

3. **References adapted:**
   - `architecture.md` — Harumi example kept; framed as reference, with guidance on documenting your own
   - `workflow.md` — 10-phase process is universal; cost/downtime tables become provider-aware
   - `examples.md` — Harumi examples as reference, notes on adapting patterns
   - `modules.md` — mostly Terraform-universal, minimal changes
   - `naming.md` — naming pattern from config, examples adapt
   - `security.md` — universal principles; provider-specific checklists branch on config

4. **6 critical rules preserved** (universal):
   - Verify with CLI before changes
   - Ask when ambiguous
   - Present downtime alternatives
   - Present cost options
   - Update documentation
   - Use correct backend-config

---

## Section 5: Platform Support

### Claude Code

- `.claude-plugin/plugin.json` — name, description, version, author, keywords
- Hooks via `hooks/hooks-cursor.json` (sessionStart → `./hooks/session-start`)
- Skills discovered from `skills/` directory

### Cursor

- `.cursor-plugin/plugin.json` — adds `skills`, `agents`, `commands`, `hooks` paths
- Same hooks mechanism as Claude Code

### GitHub Copilot

- Session-start hook detects `COPILOT_CLI` env var
- Outputs `additionalContext` in SDK standard format
- Same skill content, different JSON output format

### Session-start hook

Platform detection via environment variables:
- `CURSOR_PLUGIN_ROOT` → Cursor format (`additional_context`)
- `CLAUDE_PLUGIN_ROOT` (no `COPILOT_CLI`) → Claude Code format (`hookSpecificOutput.additionalContext`)
- `COPILOT_CLI` or fallback → SDK standard format (`additionalContext`)

The hook:
1. Reads `skills/using-devops/SKILL.md`
2. Reads and merges `.devops.yaml` from working directory with plugin defaults
3. Escapes content for JSON
4. Outputs platform-appropriate JSON with bootstrap context + config

---

## Section 6: Future Skills Roadmap (Post-MVP)

Each follows the same `SKILL.md` + `references/` pattern:

| Skill | Trigger | Key capabilities |
|-------|---------|-----------------|
| `kubernetes` | K8s manifests, Helm, ArgoCD | Debug workflow, rollout management, ArgoCD sync troubleshooting |
| `cicd` | `.github/workflows/`, pipeline configs | Workflow authoring, deployment patterns, rollback procedures |
| `cost-optimization` | Cost discussions, sizing | Right-sizing, unused resource detection, reserved vs on-demand |
| `observability` | Monitoring configs, alerts, dashboards | PromQL/LogQL, alert authoring, USE/RED method dashboards |
| `security-operations` | IAM, security groups, secrets | Least-privilege audit, secrets rotation, compliance checklists |
| `containers` | Dockerfiles, compose, image configs | Multi-stage builds, image optimization, registry management |

---

## MVP Scope

1. Repository scaffold with all platform manifests
2. Session-start hook with config loading
3. `using-devops` bootstrap skill
4. `devops` skill (ported and generalized)
5. Default `.devops.yaml` config (Harumi defaults)
6. README with installation instructions for all 3 platforms
