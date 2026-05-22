# MCP Servers

> Model Context Protocol — a JSON-RPC contract between **agents** (Copilot CLI, VS
> Code Copilot Chat, Claude, Cursor, etc.) and **servers** (processes that expose
> tools, resources, and prompts).

## The mental model in one paragraph

Each MCP server is a standalone process (Node, Python, Go, whatever). It exposes
tools, prompts, and resources over stdio (or streamable HTTP). Each agent has its
**own config file** listing which servers to launch and how. There is no system
registry. Configuring a server for VS Code does not configure it for the Copilot CLI;
they each load their own config and run their own copy of the server.

## Per-agent config locations

| Agent | Config path |
|---|---|
| VS Code Copilot Chat | `<repo>/.vscode/mcp.json` or workspace `.mcp.json` |
| GitHub Copilot CLI | `~/.copilot/mcp-config.json` |
| Claude Desktop | `%APPDATA%\Claude\claude_desktop_config.json` (Win) |
| Cursor | `~/.cursor/mcp.json` |

Wrapper keys differ across agents. VS Code uses `servers:`. Copilot CLI uses
`mcpServers:`. Claude uses `mcpServers:`. Cursor uses `mcpServers:`. Always check.

## Local-host vs. Smithery-hosted

- **Local-host.** You clone the server's repo, build/install it, and your agent
  launches it via `command + args`. Fast, free, full control. Brittle across
  machines because absolute paths are baked into the config.
- **Smithery-hosted.** Smithery hosts MCP servers behind a stable HTTP endpoint with
  auth. Cross-machine sharing is trivial; you lose the "fork the server and tweak"
  loop and have a network hop. See [tools/smithery/](../smithery/).

A useful rule of thumb: **start local**, move to Smithery only when you need a
server from more than one machine or surface.

## Servers I've wired up locally

### Spotify
- Repo: <https://github.com/marcelmarais/spotify-mcp-server> (Node)
- ~28 tools: search, playback, queue, library CRUD, playlist CRUD, devices.
- Requires OAuth bootstrap (`npm run auth`) once. Tokens auto-refresh.
- Smoke test: ask `what's currently playing?` from your agent.

### AutoCAD
- Several community AutoCAD MCP servers exist; the COM-bridge pattern below uses any of them
  (e.g. [`puran-water/autocad-mcp`](https://github.com/puran-water/autocad-mcp) or
  [`thepiruthvirajan/autocad-mcp-server`](https://github.com/thepiruthvirajan/autocad-mcp-server)).
- ~12 drawing primitives plus `save_drawing` and natural-language `process_command`.
- Windows-only; needs full AutoCAD installed (not LT). AutoCAD is launched lazily on
  first `draw_*` tool call.

### Filesystem (scoped)
- Built into most agent toolchains; not a separate server you install.
- Always scope to a single repo root via the agent config; never the home directory.

### Memory graph
- Repo: <https://github.com/modelcontextprotocol/servers> (the `memory` server)
- Persistent JSON store of entities/observations/relations the agent can write
  during a session and read in the next.
- Path the store via env var (`MEMORY_FILE_PATH=…/memory-graph.json`) so it
  travels with the repo.

## Walkthrough

The full setup walkthrough for Spotify + AutoCAD specifically — including the
config-key-mismatch gotcha that confuses people — is in
[**tutorials/wiring-local-mcp-servers-spotify-and-autocad.md**](../../tutorials/wiring-local-mcp-servers-spotify-and-autocad.md).

## Building your own

The simplest viable server is ~30 lines of Python using the official `mcp` SDK:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP(name="hello-mcp", instructions="A trivial example.")

@mcp.tool(name="hello", description="Say hello to a name.")
def hello(name: str) -> str:
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run()  # stdio by default
```

Register it in your agent's config:

```jsonc
"hello": {
  "command": "C:\\path\\to\\python.exe",
  "args": ["C:\\path\\to\\hello.py"]
}
```

Reload the agent. You now have a `hello` tool.

### Gotchas building servers

- `mcp.list_tools()` is **async**. Tests need `asyncio.run(mcp.list_tools())`.
- `mcp.run()` defaults to **stdio**. Pass `transport="streamable-http"` for HTTP.
- `FastMCP.call_tool` return shape differs between SDK versions — can be `tuple[seq,
  dict]`, plain `dict`, or `Sequence[ContentBlock]`. Tests should defensively
  normalize.
- Pydantic v2 `model_config = ConfigDict(extra="forbid", frozen=True)` is the right
  default for tool input/output schemas. `frozen=True` prevents accidental mutation
  by tool implementations.

## Reference

- Spec: <https://modelcontextprotocol.io/>
- Official servers: <https://github.com/modelcontextprotocol/servers>
- Python SDK: <https://github.com/modelcontextprotocol/python-sdk>
- TypeScript SDK: <https://github.com/modelcontextprotocol/typescript-sdk>
