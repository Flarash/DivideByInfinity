# Guides

Reference-style guides, cheat sheets, and conceptual explainers. Less "do this in
order", more "look this up when you need it".

## Index

### Personal agent workspaces

- [**The portable agent workspace pattern**](./portable-agent-workspace-pattern.md) — model-agnostic folder layout (`AGENT.md`, `SECURITY.md`, `prompts/`, `skills/`, `tools/`, `mcp/`, `context/`, `memory/`, `workspace/`) that survives swapping AI vendors. The structural reference.
- [**Secrets management for personal AI agents**](./secrets-management-for-ai-agents.md) — the twin-file pattern (`*.template.json` vs `*.json`), the `.gitignore` block that actually works, agent-side "never leak secrets" rules, and rotation steps.
- [**Persistent memory for stateful agents**](./persistent-memory-for-stateful-agents.md) — `memory/journal/` + `memory/decisions/` with explicit triggers, naming for greppability, and the cross-reference rule that makes it compound over months.

### Retrieval, embeddings & vision pipelines

- [**Setting up a local RAG pipeline you can actually trust**](./local-rag-pipeline.md) — ChromaDB + sentence-transformers (`all-MiniLM-L6-v2`) on disk, sidecar-first corpus shape, embedding-model pinning, UMAP+HDBSCAN clustering, and turning a corpus into a synthesizable profile.
- [**Image-analysis pipelines with LLMs — the reality**](./image-analysis-pipelines-with-llms.md) — model tier selection, per-model rate limits, checkpointing every 25 items, Pillow downscale economics, manifest-based resumption, and the 12-point pre-flight checklist for unattended runs.

### Patterns for AI-driven repos

- [**Hybrid agent-workspace + `tools/` pattern**](./hybrid-agent-workspace-pattern.md) — a repo layout convention for projects mixing human-edited content with agent automation. Covers `workspace/`, `tools/`, `prompts/`, `memory/`, `context/`, and `AGENTS.md`.
- [**PR-as-publish-gate**](./pr-as-publish-gate.md) — using PRs as the approval boundary for side-effecting automation (social posts, deployments, outbound email). File contracts, validation CI, rollback flow, and OAuth gotchas across LinkedIn/Threads/Instagram.
- [**Scheduled "one-PR-per-run" audit workflows**](./scheduled-perf-audit-workflow.md) — pattern for daily/weekly perf, docs, or dependency audits where the audit list lives in the most-recently-merged PR's body. Self-chaining, no external store.

### Android / device modding

- [**Rooting Samsung Galaxy S25 Ultra (SM-S938B)** on One UI 7 via firmware downgrade + Magisk](./root-samsung-s25-ultra.md) — end-to-end walkthrough covering ADB setup, debloating, One UI 7 firmware downgrade to re-enable OEM Unlock, bootloader unlock, and Magisk root with Play Integrity bypass. International Exynos model only.

### Offensive security / mobile pentest

- [**Building a phone-based pentest environment on a rooted Android**](./android-pentest-environment-on-rooted-phone.md) — Termux + Kali (`proot-distro` chroot) + XFCE over VNC + Frida, end-to-end. Includes the headless-drive trick (`adb shell run-as com.termux`), the `proot-distro` v5 breaking change, why `frida-tools` won't `pip install` in Termux, and the gotchas you'd otherwise rediscover the hard way.

## Conventions

- One guide per Markdown file, kebab-case filename.
- Lead with a one-paragraph summary so readers know if it's what they need.
- Cross-link to relevant tutorials and tool pages where useful.
