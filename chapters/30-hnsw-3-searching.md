---
title: "HNSW III: Searching and Tuning"
chapter: 30
part: "Part IV — Index Structures"
slug: hnsw-3-searching
readingTime: "14 min"
summary: "Beam search with one runtime knob. How efSearch works, how to read your recall curve, and how to choose a point on it deliberately rather than by superstition."
tags: [hnsw, beam-search, efsearch, tuning, recall-curve]
prev: 29-hnsw-2-building
next: 31-hnsw-production
---

# HNSW III: Searching and Tuning

**The one-paragraph version.** An HNSW search drops through the upper layers of the graph
greedily. Then it walks the bottom layer with a **beam search**, keeping a shortlist of the best
`efSearch` pages found so far instead of just one. A longer shortlist explores more of the
graph, escapes more dead ends and finds more of the true neighbours, at the price of more time.
`efSearch` is set at query time, so we can change it per query, per customer, or under heavy
load, without rebuilding anything.

In this chapter, we will learn how HNSW actually answers a question, what the `efSearch` knob
does, and how to pick its value on purpose. We will also see where a graph search spends its
time, and why that surprises people.

We will cover the following:

- What is efSearch
- Why we need a shortlist
- How HNSW search works
- An example of HNSW search
- How to choose efSearch
- Where the time actually goes
- Small efSearch vs large efSearch
- When to use which one

---

## What is efSearch

Let's recall two things from the last two chapters.

**HNSW** is a graph of pages stacked in layers. The top layers are sparse and let us jump
across the Map Room in a few hops. The bottom layer, layer 0, holds every page, each linked to
its nearest neighbours. Chapter 29 built it, with `M` neighbours per page.

**Greedy search** (Chapter 28) always moves to the neighbour closest to the question, and stops
when no neighbour is closer. That stopping point is called a **local minimum**: better than
everything next to it, but not necessarily the best in the whole graph.

Now the new idea.

**Beam search = a graph walk that keeps a shortlist of the best few candidates found so far,
instead of only the single best one.**

**efSearch = the length of that shortlist while we walk layer 0.**

In simple words, `efSearch` is how many "maybe" pages the Librarian keeps in hand while walking
through the Card Catalog. Greedy search is beam search with a shortlist of one.

The name comes from the HNSW paper, where `ef` stands for the size of the dynamic candidate
list. Chapter 29's `efConstruction` is the same shortlist, used while building. `efSearch` is
the one used while searching.

---

## Why we need a shortlist

Let's take our running example. The Great Library is Acme's support knowledge base, 40,000
pages, indexed with HNSW. A user asks:

> *"Does the Pro plan include single sign-on?"*

A complete answer needs two pages. Page 212 says "SSO is included on the Pro plan." Page 1,140
adds the catch: "On Pro, SSO needs the Security add-on for teams under 50 seats."

Start with a plain greedy walk. It begins at a billing FAQ page. It hops to the Pro plan
overview page, then to page 212. Page 212 is very close to the question. None of its neighbours
is closer. The walk stops.

The only road to page 1,140 runs through page 530, an old "Security settings" page. Page 530
looks *less* relevant than the pages the walk already holds. So the greedy walk never steps
onto it, and never sees page 1,140.

The Scholar gets page 212 alone and writes: "Yes, SSO is included on Pro." The answer is wrong
for every team under 50 seats.

So how do we let the walk take a step that looks worse, in case it leads somewhere better?

The answer is a shortlist. Think of it like house-hunting by asking homeowners to recommend
nearby houses:

- The houses are the pages.
- Each homeowner's recommendations are that page's neighbours in the graph.
- Greedy search visits one house, takes the single best recommendation, and never looks back.
- Beam search keeps a list of the 40 most promising houses heard about so far, always visits
  the best one not yet visited, and adds its recommendations to the list.
- The search stops when the best unvisited house is worse than the worst house already on the
  shortlist.

With a shortlist, page 530 gets a place, because there is room for it. Visiting it reveals page
1,140. That is it. That is the whole trick.

---

## How HNSW search works

The search has two phases.

**Phase 1: Drop through the upper layers.**

- **Step 1:** Start at the graph's entry point in the top layer.
- **Step 2:** Walk greedily to the closest page in this layer (Chapter 28).
- **Step 3:** Drop to the layer below, starting from that page.
- **Step 4:** Repeat until we reach layer 0.

**Phase 2: Beam search on layer 0.** We keep two lists. "To explore" holds pages we have heard
about but not yet visited. "Best so far" holds the `efSearch` closest pages found.

- **Step 1:** Put the page from Phase 1 into both lists.
- **Step 2:** Take the closest page out of "to explore".
- **Step 3:** If that page is farther than the worst page in "best so far", and "best so far"
  is already full, stop.
- **Step 4:** Otherwise, look at each of its neighbours that we have not visited yet, and
  measure its distance to the question.
- **Step 5:** If a neighbour is closer than the worst page in "best so far", or the list is not
  full yet, add it to both lists.
- **Step 6:** If "best so far" now holds more than `efSearch` pages, drop the worst one.
- **Step 7:** Go back to Step 2.
- **Step 8:** When we stop, return the top `k` pages of "best so far".

Step 3 is the clever part. Every page in "to explore" is at least as far as the one we just
took out. If even that one is worse than everything on a full shortlist, none of the waiting
pages can enter the shortlist themselves, so we stop. The search ends when the shortlist stops
improving, not after a fixed number of steps.

This stopping rule is a bet, and Step 5 makes the same bet each time it skips a neighbour. A
worse page can still lead to a better one. In the example below, the answer is lost at Step 5,
not at the stop. The search skips page 530, the only road to page 1,140. A longer shortlist makes
both bets less often.

In simple words, the search keeps walking while the waiting pages could still join the
shortlist, and gives up when they cannot. A longer shortlist gives up later.

**Note:** `efSearch` must be at least `k`. We cannot return 10 pages from a shortlist of 4.
Some libraries quietly raise `efSearch` to `k` if we set it lower. Others return fewer results.

---

## An example of HNSW search

Let's run Phase 2 on a tiny piece of Acme's layer 0. These are toy numbers. Smaller distance
means closer to the SSO question. We ask for `k = 2` pages.

| Page | Topic | Distance | Linked to |
|---|---|---|---|
| 40 | Billing FAQ | 0.90 | 45, 77 |
| 77 | Invoices | 0.95 | 40 |
| 45 | Pro plan overview | 0.60 | 40, 212, 530 |
| 212 | SSO on Pro | 0.20 | 45 |
| 530 | Security settings (old) | 0.70 | 45, 1,140 |
| 1,140 | Security add-on | 0.25 | 530 |

Phase 1 dropped us at page 40.

Let's first see what happens with `efSearch = 2`.

- Visit page 40. Its neighbours are 45 (0.60) and 77 (0.95). The shortlist has room for 45.
  Now it is full: [40, 45]. Page 77 is worse than its worst (0.90), so we skip 77.
- Visit page 45. Neighbour 212 (0.20) is better than the worst, so it goes in and page 40 drops
  out. Shortlist: [212, 45].
- Neighbour 530 (0.70) is worse than the worst on the shortlist (0.60). We skip it.
- Visit page 212. It has no unvisited neighbours. Nothing is left to explore. We stop.

The search thinks like this: *"Page 530 scores 0.70. My shortlist is full, and its worst page
scores 0.60. Page 530 cannot help me."* It returns pages 212 and 45. Page 1,140 was never seen.
The answer is wrong.

Now, let's see what happens with `efSearch = 4`.

- Visit page 40. Both 45 and 77 fit. Shortlist: [45, 40, 77].
- Visit page 45. Page 212 fits and fills the list. Page 530 (0.70) beats the worst (77, at
  0.95), so 530 goes in and 77 drops out. Shortlist: [212, 45, 530, 40].
- Visit page 212. Nothing new.
- Visit page 530. Its neighbour 1,140 (0.25) beats the worst (40, at 0.90). In it goes.
  Shortlist: [212, 1,140, 45, 530].
- Visit page 1,140. Nothing new.
- The only page left to explore is 77, at 0.95. That is worse than the worst on a full
  shortlist (0.70). We stop.

This time the search thinks: *"Page 530 is not great, but I have room. Keep it and look."* It
returns pages 212 and 1,140. The Scholar reads both and answers: "SSO is on Pro, but teams under
50 seats need the Security add-on." The answer is correct.

The graph did not change. Only the shortlist did.

---

## How to choose efSearch

`efSearch` is the only parameter we tune at query time. Here is a typical recall curve.

| `efSearch` | Recall@10 (typical, 768-d text) | Relative latency |
|---|---|---|
| 10 | 0.70 | 1.0× |
| 32 | 0.90 | 1.8× |
| 64 | 0.95 | 2.8× |
| 128 | 0.98 | 4.5× |
| 256 | 0.99 | 7.5× |
| 512 | 0.995 | 13× |

This is Chapter 18's recall–latency curve again. From 32 upward, doubling `efSearch` roughly
halves the remaining error (0.10, 0.05, 0.02, 0.01, 0.005). Latency grows about 1.6–1.7× each
time `efSearch` doubles.

Means, each doubling buys half as much improvement as the last one, for more than half again the
time. **Diminishing returns are not a flaw. They are the shape of the problem.** Knowing the
shape lets us choose on purpose.

Four properties make `efSearch` pleasant to work with:

- **Runtime, not build time.** We change it per request. No reindex.
- **Monotonic in practice.** A higher value does not make recall worse. There is no strange
  region where turning it up hurts.
- **Per-query tunable.** High-value queries can get 256. Autocomplete can get 32.
- **A load-shedding lever.** Load shedding means doing a little less work per request so the
  system survives a traffic spike. Under a spike, we lower `efSearch` and quality dips slightly
  instead of requests timing out. This is one of HNSW's most useful operational properties, and
  it is widely underused.

The table above is only illustrative. Our own curve depends on our data's intrinsic dimension
(Chapter 6), our `M`, and our `efConstruction`. So we measure it, with brute force (Chapter 17)
as the ground truth:

```python
import time, numpy as np

truth = np.argsort(-(Q @ V.T), axis=1)[:, :10]        # brute-force ground truth

for ef in [10, 16, 32, 64, 128, 256, 512]:
    index.set_ef(ef)
    t0 = time.perf_counter()
    got = [index.knn_query(q, k=10)[0][0] for q in Q]
    ms = (time.perf_counter() - t0) / len(Q) * 1000
    rec = np.mean([len(set(a) & set(t)) / 10 for a, t in zip(got, truth)])
    print(f"ef={ef:4d}  recall@10={rec:.4f}  avg={ms:.2f}ms")
```

In simple words, for each setting we run every test question, time it, and count how many of the
true top 10 pages came back. (The timing here is an average. For p50 and p99, record each query's
time separately.)

Then we choose with this procedure, not with intuition:

- **Step 1: Find the latency budget.** If the Scholar takes 2 seconds to write an answer, 20 ms
  of retrieval is invisible, and cutting it to 5 ms buys nothing.
- **Step 2: Find the recall the end task needs.** Measure *answer quality*, not index recall.
  Often `efSearch = 64` and `efSearch = 256` give identical final answers, because a reranker
  (Chapter 13) reorders everything anyway.
- **Step 3: Take the knee of the curve.** The knee is the point where the curve bends and extra
  latency stops buying much recall. For most text workloads it sits between 64 and 128.
- **Step 4: Check p99, not just p50.** Graph search has a tail. Queries that land in sparse
  parts of the graph walk further. The gap between p50 and p99 widens as `efSearch` grows.

> **The useful rule:** retrieve more candidates than we need and let the reranker sort them out.
> `efSearch = 128` with `k = 100` feeding a cross-encoder usually beats `efSearch = 512` with
> `k = 10`, on both quality and latency.

---

## Where the time actually goes

What does one search actually cost?

```
distance computations:  ~ efSearch × M₀ × (a fraction, due to visited-set hits)
                        ≈ 1,000–4,000 for efSearch = 128, M = 16
each computation:       768 multiply-adds (SIMD)
memory access pattern:  RANDOM, and this is the real cost
```

`M₀` is the number of neighbours per page on layer 0, which is `2 × M = 32` here. So the most
the search computes is roughly 128 × 32 = 4,096 distances, plus a few dozen in the upper layers.
Many neighbours were already visited, so the real count is lower.

That last line is the one that matters. Brute force (Chapter 17) reads memory in one long, neat
stream, which hardware handles extremely well. **Graph search jumps around memory at random.**

Every hop lands on a page whose vector is somewhere unpredictable in RAM. The processor keeps a
small, very fast memory called the **cache**. When the vector it needs is not in the cache, that
is a **cache miss**, and the processor waits for main memory. Reading bytes from random places
costs several times more than reading the same number of bytes in one sequential stream.

It is like a shopping trip. Brute force walks every aisle in order with a trolley. Graph
search runs back and forth across the store for one item at a time. The number of items is
smaller, but the running dominates.

Three consequences follow, and they explain most HNSW performance mysteries:

- **Quantization speeds search up, not just shrinks it.** int8 vectors (Chapter 26) are 4×
  smaller, so 4× more of them fit in the cache, and fewer hops miss. Cheaper integer arithmetic
  adds to the gain.
- **Batching helps far less than for brute force.** Batching means answering many queries in one
  pass. Brute force queries all stream the same memory, so they share the work. Graph queries
  each jump to their own places, so there is little to share.
- **Memory layout details matter** on large single-machine indexes, more than people expect. The
  Ninja notes name the two that matter most.

---

## Small efSearch vs large efSearch

| | Small `efSearch` (16–32) | Large `efSearch` (256–512) |
|---|---|---|
| Recall@10 (typical) | ~0.80–0.90 | ~0.99–0.995 |
| Latency | lowest | several times higher |
| p99 tail | short | long |
| Risk | misses pages like 1,140 | spends time the pipeline may not need |
| Good for | autocomplete, quick previews, load spikes | final answers with no reranker, high-value queries |

**Advantages of a large `efSearch`:** higher recall, fewer dead ends, fewer half answers.

**Disadvantages of a large `efSearch`:** more distance computations, more random memory hops, a
longer tail, and recall gains that a reranker may throw away.

---

## When to use which one

We must use a **small `efSearch`** when latency is tight and the results are a rough draft, such as
search as you type, quick previews, or serving through a traffic spike.

We must use a **large `efSearch`** when the returned pages go straight to the Scholar with no
reranker, and a missing page like 1,140 means a wrong answer.

We must use a **middle value near the knee** (usually 64–128) when a reranker follows, with `k` as
large as the reranker can take. The same value is the right start when we do not yet know. Then we
measure.

Many strong systems use both. They start small and raise `efSearch` only for the queries that
look hard. The Ninja notes show how.

---

### Under the hood

Here is Phase 2 in Python, run on the toy graph from our example. It uses the same names as Chapters
28 and 29: `graph.neighbours(node, layer)` and `dist(query, node)`. Two heaps do the work. A
**heap** is a structure that hands back its smallest item instantly. "To explore" is a min-heap, so
the closest page comes out first. "Best so far" is stored with negated distances, so its *worst*
page comes out first and is cheap to drop.

```python
import heapq

def search_layer(graph, layer, entry_points, query, dist, ef):
    visited = set(entry_points)
    candidates = [(dist(query, e), e) for e in entry_points]   # "to explore": min-heap
    heapq.heapify(candidates)
    results = [(-d, e) for d, e in candidates]                 # "best so far": max-heap via negation
    heapq.heapify(results)

    while candidates:
        d_c, c = heapq.heappop(candidates)                     # Step 2: closest unexplored
        if d_c > -results[0][0] and len(results) >= ef:
            break                                              # Step 3: none can join, stop

        for n in graph.neighbours(c, layer):                   # Step 4: unvisited neighbours
            if n in visited:
                continue
            visited.add(n)
            d_n = dist(query, n)
            if len(results) < ef or d_n < -results[0][0]:      # Step 5: good enough?
                heapq.heappush(candidates, (d_n, n))
                heapq.heappush(results, (-d_n, n))
                if len(results) > ef:
                    heapq.heappop(results)                     # Step 6: drop the worst
    return sorted((-d, e) for d, e in results)                 # Step 8: best first

def hnsw_search(graph, query, dist, k, ef_search):
    entry = search_entry_point(graph, query, dist)             # Phase 1 (Chapter 28)
    best = search_layer(graph, 0, [entry], query, dist, max(ef_search, k))   # Phase 2
    return best[:k]

# A tiny piece of Acme's layer 0. Distances to the SSO question are given as toy numbers.
class ToyGraph:
    def __init__(self, links):
        self.links = links
    def neighbours(self, node, layer):
        return self.links[node]

toy = ToyGraph({40: [45, 77], 77: [40], 45: [40, 212, 530],
                212: [45], 530: [45, 1140], 1140: [530]})
distance = {40: 0.90, 77: 0.95, 45: 0.60, 212: 0.20, 530: 0.70, 1140: 0.25}
toy_dist = lambda query, page: distance[page]

for ef in (2, 4):
    top = search_layer(toy, 0, [40], "SSO question", toy_dist, ef)[:2]   # k = 2
    print(f"efSearch={ef}: pages {[page for _, page in top]}")
# efSearch=2: pages [212, 45]
# efSearch=4: pages [212, 1140]
```

In simple words, the code is the Steps, line for line, and it reproduces the example: a
shortlist of 2 misses page 1,140, and a shortlist of 4 finds it.

A full query is `hnsw_search`. It runs greedy descent through the upper layers with Chapter 28's
`search_entry_point`, makes **one call** to `search_layer` at layer 0 with `ef = efSearch`
(raised to `k` if needed), and keeps the top `k`.

---

### What people get wrong

**Setting `efSearch < k`.** You cannot return 100 results from a shortlist of 40. Some libraries
raise `efSearch` to `k` silently, others return fewer results. Either way nothing warns you.

**Tuning `efSearch` against index recall instead of answer quality.** You may be buying recall
your pipeline discards.

**Ignoring p99.** A 4 ms p50 with a 120 ms p99 shows up as user-visible stuttering that a p50
dashboard never reveals.

**Assuming `efSearch` fixes a weak graph.** If `M` is too small for your data, recall plateaus,
and raising `efSearch` only burns latency. If recall stalls below 0.95 no matter how high you go,
rebuild with a larger `M`.

**Using one `efSearch` for every query type.** Navigational lookups ("billing page") need much
less than multi-constraint questions like our SSO one.

---

### Ninja notes

**Adaptive `efSearch` is genuinely worth building.** Start at a low value. Escalate only if the
result set looks weak. For example, the top score is below a calibrated threshold, or the 1st
and 10th results score almost the same (the search never found a clear winner). Most
queries are easy and finish at `efSearch = 32`. The hard tail escalates to 256. Average latency
drops substantially while tail quality *improves*. This mirrors the adaptive routing idea from
Chapter 13 and is similarly underused.

**Prefetching matters.** Some implementations prefetch neighbour vectors while computing the
current distance, hiding memory latency behind arithmetic. Much of the speed gap between
libraries can come from here. The algorithm is identical, the memory choreography is not.

**TLB misses, NUMA and huge pages.** Every random hop can also miss the TLB, the processor's
cache of virtual-to-physical address translations. Huge pages (2 MB instead of 4 KB) cut those
misses sharply on a 100 GB index. On multi-socket servers, NUMA means memory attached to the
other socket is slower to reach, so pin the index and its query threads to the same node.

**Beware benchmark transfer.** Public HNSW numbers are usually measured on SIFT1M or GloVe,
older datasets whose structure differs from modern text embeddings. Your recall at
`efSearch = 64` can be quite different, so always measure on your own data. Skipping this is a
common reason a deployment underperforms its expected numbers.

---

### Key takeaways

- **efSearch = the length of the shortlist a beam search keeps while walking HNSW's bottom
  layer.** Greedy search is a shortlist of one.
- Once the shortlist is full, the search skips any neighbour worse than everything on it, and
  stops when the closest unexplored page is that bad. Both are bets, not guarantees. A longer
  shortlist bets less often, so it can step through a less relevant page (530) to reach the one
  we need (1,140).
- `efSearch` is a runtime knob: monotonic in practice, per-query tunable, and a natural
  load-shedding lever.
- Recall error roughly halves per doubling while latency grows about 1.6–1.7×. Measure your own
  curve and pick the knee, typically 64–128.
- Retrieve wide and rerank rather than pushing `efSearch` very high.
- Graph search is bound by random memory access, so quantization speeds it up as well as
  shrinking it, and batching helps less than you would hope.
- If recall plateaus regardless of `efSearch`, your `M` is too small. Rebuild.

### What's next

The algorithm is complete. [Chapter 31](./31-hnsw-production.md) covers what happens after we
deploy it: deletes, updates, memory pressure, and the rebuild we will eventually have to
schedule.

We now understand how an HNSW search walks the graph, why a longer shortlist finds pages a
greedy walk misses, and how to choose `efSearch` with a measured curve instead of a guess.
