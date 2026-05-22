<div align="center">

# ➗ DivideByInfinity

### Field notes from the edge of **AI tooling, agents & integrations**

_Plain-Markdown tutorials, battle-tested patterns, and the gotchas nobody warned me about._

<br />

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-1f6feb?style=for-the-badge&labelColor=0d1117)](./LICENSE)
[![Tutorials](https://img.shields.io/badge/Tutorials-3-238636?style=for-the-badge&labelColor=0d1117)](./tutorials/)
[![Guides](https://img.shields.io/badge/Guides-4-8957e5?style=for-the-badge&labelColor=0d1117)](./guides/)
[![Tools](https://img.shields.io/badge/Tools-3-db6d28?style=for-the-badge&labelColor=0d1117)](./tools/)

<sub>No site to build. No newsletter to sign up to. Just files.</sub>

</div>

---

## 👋 Welcome

Welcome. This repo is where I publish the **practical, lessons-learned** version of things I'm actively shipping — Copilot Coding Agents that open their own PRs, MCP servers that drive AutoCAD, OAuth pipelines that post to LinkedIn, and the occasional unrelated tangent like rooting a Galaxy S25 Ultra.

If you came here for one specific thing, jump straight to it:

| If you want to…                                              | Start here                                                                            |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| 🤖 Build a repo where Copilot opens its own PRs              | [Copilot-Coding-Agent-driven repo](./tutorials/copilot-coding-agent-driven-repo.md)   |
| 🔌 Wire MCP servers into your editor & CLI                   | [Local MCP servers — Spotify + AutoCAD](./tutorials/wiring-local-mcp-servers-spotify-and-autocad.md) |
| 🎨 Extract a taste profile from a Pinterest folder           | [Pinterest taste extraction](./tutorials/local-pinterest-taste-extraction-with-agent-mode.md) |
| 🚦 Use PRs as the approval gate for risky automation         | [PR-as-publish-gate](./guides/pr-as-publish-gate.md)                                  |
| 🗂️ Lay out a repo that mixes humans + agents                 | [Hybrid agent-workspace pattern](./guides/hybrid-agent-workspace-pattern.md)          |
| ⏰ Run scheduled audits that write one PR per run            | [Scheduled audit workflows](./guides/scheduled-perf-audit-workflow.md)                |
| 📱 Root a Galaxy S25 Ultra (because why not)                 | [Root S25 Ultra on One UI 7](./guides/root-samsung-s25-ultra.md)                      |

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

## ✨ Highlights

> **🤖 [Copilot-Coding-Agent-driven repos](./tutorials/copilot-coding-agent-driven-repo.md)** — set up scheduled workflows that open issues, assign them to `copilot-swe-agent`, and produce focused PRs. Includes the PAT-assignment trick that the docs gloss over.

> **🔌 [Wiring local MCP servers](./tutorials/wiring-local-mcp-servers-spotify-and-autocad.md)** — Spotify and AutoCAD, wired into both VS Code Copilot Chat *and* the Copilot CLI. Plus the wrapper-key mismatch that silently breaks half the tutorials online.

> **🚦 [PR-as-publish-gate](./guides/pr-as-publish-gate.md)** — let agents draft side-effecting actions (social posts, deploys, outbound email), but make a merged PR the only thing that fires them. Comes with the LinkedIn/Threads/Instagram OAuth gotchas I had to learn the hard way.

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
