# Parallel Agent Architecture — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable cross-skill parallelism by creating thin agent wrappers around existing skills and adding orchestration rules to `using-devops`.

**Architecture:** Each agent is an `AGENT.md` file in `agents/<skill-name>/` that accepts task inputs and invokes a single skill. The `using-devops` skill gains a Parallelism Rules section that declares which agents can run simultaneously and how the orchestrator should dispatch them.

**Tech Stack:** Markdown agent definitions, Claude Code Agent tool for dispatch

---

### Task 1: Create `run-kubernetes` agent

**Files:**
- Create: `agents/run-kubernetes/AGENT.md`

- [ ] **Step 1: Create the agent file**

```markdown
---
name: run-kubernetes
description: "Run the kubernetes skill in a fresh context for a specific task"
skill: harumi-devops-plugin:kubernetes
inputs:
  - task: string      # what to investigate or apply
  - context: string   # kubectl context
  - namespace: string # target namespace (optional)
---

# Run Kubernetes Agent

You are running the kubernetes skill to complete a specific task in a fresh, isolated context.

## Your Task

{task}

## Context

- **kubectl context:** {context}
- **Namespace:** {namespace}

## Steps

1. Invoke the `harumi-devops-plugin:kubernetes` skill using the Skill tool
2. Follow the skill to complete: {task}
3. Report findings or results clearly

## Output Format

Summarize your results as:
- **Status:** [what was found / applied]
- **Details:** [relevant facts, resource states, error messages]
- **Actions taken:** [if any writes were performed, list the handoff commands provided]
```

- [ ] **Step 2: Verify file exists and format is correct**

Run: `head -5 agents/run-kubernetes/AGENT.md`
Expected: frontmatter starting with `---` and `name: run-kubernetes`

- [ ] **Step 3: Commit**

```bash
git add agents/run-kubernetes/AGENT.md
git commit -m "feat(agents): add run-kubernetes agent wrapper"
```

---

### Task 2: Create `run-infrastructure` agent

**Files:**
- Create: `agents/run-infrastructure/AGENT.md`

- [ ] **Step 1: Create the agent file**

```markdown
---
name: run-infrastructure
description: "Run the infrastructure skill in a fresh context for a specific task"
skill: harumi-devops-plugin:infrastructure
inputs:
  - task: string      # what to plan, review, or modify
  - module: string    # terraform module path (optional)
---

# Run Infrastructure Agent

You are running the infrastructure skill to complete a specific task in a fresh, isolated context.

## Your Task

{task}

## Context

- **Terraform module:** {module}

## Steps

1. Invoke the `harumi-devops-plugin:infrastructure` skill using the Skill tool
2. Follow the skill to complete: {task}
3. Report findings or results clearly

## Output Format

Summarize your results as:
- **Status:** [what was found / planned / modified]
- **Details:** [relevant facts, resource states, plan output]
- **Actions taken:** [if any writes were performed, list the handoff commands provided]
```

- [ ] **Step 2: Verify file exists and format is correct**

Run: `head -5 agents/run-infrastructure/AGENT.md`
Expected: frontmatter starting with `---` and `name: run-infrastructure`

- [ ] **Step 3: Commit**

```bash
git add agents/run-infrastructure/AGENT.md
git commit -m "feat(agents): add run-infrastructure agent wrapper"
```

---

### Task 3: Create `run-observability` agent

**Files:**
- Create: `agents/run-observability/AGENT.md`

- [ ] **Step 1: Create the agent file**

```markdown
---
name: run-observability
description: "Run the observability skill in a fresh context for a specific task"
skill: harumi-devops-plugin:observability
inputs:
  - task: string      # what to investigate, query, or build
  - mode: string      # "author" or "investigate"
---

# Run Observability Agent

You are running the observability skill to complete a specific task in a fresh, isolated context.

## Your Task

{task}

## Context

- **Mode:** {mode}

## Steps

1. Invoke the `harumi-devops-plugin:observability` skill using the Skill tool
2. Follow the skill in **{mode}** mode to complete: {task}
3. Report findings or results clearly

## Output Format

Summarize your results as:
- **Status:** [what was found / authored / diagnosed]
- **Details:** [relevant metrics, log patterns, queries written, root cause analysis]
- **Actions taken:** [if any artifacts were created or handoff commands provided]
```

- [ ] **Step 2: Verify file exists and format is correct**

Run: `head -5 agents/run-observability/AGENT.md`
Expected: frontmatter starting with `---` and `name: run-observability`

- [ ] **Step 3: Commit**

```bash
git add agents/run-observability/AGENT.md
git commit -m "feat(agents): add run-observability agent wrapper"
```

---

### Task 4: Create `run-argocd` agent

**Files:**
- Create: `agents/run-argocd/AGENT.md`

- [ ] **Step 1: Create the agent file**

```markdown
---
name: run-argocd
description: "Run the argocd skill in a fresh context for a specific task"
skill: harumi-devops-plugin:argocd
inputs:
  - task: string      # what to investigate, sync, or manage
  - app: string       # ArgoCD application name (optional)
---

# Run ArgoCD Agent

You are running the argocd skill to complete a specific task in a fresh, isolated context.

## Your Task

{task}

## Context

- **ArgoCD application:** {app}

## Steps

1. Invoke the `harumi-devops-plugin:argocd` skill using the Skill tool
2. Follow the skill to complete: {task}
3. Report findings or results clearly

## Output Format

Summarize your results as:
- **Status:** [what was found / synced / modified]
- **Details:** [sync status, health, drift details, error messages]
- **Actions taken:** [if any writes were performed, list the handoff commands provided]
```

- [ ] **Step 2: Verify file exists and format is correct**

Run: `head -5 agents/run-argocd/AGENT.md`
Expected: frontmatter starting with `---` and `name: run-argocd`

- [ ] **Step 3: Commit**

```bash
git add agents/run-argocd/AGENT.md
git commit -m "feat(agents): add run-argocd agent wrapper"
```

---

### Task 5: Create `run-debug-pod` agent

**Files:**
- Create: `agents/run-debug-pod/AGENT.md`

- [ ] **Step 1: Create the agent file**

```markdown
---
name: run-debug-pod
description: "Run the debug-pod skill in a fresh context for a specific task"
skill: harumi-devops-plugin:debug-pod
inputs:
  - task: string      # what pod to debug and symptoms
  - context: string   # kubectl context
  - namespace: string # target namespace
  - pod: string       # pod name or label selector
---

# Run Debug Pod Agent

You are running the debug-pod skill to complete a specific task in a fresh, isolated context.

## Your Task

{task}

## Context

- **kubectl context:** {context}
- **Namespace:** {namespace}
- **Pod:** {pod}

## Steps

1. Invoke the `harumi-devops-plugin:debug-pod` skill using the Skill tool
2. Follow the skill's diagnostic sequence for: {task}
3. Report findings or results clearly

## Output Format

Summarize your results as:
- **Status:** [pod state, root cause identified or not]
- **Details:** [diagnostic findings, error logs, resource constraints, event timeline]
- **Actions taken:** [if any remediation handoff commands were provided]
```

- [ ] **Step 2: Verify file exists and format is correct**

Run: `head -5 agents/run-debug-pod/AGENT.md`
Expected: frontmatter starting with `---` and `name: run-debug-pod`

- [ ] **Step 3: Commit**

```bash
git add agents/run-debug-pod/AGENT.md
git commit -m "feat(agents): add run-debug-pod agent wrapper"
```

---

### Task 6: Create `run-deploy-app` agent

**Files:**
- Create: `agents/run-deploy-app/AGENT.md`

- [ ] **Step 1: Create the agent file**

```markdown
---
name: run-deploy-app
description: "Run the deploy-app skill in a fresh context for a specific task"
skill: harumi-devops-plugin:deploy-app
inputs:
  - task: string        # what app to deploy or update
  - environment: string # "dev" or "prod"
  - app-name: string    # application name
---

# Run Deploy App Agent

You are running the deploy-app skill to complete a specific task in a fresh, isolated context.

## Your Task

{task}

## Context

- **Environment:** {environment}
- **Application:** {app-name}

## Steps

1. Invoke the `harumi-devops-plugin:deploy-app` skill using the Skill tool
2. Follow the skill to complete: {task}
3. Report findings or results clearly

## Output Format

Summarize your results as:
- **Status:** [what was generated / deployed / updated]
- **Details:** [manifests created, CI workflow generated, environment routing]
- **Actions taken:** [if any writes were performed, list files created and handoff commands]
```

- [ ] **Step 2: Verify file exists and format is correct**

Run: `head -5 agents/run-deploy-app/AGENT.md`
Expected: frontmatter starting with `---` and `name: run-deploy-app`

- [ ] **Step 3: Commit**

```bash
git add agents/run-deploy-app/AGENT.md
git commit -m "feat(agents): add run-deploy-app agent wrapper"
```

---

### Task 7: Add Parallelism Rules to `using-devops`

**Files:**
- Modify: `skills/using-devops/SKILL.md` — append new section before `## Configuration`

- [ ] **Step 1: Read the current file to confirm insertion point**

Run: `grep -n "## Configuration" skills/using-devops/SKILL.md`
Expected: Line number where `## Configuration` appears (should be around line 136)

- [ ] **Step 2: Add Parallelism Rules section**

Insert the following block **before** the `## Configuration` section in `skills/using-devops/SKILL.md`:

```markdown
## Parallel Agent Dispatch

When a user request spans multiple independent domains, dispatch agents in parallel for faster results and cleaner context isolation. Each agent runs the corresponding skill in a fresh context window.

### Agent Inventory

| Agent | Skill | Typical Use |
|-------|-------|-------------|
| `run-kubernetes` | `harumi-devops-plugin:kubernetes` | K8s investigation, manifest work |
| `run-infrastructure` | `harumi-devops-plugin:infrastructure` | Terraform state, IaC changes |
| `run-observability` | `harumi-devops-plugin:observability` | Metrics, logs, incident investigation |
| `run-argocd` | `harumi-devops-plugin:argocd` | Sync status, GitOps operations |
| `run-debug-pod` | `harumi-devops-plugin:debug-pod` | Pod troubleshooting |
| `run-deploy-app` | `harumi-devops-plugin:deploy-app` | App onboarding |

### Compatibility Matrix

| Agents | Parallel? | Notes |
|--------|-----------|-------|
| `run-infrastructure` + `run-kubernetes` | Yes | Independent targets |
| `run-kubernetes` + `run-observability` | Yes | Common in incident investigation |
| `run-infrastructure` + `run-observability` | Yes | No shared write targets |
| `run-argocd` + `run-kubernetes` | Yes | Investigation compatible |
| `run-deploy-app` + `run-debug-pod` (same app) | No | Sequence: debug first |
| Any two agents writing to the same resource | No | Always sequential |

### Dispatch Rules

When the user request maps to 2+ compatible skills:

1. **Identify** the relevant agents from the inventory
2. **Check compatibility** using the matrix — if agents are compatible, dispatch in parallel; if not, sequence them
3. **Dispatch** compatible agents simultaneously using the Agent tool, passing the specific task and context as inputs to each
4. **Collect** all results before executing any write operations
5. **Sequence writes** — execute writes one at a time, respecting declared dependencies

### Error Handling

**Parallel agent failure:**
1. Preserve and display results from successful agents
2. Name the failed agent and its error explicitly
3. Ask the user whether to proceed with partial results or abort
4. Never silently proceed into write operations with incomplete information

**Write failure:**
1. Surface the full error
2. Stop the write sequence
3. Present a manual recovery handoff command for the user to execute

**No silent retries.** All error recovery is user-gated.
```

- [ ] **Step 3: Verify the section was added**

Run: `grep -n "Parallel Agent Dispatch" skills/using-devops/SKILL.md`
Expected: Line number showing the new section header

- [ ] **Step 4: Commit**

```bash
git add skills/using-devops/SKILL.md
git commit -m "feat(using-devops): add parallel agent dispatch rules"
```
