# Retrieval-Augmented Generation (RAG) — Study Notes

## 1. What Is RAG?

**RAG (Retrieval-Augmented Generation)** solves a core LLM limitation: models only know what was in their training data, and can't access private, recent, or proprietary information. RAG gives the LLM an **"open-book exam"** instead of forcing it to answer from memory alone.

The three-step loop:

| Step | What Happens |
|---|---|
| **Retrieve** | On a user query, search an external knowledge base (PDFs, internal docs, wikis, etc.) for relevant context. |
| **Augment** | Insert that retrieved context directly into the prompt, alongside the user's query. |
| **Generate** | The LLM reads the query + context and produces an answer grounded in real evidence. |

**Mental model:** Retrieval (find it) → Augmentation (hand it over) → Generation (reason over it).

---

## 2. The RAG Pipeline — Two Halves

RAG has an **offline half** (build the knowledge base once) and an **online half** (runs per query).

### A. Offline: Document Ingestion (build the knowledge base)

1. **Chunking** — Documents are split into smaller pieces. This is a genuine trade-off, not just a mechanical step:
   - *Smaller chunks* → higher retrieval **precision**, but risk **losing surrounding context**.
   - *Larger chunks* → better **context preservation**, but risk **retrieval accuracy** (dilutes the embedding, pulls in irrelevant text).
   - In practice: chunk sizes are tuned per use case, often with overlap between chunks to soften the trade-off.
2. **Embedding** — Each chunk is converted into an embedding: a numerical (vector) representation of its *meaning*, not just its words.
3. **Storage** — Embeddings are stored in a **vector database** (e.g., Pinecone, Weaviate, Chroma, pgvector), indexed for fast similarity search.

### B. Online: Retrieval + Generation (runs per query)

1. The **user query** is embedded using the same embedding model — producing a vector representation of its meaning.
2. **Similarity search** compares the query embedding against stored document embeddings (commonly via **cosine similarity**) to return the **top-K** most relevant chunks.
3. Those chunks are **combined with the query** and inserted into a constructed prompt.
4. The LLM **generates a response** grounded in the retrieved evidence.

---

## 3. Improving Retrieval Quality

Naive top-K vector search is often not enough on its own. Common upgrades:

- **Hybrid Search** — Combines keyword search (e.g., BM25) with vector/semantic search. Keyword search catches exact terms (product codes, names) that embeddings can blur; vector search catches paraphrases and meaning. Together they outperform either alone.
- **Reranking** — After initial retrieval, a second model (often a cross-encoder or an LLM) scores and reorders the candidate chunks by true relevance to the query, since the first-pass similarity search is a coarse filter.
- **Query Expansion** — Generates alternative phrasings of the user's query to widen the retrieval net and catch relevant documents that don't share the exact wording of the original question.

---

## 4. Generation With Context

1. **Combine** query + retrieved chunks.
2. **Construct the prompt** — typically with instructions, retrieved context, and the user's question laid out clearly (e.g., "Using only the context below, answer the question...").
3. **Generate** the final, informed response.

---

## 5. Core Failure Modes and Fixes

| Problem | Why It Happens | Solutions |
|---|---|---|
| **Too much / too little context** | Fixed top-K or fixed context windows don't adapt to query complexity. | • Dynamic context window management<br>• Relevance scoring (drop low-relevance chunks)<br>• Intelligent truncation |
| **Hallucination** | LLM ignores retrieved context and falls back on parametric (training-time) knowledge, or fills gaps when context is incomplete. | • Better prompt engineering (explicit grounding instructions)<br>• Model fine-tuning<br>• Confidence scoring mechanisms |
| **Latency** | Embedding, retrieval, reranking, and generation all add sequential time. | • Caching strategies (cache embeddings, common queries)<br>• Async processing<br>• Optimized embedding computation |

---

## 6. Advanced RAG Patterns

| Pattern | Idea |
|---|---|
| **Multi-Step RAG (Self-RAG)** | Iteratively refines searches and applies self-correction — the model can critique its own retrieved context or answer and re-retrieve if needed. |
| **Agentic RAG** | Routes queries to specialized retrievers (e.g., one for docs, one for a SQL database, one for the web) and combines multiple sources into one answer. |
| **Fine-Tuned RAG** | Uses domain-specific embedding models and fine-tuned LLMs, trained on the target domain's vocabulary and data, for higher accuracy than general-purpose models. |

---

## 7. Quick Recap (for recall)

- RAG = Retrieve → Augment → Generate.
- Two phases: **offline ingestion** (chunk → embed → store) and **online query** (embed → search → generate).
- Chunk size is a precision/context trade-off.
- Cosine similarity + top-K is the baseline retrieval method; hybrid search, reranking, and query expansion improve on it.
- Three big failure modes: context sizing, hallucination, latency — each has established mitigations.
- Advanced patterns (Self-RAG, Agentic RAG, Fine-Tuned RAG) push beyond the basic pipeline for harder use cases.
