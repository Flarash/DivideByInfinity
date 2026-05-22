# Pinterest Taste Extraction with Image-Analysis Sidecars

> **What you'll build**: an offline pipeline that turns a folder of Pinterest images
> into rich JSON "sidecars" describing each image's aesthetic, color, symbolism, and
> taste signals — then clusters and aggregates them into a master taste profile you can
> feed back into prompts.
>
> **Prerequisites**: Python 3.11+, a local AI agent that can view images (Copilot CLI
> agent mode, Claude code, or similar), and ~500 images in a folder.

---

## Why local agent mode beats cloud here

I initially built this as a GitHub Actions workflow (`analyze-images.yml`) running
against GitHub Models. It works, but two things bite:

1. **Daily quotas**. GPT-class image models on GitHub Models have aggressive per-day
   limits. Mid-batch you hit `retry_after=54225s` (~15 hours) and the workflow stalls.
2. **Iteration cost**. Each batch round-trips through `git push → workflow run → git
   pull`. Tweaking the prompt is painful.

Running locally with an agent that already has `view` on image paths bypasses both. The
trade-off is your context budget — analyzing 400 images at ~4KB JSON each adds up
quickly. Budget for ~70K tokens per batch of 8 images on a 1M-context model.

---

## Phase 1 — Define the sidecar schema

The schema is the contract. Lock it before you generate anything, because retroactive
schema changes mean re-running the whole corpus.

`prompts/image-analysis.md` (the agent reads this verbatim):

````markdown
# Image analysis sidecar

For each image given, write a JSON file with the same stem and `.json` extension.

## Schema

```json
{
  "schema_version": "1.0",
  "image": "pinterest_NNN.jpg",
  "description": "<one paragraph, specific, no padding>",
  "aesthetic_tags": ["15-40 specific kebab-case tags"],
  "emotional_tones": ["3-8 tones"],
  "visual_design": {
    "composition_style": "...",
    "symmetry": "...",
    "visual_density": "low|medium|high",
    "contrast": "low|medium|high",
    "negative_space": "...",
    "layering": "...",
    "shape_language": "...",
    "typography_present": true,
    "typography_notes": "...",
    "realism_vs_stylization": "...",
    "cinematic_qualities": "..."
  },
  "color": {
    "dominant": ["..."],
    "accent": ["..."],
    "saturation": "low|medium|high",
    "temperature": "warm|cool|neutral",
    "palette_style": "...",
    "palette_emotional_effect": "..."
  },
  "lighting": "...",
  "materials": ["..."],
  "textures": ["..."],
  "style_influence": {
    "art": ["..."],
    "design": ["..."],
    "fashion": ["..."],
    "architecture": ["..."],
    "internet": ["..."],
    "cinematic": ["..."],
    "cultural": ["..."],
    "subcultures": ["..."]
  },
  "symbolic_signals": {
    "themes": ["..."],
    "emotional_need": "...",
    "identity_appeal": "...",
    "fantasy_or_aspiration": "...",
    "worldview_signals": "..."
  },
  "save_reason": "<why someone would pin this>",
  "taste_signals": ["5-12 cross-image patterns this image contributes to"],
  "confidence": 0.0,
  "notes": "..."
}
```

## Rules
- Valid JSON. No trailing commas, no comments.
- `aesthetic_tags` must be kebab-case. Specific beats safe.
  (`amoled-black-background`, not just `dark`.)
- Never invent OCR text. If the image has unreadable text, say so in `notes`.
- One paragraph for `description`, no bullet lists inside it.
- `confidence` ∈ [0, 1] reflecting how readable the image is.
````

Keep the schema in source control. Every sidecar version-tags itself via
`schema_version` so future migration scripts can target old versions.

---

## Phase 2 — Folder structure

```
workspace/
  aesthetic-reference/
    pinterest/
      <board-name>/
        pinterest_001.jpg
        pinterest_001.json     ← sidecar, written by agent
        pinterest_002.jpg
        pinterest_002.json
        ...
    syntheses/                 ← per-board markdown summaries (phase 4)
    Taste_Profile.md           ← master profile (phase 5)
    Taste_Profile.json
prompts/
  image-analysis.md
tools/
  build_aggregations.py
  cluster_sidecars.py
  build_taste_profile.py
```

The board-per-folder split matters because aesthetics cluster by board context.
Mixing them at sidecar-time loses signal.

---

## Phase 3 — Generate sidecars (the loop)

Tell your local agent:

> Read `prompts/image-analysis.md`. Then, for every `pinterest_*.jpg` in
> `workspace/aesthetic-reference/pinterest/<board>/` that does **not** already have a
> matching `.json` sidecar, view the image, produce the JSON per schema, and write it
> as a sibling file. Process in batches of 8. After each batch, commit with message
> "Add sidecars batch N (board=<board>)".

Why batches of 8:
- Small enough to keep context fresh per batch.
- Large enough to amortize the "read prompt + remember rules" overhead.
- Commits every batch mean a crash or context-overflow loses at most 8 sidecars.

Tracking progress:

```powershell
Get-ChildItem workspace\aesthetic-reference\pinterest\<board>\ -Filter *.jpg |
  Where-Object { -not (Test-Path ($_.FullName -replace '\.jpg$','.json')) } |
  Measure-Object
```

---

## Phase 4 — Aggregations & clustering

Once a board is fully covered, run aggregations to roll sidecars into board-level
distributions.

`tools/build_aggregations.py` (sketch):

```python
import json
from collections import Counter
from pathlib import Path

def aggregate(board_dir: Path) -> dict:
    tags = Counter()
    tones = Counter()
    signals = Counter()
    n = 0
    for p in board_dir.glob("*.json"):
        d = json.loads(p.read_text(encoding="utf-8"))
        tags.update(d.get("aesthetic_tags", []))
        tones.update(d.get("emotional_tones", []))
        signals.update(d.get("taste_signals", []))
        n += 1
    return {
        "board": board_dir.name,
        "n_images": n,
        "top_tags": tags.most_common(50),
        "top_tones": tones.most_common(20),
        "top_signals": signals.most_common(50),
    }
```

For clustering, embed the sidecars (any sentence-transformer works on the
`description + aesthetic_tags` join), then UMAP → HDBSCAN:

```python
# requirements: umap-learn, hdbscan, scikit-learn, sentence-transformers
import umap, hdbscan
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")
texts = [f"{d['description']} {' '.join(d['aesthetic_tags'])}" for d in sidecars]
emb = model.encode(texts, show_progress_bar=True)

reducer = umap.UMAP(n_components=8, metric="cosine", random_state=42)
reduced = reducer.fit_transform(emb)

clusterer = hdbscan.HDBSCAN(min_cluster_size=8, metric="euclidean")
labels = clusterer.fit_predict(reduced)
```

Pin the embedding model name in your `clusters.json` output — if you re-embed with a
different model later, the cluster IDs are no longer comparable.

---

## Phase 5 — Master taste profile

The taste profile is the document you can paste into prompts to give other agents your
aesthetic context. It's an opinionated synthesis, not raw stats.

Structure:

```markdown
# Master Taste Profile

## Thesis
<one paragraph, human-authored>

## Top descriptors
- <pattern>: <evidence>
- ...

## Color logic
- ...

## Symbolic preoccupations
- ...

## What this taste rejects
- ...

## System prompt
<copy-pasteable block another agent can prepend>
```

The companion `Taste_Profile.json` mirrors the structure with weighted lists, e.g.:

```json
{
  "schema_version": "1.0",
  "thesis": "...",
  "tag_weights": [{"tag": "amoled-black-background", "weight": 0.43}, ...],
  "tone_weights": [...],
  "signal_weights": [...],
  "system_prompt": "..."
}
```

---

## Performance notes (from running this end-to-end)

- **Image downscaling matters.** Pillow downscale to max edge 1024px cuts token cost
  roughly in half with no observable quality drop on Pinterest-quality source images.
- **Parallel workers help when API-bound, not when context-bound.** A local agent is
  context-bound; parallelism doesn't apply. The cloud workflow version uses
  `ThreadPoolExecutor(max_workers=6)` and that was the right sweet spot.
- **Token-cap your output.** Set `max_output_tokens` ~1200 per sidecar. Without it the
  model will pad descriptions when it has nothing more to say.
- **Reasoning-model token budgets are not the same as completion budgets.** If you
  swap to a reasoning model, bump the budget to ≥4000 or you'll truncate mid-JSON.

---

## What this is good for

Once you have a taste profile, you can:

- Generate prompts for image-generation models conditioned on your taste.
- Score new images against your taste vector (cosine similarity on the same embedder).
- Power a "do I like this?" classifier for new content sources.
- Reuse the per-board syntheses as Notion / Obsidian aesthetic context.

The pipeline isn't Pinterest-specific. Any folder of images with a hand-curated
aesthetic (Are.na, screenshot collections, mood boards) works the same way.
