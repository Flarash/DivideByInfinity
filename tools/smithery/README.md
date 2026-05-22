# Smithery

> A hosted directory + runtime for MCP servers. You point your agent at a Smithery
> connection URL; Smithery runs the server in the cloud and proxies stdio over
> Streamable HTTP / SSE.

## When Smithery is the right choice

- You want a server available on **multiple machines** without cloning + building it
  on each. Smithery hosts it once.
- You want servers your **GitHub Actions workflows** can call — there's no local
  process to launch in a CI runner; the workflow hits a URL.
- The server has cumbersome dependencies (Docker, native libs) you don't want to
  install everywhere.

## When local is still the right choice

- Latency matters (audio, drawing, anything visual-feedback driven).
- The server reads your local filesystem (Smithery doesn't have access to it).
- You need to fork the server and patch it — Smithery hosts the upstream.
- You're behind a corporate firewall that blocks outbound to `smithery.ai`.

## Wiring it up

### Set up auth

You need two values:

- `SMITHERY_API_KEY` — your account's key, from <https://smithery.ai/account>.
- `SMITHERY_NAMESPACE` — the namespace under which you want to address servers
  (typically your username or org). **Don't paste the wrong tab from your account
  page** — `SMITHERY_NAMESPACE` is the slug, not the display name.

For local agents:

```powershell
# Copilot CLI: ~/.copilot/mcp-config.json
{
  "mcpServers": {
    "smithery": {
      "url": "https://server.smithery.ai/<namespace>/smithery/mcp",
      "headers": { "Authorization": "Bearer ${SMITHERY_API_KEY}" }
    }
  }
}
```

For GitHub Actions:

```yaml
env:
  SMITHERY_API_KEY: ${{ secrets.SMITHERY_API_KEY }}
  SMITHERY_NAMESPACE: ${{ secrets.SMITHERY_NAMESPACE }}
```

**Important**: Read `SMITHERY_NAMESPACE` **from secrets**, not from a hardcoded env
var or a workflow input. Hardcoding it makes it the wrong shape for anyone else
trying to use the workflow. (This bit me in a real repo — fixed by a one-line PR
reading both values from secrets.)

### The "search + execute" pattern

The Smithery meta-server's `search_toolbox` returns matches but doesn't invoke them.
To call a found tool, use `execute`:

```javascript
async () => {
  const matches = await connections.smithery.search_toolbox({ query: "weather" });
  // Matches: [{ server, tool, description, inputSchema }, ...]
  return await connections.smithery.execute({
    code: `async () => connections.${matches[0].server}.${matches[0].tool}({ city: "Lisbon" })`
  });
}
```

The point: `search_toolbox` is for discovery; `execute` is for actual calls into
the broader toolbox of your installed Smithery connections.

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Bad/expired API key | Regenerate at <https://smithery.ai/account>, update secret |
| `404 namespace not found` | Wrong slug in URL | Double-check `SMITHERY_NAMESPACE` is the slug, not the display name |
| `502 Bad Gateway` | Smithery cold-starting your server | Retry. If persistent, the server itself is failing — check the Smithery dashboard for that server's logs |
| Tool list empty | Your namespace has no servers installed | Visit <https://smithery.ai/> and install servers into your namespace |
| Works locally, fails in GHA | `SMITHERY_NAMESPACE` not in repo secrets | Add via `gh secret set SMITHERY_NAMESPACE` |

## Walkthroughs from real repos

- **Bellwether** uses Smithery for `world-briefing` (macro-economic snapshots) inside
  a scheduled `daily-report` workflow. The macro snapshot call goes through
  Smithery's REST API, not via MCP-over-stdio — GHA can't host an MCP client
  easily, so REST is the right shape. The function that does this lives in
  `tools/open_daily_issue.py` and uses `Smithery Connect REST` to fetch the snapshot.

- **Envision** uses Smithery to test cross-machine setups: a server I have locally
  during dev is also installed in my Smithery namespace, so workflows can hit the
  cloud copy.

## Reference

- Smithery: <https://smithery.ai/>
- Smithery Connect REST docs: <https://smithery.ai/docs/connect/rest>
- MCP spec: <https://modelcontextprotocol.io/>
- Local MCP servers — [tools/mcp-servers/](../mcp-servers/)
