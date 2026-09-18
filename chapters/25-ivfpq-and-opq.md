---
title: "IVF-PQ and OPQ in Practice"
chapter: 25
part: "Part IV — Index Structures"
slug: ivfpq-and-opq
readingTime: "12 min"
summary: "Combine clustering with quantization and you get the index that serves most of the world's billion-vector workloads. Here is how to configure and tune it without guessing."
tags: [ivfpq, opq, faiss, tuning, billion-scale, residual]
prev: 24-product-quantization
next: 26-scalar-binary-quantization
---

# IVF-PQ and OPQ in Practice

**The one-paragraph version.** IVF (Chapter 23) cuts down *how many* vectors we score. PQ
(Chapter 24) makes *each* score cheap and each vector tiny. Stack the two and a billion
768-dimensional vectors fit in about 100 GB instead of 3 TB, searchable in milliseconds. Three
refinements make the combination work properly. We compress the **residual**, the gap between
a vector and its cluster centre, instead of the raw vector. We apply a learned rotation,
**OPQ**, so every piece of the vector carries a fair share of the information. And we
**rescore** the shortlist with full vectors before answering.

In this chapter, we will learn about IVF-PQ, the index behind most very large vector search
systems. We will also see the residual trick, OPQ, the rescoring step, how to size the index,
how to tune it without guesswork, and when HNSW is the better choice.

We will cover the following:

- What is IVF-PQ
- Why we need IVF-PQ
- How IVF-PQ works
- The residual trick
- What is OPQ
- What is rescoring
- Configuring and tuning IVF-PQ
- IVF-PQ vs HNSW, and when to use which one

---

## What is IVF-PQ

A quick reminder of the two parts. **IVF** groups the vectors into clusters with k-means and
searches only the few clusters nearest the query. **PQ** stores each vector as a short string
of codebook IDs and scores it with a lookup table.

**IVF-PQ = IVF to choose which vectors to score + PQ to make each vector small and each score
cheap.**

Two separate costs get two separate fixes.

- **How many vectors do we compare?** IVF fixes that. We probe a few clusters out of
  thousands.
- **How expensive is each comparison?** PQ fixes that. Each vector is 96 bytes and a table
  lookup.

The savings multiply. Suppose IVF cuts the candidate set 100×, and PQ makes each candidate
32× smaller and several times cheaper to score. Together they answer in milliseconds what
brute force would take seconds to do, at about 3% of the memory.

In FAISS (Meta's vector search library), this index is written `IVF{nlist},PQ{m}`. It has been
the backbone of web-scale similarity search for a decade.

---

## Why we need IVF-PQ

Let's go back to Acme at scale. The knowledge-base product hosts 5,000 customer companies and
about 100 million chunks. As 768-d `float32` vectors, that is 307 GB.

Now, the question is, why not use just one of the two ideas?

**IVF alone** searches only a few clusters, but it stores every vector in full. The memory
bill stays at 307 GB.

**PQ alone** shrinks the vectors to 9.6 GB, but it still scores all 100 million codes for every
question. Even at 96 lookups each, that is billions of lookups per query.

**IVF-PQ** gets both wins. With `nlist = 16,384` clusters and `nprobe = 16`, a query touches
about 16/16,384 of the collection, roughly 98,000 chunks, and each of those is 96 bytes.

In simple words, IVF shrinks the haystack, and PQ shrinks every straw in it.

---

## How IVF-PQ works

**Phase 1: Building the index.**

**Step 1:** Take a well-shuffled sample of the vectors.

**Step 2:** (Optional, recommended) Learn an OPQ rotation, explained below, and rotate the
sample with it.

**Step 3:** Run k-means on the sample to find `nlist` cluster centres, the **centroids**.

**Step 4:** For each sample vector, find its nearest centroid and subtract it. What is left is
the residual.

**Step 5:** Train the PQ codebooks on those residuals.

**Phase 2: Adding vectors.**

**Step 1:** Rotate the vector with OPQ, if we learned a rotation.

**Step 2:** Find its nearest centroid.

**Step 3:** Subtract the centroid to get the residual.

**Step 4:** Encode the residual with PQ into 96 bytes.

**Step 5:** Append the vector's ID and its 96 bytes to that centroid's list.

**Phase 3: Answering a query.**

**Step 1:** Rotate the query with the same OPQ rotation.

**Step 2:** Compare the query with all `nlist` centroids.

**Step 3:** Keep the `nprobe` nearest clusters.

**Step 4:** In each of those clusters, score every stored code with PQ's lookup table
(Chapter 24).

**Step 5:** Keep the best few hundred candidates.

**Step 6:** Rescore those candidates with their full vectors and return the top k.

Do not worry, we will learn about residuals, OPQ and rescoring one at a time.

---

## The residual trick

Here is the refinement that makes IVF-PQ much better than IVF and PQ bolted together naively.

We do not compress the vector. We compress what is *left over* after subtracting its
cluster's centroid:

$$ r = v - c_{\text{assigned}} $$

Then we store `(cluster_id, PQ_code(r))`. In simple words, the centroid already says roughly
where the vector is, so PQ only has to describe the small step from the centroid to the
vector. (Chapter 24 used the word residual for the part PQ got wrong. In both cases, a residual
is whatever a coarser step left over.)

Let's take an example with a single number, so we can do it by hand. Say one coordinate of
the page vectors ranges from −1 to 1 across the whole Library. We have 2 bits for it, so 4
levels.

**Without the residual trick.** The 4 levels must cover the whole range: −0.75, −0.25, 0.25
and 0.75.

- Page 212 has 0.62 here. The nearest level is 0.75. The error is 0.13.
- Page 1,140 has 0.55. The nearest level is also 0.75. The error is 0.20.

Both pages get the same code. On this coordinate, PQ cannot tell them apart.

**With the residual trick.** Suppose both pages sit in the same "login and security" cluster,
whose centroid has 0.58 on this coordinate.

- Page 212's residual is 0.62 − 0.58 = +0.04.
- Page 1,140's residual is 0.55 − 0.58 = −0.03.

Residuals are small, so the 4 levels only need to cover roughly −0.1 to 0.1: −0.075,
−0.025, 0.025 and 0.075.

- +0.04 goes to 0.025. The error is 0.015.
- −0.03 goes to −0.025. The error is 0.005.

Same 2 bits. The two pages now get different codes, and the errors are about ten times
smaller.

Why does one set of codebooks work for residuals from every cluster? Because residuals are
**small and centred on zero**. Each residual has had its own cluster's centre removed, so all
of them occupy roughly the same compact region. Raw vectors, by contrast, force 256 centroids
per position to cover the entire spread of the Map Room at once.

The improvement is large and free, because we already computed the centroid during
assignment. Every serious IVF-PQ implementation does this by default.

---

## What is OPQ

PQ cuts a vector into contiguous blocks: numbers 0–7, then 8–15, and so on. That cut is
arbitrary, and it matters.

Suppose numbers 0–7 carry most of the variation between pages, while numbers 600–607 are
nearly the same for every page. Subvector 0 then has a very hard job, with 256 centroids to
cover a wide spread. Subvector 75 spends its whole codebook describing almost nothing. The
total error is dominated by the overloaded blocks.

**OPQ = Optimized Product Quantization: learn a rotation that spreads the variation evenly
across the blocks, then run PQ.**

A rotation is multiplication by an **orthogonal** matrix $R$. Orthogonal means
$R^\top R = I$: flipping the matrix over its diagonal gives exactly its undo button.

$$ \text{code} = \text{PQ}(Rv), \qquad R^\top R = I $$

In simple words, a rotation turns the whole Map Room without stretching or squashing it.
Every distance and every angle stays exactly the same. Only the directions we call "number 0",
"number 1" and so on change.

Let's see it on four numbers split into two blocks, `[x0, x1]` and `[x2, x3]`. Suppose
`x0` and `x1` each have a **variance** (the average squared distance from their mean, a measure
of spread) of 4, while `x2` and `x3` each have variance 0.01, and all four vary independently.
Block 1 carries a variance of 8. Block 2 carries 0.02, and its codebook is nearly wasted.

Now rotate each pair `(x0, x2)` and `(x1, x3)` by 45 degrees. The new first number is
$(x_0 + x_2)/\sqrt{2}$, an equal mix of one number from each block, so its variance is
(4 + 0.01) / 2 = 2.005. The same holds for the other three new numbers. Both blocks
now carry 4.01. Every codebook has an equally hard job, and none is wasted.

OPQ learns its rotation from the data. It aims to make the variance as even as possible across
the $m$ blocks, and the blocks as independent of each other as possible.

The cost is one $d \times d$ matrix multiply per query and per indexed vector, plus the matrix
in memory (768 × 768 floats, about 2.4 MB). The benefit is typically several points of recall
at the same compression. **Use OPQ unless you have measured that it does not help your data.**

In FAISS, it is part of the factory string: `OPQ96,IVF4096,PQ96`.

---

## What is rescoring

PQ scores are approximate (Chapter 24). Near the top of the list, small errors swap pages
around.

**Rescoring = take a wider shortlist from the compressed index, then re-check just those
candidates with their full, uncompressed vectors.** FAISS calls this step a **refine**.

Let's first see what happens without it. The Librarian asks, *"Does the Pro plan include
single sign-on?"* IVF-PQ returns its top 10. Page 212 is at rank 1. PQ's error has pushed page
1,140 down to rank 14, just outside the cut.

The Scholar reads page 212 alone and answers, "Yes, SSO is included on Pro." That answer
misses the add-on rule for teams under 50 seats.

Now, let's see what rescoring does.

**Step 1:** Ask IVF-PQ for 100 candidates instead of 10. Page 1,140, at rank 14, is inside.

**Step 2:** Fetch the full 768-d vectors of those 100 candidates from wherever we keep them.

**Step 3:** Compute the exact dot product of the query with each of the 100.

**Step 4:** Sort by exact score and keep the top 10.

Page 1,140 moves up to rank 2, where its true score puts it. The Scholar reads both pages and
answers, "Yes, but teams under 50 seats need the Security add-on." Now the answer is complete.

In simple words, PQ only has to get the right pages *into* the top 100, and it does that
reliably. The exact check puts them in the right order. Asking for 10–20× more candidates than
we need is the usual setting.

---

## Configuring and tuning IVF-PQ

For $n$ vectors of dimension $d$:

| Parameter | Rule of thumb | Notes |
|---|---|---|
| `nlist` | $\sqrt{n}$ to $4\sqrt{n}$ | 4,096 for 10M; 16,384 for 100M; 65,536 for 1B |
| `m` (subvectors) | $d/4$ to $d/16$ | 96 for 768-d gives 32× compression |
| `nbits` | 8 | 256 centroids per codebook; 4-bit variants are faster, coarser |
| `nprobe` | 8–64 | Runtime knob. Sweep it against your eval set |
| Training sample | 30–256 × `nlist` vectors | FAISS warns if too few |
| Rescore depth | 10–20 × final k | The step people skip |

**Memory, for Acme's 100 million 768-d chunks:**

```
raw float32:            1e8 × 3,072 B  = 307 GB
IVF-PQ (m=96):          1e8 ×    96 B  = 9.6 GB
  + IDs (8 B)                          = 0.8 GB
  + centroids (16k×768×4)              = 0.05 GB
                                  total ≈ 10.5 GB
```

**And if Acme grows to a billion:**

```
raw float32:            1e9 × 3,072 B  = 3.07 TB
IVF-PQ (m=96):          1e9 ×    96 B  =  96 GB
  + IDs (8 B)                          =   8 GB
  + centroids (65k×768×4)              = 0.2 GB
                                  total ≈ 104 GB
```

Three terabytes becomes about a hundred gigabytes. That is the difference between a cluster of
machines and a single large one, and it is the entire reason this index exists.

**The tuning procedure.** Do this in order. It takes a day or two, mostly waiting on batch builds,
and removes the guesswork.

**Step 1:** Build ground truth. Compute the exact top 10 for 200 real queries by brute force over
the full collection (Chapter 19). It is a one-off batch job.

**Step 2:** Build the index at full size, with `nlist` fixed at $\sqrt{n}$. Do not tune `nlist`
first, because it interacts with everything. On a small test slice, each list covers a much
larger share of the data, so the `nprobe` it suggests would not carry over.

**Step 3:** Sweep `nprobe` over 1, 2, 4, 8, 16, 32, 64, 128. Plot recall against latency. This
is your index's curve.

**Step 4:** Choose the knee, where the curve flattens. Usually 16–32.

**Step 5:** At that `nprobe`, sweep `m` over 48, 96, 192. Plot recall against memory.

**Step 6:** Add rescoring and measure again. Recall jumps. Now check whether a smaller `m` is
affordable after all.

**Step 7:** Only then consider other `nlist` values, and only if p99 latency is a problem.

---

## IVF-PQ vs HNSW, and when to use which one

HNSW is the graph index of Chapters 28–31. It usually keeps full or int8 vectors in RAM.

| | IVF-PQ (with OPQ, rescoring) | HNSW |
|---|---|---|
| Memory at 100M × 768-d | ~10.5 GB in RAM + full vectors elsewhere | ~320 GB float32, ~90 GB int8 |
| Training | Centroids, rotation, codebooks | None |
| Recall | Good, near-exact after rescoring | High without rescoring |
| Staleness | Centroids and codebooks drift | No trained parts to drift |
| Knob at query time | `nprobe`, rescore depth | `efSearch` |

**Advantages of IVF-PQ:** the smallest RAM footprint of any mainstream index, one clean
runtime knob, and a well-understood path to billions of vectors.

**Disadvantages of IVF-PQ:** training to do and redo, approximate scores that need a rescoring
store, and more parameters to get right.

We must use **HNSW** when the vectors fit in RAM: typically tens of millions of vectors, or around
100 million with int8 (Chapter 26). It gives better recall at lower latency, with no training, no
codebooks and nothing to go stale. We must use **IVF-PQ** when memory is the binding constraint,
which in practice means beyond roughly 100 million vectors per machine. Many strong systems use both
ideas at once, as the Ninja notes show.

---

### Under the hood

```python
import faiss, numpy as np

d, nlist, m = 768, 4096, 96

index = faiss.index_factory(d, f"OPQ{m},IVF{nlist},PQ{m}", faiss.METRIC_INNER_PRODUCT)
index.train(train_sample)          # learns rotation + centroids + codebooks (Phase 1)
index.add(V)                       # encodes residuals (Phase 2)

faiss.extract_index_ivf(index).nprobe = 32   # the OPQ wrapper hides the IVF layer
D, I = index.search(Q, k=100)      # 100 candidates, approximate (Phase 3, Steps 1–5)

# Rescore with full-precision vectors (kept on disk / object store / a KV store)
def rescore(q, candidate_ids, k=10):                # Phase 3, Step 6
    candidate_ids = candidate_ids[candidate_ids >= 0]   # FAISS pads short results with -1
    full = fetch_full_vectors(candidate_ids)        # (≤100, 768) float32
    s = full @ q
    top = np.argsort(-s)[:k]
    return [(candidate_ids[i], float(s[i])) for i in top]
```

`faiss.extract_index_ivf` reaches past the OPQ rotation to the IVF layer, where `nprobe`
lives. In simple words, setting `index.nprobe` directly does not work once OPQ wraps the
index, so we set it one level down.

That `rescore` function is not optional. Without it, PQ's approximation error decides the
final ranking. With it, PQ only has to get the right answers *into* the top 100, which it does
reliably. This one step often moves recall@10 from around 0.90 to around 0.98, and costs about
a millisecond when the full vectors are close at hand.

---

### What people get wrong

**Under-training.** FAISS prints a warning when you train on too few vectors, and people
ignore it. Use at least 30 × `nlist` vectors, ideally 256 ×.

**Training on a non-representative sample.** The first 100,000 rows of a time-ordered corpus
are not a sample. Shuffle.

**Skipping rescoring**, then concluding PQ is inaccurate. It is inaccurate *by design*, and
the design assumes a rescore.

**Comparing IVF-PQ to HNSW on recall alone.** They sit at different corners of the triangle
(Chapter 18). Compare recall *at equal memory*, and IVF-PQ often wins at large scale because
HNSW cannot be shrunk as far.

**Forgetting to store full vectors somewhere.** If you compress to PQ codes and keep nothing
else, you cannot rescore, cannot re-index with different parameters, and cannot migrate
models. Keep the originals in cheap object storage.

---

### Ninja notes

**Know when HNSW beats this.** While the vectors fit in RAM, HNSW gives better recall at lower
latency and simpler operations. IVF-PQ earns its complexity when memory
is the binding constraint.

**Consider the hybrid: an HNSW graph over the centroids.** With `nlist = 65,536`, comparing the
query to every centroid becomes a real cost: 65,536 dot products per query. The FAISS factory
string `IVF65536_HNSW32,PQ96` builds an HNSW graph *over the centroids*, so finding the nearest
`nprobe` centroids is itself sublinear. This is the coarse-to-fine idea nested one level
deeper. It makes very large `nlist` values practical, which in turn keeps lists short and scan
costs low.

**Watch for cluster imbalance at scale.** As in Chapter 23, uneven lists wreck p99. At a
billion vectors this is not a subtlety. It can be the difference between a 20 ms tail and a
400 ms one.

---

### Key takeaways

- **IVF-PQ = IVF to choose which vectors to score + PQ to make each vector small and each
  score cheap.**
- Quantize the **residual** from the centroid, not the raw vector. Better codebooks, for free.
- **OPQ** learns a rotation that spreads variance evenly across subvectors. Use it.
- **Rescoring** re-checks a 10–20× wider shortlist with full vectors. It is the step that makes
  the index work.
- Acme's 100M chunks: 307 GB raw → ~10.5 GB. A billion 768-d vectors: 3 TB raw → ~104 GB with
  `OPQ96,IVF65536,PQ96`.
- Tune in order: fix `nlist`, sweep `nprobe`, then `m`, then add rescoring.
- While the vectors fit in RAM (around 100M with int8), prefer HNSW.

### What's next

PQ is not the only way to compress. [Chapter 26](./26-scalar-binary-quantization.md) covers the
simpler alternatives, including the one that throws away everything but the sign of each
number and still works.

We now know how IVF and PQ combine, why residuals, a rotation and a rescore make the
combination strong, and how to size and tune it for a hundred million chunks.
