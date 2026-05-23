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
| 🤖 [**`agentic-tooling/`**](./agentic-tooling/) | Patterns for building MCP servers, Copilot CLI skills, multi-agent orchestration, and eval harnesses. |
| 🏛️ [**`cloud-architecture/`**](./cloud-architecture/) | Design and architecture for cloud systems. Well-architected trade-offs, DR playbooks, multi-region/HA patterns, and architecture docs that survive. |
| 🧱 [**`iac/`**](./iac/) | Infrastructure as Code patterns. Terraform vs Pulumi vs CloudFormation, module design, state management, policy-as-code, and drift detection. |
| ⎈ [**`kubernetes/`**](./kubernetes/) | Kubernetes-specific deep dives. Helm vs Kustomize, service mesh decisions, GitOps with Argo CD / Flux, and pod autoscaling (HPA / VPA / KEDA / Karpenter). |
| 🔐 [**`security/`**](./security/) | Cloud security patterns. IAM least-privilege, secrets management, network defense in depth, and threat modeling without the bureaucracy. |

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
- [**Deployment strategies**](./automation-patterns/deployment-strategies.md) — recreate / rolling / blue-green / canary / feature flags compared, per-platform implementation, and why the rollback path is the other half most teams skip.
- [**Progressive delivery with feature flags**](./automation-patterns/progressive-delivery-with-flags.md) — flag categories (release/experiment/permission/kill-switch/config), OpenFeature, kill-switches that default ON, the flag-debt problem, and audit-trail requirements.
- [**Multi-environment promotion**](./automation-patterns/multi-environment-promotion.md) — build-once-promote-many, config injection patterns, GitOps vs pipeline-as-code promotion, environment parity, ephemeral per-PR environments, and the multi-step schema migration discipline.

### ☁️ Cloud platform — [`cloud-platform/`](./cloud-platform/)

- [**Terraform on Azure — layout, state, and the gotchas**](./cloud-platform/terraform-on-azure.md) — folders-not-workspaces, remote state in Azure Storage, module rules, secrets via Key Vault, Workload Identity for CI auth, drift detection, and the gotcha collection.
- [**Kubernetes (AKS) production checklist**](./cloud-platform/aks-production-checklist.md) — multi-zone topology, system/user node pools, Azure CNI Overlay vs flat, NGINX + cert-manager, Key Vault CSI driver, PDBs, Velero-based DR, the first-hour checklist.
- [**CI/CD pipeline patterns — Azure DevOps, GitLab CI, Jenkins**](./cloud-platform/cicd-pipeline-patterns.md) — universal pipeline shape, per-platform strengths/gotchas, build-once-deploy-many, per-PR ephemeral envs, change-only deploys, picking one for a new org.
- [**Observability stack: Prometheus + Grafana + ELK**](./cloud-platform/observability-stack.md) — metrics/logs/traces split, Prometheus label cardinality trap, ELK vs Loki trade-off, dashboards-as-code, alert quality rules, where Splunk fits, DR for the observability stack itself.

### 📱 Android — [`android/`](./android/)

- [**Rooting Samsung Galaxy S25 Ultra (SM-S938B)** on One UI 7 via firmware downgrade + Magisk](./android/root-samsung-s25-ultra.md) — end-to-end walkthrough covering ADB setup, debloating, One UI 7 firmware downgrade to re-enable OEM Unlock, bootloader unlock, and Magisk root with Play Integrity bypass. International Exynos model only.
- [**Building a phone-based pentest environment on a rooted Android**](./android/android-pentest-environment-on-rooted-phone.md) — Termux + Kali (`proot-distro` chroot) + XFCE over VNC + Frida, end-to-end. Includes the headless-drive trick (`adb shell run-as com.termux`), the `proot-distro` v5 breaking change, why `frida-tools` won't `pip install` in Termux, and the gotchas you'd otherwise rediscover the hard way.

### 🤖 Agentic tooling — [`agentic-tooling/`](./agentic-tooling/)

- [**Writing your own MCP server**](./agentic-tooling/writing-an-mcp-server.md) — stdio vs HTTP transport, schema design rules, the "description IS the prompt" lesson, logging-to-stderr trap, and the gotchas (schema drift, name collisions, timeouts, token-stuffing) that ship with every first MCP server.
- [**Building Copilot CLI skills**](./agentic-tooling/copilot-cli-skills.md) — `SKILL.md` anatomy, description-as-activation-trigger, procedural vs declarative bodies, one-skill-per-intent scoping, and the drift problems that hit every long-lived skill folder.
- [**Multi-agent orchestration patterns**](./agentic-tooling/multi-agent-orchestration.md) — when to delegate vs do it yourself, parallel-only-if-independent rule, sub-agent prompt anatomy, owner-per-scope discipline, cost reality, and the anti-patterns ("planner + implementer" splits, speculative sub-agents) to avoid.
- [**Building an agent eval harness**](./agentic-tooling/agent-eval-harness.md) — three eval layers (prompt/tool/e2e), property-based golden cases, LLM-as-judge biases (self-preference, verbose-wins, order), cost+latency tracking, eval drift, and when NOT to build a harness at all.

### 🏛️ Cloud architecture — [`cloud-architecture/`](./cloud-architecture/)

- [**Well-architected decisions in practice**](./cloud-architecture/well-architected-decisions.md) — the five pillars as a forced trade-off framework, the ADR template, the 30-minute review ritual, and where "best practices" actually conflict.
- [**Disaster recovery playbook**](./cloud-architecture/disaster-recovery-playbook.md) — RTO/RPO honesty, the four DR tiers, runbook anatomy, the quarterly drill, and the non-region gotchas (DNS TTLs, secrets replication, cross-region IAM).
- [**Multi-region and HA patterns**](./cloud-architecture/multi-region-and-ha-patterns.md) — the HA ladder, single-region multi-AZ as the real baseline, active-passive vs active-active, split-brain prevention, and when single-region is the right call.
- [**Architecture documentation that survives**](./cloud-architecture/architecture-documentation-that-survives.md) — why most docs rot, the four artifacts that survive (system overview, C4 diagrams, service READMEs, ADRs), Mermaid C4 examples, and the anti-patterns that look like documentation.

### 🧱 Infrastructure as Code — [`iac/`](./iac/)

- [**Terraform vs Pulumi vs CloudFormation**](./iac/terraform-vs-pulumi-vs-cloudformation.md) — honest comparison across coverage, state, testing, ergonomics, and lock-in. The "which one for which team" decision matrix plus the CDK conversation.
- [**Terraform module design**](./iac/terraform-module-design.md) — root vs reusable modules, the thin-wrapper anti-pattern, input/output discipline, semver, registry vs in-repo, and testing strategies (validate / tflint / Terratest / `terraform test`).
- [**Terraform state management**](./iac/terraform-state-management.md) — remote backends, locking, the workspace-vs-folder debate, state surgery (`mv`/`rm`/`import`/`moved`), drift, and recovering from corruption.
- [**Policy as code and drift detection**](./iac/policy-as-code-and-drift-detection.md) — Checkov/tfsec/OPA/Sentinel comparison, where policies run in the pipeline, a concrete Checkov+Conftest setup, scheduled drift detection, and the exception process.

### ⎈ Kubernetes — [`kubernetes/`](./kubernetes/)

- [**Helm vs Kustomize vs raw YAML**](./kubernetes/helm-vs-kustomize-vs-raw-yaml.md) — packaging vs layering, the two-question decision matrix, Helm chart shape that works, Kustomize patch flavors, the values.yaml sprawl problem, and when raw YAML is correct.
- [**Service mesh — when and why (mostly, when not)**](./kubernetes/service-mesh-when-and-why.md) — what a mesh actually does, when adoption is justified, Istio vs Linkerd vs Cilium vs Consul, sidecar vs sidecarless (ambient/eBPF), the operational tax, and the pragmatic ramp-up plan.
- [**GitOps with Argo CD and Flux**](./kubernetes/gitops-with-argocd-and-flux.md) — the four GitOps requirements, Argo's Application/ApplicationSet model, Flux's modular controllers, repo layout, sync waves, drift handling, multi-cluster topologies, secret management, bootstrap, and the app-of-apps anti-pattern.
- [**Pod autoscaling deep dive**](./kubernetes/pod-autoscaling-deep-dive.md) — HPA, VPA, KEDA, Cluster Autoscaler, and Karpenter — what each scales, how they interact, the metrics-server gotcha, custom metrics via prometheus-adapter, and the "scaled to zero and nothing wakes up" failure mode.

### 🔐 Security — [`security/`](./security/)

- [**IAM least-privilege in practice**](./security/iam-least-privilege-in-practice.md) — wildcards as the canonical sin, RBAC vs ABAC, modeling across AWS/Azure/GCP, workload identity for services, role assumption chains, the quarterly audit cadence, and break-glass account discipline.
- [**Secrets management for cloud workloads**](./security/secrets-management-for-cloud-workloads.md) — Key Vault / Secrets Manager / Vault comparison, workload identity as the gateway, the Secrets Store CSI driver pattern for Kubernetes, rotation patterns, and dynamic secrets.
- [**Network security layers**](./security/network-security-layers.md) — VPC/VNet topology, subnet segmentation, security groups vs NACLs, private endpoints, WAF, DDoS, egress filtering, east-west controls, and modern bastion alternatives.
- [**Threat modeling without the bureaucracy**](./security/threat-modeling-without-the-bureaucracy.md) — the four-question framework, STRIDE as a lens, the 30-minute team exercise, documenting the result, and the specific patterns that surface in 80% of models.

## Conventions

- One guide per Markdown file, kebab-case filename.
- Lead with a one-paragraph summary so readers know if it's what they need.
- Cross-link to relevant tutorials and tool pages where useful.
- New topic? Add a subfolder with its own `README.md` index, then add a row to the topics table above.
