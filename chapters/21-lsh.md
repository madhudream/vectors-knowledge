---
title: "LSH: Hashing Things That Are Alike"
chapter: 21
part: "Part IV — Index Structures"
slug: lsh
readingTime: "12 min"
summary: "A hash function designed to collide. Rarely the right choice for dense retrieval today, but the ideas inside it reappear in binary quantization and in MUVERA."
tags: [lsh, hashing, simhash, minhash, random-projection]
prev: 20-ann-family-tree
next: 22-trees-and-annoy
---

# LSH: Hashing Things That Are Alike

**The one-paragraph version.** A normal hash function scatters similar inputs to wildly
different outputs. **Locality-Sensitive Hashing** does the opposite on purpose: similar vectors
land in the same bucket with high probability. We look up the query's bucket and scan only what
is inside. For dense retrieval, graph indexes have largely replaced LSH. It remains the right tool
for near-duplicate detection at scale, and its core trick is the direct ancestor of binary
quantization and of MUVERA's encodings.

In this chapter, we will learn about LSH, the first of the four ANN families from Chapter 20. We
will see how a few random slices through the Map Room become a bucket address, and how extra
tables rescue a missed page. We will also see how MinHash finds near-duplicate pages, and why LSH
lost the race for dense retrieval anyway.

We will cover the following:

- What is a hash function
- What is LSH
- How SimHash works
- An example of SimHash on our SSO question
- Tuning LSH with AND and OR
- MinHash: LSH for sets
- Why LSH lost for dense retrieval
- When to use LSH

---

## What is a hash function

Before jumping into LSH, we must know what an ordinary hash function is.

**Hash function = a recipe that turns any input into a short, fixed-size code.**

A **hash table** uses that code as a shelf number, so finding an item is instant. SHA-256 is a
famous hash function. Change one character of a sentence and SHA-256 gives a completely unrelated
code.

That is exactly what we want for a hash table. It is exactly what we do *not* want for similarity
search.

Take two sentences from Acme's Library: "SSO is included on the Pro plan" and "SSO is included in
the Pro plan". They mean the same thing. SHA-256 sends them to unrelated shelves, and nothing
about the two codes says the sentences are alike.

---

## What is LSH

**LSH = Locality-Sensitive Hashing = a hash function built so that similar things collide.**

Let's break it down:

- **Locality** means nearness: things that sit close together in the Map Room.
- **Sensitive** means the hash notices that nearness and keeps it.
- **Hashing** means turning each vector into a short code, which names its **bucket**.

Written as a rule:

> $$P[h(a) = h(b)] \text{ is high when } a \approx b, \text{ and low when } a \not\approx b$$

In simple words, the closer two vectors are, the more likely they are to get the same code.

If similar items collide, then collision becomes a filter. We only ever compare the query with
things that already landed in its bucket.

Think of it like a cloakroom that hangs similar coats on the same hook. The coats are the page
vectors. The hook number is the hash code. The coats on one hook are a bucket. Checking one hook
instead of the whole cloakroom is scanning one bucket instead of the whole Library.

---

## How SimHash works

For cosine similarity (Chapter 4), the simplest LSH is beautiful.

One word first. A **hyperplane** is a flat sheet that passes through the centre of the Map Room
and splits it into two sides. In 2 dimensions it is a line. In 3 dimensions it is a flat plane.
In 768 dimensions we cannot picture it, but it still has exactly two sides.

**Phase 1: Building the table.**

**Step 1:** Pick a random direction `r`. The hyperplane is every point at a right angle to `r`.
**Step 2:** For every page vector `v`, ask one question: which side of the sheet is it on?
**Step 3:** Repeat with 16 different random directions. Each page now has 16 bits, its signature.
**Step 4:** Store each page's ID in the bucket named by its signature.

**Phase 2: Answering a query.**

**Step 1:** Compute the query's 16 bits, using the same 16 random directions.
**Step 2:** Look up the bucket with that signature.
**Step 3:** Score only the pages in that bucket with real cosine similarity, and keep the best.

The side test in Phase 1, Step 2 is one line of maths:

$$ h_r(v) = \begin{cases} 1 & \text{if } r \cdot v \ge 0 \\ 0 & \text{otherwise} \end{cases} $$

Means, the sign of the dot product tells us the side. Positive or zero gives 1. Negative gives 0.

Here is the elegant part. Two vectors land on the same side of a random sheet *unless the sheet
happens to pass between them*. The chance of that grows with the angle between them:

$$ P[h_r(a) = h_r(b)] = 1 - \frac{\theta(a,b)}{\pi} $$

In simple words, θ is the angle between the two vectors (Chapter 4), and π is 180° written in
radians. Vectors 10° apart agree on any one bit 1 − 10/180 = 94% of the time. Vectors 90° apart
agree only 50% of the time, a coin flip. So the number of matching bits between two signatures is
a direct estimate of the angle between them.

**SimHash = the signature made of the signs of random projections.** Each bit records which side
of one random sheet a vector falls on.

**Note:** Taking the sign of a random projection is essentially binary quantization with random
axes. Chapter 26 uses the model's own axes instead. The difference is one of framing. LSH treats
the bits as a bucket address to look up. Binary quantization treats them as a compressed vector,
compared by **Hamming distance** (the count of bits that differ, Chapter 26). Same bits, two uses.

---

## An example of SimHash on our SSO question

Our running question is *"Does the Pro plan include single sign-on?"* The answer needs page 212
(SSO is included on Pro) and page 1,140 (Pro teams under 50 seats need the Security add-on).

Let's use toy angles. Both pages sit 10° from the question. The invoices page sits 60° away.

We start with one table of 16 bits. Page 212 happens to match the question on all 16
bits, so it lands in the question's bucket. Page 1,140 matches on 15 bits. On bit 11, one random
sheet happened to slice between it and the question. So page 1,140 lands in a neighbouring
bucket, and we never open that bucket.

This is not bad luck we can ignore. At 10°, the chance of matching all 16 bits is 0.944¹⁶ ≈ 0.40.
Most of the time, a page this close lands somewhere else.

The Librarian returns page 212 alone. The Scholar thinks: "SSO is included on Pro. Nothing
mentions a condition." It answers, "Yes, the Pro plan includes SSO."

That answer misleads every Pro team under 50 seats.

So we build 8 tables instead of 1, each with its own 16 random sheets. We check the
question's bucket in all 8 and take the union. Page 1,140 only needs to collide in *one* table.
The chance of that is 1 − (1 − 0.40)⁸ ≈ 0.98.

The Librarian now returns both pages. The Scholar answers, "Yes, but teams under 50 seats need the
Security add-on." Now the answer is complete.

And the invoices page, 60° away? Its chance of sneaking into the union of 8 tables is about 1%.
Extra tables rescue near pages without flooding us with far ones.

---

## Tuning LSH with AND and OR

One 16-bit signature is a crude filter, as page 1,140 just showed. A true neighbour that differs
on a single bit lands elsewhere, and no amount of rescoring can recover a page we never saw.

LSH fixes this with two composition rules. (Here `k` counts bits per table, as in the LSH
literature. It is not the top-k of Chapter 3.)

**AND (more bits per table).** Require all $k$ bits to match. Collisions become rarer, so buckets
get smaller and precision rises. But recall falls.

**OR (more tables).** Build $L$ independent hash tables, each with its own random sheets. Check
all of them and union the results. A true neighbour only needs to collide in *one* table, so
recall rises. The cost is $L$ times the memory and $L$ lookups.

Together, the probability of retrieving a neighbour whose per-bit agreement is $p$ is:

$$ P_{\text{found}} = 1 - (1 - p^k)^L $$

In simple words, $p^k$ is the chance of matching one whole table, and the rest is the chance of
matching at least one of $L$ tables.

Here is that formula for $k = 16$ bits:

| Angle to query | $p$ per bit | 1 table | 8 tables |
|---|---|---|---|
| 10° | 0.94 | 0.40 | 0.98 |
| 30° | 0.83 | 0.05 | 0.36 |
| 60° | 0.67 | 0.002 | 0.01 |
| 90° | 0.50 | < 0.001 | < 0.001 |

This produces an S-curve. Near pages are almost always found, far pages almost never. We tune
$k$ to place the curve's steep section at the similarity we care about, and $L$ for how sharply
it rises. This is the recall knob of Chapter 18 in its earliest form. Unusually, it comes with a
formula we can solve on paper, rather than a setting we must find by trial.

---

## MinHash: LSH for sets

A different flavour, for a different kind of similarity.

Acme's Library holds many near-copies. The Pro pricing page was republished each year with one
sentence changed. When we ask our SSO question, three copies of the same pricing text can fill the
top 5, crowding out page 1,140. We want to find those near-duplicates and keep one.

For this job we compare pages as *sets* of word runs, not as vectors.

**Shingles = overlapping runs of, say, five consecutive words.**

For example, "SSO is included on the Pro plan" has three 5-word shingles: "SSO is included on
the", "is included on the Pro" and "included on the Pro plan".

**Jaccard similarity = the number of shingles two pages share ÷ the number of distinct shingles
across both.**

$$ J(A,B) = \frac{|A \cap B|}{|A \cup B|} $$

Means, if two pricing pages share 90 shingles and have 110 distinct shingles between them,
J = 90 / 110 = 0.82.

**MinHash = an LSH that estimates Jaccard similarity from short signatures.** Here is how it
works.

**Step 1:** Turn each page into its set of shingles.
**Step 2:** Shuffle the list of all possible shingles into a random order.
**Step 3:** For each page, record which of its shingles comes first in that order. That is its
minimum.
**Step 4:** Repeat with 128 different shuffles. Each page now has 128 minimums, its signature.
**Step 5:** Estimate J as the fraction of the 128 positions where two signatures agree.

Why does this work? Look at all the shingles in either page. After a shuffle, one of them comes
first, and each is equally likely to. The two pages record the same minimum exactly when that
first shingle belongs to *both* pages. The chance of that is shared ÷ distinct, which is J.

With 128 shuffles, the estimate's standard error (Chapter 19) is at most about 4 points. In
practice, a hash function stands in for each shuffle.

MinHash over word shingles remains the standard method for **near-duplicate detection at web
scale**. It matters to this book directly. Deduplicating the corpus before embedding is one of the
highest-value preprocessing steps in RAG. Duplicate chunks fill the top-k with the same content
five times, crowding out the variety the Scholar needs. MinHash-LSH will find them across a hundred
million documents on one machine.

---

## Why LSH lost for dense retrieval

Benchmarks are consistent: HNSW reaches higher recall at lower latency with less memory. The
reasons are structural, not accidental. There are three.

**First, LSH is data-oblivious.** The sheets are random. They know nothing about where our data
actually is. Embeddings tend to crowd into a narrow cone of directions, as Chapter 3 showed. Most
random sheets then pass nowhere near the crowd, and their bits are wasted. HNSW's links, by
contrast, are built *from* the data.

**Second, its guarantees are asymptotic.** LSH's theory is beautiful, but it describes behaviour
as the collection grows toward infinity, under specific assumptions about distances. At ten
million real vectors, the constant factors dominate and the theory does not predict what we
observe.

**Third, memory adds up.** $L$ tables mean $L$ copies of every ID, and decent recall often needs
8–32 tables.

---

## When to use LSH

| | LSH | Graph index (HNSW) |
|---|---|---|
| Built from the data? | No, random | Yes |
| Recall on dense embeddings | Lower | Higher |
| Build step | None, add items as they stream in | Graph construction |
| Theory | Closed-form guarantees | Mostly empirical |
| Set similarity (Jaccard) | Yes, via MinHash | No |
| Memory | One ID copy per table | Links per vector, vectors in RAM |

**Advantages of LSH.** No training and no data-dependent build. Items can stream in forever. The
recall curve comes with a formula. MinHash handles set similarity, which graphs do not.

**Disadvantages of LSH.** Wasted bits on real, clumpy embeddings. Many tables for decent recall.
Always needs a rescoring step.

We must use **LSH** when we need (a) provable guarantees, (b) streaming with no index build, (c)
set similarity rather than vector similarity, or (d) the duplicate-detection job above.

We must use **a graph index** when we want the best recall and latency for dense embeddings in
memory, which is most semantic search.

Many strong pipelines use both: MinHash-LSH to deduplicate the Library once, and HNSW to search
what remains.

---

### Under the hood

The SimHash index from our example, with 16 bits and 8 tables:

```python
import numpy as np

class SimHashLSH:
    def __init__(self, dim, n_bits=16, n_tables=8, seed=0):
        rng = np.random.default_rng(seed)
        self.planes = rng.normal(size=(n_tables, n_bits, dim)).astype(np.float32)
        self.tables = [{} for _ in range(n_tables)]
        self.pow2 = (1 << np.arange(n_bits))

    def _sig(self, v):                                  # (n_tables,) int keys
        bits = (self.planes @ v) >= 0                   # (n_tables, n_bits)
        return bits @ self.pow2

    def add(self, v, doc_id):
        for t, key in enumerate(self._sig(v)):
            self.tables[t].setdefault(int(key), []).append(doc_id)

    def query(self, v):
        out = set()
        for t, key in enumerate(self._sig(v)):
            out.update(self.tables[t].get(int(key), ()))
        return out                                      # then rescore exactly
```

`_sig` does Phase 1, Steps 1–3, for every table at once. `bits @ self.pow2` packs the 16 yes/no
answers into one integer, the bucket name. In simple words, the random sheets decide the bits,
and the bits decide the shelf.

That last comment is the pattern again. LSH produces *candidates*, never final rankings. We always
rescore with the real metric. Retrieve coarsely, refine precisely.

---

### What people get wrong

**"LSH is obsolete."** For dense ANN, mostly. For near-duplicate detection over billions of
documents, MinHash-LSH is still the standard and still excellent.

**Choosing $k$ and $L$ by intuition.** They interact through that S-curve. Decide what similarity
threshold you care about, then solve for parameters that place the curve there. It is one of the
few tunings you can do analytically instead of by sweeping.

**Using LSH without rescoring.** The hash is an approximation of an approximation. Always compute
real similarities on the candidate set.

**Confusing SimHash and MinHash.** Cosine on vectors versus Jaccard on sets. Different problems,
different guarantees.

---

### Ninja notes

The reason LSH earns a chapter in a 2026 book is that its central idea, *a random projection
followed by a sign*, has proved far more durable than LSH itself.

It reappears as **binary quantization** (Chapter 26), where the bits become a 32×-compressed
vector compared by `popcount`, which modern CPUs execute in a single instruction.

It reappears in **MUVERA** (Chapter 38), which uses SimHash to partition space into buckets. It
then aggregates a document's token vectors within each bucket, building a single fixed-dimensional
encoding whose dot product approximates multi-vector similarity. The partitioning step *is* LSH.
MUVERA's contribution is what it does inside the buckets.

And the Johnson–Lindenstrauss lemma underneath it all (Chapter 6) is the licence for every
dimensionality reduction in the field.

So LSH's descendants are everywhere, even where the name is not. Learning it is learning a
primitive, not an algorithm.

---

### Key takeaways

- **LSH = Locality-Sensitive Hashing = a hash function built so that similar things collide.**
- **SimHash** = the signs of random projections. Matching bits estimate the angle, since
  P(same bit) = 1 − θ/π.
- AND (more bits) raises precision. OR (more tables) raises recall. Together they form a tunable
  S-curve with closed-form theory: one table found page 1,140 40% of the time, eight tables 98%.
- **MinHash** estimates Jaccard similarity over **shingles** and is still the standard for
  near-duplicate detection, which you should run before embedding.
- Graph indexes beat LSH for dense retrieval because they are built from the data.
- The random-projection-plus-sign primitive survives in binary quantization and MUVERA.

### What's next

[Chapter 22](./22-trees-and-annoy.md) takes on the second family, carving space into regions, and
shows exactly why it stops working somewhere around twenty dimensions.

We now know how LSH turns random slices into bucket addresses, how to tune it, where MinHash still
wins, and why graphs overtook it for dense search.
