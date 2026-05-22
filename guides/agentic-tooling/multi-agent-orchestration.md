# Multi-agent orchestration patterns

> When and how to delegate to sub-agents from a parent agent. Most "multi-agent" advice online is hype. This is the version that survives contact with real tasks.

---

## TL;DR

- **Default to NOT spawning a sub-agent.** Parent doing the work directly is faster, cheaper, and easier to debug.
- **Spawn when context cost > delegation cost.** A sub-agent gets a fresh context window; if you'd otherwise burn 30KB on exploration, that's the signal.
- **Parallel sub-agents only when truly independent.** Three subagents researching three unrelated modules — yes. Three subagents on serial steps of the same task — no, you've just added overhead.
- **Sub-agents are stateless.** No `ask_user`, no follow-up turn from you. Give them a complete prompt and read the result. If you need a dialogue, do it yourself.
- **One owner per scope.** Once you delegate, the sub-agent owns that scope until done. Don't second-guess by also running grep yourself.

---

## When to delegate vs do it yourself

Pretend each option has a token cost. It does.

| Task shape | Do it yourself | Spawn a sub-agent |
|---|---|---|
| Read 2 files, make 1 edit | ✅ | ❌ overhead > value |
| Search 3 unrelated subsystems for a pattern | ❌ context bloat | ✅ parallel, separate windows |
| Investigate one CI failure with stack trace | ✅ you have the trace | ❌ |
| Investigate 5 CI failures across 5 services | ❌ context blast | ✅ parallel, one per service |
| Implement a feature end-to-end | ✅ context coherence matters | ❌ sub-agent loses thread |
| Generate a 1500-word report from research | ❌ | ✅ sub-agent gathers + you write |
| Quick lookup of a function definition | ✅ grep | ❌ |
| Cross-cutting code review (security, perf, types) | ❌ one pass mixes concerns | ✅ one sub-agent per lens |

The pattern: **delegate when sub-agents can run in parallel and the work is independent**. Don't delegate the critical path — you'll just be sitting around.

---

## Parent / sub-agent architecture

```
PARENT (interactive, has user, has memory)
   │
   ├── decides delegation
   ├── prompts sub-agent with COMPLETE context
   │   (sub-agent has no access to chat history)
   │
   ▼
SUB-AGENT (stateless, single turn or short)
   │
   ├── does scoped work
   ├── returns result to parent
   │
   ▼
PARENT (reads result, integrates, continues with user)
```

The parent **always** owns the user relationship. Sub-agents can't ask the user clarifying questions, can't see prior turns, can't be steered mid-task. Treat them like RPCs.

---

## Prompting sub-agents

Brevity rules that apply to your user-facing replies **do not apply to sub-agent prompts**. Give them everything they need in one shot:

1. **The task.** "Find every place in this repo where we read from `process.env.SECRET_*`."
2. **The repo layout / where to look.** "Code is in `src/`, tests in `tests/`. Skip `node_modules/` and `dist/`."
3. **What 'done' looks like.** "Return a list of `{file, line, env_var}` tuples."
4. **What NOT to do.** "Do not propose fixes. Do not modify files. Just report."
5. **Format expectations.** "Plain markdown bullets, no preamble."

Sub-agents that wander are sub-agents that got vague prompts.

---

## Parallel delegation

When you have N independent investigations:

- **Launch them all in the same response.** Most hosts dispatch sub-agent calls in parallel only if they're in the same turn.
- **Don't read N-1 results before launching the Nth.** That's sequential.
- **Give each one its own scoped context.** Don't paste the same 20KB prompt N times when each one only needs 5KB of it.

Anti-pattern that wastes the parallelism:

```
spawn sub-agent A → wait → read result
spawn sub-agent B → wait → read result
spawn sub-agent C → wait → read result
```

That's three serial RPCs. The parent gains nothing over doing the work itself.

Correct:

```
spawn A, B, C in one response
... receive notification when all three complete ...
read all three results
```

---

## Sequential sub-agents (rare, but real)

Sometimes you want a sub-agent's output to feed another sub-agent. This is fine, but:

- **Each sub-agent re-pays the cold-start cost** (no shared context).
- **The parent has to relay** — paste sub-agent A's result into sub-agent B's prompt.
- **You usually want the parent to do step 2.** If the parent is going to read A's output anyway, the parent might as well act on it.

Sequential sub-agents make sense when each step is itself heavy enough to justify a fresh window:

1. Sub-agent A: research 8 candidate libraries, return a comparison table.
2. Parent: read table, pick one.
3. Sub-agent B: write the integration code for the chosen library.

If A and B are both small, just do it yourself.

---

## What sub-agents cannot do

- **Ask the user a question.** No `ask_user`. If your sub-agent hits ambiguity, it has to make a call and tell you what it assumed. Build that expectation into the prompt.
- **Read prior turns.** They start fresh every time. Re-paste any context they need.
- **Span multiple turns of dialogue with you.** The parent prompts; the sub-agent answers; done. Multi-step orchestration belongs in the parent.
- **Mutate state the parent will read later through a back channel.** No shared memory unless you explicitly persist (filesystem, DB) and the parent re-reads.

---

## Owner-per-scope rule

> Once you delegate a scope to an agent, that agent owns it until it completes or fails; do not investigate the same scope yourself.

The most common multi-agent failure mode: parent spawns sub-agent to grep the codebase, then while waiting, parent also greps the codebase. You've doubled token spend for nothing, and now you have two slightly-different reports to reconcile.

Same rule for parallel sub-agents: if A is investigating module X, parent and sub-agent B should not also investigate module X. Trust your delegation.

---

## Failure handling

Sub-agents fail. Stack overflow on a parsing task, timeout on a deep search, return-nothing on a malformed prompt.

- **If a sub-agent fails twice in a row on the same prompt, do the task yourself.** Don't loop indefinitely.
- **If the failure is "no results," consider that a real answer.** Maybe the thing genuinely isn't there. Don't just retry with broader terms — that's a new prompt and a new judgment call.
- **Sub-agents can return garbage confidently.** Especially under tight context budgets. Sanity-check critical results before acting on them.

---

## Cost reality

Each sub-agent call:

- Re-pays cold-start prompt tokens (system prompt + tool list + skill activations).
- Generates its own response tokens.
- Returns a result the parent then reads (more tokens, into parent's context).

A sub-agent that grep-replaces three lines costs more than the parent doing it. Reach for sub-agents when the **alternative** is the parent burning lots of context exploring something it won't act on directly.

---

## Concrete patterns that work

**1. Research-then-write.** Sub-agent gathers; parent synthesizes. Useful when the research is wide (many files / many sources) but the write-up is narrow.

**2. Cross-lens review.** Three sub-agents review the same diff: one for security, one for perf, one for tests/types. Each fits its scope in fresh context; parent merges findings.

**3. Per-target deployment check.** N sub-agents check N environments in parallel for drift / health. Parent reports.

**4. Search-then-decide.** Sub-agent finds candidates (libraries, configs, files). Parent decides. Useful when "candidates" is a long list but the decision is small.

**5. Background long-running task.** Sub-agent does a 5-min task while parent keeps talking to the user. Parent reads result on notification.

---

## Anti-patterns

- **Sub-agent for a single tool call.** If the parent could have just called the tool, don't wrap it in delegation.
- **"Architect" sub-agent that plans, then "implementer" sub-agent that executes the plan.** The plan loses fidelity in the handoff; parent should do one or both.
- **Sub-agents that recursively spawn sub-agents.** Possible, almost never warranted. The recursion tree gets unauditable fast.
- **Speculative sub-agents.** "I'll spawn this to research X just in case I need it later." You won't. Spawn on demand.

---

## Gotchas

- **Notifications are how you find out a background sub-agent finished.** Don't poll — most hosts will notify the parent. Polling burns context for no reason.
- **Sub-agents inherit the parent's tool list, not the parent's permissions.** A sub-agent can call the same tools but may not have e.g. the same file-system access. Test the actual permission boundary on your host.
- **Sub-agent prompts are immortalized in logs.** Don't paste secrets or PII into the delegation prompt. The parent's prompt-redaction may not apply to the sub-call.
- **"Multi-agent" is often just "one agent with good tools."** Before architecting a 5-agent system, ask whether the parent agent with 5 well-described tools (or MCP servers) would do the same job in half the tokens.

---

## Related

- [Writing your own MCP server](./writing-an-mcp-server.md) — tools that sub-agents can call
- [Building Copilot CLI skills](./copilot-cli-skills.md) — skills can encode "spawn sub-agent when X" patterns
- [Agent eval harness](./agent-eval-harness.md) — testing whether your delegation prompts actually return what you expect
