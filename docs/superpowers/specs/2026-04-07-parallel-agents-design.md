# Parallel Agent Architecture — Design Spec

**Date:** 2026-04-07  
**Status:** Draft

## Problem

All skills in `harumi-devops-plugin` execute sequentially within a single context window. Operations that are logically independent — inspecting pod state while checking cluster health, or running `infrastructure` and `kubernetes` skills for a combined task — are bottlenecked into one thread. This wastes wall-clock time and pollutes each skill's context with unrelated prior work.

## Goal

Enable two levels of parallelism:

- **Level 1 (within a skill):** Investigation steps dispatched as parallel sub-agents via `AGENT.md` files in the `agents/` directory
- **Level 2 (across skills):** `using-devops` coordinates multiple skills running simultaneously when a request spans compatible domains

Both levels share a common principle: investigation in parallel, writes sequential.

## Non-Goals

- Autonomous retry or self-healing — all error recovery is user-gated
- Nested agent dispatch (agents calling agents)
- Real-time streaming between agents

---

## Level 1: Agent Files

### Directory Structure

```
agents/
  <agent-name>/
    AGENT.md
```

Each `AGENT.md` is a self-contained mini-playbook. The `agents/` directory already exists in the repo root; this design activates it.

### `AGENT.md` Format

```markdown
---
name: check-pod-state
description: "Fetch logs, events, and restart count for a pod"
inputs:
  - pod: string          # pod name
  - namespace: string    # k8s namespace
  - context: string      # kubectl context
phase: read              # read | write  (advisory; calling skill controls actual sequencing)
---

# Check Pod State

## Steps

1. ...
2. ...

## Output Format

Report findings as:
- **Key:** [value]
```

**Rules:**
- Input placeholders use `{input-name}` syntax — resolved by the calling skill before dispatch
- `phase` is advisory metadata only; the calling skill decides when to dispatch
- Output format is structured prose; no JSON required
- Fully self-contained — no references to other agents or skills

### Agent Inventory (Initial)

Agents to create for the first wave of skill migrations:

| Agent | Phase | Used By |
|-------|-------|---------|
| `check-pod-state` | read | `debug-pod` |
| `check-pod-logs` | read | `debug-pod` |
| `check-cluster-health` | read | `debug-pod`, `scale-deployment`, `deploy-app` |
| `check-namespace` | read | `deploy-app`, `create-namespace` |
| `check-ecr-repo` | read | `deploy-app` |
| `check-argocd-app` | read | `deploy-app`, `rollback-deployment` |
| `check-hpa` | read | `scale-deployment` |
| `check-terraform-state` | read | `infrastructure` |
| `apply-namespace` | write | `create-namespace` |
| `apply-argocd-app` | write | `deploy-app` |

---

## Level 1: Skill Integration Pattern

Skills are restructured into two explicit phases.

### Phase 1 — Parallel Investigation

```markdown
## Phase 1: Parallel Investigation

Dispatch simultaneously using the task tool:
- `agents/check-pod-state` — inputs: pod={pod}, namespace={namespace}, context={context}
- `agents/check-cluster-health` — inputs: context={context}
- `agents/check-namespace` — inputs: namespace={namespace}, context={context}

Collect all outputs before proceeding to Phase 2.
```

All listed agents are dispatched at once. The main agent waits for all results before continuing.

### Phase 2 — Sequential Writes

```markdown
## Phase 2: Apply (Sequential)

Based on Phase 1 results:
1. If namespace missing: dispatch `agents/apply-namespace` (user confirmation required)
2. If ArgoCD app missing: dispatch `agents/apply-argocd-app` (user confirmation required)
```

Write agents are dispatched one at a time, in dependency order. Each write gate requires explicit user confirmation consistent with existing safety rules.

### Skills to Migrate

**High value (migrate now):**
- `debug-pod` — 3+ sequential inspection steps, all read-only
- `deploy-app` — cluster state + ECR + ArgoCD checks before writes
- `scale-deployment` — HPA check + node capacity check before scaling
- `rollback-deployment` — history fetch + current state check before rollback
- `kubernetes` — multi-step investigation patterns
- `infrastructure` — Terraform state + resource existence checks before plan/apply

**Low value (skip for now):**
- `create-iam-user`, `remove-iam-user`, `rotate-access-keys` — single CLI call + plan
- `list-vpn-users`, `create-vpn-creds`, `revoke-vpn-creds` — single script execution
- `create-namespace` — minimal investigation; apply phase uses agent architecture but no parallel investigation needed

---

## Level 2: Cross-Skill Orchestration

`using-devops` gains a **Parallelism Rules** section that declares which skills are safe to dispatch simultaneously and which must be sequenced.

### Compatibility Matrix

| Skills | Can run together | Notes |
|--------|-----------------|-------|
| `infrastructure` + `kubernetes` | ✅ | Reads parallel; writes sequential |
| `kubernetes` + `observability` | ✅ | Common in incident investigation |
| `infrastructure` + `observability` | ✅ | No shared write targets |
| `argocd` + `kubernetes` | ✅ | Investigation compatible |
| `deploy-app` + `create-namespace` | ⚠️ | Sequence: namespace first |
| Any two skills writing to the same resource | ❌ | Always sequential |

### Dispatch Rule (in `using-devops`)

When the main agent determines a user request maps to 2+ compatible skills:

1. Dispatch all compatible skills as parallel sub-agents using the `task` tool
2. Each skill runs its full investigation phase independently (fresh context)
3. Collect all results before starting any write phase
4. Execute write phases sequentially, respecting any declared dependencies

---

## Error Handling

### Phase 1 Agent Failures

If an investigation agent fails or returns incomplete data:
1. Report what partial data was collected from the other agents
2. Explicitly name the failed agent and its error
3. Ask the user whether to proceed with incomplete information or abort
4. Never silently proceed into Phase 2 writes

### Phase 2 Agent Failures

Write agents fail loudly. On failure:
1. Surface the full error
2. Stop the write sequence
3. Present a manual recovery handoff command for the user to execute

This follows the existing "never silently apply" safety rule.

### Level 2 Cross-Skill Failures

If one of multiple parallel skills fails during investigation:
1. Preserve and display the successful skills' results
2. Report the failed skill and its error
3. User decides whether to proceed with partial results or retry

**No silent retries.** All error recovery is user-gated.

---

## File Changes Required

| Action | Path | Purpose |
|--------|------|---------|
| Create | `agents/<agent-name>/AGENT.md` | Per-agent per the inventory above |
| Modify | `skills/debug-pod/SKILL.md` | Add Phase 1/2 structure |
| Modify | `skills/deploy-app/SKILL.md` | Add Phase 1/2 structure |
| Modify | `skills/scale-deployment/SKILL.md` | Add Phase 1/2 structure |
| Modify | `skills/rollback-deployment/SKILL.md` | Add Phase 1/2 structure |
| Modify | `skills/kubernetes/SKILL.md` | Add Phase 1/2 structure |
| Modify | `skills/infrastructure/SKILL.md` | Add Phase 1/2 structure |
| Modify | `skills/using-devops/SKILL.md` | Add Parallelism Rules section |

---

## Design Principles

1. **Agents are dumb building blocks** — they execute one task, return structured output, know nothing about orchestration
2. **Skills are the orchestrators** — they decide what to dispatch, in what order, and what to do with results
3. **using-devops is the meta-orchestrator** — it sees across skills and enables Level 2 parallelism
4. **Fresh context by construction** — each dispatched agent/skill starts with no conversation history; context is passed explicitly via inputs
5. **Safety is never relaxed** — parallelism doesn't bypass confirmation gates or user-gated writes
