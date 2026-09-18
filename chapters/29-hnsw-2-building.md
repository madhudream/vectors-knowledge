---
title: "HNSW II: Building the Graph"
chapter: 29
part: "Part IV — Index Structures"
slug: hnsw-2-building
readingTime: "12 min"
summary: "M, efConstruction, and the neighbour-selection heuristic that decides whether your graph is navigable or merely connected. This is where HNSW is won or lost."
tags: [hnsw, graph-construction, pruning, heuristic, parameters]
prev: 28-hnsw-1-small-worlds
next: 30-hnsw-3-searching
---

# HNSW II: Building the Graph

**The one-paragraph version.** We build an HNSW graph by inserting nodes one at a time. Each new
node rolls a random top layer, where most nodes stay at layer 0 and only a few reach higher. For
each layer it belongs to, we search the existing graph for its nearest neighbours and link it to
`M` of them. The crucial detail is *which* `M`. Not simply the closest, but a **diverse** set,
chosen by a pruning rule that keeps links pointing in different directions. Get that rule wrong
and the graph becomes a set of tight cliques that greedy search cannot travel between.

In this chapter, we will learn how an HNSW graph is built, one insert at a time. We will also
see the neighbour-selection heuristic that makes the graph navigable, the two parameters we
actually set, and how much memory the graph really costs.

We will cover the following:

- What an insert has to do
- How HNSW inserts a node
- What is the neighbour-selection heuristic
- An example of the heuristic
- The two parameters we set: M and efConstruction
- How much memory the graph uses

---

## What an insert has to do

A quick reminder from Chapter 28. HNSW is a stack of graph layers. Layer 0 holds every node with
short edges. Higher layers hold fewer and fewer nodes, with longer edges. A search starts at the
top **entry point**, walks greedily down through the layers, and finishes at layer 0 with a
beam search, a walk that keeps a shortlist of the best candidates.

Now, the question is, how does a new page from Acme's Library get into this structure?

An insert has three jobs.

- **Decide how high the node lives.** Which layers get a copy of it?
- **Find its neighbours.** On each of those layers, which existing nodes are near it?
- **Choose which of them to link.** It may keep at most a fixed number of edges per layer.

The first two jobs reuse ideas we already have: a random roll, like a skip list, and a search,
like a query. The third job is where HNSW is won or lost.

Before the steps, let's define the two numbers that control an insert.

**M = the maximum number of neighbours a node keeps on each upper layer.** Layer 0 usually gets
twice as many, $M_0 = 2M$, because it does the precise final work.

**efConstruction = the shortlist size used while inserting.** It is how many candidate
neighbours the insert's search collects before choosing `M` of them.

---

## How HNSW inserts a node

**Step 1: Roll the node's top layer.**

$$ \ell = \lfloor -\ln(\text{uniform}(0,1)) \cdot m_L \rfloor $$

In simple words, we draw a random number between 0 and 1, take its negative natural log
(Chapter 8), scale it by $m_L$, and round down. The result is an **exponentially decaying
distribution**: most nodes get $\ell = 0$, and each higher layer is much rarer.

The standard choice is $m_L = 1/\ln(M)$. Then each layer holds roughly $1/M$ of the nodes below
it, exactly the skip-list shape from Chapter 28. With `M = 16`, $m_L \approx 0.36$. Drawing 0.5
gives $0.69 \times 0.36 = 0.25$, so layer 0. Drawing 0.03 gives $3.51 \times 0.36 = 1.26$, so
layer 1.

For Acme's 40,000 pages at `M = 16`, that means about 2,500 pages reach layer 1, about 156 reach
layer 2, and about 10 reach layer 3.

**Step 2: Start at the entry point** on the top layer.

**Step 3: Descend to the node's top layer without making edges.** On every layer above $\ell$,
run single-candidate greedy search (Chapter 28) to get closer to the new node. Take the node we
find down to the next layer.

**Step 4: Collect candidates.** On layer $\ell$, run a shortlist search with shortlist size
`efConstruction`. This finds up to `efConstruction` existing nodes near the new one.

**Step 5: Choose neighbours.** Pick at most `M` of the candidates ($M_0$ on layer 0) using the
neighbour-selection heuristic, explained in the next section.

**Step 6: Link both ways.** Add an edge from the new node to each chosen neighbour, and from each
neighbour back to the new node.

**Step 7: Prune overfull neighbours.** If a neighbour now has more edges than its limit (`M`, or
$M_0$ on layer 0), re-run the heuristic on *its* list and drop the extras.

**Step 8: Move down a layer.** Use this layer's candidates as the starting points, and repeat
Steps 4 to 7 until layer 0 is done.

**Step 9: Promote if needed.** If the new node's top layer is higher than the graph's, it becomes
the new entry point.

Step 7 is easy to overlook and matters enormously. Edges go both ways, so every insert adds an
edge to each of its neighbours. Without pruning, popular nodes would collect edges without
limit. Pruning keeps every list bounded. And because it re-applies the heuristic, it keeps every
list diverse too.

---

## What is the neighbour-selection heuristic

**Neighbour-selection heuristic = consider candidates from nearest to farthest, and accept a
candidate only if it is closer to the new node than to every neighbour already accepted.**

People also call it the **diversity rule**. It is the single most important implementation
detail in HNSW, and it is what most simplified explanations leave out.

Imagine building a social network where each new member may make five connections.

- The members are the nodes.
- A connection is an edge.
- "Similar interests" is closeness in the Map Room.

If every new jazz drummer connects to the five most similar people, they connect to five other
jazz drummers. Repeat a thousand times, and the network is a set of tight cliques with no bridges.
From jazz to marine biology, there is no path at all.

The better strategy is to connect to people who are close, but who are also different from each
other: a drummer, and also a sound engineer who works in architecture. That second link is a
bridge. Bridges are what make a network navigable.

In simple words, the heuristic says: "Do not add a neighbour I can already reach through a
neighbour I have."

The rejected candidates are not orphaned. They are reachable through the neighbour that *was*
accepted. The rule removes redundancy, not connectivity.

---

## An example of the heuristic

Let's insert a new page into Acme's Library, the "Plan price list". We place it at `(0, 0)`
on a toy 2-D map, and set `M = 3`. The shortlist search found five candidates.

| Candidate | Map position | Distance to new page |
|---|---|---|
| A: Pro plan storage limits | (1.0, 0.0) | 1.00 |
| C: Pro plan pricing | (1.1, -0.4) | 1.17 |
| B: Pro plan API limits | (1.2, 0.3) | 1.24 |
| D: Page 212, SSO included on Pro | (-0.2, 1.5) | 1.51 |
| E: Billing FAQ | (-1.6, -0.5) | 1.68 |

A, B and C sit tightly together to the east. D sits to the north. E sits to the south-west.

**Naive selection** takes the three closest: A, C and B. All three point east.

Now let's run the heuristic, nearest candidate first.

**Step 1:** A is first. Nothing is accepted yet, so accept A.

**Step 2:** C is 1.17 from the new page but only 0.41 from A. It is closer to A, so reject it.

**Step 3:** B is 1.24 from the new page but only 0.36 from A. Reject it.

**Step 4:** D is 1.51 from the new page and 1.92 from A. It is closer to the new page, so accept
it.

**Step 5:** E is 1.68 from the new page, 2.65 from A and 2.44 from D. Accept it.

The heuristic keeps **A, D and E**, one edge east, one north and one south-west.

Why does this matter? Let's walk a query through each graph. The Librarian asks, *"Does the Pro
plan include single sign-on?"* Its toy position is `(-0.1, 1.6)`, right beside page 212. Say
the walk has arrived at the Plan price list, which is 1.60 from the question.

**With naive selection**, its slip lists A, C and B, which are 1.94, 2.33 and 1.84 from the
question. Every one is farther than where we stand. Greedy search stops at the Plan price list.

The algorithm thinks like this: *"No neighbour is closer. This must be the best page."* The
Scholar gets a list of plan prices that never mentions SSO, and cannot answer the question.

**With the heuristic**, the slip also lists page 212, which is only 0.14 from the question. The
walk hops there, and page 1,140 sits on page 212's own slip. The Scholar gets both pages and
gives the full answer, add-on rule included.

This is what turns a graph that is merely *connected* into one that is **navigable**. In the
HNSW paper's experiments, the heuristic helped most on clustered and lower-dimensional data,
which is exactly where cliques form. Real text collections are full of clusters.

---

## The two parameters we set: M and efConstruction

**`M`, the maximum neighbours per node** (with $M_0 = 2M$ on layer 0).

| `M` | Effect |
|---|---|
| 4–8 | Small memory, lower recall ceiling; fine for low-dimensional or easy data |
| **12–16** | **Standard default. Good recall for most text embeddings** |
| 32–48 | Higher recall ceiling, ~2–3× the graph memory, slower build |
| 64+ | Diminishing returns; consider whether your data is unusually hard |

`M` sets the *ceiling* on achievable recall. No amount of `efSearch` (Chapter 30) will rescue a
graph built with `M = 4` on data with a high intrinsic dimension (Chapter 6), because the needed
edges do not exist.

**`efConstruction`, the shortlist size used while inserting.**

| `efConstruction` | Effect |
|---|---|
| 40 | Fast build, mediocre graph |
| **100–200** | **Standard. Good quality/time balance** |
| 400–800 | Better graph, 2–4× build time; worth it for a static index |

A higher `efConstruction` means each node sees more candidates before choosing its neighbours.
The heuristic then has better material to work with. It costs **build time only**, with zero
query-time cost and zero extra memory. For an index we build once and query billions of times,
being generous here is close to free.

> **The asymmetry to remember:** `M` costs memory forever. `efConstruction` costs build time
> once. `efSearch` (Chapter 30) costs latency per query. Three parameters, three different
> budgets.

We must raise **`M`** when recall stops improving no matter how high `efSearch` goes. We must
raise **`efConstruction`** when the index is built rarely and queried constantly. For most text
embeddings, `M = 16` and `efConstruction = 200` are the right place to start.

---

## How much memory the graph uses

Each edge is a neighbour's ID, stored as a 4-byte integer.

```
per node, layer 0:      M₀ × 4 bytes  (neighbour IDs, int32)  = 2M × 4
per node, upper layers: M  × 4 bytes × expected upper layers  = M × 4 × 1/(M-1)
vector:                 d × 4 bytes
```

Why $1/(M-1)$? A node reaches layer 1 with probability $1/M$, layer 2 with $1/M^2$, and so on.
Adding those up gives $1/(M-1)$ upper layers per node on average. In simple words, a typical
node spends almost all of its edge budget on layer 0.

For 768-d `float32` vectors with `M = 16`:

```
vector:         768 × 4        = 3,072 B
layer-0 edges:   32 × 4        =   128 B
upper edges:    16 × 4 / 15    ≈   4.3 B
overhead/IDs:                   ≈    20 B
                          total ≈ 3,224 B   (~1.05× raw)
```

For 10 million vectors, that is **~32 GB**. For a hundred million, **~320 GB**, which is when the
conversation turns to int8 (Chapter 26) or DiskANN (Chapter 32).

**Note:** For high-dimensional vectors, the graph overhead is modest, about 5–10%, because the
vectors dominate. The often-quoted "HNSW uses 1.5–2× memory" applies to *low*-dimensional vectors
with a larger `M`. Take 128-d vectors (512 B each) with `M = 32`. Layer-0 edges alone are
64 × 4 = 256 B. With upper edges and overhead, a node takes about 792 B, roughly 1.55× the raw
vector. At 96 dimensions it is about 1.7×. Know which regime you are in before sizing a machine.

---

### Under the hood

Here is the neighbour-selection heuristic, and the insert, condensed. The comments match Steps 1
to 9. `greedy_search_layer` is the single-candidate walk from Chapter 28. `search_layer` is the
shortlist search from Chapter 30, which returns `(distance, node)` pairs, best first. Both take
the same `graph`, `layer`, `query` and `dist` arguments as Chapter 28's code. Here `dist` takes
two node IDs, because the query is the node being inserted.

```python
import math, random

def select_neighbours_heuristic(q, candidates, M, dist):
    selected = []
    for c in sorted(candidates, key=lambda x: dist(q, x)):
        if len(selected) >= M:
            break
        # accept c only if it is closer to q than to every neighbour already chosen
        if all(dist(q, c) < dist(c, s) for s in selected):
            selected.append(c)
    return selected

def insert(graph, q, M, efc, mL, dist):
    M0 = 2 * M
    level = int(-math.log(1.0 - random.random()) * mL)      # Step 1: roll the top layer
    graph.add_node(q, level)                                 #   empty slips on 0..level
    if graph.entry_point is None:                            # the very first node
        graph.max_layer, graph.entry_point = level, q
        return
    ep = graph.entry_point                                   # Step 2

    for lc in range(graph.max_layer, level, -1):             # Step 3: descend, no edges
        ep = greedy_search_layer(graph, lc, ep, q, dist)

    entry_points = [ep]
    for lc in range(min(level, graph.max_layer), -1, -1):
        W = search_layer(graph, lc, entry_points, q, dist, efc)   # Step 4: shortlist of
        W = [node for _, node in W]                               #   efConstruction nodes
        m = M0 if lc == 0 else M
        neighbours = select_neighbours_heuristic(q, W, m, dist)   # Step 5
        graph.connect(q, neighbours, lc)                          # Step 6: both directions
        for n in neighbours:                                      # Step 7: prune
            if graph.degree(n, lc) > m:
                kept = select_neighbours_heuristic(n, graph.neighbours(n, lc), m, dist)
                graph.set_neighbours(n, kept, lc)
        entry_points = W                                          # Step 8: next layer down

    if level > graph.max_layer:                                   # Step 9: new top node
        graph.max_layer, graph.entry_point = level, q
```

`1.0 - random.random()` gives a number in (0, 1], so the log never sees zero. In simple words,
every insert is a search followed by a careful choice of links.

Two things to notice. Construction *uses search*: we cannot build the graph without searching
it, which is why build time grows with `efConstruction`. And the pruning step re-applies the
heuristic to existing nodes, so the diversity property is maintained all the time, not only at
insertion.

---

### What people get wrong

**Using naive top-`M` selection.** If you implement HNSW yourself and skip the heuristic, your
recall will be poor and you will blame the algorithm. Every production library implements it.

**Setting `efConstruction` low to speed up builds.** You are permanently degrading an index you
will query millions of times to save minutes once.

**Raising `M` when `efSearch` would do.** `M` costs memory forever. Try raising `efSearch` first.
It costs no memory, only latency, and it keeps helping until you hit `M`'s ceiling.

**Inserting in sorted order.** Inserting a corpus sorted by topic or time creates a graph whose
early structure reflects that ordering, and whose entry point sits in a corner of the space.
**Shuffle before building.** This is a real, measurable effect and a one-line fix.

**Expecting deterministic builds.** Layer assignment is random. Two builds of the same data give
different graphs with slightly different recall. Seed it if you need reproducibility.

---

### Ninja notes

**Build time is the hidden cost.** HNSW construction is roughly $O(n \log n)$ distance
computations with a large constant, because each insertion is itself a search. For 100 million
vectors with `efConstruction = 200`, expect hours on a many-core machine. Plan for it in your
re-indexing budget (Chapter 59). It is the number that decides whether a model migration takes
an afternoon or a week.

**Parallel construction is subtle.** Inserts change shared neighbour lists, so implementations
use per-node locks or lock-free updates. Most libraries handle this. If you are building your
own, it is the hardest part. A common pragmatic approach is to shard the corpus, build
independent graphs in parallel, and query them as a set, which is Chapter 58's territory.

**Consider `efConstruction ≈ efSearch × 2` as a starting point.** The graph should be built with
at least as much care as you intend to search it with. If you plan to serve at
`efSearch = 128`, building at `efConstruction = 200–256` is reasonable. Building at 40 leaves
recall on the table that no query-time setting can recover.

---

### Key takeaways

- **Neighbour-selection heuristic = accept a candidate only if it is closer to the new node than
  to every neighbour already accepted.** It creates bridges and prevents cliques.
- Each node gets a random top layer from an exponentially decaying distribution. With
  $m_L = 1/\ln M$, each layer holds about $1/M$ of the one below.
- An insert descends without edges, collects `efConstruction` candidates per layer, links to `M`
  of them both ways, and prunes overfull neighbours.
- **`M` = maximum neighbours per node.** It costs memory forever and sets the recall ceiling.
- **`efConstruction` = the shortlist size while inserting.** It costs build time once.
- Defaults: `M = 16`, `efConstruction = 200`. Shuffle your data before building.
- At 768-d and `M = 16`, the graph adds about 5% (≈3,224 B per node). "1.5–2×" is a
  low-dimensional, large-`M` figure.
- Build time is $O(n \log n)$ distance computations. Budget hours for 100M vectors.

### What's next

The graph exists. [Chapter 30](./30-hnsw-3-searching.md) queries it: beam search, `efSearch`,
and how to choose a point on the recall curve on purpose.

We now know how HNSW inserts a node, why the diversity rule makes the graph navigable, and what
`M` and `efConstruction` cost in memory and build time.
