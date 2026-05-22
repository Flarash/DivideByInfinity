# Terraform on Azure: layout, state, and the gotchas

> Opinionated patterns for running Terraform against Azure in a real team — repo
> layout, remote state, module boundaries, secrets, and the gotchas that bite
> once you have more than one environment and more than one engineer.

This is not "Terraform 101". It assumes you already know what `resource`, `module`
and `terraform plan` are. It's the layer above that — *how do you keep this thing
reviewable and replayable 18 months in*.

---

## TL;DR

- **One repo, multiple environment folders. Not workspaces.** Workspaces are
  shared state with a flag; folders are real isolation.
- **Remote state in an Azure Storage Account, locked via blob lease.** One
  storage account per *organization*, one container per *environment*, one
  state file per *stack*.
- **Modules earn their keep by being used 2+ times.** Don't pre-modularize.
- **Secrets never live in `.tfvars`.** They live in Key Vault, fetched via
  `data` blocks at plan time. The state file still ends up with the value —
  treat the state container as a secret.
- **Authenticate CI via federated credentials (Workload Identity), not SP
  secrets.** No more rotating client secrets.
- **`prevent_destroy` on anything stateful.** RG-level locks for production.
- **Run `terraform plan` on a schedule** and fail loudly on drift.

---

## Repo layout

```
infra/
├── modules/
│   ├── aks/
│   ├── postgres-flexible/
│   ├── storage-with-private-endpoint/
│   └── network-baseline/
├── stacks/
│   ├── platform-network/            # shared VNets, hub, firewall
│   │   ├── envs/
│   │   │   ├── dev.tfvars
│   │   │   ├── staging.tfvars
│   │   │   └── prod.tfvars
│   │   ├── main.tf
│   │   ├── providers.tf
│   │   ├── backend.tf
│   │   └── variables.tf
│   ├── platform-aks/
│   └── app-orders-api/
└── .github/workflows/
    ├── terraform-plan.yml           # on PR, against all stacks
    └── terraform-apply.yml          # on merge, env-gated
```

### Why folders not workspaces

`terraform workspace` keeps **all** environments in the same state file with a
key prefix. That sounds clever until:

- A bug in your code can apply prod changes when you thought you'd selected dev.
- Provider versions and required variables get coupled across environments —
  upgrading dev forces upgrading prod.
- State migrations apply to every workspace at once.

Folders give you the isolation that the word "environment" already implies.

### Stack = one state file = one blast radius

A *stack* is a directory you `terraform init && plan && apply` in. Each stack
gets:

- Its own state file (its own blob in the storage container)
- Its own backend config
- Its own variables and tfvars-per-env

Split stacks along **blast-radius lines**, not "service" lines:

- `platform-network` — VNets, subnets, firewall, DNS. Changes here are rare,
  reviewed by infra owners.
- `platform-aks` — AKS clusters, node pools, ingress controller, cert-manager.
- `app-<service>` — the per-service Azure resources (storage accounts, queues,
  databases). Owned by the service team.

Bad split: one giant stack covering everything. A typo blows up production
networking.

---

## Remote state

```hcl
# stacks/platform-aks/backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate-prod"
    storage_account_name = "sttfstateacme"
    container_name       = "prod"
    key                  = "platform-aks.tfstate"
    use_azuread_auth     = true
  }
}
```

### Rules

- **One storage account per organization.** Multiple containers, one per env.
- **`use_azuread_auth = true`** — AAD-based access, not storage account keys.
  Then assign `Storage Blob Data Contributor` to the human/CI principals that
  need access.
- **Versioning + soft delete on.** Recovering from `terraform destroy` typos
  has saved me more times than I want to admit.
- **Lifecycle policy: don't auto-delete old versions for at least 90 days.**

### Bootstrapping the backend (chicken-and-egg)

The storage account that holds state can't be managed by Terraform without a
state file. Two options:

1. **Click-ops once, then import.** Create the account by hand or with a tiny
   Bicep file, then import it into a `bootstrap` stack.
2. **`terraform init -backend=false` + `local` backend for bootstrap.** Then
   migrate state into itself after first apply. Feels weird, works.

Either way, **back up that storage account's contents to a different
subscription**. Losing it loses every state file you have.

---

## Modules: when to write one

Heuristic: write a module the **third time you copy-paste a block**. Before that
it's premature.

When you do write one, the contract is:

- **Inputs are typed and documented** (`variable "x" { type = ..., description = "..." }`).
- **Outputs are the minimum surface the consumer needs**, not "everything".
- **No `count = var.enabled ? 1 : 0` toggles.** If a caller needs the module
  off, don't call it. Toggles inside modules become magic.
- **No backend block.** The caller owns state, not the module.
- **Pin the module version** when consuming (`source = "git::...?ref=v1.2.0"`).
  Pulling `main` will eventually surprise you.

### `for_each` vs `count`

Rule: **use `for_each` whenever the inputs aren't a contiguous sequence**.
`count = 3` is fine for "give me three identical things". The moment items
get names — `["dev", "staging", "prod"]` — switch to `for_each`. Reordering
the input list with `count` re-creates everything; with `for_each` it doesn't.

---

## Secrets

The painful truth: **anything that ever goes into a Terraform variable, output,
or resource attribute ends up in the state file in plaintext.** Even `sensitive
= true` only hides it from the CLI output — the JSON has it.

### The pattern that works

1. Store the secret in **Azure Key Vault** outside Terraform (or in a different
   stack that only runs by-hand).
2. Reference it as `data` at plan time:

   ```hcl
   data "azurerm_key_vault_secret" "db_password" {
     name         = "orders-db-password"
     key_vault_id = data.azurerm_key_vault.shared.id
   }

   resource "azurerm_postgresql_flexible_server" "orders" {
     administrator_password = data.azurerm_key_vault_secret.db_password.value
     # ...
   }
   ```

3. Treat the **state container itself as a secret store** — restrict access,
   audit reads, alarm on bulk downloads.

### Don'ts

- Don't pass secrets via `-var "password=..."` in CI — they show up in process
  lists.
- Don't put them in `.tfvars` files in git.
- Don't put them in environment variables on shared runners (any concurrent
  job can read them).

---

## CI auth: Workload Identity, not SP secrets

Old way: create a Service Principal, store its `client_id`/`client_secret` as
CI secrets, rotate every 90 days, forget to rotate, scramble.

New way: **federated credentials** (OIDC). The CI pipeline presents a JWT,
Azure AD trusts the issuer (GitHub Actions, Azure DevOps, GitLab), no secret
exchanged.

```yaml
# GitHub Actions
- uses: azure/login@v2
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
    # no client-secret
```

The federated credential on the SP/Managed Identity has `subject` like
`repo:org/repo:environment:prod`. **Environment-scoped credentials** are the
unlock: dev/staging/prod can each have their own SP, and merging to main
doesn't give you prod automatically.

---

## Drift detection

Drift is when reality differs from `*.tf`. It happens because somebody
clicked something in the portal at 2am.

```yaml
# .github/workflows/terraform-drift.yml
on:
  schedule:
    - cron: "17 6 * * *"   # daily-ish, offset to avoid hourly spike

jobs:
  plan:
    strategy:
      matrix:
        stack: [platform-network, platform-aks, app-orders-api]
        env:   [dev, staging, prod]
    steps:
      - run: terraform plan -detailed-exitcode -var-file=envs/${{matrix.env}}.tfvars
        # exit code 2 = drift, opens an issue
```

`-detailed-exitcode` gives you:

- `0` — no changes
- `1` — error
- `2` — changes pending (= drift, since nobody applied)

Open an issue on exit-2, ping the stack owner. Don't auto-apply.

---

## The gotcha collection

### AKS node pools want to replace on small changes

Changing `vm_size`, `os_disk_type`, or `availability_zones` on the default
node pool **replaces the entire AKS cluster** because the default node pool is
part of `azurerm_kubernetes_cluster`. Workaround: keep the default node pool
minimal (just system pods), put real workloads on `azurerm_kubernetes_cluster_node_pool`
which can be replaced independently.

### Resource group locks block destroy

If prod has a `CanNotDelete` lock at the RG level, `terraform destroy` will
fail loudly. That's the point of the lock. To do legitimate destroys, remove
the lock by hand, run destroy, re-add the lock. Don't add the lock removal
into the destroy path — that defeats the purpose.

### `prevent_destroy` is half a seatbelt

```hcl
resource "azurerm_postgresql_flexible_server" "orders" {
  lifecycle {
    prevent_destroy = true
  }
}
```

This stops `terraform destroy` and `terraform apply` from destroying the
resource. It does **not** stop someone removing the resource block from `.tf`
and applying — then Terraform sees no resource and removes it from state, and
on the next apply the resource gets deleted because state shows it should
exist but it's gone. Combine with an Azure-side delete lock for real safety.

### `azurerm` provider version pinning

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.114"
    }
  }
}
```

The `~>` pin lets patches in. Major version bumps (3.x → 4.x) regularly
introduce breaking changes. Read the upgrade notes for each major.

### State file size

Above ~10MB state files, plans get slow. If you're past that, your stack is
too big — split it. Don't try to tune the backend.

### Sensitive outputs cross-stack

When stack A outputs a secret that stack B needs, the secret is in stack A's
state. Stack B reading it via `terraform_remote_state` puts it in stack B's
state too. **Prefer indirecting through Key Vault** — stack A writes the
secret to KV, stack B reads from KV. Now both states have only a KV reference.

---

## Pre-flight checklist

Before opening a Terraform PR:

- [ ] `terraform fmt -recursive` clean
- [ ] `terraform validate` passes in every changed stack
- [ ] `tflint` + a security scanner (`tfsec` or `checkov`) clean
- [ ] `terraform plan` output attached to the PR (auto-comment via CI)
- [ ] No secrets in `.tfvars` (use a pre-commit hook with `gitleaks`)
- [ ] Any new resource type is added to the docs / runbook
- [ ] Stateful resources have `prevent_destroy` set

---

## Related

- [Hybrid agent-workspace pattern](../agent-workspaces/hybrid-agent-workspace-pattern.md) — for repos where agents also touch infra
- [PR-as-publish-gate](../automation-patterns/pr-as-publish-gate.md) — same idea: humans approve, automation acts
- [Kubernetes (AKS) production checklist](./aks-production-checklist.md) — the layer above this
- [CI/CD pipeline patterns](./cicd-pipeline-patterns.md) — where `terraform apply` actually runs
