# Scheduled "One-PR-Per-Run" Audit Workflows

> A pattern for daily/weekly automation where each run ships **one focused PR** —
> not a multi-item PR, not a never-ending issue, not a queue. The workflow always
> opens an issue whose body contains the next prioritized item from a maintained
> audit list. Used for perf audits, doc-debt cleanups, dependency bumps, etc.

---

## Why "one PR per run"

The alternatives all decay:

| Pattern | How it decays |
|---|---|
| One mega-PR with all items | Becomes unmergeable; reviews stall; conflicts compound |
| Long-running issue with a checklist | Items pile up faster than they're done; agent re-attempts the same item repeatedly |
| One PR per item, queued in parallel | Conflicts; non-deterministic ordering; reviewer fatigue |

One-PR-per-run gives you a steady, reviewable trickle. If the agent flubs an item,
the cost is one wasted PR — not a corrupted shared queue.

---

## The shape

The pattern has three persistent artifacts:

1. **The audit list** — a 10-ish item ordered checklist maintained in the **body of
   the most recently merged `<workflow>/...` PR**. Yes, the PR body. Counter-intuitive,
   but it works: the body is plain-text, diff-reviewable, and authoritative.

2. **The scheduled workflow** — opens an issue + brief, assigns Copilot.

3. **The brief** — instructs the agent to:
   - find the most recent merged PR for this workflow,
   - parse its body,
   - pick the highest-priority item not marked `[x]`,
   - implement it in one focused PR,
   - update the audit list in **the new PR's body** (mark the item done, optionally
     re-prioritize the rest, optionally append new items discovered).

When the new PR merges, *its* body becomes the new source of truth. The list survives
indefinitely without an external store.

---

## The audit list format

PR body template:

```markdown
## Audit: <workflow-name>

Maintained list, ordered by priority. Highest first. When you ship the next item,
mark it `[x]`, append to "Changelog", and re-evaluate priorities of the rest.

### Open

- [ ] **1. <Item title>**
      _Why:_ <one-line rationale>
      _Files:_ `path/to/likely/file.py`
      _Acceptance:_ <verifiable criterion>
- [ ] **2. ...**
- [ ] **3. ...**

### Done

- [x] **Lazy-load embeddings module** (PR #14, 2026-05-18) — saved ~600ms startup.
- [x] **...**

### Changelog
- 2026-05-21: Item 1 promoted from #4 after profiling showed it's the biggest win.
- 2026-05-18: Items 8-10 added (new findings from py-spy run).
```

The **Acceptance** field is what keeps the agent honest. Without it, "make X faster"
gets shipped as a 0.3% improvement that isn't real. With it, the agent has to prove
the change met the criterion.

---

## The brief

`prompts/perf-audit.md`:

````markdown
# Perf audit — next item

You are running on the daily perf-audit cron. Ship the next viable item from the
audit list. One item per PR. No exceptions.

## Find the list

1. Run:
   ```bash
   gh pr list --repo <owner>/<repo> --state merged \
     --search "head:perf/" --limit 5 \
     --json number,title,headRefName,body,mergedAt \
     --jq 'sort_by(.mergedAt) | reverse | .[0]'
   ```
2. Parse the body. Items live between `### Open` and the next `### Done` heading.

## Pick the item

The **first unchecked** item is the next one — the list is priority-ordered, so don't
re-rank as part of this run. If you believe priorities are wrong, update the order in
your new PR's body (in the audit-list block) and explain the rationale in the
Changelog section. But still implement item 1 of the *new* order.

## Implement

- Branch: `perf/<yyyy-mm-dd>-<slug>`
- Make the focused change. Resist scope creep — anything else you find goes into the
  list as a new item, not into this PR.
- Run benchmarks. The PR description must include a before/after measurement that
  satisfies the item's **Acceptance** criterion.

## Update the audit list

In the new PR's body, include the **full updated audit list**:
- Move the implemented item from `### Open` to `### Done` with `[x] (PR #<num>, <date>) — <one-line outcome>`.
- Re-prioritize remaining items if new evidence justifies it.
- Append any new findings to `### Open` at the appropriate priority.
- Add a Changelog entry.

## Done when

- A PR exists on branch `perf/<...>` with the focused change.
- The PR body contains an updated audit list (Open + Done + Changelog).
- Benchmarks in the PR body show the Acceptance criterion is met.
- All existing tests still pass; new tests added for any non-trivial change.

## Hard rules

- One item per PR. Even if items 1 and 2 look "related."
- Never edit `.github/workflows/perf-audit.yml` or this prompt as part of a perf-audit
  PR. Open a separate `tools/audit-workflow` PR for that.
- If item 1's Acceptance criterion isn't reachable today (e.g., needs benchmark data
  you don't have), comment that on the issue, skip to item 2, and explain the skip in
  the new PR's Changelog.
````

---

## Bootstrap

The chicken-and-egg: the brief looks up the most recent `perf/*` merged PR, but on
day 1 there isn't one.

Bootstrap manually: create the first `perf/0000-bootstrap` PR by hand with the
initial audit list in its body, no code changes. Merge it. From then on, the cron
chains itself.

---

## What this gives you

- **Reviewable cadence.** One focused PR per day fits a 5-minute review window.
- **Surfaced priorities.** The list itself is the artifact; you can see at a glance
  what's left and in what order.
- **Self-healing.** A bad PR can be reverted without losing the list — the list
  lives in the most recent *merged* PR, so reverts just promote the previous one
  back to authority.
- **No external store.** No Notion, no Airtable, no Jira. GitHub's primitives are
  sufficient.

---

## Failure modes & guardrails

**"The agent keeps re-shipping item 1 because it didn't actually meet Acceptance."**
The Acceptance criterion needs to be machine-verifiable in the brief. "Faster" is
not a criterion; "p99 startup time ≤ 800ms measured via `python -X importtime`" is.

**"The list grew to 40 items and the brief now takes too long to parse."**
Hard-cap the list at 10–15 open items. New findings displace lower-priority items
into a `### Backlog` section the brief doesn't read, so they're preserved but
out-of-scope for the cron.

**"The agent reordered the list aggressively every run, hiding stale items."**
The brief's "implement item 1 of the *new* order" rule is the lock-in. If the agent
reorders, it still must ship the new #1 — so frivolous reorders are self-punishing.

**"Audits drift to other audit types (perf, docs, deps) and collide."**
Use distinct branch prefixes (`perf/`, `docs/`, `deps/`) and distinct prompt files.
Each audit has its own cron, its own brief, its own list-bearing PR chain.

---

## Variants

- **Doc-debt audit** — items are TODO blocks, outdated examples, missing
  parameter docs. Acceptance: `doctest`/`mkdocs` builds clean.
- **Dependency-bump audit** — items are `name@old → name@new` with changelog
  link. Acceptance: tests + targeted smoke runs pass.
- **Issue-triage audit** — items are stale issues to investigate. Acceptance:
  issue is closed or relabeled with a definite next step.

The infrastructure is identical; only the brief and the audit list contents change.

---

## Related

- [Tutorial: Copilot Coding Agent-driven repo](../../tutorials/copilot-coding-agent-driven-repo.md)
- [Guide: PR-as-publish-gate](./pr-as-publish-gate.md) — complementary pattern when
  the PR's effect is external rather than internal.
