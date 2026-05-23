# Cloud architecture

Design patterns, decision frameworks, and documentation practices for cloud systems.
Less "how to click in the portal", more "how to think about the trade-offs".

## Guides

- [**Well-architected decisions in practice**](./well-architected-decisions.md) — the
  five pillars as a trade-off framework, the ADR template, and the 30-minute review
  ritual. Where "best practices" actually conflict and how to decide.
- [**Disaster recovery playbook**](./disaster-recovery-playbook.md) — RTO/RPO honesty,
  the four DR tiers, runbook anatomy, the quarterly drill, and the non-region gotchas
  (DNS TTLs, secrets replication, cross-region IAM) that turn 25-min runbooks into
  6-hour incidents.
- [**Multi-region and HA patterns**](./multi-region-and-ha-patterns.md) — the HA
  ladder, single-region multi-AZ as the real baseline, active-passive vs active-active,
  split-brain prevention, and when single-region is the right call.
- [**Architecture documentation that survives**](./architecture-documentation-that-survives.md)
  — why most docs rot, the four artifacts that actually survive (system overview, C4
  diagrams, service READMEs, ADRs), Mermaid C4 examples, and the anti-patterns that
  look like documentation.
