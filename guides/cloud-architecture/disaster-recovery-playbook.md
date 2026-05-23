# Disaster recovery playbook

**TL;DR:** DR plans that haven't been *executed* in the last 90 days are theatre. Pick
your RTO/RPO honestly, choose the cheapest tier that meets them, then *actually run
a failover* every quarter. Most outages are not "the region died" — they're "we never
tested the runbook and now nothing works".

## RTO and RPO — set them before choosing a strategy

| Term | What it asks | Honest answer ranges |
|---|---|---|
| **RTO** (Recovery Time Objective) | How long can the service be down? | 5 min (trading) → 4 hours (most SaaS) → 24 hours (internal tools) |
| **RPO** (Recovery Point Objective) | How much data can we lose? | 0 (payments) → 1 min → 1 hour → 24 hours |

Two rules teams get wrong:

1. **Different services can have different RTO/RPO.** The marketing site and the
   payments API are not the same tier. Stop budgeting like they are.
2. **RTO/RPO are business decisions, not engineering decisions.** If the business
   says "we can't be down for an hour", confirm in writing — because the bill
   for RTO=5min looks very different from RTO=4hr.

## The four DR tiers (cost climbs steeply)

| Tier | Pattern | RTO | RPO | Cost (relative) |
|---|---|---|---|---|
| **Backup & restore** | Snapshots in another region. Restore on demand. | hours-days | hours | $ |
| **Pilot light** | Minimal services running in DR region, scale up on failover. | ~1 hour | minutes-hour | $$ |
| **Warm standby** | Scaled-down full stack in DR region, scale up on failover. | minutes | seconds-minutes | $$$ |
| **Active-active** | Full stack in both regions, load-balanced normally. | seconds | ~0 (sync replication) | $$$$ |

Choosing wrong:

- Defaulting to active-active because it sounds responsible → 2x infrastructure
  bill, sync replication latency, split-brain risk.
- Defaulting to backup & restore for the payments database → 6-hour outage during
  the real event, lost transactions, regulator phone call.

Match the tier to the data, not to the team's anxiety level.

## Runbook anatomy

The single artifact that matters. If this doesn't exist, you don't have DR.

```markdown
# Runbook: Failover primary database to secondary region

**Trigger:** Primary region unreachable for >10 min OR primary DB CPU=100%
            for >15 min with no recovery.
**Owner:** Platform on-call.
**Estimated duration:** 25 minutes.
**Last successfully executed:** 2025-04-12 (DR drill).

## Pre-checks (5 min)
1. Confirm primary is actually down: `az resource show ...` + dashboard link.
2. Check secondary replica lag: `SELECT ... FROM pg_stat_replication;`
3. Page the database owner.

## Failover (10 min)
1. Promote read replica:
   ```
   az postgres flexible-server replica promote --name ... --resource-group ...
   ```
2. Update Front Door origin priority: secondary → 1, primary → disabled.
   (Link to exact portal page.)
3. Rotate DNS for `db.internal.example.com` to secondary endpoint.
   TTL is 30s. Verify with `dig`.

## Verification (5 min)
1. App health check: `curl https://api.example.com/healthz` (expect 200).
2. Run smoke transaction: log in as `dr-canary@example.com`, confirm 200.
3. Check error rate in Grafana — link to dashboard pinned at top of runbook.

## Failback (later, when primary recovers)
... (separate section, equally detailed) ...

## Comms
- Status page: post within 5 min.
- #incidents Slack channel: announce start + completion.
- Customer email if outage >30 min: template in /docs/templates/.
```

Things to put *in* the runbook:

- Exact commands, copy-pasteable.
- Screenshots of portal pages that don't have CLI equivalents.
- Verification commands with expected output.
- Links to dashboards, with the metric you're looking for.
- Comms templates.

Things to keep *out* of the runbook:

- Explanations of why (link to the ADR or design doc).
- Anything that requires "ask Alice who knows the trick".
- Conditional logic with more than two branches (split into separate runbooks).

## The quarterly DR drill

The non-negotiable practice. If you do nothing else, do this.

- **Schedule it.** Recurring calendar entry, every 90 days. Don't reschedule.
- **Notify** but don't pre-stage. The point is to find what's broken.
- **Run the runbook as written**, not from memory. Note every step that confused
  someone or didn't work.
- **Time it.** Did you hit RTO? If not, why?
- **File the gaps** as tickets *during* the drill, not "later".
- **Rotate who runs it.** The senior engineer who wrote the runbook isn't who
  will be on call at 3am.

The drill *will* fail the first time. That's the value. The cost of "we never
tested it" is paid in full on the day it matters.

## Backups — the substrate everything else assumes

Backups must be:

- **Automated** — humans forget.
- **Geographically separated** — same region = same disaster.
- **Encrypted** — at rest *and* in transit to backup storage.
- **Tested** — restore at least one backup per quarter. Untested backups have a
  ~30% chance of being unusable (pick your favorite survey; the number is
  always uncomfortably high).
- **Immutable for a window** — ransomware/insider deletion protection. WORM
  policies, S3 Object Lock, Azure Blob immutability, GCS bucket lock.
- **Retained on a schedule that matches compliance** — 30 days hot, 1 year cold,
  7 years for financial data.

The retention conversation is where finance and security disagree most. Get
both in a room, settle it once, write it down.

## The non-region gotchas

People plan for "the region died". The actual outages tend to be:

- **DNS provider outage** — your failover *uses DNS*. Plan for it.
- **IdP outage** — failover requires the on-call to log in. If SSO is down, do
  you have break-glass admin accounts?
- **Cert renewal failure** — Let's Encrypt rate limits + an automated renewal
  outage = no certs in the DR region.
- **Secrets not replicated** — Key Vault / Secrets Manager regional resources
  don't automatically span regions. Set up replication.
- **TTL too long** — DNS TTLs of 1 hour mean your failover finishes in 30s and
  takes 60 minutes to be visible. Set 30-60s TTL for DR-critical records.
- **Quota in the secondary region** — never used it = never asked for capacity.
  The day you need it, you find out the quota is 4 vCPUs.
- **Cross-region IAM** — the user that can write to the primary KMS key cannot
  decrypt with the secondary region's CMK. Test the keys.

These are the ones that turn a 25-minute runbook into a 6-hour incident.

## DR for stateful systems

Generally hardest. Order of difficulty (easiest to worst):

1. **Object storage** — built-in cross-region replication, trivial.
2. **Managed databases** — read replicas + promotion path, well-documented.
3. **Search clusters** (Elasticsearch/OpenSearch) — cross-cluster replication
   works but has gotchas around index lifecycle.
4. **Caches** (Redis) — usually rebuild from source-of-truth, not replicated.
5. **Message queues** — depends. RabbitMQ federation, Kafka MirrorMaker, Azure
   Service Bus geo-DR all have different semantics.
6. **In-memory state in your own apps** — there is no DR for "we kept it all in
   RAM and didn't checkpoint". Don't do this for anything that matters.

## Gotchas

- **"It's encrypted at rest in the backup" ≠ "we can decrypt it".** Verify the
  *restore* path includes the keys.
- **Test in production-like, not in toy environments.** Your dev environment
  doesn't have the 30TB database.
- **Don't share a control plane between primary and DR.** A misconfigured CI
  pipeline can deploy bad code to both at once.
- **Document who can declare a disaster.** "Should we fail over?" is the slow
  step, not the failover itself.
- **DR plans go stale.** The architecture changed; the runbook didn't. Pin the
  runbook to the design doc and review both together.
- **The CEO/CTO email template needs to exist** *before* the incident, not
  drafted at 3am.

## When NOT to invest heavily in DR

- **Pre-product-market-fit** — your DR plan is "rebuild from git". That's fine
  for now.
- **Internal tools nobody depends on** — a 24-hour RTO is acceptable. Backups
  are enough.
- **Vendor-managed SaaS you don't operate** — your DR is *their* DR, plus a
  procedural fallback (e.g., switch to manual process for 24 hours).

## Related

- [`well-architected-decisions.md`](./well-architected-decisions.md) — DR is
  the operational form of the reliability pillar.
- [`multi-region-and-ha-patterns.md`](./multi-region-and-ha-patterns.md) — the
  topology decisions DR depends on.
- [`../cloud-platform/aks-production-checklist.md`](../cloud-platform/aks-production-checklist.md)
  — Velero, etcd backups, and the Kubernetes-specific DR pieces.
- [`../cloud-platform/observability-stack.md`](../cloud-platform/observability-stack.md)
  — DR for the observability stack itself (you need it most when it's gone).
