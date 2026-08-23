# Reranking - STEP 2 — Study Notes

## 1. What Is Reranking?

**Reranking** is a second-pass scoring step that takes the chunks an initial retrieval already pulled back, and re-orders them by *true* relevance to the query using a more accurate (but slower) model.

**Why it exists:** initial vector retrieval (bi-encoder search over an ANN index — see your Retrieval Methods notes) is fast because query and document embeddings are computed *independently* and compared with a simple similarity score. That speed comes at a cost: it's a coarse approximation of relevance. A **reranker** instead looks at the query and a candidate chunk *together*, so it can judge relevance far more precisely — it's just too slow to run over millions of chunks, which is why it only runs on the shortlist retrieval already narrowed down.

**Mental model:** initial retrieval is a wide, fast net; reranking is a careful second look at what the net caught.

### The Pipeline

```
User query
   │
   ▼
Vector Database  →  Initial Retrieval (top-K candidates, fast/coarse)
   │
   ▼
Reranker  →  scores each (query, chunk) pair directly, more accurate/slower
   │
   ▼
Top-N selected (small, high-precision set)
   │
   ▼
LLM generates the answer, grounded in these top-N chunks
```

The reranker sits *between* retrieval and generation — it never touches the LLM's output, only the chunks the LLM is about to receive.

---

## 2. Reranker Benefits

| Benefit | Why It Happens |
|---|---|
| **Improves Relevance and Specificity** | Because the reranker scores the query and chunk *jointly* (rather than comparing two independently-computed embeddings), it can pick up on nuance — like a chunk that shares vocabulary with the query but answers a different question — that similarity search alone often misses. |
| **Filters Noisy Retrievals** | Initial retrieval is tuned for recall (cast a wide net, don't miss anything), so it's expected to include some irrelevant or loosely-related chunks in its top-K. Reranking's job is to push those down or out before they ever reach the LLM. |
| **Picks Better Context for LLMs** | LLMs are sensitive to *what* they're given — even a few irrelevant or misleading chunks in the prompt can pull an answer off track. Reranking narrows top-K down to a small, high-confidence top-N, so the LLM's context window is spent almost entirely on genuinely useful material. |

---

## 3. Best Scenarios for Reranking

Reranking adds real latency and compute cost, so it earns its place when:

- **Precision matters more than raw speed** — e.g., customer support, legal/medical answers, or anything where a wrong context chunk could produce a wrong or misleading answer.
- **Initial retrieval is noisy** — large, diverse, or loosely-organized corpora where top-K from vector search alone regularly includes off-topic chunks.
- **The corpus has many near-duplicate or overlapping chunks** — reranking is good at distinguishing "close in embedding space" from "actually answers the question."
- **You're using hybrid search** (keyword + vector, from your Retrieval Methods notes) — reranking is a natural next step to fuse and re-order the combined candidate set from both methods.

**When it's probably not worth it:** ultra-low-latency use cases, small/clean corpora where initial retrieval is already precise, or early prototyping where the extra pipeline complexity isn't yet justified by evidence of a quality problem.

---

## 4. Reranking vs. Initial Retrieval

| | Initial Retrieval | Reranking |
|---|---|---|
| **Model type** | Bi-encoder — query and document embedded *separately* | Cross-encoder (or LLM-based scorer) — query and document processed *together* |
| **Speed** | Fast — enables search over millions of chunks via an ANN index | Slow — impractical at full-corpus scale |
| **Scope** | Runs over the entire indexed corpus | Runs only over the shortlist retrieval already returned (e.g., top-50) |
| **Optimized for** | Recall — don't miss potentially relevant chunks | Precision — correctly rank what's actually most relevant |
| **Output** | A broad candidate set (top-K) | A narrow, high-confidence final set (top-N, N < K) |

**Mental model:** initial retrieval answers "what *might* be relevant?" across everything; reranking answers "what *is* relevant?" across a much smaller, pre-filtered set. They're complementary stages, not competing methods — one without the other is either too slow (reranking everything) or too imprecise (skipping reranking entirely).

---

## Quick Recap

- Reranking is a second-pass, more-accurate-but-slower scoring step applied to the shortlist from initial retrieval, not to the LLM's output.
- It exists because fast retrieval (bi-encoder/ANN) is a coarse approximation; a cross-encoder judges query + chunk together for real precision.
- Benefits: better relevance/specificity, filters noisy candidates, gives the LLM a cleaner context window.
- Best used when precision matters, the corpus is large/noisy, or you're already doing hybrid search — skip it for latency-critical or already-clean small corpora.
- Initial retrieval optimizes for recall over the whole corpus; reranking optimizes for precision over a small shortlist. They're two stages of one pipeline, not alternatives.
