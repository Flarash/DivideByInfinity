# Well-architected decisions in practice

**TL;DR:** The five pillars (reliability, security, cost, performance, operational
excellence) are not a checklist — they're a forced trade-off framework. Every meaningful
architecture decision sacrifices at least one pillar to strengthen another. Write the
trade-off down as an ADR or you will re-litigate it every quarter.

## What the pillars actually mean once you stop quoting the docs

- **Reliability** — survives failures you've designed for. Not "100% uptime", but
  "degrades to a known mode within RTO/RPO".
- **Security** — least privilege everywhere by default, secrets never in plaintext,
  blast radius bounded.
- **Cost optimization** — spend matches business value. Not the cheapest option; the
  *most-justified* option.
- **Performance efficiency** — right tool for the workload, not the trendiest tool.
- **Operational excellence** — on-call humans can diagnose and recover at 3am with
  the docs alone.

The frameworks (AWS Well-Architected, Azure WAF, Google Cloud Architecture Framework)
all converge on these five. The wording differs; the trade-offs don't.

## The trade-off matrix

These are the conflicts you actually hit. They're not solvable — they're decidable.

| Tension | Strengthens | At the cost of |
|---|---|---|
| Multi-region active-active | Reliability | Cost, complexity, consistency model |
| Reserved instances / savings plans | Cost | Reliability (locked-in capacity if workload shifts) |
| Auto-scaling aggressively | Cost, performance | Reliability (cold starts, scale-in races) |
| Strong consistency | Correctness | Performance, regional reliability |
| Heavy WAF rules | Security | Performance, operational toil (false positives) |
| Wide SSO/SCIM | Operational | Security blast radius if IdP compromised |
| Managed services everywhere | Operational | Cost, vendor lock-in |
| Build-your-own platform | Cost (long term?), control | Operational (you own the toil) |

Pick the column you care about most, then **say out loud what you're giving up.** That
sentence is the ADR.

## Architecture Decision Records (ADRs) — the only doc that survives

ADRs are short, immutable, dated. New decision = new ADR superseding the old one.
Do not edit old ADRs except to mark them superseded.

### ADR template that actually gets filled in

```markdown
# ADR-0042: Use Azure Front Door instead of regional Application Gateways

**Status:** Accepted
**Date:** 2025-03-14
**Deciders:** platform team

## Context
We need global TLS termination + WAF for the public API. Latency from Asia is
currently 320ms p95 (origin in West Europe). Two regional Application Gateways
in WEU + SEA would add operational overhead and DNS-failover complexity.

## Decision
Adopt Azure Front Door (Premium) as the single global entry point. Origins
stay regional. WAF rules live on Front Door, not App Gateway.

## Consequences
+ Asia p95 drops to ~90ms (anycast + edge TLS).
+ One WAF policy to maintain, not N.
+ Origin shielding reduces backend load.
- Lock-in: WAF rules are Front Door-specific syntax.
- Cost: ~$330/mo Premium tier minimum.
- Single global control plane = single global blast radius.

## Alternatives considered
- Cloudflare in front of App Gateways — rejected, billing complexity + we
  already pay for Azure.
- Regional App Gateways with DNS GSLB — rejected, DNS-based failover SLA
  is poor and TTLs make rollback slow.
```

That's it. Five sections. Live in a `docs/adr/` folder, numbered sequentially,
never deleted.

### Where ADRs go wrong

- **Written after the fact** — they become marketing, not decisions.
- **Edited instead of superseded** — losing the history defeats the point.
- **Living only in someone's head / Confluence wiki** — keep them in the repo with
  the code they justify.
- **Too long** — if it doesn't fit on one page, you haven't decided yet.

## The five-pillar review (a 30-minute ritual)

When stuck or before a major launch, walk an architecture diagram and ask one
question per pillar:

1. **Reliability** — what is the single point of failure on this page right now?
2. **Security** — if this component is compromised, what does the attacker reach?
3. **Cost** — what's the most expensive component, and is the cost proportional
   to its business value?
4. **Performance** — where is the latency budget spent? (Estimate, then measure.)
5. **Ops** — if this breaks at 3am, what's the first command the on-call runs?

If you can't answer all five in plain language, the architecture isn't done — the
*diagram* might be done, but the design isn't.

## When "best practice" actually conflicts

Real cases where the textbook answer is wrong:

- **"Use managed databases"** vs **"Avoid lock-in"** — pick one. Both can't be
  load-bearing principles for the same team.
- **"Encrypt everything in transit"** vs **"Debug-friendly observability"** —
  mTLS makes packet captures useless; you need app-level logging instead.
- **"Multi-AZ by default"** vs **"Cost discipline for dev environments"** — dev
  doesn't need 99.99%. Document the exception.
- **"Immutable infrastructure"** vs **"Fast hotfixes"** — both are true *most* of
  the time. Define the break-glass procedure for the 1% case.

The mistake is pretending these don't conflict. The fix is naming the conflict
in an ADR and choosing.

## Gotchas

- **Pillars are not equally weighted.** A bank weighs reliability + security
  higher; a side project weighs cost higher. Make the weighting explicit per
  product, or every review devolves into "but security says…".
- **"Operational excellence" is the silent killer.** Teams happily over-engineer
  security and reliability while accumulating runbook debt. The pager doesn't
  lie — track on-call interrupts as the operational health metric.
- **The framework reviews from cloud vendors are sales tools.** Useful for
  finding quick wins. Not a substitute for an honest internal review.
- **ADRs without enforcement are wishes.** Reference them in PR templates,
  link them in module READMEs, and reject PRs that contradict accepted ADRs
  without superseding them.
- **Don't write ADRs for trivia.** "We use TypeScript" doesn't need an ADR.
  "We don't allow JavaScript in new services" does.

## When NOT to use the pillar framework

- **Throwaway prototype** — overhead exceeds value.
- **Tiny single-service apps** — read the docs, ship it, revisit if it grows.
- **Vendor-mandated architecture** — if the SaaS forces a shape on you, the
  ADR is "we accept the vendor's design" and that's the whole conversation.

## Related

- [`disaster-recovery-playbook.md`](./disaster-recovery-playbook.md) — pillar
  1 (reliability) in operational detail.
- [`multi-region-and-ha-patterns.md`](./multi-region-and-ha-patterns.md) — the
  topology choices behind the reliability/cost trade-off.
- [`architecture-documentation-that-survives.md`](./architecture-documentation-that-survives.md)
  — how ADRs fit alongside C4 diagrams and READMEs.
- [`../cloud-platform/terraform-on-azure.md`](../cloud-platform/terraform-on-azure.md)
  — IaC layout for a multi-pillar architecture.
