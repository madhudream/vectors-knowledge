---
title: "Sharding, Replication, and Routing"
chapter: 58
part: "Part VIII — Scale and Production"
slug: sharding-and-routing
readingTime: "11 min"
summary: "How one index becomes forty. Random versus semantic sharding, scatter-gather and its tail latency, replication for throughput, and routing that avoids asking every shard."
tags: [sharding, replication, routing, distributed, tail-latency, scale]
prev: 57-architecture-at-scale
next: 59-freshness
---

# Sharding, Replication, and Routing

**The one-paragraph version.** When an index outgrows one machine, we split it into
**shards**. Split *randomly*, and every query must ask every shard. That is simple and even, but
sensitive to tail latency. Split *semantically* by cluster, and a router can ask only the
relevant few. That is efficient, but prone to imbalance and boundary misses. Split *by tenant*,
and routing is free. Then we **replicate** each shard for throughput and availability. Sharding
solves capacity. Replication solves traffic. Routing decides how much of either each query
costs.

In this chapter, we will learn what shards and replicas are, the three main ways to split an
index, and why asking many shards makes slow requests common. We will also see a comparison of
the strategies, when to use which one, and the code most teams end up writing.

We will cover the following:

- What is sharding
- Strategy 1: random (hash) sharding
- Strategy 2: semantic (cluster) sharding
- Strategy 3: tenant sharding
- An example: one Acme question, forty shards or one
- Sharding strategies compared
- When to use which one
- What is replication

---

## What is sharding

**Shard = one piece of an index, stored and searched on its own machine.**

**Sharding = splitting one index into shards, so that together they hold more than any single
machine can.**

Two more terms come with it.

**Fan-out = sending one query to many shards at once.**

**Routing = deciding which shards a query should ask.**

Think of it like the Great Library outgrowing its building. We open twenty branches. There are
three ways to distribute the books.

- **Randomly.** Every branch holds a random sample. Loads are even and no branch is special. But
  a patron must phone all twenty branches and wait for the slowest one to answer.
- **By subject.** Science in one branch, history in another. A patron with a science question
  phones one branch. But the science branch is overwhelmed, and a book on the history of science
  might be in either.
- **By owner.** Each organisation's collection lives in its own branch. A patron from
  organisation 7 only ever phones branch 7.

Here, each branch is a shard. Each patron is a query. Phoning every branch is fan-out. Deciding
whom to phone is routing. The three ways are the three sharding strategies, and the right one
depends on the query pattern.

Our scenario is Acme's knowledge-base product from Chapter 57: 5,000 customer companies (tenants)
and about 100 million chunks. Acme splits the index into 40 shards of about 2.5 million chunks
each. A single big machine could hold 100 million int8 vectors (about 93 GB, Chapter 60). Acme
shards anyway, for growth and so that forty small shards rebuild in parallel instead of one
huge index rebuilding for hours.

---

## Strategy 1: random (hash) sharding

Assign each vector to `hash(doc_id) % N`. Means, fingerprint the document id (Chapter 57), take
the remainder after dividing by the number of shards N, and use that as the shard number. Every
document lands on an effectively random shard.

That plain version hides a trap. **A plain `hash % N` moves almost every vector when N
changes.** Going from 40 shards to 41 moves about 98% of them, because only one document in 41
gets the same remainder both ways. So real systems put a fixed layer in between. They hash each
document into one of a fixed number of **virtual buckets**, say 4,096, and keep a small table
that says which shard holds each bucket. Adding a shard moves two or three whole buckets from each
existing shard onto it, about 1/N of the data. **Consistent hashing** is the other common choice.
It places shards and documents on one ring of hash values, and a new shard takes over only its
stretch of the ring.

In simple words, the document's bucket never changes. Only the small bucket-to-shard table does.

**Query path: scatter-gather.**

```
query ──► router ──┬──► shard 1  ─► local top-k ─┐
                   ├──► shard 2  ─► local top-k ─┤
                   ├──►   ...                    ├──► merge by score ─► global top-k
                   └──► shard N  ─► local top-k ─┘
```

**Step 1:** The router sends the query to all N shards.

**Step 2:** Each shard searches its own index and returns its local top-k.

**Step 3:** The router merges the lists by score and keeps the best k overall.

Every vector lives in exactly one shard, and each shard returns its own best k. So the global
top-k is always inside the merged lists, up to each shard's ANN approximation. This holds for
any way of splitting, as long as we ask every shard. What randomness adds is balance.

**Strengths:** perfectly balanced storage and load, no hot shards, easy to add shards (with
virtual buckets or consistent hashing, about 1/N of the data moves), no dependency on the data
distribution, and correctness that is easy to reason about.

**The cost: tail latency.** **Tail latency = the slow end of the latency distribution**, the p99
and beyond (Chapter 18). A query is as slow as its *slowest* shard.

Say each shard has a p99 of 20 ms. That means 1 request in 100 takes longer than 20 ms. A query
fanning out to 40 shards hits at least one shard's p99 on about a third of requests. The math:

$$ P(\text{no shard exceeds its p99}) = 0.99^{N} $$

In simple words, each shard is fast 99% of the time, and all N must be fast at once.

At $N = 40$ that is 0.67, so a third of queries experience at least one p99-slow shard. The
share grows quickly with N:

| Shards asked | Queries that hit at least one shard's p99 |
|---|---|
| 1 | 1% |
| 10 | 9.6% |
| 20 | 18.2% |
| 40 | 33.1% |
| 70 | 50.5% |
| 100 | 63.4% |

The typical query drifts toward the slow tail. At 40 shards, the system's median latency is
already the shard's 98th percentile. At about 70 shards, the system's median equals the shard's
p99.

**Mitigations:**

- **Hedged requests.** A **hedged request** is a duplicate sent to another replica of the same
  shard after a short delay. We take whichever answer returns first. Very effective, for modest
  extra load.
- **Partial results with a deadline.** Return the merge of the shards that answered within the
  budget. For RAG this is often acceptable, but it is not free. Missing one shard of forty drops
  one of the top 10 in about one query in five, and costs about 2.5% of recall@10 on average.
  Log coverage and alert on it.
- **Fewer, larger shards**, where memory allows.

---

## Strategy 2: semantic (cluster) sharding

Run k-means over a sample to get N centroids. This is Chapter 23's IVF at the machine level.
Each shard holds one cluster. A router compares the query to the centroids and asks only the
nearest few shards.

```
query ──► router: nearest 3 of 40 centroids ──► shards 7, 19, 31 ──► merge
```

**Strengths:** most queries touch a small fraction of shards, so total compute and fan-out tail
latency both drop dramatically.

**Weaknesses, which are the IVF problems at cluster scale:**

- **Imbalance.** Real data produces lopsided clusters (Chapter 23). Every one of Acme's 5,000
  customers has login and SSO pages, so the "login" shard may hold 15% of the corpus and receive
  30% of the traffic. Use balanced k-means or split oversized clusters.
- **Boundary misses.** A page about SSO billing sits near the edge between the billing cluster
  and the SSO cluster. It lives in one shard, and a query routed to the other shard misses it.
  Probe more shards, or copy boundary vectors into both.
- **Drift.** New content shifts the distribution. Clusters grow unbalanced over time and need
  periodic rebalancing, which means moving data between machines.

---

## Strategy 3: tenant sharding

If every query is scoped to one tenant, workspace or customer, **shard by that key**. Routing
costs nothing, and each query touches exactly one shard. This is Chapter 33's advice to
partition instead of filtering, at the machine level.

Place large tenants on dedicated shards and pack small tenants together. This also delivers
isolation, per-tenant deletion and per-tenant scaling. Those properties have security
consequences, which Chapter 62 covers.

---

## An example: one Acme question, forty shards or one

A user of Acme's own help centre asks *"Does the Pro plan include single sign-on?"* Acme's own
40,000 pages are one tenant among 5,000. That is about 320,000 chunks, or 0.32% of the corpus.

**First, random sharding with a post-filter.** Acme's chunks are spread evenly, about 8,000 per
shard, so the router must ask all 40 shards. Each shard returns its top 100 by similarity, then
drops everything that is not Acme's.

Here is the trap. Almost every customer has pages about single sign-on. So each shard's top 100
is mostly *other companies'* SSO pages. After filtering, only a handful of Acme's chunks
survive. Page 212 makes it, and page 1,140, which is mostly about audit logs, does not. The
Scholar answers "Yes, SSO is included on Pro." The answer is wrong. On top of that, every query
pays for 40 shard calls, and one in three waits on a p99-slow shard.

Put simply, the post-filter threw away the page we needed, and the fan-out made us wait for it.

**Now, tenant sharding.** The router looks up Acme's shard and sends the query to that one
shard. Every candidate there belongs to the right tenant, so nothing is filtered away. Pages 212
and 1,140 both reach the top results. The Scholar reasons: "Page 212 says SSO is included. Page
1,140 adds the Security add-on rule for teams under 50 seats."

> "Yes, the Pro plan includes single sign-on [page 212]. Teams under 50 seats also need the
> Security add-on [page 1,140]."

The answer is correct, from one shard call instead of forty.

**Note:** Random sharding can be rescued with pre-filtering or in-filtering inside each shard
(Chapter 33). But when every query carries a tenant, tenant sharding removes the problem instead
of managing it.

---

## Sharding strategies compared

| Strategy | Where a chunk goes | Shards per query | Load balance | Main risk | Best for |
|---|---|---|---|---|---|
| **Random (hash)** | `hash(doc_id)` → virtual bucket → shard | All N | Even | Tail latency from fan-out | Global search, simplest operations |
| **Semantic (cluster)** | Nearest centroid | A few nearest | Uneven, drifts | Imbalance, boundary misses, rebalancing | Global search over very large corpora, compute-sensitive |
| **Tenant** | Its owner's shard | Exactly one | As uneven as tenants | Large tenants need their own shards | Queries always scoped to one customer |
| **Time-partitioned** (Ch. 59) | Its date's partition | Partitions in the time range | Recent partition is busiest | Queries that ignore time ask everything | Time-scoped queries dominate |

---

## When to use which one

We must use **tenant sharding** when every query is scoped to one tenant or workspace. Acme's
product is exactly this case.

We must use **random sharding** when search is global, load must be even, and operations must be
simple. Add hedged requests.

We must use **semantic sharding** when search is global, the corpus is very large, and compute
per query matters more than simplicity. Probe several shards and rebalance.

Many strong systems combine them. Partition by tenant, then shard the largest tenants randomly.

| Query pattern | Strategy |
|---|---|
| Always scoped to one tenant/workspace | **Tenant sharding** |
| Global search, even load, simplest ops | **Random sharding** + hedged requests |
| Global search, very large corpus, compute-sensitive | **Semantic sharding** with multi-probe |
| Time-scoped queries dominate | **Time-partitioned** shards (Chapter 59) |
| Mixed | Tenant or time at the top level, random within |

Composite schemes are normal.

---

## What is replication

**Replication = keeping several identical copies of each shard.** Each copy is a **replica**
(Chapter 57).

Sharding increases **capacity**. Replication increases **throughput and availability**.

- **Throughput:** any replica of a shard can serve its queries. Doubling replicas roughly
  doubles queries per second.
- **Availability:** losing a replica reduces capacity, not correctness.
- **Tail latency:** replicas make hedged requests possible.

A useful mental model: **shards scale with data, replicas scale with traffic.** Size them
independently. If Acme's corpus doubles, it needs more shards. If its traffic doubles, it needs
more replicas.

Vector indexes are read-heavy, which makes replication straightforward. Build or update on one
node, then ship the immutable index segment to the replicas.

---

### Under the hood

Scatter-gather with a deadline and hedging, the pattern most teams end up building:

```python
import asyncio

async def search_shard(shard, q, k, hedge_after=0.015):
    primary = asyncio.create_task(shard.replicas[0].search(q, k))
    backup = None
    try:
        done, _ = await asyncio.wait({primary}, timeout=hedge_after)
        if done:
            return primary.result()
        backup = asyncio.create_task(shard.replicas[1].search(q, k))   # hedge
        done, _ = await asyncio.wait({primary, backup},
                                     return_when=asyncio.FIRST_COMPLETED)
        return done.pop().result()
    finally:                              # runs on return and on cancellation
        for t in (primary, backup):
            if t is not None:
                t.cancel()                # stop the loser, or both if we were cancelled

async def scatter_gather(shards, q, k=100, deadline=0.080):
    tasks = [asyncio.create_task(search_shard(s, q, k)) for s in shards]
    done, pending = await asyncio.wait(tasks, timeout=deadline)
    for p in pending:
        p.cancel()                                   # accept partial results
    await asyncio.gather(*pending, return_exceptions=True)   # wait until they stop
    hits = {h.id: h for t in done for h in t.result()}       # de-dup copied vectors
    coverage = len(done) / len(shards)               # log this!
    return sorted(hits.values(), key=lambda h: -h.score)[:k], coverage
```

In simple words: ask the first replica, and if it has not answered in 15 ms, ask a second one
too and keep whichever wins. Across shards, wait at most 80 ms, then merge whatever arrived.
Cancelling a late shard runs its `finally` block, which cancels its replica searches too, so the
deadline really sheds load. The merge keys hits by id, so a boundary vector copied into two
shards counts once.

Log `coverage`, the share of shards that answered in time, on every query. A slow creep downward
is the earliest signal of a struggling shard, long before an alert on error rate.

---

### What people get wrong

**Sharding too early.** A single machine with 256 GB of RAM holds 100 million 768-d vectors in
int8 (about 93 GB, Chapter 60), or even more with TurboQuant compression. Sharding adds a network
hop, merge logic and tail-latency risk. Compress first (Chapter 31).

**Ignoring tail amplification.** At about 70 shards of fan-out, the shard p99 becomes the system
p50.

**Semantic sharding without rebalancing.** Hot shards emerge and stay.

**Merging scores from shards using different index parameters or quantization.** Scores must be
comparable across shards. Keep configurations identical, or rescore the merged candidates with
full-precision vectors.

**Requesting too few results per shard.** Ask each shard for more than the final `k`, especially
with semantic sharding, where the best results may concentrate in one shard.

---

### Ninja notes

**Rescore after merge.** Shards using compressed vectors return approximate scores. Collect
about 2–5× the final `k` across shards for int8, 10–20× for PQ (Chapter 25) or about 100× for
binary (Chapter 26), fetch full-precision vectors for the merged candidates from
the source of truth (Chapter 57), and rescore centrally. This removes cross-shard score
inconsistencies and recovers compression losses in a single step.

**Two-level routing is how the largest systems work.** Semantic routing picks shards. Within each
shard, an HNSW or DiskANN index does the fine-grained search. It is the IVF-then-graph cascade of
Chapter 23, with a network between the levels. The design question is how many shards to probe,
and it is tuned exactly like `nprobe`: sweep it against recall and latency on your eval set.

**Rebuild shards independently.** On forty machines at once, forty shards rebuild roughly forty
times faster than one large index, and each shard swap affects only a fraction of traffic. That
makes Chapter 59's maintenance and Chapter 63's model migrations far less risky.

**Hedging has a price tag.** Hedging after the shard's p95 duplicates roughly 5% of shard
requests. That extra load is why the hedge delay should sit near a high percentile, not near
the median.

---

### Key takeaways

- **Sharding = splitting one index into pieces on separate machines.** Shards scale capacity.
  Replicas scale throughput and availability. Size them separately.
- **Random sharding**: even and simple, but every query fans out and tail latency amplifies. At
  40 shards, about a third of queries hit at least one shard's p99.
- **Semantic sharding**: queries touch few shards. Beware imbalance, boundary misses and drift.
- **Tenant sharding**: free routing plus isolation, whenever queries are tenant-scoped.
- Use hedged requests and deadlines with partial results to control tail latency.
- Keep shard configurations identical, over-fetch per shard, and rescore after merging.
- Compress before you shard.

### What's next

The corpus never stops changing. [Chapter 59](./59-freshness.md) keeps the index current without
constant rebuilds.

We now know how one index becomes forty, how a query finds the right pieces, and why asking fewer
shards is almost always the better bargain.
