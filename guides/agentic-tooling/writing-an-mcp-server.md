# Writing your own MCP server

> Build a Model Context Protocol server that an agent (Copilot CLI, Claude Desktop, Cursor, etc.) can call as a first-class tool. This guide is the lessons-learned version, not the protocol spec.

---

## TL;DR

- **Pick stdio for local, HTTP/SSE for remote.** Stdio is one binary launched by the host, no network. HTTP is for servers other people connect to over the wire.
- **Tool descriptions ARE the prompt.** The model reads your `description` field and decides whether to call you. Write it like a UX label, not API docs.
- **Schema your inputs hard.** JSON Schema with required fields, enums, and `description` per field. Loose schemas → the agent hallucinates parameters → 50% of your "bugs" disappear when you tighten the schema.
- **Idempotent + side-effect-cheap tools first.** Read tools (search, query, fetch) before write tools (create, delete, deploy). Build trust before you build power.
- **Logging goes to stderr.** Stdout is the JSON-RPC channel. Log to stdout once and the host disconnects.

---

## Transport: stdio vs HTTP

| | stdio | HTTP / SSE |
|---|---|---|
| Where it runs | Host machine, child process | Anywhere reachable |
| Auth | Inherits host's env / OS user | You build it (OAuth, API key, mTLS) |
| Startup cost | Process spawn per host launch | Long-running server, near-zero |
| Multi-client | One client per process | N clients, one server |
| Use when | Personal dev tools, filesystem, local DB | Shared internal services, SaaS integrations |

Default to **stdio** for anything you're writing for yourself. The protocol is the same; the transport is just plumbing. Move to HTTP only when you actually need multiple clients or a remote host.

---

## Minimum viable server (Python)

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("notes-server")

@mcp.tool()
def search_notes(query: str, limit: int = 10) -> list[dict]:
    """Search the local notes vault for matches.

    Use this when the user asks 'where did I write about X' or
    wants to find prior context. Returns title, path, snippet."""
    # ... your impl
    return results

if __name__ == "__main__":
    mcp.run()
```

That's it. The decorator does:

1. Introspects type hints → JSON Schema input.
2. Reads the docstring → tool description the model sees.
3. Wraps the function in JSON-RPC handling.

**The docstring is the most important line of code in this file.** Treat it like the answer to "when should I, the agent, call this?"

---

## Schema design rules

**Required fields are your friend.** If `path` is required and the model doesn't have one, it'll ask the user instead of guessing.

**Enums beat free-form strings.** `status: Literal["pending", "done", "blocked"]` → the model can't typo `"completed"` and break you.

**`description` on every field.** Not just on the tool. Field descriptions show up in the tool schema and steer the model.

**One concept per tool.** Don't ship a `manage_notes(action, ...)` mega-tool. Ship `create_note`, `search_notes`, `delete_note`. The agent's planner reasons about tools, not nested actions.

**Return structured data when you can.** Tables, lists of dicts, objects. The model can `.filter()` and `.map()` over structure but can only regex over a markdown blob.

---

## Auth + secrets

For **stdio**, the host launches you with the user's env. Read `os.environ["GITHUB_TOKEN"]` directly. Do not invent your own config file when an env var works.

For **HTTP**, pick one:

- **Bearer token in `Authorization` header** — fine for personal services.
- **OAuth 2.1 with PKCE** — what the MCP spec recommends for public servers; painful to implement; do it last.
- **mTLS** — overkill for most cases; correct for internal infra.

**Never log the auth header.** Strip it in your request-logging middleware before it hits any sink. The day you forward your logs to a third-party SaaS is the day this matters.

---

## Tool description anatomy

Bad:

```python
@mcp.tool()
def query(q: str) -> str:
    """Query the database."""
```

The model sees this and either over-calls it (any question that mentions "data") or never calls it (no idea what database). Both fail modes are common.

Better:

```python
@mcp.tool()
def query_sales_db(
    sql: str,
    limit: int = 100,
) -> list[dict]:
    """Run a read-only SELECT against the sales analytics DB.

    Use this when the user asks about revenue, deals, pipeline, or
    quarterly numbers. Do NOT use for customer PII queries (use the
    pii_search tool instead). Tables: deals, accounts, line_items.

    Returns at most `limit` rows. For aggregates, write GROUP BY in
    the SQL — don't fetch raw rows and aggregate client-side."""
```

Three things this does:

1. **Says when to call it.** "revenue, deals, pipeline, quarterly"
2. **Says when NOT to call it.** Steers PII queries elsewhere.
3. **Teaches usage.** "Use GROUP BY, don't aggregate client-side."

The agent reads this *every turn*. It's not docs — it's a prompt slice you ship as part of your binary.

---

## Logging without breaking the protocol

Stdio servers communicate over stdout. **Any `print()` to stdout corrupts the JSON-RPC stream** and the host will disconnect. This is the most common first-day bug.

Always log to stderr:

```python
import sys
import logging

logging.basicConfig(
    level=logging.INFO,
    stream=sys.stderr,
    format="%(asctime)s %(levelname)s %(message)s",
)
log = logging.getLogger("notes-server")
log.info("starting")  # safe — goes to stderr
```

For HTTP servers, log normally. The protocol moves to HTTP body; stdout is yours again.

---

## Testing without an agent in the loop

You don't need to fire up the host every time. Spawn the server and pipe JSON-RPC directly:

```bash
# request the tool list
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | python server.py
```

Build a tiny test harness that:

1. Spawns the server as a subprocess.
2. Sends `initialize` → `tools/list` → `tools/call`.
3. Asserts on the result.

This catches schema-vs-impl drift, missing required fields, exception → silent JSON parse failure, etc. Much faster than the click-test-in-the-host loop.

---

## Gotchas

- **Schemas drift from impl.** You change the function signature, forget to update the docstring or type hints, and now the schema lies to the agent. Add a CI check that the deployed schema matches a checked-in fixture.
- **Tool name collisions.** Two MCP servers both expose `search`. The host picks one (usually first registered). Prefix your tool names by domain when the server isn't single-purpose — `notes_search`, not `search`.
- **Long-running tools time out invisibly.** Hosts have per-call timeouts (30s-2min typical). For anything that might take longer, return a job-id and add a separate `get_job_status` tool. Don't try to be clever with progress streaming on stdio.
- **Errors as exceptions vs as result values.** Raising blows up the call. Returning `{"error": "..."}` lets the agent reason about it. Prefer return-as-data for *expected* failures (not found, validation error). Raise only for *bugs* (programmer errors, infra down).
- **Token-stuffing in responses.** A tool that returns a 50KB blob costs every subsequent turn of the conversation. Paginate, truncate with a `has_more` flag, or summarize server-side. Your tool's response size is a budget you spend out of every later prompt.
- **Description drift over time.** You write a great description on day 1, then add 4 new args, and the description no longer mentions them. Re-read every tool description quarterly the same way you'd review on-call runbooks.

---

## When to NOT write an MCP server

- The agent already has a CLI and a shell. If your "tool" is `git log --oneline -10`, just let the agent run that. MCP servers are for things the shell can't express *well* — typed inputs, structured outputs, multi-step internal state.
- The integration is one-off. Three function calls in a Python script that the agent invokes via `python script.py X Y` is fine. Build the server when there are 5+ tools that share state or auth.

---

## Related

- [Building Copilot CLI skills](./copilot-cli-skills.md) — when to use a skill instead of (or alongside) an MCP server
- [Multi-agent orchestration patterns](./multi-agent-orchestration.md) — how an MCP server's tools become a sub-agent's capability set
- [Agent eval harness](./agent-eval-harness.md) — regression-testing the tool descriptions you ship
