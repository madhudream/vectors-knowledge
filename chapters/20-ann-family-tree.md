---
title: "The ANN Family Tree"
chapter: 20
part: "Part III — Finding Neighbours"
slug: ann-family-tree
readingTime: "11 min"
summary: "Every approximate nearest neighbour algorithm ever built is one of four ideas, or a combination of them. Learn the four and the zoo becomes a map."
tags: [ann, taxonomy, hnsw, ivf, lsh, mental-model]
prev: 19-measuring-retrieval-quality
next: 21-lsh
---

# The ANN Family Tree

**The one-paragraph version.** There are four strategies for avoiding a full scan. **Hash**
similar things into the same bucket. **Partition** space into regions and search only nearby
ones. **Cluster** the data and search only nearby clusters. **Build a graph** and walk toward the
query. Every named algorithm (Annoy, IVF, LSH, HNSW, ScaNN, DiskANN) is one of these, or two of
them stacked. Compression is a separate dial that combines with all four.

In this chapter, we will learn the four ideas behind every approximate nearest neighbour index,
the Card Catalogs of our story. We will also watch one question go wrong in one family and right
in another. Then we will see why graphs won for in-memory search, how compression combines with
each family, and how to pick an index from our constraints in one table.

We will cover the following:

- What is an ANN family
- The four families
- An example: one question, two families
- Why graphs won
- Compression, the dial that combines with all four
- The four families side by side
- Choosing an index in one table

---

## What is an ANN family

Brute force (Chapter 17) compares the question with every vector in the Map Room. An **ANN
index** (approximate nearest neighbour, Chapter 3) compares it with only a small, well-chosen
subset. It accepts that it will sometimes miss a true neighbour, and Chapter 18 showed how to
measure how often.

There are hundreds of named ANN algorithms. The good news is that they are all built from four
ideas.

**ANN family = one basic way of choosing which small subset of vectors to compare.**

- **Hash family** = give every vector a short tag, built so that similar vectors get the same
  tag. Compare only the vectors that share the question's tag.
- **Partition family** = cut the space into regions, again and again. Compare only the vectors in
  the question's region and the regions beside it.
- **Cluster family** = group the vectors into clumps around learned centres. Compare only the
  vectors in the clumps nearest the question.
- **Graph family** = link every vector to a few of its near neighbours. Walk along the links
  toward the question.

In simple words: tags, cuts, clumps or links. That is the whole taxonomy. Everything else is
engineering.

Do not worry about the algorithm names that follow. We will learn about each of them in detail in
Part IV.

---

## The four families

The Librarian cannot read every book in a big enough Library. There are exactly four ways to
narrow the search, and people invented all four long before computers existed.

**1. Hash: the coat check.** Every book gets a tag computed from its contents, designed so that
similar books get the same tag. The Librarian computes the question's tag and looks only at books
carrying that tag. It is fast, and it fails when two similar books happen to get different tags.
→ **LSH** (Chapter 21).

Here, the tag is the hash, and the rack of books sharing one tag is a **bucket**.

**2. Partition: the floor plan.** Cut the building in half, then into quarters, then eighths,
recording which books sit where. Walk down the plan toward the question's spot. It is elegant in
low dimensions. It degrades badly as dimensions rise, because the question's neighbours scatter
across many regions. → **KD-trees, Annoy** (Chapter 22).

Here, each cut is a splitting line (a flat sheet in many dimensions), and the smallest rooms are
the tree's **leaves**.

**3. Cluster: the wings.** Group the books by subject into wings. Hang a sign at each wing's door
that describes the wing's average book. Compare the question with the signs, walk into the two or
three closest wings, and scan them. It is simple, tunable, and it works because real data is
clumpy (Chapter 3). → **IVF** (Chapter 23).

Here, the signs are **centroids** (average positions, Chapter 18), and each wing is a list of
vectors.

**4. Graph: the recommendation chain.** Every book carries a slip listing a few similar books.
Start anywhere, read the slip, move to whichever listed book is closer to what we want, and
repeat. It is surprisingly effective. → **HNSW, DiskANN** (Chapters 28–32), and cousins such as
NSG.

Here, each slip holds a book's **edges** (links to its neighbours), and following one link is a
**hop**. NSG is another graph index, a close cousin of HNSW.

---

## An example: one question, two families

Let's hand the Librarian our running question: *"Does the Pro plan include single sign-on?"*

The right answer needs two pages from Acme's Library. Page 212 says SSO is included on the Pro
plan. Page 1,140 says Pro teams under 50 seats need the Security add-on.

Page 212 sits among the plan pages in the Map Room. The SSO chunk of page 1,140 (Chapter 18) is
mostly about the Security add-on itself, so its dot sits closer to the security pages. It lies
near the border between two neighbourhoods.

Let's first see what a family that commits early does. A partition index walks down its floor
plan. The question's dot falls on the plan side of a cut, so the index searches only that side.
It finds page 212. Page 1,140 sits just across the cut, so it is never checked.

The Scholar reads page 212 and thinks: "SSO is included on Pro. Nothing mentions a condition."
It answers, "Yes, the Pro plan includes SSO." The answer is wrong for teams under 50 seats.

Now, let's see what a graph index does. It hops toward the question and reaches page 212. Page
212's slip lists its nearest neighbours. Page 1,140 sits just across the border, close enough to
be one of them. One more hop, and page 1,140 joins the results.

The Scholar reads both pages and thinks: "Page 212 says SSO is included on Pro. Page 1,140 adds a
condition for small teams." It answers, "Yes, but teams under 50 seats need the Security add-on."
The answer is correct.

**Note:** The partition and cluster families are not hopeless here. Checking the neighbouring
region as well (backtracking in a tree, or probing one more wing in IVF) would also find page
1,140. The point is *where* each family pays. Partitions and clusters pay at their borders.
Graphs pay in memory: for their links, and for keeping every vector in RAM.

---

## Why graphs won

For in-memory search, graph methods lead the benchmarks. It is worth understanding why, rather
than taking it on faith. There are three reasons.

**First, they adapt to the data's shape.** Partitioning imposes a grid on a space that is not
grid-shaped. Clustering imposes round clumps on groups that are not round. A graph just connects
each point to whatever happens to be near it. So it follows the thin, low-dimensional surface
where the data actually lives, which Chapter 6 calls a **manifold**. The curse of dimensionality
(Chapter 6) hurts partitioning badly and graphs far less.

**Second, search is progressive.** Every hop moves to a closer neighbour, so we can stop whenever
we are satisfied. The walk can get stuck in a dead end, a spot where no neighbour is closer even
though better pages exist elsewhere. That is exactly what the `efSearch` shortlist in Chapter 30 is
for: it keeps several promising candidates instead of just one. Partition methods commit to a
region before knowing whether it was the right one.

**Third, the cost is precisely tunable.** One parameter, `efSearch`, directly controls how many
nodes a search visits. That gives the clean recall–latency curve of Chapter 18.

The price is memory. The graph's links are extra data. They add a modest ~5–10% on top of 768-d
vectors, but up to 1.5–2× for low-dimensional vectors with many links each (Chapter 29 does the
arithmetic). And the vectors *plus* links must all sit in RAM, because the walk jumps to
unpredictable places. That single fact is why Chapters 24, 26, 27 and 32 exist.

---

## Compression, the dial that combines with all four

Compression is not a fifth strategy. It is a separate dial that combines with any of the four
families, and it is the key to reading index names.

**Compression = storing each vector in fewer bytes, in exchange for a little accuracy.** Chapter
4 called this **quantization**.

| Technique | Bytes per 768-d vector | Loss |
|---|---|---|
| float32 | 3,072 | none |
| float16 | 1,536 | negligible |
| int8 (scalar quantization) | 768 | small |
| Product quantization (m=96) | 96 | moderate |
| Binary (1 bit/dim) | 96 | large, but recoverable by rescoring |

In simple words, float16 and int8 store each number with fewer bits. Product quantization
replaces groups of numbers with short codes. Binary keeps only one bit per number. Rescoring means
recomputing exact scores for the few survivors. Chapters 24–27 cover all of them.

This is why index names look like compounds:

- **IVF-PQ** = cluster + product quantization
- **IVF-Flat** = cluster + no compression
- **HNSW-SQ** = graph + scalar quantization
- **DiskANN** = graph + PQ in RAM + full vectors on SSD (a fast solid-state disk)
- **ScaNN** = cluster + a smarter, score-aware quantization

Read any index name as `{search strategy}-{compression}`, and we can predict its behaviour before
reading the documentation.

---

## The four families side by side

| Family | Named examples | Strong at | Weak at |
|---|---|---|---|
| Hash | LSH | Near-duplicate detection, set similarity, no training | Recall on dense embeddings |
| Partition | KD-tree, Annoy | Low dimensions, memory-mapped files | High dimensions, updates (Annoy) |
| Cluster | IVF, IVF-PQ, ScaNN | Memory (with compression), fast builds, easy deletes and filters | Borders between clusters |
| Graph | HNSW, DiskANN, NSG | Recall at low latency | Memory (in-RAM graphs such as HNSW), deletes, filters |

**Advantages of knowing the families.** A new index name stops being a mystery. We can predict
where it will be strong and where it will break before running a single benchmark.

**Disadvantages of stopping there.** The family tells us the shape of the trade-off, not the
numbers. Two graph indexes can differ a lot in practice. We still measure on our own data
(Chapters 18 and 19).

---

## Choosing an index in one table

Now, the question is, which one should we use?

We pick our constraints first: corpus size, latency budget, memory budget, how often the data
changes, and how much filtering we need. The index then follows almost mechanically.

| Your situation | Use | Chapter |
|---|---|---|
| < 100k vectors | Flat (brute force) | 17 |
| < 1M, iterating fast, frequent updates | Flat, or HNSW with defaults | 17, 28 |
| 1M–100M, RAM available, want quality | **HNSW** | 28–31 |
| 1M–100M, memory-constrained | HNSW + int8, or IVF-PQ | 25, 26 |
| > 100M, memory is the binding constraint | IVF-PQ, or DiskANN | 25, 32 |
| > 1B | DiskANN, or sharded IVF-PQ | 32, 58 |
| Heavy metadata filtering | Depends entirely: read Chapter 33 first | 33 |
| Batch/offline workload | Flat on GPU, batched | 17 |
| Multi-vector (ColBERT/ColPali) | PLAID, or MUVERA + any of the above | 37, 38 |

A few words in that table are new. **Sharded** means split across several machines (Chapter 58).
**Multi-vector** models such as ColBERT and ColPali keep many vectors per page instead of one
(Part V).

Where does Acme sit today? Its Library has 40,000 pages, a few hundred thousand chunks, so it
sits in the second row. Brute force is the right answer. Acme's knowledge-base product in
Part VIII, about 100 million chunks, is where the lower rows start to matter.

**The honest default for most teams: HNSW, with int8 quantization if memory is tight.** Reach
past it when you have measured a specific reason to.

---

### Under the hood

The four strategies as four short pieces of pseudocode. Helper functions like `scan`, `topk` and
`collect_leaves` stand for the obvious loops. Side by side, the family resemblance is plain:

```python
# 1. HASH: compute a bucket in each table, scan them
def lsh_search(q, tables):                     # each table has its own hash function
    return scan([d for t in tables for d in t.buckets[t.hash(q)]], q)

# 2. PARTITION: descend a tree, scan leaves, backtrack a bit
def tree_search(q, root, n_leaves):
    return scan(collect_leaves(root, q, n_leaves), q)

# 3. CLUSTER: find nearest centroids, scan their lists
def ivf_search(q, centroids, lists, nprobe):
    near = topk(centroids @ q, nprobe)
    return scan([d for c in near for d in lists[c]], q)

# 4. GRAPH: greedy walk, keep a candidate frontier
def graph_search(q, graph, entry, ef, k):
    frontier, visited = [entry], {entry}
    while improving(frontier):
        for n in graph.neighbours(best_unexpanded(frontier)):
            if n not in visited:
                visited.add(n); frontier = keep_best(frontier + [n], ef)
    return topk(frontier, k)
```

Every one of them ends in `scan` or `topk` over a *small* subset. That is the entire point of an
ANN index: **it is not a faster way to compare vectors, it is a way to compare fewer of them.**
The comparison itself is still the dot product from Chapter 4.

---

### What people get wrong

**"HNSW is always best."** It is the best default for in-memory search at moderate scale. At a
billion vectors its memory cost dominates the architecture, and at ten thousand it is pointless
overhead.

**"ANN means unreliable."** ANN means tunable. At `efSearch = 512` (a large search shortlist,
Chapter 30) HNSW recall is typically around 0.995 on text embeddings, and higher on easy data. That
is close to exact for most practical purposes, and still far faster than brute force at scale.

**"I'll pick the index first."** Pick your *constraints* first: corpus size, latency budget,
memory budget, update frequency, filtering needs. The index follows from those almost
mechanically, as the table above shows.

**"The vector database chooses for me."** It chooses a default. Defaults are tuned for demos and
median workloads, and the tuning parameters are exposed for a reason.

---

### Ninja notes

**Learned indexes** can look like a fifth idea, but they are not. They keep the partition or
cluster family, and replace the hand-written routing rule (which side of a cut, which centroid is
nearest) with a small trained model that predicts which partitions to search. Various
learned-routing approaches head in this direction. The appeal is exactly the argument for graphs:
a learned router adapts to the actual distribution of the data rather than assuming a shape. In
simple words, the families stay four. A learned index is a partition or cluster index whose
routing rule is trained rather than written by hand.

Also worth knowing: for **multi-vector retrieval** (Part V), the taxonomy needs extending. We are
no longer searching for the nearest vector, but for the document whose *set* of vectors best
matches a *set* of query vectors. Two answers exist. PLAID (Chapter 37) uses centroid-based
pruning, which is strategy 3 applied to token vectors. MUVERA (Chapter 38) does something
cleverer. It transforms the set-matching problem into an ordinary single-vector problem, so all
four strategies above become available again unchanged. That reduction is why MUVERA matters.

---

### Key takeaways

- **ANN family = one basic way of choosing which small subset of vectors to compare.** There are
  four: hash, partition, cluster, graph. Everything else is a combination.
- Partitions and clusters pay at their borders, as page 1,140 showed. Graphs pay in memory.
- Graphs lead in-memory benchmarks because they follow the data's shape and degrade gracefully
  in high dimensions.
- Compression is a separate dial. Read index names as `strategy-compression`.
- ANN does not compare vectors faster. It compares fewer vectors.
- Choose from constraints (size, latency, memory, churn, filters), not from fashion.
- Default to HNSW, and deviate when you have measured a reason.

### What's next

Part IV begins. [Chapter 21](./21-lsh.md) starts with the first of the four ideas, hashing, and
the delightfully counterintuitive notion of a hash function designed to *collide*.

We now know the four families every Card Catalog belongs to, where each one pays, and how to pick
one from our constraints.
