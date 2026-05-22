# Persistent memory for stateful agents

The single biggest upgrade you can give a personal agent isn't a better
model — it's a `memory/` folder that actually gets written to.

This guide is about the **journal + decisions** pattern: simple, model-agnostic,
greppable session continuity in plain Markdown.

---

## Two kinds of memory, two folders

```
memory/
├── journal/      ← What happened this session
└── decisions/    ← Why we chose X over Y
```

They sound similar; they aren't.

- **Journal** = chronological. *Today I did A, B, C. Open question: D.*
- **Decisions** = topical. *We picked Postgres over SQLite because…*

Mixing them turns your `memory/` into a feed instead of a knowledge base.
Keep them split.

## Triggers — when to write what

Without explicit triggers, agents default to "answer and forget." The
following rules go in `.instructions.md` so the agent applies them
automatically.

### Journal entry — `memory/journal/YYYY-MM-DD-short-topic.md`

**Write one when:** the session involved **3+ substantive actions** — file
edits, research, skill installations, structural changes, or non-trivial
problem-solving.

**Skip when:** quick Q&A, single-step task, trivial fix.

**Format:**

```markdown
# YYYY-MM-DD — <short topic>

## Context
Why this session happened. One paragraph.

## What was done
- Bullet list of concrete actions, with file paths or links.

## Insights
What did we learn that future sessions should know?
Be honest about confidence: speculative vs. verified.

## Open questions
Anything left hanging. The next session reads this first.

## Next steps
Concrete actions, ordered. The next session executes these.
```

### Decision record — `memory/decisions/YYYY-MM-DD-short-title.md`

**Write one when:** a structural, methodological, or strategic choice was
made and alternatives were considered (even implicitly).

**Skip when:** trivial, instantly reversible, or purely cosmetic.

**Format:**

```markdown
# YYYY-MM-DD — <short title>

## Context
What problem prompted this decision?

## Decision
What we picked. One sentence if possible.

## Alternatives considered
- Option A — pros / cons
- Option B — pros / cons
- (no need to list options that were never seriously on the table)

## Rationale
Why the choice beat the alternatives for *this* context.

## Consequences
What changes downstream. What we're now committed to.

## Revisit if
Trigger conditions that would make us reconsider.
```

The "Revisit if" section is the one most people skip. It's also the most
valuable — it's what makes a decision *retractable* later without restarting
the whole debate.

## Naming for greppability

- Journal: `YYYY-MM-DD-kebab-case-topic.md`
- Decisions: `YYYY-MM-DD-kebab-case-title.md`

Dates first so the folder sorts chronologically. Kebab-case so the filename
is a usable phrase, not three words jammed together. Topic at the end so
`ls memory/journal/ | grep rag` finds every RAG-related session in a glance.

## What memory is *not* for

`memory/` is **chronology and choices**. It is not:

- Reference material → that's `workspace/`.
- Domain knowledge → that's `skills/`.
- Style preferences → that's `context/conventions.md`.
- The user's identity → that's `context/user-profile.md`.

The single most common failure: dumping research findings into a journal
entry. The finding is reusable knowledge; the journal entry is "what I did
on Tuesday." Put the finding in `workspace/`; link to it from the journal.

## Reading memory on session start

Add this to `AGENT.md` so the agent does it automatically:

```markdown
### Starting a Session
1. Read this file and `context/user-profile.md` for baseline context.
2. Check `memory/journal/` for the 3-5 most recent entries.
3. Review active projects in `workspace/` relevant to the user's interest.
4. Ask for clarification only when genuinely needed — prefer action.
```

The "3-5 most recent" cap matters. Without it, the agent will try to read
everything and waste tokens.

## Cross-references multiply value

A journal entry is a thread. A decision record is a node. Make them link
to each other and to workspace files:

```markdown
## What was done
- Pivoted the embedding model from `text-embedding-3-small` to
  `all-MiniLM-L6-v2`. See [`decisions/2026-05-12-pick-minilm-over-openai-embed.md`](../decisions/2026-05-12-pick-minilm-over-openai-embed.md).
- Updated the schema in [`../../workspace/rag-pipeline/schema.md`](../../workspace/rag-pipeline/schema.md).
```

With three months of these, you can `grep -r "MiniLM" memory/` and find
every relevant session in 50 ms. Try doing that with a vendor's hosted
memory.

## Pitfalls

- **Too many entries.** One thin journal per micro-session beats none, but
  three thin ones beat one good one. Cap it: aim for *one entry per
  contextually-complete piece of work*, not per chat turn.
- **Duplication.** Before writing, the agent should grep for existing
  coverage. Updating beats creating.
- **Premature decision records.** If you're still actively choosing, write
  it as a journal entry first. A decision record implies the door is closed.
- **Transcripts.** A journal entry isn't a chat log. Clean it up. Drop the
  conversational throat-clearing.

## What to redact

Journal entries can leak more than you'd think:

- File paths under `/Users/<your-name>/...` reveal usernames.
- Errors quoting full URLs to internal services reveal infra.
- "I asked X for help" reveals who you work with.

If you ever plan to publish these (e.g. a public examples repo), add a
redact step to `.instructions.md`:

> Before publishing any `memory/` content, strip absolute paths,
> service URLs, and personal names. Replace with `/path/to/...`,
> `<service-url>`, `<person>`.

## See also

- [portable-agent-workspace-pattern.md](portable-agent-workspace-pattern.md) — where `memory/` fits in.
- [secrets-management-for-ai-agents.md](secrets-management-for-ai-agents.md) — the corollary "what to redact" applies double to creds.
