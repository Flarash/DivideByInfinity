# Tutorials

Step-by-step, hands-on walkthroughs. Each tutorial is self-contained and walks you from
zero to a working result.

## Index

### AI tooling & agent workflows

- [**Bootstrapping a personal agent workspace**](./bootstrapping-a-personal-agent-workspace.md) — end-to-end walkthrough from empty folder to a model-agnostic agent with autonomous memory, in ~30 minutes. The companion to the [portable agent workspace pattern](../guides/agent-workspaces/portable-agent-workspace-pattern.md) guide.
- [**Building a Copilot-Coding-Agent-driven repo**](./copilot-coding-agent-driven-repo.md) — set up a repo where scheduled workflows open issues, assign them to `copilot[bot]`, and produce focused PRs. Covers the PAT-based assignment trick, prompt structure, and iteration loop.
- [**Pinterest taste extraction with agent-mode image analysis**](./local-pinterest-taste-extraction-with-agent-mode.md) — turn a folder of pinned images into JSON sidecars, cluster, and synthesize a master taste profile. Locally-driven, with notes on why it beats the cloud-workflow version.
- [**Wiring local MCP servers (Spotify + AutoCAD)**](./wiring-local-mcp-servers-spotify-and-autocad.md) — practical setup for two real MCP servers across VS Code Copilot Chat and the Copilot CLI, plus the gotchas (different config files, wrapper-key mismatches).

## Conventions

- One tutorial per Markdown file, kebab-case filename.
- Start with a short "What you'll build" + "Prerequisites" section.
- Prefer copy-pasteable commands and small, complete code blocks.
