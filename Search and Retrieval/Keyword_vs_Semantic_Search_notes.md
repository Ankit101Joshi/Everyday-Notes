# Keyword Search vs. Semantic Search — Study Notes

## The Core Difference

- **Keyword Search** matches documents based on shared *exact words* between the query and the document.
- **Semantic Search** matches documents based on shared *meaning*, even if no words overlap at all.

**Mental model:** keyword search asks "does this document contain these words?" Semantic search asks "is this document *about* what I'm asking about?" You've actually already taken detailed notes on the mechanism behind each of these — this note is really about connecting them.

---

## Keyword Search

This is the "sparse" side of retrieval from your Retrieval Methods notes — and TF-IDF / BM25 (your last two notes) are exactly how it works under the hood.

- **How it works:** tokenize the query into words, find documents containing those exact words (or close variants via stemming — "running" matching "run"), and rank by something like BM25.
- **Strength:** excellent at exact matches — product codes, names, error messages, rare technical terms. If a user searches "ERR_CONNECTION_TIMEOUT," they want documents containing that *exact string*, not documents that are vaguely "about connection problems."
- **Weakness:** no understanding of meaning. A search for "automobile" will not match a document that only says "car" — zero word overlap means zero score, even though the concepts are identical (this is the exact limitation you flagged in your TF-IDF notes).

---

## Semantic Search

This is the "dense" side of retrieval from your Retrieval Methods notes — vector search over embeddings.

- **How it works:** an embedding model converts both the query and each document chunk into vectors (points in high-dimensional space) such that similar *meanings* end up close together, regardless of exact wording. Retrieval is then a nearest-neighbor search (your HNSW/IVF notes) over those vectors.
- **Strength:** handles synonyms, paraphrasing, and conceptual similarity. "How do I fix a slow laptop?" can retrieve a document titled "Improving computer performance" even though barely any words match.
- **Weakness:** can struggle with exact-match precision — a rare product code or a specific name may not be well represented in embedding space, since embeddings are built to capture general meaning, not exact tokens. It's also less interpretable — you can't easily explain *why* two vectors ended up close together, whereas with keyword search you can literally point at the shared word.

---

## Side-by-Side Comparison

| | Keyword Search | Semantic Search |
|---|---|---|
| **Matches on** | Exact words / tokens | Meaning / concept |
| **Underlying method** | TF-IDF, BM25 | Embeddings + vector similarity (ANN index) |
| **Good at** | Exact terms, codes, names, rare technical vocabulary | Synonyms, paraphrasing, conceptual queries |
| **Bad at** | Synonyms, paraphrase, "vague" queries | Precise exact-match terms, rare tokens not well represented in embedding space |
| **Speed** | Very fast, cheap, no ML model required at query time | Requires an embedding model + ANN search — more compute |
| **Interpretability** | High — you can see exactly which words matched | Low — similarity is a distance in vector space, hard to explain plainly |
| **Fails silently when** | Query uses different words than the document, even if the meaning is identical | A rare exact term isn't well captured by the embedding space |

---

## Why RAG Systems Use Both (Hybrid Search)

This is the callback to your Retrieval Methods notes: neither approach alone is reliable enough for real-world queries, because their failure modes are near-opposite of each other. **Hybrid search** runs both in parallel and combines their scores, so:

- A query with a specific product code or name still gets an exact match via keyword search, even if the embedding model doesn't represent that token well.
- A vaguely-phrased or paraphrased query still gets matched via semantic search, even if it shares no words with the target document.

This is exactly the "Initial Retrieval: hybrid search returns top-50 candidates based on combined sparse and dense scores" step from your Reranking notes — keyword search is the sparse half, semantic search is the dense half, and reranking is what sorts out the combined shortlist afterward.

---

## Quick Recap

- Keyword search matches exact words (TF-IDF/BM25); semantic search matches meaning (embeddings + vector similarity).
- Keyword search wins on precision for exact terms; semantic search wins on handling synonyms and paraphrase.
- Each one's weakness is the other's strength — which is exactly why hybrid search combines both rather than picking one.
- This note is really the "big picture" view tying together four things you've already studied separately: TF-IDF, BM25, vector search (HNSW/IVF), and hybrid search from your Retrieval Methods notes.
