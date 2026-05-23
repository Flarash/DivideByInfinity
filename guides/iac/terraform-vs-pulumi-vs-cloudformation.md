# Terraform vs Pulumi vs CloudFormation

**TL;DR:** Terraform wins on provider coverage, ecosystem, and "engineer-anywhere can
read it". Pulumi wins on language ergonomics and testability. CloudFormation wins
only when you're 100% AWS, want zero new tooling, and prize tight AWS integration.
Pick by team and blast-radius, not by Twitter.

## The honest comparison table

| Dimension | Terraform | Pulumi | CloudFormation |
|---|---|---|---|
| **Language** | HCL (declarative DSL) | Real language (TS, Python, Go, C#, Java) | YAML/JSON + intrinsic functions |
| **Cloud coverage** | Excellent (all 3 + 100s more) | Excellent (mostly via Terraform providers) | AWS only |
| **State** | Remote backend (S3/Azure Storage/GCS/TFC) | Pulumi Cloud / S3 / Azure / self-hosted | AWS-managed (in your stack) |
| **Modules / reuse** | Modules, registry | Component resources, packages | Nested stacks (clunky) |
| **Drift detection** | `terraform plan` + drift workflows | `pulumi refresh` | Stack drift in console |
| **Testing** | Terratest (Go), terraform test | Native (jest/pytest/...) | Limited (TaskCat, cfn-lint) |
| **Policy as code** | OPA, Sentinel, Checkov, tfsec | CrossGuard, OPA, Checkov | cfn-guard, OPA |
| **Cost (tooling)** | Free (OSS) + Terraform Cloud tier | Free (OSS) + Pulumi Cloud tier | Free |
| **Lock-in** | Multi-cloud-friendly | Multi-cloud, but pulumi APIs are vendor-flavored | AWS-only |
| **Learning curve** | Low to moderate | Low if you know the language | Low for AWS folks, painful for newcomers |
| **Hiring pool** | Largest | Growing | Large (in AWS shops) |

## When Terraform is the right choice (default)

- **You work across more than one cloud** (or even one cloud + Cloudflare + Datadog +
  GitHub). Terraform providers are the broadest ecosystem.
- **Your team includes ops + dev + SRE + occasional contractors.** HCL is readable
  by everyone. Real languages have steeper onboarding for non-developers.
- **You want to hire fluently.** The job market knows Terraform.
- **You want the path with the most documentation, blog posts, and Stack Overflow
  answers.** No question.

The downside you sign up for:

- HCL's `for_each`/`dynamic`/`merge`/`lookup` calisthenics. For complex logic you'll
  write things you don't want to maintain.
- Module versioning discipline is on you.
- Testing is doable (Terratest) but feels bolted-on.

## When Pulumi is the right choice

- **Your team is mostly engineers and reads tests as documentation.** Pulumi in
  TypeScript reads like normal code: loops, conditionals, types, imports.
- **You need real testing.** Unit tests with mocked providers, integration tests,
  property-based tests — Pulumi can do them with native tooling.
- **You have complex parameterization.** When the choice is "another `dynamic`
  block in HCL" or "a function in TypeScript", the function wins.
- **You're building an internal platform** where the IaC is *part of* the
  platform product (Backstage golden paths, custom CLIs).

The downside you sign up for:

- Smaller community + fewer blog posts.
- Pulumi Cloud cost for state if you want managed state.
- Real-language flexibility cuts both ways — teams build clever abstractions that
  the next engineer can't read.

## When CloudFormation is the right choice

- **You're committed to AWS** and not leaving (regulatory, contractual, or strong
  technical reasons).
- **You want zero new dependencies in CI.** AWS CLI ships everywhere already.
- **You need StackSets** (CloudFormation has the best multi-account/multi-region
  deployment story on AWS).
- **You're operating in regulated environments** that have already vetted
  CloudFormation. Adding Terraform is paperwork.

The downside you sign up for:

- AWS-only, period.
- YAML/JSON ergonomics. CDK softens this if you adopt it.
- Drift detection is awkward.
- Stack updates have failure modes (UPDATE_ROLLBACK_FAILED) that you'll meet at
  the worst possible time.

**Aside on AWS CDK:** CDK *generates* CloudFormation from TypeScript/Python/etc.
It gives you real-language ergonomics with CloudFormation's state semantics. If
you're locked to AWS and want better ergonomics, CDK is the answer — not raw
CloudFormation.

## Blast radius — the dimension nobody mentions

The most expensive mistake in any IaC is "I ran apply and it deleted the prod
database". Compare blast-radius defaults:

- **Terraform** — `terraform apply` shows a plan. If you don't read it, the plan
  shows red minuses and you proceed. Easy to miss "destroy and recreate".
- **Pulumi** — same shape as Terraform but with stronger code-review culture
  because the IaC is in PRs alongside app code.
- **CloudFormation** — change sets are explicit. You have to create and review
  a change set before executing. Marginally safer by default.

All three are equally safe *if you wire them to CI with required reviews and
plan-on-PR.* All three are equally dangerous if you let people apply from
laptops.

## The "we already started, can we switch?" question

You usually can't, cleanly. Each tool tracks its own state. Migration paths:

- **CFN → Terraform:** `terraformer` can import existing resources. State surgery
  follows. Plan for weeks per stack.
- **CFN → CDK:** mostly painless because CDK generates CFN. Importing existing
  resources into CDK constructs is the work.
- **Terraform → Pulumi:** `pulumi import` exists. Tolerable for small stacks.
- **Pulumi → Terraform:** harder. No first-class export. Hand-translation.

The pragmatic answer: pick one for *new* work, leave the old in place, migrate
opportunistically when you'd touch it anyway.

## The CDK conversation, briefly

AWS CDK and CDK for Terraform (CDKTF) are both "real language → declarative IaC"
patterns:

- **AWS CDK** — TypeScript/Python → CloudFormation. AWS-only. Mature.
- **CDKTF** — TypeScript/Python/Go → Terraform JSON. Multi-cloud. Less mature.

Both add a layer between you and the underlying IaC. The trade-off: more
ergonomic code, but debugging requires understanding two layers when things
break. CDKTF is the right choice for teams that want Pulumi-style ergonomics
without leaving Terraform's state ecosystem.

## Decision matrix (the "what for which team" cheat sheet)

| Team shape | Cloud shape | Recommendation |
|---|---|---|
| Mixed ops+dev, multi-cloud | Multi-cloud | **Terraform** |
| Engineer-heavy, internal platform | Multi-cloud | **Pulumi** or **CDKTF** |
| Strict AWS shop, ops-led | AWS only | **CloudFormation** |
| Strict AWS shop, engineer-led | AWS only | **AWS CDK** |
| Contractor-heavy, frequent handoffs | Any | **Terraform** |
| Tightly-typed-language fans | Any | **Pulumi** (Go or TS) |
| Heavily regulated, vetted tooling | AWS only | **CloudFormation** (already approved) |

## Gotchas (apply to all three)

- **State is the asset.** Lose state = lose ability to manage your infra
  cleanly. Backup state. Treat it as production data.
- **State lock contention.** Multiple humans + CI applying at once = locks
  fighting. Single-writer applies (CI-only) is the sane pattern.
- **Provider versions matter.** A floating provider version means yesterday's
  apply and today's apply are different programs. Pin versions.
- **Manual changes in the console are betrayals.** Drift detection catches some;
  not all. Policy-as-code can block them.
- **"It works in dev"** — IaC behavior diverges across cloud regions, account
  tiers, and quotas. Test promotion paths.
- **Don't use IaC to wire secrets directly** — store secret references
  (Key Vault URI, Secrets Manager ARN). The actual secret lives in the secret
  store and never touches state.

## When NOT to use any of them

- **One-off resource you'll never touch again** — sometimes a console click is
  fine. Don't religiously IaC the dev-team test S3 bucket.
- **Resources with poor provider support** — bleeding-edge service that
  Terraform/Pulumi/CFN doesn't model well. Manual + documented + ticket for
  later.
- **For application code** — IaC is for infra. Don't model your Lambda's
  business logic in CloudFormation.

## Related

- [`terraform-module-design.md`](./terraform-module-design.md) — module
  composition rules and the thin-wrapper anti-pattern.
- [`terraform-state-management.md`](./terraform-state-management.md) — backends,
  locking, and state surgery.
- [`policy-as-code-and-drift-detection.md`](./policy-as-code-and-drift-detection.md)
  — OPA / Checkov / tfsec and the manual-change problem.
- [`../cloud-platform/terraform-on-azure.md`](../cloud-platform/terraform-on-azure.md)
  — Terraform-specific Azure patterns and gotchas.
