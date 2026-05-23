# Terraform module design

**TL;DR:** Modules are the unit of reuse and also the unit of pain. Bad modules
proliferate, drift in subtle ways, and become impossible to upgrade. Good modules
do one thing, expose few inputs, have explicit outputs, and are versioned. The
"thin-wrapper module" is almost always a mistake. The "root module is the only
module" is almost always also a mistake.

## What a module is for

A Terraform module is *a packaged set of resources with inputs and outputs*. Use
modules when:

- Multiple environments / stacks need the same shape.
- The resource group has non-trivial defaults you want to standardize (tagging,
  naming, security baseline).
- You want versioned upgrades (bump module v1.2 → v1.3 across consumers).

Don't use modules to:

- Wrap a single `azurerm_resource_group` resource (the thin-wrapper trap).
- Hide complexity that consumers genuinely need to see.
- Create your own meta-DSL (HCL is already a DSL; adding another layer is
  usually regret-driving).

## Root vs reusable modules

| Type | Where it lives | Purpose | Versioning |
|---|---|---|---|
| **Root module** | `live/<env>/` folder | Wires reusable modules together for a specific environment | Not versioned; just code |
| **Reusable module** | `modules/<name>/` folder or separate repo | Encapsulates a unit of infrastructure | Semver, tagged |

Two patterns of organization:

- **Mono-repo:** `modules/` and `live/` in the same repo. Easy to refactor across
  modules and consumers atomically. Hard to enforce versioned consumption.
- **Multi-repo:** modules in their own repos with semver tags. Consumers pin
  versions. Slower iteration; safer upgrade flow.

Default to mono-repo for small teams; split to multi-repo when modules
stabilize and consumer count grows.

## The thin-wrapper anti-pattern

Wrapping one resource in a module to "abstract" it.

```hcl
# BAD: this module adds nothing.
module "my_rg" {
  source = "./modules/resource-group"
  name   = "rg-payments-prod"
  location = "westeurope"
}

# modules/resource-group/main.tf
resource "azurerm_resource_group" "this" {
  name     = var.name
  location = var.location
}
```

The wrapper doesn't centralize policy, doesn't add safety, doesn't reduce
boilerplate. It hides the type of resource being created (you have to grep to
know it's a resource group) and adds indirection for nothing.

The fix: write the resource directly. Modules are for composition, not for
renaming.

A *non*-thin wrapper would centralize policy:

```hcl
# GOOD: encapsulates the team's standard for resource groups.
module "rg" {
  source   = "./modules/standard-rg"
  name_base = "payments"
  env       = "prod"
  # module enforces: location whitelist, mandatory tags, naming convention,
  # lock against accidental deletion, diagnostic settings.
}
```

If a module reduces to "rename `name` and pass it through", delete it.

## Input/output discipline

The biggest pain in shared modules comes from inputs you can never remove and
outputs that the world ends up depending on.

### Inputs — the rules

- **Minimum viable.** Every input is a maintenance burden. Add only what
  consumers truly need to vary.
- **Types and validation.** Use `type = string`, `type = list(string)`, etc.
  Add `validation` blocks for invariants ("region must be in approved list").
- **Defaults for sane choices.** Make the easy path the right path.
- **Required inputs have no default.** "Pass in a name; we won't guess one."
- **No magic values in defaults.** If `default = "westeurope"`, document why
  in the variable description.
- **Don't accept "config" objects you don't read.** If a module accepts a giant
  map and only uses 3 keys, the contract is wrong.

### Outputs — the rules

- **Output only what consumers consume.** Every output is a contract you can't
  remove without breaking consumers.
- **Output IDs, not nested resource objects.** `output "id"` is stable;
  `output "resource"` exposes implementation.
- **Use `description` on every output.** "Resource group name" is not a
  description. "Resource group name; used by consumers to scope role
  assignments" is.

### Versioning the contract

When you add an input with a default, it's backward-compatible. When you
remove or rename an input, it's a major version bump. Use semver:

- **Patch (1.0.0 → 1.0.1):** Bug fixes that don't change behavior.
- **Minor (1.0.0 → 1.1.0):** New optional inputs / outputs. No breaking changes.
- **Major (1.0.0 → 2.0.0):** Removed inputs, renamed outputs, behavioral
  changes. Consumers must update.

If you don't version modules, every change ripples to every consumer. Versioning
is the only way to ship without coordination.

## Module composition

Modules call other modules. The hierarchy that works:

```
live/prod/main.tf        (root, environment-specific wiring)
  → module "app"          (reusable: full app stack)
      → module "api"      (reusable: a single service)
        → module "rg"     (reusable: standard resource group)
        → module "kv"     (reusable: standard Key Vault)
```

Rules:

- **Max 3 levels deep.** Beyond that, debugging plans becomes painful.
- **Root modules don't call other root modules.** That's environment leakage.
- **Reusable modules don't reference environment-specific values** (account IDs,
  workspace names, secrets). Take them as inputs.
- **No cycles.** Terraform will catch most, but compositional cycles in
  modules-calling-modules can sneak in.

## Registry vs in-repo modules

| Approach | Best for | Trade-off |
|---|---|---|
| **In-repo `modules/` folder** | Most teams, most of the time | Fast iteration; harder to enforce versioned consumption |
| **Public Terraform Registry** | Generic, well-known infra (azurerm/aks, hashicorp/consul) | Other people maintain it; opinions might not match yours |
| **Private registry (Terraform Cloud, GitLab, Artifactory)** | Multi-team orgs wanting strict versioning | Operational overhead; CI must auth to registry |
| **Git source (`source = "git::..."`)** | Mid-size orgs without registry investment | Works fine; pin to tags, not branches |

Start with in-repo. Promote modules to a registry when:

- More than 3 teams consume them.
- You've shipped enough major versions to need a release process.
- You want CI-enforced consumption (e.g., "only modules from our private
  registry allowed").

## Testing modules

You should test modules. You usually don't until something breaks.

Options:

| Tool | What it does | When |
|---|---|---|
| `terraform validate` | Syntax + reference correctness | Always; runs in seconds |
| `terraform fmt -check` | Style enforcement | CI lint |
| **tflint** | Catches issues `validate` misses (deprecated args, naming) | CI lint |
| **Terratest** (Go) | Spins up real infra, asserts, tears down | Critical modules, slow but real |
| **terraform test** (built-in, since 1.6) | HCL-native test framework | New modules, no Go required |
| **Snapshot tests** (e.g., `terraform-compliance`) | Plan output matches expectations | Cheap; runs in seconds |

The pragmatic stack:

1. `validate` + `fmt` + `tflint` on every PR. Fast and catches most things.
2. Snapshot tests on critical modules — plan against a fixture, diff vs golden.
3. Terratest or `terraform test` for modules where wrong behavior is expensive
   (anything touching production data, security, IAM).

Don't try to test everything to 100%. Test the modules where mistakes cost real
money.

## Gotchas

- **`for_each` requires set/map known at plan time.** Using `for_each` on a
  computed list will fail. Convert to `count` or restructure.
- **`count`-based modules can't be re-ordered.** Removing the second item
  shifts indices and destroys the third. `for_each` with stable keys avoids
  this.
- **Module sources can't be variables.** `source = var.module_source` is
  invalid. Sources are evaluated before variables.
- **Hidden state coupling.** Two modules both manage tags on the same
  resource = fight every apply. Decide ownership.
- **Provider configuration in modules.** Don't. Pass the provider in, or use
  default providers. Provider blocks in modules surprise consumers.
- **Module upgrades change resource addresses.** Renaming a resource inside a
  module = destroy + recreate. Use `moved` blocks (since Terraform 1.1) to
  preserve identity.

## When NOT to write a module

- **One-off resource you create once and forget.** Just write it in the root.
- **The "abstraction" obscures what's actually deployed.** A module that hides
  10 resources behind 2 inputs sounds good until you debug it at 2am.
- **Premature abstraction.** Write the resource three times in three places,
  notice the pattern, *then* extract a module. The shape of the abstraction is
  clearer after three concrete uses.

## Related

- [`terraform-vs-pulumi-vs-cloudformation.md`](./terraform-vs-pulumi-vs-cloudformation.md)
  — the framing around which tool to use.
- [`terraform-state-management.md`](./terraform-state-management.md) — how state
  intersects with multi-module setups.
- [`policy-as-code-and-drift-detection.md`](./policy-as-code-and-drift-detection.md)
  — enforcing module-conformance rules in CI.
- [`../cloud-platform/terraform-on-azure.md`](../cloud-platform/terraform-on-azure.md)
  — Azure-specific layout patterns.
