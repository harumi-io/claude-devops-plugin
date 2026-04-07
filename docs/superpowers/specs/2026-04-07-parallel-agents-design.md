# Parallel Agent Architecture — Design Spec

**Date:** 2026-04-07  
**Status:** Draft (Revised)

## Problem

All skills in `harumi-devops-plugin` execute sequentially within a single context window. Operations that are logically independent — running `infrastructure` and `kubernetes` skills for a combined task, or investigating an incident with both `kubernetes` and `observability` — are bottlenecked into one thread. This wastes wall-clock time and pollutes each skill's context with unrelated prior work.

## Goal

Enable cross-skill parallelism: multiple skills run simultaneously in independent fresh context windows when a request spans compatible domains.

**Constraint:** Existing skills are not modified. Agents are thin wrappers that invoke skills — all skill logic stays where it is.

## Non-Goals

- Modifying existing skill `SKILL.md` files
- Within-skill step parallelism (internal steps remain sequential) — deferred
- Autonomous retry or self-healing — all error recovery is user-gated
- Nested agent dispatch (agents calling agents)
- Real-time streaming between agents

---

## Agent Files

### Directory Structure

```
agents/
  <skill-name>/
    AGENT.md
```

Each `AGENT.md` is a thin wrapper that:
1. Accepts inputs describing the specific task
2. Invokes the corresponding skill via the `skill` tool
3. Returns results to the caller

The `agents/` directory already exists in the repo root; this design activates it.

### `AGENT.md` Format

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

## Steps

1. Invoke the `skill: harumi-devops-plugin:kubernetes` tool
2. Follow the skill to complete: {task}
3. Report findings or results clearly

## Output Format

Summarize your results as:
- **Status:** [what was found / applied]
- **Details:** [relevant facts]
- **Actions taken:** [if any writes were performed]
```

**Rules:**
- Input placeholders use `{input-name}` syntax — resolved by the calling orchestrator before dispatch
- Each agent maps 1:1 to a skill
- Agents are stateless — no shared state with other agents
- Output is structured prose for the orchestrator to collect

### Agent Inventory

One agent per parallelizable skill:

| Agent | Skill | Typical use |
|-------|-------|-------------|
| `run-kubernetes` | `harumi-devops-plugin:kubernetes` | K8s investigation, manifest work |
| `run-infrastructure` | `harumi-devops-plugin:infrastructure` | Terraform state, IaC changes |
| `run-observability` | `harumi-devops-plugin:observability` | Metrics, logs, incident investigation |
| `run-argocd` | `harumi-devops-plugin:argocd` | Sync status, GitOps operations |
| `run-debug-pod` | `harumi-devops-plugin:debug-pod` | Pod troubleshooting |
| `run-deploy-app` | `harumi-devops-plugin:deploy-app` | App onboarding |

---

## Cross-Skill Orchestration

`using-devops` gains a **Parallelism Rules** section that declares which agents are safe to dispatch simultaneously and which must be sequenced.

### Compatibility Matrix

| Agents (skills) | Can run together | Notes |
|-----------------|-----------------|-------|
| `run-infrastructure` + `run-kubernetes` | ✅ | Independent targets |
| `run-kubernetes` + `run-observability` | ✅ | Common in incident investigation |
| `run-infrastructure` + `run-observability` | ✅ | No shared write targets |
| `run-argocd` + `run-kubernetes` | ✅ | Investigation compatible |
| `run-deploy-app` + `run-debug-pod` (same app) | ⚠️ | Sequence: debug first |
| Any two agents writing to the same resource | ❌ | Always sequential |

### Dispatch Rule (in `using-devops`)

When the main agent determines a user request maps to 2+ compatible skills:

1. Identify the relevant agents from the inventory above
2. Dispatch all compatible agents simultaneously using the `task` tool, passing the specific task and context as inputs
3. Each agent invokes its skill in a fresh, isolated context window
4. Collect all results before executing any write operations
5. Execute writes sequentially, respecting any declared dependencies

---

## Error Handling

### Parallel Agent Failures

If one of the parallel agents fails or returns incomplete data:
1. Preserve and display the successful agents' results
2. Explicitly name the failed agent and its error
3. Ask the user whether to proceed with partial results or abort
4. Never silently proceed into write operations with incomplete information

### Write Failures

If a skill encounters an error during its write phase:
1. Surface the full error
2. Stop the write sequence
3. Present a manual recovery handoff command for the user to execute

This follows the existing "never silently apply" safety rule.

**No silent retries.** All error recovery is user-gated.

---

## File Changes Required

| Action | Path | Purpose |
|--------|------|---------|
| Create | `agents/run-kubernetes/AGENT.md` | Thin wrapper invoking kubernetes skill |
| Create | `agents/run-infrastructure/AGENT.md` | Thin wrapper invoking infrastructure skill |
| Create | `agents/run-observability/AGENT.md` | Thin wrapper invoking observability skill |
| Create | `agents/run-argocd/AGENT.md` | Thin wrapper invoking argocd skill |
| Create | `agents/run-debug-pod/AGENT.md` | Thin wrapper invoking debug-pod skill |
| Create | `agents/run-deploy-app/AGENT.md` | Thin wrapper invoking deploy-app skill |
| Modify | `skills/using-devops/SKILL.md` | Add Parallelism Rules section |

No existing skills are modified.

---

## Design Principles

1. **Agents are thin wrappers** — they pass context to a skill and return results; no business logic
2. **Skills are unchanged** — all domain knowledge stays in `SKILL.md` files
3. **using-devops is the orchestrator** — it sees across agents and decides what to dispatch in parallel
4. **Fresh context by construction** — each dispatched agent starts with no conversation history; context is passed explicitly via inputs
5. **Safety is never relaxed** — parallelism doesn't bypass confirmation gates or user-gated writes
