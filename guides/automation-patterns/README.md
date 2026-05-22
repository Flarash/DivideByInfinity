# Automation patterns

Repo-shape patterns for letting agents do real work — including work with
real-world side effects — while keeping humans in the loop where it matters.

## Guides

- [**PR-as-publish-gate**](./pr-as-publish-gate.md) — using merged PRs as the approval boundary for side-effecting automation (social posts, deployments, outbound email). File contracts, validation CI, rollback flow, OAuth gotchas across LinkedIn/Threads/Instagram.
- [**Scheduled "one-PR-per-run" audit workflows**](./scheduled-perf-audit-workflow.md) — daily/weekly perf, docs, or dependency audits where the audit list lives in the most-recently-merged PR body. Self-chaining, no external store.

## See also

- Tutorial: [Building a Copilot-Coding-Agent-driven repo](../../tutorials/copilot-coding-agent-driven-repo.md) — the foundation these patterns sit on top of.
- Tool: [`tools/copilot-coding-agent/`](../../tools/copilot-coding-agent/) — quirks of the agent that drives these workflows.
