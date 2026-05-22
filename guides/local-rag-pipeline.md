# Setting up a local RAG pipeline you can actually trust

> **Stack covered:** ChromaDB + sentence-transformers (`all-MiniLM-L6-v2`) on disk, optional UMAP + HDBSCAN for clustering, JSON sidecars as the source-of-truth corpus. All local, no API keys.

This guide is the lessons-learned version of building a local RAG pipeline over a corpus of LLM-tagged image sidecars (~700 JSON files, ~390 indexed). The same shape works for tagged documents, code chunks, or any other corpus where each item has structured tags + free-form text.

It is **not** a "what is RAG" explainer. It is the set of decisions and traps that, in hindsight, mattered far more than picking a vector DB.

---

## 1. The corpus shape that makes the rest easy

Before anything else, settle on a **sidecar file per item** with a stable schema:

```
corpus/
  category-a/
    item-001.jpg
    item-001.json       ← sidecar
    item-002.jpg
    item-002.json
  category-b/
    ...
```

```jsonc
{
  "schema_version": "1.0",
  "source_path": "corpus/category-a/item-001.jpg",
  "tags": ["tag-1", "tag-2", "..."],
  "tones": ["serene", "muted"],
  "subjects": ["..."],
  "notes": "Free-form short description, 1–3 sentences."
}
```

**Why a sidecar per item, not one big file?**

- Resumable: you can incrementally process the corpus across days/sessions.
- Diffable: each new tagging run shows up cleanly in `git diff`.
- Cheap to re-index: dropping ChromaDB and reindexing from sidecars is a one-liner.
- Survives schema migrations: you can write small one-off scripts to upgrade old sidecars without touching the vector store.

**Pin a `schema_version` from day one.** Even if it never changes, you'll thank yourself when it does.

**Use Pydantic v2 with `extra="forbid"`** when generating sidecars. A typo in a field name silently no-ops without it.

---

## 2. Pick an embedding model — and pin it everywhere

The biggest cause of "RAG works in the demo but returns garbage in production" is **inconsistent embedding models** across indexing, querying, and re-indexing.

```python
EMBEDDING_MODEL = "sentence-transformers/all-MiniLM-L6-v2"
```

`all-MiniLM-L6-v2` is the right starting default because:

- 384 dims (cheap to store, fast to compare)
- Runs on CPU in seconds for thousands of items
- Battle-tested; most RAG tutorials use it, so you can compare results
- Apache 2.0, no API key

**Critical rule — pin the model in three places:**

1. The indexing script (`index_corpus.py`)
2. The querying script (`query.py`)
3. The downstream clustering or re-ranking scripts

And **store the model name as a ChromaDB collection-level metadata field** so any script can validate before doing work:

```python
collection = client.get_or_create_collection(
    name="corpus",
    metadata={"embedding_model": EMBEDDING_MODEL},
    embedding_function=SentenceTransformerEmbeddingFunction(
        model_name=EMBEDDING_MODEL
    ),
)

# In downstream scripts:
existing_model = collection.metadata.get("embedding_model")
if existing_model != EMBEDDING_MODEL:
    raise SystemExit(
        f"Embedding model mismatch: collection={existing_model} "
        f"script={EMBEDDING_MODEL}. Re-index or fix the constant."
    )
```

This single check catches the most painful class of RAG bug: silently embedding queries with one model and corpus with another, returning answers that look plausible but are 30% wrong.

---

## 3. Indexing — what to embed

Per sidecar, generate the embedding from **the same text shape you'd give a human to skim**:

```python
def sidecar_to_text(sc: dict) -> str:
    parts = []
    if sc.get("tags"):
        parts.append("Tags: " + ", ".join(sc["tags"]))
    if sc.get("tones"):
        parts.append("Tones: " + ", ".join(sc["tones"]))
    if sc.get("subjects"):
        parts.append("Subjects: " + ", ".join(sc["subjects"]))
    if sc.get("notes"):
        parts.append("Notes: " + sc["notes"])
    return "\n".join(parts)
```

Keep the per-document metadata small (filterable fields only) and put the heavy text in `documents`:

```python
collection.add(
    ids=[sc["source_path"]],
    documents=[sidecar_to_text(sc)],
    metadatas=[{
        "category": category_from_path(sc["source_path"]),
        "schema_version": sc["schema_version"],
    }],
)
```

**Use the relative file path as the ID.** It's stable across re-indexes and makes "show me which file matched" trivial.

**Re-index pattern:** add a `--reset` flag that drops the collection then rebuilds it from disk. Cheap, deterministic, lets you re-run after every schema change without thinking.

```bash
python tools/index_corpus.py            # incremental
python tools/index_corpus.py --reset    # nuke + rebuild
```

---

## 4. Querying — the part that surprised me

A naive `collection.query(query_texts=[user_question], n_results=10)` works fine for ~80% of queries. For the other 20% the failures are predictable:

1. **One dominant category drowns the query.** If 60% of your corpus is "category-a", `category-a` items will dominate even when the query is about "category-b". Fix: pass a metadata filter, or run one query per category and merge top-k.
2. **The query is a tag, not a sentence.** "muted palette" against an embedded sentence about "muted color palette" works; "muted" alone against the full corpus is weaker. Either expand the query (`f"reference for {q}"`) or add a keyword pre-filter on tags.
3. **The user actually wanted to retrieve by tag intersection, not similarity.** Don't make every retrieval go through embeddings. For "items tagged X AND Y", scan sidecars directly — faster, exact, no surprises.

A hybrid retriever (one path that does keyword filtering, one path that does vector retrieval, merged with a simple score) consistently outperforms either alone for tagged corpora.

---

## 5. Going beyond retrieval — clustering on top of vectors

Once you have embeddings for the whole corpus, you can do far more than "look up similar items". The pattern that has paid off most:

**UMAP + HDBSCAN to surface hidden categories.**

```python
# tools/cluster_corpus.py
import umap, hdbscan
import numpy as np

# Validate embedding model matches first!
ids, vectors = pull_all_embeddings_from_chroma(collection)

reducer = umap.UMAP(n_components=15, n_neighbors=15, min_dist=0.0, metric="cosine")
reduced = reducer.fit_transform(np.array(vectors))

clusterer = hdbscan.HDBSCAN(min_cluster_size=8, min_samples=3, metric="euclidean")
labels = clusterer.fit_predict(reduced)
```

What you get is a `clusters.json` mapping each item to a cluster id (or `-1` for "noise"), plus a `clusters_summary.json` with the top tags per cluster. This lets you:

- Spot mislabeled items (one item alone in a cluster of a different theme)
- Find emergent sub-categories your tagger never named
- Generate per-cluster prompts/summaries for a follow-up LLM pass

**Numbers that worked on a ~400-item corpus:** `n_neighbors=15`, `n_components=15`, `min_cluster_size=8`. Smaller `min_cluster_size` over-fragments; larger collapses everything into one blob.

**Reproducibility tip:** if you need stable cluster IDs across runs, set `random_state` on UMAP and persist the reduced vectors — HDBSCAN itself is deterministic given the same input.

---

## 6. From RAG to a "profile" — synthesizing across the corpus

The final layer that turns a search engine into something more interesting: **aggregate across the whole corpus to produce a single artifact describing the corpus itself**.

For an image-tag corpus, that's a "taste profile": top tags, top tones, top subjects, weighted by frequency, optionally normalized per cluster. For a code-chunk corpus, that's a "house style profile". For a doc corpus, it's a "topic atlas".

The pattern:

1. **`aggregate.py`** — walks all sidecars, counts every facet (tags, tones, etc.), emits `aggregations.json` with top-N per facet.
2. **`build_profile.py`** — consumes `aggregations.json` + `clusters_summary.json`, emits a single `Profile.json` with weighted tag lists per facet.
3. **(optional) hand-authored layer** — a human-written `Profile.md` that names the clusters, articulates a thesis, and tells the next LLM "act as someone with this taste".

The machine-readable + human-readable split matters. The JSON is for downstream tools (a system-prompt generator, a filter, a recommender). The Markdown is for you and the next LLM you talk to.

---

## 7. Operational gotchas

A scrapbook of things that bit me. None are obvious from any single tutorial.

- **`extra="forbid"` on your Pydantic sidecar schema is load-bearing.** Without it, a typo in a field name (`subject` vs `subjects`) silently drops the value. You'll only notice when retrieval returns nothing for that facet.
- **ChromaDB persistence layer:** `PersistentClient(path="./chroma_db")` is fine for hundreds of thousands of items. Past that, look at a server-mode deployment, but you almost certainly don't need to until you do.
- **Don't put large free-form text in metadata.** Metadata is for filtering. Documents are for embedding + retrieval. Mixing them up bloats the DB and confuses filters.
- **Re-index after every schema change.** Embeddings include all the text you passed in; if you change `sidecar_to_text()`, the old embeddings are now stale. Drop and rebuild — it's cheap.
- **Sentence-transformer downloads:** first run pulls the model from Hugging Face. Cache it (`HF_HOME=...`) if you're going to run on CI or a container.
- **Backup is just a folder copy.** `chroma_db/` is the whole DB. Tarball it before any destructive operation.
- **Don't fight ChromaDB on multi-process writes.** It's not built for it. Have one indexer process; if you need parallel embedding, batch in-memory first and write in one shot.

---

## 8. Skeleton repo layout

```
your-rag-repo/
  corpus/                       # the sidecars + assets
  chroma_db/                    # the persistent vector store (gitignored)
  tools/
    index_corpus.py             # build / rebuild the vector store
    query.py                    # ad-hoc CLI queries
    rag_answer.py               # retrieve + call an LLM with retrieved context
    aggregate.py                # facet counts → aggregations.json
    cluster_corpus.py           # UMAP + HDBSCAN → clusters.json
    build_profile.py            # → Profile.json (corpus-wide synthesis)
  workspace/
    aggregations.json
    clusters.json
    clusters_summary.json
    Profile.json
    Profile.md                  # hand-authored layer
  requirements.txt
  README.md
```

`requirements.txt` essentials:

```
chromadb>=0.4
sentence-transformers>=2.5
pydantic>=2.0
umap-learn>=0.5.6,<0.6.0       # only if you cluster
hdbscan>=0.8                    # only if you cluster
scikit-learn>=1.5
```

---

## See also

- [Pinterest taste extraction with agent-mode image analysis](../tutorials/local-pinterest-taste-extraction-with-agent-mode.md) — the tutorial that produces the sidecar corpus this guide indexes.
- [Image-analysis pipelines with LLMs](./image-analysis-pipelines-with-llms.md) — companion guide on the upstream tagging pipeline.
- [Hybrid agent-workspace pattern](./hybrid-agent-workspace-pattern.md) — repo layout convention this RAG pipeline fits inside.
