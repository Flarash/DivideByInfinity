# AI pipelines

Retrieval, embeddings, and vision pipelines you can run **locally** over your
own corpus. No managed services required.

## Guides

- [**Setting up a local RAG pipeline you can actually trust**](./local-rag-pipeline.md) — ChromaDB + sentence-transformers (`all-MiniLM-L6-v2`) on disk, sidecar-first corpus shape, embedding-model pinning, UMAP+HDBSCAN clustering, profile synthesis.
- [**Image-analysis pipelines with LLMs — the reality**](./image-analysis-pipelines-with-llms.md) — model tier selection, rate limits, checkpointing, Pillow downscale economics, manifest-based resumption, 12-point pre-flight checklist.

## See also

- Tutorial: [Pinterest taste extraction with agent-mode image analysis](../../tutorials/local-pinterest-taste-extraction-with-agent-mode.md) — the concrete walkthrough these guides are the reference layer for.
