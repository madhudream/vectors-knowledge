---
title: "DiskANN: Billion-Scale on SSD"
chapter: 32
part: "Part IV — Index Structures"
slug: diskann
readingTime: "14 min"
summary: "Keep compressed vectors in RAM to decide where to walk, and read full vectors from SSD only when you arrive. A billion vectors on one machine, at single-digit milliseconds."
tags: [diskann, vamana, ssd, billion-scale, memory, graph]
prev: 31-hnsw-production
next: 33-filtered-search
---

# DiskANN: Billion-Scale on SSD

**The one-paragraph version.** HNSW needs the whole index in RAM. **DiskANN** splits it in two.
Heavily compressed PQ codes stay in memory and guide the walk through the graph. The graph's
neighbour lists and the full-precision vectors live on an SSD, and are read only for the nodes
the walk actually visits. The graph is flat, single-layer, and built so that paths are short. So
a query visits only about a hundred nodes, and a small number of SSD reads answers it. One
machine, a billion vectors, single-digit milliseconds.

In this chapter, we will learn how DiskANN searches a billion vectors on one machine without
holding them in RAM. We will see why HNSW cannot simply move to disk, how the Vamana graph is
built, and how a query reads from the SSD in parallel. We will also do the money maths, and see
when DiskANN is worth its complexity.

We will cover the following:

- What is DiskANN
- Why HNSW cannot simply move to disk
- How the Vamana graph is built
- How a DiskANN query works
- An example of a DiskANN query
- The economics
- Updates: Fresh-DiskANN
- When to use which one

---

## What is DiskANN

Before jumping into DiskANN, we must know what an SSD is.

**SSD = solid-state drive.** It is storage made of flash memory chips, with no moving parts. It
keeps data when the power is off. Per byte, it is far cheaper than RAM, and far slower to reach.
Modern SSDs connect through **NVMe**, a fast interface that talks to the processor directly.

A drive's speed for this chapter is measured in **IOPS**.

**IOPS = input/output operations per second.** In simple words, how many separate random reads
a drive can do each second. A good NVMe drive does hundreds of thousands.

Now the definition.

**DiskANN = a graph index that keeps a small compressed copy of every vector in RAM to decide
where to walk, and keeps the graph and the full vectors on SSD.**

Think of it like a Librarian with a small desk and a huge warehouse:

- The warehouse is the SSD. It holds every full book and the "see also" list for each one.
- The desk is RAM. It holds one small index card per book, a crude summary (the PQ code from
  Chapter 24).
- The index cards are not good enough to answer anything for sure. They are good enough to decide
  *which shelf to walk to next*.
- A trip to the warehouse is slow, so the Librarian fetches a book only when the cards say it is
  worth a look.

DiskANN is exactly that division of labour:

| | Contents | Size for 1B × 768-d |
|---|---|---|
| **RAM** | PQ-compressed vectors (index cards) | ~32–64 GB |
| **SSD** | Graph neighbour lists + full float32 vectors | ~4.1 TB |

The compressed vectors are used to *navigate*. The full vectors are used only to *rescore* the
final shortlist (Chapter 25). The navigation is approximate and cheap. The final ranking uses
exact distances and needs no extra reads.

---

## Why HNSW cannot simply move to disk

Let's take a scenario. Acme's knowledge-base product keeps growing. It hosts the knowledge bases
of thousands of customer companies, and the index is heading for **one billion chunks** (pieces
of pages), each a 768-d vector. Plain HNSW would need about 3.2 TB of RAM. Even int8 HNSW needs
about 900 GB.

So the obvious idea is to leave HNSW exactly as it is, and put it on an SSD. Let's see what
happens.

An SSD random read takes roughly **100 microseconds** (μs). A RAM access takes about **100
nanoseconds**, a thousand times faster. Chapter 30 showed that an HNSW search computes a few
thousand distances. On disk, each one needs the neighbour's vector, which is a random read.

A search touching 3,000 nodes, one read at a time, is 3,000 × 100 μs = **300 milliseconds**.
Plus the upper layers. The user asks *"Does the Pro plan include single sign-on?"* and waits a
third of a second for retrieval alone. The design is wrong.

So how do we make a graph search that waits on the disk only a few dozen times?

There are three ideas, and DiskANN uses all three:

1. **Visit far fewer nodes**, by building a graph with shorter paths. This is Vamana.
2. **Never read the disk to decide where to go.** Use the PQ codes in RAM for that.
3. **Read several nodes at once**, in parallel, instead of one after another.

---

## How the Vamana graph is built

DiskANN's graph construction algorithm is called **Vamana**. It optimises for one thing: short
search paths, at the price of a denser graph than HNSW's.

**Vamana = a flat, single-layer graph whose neighbour-selection rule deliberately keeps some
long-range shortcuts.**

Two design choices make that work.

**No hierarchy.** HNSW's layers exist to shorten routing. Vamana gets short paths from its
edge-selection rule instead. No layers means no extra reads for the upper layers.

**Alpha-pruning.** Vamana's neighbour selection generalises HNSW's diversity heuristic (Chapter 29)
with a relaxation parameter $\alpha \ge 1$. While choosing neighbours for a node $p$:

> Drop candidate $c$ if some already-selected neighbour $s$ satisfies
> $\alpha \cdot d(s, c) \le d(p, c)$.

In simple words, we drop $c$ when an existing neighbour $s$ is already much closer to $c$ than $p$
is, because the walk can reach $c$ through $s$.

With $\alpha = 1$ this is essentially HNSW's rule. With $\alpha \approx 1.2$ the rule is *more
permissive*: $s$ must be clearly closer before $c$ is dropped. So Vamana keeps some edges the
strict rule would prune. Those extra edges are long-range shortcuts, and they shorten search paths
measurably.

The graph is denser, typically 64–128 neighbours per node versus 32–64 on HNSW's layer 0. But
each query visits far fewer nodes, and on an SSD that trade is overwhelmingly correct.

Here is the build, step by step:

- **Step 1:** Start with a random graph, where each node links to `R` random others. `R` is the
  maximum number of neighbours per node.
- **Step 2:** Pick the **medoid**, the data point nearest the centre of all the data. Every search
  will start there.
- **Step 3:** Take the nodes one by one in random order. For each node, run a search for it from
  the medoid, with a search list of size `L` (often 75–200, the `efConstruction` of Vamana).
  Collect every node the search visited.
- **Step 4:** Choose the node's neighbours from those visited nodes with the alpha-pruning rule,
  keeping at most `R`.
- **Step 5:** Add a link back from each chosen neighbour. If that pushes a neighbour past `R`
  links, prune its list too.
- **Step 6:** Do the whole pass **twice**. The first pass uses $\alpha = 1$. The second uses the
  larger $\alpha$, refining the graph once a basic structure exists.

---

## How a DiskANN query works

The query has two limits that we set.

**Search list L = how many of the best candidates found so far the query remembers.** It plays
the role of `efSearch` in HNSW (Chapter 30). A typical value is about 100.

**Beam width W = how many nodes the query reads from the SSD at the same time.** Typical values
are small, about 2 to 8.

Each node sits on the SSD as one **4 KB block**: its neighbour list and its full float32 vector,
side by side. So a single read returns both.

- **Step 1:** Build the PQ distance tables for the query (Chapter 24). RAM only, microseconds.
- **Step 2:** Put the medoid into the search list.
- **Step 3:** Pick the `W` closest nodes in the list that we have not visited yet.
- **Step 4:** Read their `W` blocks from the SSD, all at the same time.
- **Step 5:** Keep each node's full vector aside for later.
- **Step 6:** For every neighbour in those blocks, estimate its distance with the PQ codes in RAM
  (no disk read). Add it to the search list, and keep only the `L` best.
- **Step 7:** Repeat from Step 3 until every node in the search list has been visited.
- **Step 8:** Rescore with the full vectors already read alongside each visited node's neighbour
  list. Return the top `k`.

The crucial property: **Step 6 decides where to go using the PQ codes in RAM.** The only disk
traffic is Step 4. And because the full vector rides along in the same block, Step 8 needs no
extra reads at all.

In simple words, the cards on the desk choose the route, and each trip to the warehouse brings
back both the book and its "see also" list.

Step 4 is where the remaining speed comes from. The reads are independent, so the query sends
all `W` of them together and waits for them together. On Linux this is done with asynchronous
I/O, such as libaio or **io_uring**, Linux's fast asynchronous I/O interface. Both let a program
submit many reads and collect the results without waiting on each one. Modern NVMe drives sustain
hundreds of thousands of IOPS, so many queries can do this at once. The big win is still Vamana
plus PQ navigation, which cuts 3,000 reads to about 100. Parallel reads then cut the wait by
another 3–4×.

Why not read 64 nodes at once? Because the query does not know which nodes will be useful until
it sees the previous round. A very wide beam spends reads on nodes that turn out to be dead ends.
A small beam of 2–8 gets most of the speedup without that waste.

---

## An example of a DiskANN query

Let's run our question, *"Does the Pro plan include single sign-on?"*, against Acme's billion
chunks, with `L = 100`, `W = 4` and `k = 10`. (Those chunks belong to many different companies.
Keeping each company's pages to itself is Chapter 33's problem, so we set it aside here.)

**Step 1.** The query's PQ tables take microseconds.

**Steps 2 to 7.** The walk starts at the medoid. Each round reads 4 blocks in parallel. About a
hundred nodes get visited before every node in the search list has been read. That is roughly 100
reads, in about 25–30 rounds.

Each round waits about 100 μs for the drive. So the whole walk spends roughly 2.5–3 ms waiting on
the SSD. Together with the in-memory PQ arithmetic, the query takes a few milliseconds.

Compare that with the naive plan. HNSW on disk did 3,000 reads, one after another: 300 ms. The
answer to "can we serve this from disk?" was no. Now it is yes.

**Step 8.** PQ distances are rough. Suppose page 1,140 ("On Pro, SSO needs the Security add-on
for teams under 50 seats") finished the walk in 14th place on the PQ estimates. Its full vector
was already read in Step 4. Rescoring with it moves page 1,140 up to 2nd place, right behind page
212 ("SSO is included on the Pro plan"). Both land in the top 10.

The Scholar reads both pages and answers correctly. The only thing that changed from the slow
design is *where* each piece of data lives and *when* it is read.

---

## The economics

This is the whole argument for DiskANN, and it is stark.

For **one billion 768-dimensional vectors**:

| Approach | RAM needed | Rough monthly cloud cost |
|---|---|---|
| HNSW, float32 | ~3.2 TB | tens of thousands of dollars, multi-machine |
| HNSW, int8 | ~900 GB | still a large, specialised fleet |
| IVF-PQ in RAM | ~105 GB | one large machine |
| **DiskANN** | **~50 GB + ~4.1 TB on an 8 TB NVMe drive, or two 4 TB drives** | **one mid-sized machine** |

Where the numbers come from:

- HNSW int8: 768 bytes of vector + 128 bytes of links + ~30 bytes of overhead ≈ 926 bytes per
  vector, so ~900 GB.
- IVF-PQ: 96 bytes of code (Chapter 24) + an 8-byte ID ≈ 104 bytes, so ~105 GB.
- DiskANN RAM: 32–64 bytes of PQ code per vector, so 32–64 GB, about 50 GB.
- DiskANN SSD: 3,072 bytes of vector + a neighbour list of 64–128 IDs ≈ 3.3–3.6 KB of data per
  node, padded to one 4,096-byte block, so ~4.1 TB.

NVMe storage costs roughly 10 to 100 times less per byte than RAM. DiskANN turns a memory problem
into a storage problem, and storage is the cheap resource.

The cost is latency. Expect **2–10 ms** where an in-memory HNSW would give 1–2 ms. For most
applications, and certainly any that then calls the Scholar, that difference is invisible.

---

## Updates: Fresh-DiskANN

What happens when Acme edits a page?

The original DiskANN was static. **Fresh-DiskANN** adds streaming updates, with the same
two-tier idea as the delta-index pattern in Chapter 31's Ninja notes:

- **Step 1:** New vectors go into a small **in-memory** Vamana index.
- **Step 2:** Deletions are recorded in a tombstone set (Chapter 31).
- **Step 3:** Every query searches both the SSD index and the in-memory one, and drops tombstoned
  results.
- **Step 4:** Periodically, a background **merge** folds the in-memory index and the deletions
  into the on-disk graph, repairing edges that pointed at deleted nodes.

This is the same log-structured shape Chapter 31 described. It keeps appearing because it is the
right answer to "fast writes, plus a structure that is expensive to change in place".

---

## When to use which one

| | HNSW in RAM (Chapters 28–31) | DiskANN |
|---|---|---|
| Where the index lives | all in RAM | PQ codes in RAM, graph + vectors on SSD |
| RAM at 1B vectors | ~3.2 TB float32, ~900 GB int8 | ~50 GB |
| Typical latency | 1–2 ms | 2–10 ms |
| Hardware needs | lots of RAM | fast NVMe with high IOPS |
| Updates | inserts + tombstones | Fresh-DiskANN merges |
| Complexity | lower | higher |

**Advantages of DiskANN:** a billion vectors on one machine, RAM cut by more than 10× even against
int8 HNSW, and exact rescoring at no extra I/O.

**Disadvantages of DiskANN:** a few more milliseconds, a hard dependency on fast local SSDs, and
more moving parts to operate.

We must use **HNSW in RAM** when the corpus is under roughly 100 million vectors, or the memory
bill is affordable. With int8, 100 million vectors fit in about 90 GB.

We must use **DiskANN** when the corpus is in the hundreds of millions or billions, and RAM is the
budget rather than the bottleneck.

Many strong systems use both: HNSW for small, hot, fast-changing data, and DiskANN for the huge,
slower-changing bulk.

---

### Under the hood

The query loop, with the RAM/SSD split made explicit. The comments match the Steps above.

```python
def diskann_search(q, pq_codes, ssd, start, k=10, L=100, beam_width=4):
    tables = build_pq_tables(q)                                  # Step 1: RAM only
    search_list = {start: pq_dist(tables, pq_codes[start])}      # Step 2: node -> rough distance
    full = {}                                                    # node -> full vector read from SSD

    while True:
        unvisited = [n for n in search_list if n not in full]
        if not unvisited:
            break                                                # Step 7: list fully visited
        beam = sorted(unvisited, key=search_list.get)[:beam_width]   # Step 3: W closest unvisited
        blocks = ssd.read_many(beam)                             # Step 4: W reads, in parallel
        for node, block in zip(beam, blocks):
            full[node] = block.full_vector                       # Step 5: keep for rescoring
            for nb in block.neighbours:                          # Step 6: rough distances, no I/O
                if nb not in search_list:
                    search_list[nb] = pq_dist(tables, pq_codes[nb])
        keep = sorted(search_list, key=search_list.get)[:L]      # keep only the L best
        search_list = {n: search_list[n] for n in keep}

    exact = [(n, float(((v - q) ** 2).sum())) for n, v in full.items()]   # Step 8: rescore
    return sorted(exact, key=lambda x: x[1])[:k]
```

Two things to notice. `pq_dist` never touches the disk, so navigation is a pure RAM operation.
And the full vectors needed for rescoring were already fetched during the walk, because they sit
in the same block as the neighbour list. So the rescore costs no extra reads.

In simple words, `ssd.read_many` is the only line that waits on the drive, and it is called about
`100 / W` times per query.

---

### What people get wrong

**Running it on spinning disks or network storage.** DiskANN assumes NVMe-class random-read
latency and high IOPS. On a cloud volume with throttled IOPS, or on a hard disk, it falls apart
completely. Check your IOPS limits before benchmarking.

**Synchronous I/O.** Without parallel reads, ~100 hops cost ~10 ms one after another, several
times slower than a beam of 4. Vamana and PQ navigation already did the big work of cutting 3,000
reads to ~100. Parallel reads cut the wait by another 3–4×, and synchronous I/O gives that away.

**Using it too early.** Under ~100 million vectors, in-memory HNSW with int8 is simpler, faster,
and cheaper to operate. DiskANN's complexity is justified by scale, not by elegance.

**Ignoring cache warmup.** A cold index is slow for the first minutes while the operating system's
file cache (the page cache) fills with hot nodes. The nodes a few hops from the medoid are read by
nearly every query, so they are the hottest of all, and implementations commonly keep them cached
in RAM. Warm the index before serving traffic, or your deploy looks like an outage.

**Setting the beam width like `efSearch`.** A beam width of 64 is not "more accurate". Accuracy
comes from `L`. `W` only sets how many reads go out at once, and wide beams waste reads.

---

### Ninja notes

**The rescoring pattern reaches its purest form here.** DiskANN is Chapter 6's
coarse-then-refine principle expressed as a memory hierarchy. Cheap approximations in the fast,
small tier decide *where to look*. Expensive exact values in the slow, large tier decide *what to
return*. The same shape appears in CPU caches, in database buffer pools, and in the cascade of
Chapter 18. Once you have the pattern, new systems become easy to read.

**Cloud object storage is the next tier.** Several recent systems put the graph in S3 or
equivalent, with aggressive caching. Latency rises to tens or hundreds of milliseconds, and cost
falls by another order of magnitude. For archival corpora ("search all our documents from the last
decade"), where a 200 ms query is perfectly acceptable, this is a legitimate and dramatically
cheaper design. Expect more of it.

**SPANN** is worth knowing as the alternative shape: a cluster-based (rather than graph-based)
disk index, keeping centroids in memory and posting lists on SSD. Strategy 3 from Chapter 20
rather than strategy 4, with the same memory-hierarchy insight.

---

### Key takeaways

- **DiskANN = PQ codes in RAM to navigate, plus the graph and full vectors on SSD.**
- An SSD read is ~1,000× slower than RAM, so HNSW moved to disk as-is takes ~300 ms per query.
- Vamana builds a flat, denser graph with $\alpha$-relaxed pruning (two passes, $\alpha = 1$ then
  $\alpha \approx 1.2$), so searches visit very few nodes.
- The query keeps a search list `L` (~100, the `efSearch` analogue) and reads a beam of `W` (~2–8)
  nodes from the SSD in parallel per round.
- Neighbour lists and full vectors share a 4 KB block, so rescoring needs no extra reads.
- Vamana plus PQ navigation cuts 3,000 reads to ~100. Parallel asynchronous reads (libaio or
  io_uring) then cut the wait by another 3–4×.
- A billion vectors on one machine: ~50 GB RAM + ~4.1 TB NVMe (an 8 TB drive, or two 4 TB
  drives), at 2–10 ms, versus ~900 GB of RAM for int8 HNSW.
- Fresh-DiskANN adds in-memory deltas plus background merges for streaming updates.
- Use it above ~100M vectors. Below that, in-memory HNSW with int8 is simpler.

### What's next

Every index so far assumed we search the whole corpus. [Chapter 33](./33-filtered-search.md)
adds `WHERE tenant_id = 42`, and shows why that innocent clause is the hardest problem in this
part of the book.

We now understand how DiskANN splits an index between RAM and SSD, why its graph is built for
short paths, and when a few extra milliseconds buy a very large drop in memory cost.
