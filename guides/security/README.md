# Security

Cloud security patterns: IAM modeling, secrets handling, network defense, and
threat modeling. Practical patterns for cloud workloads, not compliance theater.

## Guides

- [**IAM least-privilege in practice**](./iam-least-privilege-in-practice.md) —
  why wildcards are the canonical sin, RBAC vs ABAC, modeling across AWS / Azure /
  GCP, workload identity for services, role assumption chains, the quarterly audit
  cadence, and break-glass account discipline.
- [**Secrets management for cloud workloads**](./secrets-management-for-cloud-workloads.md)
  — Azure Key Vault / AWS Secrets Manager / GCP Secret Manager / Vault comparison,
  workload identity as the gateway, the Secrets Store CSI driver pattern for
  Kubernetes, rotation patterns, dynamic secrets (Vault), and why env vars are
  last decade.
- [**Network security layers**](./network-security-layers.md) — VPC/VNet topology,
  subnet segmentation, security groups vs NACLs, private endpoints, WAF, DDoS,
  egress filtering, east-west controls, and modern bastion alternatives (SSM
  Session Manager, Azure Bastion, IAP).
- [**Threat modeling without the bureaucracy**](./threat-modeling-without-the-bureaucracy.md)
  — the four-question framework, STRIDE as a lens, the 30-minute team exercise,
  documenting the result, the specific patterns that surface in 80% of models,
  and integrating with the SDLC.
