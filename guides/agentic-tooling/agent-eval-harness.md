# Building an agent eval harness

> If you can't measure regressions in your prompts, skills, and tools, you're flying blind. This is the lessons-learned guide to building an eval harness that's worth running — without becoming its own full-time project.

---

## TL;DR

- **A golden set + an automated runner is 80% of the value.** Fancier comes later.
- **Treat prompts/skills like code: they get a regression suite.** Editing a skill without an eval is the same risk as editing a function without tests.
- **Judge models lie too.** Score with a stronger model than the one under test, and spot-check the judge on a random sample yourself.
- **Track cost and latency, not just correctness.** A 10% accuracy bump for 5x cost may be a regression in production terms.
- **Evals rot.** The same prompt that passes today fails in 3 months when the model is silently updated upstream. Re-run on a schedule, not just on edit.

---

## What you're actually evaluating

Three layers, each with different signal:

| Layer | What you measure | Example |
|---|---|---|
| **Prompt / skill** | Does the model do the right thing for THIS task? | "When asked to refactor, did it preserve behavior?" |
| **Tool / MCP server** | Does the agent call the right tool with right args? | "Did it call `search_db` with the right `limit`?" |
| **End-to-end agent** | Does the system produce the final outcome? | "PR opened, CI green, threads resolved." |

Layer 1 is the cheapest and where most regressions hide. Layer 3 is the truest but slowest. Build layer 1 first; layer 3 last (and only if you have automation for actually running it).

---

## The minimum viable harness

You need three pieces:

1. **A `golden/` folder** with input + expected output pairs (or expected *properties*, see below).
2. **A runner** that loads each case, invokes the agent, compares to expected, prints pass/fail.
3. **A way to mark a case as accepted-failure** when the expected output is "no, we want X to fail this way" or when a flake is known.

That's it. Don't over-build before you have 30 cases.

```
evals/
  golden/
    refactor-pure-function.yaml
    refactor-with-side-effects.yaml
    review-security-bug.yaml
    ...
  runner.py
  judge.py
  results/
    2025-04-15-claude-opus-4.json
    2025-04-15-gpt-5.json
```

---

## Designing a golden case

Don't just store `{input, expected_output_string}`. Outputs are stochastic; exact-match will fail on every minor wording change. Store **properties**:

```yaml
id: refactor-pure-function-001
input: |
  Refactor this function to use early returns:
  function classify(x) {
    if (x > 0) {
      return "positive";
    } else {
      if (x < 0) {
        return "negative";
      } else {
        return "zero";
      }
    }
  }

expectations:
  - kind: must_contain
    value: "return"
  - kind: must_not_contain
    value: "} else {"
  - kind: must_compile
    language: javascript
  - kind: preserves_behavior
    test_cases:
      - input: 5
        output: "positive"
      - input: -3
        output: "negative"
      - input: 0
        output: "zero"
  - kind: judge
    rubric: |
      Did the refactor flatten the nesting using early returns?
      Yes/no/partial.
```

Each `kind` is a check the runner knows how to evaluate. Mix deterministic (must_contain, compile, behavior preservation) with judge-based (rubric). Deterministic checks are cheap and reliable; judge checks catch the rest.

---

## Judges: useful, also wrong

LLM-as-judge works. It also fails in specific, predictable ways:

- **Self-preference bias.** GPT-class models rate GPT-class outputs higher; Claude-class models rate Claude-class outputs higher. Don't judge a model with itself.
- **Verbose-wins bias.** Longer answers score higher than shorter ones, even when the short one is correct. Add explicit "concise + correct beats long + correct" to the rubric.
- **Order bias.** When comparing A vs B, judges prefer whichever came first. Randomize order across runs.
- **Rubric drift.** Vague rubrics ("Is the answer good?") get noisy scores. Specific rubrics ("Does the answer include a step-by-step plan AND a code example AND a gotcha section?") get stable ones.

**Spot-check your judge.** Take 20 random cases the judge marked as pass and read them yourself. If even 2 are actually fails, the judge is unreliable for that rubric and you need to tighten it.

---

## What to run on every change

Triggers in increasing scope:

| Trigger | What runs | Cost target |
|---|---|---|
| Pre-commit (skill or prompt edited) | Just the cases touching that skill | <1 min, free if cached |
| PR opened | Full deterministic suite (no judge) | <5 min, low cost |
| Nightly | Full suite + judge | tracked but acceptable |
| Weekly | Full suite + judge + cost/latency budget | tracked, paged on regression |

The first one is the load-bearing trigger. If "edit skill → re-run cases that touch this skill" is one command, you'll actually use it.

---

## Tracking cost + latency

A "passing" eval that costs 3x more than last week is a silent regression. Track:

- **Tokens in / tokens out** per case.
- **Wall-clock latency** per case.
- **Tool-call count** per case (a regression that adds tool calls bloats every prompt).
- **Total cost in USD** per full-suite run.

Trend them. Alert on >20% jumps. Especially after model upgrades — sometimes a "smarter" model is also a chattier model.

---

## Eval drift

Three kinds, all real:

1. **Model drift.** Upstream model gets silently updated. Last week's pass becomes this week's fail with no code change on your side. Re-run on a schedule, not just on edit, to catch this fast.
2. **Test drift.** Your golden cases stop reflecting real usage. Periodically replay real (anonymized) production sessions and check whether they look like your evals. If not, your evals are testing the wrong thing.
3. **Rubric drift.** The judge starts marking "borderline" cases differently as the judge model itself changes. Re-evaluate rubric reliability quarterly.

---

## Anti-patterns

- **Single-metric optimization.** "We chase accuracy." Then cost triples, latency doubles, the agent rewrites entire files when asked to fix one line. Track 4-6 metrics; let no single one drive decisions.
- **Evals that only run when someone remembers.** If it's not on a schedule, it doesn't exist. CI cron job, or nothing.
- **Snapshot testing for free-form outputs.** Storing the exact text last produced and diffing fails on every minor variation. Property-based checks survive; snapshots fight you.
- **Building elaborate eval infra before having 20 cases.** The infra is the easy part. The cases are the value. Start with 5 cases in YAML and a Python script. Scale when reality demands.
- **Evals that all pass.** If your eval suite is 100% green, it's not covering the failure modes — you've curated cases the model already handles. Add the cases that fail; that's where signal lives.

---

## Adding new cases

Two sources keep evals fresh:

1. **Bug reports / production regressions.** Every time the agent does something wrong in real use, turn it into a case before fixing it. The case is the regression test.
2. **Edge-case brainstorming.** When you ship a new skill or tool, write 3-5 cases that try to break it — empty input, malformed input, conflicting instructions, prompts in the wrong language. Most production failures are these.

If you only add cases the agent already passes, the suite asymptotes to useless.

---

## Versioning

- **Tag every run with model + skill-set + tool-set version.** Reproducibility means re-running last week's failure against last week's setup, not just last week's code.
- **Don't delete cases when they pass.** Move them to a passing tier; keep them in the schedule. They'll regress eventually.
- **Don't modify a case to make it pass.** That's defeating the test. Either the model needs to handle the original case, or the case was wrong (rare but possible) and gets retired with a note.

---

## When NOT to build an eval harness

- **You have one skill, no MCP servers, no tools you wrote.** You're consuming an agent, not building one. Manual spot-checks are fine.
- **The agent's outputs are creative writing.** Evals work for tasks with right-ish answers. For "write me a cool README" — taste reviews beat eval scores.
- **You're 2 weeks from launch and have no infra.** Don't build the harness first. Ship, then build the harness against the first real production failures.

---

## Gotchas

- **Non-determinism is your enemy and your friend.** Set temperature to 0 for reproducibility, but also run a sample at production temperature so you see real variance.
- **Caching can mask regressions.** If your runner caches model responses to save cost, a code change won't trigger a re-run. Bust the cache on any change to skill / prompt / tool definition.
- **Judges hallucinate scores too.** Especially with sparse rubrics. Force the judge to quote evidence from the response before scoring — "the response says X, therefore criterion Y is met." Forces actual evaluation rather than vibes.
- **Cost runs away on judge-heavy suites.** Each judge call is another model call. A 200-case suite with 4 judge rubrics per case = 800 extra calls. Budget for it or use a smaller judge model on most cases and reserve the strong judge for borderline.
- **"Pass" means "passed our checks."** It does not mean "would be acceptable in production." Calibrate by manually reviewing a random 10 cases from the most recent passing run every month.

---

## Related

- [Writing your own MCP server](./writing-an-mcp-server.md) — tools need eval coverage too; tool descriptions are prompt
- [Building Copilot CLI skills](./copilot-cli-skills.md) — skills are the unit of test for prompt regressions
- [Multi-agent orchestration patterns](./multi-agent-orchestration.md) — eval the parent's delegation decisions, not just the sub-agent's output
- [PR-as-publish-gate](../automation-patterns/pr-as-publish-gate.md) — pattern for shipping eval-suite changes themselves through a review boundary
