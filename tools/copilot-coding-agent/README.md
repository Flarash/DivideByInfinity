# Copilot Coding Agent

> GitHub's cloud-side coding agent. Assigned a GitHub **issue** (not chat-prompted),
> opens a PR against the repo, iterates on PR comments.

## What it is — practical version

- A `copilot[bot]` user (login `copilot-swe-agent`) that GitHub hosts.
- Triggered by assigning it to an issue. There is no "chat with the agent" surface
  from the web UI — issue-assign is the contract.
- Has access to the repo it's assigned in, can read and write any file it's not
  branch-protected out of, runs its own ephemeral sandbox.
- Available on plans with Copilot Coding Agent seats. Check
  `Settings → Copilot → Coding agent` on the repo.

## When to reach for it

- **Scheduled / recurring tasks.** Daily reports, weekly audits, periodic crawls.
  See [Tutorial: Copilot-Coding-Agent-driven repo](../../tutorials/copilot-coding-agent-driven-repo.md).
- **One-off feature work where you want a PR draft.** Open an issue, assign Copilot,
  iterate via `@copilot please <change>` comments on the resulting PR.
- **Refactors with a clear "Done when".** Verifiable criteria let the agent
  self-check before opening the PR.

## When *not* to reach for it

- **Anything needing real-time interactivity.** It runs in batches; iteration
  loops are ~2-10 min per cycle.
- **Architecture decisions.** It defers to whatever you write in the brief. Vague
  briefs → uninspired structure.
- **Work that requires read-access to systems outside the repo** that aren't
  reachable via standard GHA + secrets (e.g., your private VPN).

## Key gotchas

### Issue-assignee API quirk

`GITHUB_TOKEN` cannot assign issues to `copilot[bot]`. You need a **classic PAT** (or
fine-grained PAT) with `repo` scope, stored as a separate repo secret, and used only
for the assignment step.

```yaml
- name: Assign Copilot
  env:
    GH_TOKEN: ${{ secrets.COPILOT_ASSIGN_PAT }}
  run: gh issue edit "$NUMBER" --add-assignee "copilot-swe-agent"
```

The login is `copilot-swe-agent`, not `copilot` or `copilot[bot]`. Display name is
"Copilot."

### Iteration via PR comments

To ask the agent to revise its PR, comment on the PR:

```
@copilot please use pathlib instead of os.path in tools/foo.py
```

The agent re-runs and pushes new commits to the same PR branch. No need to reopen the
issue.

### Killing a stuck run

Close the PR. The agent will not re-open it. Reopen the original issue (or open a
fresh one) to start over.

### Branch protection interactions

If `main` requires status checks, Copilot's PRs will sit waiting on the checks too.
Make sure the agent has the secrets/permissions it needs to run them; otherwise it
will block on its own CI.

## Reference

- Official docs: <https://docs.github.com/en/copilot/concepts/about-copilot-coding-agent>
- Tutorial in this repo: [Building a Copilot-Coding-Agent-driven repo](../../tutorials/copilot-coding-agent-driven-repo.md)
- Related patterns:
  - [Scheduled "one-PR-per-run" audits](../../guides/scheduled-perf-audit-workflow.md)
  - [PR-as-publish-gate](../../guides/pr-as-publish-gate.md)
  - [Hybrid agent-workspace pattern](../../guides/hybrid-agent-workspace-pattern.md)
