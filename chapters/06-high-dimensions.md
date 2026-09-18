---
title: "High Dimensions Are a Strange Country"
chapter: 6
part: "Part I — Foundations: What a Vector Actually Is"
slug: high-dimensions
readingTime: "11 min"
summary: "In 768 dimensions, almost everything is far away, almost everything is orthogonal, and distance stops being informative. Vector search works anyway — and the reason why is the most important idea in Part I."
tags: [foundations, curse-of-dimensionality, intuition, manifold]
prev: 05-normalization
next: 07-dense-vs-sparse
---

# High Dimensions Are a Strange Country

**The one-paragraph version.** As the number of dimensions grows, space grows so fast that any
amount of data becomes hopelessly spread out. All the distances between points drift toward the
same value, and "nearest neighbour" starts to lose its meaning. This is the **curse of
dimensionality**, and taken at face value it says vector search should not work. It works because
real embeddings do not fill their space. They lie on a thin, low-dimensional surface curled up
inside it. Understanding that rescue is what turns ANN (approximate nearest neighbour search,
Chapter 3) from magic into engineering.

In this chapter, we will learn why 768-dimensional space is so strange, and measure its
strangeness with a small experiment. We will watch our running question fail in a Map Room made
of random dots, then succeed in a real one. Then we will see why real data escapes the curse, and
how to measure how hard our own data will be to search.

We will cover the following:

- What is the curse of dimensionality
- Why high-dimensional space is so empty
- Distance concentration: when everything is equally far
- An example: our question in a room of random dots
- Why vector search works anyway
- What this means in practice

---

## What is the curse of dimensionality

In Chapter 1 we agreed to think in 3 dimensions, compute in 768, and stay alert for the few places
where high dimensions behave strangely. This chapter is about those few places.

**Curse of dimensionality = as dimensions grow, space becomes vast and empty, and distances stop
telling points apart.**

It has two symptoms, and we will look at each with real numbers:

1. **The space is empty.** Any amount of data is spread absurdly thin.
2. **Distances concentrate.** The nearest point and the farthest point end up almost equally far
   away.

After that, we will see why vector search escapes both.

---

## Why high-dimensional space is so empty

Think of it like building a hotel with one room for every possible kind of guest. We describe
guests by a few attributes, and we allow ten "slots" along each attribute.

- One attribute (age): 10 rooms.
- Two attributes (age, income): 10 × 10 = 100 rooms.
- Three: 1,000 rooms.
- Ten: 10 billion rooms.
- Twenty: 10²⁰ rooms, more than the grains of sand on Earth.
- Seven hundred and sixty-eight: a number with 769 digits, vastly more than the count of atoms in
  the observable universe (about 10⁸⁰).

In simple words, every extra attribute multiplies the number of rooms by 10.

Here is the mapping:

- The hotel is the Map Room.
- Each attribute is one dimension.
- Each room is one small region of the space.
- Each guest is one page's dot.

Now let's put Acme's 40,000 pages into the 768-attribute hotel, one guest per page. **Every page
gets a room to itself.** No two pages share a room. The nearest other guest is many rooms away,
and almost every room is empty.

That is the first half of the curse in one image. The volume of the space multiplies with every
dimension, so any fixed amount of data becomes absurdly sparse.

**Note:** "Sparse" here just means spread thin. It is not the "sparse vectors" of Chapter 7, which
are a different idea.

---

## Distance concentration: when everything is equally far

Emptiness is unsettling. This next one is genuinely dangerous.

**Distance concentration = in high dimensions, the nearest point and the farthest point end up
at almost the same distance.**

Let's measure it. We scatter 40,000 random dots, one per Acme page, inside a cube. We drop one
random question dot. Then we measure the distance from the question to its nearest dot and to its
farthest dot. We repeat this with the cube in 2, 10, 100 and 768 dimensions. (The code is in
Under the hood.)

| Dimensions | Nearest dot | Farthest dot | (farthest − nearest) ÷ nearest |
|---|---|---|---|
| 2 | 0.003 | 1.05 | about 380 |
| 10 | 0.34 | 2.15 | about 5.3 |
| 100 | 3.07 | 4.89 | about 0.59 |
| 768 | 10.28 | 12.20 | about 0.19 |

In 2 dimensions, the farthest dot is hundreds of times farther away than the nearest one. In 768
dimensions, it is only about 19% farther.

The formal statement is that for random data like this, as the number of dimensions grows:

$$ \frac{D_{\max} - D_{\min}}{D_{\min}} \to 0 $$

Here $D_{\max}$ is the distance to the farthest point and $D_{\min}$ the distance to the nearest.
The arrow means "shrinks toward". Let's check the 768 row: (12.20 − 10.28) ÷ 10.28 ≈ 0.19.

Means, the farthest point and the nearest point become *the same distance away*, relatively
speaking. And if the nearest neighbour is not meaningfully nearer than the farthest, then
"nearest neighbour" is not a meaningful question. No Card Catalog can make it one.

There is a matching fact on the sphere of unit vectors from Chapter 5. Two random unit vectors in
high dimensions are almost certainly near-orthogonal. In 768 dimensions the cosine similarity of a
random pair concentrates tightly around 0, with a standard deviation (typical spread) of roughly
$1/\sqrt{d} = 1/\sqrt{768} \approx 0.036$. Everything is at right angles to everything else.

---

## An example: our question in a room of random dots

Let's make this concrete with our running question.

Imagine a broken embedding model that puts Acme's 40,000 pages at random spots on the
768-dimensional sphere. It puts our question, *"Does the Pro plan include single sign-on?"*, at
another random spot.

Let's first see what the Librarian does in this room. It asks for the top 5. When we simulate it,
the best page scores a cosine of about 0.15 and the worst about −0.14. About 95% of all pages
crowd between −0.07 and 0.07. The "top 5" are simply five pages that happen to sit a hair closer
than the rest: perhaps a refund rule, a release note and an invoice template. The answer is
wrong. Worse, it is random. Ask a slightly different question and we get five different pages.

By this argument, embedding search is doomed. Except that it works spectacularly.

Now let's see what happens in a real Map Room built by a real model. The same question lands right
beside page 212, the Pro plan page, and page 88, the guide to setting up single sign-on. Their
scores stand clearly above the thousands of pages about billing and refunds, by a gap wide enough
to rank on. The Librarian brings back genuine single sign-on pages instead of a random handful. The
ranking means something again.

So something in the argument does not apply to real data. Let's find out what.

---

## Why vector search works anyway

The escape is simple to state and worth reading slowly.

**The curse applies to data spread evenly through the space. Real embeddings are not spread
evenly. They occupy a thin, curved, low-dimensional surface inside the high-dimensional space.**

Mathematicians call such a surface a **manifold**.

Think of it like a sheet of paper crumpled into a ball:

- The ball sits in three dimensions. That is the paper's **ambient dimension**, the space it sits
  in.
- The paper itself is two-dimensional. That is its **intrinsic dimension**, the number of
  directions we can actually move along the paper.
- An ant walking on the paper measures distances along the paper, and those distances stay
  meaningful even though the paper is tangled up in a bigger space.

**Intrinsic dimension = the number of directions the data actually varies along, however many
numbers each vector has.**

Embeddings are that crumpled sheet. They have 768 ambient dimensions and an intrinsic dimension
frequently measured in the tens. There is a reason for this. The sentences people actually write
are a vanishingly small part of all the sentences that could be written. Grammar, topic
coherence and the statistics of real language all constrain where points can land, and the model
has arranged the plausible ones on a surface.

The consequences follow immediately:

- Distances between *real* embeddings do **not** concentrate the way random points do. The
  nearest neighbour genuinely is nearer.
- The structure is **locally low-dimensional**, which is what algorithms exploit. A graph index
  (a Card Catalog that links each dot to its neighbours, Chapter 28) only needs to follow the
  surface, not explore the void.
- The curse is not defeated. It is **avoided**. Push far enough off the manifold, with random
  vectors, adversarial inputs (text crafted to fool a model) or content wildly unlike the training
  data, and the strange country comes back.

> **This is the intellectual foundation of Part IV.** Every ANN index is a bet that your data has
> exploitable low-dimensional structure. When an index underperforms its published benchmarks
> (standard test datasets with reported scores) on your data, the usual reason is that your data
> has less structure than the benchmark's. Often that is because it is templated, boilerplate or
> near-duplicate text that the encoder (the embedding model) cannot spread out.

---

## What this means in practice

**Intrinsic dimension predicts how hard the index will be.** Data with low intrinsic dimension
indexes beautifully. Data with high intrinsic dimension needs us to search a bigger slice of the
index to find the same true neighbours, so it costs more time and memory. Public benchmark
datasets are often easy. Real corpora are often harder.

**We can often cut dimensions for almost free.** If the intrinsic dimension is about 40, a
768-dimensional representation is largely redundant. Two tools can cut dimensions by half or
three-quarters with tiny quality loss. One is **PCA** (principal component analysis, a classical
method that re-orders dimensions by how much they vary). The better one is a Matryoshka-trained
model (Chapter 15), trained so that a shortened vector still works well. Memory and latency (the
time each search takes) drop in proportion. For Acme, cutting 768 dimensions to 192 shrinks each
vector from 3,072 bytes to 768, and the whole Library from about 123 MB to about 31 MB.

**Random projections work, and that is not obvious.** A random projection multiplies every vector
by the same table of random numbers, to get a shorter vector. A result called the
Johnson–Lindenstrauss lemma says this keeps all the distances roughly intact. The number of
dimensions it needs depends only on how many points we have and how much stretching we accept.
It never depends on how many dimensions we started with.

And the number grows very slowly with the number of points. Going from 40,000 points to 40
billion, a million times more, needs only about 2.3 times as many dimensions. This is the
theoretical licence behind LSH (Chapter 21, hashing similar vectors into the same bucket) and
behind MUVERA's fixed-dimensional encodings (Chapter 38).

In simple words, real data only uses a few of its many directions, and that is what every fast
search method in this book quietly relies on.

---

### Under the hood

**Experiment 1: distance concentration.** This produces the table from earlier in the chapter.

```python
import numpy as np

rng = np.random.default_rng(1)
for d in (2, 10, 100, 768):
    pages = rng.random((40_000, d), dtype=np.float32)     # 40,000 random dots in a cube
    question = rng.random(d, dtype=np.float32)            # one random question dot
    dist = np.linalg.norm(pages - question, axis=1)
    near, far = dist.min(), dist.max()
    print(d, round(float(near), 3), round(float(far), 3), round(float((far - near) / near), 2))
# → 2 0.003 1.048 379.56
#   10 0.341 2.152 5.31
#   100 3.071 4.886 0.59
#   768 10.283 12.2 0.19
```

Means, we measure one question against 40,000 random dots, and the gap between nearest and
farthest shrinks as the dimensions grow.

**Experiment 2: measure your own data's intrinsic dimension.** We use a two-nearest-neighbour
estimator. It takes a minute and tells us how hard the index will be.

**Step 1:** Take a random sample of up to 5,000 vectors, and drop exact duplicates. Two identical
vectors are 0 apart, and Step 4 would divide by that 0.

**Step 2:** Drop near-copies too, such as two pages that differ by one word. We remove every
vector whose nearest neighbour is closer than half the typical (median) nearest-neighbour
distance. Both vectors of such a pair go.

**Step 3:** For each vector that is left, find the distance to its nearest neighbour ($r_1$) and
to its second nearest ($r_2$).

**Step 4:** Compute the ratio $r_2 / r_1$ for each vector.

**Step 5:** Divide the number of vectors by the sum of the logs of those ratios. (The log, the
natural logarithm, turns big ratios into small numbers. Chapter 8 uses it again.)

In simple words, when data has few free directions, the second neighbour is noticeably farther
than the first. When it has many, the two are almost equally far, just as in the table above.

That is also why Step 2 matters. A near-copy's nearest neighbour is its twin, almost 0 away, so
its ratio is huge. Huge ratios say "few free directions", so a handful of near-copies drags the
whole reading far down.

```python
import numpy as np
from sklearn.neighbors import NearestNeighbors

def intrinsic_dim(V, sample=5000, near=0.5):
    X = V[np.random.choice(len(V), min(sample, len(V)), replace=False)]
    X = np.unique(X, axis=0)                  # Step 1: drop exact duplicates
    r1 = NearestNeighbors(n_neighbors=2).fit(X).kneighbors(X)[0][:, 1]
    X = X[r1 >= near * np.median(r1)]         # Step 2: drop both vectors of a too-close pair
    d, _ = NearestNeighbors(n_neighbors=3).fit(X).kneighbors(X)
    r1, r2 = d[:, 1], d[:, 2]                 # Step 3: 1st and 2nd NN distances
    return len(r1) / np.log(r2 / r1).sum()    # Steps 4-5: TWO-NN / MLE estimate
```

As a sanity check, on test data that secretly lies on a flat 10-dimensional sheet inside 768
dimensions, this function returns about 10. The estimate runs low as the true value grows. A
40-dimensional sheet reads about 30 with 5,000 samples, so treat it as a rough gauge.

Near-copies make a corpus harder to search, yet they make this reading lower. Replace 5% of the
40-dimensional sheet's points with near-copies of other points, a tenth of the typical gap away.
Without Step 2 the function reads about 4. With Step 2 it reads about 28, as the clean sheet
does. Step 2 only catches pairs closer than half the typical gap, though. Copies half the gap
away slip through, and the reading stays near 9.

Rough reading of the result, using this estimator, which reads low: **under ~15** is easy, and a
small index searched shallowly will do. On text, give such a reading a second look. Print the
closest pairs left in the sample. If they are near-copies, deduplicate the corpus (or raise
`near`) and measure again. **15–40** is typical for good text embeddings.
**Over ~40**, expect to pay for recall (how many of the true nearest neighbours the index actually
finds). **Over ~60**, also consider whether the encoder is a good fit for the corpus.

---

### What people get wrong

**"The curse means vector search doesn't work."** It means *uniform* high-dimensional search does
not work. Embeddings are not uniform, by construction.

**"More dimensions always helps."** Past the intrinsic dimension of your data, extra dimensions
add cost and noise. This is why 384-d models often match 1536-d ones on real retrieval tasks.

**"My ANN recall is bad, so the index library is broken."** Far more often, your data is harder
than the benchmark: near-duplicates, boilerplate, or a domain the encoder never saw. Look at the
nearest-neighbour similarity distribution first. A pile-up close to 1 means near-copies. Then
measure intrinsic dimension with the near-copies removed, since they fake a low reading, before
filing the issue.

---

### Ninja notes

In index-tuning terms: datasets with high intrinsic dimension need higher `efSearch` (HNSW,
Chapter 30), more probes (IVF's `nprobe`, Chapter 23), more memory. The same recall costs more.
Benchmarks like SIFT1M are easy. Some real corpora are much harder.

The precise Johnson–Lindenstrauss statement: projecting $n$ points into $O(\log n / \epsilon^2)$
dimensions preserves all pairwise distances within a factor of $1 \pm \epsilon$, *regardless of
the original dimension*. With the common Dasgupta–Gupta constants, 40,000 points need about 510
dimensions at $\epsilon = 0.5$ and about 2,450 at $\epsilon = 0.2$. The bound guarantees every
pair at once, so it is conservative. Note that for Acme the $\epsilon = 0.2$ figure is bigger
than the 768 we started with, and the $\epsilon = 0.5$ one lets distances drift by up to half. So
the bound alone does not justify shrinking our vectors. Its value is the theory behind LSH and
MUVERA. On real data, projections usually do much better than the bound, because the data has a
low intrinsic dimension and the bound plans for the worst case.

Distance concentration is precisely why **rescoring** is such a reliable pattern. Aggressive
compression, binary quantization for example, destroys fine distinctions. But it preserves coarse
structure well enough to shortlist a few hundred candidates out of millions. Then you rescore
those few hundred with full-precision vectors, where the surviving distance differences *are*
meaningful. The shortlist stage needs 32× less memory, and recall stays close to exact.

Retrieve coarsely at scale, refine precisely at small scale. That pattern recurs in Chapters 24,
26, 27, 32, 37 and 38. Once you see it, you see it everywhere.

---

### Key takeaways

- **Curse of dimensionality = as dimensions grow, space becomes vast and empty, and distances stop
  telling points apart.**
- In 768 dimensions, the farthest of 40,000 random dots is only about 19% farther than the
  nearest.
- That curse applies to evenly spread data. Real embeddings lie on a low-dimensional manifold, so
  distances stay meaningful.
- **Intrinsic dimension**, not the number of dimensions a vector has, predicts how hard your index
  will be.
- Random projections preserve distances, which licenses LSH and MUVERA.
- Coarse-retrieve-then-rescore is the master pattern for beating the curse in practice.

### What's next

We have treated embeddings as dense lists of floats. [Chapter 7](./07-dense-vs-sparse.md)
introduces the other kind of vector: mostly zeros, tens of thousands of dimensions wide, and still
running in production almost everywhere.

Now we know why high-dimensional space is strange, why real embeddings escape its curse, and how
to measure how hard our own data will be.
