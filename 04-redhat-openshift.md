# PROMPT — Red Hat OpenShift CI/CD — UBI Images, ACS Security, OpenShift GitOps (GitHub Actions)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `CLUSTERS` | OpenShift 4.x (ROSA / ARO / self-managed): `dev`, `staging`, `prod` |
| `REGISTRY` | Red Hat Quay (or the internal registry mirror). Images built on **UBI 9 (ubi-minimal / ubi-micro)** |
| `BUILD_TOOL` | `redhat-actions/buildah-build` + `push-to-registry` (rootless, no Docker daemon) |
| `SECURITY` | Red Hat Advanced Cluster Security (ACS / StackRox) via `roxctl` |
| `DEPLOY` | **OpenShift GitOps (Argo CD), pull model**, plus Argo Rollouts. Helm or Kustomize manifests in a separate `config` repo |
| `SECRETS` | External Secrets Operator → HashiCorp Vault / AWS Secrets Manager |

---

## OBJECTIVE
Ship workloads to OpenShift that **pass ACS policy, run under the `restricted-v2` SCC, are signed and verified, and are deployed only by GitOps**. CI never holds cluster-admin credentials or runs `oc apply` against prod.

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
- **Pull-based GitOps:** CI has **no cluster credentials**. It only builds, scans and signs images, then opens a PR to the `config` repo. Argo CD inside each cluster reconciles from that repo.
- The ACS API token is scoped to the **Continuous Integration** role and stored as a GitHub Environment secret. Rotate it every 90 days.

---

## PIPELINE — PR / merge_group
| Stage | Tools | Fail condition |
|---|---|---|
| Code quality | language linters, type check, unit + integration tests, coverage gate (85% line / 90% diff) | any failure |
| SAST / secrets / SCA | CodeQL, Semgrep, gitleaks, dependency-review | high/critical, or any secret |
| Containerfile | hadolint; must `FROM registry.access.redhat.com/ubi9/...@sha256` | any warning or above |
| Build | buildah (rootless, multi-arch), OCI labels, SBOM via Syft | failure |
| OpenShift compatibility | image runs as an **arbitrary UID with GID 0** (group-writable dirs), no fixed `USER 0`, no privileged ports | test failure |
| ACS image check | `roxctl image scan` + `roxctl image check` (enforced policies: fixable CVSS ≥ 7, no package managers in prod image, required labels) | any enforced violation |
| ACS deploy check | `roxctl deployment check` on rendered manifests (no privileged, no hostPath, resource limits, probes required) | any enforced violation |
| Manifests | `helm lint`/`kustomize build`, kubeconform with OpenShift CRD schemas (Route, SCC, DeploymentConfig not allowed), kube-linter, Conftest | any error |
| SCC simulation | render then check `securityContext` against `restricted-v2` (runAsNonRoot, drop ALL, seccomp RuntimeDefault) | violation |

---

## CD — GitOps promotion
1. On merge to `main`:
   - Push the image to Quay and sign it with cosign.
   - Attach SLSA provenance. Optionally also sign with **Red Hat Trusted Artifact Signer**.
   - Open a PR to `config/overlays/dev` that bumps the image **digest**. It auto-merges.
2. Argo CD syncs dev (`automated: {prune: true, selfHeal: true}`).
3. A post-sync hook Job runs the smoke tests. When they pass, the pipeline opens a promotion PR to staging, which auto-merges after the e2e and ZAP DAST jobs pass.
4. **Prod:** the promotion PR needs CODEOWNER approval. Argo Rollouts runs a canary (10/25/50/100) with an AnalysisTemplate against OpenShift Monitoring (Prometheus/Thanos): 5xx < 1%, p99 < SLO.
5. **Admission:** an ACS policy enforces signature verification (signature integration) and blocks deploy-time violations. Every namespace has a default-deny NetworkPolicy, Routes use TLS `reencrypt`/`edge`, and HSTS is on.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (OpenShift-specific)
- Argo CD `selfHeal` reverts manual `oc edit` drift automatically. An out-of-sync alert fires if the revert fails.
- Argo Rollouts aborts and rolls back automatically on a failed analysis.
- ACS runtime policies alert or kill pods on anomalous processes. A nightly ACS re-scan of the running images opens issues for new CVEs.
- A weekly UBI base rebuild (Renovate on the UBI digest) triggers a rebuild and promotion PR.

---

## DELIVERABLES
```
platform-workflows/.github/workflows/{ocp-ci,ocp-release,ocp-promote,ocp-rescan}.yml
platform-workflows/.github/actions/{buildah-build-sign,roxctl-check,gitops-bump}/action.yml
example-service/{Containerfile,.hadolint.yaml,tests/}
config-repo/{base/,overlays/{dev,staging,prod}/,rollouts/analysis-template.yaml,argocd/applicationset.yaml}
policy/{acs-policies.json,conftest/*.rego}
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-rollback.md,RUNBOOK-acs-violation.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. A Containerfile that is non-UBI, runs as root, or needs `anyuid` is blocked before merge.
2. An ACS-enforced violation (fixable critical CVE, privileged pod) blocks the PR.
3. CI has no kubeconfig or cluster token, verified by grepping secrets and workflow files.
4. A manual `oc scale`/`oc edit` in prod is reverted by Argo CD within the sync interval.
5. A canary with an injected error spike is aborted and rolled back automatically.
6. An unsigned image is denied at admission in every cluster.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
