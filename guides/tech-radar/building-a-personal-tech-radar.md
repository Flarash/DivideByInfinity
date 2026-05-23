# Building a personal tech radar

**TL;DR:** A personal tech radar is the lightweight version of the ThoughtWorks
team-radar: a short list of things you're using (Adopt), trying (Trial), evaluating
(Assess), or have decided against (Hold). Reviewed quarterly. It's the discipline
that turns "I should look at X" into "I've made a decision about X and moved on."
The point isn't the artifact; it's the *forced quarterly act* of categorizing
what you actually use.

## The four rings

ThoughtWorks invented the format. The rings:

| Ring | Meaning | Action |
|---|---|---|
| **Adopt** | Use confidently in production / daily work | Default choice for relevant problems |
| **Trial** | Worth trying in a real project, with risk awareness | Spike, evaluate, then promote or demote |
| **Assess** | Worth understanding; not ready to use | Read about it, watch a talk, no commitment |
| **Hold** | Don't start using it; phase out if you are | Active avoidance, with a documented reason |

The categories cover the four legitimate responses to a new technology:
"yes," "let me try it," "let me read about it," and "no, and here's why."

Anything else — "interesting, I'll think about it later", "we might use this
next year" — is procrastination. The radar forces a decision.

## Why personal, not team

Team radars (ThoughtWorks-style) are great for large orgs. The personal
radar serves a different purpose:

- **Your tech context is unique.** Languages you use, problems you solve,
  scale you operate at.
- **You don't need consensus.** No politics, no "but the architect said".
- **It's a record of your own evolution.** Old radars show how your views
  shift.
- **It enforces curation.** You can read about 100 things; you can only
  *commit to* a handful.

The team radar is for alignment. The personal radar is for personal sanity
and learning hygiene.

## Quadrants — group by category

The classic quadrants:

1. **Languages & Frameworks** — TypeScript, Go, Rust, React, FastAPI.
2. **Tools** — VS Code, ripgrep, Helm, Terraform.
3. **Platforms** — Kubernetes, Azure, Cloudflare, Supabase.
4. **Techniques** — TDD, GitOps, feature flags, infrastructure as code.

Adapt to your domain. A frontend person might split Tools into "build" vs
"runtime". A platform person might split Platforms into "cloud" vs
"observability".

The point is the grouping, not the specific buckets.

## A realistic personal radar (sample)

A made-up example to show what real entries look like:

### Languages & Frameworks

| Ring | Item | Note |
|---|---|---|
| Adopt | TypeScript | Default for new web work |
| Adopt | Python (3.12) | Default for scripting + ML |
| Trial | Rust | For perf-critical CLI tools |
| Assess | Zig | Watch the 1.0 trajectory |
| Hold | jQuery | Migrate any remaining jQuery before further work |

### Tools

| Ring | Item | Note |
|---|---|---|
| Adopt | ripgrep / fd | Replaced grep/find years ago, no regrets |
| Adopt | Helm 3 | Production standard |
| Trial | OpenTofu | After HashiCorp license change |
| Assess | mise (formerly rtx) | Modern asdf replacement |
| Hold | Vagrant | Replaced by containers + cloud dev environments |

### Platforms

| Ring | Item | Note |
|---|---|---|
| Adopt | Azure | Primary cloud for current work |
| Trial | Cloudflare Workers | For edge functions |
| Assess | Hetzner | Cheaper option for non-prod / hobby |
| Hold | Heroku | Migrated away; pricing + acquisition concerns |

### Techniques

| Ring | Item | Note |
|---|---|---|
| Adopt | GitOps (Argo CD) | For all production K8s |
| Adopt | Feature flags | Default for risky rollouts |
| Trial | LLM-augmented code review | Promising but inconsistent |
| Assess | Property-based testing | Worth applying to invariant-heavy code |
| Hold | "Microservices for everything" | Default to majestic monolith first |

Notice: each entry is **one sentence**. The radar isn't a wiki; it's a
ledger of decisions.

## The quarterly review ritual

Once every 90 days, 30 minutes:

1. **Walk the radar.** For each entry, ask: still in this ring?
2. **Promote / demote.** Trial → Adopt if it stuck. Assess → Hold if you've
   decided not to pursue.
3. **Remove the dead.** Things that no longer matter (technology died, you
   moved away from that domain).
4. **Add new entries.** What did you start using / try / evaluate this
   quarter?
5. **Cap the size.** Personal radars over ~50 items are overwhelming.
   Aggressively retire.

Date the version. Compare to last quarter. Notice patterns.

## Input sources — where new entries come from

You can't add to the radar what you never hear about. Build inputs:

| Source | Cadence | Why |
|---|---|---|
| **ThoughtWorks Tech Radar** (biannual) | 2x/year | Aggregated view of industry consensus |
| **GitHub release feeds** for your top 20 tools | Continuous | Catch breaking changes early |
| **HN front page** | Daily skim | Signals; not all worth pursuing |
| **DEV.to / Lobsters / Mastodon** | Selective | Quality varies |
| **Curated newsletters** (e.g., Console, Pointer, KubeWeekly) | Weekly | Pre-filtered |
| **Conference talks** | Batch quarterly | Deeper than blog posts |
| **Twitter / X / Bluesky** of trusted operators | Daily skim | Real-time, noisy |
| **Your team's internal Slack** | Continuous | What your peers are excited about |

The discipline is **batching low-priority sources**. HN every morning is a
productivity disaster. HN top of week, once a week, is sustainable.

## The "second blog post" rule

A heuristic for evaluating new tech:

- **Wait for the second blog post** before adopting. The first is by the
  creator (selling vibes). The second is by someone who used it for a real
  project (selling pain).
- **Wait for the production case study** before recommending. "We use X at
  scale and here's what we learned" outweighs ten "what is X?" posts.
- **Wait for the v1.0 plus six months** before relying on it. Pre-1.0 APIs
  shift. Post-1.0 they (mostly) stabilize.

This sounds conservative because it is. Most "new" tech that turns out to
be load-bearing took 2-3 years to become production-ready. The benefit of
waiting is enormous; the cost is being slightly behind on novelty.

## Where the radar should live

The format matters less than the discipline. Options:

- **A markdown file in a personal repo.** Versioned, greppable, no service
  dependency.
- **A static-site radar tool** (e.g., AOE Tech Radar, ThoughtWorks build-your-own).
  Pretty visualization; more setup.
- **A simple table in your notes app** (Obsidian, Notion, etc.). Lowest
  friction; private.

Pick what you'll actually keep updated. A perfect-looking radar you never
update is worse than a simple list you do.

## Anti-patterns

- **Radar as wishlist.** Trial ring filled with things you mean to try
  someday. Trial implies *active* trial. Move to Assess.
- **Radar as marketing.** "Adopt: Kubernetes, microservices, GitOps" with no
  explanation. The point is the *reasoning*, not the badges.
- **Radar updated annually.** Quarterly is the minimum to catch drift;
  annual is just a snapshot.
- **Adopting things from someone else's radar without context.** ThoughtWorks
  serves Fortune 500 enterprises. Their Adopt may be your Assess.
- **No Hold ring.** Without explicit Hold, you can't say no with discipline.
  Hold is the most useful ring for personal sanity.

## What changes when you have one

A radar shifts the question from "should I learn / try X?" to "where on the
radar does X go?" Every novelty triggers a categorization. Most novelties
end up in Assess for a quarter, then drop off. A few graduate to Trial.
Even fewer reach Adopt. The rest don't survive the review.

The output is **calm**: a small set of things you're actively using, a small
set you're trying, and explicit permission to ignore everything else.

## Gotchas

- **Conflating "I tried this once" with Trial.** Trial means a real
  project. A weekend hack is Assess.
- **Holding things you actually still use.** Be honest. If it's load-bearing,
  it's at least Trial.
- **No retirement step.** Old tech doesn't move to Hold automatically; it
  fades from your radar. You have to actively remove dead entries.
- **Personal radar going public.** Be aware: a public radar shapes how
  others perceive you. Adopt/Hold opinions can be controversial.
- **Imposter syndrome from comparing radars.** Other people's radars are
  edited highlight reels. Compare your radar this quarter to your radar
  last quarter, not to anyone else's.

## When NOT to keep a radar

- **You're in a stable, mature stack and not exploring.** A radar adds no
  value.
- **You're on the bleeding edge of one specific technology.** Your "radar" is
  reading the GitHub commits.
- **You're already drowning in commitments.** Pick up the practice when you
  have 30 minutes a quarter.

## Related

- [`keeping-current-without-burning-out.md`](./keeping-current-without-burning-out.md)
  — the information-diet practices that feed the radar.
- [`../cloud-architecture/well-architected-decisions.md`](../cloud-architecture/well-architected-decisions.md)
  — ADRs are to architecture what the radar is to tooling: small, dated,
  immutable.
