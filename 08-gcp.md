# PROMPT — Google Cloud CI/CD — Cloud Run/GKE, Artifact Registry, Binary Authorization, Cloud Deploy (GitHub Actions)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `PROJECTS` | `<app>-dev`, `<app>-staging`, `<app>-prod` under an org folder |
| `RUNTIME` | **Cloud Run** (GKE Autopilot supported via the same Cloud Deploy pipeline) |
| `REGISTRY` | Artifact Registry (regional, CMEK, vulnerability scanning on) |
| `DELIVERY` | **Cloud Deploy**: canary 10/50/100 with verification + automation rules |
| `GUARDRAILS` | Binary Authorization (attestor-required), Org Policies, VPC-SC for sensitive projects |
| `IAC` | Terraform `google` provider |

---

## OBJECTIVE
Deploy to GCP with **Workload Identity Federation, attestation-enforced images, and Cloud Deploy canaries that verify and roll back on their own**.

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
- Auth: `google-github-actions/auth` with **Workload Identity Federation**. The provider has an attribute condition `assertion.repository_owner == '<org>'`, and access is per-env service account impersonation. **No JSON keys.** Org policy `iam.disableServiceAccountKeyCreation` is enforced.

---

## PIPELINE — PR / merge_group
| Stage | Tools | Fail condition |
|---|---|---|
| Code quality + tests | linters, type check, unit/integration (emulators: Firestore/PubSub, Testcontainers), coverage gate | any failure |
| SAST / secrets / SCA | CodeQL, Semgrep, gitleaks, dependency-review, OSV-Scanner | high/critical, or any secret |
| Image | Buildx → Trivy/Grype → SBOM | fixable critical/high |
| IaC | terraform fmt/validate/tflint (google ruleset), Checkov, Trivy config, Conftest (no `allUsers` IAM, CMEK, private GKE, uniform bucket access) | any high/deny |
| Cloud Deploy config | `gcloud deploy apply --dry-run` style validation of `clouddeploy.yaml` + skaffold render | invalid |

---

## CD — merge to `main`
1. Push to Artifact Registry by digest. Sign with cosign **and** create a Binary Authorization attestation (KMS-backed attestor) after all scans pass.
2. `gcloud deploy releases create` → Cloud Deploy pipeline dev → staging → prod. Prod requires a **Cloud Deploy approval**, wired to a GitHub Environment approval.
3. Canary 10 → 50 → 100 with a `verify` job at each phase (smoke + Cloud Monitoring SLO query). Automation rules promote on success and **roll back automatically on a failed verify**.
4. Binary Authorization in **enforce** mode on Cloud Run/GKE: an image without the attestation is denied.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (GCP-specific)
- Cloud Deploy automation: `repairRollout` retries transient verify failures once, then rolls back.
- Cloud Monitoring burn-rate alerts → Pub/Sub → Cloud Function → `gcloud deploy targets rollback`.
- Scheduled Terraform drift detection. Security Command Center findings are exported to GitHub issues.

---

## DELIVERABLES
```
platform-workflows/.github/workflows/{gcp-ci,gcp-release,gcp-drift}.yml
platform-workflows/.github/actions/{gcp-wif-auth,binauthz-attest}/action.yml
deploy/{clouddeploy.yaml,skaffold.yaml,run-service.yaml}
infra/{bootstrap/(wif,artifact-registry,binauthz,kms,org-policies).tf,live/{dev,staging,prod}/}
policy/conftest/*.rego
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-rollback.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. No service account keys exist. Audit logs show WIF token exchange only.
2. An image without a BinAuthz attestation is denied at deploy time.
3. A canary with a failed verification rolls back automatically.
4. An `allUsers` IAM binding or a public bucket is blocked in CI.
5. Prod releases require a recorded approval.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
