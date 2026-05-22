# Guides

Reference-style guides, cheat sheets, and conceptual explainers. Less "do this in
order", more "look this up when you need it".

## Index

### Patterns for AI-driven repos

- [**Hybrid agent-workspace + `tools/` pattern**](./hybrid-agent-workspace-pattern.md) — a repo layout convention for projects mixing human-edited content with agent automation. Covers `workspace/`, `tools/`, `prompts/`, `memory/`, `context/`, and `AGENTS.md`.
- [**PR-as-publish-gate**](./pr-as-publish-gate.md) — using PRs as the approval boundary for side-effecting automation (social posts, deployments, outbound email). File contracts, validation CI, rollback flow, and OAuth gotchas across LinkedIn/Threads/Instagram.
- [**Scheduled "one-PR-per-run" audit workflows**](./scheduled-perf-audit-workflow.md) — pattern for daily/weekly perf, docs, or dependency audits where the audit list lives in the most-recently-merged PR's body. Self-chaining, no external store.

### Android / device modding

- [**Rooting Samsung Galaxy S25 Ultra (SM-S938B)** on One UI 7 via firmware downgrade + Magisk](./root-samsung-s25-ultra.md) — end-to-end walkthrough covering ADB setup, debloating, One UI 7 firmware downgrade to re-enable OEM Unlock, bootloader unlock, and Magisk root with Play Integrity bypass. International Exynos model only.

## Conventions

- One guide per Markdown file, kebab-case filename.
- Lead with a one-paragraph summary so readers know if it's what they need.
- Cross-link to relevant tutorials and tool pages where useful.
