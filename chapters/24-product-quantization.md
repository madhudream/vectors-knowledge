---
title: "Product Quantization"
chapter: 24
part: "Part IV — Index Structures"
slug: product-quantization
readingTime: "15 min"
summary: "Chop a vector into pieces, replace each piece with the nearest entry from a small codebook, and store only the codebook IDs. 3 KB becomes 96 bytes and search still works."
tags: [product-quantization, pq, compression, faiss, adc]
prev: 23-ivf
next: 25-ivfpq-and-opq
---

# Product Quantization

**The one-paragraph version.** Cut a 768-dimensional vector into 96 small pieces of 8
numbers each. For each piece position, keep a short list of 256 typical pieces, called a
**codebook**. Then store each piece as one byte: the ID of the closest entry in its
codebook. The vector shrinks from 3,072 bytes to 96 bytes, which is **32× smaller**. When a
query arrives, we score it against these tiny codes with a lookup table, without ever
rebuilding the full vectors.

In this chapter, we will learn about product quantization (PQ), a way of storing vectors in a
fraction of the memory while still being able to search them. We will also see how the
encoding works step by step, how a query is scored against codes, what PQ costs in accuracy,
and when to use it.

We will cover the following:

- What is product quantization
- Why we need product quantization
- How product quantization works
- An example of product quantization
- Choosing the number of pieces
- Problems with product quantization
- When to use product quantization

---

## What is product quantization

Before jumping into PQ, we must know what **quantization** means. Chapter 4 gave it a short
gloss. Here is the full idea.

**Quantization = replacing a precise value with the nearest value from a small, fixed set.**

Rounding £12.47 to £12 is quantization. We lose some detail, and we gain a much shorter
description.

**Product quantization = cut each vector into small pieces, and store each piece as the ID
of its nearest entry in a small list of typical pieces.**

Let's break it down.

- A **subvector** is one of those pieces. A 768-number vector cut into 96 pieces gives 96
  subvectors of 8 numbers each.
- A **codebook** is the list of typical pieces for one position. Subvector 0 has its own
  codebook of 256 entries. So does subvector 1, and so on.
- A **code** is the ID of the closest codebook entry. With 256 entries, an ID fits in one
  byte (0 to 255).

In simple words, we stop storing the numbers and start storing "which typical piece is
closest".

Think of it like describing what someone is wearing using a clothing catalogue.

- The whole outfit is the vector.
- Each garment slot (hat, shirt, trousers, shoes) is a subvector.
- The catalogue page for that slot, with 256 standard hats or 256 standard shirts, is a
  codebook.
- "Shirt #204" is a code.

So an outfit becomes four numbers: `hat #17, shirt #204, trousers #88, shoes #131`. Four
bytes describe the whole outfit. Shirt #204 is only the closest match in the catalogue, not
the exact shirt. If the catalogue is well chosen, it is very close indeed.

**Note:** The word "product" has nothing to do with shopping. It is the mathematical
*Cartesian product*. The set of all possible outfits is every hat combined with every shirt,
every pair of trousers and every pair of shoes. With 256 options in each of 4 slots, we can
express $256^4 \approx 4.3$ billion outfits using only 4 bytes. That multiplication is the
whole trick, and we will see why it matters in the next section.

---

## Why we need product quantization

Let's go back to Acme. Our Great Library of 40,000 pages takes about 123 MB as 768-d
`float32` vectors. That fits anywhere.

Now look ahead to Part VIII. Acme's knowledge-base product grows until it hosts knowledge
bases for 5,000 customer companies, about 100 million chunks. At 3,072 bytes each, the Card
Catalog now needs **307 GB of RAM** just for the vectors. That is a very large and very
expensive machine.

So how do we make each vector much smaller without losing the ability to search?

Let's first try the obvious way. We run k-means (Chapter 23) over whole vectors and keep 256
centroids, typical whole vectors. Each chunk is then stored as one byte: the ID of its
nearest centroid.

The Librarian asks, *"Does the Pro plan include single sign-on?"*

Page 212 ("SSO is included on the Pro plan") lands on centroid #37, the one for login and
security pages. Page 1,140 ("On Pro, SSO needs the Security add-on for teams under 50 seats")
lands there too.

With only 256 codes for 100 million chunks, about 390,000 chunks share each code on average.
Every one of them gets exactly the same score, because the query is compared with centroid
#37, not with the chunks themselves. The Librarian sees a giant tie between login pages and
picks five more or less at random. Page 212 happens to be picked. Page 1,140 is not.

The answer is wrong. The Scholar reads page 212 alone and says SSO is simply included.

To match the resolution of 96 bytes with a single codebook, we would need $256^{96}$
centroids. Nobody can train or store that.

Here is what PQ does instead. It cuts each vector into 96 pieces and gives each piece its own
small codebook of 256 entries. That is only $96 \times 256 = 24{,}576$ small centroids to
train. Yet the number of different codes it can express is $256^{96}$, the same as the
impossible single codebook.

Page 212 and page 1,140 now agree on many pieces, but not on all 96. Their scores differ,
both rank near the top, and the Librarian brings back both pages. The answer is correct.

In simple words, PQ gets the resolution of an astronomically large codebook from a pile of
small ones.

---

## How product quantization works

PQ has three phases. The first happens once. The second happens for every vector we store.
The third happens for every query.

**Phase 1: Training the codebooks.**

**Step 1:** Take a representative sample of our vectors, say 100,000 of them.

**Step 2:** Cut every sample vector into $m$ subvectors of $d/m$ numbers. For $d = 768$ and
$m = 96$, that is 96 pieces of 8 numbers.

```
768-d vector, m = 96:
[ d0..d7 ][ d8..d15 ][ d16..d23 ] ... [ d760..d767 ]
  sub 0     sub 1      sub 2            sub 95
```

**Step 3:** Collect all the sample pieces for position 0. Run k-means on them with
$k = 256$. The 256 centroids are codebook 0.

**Step 4:** Repeat Step 3 for positions 1 to 95. We now have 96 codebooks, each holding 256
centroids of 8 numbers.

**Phase 2: Encoding a vector.**

**Step 1:** Cut the vector into the same 96 subvectors.

**Step 2:** For subvector 0, measure its distance to all 256 centroids in codebook 0.

**Step 3:** Store the ID of the closest centroid as one byte.

**Step 4:** Do the same for the other 95 subvectors.

**Step 5:** Keep the 96 bytes and throw the original away (or move it to cheap storage).

```
original: 768 floats × 4 bytes = 3,072 bytes
encoded:  96 bytes                        ← 32× smaller
```

If we ever need an approximate vector back, we glue together the 96 centroids that the codes
point to. The gap between the original vector and this approximation is called the
**residual**. In simple words, the residual is the part PQ got wrong. We will rarely rebuild
vectors in practice, because of Phase 3.

**Phase 3: Scoring a query with ADC.**

**ADC = Asymmetric Distance Computation.** The query stays at full precision. Only the stored
vectors are compressed.

**Step 1:** Cut the query into the same 96 subvectors.

**Step 2:** For each position, compute the score between the query's piece and all 256
centroids in that codebook. This fills a $96 \times 256$ table, about 25,000 entries. We
compute it once per query, and each entry is an 8-number dot product.

**Step 3:** For each stored vector, read its 96 codes.

**Step 4:** Look up one table entry per code and add the 96 values. That sum is the
approximate score.

**Step 5:** Keep the highest-scoring vectors.

```python
# Once per query
tables = np.stack([codebooks[i] @ q_sub[i] for i in range(m)])   # (m, 256)

# Per stored vector: 96 lookups and an add. No multiplication.
score = tables[np.arange(m), codes[doc_id]].sum()
```

In simple words, all the multiplying happens once, up front, in the table. Scoring each
stored vector is then 96 memory lookups and 95 additions, with no multiplications at all.
Reading 96 bytes instead of 3,072 helps too. Together, these can make a PQ scan several times
faster than an exact scan, on top of being 32× smaller.

**Note:** "Asymmetric" means only one side is compressed. We could compress the query as well
and compare code with code. That throws away extra detail for no gain, because the query is
encoded only once per search anyway. Keeping the query exact usually gives noticeably less
error.

---

## An example of product quantization

Real vectors have 768 numbers, which is too many to follow by hand. So let's shrink
everything and keep the same shape.

Our toy vectors have **8 numbers**. We cut them into **4 subvectors** of 2 numbers. Each
position has a codebook of **4 centroids**, so each code needs only 2 bits. Four codes of 2
bits are 8 bits, which is one byte.

Here are the four codebooks. Treat them as already trained.

| Codebook | Centroid 0 | Centroid 1 | Centroid 2 | Centroid 3 |
|---|---|---|---|---|
| position 0 | [0.8, 0.2] | [0.1, 0.9] | [-0.5, 0.4] | [0.3, -0.6] |
| position 1 | [0.9, 0.0] | [0.0, 0.9] | [0.5, 0.5] | [-0.6, -0.2] |
| position 2 | [0.2, 0.2] | [0.7, 0.6] | [-0.4, 0.8] | [0.6, -0.5] |
| position 3 | [0.0, 0.7] | [0.6, 0.1] | [0.9, 0.8] | [-0.3, -0.3] |

And here are three toy page vectors from the Library:

| Page | Vector |
|---|---|
| 212 (SSO included on Pro) | [0.7, 0.3, 0.4, 0.6, 0.8, 0.5, 0.9, 0.9] |
| 1,140 (SSO needs the add-on) | [0.8, 0.1, 0.9, 0.2, 0.6, 0.7, 0.7, 0.8] |
| 87 (two-factor login on Basic) | [0.2, 0.8, 0.1, 0.8, 0.3, 0.1, 0.5, 0.2] |

**Encoding page 212.**

**Step 1:** Cut it into pieces: `[0.7, 0.3]`, `[0.4, 0.6]`, `[0.8, 0.5]`, `[0.9, 0.9]`.

**Step 2:** Compare piece 0, `[0.7, 0.3]`, with codebook 0. We use squared distance: subtract
position by position, square the gaps, add them.

```
to centroid 0 [0.8, 0.2]:  (0.7-0.8)² + (0.3-0.2)² = 0.01 + 0.01 = 0.02   ← closest
to centroid 1 [0.1, 0.9]:  (0.6)² + (-0.6)²          = 0.36 + 0.36 = 0.72
to centroid 2 [-0.5, 0.4]: (1.2)² + (-0.1)²          = 1.44 + 0.01 = 1.45
to centroid 3 [0.3, -0.6]: (0.4)² + (0.9)²           = 0.16 + 0.81 = 0.97
```

So piece 0 gets code **0**.

**Step 3:** Do the same for the other pieces. Piece 1 is closest to centroid 2
(distance 0.02). Piece 2 is closest to centroid 1 (0.02). Piece 3 is closest to centroid 2
(0.01).

**Step 4:** Store the codes. Page 212 becomes **[0, 2, 1, 2]**.

The same steps turn page 1,140 into **[0, 0, 1, 2]** and page 87 into **[1, 1, 0, 1]**. Notice
that pages 212 and 1,140 share three codes out of four. They are similar pages, so most of
their pieces land on the same typical pieces. They still differ in one place, and that one
place is what keeps them apart.

The storage bill: 8 `float32` numbers take 32 bytes. Four 2-bit codes take 1 byte. That is
32× smaller, the same ratio as 3,072 bytes to 96 bytes in the real thing.

**Scoring the query.**

The Librarian's question, *"Does the Pro plan include single sign-on?"*, has the toy vector
`[0.8, 0.2, 0.5, 0.5, 0.7, 0.6, 0.6, 0.7]`.

Encoding used distance, to find the closest typical piece. Scoring uses the dot product (Chapter 4),
the similarity we rank by. First, we build the ADC table. Each entry is the dot product of the
query's piece with one centroid. For example, position 0, centroid 0 is `0.8×0.8 + 0.2×0.2 = 0.68`.

| Position | Centroid 0 | Centroid 1 | Centroid 2 | Centroid 3 |
|---|---|---|---|---|
| 0 | 0.68 | 0.26 | -0.32 | 0.12 |
| 1 | 0.45 | 0.45 | 0.50 | -0.40 |
| 2 | 0.26 | 0.85 | 0.20 | 0.12 |
| 3 | 0.49 | 0.43 | 1.10 | -0.39 |

Then, for each page, we read one entry per code and add them up.

```
page 212,   codes [0, 2, 1, 2]:  0.68 + 0.50 + 0.85 + 1.10 = 3.13
page 1,140, codes [0, 0, 1, 2]:  0.68 + 0.45 + 0.85 + 1.10 = 3.08
page 87,    codes [1, 1, 0, 1]:  0.26 + 0.45 + 0.26 + 0.43 = 1.40
```

The algorithm thinks like this: *"Page 212 and page 1,140 both score above 3. Page 87 is far
behind. The top two are 212 and 1,140."*

Let's check against the exact dot products with the uncompressed vectors: page 212 scores
3.15, page 1,140 scores 3.03, and page 87 scores 1.48. The PQ scores are a little off, but the
order is the same. The Librarian returns both SSO pages. The answer is correct.

And that is PQ in full: cut, look up the nearest typical piece, store the IDs,
and score by adding table entries.

---

## Choosing the number of pieces

The main setting in PQ is $m$, the number of subvectors. With 8-bit codes, each subvector
costs one byte, so $m$ is also the size of each stored vector in bytes.

For 768-dimensional vectors:

| $m$ (subvectors) | Bytes/vector | Compression | Illustrative recall@10 (no rescoring) |
|---|---|---|---|
| — (no PQ) | 3,072 | 1× | 1.00 |
| 192 | 192 | 16× | ~0.95 |
| 96 | 96 | 32× | ~0.90 |
| 48 | 48 | 64× | ~0.80 |
| 24 ($d/m = 32$, outside the safe range below) | 24 | 128× | ~0.60 |

The recall numbers are illustrative. Real values depend on the data and the model, so measure
them on your own queries (Chapter 19).

Two rules follow from how PQ is built.

**More subvectors means more accuracy and less compression.** Each subvector gets its own
256-entry codebook, so more of them means a finer description. Also, $m$ must divide $d$
exactly.

**Each subvector should stay small.** With 8 numbers per piece, 256 centroids cover the
piece's space reasonably well. With 64 numbers per piece, 256 centroids are hopelessly
spread out. That is the curse of dimensionality from Chapter 6, now happening *inside* each
piece. Keep $d/m$ between 4 and 16.

---

## Problems with product quantization

What does PQ cost us? There are four costs.

**Problem 1: The scores are approximate.** In our example, page 1,140's score moved from 3.03
to 3.08. With 100 million chunks, small errors like this shuffle the order near the top. A
page that belongs at rank 3 can drop to rank 14 and miss a top-10 cut.

**Problem 2: It needs training, and the training can go stale.** The codebooks are learned
from a sample. If Acme's customers start writing about a brand-new product, the new pages
fit the old codebooks badly, and accuracy decays quietly.

**Problem 3: Bad settings ruin it.** If we choose $m$ so that each piece has 64 numbers, the
codebooks cannot describe the pieces well, and every score gets worse.

**Problem 4: On its own, it is still a scan.** PQ makes each comparison cheap, but it still
compares the query with every stored vector. For 100 million chunks, that is 100 million sets
of 96 lookups per query.

Each problem has a standard fix. For Problem 1, we **rescore**: take a wider shortlist from
PQ, then re-check those few candidates with the full, uncompressed vectors (Chapter 25 covers
this in detail). For Problem 2, we sample widely and retrain now and then. For Problem 3, we
keep $d/m$ between 4 and 16. For Problem 4, we put PQ inside another index, so that we score
only a small part of the collection.

---

## When to use product quantization

Let's compare PQ with the flat index from Chapter 17, which stores every vector in full and
checks them all.

| | Flat index (float32) | PQ, $m = 96$ |
|---|---|---|
| Bytes per 768-d vector | 3,072 | 96 |
| 100 million chunks | 307 GB | 9.6 GB |
| Scores | Exact | Approximate |
| Training | None | Codebooks (k-means per position) |
| Cost per comparison | 768 multiply-adds | 96 lookups and adds |

**Advantages of PQ:** 32× less memory at $m = 96$ (16× to 64× with $d/m$ in the safe range),
scoring with no multiplication, and search directly on the codes.

**Disadvantages of PQ:** approximate scores that need rescoring, codebooks to train and
refresh, and, on its own, a scan that still touches every vector.

We must use a **flat index** when the collection fits comfortably in memory, as Acme's 40,000
pages do. We must use **PQ** when memory is the limit, as it is for 100 million chunks.

**PQ is almost never used alone.** Its natural home is inside IVF (Chapter 25), where
clustering decides *which* vectors we score and PQ makes each score cheap. It also lives
inside DiskANN (Chapter 32, an index that keeps most of its data on SSD), where PQ codes sit
in RAM for navigation and full vectors sit on SSD for rescoring. Many strong systems use PQ
this way, as one component of a larger index.

---

### Under the hood

Here is a complete PQ in about thirty lines. It follows the three phases above: `train` is
Phase 1, `encode` is Phase 2 and `search` is Phase 3.

```python
import numpy as np
from sklearn.cluster import KMeans

class PQ:
    def __init__(self, d, m=96, nbits=8):
        assert d % m == 0
        self.d, self.m, self.k, self.dsub = d, m, 2 ** nbits, d // m

    def train(self, V):                                    # Phase 1
        self.codebooks = np.empty((self.m, self.k, self.dsub), dtype=np.float32)
        for i in range(self.m):
            sub = V[:, i*self.dsub:(i+1)*self.dsub]
            self.codebooks[i] = KMeans(self.k, n_init=3).fit(sub).cluster_centers_

    def encode(self, V, batch=20_000):                     # Phase 2 → (n, m) uint8
        codes = np.empty((len(V), self.m), dtype=np.uint8)
        for i in range(self.m):
            C = self.codebooks[i]
            c_sq = (C ** 2).sum(1)                         # nearest-centre trick, Chapter 23
            for s in range(0, len(V), batch):              # batches keep memory small
                sub = V[s:s+batch, i*self.dsub:(i+1)*self.dsub]
                codes[s:s+batch, i] = (c_sq - 2 * sub @ C.T).argmin(1)
        return codes

    def search(self, q, codes, k=10):                      # Phase 3 (ADC)
        tables = np.empty((self.m, self.k), dtype=np.float32)
        for i in range(self.m):                            # build the table
            tables[i] = self.codebooks[i] @ q[i*self.dsub:(i+1)*self.dsub]
        scores = tables[np.arange(self.m), codes].sum(axis=1)   # vectorised lookup
        top = np.argpartition(-scores, min(k, len(scores) - 1))[:k]
        return top[np.argsort(-scores[top])]

# pq = PQ(d=768, m=96)
# pq.train(sample_of_chunk_vectors)       # e.g. 100,000 of the 100 million chunks
# codes = pq.encode(all_chunk_vectors)    # 96 bytes per chunk
# top10 = pq.search(sso_query_vector, codes)
```

The line that computes `scores` does the work for the entire collection, using nothing but
fancy indexing and a sum. In simple words, it is Step 4 of Phase 3 for every page at once.
FAISS (Meta's vector search library) implements the same idea with SIMD kernels. It also
offers 4-bit "fast scan" variants whose small tables fit inside the CPU's SIMD registers.

---

### What people get wrong

**Training on unrepresentative data.** Codebooks fitted to one slice of your corpus will
quantize the rest badly. Sample across document types, languages and time. The first 100,000
rows of a time-ordered table are a biography of last year, not a sample.

**Choosing $m$ so that subvectors are large.** $d/m = 64$ produces terrible codebooks. Aim for
4–16 dimensions per subvector.

**Not rescoring.** PQ distances are approximations. Retrieve 10–20× more candidates than you
need and rescore the survivors with full-precision vectors. This one step recovers most of
the lost recall for a tiny cost.

**Expecting PQ to fix a bad model.** Compression cannot add information. A 32× compressed
good embedding beats an uncompressed bad one.

**Ignoring codebook staleness.** As with IVF centroids, codebooks fitted to old data slowly
stop matching new data. Recall decays without any error message. Retrain periodically.

---

### Ninja notes

**Always pair PQ with a rescoring stage.** The pattern:

```
PQ index over 100M vectors  →  top 100–200 candidates   (fast, approximate)
full-precision rescore of those candidates               (exact, ~1 ms)
→ top 10
```

With $m=96$ (32× compression), raw PQ recall@10 might be 0.90, but recall measured over the
wider shortlist is much higher. The right answers are usually *in* the candidate set, merely
ordered imperfectly. Rescoring fixes the ordering exactly. You end up near-exact at 32×
compression, provided you can read the full vectors for that shortlist from somewhere (RAM,
SSD, or object storage). This is the single most important practical fact about PQ.

**Know the alternatives to plain PQ.** **OPQ** (Chapter 25) learns a rotation first, so that
variance is spread evenly across subvectors. **Residual quantization** encodes what PQ got
wrong (the residual) with a second codebook, stacking layers of refinement. ColBERTv2 (the
multi-vector model of Chapter 37) uses a close cousin: a centroid ID plus a 1–2-bit *scalar*
code for each residual coordinate. **Anisotropic quantization**, used by Google's ScaNN
library, weights errors that change the inner product more heavily than errors that do not.
For maximum inner product search (MIPS), that is a genuinely better objective than plain
reconstruction error.

---

### Key takeaways

- **Product quantization = cut each vector into small pieces, and store each piece as the ID
  of its nearest entry in a small codebook.**
- Typical result: 3,072 bytes → 96 bytes, a 32× reduction. 100 million vectors go from 307 GB
  to 9.6 GB.
- The "product" is the trick. 96 codebooks of 256 entries express $256^{96}$ different codes,
  which no single codebook could.
- ADC scores a query by table lookup: one $96 \times 256$ table per query, then 96 lookups
  and adds per stored vector.
- Keep 4–16 dimensions per subvector, and train codebooks on representative data.
- PQ is almost always a component (IVF-PQ, DiskANN), not a standalone index.
- Always rescore the top candidates at full precision. It recovers most of the lost recall.

### What's next

[Chapter 25](./25-ivfpq-and-opq.md) puts clustering and quantization together into the index
that serves most of the world's billion-vector workloads, and shows how to tune it properly.

We now know how PQ shrinks a vector 32×, how a query is scored against the codes with a
table, and why a rescoring step belongs right behind it.
