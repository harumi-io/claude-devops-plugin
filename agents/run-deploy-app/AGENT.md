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
