# FinOps, tagging, and showback

**TL;DR:** You can't optimize a bill you can't allocate. Mandatory tags + policy
enforcement + per-team dashboards turn "the cloud bill is too high" (vague,
unactionable) into "the search team spent $50k on Elasticsearch this month and
needs to cut 20%" (specific, actionable). Showback drives behavior; chargeback
drives accountability; both require the tag discipline most teams skip until
it hurts.

## What FinOps actually is

A cultural and operational practice for *managing* (not just observing) cloud
spend. Three principles:

1. **Visibility** — everyone can see what they spend.
2. **Optimization** — engineers act on cost signals as part of their job.
3. **Accountability** — costs land on the teams that drive them.

The FinOps Foundation has a formal framework. The realistic minimum: tag
discipline, monthly per-team showback, a quarterly review of the biggest
lines.

## The tagging taxonomy that survives

Tags decay without enforcement. Pick a *small* set of mandatory tags and
enforce them via policy.

The taxonomy that works for most orgs:

| Tag | Purpose | Example |
|---|---|---|
| `owner` | Team or person | `team:payments` or `owner:alice@example.com` |
| `cost-center` | Billing allocation | `cc:1234` |
| `environment` | Lifecycle | `env:prod`, `env:staging`, `env:dev` |
| `app` or `service` | What this resource serves | `app:checkout-api` |
| `created-by` | IaC pipeline or human | `created-by:terraform` or `created-by:console-manual` |

Five tags, mandatory. Beyond that, optional tags per team's needs.

**Anti-patterns:**

- **Free-text owner tags.** `owner:bob`, `owner:Bob`, `owner:bob.smith@example.com`,
  `owner:Bob Smith`, etc. — all different to the cost system. Pick a format
  and enforce it.
- **Tags that drift** — `env:production` vs `env:prod` vs `Env:Production`.
  Standardize values, not just keys.
- **Hundreds of tags** — nobody fills them in, the data is garbage.

## Enforcement — the tag dies without it

Tags are aspirational without enforcement. Three layers:

1. **CI policy** (Checkov, OPA/Conftest, tfsec) — block IaC PRs that
   create resources without required tags.
2. **Cloud-native policy** (Azure Policy, AWS Config, GCP Organization
   Policy) — catch resources created outside IaC (console clicks, console
   bots).
3. **Auto-tagging from inheritance** (AWS resource groups, Azure Policy
   inheritance) — fill defaults from parent scope.

Combined, this gets you to ~95% tag coverage. Without enforcement, you'll
sit at 30-50%.

## Cost allocation — what the data tells you

With tags in place, costs roll up:

- **Per team** — who's the biggest consumer? Drill into top-N teams.
- **Per service** — within a team, which service is expensive? Often the
  database or the search cluster.
- **Per environment** — what fraction is dev/staging vs prod? Often the
  dev/staging line is shockingly close to prod.
- **Per resource type** — compute vs storage vs network vs managed services.
  Usually compute dominates; sometimes network egress surprises.

This is the report that turns "the bill is too high" into a list of
specific conversations.

## Showback vs chargeback

| Approach | Mechanism | Adoption pain | Behavior driver |
|---|---|---|---|
| **Showback** | Teams see their spend, no internal billing | Low | Soft — relies on team culture |
| **Chargeback** | Teams are billed internally for their spend | High — accounting changes | Strong — teams optimize because it hits their budget |

Most orgs start with showback. Chargeback works when finance is on board
and accounting structure supports cross-team transfers.

**The pragmatic middle:** showback to teams, with a quarterly "if we
chargeback, here's what your team would owe" report. Teams self-regulate
when the number becomes real.

## Per-team dashboards that drive behavior

What goes on a useful team dashboard:

- **This month's spend** vs last month, vs forecast.
- **Top 5 services by cost** within the team.
- **Trend over 6 months** — are we growing? Steady?
- **Cost per business metric** (cost per user, cost per transaction). The
  ratio matters more than the absolute number.
- **Right-sizing opportunities** — flagged by cloud advisor / cost tools.
- **Untagged resources** — the team's "data quality" score.

Dashboards live in Grafana, Cloud-native console, or a FinOps tool. The
specific tool matters less than the cadence — teams should look at it
weekly, not quarterly.

## Tools

| Tool | Best for | Notes |
|---|---|---|
| **AWS Cost Explorer + CUR** | AWS shops | CUR = raw data into your warehouse |
| **Azure Cost Management** | Azure shops | Decent built-in dashboards |
| **GCP Billing + BigQuery export** | GCP shops | Strong with BigQuery analysis |
| **Kubecost** | Kubernetes-specific | Per-namespace / per-pod cost attribution |
| **CloudHealth / Cloudability / Apptio Cloudability** | Enterprise, multi-cloud | Expensive but comprehensive |
| **Vantage / Spendlogic / nOps** | Newer, easier to adopt | Focused on actionable optimization |
| **Custom dashboards on data warehouse** | Detailed analysis | Most flexibility, most work |

Start cloud-native. Add Kubecost if you're Kubernetes-heavy. Consider a
multi-cloud tool when you're spending $1M+/year and the analysis becomes a
job.

## Untagged-resource hygiene

The graveyard. Every cloud account has it. The process:

1. **Inventory all untagged resources.**
2. **Try to attribute** — who created it? IaC blame, console audit logs.
3. **Notify** — ping the likely owner: "your resource is untagged; please
   tag it by date X or it'll be flagged for cleanup."
4. **Tag deadline reached** — move to a quarantine: tagged for deletion in
   N days unless claimed.
5. **Delete** — quarantine period elapsed, no claim. Delete it.

Some teams skip steps 4-5 ("we can't delete stuff!"). Then untagged
resources stay untagged forever. The discipline only works with consequences.

## Common patterns of waste

Beyond the obvious "over-provisioned compute" (see
[`right-sizing-and-savings-plans.md`](./right-sizing-and-savings-plans.md)):

- **Dev / staging running 24/7** — schedule off-hours shutdowns.
- **Stale snapshots** — daily snapshots kept for 5 years = enormous bill.
- **Orphaned load balancers** — load balancer with no targets, still
  billing.
- **Public IPs detached** — billed even when not in use.
- **NAT gateway costs** — surprising; bandwidth charges on cross-AZ NAT
  traffic add up.
- **Cross-AZ data transfer** — within-AZ is free; cross-AZ is not. Architect
  to localize.
- **Premium-tier managed services in non-prod** — Premium SSD, HA database
  tier, etc.
- **CloudWatch logs / Application Insights** with no retention — collect
  forever, pay forever.
- **Underused Kubernetes clusters** — a cluster with 5% utilization is still
  billing for all nodes.

## The "we never asked for that" mystery line

Every cloud bill has surprises. Process:

1. **Filter to the surprise line.**
2. **Group by resource** to find the cause.
3. **Check the cloud's billing FAQ** — some services bill in unintuitive
   ways (e.g., NAT gateway processing fees, egress per-GB, KMS API call
   fees).
4. **Confirm against the resource owner.** Often it's a legitimate but
   forgotten resource.
5. **Fix the root cause** (delete, optimize, or document).

Most surprises are real spend on real resources. A small fraction are
billing errors — those get filed as support tickets.

## The unit-economics conversation

The most strategic FinOps question: **what does it cost us per user / per
transaction / per business unit?**

If cost-per-user is going up, scale won't save you. If cost-per-user is
going down, you have operating leverage.

Tracking this requires:

- A clean cost dataset (per service, per environment).
- A clean usage dataset (active users, transactions, requests).
- A pipeline that joins them.

The business asks "are we becoming more or less efficient?". This data
answers it. Without it, every cost conversation is anecdotal.

## Gotchas

- **Tags don't propagate to child resources** in most clouds. The VM is
  tagged; the attached volume isn't. Use IaC modules that tag everything
  consistently.
- **Tag changes don't backfill historical cost data.** If you tag today,
  yesterday's cost is still untagged.
- **Cost data has 24-48 hour latency.** Don't expect to see today's spend
  this afternoon.
- **Reserved capacity savings appear as a credit**, not a discount on the
  resource. Make sure your dashboards reconcile properly.
- **Free-tier and credits expire.** Plan for the post-credit bill.
- **"Cost optimization" can be a backdoor to "let's not do anything new".**
  The point is to spend more efficiently, not to spend less in absolute
  terms.

## When FinOps is overkill

- **Solo dev, < $500/month.** Just use cloud-native billing alerts.
- **Single-product startup pre-revenue.** Be aware, but don't build
  dashboards. Iteration speed matters more.
- **Truly fixed-cost environments** (on-prem-only).

Most orgs at $50k+/month benefit from real FinOps practice.

## Related

- [`right-sizing-and-savings-plans.md`](./right-sizing-and-savings-plans.md)
  — the optimization side; FinOps surfaces the opportunities.
- [`elasticsearch-cost-and-performance.md`](./elasticsearch-cost-and-performance.md)
  — domain-specific cost story.
- [`../iac/policy-as-code-and-drift-detection.md`](../iac/policy-as-code-and-drift-detection.md)
  — policies enforce tag mandates.
- [`../cloud-architecture/well-architected-decisions.md`](../cloud-architecture/well-architected-decisions.md)
  — cost is one of the five pillars.
