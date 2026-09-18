---
title: "TurboQuant: Near-Optimal Quantization Without Codebooks"
chapter: 27
part: "Part IV — Index Structures"
slug: turboquant
readingTime: "15 min"
summary: "Scalar quantization is cheap but wasteful. Product quantization is efficient but needs trained codebooks. TurboQuant rotates the problem away and gets near-optimal compression with no training at all."
tags: [turboquant, quantization, memory, data-oblivious, rotation, mips]
prev: 26-scalar-binary-quantization
next: 28-hnsw-1-small-worlds
---

# TurboQuant: Near-Optimal Quantization Without Codebooks

**The one-paragraph version.** Until now, every quantizer forced a choice. Scalar and binary
quantization (Chapter 26) need no training but waste bits. Product quantization (Chapter 24)
uses bits well but needs trained codebooks. **TurboQuant** (Zandieh, Daliri, Hadian & Mirrokni,
Google, 2025, arXiv 2504.19874) avoids the trade. It first applies a random rotation, so every
coordinate follows the same known distribution. Then it rounds each coordinate with the best
scalar quantizer for that distribution. The paper proves the error stays within a small
constant factor (about 2.7×) of the best any quantizer could guarantee, at every bit budget. And
there are **no codebooks, no training pass, and nothing to go stale.**

In this chapter, we will learn about TurboQuant, a way of compressing vectors with no training.
We will also see why rotating first helps, how encoding works step by step, and how a second
stage keeps dot products honest. Finally, we will see what it saves and how it compares with PQ
and binary quantization.

We will cover the following:

- What is TurboQuant
- Why we need TurboQuant
- How TurboQuant works
- An example of TurboQuant
- The second stage: keeping dot products honest
- How TurboQuant helps memory
- TurboQuant vs PQ vs binary, and when to use which one

---

## What is TurboQuant

Before jumping in, we must know two ideas.

A **distribution** is the shape of how values spread out: which values are common and which are
rare. Many measurements follow a **bell curve**, also called a **Gaussian**. Most values sit
near the middle, and fewer sit far out.

A **random rotation** is multiplication by an orthogonal matrix chosen at random. We met
orthogonal matrices with OPQ in Chapter 25. In simple words, it turns the whole Map Room in a
random direction without stretching anything. Every distance and every dot product stays
exactly the same. Only the coordinates change.

**TurboQuant = random rotation + the best fixed scalar quantizer for the rotated coordinates
(+ a 1-bit correction for dot products).**

Let's break it down. The *rotation* makes every coordinate follow the same known
distribution. The *quantizer* rounds each coordinate to a few levels placed where values
actually fall, and those levels come from mathematics, not from our data. The *correction* is an
optional second stage that makes estimated dot products right on average.

Picture dealing cards from a deck with every ace and king stacked on top.

- The deck is one vector.
- Each card is one original coordinate. The high cards are coordinates with big values.
- Each player's hand is one rotated coordinate, a mix of several cards.
- Shuffling and dealing is the random rotation.
- Judging every hand by the same rule is the fixed quantizer.

Deal from the unshuffled deck and one player gets every high card, while the others get only low
ones. No single rule for judging hands suits them all. So we shuffle first. We still do not know
any one hand, but we know what a *typical* hand looks like. An embedding vector is often that
unshuffled deck. **TurboQuant shuffles first**, then judges every hand with the same
well-designed quantizer.

---

## Why we need TurboQuant

Let's go back to Acme at scale: 100 million chunks, 307 GB as `float32`. Chapters 24 and 26 gave
us two ways down to about 96 bytes per chunk. Each one has a cost.

**Plain scalar and binary quantization waste bits.** There are two reasons.

- **Coordinates are uneven.** Some range over $[-0.9, 0.9]$, and some barely leave
  $[-0.02, 0.02]$. A uniform quantizer spends the same bits on both, so the bits on the second
  are mostly wasted.
- **The distribution is unknown and not uniform.** Coordinates are usually bell-shaped. Evenly
  spaced levels leave the outer levels almost empty.

**Distortion** is the error that quantization adds. For any bit budget, information theory
gives a **distortion-rate lower bound**, the smallest distortion any quantizer could guarantee.
Plain scalar quantization can sit well above it, especially at 1–2 bits per coordinate, which is
exactly where the big memory savings are.

**Product quantization closes much of the gap, but it needs training.** That means a k-means
pass over a large sample and 96 codebooks to store. It also means slow drift, as Acme's
customers write about things the codebooks never saw (Chapter 25).

Now, the question is, can we get efficient bits without any training?

> **The idea:** if we cannot design a good quantizer for an unknown, uneven distribution, we
> transform the data until its distribution is known and even, then use the best quantizer for
> *that*.

The paper relies on what a random rotation does in high dimensions. Every rotated coordinate
follows the same concentrated **Beta distribution** (a family of hump-shaped curves on a fixed
range), which at embedding sizes is very close to a bell curve. Different coordinates also
become nearly independent, and the piled-up energy spreads across all of them.

So we know every coordinate's distribution in advance, and we can use the **optimal scalar
quantizer for that distribution**. Nothing is fitted to our data, so nothing goes stale.

---

## How TurboQuant works

**Phase 1: Setting up. This happens once, before we see any data.**

**Step 1:** Pick a random seed. From it, build a random 768 × 768 rotation: fill a matrix with
random bell-curve numbers, then make it orthogonal with a standard routine called a QR
decomposition.

**Step 2:** Pick a bit budget, say 2 bits per coordinate.

**Step 3:** Write down the 4 optimal levels for a standard bell curve: −1.51, −0.45, 0.45 and
1.51. They come from running k-means (Chapter 23) in one dimension on the bell curve itself,
which is called the **Lloyd–Max quantizer**. These are constants. Every corpus in the world uses
the same ones.

**Phase 2: Encoding a vector.**

**Step 1:** Measure the vector's length (its norm, Chapter 5). Store it as one `float32`, 4
bytes.

**Step 2:** Rotate the vector.

**Step 3:** Rescale the rotated coordinates: divide by the length, multiply by $\sqrt{768}$.

**Step 4:** Round each coordinate to its nearest level. Store the level's number, 2 bits.

**Phase 3: Answering a query.**

**Step 1:** Rotate the query with the same rotation. The query stays at full precision, as with
ADC in Chapter 24.

**Step 2:** For each stored vector, rebuild its rotated coordinates from the levels and the
stored length.

**Step 3:** Take the dot product with the rotated query. Rotation keeps dot products unchanged,
so this estimates the true score.

**Step 4:** Keep a shortlist, and rescore it with full vectors (Chapter 25).

In simple words, we shuffle the numbers with one fixed random rotation, then round every number
to the same four levels.

**Note:** Why $\sqrt{768}$? A vector of length 1 spreads its length over 768 coordinates, so
after a random rotation a typical coordinate is about $1/\sqrt{768}$ in size. Multiplying by
$\sqrt{768}$ brings it to about 1, the scale the levels were designed for.

---

## An example of TurboQuant

Let's shrink everything to 4 coordinates and 1 bit each, so we can do it by hand.

The real method uses a random rotation. To keep the arithmetic easy, we use one fixed rotation
of 4 numbers: add and subtract them in four fixed patterns, then halve.

```
new 1 = (x1 + x2 + x3 + x4) / 2
new 2 = (x1 - x2 + x3 - x4) / 2
new 3 = (x1 + x2 - x3 - x4) / 2
new 4 = (x1 - x2 - x3 + x4) / 2
```

| Vector | Numbers | Length² |
|---|---|---|
| Query: *"Does the Pro plan include single sign-on?"* | [0.6, 0.1, 0.1, 0.8] | 1.02 |
| Page 212: SSO included on Pro | [0.6, 0.1, 0.2, 0.8] | 1.05 |
| Page 87: two-factor login on Basic | [0.4, 0.3, 0.8, 0.4] | 1.05 |

The exact dot products are 1.03 for page 212 and 0.67 for page 87. Page 212 should rank first.

**Before: 1 bit per coordinate, no rotation.**

Every number in both pages is positive, so both pages get the bits `1111`. To score a page, we
add up the query's numbers, with a plus where the page's bit is 1 and a minus where it is 0.

```
page 212, bits 1111:  +0.6 +0.1 +0.1 +0.8 = 1.6
page 87,  bits 1111:  +0.6 +0.1 +0.1 +0.8 = 1.6
```

Both pages also have the same length. It is a perfect tie, so page 87 comes first about half
the time. The ranking is wrong. All four bits were wasted, because every bit just said
"positive".

**After: rotate first, then 1 bit.**

**Step 1:** Rotate the query: `[0.8, -0.1, -0.1, 0.6]`.

**Step 2:** Rotate page 212: `[0.85, -0.05, -0.15, 0.55]`. Its bits are `1001`.

**Step 3:** Rotate page 87: `[0.95, 0.25, -0.25, -0.15]`. Its bits are `1100`.

**Step 4:** Score both pages with the rotated query.

```
page 212, bits 1001:  +0.8 -(-0.1) -(-0.1) +0.6 = 1.6
page 87,  bits 1100:  +0.8 +(-0.1) -(-0.1) -0.6 = 0.2
```

The algorithm now thinks like this: *"Page 212 agrees with the query on every rotated
coordinate. Page 87 disagrees on the big fourth one."* Page 212 ranks first. The ranking is
correct.

Same 4 bits. The rotation gathered the shared "everything is positive" part into the first
coordinate, and spread the real differences across the others, where the bits can see them.

**Note:** The real score also multiplies by the level size and the stored length, which are the
same for both pages here. And a *random* rotation, not a fixed one, is what lets the method
promise good behaviour for every input.

---

## The second stage: keeping dot products honest

What ranks our results is the **dot product** between query and page (Chapter 4), not how well
we can rebuild the vector. ScaNN's anisotropic quantization (Chapter 24) comes from the same
insight.

The paper points out a catch. A quantizer tuned for the smallest rebuilding error gives
**biased** dot-product estimates. Biased means the errors lean one way on average instead of
cancelling out. So TurboQuant adds a second stage built on QJL.

**QJL = Quantized Johnson–Lindenstrauss: project a vector onto random directions and keep only
the signs.** It is the SimHash trick from Chapter 21. The Johnson–Lindenstrauss lemma
(Chapter 6) says random projections keep distances roughly intact.

Here is the paper's two-stage version, for a budget of $b$ bits per coordinate.

**Step 1:** Run the first stage with one bit fewer, $b - 1$ bits per coordinate.

**Step 2:** Compute the **residual** (Chapter 24): the original vector minus what the first
stage can rebuild.

**Step 3:** Multiply the residual by a random matrix and keep only the signs, 1 bit per
coordinate.

**Step 4:** Store the residual's length as one more number.

**Step 5:** At query time, add a small correction, computed from those signs, to the
first-stage score.

The combined estimate is **unbiased**. In simple words, it is correct on average, and its
errors do not lean in any direction. So errors tend to cancel instead of piling up, and no
vector's score is systematically pushed up or down. That may let rescoring work with a shorter
shortlist. How much shorter is something to measure.

---

## How TurboQuant helps memory

**1. Better accuracy at the same byte count.** A near-optimal quantizer can distort vectors
less than naive scalar or binary quantization with the same bits, so we can go to fewer bits
before recall drops. How many depends on the data.

| 768-d vector as | Bytes | 100M vectors | Training needed |
|---|---|---|---|
| float32 | 3,072 | 307 GB | — |
| int8 (Ch. 26) | 768 | 77 GB | min/max |
| PQ, m=96 (Ch. 24) | 96 | 9.6 GB + codebooks | k-means codebooks |
| Binary (Ch. 26) | 96 | 9.6 GB | none |
| **TurboQuant, 2-bit** | **192 + 4 B scale = 196** | **19.6 GB** | **none** |
| **TurboQuant, 1-bit** | **96 + 4 B scale = 100** | **10 GB** | **none** |

The "4 B scale" is the stored length from Phase 2, Step 1. The table counts the first stage,
which is what the code below builds. The two-stage version keeps one more `float32` per vector,
the residual's length.

**Compare rows at equal bytes.** At about 96 bytes, TurboQuant's 1-bit mode is sign
quantization after a random rotation. So it is protected against the uneven coordinates that hurt
plain binary. At the same budget, the two-stage version has no bits left for its first stage. It
is QJL alone, and it gives an unbiased score estimate instead. Whether TurboQuant matches PQ at
equal bytes on *your* data is something to measure, not assume. For reference, the paper's
nearest-neighbour experiments (at 2 and 4 bits per coordinate) report higher recall than the
product quantization baselines it tested, with indexing time close to zero.

**2. No codebooks and no training pass.** Encoding is a rotation and a fixed rounding, one
vector at a time, which suits streaming ingestion. The paper calls the method data-oblivious (it
never looks at the data to set itself up) and suited to online use.

**3. In principle it can replace the compression step in every other structure in this book.**

- **HNSW vectors** (Chapter 31), when int8 still does not fit in RAM.
- **DiskANN's in-RAM navigation codes** (Chapter 32).
- **ColBERT token vectors** (Chapter 37).
- **MUVERA's FDEs** (fixed-size vectors that stand in for many, Chapter 38).
- **ColPali patch vectors** (one vector per small square of a page image, Chapter 45), at 1,030
  vectors per page.

The last three could matter most. **Multi-vector retrieval stores hundreds to about a thousand
times more vectors** than single-vector retrieval, so compression decides what is affordable.

**Note:** These integrations are where the method *could* go, not results from the paper.
Adoption has started. Qdrant 1.18 (May 2026) ships an extended variant of the first stage. It
uses a fast Hadamard rotation (Under the hood) instead of a dense one. It adds a per-coordinate
calibration pass for real embeddings, and a length renormalization step borrowed from RaBitQ,
another rotation-based quantizer. Check what your index library supports, and treat TurboQuant
as something to benchmark against PQ on your own data.

---

## TurboQuant vs PQ vs binary, and when to use which one

| | Binary (Ch. 26) | PQ, m=96 (Ch. 24) | TurboQuant, 1–2 bits |
|---|---|---|---|
| Bytes per 768-d vector | 96 | 96 | 100 or 196 |
| Training | None | k-means codebooks | None |
| Uneven coordinates | Wastes bits | Handled by learning | Handled by random rotation |
| Goes stale as data drifts | No | Yes | No |
| Proven distortion bound | No | No | Yes, within ~2.7× of the lower bound |
| Extra work per vector | None | Table lookups | One rotation per vector and query |
| Maturity | Everywhere | Everywhere | New, early adoption |

**Advantages of TurboQuant:** no training, a proven distortion bound, and streaming-friendly
encoding. **Disadvantages:** a rotation per vector and query, young tooling, and a guarantee
about distortion rather than about recall on your queries.

We must use **binary + rescoring** when we want the simplest 32× and our model's bits are
already well balanced. We must use **PQ** when it is built into the index we already run, such
as IVF-PQ or DiskANN. We should try **TurboQuant** when we want efficient bits without training,
when data arrives as a stream, or when a huge multi-vector collection needs 1–4 bits per
coordinate. Many strong systems use the same outer shape for all three: compressed codes to
shortlist, full vectors to rescore.

---

### Under the hood

We build the rotation the way the paper does: a dense, uniformly random orthogonal matrix from
a QR decomposition. The dimension stays at 768, so 2-bit codes are exactly 192 B + 4 B, as in
the table.

```python
import numpy as np

def random_rotation(d, seed=0):
    """Dense, uniformly random orthogonal matrix: QR of a Gaussian matrix."""
    rng = np.random.default_rng(seed)
    Q, R = np.linalg.qr(rng.standard_normal((d, d)))
    return (Q * np.sign(np.diag(R))).astype(np.float32)   # sign fix keeps it uniform

# Optimal (Lloyd–Max) levels for a standard bell curve. After rotating and rescaling,
# every coordinate follows (almost exactly) this curve, so these constants never change.
LEVELS = {1: np.array([-0.7979, 0.7979], dtype=np.float32),
          2: np.array([-1.5104, -0.4528, 0.4528, 1.5104], dtype=np.float32)}

class TurboQuantMSE:
    """First stage only (the QJL residual stage is omitted for brevity)."""
    def __init__(self, d=768, bits=2, seed=0):
        self.d, self.levels = d, LEVELS[bits]
        self.P = random_rotation(d, seed)          # store the seed, not the matrix

    def encode(self, V):                                       # Phase 2
        norms = np.linalg.norm(V, axis=1, keepdims=True)       # the +4 B per vector
        R = (V @ self.P.T) / norms * np.sqrt(self.d)           # rotate, rescale to ~N(0, 1)
        codes = np.abs(R[..., None] - self.levels).argmin(-1)  # nearest fixed level
        return codes.astype(np.uint8), norms.astype(np.float32)

    def decode(self, codes, norms):
        R = self.levels[codes] * norms / np.sqrt(self.d)
        return R @ self.P                                      # rotate back

    def scores(self, q, codes, norms):                         # Phase 3, Steps 1–3
        q_rot = self.P @ q                                     # Step 1: rotate the query once
        R = self.levels[codes] * norms / np.sqrt(self.d)       # Step 2: rebuild, no rotate-back
        return R @ q_rot                                       # Step 3: approximate dot products
```

For readability, the codes sit one per byte. A real implementation packs them into 192 bytes.
`LEVELS` is a constant that every corpus shares. In simple words, that is what "data-oblivious"
means. As a self-test, on vectors of length 1 the average squared error should come out near
the paper's figures. That is about 0.36 at 1 bit and 0.117 at 2 bits, however uneven the input.

**The fast stand-in: the randomized Hadamard transform.** The dense rotation has a real cost.
Rotating one vector takes 768 × 768, about 590,000 multiply-adds, and the matrix takes 2.4 MB.

**Hadamard matrix = a square matrix of +1s and −1s whose rows are all at right angles.** Our
hand example used the 4 × 4 one, scaled by 1/2.

**Randomized Hadamard transform = flip the sign of each coordinate at random, then apply a
scaled Hadamard matrix.** Every entry is ±1. So a routine called the fast Walsh–Hadamard
transform needs only additions and subtractions, in $O(d \log d)$ steps. For 1,024 coordinates,
that is about 10,000 steps instead of about a million multiply-adds.

```python
def randomized_hadamard(x, signs):
    """Fast stand-in for the dense rotation. len(x) must be a power of two."""
    y = (x * signs).astype(np.float32)             # Step 1: random sign flips
    h = 1
    while h < len(y):                              # Step 2: fast Walsh–Hadamard
        for i in range(0, len(y), 2 * h):
            a, b = y[i:i+h].copy(), y[i+h:i+2*h].copy()
            y[i:i+h], y[i+h:i+2*h] = a + b, a - b
        h *= 2
    return y / np.sqrt(len(y))                     # rescale so lengths are kept
```

Two cautions come with it.

- **It is a stand-in, not the paper's construction.** The paper's guarantee is for a dense,
  uniformly random rotation. The randomized Hadamard transform is the fast version used in
  practice (Qdrant's implementation uses a Hadamard-based rotation, for example). It mixes
  coordinates well on typical data, but it does not carry the paper's proof.
- **It needs a power-of-two size, and 768 is not one.** We can **pad** to 1,024 with zeros.
  Everything gets mixed, but the codes grow by a third: 2-bit codes become 256 B + 4 B. Or we
  can rotate **blocks** of 256 separately (768 = 3 × 256). The codes stay at 192 B + 4 B, but
  numbers only mix within their own block, so energy piled into one block stays there.

---

### What people get wrong

**"It's just scalar quantization with extra steps."** The rotation is what matters. It turns an
unknown, uneven distribution into a known, even one, and only then does one fixed optimal scalar
quantizer exist.

**Forgetting to rotate the query.** Queries must go through the *same* rotation. Store the
rotation seed with the index.

**Skipping rescoring.** Like every method in Chapters 24–26, retrieve a wide candidate set with
the compressed vectors, then rescore at full precision. The unbiased estimate may let you use
*fewer* candidates. It does not let you use none.

**Treating published results as your results.** The guarantee is about distortion, and the
benchmarks are not your data. Measure recall at each bit budget on your own eval set
(Chapter 19).

---

### Ninja notes

**The numbers behind "near-optimal".** At $b$ bits per coordinate, the paper proves no quantizer
can guarantee an expected squared error below $1/4^b$ for every unit vector. TurboQuant's first
stage stays below $(\sqrt{3}\,\pi/2)/4^b \approx 2.7/4^b$. Its computed values are closer still:
about 0.36, 0.117, 0.03 and 0.009 at 1–4 bits, against lower bounds of 0.25, 0.0625, 0.0156
and 0.0039. The paper also compresses an LLM's KV cache (its memory of the text so far) and
reports no quality loss at 3.5 bits per channel.

**The transferable idea: reshape the problem, then solve the easy version.** OPQ (Chapter 25)
*learns* a rotation to spread variance across PQ subvectors. TurboQuant shows that a *random*
rotation is good enough, which is why it needs no training. LSH (Chapter 21), MUVERA
(Chapter 38) and TurboQuant are all **randomized, training-free constructions with provable
guarantees**. They do not always win, as LSH's loss on dense vectors shows. Where they do win, it
is because they remove training, staleness and codebook management. In production, a cost you never
have to pay is often worth more than a small accuracy edge you have to maintain.

---

### Key takeaways

- **TurboQuant = random rotation + the best fixed scalar quantizer for the rotated coordinates
  (+ a 1-bit QJL correction for dot products).** Zandieh, Daliri, Hadian & Mirrokni, 2025.
- Plain scalar and binary quantization waste bits on uneven coordinates. PQ fixes that with
  trained codebooks. TurboQuant fixes it with a random rotation and fixed levels.
- The paper proves distortion within about 2.7× of the lower bound at every bit budget.
- The two-stage version adds a 1-bit QJL code on the residual, making the dot-product estimate
  **unbiased**.
- Memory: about 100 B (1-bit) or 196 B (2-bit) per 768-d vector, including the 4-byte length.
  Benchmark against PQ at equal bytes on your own data.
- The proof covers a dense random rotation. The randomized Hadamard transform is the fast
  stand-in and needs a power-of-two size (pad, or rotate in blocks).
- Always rotate queries with the same seed, and always rescore the survivors.

### What's next

With compression covered, [Chapter 28](./28-hnsw-1-small-worlds.md) begins four chapters on
HNSW, the graph algorithm that dominates in-memory vector search.

We now know how TurboQuant rotates, rounds to fixed levels and corrects its dot products, and
where it fits beside PQ and binary quantization.
