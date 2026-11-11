# PROMPT — Docker Image CI/CD & Secure Container Supply Chain (GitHub Actions)

---

## ROLE
Act as a Principal DevOps/Platform Engineer (10+ yrs) who has shipped regulated production systems on this platform. Make the decisions. Don't hand me options. Where you have to assume something, write it under `ASSUMPTIONS.md` and keep going. Output complete, working files, not snippets, and no TODOs or placeholders left in code.

---

## CONTEXT (fill in, or use the defaults)
| Variable | Default |
|---|---|
| `IMAGE_NAME` | `ghcr.io/<org>/<service>` (mirrored to ECR/ACR/Artifact Registry if needed) |
| `BASE_IMAGE` | Chainguard / distroless, **pinned by digest**. Use UBI-micro if Red Hat support is required |
| `PLATFORMS` | `linux/amd64,linux/arm64` |
| `LANGUAGE` | Python 3.12. Adapt for Go/Node/Java |

---

## OBJECTIVE
Every image that ships is **minimal, non-root, reproducible, scanned, signed, attested, and traceable to a commit**. Nothing unscanned or unsigned can be pulled into any environment, and base-image CVEs are patched automatically.

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

## PIPELINE — PR / push / merge_group
| Stage | Tools | Fail condition |
|---|---|---|
| Dockerfile lint | hadolint (strict), Trivy config | any warning or above |
| Build | Docker Buildx + BuildKit, multi-stage, multi-arch, `--provenance=mode=max --sbom=true`, GHA cache (`type=gha,mode=max`), `SOURCE_DATE_EPOCH` for reproducibility | build failure |
| Image best practice | dockle (CIS Docker), container-structure-test (file perms, user ≠ 0, entrypoint, no shell if distroless) | any fatal/warn |
| Vulnerability scan | Trivy image + Grype (two engines, so one engine's blind spots are covered) | critical/high with a fix available |
| Secrets in layers | Trivy `--scanners secret`, gitleaks on the build context | any finding |
| SBOM | Syft, SPDX + CycloneDX, attached as an OCI attestation | n/a |
| License | Trivy license scan | any denied license (GPL/AGPL in a distributed image) |
| Size / regression | `dive --ci` (efficiency ≥ 95%), image size budget vs `main` (+10% max) | budget breached |
| Runtime smoke | `docker run` with `--read-only --cap-drop=ALL --security-opt=no-new-privileges --user 65532`, then hit the health endpoint | non-200 / crash |

Dockerfile rules to enforce:
- `FROM` pinned by `@sha256`.
- `USER` non-root, set numerically.
- Has a `HEALTHCHECK`.
- No `ADD` from remote URLs.
- No secrets in `ARG`/`ENV`; use `RUN --mount=type=secret`.
- OCI labels: `org.opencontainers.image.{source,revision,created,version,licenses}`.
- A `.dockerignore` is required.

---

## RELEASE — merge to `main`
1. **Don't rebuild.** Promote the merge-queue artifact by **digest**.
   - Tags: `sha-<gitsha>` plus semver from release-please.
   - `latest` is never used outside dev.
2. Sign keyless with **cosign** (Sigstore OIDC). Attach `actions/attest-build-provenance` (SLSA L3) and the SBOM attestation.
3. Push to every registry: GHCR is the primary, mirrored to the cloud registry. Registries enforce immutable tags and scan-on-push.
4. Consumers verify before running:
   - In CI: `cosign verify --certificate-identity-regexp '^https://github.com/<org>/' --certificate-oidc-issuer https://token.actions.githubusercontent.com`.
   - In clusters: a Kyverno/Gatekeeper `verifyImages` policy.

---

## SELF-CORRECTION (common to every platform)
- Retry **network-bound steps only** (pulls, installs, pushes), up to 3× with backoff. Tests are never retried to hide flakiness.
- **Flaky-test quarantine:** a nightly rerun finds flaky tests and opens an issue for CODEOWNERS. A quarantined test is non-blocking for 7 days max, then blocks again.
- **Renovate:** pins by digest. Patch/minor updates auto-merge only after `ci-gate` passes in the merge queue. Majors need a human.
- **Expired-exception sweeper:** a scheduled job fails and opens an issue when a security-scan exception has passed its expiry date.
- **Failure summary:** on failure, post the failed step, the last 50 log lines, and a link to the PR and to Slack/Teams.
## SELF-CORRECTION (Docker-specific)
- **Weekly scheduled rebuild** picks up patched base layers. If the scan improves, the pipeline opens a PR and auto-merges it on green.
- Renovate tracks base-image digests (`pinDigests`) and Dockerfile `ARG` versions.
- A nightly **re-scan of every deployed digest** catches newly disclosed CVEs and opens an issue with the affected image and fix version.

---

## DELIVERABLES
```
platform-workflows/.github/workflows/{docker-ci,docker-release,docker-rescan,docker-rebuild}.yml
platform-workflows/.github/actions/{buildx-setup,scan-image,sign-attest,verify-image}/action.yml
example-service/{Dockerfile,.dockerignore,container-structure-test.yaml,.hadolint.yaml,.trivyignore,renovate.json}
policy/kyverno/verify-images.yaml
docs/{ARCHITECTURE.md,ONBOARDING.md,RUNBOOK-cve-response.md}
```

---

## ACCEPTANCE CRITERIA (verify each)
1. A Dockerfile that runs as root, has an unpinned base, or puts a secret in `ENV` is blocked by a named check.
2. An image with a fixable critical CVE is blocked, and both scanners report it.
3. The released digest equals the merge-queue digest. `cosign verify` and `gh attestation verify` both pass.
4. An unsigned image is rejected by the cluster admission policy.
5. Images build for amd64 + arm64. A cached rebuild with no changes takes ≤ 2 min.
6. Smoke-run passes with a read-only root filesystem, all capabilities dropped, and a non-root UID.

---

## OUTPUT FORMAT
1. Write `ASSUMPTIONS.md` first.
2. Write every deliverable file, complete. Include a `Makefile` with `make ci-local` that mirrors CI, and a `.pre-commit-config.yaml` that mirrors the CI linters.
3. End with a validation report: each acceptance criterion, the command used to verify it and its output, then PASS/FAIL. If something can't be verified locally, state exactly what to run in the real environment.
