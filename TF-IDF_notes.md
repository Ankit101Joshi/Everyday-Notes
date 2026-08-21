# TF-IDF Algorithm — Study Notes

## Intuition Behind TF-IDF

**Term Frequency–Inverse Document Frequency** is a scoring method for how important a word is to a specific document, relative to a whole collection of documents. The core idea: common words get low scores, rare (but document-relevant) words get high scores.

**Mental model:** a word that shows up constantly *everywhere* (like "the") tells you nothing distinctive about any one document. A word that shows up rarely across the collection, but repeatedly *in this document*, is a strong signal of what that document is actually about.

### The Pipeline

```
Input search terms
   │
   ▼
Split text into words (tokenize)
   │
   ▼
Count words per document (TF)
   │
   ▼
Measure rarity across the library (IDF)
   │
   ▼
Multiply TF × IDF
   │
   ▼
Rank documents by score
```

---

## Term Frequency (TF)

**"How loud a word is inside its own document."** The more a word repeats within a document, the more it signals what that document is about.

**Formula:**

```
TF(word, doc) = (number of times word appears in doc) / (total number of words in doc)
```

Dividing by the document's total word count matters — otherwise a longer document would score higher on every word just by virtue of having more words, not because any word is actually more important to it.

---

## Inverse Document Frequency (IDF)

**Zooms out** — instead of asking "how important is this word *in this document*," it asks "how many documents in the whole library contain this word at all?"

- **Many documents contain it** → small IDF value. Example: "the," "is," "a" — these appear almost everywhere, so they're bad at distinguishing documents from each other.
- **Few documents contain it** → large IDF value. Example: "piston" — if only a handful of documents mention it, its presence is a strong, distinctive signal.

**Formula:**

```
IDF(word) = log( N / number of documents containing word )
```

where `N` is the total number of documents in the library. The `log` matters: without it, a word appearing in just 1 out of 1,000,000 documents would produce a raw ratio of 1,000,000 — the log compresses that into a much more usable, less extreme scale.

---

## The Formula (Combined)

```
TF-IDF(word, doc) = TF(word, doc) × IDF(word)
```

**Frequent in the document, rare in the library → high score:**

```
High TF × High IDF = High TF-IDF
```

A word only scores highly if *both* conditions hold — repeating a common word a lot (high TF, low IDF) still nets a low score, and a rare word that only appears once in this one document (low TF, high IDF) doesn't score as high as a rare word this document really emphasizes.

### Worked Example

Say the word **"piston"** appears 3 times in a 100-word document, and out of 10,000 documents in the library, only 10 contain "piston" at all:

```
TF   = 3 / 100        = 0.03
IDF  = log(10,000/10) = log(1,000) ≈ 3
TF-IDF = 0.03 × 3      ≈ 0.09
```

Compare that to the word **"the"**, which appears 8 times in the same 100-word document, but shows up in essentially all 10,000 documents:

```
TF   = 8 / 100         = 0.08
IDF  = log(10,000/9,900) ≈ log(1.01) ≈ 0.004
TF-IDF = 0.08 × 0.004   ≈ 0.0003
```

Even though "the" appears more often in the document, its near-zero IDF crushes its score — "piston" ends up far more important to how this document gets ranked, which matches the intuition.

---

## Where You See TF-IDF in Real Life

| Use Case | Why TF-IDF Fits |
|---|---|
| **Classic Search Engines** | Ranks documents against a query by summing TF-IDF scores of the query's words — the original approach behind keyword search, before dense retrieval existed. |
| **Keyword Extraction** | The words with the highest TF-IDF score in a document are, by construction, its most distinctive terms — a cheap way to auto-tag or summarize content. |
| **Quick Baselines** | Because it requires no training and no embedding model, it's a fast, interpretable baseline to compare fancier retrieval methods against. |
| **Deduplication** | Documents with very similar TF-IDF vectors are likely near-duplicates — useful for flagging redundant content before it clutters an index. |
| **Content Analysis** | Comparing TF-IDF profiles across documents/time periods can surface shifting topics or terminology (e.g., which terms are trending in a corpus). |

---

## Where This Fits in RAG (Link to Retrieval Methods)

Recall your Retrieval Methods notes: hybrid search combines **sparse** (keyword-based) and **dense** (vector/embedding-based) retrieval. TF-IDF — or more precisely its refined successor, **BM25** — *is* the sparse half.

- **BM25** builds directly on TF-IDF's idea but adds term-frequency saturation (a word's 10th occurrence adds much less signal than its 2nd) and document-length normalization. It's what most production "sparse" retrievers actually use today.
- This is exactly why hybrid search is valuable: sparse (TF-IDF/BM25) is excellent at exact keyword/term matches — a product code, a name, a rare technical term — while dense retrieval is better at *paraphrases* and *meaning* that share no exact words. Combining both catches cases either one alone would miss.

### Limitations of TF-IDF (Why Vector Search Exists at All)

- **No understanding of meaning** — "car" and "automobile" get completely unrelated scores, even though they mean the same thing.
- **No word order or context** — "dog bites man" and "man bites dog" produce identical TF-IDF vectors.
- **Struggles with synonyms and paraphrase** — this is precisely the gap embedding-based (dense) retrieval was built to close.

---

## Quick Recap

- TF-IDF scores a word's importance to a document by multiplying how often it appears *in that document* (TF) by how rare it is *across the whole library* (IDF, with a log to tame extreme ratios).
- High score = frequent in this document, rare everywhere else.
- It's the classic, training-free "sparse" retrieval method — still used today via its refined descendant, BM25, as the keyword half of hybrid search.
- Its core weakness — no grasp of meaning, synonyms, or word order — is exactly the gap that dense/vector retrieval was built to fill.
