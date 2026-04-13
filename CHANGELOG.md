# Changelog

## 0.8.0

### Breaking Changes

- Removed `.devops.yaml` config system and `config/default.devops.yaml` — replaced by `harumi.yaml` per-repo manifest
- Removed `setup-devops-config` skill — replaced by `sync-docs`
- Removed generic multi-provider abstractions (GCP, Azure) from all skills — plugin is now AWS-only

### Added

- `sync-docs` skill — maintains generated docs (`docs/architecture/*`, `harumi.yaml`) automatically and proposes edits to human-authored docs (`README.md`, `CLAUDE.md`, `AGENTS.md`, `docs/runbooks/*`) with user approval
- Drift detection on session start — compares `.harumi-last-sync` commit SHA with HEAD, classifies changed files, triggers sync-docs when drift is detected
- Multi-repo awareness — bootstrap skill declares `harumi-io/infrastructure` and `harumi-io/harumi-k8s` as managed repos
- Cluster read-access rules — explicit allowed/forbidden kubectl command lists in bootstrap skill
- Observability endpoints in `harumi.yaml` — skills can query Prometheus, Grafana, Loki, Tempo, Alertmanager directly
- `docs/architecture/` directory for generated architecture docs
- `docs/runbooks/` directory for operational runbooks

### Changed

- Session-start hook reads `harumi.yaml` instead of `.devops.yaml`
- Bootstrap skill (`using-devops`) updated with harumi-specific context
- Infrastructure skill and references cleaned of GCP/Azure content
- Observability skill updated to reference `harumi.yaml` endpoints

## 0.3.0

- 7 new operations command skills for daily DevOps tasks:
  - `create-iam-user`: generate Terraform files for new IAM developer/admin users
  - `remove-iam-user`: remove IAM user Terraform config and directory
  - `create-service-account`: create IAM service accounts (simple or with access keys + Secrets Manager)
  - `create-vpn-creds`: generate VPN client certificate and export `.ovpn` config
  - `revoke-vpn-creds`: revoke VPN client certificates
  - `list-vpn-users`: list all VPN certificates and their status
  - `rotate-access-keys`: rotate IAM access keys (create new, deactivate old)
- Registered all operations commands in `using-devops` bootstrap skill with trigger rules
- Evals for all 7 operations command skills

## 0.2.0

- `setup-devops-config` skill: detects stack from codebase and generates `.devops.yaml`
- Registered `setup-devops-config` in the `using-devops` bootstrap skill trigger rules
- Eval infrastructure: `eval-viewer/` and `scripts/aggregate_benchmark.py` (ported from skill-creator)
- Evals for all three skills: `devops`, `setup-devops-config`, `using-devops`
- Sample mock-repo fixtures for `setup-devops-config` evals (AWS+GitHub Actions, GCP+GitLab CI)

## 0.1.0

- Initial release
- Plugin scaffold with Claude Code and Cursor manifests
- Session-start hook with `.devops.yaml` config loading
- `using-devops` bootstrap skill
- `devops` skill (generalized from Harumi.io devops skill)
- Default `.devops.yaml` config
