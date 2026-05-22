# Image-analysis pipelines with LLMs — the reality

> **Scope:** running an LLM over hundreds or thousands of images to produce structured tags / JSON sidecars. Covers model selection, rate limits, parallelism, checkpointing, and the failure modes that cost the most time.

This is a reference, not a tutorial. If you want a step-by-step walkthrough, start with the [Pinterest taste extraction tutorial](../tutorials/local-pinterest-taste-extraction-with-agent-mode.md). The guide here is the set of patterns and gotchas that determine whether a 700-image run finishes in 6 hours or 6 days.

---

## 1. The shape of every working pipeline

A reliable image-analysis pipeline has the same 5 boxes regardless of which model or runtime you use:

1. **Inventory** — what's in the corpus, what's already analyzed, what's left.
2. **Worker loop** — read image, call vision model, validate response, write sidecar, commit/checkpoint.
3. **Failure capture** — `.failed.json` per failed image, never throw away the error.
4. **Resumption** — re-runs skip sidecars that already exist and pass validation.
5. **Re-pass selection** — pick a subset (uncertain ones, low-confidence ones) for a stronger model later.

If you skip step 4 you'll re-process images you've already paid for. If you skip step 3 you'll be debugging silent skips for hours.

---

## 2. Picking a model — the tier reality

Vision models cluster into three tiers regardless of provider:

| Tier   | What it gets you                                                                          | What it costs                                                 |
| ------ | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Cheap  | Tags, basic descriptions, color palettes, simple object lists. ~80% of what you need.     | Pennies. High throughput. Generous daily quotas on free tiers.|
| Strong | Subtle aesthetic distinctions, mood, "house style" judgments, longer notes.               | 5–20× more. Tighter quotas. Worth it for re-passes only.      |
| Frontier | Genuine multi-image reasoning, deep style comparisons, edge-case OCR.                   | 50–100× more. Save for hand-picked items.                     |

**Real pacing on a ~700-image corpus:** cheap-tier (`gpt-4o-mini`-class) at ~12 RPM ran ~17 s/image at low load, ~60–85 s/image under sustained load. Plan budgets in **wall-clock hours**, not request counts.

**A two-pass strategy beats a one-pass strategy 90% of the time:**

1. **Round 1** — cheap tier over the whole corpus.
2. **Round 2** — strong tier over a *re-pass manifest* of sidecars with uncertainty markers in their `notes` ("possibly", "unclear", "hard to tell").

The re-pass manifest is usually only 5–15% of the corpus, so Round 2 finishes in a fraction of the time Round 1 took, despite the per-call cost.

---

## 3. Rate limits — they are not what the docs say

The docs say "X requests per minute". The reality you'll hit:

- **Quotas are often per-model**, not per-tier or per-account. Cheap-A getting 429'd does **not** mean cheap-B is also exhausted. Empirically: `gpt-4.1-mini` returned a 429 with `Retry-After: 63814s` (~17h) on an account where `gpt-4o-mini` worked instantly on the same minute. If one cheap model is exhausted, try its sibling before sleeping for 17 hours.
- **A 429 with `Retry-After >> 1h` is a daily-quota reset, not a transient backoff.** Don't sleep for it inside the worker. Bail out, log loudly, resume tomorrow.
- **The 429 response body for "quota exhausted" and "abuse detection" is often identical.** Only the `Retry-After` magnitude hints at which one. Treat any retry > 30 min as "this run is done, write a clean exit".
- **Rate-limit headers (`x-ratelimit-remaining`, `x-ratelimit-reset`) are sometimes empty on 429s** even when present on 200s. Don't rely on them for backoff math.

**Cap `Retry-After` aggressively:**

```python
MAX_RETRY_WAIT = 1800   # 30 min

if retry_after > MAX_RETRY_WAIT:
    log(f"FATAL: Retry-After={retry_after}s exceeds cap. "
        f"Likely daily quota. Exiting cleanly.")
    write_partial_results()
    sys.exit(2)
```

This single change converts "ran for 17 hours doing nothing" into "exited in 3 seconds with a clean diagnostic".

---

## 4. Checkpointing — the line between a 4-hour run and a wasted run

The worst failure mode in this whole space is: process 423 images over 5h50m, never commit, hit a timeout, lose everything.

**Checkpoint every N items AND every M seconds, whichever comes first.** Realistic defaults:

- Every **25 images** processed
- Every **600 seconds** elapsed
- A try/finally **final checkpoint** on any exit, including SIGTERM
- Signal handlers that flip a `should_exit` flag and let the current image finish before flushing

```python
PERIODIC_N = 25
PERIODIC_SEC = 600
last_checkpoint = time.time()

for i, img in enumerate(images):
    if should_exit:
        break
    process_one(img)
    if (i + 1) % PERIODIC_N == 0 or time.time() - last_checkpoint > PERIODIC_SEC:
        git_checkpoint(f"progress={i + 1}/{total}")
        last_checkpoint = time.time()

git_checkpoint(f"final, progress={i + 1}/{total}")   # in try/finally
```

`git_checkpoint()` itself needs a **3-retry-with-rebase loop** if you're committing to a shared branch (CI bot + human can race). The retry pattern: `stash → pull --rebase → pop → commit → push`.

**Workflow shell gotcha:** under `set -e`, `git add 'some/pattern/*.json'` exits 128 if the pattern matches nothing. Wrap with `2>/dev/null || true` to keep the step succeeding while still logging the attempt.

---

## 5. Parallelism — the right amount, not "all of it"

A `ThreadPoolExecutor` with **6–8 workers** is the sweet spot for cheap-tier vision models with per-account RPM caps around 10–15. More workers don't make calls faster; they just bunch up against the RPM limiter and increase 429s.

Per-thread budget:
- 1 image read + Pillow downscale to max-edge 1024px (~10–30 ms)
- 1 vision-API call (~5–60 s)
- 1 JSON-schema validation
- 1 sidecar write

The Pillow downscale alone roughly **halves token cost** without any observable quality drop on most use cases. Do it before the API call, not after.

```python
from PIL import Image

def downscale(path, max_edge=1024):
    img = Image.open(path)
    img.thumbnail((max_edge, max_edge))
    buf = io.BytesIO()
    img.convert("RGB").save(buf, format="JPEG", quality=85)
    return base64.b64encode(buf.getvalue()).decode()
```

**RPM limiter pattern that works under threads:** one shared `threading.Semaphore` released by a background ticker once per `60 / rpm` seconds. Simple, correct, and easy to reason about.

---

## 6. Validation — Pydantic with `extra="forbid"`, every time

```python
from pydantic import BaseModel, Field, ConfigDict

class Sidecar(BaseModel):
    model_config = ConfigDict(extra="forbid")

    schema_version: str = Field(pattern=r"^\d+\.\d+$")
    source_path: str
    tags: list[str] = Field(min_length=5, max_length=40)
    tones: list[str] = Field(default_factory=list)
    subjects: list[str] = Field(default_factory=list)
    notes: str = ""
```

`extra="forbid"` is non-negotiable. Without it, the model returning `{"tag": "x"}` (singular) instead of `{"tags": ["x"]}` silently drops the value and your sidecar has an empty tag list that you won't notice for weeks.

On validation failure: write `<basename>.failed.json` with the raw response + the validation error. Never throw away the raw model output.

---

## 7. Logging that lets you diagnose at 2am

Real logs from real runs that paid off:

- **Timestamped lines** (`[2026-05-22 14:22:01] ...`) — humans + `grep` both win.
- **Per-image diagnostics on failure**: HTTP status, request ID, `Retry-After`, `x-ratelimit-*`, model name, token counts, raw response prefix. All in one line.
- **Heartbeat with ETA** every N images: `progress=37/450, elapsed=22m, eta=4h12m`. Lets you decide "kill it or let it run" without `top`.
- **At-exit summary**: total processed, succeeded, failed, total tokens, total wall time.

```python
def log(*args):
    print(f"[{datetime.now().isoformat(timespec='seconds')}]", *args, flush=True)
```

`flush=True` matters. CI runners buffer stdout; without it, your last 30 seconds of logs disappear on timeout.

---

## 8. Signal handling — PEP 475 will get you

`time.sleep()` in Python ≥ 3.5 transparently retries across signals. That means a `Ctrl-C` during a 17-hour rate-limit sleep does *nothing visible*. The fix:

```python
def sleep_interruptible(seconds, should_exit_fn):
    end = time.time() + seconds
    while time.time() < end and not should_exit_fn():
        time.sleep(min(1.0, end - time.time()))
```

Pair it with signal handlers that flip a global flag, not ones that call `sys.exit()` directly. You want the current image to finish, the final checkpoint to land, and *then* the process to exit.

---

## 9. Cost / time budgeting

A rough framework that's held up across multiple corpora:

```
budget_minutes = (n_images / workers) * (avg_seconds_per_image / 60) * fudge_factor
```

With `workers=6`, `avg_seconds_per_image=45`, `fudge_factor=1.5`:

| Corpus size | Budget |
| ----------- | ------ |
| 100   | ~20 min |
| 500   | ~1.5 h  |
| 2000  | ~6 h    |
| 10000 | ~30 h   |

If your CI runner caps step time at 6 hours (GitHub Actions default-ish: 6h job, configurable), **a 2000+ image corpus needs incremental checkpoints or a self-restarting workflow**. Don't try to one-shot it.

---

## 10. The "manifest" pattern that makes everything resumable

Instead of always processing the whole corpus, the worker accepts an optional `--manifest path/to/list.txt` of relative image paths. This makes all of these one-liners:

- **Retry failures**: `list_failed.py > failed_manifest.txt; analyze --manifest failed_manifest.txt --rpm 6`
- **Strong-model re-pass**: `select_repass.py > repass_manifest.txt; analyze --manifest repass_manifest.txt --model strong-tier --rpm 6`
- **Targeted run**: `echo corpus/category-a/*.jpg > a.txt; analyze --manifest a.txt`

```python
parser.add_argument("--manifest", help="Newline-delimited image paths; "
                                       "if omitted, scan whole corpus")
```

Once you have the manifest flag, almost every "I want to re-process X" turns from a script edit into a one-line invocation.

---

## 11. Final checklist before pressing go on a large run

- [ ] `--max-retry-wait` flag with a 30-min cap.
- [ ] Periodic checkpoint every 25 items / 600 seconds.
- [ ] Signal handlers + `sleep_interruptible`.
- [ ] `git_checkpoint` with 3-retry-with-rebase loop (if committing).
- [ ] Pydantic validation with `extra="forbid"`.
- [ ] `.failed.json` written for any validation or HTTP failure.
- [ ] `--manifest` flag implemented.
- [ ] Pillow downscale to max-edge 1024 before API call.
- [ ] RPM limiter set to the **conservative** end of the documented quota.
- [ ] Heartbeat log with ETA every N items.
- [ ] try/finally final checkpoint.
- [ ] CI step `timeout-minutes` ≥ your estimated budget × 1.5.

If you can tick all 12, you can leave the run unattended for 4+ hours and know it'll fail loudly if anything's wrong instead of silently losing work.

---

## See also

- [Pinterest taste extraction with agent-mode image analysis](../tutorials/local-pinterest-taste-extraction-with-agent-mode.md) — the concrete tutorial that this guide is the reference layer for.
- [Setting up a local RAG pipeline](./local-rag-pipeline.md) — the downstream side: what to do with the sidecars once you have them.
- [Scheduled audit workflows](./scheduled-perf-audit-workflow.md) — a different "one-PR-per-run" automation pattern with similar checkpointing concerns.
