---
title: "MUVERA: Multi-Vector Made Single"
chapter: 38
part: "Part V — Beyond One Vector"
slug: muvera
readingTime: "14 min"
summary: "What if you could squash 200 token vectors into one fixed-length vector whose dot product approximates MaxSim? Then every index in Part IV works again, unchanged. That is MUVERA."
tags: [muvera, fde, chamfer, multi-vector, simhash, mips]
prev: 37-colbert-3-serving
next: 39-splade
---

# MUVERA: Multi-Vector Made Single

**The one-paragraph version.** **MUVERA** (Multi-Vector Retrieval via Fixed Dimensional
Encodings) turns a *set* of vectors into a *single* vector, called a **Fixed Dimensional
Encoding** or FDE. Its key property is that the dot product of two FDEs approximates the
Chamfer (MaxSim) similarity of the original sets. Once that is true, multi-vector retrieval
becomes ordinary maximum inner product search, and every index from Part IV works unchanged. We
retrieve a shortlist with the FDE, then rerank it with exact MaxSim.

In this chapter, we will learn how MUVERA sorts a page's token vectors into regions and packs
them into one vector, and why the dot product of those vectors behaves like MaxSim. We will also
see what MUVERA guarantees, how it compares with PLAID, and when to use which one.

We will cover the following:

- What is MUVERA
- Why we need MUVERA
- The idea: sorting tokens into regions
- How to build an FDE, step by step
- An example of MUVERA, by hand
- What MUVERA guarantees
- The MUVERA pipeline and its results
- MUVERA vs PLAID
- When to use which one

---

## What is MUVERA

Before jumping in, we must recall one term from Chapter 36.

**Chamfer similarity = for each query token vector, its best match in the document's set of
vectors, added up over the query.** It is exactly ColBERT's MaxSim score.

Now the new terms.

**MUVERA = Multi-Vector Retrieval via Fixed Dimensional Encodings.** Let's break it down.

- **Multi-Vector Retrieval** is ColBERT-style search, with many vectors per page.
- **Fixed Dimensional** means every page gets a vector of the same length, whatever its size.
- **Encoding** means we transform the whole set into that one vector.

**FDE = Fixed Dimensional Encoding = one ordinary vector that stands in for a whole set of
token vectors.** We build one FDE for the query and one for each page, such that:

$$ \langle \text{FDE}(Q),\ \text{FDE}(D) \rangle \approx \text{Chamfer}(Q, D) $$

In simple words, one dot product between two long vectors ranks pages in roughly the same order
as all of ColBERT's token-by-token matching.

And why does that help? Every Card Catalog in Part IV answers one question: which stored vectors
have the largest dot product with this one? That task is called **maximum inner product
search**, or MIPS. Once a page is one vector, multi-vector search *is* MIPS.

---

## Why we need MUVERA

Acme already runs one HNSW index over its pages. It has tenant filters, sharding and
quantization, all tuned. Now Acme wants ColBERT's accuracy on our running question, *"Does the
Pro plan include single sign-on?"*

Option one is PLAID (Chapter 37). It is impressive engineering: specialised compression, a
centroid inverted index and a custom multi-stage scorer. It is also a whole second stack to
build, tune and keep running beside the first.

Option two is the obvious shortcut. Average each page's token vectors into one vector, and put
that into the existing HNSW index. But averaging is pooling, and pooling is what Chapter 34 warned
us about. Page 1,140 holds the catch, "On Pro, SSO needs the Security add-on for teams under 50
seats", inside a long page about the Security add-on. Its average is pulled toward everything
else on the page: audit logs, allow-lists, retention. In the example below, a generic "Plans
overview" page outscores it. The Scholar never sees the catch, and the answer is wrong.

MUVERA asks a different question:

> Instead of building new machinery for set-to-set similarity, can we **transform the sets into
> single vectors** so the machinery we already have applies?

This is the mathematician's move: reduce the new problem to a solved one. If it works, we get
HNSW, IVF-PQ, DiskANN, filtering, sharding and every operational tool in this book, for free,
applied to multi-vector retrieval.

It works.

---

## The idea: sorting tokens into regions

Here is the idea before any formulas.

MaxSim asks, for each query token, which page token matches it best. Matching tokens sit *near
each other* in the Map Room. So if we chop the Map Room into regions, a query token and its best
match will usually land in the **same region**.

Think of it like a post office sorting letters into pigeonholes by postcode.

- The pigeonholes are the regions of the Map Room.
- The query's letters are the query token vectors.
- The page's letters are the page token vectors.
- Comparing the two sorting racks pigeonhole by pigeonhole is the dot product.

So we do this. We divide the space into $B$ regions. For each region:

- **On the query side**, we add up all the query vectors that fell in it.
- **On the page side**, we take the average of all the page vectors that fell in it.

Then we lay the per-region results side by side, one long vector for the query and one for the
page. Their dot product pairs up region by region. Inside each region, a query vector is compared
only with the page vectors nearest to it. That approximates *"each query token, matched against
its nearest page token"*, which is MaxSim.

```
space partitioned into B = 8 regions

query tokens  → region 3, 3, 5, 7
page tokens   → region 1, 3, 3, 3, 5, 5, 6, 7, 7 ...

query FDE  = [ 0 | 0 | Σq₃ | 0 | q₅ | 0 | q₇ | 0 ]
page  FDE  = [ d̄₁ | d̄₂ | d̄₃ | d̄₄ | d̄₅ | d̄₆ | d̄₇ | d̄₈ ]

dot product  =  Σq₃·d̄₃ + q₅·d̄₅ + q₇·d̄₇   ≈  Chamfer(Q, D)
```

---

## How to build an FDE, step by step

**Phase 1: Draw the random ingredients (once, shared by every query and page).**

**Step 1:** For each of $R_{reps}$ repetitions, draw $k_{sim}$ random hyperplanes. Together they
cut space into $B = 2^{k_{sim}}$ regions.

**Step 2:** For each repetition, draw a random projection matrix that shrinks $d$ dimensions down
to $d_{proj}$.

**Phase 2: Encode one set of vectors (a query, or a page).**

**Step 1:** For each vector, note which side of each hyperplane it falls on. That string of bits
is its region number.

**Step 2:** Project every vector down from $d$ to $d_{proj}$ dimensions.

**Step 3:** In each region, **sum** the vectors if this is a query. **Average** them if this is
a page.

**Step 4:** If this is a page, fill every empty region with the page vector whose region number
is closest in Hamming distance (Chapter 26).

**Step 5:** Lay the $B$ region blocks side by side.

**Step 6:** Repeat Steps 1–5 for every repetition, each with its own hyperplanes and projection,
and join the results end to end.

Now, the question is, why each step? Let's take them in order.

**Regions come from SimHash.** This is exactly the LSH of Chapter 21. Similar vectors land on
the same side of a random hyperplane with a probability tied to their angle, which is precisely
the property we need.

**Sum for the query, average for the page.** As formulas, for each region $k$:

$$ \vec{q}_k = \sum_{i:\,\phi(q_i)=k} q_i \qquad\qquad \vec{d}_k = \frac{1}{|\{j:\phi(d_j)=k\}|}\sum_{j:\,\phi(d_j)=k} d_j $$

Here $\phi$ gives a vector's region. This mirrors MaxSim's own asymmetry (Chapter 36). The outer
sum over query tokens becomes a literal sum. The inner max over page tokens becomes an average
within a small region, which approximates the max when regions are fine enough.

**Fill empty page regions.** A region with no page vectors would contribute zero, and lose a
match that true MaxSim would have found. Borrowing the vector from the nearest region is a
cheap, effective repair.

**Project down.** $B \times d$ numbers per repetition is large. A random projection shrinks each
block, and Johnson–Lindenstrauss (Chapter 6) says dot products survive approximately.

**Repeat and concatenate.** One random partition is noisy, because a query token and its best match
may fall on opposite sides of a boundary. Independent repetitions average out each other's boundary
accidents. This resembles Chapter 21's many hash tables, but here the repetitions' scores are added
together, not unioned. One repetition's miss is diluted rather than rescued.

The final dimension is $B \times d_{proj} \times R_{reps}$. With $B = 16$ and $d_{proj} = 16$,
$R_{reps} = 10$ gives 2,560 dimensions. $R_{reps} = 16$ gives $16 \times 16 \times 16 = 4{,}096$,
the ~4,000-d FDE used in later chapters. Crucially, **we choose that size, not the page.** A
50-token passage and a 5,000-token document produce FDEs of identical size.

**Note:** Nothing here is learned. The hyperplanes and projections are random and fixed. There is
no training step, and nothing goes stale when the corpus changes.

---

## An example of MUVERA, by hand

Let's shrink the Map Room to 2 dimensions so we can do the arithmetic by hand. We use one
repetition, no projection, and two lines as our hyperplanes: the horizontal and vertical axes.
They cut the map into four regions, NE, NW, SW and SE. All the vectors are toy numbers.

Our query keeps two tokens. Page 1,140 keeps five, and the overview page keeps four.

| Set | Token | Vector | Region |
|---|---|---|---|
| Query | `sign-on` | [0.98, 0.17] | NE |
| Query | `pro` | [−0.17, 0.98] | NW |
| Page 1,140 | `sso` | [0.94, 0.34] | NE |
| Page 1,140 | `security` | [0.50, 0.87] | NE |
| Page 1,140 | `pro` | [−0.09, 1.00] | NW |
| Page 1,140 | `audit` | [0.17, −0.98] | SE |
| Page 1,140 | `retention` | [0.47, −0.88] | SE |
| Plans overview | `plans` | [0.71, 0.71] | NE |
| Plans overview | `features` | [0.64, 0.77] | NE |
| Plans overview | `teams` | [−0.71, 0.71] | NW |
| Plans overview | `pricing` | [−0.87, 0.50] | NW |

**First, the shortcut: averaging.** The query's average is [0.41, 0.58]. Page 1,140's average is
[0.40, 0.07], and its dot product with the query is about 0.20. The overview page's average is
[−0.06, 0.67], and its dot product is about 0.36. The overview page ranks first. Normalizing the
averages first does not rescue page 1,140: the cosines are 0.71 and 0.77. The answer is wrong.

**Now, MUVERA.** We follow Phase 2.

**Step 1:** Sort tokens into regions, as in the table.

**Step 2:** We skip projection in this toy.

**Step 3:** The query sums per region. NE = [0.98, 0.17], NW = [−0.17, 0.98], SW and SE are zero.
The pages average per region. Page 1,140 gets NE = average of `sso` and `security` = [0.72, 0.61],
NW = [−0.09, 1.00], and SE = average of `audit` and `retention` = [0.32, −0.93]. The overview
page gets NE = [0.68, 0.74] and NW = [−0.79, 0.61].

**Step 4:** Empty page regions are filled from a neighbour. The query is zero in SW and SE, so
those regions add nothing to this score.

**Step 5:** Join the blocks. The query FDE is [0.98, 0.17 | −0.17, 0.98 | 0, 0 | 0, 0].

Now the dot products, region by region:

```
page 1,140:  NE  0.98×0.72  + 0.17×0.61  = 0.81
             NW  −0.17×−0.09 + 0.98×1.00 = 1.00   (0.995)
             total ≈ 1.80

overview:    NE  0.98×0.68  + 0.17×0.74  = 0.79
             NW  −0.17×−0.79 + 0.98×0.61 = 0.73
             total ≈ 1.52
```

Page 1,140 (1.80) beats the overview page (1.52). The answer is correct. For comparison, exact
Chamfer similarity gives 1.97 and 1.63, so the FDE put the two pages in the same order.

Look at why the FDE came in lower than exact Chamfer, 1.80 instead of 1.97. In NE, `sign-on` met
the *average* of `sso` and `security`, not its best match `sso`. More hyperplanes would split
those two apart, and the estimate would climb toward 1.97. Notice also that `audit` and
`retention` in SE contributed nothing, because no query token lives there. That is MaxSim's "extra
content is not penalised", carried over.

In simple words, the averaged page lost the catch, and the region-sorted page kept it.

---

## What MUVERA guarantees

The pleasing part is that this is not a heuristic. MUVERA comes with a proof, and it is worth
stating carefully.

For any $\epsilon > 0$, FDEs whose dimension grows polynomially with the number of vectors give
a ±ε approximation of the *query-length-normalised* Chamfer score, with high probability.

Let's break that down.

- **Query-length-normalised** means Chamfer divided by the number of query vectors. It is the
  average best-match score per query token.
- **±ε** on that average means an error of at most ε per query token. A 32-token query can be
  off by up to 32ε in total.
- **Dimension grows with the number of vectors** means the proof needs bigger FDEs for bigger
  sets to keep the same ε.

In practice, a fixed few-thousand-dimensional FDE shortlists well regardless of document length,
as long as exact MaxSim reranks the shortlist. That is the property we actually use.

The guarantee still matters. It tells us the failure mode is *bounded error*, not *arbitrary
error*. We can trade dimension for accuracy knowingly, and the reranking stage cleans up what
remains.

---

## The MUVERA pipeline and its results

```
Index time:
  page → token vectors → FDE (one vector, ~4,000-d)
       → any ANN index (HNSW / DiskANN / IVF-PQ)
       → token vectors also stored, compressed, for reranking

Query time:
  query → token vectors → FDE
        → ANN search                       →  top ~1,000 candidates
        → exact Chamfer/MaxSim on those    →  top 10
```

The reported results are why this matters. Compared with PLAID, MUVERA achieves roughly **10%
higher recall at about 90% lower latency**, averaged across the paper's benchmarks.

The paper also compares FDEs with a simpler single-vector heuristic: search a token-level index
with each query token separately and pool the pages found. Against *that* heuristic, FDEs
reach the same recall while retrieving **2–5× fewer candidates** for reranking.

FDEs also compress well with product quantization, around **32×** with little quality loss.
They are ordinary dense vectors, so Chapter 24 applies to them directly.

---

## MUVERA vs PLAID

| | PLAID (Chapter 37) | MUVERA |
|---|---|---|
| Search index | Specialised centroid inverted index | Any single-vector ANN index |
| Training step | k-means centroids, learned from data | None, everything is random |
| Staleness when the corpus drifts | Centroids can go stale | None |
| Filtering, sharding, quantization | Must be built into the engine | Inherited from the existing index |
| Storage | Compressed token vectors | FDEs plus compressed token vectors for reranking |
| Reported quality and speed | Baseline | ~10% higher recall, ~90% lower latency |

---

## When to use which one

We must use **MUVERA** when we already run a vector database with filters, sharding and
quantization, and we want multi-vector accuracy without a second stack. It is also the natural
choice for page images searched ColPali-style (one vector per small square of the page image,
Chapter 45), where each page carries around a thousand vectors.

We must use **PLAID** when we run the ColBERT stack end to end, through its own libraries, and
do not need our database's filtering or sharding around it.

We must use **a plain single-vector model** when questions are short and topical, and the extra
FDE size and reranking step do not pay for themselves on our eval set.

Many strong systems use MUVERA to shortlist and exact MaxSim to rerank. That pairing is the
design, not an optional extra.

---

### Under the hood

The code follows Phase 1 and Phase 2 above. `encode` runs once per page at index time and once
per query at search time.

```python
import numpy as np

class FDE:
    def __init__(self, dim, k_sim=4, d_proj=16, reps=16, seed=0):
        rng = np.random.default_rng(seed)
        self.B = 2 ** k_sim                                               # regions per repetition
        self.planes = rng.normal(size=(reps, k_sim, dim))                 # Phase 1, Step 1: SimHash
        self.proj   = rng.normal(size=(reps, dim, d_proj)) / np.sqrt(d_proj)   # Phase 1, Step 2
        self.reps, self.d_proj = reps, d_proj
        self.pow2 = 1 << np.arange(k_sim)

    def _regions(self, V, r):
        return ((V @ self.planes[r].T) > 0) @ self.pow2                   # Step 1: (n,) in [0, B)

    def encode(self, V, is_query):
        out = []
        for r in range(self.reps):                                        # Step 6: repeat
            b = self._regions(V, r)
            P = V @ self.proj[r]                                          # Step 2: (n, d_proj)
            block = np.zeros((self.B, self.d_proj), dtype=np.float32)
            for k in range(self.B):
                m = b == k
                if m.any():                                               # Step 3: sum or average
                    block[k] = P[m].sum(0) if is_query else P[m].mean(0)
                elif not is_query:                                        # Step 4: fill empty region
                    hamming = [bin(k ^ int(x)).count("1") for x in b]
                    block[k] = P[int(np.argmin(hamming))]
            out.append(block.ravel())                                     # Step 5: side by side
        return np.concatenate(out)                                        # length B·d_proj·reps

# fde = FDE(dim=128)
# score = fde.encode(Q, True) @ fde.encode(D, False)   # a ranking proxy for Chamfer(Q, D)
```

With `k_sim=4`, `d_proj=16` and `reps=16`, the dimension is $16 \times 16 \times 16 = 4{,}096$.
That is one vector per page, regardless of length, ready for any index in Part IV.

The raw score is not Chamfer, and not a fixed multiple of it either. Each repetition adds its own
rough estimate of Chamfer, and that estimate usually runs low, as in our hand example. It drops
further as more page tokens share a region, which happens on longer pages. On toy data with
32-token queries, the ratio of FDE score to Chamfer fell several-fold from 4-token pages to
200-token pages. So the raw FDE score is a ranking *proxy*, not a scaled Chamfer value, and it
leans against long pages. Exact MaxSim reranking is what restores the true order.

Put simply, note what is *not* in this code: no training, no learned parameters, no codebooks.
**The FDE transformation is data-oblivious.** That means no training step, no staleness, and no
retraining when the corpus drifts, a real operational advantage over PLAID's learned centroids.

---

### What people get wrong

**"MUVERA replaces reranking."** It does not. The FDE is an approximation, and exact MaxSim on
the top candidates is part of the design, not an optional extra.

**"It works for any set similarity."** It is built for Chamfer/MaxSim specifically. A different
aggregation would need a different construction.

**"The proof says a fixed size is enough for any document."** It does not. The proven bound
needs dimension to grow with the number of vectors. A fixed few-thousand-d FDE shortlists well in
practice when exact MaxSim reranks, and your eval set is what confirms it for your data.

**Setting `reps` too low.** One or two repetitions are noisy, because boundary effects dominate.
Published configurations use around 10–20.

**Setting `k_sim` too high.** More regions means finer partitioning and a better approximation of
the max. It also means more empty regions and a much larger FDE. There is a knee, so find it on
your data.

**Forgetting that we still store token vectors.** Reranking needs them. MUVERA reduces the
*search* cost, not the *storage* cost, so ColBERTv2-style residual compression (Chapter 37) still
belongs underneath.

---

### Ninja notes

**The deeper lesson is the reduction itself.** MUVERA's contribution is less a specific algorithm
than a demonstration that a hard retrieval problem can be *reduced* to single-vector MIPS. At
that point thirty years of index engineering applies for free.

Watch for this pattern elsewhere. The same move would let you reuse Part IV's machinery for any
similarity that can be approximated by an inner product after a suitable encoding: graph
similarity, structured-record matching, set overlap, time-series alignment. When you meet a
novel similarity function, the productive first question is not *"what index can I build for
this?"* but *"can I encode this as an inner product?"*

**Practical adoption.** MUVERA-style FDEs are appearing in vector databases as first-class
support for multi-vector retrieval, precisely because they slot into existing infrastructure. If
you are choosing a platform for ColBERT or ColPali workloads, ask whether it implements FDEs or
PLAID-style native multi-vector indexes. The operational difference is significant, and FDE
support means your filtering, sharding and quantization all keep working.

**And it composes with Chapter 47.** ColPali produces about 1,000 vectors per page image (1,030
at its fixed resolution). MUVERA collapses those into one FDE for retrieval, then exact MaxSim
reranks the shortlist. That combination is currently among the most practical known designs for
searching millions of document images, and it is where Part VI lands.

---

### Key takeaways

- **MUVERA = turn a set of vectors into one Fixed Dimensional Encoding (FDE) whose dot product
  approximates Chamfer/MaxSim.**
- Construction: SimHash regions → sum for queries, average for pages → fill empty page regions →
  random projection → repeat and concatenate.
- FDE size is $B \times d_{proj} \times R_{reps}$, chosen by us, not by the page. 16 × 16 × 16 =
  4,096, so ~4,000-d is typical.
- The proof bounds error on *query-length-normalised* Chamfer, with dimension growing with set
  size. In practice a fixed few-thousand-d FDE shortlists well when exact MaxSim reranks.
- It is data-oblivious: no training, no codebooks, no staleness.
- Retrieve with the FDE using any ordinary ANN index, then rerank with exact MaxSim. The raw FDE
  score is a ranking proxy, not a scaled Chamfer value, and it runs lower on longer pages.
- Reported: ~10% better recall and ~90% lower latency than PLAID, 2–5× fewer candidates than a
  single-vector heuristic, and 32× PQ compression available.
- The transferable idea: reduce an exotic similarity to an inner product and inherit all existing
  infrastructure.

### What's next

[Chapter 39](./39-splade.md) takes the opposite route to a similar goal: a neural network that
writes *sparse* weights into a classical inverted index.

We now know how MUVERA sorts a page's token vectors into regions, why one dot product then
behaves like MaxSim, and why that lets every index we already have serve multi-vector search.
