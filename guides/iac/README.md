# Infrastructure as Code (IaC)

Patterns and trade-offs for managing cloud infrastructure with Terraform, Pulumi,
or CloudFormation. Less "click in the portal", more "version your infrastructure
the way you version your code".

## Guides

- [**Terraform vs Pulumi vs CloudFormation**](./terraform-vs-pulumi-vs-cloudformation.md)
  — honest comparison across coverage, state, testing, ergonomics, and lock-in.
  Includes the "which one for which team" decision matrix and the CDK conversation.
- [**Terraform module design**](./terraform-module-design.md) — root vs reusable
  modules, the thin-wrapper anti-pattern, input/output discipline, semver for
  modules, registry vs in-repo, and testing strategies (validate / tflint /
  Terratest / `terraform test`).
- [**Terraform state management**](./terraform-state-management.md) — remote
  backends (S3+DynamoDB, Azure Storage, GCS, TFC), state locking, the workspace-vs-folder
  debate, state surgery (`mv`/`rm`/`import`/`moved`), drift, and recovering from
  corruption.
- [**Policy as code and drift detection**](./policy-as-code-and-drift-detection.md)
  — Checkov / tfsec / OPA / Sentinel comparison, where policies run in the pipeline,
  a concrete Checkov + Conftest setup, scheduled drift detection, and the exception
  process that doesn't decay.
