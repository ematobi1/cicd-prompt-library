# PROMPT — Azure CI/CD — Container Apps/AKS, ACR, Key Vault, Azure Policy (GitHub Actions)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `SUBSCRIPTIONS` | `dev`, `staging`, `prod`, in separate subscriptions under a Management Group |
| `RUNTIME` | **Azure Container Apps** (revisions + traffic split). For AKS, use the Kubernetes GitOps prompt |
| `IAC` | Terraform `azurerm` (consistent with the Terraform prompt). Add `bicep build` + PSRule if Bicep is used |
| `REGISTRY` | ACR Premium: private endpoint, content trust via Notation/cosign, Defender for Containers |
| `SECRETS` | Key Vault (RBAC mode, purge protection) via managed identity |
| `NETWORK` | Hub-spoke VNets, Private Endpoints, NSGs, no public data-plane endpoints |

---

## OBJECTIVE
Ship to Azure with **zero stored credentials, policy-compliant infrastructure, private networking, and progressive traffic shifting that rolls back on Azure Monitor alerts**.

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
- Auth: `azure/login` with **OIDC federated credentials** on a user-assigned managed identity per environment. The subject is `repo:<org>/<repo>:environment:<env>`. RBAC is scoped to the resource group, never Owner at subscription level.
- Azure Policy (DeployIfNotExists/Deny) plus Defender for Cloud regulatory compliance (e.g. HIPAA/HITRUST, CIS Azure) are assigned at Management Group scope.

---

## PIPELINE — PR / merge_group
| Stage | Tools | Fail condition |
|---|---|---|
| Code quality + tests | linters, type check, unit/integration (Azurite, Testcontainers), coverage gate | any failure |
| SAST / secrets / SCA | CodeQL, Semgrep, gitleaks, dependency-review | high/critical, or any secret |
| Image | Buildx → Trivy/Grype → Syft SBOM | fixable critical/high |
| IaC | terraform fmt/validate/tflint (azurerm ruleset), Checkov, Trivy config, **PSRule for Azure** (Well-Architected rules) | any high |
| Policy what-if | `terraform plan` + Conftest (no public IPs on data services, private endpoints required, TLS 1.2+, diagnostic settings to Log Analytics, tags) | any deny |
| Cost | Infracost (azurerm) | over budget without approval |

---

## CD — merge to `main`
1. Push to ACR by digest. Sign with cosign/Notation. Attach provenance.
2. Terraform apply (saved plan) dev → staging → prod. Prod uses a GitHub Environment with required reviewers.
3. **Container Apps:** deploy a new revision at 0% traffic, smoke-test its revision-specific FQDN, then shift 10 → 50 → 100%. At each step, query Azure Monitor/App Insights (failed requests < 1%, p95 < SLO).
4. On a breach, shift 100% back to the previous revision and deactivate the bad one.
5. **App Service alternative:** use a staging slot, warm-up, then swap with preview. Auto-swap back on alert.
6. Staging runs Playwright e2e, k6 and ZAP. Defender for Containers must show no new high findings.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (Azure-specific)
- An Azure Monitor alert action group calls a GitHub `repository_dispatch` → rollback workflow, which shifts traffic to the last good revision.
- Scheduled Terraform drift detection, as in the Terraform prompt. Azure Policy remediation tasks auto-fix DeployIfNotExists findings.
- A Defender for Cloud recommendation export (continuous export → Logic App / GitHub issue) creates tracked work items for new findings.

---

## DELIVERABLES
```
platform-workflows/.github/workflows/{az-ci,az-deploy,az-rollback,az-drift}.yml
platform-workflows/.github/actions/{az-oidc-login,aca-traffic-shift,az-verify-image}/action.yml
infra/{bootstrap/(federated-identities,acr,keyvault,log-analytics,policy-assignments).tf,live/{dev,staging,prod}/}
policy/{conftest/*.rego,psrule/ps-rule.yaml}
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-rollback.md,RUNBOOK-key-vault-access.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. No client secrets exist in GitHub. Sign-in logs show only federated credential use.
2. A storage account with public access or no private endpoint is blocked by policy in CI and denied by Azure Policy.
3. A bad revision with a 5xx spike is shifted back to 0% automatically.
4. Prod deploys only from `main` after approval, using the reviewed plan.
5. All resources send diagnostics to Log Analytics, verified by a policy compliance query.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
