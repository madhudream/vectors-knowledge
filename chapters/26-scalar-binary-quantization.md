---
title: "Scalar and Binary Quantization"
chapter: 26
part: "Part IV — Index Structures"
slug: scalar-binary-quantization
readingTime: "12 min"
summary: "Two compression schemes with no codebooks and no training. int8 gives 4× for almost nothing. One bit per dimension gives 32× and, with rescoring, loses surprisingly little."
tags: [quantization, int8, binary, hamming, rescoring, memory]
prev: 25-ivfpq-and-opq
next: 27-turboquant
---

# Scalar and Binary Quantization

**The one-paragraph version.** **Scalar quantization** stores each number of a vector as a
small whole number, usually one byte (**int8**) instead of four. That is 4× smaller, nearly
lossless, one line of code, and needs no training. **Binary quantization** keeps only whether
each number is positive or not, one bit per number. That is 32× smaller, and vectors are
compared by counting differing bits instead of multiplying. On its own, binary gets the order
of results wrong quite often. Paired with a rescoring pass, it works remarkably well. Unlike PQ,
neither scheme has codebooks to train, store or keep fresh.

In this chapter, we will learn about scalar quantization and binary quantization, the two
simplest ways to shrink vectors. We will also see how bits are compared with XOR and popcount,
why binary alone gives the wrong answer, how rescoring fixes it, and how to choose between
int8, binary and PQ.

We will cover the following:

- What is scalar quantization
- How scalar quantization works
- What is binary quantization
- How Hamming distance works
- Why binary quantization needs rescoring
- Choosing a compression scheme

---

## What is scalar quantization

Before we start, one reminder. A **bit** is a single 0 or 1. Eight bits make a **byte**. A
`float32` number (Chapter 1) takes 4 bytes, which is 32 bits.

**Scalar quantization = store each number of the vector on a coarser scale, independently of
the others.**

"Scalar" means we treat one number at a time. There are no pieces and no codebooks, unlike PQ
in Chapter 24.

**int8 = an 8-bit whole number, from −128 to 127.** The most common scalar quantization maps
every `float32` in the vector to an int8.

Think of it like writing down expenses.

- The exact amount, £12.4783, is the `float32`.
- Writing £12 instead is scalar quantization.
- Writing only "expensive" or "cheap" is binary quantization, which we will meet shortly.

We lose the pennies and keep everything that matters for deciding whether lunch was expensive.

In simple words, scalar quantization is rounding, done carefully.

---

## How scalar quantization works

Let's go back to Acme at scale: 100 million chunks, 307 GB as `float32`. int8 makes that
76.8 GB.

**Phase 1: Choosing the range.**

**Step 1:** Look at a sample of the vectors and find the smallest and largest numbers. These
are `min` and `max`.

**Phase 2: Encoding.**

**Step 1:** For each number, measure how far along the range it sits, from 0 (at `min`) to 1
(at `max`).

**Step 2:** Stretch that onto 256 steps by multiplying by 255, and round.

**Step 3:** Subtract 128, so the result fits in a signed byte from −128 to 127.

**Step 4:** Clip anything below −128 up to −128, and anything above 127 down to 127. A number
outside the sample's range would otherwise wrap around to the wrong end and flip its sign.

$$ q_i = \text{round}\!\left(\frac{v_i - \min}{\max - \min} \times 255\right) - 128 $$

Let's try it on page 212 of the Library, the page that says SSO is included on the Pro plan.
Its toy vector starts `[0.5, -0.3, 0.3, 0.2, ...]`. Say the range is `min = -1`, `max = 1`.

```
 0.5:  (0.5 + 1) / 2 × 255 = 191.25 → 191 → 191 − 128 =  63
-0.3:  (-0.3 + 1) / 2 × 255 = 89.25 →  89 →  89 − 128 = -39
 0.3:  (0.3 + 1) / 2 × 255 = 165.75 → 166 → 166 − 128 =  38
 0.2:  (0.2 + 1) / 2 × 255 = 153.00 → 153 → 153 − 128 =  25
```

To read the numbers back, we run the same steps backwards. 63 becomes 0.498, −39 becomes
−0.302, 38 becomes 0.302, and 25 becomes 0.200. Every error is 0.002 or less.

In simple words, the numbers barely move, and each one now takes 1 byte instead of 4.

**768 numbers × 1 byte = 768 bytes**, down from 3,072. The typical recall loss is under 1%,
and often too small to measure.

```python
def train_sq(V):
    return V.min(), V.max()                       # global, or per-dimension

def encode_sq(V, lo, hi):                         # Steps 1-4, clipped so nothing wraps
    return np.clip(np.round((V - lo) / (hi - lo) * 255 - 128), -128, 127).astype(np.int8)
```

There are two variants worth telling apart.

- **Global:** one `(min, max)` for the whole corpus. This is simplest. It works well when
  vectors are normalized (as they should be, Chapter 5), because all numbers then lie in a
  narrow, predictable band.
- **Per-dimension:** separate bounds for each of the 768 positions. This is better when
  positions have genuinely different scales. It costs `2 × d` floats of extra storage.

int8 is often *faster* than `float32`, not just smaller. Many modern CPUs have instructions made
for int8 dot products (VNNI on x86 chips, the dot-product extensions on ARM chips). This is
close to a free lunch. If you run HNSW and memory is even slightly tight, turn on int8 first and
measure before trying anything more exotic.

**Note:** `float16` deserves a mention too. It halves memory with essentially no quality loss,
and it is supported almost everywhere. If int8 feels like a step too far, `float16` is the
no-thought option.

---

## What is binary quantization

Now let's push the rounding as far as it can go.

**Binary quantization = keep one bit per number: 1 if the number is positive, 0 if it is not.**

$$ b_i = \begin{cases} 1 & v_i > 0 \\ 0 & v_i \le 0 \end{cases} $$

In simple words, we throw away every size and keep only the direction, "above zero" or "not".

**768 numbers → 768 bits = 96 bytes.** That is 32× smaller than `float32`. Acme's 100 million
chunks drop from 307 GB to 9.6 GB.

Going back to the expenses: we no longer write the amount at all, only "expensive" or "cheap".
We have lost almost everything. And yet a list of 768 such marks still tells a week of fine
dining apart from a week of sandwiches, because the *pattern* survives when the amounts do not.
That surviving pattern is the whole reason binary quantization works.

Here is another way to see it, from Chapter 21. The sign of a number tells us which side of a
dividing line the vector sits on. Agreeing on many such sides means two vectors point in
similar directions. Binary quantization is SimHash that uses the model's own dimensions instead
of random ones. On well-trained models, the model's own axes usually work about as well as random
ones. When some dimensions have far more spread than others, a random rotation first helps
(Chapter 27).

---

## How Hamming distance works

So how do we compare two lists of bits?

**Hamming distance = the number of positions where two bit lists differ.**

The computer finds it with two operations.

- **XOR** ("exclusive or") compares two bits and gives 1 if they differ, 0 if they match.
- **popcount** ("population count") counts how many 1s are in a group of bits.

So **Hamming distance = popcount(a XOR b)**.

Let's see it with 8 numbers instead of 768. Here are toy vectors for the query and three
pages. They are all close to unit length.

| Vector | Numbers | Bits |
|---|---|---|
| Query: *"Does the Pro plan include single sign-on?"* | [0.5, -0.3, 0.4, 0.1, -0.6, 0.2, -0.1, 0.3] | 10110101 |
| Page 212: SSO included on Pro | [0.5, -0.3, 0.3, 0.2, -0.6, 0.2, -0.1, 0.3] | 10110101 |
| Page 1,140: SSO needs the add-on | [0.6, -0.4, 0.5, -0.1, -0.5, -0.1, 0.1, 0.2] | 10100011 |
| Page 87: two-factor login on Basic | [0.1, -0.1, 0.1, 0.7, -0.1, 0.6, -0.3, -0.1] | 10110100 |

**Step 1:** Turn each vector into bits. A positive number becomes 1. Anything else becomes 0.

**Step 2:** XOR the query's bits with each page's bits.

```
query      10110101        query      10110101        query      10110101
page 212   10110101        page 1,140 10100011        page 87    10110100
XOR        00000000        XOR        00010110        XOR        00000001
```

**Step 3:** Popcount each result.

```
page 212:   popcount(00000000) = 0
page 1,140: popcount(00010110) = 3
page 87:    popcount(00000001) = 1
```

**Step 4:** Rank by Hamming distance, smallest first: page 212 (0), page 87 (1), page 1,140 (3).

That is it. No multiplication anywhere. A CPU does XOR and popcount on 64 bits in a single
instruction each. Comparing two 768-bit vectors is 12 XORs and 12 popcounts, which is many
times cheaper than 768 floating-point multiply-adds.

---

## Why binary quantization needs rescoring

Look again at that ranking. Page 87 came second, ahead of page 1,140.

Let's see what happens if we trust binary on its own. At full scale, binary's standalone recall@10
is often only around 0.70–0.85, and our toy shows why. The Librarian asks for the top 2 pages and
gets page 212 and page 87. Page 87 is about two-factor login on the Basic plan. The Scholar reads
"SSO is included on Pro" plus an unrelated page, and answers, "Yes, SSO is included on Pro."

The answer is wrong. The add-on rule on page 1,140 never arrived.

Why did binary get it wrong? Page 1,140 disagrees with the query on three signs, but on
numbers that are tiny: −0.1, −0.1 and 0.1. Page 87 agrees on almost every sign, but its big
numbers sit where the query's are small. Binary cannot see size, so it cannot tell a tiny
disagreement from a big one.

Now, let's add a rescoring step. **Rescoring** (Chapter 25) means taking a wider shortlist from
the compressed index and re-checking it with full vectors.

**Step 1:** Ask binary for 3 candidates instead of 2: pages 212, 87 and 1,140.

**Step 2:** Fetch their full vectors.

**Step 3:** Compute exact dot products with the query: page 212 scores 0.98, page 1,140 scores
0.94, and page 87 scores 0.37.

**Step 4:** Keep the top 2: page 212 and page 1,140.

The algorithm now thinks like this: *"Page 1,140 disagrees on three small numbers but agrees on
all the big ones. Its true score is almost as high as page 212's."* The Scholar reads both pages
and answers, "Yes, but teams under 50 seats need the Security add-on." The answer is correct.

**Binary alone is a blunt instrument. Binary plus rescoring is one of the best memory deals in
the field.** At full scale the cascade looks like this:

```
Stage 1 — search the 100M binary codes (9.6 GB), usually through
          an index such as HNSW built over them.
          Hamming distance, SIMD popcount
          → top 1,000 candidates                       a few ms

Stage 2 — rescore those 1,000 with int8 or float32 vectors
          → top 10                           ~1 ms in RAM, a few ms from SSD
```

If the full vectors stay on SSD, Acme's 100 million chunks need only about 25 GB of RAM. That
covers the 9.6 GB of codes, plus the graph links and IDs built over them (Chapter 60 itemises it).

The reason this works: **recall@1000 for binary is very high (often above 0.98) even when
recall@10 is poor.** Binarising destroys the fine ordering but keeps the coarse neighbourhood.
The right answers are in the candidate set. They are simply mis-ranked, and the rescore puts
them back in order. For binary, the usual depth is about 100× the final k, as here: 1,000
candidates for a top 10.

Published reports for this cascade often land around **95–99% of full-precision quality at
roughly 3% of the vector bytes**, frequently with *lower* latency than a `float32` index. With an
HNSW graph on top and the full vectors on SSD, the index needs closer to 8% of the RAM of the
`float32` version (Chapter 60).

**Note:** There is a refinement worth knowing: **asymmetric scoring**. The query stays
`float32` and is compared directly against the documents' bits, instead of being binarised too.
It is more accurate than comparing bits with bits, at almost no extra cost, for the same reason
ADC's asymmetry helps PQ (Chapter 24).

---

## Choosing a compression scheme

| Scheme | Bytes (768-d) | Compression | Recall (standalone) | Needs training? |
|---|---|---|---|---|
| float32 | 3,072 | 1× | 1.00 | No |
| float16 | 1,536 | 2× | ~1.00 | No |
| int8 (SQ) | 768 | 4× | ~0.99 | Min/max only |
| PQ (m=96) | 96 | 32× | ~0.90 | Yes — codebooks |
| Binary | 96 | 32× | ~0.75 | No |
| Binary + rescore | 96 + fetch | ~32× | **~0.97** | No |

The recall figures are typical, not guaranteed. Measure them on your own queries (Chapter 19).

Binary and PQ land at the same size. The differences are what matter.

- **PQ needs trained codebooks** that must be stored, kept fresh as data drifts, and retrained
  when the model changes. **Binary needs nothing.**
- **PQ has higher standalone recall.** **Binary is faster** (popcount instead of table lookups)
  and simpler.
- **Both are transformed by rescoring.** Neither should be used without it.

**Advantages of scalar and binary quantization:** no codebooks, no training beyond a min and
max, one line of code, and fast integer or bit arithmetic.

**Disadvantages:** int8 stops at 4×. Binary's standalone ranking is poor, it depends heavily
on the model (see Ninja notes), and it needs the full vectors kept somewhere for rescoring.

We must use **int8** when we need about 4× and want to change nothing else. We must use
**binary + rescoring** when we need 32× and can reach the full vectors for a shortlist. We use
**PQ** when it is already built into the index we run, such as IVF-PQ or DiskANN. Many strong
systems combine them: binary codes to shortlist, int8 or `float32` vectors to rescore.

---

### Under the hood

```python
import numpy as np

def encode_binary(V):                       # (n, d) float → (n, d/8) uint8
    return np.packbits(V > 0, axis=1)

POPCOUNT = np.unpackbits(np.arange(256, dtype=np.uint8)[:, None], axis=1).sum(1)

def hamming(codes, q_code):                 # (n, d/8) vs (d/8,) → (n,)
    return POPCOUNT[codes ^ q_code].sum(axis=1)

def search_binary_then_rescore(q, codes, full_vectors, k=10, depth=1000):
    q_code = np.packbits(q > 0)
    dists  = hamming(codes, q_code)                        # XOR + popcount
    cand   = np.argpartition(dists, min(depth, len(dists) - 1))[:depth]   # stage 1
    scores = full_vectors[cand] @ q                        # stage 2
    top    = np.argsort(-scores)[:k]
    return cand[top], scores[top]
```

`np.packbits` turns each row of True/False values into bytes, 8 bits at a time. `codes ^ q_code`
is the XOR. `POPCOUNT` is a 256-entry table holding the number of 1s in every possible byte. In
simple words, the code does exactly Steps 1 to 4 from the Hamming example, then the rescore.

NumPy's `POPCOUNT` lookup table is a stand-in. Real implementations use the CPU's
`POPCNT`/`VPOPCNTDQ` instructions or GPU equivalents, which is where the speed actually comes
from.

Note the requirement hidden in the last function: `full_vectors` must be *reachable*. In
production those 1,000 vectors come from disk, a cache, or a separate store. On SSD, a batched
read of 1,000 × 3 KB is about 3 MB, which a fast SSD serves in milliseconds. Plan the storage
for this explicitly.

---

### What people get wrong

**Binary without rescoring.** This is the single most common mistake, and it makes binary
quantization look far worse than it is.

**Quantizing before normalizing.** Normalize first (Chapter 5), always. Quantizing unnormalized
vectors with varying lengths wastes most of your range.

**Discarding full-precision vectors.** You need them for rescoring, re-indexing and migration.
Keep them in object storage. It is cheap, and it is insurance.

**Assuming compression composes freely.** Binary + Matryoshka truncation is often excellent
(Chapter 15). Binary + PQ is usually not. Measure the combination, not the parts.

**Global min/max on wildly skewed data.** A few outlier values can squash everything else into
a handful of int8 steps. Check the distribution, or clip at the 1st and 99th percentiles.

**Calling Stage 1 a "binary search".** A binary search is the halve-the-sorted-list algorithm.
Searching binary codes is a different thing. The name collision causes real confusion in design
reviews.

---

### Ninja notes

**Binary quantization quality depends heavily on the model.** It works well when embedding
dimensions are roughly zero-centred and high-variance. Well-trained contrastive models tend to
produce that, because the uniformity term (Chapter 12) spreads vectors over the sphere. It works
badly on models with strong anisotropy (Chapter 3) or dimension collapse, where many dimensions
have tiny variance and their sign is essentially noise.

A cheap diagnostic: compute the per-dimension mean and standard deviation over a corpus sample.
Dimensions whose mean is far from zero relative to their standard deviation produce a nearly
constant bit, which is a wasted bit. If many dimensions look like that, **centre the vectors
before binarising**: subtract the corpus mean, then take the sign. This one adjustment can add
several points of recall, costs a single vector of metadata, and is omitted from most
implementations. Chapter 27 takes the same instinct much further, with a random rotation.

---

### Key takeaways

- **Scalar quantization = store each number on a coarser scale.** int8 is 4× smaller, loses
  about 1% recall, needs no codebooks, and is often faster.
- **Binary quantization = keep one bit per number, its sign.** 32× smaller, no training.
- **Hamming distance = popcount(a XOR b)**, the number of differing bits.
- Binary's standalone ranking is poor, because it cannot see size. **Binary + rescoring
  reaches ~95–99% of exact.**
- The cascade works because recall@1000 stays high even when recall@10 collapses. Rescore about
  100× the final k.
- Always normalize before quantizing, and always keep full vectors somewhere.
- Centre your vectors before binarising if dimensions are not zero-centred.

### What's next

Scalar and binary quantization are simple and somewhat wasteful. Product quantization is
efficient and needs training. [Chapter 27](./27-turboquant.md) shows a way to get close to both:
near-optimal compression with no codebooks at all.

We now know how int8 rounds, how bits are compared with XOR and popcount, why binary alone gets
the order wrong, and why a rescore makes it one of the best memory deals available.
