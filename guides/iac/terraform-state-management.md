# Terraform state management

**TL;DR:** State is the most important file in your Terraform setup. Treat it as
production data: remote, locked, backed up, encrypted, and never edited by hand
unless you have to — in which case, have a backup of the backup. Most "Terraform
broke prod" stories are state-management stories.

## What state actually is

Terraform state is the *map between configuration and reality*. It records:

- Every resource Terraform manages, by address (`azurerm_resource_group.this`).
- The cloud provider's identifier for each one (the actual resource ID).
- The full set of attributes as they were at the last apply.
- Dependencies between resources (the graph).

Without state, Terraform doesn't know what it manages. With *wrong* state,
Terraform makes wrong decisions — possibly destructive ones.

## Remote backends — the only acceptable answer for teams

`terraform.tfstate` on a laptop is fine for a tutorial. For anything else, use
a remote backend:

| Backend | Lock mechanism | Best for |
|---|---|---|
| **S3 + DynamoDB** | DynamoDB table | AWS-native shops |
| **Azure Storage** | Blob lease | Azure-native shops |
| **GCS** | Built-in object versioning + lock | GCP-native shops |
| **Terraform Cloud / Enterprise** | First-party lock service | Multi-cloud, want managed UX |
| **HTTP backend** | Custom | Self-hosted Atlantis, Spacelift, env0 |
| **Consul / etcd** | Native | Niche, mostly legacy |

What the backend gives you:

1. **Shared state** — your laptop and CI both see the same map.
2. **Locking** — only one apply at a time. No two engineers stomping each other.
3. **Versioning** — every apply produces a new version you can roll back to.
4. **Encryption at rest** — the cloud storage handles it.
5. **Access control** — IAM gates who can read/write state.

A minimal Azure backend:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "stfstateteamprod"
    container_name       = "tfstate"
    key                  = "payments/prod.tfstate"
    use_oidc             = true   # Workload Identity in CI, no creds in env
  }
}
```

The `key` is the state file path. Use a convention like
`<project>/<environment>.tfstate`. One key per environment per project.

## Locking — and what happens when it fails

State locking prevents two `terraform apply` runs from stepping on each other.
When it works, you don't notice. When it fails:

- **Stale lock** — a previous run crashed and didn't release the lock. Symptom:
  `Error acquiring the state lock`. Fix: `terraform force-unlock <lock-id>`.
  Only do this when you're *sure* no apply is actually running.
- **Lock contention** — humans + CI both trying to apply. Don't do this. CI is
  the single applier; humans only `plan`.
- **Lost lock** — backend storage outage drops the lock mid-apply. Rare;
  recovery is "wait for backend, re-acquire, continue or rollback".

The discipline: **CI is the only applier in production.** Humans run `plan`
locally to see what would change; merging the PR triggers CI to apply.

## Workspace strategies — the eternal debate

Three competing approaches to "multi-environment":

### 1. Folder per environment (recommended)

```
live/
  dev/main.tf       (backend key: project/dev.tfstate)
  staging/main.tf   (backend key: project/staging.tfstate)
  prod/main.tf      (backend key: project/prod.tfstate)
```

Each environment is a separate root module with its own state file. Different
values, different backends if you want, total isolation.

**Pros:** explicit, hard to mix up, environment-specific tweaks live where you
see them.

**Cons:** some duplication. Mitigate with shared modules.

### 2. Terraform workspaces (`terraform workspace`)

Same code, switch workspaces, each has its own state file under the same
backend.

**Pros:** less code duplication.

**Cons:** invisible state — you have to remember `terraform workspace select prod`.
Easy to apply to the wrong workspace. The Terraform docs *themselves* recommend
folders for environments. Workspaces are good for short-lived parallel state
(per-PR ephemeral envs), not for prod/staging/dev.

### 3. Branch per environment

Every environment is a branch. Merge dev → staging → prod.

**Pros:** GitOps fans love it.

**Cons:** divergence across branches becomes routine. Painful to keep in sync.
The thing you do most (deploy to prod) involves the most rebasing.

**Verdict:** folders for envs, workspaces for ephemeral, branches never (for IaC).

## State surgery — when you have to touch state directly

Sometimes the world drifts from state and `terraform plan` shows nonsense.
Time to operate on state.

### `terraform state list`
Lists every resource in state. Always run before any surgery.

### `terraform state show <addr>`
Inspect what state thinks a resource looks like.

### `terraform state mv <src> <dst>`
Rename a resource in state without destroying/recreating it. Critical when
refactoring modules or renaming resources.

```bash
terraform state mv 'azurerm_storage_account.foo' 'module.storage.azurerm_storage_account.this'
```

### `terraform state rm <addr>`
Remove a resource from state. **Does not delete the real resource.** Used when
you want Terraform to stop managing something. The resource still exists in
the cloud; Terraform will ignore it.

### `terraform import <addr> <cloud-id>`
The opposite of `rm`. Adopt an existing cloud resource into state. Required
when you create something manually and want Terraform to manage it going forward.

```bash
terraform import azurerm_storage_account.this /subscriptions/.../storageAccounts/mysa
```

### `moved` blocks (Terraform 1.1+)
Codify a `state mv` so the next apply does it automatically. Better than ad-hoc
`state mv` commands.

```hcl
moved {
  from = azurerm_storage_account.foo
  to   = module.storage.azurerm_storage_account.this
}
```

### Before any state surgery: **back up the state**.

```bash
# pull current remote state to a local file
terraform state pull > /tmp/state-backup-$(date +%s).json
```

If your surgery goes wrong, restore with `terraform state push`. (Carefully —
this overwrites remote state.)

## State drift — when reality and state disagree

Drift happens. Someone clicks in the console. A budget process auto-deletes
something. An emergency hotfix bypasses CI.

Detection:

- `terraform plan` shows changes you didn't expect = drift.
- Drift-detection workflows: run `terraform plan` on a schedule, alert on
  non-empty diff.
- Tools: Atlantis, Spacelift, env0 all build this in. Terraform Cloud has it.

Resolution:

1. **Investigate.** Why is the cloud different from state? Conversation, not
   blame.
2. **Decide direction.** Bring the cloud back to match the code (re-apply), or
   update the code to match the cloud (refactor + apply).
3. **Add prevention.** Policy-as-code, IAM restrictions, change-management
   process. (See [`policy-as-code-and-drift-detection.md`](./policy-as-code-and-drift-detection.md).)

## Recovering from corrupted state

State corruption is rare but catastrophic. Causes:

- Manual state edit gone wrong.
- Backend storage corruption.
- `terraform state push` of an older backup that's missing recent additions.

Recovery hierarchy:

1. **Roll back to a previous version** of the state file from the backend
   (S3 versioning, Azure Storage blob versioning, GCS versioning). All major
   backends support this — verify yours has it enabled.
2. **Re-import** if you have the cloud resource IDs. `terraform import` for
   each one. Painful but works.
3. **Delete and re-create** if the workload tolerates it (dev environment,
   stateless services). Don't do this for prod databases or anything with
   data.

The cheapest insurance:

- Enable versioning on the state backend bucket/container.
- Lifecycle policy: keep state-file versions for 90 days minimum.
- Periodic `terraform state pull > backup-$(date).json` to a separate bucket.

## Multi-team state ownership

When multiple teams share infrastructure, who owns what?

**Pattern: one team per state file.** Each team manages their own state. Cross-team
references use `terraform_remote_state` data source:

```hcl
data "terraform_remote_state" "network" {
  backend = "azurerm"
  config = {
    storage_account_name = "stfstateteamprod"
    container_name       = "tfstate"
    key                  = "network/prod.tfstate"
  }
}

resource "azurerm_subnet" "app" {
  virtual_network_name = data.terraform_remote_state.network.outputs.vnet_name
  # ...
}
```

The consuming team reads outputs from the producing team's state. The
producing team owns the contract (the outputs they expose).

**Anti-pattern:** every team in the same state file. Locks fight constantly,
blast radius is everything, change reviews involve everyone.

## Gotchas

- **State files contain secrets.** Anything Terraform read (even via `data`
  blocks) ends up serialized in state. Encrypt the bucket, restrict access,
  scan for leaked secrets in commits if state ever ends up in git.
- **Backends can't use variables.** `key = var.something` is invalid in
  `backend` blocks. They're evaluated too early. Use partial backend config +
  `-backend-config=` on init.
- **`terraform.tfstate.backup`** exists locally too. Don't commit it.
- **`terraform refresh` (now `terraform apply -refresh-only`)** can quietly
  reconcile state to match the cloud. This is *not* the same as drift
  resolution — it just updates state to match reality without changing the
  cloud. Sometimes this hides real drift.
- **Provider upgrades change state schema.** Major-version provider bumps can
  require state migrations. Read changelogs. Test in non-prod.
- **`terraform destroy` with the wrong workspace selected** is how prod gets
  taken out. CI-only applies for prod. Local destroy in prod environments
  should be impossible (IAM-blocked).

## When manual state edits are acceptable

Almost never. Exceptions:

- Recovering from a known-bad import.
- After a backend migration where automatic state isn't recoverable.
- Removing a resource Terraform can't physically destroy (deleted out-of-band).

In every case: back up first. Inspect with `state show`. Document what you did
in a runbook. Don't do this casually.

## Related

- [`terraform-vs-pulumi-vs-cloudformation.md`](./terraform-vs-pulumi-vs-cloudformation.md)
  — how state semantics differ across IaC tools.
- [`terraform-module-design.md`](./terraform-module-design.md) — module
  composition and the `moved` block for state-safe refactors.
- [`policy-as-code-and-drift-detection.md`](./policy-as-code-and-drift-detection.md)
  — preventing drift from happening, and detecting it when it does.
- [`../cloud-platform/terraform-on-azure.md`](../cloud-platform/terraform-on-azure.md)
  — Azure Storage backend specifics.
