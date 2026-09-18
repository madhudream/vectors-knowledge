---
title: "HNSW I: Small Worlds and Skip Lists"
chapter: 28
part: "Part IV — Index Structures"
slug: hnsw-1-small-worlds
readingTime: "11 min"
summary: "Six degrees of separation is not a coincidence — it is a property you can engineer. Combine it with the express-lane trick from skip lists and you get one of the best in-memory vector indexes we know how to build."
tags: [hnsw, small-world, graph, skip-list, intuition]
prev: 27-turboquant
next: 29-hnsw-2-building
---

# HNSW I: Small Worlds and Skip Lists

**The one-paragraph version.** HNSW stands for **Hierarchical Navigable Small World**. It rests
on two old ideas. First, a graph where most links are local but a few are long lets us reach any
node in a handful of hops, the "six degrees of separation" effect. Second, stacking sparse
express lanes on top of a dense local network lets us travel far quickly and then arrive
precisely. Put the two together and a simple greedy walk finds near neighbours in a number of
steps that grows roughly with the logarithm of the collection size.

In this chapter, we will learn about the two ideas inside HNSW, the graph index that dominates
in-memory vector search. We will also see how a greedy walk works, where it goes wrong, how
skip lists add express lanes, and why the layers make such a large difference.

We will cover the following:

- What is a small world
- What is greedy search
- Problems with greedy search
- What is a skip list
- What is HNSW
- An example of HNSW
- Why the hierarchy matters

---

## What is a small world

First, a quick reminder of what a graph is. A **graph** is a set of dots, called
**nodes**, joined by links, called **edges** (Chapter 20). Moving along one edge is a **hop**.
In our Library, every page is a node, and each page carries a slip listing a few similar pages.
Those listings are its edges.

In 1967, Stanley Milgram asked people in Nebraska to get a letter to a stranger in Boston. They
could only pass it to someone they knew personally. The letters that arrived took about six
hops.

Each of us knows a few hundred people, and most of them live nearby. So how does a letter cross a
continent in six steps?

The answer is that **almost everyone has a few unusual connections.** A cousin who moved
abroad. A former colleague in another industry. A university friend on the other coast. Most
links are local, but a handful are long, and those few links shrink the whole network.

**Small world = a graph where most edges are local, but a few are long-range.**

- The local edges give **high clustering**: our neighbours know each other. Once we are close,
  we can refine precisely.
- The long edges give **short paths**: we can get close from anywhere, fast.

A small world is **navigable** when a simple rule, "always hop to the neighbour closest to the
destination", reaches the target in a small number of hops. For a well-built flat graph of $n$
nodes, that number grows **polylogarithmically**. In simple words, it grows like a small power of
$\log n$, such as $(\log n)^2$. Doubling the collection adds only a few hops.

That is exactly what a search algorithm wants. No global map, no backtracking, no coordinates.
Just keep stepping closer.

---

## What is greedy search

**Greedy search = at every node, hop to whichever neighbour is closest to the query, and stop
when no neighbour is closer.**

"Greedy" means it always takes the best-looking step right now, without planning ahead.

**Step 1:** Start at an entry node.

**Step 2:** Look at the current node's neighbours.

**Step 3:** If any neighbour is closer to the query than the current node, hop to the closest
one.

**Step 4:** Repeat Steps 2 and 3.

**Step 5:** When no neighbour is closer, stop. This node is the answer.

```
start at some entry node
loop:
    look at the current node's neighbours
    if any neighbour is closer to the query than the current node:
        move there
    else:
        stop — you are at a local minimum
```

In the Great Library: we start at any page, read its slip, walk to whichever listed page is
closer to our question, and repeat. We never consult a catalogue. We never read a whole shelf.
We just keep moving toward the question.

This works astonishingly well. It also has two problems, which the rest of HNSW exists to solve.

---

## Problems with greedy search

**Problem 1: Local minima.** A **local minimum** is a node whose neighbours are all farther from
the query, even though it is not the true nearest node. Greedy search stops there and returns
the wrong page. The fix is **beam search** (Chapter 30). Beam search is a greedy walk that keeps
a shortlist of the best few candidates instead of just one. The shortlist's size is a setting
called `efSearch`. That lets it escape shallow traps.

**Problem 2: Slow starts.** If we begin on the far side of a large graph, and every edge is
short, we need many small hops to cross it. With a million nodes and only local edges, "greedy"
becomes a long walk. More hops also means more chances to hit a local minimum on the way.

The second problem is what the hierarchy solves. The solution was borrowed from a data structure
invented for a completely different purpose.

---

## What is a skip list

A **linked list** is a chain of items where each item points to the next one. To find an item in
a sorted linked list, we must walk the chain from the start, one item at a time.

**Skip list = a sorted linked list with extra express lanes stacked on top.**

```
Level 3:  1 ─────────────────────────────────► 50
Level 2:  1 ──────────► 20 ──────────────────► 50
Level 1:  1 ────► 10 ─► 20 ─────► 35 ────────► 50
Level 0:  1 ─► 5 ─► 10 ─► 15 ─► 20 ─► 25 ... ► 50
```

Level 0 holds every item. Each higher level holds a random subset of the level below. An item
appears in level $\ell$ with probability $p^\ell$, so higher levels are exponentially sparser.

Let's search for 35.

**Step 1:** Start at 1 on level 3. The next item is 50, which overshoots. Drop to level 2.

**Step 2:** From 1, hop to 20. That does not overshoot, so take it.

**Step 3:** From 20, the next item on level 2 is 50, which overshoots. Drop to level 1.

**Step 4:** From 20, hop to 35. Found.

Four steps, instead of walking through eight items on level 0. The expected search time is
$O(\log n)$. In simple words, the work grows with the logarithm of the list size. With
$p = 1/2$, a list a thousand times longer has only about ten more levels, since
$\log_2 1{,}000 \approx 10$, and each level adds just a couple of steps.

---

## What is HNSW

**HNSW = a skip list for vectors: a stack of small-world graphs, sparse with long edges at the
top, dense with short edges at the bottom.**

```
Layer 2:   few nodes, very long edges       ← cross the space in 2–3 hops
Layer 1:   more nodes, medium edges         ← narrow to a region
Layer 0:   every node, short local edges    ← find the exact neighbours
```

Let's break down the name. *Hierarchical*: the layers. *Navigable small world*: each layer is a
graph that greedy search can walk. Every node lives in layer 0. A random few also live in layer
1, and fewer still in layer 2, just like a skip list.

Searching HNSW takes five steps.

**Step 1:** Start at the single **entry point**, a node in the top layer.

**Step 2:** Run greedy search in this layer until no neighbour is closer.

**Step 3:** Drop to the layer below, starting from the node we just found.

**Step 4:** Repeat Steps 2 and 3 until we reach layer 0.

**Step 5:** In layer 0, run beam search from that node to collect the final neighbours.

Long-distance travel happens in the sparse upper layers, where each hop covers enormous ground.
Precision happens at the bottom. That is the entire architecture.

---

## An example of HNSW

The Librarian asks, *"Does the Pro plan include single sign-on?"* The best page is page 212.
Page 1,140, with the add-on rule, sits right beside it in the Map Room.

Let's first see what a flat graph does, with only short edges and one layer.

The walk starts at the entry node, page 30,015, "How to export invoices as PDF". It hops to an
invoice-settings page, then a payment-methods page, then a plan-pricing page. Each hop moves only
a short way, because every edge is short. After nine hops it reaches page 87, "Two-factor login
is included on the Basic plan".

Page 87's slip lists other two-factor pages and other Basic-plan pages. Every one of them is
farther from the question than page 87 itself. Greedy search stops.

The algorithm thinks like this: *"Nothing on this slip is closer. I must be there."* The
Librarian returns page 87. The Scholar says nothing useful about SSO on Pro. The answer is wrong.

Now, let's see what HNSW does.

**Step 1:** Start at the entry point in layer 2, page 5, "Acme product overview".

**Step 2:** Layer 2 has long edges. One hop reaches page 80, "Security and login overview", on
the far side of the Map Room. No layer-2 neighbour is closer, so drop down.

**Step 3:** Layer 1 has medium edges. One hop reaches page 88, "Setting up single sign-on". Drop
down.

**Step 4:** Layer 0 has short edges. One hop reaches page 212.

**Step 5:** Beam search at layer 0 keeps a shortlist, and page 1,140 sits on page 212's slip.
Both pages come back.

Three hops of real travel instead of nine, and the walk entered layer 0 already in the SSO
neighbourhood. It never wandered near the two-factor pages. The Scholar reads both pages and
answers, "Yes, but teams under 50 seats need the Security add-on." The answer is correct.

**Note:** The hierarchy does not remove local minima. It makes them much rarer, because the walk
starts close to the target and takes few steps at the bottom. The shortlist in Chapter 30
handles the traps that remain.

---

## Why the hierarchy matters

A flat navigable small-world graph, called **NSW**, was HNSW's predecessor. It works, but its
search cost grows faster. Reaching the right region takes many local hops, so the number of
hops grows polylogarithmically, with worse constants.

The hierarchy changes routing from "walk across the graph" to "descend through $O(\log n)$
layers, doing a small, roughly constant amount of work in each". That is the difference between
good and excellent. It is why HNSW displaced NSW almost as soon as it was published.

There is a second, subtler benefit. In the upper layers, the graph is sparse enough that its
long edges are *informative*. They connect genuinely distant regions rather than duplicating
short edges. Chapter 29's neighbour-selection rule is designed to keep this property. It is the
difference between an HNSW index that performs as advertised and one that does not.

| | Flat small-world graph (NSW) | HNSW |
|---|---|---|
| Layers | One | A few, growing with $\log n$, assigned at random |
| Where the walk starts | Anywhere, often far away | Top layer, then drops closer each layer |
| Hops as $n$ grows | Polylogarithmic | Roughly logarithmic |
| Risk of stopping early | Higher, more hops to get stuck on | Lower, starts near the target |
| Extra memory | Edges on one layer | Edges on every layer a node is in (small, Chapter 29) |

**Advantages of HNSW:** very fast, high-recall search, no training step, and new vectors can be
added at any time. **Disadvantages:** the vectors and the graph must sit in RAM, building takes
real time, and deleting nodes is awkward. Chapters 29 to 31 cover each of these.

We must use a **flat index** (Chapter 17) when the collection is small enough to scan. We must
use **HNSW** when we need fast in-memory search over millions of vectors. We must use **IVF-PQ**
(Chapter 25) or **DiskANN** (Chapter 32) when the vectors no longer fit in RAM.

---

### Under the hood

Greedy search in one layer is the primitive the whole algorithm is built from. It follows Steps
1 to 5 of greedy search above.

```python
def greedy_search_layer(graph, layer, entry, query, dist):
    current = entry
    current_d = dist(query, current)
    improved = True
    while improved:
        improved = False
        for neighbour in graph.neighbours(current, layer):
            d = dist(query, neighbour)
            if d < current_d:
                current, current_d, improved = neighbour, d, True
    return current            # local minimum in this layer
```

And the descent through layers, which is Steps 1 to 4 of the HNSW search:

```python
def search_entry_point(graph, query, dist):
    node = graph.entry_point
    for layer in range(graph.max_layer, 0, -1):        # top layer down to 1
        node = greedy_search_layer(graph, layer, node, query, dist)
    return node               # a good starting point for layer 0
```

In simple words, the first function walks downhill inside one layer, and the second calls it once
per layer on the way down. Two names recur: `graph.neighbours(node, layer)` reads a page's slip on
one layer, and `dist(query, node)` measures a distance. Chapters 29 and 30 use the same names, so
the code in all three chapters fits together. Inside the `for` loop, `current` keeps updating to the
closest neighbour seen so far, so each pass through a slip ends at the closest listed page, exactly
as in Step 3.

Layers above 0 use *single-candidate* greedy search: find the local minimum and drop. Only layer
0 uses the wider beam search of Chapter 30. This asymmetry is deliberate. The upper layers only
need to get us approximately right, and paying for precision there would be wasted.

---

### What people get wrong

**"The hierarchy stores different data at each level."** No. Every node in layer 2 also exists
in layers 1 and 0, with its own neighbour list at each. The layers are different *edge sets* over
overlapping node sets, not different data.

**"HNSW is a tree."** It is a graph with cycles, at every layer. There is no parent, no root and
no unique path.

**"Greedy search gets stuck constantly."** With a well-constructed graph and a candidate
shortlist, it rarely does. The construction rule in Chapter 29 exists precisely to make the graph
navigable enough that it does not.

**"More layers is better."** The number of layers comes from the level-assignment probability
and grows as $O(\log n)$ on its own. You do not set it directly, and the parameter that controls
it (`mL`, defined in Chapter 29) almost never needs tuning.

---

### Ninja notes

**Why "polylogarithmic" and not "logarithmic" for flat graphs.** Jon Kleinberg (2000) studied
greedy routing on a grid with random long-range links. It needs about $(\log n)^2$ steps, but
only when the long links follow the right distance distribution. With the wrong distribution,
greedy routing needs polynomially many steps, even though short paths still exist. Short paths
existing is not the same as a greedy walker finding them. HNSW's layers are an engineered answer
to that gap.

**The entry point is a shared hot spot.** Every query starts at the same top-layer node, and
the first few hops are shared by all of them. Reading one node from many threads at once is not,
by itself, a contention point, so this is not a proven bottleneck.

One option is to keep several entry points and pick one per query. That spreads the access
pattern and might help throughput under high concurrency, but the gain is not well established.
It can also add a little robustness, since a single entry point in an odd region of the space
biases the early hops of every search. Measure before relying on it.

A related point for Chapter 31: because everything starts at the top, **deleting the entry point
node** is a real event. Implementations must handle it explicitly, usually by promoting another
high-layer node. This is one of several reasons deletion in HNSW is harder than it looks.

---

### Key takeaways

- **HNSW = a skip list for vectors: a stack of small-world graphs, sparse with long edges at the
  top, dense with short edges at the bottom.**
- **Small world = mostly local edges plus a few long-range ones.** Greedy search can navigate it
  in a polylogarithmic number of hops.
- Greedy search's weaknesses are local minima and slow starts.
- HNSW borrows the skip list's express lanes: sparse upper layers for travel, a dense layer 0
  for precision. Search cost grows roughly with $\log n$.
- Upper layers use single-candidate greedy search. Only layer 0 uses a beam search shortlist.
- Every node exists at layer 0. Higher layers are progressively sparser subsets.

### What's next

[Chapter 29](./29-hnsw-2-building.md) builds the graph: the parameters `M` and
`efConstruction`, and the neighbour-selection rule that is the single most important detail in
the algorithm.

We now know what a small world is, how greedy search walks it, and why stacking express lanes on
top turns a good graph index into an excellent one.
