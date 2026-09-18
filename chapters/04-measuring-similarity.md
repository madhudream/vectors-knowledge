---
title: "Measuring Similarity: Dot, Cosine, Euclidean"
chapter: 4
part: "Part I — Foundations: What a Vector Actually Is"
slug: measuring-similarity
readingTime: "11 min"
summary: "Three rulers for the Map Room, what each one actually measures, when they agree, and the specific ways each one will mislead you."
tags: [foundations, math, similarity, cosine, dot-product]
prev: 03-geometry-of-meaning
next: 05-normalization
---

# Measuring Similarity: Dot, Cosine, Euclidean

**The one-paragraph version.** There are three standard ways to score how alike two vectors are.
**Euclidean distance** measures the straight line between two dots. **Cosine similarity**
measures the angle between them and ignores length. The **dot product** measures the angle *and*
rewards length. On normalized vectors (vectors scaled to length 1) all three produce the same
ranking, which is why most systems normalize and stop worrying. When vectors are *not*
normalized, the choice changes the results, and using the wrong one is among the most common bugs
in production.

In this chapter, we will learn the three rulers for the Map Room, one at a time, with every
calculation done by hand. We will run all three on our running question and watch them pick
three different winners. Then we will see the one small step that makes them agree, and when to
use which one.

We will cover the following:

- Three ways to answer "how alike?"
- Dot product
- Cosine similarity
- Euclidean distance
- An example: three rulers, three different answers
- Why normalizing makes all three agree
- When to use which one

---

## Three ways to answer "how alike?"

Chapter 3 kept saying "close" and "far". Now we make that precise.

Here is our setup. The Librarian holds our main question, *"Does the Pro plan include single
sign-on?"* It must decide which of three texts is most like that question:

- **A:** the paraphrase twin, *"Can our team log in with our company accounts?"* Different words,
  same meaning.
- **B:** *"Does the Pro plan include two-factor login?"* Similar words, different question.
- **C:** Acme's long *Security overview* page, 4,000 words covering every security feature.

The right answer is A, because it asks exactly the same thing.

To keep the arithmetic small, let's pretend our model uses only 2 numbers. Position 1 says how
much a text is about logging in and security. Position 2 says how much it is about accounts and
plans. Real models use hundreds of positions with no names (Chapter 1), but the arithmetic is
identical.

| Text | Vector | In words |
|---|---|---|
| Question q | [4, 3] | Our main question |
| A: paraphrase twin | [8, 6] | Points exactly the same way as q, but twice as long |
| B: two-factor question | [3, 4] | Close to q, pointing a little differently |
| C: Security overview | [12, 2] | Very long, mostly about security |

Why would texts get vectors of different lengths? Some models do not keep lengths equal, so
length can vary from text to text for reasons that have nothing to do with meaning. For this
example, we simply take the lengths as given.

Chapter 1 mentioned a third way to read a vector: as an *arrow* from the corner of the room to
the dot. Angles need that reading. Two arrows can point the same way even if one is much longer.

So there are two things we could compare. We could ask **how far apart** the two dots are. Or we
could ask **whether the two arrows point the same way**. Euclidean distance asks the first.
Cosine similarity asks the second. The dot product asks the second and also rewards length.

---

## Dot product

**Dot product = multiply matching positions, then add everything up.** It is also called the
**inner product**.

$$ a \cdot b = \sum_{i=1}^{d} a_i b_i $$

In simple words, the big Σ sign means "add up". $a_i b_i$ means "position i of a times position i
of b". And d is the number of dimensions, here 2.

Let's score our three texts against the question q:

```
q · A = 4×8  + 3×6 = 32 + 18 = 50
q · B = 4×3  + 3×4 = 12 + 12 = 24
q · C = 4×12 + 3×2 = 48 +  6 = 54
```

The dot product ranks C first, then A, then B.

Read the dot product as: *how much do these two vectors agree, weighted by how big they are?*
Agreement on a position where both numbers are large adds a lot. A position where one number is
positive and the other negative subtracts.

Two facts make the dot product central:

- Cosine and Euclidean can both be written in terms of it, so hardware is built to do it fast.
  One dot product on 768 dimensions is 768 multiply-adds. A modern processor core does billions
  of these per second using **SIMD** (single instruction, multiple data: processor instructions
  that do several multiplies at once). A GPU, a chip built for exactly this kind of bulk
  arithmetic, does far more.
- Searching for the highest dot product has its own name, **MIPS = Maximum Inner Product
  Search**. It has its own research literature, because it behaves differently from
  nearest-neighbour search in ways that matter for index design (see this chapter's Ninja notes).

**Its trap: the dot product rewards long vectors.** C is long, so it scores high against almost
everything. If a model does not produce equal-length vectors and we use the raw dot product, a
handful of long pages can dominate every result list. Engineers usually discover this by noticing
the same three pages in every search.

---

## Cosine similarity

**Cosine similarity = the dot product divided by both lengths, so only direction counts.**

$$ \cos(\theta) = \frac{a \cdot b}{\lVert a \rVert \, \lVert b \rVert} $$

Here θ (theta) is the angle between the two vectors. $\lVert a \rVert = \sqrt{\sum a_i^2}$ is the
vector's length: square each number, add them up, take the square root. Chapter 5 calls this
length the *norm*.

First, the lengths:

```
|q| = √(4² + 3²)  = √25  = 5
|A| = √(8² + 6²)  = √100 = 10
|B| = √(3² + 4²)  = √25  = 5
|C| = √(12² + 2²) = √148 ≈ 12.17
```

Now divide each dot product by the two lengths:

```
cos(q, A) = 50 / (5 × 10)    = 1.00
cos(q, B) = 24 / (5 × 5)     = 0.96
cos(q, C) = 54 / (5 × 12.17) ≈ 0.89
```

Cosine ranks A first, then B, then C. Dividing by the lengths took away C's advantage, and A's
extra length stopped mattering.

The result always lies between −1 and 1:

| Value | Meaning |
|---|---|
| 1.0 | Same direction, as similar as the model can say |
| 0.0 | Orthogonal (at right angles), no shared direction at all |
| −1.0 | Opposite direction |

In practice, with modern text embeddings, we will rarely see values below zero. Many models place
nearly everything in a narrow cone (the anisotropy from Chapter 3), so the real range might be
0.55 to 0.95. **That is normal.** It is also why absolute thresholds must be calibrated per
model.

Cosine is the default for text retrieval because the length of a document should not decide its
relevance. A crisp two-sentence answer like page 212 and a rambling four-page one should both be
findable.

---

## Euclidean distance

**Euclidean distance = the straight-line gap between two dots.** People also call it the **L2**
distance.

$$ \lVert a - b \rVert_2 = \sqrt{\sum_{i=1}^{d} (a_i - b_i)^2} $$

In simple words, subtract position by position, square each gap, add the squares, and take the
square root. It is Pythagoras' theorem stretched to d dimensions. It is also exactly what
`np.linalg.norm(espresso - cold_brew)` did in Chapter 1.

```
q − A = [4−8,  3−6] = [−4, −3]  →  √(16 + 9) = 5.00
q − B = [4−3,  3−4] = [ 1, −1]  →  √(1 + 1)  ≈ 1.41
q − C = [4−12, 3−2] = [−8,  1]  →  √(64 + 1) ≈ 8.06
```

Euclidean ranks B first, then A, then C. A points the same way as q, but its dot sits twice as
far out from the corner, so the gap between the dots is 5 steps.

Euclidean is a **distance**, so smaller is better. That is the opposite convention from the two
similarities above. This flip is a genuine source of bugs when someone swaps metrics (rulers) in a
config file and forgets to flip the sort order. The symptom: search returns the *least* relevant
results, perfectly ranked.

Euclidean is the right choice when magnitude (how big the numbers are, not just their mix) is
real information: physical measurements, colour values, map coordinates, some image feature
spaces.

---

## An example: three rulers, three different answers

Let's put all the scores side by side.

| Ruler | A: paraphrase twin | B: two-factor question | C: Security overview | Picks |
|---|---|---|---|---|
| Dot product (higher = closer) | 50 | 24 | **54** | C |
| Cosine (higher = closer) | **1.00** | 0.96 | 0.89 | A |
| Euclidean (lower = closer) | 5.00 | **1.41** | 8.06 | B |

Three rulers, three different winners.

Let's first see what happens if the Librarian uses the dot product. It says the Security
overview is most like our question. The answer is wrong. That page is a long tour of security
features and never says whether Pro includes SSO.

Now Euclidean distance. It says the two-factor question is most like ours. The answer is wrong.
That text asks about a different feature.

Now cosine similarity. It says the paraphrase twin is most like ours. The answer is correct. It
asks exactly the same thing in different words.

Why did two rulers fail? The dot product was fooled by length: C is long, so it won. Euclidean
was fooled by length too, in the opposite way: A is long, so it was punished for sitting far out.
Only cosine ignored length completely.

Put simply, length meant nothing in this example, so the ruler that ignores length won.

**Note:** No ruler is wrong in general. They answer different questions. If length carried real
meaning, such as how popular an item is, the dot product would be the right ruler.

---

## Why normalizing makes all three agree

Now, the question is, can we make the choice of ruler stop mattering?

Yes. We make every vector the same length, 1, before comparing. This is called *normalizing*,
and Chapter 5 covers it fully. To normalize, divide each number by the vector's length:

```
q = [4, 3]  / 5     = [0.80, 0.60]
A = [8, 6]  / 10    = [0.80, 0.60]
B = [3, 4]  / 5     = [0.60, 0.80]
C = [12, 2] / 12.17 ≈ [0.99, 0.16]
```

Now the dot products again:

```
q · A = 0.80×0.80 + 0.60×0.60 = 1.00
q · B = 0.80×0.60 + 0.60×0.80 = 0.96
q · C = 0.80×0.99 + 0.60×0.16 ≈ 0.89
```

These are exactly the cosine scores. Every length is now 1, and dividing by 1 changes nothing.

Euclidean distance follows too, because of one identity. If both vectors have length 1, then:

$$ \lVert a - b \rVert_2^2 = 2 - 2(a \cdot b) $$

Means, the squared distance is 2 minus twice the dot product. Let's check it on B:
2 − 2 × 0.96 = 0.08, and √0.08 ≈ 0.28. For A: 2 − 2 × 1.00 = 0, so the distance is 0. For C:
2 − 2 × 0.89 ≈ 0.22, and √0.22 ≈ 0.47.

| Ruler (normalized vectors) | A | B | C | Picks |
|---|---|---|---|---|
| Dot product | **1.00** | 0.96 | 0.89 | A |
| Cosine | **1.00** | 0.96 | 0.89 | A |
| Euclidean | **0.00** | 0.28 | 0.47 | A |

Euclidean distance goes down exactly when the dot product goes up. And with unit length, cosine
similarity *is* the dot product, since the lengths it divides by are 1.

**Therefore: on normalized vectors, dot product, cosine similarity and Euclidean distance all
produce exactly the same ranking.** Different numbers, identical order. Search only cares about
order, so the choice becomes free.

This is the single most useful fact in this chapter. Normalize the vectors, use the dot product
(it is the cheapest, with no square root and no division), and the metric question disappears.

---

## When to use which one

| Situation | Use | Why |
|---|---|---|
| Text embeddings, normalized (the common case) | Dot product | Fastest, identical ranking to cosine |
| Text embeddings, not normalized | Cosine | Prevents long vectors dominating |
| Recommender with popularity baked into magnitude | Dot product, deliberately | You *want* popular items boosted |
| Image features, physical measurements | Euclidean | Magnitude is real information |
| Binary / quantized codes | Hamming | Count differing bits (Chapter 26) |
| Multi-vector (ColBERT, ColPali) | MaxSim / Chamfer | A different object entirely (Chapter 36) |

A **recommender** is a system that suggests items, such as products or songs. *Quantized* codes
are vectors squeezed into a few bits per number. **Hamming distance** counts how many bits differ
between two such codes. ColBERT and ColPali are models that keep many
vectors per document, which we meet in Parts V and VI. MaxSim and Chamfer are scores for comparing
a *set* of vectors with another set. Do not worry about the last two rows yet.

We must use **cosine**, or normalize and use the dot product, when length means nothing. For text,
that is almost always. We must use the **raw dot product** when length is deliberate information,
such as popularity. We must use **Euclidean distance** when the numbers are real measurements.
Most text search systems simply normalize everything and use the dot product.

---

### Under the hood

All three rulers on our toy example, then the batched form real systems use:

```python
import numpy as np

def dot(a, b):
    return a @ b                                  # multiply matching positions, add up

def cosine(a, b):
    return (a @ b) / (np.linalg.norm(a) * np.linalg.norm(b))

def euclidean(a, b):
    return np.linalg.norm(a - b)

q = np.array([4.0, 3.0])
A, B, C = np.array([8.0, 6.0]), np.array([3.0, 4.0]), np.array([12.0, 2.0])
[dot(q, v) for v in (A, B, C)]         # → [50, 24, 54]         C wins
[cosine(q, v) for v in (A, B, C)]      # → [1.0, 0.96, 0.888]   A wins
[euclidean(q, v) for v in (A, B, C)]   # → [5.0, 1.414, 8.062]  B wins

# Batched: one question against 1,000,000 normalized page vectors.
# V is (1_000_000, 768) float32 and already L2-normalized. query is the question's 768-d vector.
query  = query / np.linalg.norm(query)          # make the question length 1 too
scores = V @ query                              # one matrix-vector product: a million dot products
top_k  = np.argpartition(-scores, 10)[:10]      # a partial sort: much cheaper than fully sorting a million scores
top_k  = top_k[np.argsort(-scores[top_k])]      # the 10 come back unordered, so sort just those 10
```

In simple words, `V @ query` does a million dot products in one call, and `argpartition` pulls
out the best 10 without putting the other 999,990 in order.

That `V @ query` is the entire brute-force search, the kind that checks every single page. Its
speed is set by memory bandwidth, and a modern CPU streams tens of gigabytes per second. That is
exactly the right way to think about its cost: brute force is **memory-bandwidth-bound, not
compute-bound**. Means, the speed limit is how fast numbers can be moved from memory to the
processor, not how fast the processor multiplies. Chapter 17 does the arithmetic, and it is
faster than most people guess.

---

### What people get wrong

**Mixing conventions.** Cosine and dot: higher is better. Euclidean: lower is better. Every vector
database has a config option for this, and every team gets it backwards at least once.

**Assuming the library normalizes for you.** Some model wrappers do, some do not, and the same
model can behave differently across two libraries. Check with `np.linalg.norm(v)`. It should
print `1.0000...`. Do this once per model, forever.

**Comparing scores across models or across index types.** A 0.83 from one model and a 0.83 from
another are unrelated quantities. Approximate indexes can also return slightly different scores
than brute force for the same pair, due to **quantization** (storing each number in fewer bits
to save memory, Chapter 26).

**Treating cosine similarity as a probability.** It is not calibrated, not a percentage, and not
comparable between queries.

---

### Ninja notes

There is one situation where normalizing actively destroys information: when vector magnitude
encodes *confidence* or *importance*. Some recommender architectures deliberately give popular
items larger norms so that the dot product naturally favours them. Normalize those, and you have
thrown away your popularity prior and will wonder why engagement dropped.

Also worth knowing: MIPS over unnormalized vectors is genuinely harder to index than
nearest-neighbour search. The "best" vector may be far away in Euclidean terms but long enough to
win. Graph indexes like HNSW assume a metric space and can degrade on raw MIPS. The classic fix
is a transform that appends an extra dimension encoding the norm, converting MIPS into a
nearest-neighbour problem. This is another quiet argument for normalizing everything you can.

---

### Key takeaways

- **Dot product = multiply matching positions and add. Cosine = dot product divided by both
  lengths. Euclidean = straight-line gap.** Dot is agreement weighted by length, cosine is angle
  only.
- On our toy example the three rulers picked three different winners, and only cosine picked the
  text with the same meaning.
- On normalized vectors, all three rank identically, so normalize and use dot.
- Higher is better for dot and cosine. Lower is better for Euclidean.
- Cosine values are not percentages and are not comparable across models.
- Magnitude sometimes matters. Know whether it does in your system before erasing it.

### What's next

We have leaned on normalization twice now. [Chapter 5](./05-normalization.md) looks at exactly
what it does, why the unit sphere is such a convenient place to live, and the cases where it is
the wrong call.

Now we know the three rulers, how each one can mislead us, and why normalizing lets us stop
choosing.
