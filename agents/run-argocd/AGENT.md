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
