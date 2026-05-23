<div align="center">

# ➗ DivideByInfinity

### Field notes from the edge of **AI tooling, agents & integrations**

_Plain-Markdown tutorials, battle-tested patterns, and the gotchas nobody warned me about._

<br />

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-1f6feb?style=for-the-badge&labelColor=0d1117)](./LICENSE)
[![Tutorials](https://img.shields.io/badge/Tutorials-4-238636?style=for-the-badge&labelColor=0d1117)](./tutorials/)
[![Guides](https://img.shields.io/badge/Guides-42-8957e5?style=for-the-badge&labelColor=0d1117)](./guides/)
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
| 🏛️ Decide between five well-architected pillars in practice  | [Well-architected decisions](./guides/cloud-architecture/well-architected-decisions.md)               |
| 🆘 Write a DR playbook that survives the quarterly drill      | [Disaster recovery playbook](./guides/cloud-architecture/disaster-recovery-playbook.md)               |
| 🌍 Decide between multi-AZ, active-passive, and active-active | [Multi-region and HA patterns](./guides/cloud-architecture/multi-region-and-ha-patterns.md)           |
| 📐 Write architecture docs that don't rot in six months       | [Architecture documentation that survives](./guides/cloud-architecture/architecture-documentation-that-survives.md) |
| 🧱 Decide between Terraform, Pulumi, and CloudFormation       | [Terraform vs Pulumi vs CloudFormation](./guides/iac/terraform-vs-pulumi-vs-cloudformation.md)        |
| 🧩 Design Terraform modules that don't become a maintenance trap | [Terraform module design](./guides/iac/terraform-module-design.md)                                |
| 🗄️ Manage Terraform state without losing prod                 | [Terraform state management](./guides/iac/terraform-state-management.md)                              |
| 🛡️ Catch bad infra in CI and detect drift after apply         | [Policy as code and drift detection](./guides/iac/policy-as-code-and-drift-detection.md)              |
| 🚀 Pick a deployment strategy (rolling/blue-green/canary/flags) | [Deployment strategies](./guides/automation-patterns/deployment-strategies.md)                       |
| 🎚️ Use feature flags without drowning in flag debt            | [Progressive delivery with feature flags](./guides/automation-patterns/progressive-delivery-with-flags.md) |
| 🔁 Promote one artifact across dev → staging → prod safely     | [Multi-environment promotion](./guides/automation-patterns/multi-environment-promotion.md)            |
| 📦 Decide between Helm, Kustomize, and raw YAML                | [Helm vs Kustomize vs raw YAML](./guides/kubernetes/helm-vs-kustomize-vs-raw-yaml.md)                 |
| 🕸️ Decide if a service mesh is worth the operational tax       | [Service mesh — when and why](./guides/kubernetes/service-mesh-when-and-why.md)                       |
| 🔄 Run GitOps with Argo CD or Flux without app-of-apps spaghetti | [GitOps with Argo CD and Flux](./guides/kubernetes/gitops-with-argocd-and-flux.md)                   |
| 📈 Combine HPA, VPA, KEDA, and Cluster Autoscaler/Karpenter     | [Pod autoscaling deep dive](./guides/kubernetes/pod-autoscaling-deep-dive.md)                         |
| 🔑 Apply least-privilege IAM without the wildcard trap          | [IAM least-privilege in practice](./guides/security/iam-least-privilege-in-practice.md)               |
| 🗝️ Store secrets correctly across cloud workloads               | [Secrets management for cloud workloads](./guides/security/secrets-management-for-cloud-workloads.md) |
| 🌐 Stack network security layers (VPC / NSG / private endpoints / WAF) | [Network security layers](./guides/security/network-security-layers.md)                          |
| ⚠️ Run a threat-modeling exercise in 30 minutes                  | [Threat modeling without the bureaucracy](./guides/security/threat-modeling-without-the-bureaucracy.md) |
| 💵 Right-size cloud resources without breaking prod              | [Right-sizing and savings plans](./guides/cost-optimization/right-sizing-and-savings-plans.md)        |
| 🏷️ Make cloud spend visible per team with FinOps + tags          | [FinOps, tagging, and showback](./guides/cost-optimization/finops-tagging-and-showback.md)            |
| 🔎 Cut Elasticsearch cost without losing search                  | [Elasticsearch cost and performance](./guides/cost-optimization/elasticsearch-cost-and-performance.md) |
| 📡 Build a personal tech radar (Adopt/Trial/Assess/Hold)         | [Building a personal tech radar](./guides/tech-radar/building-a-personal-tech-radar.md)               |
| 🌊 Stay current without drowning in firehose feeds              | [Keeping current without burning out](./guides/tech-radar/keeping-current-without-burning-out.md)     |

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
