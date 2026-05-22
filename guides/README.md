# Guides

Reference-style guides, cheat sheets, and conceptual explainers. Less "do this in
order", more "look this up when you need it".

Guides are grouped into topic folders. Each folder has its own `README.md` index.

## 📂 Topics

| Folder | What's in it |
|---|---|
| 🧩 [**`agent-workspaces/`**](./agent-workspaces/) | How to lay out a personal agent's home folder. Portable structure, secrets, memory, hybrid human+agent repos. |
| 🧠 [**`ai-pipelines/`**](./ai-pipelines/) | Retrieval, embeddings, and vision pipelines you run locally. RAG with ChromaDB, batch LLM image tagging. |
| ⚙️ [**`automation-patterns/`**](./automation-patterns/) | Patterns for repos where agents do work. PR-as-publish-gate, scheduled one-PR-per-run audits. |
| ☁️ [**`cloud-platform/`**](./cloud-platform/) | Platform-layer patterns for cloud infra. Terraform on Azure, AKS production checklist, CI/CD across the big three, Prometheus + Grafana + ELK observability. |
| 📱 [**`android/`**](./android/) | Phone-side adventures. Rooting Samsung Galaxy S25 Ultra, turning a rooted phone into a Termux+Kali+Frida pentest rig. |

## 🗺️ Full guide index

### 🧩 Agent workspaces — [`agent-workspaces/`](./agent-workspaces/)

- [**The portable agent workspace pattern**](./agent-workspaces/portable-agent-workspace-pattern.md) — model-agnostic folder layout (`AGENT.md`, `SECURITY.md`, `prompts/`, `skills/`, `tools/`, `mcp/`, `context/`, `memory/`, `workspace/`) that survives swapping AI vendors. The structural reference.
- [**Secrets management for personal AI agents**](./agent-workspaces/secrets-management-for-ai-agents.md) — the twin-file pattern (`*.template.json` vs `*.json`), the `.gitignore` block that actually works, agent-side "never leak secrets" rules, and rotation steps.
- [**Persistent memory for stateful agents**](./agent-workspaces/persistent-memory-for-stateful-agents.md) — `memory/journal/` + `memory/decisions/` with explicit triggers, naming for greppability, and the cross-reference rule that makes it compound over months.
- [**Hybrid agent-workspace + `tools/` pattern**](./agent-workspaces/hybrid-agent-workspace-pattern.md) — a repo layout convention for projects mixing human-edited content with agent automation. Covers `workspace/`, `tools/`, `prompts/`, `memory/`, `context/`, and `AGENTS.md`.

### 🧠 AI pipelines — [`ai-pipelines/`](./ai-pipelines/)

- [**Setting up a local RAG pipeline you can actually trust**](./ai-pipelines/local-rag-pipeline.md) — ChromaDB + sentence-transformers (`all-MiniLM-L6-v2`) on disk, sidecar-first corpus shape, embedding-model pinning, UMAP+HDBSCAN clustering, and turning a corpus into a synthesizable profile.
- [**Image-analysis pipelines with LLMs — the reality**](./ai-pipelines/image-analysis-pipelines-with-llms.md) — model tier selection, per-model rate limits, checkpointing every 25 items, Pillow downscale economics, manifest-based resumption, and the 12-point pre-flight checklist for unattended runs.

### ⚙️ Automation patterns — [`automation-patterns/`](./automation-patterns/)

- [**PR-as-publish-gate**](./automation-patterns/pr-as-publish-gate.md) — using PRs as the approval boundary for side-effecting automation (social posts, deployments, outbound email). File contracts, validation CI, rollback flow, and OAuth gotchas across LinkedIn/Threads/Instagram.
- [**Scheduled "one-PR-per-run" audit workflows**](./automation-patterns/scheduled-perf-audit-workflow.md) — pattern for daily/weekly perf, docs, or dependency audits where the audit list lives in the most-recently-merged PR's body. Self-chaining, no external store.

### ☁️ Cloud platform — [`cloud-platform/`](./cloud-platform/)

- [**Terraform on Azure — layout, state, and the gotchas**](./cloud-platform/terraform-on-azure.md) — folders-not-workspaces, remote state in Azure Storage, module rules, secrets via Key Vault, Workload Identity for CI auth, drift detection, and the gotcha collection.
- [**Kubernetes (AKS) production checklist**](./cloud-platform/aks-production-checklist.md) — multi-zone topology, system/user node pools, Azure CNI Overlay vs flat, NGINX + cert-manager, Key Vault CSI driver, PDBs, Velero-based DR, the first-hour checklist.
- [**CI/CD pipeline patterns — Azure DevOps, GitLab CI, Jenkins**](./cloud-platform/cicd-pipeline-patterns.md) — universal pipeline shape, per-platform strengths/gotchas, build-once-deploy-many, per-PR ephemeral envs, change-only deploys, picking one for a new org.
- [**Observability stack: Prometheus + Grafana + ELK**](./cloud-platform/observability-stack.md) — metrics/logs/traces split, Prometheus label cardinality trap, ELK vs Loki trade-off, dashboards-as-code, alert quality rules, where Splunk fits, DR for the observability stack itself.

### 📱 Android — [`android/`](./android/)

- [**Rooting Samsung Galaxy S25 Ultra (SM-S938B)** on One UI 7 via firmware downgrade + Magisk](./android/root-samsung-s25-ultra.md) — end-to-end walkthrough covering ADB setup, debloating, One UI 7 firmware downgrade to re-enable OEM Unlock, bootloader unlock, and Magisk root with Play Integrity bypass. International Exynos model only.
- [**Building a phone-based pentest environment on a rooted Android**](./android/android-pentest-environment-on-rooted-phone.md) — Termux + Kali (`proot-distro` chroot) + XFCE over VNC + Frida, end-to-end. Includes the headless-drive trick (`adb shell run-as com.termux`), the `proot-distro` v5 breaking change, why `frida-tools` won't `pip install` in Termux, and the gotchas you'd otherwise rediscover the hard way.

## Conventions

- One guide per Markdown file, kebab-case filename.
- Lead with a one-paragraph summary so readers know if it's what they need.
- Cross-link to relevant tutorials and tool pages where useful.
- New topic? Add a subfolder with its own `README.md` index, then add a row to the topics table above.
