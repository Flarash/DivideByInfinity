# Agent workspaces

Patterns for the **home folder of a personal AI agent** — where its system
prompt, secrets, memory, tools, and working notes live. Model-agnostic by
design (works for Claude, Copilot, ChatGPT, custom agents).

## Guides

- [**The portable agent workspace pattern**](./portable-agent-workspace-pattern.md) — the structural reference. `AGENT.md`, `SECURITY.md`, `prompts/`, `skills/`, `tools/`, `mcp/`, `context/`, `memory/`, `workspace/`. Read this first.
- [**Secrets management for personal AI agents**](./secrets-management-for-ai-agents.md) — twin-file pattern (`*.template.json` vs `*.json`), the `.gitignore` block, agent-side rules, rotation.
- [**Persistent memory for stateful agents**](./persistent-memory-for-stateful-agents.md) — `memory/journal/` + `memory/decisions/`, explicit triggers, greppable naming, cross-reference rules.
- [**Hybrid agent-workspace + `tools/` pattern**](./hybrid-agent-workspace-pattern.md) — repo layout for projects mixing human-edited content with agent automation. `workspace/`, `tools/`, `prompts/`, `memory/`, `context/`, `AGENTS.md`.

## See also

- Tutorial: [Bootstrapping a personal agent workspace](../../tutorials/bootstrapping-a-personal-agent-workspace.md) — end-to-end walkthrough using these patterns.
- Tool: [`tools/agent-template/`](../../tools/agent-template/) — copy-pasteable skeleton.
