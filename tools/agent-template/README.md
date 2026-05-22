# Agent template — one-page skeleton

A drop-in folder skeleton for the
[portable agent workspace pattern](../../guides/portable-agent-workspace-pattern.md).

This page exists to be **copy-pasted as the starting point for a new agent**.
For the *why*, read the guide. For the *how to wire it up end-to-end*, see
[tutorials/bootstrapping-a-personal-agent-workspace.md](../../tutorials/bootstrapping-a-personal-agent-workspace.md).

---

## Skeleton tree

```
<AgentName>/
├── README.md
├── AGENT.md
├── SECURITY.md
├── .instructions.md
├── .gitignore
├── requirements.txt              # optional
│
├── prompts/
│   ├── system/default.md
│   ├── tasks/
│   │   ├── code-review.md
│   │   ├── research.md
│   │   └── problem-solving.md
│   └── chains/README.md
│
├── skills/
│   ├── active/                   # populate per-agent
│   └── templates/SKILL_TEMPLATE.md
│
├── tools/
│   ├── scripts/README.md
│   ├── automations/README.md
│   └── registry.md
│
├── mcp/
│   ├── mcp-config.template.json
│   └── templates/SERVER_TEMPLATE.md
│
├── context/
│   ├── user-profile.md
│   ├── conventions.md
│   └── knowledge-base/README.md
│
├── memory/
│   ├── journal/README.md
│   └── decisions/README.md
│
└── workspace/
    └── _templates/PROJECT_TEMPLATE.md
```

## Minimal file contents

The four files that *must* exist and contain real content from day one:

### `.gitignore`

```gitignore
.env
.env.local
**/mcp-config.json
**/.vscode/mcp.json
**/secrets.env
**/credentials.json
**/*.private.*
.venv/
__pycache__/
node_modules/
.DS_Store
```

### `mcp/mcp-config.template.json`

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "<absolute-path-to-this-workspace>"]
    }
  }
}
```

### `AGENT.md` (minimal)

```markdown
# Agent Identity

You are an AI partner working in this workspace. Read SECURITY.md and
context/user-profile.md before doing anything else. On every session
start, check the 3-5 most recent memory/journal/ entries.

Be direct. Don't fabricate. Match depth to task.
```

### `.instructions.md` (minimal)

```markdown
You have standing authority to write memory/ and workspace/ without asking.

- Journal (memory/journal/YYYY-MM-DD-topic.md) — sessions with 3+ actions.
- Decisions (memory/decisions/YYYY-MM-DD-title.md) — structural choices.
- Workspace docs (workspace/<project>/Title_Case.md) — research output.

See the full guide at:
https://github.com/Flarash/DivideByInfinity/blob/master/guides/persistent-memory-for-stateful-agents.md
```

## Folder-purpose cheatsheet

| Folder | One-line purpose |
|---|---|
| `prompts/` | Reusable templates the agent loads on demand |
| `skills/` | Bounded domain knowledge packs |
| `tools/` | Scripts and automations the agent can invoke |
| `mcp/` | MCP server configs (twin-file pattern) |
| `context/` | Meta-level info about the user (NOT domain knowledge) |
| `memory/` | Session chronology + decision log |
| `workspace/` | Active projects and produced knowledge |

## Boot order (what the agent reads first)

1. `README.md` (this skeleton has one — point at the structural guide)
2. `AGENT.md`
3. `SECURITY.md`
4. `context/user-profile.md`
5. `context/conventions.md` (if present)
6. 3-5 most recent `memory/journal/` entries
7. Then the user's prompt.

## Reference

- Structural guide: [portable-agent-workspace-pattern.md](../../guides/portable-agent-workspace-pattern.md)
- Secrets: [secrets-management-for-ai-agents.md](../../guides/secrets-management-for-ai-agents.md)
- Memory: [persistent-memory-for-stateful-agents.md](../../guides/persistent-memory-for-stateful-agents.md)
- End-to-end tutorial: [tutorials/bootstrapping-a-personal-agent-workspace.md](../../tutorials/bootstrapping-a-personal-agent-workspace.md)
