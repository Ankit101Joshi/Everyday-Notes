# Limitations of Keyword Search — Study Notes

**What this note covers:**
1. How keyword search actually works under the hood
2. The four common failure modes of keyword-only search
3. How RAG's semantic retrieval improves on each
4. An implementation checklist for building a RAG pipeline that avoids these pitfalls

---

## How Keyword Search Actually Works

### The Inverted Index

The core data structure behind fast keyword search — it's the same *kind* of role an ANN index (HNSW/IVF) plays for vector search: turning a naive slow search into a fast one.

**Forward index (naive/slow):** for each document, store its list of words. To find "piston," you'd have to scan *every document* checking whether it contains that word.

**Inverted index (what's actually used):** flip it — for each *word*, store the list of documents that contain it.

```
"piston"  → [doc3, doc17, doc42]
"engine"  → [doc3, doc9, doc17, doc88]
"the"     → [doc1, doc2, doc3, ... doc9999]  (nearly everything)
```

Now a search for "piston" is a direct lookup, not a scan — you jump straight to its list of matching documents. This is what makes keyword search fast even over huge collections, and it's what BM25/TF-IDF scoring runs on top of.

### Exact vs. Near-Exact Matching

- **Exact match:** the query token and document token are identical strings — "run" only matches "run."
- **Near-exact match:** light normalization is applied before matching — stemming ("running" → "run"), lowercasing, removing punctuation. This catches trivial variations but is still fundamentally string-based — it has no concept of meaning, only of normalized spelling.

---

## TF-IDF and BM25 — Quick Recap

You've already got dedicated notes on both of these, so just the essentials here:

- **TF-IDF** weights a word by how often it appears in a document (TF) versus how rare it is across the whole collection (IDF) — frequent-here, rare-elsewhere words score highest.
- **BM25** ("the modern librarian") refines this with two fixes: **length normalization** (prevents long documents from winning purely by being long) and a **saturation curve** (the 2nd occurrence of a word matters a lot, the 50th barely matters).
- **The saturation effect, side by side:** TF-IDF's score keeps climbing roughly linearly with each repeat of a word; BM25's score climbs fast at first then flattens out — this is exactly what stops keyword stuffing from gaming the ranking (see your BM25 notes for the full formula).

**The "Invisible Librarian" metaphor**, tying the pieces together: the inverted index is the librarian's **organization** system (instant lookup by word), TF-IDF/BM25 is the **mathematics** deciding which matching document is most relevant, and the whole thing runs on the **intuition** that rare + repeated = important. It's a fast, precise, but purely *lexical* librarian — it only knows words, never meaning.

---

## Failure Modes — and How RAG's Semantic Retrieval Fixes Each

### 1. Lexical Mismatch

**The failure:** a query and a relevant document simply use different words for the same concept. Searching "automobile" won't match a document that only ever says "car" — zero token overlap means zero score, no matter how relevant the content actually is.

**How semantic retrieval fixes it:** embeddings place synonyms and paraphrases close together in vector space regardless of exact wording, so "automobile" and "car" end up as near-neighbors even with no shared characters.

### 2. Polysemy and Context Blindness

**The failure:** keyword search matches the *token*, not the *sense* of the word. A search for "bank" returns documents about river banks and financial banks equally, because the string "bank" is identical either way — the algorithm has no way to know which sense the query meant.

**How semantic retrieval fixes it:** an embedding for "bank" is generated *in context* (surrounding words shift the vector), so "river bank" and "savings bank" actually land in different regions of vector space — the query's context disambiguates the match automatically.

### 3. Long Documents and Passage Granularity

**The failure:** a keyword match proves a *document* contains a term somewhere — not that the specific passage matched is actually the relevant part. A 50-page manual might match a query because of one incidental mention on page 40, while the genuinely useful section on page 3 uses different phrasing and never surfaces.

**How semantic retrieval fixes it:** this is really a chunking problem more than a pure retrieval one — RAG retrieves at the *chunk* level (see your Chunking notes), not the whole-document level, so relevance is judged on a focused passage rather than "does this huge document contain the word anywhere."

### 4. Noisy Language

**The failure:** typos, abbreviations, slang, and informal phrasing break exact/near-exact token matching. "gr8" won't match "great"; "how 2 fix" won't cleanly match "how to fix."

**How semantic retrieval fixes it:** embedding models are trained on huge amounts of real-world (often informal) text, so they're considerably more robust to minor spelling variation and casual phrasing than exact string matching — though extreme noise (heavy typos, non-standard abbreviations) can still degrade even semantic matches, which is part of why data hygiene still matters (see checklist below).

---

## Implementation Checklist for Your Own RAG Pipeline

| Step | Why It Matters |
|---|---|
| **1. Use Hybrid Search (BM25 + dense)** | Neither approach's weaknesses are covered by the other's strengths — combining them catches both exact-term queries and paraphrased ones (see your Keyword vs. Semantic Search notes). |
| **2. Optimize Chunking** | Directly addresses the "Long Documents / Passage Granularity" failure — good chunk boundaries (see your Chunking notes) are what let retrieval be precise instead of vague. |
| **3. Choose Appropriate Embeddings** | Different embedding models are trained on different domains/languages — a general-purpose model may underperform on specialized text (legal, medical, code) compared to a domain-tuned one. |
| **4. Apply Cross-Encoder Re-ranking** | Initial hybrid retrieval is still a coarse shortlist — reranking (see your Reranking notes) is what pushes real precision into the final top-N. |
| **5. Consider Query Expansion** | Automatically adding synonyms or related terms to a query before search softens the Lexical Mismatch problem even on the keyword side, not just the semantic side. |
| **6. Ensure Data Hygiene** | Garbage in, garbage out — inconsistent formatting, broken encoding, or duplicate content degrades both keyword and semantic retrieval alike. |
| **7. Maintain Synonym Maps** | A manually curated list of domain-specific equivalent terms (e.g., "MI" ↔ "myocardial infarction") gives keyword search some of semantic search's flexibility, cheaply, for known important terms. |
| **8. Evaluate Thoroughly** | Don't assume any of the above helped — measure retrieval quality on real queries (recall/precision on a labeled test set) before and after each change, the same "test coverage rigorously" principle from your Chunking notes. |

---

## When to Still Use Keyword Search

Semantic and hybrid retrieval don't make pure keyword search obsolete. It's still the right (or at least a necessary) tool when:

- **Exact-match precision is non-negotiable** — product SKUs, legal citation numbers, error codes, or names where a "close in meaning" result is actually useless.
- **Low latency and low cost are critical** — no embedding model or ANN index required, just an inverted index lookup; much cheaper to run at scale.
- **Interpretability/auditability matters** — you can point at the exact word that caused a match, which some regulated or compliance-sensitive contexts require and vector similarity can't provide.
- **The corpus is small or highly structured** — with a small, well-organized collection, keyword search's blind spots (synonyms, paraphrase) are less likely to actually bite, and the simplicity isn't worth trading away.
- **As the sparse half of hybrid search** — even in a fully modern RAG pipeline, keyword search rarely disappears entirely; it becomes a component rather than the whole system.

---

## Quick Recap

- Keyword search runs on an inverted index (word → documents) plus BM25/TF-IDF scoring — fast, precise, but purely lexical (string-based, not meaning-based).
- Its four core failure modes: lexical mismatch (different words, same meaning), polysemy (same word, different meaning), passage granularity (relevant term buried in a big irrelevant document), and noisy language (typos/slang break exact matching).
- Semantic retrieval fixes each by matching on meaning via embeddings, judged at the chunk level, and trained on real-world (often messy) text.
- A solid RAG pipeline doesn't abandon keyword search — it combines it with dense retrieval (hybrid), reranks the result, and still leans on pure keyword search wherever exact-match precision, low latency, or auditability matter most.
