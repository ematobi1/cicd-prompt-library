# PROMPT — GitLab CI/CD — Automated Testing & Deployment to AWS (Enterprise, Reusable Components)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `GITLAB` | GitLab.com / Self-Managed 17.x+ (Ultimate features used where available, OSS fallbacks listed) |
| `TARGET` | AWS (EKS via GitOps, or ECS / Lambda). Reuse the matching platform prompt's CD section |
| `REGISTRY` | GitLab Container Registry (protected tags, cleanup policies) mirrored to ECR |
| `IAC` | Terraform with GitLab-managed state **or** S3 backend. MR plan widget enabled |

---

## OBJECTIVE
Achieve on GitLab the same bar as the GitHub prompts: **nothing merges unless every job passes and every scanner is clean; artifacts are built once, signed and promoted by digest; deploys are progressive and self-rolling-back.**

---

## NON-NEGOTIABLE GUARDRAILS (GitLab CI/CD)
- **Federated auth:** `id_tokens` (OIDC) → AWS/Azure/GCP roles scoped to `project_path:<group>/<project>:ref_type:branch:ref:main` + environment. No long-lived cloud keys in CI/CD variables.
- **Protected `main`:**
  - No direct push. Merge requests require **Code Owner approval** and at least 2 approvals, and the author can't approve.
  - Approvals reset on push. "Pipelines must succeed" and "All threads must be resolved" are on.
  - **Merge trains** are enabled. Signed commits are required (push rule). Fast-forward/semi-linear history only.
- **Reusable:**
  - Logic lives in **CI/CD components** published to the CI/CD Catalog (`<group>/ci-components`), with typed `spec:inputs` and semver releases.
  - Projects `include: component: ...@<version>`.
  - Org-wide enforcement via a **compliance pipeline / pipeline execution policy** that injects the security jobs.
- **Pipeline hygiene:**
  - `workflow:rules` (no duplicate branch+MR pipelines). `rules:` only, never `only/except`.
  - `needs:` DAG for speed. `interruptible: true` on MR jobs. A `timeout:` on every job.
  - `retry: {max: 2, when: [runner_system_failure, stuck_or_timeout_failure, api_failure]}`, never on `script_failure`.
- **Runners:**
  - Ephemeral Kubernetes-executor runners (EKS/GKE), with tagged runner pools.
  - **No privileged Docker-in-Docker.** Build with rootless BuildKit or buildah.
  - Protected runners for protected branches/environments only.
- **Variables:** masked + protected + hidden. Prefer an external secrets integration (Vault / AWS Secrets Manager via `secrets:`).
- **Environments:** protected environments with deployment approvals for prod, `resource_group` per environment to serialize deploys, deployment freezes for change windows.
- **Zero tolerance:** the MR is blocked on any failed job, critical/high fixable vulnerability (security policy approval rules), or secret. Exceptions are tracked with an expiry date.

---

## PIPELINE — MR / merge train (`stages: [lint, test, build, scan, plan, deploy, verify]`)
| Stage | Tools (GitLab template → OSS fallback) | Fail condition |
|---|---|---|
| CI lint | `glab ci lint` / CI Lint API on every `.gitlab-ci.yml` + components change | invalid |
| Quality | linters, type check, unit + integration (services:), JUnit + Cobertura reports in the MR widget | any failure; coverage < 85% (`coverage:` regex + diff coverage) |
| SAST | `Jobs/SAST.gitlab-ci.yml` → Semgrep | high/critical |
| Secret Detection | `Jobs/Secret-Detection` → gitleaks | any finding |
| Dependency Scanning | `Jobs/Dependency-Scanning` → Trivy fs / OSV | fixable high/critical |
| Container Scanning | `Jobs/Container-Scanning` (Trivy) + Grype | fixable high/critical |
| IaC | `Jobs/SAST-IaC` → Checkov/Trivy config + tflint + Conftest | any high/deny |
| License | License scanning / Trivy license | denied license |
| Build | rootless BuildKit/buildah, multi-arch, SBOM (CycloneDX, attached as a `cyclonedx` report) | failure |
| Terraform plan | `gitlab-terraform plan` + `terraform` report artifact for the MR widget | error / policy deny |
| DAST (staging) | `DAST.gitlab-ci.yml` → ZAP baseline | high |

**Scan result policies** (Security Policies project) require Security-team approval when new critical/high findings appear. That makes the security gate non-bypassable per project.

---

## CD — on `main` after the merge train
1. Promote by digest. Sign with **cosign keyless using GitLab OIDC** (`SIGSTORE_ID_TOKEN`). Generate SLSA provenance (`RUNNER_GENERATE_ARTIFACTS_METADATA: "true"`).
2. Run `environment:` jobs dev (auto) → staging (auto) → prod (protected environment + approval rules + `resource_group: prod`). Each has `deployment_tier` set for DORA.
3. Deploy with the target's progressive mechanism (Argo Rollouts / CodeDeploy canary), with alarm-based auto rollback. Smoke tests run in the `verify` stage. `on_stop` environments clean up review apps.
4. **Review apps** per MR (ephemeral namespace/stack) with `auto_stop_in: 2 days`.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (GitLab-specific)
- Scheduled pipelines handle Terraform drift, container re-scans of deployed digests, and the expired-exception sweeper.
- **Renovate** runs as a scheduled pipeline (renovate-runner). Auto-merge goes through the merge train on green.
- Automatic rollback: a failed `verify` job triggers a rollback job (`when: on_failure`) that redeploys the last successful environment deployment. GitLab "Rollback environment" is the manual fallback.
- DORA metrics + Value Stream Analytics dashboards track change-failure rate and MTTR.

---

## DELIVERABLES
```
ci-components/templates/{quality,security,build-sign,terraform,deploy-eks,deploy-ecs,deploy-lambda,verify}.yml   # spec:inputs, semver-released
security-policies/{scan-result-policy.yml,pipeline-execution-policy.yml}
example-service/.gitlab-ci.yml          # ~20 lines of component includes
infra/bootstrap/{gitlab-oidc-provider.tf,iam-roles.tf,gitlab-project-settings.tf (gitlab provider: protected branches, approval rules, push rules, protected envs)}
runners/{values-eks.yaml}               # gitlab-runner Helm, k8s executor, no privileged
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-rollback.md,RUNBOOK-runner.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. An MR with a failing test, lint error, secret, fixable critical CVE, or IaC violation cannot merge. Each is blocked by a named job or policy.
2. No AWS keys exist in CI/CD variables. CloudTrail shows `AssumeRoleWithWebIdentity` from GitLab only.
3. Merges happen only through the merge train. Direct pushes and unsigned commits are rejected.
4. The prod deploy requires approval, is serialized by `resource_group`, and deploys the signed digest tested in the train.
5. A failed post-deploy verify triggers an automatic rollback.
6. No job runs privileged. Runners are ephemeral.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
