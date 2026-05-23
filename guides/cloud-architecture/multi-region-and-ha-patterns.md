# Multi-region and HA patterns

**TL;DR:** Multi-region is expensive and complex. Most apps don't need it; they need
*good* single-region HA across availability zones. If you do need multi-region, pick
active-passive unless you can justify (with money and a CAP-theorem conversation) why
you need active-active.

## The HA ladder

Each rung costs more and protects against larger failures:

1. **Single instance** — no HA. Don't.
2. **Multi-instance, single AZ** — survives instance failure. Doesn't survive an AZ.
3. **Multi-AZ single region** — survives AZ failure. The default for production.
4. **Multi-region active-passive** — survives region failure. Failover is manual or
   triggered.
5. **Multi-region active-active** — no failover; both regions always serve. Most
   complex; highest cost; tightest consistency constraints.

Skipping rungs is how teams end up with a million-dollar bill protecting a workload
that needed rung 3.

## Single-region multi-AZ — the baseline you should *actually* be at

Most cloud regions span 3 AZs. Use them:

- **Compute** — node pools / ASGs / scale sets span ≥2 AZs (3 is better).
- **Managed databases** — zone-redundant tier. The single-zone tier is a tax on
  your future self.
- **Load balancers** — zone-redundant SKU.
- **Object storage** — ZRS (zone-redundant) or equivalent.
- **DNS** — vendor (Azure DNS, Route 53, Cloud DNS) already multi-AZ.

This survives the most likely failure (one AZ goes dark) without the operational
tax of multi-region. If you haven't done this yet, do it before thinking about
multi-region.

## Active-passive multi-region

Primary handles traffic. Secondary is warm or pilot-light, ready to take over.

**Mechanism:**
- DNS-based or anycast-based failover.
- Health checks at the global tier (Front Door, Route 53, Cloud Load Balancing).
- TTL low (30-60s) for DNS-based; instant for anycast.

**Data:**
- One-way replication primary → secondary.
- Async usually (sync across regions adds 10-100ms to every write).
- Replication lag is your RPO ceiling.

**Failover:**
- Automated for health-check-driven failures (region down).
- Manual for "something feels off" (saves you from flapping).
- Documented runbook. Tested quarterly. (See
  [`disaster-recovery-playbook.md`](./disaster-recovery-playbook.md).)

**Cost:**
- Pay for secondary stack at reduced scale.
- Plus replication egress.
- Roughly 30-60% of primary's cost.

This is the right answer for most "we need multi-region" requests.

## Active-active multi-region

Both regions serve traffic continuously. Failover = remove one from rotation.

**When it's justified:**
- Truly global user base where regional latency matters business-wise.
- Stringent RTO (seconds) that active-passive can't hit.
- Regulatory data residency in multiple regions.

**When it's *not* justified but teams do it anyway:**
- "We want to be modern." (Buy the t-shirt instead.)
- "It's better disaster recovery." (Active-passive is enough.)
- "Latency." (CDN + regional caches usually fixes this for less.)

**The hard parts:**
- **Data:** either accept eventual consistency (and design for it) or use a
  globally-distributed database (Cosmos DB multi-region, DynamoDB Global Tables,
  Spanner). The latter is expensive and constrains your data model.
- **Identity:** sessions need to work in both regions. Sticky-session is a smell;
  signed tokens (JWT) are better.
- **Writes:** can both regions accept writes for the same record? If yes,
  you've signed up for conflict resolution. If no, it's not really active-active
  for writes (which is fine — just be honest about it).
- **Split-brain:** network partition between regions, both think they're primary.
  Picking the wrong consistency model here loses data permanently.

## Traffic routing — the global tier

The four common shapes:

| Layer | Examples | What it routes by |
|---|---|---|
| **Anycast IP** | Cloudflare, Azure Front Door (anycast), GCP global LB | Network topology — nearest PoP wins |
| **GeoDNS** | Route 53 latency/geo, Azure Traffic Manager, NS1 | DNS answer varies by resolver location |
| **Weighted DNS** | Most DNS providers | Manual percentage split (good for migrations) |
| **Smart client** | App-side region selection from a service catalog | App knows; complex but explicit |

**Anycast** is the gold standard for performance. Failover is instant — withdraw a
route, packets reroute. No TTLs to wait for.

**GeoDNS** is good enough for most teams. Failover is bounded by TTL (so keep it
low for failover-critical records).

**Weighted DNS** is a tactical tool, not a strategy. Useful for migrations and
canaries. Bad for steady-state.

**Smart client** is for control-freaks (gaming, trading platforms). High blast
radius if you ship a bad service catalog.

## Data replication — sync vs async

| Mode | Latency cost | Data loss on failover | When |
|---|---|---|---|
| Synchronous | Cross-region round-trip on every write (10-100ms) | None | Same AZ / same region only |
| Asynchronous | None on hot path | Up to replication lag (seconds to minutes) | Cross-region default |
| Quasi-sync (e.g., Postgres) | Some replicas must ack | None for acked replicas | Hybrid HA setups |

Sync across regions is almost always wrong. The latency tax compounds across every
write in the system. The exceptions are payments and ledgers where data loss is
unacceptable — and even then, you architect around it (e.g., write to a single
region but replicate to a synchronous standby within the same region).

## Split-brain prevention

The failure mode that haunts active-active. Two regions, network partition,
both think they're the leader. Both accept writes. Now you have conflicting
state.

The mitigations:

- **Single-writer per shard** — partition the data so each region owns disjoint
  shards. Reads can go anywhere; writes go to the shard owner. (Cosmos DB
  multi-master is the opposite — convenient but you sign up for the merge logic.)
- **Quorum-based systems** — Raft / Paxos / etcd-style. Two regions can't form
  quorum if a third arbiter region is down. Three-region setups are stable.
  Two-region active-active with quorum is *not* stable — there's no tiebreaker.
- **Vector clocks / CRDTs** — let conflicts happen, merge deterministically.
  Works for some data shapes (counters, sets); not for "the user's current
  balance".
- **Last-write-wins** — the easy option. Loses data. Use only when "losing the
  loser's write" is acceptable (which is rarer than you think).

If you don't have a deliberate answer to "what happens when both regions write
to the same record at the same instant", you don't have active-active — you
have a future incident.

## When single-region is the right call

- **You're not at multi-AZ yet.** Fix that first.
- **Your users are concentrated geographically.** Latency from one region is
  acceptable; multi-region adds cost without benefit.
- **Your data residency is single-jurisdiction.** Multi-region complicates
  compliance.
- **You're early-stage.** Multi-region overhead is a tax on iteration speed.
  Plan for it; don't build it yet.
- **Your RTO is hours, not minutes.** Backup & restore to another region on
  demand is cheaper than warm standby.

## Gotchas

- **Egress costs are the silent killer.** Cross-region data transfer is 5-50x
  the cost of intra-region. Replication and chatter add up. Read the bill.
- **Quotas are per-region.** The day you fail over is the day you find out the
  DR region's quota is 4 vCPUs.
- **Managed services are not all multi-region.** Some are single-region only
  (specific Azure SQL tiers, certain DynamoDB features, etc.). Read the SLA
  page before designing.
- **Service mesh complicates this.** Multi-region mesh is its own discipline
  — most teams shouldn't try.
- **Observability needs to be multi-region too** — or you're flying blind
  during the exact incident you built this for.
- **Don't trust the cloud provider's "zone-redundant" badge blindly.** Read
  what it means. For some services, "zone-redundant" means "we'll restore in
  another zone within X hours", not "no downtime".

## When NOT to go multi-region

- **You haven't measured availability.** If you don't know your current
  uptime, you don't know what you're improving.
- **You haven't done multi-AZ properly.** Fix the foundation first.
- **You don't have automated tests for failover.** Without those, multi-region
  is a more expensive way to be down.
- **The CEO read a Hacker News post about it.** Push back, calmly, with numbers.

## Related

- [`well-architected-decisions.md`](./well-architected-decisions.md) — the
  trade-off framing for the cost/reliability conversation.
- [`disaster-recovery-playbook.md`](./disaster-recovery-playbook.md) — the
  operational side of multi-region.
- [`../cloud-platform/aks-production-checklist.md`](../cloud-platform/aks-production-checklist.md)
  — multi-AZ Kubernetes baseline.
- [`../cloud-platform/observability-stack.md`](../cloud-platform/observability-stack.md)
  — how to know when failover happened.
