---
title: "Normalization and the Unit Sphere"
chapter: 5
part: "Part I — Foundations: What a Vector Actually Is"
slug: normalization
readingTime: "9 min"
summary: "Dividing a vector by its own length is a one-line operation with outsized consequences. Here is what it buys, what it costs, and when to skip it."
tags: [foundations, math, normalization]
prev: 04-measuring-similarity
next: 06-high-dimensions
---

# Normalization and the Unit Sphere

**The one-paragraph version.** To normalize a vector, we divide it by its own length. Every
vector then has length exactly 1 and sits on the surface of a sphere of radius 1: all direction,
no magnitude. This makes the dot product equal cosine similarity, makes all three rulers from
Chapter 4 rank identically, and makes scores comparable across documents. It also makes
**quantization** (storing each number in fewer bits to save memory) better behaved. It costs
whatever information was stored in the length, which for text retrieval is usually nothing we
want.

In this chapter, we will learn what normalization is and why a search without it can return the
same page for every question. We will normalize a vector by hand and in one line of code. We will
also see what normalization buys, what it costs, where in a system to do it, and when to skip it.

We will cover the following:

- What is normalization
- Why we need normalization
- How to normalize a vector
- What normalization buys us
- What normalization costs, and when to skip it
- Where to normalize

---

## What is normalization

Before we start, the length of a vector needs a proper name.

**Norm = the length of a vector: square each number, add the squares, take the square root.**

We met this in Chapter 4 as "length". For example, `[3, 4]` has norm √(3² + 4²) = √25 = 5.

**Normalization = dividing a vector by its own norm, so that its length becomes exactly 1.**

**Unit vector = a vector whose length is exactly 1.** Normalizing turns any vector, except one
made of all zeros, into a unit vector.

Written as a formula:

$$ \hat{a} = \frac{a}{\lVert a \rVert_2}, \qquad \lVert a \rVert_2 = \sqrt{\sum_i a_i^2} $$

Here $\hat{a}$ ("a-hat") is the normalized version of $a$, and $\lVert a \rVert_2$ is its norm.

Let's normalize `[3, 4]`. We divide each number by the norm, 5, and get `[0.6, 0.8]`. Check the
new length: √(0.6² + 0.8²) = √(0.36 + 0.64) = √1 = 1.

In simple words, we keep the direction the vector points in, and throw away how long it is.

Think of it like two hikers describing their routes:

- Hiker A says, "I went 12 km northeast."
- Hiker B says, "I went 3 km northeast."
- Each route is a vector. The kilometres are its length. "Northeast" is its direction.
- Normalizing throws away the kilometres and keeps the bearing.

Different journeys, identical **bearing**. After normalizing, both hikers are simply "northeast",
and now they are trivially comparable.

Every unit vector lies on the surface of a ball of radius 1, centred on the corner of the room.
Mathematicians call that surface the **unit hypersphere**. We can just call it the sphere. In 2
dimensions, it is simply a circle of radius 1.

---

## Why we need normalization

Let's see what goes wrong without it. We reuse the toy 2-number model from Chapter 4. Position 1
says how much a text is about logging in and security. Position 2 says how much it is about
accounts and plans.

Here are three Acme pages, stored exactly as the model produced them:

| Page | Vector | Length |
|---|---|---|
| Page 212: "Single sign-on is included on the Pro plan." | [1.6, 1.2] | 2 |
| "Changing your plan" | [0.56, 1.92] | 2 |
| Security overview (4,000 words) | [12, 2] | about 12.17 |

And two questions, each already of length 1:

- **q1:** *"Does the Pro plan include single sign-on?"* = `[0.8, 0.6]`
- **q2:** *"Can I switch from Basic to Pro in the middle of the month?"* = `[0.28, 0.96]`

The Librarian scores every page with the dot product from Chapter 4:

```
q1 · page 212  = 0.8×1.6   + 0.6×1.2   = 1.28  + 0.72  =  2.00
q1 · changing  = 0.8×0.56  + 0.6×1.92  = 0.448 + 1.152 =  1.60
q1 · security  = 0.8×12    + 0.6×2     = 9.6   + 1.2   = 10.80   ← wins

q2 · page 212  = 0.28×1.6  + 0.96×1.2  = 0.448 + 1.152 =  1.60
q2 · changing  = 0.28×0.56 + 0.96×1.92 = 0.157 + 1.843 =  2.00
q2 · security  = 0.28×12   + 0.96×2    = 3.36  + 1.92  =  5.28   ← wins
```

The Security overview wins both questions. The Scholar answers the single sign-on question from a
general tour of security features. Then it answers the plan-switching question from the very
same page. The answer is wrong, twice.

This is the classic symptom: **the same page comes back no matter what we ask.**

Now let's normalize the pages. We divide each one by its length:

```
page 212 = [1.6, 1.2]   / 2     = [0.80, 0.60]
changing = [0.56, 1.92] / 2     = [0.28, 0.96]
security = [12, 2]      / 12.17 ≈ [0.99, 0.16]
```

And score again:

```
q1 · page 212 = 1.00   ← wins      q2 · page 212 = 0.80
q1 · changing = 0.80               q2 · changing = 1.00   ← wins
q1 · security ≈ 0.89               q2 · security ≈ 0.43
```

Page 212 wins the single sign-on question. "Changing your plan" wins the plan-switching question.
The answer is correct, both times.

In simple words, before normalization the longest page shouted over everyone else. After
normalization every page speaks at the same volume, and only direction decides.

**Note:** Normalizing only the question changes nothing about the order. Multiplying a question by
some number multiplies every page's score by that same number, so the ranking stays the same. It
is the pages that need normalizing.

---

## How to normalize a vector

**Step 1:** Square every number in the vector.

**Step 2:** Add the squares up.

**Step 3:** Take the square root. That is the norm.

**Step 4:** Divide every number in the vector by the norm.

```python
import numpy as np

def normalize(v):
    return v / np.linalg.norm(v)

v = np.array([3.0, 4.0])
normalize(v)                    # → [0.6, 0.8]
np.linalg.norm(normalize(v))    # → 1.0
```

Means, `np.linalg.norm` does Steps 1 to 3, and the division does Step 4.

For a batch, where each row is one vector, we normalize along the last axis and guard against
all-zero vectors:

```python
def normalize_batch(V, eps=1e-12):
    return V / np.maximum(np.linalg.norm(V, axis=1, keepdims=True), eps)
```

That `eps`, a tiny number used instead of zero, is not paranoia. Empty strings, whitespace-only
pieces of text and some edge cases in how a model handles padding (filler added to make inputs
the same length) can produce all-zero embeddings. An all-zero vector has length 0. Dividing by
zero yields `NaN` ("not a number"). A single `NaN` vector can poison an entire index build, with
an error message that points nowhere near the actual cause.

---

## What normalization buys us

**1. The metric question (which ruler to use) disappears.** As shown in Chapter 4, on unit vectors
the dot product, cosine similarity and Euclidean distance all produce the same ranking. So we can
use the cheapest one, the raw dot product, with no downside.

**2. Scores become comparable.** Without normalization, a long document's embedding may simply
have a bigger norm, so it scores higher against every query. That is exactly what the Security
overview did above. Normalizing removes this systematic bias.

**3. A bounded, interpretable range.** Every score between two unit vectors lies between −1 and 1.
We can set thresholds, write score-based rules, and blend with other signals without
worrying about scale. Bounded is not the same as calibrated, though. A 0.8 still means different
things in different models (Chapter 3).

**4. Quantization behaves.** Scalar quantization (Chapter 26) works by mapping a range of values
onto a few bits. If every vector lives on the same sphere, that range is consistent
across the whole Library, and a single calibration works for everything. For example, no number
in a unit vector can be above 1 or below −1. The raw Security overview vector has a 12 in it,
which would blow past any range set for page 212's `[1.6, 1.2]`. Vectors with wildly different
magnitudes force per-vector scale factors, or accept much worse quantization error. (Binary
quantization keeps only the sign of each number, so length does not affect it.)

---

## What normalization costs, and when to skip it

We lose the magnitude. Whether that matters depends entirely on whether the model put anything
there.

For most modern text embedding models, magnitude is close to meaningless. Many are *trained* with
normalized outputs, so the model never learned to use length for anything. Normalizing them is
free.

But there are real cases where length carries signal:

- **Recommenders.** Two-tower models (one neural network for users, one for items) often let
  popular items develop larger norms, so the dot product naturally boosts them. Normalize, and the
  popularity signal is gone.
- **Some sparse and learned-sparse representations.** These are keyword-style vectors, covered in
  Chapters 7 and 39. Term weights and document length interact on purpose there. BM25, the
  classic keyword-scoring formula, already handles length in its own way (Chapter 8).
- **Confidence-carrying representations.** A few architectures store "how certain am I about this
  item" in the magnitude.

> **The rule:** normalize unless you can name the thing that magnitude encodes in your system. If
> you can name it, think twice.

---

## Where to normalize

There are three places to do it. The important thing is to pick one and be consistent. (A
*vector database* is software that stores the vectors together with the Card Catalog built over
them.)

| Where | How | When to prefer |
|---|---|---|
| At encode time | `model.encode(..., normalize_embeddings=True)` | Default. Normalize once, store normalized. |
| At index time | Database config, e.g. cosine metric | Fine, but check whether it normalizes or just divides at query time |
| At query time only | Manual | Almost always a bug — mismatched query and document treatment |

The failure everyone hits at least once: documents stored unnormalized, queries normalized. That
is exactly the broken example above. Every score is scaled by an arbitrary per-document constant,
the ranking is quietly wrong, and nothing throws an error. Evaluation numbers come out mediocre in
a way that looks like "embeddings just aren't very good."

**Assert it in the ingestion pipeline**, the code that turns pages into stored vectors. One line,
permanently:

```python
assert np.allclose(np.linalg.norm(V, axis=1), 1.0, atol=1e-4), "vectors not normalized"
```

In simple words, this line stops the pipeline if any stored vector has a length that is not 1.

---

### Under the hood

A note on the sphere itself, because it explains something in Chapter 6.

In two dimensions, the unit sphere is a circle. Random points on it spread out nicely, and two
random points can have any cosine from −1 to 1. In 768 dimensions, the surface is unimaginably
vast, and two randomly chosen points on it are *almost always* close to orthogonal. Their cosine
similarity concentrates tightly around 0.

We can watch it happen:

```python
rng = np.random.default_rng(0)
X = normalize_batch(rng.standard_normal((20_000, 768)))   # 20,000 random unit vectors
cos = (X[:10_000] * X[10_000:]).sum(axis=1)               # cosine of 10,000 random pairs
cos.mean(), cos.std()                                     # → about 0.000 and 0.036
```

Means, in 768 dimensions about 95% of random pairs score between −0.07 and +0.07.

This is why real embeddings showing a random-pair similarity of 0.7 (Chapter 3) is such a
striking fact. It means the model is *not* using the sphere evenly. It has crammed everything into
a narrow cone. That cone is wasted capacity, and reducing it is an active research area. One
approach is whitening (see Ninja notes). Another is isotropy regularisation, a training penalty
that rewards spreading out. A third is the contrastive training of Chapter 12, which pushes
unrelated texts apart and so tends to widen the cone.

---

### What people get wrong

**Normalizing twice.** Harmless mathematically, since the second one changes nothing. But it
usually signals that nobody knows where normalization happens, which means someday it will happen
in only one place.

**Normalizing after quantization.** Order matters. Normalize, *then* quantize.

**Forgetting zero vectors.** Empty or whitespace chunks → zero vector → `NaN` → index build
failure hours later, with a stack trace that blames the index.

**Assuming the database does it.** "Cosine distance" in a vector database config may only mean it
divides by norms at query time, not that your stored vectors are unit length. Some databases
normalize on insert and some do not, so check. That distinction matters for quantization quality.

---

### Ninja notes

If your model produces a badly anisotropic space, **whitening** can help. Centre the embeddings
(subtract the corpus mean), decorrelate the dimensions using the inverse square root of the
covariance matrix, then re-normalize. On some older models this measurably improves retrieval at
essentially zero serving cost, because it is a fixed transform (subtract a mean, multiply by a
matrix) you can add as a final step of the encoder.

Two cautions. It must be fitted on a large, representative sample of your corpus. And it must be
applied identically to queries and documents. A whitening transform fitted on documents but not
applied to queries is a subtle, catastrophic mismatch. On well-trained modern models the gain is
usually small. On a model that is struggling on your domain, it is worth thirty minutes to test.

---

### Key takeaways

- **Normalization = dividing a vector by its norm, so its length becomes 1.** It keeps direction
  and discards magnitude.
- Without it, a long page can win every search, as the Security overview did.
- It makes all three rulers rank identically, so we can use the fastest one, the dot product.
- It makes scalar quantization better behaved, because one range fits every vector.
- Skip it only when you can name what magnitude encodes, such as popularity or confidence.
- Guard against zero vectors, and assert normalization in your ingestion pipeline.

### What's next

We have twice mentioned that 768-dimensional space behaves oddly.
[Chapter 6](./06-high-dimensions.md) confronts that directly: the curse of dimensionality, why it
should make vector search impossible, and why it does not.

In short, normalization is one line of code that settles the ruler question, as long as we do it
once, in one agreed place, and only when length means nothing.
