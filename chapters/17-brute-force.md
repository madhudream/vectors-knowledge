---
title: "Brute Force and the Flat Index"
chapter: 17
part: "Part III — Finding Neighbours"
slug: brute-force
readingTime: "13 min"
summary: "Comparing the query to every single vector is exact, trivial to implement, and far faster than most people believe. Know exactly when you can get away with it."
tags: [brute-force, flat-index, exact-search, baselines, performance]
prev: 16-choosing-a-model
next: 18-recall-latency-memory
---

# Brute Force and the Flat Index

**The one-paragraph version.** The simplest search compares the question with every vector
and keeps the best k. It is **exact**: it always finds the true nearest neighbours, by
definition. It is also a single matrix multiplication. On a modern machine it searches a
million 768-dimensional vectors in tens of milliseconds, and Acme's 40,000 pages in about a
millisecond. Most systems reach for an approximate index far earlier than they need to, and
pay in complexity, memory and correctness for speed they were not short of.

In this chapter, we will learn about brute-force search and the flat index, the simplest
Card Catalog there is. We will see how it works, why it is exact, and why it is much faster
than it sounds. We will use it to settle a real argument about who lost page 1,140. We will
also see how to stretch it further, and when it finally stops being enough.

We will cover the following:

- What is brute-force search
- How brute-force search works
- Why it is faster than it sounds
- How fast it is, in numbers
- An example: who lost page 1,140?
- Why it is the right default while we build
- Making brute force go further
- Brute force vs an approximate index
- When to use which one

---

## What is brute-force search

The Librarian has a question. The **Card Catalog**, our index, has 40,000 cards, one per
book. Each card holds one page's vector.

**Brute-force search = compare the question with every single vector, and keep the k best.**

**Flat index = a Card Catalog that keeps all the vectors in one plain table, with no
shortcuts, and searches it by brute force.**

It is called *flat* because there is no structure on top: no clusters, no graph, no tree.
There is just one table of numbers, one row per page.

In Chapter 1 we said the Card Catalog is the structure that lets us find nearby dots
*without* checking every dot. The flat index is the one Card Catalog that does check every
dot. Every other index in Part IV is a shortcut around it.

Because it checks every card, brute force is **exact**. It always returns the true k nearest
vectors. In Chapter 3 we met ANN, *approximate* nearest neighbour search. Brute force is the
one search that is not approximate.

In simple words, brute force is the honest baseline. It is slow only if the Library is huge,
and it is never wrong about which vectors are nearest.

---

## How brute-force search works

**Phase 1: Building the index.**

**Step 1:** Embed all 40,000 of Acme's pages.

**Step 2:** Normalize every vector to length 1 (Chapter 5), so that the dot product equals
the cosine similarity.

**Step 3:** Stack the vectors into one table of 40,000 rows and 768 columns of `float32`.
That table, 122.9 MB, is the whole index.

**Phase 2: Answering a question.**

**Step 1:** Embed the question and normalize it.

**Step 2:** Multiply the table by the question vector. Out come 40,000 scores, one per page,
from a single matrix multiplication.

**Step 3:** Pick out the k highest scores, without sorting all 40,000.

**Step 4:** Sort just those k, and return their page numbers.

That is it. There is nothing else to build, tune or repair.

---

## Why it is faster than it sounds

Let's make the job harder than Acme's: a Card Catalog with a million cards instead of 40,000. The
brute-force approach is exactly what it sounds like: pick up every card, score it, keep the ten
best.

Now, the question is, how can checking a million cards possibly be fast?

Because a computer doing this is not "reading". It streams one continuous block of memory
through a vector (SIMD) unit that performs eight or sixteen multiply-adds per cycle. SIMD,
from Chapter 4, means one instruction working on several numbers at once. The work is so
regular that it runs close to the hardware's limit.

That leads to the key idea of this chapter: **brute-force vector search is bound by memory
bandwidth, not by arithmetic.** **Memory bandwidth** is how many bytes per second the
processor can pull in from main memory. The cost is "how many bytes must we read", and modern
hardware reads tens of gigabytes per second.

Think of it like a supermarket checkout:

- The cashier is the processor's SIMD unit.
- The groceries are the numbers in the table.
- The conveyor belt is memory bandwidth.
- The cashier can scan faster than the belt delivers items, so the belt sets the pace.

In simple words, the processor can multiply faster than memory can feed it numbers. So the
bill for brute force is paid in bytes moved, not in sums done.

---

## How fast it is, in numbers

One million vectors, 768 dimensions, `float32`:

```
Memory to scan:  1,000,000 × 768 × 4 bytes      =  3.07 GB
Operations:      1,000,000 × 768 multiply-adds  =  768 million multiply-adds
```

Means, a million-card search reads about 3 GB of memory and does 768 million
multiply-adds.

A modern server CPU with AVX-512 (the SIMD instruction set on recent Intel and AMD server
chips) sustains roughly 20–50 GB/s of streaming bandwidth. So scanning 3 GB takes on the
order of **60–150 ms** on one core, and **10–30 ms** across a handful of cores. A GPU, with
1–3 TB/s of memory bandwidth, does it in **1–3 ms**.

Acme's 40,000 pages are only 122.9 MB. That is **2.5–6 ms** on one core, **about 1 ms** on
several, and well under a millisecond on a GPU.

Now let's scale the table:

| Corpus | float32 size | CPU (multi-core) | GPU |
|---|---|---|---|
| 10,000 | 30 MB | < 1 ms | < 1 ms |
| 100,000 | 307 MB | ~2 ms | < 1 ms |
| 1,000,000 | 3.1 GB | ~20 ms | ~2 ms |
| 10,000,000 | 31 GB | ~200 ms | ~20 ms |
| 100,000,000 | 307 GB | ~2 s | too big for one GPU |

Read this table as a set of decisions:

- **Under 100k vectors: use brute force.** An ANN index adds build time, memory overhead,
  tuning parameters and approximation error, to save a millisecond nobody will notice.
- **100k–1M: brute force is still viable**, especially with quantization, and especially
  while we are still changing chunking and models.
- **Above ~1M with a latency target: we need an index.** Latency is how long one query takes
  (Chapter 18). Part IV builds the indexes.

> **The most common premature optimisation in this field** is building an HNSW index (the
> graph index of Chapters 28–31) over 40,000 chunks. It costs exactness, memory and a tuning
> surface. It buys a speedup from about 1 ms to 0.4 ms, in a pipeline where the LLM call takes
> 3,000 ms.

---

## An example: who lost page 1,140?

Let's return to our running question:

> *"Does the Pro plan include single sign-on?"*

Acme's team has built an HNSW index over its 40,000 pages. HNSW is approximate. They search,
hand the top 5 pages to the Scholar (the LLM), and get back half an answer: *"Yes, the Pro
plan includes single sign-on."* Page 1,140, with its add-on caveat, is missing.

Now, the question is, who lost page 1,140? There are two suspects:

- **The Map Room.** The embedding model placed page 1,140 far from the question.
- **The Card Catalog.** The approximate index skipped page 1,140, even though it was close.

Let's first see what a team without an exact baseline does. They cannot tell the two suspects
apart. HNSW is approximate, so they blame the index. They spend a week raising its search
settings and rebuilding it. Page 1,140 still does not appear in the top 5.

The diagnosis was wrong, and so was the week.

Now, let's see what a team with a flat index does. They run the same question through brute
force, which cannot miss:

| Page | What the page is about | Exact rank (flat index) | Rank from HNSW |
|---|---|---|---|
| 212 | Pro plan page: SSO included on Pro | 1 | 1 |
| 88 | Setting up single sign-on (how-to guide) | 2 | 2 |
| 2,301 | SSO on the Enterprise plan | 3 | 3 |
| 45 | Pro plan overview | 4 | 4 |
| 9,012 | Troubleshooting SSO login errors | 5 | 5 |
| 1,140 | The Security add-on | 23 | 23 |

The search took about a millisecond, and the table settles the argument. Page 1,140 truly is
the 23rd-nearest page. The index did not lose it. The embedding model put it there, exactly as
we saw in Chapter 13.

So they fix the right thing. They fetch the top 100 instead of the top 5, and add the
cross-encoder reranker from Chapter 13. Page 1,140 moves up to rank 2, and the Scholar
answers: *"Yes, SSO is included on Pro, but teams under 50 seats also need the Security
add-on."*

The diagnosis was correct, and it took minutes instead of a week.

**Note:** If brute force had put page 1,140 at rank 4 while HNSW left it out, the index would
have been the culprit. Either way, the flat index gives a definite answer. And at 40,000
pages, Acme could simply keep the flat index in production: exact, and about a millisecond.

---

## Why it is the right default while we build

Brute force has properties that matter more than speed while a system is still being built.

**It is exact.** Recall, the share of the true nearest neighbours that a search returns
(Chapter 18), is 100% by construction. When results are bad, we know with certainty that the
problem is upstream: the model, the chunking or the query. Retrieval cannot be at fault. That
certainty is worth a great deal when debugging, as the example showed.

**It is the measuring stick.** We cannot evaluate an ANN index without exact ground truth to
compare against. Every recall number in Part IV is defined as *agreement with brute force*.
If we never run brute force, we have no idea what our approximate index is missing.

**It supports arbitrary filters for free.** Take a filter such as `WHERE tenant_id = 42 AND
created_at > '2026-01-01'`. Means, "only this one customer's pages, and only ones written
this year". With a flat index that is trivial: filter the table first, then scan what
remains. Chapter 33 shows how genuinely hard this is for graph-based indexes.

**Updates are instantaneous.** To add a page, append a row. There is no rebuild, no graph
repair, and no compaction (the periodic clean-up some indexes need, Chapters 31 and 59) to
schedule.

---

## Making brute force go further

Before jumping to an approximate index, three techniques buy roughly an order of magnitude
while staying exact or near-exact.

**1. Quantize.** Quantization means storing each number in fewer bits (Chapter 26). float16
halves the bytes, and therefore roughly halves the time. int8 quarters it. Binary
quantization is 32× smaller, and compares vectors with `popcount` (a single CPU instruction
that counts the 1-bits in a word) instead of multiplication. A 100-million-vector binary scan
reads only 9.6 GB, which is genuinely feasible on one machine.

**2. Truncate.** With an MRL model (Chapter 15), scanning only the first 256 of 768 dimensions
is 3× less memory traffic. Then rescore the top 1,000 at full width.

**3. Partition by metadata.** Metadata means the extra fields stored with each page, such as
which customer owns it. If every query is filtered to one tenant (one customer's data), we are not
scanning 100 million vectors. We are scanning that tenant's slice. In Part VIII,
Acme hosts knowledge bases for 5,000 customer companies, about 100 million chunks in total.
That is only 20,000 chunks per customer on average, which is brute-force territory. Many "we
need a billion-scale index" problems dissolve on inspection into "we need thousands of small
ones", which Chapters 58 and 62 cover.

Combine all three, and brute force covers far more of the problem space than its reputation
suggests.

---

## Brute force vs an approximate index

| | Flat index (brute force) | Approximate index (Part IV: IVF, HNSW, …) |
|---|---|---|
| Results | Exact, always | Approximate: recall below 100% |
| Query cost | Grows in step with the corpus | Grows far more slowly |
| Build time | None | Minutes to hours |
| Extra memory | None | Graph or cluster overhead |
| Filters | Trivial | Harder, especially for graphs (Chapter 33) |
| Updates | Append a row | Graph repair or re-clustering, periodic rebuilds |
| Knobs to tune | None | Several |
| Batched queries | Nearly free (Ninja notes) | Much smaller gain |

**Advantages of brute force.**

- Exact results, so it doubles as ground truth.
- Nothing to build, tune or repair. Filters and updates are trivial.

**Disadvantages of brute force.**

- Query time grows in step with the corpus. At 100 million vectors it takes seconds.
- The whole table must fit in fast memory, or be read from disk on every query.

---

## When to use which one

We must use **brute force** when the corpus is under ~100k vectors, or while we are still
iterating on models and chunking. We must also use it whenever we need ground truth, for batch
jobs, and when every query is filtered down to a small partition.

We must use an **approximate index** when a single searchable collection passes ~1 million
vectors and we have a latency target that brute force cannot meet.

Many strong systems use both: an approximate index to serve live traffic, and a flat index
over a sample of queries to measure what the approximate one is missing.

---

### Under the hood

The whole implementation:

```python
import numpy as np

class FlatIndex:
    def __init__(self, dim):
        self.V = np.empty((0, dim), dtype=np.float32)
        self.ids = []

    def add(self, vectors, ids):
        v = vectors / np.linalg.norm(vectors, axis=1, keepdims=True)
        self.V = np.vstack([self.V, v.astype(np.float32)])
        self.ids.extend(ids)

    def search(self, q, k=10):
        q = (q / np.linalg.norm(q)).astype(np.float32)
        scores = self.V @ q                                # the entire search
        idx = np.argpartition(-scores, min(k, len(scores)-1))[:k]
        idx = idx[np.argsort(-scores[idx])]
        return [(self.ids[i], float(scores[i])) for i in idx]

index = FlatIndex(768)
index.add(page_vectors, page_ids)                          # Acme's 40,000 pages, 122.9 MB
index.search(embed("Does the Pro plan include single sign-on?"), k=5)
```

In simple words, `add` normalizes and stacks rows, and `search` is one multiplication plus
picking the top k. The class implements Phase 1 and Phase 2 above, line for line.

Three details are worth more than they look.

**`argpartition`, not `argsort`.** **Big-O** notation describes how work grows as the input
grows. Fully sorting n scores is $O(n \log n)$. Partitioning, which only separates the top k
from the rest, is $O(n)$: its time grows in step with n, rather than with n times log n. On
ten million scores that is a meaningful fraction of total query time.

**`float32`, not `float64`.** NumPy defaults to `float64`. Using it doubles the memory
traffic and, since this work is bandwidth-bound, roughly halves the speed, for precision
nobody can use.

**Contiguity.** `V` must be one contiguous, C-ordered array, meaning all the rows sit end to
end in a single block of memory. A list of separate arrays destroys the streaming access
pattern and can cost an order of magnitude.

In practice we would use FAISS's `IndexFlatIP` (FAISS is Meta's open-source vector search
library), which is the same algorithm with hand-tuned SIMD kernels, or a GPU implementation.
But the code above is genuinely all it is.

---

### What people get wrong

**"Brute force means slow."** It means exact. At the scale most applications actually
operate, it is also fast.

**"I'll add an index now to avoid migrating later."** Migrating from flat to HNSW is a few
lines of configuration. Debugging a system where you never established exact ground truth is
not.

**Using float64.** Doubles memory bandwidth for precision that is irrelevant here.

**Sorting all scores.** Use a partial selection.

**Rebuilding the array on every insert.** `np.vstack` copies the whole table. For ingestion,
preallocate or batch.

---

### Ninja notes

The bandwidth-bound framing predicts something useful: **query batching is nearly free.**
Searching one query against a million vectors reads 3 GB. Searching a hundred queries against
the same million vectors *also* reads 3 GB. The table is streamed once, and every query
reuses each block of memory already pulled into the CPU's cache (the small, very fast memory
right next to the processor). Throughput per query is often tens of times higher, and on a GPU
can reach 50–100×.

So if a workload permits batching (offline evaluation, bulk deduplication, nightly
recommendation refresh, backfills), brute force scales to sizes that seem absurd.
Deduplicating ten million documents pairwise is a blocked matrix multiplication, and a single
modern GPU will do it in tens of minutes.

The pattern generalises: **whenever you are memory-bandwidth-bound, amortise the memory
traffic across as many queries as you can.** This is also why ANN indexes sometimes *lose* to
brute force on batch workloads. Random access into a graph cannot amortise anything.

---

### Key takeaways

- **Brute force = compare the question with every vector and keep the best k.** A flat index
  does it with one matrix multiplication, and is exact by definition.
- It is bound by memory bandwidth: the cost is bytes scanned, not operations performed.
- Under ~100k vectors it is the right answer. Up to ~1M it is often still right. Acme's
  40,000 pages take about a millisecond.
- It is the ground truth every ANN recall number is measured against, and the fastest way to
  tell a model problem from an index problem.
- Filters and updates are free, which is not true of graph indexes.
- Quantization, truncation and partitioning extend its range by an order of magnitude.
- Batched queries are almost free. Exploit that for any offline workload.

### What's next

Once brute force stops being enough, we must trade something away.
[Chapter 18](./18-recall-latency-memory.md) names the three things we trade between, and why
we can never have all three.

The flat index is exact, simple and faster than it sounds, and it remains the ruler every clever
index is measured against.
