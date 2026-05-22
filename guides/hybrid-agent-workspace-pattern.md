# The Hybrid Agent-Workspace + `tools/` Repo Pattern

> A repo layout for projects that mix **human-edited content/documents** with **agent
> automation** (Copilot Coding Agent, local agent mode, scheduled workflows). Optimizes
> for the agent being able to find the right thing without re-reading the world.

---

## Summary

```
<repo>/
  workspace/         ← human-edited; agent reads, sometimes writes
  tools/             ← agent-executable Python/Node scripts (deterministic)
  prompts/           ← human-authored briefs the agent reads verbatim
  memory/            ← decisions, journal, agent-written notes
  context/           ← short, high-density context files the agent loads first
  .github/workflows/ ← scheduled triggers and CI
  README.md
  AGENTS.md          ← agent-only orientation file
```

That's the whole pattern. The rest of this guide is *why* each folder is there.

---

## The two failure modes this avoids

1. **The "where does this go?" tax.** Without a convention, the agent (and you, three
   months later) waste cycles guessing. Worse, agents will *invent* new top-level
   folders on every fresh session, leading to repos with `data/`, `inputs/`, `assets/`,
   `references/`, `notes/`, and `scratch/` all meaning roughly the same thing.

2. **The "agent overwrote my draft" disaster.** Without an explicit separation of
   human-edited vs. agent-written content, scheduled workflows will eventually
   clobber something you cared about. The folder names below encode the boundary.

---

## `workspace/` — the human's content

Anything you author by hand goes here. Subfolders are domain-specific:

```
workspace/
  drawings/svg/
  aesthetic-reference/
  drafts/
  research/
```

The agent treats `workspace/` as read-mostly. It can append (e.g., write image
sidecars next to images), but it should not modify your existing files without an
explicit instruction.

Make this a rule in `AGENTS.md`:

> Files under `workspace/` are human-authored unless they end in `.json` and live next
> to a `.jpg` / `.png` / `.svg` of the same stem. You may append new files freely; you
> may not modify existing files there without confirmation.

---

## `tools/` — deterministic agent-executable scripts

Anything the agent should *run* rather than *reimplement* lives here. Examples:

```
tools/
  fetch_macro.py            # opens a daily report
  analyze_images.py         # generates image sidecars
  rollback_post.py          # un-publishes a queued post
  requirements.txt          # pinned deps
```

Rules of thumb:

- **Pin dependencies.** `requirements.txt` and `package-lock.json` exist so the agent
  doesn't have to debug version drift on a fresh runner.
- **Make scripts idempotent.** The agent will re-run them on retries. `analyze_images.py`
  should skip images that already have sidecars. `fetch_macro.py` should overwrite
  today's snapshot but not yesterday's.
- **Exit code 0 == success; >0 == failure.** Agents key on this. Don't print "ERROR:"
  and exit 0.

---

## `prompts/` — the briefs

Every scheduled workflow that calls the Copilot Coding Agent should have a
corresponding `prompts/<workflow-name>.md`. Two reasons:

1. **Diffable.** When you tune the prompt, the diff is reviewable.
2. **Loadable from the workflow.** `gh issue create --body-file prompts/foo.md`
   beats inline heredocs that PowerShell mangles.

Brief structure that consistently works:

```markdown
# <Task name>

You are running on <trigger>. <One-sentence goal.>

## Inputs
- Concrete commands to run, with paths.

## Output
- Concrete files to create/modify, with paths.

## Rules
- Hard constraints: what NOT to touch, what to error out on.

## Done when
- Verifiable criteria.
```

Avoid: persona prompts, "be a 10x engineer" exhortations, vague success conditions.
The agent already has a persona; what it lacks is your domain context.

---

## `memory/` — agent-written notes

```
memory/
  decisions/
    2026-05-21-image-analysis-throughput.md
  journal/
    2026-05-21-pinterest-taste-engine.md
```

This is where the agent puts decisions it made and journals what it did, so the
*next* session of any agent can read it instead of asking. Date-prefixed filenames so
chronology is obvious from `ls`.

Pattern for decision docs:

```markdown
# 2026-05-21 — Image analysis throughput

## Context
<what state of the world prompted this>

## Decision
<the choice, in one sentence>

## Trade-offs
<what we gave up>

## Reversal cost
<how hard it is to undo if we change our mind>
```

The agent can write this in 30 seconds at the end of a session and save you 30
minutes of "wait, why did we do it this way?" later.

---

## `context/` — high-density orientation

Files in `context/` are short, opinionated summaries the agent loads at the start of
a task. Think of them as the single-page briefings.

```
context/
  drawing-aesthetic.md       # the master taste profile, summarized
  api-quirks.md              # gotchas with external APIs
  history.md                 # narrative of why this repo exists
```

Crucial constraint: **keep each file under ~3KB**. The point of `context/` is that an
agent on a budget can load all of it; if any single file is huge, the agent will skip
or summarize and lose nuance.

---

## `AGENTS.md` — the agent orientation file

A growing convention (Cursor, OpenDevin, etc.) is for repos to ship an `AGENTS.md` at
the root. Mine looks like:

```markdown
# Agent orientation

## You are
A coding agent operating inside <repo name>. <One-line goal of the repo>.

## Read first
- `context/history.md` — why this repo exists.
- `context/drawing-aesthetic.md` — the taste profile.
- `prompts/<your-trigger>.md` — the specific brief.

## Folder rules
- `workspace/` — human-edited; append only.
- `tools/` — run, don't re-implement.
- `memory/` — your scratch and decision log; date-prefix new files.

## Commit conventions
- Co-author trailer required: `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>`
- Branch naming: `<workflow-name>/<yyyy-mm-dd>-<short-slug>`

## When stuck
Open an issue titled `[blocker] <description>`, assign it to the human, exit cleanly.
```

The point is to consolidate the "stuff every fresh agent session needs to know"
that you'd otherwise put in the system prompt of every brief.

---

## How this scales

Two of my repos use this layout end-to-end:

- **Envision** — knowledge/writing/research workspace, Pinterest taste pipeline, image
  analysis workflows. Heavy on `workspace/aesthetic-reference/` and `tools/`.
- **Sun** (private) — social-media automation. Heavy on `prompts/`,
  `tools/sun/clients/`, and a `queue/` folder layered on top for the
  [PR-as-publish-gate](./pr-as-publish-gate.md) pattern.

A third (**Bellwether**) bootstrapped from scratch using this layout in 9 sequential
phases; the bootstrap walkthrough is in
[tools/copilot-coding-agent/](../tools/copilot-coding-agent/).

---

## What this pattern is *not*

- **It is not a framework.** No `npm install` step makes a repo "agent-workspace
  compatible." It's a folder convention.
- **It is not a substitute for clear briefs.** The folders point the agent at the
  right files; they don't tell it what to do.
- **It is not for libraries.** A Python package or npm module shouldn't ship a
  `workspace/`. This is for *projects* — repos with content, not just code.

---

## Related

- [Tutorial: Copilot Coding Agent-driven repo](../tutorials/copilot-coding-agent-driven-repo.md)
- [Guide: PR-as-publish-gate](./pr-as-publish-gate.md)
- [Guide: Scheduled perf-audit workflow](./scheduled-perf-audit-workflow.md)
