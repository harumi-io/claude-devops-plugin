# DevOps Plugin Rollout Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Align `harumi-devops-plugin` with the real Harumi rollout workflow used for the frontend EKS prod deployment so the plugin detects live config correctly, emits accurate guidance, and preflights rollout prerequisites before execution.

**Architecture:** Add a small compatibility layer in the session-start hook to normalize repo config input (`harumi.yaml` and legacy `.devops.yaml`), then update the bootstrap/core skills and deploy references to consume that normalized reality. Refresh rollout skills around app-repo deploys, ArgoCD branch lifecycles, ingress exposure, and Terraform drift handling, then lock the behavior in with focused evals.

**Tech Stack:** Bash hooks, Markdown skills, JSON eval fixtures, ripgrep/git/kubectl/Terraform command guidance

---

## File Map

- **Modify:** `hooks/session-start` — load either `harumi.yaml` or `.devops.yaml`, emit config source, and surface stale kube-context warnings in session context.
- **Modify:** `README.md` — document supported config formats, repo model, and rollout workflow expectations.
- **Modify:** `skills/using-devops/SKILL.md` — stop assuming only two managed repos and only `harumi.yaml`; describe normalized config and runtime validation.
- **Modify:** `skills/infrastructure/SKILL.md` — add quoted `-target` guidance and unrelated-drift workflow.
- **Modify:** `skills/kubernetes/SKILL.md` — add ingress exposure decision, context verification, and user-approved operational recovery playbooks.
- **Modify:** `skills/argocd/SKILL.md` — add feature-branch `targetRevision` lifecycle and isolated rollout-app pattern.
- **Modify:** `skills/deploy-app/SKILL.md` — add app-repo deployment reality, naming variants, and rollout preflight checklist.
- **Modify:** `skills/deploy-app/references/ci-writeback-prod.md` — stop assuming only `argocd-app.yaml`/`cd-eks.yaml`; add isolated prod test variant and preflight requirements.
- **Modify:** `skills/deploy-app/references/ci-writeback-pattern.md` — align dev pattern wording with normalized config model and branch-aware Argo guidance.
- **Modify:** `skills/sync-docs/SKILL.md` — teach doc sync to understand both config filenames or the normalized manifest source.
- **Modify:** `skills/observability/SKILL.md` — update config references from hard-coded `harumi.yaml` wording to normalized config wording.
- **Modify:** `skills/rollback-deployment/SKILL.md` — replace hard-coded `harumi.yaml` trigger language with normalized config wording.
- **Modify:** `skills/debug-pod/SKILL.md` — replace hard-coded `harumi.yaml` trigger language with normalized config wording.
- **Modify:** `skills/scale-deployment/SKILL.md` — replace hard-coded `harumi.yaml` trigger language with normalized config wording.
- **Modify:** `skills/create-namespace/SKILL.md` — replace hard-coded `harumi.yaml` trigger language with normalized config wording.
- **Modify:** `skills/sync-docs/SKILL.md` — make live AWS/Kubernetes state the source of truth when access is available, but degrade cleanly to repo-only sync when it is not.
- **Modify:** `skills/using-devops/SKILL.md` — clarify that drift review prefers live state when reachable but must not assume access.
- **Modify:** `README.md` — document that drift sync can refresh `harumi.yaml` and generated docs from live state when available.
- **Add:** `skills/deploy-app/evals/evals.json` entries — cover missing secret, wrong secret schema, missing ECR repo, feature-branch targetRevision, and isolated prod app naming.
- **Add:** `skills/using-devops/evals/evals.json` entries — cover `.devops.yaml` fallback and stale kube context detection.
- **Add:** `skills/infrastructure/evals/evals.json` entries — cover unrelated Terraform drift and quoted indexed targets.
- **Add or modify:** `skills/sync-docs/evals/evals.json` — cover live-state drift refresh and repo-only fallback behavior.

## Task 1: Add config compatibility and live context validation

**Files:**
- Modify: `hooks/session-start`
- Modify: `README.md`
- Modify: `skills/using-devops/SKILL.md`

- [ ] **Step 1: Update the hook design in `hooks/session-start`**

Add config resolution logic so the hook prefers `harumi.yaml` but falls back to `.devops.yaml` and records which file was used.

```bash
repo_config_path="${PWD}/harumi.yaml"
legacy_config_path="${PWD}/.devops.yaml"

if [ -f "$repo_config_path" ]; then
    user_config=$(cat "$repo_config_path" 2>/dev/null || true)
    config_source="harumi.yaml"
elif [ -f "$legacy_config_path" ]; then
    user_config=$(cat "$legacy_config_path" 2>/dev/null || true)
    config_source=".devops.yaml"
else
    config_source="not found"
    user_config="# No repo config found. Create harumi.yaml or .devops.yaml."
fi
```

- [ ] **Step 2: Add kube-context validation to the hook**

Append a warning block when configured contexts are missing from the local kube config so skills stop emitting dead contexts.

```bash
context_warnings=""
if command -v kubectl >/dev/null 2>&1; then
    available_contexts=$(kubectl config get-contexts -o name 2>/dev/null || true)
    while IFS= read -r configured_context; do
        [ -z "$configured_context" ] && continue
        if ! printf '%s\n' "$available_contexts" | grep -Fxq "$configured_context"; then
            context_warnings="${context_warnings}- Missing kube context: ${configured_context}\n"
        fi
    done <<EOF
$(printf '%s\n' "$user_config" | sed -n 's/^[[:space:]]*context:[[:space:]]*//p')
EOF
fi
```

- [ ] **Step 3: Surface the normalized-config behavior in `skills/using-devops/SKILL.md`**

Replace `harumi.yaml`-only wording with normalized wording that matches the hook output.

```md
Read the active repo config injected at session start. The hook may load either `harumi.yaml` or legacy `.devops.yaml`; always trust the reported source and any validation warnings over stale examples in docs.
```

- [ ] **Step 4: Update `README.md` configuration section**

Document the compatibility policy and the preferred future state.

```md
## Configuration

Preferred: `harumi.yaml`
Backward-compatible: `.devops.yaml`

The session-start hook loads `harumi.yaml` when present, otherwise falls back to `.devops.yaml`. It also validates configured Kubernetes contexts against the local kubeconfig and surfaces any mismatches in session context.
```

- [ ] **Step 5: Verify the hook and docs changes**

Run:

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
bash hooks/session-start >/tmp/harumi-session-start.json
rg -n "harumi.yaml|\\.devops.yaml|Missing kube context" /tmp/harumi-session-start.json README.md skills/using-devops/SKILL.md
```

Expected:
- Hook output includes `source: harumi.yaml` or `source: .devops.yaml`
- Missing contexts are called out when absent
- README/bootstrap skill both describe the compatibility behavior

- [ ] **Step 6: Commit**

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
git add hooks/session-start README.md skills/using-devops/SKILL.md
git commit -m "feat: support repo config compatibility in devops hook"
```

## Task 2: Refresh deploy-app to match real Harumi rollout patterns

**Files:**
- Modify: `skills/deploy-app/SKILL.md`
- Modify: `skills/deploy-app/references/ci-writeback-prod.md`
- Modify: `skills/deploy-app/references/ci-writeback-pattern.md`

- [ ] **Step 1: Add rollout-shape decision points to `skills/deploy-app/SKILL.md`**

Add explicit questions for isolated prod app vs shared prod app, branch strategy, ingress exposure, and naming variant.

```md
Ask for these if not provided:
1. App name
2. Environment
3. Shared deployment or isolated rollout app (`shared-prod` vs `isolated-prod-test`)
4. ArgoCD app filename (`argocd-app.yaml` vs `argocd-app-prod.yaml`)
5. CI workflow filename (`cd-eks.yaml` vs `cd-eks-prod.yaml`)
6. Initial `targetRevision` strategy (`main` vs temporary feature branch)
7. Ingress exposure (`internal` vs `internet-facing`)
```

- [ ] **Step 2: Replace the prod hard-coding in the environment table**

Update the prod row to acknowledge multiple valid conventions.

```md
| prod | `references/ci-writeback-prod.md` | `deploy/` | usually `main` | `cd-eks.yaml` or `cd-eks-prod.yaml` |
```

- [ ] **Step 3: Expand `ci-writeback-prod.md` with the isolated rollout variant**

Add a second architecture block showing the pattern that worked in the frontend rollout.

```md
### Variant: isolated prod rollout

<app-repo>/
├── argocd-app-prod.yaml
├── deploy/
└── .github/workflows/
    └── cd-eks-prod.yaml

Use this when validating a prod deployment path without immediately re-pointing the stable app definition.
```

- [ ] **Step 4: Add deploy preflight checks to `skills/deploy-app/SKILL.md`**

Insert a required preflight before manifest generation.

```md
Before generating files, verify:
- AWS Secrets Manager secret exists
- required JSON keys exist
- ECR repository exists
- OIDC role can assume and push
- ArgoCD repo secret exists in-cluster
- target cluster context exists locally
```

- [ ] **Step 5: Add the exact preflight commands to `ci-writeback-prod.md`**

```bash
aws secretsmanager get-secret-value --secret-id <secret-path> --region <aws-region>
aws ecr describe-repositories --repository-names <ecr-repo> --region <aws-region>
kubectl get secret -n argocd --context <context> | rg repo
kubectl config get-contexts -o name | rg "^<context>$"
```

- [ ] **Step 6: Verify deploy-app doc coverage**

Run:

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
rg -n "cd-eks-prod|argocd-app-prod|feature branch|internet-facing|Secrets Manager|ECR repository exists" skills/deploy-app
```

Expected:
- All rollout-specific terms appear in deploy-app docs
- No prod-only guidance forces `cd-eks.yaml`/`argocd-app.yaml`

- [ ] **Step 7: Commit**

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
git add skills/deploy-app/SKILL.md skills/deploy-app/references/ci-writeback-prod.md skills/deploy-app/references/ci-writeback-pattern.md
git commit -m "feat: align deploy-app with real rollout patterns"
```

## Task 3: Improve ArgoCD and Kubernetes skills for rollout lifecycles

**Files:**
- Modify: `skills/argocd/SKILL.md`
- Modify: `skills/kubernetes/SKILL.md`

- [ ] **Step 1: Add targetRevision lifecycle guidance to `skills/argocd/SKILL.md`**

```md
### TargetRevision lifecycle

Supported rollout path:
1. point app at a temporary feature branch for validation
2. verify sync, health, workflow write-back, and external reachability
3. move `targetRevision` back to `main` once rollout is stable
```

- [ ] **Step 2: Add isolated app-manifest guidance to `skills/argocd/SKILL.md`**

```md
When an app repo owns its own ArgoCD Application manifest (for example `argocd-app-prod.yaml`), treat that file as rollout control state even if the main GitOps repo remains unchanged.
```

- [ ] **Step 3: Add ingress exposure decision and ALB conflict warning to `skills/kubernetes/SKILL.md`**

```md
Before editing ingress:
1. Confirm whether traffic should be `internal` or `internet-facing`
2. Verify the selected subnets match that exposure model
3. Warn that changing ALB scheme can require ALB recreation and may trigger target-group association conflicts
```

- [ ] **Step 4: Add user-approved operational recovery handoff patterns**

Document explicit handoffs for:
- ingress delete/recreate
- ArgoCD app registration
- secret bootstrap/copy

```md
If the user explicitly authorizes a mutation for operational recovery, provide a step-by-step handoff with exact commands, rollback note, and verification commands instead of generic “apply/delete” language.
```

- [ ] **Step 5: Verify Argo/Kubernetes doc updates**

Run:

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
rg -n "targetRevision lifecycle|feature branch|argocd-app-prod|internet-facing|target-group association|operational recovery" skills/argocd/SKILL.md skills/kubernetes/SKILL.md
```

Expected:
- Both skills explicitly cover the rollout lifecycle and ingress pivot behavior

- [ ] **Step 6: Commit**

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
git add skills/argocd/SKILL.md skills/kubernetes/SKILL.md
git commit -m "feat: add rollout lifecycle guidance to argocd and kubernetes skills"
```

## Task 4: Make infrastructure and cross-skill docs drift-aware

**Files:**
- Modify: `skills/infrastructure/SKILL.md`
- Modify: `skills/sync-docs/SKILL.md`
- Modify: `skills/observability/SKILL.md`
- Modify: `skills/debug-pod/SKILL.md`
- Modify: `skills/rollback-deployment/SKILL.md`
- Modify: `skills/scale-deployment/SKILL.md`
- Modify: `skills/create-namespace/SKILL.md`

- [ ] **Step 1: Add quoted indexed-target examples to `skills/infrastructure/SKILL.md`**

```md
When emitting a targeted Terraform command with an indexed resource, always quote it for shell safety:

terraform plan -var-file=prod.tfvars -target='module.dns.aws_route53_record.platform_prod[0]'
```

- [ ] **Step 2: Add unrelated-drift handling**

```md
If plan output includes unrelated drift outside the requested scope, first prove whether the drift is pre-existing by diffing against the branch baseline and checking the live resource with cloud CLI. If the requested change is isolated, recommend a quoted targeted plan for operator review.
```

- [ ] **Step 3: Replace hard-coded `harumi.yaml` wording across secondary skills**

Normalize wording in the listed skills so they refer to “active repo config injected at session start” rather than a single filename.

```md
Read the active repo config injected at session start. It may come from `harumi.yaml` or legacy `.devops.yaml`; trust the injected source and warnings.
```

- [ ] **Step 4: Update `skills/sync-docs/SKILL.md`**

Teach sync-docs to treat both config filenames as valid inputs and to prefer writing the canonical future format explicitly.

```md
If both files exist, treat `harumi.yaml` as canonical and report `.devops.yaml` as migration debt.
If only `.devops.yaml` exists, sync against it but recommend migration.
```

- [ ] **Step 5: Verify cross-skill config wording**

Run:

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
rg -n "harumi\\.yaml" skills README.md | cat
```

Expected:
- Remaining `harumi.yaml` mentions are only in explicit migration docs or canonical examples
- Core operational guidance no longer assumes only one config filename

- [ ] **Step 6: Commit**

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
git add skills/infrastructure/SKILL.md skills/sync-docs/SKILL.md skills/observability/SKILL.md skills/debug-pod/SKILL.md skills/rollback-deployment/SKILL.md skills/scale-deployment/SKILL.md skills/create-namespace/SKILL.md
git commit -m "chore: make devops skills config-compatible and drift-aware"
```

## Task 5: Add eval coverage for the rollout failures we hit

**Files:**
- Modify: `skills/using-devops/evals/evals.json`
- Modify: `skills/deploy-app/evals/evals.json`
- Add or modify: `skills/infrastructure/evals/evals.json`

- [ ] **Step 1: Add bootstrap evals for config fallback and stale contexts**

Add eval cases asserting the assistant handles:
- repo only has `.devops.yaml`
- configured context is `eks-prod` but local kube context is `harumi-prod-eks`

```json
{
  "description": "Falls back to .devops.yaml and warns about missing kube context",
  "prompt": "Use the plugin in a repo that has only .devops.yaml with eks-prod context, but kubectl only has harumi-prod-eks."
}
```

- [ ] **Step 2: Add deploy-app evals for missing prerequisites**

Add cases for:
- missing Secrets Manager secret
- secret exists but missing `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- missing ECR repo
- isolated prod rollout naming (`argocd-app-prod.yaml`, `cd-eks-prod.yaml`)

```json
{
  "description": "Detects missing ECR repo before claiming rollout is ready",
  "prompt": "Create a prod deploy setup for frontend where the configured ECR repository does not yet exist."
}
```

- [ ] **Step 3: Add infrastructure eval for unrelated drift and quoted targets**

```json
{
  "description": "Recommends a quoted targeted plan when unrelated subnet drift appears",
  "prompt": "Terraform plan for platform.prod shows unrelated subnet tag removals and the user only wants the DNS change."
}
```

- [ ] **Step 4: Run the plugin benchmark aggregation**

Run:

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
python scripts/aggregate_benchmark.py
```

Expected:
- New eval entries are included without JSON/schema errors

- [ ] **Step 5: Commit**

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
git add skills/using-devops/evals/evals.json skills/deploy-app/evals/evals.json skills/infrastructure/evals/evals.json
git commit -m "test: add rollout regression evals for devops plugin"
```

## Task 6: Make drift sync treat AWS and Kubernetes as the source of truth when available

**Files:**
- Modify: `skills/sync-docs/SKILL.md`
- Modify: `skills/using-devops/SKILL.md`
- Modify: `README.md`
- Add or modify: `skills/sync-docs/evals/evals.json`

- [ ] **Step 1: Update `skills/sync-docs/SKILL.md` source-of-truth rules**

Add an explicit priority order for drift reconciliation.

```md
### Source-of-truth order

When live access is available:
1. AWS and Kubernetes live state are the source of truth for deployed reality
2. Repository files are the source of truth for intended configuration
3. Drift means generated artifacts (for example `harumi.yaml` and architecture docs) must be refreshed to match observed reality, and any mismatch in human-authored docs must be proposed to the user

When live access is unavailable:
- Fall back to repo-only sync
- Report that live drift could not be verified
- Do not invent cloud or cluster state
```

- [ ] **Step 2: Extend the generated-doc workflow to refresh `harumi.yaml` from drift**

Teach sync-docs that `harumi.yaml` is not just a static template; it is a generated projection that must be rewritten when Terraform outputs, AWS resources, cluster contexts, domains, or registries drift from the checked-in file.

```md
For `harumi.yaml`, rebuild these sections from the best available evidence:
- `aws.*` from Terraform files and live AWS account metadata when reachable
- `clusters[*].context`, `domain`, and `registry` from repo config plus live cluster / ingress / registry state when reachable
- `argocd.*` from GitOps repo structure and discovered app-of-apps paths

If the live source disproves the checked-in file, rewrite `harumi.yaml` to match the observed state and mention the drift in the sync summary.
```

- [ ] **Step 3: Add an explicit drift-review decision for human-authored docs**

```md
When live AWS or Kubernetes state disagrees with README, runbooks, or other human-authored docs:
- show the stale claim
- show the live fact (or repo-only fallback fact if live access is unavailable)
- propose the exact edit
- ask for approval before changing the human-authored file
```

- [ ] **Step 4: Update `skills/using-devops/SKILL.md` bootstrap guidance**

Clarify that drift review should prefer live state when reachable, but must not assume access is present for every session.

```md
If drift is detected, invoke `sync-docs` before other work. When AWS or Kubernetes read access is available, treat live state as deployed reality and refresh generated docs from it. When access is unavailable, fall back to repo-only reconciliation and tell the user what could not be verified.
```

- [ ] **Step 5: Update `README.md` sync behavior**

Document the same rule in user-facing docs so operators know what the plugin will rewrite automatically.

```md
Generated files such as `harumi.yaml` and architecture docs are refreshed from the best available source of truth. Live AWS / Kubernetes state wins when reachable; otherwise the plugin falls back to repository data and reports that live drift could not be checked.
```

- [ ] **Step 6: Add `sync-docs` eval coverage**

Add eval cases asserting:
- live Kubernetes context shows a different domain / context than the checked-in `harumi.yaml`, so the assistant says `harumi.yaml` should be regenerated
- live access is unavailable, so the assistant falls back to repo-only sync and explicitly calls out that live drift was not verified

```json
{
  "description": "Regenerates harumi.yaml when live cluster state disagrees with checked-in config",
  "prompt": "Sync docs for a repo where harumi.yaml says context eks-prod but live kubectl access shows harumi-prod-eks and a different ingress domain."
}
```

- [ ] **Step 7: Verify the drift-sync wording**

Run:

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
rg -n "source of truth|live state|repo-only|harumi.yaml" skills/sync-docs/SKILL.md skills/using-devops/SKILL.md README.md skills/sync-docs/evals/evals.json
```

Expected:
- sync-docs explicitly prefers live AWS/Kubernetes state when reachable
- bootstrap/docs say access is preferred but not assumed
- eval coverage exists for both live-state drift and repo-only fallback

- [ ] **Step 8: Commit**

```bash
cd /Users/wagnersza/git/harumi-io/harumi-devops-plugin
git add skills/sync-docs/SKILL.md skills/using-devops/SKILL.md README.md skills/sync-docs/evals/evals.json
git commit -m "feat: sync generated docs from live drift when available"
```

## Self-Review

- **Spec coverage:** This plan covers config compatibility, deploy-app convention drift, prerequisite preflights, Argo/Kubernetes rollout lifecycle guidance, Terraform drift handling, live-state drift sync for `harumi.yaml` and docs, and eval coverage for the failure modes seen in the frontend prod rollout.
- **Placeholder scan:** No `TODO`, `TBD`, or “similar to previous task” shortcuts remain.
- **Type consistency:** The plan consistently uses `harumi.yaml` / `.devops.yaml`, `argocd-app-prod.yaml`, `cd-eks-prod.yaml`, `targetRevision`, and `platform_prod` naming throughout.

## Execution Handoff

**Plan complete and saved to `docs/superpowers/plans/2026-04-22-devops-plugin-rollout-alignment.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
