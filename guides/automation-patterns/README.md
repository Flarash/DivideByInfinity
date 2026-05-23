# Automation patterns

Repo-shape patterns for letting agents do real work — including work with
real-world side effects — while keeping humans in the loop where it matters.

## Guides

- [**PR-as-publish-gate**](./pr-as-publish-gate.md) — using merged PRs as the approval boundary for side-effecting automation (social posts, deployments, outbound email). File contracts, validation CI, rollback flow, OAuth gotchas across LinkedIn/Threads/Instagram.
- [**Scheduled "one-PR-per-run" audit workflows**](./scheduled-perf-audit-workflow.md) — daily/weekly perf, docs, or dependency audits where the audit list lives in the most-recently-merged PR body. Self-chaining, no external store.
- [**Deployment strategies**](./deployment-strategies.md) — recreate / rolling / blue-green / canary / feature flags. The strategies at a glance, per-platform implementation (Kubernetes, App Service, Lambda), and why the rollback path is the other half of the work most teams skip.
- [**Progressive delivery with feature flags**](./progressive-delivery-with-flags.md) — flag categories (release, experiment, permission, kill-switch, config), OpenFeature + provider landscape, kill-switches that default ON, the flag-debt problem, and audit-trail requirements.
- [**Multi-environment promotion**](./multi-environment-promotion.md) — build-once-promote-many, config injection patterns, GitOps vs pipeline-as-code promotion, environment parity, ephemeral per-PR environments, and the asymmetric problem of database migrations.

## See also

- Tutorial: [Building a Copilot-Coding-Agent-driven repo](../../tutorials/copilot-coding-agent-driven-repo.md) — the foundation these patterns sit on top of.
- Tool: [`tools/copilot-coding-agent/`](../../tools/copilot-coding-agent/) — quirks of the agent that drives these workflows.
