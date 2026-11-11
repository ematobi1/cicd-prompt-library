# PROMPT — Enterprise CI/CD on GitHub Actions → AWS (Production-Grade)

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated, multi-account AWS production systems. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `APP_NAME` | `my-service` |
| `LANGUAGE` | Python 3.12 (FastAPI). Adapt the toolchain if it's Go or Node |
| `DEPLOY_TARGET` | Amazon EKS (Helm + Argo Rollouts canary). Alt: ECS Fargate + CodeDeploy blue/green |
| `IAC` | Terraform ≥ 1.9, remote state in S3 with native lockfile, KMS-encrypted |
| `AWS_ACCOUNTS` | `dev`, `staging`, `prod`, one account each, under AWS Organizations |
| `AWS_REGION` | `us-east-1` |
| `REGISTRY` | Amazon ECR, immutable tags, scan-on-push, KMS encryption |
| `GITHUB_ORG` | `<org>`. Reusable workflows live in `<org>/platform-workflows` |

## OBJECTIVE
Build a **reusable, scalable, self-correcting** CI/CD system where:
1. **Nothing reaches `main` unless every check passes.** That means zero failing checks, zero critical/high vulnerabilities with an available fix, zero leaked secrets, zero lint/type errors, and coverage at or above the gate.
2. Every push and every PR runs the full quality and security gate.
3. Artifacts are **built once, signed, attested, and promoted by digest** through dev → staging → prod.
4. Production deploys are progressive and **roll back automatically** on SLO/alarm breach, with no human in the loop.
5. Any service repo can adopt the whole thing with a ~20-line caller workflow.

---

## NON-NEGOTIABLE GUARDRAILS
- **Auth:** GitHub OIDC → AWS IAM roles only. No long-lived AWS keys anywhere. Use one role per environment, with trust conditions scoped to `repo:<org>/<repo>:environment:<env>` (plus `ref:refs/heads/main` for the prod plan/apply roles). Each role is least-privilege.
- **Workflow security:**
  - Top-level `permissions: {}`, with job-level grants only (`contents: read` by default).
  - Every third-party action is **pinned to a full commit SHA** with a version comment. Renovate keeps them updated.
  - `step-security/harden-runner` is the first step in every job (egress audit in non-prod, `block` with an allowlist on deploy jobs).
  - No `pull_request_target` that checks out PR code. Fork PRs never get secrets or OIDC.
  - `actionlint` and `zizmor` lint every workflow change, and that check is required.
  - `timeout-minutes` is set on every job. No job runs unbounded.
  - `concurrency` is set: cancel in-progress runs on PR branches, never cancel deploys (queue them per environment).
- **Repo rulesets on `main`** (codify them with Terraform `github` provider or `gh api`, not ClickOps):
  - PRs are required, with at least 2 approvals, including **CODEOWNERS**.
  - Stale approvals are dismissed on push, and the last pusher can't self-approve.
  - All required status checks must pass, and the branch must be up to date. **Merge queue** is enabled (workflows listen to `merge_group`).
  - Signed commits, linear history, and resolved conversations are required.
  - No force pushes, no deletions, and no bypass actors except the break-glass team, which is audited.
  - Tag protection covers `v*`.
- **Secrets:** AWS Secrets Manager or SSM Parameter Store (SecureString), fetched at runtime through OIDC. GitHub Environment secrets are used only for non-AWS tokens. Push protection and secret scanning are on at org level.

---

## PIPELINE DESIGN

### A. PR / push / merge_group — `ci.yml` (one required check: `ci-gate`)
Run these as parallel jobs, then **one aggregating `ci-gate` job** (`if: always()`) that fails if any upstream job failed or was skipped. Branch protection requires only `ci-gate`, so the rule stays stable as jobs evolve.

| Stage | Tools | Fail condition |
|---|---|---|
| Workflow lint | actionlint, zizmor | any finding |
| Format / lint | ruff format --check, ruff, (gofmt/golangci-lint, eslint/prettier) | any finding |
| Type check | mypy --strict (tsc --noEmit) | any error |
| Unit tests | pytest + pytest-xdist, junit + coverage XML | any failure; line coverage < **85%**, or diff coverage < **90%** (diff-cover) |
| Integration tests | testcontainers / GH service containers (Postgres, LocalStack) | any failure |
| Contract / API tests | schemathesis against the OpenAPI spec | any failure |
| SAST | CodeQL (security-extended) + Semgrep (p/ci, p/owasp-top-ten) | any high/critical |
| Secrets | gitleaks (full history on push to main, diff on PR) | any finding |
| Dependency review | `actions/dependency-review-action` | new vuln ≥ moderate, or denied license (GPL/AGPL) |
| SCA | Trivy fs / osv-scanner | critical/high with a fix available |
| IaC | `terraform fmt -check`, `validate`, tflint, Trivy config, Checkov, Conftest (OPA policies in `policy/`) | any failure / any high finding |
| Terraform plan | per env, posted as a PR comment, saved as an artifact | plan error, or a policy violation (no public S3, no `0.0.0.0/0` ingress, encryption required, tags required) |
| Container build | Docker Buildx, multi-stage, distroless/chainguard base, non-root, `--provenance` + `--sbom`, GHA cache | build failure |
| Image scan | Trivy image + Grype | critical/high with a fix available |
| Dockerfile lint | hadolint | ≥ warning |
| Helm/K8s | helm lint, kubeconform, kube-linter / Polaris | any error |
| SBOM | Syft (SPDX + CycloneDX) uploaded as an artifact | n/a |

Rules:
- Upload all SARIF to GitHub code scanning. Write a `$GITHUB_STEP_SUMMARY` table for every job.
- Use `.trivyignore` / `.semgrepignore` only with an owner, a reason, and an **expiry date**. A CI check fails the build once an exception has expired.
- Cache everything that's safe to cache (setup-* caches, buildx GHA cache, Terraform plugin cache). The target is PR CI wall-clock **≤ 10 min p95**.

### B. Merge to `main` — `release.yml`
1. Rebuild **is not allowed**. Promote the image that the merge-queue run built and tested, by **digest** (use a tag of `sha-<git sha>`, and always reference `@sha256:`).
2. Push to ECR, then **cosign keyless sign** (Sigstore OIDC) and attach SLSA build provenance with `actions/attest-build-provenance`.
3. Release automation: **release-please** (Conventional Commits enforced by a commitlint PR check) handles semver, the changelog and GitHub Releases.

### C. CD — `deploy.yml` (reusable, called per environment)
`dev` (auto) → `staging` (auto, gated on dev health) → `prod` (GitHub Environment: required reviewers, 5-min wait timer, deployment-branch policy `main` only).

Each deploy runs these steps in order:
1. Assume the env role through OIDC.
2. Run **`cosign verify` + `gh attestation verify`** on the digest. If verification fails, the deploy stops. In-cluster, enforce the same check with a Kyverno `verifyImages` policy.
3. `terraform apply` the **saved, reviewed plan**. Re-plan and fail if it drifts from the approved plan.
4. Run the progressive rollout:
   - **EKS:** Argo Rollouts canary at 10% → 25% → 50% → 100%, with an `AnalysisTemplate` that queries CloudWatch/Prometheus for 5xx rate < 1% and p99 latency < SLO.
   - **ECS:** CodeDeploy `CanaryTenPercent5Minutes`, with CloudWatch alarms attached.
5. Post-deploy:
   - Smoke tests.
   - In staging, also run k6 load tests (thresholds = SLOs), the Playwright e2e suite, and an OWASP ZAP baseline DAST scan, failing on any high finding.
6. **Automatic rollback** if the analysis or an alarm fails. The pipeline marks the deployment failed, reverts to the last good digest, and notifies the team.
7. Record a GitHub Deployment status. Emit DORA metrics (deploy frequency, lead time, change-failure rate, MTTR).

---

## SELF-CORRECTION (built in, not aspirational)
- **Auto-rollback:** see C.6. A rollback needs no human action.
- **Transient-failure retry:** retry network-bound steps only (registry pulls, package installs, ECR push) up to 3× with backoff. **Never** retry tests to mask flakiness.
- **Flaky-test quarantine:** a nightly job reruns failures, and any test that passes then fails across reruns is tagged `@flaky`. The job opens an issue assigned to CODEOWNERS. Quarantined tests still run but are non-blocking for 7 days max, after which the build fails again.
- **Drift detection:** a scheduled (every 6 h) `terraform plan -detailed-exitcode` runs per env. When it finds drift, it opens or updates a GitHub issue and alerts Slack.
- **Dependency hygiene:** Renovate (grouped and scheduled, with `pinDigests: true`). Patch and minor updates **auto-merge only after `ci-gate` passes in the merge queue**. Majors need a human.
- **Expired-exception sweeper:** the job fails and opens an issue when a security-ignore entry expires.
- **Failure triage:** on failure, the pipeline posts a job-summary root-cause section (failed step, last 50 log lines, link) to the PR or Slack.

---

## REUSABILITY & SCALE
- **`<org>/platform-workflows`** holds:
  - `.github/workflows/` with `ci.yml`, `release.yml`, `deploy.yml`, `terraform.yml`, `security-scan.yml`, `drift.yml` (all `on: workflow_call`, with typed `inputs` and `secrets: inherit` avoided; secrets are passed explicitly).
  - `.github/actions/`: composite actions `setup-toolchain`, `aws-oidc-login`, `build-sign-push`, `verify-artifact`, `notify`.
  - Versioning: semver tags (`v1`, `v1.2.0`). Service repos pin to a SHA via Renovate.
- **Service repo** needs only:
  ```yaml
  jobs:
    ci:     { uses: <org>/platform-workflows/.github/workflows/ci.yml@<sha>, with: { language: python } }
    deploy: { needs: ci, if: github.ref == 'refs/heads/main', uses: <org>/platform-workflows/.github/workflows/deploy.yml@<sha>, with: { environment: dev } }
  ```
- **Monorepo support:** `dorny/paths-filter` builds a dynamic matrix so only the services that changed get built, tested and deployed.
- **Runners:** GitHub-hosted runners by default. For heavy or VPC-bound jobs, use self-hosted **ephemeral** runners via Actions Runner Controller on EKS, scaling to zero, with one job per pod and no persistent state.
- **Org-level enforcement:** required workflows / repository rulesets at the org level, so every repo gets `ci-gate`, secret scanning and CodeQL automatically.

---

## DELIVERABLES (produce all of these)
```
platform-workflows/
  .github/workflows/{ci,release,deploy,terraform,security-scan,drift,flaky-quarantine,exception-sweeper}.yml
  .github/actions/{setup-toolchain,aws-oidc-login,build-sign-push,verify-artifact,notify}/action.yml
  policy/{terraform,kubernetes}/*.rego          # Conftest/OPA policies
  docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-rollback.md,RUNBOOK-break-glass.md}
infra/bootstrap/                                  # Terraform
  github-oidc-provider.tf, iam-roles-per-env.tf, ecr.tf, kms.tf, state-backend.tf
  github-rulesets.tf, github-environments.tf      # branch protection + envs as code
example-service/
  .github/workflows/{ci-cd.yml}, Dockerfile, helm/ (Rollout + AnalysisTemplate), tests/, renovate.json, .pre-commit-config.yaml, CODEOWNERS
ASSUMPTIONS.md
```
- `.pre-commit-config.yaml` mirrors the CI linters (ruff, mypy, gitleaks, terraform fmt, actionlint, hadolint), so failures are caught before the push.
- A `Makefile` with `make ci-local` that runs the same checks in a container, using `act` for workflow testing.

## ACCEPTANCE CRITERIA (verify each, then report pass/fail)
1. A PR with a lint error, a failing test, a coverage drop, a hard-coded secret, a critical CVE, or a public S3 bucket is **blocked**, each by a distinct, clearly named check.
2. A clean PR passes `ci-gate` in ≤ 10 min. It can only merge via the merge queue, with CODEOWNERS approval.
3. A direct push to `main` is rejected. An unsigned commit is rejected.
4. The digest deployed to prod is byte-identical to the one tested in the merge queue. Signature and provenance verification pass, and a tampered or unsigned image is refused.
5. A canary with an injected 5xx spike is **automatically rolled back** with no human action, and an alert is sent.
6. No AWS access keys exist in GitHub. `grep -r "AKIA"` finds nothing. CloudTrail shows only `AssumeRoleWithWebIdentity`.
7. `actionlint` and `zizmor` are clean. Every action is SHA-pinned, and every job has `timeout-minutes` and an explicit `permissions` block.
8. A new service is onboarded with only the caller workflow and `helm/` values, in under 30 minutes, following `ONBOARDING.md`.

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every file in the tree above, complete.
3. End with a validation report: each acceptance criterion, how you verified it (command and output), and its pass/fail status. If something can't be verified locally, state exactly what to run in the real environment.
