# PR-as-Publish-Gate

> A pattern for automating side-effecting content (social posts, deployments,
> outbound emails, anything the world will see) without giving the automation
> direct write access to the destination. The human's `Merge` click is the publish
> action; the workflow that runs on merge does the actual posting.

---

## The problem

You want an LLM agent to draft tomorrow's LinkedIn post. You want a scheduler to post
it at the right time. You **don't** want the agent to be able to publish without you
seeing it first, and you don't want to babysit it manually every time.

The naive setup — agent writes a draft, scheduler reads it, posts — has no review
boundary. A bad day for the agent is a bad day for your timeline.

---

## The pattern

```
┌──────────────────┐
│ agent writes     │
│ draft to         │
│ workspace/       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ promote to       │
│ queue/<plat>/    │
│ (status: draft)  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ open PR          │
│ validate-pr CI:  │
│ schema, secrets, │
│ image checks     │
└────────┬─────────┘
         │
   ┌─────┴─────┐
   ▼           ▼
 reject     human merges
            (sets status: queued)
                  │
                  ▼
         ┌─────────────────────┐
         │ post-scheduled      │
         │ workflow (cron)     │
         │ posts due items,    │
         │ commits status:     │
         │ posted back to      │
         │ main                │
         └─────────────────────┘
```

The merge is the approval. The PR diff *is* the review surface (you see exactly the
text that will be posted). Nothing publishes until you click merge.

---

## File contracts

A queued item is a YAML+Markdown file. The frontmatter is a strict schema; the body
is the post text.

```markdown
---
id: 2026-05-21-2310-hello-world
platform: linkedin
status: queued
scheduled_at: 2026-05-21T23:10:00+01:00
tags: [intro, philosophy]
hashtags: [DeepWork]
media:
  - path: media/2026-05-21-hello.jpg
    alt: "B&W photograph of a hand drawing a circle in sand"
visual_brief: |
  Single black-and-white photograph. Square crop. No text overlay.
---
Hello world. <body of the actual post goes here.>
```

Rules that have to be machine-enforced via `validate-pr.yml`:

| Check | Why |
|---|---|
| `id` regex `^\d{4}-\d{2}-\d{2}-\d{4}-[a-z0-9-]+$` and matches filename stem | Prevents two queued items with the same id |
| `platform` ∈ allowed set | Wrong destination = wrong API call |
| `status` is one of `draft`/`queued`/`posted` and matches folder | Drafts can't accidentally go live |
| `scheduled_at` parses as ISO with offset | Cron logic compares UTC |
| `media.*.alt` non-empty when `media` present | Accessibility + Twitter/LinkedIn API requirement |
| `hashtags` have no leading `#`, no spaces | Posters add the `#`; spaces silently break tags |
| No raw API tokens or `_secret`-suffixed values | Catches accidental paste-leaks before merge |

Use `pydantic v2` with `extra="forbid"` to enforce the schema. `extra="forbid"` is the
load-bearing setting — without it, a typo in a field name silently does nothing.

---

## The two workflows

### `validate-pr.yml` — runs on every PR touching `queue/`

```yaml
name: validate-pr
on:
  pull_request:
    paths: ["queue/**", "workspace/**"]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r tools/requirements.txt
      - run: python -m tools.validate --changed-files
      - run: python -m tools.scan_secrets --staged
```

### `post-scheduled.yml` — runs on cron, also on merges to main

```yaml
name: post-scheduled
on:
  schedule:
    - cron: "*/15 * * * *"   # every 15 min
  workflow_dispatch:
    inputs:
      dry_run:
        type: boolean
        default: false

permissions:
  contents: write

jobs:
  post:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.SUN_GH_PAT }}   # so commits can push back
      - uses: actions/setup-python@v5
      - run: pip install -r tools/requirements.txt
      - name: Post due items
        env:
          LINKEDIN_ACCESS_TOKEN: ${{ secrets.LINKEDIN_ACCESS_TOKEN }}
          THREADS_ACCESS_TOKEN: ${{ secrets.THREADS_ACCESS_TOKEN }}
        run: python -m tools.poster --dry-run ${{ inputs.dry_run || 'false' }}
      - name: Push status-update commits
        if: always() && (github.event_name == 'schedule' || inputs.dry_run == 'false')
        run: |
          git config user.name "scheduler-bot"
          git config user.email "scheduler-bot@users.noreply.github.com"
          git push
```

**Subtle bug worth flagging**: the `Push` step needs `if: always()` plus the run
conditions. With `set -e` semantics, a failed post earlier in the job will abort
before push, and the "status: failed" commits the poster wrote locally will be lost.
`if: always()` forces the push step to run even after a non-zero earlier step.

---

## Rollback flow

If you merge something that turns out wrong, you need rollback. The pattern:

```powershell
# Triggers a rollback workflow that calls the platform's DELETE API
gh workflow run rollback.yml -f id=2026-05-21-2310-hello-world -f platform=linkedin
```

The workflow:

1. Reads the queue file by id.
2. Confirms `status: posted` and `post_url` is set.
3. Calls the platform's DELETE (LinkedIn `DELETE /rest/posts/{urn}`, Threads
   `DELETE /v1.0/{id}`, etc.).
4. On success, updates the file's frontmatter to `status: rolled_back`.
5. Commits and pushes.

Two pitfalls hit:

- **`--platform` is mandatory** when an id appears in multiple platform folders.
  Same draft cross-posted to LinkedIn and Threads will share an id; the rollback
  resolver needs the disambiguator.

- **URN escaping for LinkedIn**: `urn:li:share:1234` must be percent-encoded in the
  delete URL. Use `urllib.parse.quote(urn, safe="")` — note `safe=""` (default `/`
  leaves colons intact, which the API rejects).

---

## Why not just use a CMS / Buffer / Hootsuite?

You could. The reasons to roll this pattern instead:

- **Diff-reviewable history.** Every post is in git. You can search "what did I post
  about <topic>?" with `git grep`.

- **Cheap variants.** A new platform is one new YAML field + one new poster client +
  one new folder. Nothing else changes.

- **The PR is the editor.** GitHub's web editor + diff view is a perfectly good
  authoring surface for short-form text. Especially if your draft *came* from an
  agent and you're really just doing copy-editing.

- **No vendor lock-in.** The queue is plain files. If LinkedIn deprecates an API or
  you switch platforms, the historical content is portable.

---

## OAuth gotchas (specific to this stack)

Hard-won notes from wiring LinkedIn, Threads, and Instagram poster clients:

- **LinkedIn auth codes expire in ~30 seconds.** You can't generate the URL, walk
  away, come back, paste, and expect it to work. Have your token-exchange command
  queued and run it immediately after the redirect.

- **Threads has its own App ID, separate from Meta App ID**, even though they're in
  the same Meta developer dashboard. `863283179477547` (Threads) and `813307381633253`
  (Meta) are different things; storing the wrong one in your secret will give you
  `Object with ID '...' does not exist or missing permissions` (Graph code 100,
  subcode 33).

- **Threads dev-mode requires Tester invite**, even for the app owner. Add yourself
  via "Add or Remove Threads Testers" on the Threads API settings page, then accept
  at `https://www.threads.net/settings/privacy/website_permissions`.

- **`oauth.pstmn.io/v1/callback`** is Postman's hosted OAuth handler — useful as a
  redirect URI placeholder when you don't want to run a local callback server. No
  Postman account required.

- **Instagram Graph posting requires a publicly reachable image URL.** If your repo
  is private, `raw.githubusercontent.com/...` won't resolve. Either host media on a
  public CDN, make the repo public, or accept that IG support requires a separate
  hosting decision.

- **PowerShell `Write-Host` bypasses stdout**, so any `Tee-Object` capture of an
  auth helper script will silently miss tokens printed via `Write-Host`. Use
  `Set-Content` inside the script to write the token directly to a file, or switch
  to `Write-Output`.

---

## Variants of this pattern

The "PR as approval boundary" generalizes well:

- **Deployments**: PR merges to `releases/` trigger the actual deploy workflow.
- **Outbound emails**: PR merges to `outbox/` send via SES / Postmark / etc.
- **Generated reports**: PR merges to `reports/` publish to a static site.

In every variant, the human's merge is the side-effect gate; the bot doing the
side-effect doesn't need write access to the queue, only to the destination.

---

## Related

- [Tutorial: Copilot-Coding-Agent-driven repo](../tutorials/copilot-coding-agent-driven-repo.md)
- [Guide: Hybrid agent-workspace pattern](./hybrid-agent-workspace-pattern.md)
