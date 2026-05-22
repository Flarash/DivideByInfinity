# Building Copilot CLI skills

> Copilot CLI (and similar agentic CLIs) lets you ship reusable instruction bundles called **skills** that activate when a task matches their description. Done well, they're force multipliers. Done badly, they fight your system prompt and degrade behavior. This is the difference.

---

## TL;DR

- **A skill is a `SKILL.md` file** with a frontmatter description + a body of instructions. The host scans available skills, matches the user's task to the description, and injects the body into the prompt.
- **The description is the activation trigger.** It's not a label — it's the *retrieval query*. Make it match how a user would phrase the task, not how you'd document the feature.
- **Body is procedural, not declarative.** "Do X, then Y, then check Z" wins over "this skill handles X-related tasks."
- **Scope each skill to one workflow.** A skill that "helps with code review and also writes tests and also explains git" will activate at the wrong times and confuse the planner.
- **Skills compete with each other.** Two skills with overlapping descriptions → one wins arbitrarily. Treat skill descriptions like API namespaces.

---

## Anatomy of a SKILL.md

```markdown
---
name: agent-merge
description: |
  Drive a pull request through the full merge lifecycle. Use when
  the user asks to "merge this PR", "check in on PR #N", or when an
  automated agent-merge tick fires. Handles review threads, CI checks,
  conflicts, and the final merge command end-to-end.
---

# Agent Merge — Automated PR Lifecycle

You are managing the lifecycle of a pull request. Your goal is to drive it to merge.

## Merge conditions

A PR is ready when ALL three are satisfied:
| Condition | What it means | How to check |
...

## Workflow

### 1. Triage
Run `gh pr view <number> --json state,...` and read the state.

### 2. First pass — address everything visible now
...
```

Three parts the host actually uses:

1. **`name`** — stable id. Used by the host to dedupe and link.
2. **`description`** — what the model reads to decide "should I activate this skill for this task." This is the *only* thing the planner sees before activation.
3. **Body** — the instructions injected into context once the skill activates.

---

## Writing the description

This is the highest-leverage 2-4 sentences in the whole skill. The agent reads N skill descriptions per turn (N can be 20+) and picks at most one or two to activate. You're competing for that slot.

**Anchor on the user's verbs.** Skills activate on "what is the user asking me to do," not "what is this skill about." Lead with verbs the user might say.

Bad:

```yaml
description: A comprehensive PR management toolkit covering reviews, CI, conflicts, and merges.
```

Good:

```yaml
description: |
  Drive a pull request through the full merge lifecycle. Use when
  the user asks to "merge this PR", "check in on PR #N", or when an
  automated agent-merge tick fires.
```

The second one says **when to activate**. The first one says **what the skill is about**. The agent needs the first.

**Include negative space.** If your skill is for `agent-merge`, say "Not for *creating* PRs — use the `pr-create` skill for that." Skills lie when they don't know their boundaries; spelling out what NOT to do is half the description's job.

---

## Writing the body

The body becomes part of the system prompt when the skill activates. That has consequences:

- **It's measured in tokens.** A 5KB skill that activates on every PR-related turn is 5KB per turn. Be ruthless. Cut every sentence that doesn't change a decision.
- **It overrides system prompt for that task.** Don't repeat global rules ("be concise", "use tool calls in parallel"). Add domain-specific rules.
- **It's read top-to-bottom, every time.** Put the most-load-bearing rule first. Don't bury "never use `--delete-branch`" on line 280.

**Structure that works:**

1. **One-line goal restatement.** "Your goal is X." Anchors the model.
2. **Conditions / contract.** What "done" looks like. Often a table.
3. **Rules / invariants.** Bullets. Things to never do, things to always do.
4. **Workflow.** Numbered steps. This is where you spend most of the tokens.
5. **Summary template.** What to say at the end. Models love a template to fill.

---

## Procedural > declarative

Bad (declarative):

> This skill handles code review responses. It supports replying to threads, resolving threads, making code changes in response to feedback, and posting summary comments.

Better (procedural):

> For each unresolved review thread, do this in order:
> 1. Read the comment and decide if it's actionable.
> 2. If actionable, make the code change and commit.
> 3. Reply to the thread explaining what you changed.
> 4. Resolve the thread via the helper script. The thread isn't done until the script exits 0.

The procedural version tells the model **what to actually do** at each decision point. The declarative version is documentation; it doesn't drive behavior.

---

## Scoping

**One skill = one user intent.** A skill that activates for both "code review" and "writing tests" will fire at the wrong times. Split it.

**Rule of thumb:** if you can't write a single description sentence that captures when to activate it, the skill is too broad. Split.

**Counter-rule:** if two skills have nearly-identical descriptions, they should probably be one. Three skills called `pr-merge`, `pr-merge-conflicts`, and `pr-merge-after-review` will all activate together and the planner will pick whichever was registered first. Merge them.

---

## Tools the skill references

Skills often assume helper tools exist (CLI commands, shell scripts, MCP tools). Be explicit:

```markdown
## Required tools
- `gh` CLI authenticated with `repo` scope
- `bash` (skill uses `sleep`, `cat <<EOF`)
- Helper script at `src/skills/agent-merge/reply-and-resolve.sh`
```

This serves three audiences:

1. **The agent**, before it tries to call something that doesn't exist.
2. **Future you**, when you're wondering why the skill broke.
3. **CI**, if you want to validate skills against available tools.

If a helper script is path-dependent, ship the path INSIDE the repo and use a stable relative path. Don't depend on a user-installed binary unless you absolutely must.

---

## Activation gotchas

- **Descriptions that overlap fight each other.** Two skills mentioning "review" both activate on review-adjacent tasks. The planner picks one, often not the one you'd expect. Use distinct vocabulary.
- **Descriptions that are too narrow never activate.** "Use this when the user types exactly 'foo bar baz'" — they won't. Phrase like a user would.
- **Skill name conflicts.** If two skills have the same `name`, the later one wins (or the host errors, depending on host). Namespace by tool/team if your skill is project-wide.
- **Skill bloat.** Every skill activated is more prompt to read. If your project has 40 skills, the planner is spending real tokens just on description-matching. Prune quarterly.

---

## When NOT to write a skill

- **One-off instructions for one PR.** Just say it in the chat. Skills are for repeating workflows.
- **Logic that should be a tool.** If the steps are "run this command, parse the output, run another command" — that's a tool (or an MCP server), not a skill. Skills are for *judgment*, not *computation*.
- **General coding style.** Put that in `.github/copilot-instructions.md` or the equivalent repo-level config. Skills are activated by task; style applies to every task.

---

## Versioning + drift

Skills get stale fast. Workflows change, helper scripts move, gotchas get fixed.

- **Date-stamp the body.** A `_Last reviewed: 2025-Q1_` line at the bottom forces a quarterly look.
- **Test the skill end-to-end after editing.** Run an actual task that should activate it and read the agent's transcript. Skills often activate at the wrong moment after edits.
- **Diff skills like code.** They're load-bearing prompts shipped in the repo. Review changes the same way you'd review a service-level prompt change.

---

## Gotchas

- **Editing the skill mid-session doesn't always reload it.** Some hosts cache. After editing, restart the agent for the next test run.
- **Skills don't see each other's state.** A skill can't say "if the `agent-merge` skill is also active, do X." Skills compose at the prompt level, not the program level.
- **Frontmatter is fragile.** A mistyped `description:` (e.g. missing the `|` for multi-line) silently produces a one-line description that's parsed wrong. Validate frontmatter in CI if your repo has many skills.
- **Body markdown is rendered to the model as text.** Headings are useful for the model's chunking, but tables aren't magic — they're parsed as text. Use tables when they help the reader (you AND the model), not for layout.

---

## Related

- [Writing your own MCP server](./writing-an-mcp-server.md) — skills are instructions, MCP servers are tools; they pair
- [Multi-agent orchestration patterns](./multi-agent-orchestration.md) — skills can guide when (and when not) to spawn sub-agents
- [Agent eval harness](./agent-eval-harness.md) — regression-test skills the same way you'd regression-test prompts
