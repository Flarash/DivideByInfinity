<div align="center">

# ➗ DivideByInfinity

### Field notes from the edge of **AI tooling, agents & integrations**

_Plain-Markdown tutorials, battle-tested patterns, and the gotchas nobody warned me about._

<br />

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-1f6feb?style=for-the-badge&labelColor=0d1117)](./LICENSE)
[![Tutorials](https://img.shields.io/badge/Tutorials-4-238636?style=for-the-badge&labelColor=0d1117)](./tutorials/)
[![Guides](https://img.shields.io/badge/Guides-18-8957e5?style=for-the-badge&labelColor=0d1117)](./guides/)
[![Tools](https://img.shields.io/badge/Tools-4-db6d28?style=for-the-badge&labelColor=0d1117)](./tools/)
[![Link check](https://github.com/Flarash/DivideByInfinity/actions/workflows/link-check.yml/badge.svg)](https://github.com/Flarash/DivideByInfinity/actions/workflows/link-check.yml)

<sub>No site to build. No newsletter to sign up to. Just files.</sub>

</div>

---

## 👋 Welcome

This repo is the **practical, lessons-learned** version of things I'm actively shipping — Copilot Coding Agents that open their own PRs, MCP servers that drive AutoCAD, OAuth pipelines that post to LinkedIn, a Galaxy S25 Ultra that boots into a pentest lab.

No site. No newsletter. Just Markdown.

---

## 🔥 Featured

<table>
<tr>
<td width="50%" valign="top">

### 📱 Root a Galaxy S25 Ultra

**[Root S25 Ultra on One UI 7 →](./guides/android/root-samsung-s25-ultra.md)**

Full-firmware Magisk patch flow on a current Samsung flagship. KnoxGuard, bootloader unlock, the firmware-version trap that bricks half the tutorials online, and how to keep banking apps working afterwards via Shamiko.

<sub>⚠️ Trips Knox. Voids warranty. Don't do this to a phone you can't replace.</sub>

</td>
<td width="50%" valign="top">

### 🛡️ Phone-as-Pentest-Rig

**[Phone pentest environment on a rooted phone →](./guides/android/android-pentest-environment-on-rooted-phone.md)**

Turn that same rooted S25 Ultra into a pocket security lab — Kali NetHunter chroot, Termux toolchain, OTG-Wi-Fi adapter for monitor mode, NetHunter Store apps, and the cheap accessory list that makes the whole rig actually portable.

<sub>🔒 Authorized targets only. Read the scope-and-consent section first.</sub>

</td>
</tr>
<tr>
<td width="50%" valign="top">

### 🤖 Copilot opens its own PRs

**[Copilot-Coding-Agent-driven repo →](./tutorials/copilot-coding-agent-driven-repo.md)**

Scheduled workflows that file issues, assign them to `copilot-swe-agent`, and ship focused PRs while you're asleep. Includes the PAT-assignment trick the official docs gloss over.

</td>
<td width="50%" valign="top">

### 🚦 PR-as-Publish-Gate

**[PR-as-publish-gate →](./guides/automation-patterns/pr-as-publish-gate.md)**

Let agents draft risky side effects — social posts, deploys, outbound email — but make a **merged PR the only thing that fires them**. Comes with the LinkedIn / Threads / Instagram OAuth gotchas I had to learn the painful way.

</td>
</tr>
</table>

---

## 🧭 Find what you want

| If you want to…                                              | Start here                                                                            |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| 🧩 Set up a model-agnostic personal agent workspace          | [Portable agent workspace pattern](./guides/agent-workspaces/portable-agent-workspace-pattern.md) → [Bootstrapping tutorial](./tutorials/bootstrapping-a-personal-agent-workspace.md) |
| 🔐 Stop leaking secrets from your agent configs              | [Secrets management for AI agents](./guides/agent-workspaces/secrets-management-for-ai-agents.md)      |
| 🧠 Give an agent memory that compounds across sessions       | [Persistent memory for stateful agents](./guides/agent-workspaces/persistent-memory-for-stateful-agents.md) |
| 🔌 Wire MCP servers into your editor & CLI                   | [Local MCP servers — Spotify + AutoCAD](./tutorials/wiring-local-mcp-servers-spotify-and-autocad.md) |
| 🎨 Extract a taste profile from a Pinterest folder           | [Pinterest taste extraction](./tutorials/local-pinterest-taste-extraction-with-agent-mode.md) |
| 🧠 Set up a local RAG pipeline over your own corpus          | [Local RAG pipeline](./guides/ai-pipelines/local-rag-pipeline.md)                                  |
| 🖼️ Run an LLM over hundreds of images without losing work    | [Image-analysis pipelines with LLMs](./guides/ai-pipelines/image-analysis-pipelines-with-llms.md)  |
| 🗂️ Lay out a repo that mixes humans + agents                 | [Hybrid agent-workspace pattern](./guides/agent-workspaces/hybrid-agent-workspace-pattern.md)          |
| ⏰ Run scheduled audits that write one PR per run            | [Scheduled audit workflows](./guides/automation-patterns/scheduled-perf-audit-workflow.md)                |
| ☁️ Lay out Terraform for Azure without state-file pain        | [Terraform on Azure](./guides/cloud-platform/terraform-on-azure.md)                                   |
| ☸️ Take an AKS cluster from "demo" to production              | [AKS production checklist](./guides/cloud-platform/aks-production-checklist.md)                       |
| 🔁 Pick between Azure DevOps, GitLab CI, and Jenkins          | [CI/CD pipeline patterns](./guides/cloud-platform/cicd-pipeline-patterns.md)                          |
| 📊 Wire up Prometheus + Grafana + logs without overpaying     | [Observability stack](./guides/cloud-platform/observability-stack.md)                                 |
| 🔧 Build an MCP server that an agent actually uses well       | [Writing an MCP server](./guides/agentic-tooling/writing-an-mcp-server.md)                            |
| 🧬 Ship Copilot CLI skills that activate at the right time    | [Building Copilot CLI skills](./guides/agentic-tooling/copilot-cli-skills.md)                         |
| 🕸️ Decide when to spawn sub-agents (and when NOT to)          | [Multi-agent orchestration patterns](./guides/agentic-tooling/multi-agent-orchestration.md)           |
| 🧪 Build an eval harness for your prompts and skills          | [Agent eval harness](./guides/agentic-tooling/agent-eval-harness.md)                                  |

---

## 🗺️ What's inside

<table>
<tr>
<td width="33%" valign="top">

### 📘 [Tutorials](./tutorials/)

**Hands-on, zero-to-working walkthroughs.**

Follow these top-to-bottom and you'll have something running by the end.

→ Best when you want to **build**.

</td>
<td width="33%" valign="top">

### 📗 [Guides](./guides/)

**Reference-style explainers and patterns.**

Reusable shapes for repos, workflows, and automations — the conceptual stuff.

→ Best when you want to **look up** or **steal a pattern**.

</td>
<td width="33%" valign="top">

### 🧰 [Tools](./tools/)

**Tool-indexed deep-dives.**

One folder per tool — Copilot Coding Agent, MCP servers, Smithery — with gotchas up front.

→ Best when you **know the tool**, want the tricks.

</td>
</tr>
</table>

---

## 🧭 How to navigate

- **Browse the folders** — every section has its own `README.md` index.
- **Use GitHub's file finder** — press <kbd>t</kbd> from anywhere in the repo to fuzzy-find a file by name.
- **Watch the repo** — click ⭐ / 👁️ at the top if you want to follow along.

---

## 📜 License

All written content is licensed under **[Creative Commons Attribution 4.0 International (CC BY 4.0)](./LICENSE)** — share, remix, and use it commercially, just give credit. Code snippets in tutorials are free to use under the same terms unless explicitly noted.

## 🐛 Found something wrong?

This is a personal notebook, but corrections and suggestions are very welcome. [**Open an issue**](https://github.com/Flarash/DivideByInfinity/issues/new) if you spot something off — typo, broken link, outdated gotcha, anything.

<div align="center">

<br />

<sub>Built in the open. Maintained in spare cycles. ➗∞</sub>

</div>
