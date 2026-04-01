# Changelog

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
