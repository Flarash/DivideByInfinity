# The portable agent workspace pattern

A folder layout for personal AI agents that **survives model swaps**, **survives tool swaps**, and **survives you forgetting what you were doing three weeks ago**.

> This is the structural pattern. The companion tutorial
> [tutorials/bootstrapping-a-personal-agent-workspace.md](../tutorials/bootstrapping-a-personal-agent-workspace.md)
> walks through standing one up from scratch.

---

## The problem this solves

You start a project with Claude. Then GPT-5 ships something you want to try.
Then Gemini gets a feature nobody else has. Then a new local model is suddenly
worth running. Each platform invents its own format for memory, prompts,
"projects," "spaces," "GPTs," etc.

If your agent's context lives inside one vendor's UI, you're hostage. Every
switch is a from-scratch restart.

The fix is unglamorous: **keep everything in plain Markdown and JSON in a
folder you own**. Any model that can read files can use it.

## The layout

```
<AgentName>/
├── README.md              ← Onboarding for the agent (read this first)
├── AGENT.md               ← Identity, role, operating principles
├── SECURITY.md            ← Credential and secrets rules
├── .instructions.md       ← Operating workflow + autonomous-writing triggers
├── .gitignore             ← Excludes credentials and machine-local artifacts
│
├── prompts/               ← Reusable prompt templates
│   ├── system/            ← Persona / constraint prompts
│   ├── tasks/             ← Task-specific templates (research, code-review, …)
│   └── chains/            ← Multi-step workflows
│
├── skills/                ← Domain knowledge packs
│   ├── active/            ← Skills currently loaded
│   └── templates/         ← Skill skeletons
│
├── tools/                 ← Scripts, automations, registry
│   ├── scripts/
│   ├── automations/
│   └── registry.md        ← Index of available tools
│
├── mcp/                   ← MCP server configs
│   ├── mcp-config.template.json   ← Safe to commit (placeholders only)
│   ├── mcp-config.json            ← Gitignored; live credentials
│   └── templates/
│
├── context/               ← Persistent meta-context
│   ├── user-profile.md    ← Who the user is, preferences, goals
│   ├── conventions.md     ← Style preferences
│   └── knowledge-base/    ← Lightweight references (NOT domain knowledge)
│
├── memory/                ← Session logs and decisions
│   ├── journal/           ← YYYY-MM-DD-topic.md session notes
│   └── decisions/         ← YYYY-MM-DD-title.md decision records
│
└── workspace/             ← Active projects and outputs
    └── _templates/        ← Project skeletons
```

## Why each folder exists (and what *not* to put in it)

| Folder | Purpose | Common mistake |
|---|---|---|
| `prompts/` | Reusable templates the agent loads on demand | Pasting one-off requests here |
| `skills/` | Bounded domain packs (e.g. "Latex for physics," "API of library X") | Mixing skills with prompts |
| `tools/` | Executable scripts and adapters | Treating it as "anything Python" — keep it agent-facing |
| `mcp/` | MCP server endpoints + auth | Committing the live config (use the twin-file pattern; see [secrets-management-for-ai-agents.md](secrets-management-for-ai-agents.md)) |
| `context/` | **Meta-level** info about the user and how the agent should behave | Dumping research notes here — those belong in `workspace/` |
| `memory/` | Session-by-session continuity | Storing reference material here — that's `workspace/` |
| `workspace/` | Domain knowledge, active projects, research output | Letting it become a junk drawer; use `_templates/` for shape |

The hardest rule in practice is **`context/` vs `workspace/`**. A good test:
*could this file be reused if I cloned the workspace for a different user?*
If yes → `context/`. If no → `workspace/`.

## The agent contract

`AGENT.md` defines the role; `.instructions.md` defines the workflow. The
contract is roughly:

1. On start: read `README.md` → `AGENT.md` → `SECURITY.md` → `context/user-profile.md`.
2. Check `memory/journal/` for recent sessions.
3. Check `workspace/` for active projects.
4. Then accept the user's prompt.

That four-step boot is the difference between an agent that picks up where it
left off and one that re-asks for context every time.

## Autonomous writing

The `.instructions.md` file grants the agent **standing authority** to write
to `memory/` and `workspace/` without asking. This is the killer feature.

Three categories, with explicit triggers:

- **Workspace docs** — any response containing research, analysis, or
  synthesis. Filename `Title_Case.md`. Cleaned-up structure, not a chat
  transcript.
- **Journal entries** — sessions with 3+ substantive actions.
  Filename `YYYY-MM-DD-short-topic.md`. Format: Context / What Was Done /
  Insights / Open Questions / Next Steps.
- **Decision records** — structural or strategic choices.
  Filename `YYYY-MM-DD-short-title.md`. Format: Context / Decision /
  Alternatives Considered / Rationale / Consequences / Revisit If.

Without these triggers, agents default to "answer and forget." With them,
every session compounds.

## Portability checklist

When you copy this skeleton to start a new agent:

- [ ] Rename the root folder.
- [ ] Fill in `context/user-profile.md` (identity, goals, preferences).
- [ ] Adjust `context/conventions.md` if your style preferences differ.
- [ ] Copy `mcp/mcp-config.template.json` → `mcp/mcp-config.json`, fill creds.
- [ ] If using VS Code Copilot, do the same for `.vscode/mcp.template.json`.
- [ ] Update the `filesystem` MCP server path to point at this workspace.
- [ ] Delete any starter "checklist" sections from the template.

## What model-agnostic actually buys you

- **Switch vendors without losing memory.** Point the new chat at the folder.
- **Diff your agent.** Track prompt and skill changes in git like code.
- **Audit your agent.** `memory/decisions/` becomes a real engineering log.
- **Share an agent.** Strip `mcp-config.json` and the workspace is publishable.
- **Run multiple agents from one shape.** Music agent, writing agent, research
  agent — same skeleton, different `AGENT.md` and `skills/`.

## Anti-patterns I've seen (and made)

- **One mega-prompt for the system message.** Doesn't fit, doesn't version,
  doesn't share across models. Split it into `AGENT.md` + `.instructions.md` +
  `context/`.
- **Memory inside the vendor.** ChatGPT memory, Claude projects, etc. Useful
  short-term, but you can't grep it, can't diff it, can't move it.
- **No `context/` vs `workspace/` distinction.** Everything ends up in one
  folder and you can't reuse the agent for a new user without surgery.
- **Skipping `SECURITY.md`.** The agent will eventually reference a config
  file. Explicit rules upfront beat finding a leaked token in a transcript.

## See also

- [secrets-management-for-ai-agents.md](secrets-management-for-ai-agents.md) — the twin-file pattern.
- [persistent-memory-for-stateful-agents.md](persistent-memory-for-stateful-agents.md) — journal and decision format.
- [tutorials/bootstrapping-a-personal-agent-workspace.md](../tutorials/bootstrapping-a-personal-agent-workspace.md) — step-by-step setup.
- [tools/agent-template/](../tools/agent-template/) — a one-page skeleton reference.
