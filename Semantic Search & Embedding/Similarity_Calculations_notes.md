# Similarity Calculations — Study Notes

**What they are:** a mathematical way of measuring how close two pieces of text are in meaning, by comparing their embedding vectors (your Embedding Models notes).

**How they work:** treat each embedding as a point (or arrow) in high-dimensional space, and measure the geometric relationship between two such points — the closer/more aligned they are, the more similar the underlying text is judged to be.

**How they matter for RAG:** this is the exact computation behind every "find the closest matches" step you've studied — vector search (HNSW/IVF), semantic chunking's topic-shift detection, and Sentence Transformer-based retrieval all boil down to running a similarity calculation, over and over, at scale.

---

## The Magic of Vector Space

Every embedding is a vector — think of it as an arrow pointing from the origin to a specific point in space. Two arrows pointing in a similar direction represent similar meanings; two arrows pointing in very different directions represent unrelated meanings. **Cosine similarity** measures this by looking at the *angle* between two vectors, ignoring how long each arrow is.

### The Three Cases

| Angle Between Vectors | cos(θ) Value | Interpretation |
|---|---|---|
| **0° — Same direction** | **1** | Maximum similarity — the two texts mean essentially the same thing. |
| **90° apart** | **0** | No similarity — the two texts are unrelated. |
| **180° — Opposite** | **−1** | Maximum dissimilarity — the two texts point in opposite semantic directions. |

In practice, real text embeddings rarely land exactly on these extremes — most similarity scores fall somewhere between 0 and 1, with values closer to 1 meaning "clearly related" and values near 0 meaning "essentially unrelated."

---

## The Formula

```
cos(θ) = (A · B) / (‖A‖ × ‖B‖)
```

Breaking that down:

- **A · B (dot product):** multiply each corresponding pair of numbers in the two vectors, then sum them all up. This captures how much the two vectors "agree" component by component.
- **‖A‖ and ‖B‖ (magnitudes):** the length of each vector — computed as the square root of the sum of its squared components. Dividing by both magnitudes is what removes the effect of vector *length*, leaving only the *direction* (angle) — this is the entire reason cosine similarity is preferred for text (see below).

### Worked Example

Say two very simplified 2D embeddings are:

```
A = [4, 3]
B = [8, 6]
```

Notice B is just A scaled by 2 — same direction, different length.

```
A · B = (4×8) + (3×6) = 32 + 18 = 50
‖A‖   = √(4² + 3²) = √25 = 5
‖B‖   = √(8² + 6²) = √100 = 10

cos(θ) = 50 / (5 × 10) = 50/50 = 1
```

A perfect similarity score of 1 — because even though B is twice as "long" as A, cosine similarity only cares that they point in exactly the same direction. This is the concrete version of "ignores magnitude, focuses on direction."

---

## From Words to Numbers: Embedding (Recap)

Quick reminder of how you get the vectors being compared in the first place (full detail in your Embedding Models and Sentence Transformers notes):

1. **Enter the Embedding Model** — text goes into a trained neural network (e.g., a Sentence Transformer).
2. **Pattern Recognition** — the model has learned, from training on similarity tasks, what patterns in language correspond to similar meaning.
3. **The Transformation** — text comes out the other side as a fixed-size numerical vector, positioned in space according to its meaning.

---

## Why Cosine Similarity (Over the Alternatives)?

| Metric | What It Measures | Behavior on Text of Different Lengths |
|---|---|---|
| **Cosine Similarity** | Angle between vectors (direction only) | Ideal — a short and a long document about the same topic can still score as highly similar, since magnitude is ignored entirely. |
| **Euclidean Distance** | Straight-line distance between the two vector endpoints | Penalizes longer texts — a longer document tends to produce a vector with a larger magnitude, which pushes its endpoint further away in raw distance terms, making it seem *less* similar purely due to length, independent of actual topic overlap. |
| **Dot Product** | Raw agreement between vectors, magnitude included | Fast to compute (no division step), but *not* length-normalized — favors longer vectors, similar failure mode to Euclidean distance. Sometimes used deliberately when vector magnitude is itself meaningful (e.g., some models encode "confidence" or "specificity" in magnitude). |

**Why this specifically matters for text:** a long document and a short document can easily be about the exact same topic — document length is a formatting choice, not a semantic signal. Cosine similarity is the metric that correctly treats length as irrelevant, which is exactly why it's the default choice across your Semantic Chunking, Vector Search, and Sentence Transformer notes.

---

## Why This Matters for RAG

**Without Good Similarity Calculations:**
Retrieval becomes unreliable in ways that are hard to debug — a query might rank a short, tangentially-related snippet above a long, deeply relevant document simply because of a length-sensitive metric, or fail to recognize that two differently-phrased chunks are actually about the same thing. The LLM then generates answers grounded in the wrong context, producing confident-sounding but incorrect or irrelevant responses — exactly the "garbage in, garbage out" failure your Retrieval Methods notes warned about.

**With Good Similarity Calculations:**
Retrieval consistently surfaces chunks that are *actually* about what the query means, regardless of document length or exact phrasing. This is what makes every later stage of the pipeline trustworthy — reranking has a sound shortlist to refine, and the LLM receives context that's genuinely relevant, directly reducing hallucination (as noted in your Embedding Models notes).

---

## Quick Recap

- Similarity calculations measure how close two embeddings are — the computational core of every "find relevant matches" step in RAG.
- Cosine similarity measures the *angle* between two vectors: 1 = same direction (highly similar), 0 = perpendicular (unrelated), −1 = opposite (maximally dissimilar).
- Formula: dot product of the two vectors, divided by the product of their magnitudes — dividing by magnitude is what makes it ignore vector length.
- Cosine similarity beats Euclidean distance and raw dot product for text specifically because document length shouldn't affect how "similar" two pieces of text are judged to be.
- This one calculation underlies vector search, semantic chunking, and Sentence Transformer retrieval alike — get it wrong, and every downstream stage (reranking, generation) inherits the error.
