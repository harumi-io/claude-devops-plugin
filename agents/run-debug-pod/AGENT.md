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
