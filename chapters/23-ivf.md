---
title: "IVF: Clustering the Space"
chapter: 23
part: "Part IV — Index Structures"
slug: ivf
readingTime: "14 min"
summary: "Group vectors into clusters, compare the query to the cluster centres, and search only the closest few. One idea, one knob, and the foundation of most billion-scale systems."
tags: [ivf, kmeans, clustering, faiss, nprobe]
prev: 22-trees-and-annoy
next: 24-product-quantization
---

# IVF: Clustering the Space

**The one-paragraph version.** **IVF** (Inverted File index) runs k-means, a clustering method we
define below, over our vectors to create `nlist` clusters. It assigns each vector to its nearest
centroid and stores one list per cluster. At query time we compare the query with the centroids
only, pick the nearest `nprobe`, and scan those lists. We replace a million comparisons with a few
thousand, and `nprobe` is a clean, single-parameter recall knob.

In this chapter, we will learn about IVF, the cluster family from Chapter 20. We will learn
k-means step by step on five of Acme's pages, then build and search an IVF index. We will also see
the boundary problem that is IVF's one real weakness, how IVF compares with HNSW, and when to use
which one.

We will cover the following:

- What is IVF
- What is k-means
- An example of k-means on five pages
- How IVF builds and searches its index
- An example of IVF on our SSO question
- The one knob that matters: nprobe
- Problems with IVF
- IVF vs HNSW
- When to use which one

---

## What is IVF

The Great Library is organised into a hundred wings. Each wing has a sign above its door:
*Billing*, *Login*, *Release notes*, and so on. A visitor arrives with a question.

The visitor does not read every book. They read the hundred signs, walk into the two or three most
promising wings, and search only those.

That is IVF entirely.

**IVF = Inverted File index = group the vectors into clusters, keep one list per cluster, and
search only the lists nearest the query.**

Here is how the picture maps onto IVF, one part at a time:

- The Library's books are our page vectors.
- Each wing is a cluster, stored as an **inverted list** of the vectors inside it.
- Each sign is the cluster's **centroid**, the average position of its vectors.
- **nlist = the number of clusters**, the number of wings in the Library.
- **nprobe = the number of nearest clusters we search for each query**, the wings the visitor
  enters.

The name comes from text search. An inverted index (Chapter 7) maps *word → pages*. IVF maps
*cluster → vectors*. Same structure, learned keys.

And the failure mode is just as easy to picture. A book that is half about billing and half about
login sits in exactly one wing. A visitor who guesses the other wing will never find it. Hold
that thought. It is IVF's whole limitation.

Now, the question is, who decides where the wings are? That is the job of k-means.

---

## What is k-means

Before building IVF, we must know k-means.

**k-means = a way to split points into k groups, by moving k centres until each one sits at the
average of its nearest points.**

The k is the number of groups we ask for. In IVF, it is `nlist`. (It is not the top-k of Chapter
3. The letter is simply popular.)

We need one more word, which Chapter 18 introduced briefly.

**Centroid = the average position of a group of points.** We add up each coordinate across the
points, then divide by how many points there are.

Here is k-means, one action per line:

**Step 1:** Pick k starting centres at random, usually k random points from the data.
**Step 2:** Assign every point to its nearest centre.
**Step 3:** Move each centre to the average of the points assigned to it.
**Step 4:** Repeat Steps 2 and 3 until no point changes centre.
**Step 5:** The final centres are the **centroids**.

In simple words, we guess some centres, let each point pick its closest centre, move every centre
into the middle of its points, and repeat until nothing moves.

---

## An example of k-means on five pages

Let's run k-means by hand on five pages from Acme's Library.

Real embeddings have 768 dimensions. For this example we use a toy Map Room with just 2 numbers
per page. The first number says how much the page talks about billing, prices and paid add-ons.
The second says how much it talks about logging in: SSO, two-factor login, passwords.

| Page | Billing (x) | Login (y) |
|---|---|---|
| Invoices | 9 | 1 |
| Refunds | 8 | 2 |
| Page 212: SSO is included on the Pro plan | 3 | 6 |
| Two-factor login | 1 | 8 |
| Page 1,140: on Pro, SSO needs the Security add-on under 50 seats | 4 | 3 |

We ask for k = 2 groups.

To measure "nearest", we use the **squared distance**: square the gap in x, square the gap in y,
and add. It picks the same nearest centre as ordinary distance, and it avoids square roots.

**Step 1: Pick starting centres.** We pick two pages at random: page 212 at (3, 6) becomes centre
A, and the two-factor page at (1, 8) becomes centre B.

**Round 1, Step 2: Assign every page to its nearest centre.**

```
                        to A (3,6)       to B (1,8)       nearest
Invoices     (9,1)      6²+5² = 61       8²+7² = 113      A
Refunds      (8,2)      5²+4² = 41       7²+6² = 85       A
Page 212     (3,6)      0                2²+2² = 8        A
Two-factor   (1,8)      2²+2² = 8        0                B
Page 1,140   (4,3)      1²+3² = 10       3²+5² = 34       A
```

**Round 1, Step 3: Move each centre to the average of its pages.**

```
A = average of Invoices, Refunds, Page 212, Page 1,140
  = ((9+8+3+4)/4, (1+2+6+3)/4) = (24/4, 12/4) = (6, 3)
B = average of Two-factor = (1, 8)
```

Centre A has been dragged toward the billing pages.

**Round 2, Step 2: Assign again, using the new centres.**

```
                        to A (6,3)       to B (1,8)       nearest
Invoices     (9,1)      3²+2² = 13       8²+7² = 113      A
Refunds      (8,2)      2²+1² = 5        7²+6² = 85       A
Page 212     (3,6)      3²+3² = 18       2²+2² = 8        B   ← changed
Two-factor   (1,8)      5²+5² = 50       0                B
Page 1,140   (4,3)      2²+0² = 4        3²+5² = 34       A
```

Page 212 switched from A to B. Centre A moved away from it.

**Round 2, Step 3: Move the centres again.**

```
A = average of Invoices, Refunds, Page 1,140 = ((9+8+4)/3, (1+2+3)/3) = (7, 2)
B = average of Page 212, Two-factor          = ((3+1)/2, (6+8)/2)     = (2, 7)
```

**Round 3, Step 2: Assign once more.**

```
                        to A (7,2)       to B (2,7)       nearest
Invoices     (9,1)      2²+1² = 5        7²+6² = 85       A
Refunds      (8,2)      1²+0² = 1        6²+5² = 61       A
Page 212     (3,6)      4²+4² = 32       1²+1² = 2        B
Two-factor   (1,8)      6²+6² = 72       1²+1² = 2        B
Page 1,140   (4,3)      3²+1² = 10       2²+4² = 20       A
```

**Step 4: Nothing changed, so we stop.**

**Step 5: The centroids are A = (7, 2) and B = (2, 7).** A is the sign for the *Billing* wing:
invoices, refunds and page 1,140. B is the sign for the *Login* wing: page 212 and two-factor
login.

That is it. That is the whole of k-means. The `kmeans` function in Under the hood reproduces these
two centroids exactly.

Notice where page 1,140 ended up. Its text is mostly about the paid Security add-on, with just
one sentence on SSO. So it scored 4 on billing but only 3 on login, and it landed in the Billing
wing, away from page 212. Hold that thought too.

---

## How IVF builds and searches its index

Now we can build the real thing. IVF works in two phases.

**Phase 1: Building the index.**

**Step 1: Train.** Run k-means on a sample of our vectors to find `nlist` centroids. A few hundred
thousand vectors is plenty for `nlist = 1000`. The usual guide is roughly 30–256 sample vectors
per centroid, so a large `nlist` needs a larger sample. In simple words: pick `nlist` starting
centres at random, assign every vector to its nearest centre, move each centre to the average of
its vectors, and repeat until nothing moves. This is the only training step, and it is why IVF
indexes have a `train()` call that HNSW does not.

**Step 2: Assign.** For every vector in the whole Library, find its nearest centroid, and append
the vector's ID to that centroid's list.

```
centroid 0  →  [id_4, id_91, id_552, ...]
centroid 1  →  [id_7, id_33, ...]
...
centroid 999 → [id_12, id_808, ...]
```

**Phase 2: Answering a query.**

**Step 1:** Compare the query with all `nlist` centroids. This is cheap: 1,000 centroids means
1,000 distance calculations.
**Step 2:** Keep the `nprobe` nearest centroids.
**Step 3:** Scan every vector in those lists exactly, and keep the top k.

Let's put numbers on it. Take 10 million vectors and `nlist = 1000`, so each list holds about
10,000 vectors. At `nprobe = 10`, we scan 100,000 vectors instead of 10,000,000. That is a 100×
reduction, plus 1,000 centroid comparisons.

---

## An example of IVF on our SSO question

Let's go back to our five pages and the two wings k-means just built. The Billing centroid is at
(7, 2). The Login centroid is at (2, 7).

Our running question, *"Does the Pro plan include single sign-on?"*, lands at (3, 5) in the toy
Map Room. It is mostly about logging in, and a little about plans.

First, the true answer. Brute force measures the question against every page:

```
Page 212     (3,6):  0²+1² = 1     ← nearest
Page 1,140   (4,3):  1²+2² = 5     ← second nearest
Two-factor   (1,8):  2²+3² = 13
Refunds      (8,2):  5²+3² = 34
Invoices     (9,1):  6²+4² = 52
```

The two nearest pages are page 212 and page 1,140, exactly the two pages the answer needs.

We start with the cheapest setting, `nprobe = 1`.

**Step 1:** Compare the question with the centroids. Login: 1² + 2² = 5. Billing: 4² + 3² = 25.
**Step 2:** Keep the nearest one: the Login wing.
**Step 3:** Scan it. Page 212 scores 1, two-factor login scores 13. Top 2: page 212, two-factor.

The Librarian hands the Scholar page 212 and the two-factor page. The Scholar thinks: "Page 212
says SSO is included on Pro. The other page is about two-factor login." It answers, "Yes, the Pro
plan includes SSO."

That answer fails every Pro team under 50 seats. Page 1,140 was never checked, because it lives
in the Billing wing.

Next, we raise the setting to `nprobe = 2`.

**Step 2:** Keep both wings.
**Step 3:** Scan them. Page 212 scores 1, page 1,140 scores 5. Top 2: page 212, page 1,140.

The Scholar thinks: "Page 212 says SSO is included on Pro. Page 1,140 adds that teams under 50
seats need the Security add-on." It answers, "Yes, but teams under 50 seats need the Security
add-on."

This time the answer is right. We paid for it by scanning one more wing.

---

## The one knob that matters: nprobe

`nprobe` is the recall–latency dial, and it always moves both the same way: more wings, more
recall, more latency. Here is a typical shape for 10 million vectors and `nlist = 1000`. The
numbers are illustrative, so we measure on our own data.

| `nprobe` | Vectors scanned (of 10M) | Recall@10 | Latency |
|---|---|---|---|
| 1 | 10,000 | ~0.65 | 1× |
| 8 | 80,000 | ~0.90 | 6× |
| 32 | 320,000 | ~0.97 | 22× |
| 128 | 1,280,000 | ~0.995 | 85× |
| 1000 (= all) | 10,000,000 | 1.00 | brute force |

There is Chapter 18's steep curve again. The last few points of recall cost more than everything
before them.

The great virtue of `nprobe` is that it is a **runtime** parameter. We can raise it for a
high-value query and lower it under load, without touching the index. HNSW's `efSearch` shares this
property. Most other tuning does not.

`nlist` is the build-time parameter. The standard rule of thumb is $\text{nlist} \approx \sqrt{n}$,
where n is the number of vectors. That gives 1,000 for a million vectors and 10,000 for a hundred
million. (Our 10-million example used 1,000 to keep the arithmetic round. The rule would suggest
about 3,000.) Treat $\sqrt{n}$ as a starting point. Chapter 25 goes up to $4\sqrt{n}$, and FAISS's
own guidelines suggest $4\sqrt{n}$–$16\sqrt{n}$.

Why not much fewer or much more? Too few, and each list is enormous, so scanning one is
expensive. Too many, and we compare against too many centroids. Each list is also so small that
`nprobe` must rise to compensate.

---

## Problems with IVF

**Problem 1: The boundary problem.** Every vector belongs to exactly **one** cluster, and
clusters have edges. A vector near an edge has true neighbours on the other side, and we miss
them unless `nprobe` happens to include that cluster. Page 1,140 showed exactly this. It is
structural, not a tuning failure. There are two mitigations.

- **Raise `nprobe`.** Probing more clusters makes it likelier the neighbouring region is
  included. This is the standard answer, and the reason IVF often needs `nprobe` of 16–64 in
  practice, not 2–3.
- **Assign each vector to several lists.** Some implementations allow soft assignment to the
  nearest 2 or 3 centroids. Recall improves, and the index grows in proportion.

A graph index has no boundaries at all. HNSW's links simply connect each point to whatever is
near it, crossing any border an IVF centroid would have drawn. That is much of why graphs win on
recall at equal latency.

**Problem 2: Centroids go stale.** Centroids are fitted to a snapshot of the data. If Acme adds
a new product line or a new language, new pages crowd into a few clusters, and recall decays
silently. The fix is to retrain.

**Problem 3: Lists come out uneven.** Real clusters are not the same size. One list may hold 5% of
the Library, and every query that probes it is slow. Ninja notes show the fix.

**Problem 4: `nlist` must suit the data size.** Too many clusters for too few vectors leaves each
list nearly empty, and `nprobe` must climb to find anything.

---

## IVF vs HNSW

Given the boundary problem, why is IVF everywhere? The answer is mostly memory, and it is subtler
than it first looks.

**Memory.** IVF-Flat stores our vectors plus a small ID list, roughly 1.0–1.1× raw size. HNSW
stores the vectors *plus a graph*, about 1.05–1.1× at 768 dimensions (Chapter 29). So uncompressed,
they are close. The real memory gap is that IVF's lists compress to PQ codes (Chapter 25) while
HNSW must keep something close to full vectors in RAM. PQ, product quantization, shrinks each
3,072-byte vector to about 96 bytes (Chapter 24).

**It composes perfectly with compression.** IVF-PQ (Chapter 25) is the canonical billion-scale
index: cluster to narrow the search, quantize to shrink the vectors. The two techniques are
independent, and their savings multiply.

**Build is fast and parallel.** k-means runs on a sample, then the assignment pass splits
trivially across machines. HNSW's graph construction is far more expensive and harder to
distribute.

**Deletion is trivial.** Remove the ID from its list. There is no graph repair, no **tombstones**
(deleted markers left in place) and no clean-up pass to schedule. Compare Chapter 31.

**Filtering is natural.** Scanning a list is a loop, so adding a condition such as
`if page.type == "pricing"` costs almost nothing. Graph traversal with filters is genuinely hard
(Chapter 33).

| | IVF | HNSW |
|---|---|---|
| Structure | One list per cluster | A graph of links between vectors |
| Memory at 768-d, uncompressed | ~1.0–1.1× | ~1.05–1.1× |
| Memory with compression | ~0.03× with PQ codes | Usually int8 (~0.25×), all in RAM |
| Recall at equal latency | Lower (boundaries) | Higher |
| Training step | Yes, k-means | No |
| Build speed | Fast, parallel | Slower, harder to distribute |
| Deletes | Remove the ID from a list | Tombstones and clean-up |
| Filters | Easy loop condition | Hard (Chapter 33) |
| Runtime knob | `nprobe` | `efSearch` |

---

## When to use which one

We must use **IVF** when memory is the binding constraint, especially at hundreds of millions of
vectors and beyond. It also fits when data arrives in large batches, deletes are frequent, or
every query carries a filter.

We must use **HNSW** when the vectors fit in RAM and we want the best recall per millisecond, with
vectors arriving one at a time.

For Acme's 40,000 pages today, brute force is still the honest answer (Chapter 17). IVF becomes
interesting once the Library grows into tens of millions of vectors.

Many strong systems use both. Ninja notes show IVF centroids routing queries to shards that each
run HNSW, and Chapter 25 puts an HNSW graph over the centroids themselves.

---

### Under the hood

The code follows the Steps above. `nearest_centre` is Step 2 of k-means. `kmeans` is Steps 1–5.
`IVFIndex` is Phase 1 (`train`, `add`) and Phase 2 (`search`).

```python
import numpy as np

def nearest_centre(X, C, batch=20_000):
    # squared distance |x - c|^2 = |x|^2 - 2 x.c + |c|^2. |x|^2 is the same for
    # every centre, so we can drop it when picking the nearest one.
    c_sq = (C ** 2).sum(axis=1)
    return np.concatenate([np.argmin(c_sq - 2 * X[i:i + batch] @ C.T, axis=1)
                           for i in range(0, len(X), batch)])

def kmeans(S, k, init=None, max_rounds=50, seed=0):
    rng = np.random.default_rng(seed)
    C = S[rng.choice(len(S), k, replace=False)] if init is None else init
    C = np.array(C, dtype=np.float32)                          # Step 1: starting centres
    assign = None
    for _ in range(max_rounds):
        new = nearest_centre(S, C)                             # Step 2: nearest centre
        if assign is not None and np.array_equal(new, assign):
            break                                              # Step 4: nothing moved
        assign = new
        for c in np.unique(assign):                            # Step 3: move to the average
            C[c] = S[assign == c].mean(axis=0)
    return C                                                   # Step 5: the centroids

class IVFIndex:
    def __init__(self, nlist=1024):
        self.nlist = nlist

    def train(self, V, sample=200_000, seed=0):                # Phase 1, Step 1
        rng = np.random.default_rng(seed)
        S = V[rng.choice(len(V), min(sample, len(V)), replace=False)]
        self.centroids = kmeans(S, self.nlist, seed=seed)
        # The empty lists are created once, here. add() only ever appends to them.
        self.ids = [[] for _ in range(self.nlist)]
        self.vecs = [np.empty((0, V.shape[1]), dtype=np.float32)
                     for _ in range(self.nlist)]

    def add(self, V, ids):                                     # Phase 1, Step 2
        ids = np.asarray(ids)
        assign = nearest_centre(V, self.centroids)
        for c in np.unique(assign):
            mask = assign == c
            self.ids[c].extend(ids[mask].tolist())
            self.vecs[c] = np.vstack([self.vecs[c], V[mask].astype(np.float32)])

    def search(self, q, k=10, nprobe=16):                      # Phase 2
        dist = ((self.centroids - q) ** 2).sum(axis=1)
        probe = np.argsort(dist)[:nprobe]                      # Steps 1-2
        cand_ids, cand_scores = [], []
        for c in probe:                                        # Step 3
            if self.ids[c]:
                cand_scores.append(self.vecs[c] @ q)           # unit vectors: dot = cosine
                cand_ids.extend(self.ids[c])
        if not cand_ids:
            return []
        s = np.concatenate(cand_scores)
        top = np.argpartition(-s, min(k, len(s) - 1))[:k]
        return [(cand_ids[i], float(s[i])) for i in top[np.argsort(-s[top])]]

# The five-page example from this chapter:
pages = np.array([[9, 1], [8, 2], [3, 6], [1, 8], [4, 3]], dtype=np.float32)
print(kmeans(pages, 2, init=pages[[2, 3]]))    # [[7. 2.]  [2. 7.]]
```

In simple words, `train` finds the wings, `add` files each page into its wing, and `search` walks
into the `nprobe` nearest wings and reads only those shelves.

Two details matter. First, `train` creates the empty lists and `add` only appends. So we can call
`add` many times, for example once per day's new pages, without losing earlier batches. Second,
`search` assumes unit-length vectors (Chapter 5), so the dot product ranks pages exactly as cosine
similarity would.

In production we would not write k-means by hand. `sklearn.cluster.MiniBatchKMeans` or FAISS's
built-in k-means does Step 1 far faster. FAISS's `IndexIVFFlat` also stores each list contiguously,
so the scan is a clean streaming matrix multiply. That is the same bandwidth-bound operation as
Chapter 17, run over about 1% of the data.

---

### What people get wrong

**Training on too little data, or on the wrong data.** Centroids fitted to 10,000 vectors from one
document type will not represent a corpus of ten million mixed ones. Sample broadly.

**Never retraining.** Centroids are fitted to a snapshot. If the corpus drifts (new product lines,
new languages, a new document type), clusters become unbalanced and recall decays silently.
Monitor the distribution of list sizes. A few enormous lists mean it is time to retrain.

**Setting `nlist` too high with too little data.** With a million vectors and `nlist = 65536`,
lists hold ~15 vectors each, and you need a huge `nprobe` to find anything. Keep roughly $\sqrt{n}$.

**Forgetting `nprobe` is per-query.** Many teams set it once in config. It is a runtime dial, so
use it.

---

### Ninja notes

Three refinements worth knowing.

**Balanced clustering.** Plain k-means on real embeddings produces wildly uneven clusters, and
one may hold 5% of the corpus. Query latency depends on the size of the lists we probe, so this
creates a bad p99: queries landing in the big cluster are slow. Balanced or capacity-constrained
k-means, or simply splitting oversized lists, flattens the tail considerably. This is a cheap p99
win that most teams never make.

**Smarter multi-assignment.** Putting a vector in its two nearest lists helps at borders but
doubles the index. Google's SOAR, used in ScaNN, refines this idea by choosing the second list so
that its errors complement the first's. It is a partition-index trick, not a learned index.

**IVF as a router, not an index.** At extreme scale, the centroids become a *routing* layer. They
decide which **shard or machine** to query, and each shard runs HNSW locally. We get IVF's cheap,
parallel partitioning at the top and the graph's superior recall at the bottom. This two-level
design underlies several distributed vector databases, and it is the natural bridge to Chapter 58.

---

### Key takeaways

- **IVF = k-means clusters + one list per cluster. Probe only the nearest few lists.**
- **k-means** = pick k centres, assign every point to its nearest centre, move each centre to
  the average of its points, repeat until nothing moves. The final centres are the **centroids**.
- `nlist` ≈ √n at build time. `nprobe` is the runtime recall knob.
- Boundary effects are the structural weakness. Page 1,140 sat in the Billing wing and was missed
  at `nprobe = 1`, then found at `nprobe = 2`.
- Uncompressed, IVF and HNSW use similar memory at 768-d. IVF's real memory win is that its lists
  compress to PQ codes.
- Its other advantages are fast parallel builds, trivial deletes and easy filtering.
- Retrain centroids as the corpus drifts, and watch for unbalanced lists in the p99.
- IVF also works beautifully as a shard router in front of graph indexes.

### What's next

IVF narrowed *how many* vectors we compare. [Chapter 24](./24-product-quantization.md) attacks the
other half of the cost: making each vector 32 times smaller.

We now know how k-means draws the wings of the Library, how IVF searches only the nearest ones,
why a page on the border can go missing, and where IVF beats a graph.
