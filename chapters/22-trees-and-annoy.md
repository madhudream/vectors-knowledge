---
title: "Trees: KD-Trees and Annoy"
chapter: 22
part: "Part IV — Index Structures"
slug: trees-and-annoy
readingTime: "10 min"
summary: "Splitting space in half repeatedly is the textbook answer to nearest-neighbour search. It is also the clearest demonstration of what the curse of dimensionality does to an algorithm."
tags: [kd-tree, annoy, space-partitioning, curse-of-dimensionality]
prev: 21-lsh
next: 23-ivf
---

# Trees: KD-Trees and Annoy

**The one-paragraph version.** A KD-tree splits space with cuts along one axis at a time,
descends to the query's leaf, and backtracks to check nearby regions. In two dimensions it is
magnificent. Above roughly twenty dimensions the backtracking consumes the entire tree, and it
becomes slower than brute force. **Annoy** patches this with random cuts and many independent
trees. It was the workhorse of production recommendation systems for years, but graph indexes
have since overtaken it.

In this chapter, we will learn about the partition family from Chapter 20: indexes that cut the
Map Room into smaller and smaller rooms. We will see why the classic KD-tree collapses in high
dimensions, how Annoy rescued the idea, and where trees still earn their place.

We will cover the following:

- What is a KD-tree
- How a KD-tree searches
- The backtracking problem
- What is Annoy
- An example of Annoy on our SSO question
- Annoy's real advantage: memory-mapped files
- Trees vs graphs, and when to use which one

---

## What is a KD-tree

Picture a game of twenty questions about a point in a room. We may only ask yes/no
questions.

*"Is it in the left half?"* Yes. *"In the front half of that?"* No. *"In the upper half?"* Yes.
Three questions narrow the room to one-eighth of itself. Twenty questions isolate one part in a
million, because 2²⁰ = 1,048,576.

Here is the mapping. The room is the Map Room. Each question is a cut along one dimension. Each
answer sends us to one side of the cut. The small space left at the end is a **leaf**.

**KD-tree = k-dimensional tree = a tree that splits the data in half again and again, cutting
along one dimension at a time.**

**Note:** The "k" in KD-tree is the number of dimensions. It has nothing to do with the top-k of
Chapter 3.

We build it in five steps.

**Step 1:** Choose a dimension to cut on, for example dimension 0.
**Step 2:** Find the **median** on that dimension, the middle value, with half the points below
it and half above.
**Step 3:** Split the points into two groups at the median.
**Step 4:** Repeat Steps 1–3 inside each group, cutting on the next dimension.
**Step 5:** Stop when a group is small enough. That group becomes a leaf.

```
                 [all points]
              split on dim 0 at 0.5
              ╱                    ╲
      x₀ < 0.5                      x₀ ≥ 0.5
   split dim 1 at 0.3          split dim 1 at 0.7
      ╱      ╲                    ╱       ╲
   leaf      leaf               leaf      leaf
```

In simple words, every level of the tree halves the points, so a million points need only about
twenty levels.

---

## How a KD-tree searches

**Step 1:** Start at the top of the tree.
**Step 2:** At each cut, go to the side the query falls on.
**Step 3:** At the leaf, compare the query with the handful of points stored there.

Descending takes $O(\log n)$ steps. Means, the number of steps grows with the number of halvings,
not with the number of points. For a million points, that is about twenty decisions instead of a
million comparisons. Beautiful.

---

## The backtracking problem

Except it is not that simple, and the reason is the entire chapter.

The nearest neighbour **need not be in the query's leaf.** If the query sits near a cut, the true
nearest point may be just across it. So after reaching a leaf, we must backtrack.

**Step 4:** For every cut on the path back up, measure the distance from the query to the cut.
**Step 5:** If that distance is less than the distance to our best point so far, the other side
could hold something closer. Descend into it too.

In two dimensions we rarely backtrack, because most cuts are comfortably far away. In 100
dimensions, **almost every cut is closer to the query than the nearest neighbour is.**

This is Chapter 6 arriving in concrete form. With 100 dimensions and 20 levels, the tree has cut
on only a fifth of the dimensions. The other 80 were never used, and the data is spread thinly
across all of them. The nearest neighbour is far away in absolute terms. So almost every cut
triggers a backtrack, and we visit most of the tree.

The rule of thumb from the literature is blunt: **KD-trees beat brute force only when
$n \gg 2^d$.** In simple words, we need far more points than 2 multiplied by itself d times. At
d = 20, that is already far more than a million points. At d = 768, it is more points than there
are atoms in the universe.

Let's see what that means for Acme. We build an exact KD-tree over the Library's 768-dimensional
page vectors. Then we ask, *"Does the Pro plan include single sign-on?"*

The tree does return the true nearest neighbours, page 212 and the SSO chunk of page 1,140
among them (Chapter 18). Exact backtracking never
skips a region that could matter. But it visits nearly every leaf to get there. It is strictly
slower than scanning the array, and it uses more memory.

---

## What is Annoy

**Annoy = Approximate Nearest Neighbors Oh Yeah**, a library from Spotify that rescued the tree
idea with three changes.

**Change 1: Cut along the data, not the axes.** Instead of "is dimension 17 below 0.4?", pick
two random points from the data and cut along the sheet exactly halfway between them. These cuts
follow the data's actual shape rather than the arbitrary coordinate axes. That is a direct
response to the fact that single embedding dimensions mean nothing on their own (Chapter 1).

**Change 2: Do not backtrack exhaustively.** Explore the tree using a **priority queue**, a
to-do list that always hands back the most promising unexplored branch first (the one whose cut
is nearest the query). Stop after examining a fixed number of nodes. This turns an exact
algorithm into an approximate one and caps the cost of every query.

**Change 3: Build many trees.** Each tree uses different random cuts, so each makes different
mistakes. Search all of them and take the union of their candidates. A neighbour missed by one
tree's unlucky cut is usually caught by another. This is exactly the OR amplification of Chapter
21, in tree form.

Two knobs come out of this. `n_trees` is the quality knob: more trees give better recall, more
memory and a slower build. `search_k` controls how many nodes to inspect at query time.

In simple words, Annoy accepts that any single tree will sometimes cut in the wrong place, and
builds enough trees that the mistakes rarely line up.

---

## An example of Annoy on our SSO question

Our running question is *"Does the Pro plan include single sign-on?"* The answer needs page 212
(SSO is included on Pro) and page 1,140 (Pro teams under 50 seats need the Security add-on).

Let's first see what a single tree does. As it builds, one of its random cuts happens to fall
between page 212 and page 1,140. The two pages end up in different leaves. Our query descends to
the leaf holding page 212, and the capped search runs out of budget before it reaches the other
leaf.

The Scholar gets page 212 alone. It thinks: "SSO is included on Pro. Nothing mentions a
condition." It answers, "Yes, the Pro plan includes SSO." The answer is wrong for teams under 50
seats.

Now, let's see what a forest of 10 trees does. Each tree cut the Library differently. In most of
them, pages 212 and 1,140 sit in the same leaf, because they are close together and a random cut
rarely slices between close points. We search all 10 trees, take the union of their leaves, and
rescore the candidates exactly. Both pages come back.

The Scholar answers, "Yes, but teams under 50 seats need the Security add-on." The answer is
correct.

---

## Annoy's real advantage: memory-mapped files

Annoy's lasting contribution was not its recall curve. It was the file format.

Before we go further, we must know two operating-system terms.

- **Memory-mapping (mmap)** lets a program treat a file on disk as if it were already in RAM.
  The operating system loads each piece only when the program touches it.
- The **page cache** is the part of RAM where the operating system keeps recently read pieces of
  files. Every program on the machine shares it.

An Annoy index is a single flat file designed to be memory-mapped. Several consequences follow,
and they were transformative for the systems of their era:

- Multiple processes on a machine share **one** copy of the index in the page cache. Forty web
  workers, one index in RAM.
- The index does not need to fit in RAM. The operating system loads only what is touched.
- Startup is instant. There is no loading step and no rebuilding of the tree in memory.
- Deployment is copying a file.

For a recommendation service running many identical workers, this was enormously practical. It is
why Annoy survived long after its recall numbers were beaten.

The cost is rigidity: **an Annoy index is immutable.** We cannot add a vector. Every update means
a full rebuild and a file swap. For a recommendation index refreshed nightly, that is fine. For
Acme's Library, where pricing pages and release notes change every day, it hurts.

---

## Trees vs graphs, and when to use which one

| | KD-tree (exact) | Annoy | HNSW (graph) |
|---|---|---|---|
| Works well up to | ~20 dimensions | Hundreds of dimensions | Hundreds to thousands |
| Result | Exact | Approximate | Approximate |
| Speed and recall on embeddings | Exact, but slower than brute force | Good | Better |
| Add a vector | Possible, tree gets unbalanced | Rebuild the whole index | Incremental |
| Shared memory-mapped file | No | Yes | Usually not |

**Advantages of trees.** Simple to understand. Exact search in low dimensions. Annoy's
memory-mapped file is shared across processes, starts instantly, and can be larger than RAM.

**Disadvantages of trees.** Exact trees collapse above ~20 dimensions. Annoy cannot be updated
incrementally, and graph indexes reach better recall at the same speed.

We must use **a KD-tree** when the data truly has few dimensions: map coordinates, 3-D graphics,
nearest neighbours over a dozen table columns.

We must use **Annoy** when many processes must share one read-only index, and the data is rebuilt
on a schedule anyway.

We must use **a graph index** when we search embeddings that change over time and want the best
recall per millisecond. That is Acme's case.

---

### Under the hood

The random-cut tree at the heart of Annoy, plus a simple forest search:

```python
import numpy as np

class RPTree:
    def __init__(self, V, ids, leaf_size=32, rng=None):
        self.rng = rng or np.random.default_rng()
        self.root = self._build(V, ids, leaf_size)

    def _build(self, V, ids, leaf_size):
        if len(ids) <= leaf_size:
            return {"leaf": ids}
        # pick two random points; split on the plane bisecting them
        i, j = self.rng.choice(len(ids), 2, replace=False)
        normal = V[i] - V[j]
        offset = normal @ (V[i] + V[j]) / 2
        mask = (V @ normal) < offset
        if mask.all() or (~mask).all():                 # degenerate split
            return {"leaf": ids}
        return {"normal": normal, "offset": offset,
                "left":  self._build(V[mask],  ids[mask],  leaf_size),
                "right": self._build(V[~mask], ids[~mask], leaf_size)}

    def leaf_for(self, q):                              # descend, no backtracking
        node = self.root
        while "leaf" not in node:
            side = "left" if q @ node["normal"] < node["offset"] else "right"
            node = node[side]
        return node["leaf"]

def forest_search(trees, V, q, k=10):
    candidates = np.unique(np.concatenate([t.leaf_for(q) for t in trees]))
    scores = V[candidates] @ q                          # exact rescore of the union
    return candidates[np.argsort(-scores)[:k]]
```

`_build` is Change 1: the cut is the sheet halfway between two sampled points. `forest_search` is
Change 3: one leaf from every tree, pooled, then scored exactly. Real Annoy also does Change 2,
exploring beyond one leaf per tree with its priority queue until `search_k` nodes are seen. We
left that out to keep the code short. Here, `V` holds unit-length vectors and `ids` holds their
row numbers.

Note how the split direction comes from the *data* (the difference between two sampled points), not
from a coordinate axis or a purely random direction. That one change is what makes trees viable in
high dimensions at all. The cuts now lie along directions where the data actually varies.

---

### What people get wrong

**"KD-trees are the classic solution to nearest neighbour search."** They are the classic
solution to *low-dimensional* nearest neighbour search: geospatial queries, 3-D graphics, k-NN on
a dozen tabular features. For embeddings they are not viable.

**"Annoy is deprecated."** It is mature rather than dead. If you need memory-mapped, zero-copy,
multi-process indexes and your data is rebuilt on a schedule, it remains a clean,
dependency-light choice. It is simply no longer the default.

**"More trees is always better."** Recall saturates while memory grows linearly. Sweep it. The
knee is usually between 10 and 50 trees.

**"I can add vectors incrementally."** Not to Annoy. Plan for rebuilds.

---

### Ninja notes

Trees are not dead. They were absorbed.

The **IVF** index of Chapter 23 is a one-level tree: partition into `nlist` regions, search
`nprobe` of them. It works where deep trees fail precisely because it is shallow, so there is no
exponential backtracking. Its regions are also learned by k-means (a clustering method, defined
next chapter) rather than imposed by axis cuts.

**Multi-level k-means partitioning**, as in ScaNN, is a deeper version of the same idea. FAISS
(Meta's vector-search library) gets a similar effect by putting a small index over the cluster
centres. Both handle the *routing* step in billion-scale systems, where even comparing the query
with every one of the `nlist` cluster centres is too slow.

And in the graph world, the **hierarchy** in HNSW (Chapter 28) plays a tree-like role. The sparse
upper layers route the search to roughly the right region, and the dense bottom layer does the
fine search. That is the same coarse-to-fine decomposition a tree provides, implemented with
edges instead of cuts. It works because the coarse step is allowed to be approximate.

**The idea survived. The data structure did not.**

---

### Key takeaways

- **KD-tree = a tree that splits the data in half again and again, one dimension at a time.** It
  descends in $O(\log n)$ steps and is excellent under ~20 dimensions.
- Backtracking is the killer. In high dimensions nearly every cut must be checked, so we visit
  the whole tree. KD-trees beat brute force only when $n \gg 2^d$.
- **Annoy** makes trees work with data-driven random cuts, a capped search and many trees. One
  unlucky cut lost page 1,140. A forest found it.
- Annoy's real edge is its memory-mapped, immutable file: shared across processes, instant
  startup, larger than RAM.
- The index cannot be updated incrementally. Rebuild and swap.
- Tree ideas live on in IVF's partitioning and HNSW's coarse-to-fine hierarchy.

### What's next

[Chapter 23](./23-ivf.md) takes the shallow-partition idea seriously. It learns the regions from
the data with k-means and produces the index that still serves most of the world's billion-scale
vector search.

We now know why cutting space in half works in two dimensions, why it collapses in 768, and how
Annoy's random forest kept the idea alive.
