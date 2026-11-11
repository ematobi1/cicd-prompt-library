# PROMPT — Terraform IaC CI/CD — Modules & Multi-Environment Live Infra (GitHub Actions)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `TF_VERSION` | ≥ 1.9 (pinned in `.terraform-version`) |
| `CLOUD` | AWS. Adapt the provider/auth for Azure/GCP |
| `LAYOUT` | `modules/` (versioned, reusable) + `live/<env>/<stack>/` (small blast radius per stack) |
| `STATE` | S3 + native lockfile (`use_lockfile = true`), KMS-encrypted, versioned, one bucket per account |
| `ENVS` | `dev`, `staging`, `prod`, in separate accounts |

---

## OBJECTIVE
Infrastructure changes are **reviewed as plans, gated by policy and cost, applied only from `main` using the exact approved plan**, continuously checked for drift, and every module is tested and versioned like software.

---

## NON-NEGOTIABLE GUARDRAILS (GitHub Actions)
- **Federated auth only:** use OIDC to the cloud and registry. No long-lived keys in GitHub. Trust is scoped to `repo:<org>/<repo>:environment:<env>`. Each role or identity is least-privilege, per environment.
- **Workflow hardening:**
  - Top-level `permissions: {}`, with job-level grants only.
  - Every third-party action is pinned to a full commit SHA, kept current by Renovate.
  - `step-security/harden-runner` is the first step of every job.
  - `actionlint` and `zizmor` checks are required.
  - Every job has `timeout-minutes`. `concurrency` cancels superseded PR runs and queues deploys per environment.
  - No `pull_request_target` that checks out PR code. Fork PRs never get secrets.
- **`main` ruleset as code:**
  - PRs are required, with at least 2 approvals including CODEOWNERS. Stale approvals are dismissed, and the last pusher can't approve.
  - Required checks pass and the branch is up to date, through the **merge queue** (`merge_group`).
  - Signed commits, linear history, resolved conversations. No force pushes or deletions.
  - The only bypass is an audited break-glass team.
- **One required check: `ci-gate`.** It aggregates every job (`if: always()`) and fails when anything upstream failed or was skipped.
- **Zero tolerance:** no failing tests, no lint/type errors, no secrets, and no critical/high vulnerabilities with a fix available.
  - Exceptions live in ignore files with an owner, a reason, and an **expiry date** that CI enforces.
  - All SARIF goes to code scanning. Every job writes a `$GITHUB_STEP_SUMMARY`.
- **Reusable by design:**
  - Logic lives in `<org>/platform-workflows` (`on: workflow_call`, typed inputs, secrets passed explicitly) plus composite actions, versioned with semver tags.
  - Consumers pin to a SHA, and a caller workflow is about 20 lines.
- **Two roles per environment:**
  - The `plan` role is read-only and can be assumed from PRs.
  - The `apply` role can only be assumed from `main` together with the GitHub Environment `<env>`, which requires reviewers for prod.
- **Destructive changes are gated.** Any plan containing `delete` or `replace` on stateful resources (DBs, buckets, KMS, EBS) fails unless the PR carries an `allow-destroy` label, approved by a CODEOWNER from the platform team.
- **Locking:** `.terraform.lock.hcl` is committed with hashes for `linux_amd64,darwin_arm64`. Providers are pinned with `~>`, and modules are pinned to exact versions.

---

## PIPELINE — PR / merge_group (only for changed stacks, via `dorny/paths-filter` → dynamic matrix)
| Stage | Tools | Fail condition |
|---|---|---|
| Format | `terraform fmt -check -recursive` | any diff |
| Validate | `terraform init -backend=false && validate` | any error |
| Lint | tflint with the provider ruleset (aws/azurerm/google) | any warning or above |
| Security | Trivy config, Checkov | any high/critical |
| Policy | Conftest/OPA on `terraform show -json plan` (encryption required, no public ingress `0.0.0.0/0`, tags required, approved regions/instance families, no `*` IAM actions) | any deny |
| Module tests | `terraform test` (native, mocked providers) + Terratest for integration in a sandbox account, destroyed after | any failure |
| Docs | terraform-docs (README in sync) | drift |
| Plan | `terraform plan -out=tfplan -lock-timeout=5m` per changed stack/env. Plan uploaded as an encrypted artifact, plus a sticky PR comment with summary (add/change/destroy) | plan error |
| Cost | Infracost diff comment | monthly delta > threshold (e.g. +$500) without the `cost-approved` label |

---

## APPLY — merge to `main`
1. Apply runs per environment, in order: dev → staging → prod. Prod requires reviewers and a wait timer.
2. **Apply the saved plan artifact from the PR's merge-queue run.** If state moved (stale plan), re-plan, fail, and require a re-review. Never use `-auto-approve` on a fresh plan in prod.
3. Run post-apply checks: smoke tests (resource reachability, Terratest assertions), then a final `terraform plan -detailed-exitcode` that must return 0.
4. **Module releases:** release-please creates semver tags and publishes to a private registry (TFC/TFE, or a Git tag source). Consumers bump through Renovate.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (Terraform-specific)
- **Drift detection every 6 h:** `plan -detailed-exitcode` runs per stack/env.
  - In **dev**, drift is auto-reconciled (apply).
  - In **staging/prod**, the job opens or updates an issue with the diff and alerts. Prod is never auto-applied.
- Stale state locks older than 2 h raise an alert with a runbook for `force-unlock`. CI never force-unlocks automatically.
- Failed applies post the partial-apply resources and the recovery steps to the run summary.

---

## DELIVERABLES
```
platform-workflows/.github/workflows/{tf-ci,tf-apply,tf-drift,tf-module-release}.yml
platform-workflows/.github/actions/{tf-setup,tf-plan-comment,tf-policy}/action.yml
modules/<name>/{main.tf,variables.tf,outputs.tf,versions.tf,tests/*.tftest.hcl,README.md}
live/{dev,staging,prod}/<stack>/{backend.tf,main.tf,terraform.tfvars}
bootstrap/{state-backend.tf,oidc-roles.tf,github-rulesets.tf,github-environments.tf}
policy/terraform/*.rego, .tflint.hcl, .checkov.yaml, infracost.yml
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-state-recovery.md,RUNBOOK-drift.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. A public S3 bucket, `0.0.0.0/0` SSH, or an untagged resource is blocked by a named policy check.
2. A PR that deletes an RDS instance is blocked without the `allow-destroy` label and approval.
3. A prod apply uses the exact reviewed plan. A stale plan fails safely.
4. The PR role cannot write: an attempted apply from a PR is denied by IAM.
5. Manual console drift in dev is reconciled within 6 h. Drift in prod opens an issue.
6. A module change runs `terraform test` and publishes a new semver tag on merge.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
