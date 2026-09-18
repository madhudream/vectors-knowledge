---
title: "The Recall–Latency–Memory Triangle"
chapter: 18
part: "Part III — Finding Neighbours"
slug: recall-latency-memory
readingTime: "13 min"
summary: "Every ANN index is a position on a three-way trade-off. Once you can name the three axes, the entire zoo of algorithms in Part IV becomes a single map."
tags: [tradeoffs, recall, latency, memory, ann, mental-model]
prev: 17-brute-force
next: 19-measuring-retrieval-quality
---

# The Recall–Latency–Memory Triangle

**The one-paragraph version.** Approximate search trades exactness for speed. We can have
high **recall**, low **latency** and low **memory**, but not all three at once. Every
algorithm in Part IV is a different way of choosing two. Every tuning knob we will ever set
moves us along one edge of this triangle. Once we can name the three corners, the algorithms
stop being a list to memorise and become points on one map.

In this chapter, we will learn about the three things every approximate index trades against
each other: finding the right pages, answering quickly, and fitting in memory. We will also see
why the last few points of recall are so expensive, and where each index in Part IV sits on the
triangle. Finally, we will see how a pipeline of cheap and exact stages escapes the trade-off.

We will cover the following:

- What is the recall–latency–memory triangle
- What is recall
- What are latency, p50 and p99
- What is memory
- Why we cannot have all three
- What "approximate" actually costs
- How each index chooses
- The fourth axis: building and updating

---

## What is the recall–latency–memory triangle

Chapter 17 showed that brute force, checking every vector, is exact and surprisingly fast. But
once the Library grows past about a million vectors, and answers must come back in a few
milliseconds, reading every card becomes too slow.

So we reach for an **ANN index** (approximate nearest neighbour, Chapter 3). It is a Card
Catalog that checks only some of the cards and accepts that it will sometimes miss.

Every such Card Catalog gives something up.

**Triangle = recall + latency + memory. Pick two, and the third one pays.**

```
                    RECALL
                 (did we find the
                  true neighbours?)
                      ╱  ╲
                     ╱    ╲
                    ╱      ╲
                   ╱        ╲
              LATENCY ──── MEMORY
            (how fast?)   (how much RAM?)
```

- **Recall** asks, "Did the Librarian bring back the true nearest pages?"
- **Latency** asks, "How long did the Librarian take?"
- **Memory** asks, "How big is the Card Catalog?"

In simple words, a faster or smaller Card Catalog must look at fewer cards, or at blurrier
cards. Looking at fewer or blurrier cards means sometimes missing the right one.

Let's define each corner properly, using our running question.

---

## What is recall

Our running question is *"Does the Pro plan include single sign-on?"* The full answer needs two
pages from Acme's Library. Page 212 says SSO is included on the Pro plan. Page 1,140 adds that
Pro teams under 50 seats need the Security add-on.

Chapter 17 showed that page 1,140's whole-page vector sits far down the list, at rank 23. So
suppose Acme has cut its pages into chunks of about 100 words (Chapter 11). Brute force, which
checks every chunk, finds the 10 chunks nearest to this question. Page 212 and the SSO chunk of
page 1,140 are both among them. Those 10 chunks are the **true nearest neighbours**. Every
approximate index is graded against them.

**Note:** Through the rest of Parts III and IV, when a search finds "page 1,140", read it as this
SSO chunk, the piece of the page that sits close to our question.

**Recall = the fraction of the true top-k nearest neighbours that the index actually returned.**

When k is 10, we write it as **recall@10**.

- Brute force returns all 10 true neighbours. Its recall is 10 / 10 = 1.0.
- An index that returns 9 of the 10 has recall 0.9 on that query.
- An index with an average recall@10 of 0.95 misses half a neighbour per query. Put simply, it
  drops one true neighbour roughly every two queries.

Now let's watch recall matter. A cheap index returns 10 chunks for our question. Page 212 is
there. The SSO chunk of page 1,140 is not. The Scholar reads the chunks and thinks: "Page 212
says SSO is included on Pro. Nothing else mentions a condition." It answers, "Yes, the Pro plan
includes SSO."

The answer is wrong for every Pro team under 50 seats. Recall on this query was 0.9, which
sounds excellent, and the one chunk it missed was the chunk that mattered.

Now we raise the index's search effort, so it explores more of the Card Catalog per query. The chunk
from page 1,140 comes back. The Scholar now answers, "Yes, but teams under 50 seats need the
Security add-on." The answer is correct. The price was a slower query. That is the triangle at work.

**Note:** This is recall in the *index* sense. Its answer key is brute force, not a person's
judgement of what is relevant. Chapter 19 meets a second, very different "recall", and explains
why mixing the two up is so costly.

---

## What are latency, p50 and p99

**Latency = the wall-clock time one query takes, from question in to pages out.** We measure it
in milliseconds (ms).

A single average hides too much, because a few queries are always much slower than the rest. So
latency is usually quoted at two points:

- **p50** is the time a typical query takes. Half of all queries are faster, half are slower.
  (It is the median.)
- **p99** is the time the slowest 1-in-100 queries take. 99 queries out of 100 finish faster
  than this.

Let's take an example. We send 1,000 questions like our SSO question to the Librarian and time
each one. We sort the 1,000 timings from fastest to slowest. The 500th timing is the p50, say
3 ms. The 990th timing is the p99, say 40 ms.

In simple words, p50 tells us how the system usually feels, and p99 tells us how it feels on a
bad moment. **The p99 is what our users feel.** One support session fires many searches, so a
"rare" slow query reaches almost every user sooner or later.

One more number goes with latency. **QPS = queries per second**, how many queries the system can
answer each second on given hardware. Latency describes one query. QPS describes the whole
crowd. Running many queries side by side can raise QPS while making each one wait a little
longer, so we always check both.

---

## What is memory

**Memory = the bytes an index needs per vector, including its own extra structures.**

Here is the baseline from Chapter 1. One 768-dimensional float32 vector takes 3,072 bytes, about
3 KB. One million of them take about 3.07 GB. A hundred million take about 307 GB.

An index adds structures on top, such as links between vectors or lists of IDs. Some indexes
also shrink the vectors themselves.

- A **flat index** stores the vectors and nothing else: 1×.
- A **graph index** such as HNSW (Chapters 28–31) adds a short list of neighbours for every
  vector. At 768 dimensions that adds only about 5–10%, so about 1.05–1.1×. Chapter 29 does the
  arithmetic.
- A **compressed index** such as IVF-PQ (Chapter 25) replaces each 3,072-byte vector with a
  96-byte code: about 0.03×.

This is the corner that ends up as a line on an invoice. It is also the one most teams ignore
until it is urgent.

---

## Why we cannot have all three

Why must something always give?

Picture the Librarian working under pressure, with three ways to find nearby books.

- Reading every card in the Card Catalog is exact but slow.
- Memorising a big map of shortcuts between cards is fast and accurate, but the map takes space.
- Carrying a tiny, blurry note for each card is fast and small, but the notes sometimes mislead.

Here is the mapping. The cards are the vectors. The map of shortcuts is an index structure, such
as a graph. The blurry notes are compressed vectors.

So we pick any two, and the third is decided for us:

- Want 99% recall and 1 ms latency? We pay in **memory**: an HNSW graph, with its vectors sitting
  in RAM.
- Want 99% recall on a small memory budget? We pay in **latency**: DiskANN (Chapter 32) keeps
  most of its data on an **SSD**, a fast solid-state disk, and waits for the disk on every query.
- Want speed and low memory? We pay in **recall**: aggressive compression such as product
  quantization (Chapter 24) blurs the vectors until some neighbours swap places.

Do not worry about these names yet. We will learn about each of them in detail in Part IV.

---

## What "approximate" actually costs

Here is the fact that makes the whole trade-off acceptable.

Going from 100% recall to 95% recall often costs very little in the answers people see, while
making search many times faster.

Why? Because the neighbours we miss at 95% recall are usually the marginal ones, ranked 8th, 9th
or 10th. Often they are not the pages the answer depends on, and the Scholar leans most on the
top few pages.

**Note:** "Usually" is doing real work in that sentence. For our SSO question, the chunk from page
1,140 might be the 9th-nearest. Miss it, and the answer is half wrong. So we judge recall by the
quality of final answers on our own questions, not by a feeling. Chapter 19 shows how.

But the cost curve is steep at the top end. Here is a typical shape. The exact numbers depend on
the data and the index.

| Recall target | Relative latency |
|---|---|
| 0.80 | 1× |
| 0.90 | ~2× |
| 0.95 | ~4× |
| 0.99 | ~15× |
| 1.00 | ~50–500× (brute force; scale-dependent) |

Read the last two rows together. At 0.95 the index costs about 4×. Brute force costs about
50–500×, and the bigger the Library, the bigger that number (Chapter 17's table shows why).
Divide one by the other. Dropping from exact search to 95% recall buys roughly a 12–125×
speed-up. That is where the common "10–100× faster" rule of thumb comes from.

**The last five points of recall cost more than the first ninety-five.** This is the single most
important shape in Part IV. It is why "just set recall to 100%" is not a strategy. We pick the
recall our answers actually need, measured on real questions, and we stop there.

---

## How each index chooses

Here is a preview of Part IV, placed on the triangle. Memory is relative to raw float32 vectors.

| Algorithm | Recall | Latency | Memory | Sacrifices |
|---|---|---|---|---|
| Flat (brute force) | 1.00 | Poor at scale | 1× | Latency |
| LSH | Low–medium | Good | Medium–high | Recall |
| Annoy (trees) | Medium | Good | Medium | Recall, and a static index |
| IVF | Tunable | Good | ~1× | Recall (boundary effects) |
| IVF-PQ | Medium | Excellent | **0.03×** | Recall (quantization error) |
| **HNSW** | **Excellent** | **Excellent** | **~1.05–1.1× at 768-d, all in RAM** | **Memory** |
| DiskANN | Excellent | Good | ~0.02× in RAM + SSD | Latency (disk reads) |
| Binary + rescore | Very good | Excellent | ~0.08× in RAM + SSD | A little recall |

In the last row, **binary** means keeping just one bit per number (Chapter 26), and **rescore**
means recomputing exact scores for the few pages that survive the first pass. The 1-bit codes
alone are 0.03×. The 0.08× also counts the graph built over them, while the full vectors wait on
SSD for the rescore (Chapter 60 itemises it).

Two rows explain most production systems.

**HNSW is the default because it puts recall and latency first.** Its graph adds only 5–10% at
768 dimensions. The real bill is that its search jumps to random places in the graph, so every
vector must sit in RAM at full or near-full size. If we can afford that RAM, HNSW is very hard to
beat.

**IVF-PQ and DiskANN exist for when we cannot.** At 100 million vectors, HNSW needs roughly
320–340 GB of RAM. IVF-PQ's 96-byte codes need about 10 GB. At a billion vectors, the memory bill
stops being a line item and becomes the architecture.

---

## The fourth axis: building and updating

The triangle hides a fourth dimension: **how long the index takes to build, and how easily it
accepts changes.**

Acme publishes release notes and edits pricing pages all the time, so this corner matters to us.

- **Annoy** must be rebuilt from scratch to add a single vector.
- **IVF** needs its cluster centres (**centroids**) trained on a sample of the vectors. A
  centroid is simply the average position of a group of similar vectors (Chapter 23 builds
  them). If new pages drift far from that sample, we must retrain.
- **HNSW** accepts new vectors one at a time. Deleting is harder. A deleted vector is only marked
  as deleted, and the space comes back later in a clean-up pass (Chapter 31).
- **Flat** accepts every change instantly.

If the Library changed every hour and an index took six hours to build, that index would be
disqualified, wherever it sits on the triangle. This is why Chapter 59 exists. It is also why
some teams with modest collections and constant changes rightly stay on brute force forever.

---

### Under the hood

Measuring our own position on the triangle takes three steps. We should do it for every index
setting we consider.

**Step 1:** Run brute force on a sample of real queries. Its top 10 per query is the answer key.
**Step 2:** Run the candidate index on the same queries, timing each query separately.
**Step 3:** Compare. Recall is the overlap with the answer key. p50 and p99 come from the timings.

```python
import time, numpy as np

# Q: sample queries, V: all page vectors (both unit length, float32).
# `index` is the ANN index under test. set_ef / search / memory_bytes are
# placeholders for whatever your library calls them.

# Step 1. Answer key: brute force, top 10 per query
truth = np.argsort(-(Q @ V.T), axis=1)[:, :10]

# Step 2. Candidate index at one setting, timing every query on its own
index.set_ef(64)
approx, times_ms = [], []
for q in Q:
    t0 = time.perf_counter()
    approx.append(index.search(q, k=10))
    times_ms.append((time.perf_counter() - t0) * 1000)

# Step 3. Recall = overlap with the answer key. Latency = percentiles, not the mean.
recall = np.mean([len(set(a) & set(t)) / 10 for a, t in zip(approx, truth)])
p50, p99 = np.percentile(times_ms, [50, 99])
print(f"recall@10={recall:.3f}  p50={p50:.2f}ms  p99={p99:.2f}ms  "
      f"mem={index.memory_bytes()/1e9:.2f}GB")
```

In simple words, we check the index's answers against the exact answers, and we time every query
on its own, so the slow ones cannot hide inside an average.

Then we sweep the tuning knob and plot recall against latency. The knob is `efSearch` (often
shortened to `ef`) for HNSW and `nprobe` for IVF. Each one controls how much of the index a query
explores (Chapters 30 and 23). That curve is the index's personality. Every decision in Part IV is
about where to sit on it. We cannot make that decision from a blog post, this book included, because
the curve depends on our data's intrinsic dimension (Chapter 6).

---

### What people get wrong

**"Recall 0.99 is the goal."** The goal is end-task quality. If a pipeline reranks 100
candidates down to 5, recall@100 of 0.95 may be indistinguishable from 0.999 in final answer
quality, at a quarter of the cost. Measure the *end task*, not the index.

**Measuring latency without p99.** Graph indexes have long tails. Some queries land in sparse
regions and wander much further. A p50 of 3 ms with a p99 of 90 ms is a completely different
system from one with a p99 of 8 ms.

**Ignoring memory until it is urgent.** 100 million × 768-d float32 vectors = 307 GB of raw
vectors, plus ~5–10% HNSW graph overhead at 768 dimensions (Chapter 29 does the arithmetic).
That is roughly 320–340 GB of RAM. Nobody rents that machine casually, and teams tend to
discover the need at the worst possible moment.

**Benchmarking on the wrong data.** SIFT1M (image features) and GloVe (word vectors) are classic
benchmark datasets, and they are easy ones. Recall at a given `ef` will usually be lower on real
text embeddings, sometimes much lower.

---

### Ninja notes

There is a way to cheat the triangle, and we have already met it several times: the **cascade**.

Nothing says one index must answer the query alone. Retrieve 1,000 candidates with a cheap,
low-recall, low-memory method, then rescore them with something exact. The cheap stage needs only
high recall@1000, a far easier target than recall@10. The expensive stage runs on 1,000 items
instead of 100 million.

```
binary codes + graph (~0.08× memory, a few ms)  →  top 1,000
       ↓
float32 rescore of 1,000 vectors (~1 ms from RAM, a few ms from SSD)  →  top 100
       ↓
cross-encoder rerank of 100 (~20–100 ms on a GPU)  →  top 10
```

End to end, this can deliver near-exact quality at a fraction of the memory of any single
high-recall index. **The triangle constrains any single method. It does not constrain a
pipeline.** Once we see this, the right question stops being "which index?" and becomes "which
cascade?". That is the question Part IV is really answering.

---

### Key takeaways

- **Triangle = recall + latency + memory.** Every ANN index picks two, and the third one pays.
- **Recall** (index sense) is the fraction of the true top-k neighbours returned. A recall@10 of
  0.95 drops about one neighbour every two queries, and the dropped page can be the one that
  matters.
- **p50** is the typical query's time and **p99** is the slowest 1-in-100. Users feel the p99.
- The last 5% of recall costs more than the first 95%. Exact search is roughly 10–100× slower
  than a 0.95-recall index.
- HNSW chooses recall and latency. Its graph adds only 5–10% at 768-d, but every vector must sit
  in RAM. IVF-PQ and DiskANN choose memory, paying in recall or in disk reads.
- Build time and updatability are a hidden fourth axis that can disqualify an index outright.
- Always measure recall against brute force on *our own* data.
- Cascades escape the triangle: cheap and broad first, exact and narrow second.

### What's next

We have used "recall" in one narrow sense. [Chapter 19](./19-measuring-retrieval-quality.md)
defines it precisely, adds MRR and nDCG, and untangles the two very different things people mean
by the word.

We now know the three corners every index trades between, why the top of the recall curve is so
expensive, and how a cascade of cheap and exact stages slips past the trade.
