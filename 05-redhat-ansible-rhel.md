# PROMPT — Ansible + RHEL Configuration Management CI/CD (GitHub Actions → Ansible Automation Platform)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `CONTENT` | Ansible **collections** (roles + modules + plugins) and playbooks, in separate repos |
| `TARGETS` | RHEL 9 (primary), RHEL 8 (supported), test images: UBI init / RHEL via Podman |
| `EXECUTION` | Ansible Automation Platform (AAP) 2.5: Controller job templates + **Execution Environments** built with `ansible-builder` |
| `HARDENING` | DISA STIG / CIS Level 2 via OpenSCAP (`scap-security-guide`) |
| `SECRETS` | HashiCorp Vault / AWS Secrets Manager lookups. **ansible-vault only for bootstrap** |

---

## OBJECTIVE
Configuration changes are **linted, idempotent, molecule-tested on every supported RHEL version, compliance-scanned, packaged as versioned collections and EEs**, and rolled out in controlled batches through AAP, with drift detected continuously.

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
- CI talks to AAP with an OAuth2 token scoped to an **execute-only** team on the specific job templates. Use a separate token per environment, stored in GitHub Environment secrets.
- No plaintext secrets in inventory/group_vars. `no_log: true` is enforced on secret-handling tasks by a custom ansible-lint rule.

---

## PIPELINE — PR / merge_group
| Stage | Tools | Fail condition |
|---|---|---|
| YAML | yamllint (strict) | any error |
| Lint | `ansible-lint --profile production` (FQCN, no `command` where a module exists, `changed_when` set, no `latest`) | any violation |
| Collection sanity | `ansible-test sanity --docker`, `ansible-test units` | any failure |
| Molecule | scenarios on a Podman driver × matrix {RHEL 8, RHEL 9 UBI-init}: converge → **idempotence (second run = 0 changed)** → verify (testinfra/goss) → destroy | any failure or non-idempotent task |
| Compliance | OpenSCAP `oscap xccdf eval` against the STIG profile on the molecule instance | score < threshold or new failed rules vs baseline |
| Secrets | gitleaks + `ansible-lint` no-plaintext-password rule | any finding |
| EE build | `ansible-builder build` (pinned base `ee-minimal-rhel9@sha256`), Trivy scan, sign | high/critical fixable CVE |
| Dry run | AAP job template launch in **check + diff mode** against the staging inventory; diff posted to the PR | job failure |

---

## CD — merge to `main`
1. release-please tags the collection semver. Publish to **Private Automation Hub**. Push the EE image (signed) to Hub's registry.
2. Update the AAP project/EE through the `ansible.controller` collection (config-as-code: job templates, credentials, inventories, RBAC are all declared in Git).
3. **Rollout:** dev → staging → prod workflow templates.
   - Prod requires GitHub Environment approval.
   - Plays use `serial: [1, "10%", "25%", "100%"]` and `max_fail_percentage: 0`. Each batch runs a health check `assert`.
4. `block/rescue` restores config backups (`backup: yes`) on failure, and the run fails loudly.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (Ansible-specific)
- **Scheduled drift runs** (AAP schedule every 4 h) in **check mode**. A non-zero `changed` count raises an alert.
  - In dev/staging, drift is auto-remediated with a real run.
  - In prod, the job opens a ticket. An AAP workflow can remediate after approval.
- A nightly OpenSCAP scan of the fleet reports to AAP/Insights. A new failing rule opens an issue with the remediation role tag.
- Red Hat Insights advisories trigger remediation playbooks for critical CVEs.

---

## DELIVERABLES
```
platform-workflows/.github/workflows/{ansible-ci,collection-release,ee-build,aap-deploy}.yml
collection/<ns>/<name>/{galaxy.yml,roles/*/molecule/default/{molecule.yml,converge.yml,verify.yml},plugins/,tests/}
playbooks/{site.yml,inventories/{dev,staging,prod}/,group_vars/}
execution-environment/{execution-environment.yml,requirements.yml,bindep.txt}
aap-config/{job_templates.yml,workflows.yml,schedules.yml,rbac.yml}
.ansible-lint, .yamllint, .gitleaks.toml
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-rollback.md,RUNBOOK-drift.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. A task using `shell` where a module exists, or a non-FQCN module, is blocked by ansible-lint.
2. A non-idempotent role fails the molecule idempotence step on both RHEL 8 and RHEL 9.
3. A role that regresses the STIG score is blocked.
4. A prod rollout stops at the first failed batch and leaves the remaining hosts untouched.
5. A manual config change on a staging host is detected and reverted within 4 h.
6. The EE image is signed and pinned by digest in the AAP job templates.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
