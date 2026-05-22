# Cloud platform guides

Patterns and gotchas for running cloud infrastructure at the platform level —
the layer below the apps. Azure-flavoured but most patterns port to AWS/GCP.

## Index

- [**Terraform on Azure — layout, state, and the gotchas**](./terraform-on-azure.md) — repo structure (folders not workspaces), remote state in Azure Storage, module rules, secrets via Key Vault, Workload Identity for CI auth, drift detection, the gotcha collection (AKS node pool replaces, RG locks, `prevent_destroy` limits, cross-stack secrets).
- [**Kubernetes (AKS) production checklist**](./aks-production-checklist.md) — multi-zone topology, system/user node pool split, Azure CNI Overlay vs flat (and the subnet-IP-exhaustion trap), NGINX + cert-manager, Key Vault CSI driver, AAD-integrated RBAC, PDBs + autoscaler interactions, HA/DR via Velero + GitOps, the first-hour checklist.
- [**CI/CD pipeline patterns — Azure DevOps, GitLab CI, Jenkins**](./cicd-pipeline-patterns.md) — the universal pipeline shape, what each platform does well and badly, per-platform gotchas (variable scoping, runner choices, approvals, plugin churn), shared patterns (build-once / promote, per-PR ephemeral envs, change-only deploys), and how to pick one for a new org.
- [**Observability stack: Prometheus + Grafana + ELK**](./observability-stack.md) — the three pillars (metrics/logs/traces), Prometheus mental model + label cardinality (the killer), ELK vs Loki trade-off, dashboards-as-code in Grafana, alert quality rules, where Splunk fits, and the always-forgotten DR for the observability stack itself.

## Conventions

- These four guides are written to chain — Terraform provisions the cluster,
  the cluster runs apps shipped by the pipelines, observability watches all
  of it. Each cross-references the others where the seam is real.
- Opinionated defaults are called out as opinions (TL;DR sections at the top
  of each).
- Azure-specific paths are flagged; the underlying patterns aren't.
