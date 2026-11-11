# PROMPT — Kubernetes CI/CD with GitOps — Helm/Kustomize, Argo CD, Argo Rollouts, Kyverno (Cloud-Agnostic)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `CLUSTERS` | EKS / AKS / GKE / on-prem; `dev`, `staging`, `prod` (prod multi-region optional) |
| `PACKAGING` | Helm chart per service (published as OCI) + Kustomize env overlays in a `config` repo |
| `GITOPS` | Argo CD (ApplicationSets, sync waves, `selfHeal` + `prune`) |
| `DELIVERY` | Argo Rollouts canary + AnalysisTemplates (Prometheus) |
| `POLICY` | Kyverno (admission + `verifyImages`) |
| `SECRETS` | External Secrets Operator → the cloud secret manager |

---

## OBJECTIVE
Every manifest change is **schema-validated, policy-tested, and proven on an ephemeral cluster** before merge. Clusters converge only from Git. Releases roll out progressively and revert themselves on bad metrics.

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
- CI never holds cluster credentials. Argo CD pulls. Environment promotion is a Git PR that changes an image **digest**.
- Argo CD's own RBAC: SSO only, no local admin, and prod projects restrict source repos, destinations and cluster-scoped resources.

---

## PIPELINE — PR / merge_group (app repo + config repo)
| Stage | Tools | Fail condition |
|---|---|---|
| Chart lint | `helm lint --strict`, `ct lint` (chart-testing, version bump enforced) | any error |
| Chart unit tests | helm-unittest | any failure |
| Render + schema | `helm template` / `kustomize build` → kubeconform (strict, CRD schemas from the datree catalog) | any invalid |
| Best practice | kube-linter, Polaris (score ≥ 90), kube-score | any error |
| Deprecated APIs | pluto against the target k8s versions | any removed API |
| Policy tests | `kyverno test` / Conftest on rendered manifests (limits, probes, non-root, no `latest`, PDB required, topologySpread for prod) | any deny |
| Ephemeral e2e | **kind** cluster in CI with Argo CD + Rollouts + Kyverno installed. `ct install`, then the chainsaw/kuttl test suite | any failure |
| Diff | `argocd app diff` (read-only token) or `kubectl diff` against staging, posted as a PR comment | n/a |

---

## CD — GitOps promotion
1. App merge → image built, signed and attested (see the Docker prompt) → bot opens a PR to `config/overlays/dev` with the new digest, which auto-merges on green.
2. Argo CD sync waves run in order: CRDs/namespaces (−2) → secrets (−1) → app (0) → post-sync smoke Job (hook).
3. Promotion dev → staging → prod is by PR. Staging auto-merges after e2e, k6 and ZAP pass. Prod needs CODEOWNER approval.
4. The Rollout canary goes 10/25/50/100 with pauses. The AnalysisTemplate requires 5xx < 1%, p99 < SLO and no pod restarts. A failure aborts the rollout and scales the canary to zero automatically.
5. Kyverno enforces in-cluster: `verifyImages` (cosign keyless, org identity), restricted Pod Security, and default-deny NetworkPolicy generation per namespace.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (K8s-specific)
- Argo CD `selfHeal: true, prune: true` reverts any `kubectl` drift. A `ServerSideApply` + `RespectIgnoreDifferences` config avoids flapping.
- Argo Rollouts aborts on analysis failure. HPA + PDB keep capacity during rollback.
- Argo CD Notifications sends OutOfSync/Degraded alerts to Slack, with a link to the runbook.
- A Renovate job tracks chart dependencies, base images and Kubernetes API versions (pluto in a scheduled job).

---

## DELIVERABLES
```
platform-workflows/.github/workflows/{k8s-manifest-ci,chart-release,gitops-promote,ephemeral-e2e}.yml
charts/<service>/{Chart.yaml,values.yaml,values.schema.json,templates/,tests/}
config-repo/{base/,overlays/{dev,staging,prod}/,argocd/{appproject.yaml,applicationset.yaml},rollouts/analysis.yaml}
policy/kyverno/{verify-images,require-limits,restrict-pss,generate-netpol}.yaml + tests/
e2e/chainsaw/*.yaml
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-rollback.md,RUNBOOK-argocd-outofsync.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. A manifest with no limits, no probes, `:latest`, or a removed API version is blocked by a named check.
2. The chart installs and passes e2e on an ephemeral kind cluster in CI.
3. CI holds no kubeconfig, and prod changes happen only through a merged config-repo PR.
4. A manual `kubectl edit` in prod is reverted automatically.
5. An injected error spike aborts the canary and returns traffic to stable automatically.
6. An unsigned image is rejected by Kyverno in every cluster.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
