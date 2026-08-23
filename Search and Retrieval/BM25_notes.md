# BM25 Algorithm — Study Notes

## The Problem: Length Bias in TF-IDF

Recall from your TF-IDF notes: TF is just raw word count divided by document length. This creates real problems in practice:

- **Favors long or repetitive documents** — a document that's simply longer has more chances to repeat any given word.
- **Repeated terms can dominate** — TF scales roughly linearly, so a word appearing 50 times scores ~5x a word appearing 10 times, even though the *10th* occurrence probably added more new information than the *50th*.
- **Enabled early SEO keyword stuffing** — stuffing a page with a keyword artificially inflated its TF-IDF score, which is exactly why early search engines were so easy to game before ranking algorithms improved.

BM25 was designed specifically to fix this.

---

## BM25 = "Best Match 25"

An improved ranking function built on TF-IDF's core idea (frequent-in-doc, rare-in-library = important), but with two key fixes:

1. **Counts how often a word appears** — same starting point as TF.
2. **Boost rises but quickly tapers (saturation)** — the *first* few occurrences of a word matter a lot; the *next* fifty barely move the needle. This directly fixes the "repeated terms dominate" problem.
3. **Prevents long documents from an unfair advantage (length normalization)** — a document's score is adjusted against the *average* document length in the collection, so being long no longer automatically means scoring higher.

**Mental model:** TF-IDF asks "how many times does this word appear?" BM25 asks "how many times does this word appear, *relative to what's normal for a document this length, and with diminishing returns per repeat*?"

---

## Break-down of the BM25 Formula

For a query term and a document:

```
BM25(word, doc) = IDF(word) × [ TF(word, doc) × (k1 + 1) ] / [ TF(word, doc) + k1 × (1 - b + b × (docLen / avgDocLen)) ]
```

Piece by piece:

| Term | What It Does |
|---|---|
| **IDF(word)** | Same rarity idea as TF-IDF — common words score low, rare words score high. |
| **TF(word, doc)** | Raw count of the word in this document — the starting signal, same as before. |
| **k1** (typically ~1.2–2.0) | Controls how quickly term-frequency **saturates**. Higher `k1` = TF keeps mattering longer before flattening out; lower `k1` = the score plateaus almost immediately after the first occurrence. This is what produces the "boost rises but quickly tapers" curve. |
| **b** (typically ~0.75, range 0–1) | Controls how strongly **document length** is normalized against the average. `b = 1` = full length normalization; `b = 0` = no length normalization at all (behaves more like plain TF-IDF on this front). |
| **docLen / avgDocLen** | The document's length relative to the average document in the collection — this ratio is what penalizes documents that are long *simply because they're long*, not because they're more relevant. |

**Why this fixes length bias:** if a document is twice the average length, the denominator grows, which shrinks the contribution of any repeated term — a long document has to earn a high score through genuine relevance, not just word count.

**Why this fixes keyword stuffing:** because of `k1`'s saturation, going from 5 occurrences of a word to 50 barely changes the score — stuffing a page with a keyword stops being a cheap way to game ranking.

---

## BM25 Workflow

```
Count term frequency (TF)
   │
   ▼
Normalize (adjust for document length, via b and docLen/avgDocLen)
   │
   ▼
Adjust for saturation (diminishing returns on repeats, via k1)
   │
   ▼
Boost for rarity (multiply by IDF)
   │
   ▼
Final BM25 score
```

Each step maps directly onto a piece of the formula above — this is really just the formula read left to right as a sequence of adjustments to raw term frequency.

---

## Where This Fits in Your RAG Notes

- **TF-IDF notes:** you already flagged that BM25 is TF-IDF's refined successor — now you have the actual mechanism for *how* it's refined (saturation + length normalization).
- **Retrieval Methods notes:** BM25 is what's typically meant by the "sparse" half of hybrid search in real systems (Elasticsearch, OpenSearch, and Postgres full-text search all use BM25 or close variants) — paired with dense/vector retrieval to catch both exact keyword matches and semantic paraphrases.

---

## Quick Recap

- TF-IDF's raw term count causes two problems: long/repetitive documents score unfairly high, and repeated terms scale almost linearly — both exploitable by keyword stuffing.
- BM25 fixes both with two tunable parameters: `k1` controls how quickly term-frequency saturates (diminishing returns per repeat), `b` controls how strongly document length gets normalized against the collection average.
- Workflow: count TF → normalize for length → apply saturation → boost by IDF → final score.
- BM25 is the modern default for "sparse" keyword retrieval — it's what search engines actually use today, and it's the standard sparse half of hybrid search in RAG pipelines.
