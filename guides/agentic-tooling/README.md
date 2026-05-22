# `agentic-tooling/`

Patterns for building (not just using) agentic tools — MCP servers, host-level skills, multi-agent orchestration, and the eval harness that keeps it all from rotting.

These four guides chain:

1. **[Writing your own MCP server](./writing-an-mcp-server.md)** — ship typed tools an agent can call. Transports, schemas, descriptions-as-prompt, the gotchas.
2. **[Building Copilot CLI skills](./copilot-cli-skills.md)** — `SKILL.md` activation rules, procedural vs declarative bodies, scoping, drift.
3. **[Multi-agent orchestration patterns](./multi-agent-orchestration.md)** — when to delegate to sub-agents (mostly: don't), parallel vs sequential, owner-per-scope.
4. **[Agent eval harness](./agent-eval-harness.md)** — golden cases, LLM-as-judge pitfalls, cost+latency tracking, when NOT to build evals.

Tooling lives downstream of all of these — write the server, then a skill that uses it, decide whether sub-agents help, then build the eval suite that catches regressions when you change any of the above.
