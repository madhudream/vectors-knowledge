---
title: "HNSW in Production"
chapter: 31
part: "Part IV — Index Structures"
slug: hnsw-production
readingTime: "11 min"
summary: "The algorithm is elegant. Operating it is not. Deletes that aren't deletes, graphs that degrade, memory that doesn't fit, and the rebuild you will eventually schedule."
tags: [hnsw, production, operations, deletes, memory, rebuild]
prev: 30-hnsw-3-searching
next: 32-diskann
---

# HNSW in Production

**The one-paragraph version.** HNSW has one big operational weakness: **we cannot really
delete from it.** Removing a page would cut roads that other pages need for navigation. So
implementations only mark the page as deleted, leave it in the graph, and skip it when returning
results. Deleted pages still use memory and still get walked through. Over months of edits,
memory and recall both slowly get worse, and the only complete fix is a rebuild. We should plan
for that rebuild from day one.

In this chapter, we will learn what happens to an HNSW index after it goes live: how deletes and
updates really work, why the graph degrades, what to monitor, and how to rebuild safely. We will
also do the honest memory bill, and see which way out to take when it does not fit.

We will cover the following:

- What is a soft delete
- Why we cannot simply delete from HNSW
- How updates work
- Graph degradation
- Monitoring: the four numbers
- The rebuild
- Memory, honestly
- When to use which one

---

## What is a soft delete

**Soft delete = marking a page as deleted while leaving it, and all its links, inside the graph.**

The mark is called a **tombstone**. A page with a tombstone still sits in the Card Catalog. The
search can still walk through it. It is just never handed back as a result.

In simple words, a soft delete hides a page instead of removing it.

The opposite is a **hard delete**: actually removing the page and its links from the graph. As
we will see, HNSW cannot safely do that.

Tombstones pile up. Clearing them out, by rebuilding or merging the index, is called
**compaction**. Chapter 59 covers compaction in full.

---

## Why we cannot simply delete from HNSW

Let's go back to the tiny piece of Acme's graph from Chapter 30. Acme's support knowledge base
has 40,000 pages, and the question was our usual one about SSO on the Pro plan. The walk
reached page 1,140 ("On Pro, SSO needs the Security add-on for teams under 50 seats") only by
passing through page 530, an old "Security settings" page.

Now Acme retires page 530. It is out of date.

Try a hard delete first. We remove page 530 and its links. Page 1,140's only road into that part
of the graph is gone. The next search for the SSO question reaches page 212 and stops. It never
finds page 1,140. The Scholar answers "Yes, SSO is included on Pro." That answer misleads every
Pro team under 50 seats, and nothing in the system reports an error.

Now try a soft delete instead. We put a tombstone on page 530 and keep its links. The search
walks through page 530 exactly as before and reaches page 1,140. Page 530 is never returned,
because it is marked deleted. The Scholar gets pages 212 and 1,140, and the add-on rule is back
in the answer.

Picture closing a shop on a busy street:

- The shops are the pages.
- The streets between shops are the graph's links.
- A hard delete demolishes the shop *and* the street in front of it, so the shops behind it
  (pages like 1,140) are cut off.
- A soft delete puts a "closed" sign on the door. Nobody shops there, but people still walk past
  it to reach the shops behind.

Layer-0 links are load-bearing like that street. That is why almost every HNSW implementation uses
soft deletes.

The price is real:

- **Memory is not given back.** Delete half the corpus and the index uses exactly as much RAM.
- **Search still walks through deleted pages.** They take part in routing, which costs distance
  computations.
- **The shortlist can shrink.** Some implementations remove tombstones only after the walk. In
  those, tombstones take shortlist places. If 30% of the pages on the shortlist are tombstones, an
  `efSearch` of 128 behaves like ~90. Recall drops, silently and gradually. Implementations that
  skip tombstones while collecting results avoid this, but still pay for the extra walking.
- **`k` can under-return** in the same implementations. Ask for 10 pages, get 7, because three
  were tombstones.

The mitigation that libraries such as hnswlib offer is `replace_deleted`. When we insert a new page,
it reuses a tombstoned slot and rewires that slot's links. This bounds memory growth for
**update-heavy** workloads, where pages are replaced about as often as they are removed. It does
nothing for workloads that are genuinely shrinking.

**Practical rule of thumb: once tombstones exceed ~20% of the index, rebuild.**

---

## How updates work

Acme's product team edits page 1,140, the Security add-on page, every quarter. What happens to the
index each time?

Most implementations have no true in-place update. (hnswlib can also overwrite an existing
label in place, at a similar cost.) The usual operation is delete-then-insert:

- **Step 1:** Re-embed the new text of the page into a new vector.
- **Step 2:** Put a tombstone on the old vector.
- **Step 3:** Insert the new vector. It searches the graph, picks diverse neighbours (Chapter 29)
  and wires itself in, possibly reusing the old slot.

This is correct but not cheap. An insert costs roughly one search at `efConstruction`.

The important consequence for system design: **a page whose text changes must be re-embedded and
re-inserted.** Suppose our chunks change often, as in a wiki, a shared document store, or a
ticketing system. Then the index does continuous insertion work, and we must size the machines
for it.

This is where IVF's trivial deletes (Chapter 23) or a two-index design (Chapter 59, and the Ninja
notes below) start to look attractive.

---

## Graph degradation

Even with no deletes at all, a long-lived HNSW index drifts away from the graph we would build
today. There are three causes.

**Insertion order shapes the graph.** Early pages are connected to whatever existed when they
arrived. Suppose Acme's knowledge base grew in order: all the 2024 pages, then all the 2025
ones. The graph's structure records that history. Its entry point may sit in a region that was
central two years ago and is on the edge now.

**Pruning is local and greedy.** Each pruning decision (Chapter 29) is made with the information
available at that moment. A series of reasonable local decisions does not add up to a good global
graph.

**The data drifts.** New page types, new languages, a new product line. The shape of the data in
the Map Room moves, and the graph's long-range links no longer bridge the regions that now
matter.

The symptom is easy to miss: **recall slowly declining at constant `efSearch`.** Not a cliff,
not an incident. A slow drift, perhaps a percentage point every month or so.

In simple words, the graph ages. That is why the next section is not optional.

---

## Monitoring: the four numbers

Now, the question is, how do we notice a slow decline before users do?

We track four numbers continuously. Most teams track none of them and are surprised twice a
year.

**1. Index recall against brute force.** Every night, sample 200 queries, compute their exact
top-10 over the full corpus with one batched flat scan (Chapter 17), and record recall@10. **This
is the early warning system.** A downward trend means rebuild.

**2. Tombstone ratio.** `deleted_count / total_count`. In simple words, the share of the graph
that is dead weight. Alert at 15%, rebuild at 20%.

**3. p99 latency and its ratio to p50.** A widening gap points to graph degradation, memory
pressure, or unbalanced regions.

**4. Resident memory per live vector.** Resident memory is the RAM the process actually holds.
If it keeps rising while the number of live pages stays flat, tombstones are piling up.

The code for a nightly health check is in "Under the hood" below.

---

## The rebuild

We will rebuild. The goal is to make it routine rather than an emergency.

The safe way is the **blue-green** pattern. Blue-green means building the new index beside the
old one, then switching all traffic to it in one move (Chapter 63 uses the same pattern for model
changes).

- **Step 1:** Build a new index from the source of truth, in a separate process or machine. The
  source of truth is our object store of full-precision vectors (object storage is cheap bulk
  storage such as S3, and Chapter 26 explained why we kept those vectors).
- **Step 2:** Replay into the new index every write that happened during the build.
- **Step 3:** Verify. Run the evaluation set against the new index and compare recall and page
  counts with the old one. **Do not skip this.** A bad rebuild that silently loses pages is worse
  than a degraded index.
- **Step 4:** Swap the pointer in one atomic step, so every query goes either to the old index or
  to the new one, never a mix. Serve from the new index.
- **Step 5:** Keep the old index warm for an hour so we can roll back instantly.

**Budget the time.** Chapter 29 showed that building 100 million vectors at `efConstruction =
200` takes hours on a many-core machine. If that is too long, split the index into **shards**
(slices of the index, each on its own machine, Chapter 58) and rebuild them one at a time. With
40 shards, each rebuild covers 1/40 of the vectors, and each swap affects only 2.5% of traffic.

---

## Memory, honestly

Let's scale the scenario up. Suppose Acme's knowledge-base product now hosts the knowledge bases
of 5,000 customer companies, about 100 million chunks (pieces of pages, Chapter 50), each a
768-d vector. Part VIII returns to this setup. Here is the plain HNSW bill:

```
100M vectors × 768 dims:
  float32 vectors:        307 GB
  graph edges (M=16):      13 GB
  IDs and overhead:         3 GB
                   total ≈ 323 GB
```

In simple words, the vectors are 95% of the bill. The graph itself is small. Each vector is
3,072 bytes, and layer 0 adds 32 links × 4 bytes = 128 bytes per vector.

Options when that does not fit, in order of how much they cost us:

| Approach | RAM | Cost |
|---|---|---|
| int8 quantization (Chapter 26) | ~90 GB | ~1% recall, **do this first** |
| Matryoshka truncation to 256-d (Chapter 15) | ~120 GB | ~1–2% recall, needs a Matryoshka model |
| int8 + Matryoshka-256 | ~40 GB | small, compounding |
| Binary + rescore (Chapter 26) | ~25 GB RAM + full vectors on SSD | ~3% recall |
| Shard across machines (Chapter 58) | unchanged in total, split per machine | network hop, coordination |
| DiskANN (Chapter 32) | ~5 GB RAM + SSD | higher latency |

The int8, combined, binary and DiskANN rows roughly match Chapter 60's cost table. The int8 row is
768 + 128 + ~30 bytes per vector, about 926 bytes, so ~90 GB for 100 million.

**The order matters.** Turn on int8 before sharding, and shard before moving to disk. Each step
adds operational complexity. Take them in the order that adds the least.

---

## When to use which one

We have a few ways to live with HNSW in production. Here is how they compare.

| | Soft deletes + scheduled rebuild | IVF (Chapter 23) | Main HNSW + small flat delta index |
|---|---|---|---|
| Deletes | tombstones | trivial, remove the ID | tombstone set over both |
| New pages visible | after insert | after insert | instantly |
| Graph degradation | grows until rebuild | no graph | reset every night |
| Operational work | rebuild every few months | retrain centroids now and then | nightly rebuild, two searches |

**Advantages of HNSW in production:** excellent recall and latency, and simple inserts.

**Disadvantages of HNSW in production:** fake deletes, slow degradation, a large memory bill, and
a rebuild we cannot avoid.

We must use **soft deletes plus scheduled rebuilds** when pages change slowly, for example a
product manual updated once a quarter.

We must use **IVF** when churn is heavy and deletes must free memory right away, and we can
accept its recall trade-offs.

We must use **a main HNSW index plus a small delta index** when pages change every day and must
be searchable within seconds. Many strong systems end up here. The Ninja notes show the design.

---

### Under the hood

The update from "How updates work", in `hnswlib`:

```python
# index created with allow_replace_deleted=True
new_vector = model.encode(new_text_of_page_1140)       # Step 1: re-embed
index.mark_deleted(doc_id)                             # Step 2: tombstone the old vector
index.add_items(new_vector, doc_id, replace_deleted=True)   # Step 3: insert, reuse a slot
```

In simple words, a mark, then a full insert.

The nightly health check follows the monitoring section.

- **Step 1:** Compute exact top-10 answers for a sample of queries, over every live vector in
  the corpus. Turn the row numbers into the index's own labels.
- **Step 2:** Ask the index for its top 10 on the same queries.
- **Step 3:** Record the overlap (recall), the tombstone ratio, and memory per live vector.

```python
import numpy as np

# exact_topk and mean_overlap are the brute-force and overlap steps from Chapter 18's code.
# live_vectors is a flat copy of every live (not deleted) vector. live_labels[i] is the index
# label of row i (a list or an array). deleted_count, live_count and memory_bytes are
# placeholders: keep your own counters if your library (hnswlib, for example) does not
# report them.
def nightly_index_health(index, live_vectors, live_labels, sample_queries):
    rows  = exact_topk(sample_queries, live_vectors, k=10)    # full corpus, one flat scan
    truth = np.asarray(live_labels)[rows]                     # row numbers -> index labels
    got   = [index.knn_query(q, k=10)[0][0] for q in sample_queries]
    return {
        "recall@10":  mean_overlap(got, truth),
        "tombstones": index.deleted_count / max(index.element_count, 1),
        "mem_per_live_vector": index.memory_bytes() / max(index.live_count, 1),
    }
```

Chart these three numbers, plus p99/p50 from your latency metrics, on one dashboard. The trend
matters more than any single night.

---

### What people get wrong

**Assuming deletes free memory.** They do not in hnswlib, Lucene (until segments merge) or most
databases. FAISS's HNSW index cannot delete at all. Vespa is a notable exception.

**Never rebuilding.** Index quality decays. Rebuilding is a maintenance task, like vacuuming a
database.

**No brute-force ground truth in production.** You cannot detect recall decay without something
to measure against. Keep a flat copy of the vectors to score sampled queries against. One
batched scan a night is cheap.

**Rebuilding without verification.** Silent document loss during a rebuild is a real and common
failure. Compare document counts *and* eval-set recall before swapping.

**Storing only the index.** If the index is your only copy of the vectors, you cannot rebuild,
rescore, migrate models, or change parameters. Keep full-precision vectors in object storage as
the source of truth. For rescoring at query time, serve a copy from SSD or RAM (Chapter 60).

**Treating HNSW as a database.** It is an index. It has no durability guarantees, no
transactions, and no schema. The source of truth lives elsewhere.

---

### Ninja notes

**The two-index pattern** is how most mature systems handle churn, and it sidesteps most of this
chapter:

```
┌─ Main index (HNSW) ────────────┐    ┌─ Delta index (flat) ────────┐
│  rebuilt nightly               │    │  today's new documents      │
│  100M vectors, stable          │    │  ~50k vectors, brute force  │
└────────────────────────────────┘    └─────────────────────────────┘
                ↓                                   ↓
                 search both, merge results by score
```

New documents go into a small flat index: instant visibility, exact search, trivial deletes.
Deletions are recorded in a tombstone set applied to both. Every night, the delta is folded into
a freshly rebuilt main index.

You get **immediate freshness**, **no graph degradation**, and **bounded rebuild cost**, at the
price of searching two indexes and merging. The delta is small, so its brute-force scan costs
a millisecond or two. This pattern is worth reaching for early. It is the same log-structured idea
that underlies Lucene segments and LSM trees, and Chapter 59 covers it properly.

---

### Key takeaways

- **Soft delete = a tombstone on a page that stays in the graph.** HNSW needs it because hard
  deletes cut the roads other pages depend on.
- Tombstones do not free memory (Vespa aside), are still walked through, and in some
  implementations shrink the effective `efSearch` and under-return `k`.
- Updates are re-embed, tombstone, and insert, costing roughly one search each.
- Graphs degrade with insertion order, local pruning, and data drift, visible as slowly
  declining recall.
- Monitor four numbers: index recall, tombstone ratio, p99/p50, and memory per live vector.
  Rebuild at ~20% tombstones.
- Rebuild blue-green, verify before swapping, and keep the old index warm.
- At 100M vectors, plain HNSW needs ~323 GB. int8 brings it to ~90 GB, binary + rescore to ~25 GB,
  DiskANN to ~5 GB of RAM plus SSD.
- Compress before sharding. Shard before going to disk.
- The main-index-plus-delta-index pattern solves freshness and degradation together.

### What's next

When 320 GB of RAM is not on the table, the graph has to live somewhere else.
[Chapter 32](./32-diskann.md) puts it on an SSD and keeps the latency respectable.

We now understand why HNSW cannot truly delete, how tombstones and updates behave, how to watch an
index age, and how to rebuild it and fit it in memory without surprises.
