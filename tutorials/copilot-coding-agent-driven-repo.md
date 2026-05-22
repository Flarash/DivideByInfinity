# Building a Copilot-Coding-Agent-Driven Repo

> **What you'll build**: a GitHub repo where scheduled workflows assign issues to
> `copilot[bot]`, which opens a focused PR you review and merge. Useful for daily
> reports, periodic perf audits, scheduled content automation, etc.
>
> **Prerequisites**: a GitHub account on a plan that includes Copilot Coding Agent
> seats for the target repo, the `gh` CLI authenticated, and ~30 min.

---

## What "Copilot Coding Agent" means here

GitHub's Copilot Coding Agent is the cloud-side agent that can be assigned a GitHub
**issue** (not just chat-prompted) and will open a PR against the repo on its own. The
pattern this tutorial sets up is:

```
scheduled workflow (cron)
        │
        ▼
opens issue with structured brief
        │
        ▼
assigns issue to copilot[bot]
        │
        ▼
Copilot Coding Agent opens a PR
        │
        ▼
you review + merge (or @copilot to iterate)
```

The two-step "open issue → assign to copilot" decoupling matters: GitHub's API does not
let `GITHUB_TOKEN` assign issues to `copilot[bot]`. You need a **personal access token
(PAT)** with `repo` scope for the assignment step. We'll cover that.

---

## Phase 1 — Repo skeleton

Start from a fresh empty repo. From PowerShell:

```powershell
gh repo create <owner>/<name> --public --add-readme
gh repo clone <owner>/<name>
cd <name>
```

Create the minimum layout this pattern needs:

```
.github/
  workflows/
    daily-report.yml         # the scheduler
prompts/
  daily-report.md            # the brief Copilot will read
tools/                       # any Python/Node helpers the agent should run
README.md
```

Reasoning:
- `.github/workflows/` is where the cron lives.
- `prompts/<workflow-name>.md` is a stable place for the human-authored brief that the
  workflow copies into the issue body. Keeps prompts diffable.
- `tools/` is where any local CLI helpers go that the agent (or CI) can invoke. Even
  if the agent does most work, having a `tools/` folder gives it concrete scripts to
  call rather than reinventing them.

---

## Phase 2 — The scheduler workflow

`.github/workflows/daily-report.yml`:

```yaml
name: daily-report

on:
  schedule:
    - cron: "0 7 * * *"      # 07:00 UTC daily
  workflow_dispatch: {}

permissions:
  contents: read
  issues: write

jobs:
  open-and-assign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Open issue with brief
        id: open
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          today=$(date -u +%Y-%m-%d)
          gh issue create \
            --title "Daily report — $today" \
            --body-file prompts/daily-report.md \
            --label automated \
            > issue-url.txt
          echo "url=$(cat issue-url.txt)" >> "$GITHUB_OUTPUT"

      - name: Assign Copilot
        env:
          GH_TOKEN: ${{ secrets.COPILOT_ASSIGN_PAT }}   # PAT, not GITHUB_TOKEN
        run: |
          number=$(echo "${{ steps.open.outputs.url }}" | awk -F/ '{print $NF}')
          gh issue edit "$number" --add-assignee "copilot-swe-agent"
```

Two important details:

1. **`copilot-swe-agent` is the assignee login**, not `copilot` or `copilot[bot]`. The
   bot user's API login is `copilot-swe-agent`; the display name is `Copilot`.

2. **The PAT must live in a separate secret** (`COPILOT_ASSIGN_PAT` above). Don't try to
   reuse `GITHUB_TOKEN` — it lacks the right scope and the assignment will silently no-op.
   Create the PAT under your account, give it `repo` scope, store it as a repo secret:

   ```powershell
   gh secret set COPILOT_ASSIGN_PAT --body "<paste PAT>"
   ```

---

## Phase 3 — The prompt brief

`prompts/daily-report.md` is the issue body. Write it as instructions to a fresh
engineer, not as terse bullets. Concrete is better than clever.

```markdown
# Daily report

You are running on a scheduled trigger. Produce today's market snapshot.

## Inputs
- Run `python tools/fetch_macro.py` to collect raw inputs.
- Run `python tools/score_predictions.py` against yesterday's report.

## Output
- Edit `reports/<YYYY-MM-DD>.md` (create if missing) following the template in
  `reports/_template.md`.
- Append a one-line entry to `reports/track-record.csv` with today's score.

## Rules
- Do not change any file under `tools/` or `.github/` — open a separate issue if
  tooling needs to change.
- If `tools/fetch_macro.py` errors out, comment the full traceback on this issue
  and exit without opening a PR.

## Done when
- A PR exists titled `Daily report — <YYYY-MM-DD>` against `main`.
- The PR body includes the report's headline + score delta vs. yesterday.
```

Notes from experience:

- The agent reads the issue body verbatim. Vague briefs → vague PRs. Concrete file
  paths, scripts to run, and explicit "Done when" criteria drastically reduce iteration.
- The **"do not change X"** rule prevents the agent from "improving" your scheduler or
  prompts mid-run. Keep tool changes in their own dedicated workflow/issue.

---

## Phase 4 — First run + iteration loop

Trigger manually first:

```powershell
gh workflow run daily-report.yml
gh run watch
```

Then:

```powershell
# Confirm the issue has Copilot assigned (not empty)
gh issue list --label automated --json number,title,assignees
```

If the issue is opened but unassigned, the PAT step failed. Common causes:

| Symptom | Fix |
|---|---|
| Workflow log says "assignee not found" | Use `copilot-swe-agent`, not `copilot` or `copilot[bot]`. |
| Step succeeds but assignees list is empty in the API response | The PAT lacks `repo` scope (or only has classic `public_repo`). Regenerate with full `repo`. |
| Step exits non-zero on `gh issue edit` | The PAT is fine but the repo doesn't have a Copilot Coding Agent seat — check `Settings → Copilot → Coding agent`. |

Once Copilot is assigned, you should see a PR appear within ~2–10 minutes.

To iterate on a PR Copilot opened, comment `@copilot please <change>` on the PR — the
agent re-runs and pushes new commits. To kill a stuck run, close the PR; Copilot does
not re-open it on its own.

---

## Phase 5 — Promote the pattern

Once one schedule works, the rest is copy-paste. The repo I built using this exact
shape ended up with 8 scheduled workflows:

- 1 daily report dispatcher
- 6 crawler workflows on staggered crons (Reddit, Bluesky, RSS, etc.)
- 1 weekly track-record scoring run

Each one opens its own issue from its own prompt file in `prompts/`. The PAT step is
identical across all of them; the only thing that changes is the prompt.

---

## Common gotchas, learned the hard way

- **`secrets.<NAME>` inside `${{ }}` blocks** confuses PowerShell quoting on Windows
  runners. If you must construct workflow strings in PowerShell, use single quotes or
  write the body to a file with `-Encoding utf8` first.

- **`workflow` scope** on your local `gh` token isn't related to the Coding Agent —
  it's for *pushing* workflow files. If your push fails with "refusing to allow GitHub
  App to update workflow", you (the human) need `workflow` scope on your own PAT.

- **Don't reuse one PAT across many automations** if you can avoid it. Rotate per
  workflow surface, and prefer fine-grained PATs scoped to the single repo.

- **GitHub Actions step-level `set -e`** aborts the workflow on the first failure.
  When your prompt step expects partial failures (e.g., a crawler that should keep
  going if one source 404s), use `continue-on-error: true` on the step, not `|| true`
  inside the shell — `|| true` masks the error from the job status entirely.

---

## Next steps

- Add a second workflow (e.g., weekly summary) using the same skeleton.
- Combine with the **[PR-as-publish-gate](../guides/pr-as-publish-gate.md)** pattern if
  the PRs publish anything externally (social posts, deployments, etc.).
- See **[tools/copilot-coding-agent/](../tools/copilot-coding-agent/)** for the tool
  page with shortcuts and links.
