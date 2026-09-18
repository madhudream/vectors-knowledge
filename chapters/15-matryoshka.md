---
title: "Matryoshka Embeddings"
chapter: 15
part: "Part II — Where Embeddings Come From"
slug: matryoshka
readingTime: "11 min"
summary: "Train a model so the first 64 dimensions are already a good embedding, the first 256 are better, and all 768 are best. Then truncation becomes a free tuning knob."
tags: [matryoshka, mrl, dimensionality, efficiency, two-stage]
prev: 14-asymmetry-and-instructions
next: 16-choosing-a-model
---

# Matryoshka Embeddings

**The one-paragraph version.** Normally, cutting an embedding in half destroys it, because
the meaning is smeared across all of its dimensions. **Matryoshka Representation Learning
(MRL)** trains the model so that meaning is packed front to back: the first dimensions carry
the broad picture, and later ones add detail. We can then keep only the first 64, 256 or 512
numbers of a vector and still have a working embedding. One model, many sizes, chosen
whenever we like.

In this chapter, we will learn about Matryoshka embeddings, vectors that still work after we
cut them short. We will see how they are trained, and watch a cut-down vector lose Acme's
answer with a normal model but keep it with an MRL model. We will also see what this buys us
in memory and speed, the catches, and a staged design that cuts the RAM for 100 million vectors
from 307 GB to about 25 GB.

We will cover the following:

- What is truncation
- What is a Matryoshka embedding
- How Matryoshka training works
- An example: cutting Acme's vectors to 64 numbers
- What Matryoshka embeddings buy us
- Problems with Matryoshka embeddings
- Matryoshka vs a normal embedding
- When to use which one

---

## What is truncation

**Truncation = keeping only the first m numbers of a vector and throwing the rest away.**

If a vector has 768 numbers, truncating it to 64 means keeping positions 1 to 64. In Python,
that is `v[:64]`.

Why would we want that? Because every number costs memory. Acme's 40,000 pages, stored as
768-dimensional `float32` vectors, take `40,000 × 768 × 4 = 122.9 MB`. Cut to 64 numbers,
they take 10.2 MB, which is 12× less. Every comparison also does 12× less arithmetic. At 100
million vectors, the same cut turns 307 GB into 25.6 GB.

Now, the question is, does a cut-down vector still work?

For a normal embedding, no. Think of it like a jigsaw puzzle:

- The full vector is the finished puzzle.
- Each dimension is one piece.
- Truncating is throwing away pieces.
- Throw away most of the pieces and we do not get a smaller picture. We get a broken one.

In simple words, a normal model has no reason to put the important information first. The
details that separate one page from another are spread across all 768 numbers, and the first
64 hold only a scattered fraction of them. Truncating a standard 768-dimensional vector to 128
dimensions typically causes a large quality drop.

---

## What is a Matryoshka embedding

A matryoshka is a Russian nesting doll. Open it, and there is a smaller, complete doll
inside. Open that one, and there is another. Every layer is a whole doll, not a fragment.

**Matryoshka embedding = a vector trained so that its first 64 numbers, its first 256, its
first 512 and all 768 each work as a complete embedding.**

Here is the mapping:

- The biggest doll is the full 768-dimensional vector.
- Each smaller doll is the vector cut short: the first 512 numbers, 256, 128, 64.
- Every doll is whole. The 64-number version is coarse but coherent. The 256-number version
  is better. All 768 are best.

**MRL = Matryoshka Representation Learning**, the training method that produces these
vectors.

In simple words, MRL packs meaning front to back. **Every prefix is a whole doll.**

**Note:** In Chapter 14, a *prefix* was a piece of text we put in front of the input. In this
chapter, a *prefix of a vector* just means its first m numbers. Same word, different job.

---

## How Matryoshka training works

The trick is almost embarrassingly simple. We apply the contrastive loss from Chapter 12 at
several lengths at once, and add the results.

**Step 1:** Take a training batch of (question, page) pairs, exactly as in Chapter 12.

**Step 2:** Embed every text into a full 768-dimensional vector.

**Step 3:** Compute the contrastive loss using only the first 64 numbers of every vector.

**Step 4:** Compute it again using the first 128, then the first 256, the first 512, and all
768.

**Step 5:** Add the five losses together, and nudge the weights to make the total smaller.

As a formula:

$$ \mathcal{L}_{\text{MRL}} = \sum_{m \in \{64,128,256,512,768\}} w_m \cdot \mathcal{L}_{\text{contrastive}}\big(v_{[:m]}\big) $$

In simple words, the total loss is five ordinary contrastive losses added up. Here
$v_{[:m]}$ means "the first m numbers of each vector", and $w_m$ says how much we care about
length m. Often every $w_m$ is simply equal.

So during each training step, the model is scored on the full vector *and* on the first 512
numbers *and* on the first 256, and so on. The gradients (Chapter 12) flow from all of them.

Why does this put the important information first? Because the first 64 numbers must perform
well on their own, the training is forced to put the most broadly useful information there.
Later numbers can only earn their keep by adding what the early ones missed. The result is an
embedding sorted roughly by importance.

That is conceptually similar to what PCA gives us (Chapter 6: a classical method that
re-orders dimensions by how much they vary). The difference is that MRL learns the order end
to end, during training, with no separate projection step when we embed.

```python
def mrl_loss(Q, D, dims=(64, 128, 256, 512, 768)):
    total = 0.0
    for m in dims:
        total += infonce(Q[:, :m], D[:, :m])     # from Chapter 12 (it re-normalizes each prefix)
    return total / len(dims)
```

Means, the loop cuts every question vector and page vector to length m, scores the batch as
usual, and averages the five scores. That average is the formula above with every $w_m$ equal
to 1/5.

It costs a little extra compute during training. It costs nothing at inference (when we run the
trained model to embed a text). This is one of the highest-leverage ideas in the recent embedding
literature precisely because it is so cheap.

---

## An example: cutting Acme's vectors to 64 numbers

Let's use the pipeline from Chapter 13. The bi-encoder fetches the top 100 pages, then a
cross-encoder reranks them. Our question, as always:

> *"Does the Pro plan include single sign-on?"*

With full 768-number vectors, page 212 comes back at rank 1 and page 1,140 at rank 23. Both
are in the top 100, so the reranker lifts them to ranks 1 and 2, and the Scholar answers
correctly.

Now Acme wants to cut memory 12× by keeping only 64 numbers per page.

Let's first see what happens with a normal model. We truncate every page vector and the
question vector to 64 numbers, re-normalize them, and search. Here is what the first stage
returns (toy ranks, to show the shape of the failure):

| Page | What the page is about | Rank at 768 numbers | Rank at 64 numbers |
|---|---|---|---|
| 45 | Pro plan overview | 4 | 1 |
| 19,884 | Resetting your password | 310 | 2 |
| 612 | Two-factor login | 95 | 3 |
| 212 | Pro plan page: SSO included on Pro | 1 | 340 |
| 1,140 | The Security add-on | 23 | 1,900 |

The first 64 numbers of this model kept only a vague "plans and logging in" signal. The
detail that makes page 212 about *single sign-on* lived somewhere in the other 704 numbers,
and we threw it away.

Neither page 212 nor page 1,140 is in the top 100, so the reranker never sees them. No
reranker can rescue a page it was never given. The Scholar reads pages about the Pro plan,
passwords and two-factor login, and thinks like this: "None of these pages mentions single
sign-on on the Pro plan." It answers:

> *"The Pro plan documentation does not mention single sign-on, so it is probably not
> included."*

The answer is wrong.

Now, let's see what an MRL model does, in two stages.

**Stage 1:** Search all 40,000 pages using only the first 64 numbers, and keep the top 1,000.
Page 212 lands at rank 4. Page 1,140 lands at rank 160 (toy ranks again). Both survive.

**Stage 2:** Rescore those 1,000 candidates using all 768 numbers. Page 212 moves to rank 1,
and page 1,140 to rank 23, exactly where the full search put them.

The top 100 now matches the full 768-number search. The reranker lifts pages 212 and 1,140 to
the top, and the Scholar answers:

> *"Yes, SSO is included on Pro, but teams under 50 seats also need the Security add-on."*

The answer is correct. The expensive 768-number comparison ran on 1,000 pages instead of
40,000.

---

## What Matryoshka embeddings buy us

**1. Adaptive retrieval, the big one.** Two-stage search with a single model and a single
stored vector:

```
Stage 1: search every page using only the first 64 dimensions
         → 12× less memory touched, roughly 10× faster
         → retrieve top 1,000 candidates

Stage 2: rescore those 1,000 using the full 768 dimensions
         → near-full-precision quality

Result: most of the speed of a 64-d index, most of the accuracy of a 768-d one
```

This is the coarse-then-refine pattern from Chapter 6, now available without a second model,
without a second index, and without a second forward pass. We store the full vector once, and
read a prefix of it when we want speed. At Acme's 40,000 pages this barely matters. At tens of
millions of pages, a first pass that touches 12× less memory is a large saving on every query.

**2. Storage tiering.** Keep this year's pages at 768 dimensions, and the old release-note
archive at 128. Each tier is searched with the question cut to its length. Same model, same
pipeline.

**3. Cost control per request.** A free-tier customer gets 256-dimension search. An
enterprise customer gets 768. One model, one stored vector per page.

**4. Honest dimension shopping.** We can *measure* the quality-versus-dimension curve on our
own data instead of guessing. Published results for several MRL models show that keeping 256
of 768 dimensions retains about 98–99% of quality on general benchmarks. That is a 3× memory
saving for a rounding error. Our corpus may differ, which is exactly why measuring beats
guessing.

---

## Problems with Matryoshka embeddings

**Problem 1: Truncation requires re-normalization.** A prefix of a unit vector (Chapter 5) is
not a unit vector. Page 212's first 64 numbers might have length 0.41 while page 45's have
length 0.55. Forget to fix this, and our dot-product scores (which we were treating as
cosines) are scaled by an arbitrary per-page constant. Page 45 gets a boost it did not earn,
and the ranking is corrupted.

```python
v64 = v[:64]
v64 = v64 / np.linalg.norm(v64)      # required, not optional
```

Means, we divide the cut-down vector by its own length, so it is a unit vector again.

**Problem 2: It only works on MRL-trained models.** Truncating a normal model breaks it, as
the example showed. Check the model card. OpenAI's `text-embedding-3` family, Nomic Embed
v1.5, mxbai-embed-large, Jina v3 and a growing set of others are MRL-trained.

**Problem 3: The curve depends on the data.** The quality cliff may be at 128 dimensions for
our corpus and at 256 for someone else's. Measure it. The code below takes ten minutes.

**Problem 4: It does not combine freely with every kind of quantization.** Quantization means
storing each number in fewer bits (Chapter 26). Truncating *and* quantizing stacks two lossy
steps. Sometimes that combination is superb (see Ninja notes), and sometimes it collapses.
Always validate the combination, not the parts.

---

## Matryoshka vs a normal embedding

| | Normal embedding | Matryoshka (MRL) embedding |
|---|---|---|
| Where meaning lives | Spread across all dimensions | Packed front to back |
| Truncated to 64 numbers | Broken | Coarse, but working |
| Full-length quality | Baseline | About the same, occasionally a hair lower |
| Training cost | Baseline | A little more |
| Cost when embedding and searching | Baseline | The same, or less if we truncate |
| Sizes available | One | Any prefix length it was trained for |

**Advantages of MRL.**

- One stored vector serves many sizes, so memory becomes a dial instead of a commitment.
- Two-stage search needs no second model and no second index.

**Disadvantages of MRL.**

- It buys flexibility, not extra accuracy.
- We still have to measure the quality cliff on our own data.
- It only helps if the model we like best for our data was trained with it.

---

## When to use which one

We must prefer an **MRL model** when memory or latency matters, when we want the option to
shrink later, or when we want cheap two-stage search from one stored vector.

A **normal model** is fine when the corpus is small and the best model for our data happens
not to be MRL-trained. Acme's 40,000 pages fit in 123 MB at full size, so for Acme today,
quality on its own questions matters more (Chapter 16).

Many strong systems combine the ideas: MRL truncation for the first pass, full vectors for
rescoring, and quantization on top.

---

### Under the hood

Find your own cliff:

```python
import numpy as np

def eval_at_dims(V, queries, gold, dims=(64, 128, 256, 384, 512, 768), k=10):
    """V: page vectors. queries: question vectors. gold[i]: index of the right page for query i."""
    for m in dims:
        Vm = V[:, :m]; Vm = Vm / np.linalg.norm(Vm, axis=1, keepdims=True)
        Qm = queries[:, :m]; Qm = Qm / np.linalg.norm(Qm, axis=1, keepdims=True)
        top = np.argsort(-(Qm @ Vm.T), axis=1)[:, :k]
        recall = np.mean([g in row for row, g in zip(top, gold)])
        print(f"dim={m:4d}  recall@{k}={recall:.3f}  mem={m*4:5d} B/vec")
```

In simple words, for each length we cut both sides, re-normalize both sides, search, and
count how often the right page lands in the top k. Recall@k gets a full definition in
Chapter 19.

Run this against a real eval set of Acme questions (Chapter 19) and we get a table that turns
directly into an infrastructure decision, often "we can cut our vector RAM by 3× tonight."

---

### What people get wrong

**Truncating a non-MRL model.** The single most common error. Quality falls off a cliff and
the cause is not obvious.

**Forgetting to re-normalize.** Silent ranking corruption.

**Truncating queries and documents to different lengths.** They must match. Comparing a 256-d
query with a 768-d document is comparing points in different spaces.

**Assuming MRL is free quality.** It is free *flexibility*. The full-dimension quality of an
MRL model is usually on par with a non-MRL equivalent, occasionally a hair below. You are
buying the option to shrink, not extra accuracy.

---

### Ninja notes

One of the strongest known combinations for large-scale, memory-constrained retrieval stacks
MRL with binary quantization (Chapter 26):

```
Stage 1: binary vectors, 768 dims → 96 bytes/vector, Hamming distance
         → 32× compression, SIMD-fast popcount, retrieve top 1,000
Stage 2: int8 vectors truncated to 256 dims → 256 bytes/vector, rescore top 1,000
Stage 3: (optional) full float32 rescore of top 100
```

Published write-ups of this pattern report roughly 30× less memory for the vectors held in RAM,
while retaining 95–99% of full-precision retrieval quality. Put 100 million vectors through it. As float32 they need
307 GB of RAM. The binary codes need 9.6 GB (96 B × 100M), 32× less. The int8 rescoring
vectors (25.6 GB) and the float32 originals (307 GB) can live on SSD, because each query reads
only ~1,000 of them. Add the index structure over the binary codes and Chapter 60 budgets
about 25 GB of RAM. That is the difference between a very large memory server and a modest
one, and Chapter 60 turns it into money.

The general principle is worth stating plainly, because it is the master pattern of Part IV:
**compression is not a single decision but a cascade.** Each stage narrows the candidate set,
so each later stage can afford more precision per item. Aggressive at the top, exact at the
bottom.

---

### Key takeaways

- **Matryoshka embedding = a vector trained so that every prefix of it is itself a working
  embedding.**
- **Truncation = keeping the first m numbers.** Only do it on MRL-trained models, always
  re-normalize, and truncate both sides to the same length.
- MRL is trained by applying the contrastive loss at several lengths at once and adding the
  results.
- It enables adaptive retrieval (coarse search on a prefix, rescore on the full vector) with
  one model and one stored vector.
- MRL buys flexibility, not extra accuracy. Measure your own quality cliff.
- MRL + binary quantization + rescoring is among the strongest memory-quality combinations
  known: about 32× less RAM for the codes, with most of the quality kept.

### What's next

[Chapter 16](./16-choosing-a-model.md) closes Part II with a practical question. Given
hundreds of embedding models, how do we pick one? And how do we read MTEB, the public
leaderboard for embedding models, without being misled by it?

Matryoshka training packs meaning front to back, so a cut-down vector stays whole, and memory
becomes a dial we can turn.
