# PROMPT — Golden Image Pipeline — Packer + Ansible, CIS/STIG-Hardened RHEL AMIs (GitHub Actions)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `BASE` | Official RHEL 9 AMI (owner `309956199498`), resolved by filter, never a hard-coded ID |
| `BUILDER` | Packer HCL2 `amazon-ebs` (add `azure-arm` / `googlecompute` sources for multi-cloud) |
| `PROVISIONER` | Ansible (reuse the RHEL hardening collection from the Ansible prompt) |
| `HARDENING` | CIS Level 2 / DISA STIG via OpenSCAP |
| `REGISTRY` | AMIs shared to org accounts. Latest AMI ID published to an SSM Parameter per channel (`/golden/rhel9/{dev,prod}`). Optional HCP Packer |

---

## OBJECTIVE
Produce **hardened, scanned, tested, encrypted, versioned machine images** on a schedule and on change, promote them through channels, and roll fleets onto them with automatic rollback.

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
- The build runs in an isolated **image-builder account/VPC** with no inbound access and egress through a proxy allowlist. Temporary instances use SSM Session Manager (no SSH keys or port 22).
- AMIs are encrypted with an org KMS key, with no public launch permissions. IMDSv2 is required.

---

## PIPELINE — PR / merge_group / schedule (weekly + on base-AMI release)
| Stage | Tools | Fail condition |
|---|---|---|
| Template | `packer fmt -check`, `packer validate`, Checkov (packer) | any error |
| Provisioning code | ansible-lint (production profile), molecule (see the Ansible prompt) | any failure |
| Build | `packer build` (temporary instance, SSM communicator) | failure |
| Image tests | goss / InSpec: services, users, sshd config, auditd, FIPS mode if required, no default creds | any failure |
| Compliance | OpenSCAP STIG/CIS eval on the build instance, HTML+ARF report artifact | score < baseline or new failed rules |
| Vulnerability | **Amazon Inspector v2** scan of the AMI (or Trivy rootfs) | fixable critical/high |
| Metadata | tags: `git_sha`, `base_ami`, `build_date`, `cis_score`, `channel`; SBOM (Syft on rootfs) stored in S3 | missing |

---

## PROMOTION / ROLLOUT
1. A successful build is published to the **`dev` channel** (SSM param). Consumer ASGs/launch templates in dev reference `resolve:ssm:/golden/rhel9/dev`.
2. After 7 days in dev with no incidents and the scans still clean, the pipeline opens a promotion PR. Approval updates the `prod` channel.
3. Fleets roll through **EC2 Auto Scaling instance refresh** with `MinHealthyPercentage: 90`, checkpoints, and **auto rollback** on CloudWatch alarms.
4. Lifecycle: AMIs older than N versions are deprecated (`DeprecationTime`). After 90 days they're deregistered and their snapshots deleted, unless still in use.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (Image-specific)
- **Scheduled weekly rebuild** plus an EventBridge trigger on a new RHEL base AMI give an automatic patched image.
- Inspector continuous scanning of running instances: a new critical CVE triggers a rebuild, then promotion.
- Instance refresh auto rollback on failed health checks. A drift check (SSM State Manager / AWS Config) flags instances not on the current channel AMI.

---

## DELIVERABLES
```
platform-workflows/.github/workflows/{image-ci,image-build,image-promote,image-lifecycle}.yml
packer/{rhel9.pkr.hcl,variables.pkr.hcl,sources.pkr.hcl}
ansible/{playbook.yml,requirements.yml}
tests/{goss.yaml,inspec/}
infra/{image-builder-account.tf,ssm-channels.tf,inspector.tf,kms.tf}
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-image-rollback.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. A build with a failed CIS control or a fixable critical CVE is never published.
2. AMIs are encrypted, private, IMDSv2-only, and tagged with git SHA and compliance score.
3. A new base RHEL AMI triggers a rebuild automatically.
4. A bad image during instance refresh rolls back automatically.
5. No SSH keys or port 22 are used anywhere in the build.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
