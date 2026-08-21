# Retrieval Methods — Study Notes

## 1. Why Retrieval Matters

Retrieval is the part of RAG that decides *what the LLM even gets to see*. If retrieval fails, nothing downstream can fix it — the generator can only reason over what it was handed.

| Failure Mode | What It Means |
|---|---|
| **Wrong Chunk Retrieved** | Irrelevant or loosely related content comes back as a "top match" — the LLM then either ignores it or, worse, builds an answer on it. |
| **Stale Index** | The vector index wasn't updated when the source data changed, so retrieval confidently returns outdated information. |
| **ACL Leak** | Retrieval returns a chunk the *user* isn't authorized to see (e.g., another tenant's private document) because access control wasn't enforced at retrieval time. |

**Mental model:** retrieval quality is a ceiling, not a floor — a great LLM with bad retrieval still gives bad answers.

---

## 2. Retrieval Pipeline at a Glance

**Ingest Phase (offline):** chunk, embed, and index the data using an **ANN (Approximate Nearest Neighbor) index** — this is what makes similarity search fast at scale instead of comparing your query against every single vector one by one.

### High Recall Wins — Ingestion Practices That Pay Off Later

*Recall* = how much of the actually-relevant content the system is capable of finding at all (as opposed to *precision*, which is how much of what it *did* return is actually relevant). You fix recall problems at ingestion time — you can't rerank your way out of data you never indexed properly.

1. **Preserve Structure** — keep titles and section headers attached to their content, so a chunk still carries its context.
2. **Maintain Logical Hierarchy** — don't flatten a document's nested structure (chapter → section → subsection) into a single undifferentiated blob.
3. **Add Rich Metadata** — tenant ID, timestamp, ACL tags, etc. This is what lets you *filter* retrieval results (e.g., "only chunks this user is allowed to see") — it's the direct fix for the ACL Leak failure mode above.
4. **Preserve Semantic Relationships** — where possible, keep signals that connect related chunks (e.g., "this chunk continues from the previous one") so retrieval doesn't return an orphaned fragment.
5. **Handle Special Content Separately** — tables, code blocks, and images often need custom parsing/embedding rather than being treated as plain text (a table flattened into a text blob usually loses its meaning entirely).

---

## 3. Vector Search — How "Similarity Search" Actually Works at Scale

Comparing a query vector against *every* stored vector (brute force) is accurate but too slow once you have millions of chunks. **ANN indexes** trade a small amount of accuracy for a large amount of speed. Two common approaches:

| Index | Mental Model | How It Works |
|---|---|---|
| **HNSW** (Hierarchical Navigable Small World) | A multi-layer "small world" graph — like flying into an airport hub, then hopping to smaller regional airports to reach your exact destination. | Vectors are nodes in a graph with links to their nearest neighbors. Search starts at a sparse top layer for fast long jumps, then descends through denser layers making smaller hops until it converges on the closest matches. |
| **IVF / IVF-PQ** (Inverted File Index / with Product Quantization) | Like a library that's pre-sorted books into rough subject bins, then only searches the bins closest to what you're looking for. | Vectors are clustered into buckets ("cells") at index-build time. A query is first matched to the nearest cluster(s), then only vectors *inside* those clusters are compared — skipping everything else. PQ additionally compresses vectors to save memory. |

### The Two Knobs That Matter

- **K (candidates):** how many nearest neighbors to retrieve. Bigger K = more candidates for later stages to work with, but more compute downstream.
- **efSearch (HNSW) / nprobe (IVF):** how thoroughly the index searches before stopping — efSearch controls how many graph nodes get explored; nprobe controls how many clusters get checked.

**The core trade-off:** turning these up increases **recall** (more likely to find the *true* best matches) at the cost of **latency** (search takes longer). Turning them down speeds things up but risks missing genuinely relevant chunks. There's no "correct" setting — it's tuned against your latency budget and your quality bar.

---

## 4. Reranking and Context Packing

Vector search alone is a fast, coarse filter — it's good at *narrowing millions of chunks down to a shortlist*, but not great at *precisely ranking* that shortlist. So production pipelines add a second, more expensive stage:

1. **Initial Retrieval:** Hybrid search (dense/vector + sparse/keyword, e.g., BM25) returns a broad shortlist — commonly the top-50 candidates, combining both scoring methods so you catch both paraphrases (vector) and exact terms like product codes or names (keyword).
2. **Cross-Encoder Rerank:** A slower, more accurate model scores each *query–document pair directly* (rather than just comparing pre-computed embeddings) and re-sorts the shortlist, selecting the top-5 or so with the highest true relevance.
3. **Context Packing:** The final chunks are deduplicated, ordered by relevance and structure, and tagged with citations — so the prompt sent to the LLM is clean, non-redundant, and traceable back to its sources.

**Why it's a two-stage pipeline and not just "rerank everything":** cross-encoders are far too slow to run over millions of chunks directly — you use cheap/fast retrieval to get a shortlist, then spend the expensive compute only on that small shortlist.

### Why Time Budgets Matter

Every stage above — embedding the query, searching the index, reranking, packing context — adds latency, and it's all sequential (each stage waits on the previous one). A pipeline that returns the *perfect* chunk 4 seconds late is still a bad experience for a user expecting a fast answer. This is why real systems set an explicit **latency budget** and tune each stage's "how thorough" knobs (K, efSearch/nprobe, rerank depth) to fit inside it — quality is deliberately traded off against speed, not maximized blindly.

---

## 5. Configuration Knobs (Summary)

These are the levers you actually tune when running a retrieval pipeline in production:

| Knob | What It Controls |
|---|---|
| **Hybrid Weights** | How much influence keyword (sparse) search vs. vector (dense) search has on the combined score. |
| **K Candidates** | How many results come out of the initial retrieval stage. |
| **efSearch / nprobe** | How thoroughly the ANN index searches — recall vs. latency, as above. |
| **Rerank Depth** | How many of the initial candidates actually get passed to the (expensive) reranker. |
| **Caching Strategy** | What gets cached to avoid redundant work — see below. |

---

## 6. Caching Strategy

Caching avoids repeating expensive work when the same (or similar) requests happen again:

1. **Embeddings** — cache the embedding for a given text so it's never computed twice (especially valuable for the query embedding on repeated/common queries).
2. **Candidate Sets** — cache the initial retrieval shortlist for a query, so repeated or near-duplicate queries skip straight to reranking.
3. **Context Packs** — cache the final, packed context for a fully repeated query, so the pipeline can skip retrieval entirely and go straight to generation.

Each layer trades a bit of "freshness risk" (see Mini-SLOs below) for a real cut in latency and compute cost — the further down the pipeline you cache, the bigger the speed win, but the bigger the risk of serving stale results if the underlying data changed.

---

## 7. Mini-SLOs

An **SLO (Service-Level Objective)** is a target you commit to for how a system should behave — e.g., "95% of requests complete in under 500ms." For a retrieval pipeline, the SLOs that matter most are:

| SLO | What It's Targeting |
|---|---|
| **Latency** | How fast retrieval + reranking + packing completes, end to end — directly shaped by the time budget and knob tuning above. |
| **Freshness** | How quickly the index reflects real-world changes to the underlying data — directly at odds with aggressive caching (see Stale Index failure mode). |
| **Security** | Whether access control (ACL tags, tenant filtering) is actually enforced at retrieval time, not just present in the metadata. |
| **Availability** | Whether the retrieval system stays up and responsive under load — a fast, accurate pipeline is worthless if it's down. |

**Mental model:** these four SLOs constantly pull against each other — more caching improves latency but hurts freshness; stricter ACL filtering improves security but can add latency; higher K/efSearch improves quality but hurts latency. Tuning a retrieval system is really about picking where on these trade-offs your use case needs to sit.

---

## Quick Recap

- Retrieval sets a *ceiling* on answer quality — wrong chunks, stale indexes, or ACL leaks can't be fixed later in the pipeline.
- Good ingestion (structure, hierarchy, metadata) drives **recall**; you can't rerank your way out of poor ingestion.
- ANN indexes (HNSW = graph hopping, IVF = clustered buckets) make vector search fast at scale, at a small accuracy cost.
- K and efSearch/nprobe both trade **recall for latency** — there's no universally "correct" setting.
- Production retrieval is two-stage: fast/broad hybrid search (top-50) → slow/precise cross-encoder rerank (top-5) → context packing with citations.
- Every stage adds latency, which is why explicit time budgets and caching (embeddings → candidate sets → context packs) matter.
- Mini-SLOs — Latency, Freshness, Security, Availability — constantly trade off against each other; tuning a retrieval system means choosing where to sit on those trade-offs for your use case.
