# Secrets management for personal AI agents

Most "AI agent leaked a token" stories start the same way: a config file with
live credentials got committed, or the agent dutifully repeated a secret back
in a transcript. Both are preventable with three small habits.

---

## The twin-file pattern

For every config that holds credentials, keep **two** files:

```
mcp/
├── mcp-config.template.json   ← Committed. Placeholder values only.
└── mcp-config.json            ← Gitignored. Live credentials.
```

Rules:

- **`*.template.json`** has placeholder strings (`"YOUR_API_KEY"`,
  `"<paste-token-here>"`) and is the file you reference in docs and onboarding.
- **`*.json`** (no `.template`) is the live file. Listed in `.gitignore`. Never
  shared.
- New users copy the template, fill in their own creds, and never touch the
  template again.

Why two files instead of one with placeholders? Because a single file gets
edited in place and accidentally committed. With the twin pattern, the live
file *doesn't exist in the repo at all*.

Same pattern for `.env`:

```
.env.example   ← Committed. Variable names with placeholder values.
.env           ← Gitignored. Live values.
```

## The `.gitignore` block

Drop this near the top of `.gitignore` for any agent workspace:

```gitignore
# --- Secrets and live credentials ---
.env
.env.local
**/mcp-config.json
**/.vscode/mcp.json
**/secrets.env
**/credentials.json
**/*.private.*
```

The `**/` prefix matters: in nested agent workspaces, a credential file two
folders deep is just as dangerous as one at the root.

## Agent-side rules

Even with `.gitignore` clean, the *agent itself* will see live credentials
when it loads configs. Add these to `SECURITY.md` so the rules are
machine-readable:

```markdown
## For AI Agents

- NEVER output, log, copy, or reference the contents of API keys, tokens,
  or passwords found in configuration files.
- When discussing MCP server configuration, reference the *.template.json
  file (the safe template), not the live config.
- If asked to share, export, or publish workspace contents, always exclude
  files listed in .gitignore.
- If you detect a credential in a file that is NOT gitignored, alert the
  user immediately and refuse to continue until it is moved.
```

The "alert and refuse" rule is what catches the case where you accidentally
pasted a token into a Markdown note. A good agent flags it before the next
`git add .`.

## Where to actually keep secrets

Roughly in increasing order of safety:

| Option | Use when | Caveat |
|---|---|---|
| Gitignored local config | Personal machine, single-user | Lives in plaintext on disk |
| `.env` + `direnv` / `dotenv-cli` | Want per-project shell env | Same plaintext concern |
| OS keyring (macOS Keychain, Windows Credential Manager, libsecret) | You can script against it | Per-machine; doesn't sync |
| `gh secret set` (GitHub Actions) | CI / Cloud Agent workflows | Visible to anyone with repo write |
| External secret manager (1Password CLI, AWS Secrets Manager, Vault) | Sharing across machines or with collaborators | Operational overhead |

For a personal agent, **gitignored config file + agent-side rules** is the
realistic floor. For anything you'd be unhappy seeing on Twitter, escalate
to keyring or a secret manager.

## Rotation and revocation

The two failure modes you'll eventually hit:

1. **You committed a secret by accident.** Even if you `git reset` and
   force-push, assume it's compromised. **Rotate the credential immediately.**
   GitHub's secret scanning may also tell the upstream provider, which
   silently revokes the token before you do anything else.
2. **A model output contained a secret.** Less likely with the agent-side
   rules, but treat it identically: rotate, don't argue with the model.

A 30-second rotation is always cheaper than a 30-minute investigation.

## CI specifics for personal repos

If your agent runs in GitHub Actions (e.g. scheduled prompts via Copilot
Coding Agent — see [tutorials/copilot-coding-agent-driven-repo.md](../tutorials/copilot-coding-agent-driven-repo.md)):

- Set secrets with `gh secret set NAME` rather than pasting in the web UI —
  fewer accidents with values ending up in browser history.
- Reference them as `${{ secrets.NAME }}` in workflows; never `echo` them.
- Mask anything sensitive the agent emits:
  ```bash
  echo "::add-mask::$DERIVED_VALUE"
  ```
  Without the mask, GitHub will print derived values (e.g. a token's first 6
  chars) in logs.

## A short checklist

Before pushing any new agent repo:

- [ ] `.gitignore` excludes all `*-config.json`, `.env`, `secrets.env`.
- [ ] `SECURITY.md` exists and includes the "For AI Agents" rules.
- [ ] Every credential file has a `*.template.*` twin with placeholders.
- [ ] No real secret appears in `git log -p` (use `gh secret-scanning alerts` if available, or `git secrets`).
- [ ] CI workflows reference `${{ secrets.* }}` only, never inline values.
- [ ] Rotation steps are documented somewhere you'll actually find them at 2 a.m.

## See also

- [portable-agent-workspace-pattern.md](portable-agent-workspace-pattern.md) — where `SECURITY.md` fits in.
- [persistent-memory-for-stateful-agents.md](persistent-memory-for-stateful-agents.md) — why redacting transcripts matters.
