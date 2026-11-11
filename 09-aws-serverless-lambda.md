# PROMPT — AWS Serverless CI/CD — Lambda, API Gateway, EventBridge with AWS CDK (GitHub Actions)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `IAC` | **AWS CDK v2 (Python)** + `cdk-nag` (AwsSolutions + HIPAA packs). SAM is acceptable if already in use |
| `RUNTIME` | Python 3.12 on arm64 (Graviton), AWS Lambda Powertools (logging, tracing, metrics, idempotency) |
| `ACCOUNTS` | `dev`, `staging`, `prod` + per-PR ephemeral stacks in `dev` |
| `DELIVERY` | Lambda aliases + CodeDeploy `Canary10Percent5Minutes` with CloudWatch alarms |

---

## OBJECTIVE
Serverless changes are **unit-tested with mocks, integration-tested in a real per-PR ephemeral stack, nag-compliant, code-signed**, and shifted to prod with automatic alarm-based rollback.

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

---

## PIPELINE — PR / merge_group
| Stage | Tools | Fail condition |
|---|---|---|
| Quality | ruff, mypy --strict, pytest + moto, coverage 85% / diff 90% | any failure |
| SAST / secrets / SCA | CodeQL, Semgrep, gitleaks, pip-audit, dependency-review | high/critical, or any secret |
| IaC | `cdk synth` → **cdk-nag** (errors fail, suppressions need a reason), Checkov on `cdk.out`, Conftest (least-privilege IAM: no `*` actions, reserved concurrency set, DLQ/on-failure destination required, X-Ray on, log retention set) | any error |
| Ephemeral stack | `cdk deploy <app>-pr-<n>` in dev → integration tests against real API GW/SQS/DynamoDB → stack destroyed on PR close/merge (`pull_request: closed`) | any failure |
| Contract | schemathesis against the deployed API | any failure |
| Diff | `cdk diff` per env, posted as a PR comment | n/a |

---

## CD — merge to `main`
1. Package the artifacts once. **Sign them with AWS Signer.** Lambda code-signing config is set to `Enforce`.
2. `cdk deploy` through **CDK Pipelines or GitHub reusable deploy**: dev → staging → prod. Prod requires approval.
3. Each function publishes a version, then an alias shift through CodeDeploy canary with pre/post traffic hooks (smoke Lambda). CloudWatch alarms (Errors, Throttles, p99 Duration, DLQ depth) trigger **automatic rollback**.
4. Staging runs k6 load tests (verify concurrency/throttling) and ZAP against API Gateway.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (Serverless-specific)
- CodeDeploy alarm rollback with no human in the loop. DLQ redrive runbook automation (Step Functions) after the fix deploys.
- Powertools idempotency stops retries from causing duplicate side effects.
- A scheduled `cdk diff` against deployed stacks detects drift. CloudFormation drift detection runs on the stateful resources.
- An orphaned ephemeral-stack janitor runs nightly and deletes `*-pr-*` stacks older than 48 h.

---

## DELIVERABLES
```
platform-workflows/.github/workflows/{sls-ci,sls-ephemeral,sls-deploy,sls-janitor}.yml
app/{app.py,stacks/,functions/<fn>/{handler.py,requirements.txt},tests/{unit,integration}/}
cdk.json, policy/conftest/*.rego, nag-suppressions.yaml
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-rollback.md,RUNBOOK-dlq-redrive.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. A Lambda with a `*` IAM action, no DLQ, or no log retention is blocked by cdk-nag/Conftest.
2. Each PR gets a live ephemeral stack, the integration tests pass against it, and it's destroyed afterwards.
3. Unsigned code is rejected by Lambda code signing.
4. An injected error rate during the canary rolls back the alias automatically.
5. There are no long-lived AWS keys, and the deploy role is scoped per environment.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
