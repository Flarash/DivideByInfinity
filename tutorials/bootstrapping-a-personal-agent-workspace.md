# Bootstrapping a personal agent workspace

End-to-end walkthrough: from empty folder to a working,
model-agnostic agent workspace you can point Copilot / Claude / GPT at.

The structural reference is the
[portable agent workspace pattern](../guides/agent-workspaces/portable-agent-workspace-pattern.md).
This tutorial assumes you've skimmed that.

---

## Goal

By the end of this tutorial, you'll have:

1. A folder structure that survives swapping AI vendors.
2. An `AGENT.md` that the agent reads on every start.
3. A working MCP config (without the live credentials in git).
4. A `memory/` folder the agent writes to automatically.
5. A reproducible setup any future-you can clone.

## Prerequisites

- An editor with MCP support (VS Code + GitHub Copilot, Claude Desktop,
  Cursor, etc.) **or** any chat UI you can point at a folder.
- Git installed.
- ~30 minutes.

## Step 1 — Create the skeleton

```powershell
# Pick a name. <AgentName> below is whatever you want to call it.
$Agent = "ResearchAgent"
New-Item -ItemType Directory -Path $Agent | Set-Location

# Top-level files
New-Item README.md, AGENT.md, SECURITY.md, .instructions.md, .gitignore | Out-Null

# Folder layout
$dirs = @(
  "prompts/system", "prompts/tasks", "prompts/chains",
  "skills/active", "skills/templates",
  "tools/scripts", "tools/automations",
  "mcp/templates",
  "context/knowledge-base",
  "memory/journal", "memory/decisions",
  "workspace/_templates"
)
$dirs | ForEach-Object { New-Item -ItemType Directory -Path $_ -Force | Out-Null }
```

(Bash equivalent is the obvious translation.)

## Step 2 — Lock down secrets *first*

Before writing anything else, fill in `.gitignore`. Doing this first prevents
the all-too-common "first commit accidentally includes a token" disaster.

```gitignore
# Secrets and live credentials
.env
.env.local
**/mcp-config.json
**/.vscode/mcp.json
**/secrets.env
**/credentials.json
**/*.private.*

# Local artifacts
.venv/
__pycache__/
node_modules/
.DS_Store

# Editor / OS
.vscode/settings.json
.idea/
```

Then write a minimal `SECURITY.md` — see
[secrets-management-for-ai-agents.md](../guides/agent-workspaces/secrets-management-for-ai-agents.md)
for the full version. The two rules that matter most:

- Live `mcp-config.json` is gitignored; commit only `mcp-config.template.json`.
- The agent must never output secret contents back to you.

## Step 3 — Write `AGENT.md`

This is the agent's identity. Keep it short — under 100 lines.

```markdown
# Agent Identity and Operating Guidelines

## Role
You are a research and development partner. You collaborate as a full
participant — propose alternatives, challenge assumptions, maintain
continuity across sessions through this workspace.

## Core Principles
1. **Intellectual partnership** — think with me, don't just validate.
2. **Continuity** — read memory/ and workspace/ before starting.
3. **Clarity** — direct, honest about uncertainty.
4. **Adaptability** — match depth to the task.

## Starting a Session
1. Read this file and context/user-profile.md.
2. Check the 3-5 most recent memory/journal/ entries.
3. Skim workspace/ for active projects.
4. Then accept the user's prompt.

## Constraints
- No fabrication. Say "I don't know" when you don't.
- Respect scope. Suggest expansions; don't act on them unilaterally.
- Read SECURITY.md before touching anything in mcp/.
```

## Step 4 — Write `.instructions.md`

This is the operating workflow — specifically the **autonomous writing** rules
that make memory work. Without these triggers, the agent never writes to
`memory/`.

```markdown
You are an AI partner working within a structured workspace.

## Onboarding (read in order)
1. AGENT.md
2. SECURITY.md
3. context/user-profile.md
4. context/conventions.md (if present)

## Autonomous Writing

You have standing authority to write to memory/ and workspace/ without
being asked.

### Journal — memory/journal/YYYY-MM-DD-topic.md
When: the session has 3+ substantive actions.
Skip when: quick Q&A or single-step tasks.
Format: Context / What Was Done / Insights / Open Questions / Next Steps.

### Decision — memory/decisions/YYYY-MM-DD-title.md
When: a structural or strategic choice was made.
Skip when: trivial or instantly reversible.
Format: Context / Decision / Alternatives / Rationale / Consequences /
Revisit If.

### Workspace docs — workspace/<project>/Title_Case.md
When: any response that contains research, analysis, or synthesis.
Format: clean Markdown, not a chat transcript.
```

The exact format details live in
[persistent-memory-for-stateful-agents.md](../guides/agent-workspaces/persistent-memory-for-stateful-agents.md).

## Step 5 — Write `context/user-profile.md`

This is the file you'll update most often. Bare minimum:

```markdown
# User Profile

## Identity
- Name: <first name only is fine>
- Background: <one paragraph>

## Goals
- Short term (this quarter): …
- Long term: …

## Preferences
- Output style: direct, terse, no preamble.
- Format: Markdown with headers, tables, Mermaid when useful.
- Math: LaTeX/KaTeX.
- Code: <your language preferences and conventions>.
```

Keep it under one screen. It's read on every session — bloat compounds.

## Step 6 — Set up MCP

Create `mcp/mcp-config.template.json`:

```json
{
  "$schema": "https://modelcontextprotocol.io/schemas/mcp.json",
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "C:\\path\\to\\your\\agent"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "<paste-here>" }
    }
  }
}
```

Then copy it and fill in real values:

```powershell
Copy-Item mcp/mcp-config.template.json mcp/mcp-config.json
# Edit mcp-config.json with your real credentials.
```

If you're using VS Code Copilot, do the same for `.vscode/mcp.template.json`
→ `.vscode/mcp.json`.

For server selection guidance, see
[tools/mcp-servers/](../tools/mcp-servers/) and
[tools/smithery/](../tools/smithery/).

## Step 7 — Seed `prompts/`

Three task prompts cover ~80% of recurring work:

- `prompts/tasks/code-review.md` — "Review the provided code for correctness,
  edge cases, and style. List blocking issues first, suggestions second."
- `prompts/tasks/research.md` — "Investigate <topic>. Produce a workspace doc
  with sections: Summary / Key Findings / Sources / Open Questions."
- `prompts/tasks/problem-solving.md` — "Walk through the problem step by step.
  State assumptions, propose 2-3 approaches, recommend one with rationale."

Keep them short. They're templates, not transcripts.

## Step 8 — First commit

```powershell
git init
git add .
git status   # verify mcp-config.json and .vscode/mcp.json are NOT staged
git commit -m "Initial agent workspace skeleton"
```

The `git status` check is non-negotiable. If you see a live config file
in the staged list, fix `.gitignore` before continuing.

## Step 9 — First session

Point your editor at the workspace. The first prompt to your agent should
be deliberately small:

> Read README.md, AGENT.md, SECURITY.md, and context/user-profile.md.
> Then summarize back to me your role and the three most important
> constraints you'll operate under. After that, wait for my next prompt.

If the agent reads the files and summarizes accurately, the wiring works.
If it hallucinates, recheck whether your MCP filesystem server actually has
access to this folder.

## Step 10 — The "second session" test

The real win shows up on session two, not session one. Close the chat,
open a new one, point it at the same folder, and ask:

> What were we working on?

A working setup answers with a coherent picture from `memory/journal/`.
A broken setup re-asks you for context.

If it's broken: check that the agent actually wrote a journal entry at the
end of session one. If not, the autonomous-writing triggers in
`.instructions.md` aren't being respected — try moving the autonomous
writing block into `AGENT.md` itself, where most models weight it more
heavily.

## Where to go from here

- Add scheduled work via Copilot Coding Agent — see
  [copilot-coding-agent-driven-repo.md](copilot-coding-agent-driven-repo.md).
- Wire local MCP servers (Spotify, AutoCAD, etc.) — see
  [wiring-local-mcp-servers-spotify-and-autocad.md](wiring-local-mcp-servers-spotify-and-autocad.md).
- Add a RAG layer for large reference corpora — see
  [local-rag-pipeline.md](../guides/ai-pipelines/local-rag-pipeline.md).
- Add image-analysis pipelines — see
  [image-analysis-pipelines-with-llms.md](../guides/ai-pipelines/image-analysis-pipelines-with-llms.md).
