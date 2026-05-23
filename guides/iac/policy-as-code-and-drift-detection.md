# Policy as code and drift detection

**TL;DR:** Policy-as-code catches bad infra *before* it ships. Drift detection
catches reality diverging from your IaC *after* it ships. You need both. Pick
one tool per layer (OPA + tfsec is a fine combo) and resist the urge to add
five.

## Why policy as code

The alternative is review-by-tribal-knowledge. Senior engineers spot
"that storage account is public" in PRs; juniors don't. Policy-as-code is
the spotting, automated and enforced.

What it catches:

- Misconfigurations (public storage, open security groups, unencrypted disks).
- Missing tags (cost allocation breaks).
- Naming-convention violations.
- Module-source restrictions ("only modules from our private registry").
- Compliance rules (HIPAA, SOC 2, PCI controls expressible as policies).

What it doesn't catch:

- Logic bugs ("this rule allows traffic from the wrong subnet" is correct *and*
  wrong).
- Architectural mistakes.
- Things the policy author didn't think of.

Policy-as-code is a floor, not a ceiling.

## The tool landscape

| Tool | Layer | Style | Strength |
|---|---|---|---|
| **OPA / Rego** | Generic policy engine | Custom rules in Rego | Most flexible; learning curve |
| **Conftest** | OPA wrapper for files | Rego against Terraform plans, YAML, etc. | Easy to wire into CI |
| **Checkov** | IaC security scanner | Hundreds of pre-built rules | Fast win, low-friction |
| **tfsec** | Terraform security scanner | Pre-built rules, fewer false positives | Specifically Terraform-focused |
| **Trivy** | Multi-tool scanner | IaC + containers + secrets | One tool, many surfaces |
| **Sentinel** | HashiCorp policy engine | Terraform Cloud/Enterprise only | First-party Terraform integration |
| **cfn-guard** | CloudFormation policy | AWS-native | If you're CloudFormation-only |
| **CrossGuard** | Pulumi policy | Pulumi-native | If you're Pulumi-only |

The pragmatic stack for most teams:

- **Checkov or tfsec** for "low-hanging fruit" security rules. Plug-and-play.
- **OPA/Conftest** for the team's *custom* rules (tagging, naming, module
  allowlists).
- **Sentinel** if you're on Terraform Cloud/Enterprise and want plan-time
  enforcement.

Don't run all of them. Pick one IaC scanner + one custom rule engine.

## Where policies run

Three points in the pipeline:

| Stage | What you catch | Latency to engineer | Cost of failure |
|---|---|---|---|
| **Pre-commit hook** | Style, syntax | Seconds | Annoying if slow |
| **CI on PR** | Policy violations against `terraform plan` | Minutes | PR check fails; fix and re-push |
| **Plan-time / apply-time** | Last-line-of-defense in TFC/Atlantis/Spacelift | Minutes | Apply blocked |
| **Post-apply drift detection** | Reality divergence | Hours/days | Schedule; alert |

The strongest setup uses all four. The minimum viable setup is **CI on PR + drift
detection on schedule**.

## A concrete Checkov + OPA setup

CI step in GitHub Actions:

```yaml
- name: Checkov
  uses: bridgecrewio/checkov-action@v12
  with:
    directory: .
    framework: terraform
    soft_fail: false
    skip_check: CKV_AWS_50  # documented exception with link

- name: Generate Terraform plan
  run: |
    terraform init
    terraform plan -out=tfplan
    terraform show -json tfplan > tfplan.json

- name: Conftest policies
  run: |
    conftest test --policy ./policies tfplan.json
```

`policies/required_tags.rego`:

```rego
package main

required_tags := {"owner", "cost-center", "environment"}

deny[msg] {
  resource := input.resource_changes[_]
  resource.change.actions[_] == "create"
  startswith(resource.type, "azurerm_")
  tag := required_tags[_]
  not resource.change.after.tags[tag]
  msg := sprintf("%s '%s' is missing required tag '%s'", [resource.type, resource.address, tag])
}
```

This forces every new Azure resource to carry the three mandatory tags. The
team that owns tagging owns this file.

## Drift detection

Drift = the cloud diverged from your IaC. Causes:

- Someone clicked in the console.
- An emergency hotfix bypassed CI.
- A managed service auto-changed something (e.g., AWS auto-updating tags).
- A separate Terraform stack accidentally manages the same resource.

### Detection mechanisms

**`terraform plan` on a schedule.** Compare to the last known-good. Non-empty
diff = drift. The crudest, most reliable mechanism.

```yaml
# scheduled GitHub Action, daily 06:00 UTC
on:
  schedule:
    - cron: "0 6 * * *"

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - id: plan
        run: |
          set +e
          terraform plan -detailed-exitcode -out=tfplan
          echo "exitcode=$?" >> $GITHUB_OUTPUT
      - name: Notify on drift
        if: steps.plan.outputs.exitcode == '2'
        run: |
          # post to Slack / open a GitHub issue / page on-call
```

`-detailed-exitcode` returns 2 if there are pending changes — perfect for drift
detection.

**Cloud-native drift tools:**

- **Driftctl** (open source, archived but still useful) — compares cloud reality
  vs Terraform state across all resources, including ones Terraform doesn't
  know about ("unmanaged" resources).
- **Terraform Cloud / Atlantis / Spacelift drift detection** — built-in,
  schedulable, with notifications.
- **AWS Config / Azure Policy** — cloud-native config-drift detection;
  IaC-agnostic.

**Pick one drift mechanism per stack.** Multiple = duplicate alerts = noise =
ignored.

### What to do when drift is detected

The temptation is always "just apply to make it match the code". Don't, until
you understand:

1. **Why is the cloud different from state?** Someone changed something. Why?
2. **Is the cloud's state actually correct?** Maybe someone fixed an
   emergency, and the code is wrong.
3. **What's the safest direction?**
   - Cloud → code (refactor IaC to reflect the change) when the change is
     valid and should persist.
   - Code → cloud (`terraform apply` to revert) when the change was a mistake
     or an emergency you've now fixed properly.

Then add prevention:

- IAM restrictions: only the CI service principal can write to prod.
- Change management: anything that touches prod outside CI requires an incident
  ticket.
- Stronger policies: catch the class of drift in CI before it ships next time.

## Policy-as-code anti-patterns

- **Rules that nobody can explain.** "We block this because the scanner said so"
  is a recipe for engineers disabling the rule.
- **Soft-fail everywhere.** "We log violations but don't fail the build" =
  nobody fixes anything. Pick the rules you'll enforce; enforce them.
- **Hundreds of rules nobody owns.** A small set of enforced rules > a large
  set of suggestions.
- **No exception mechanism.** Real life has exceptions; pretending otherwise
  pushes teams to disable the entire policy framework.
- **No update cadence.** Policies that don't evolve become anti-correlated with
  real risk over time.

## The exception process

Pretending there are no exceptions is naive. Codify the exception:

```hcl
resource "azurerm_storage_account" "public_assets" {
  # tfsec:ignore:azure-storage-default-action-deny
  # exception: this account hosts public marketing PDFs.
  # owner: marketing-eng@example.com
  # reviewed: 2025-03-14 by security
  # next review: 2026-03-14
  ...
}
```

Rules:

- Exception lives next to the code, not in a wiki.
- Owner, reason, review date, next review.
- CI lints exceptions older than a year — forces re-review.

## Gotchas

- **Plan-output-as-input fragility.** Conftest against `terraform show -json`
  works, but the JSON schema can shift across Terraform versions. Pin
  Terraform versions in CI.
- **Plan can lie.** A `for_each` over computed values shows up as "known after
  apply", which policies can't evaluate. Some violations only surface at
  apply-time.
- **Scanners disagree.** Run both Checkov and tfsec and you'll see different
  findings for the same code. Pick the union you'll fix; document the rest as
  exceptions.
- **Drift detection costs API calls.** A daily plan on every stack adds up.
  Stagger schedules or use `-target=` for narrower checks.
- **Policy code is code.** Test the policies themselves. OPA has `opa test`.
  Run it in CI.
- **Don't put secrets in policy violation messages.** They end up in CI logs.

## When policy-as-code is overkill

- **Solo project.** You're the policy.
- **Tiny team, single environment.** Manual review is faster.
- **No production traffic.** Risk is low; iteration speed matters more.

The threshold for adopting policy-as-code is roughly: more than one team
contributing, or more than one environment, or any compliance scope.

## Related

- [`terraform-vs-pulumi-vs-cloudformation.md`](./terraform-vs-pulumi-vs-cloudformation.md)
  — the IaC layer policies enforce against.
- [`terraform-module-design.md`](./terraform-module-design.md) — module-allowlist
  policies depend on having modules to allowlist.
- [`terraform-state-management.md`](./terraform-state-management.md) — drift
  detection works against state; locking and surgery interact.
- [`../security/iam-least-privilege-in-practice.md`](../security/iam-least-privilege-in-practice.md)
  — preventing drift by restricting who can change prod outside CI.
