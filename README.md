# CI/CD Prompt Library — Enterprise, Production-Ready

Paste-ready prompts for an AI coding agent (Claude Code, etc.). Each one is standalone and follows the same bar:
- Zero failing checks before merge.
- Federated, keyless auth.
- Signed, attested artifacts promoted by digest.
- Progressive delivery with automatic rollback.
- Reusable workflows.
- Acceptance criteria the agent must prove.

| # | File | Use it for |
|---|---|---|
| 01 | `01-aws-github-actions.md` | App CI/CD to AWS (EKS/ECS) end-to-end |
| 02 | `02-docker-container-supply-chain.md` | Image build, scan, sign, SBOM, rebuild automation |
| 03 | `03-terraform-iac.md` | Terraform modules + multi-env plan/apply/drift |
| 04 | `04-redhat-openshift.md` | OpenShift with UBI, ACS (roxctl), OpenShift GitOps |
| 05 | `05-redhat-ansible-rhel.md` | Ansible collections, Molecule, STIG, AAP rollouts |
| 06 | `06-kubernetes-gitops-argocd.md` | Any K8s cluster: Helm/Kustomize, Argo CD/Rollouts, Kyverno |
| 07 | `07-azure.md` | Azure Container Apps/AKS, ACR, Key Vault, Azure Policy |
| 08 | `08-gcp.md` | Cloud Run/GKE, Binary Authorization, Cloud Deploy |
| 09 | `09-aws-serverless-lambda.md` | Lambda/API GW with CDK, ephemeral PR stacks, code signing |
| 10 | `10-packer-golden-images.md` | Hardened RHEL AMIs, Inspector, instance-refresh rollout |
| 11 | `11-gitlab-ci-aws.md` | Same standard on GitLab (components, merge trains, OIDC) |

**Combining prompts:** for a service on EKS, run **02 → 03 → 06** (or 01 alone for a single-shot build). Each file's CONTEXT table holds the defaults to edit before pasting.
