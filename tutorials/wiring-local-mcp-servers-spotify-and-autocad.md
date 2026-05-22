# Wiring Local MCP Servers (Spotify + AutoCAD example)

> **What you'll build**: a working local MCP setup with two real servers — Spotify (Node)
> and AutoCAD (Python COM bridge) — and an understanding of the difference between
> VS Code Copilot Chat's MCP wiring and other agents' wiring.
>
> **Prerequisites**: Windows 10/11, Node.js 18+, Python 3.11+, AutoCAD installed (only
> needed if you actually want the CAD server to draw), VS Code with the Copilot
> extension.

---

## The mental model first

MCP ("Model Context Protocol") is a JSON-RPC contract between an **agent** (the LLM
client) and **servers** (processes that expose tools, resources, and prompts). The
servers don't know which agent is calling them; the agents don't know how the servers
are implemented.

Crucially, **each agent loads its own MCP config**. There is no system-wide registry.

| Agent surface | Config file |
|---|---|
| VS Code Copilot Chat | `<repo>/.vscode/mcp.json` and/or workspace `.mcp.json` |
| GitHub Copilot CLI | `~/.copilot/mcp-config.json` |
| Claude Desktop | `%APPDATA%\Claude\claude_desktop_config.json` |
| Cursor | `~/.cursor/mcp.json` |

This trips people up constantly: you wire a server into VS Code, then ask your
**terminal** Copilot CLI session "can you play music?" — and it can't, because the CLI
has its own config and the Spotify server isn't in it.

---

## Phase 1 — Get the two servers locally

We'll use two community servers:

- **Spotify MCP** — a Node server that wraps the Spotify Web API with OAuth.
- **AutoCAD MCP** — a Python server that talks to AutoCAD over COM (Windows-only).

Clone them somewhere stable. **Pick a path you won't move later** — every agent config
is going to hard-code the absolute path.

```powershell
# Pick a single root for all MCP servers
$root = "C:\Users\$env:USERNAME\.mcp-servers"
New-Item -ItemType Directory -Force $root | Out-Null
cd $root

git clone https://github.com/marcelmarais/spotify-mcp-server.git
git clone https://github.com/oraltherapy/CAD-MCP.git
```

Build the Node server:

```powershell
cd $root\spotify-mcp-server
npm install
npm run build      # produces build\index.js
```

Set up the Python server's venv (optional but tidier):

```powershell
cd $root\CAD-MCP
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
deactivate
```

---

## Phase 2 — Spotify OAuth (do this once)

Spotify requires per-user OAuth. The server ships an auth helper:

```powershell
cd $root\spotify-mcp-server
# Edit spotify-config.json:
#   - clientId, clientSecret from your Spotify Developer dashboard
#   - redirectUri = http://localhost:8888/callback  (or whatever port you use)
npm run auth
```

That opens a browser, you approve, tokens get written to `spotify-config.json`. Tokens
auto-refresh from then on. **Never commit `spotify-config.json` to git** — gitignore is
your friend.

Smoke test the server outside any agent:

```powershell
node build\index.js
```

It should sit idle on stdin. Hit Ctrl-C; that's expected. MCP servers communicate over
stdio when launched by an agent, so a "no response on launch" is correct.

---

## Phase 3 — Wire into VS Code Copilot Chat

In your repo (or workspace) create `.vscode/mcp.json`:

```jsonc
{
  "servers": {
    "spotify": {
      "command": "node",
      "args": [
        "C:\\Users\\<you>\\.mcp-servers\\spotify-mcp-server\\build\\index.js"
      ]
    },
    "cad": {
      "command": "C:\\Users\\<you>\\.mcp-servers\\CAD-MCP\\.venv\\Scripts\\python.exe",
      "args": [
        "C:\\Users\\<you>\\.mcp-servers\\CAD-MCP\\src\\server.py"
      ]
    }
  }
}
```

Reload VS Code. In Copilot Chat, click the tools icon — `spotify` and `cad` should
show as available. Click into one to confirm its tools listed.

For Spotify expect ~28 tools (search, playback, queue, library, playlists, devices).
For AutoCAD expect ~12 (`draw_line`, `draw_circle`, `draw_arc`, `draw_ellipse`,
`draw_polyline`, `draw_rectangle`, `draw_text`, `draw_hatch`, `add_dimension`,
`save_drawing`, `process_command`).

**AutoCAD is launched lazily** — the COM bridge only spins up AutoCAD when the agent
calls a `draw_*` tool for the first time. So Phase 3 verification works even if
AutoCAD isn't running.

---

## Phase 4 — Wire into Copilot CLI (different file!)

This is where the "each agent has its own config" thing matters. The CLI reads
`~/.copilot/mcp-config.json` (Windows: `C:\Users\<you>\.copilot\mcp-config.json`).

Edit / create it with the same server entries:

```jsonc
{
  "mcpServers": {
    "spotify": {
      "command": "node",
      "args": ["C:\\Users\\<you>\\.mcp-servers\\spotify-mcp-server\\build\\index.js"]
    },
    "cad": {
      "command": "C:\\Users\\<you>\\.mcp-servers\\CAD-MCP\\.venv\\Scripts\\python.exe",
      "args": ["C:\\Users\\<you>\\.mcp-servers\\CAD-MCP\\src\\server.py"]
    }
  }
}
```

Note the wrapper key differs (`servers` in VS Code, `mcpServers` in Copilot CLI). This
is a frequent source of "I copy-pasted and nothing loads" — check the wrapper key for
your specific agent.

Restart the CLI session. In a fresh `copilot` invocation, ask `what tools do you have
for music?`. You should see `spotify-*` tools listed.

---

## Phase 5 — Sanity tests

**Spotify**:
```
> what's currently playing?
> queue Brian Eno - 1/1
> set volume to 40%
```

**AutoCAD** (this actually launches AutoCAD if it isn't running):
```
> draw a circle at 0,0 radius 50 on the current drawing
> add a horizontal line from -100,0 to 100,0
> save the drawing as test.dwg
```

---

## Common pitfalls

### "Server starts but no tools show"

Almost always a config-format mismatch. Re-read which wrapper key the agent expects
(`servers` vs `mcpServers`) and whether `command` + `args` is the right shape for that
agent's schema. Some agents want `transport: "stdio"` explicit; others infer it.

### "Spotify works in VS Code but not in CLI"

Two separate configs, as described in Phase 4. Each agent loads its own.

### "Tools list is empty after a reload"

Check the agent's MCP log. In VS Code: `View → Output → MCP`. In Copilot CLI:
`copilot --verbose`. The most common error is the server process exiting on startup
because of a bad path or missing dependency. The log shows stderr from the server.

### "AutoCAD says COM not available"

You launched the Python server on a machine without AutoCAD installed (or with an
unsupported AutoCAD edition). The COM bridge needs a registered AutoCAD application
type library. AutoCAD LT doesn't work; full AutoCAD does.

### "Same server, different agents, drifting configs"

Maintain a single canonical block per server in a `setup/mcp-servers.md` doc in your
dotfiles. When you add a new agent, copy from there. Don't try to script it — every
agent's config shape is just different enough that scripting is more brittle than
copy-paste.

---

## Where to go next

- **[tools/smithery/](../tools/smithery/)** — when to host MCP servers locally vs use Smithery.
- **[tools/mcp-servers/](../tools/mcp-servers/)** — quick links to the servers I run.
- The MCP spec itself: <https://modelcontextprotocol.io/>.
