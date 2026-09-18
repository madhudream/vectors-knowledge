---
title: "Filtered Vector Search"
chapter: 33
part: "Part IV — Index Structures"
slug: filtered-search
readingTime: "13 min"
summary: "Adding WHERE tenant_id = 42 to a vector search sounds trivial and is the hardest easy problem in the field. Three strategies, each broken in a different regime."
tags: [filtering, metadata, multi-tenancy, pre-filter, post-filter, acorn]
prev: 32-diskann
next: 34-single-vector-bottleneck
---

# Filtered Vector Search

**The one-paragraph version.** Real queries are almost never just *"find similar pages"*. They
are *"find similar pages **that belong to this customer, from this year, in English, not
archived**"*. Combining a metadata condition with an ANN search is genuinely hard, because the
index was built over the *whole* corpus, and the filter removes an arbitrary part of it. There are
three strategies: **post-filtering**, **pre-filtering** and **in-filtering**. Each one fails badly
in a different situation. Knowing which situation we are in is the entire skill.

In this chapter, we will learn how to search only the part of the Map Room a query is allowed to
see. We will measure every filter by its pass rate, watch each strategy succeed and fail on the
same question, and see why the best answer is often not to filter at all.

We will cover the following:

- What is filtered search
- Why filtering is hard
- Post-filtering
- Pre-filtering
- In-filtering
- Choosing a strategy by pass rate
- The best answer: partition instead of filter
- When to use which one

---

## What is filtered search

First, one word we need: metadata.

**Metadata** is the extra facts stored next to each vector: which company owns the page, its
language, its date, whether it is archived. Chapter 51 is all about it.

A **filter** is a yes/no test on that metadata, such as `tenant == "acme"`. Database people call
such a test a **predicate**. The two words mean the same thing here.

**Filtered search = vector search that only returns items whose metadata passes a filter.**

Every filter has one number that decides everything in this chapter.

**Pass rate = the fraction of the corpus a filter lets through.**

A filter that lets through 60% of the vectors has a pass rate of 0.6. A filter that lets through
one vector in a thousand has a pass rate of 0.001. A **selective** filter is one with a low pass
rate. In simple words, the lower the pass rate, the fewer items survive the filter.

**Note:** Many papers and databases call this number *selectivity*. Some use "highly selective" to
mean a low pass rate, and some use "selectivity 0.9" to mean a high one. That clash confuses
people, so this book says "pass rate" and means only one thing by it.

---

## Why filtering is hard

Let's take our scenario. Acme's knowledge-base product has grown up. It now hosts the knowledge
bases of **5,000 customer companies**, about **100 million** chunks (pieces of pages) in one
shared HNSW index. Acme's own 40,000-page support knowledge base is just one of those companies.
To keep the numbers simple, let's count Acme's knowledge base as 40,000 vectors.

Each customer company is a **tenant**. The rule is strict: one tenant must never see another
tenant's pages (Chapter 62 covers why).

A user on Acme's help centre asks:

> *"Does the Pro plan include single sign-on?"*

Lots of software companies have a "Pro plan" and write about SSO. So the Map Room holds
thousands of pages from other tenants that look almost exactly like Acme's pages 212 and 1,140.

Try it with no filter at all. The Librarian returns the 10 nearest pages from the whole index.
The top one belongs to another tenant: "SSO is included on our Pro plan for all team sizes." The
Scholar answers "Yes, for all team sizes." That is the wrong company's answer, and we just showed
Acme's user another company's page.

So we must add the filter `tenant == "acme"`. Its pass rate is 40,000 / 100,000,000 = **0.0004**,
or 0.04%.

So how do we combine that filter with an index that knows nothing about tenants?

Picture a huge shared library whose card catalogue is arranged by topic. We want books on
our topic, but only from one owner's shelf. We have three options:

- **Search, then filter.** Pull the best books on the topic, then throw away the ones from other
  owners. If our owner has very few books, we end up with nothing.
- **Filter, then search.** First list every book our owner has, then compare only those. If the
  owner has four million books, making and scanning that list is slow.
- **Filter while searching.** Walk the catalogue as usual, but only keep books from our owner. That
  works, unless our owner's books are so scattered that the walk wanders through other shelves
  forever.

Those three options are the three strategies. Each has a situation where it is clearly correct.

---

## Post-filtering

**Post-filter = search the whole index first, then throw away the results that fail the filter.**

- **Step 1:** Run the normal ANN search and fetch many candidates.
- **Step 2:** Drop every candidate that fails the filter.
- **Step 3:** Return the first `k` survivors.

```python
candidates = index.search(q, k=1000)                         # ignore the filter
results = [c for c in candidates if c.tenant == "acme"][:10]
```

**Cheap and simple.** It uses the index exactly as designed, with no special support.

**It breaks when the filter is selective.** Let's run our question. We fetch 1,000 candidates.
If Acme's pages were spread evenly among them, we would expect 1,000 × 0.0004 = 0.4 Acme pages.
About two queries in three come back with zero Acme pages. Most of the rest come back with one.

So the Scholar gets nothing, or gets page 212 alone and says "Yes, SSO is on Pro". Either way,
the add-on rule is lost.

The naive fix is "just fetch more". It has a hard floor. If the filter's pass rate is $p$, we need
at least $k / p$ candidates just to *expect* `k` survivors, and about $2k / p$ to reliably get
them. For Acme, with $k = 10$, that is 50,000 candidates, and the search is no longer cheap. At a
pass rate of 0.001 it is 20,000. At 0.00001, post-filtering is hopeless.

Why $2k / p$ rather than $k / p$? Let's check at a pass rate of 0.3 with $k = 10$. Fetching
$3k = 30$ candidates gives 9 survivors on average, and in a simulation came up short on about 6
queries in 10. Fetching $2k / p = 66$ gives about 20 on average, and came up short on fewer than 2
queries in 1,000.

In simple words, fetching exactly what we expect to need comes up short nearly half the time.
Fetching double is safe.

**Use when:** the pass rate is above ~30% (10–30% also works, with a larger fetch).

---

## Pre-filtering

**Pre-filter = find the allowed set first using the metadata, then search only inside it.**

- **Step 1:** Ask the metadata index for the IDs of every item that passes the filter.
- **Step 2:** Compare the query with every allowed vector, by brute force (Chapter 17).
- **Step 3:** Return the top `k`.

```python
allowed = metadata_index.lookup(tenant="acme")               # an array of IDs
top = brute_force_search(q, vectors[allowed], k=10)          # exact, subset positions
results = allowed[top]                                       # positions -> page IDs
```

Run our question this way. The allowed set is Acme's 40,000 vectors. Brute force over 40,000
vectors takes about a millisecond on a few cores. It compares the question with every Acme
vector, so page 212 and the SSO chunk of page 1,140 (Chapter 18) both come back in the top 10. No
other tenant's page is even looked at.

The Scholar answers: "SSO is included on Pro, but teams under 50 seats need the Security add-on."
This time the answer is complete and comes only from Acme's pages.

**Exact and correct.** We get the true top-k of the filtered set, guaranteed.

**It breaks when the allowed set is large.** Suppose the filter is `language == "en"`, passing 70
million vectors. Brute-forcing 70 million vectors is the very problem the ANN index existed to
solve.

**Use when:** the allowed set is below roughly 50,000 vectors (anywhere from 10,000 to 100,000,
depending on hardware). Recall Chapter 17: brute force over 50,000 vectors takes a millisecond or
two on a few cores. Many "filtered vector search" problems turn out to be this case, and it is
both exact and simple.

---

## In-filtering

**In-filter = apply the filter during the graph walk itself.** Some people call it filtered
traversal.

- **Step 1:** Walk the HNSW graph as usual (Chapter 30).
- **Step 2:** When the walk reaches a node that passes the filter, it may enter the results.
- **Step 3:** When it reaches a node that fails the filter, it never enters the results, but the
  walk still steps through it to reach its neighbours.
- **Step 4:** Stop by the usual rule, and return the top `k` results.

```
walk the graph as usual
  ↳ node passes filter?  → may enter the results
  ↳ node fails filter?   → never a result, but still expand its neighbours
```

**This is what modern vector databases implement.** It is the only strategy that works in the
awkward middle. There, the pass rate is below about 10%, and the allowed set is too big to brute
force.

For example, Globex, one of Acme's largest customers, has 2 million chunks: a pass rate of 2%.
Pre-filtering means brute-forcing 2 million vectors. Post-filtering means fetching about 1,000
candidates for every query. In-filtering walks the graph once and collects Globex pages as it
meets them.

**But it has a real failure mode, and it is worth understanding.** The graph's links were built
to connect *similar* pages, with no idea of tenants. Suppose one tenant's pages are scattered
thinly across the whole graph. Then getting from one of its pages to the next means walking
through many pages that fail the filter.

The walk does far more work than the result count suggests. In the worst case, the tenant's pages
are **disconnected**: some can only be reached through regions the search never explores. Recall
collapses, and nothing reports an error.

Systems handle this with a routing policy: estimate the filter's pass rate first, then choose the
strategy. Most mature databases do exactly this.

---

## Choosing a strategy by pass rate

We check the size of the allowed set first, then the pass rate:

| Situation | Strategy | Why |
|---|---|---|
| Allowed set < ~50k vectors | **Pre-filter + brute force** | Set is small, so exact and fast |
| Pass rate > 30% | Post-filter, fetch $2k$ / pass rate | Cheap, plenty of survivors |
| Pass rate 10–30% | In-filter, or post-filter with a larger fetch | Either works |
| Pass rate < 10%, allowed set > 50k | **In-filter** | Post-filter starves, pre-filter set too big |
| Always one value (e.g. tenant) | **Partition, see below** | Best answer of all |

The absolute size is checked first on purpose. At 100 million vectors, a pass rate of 0.04% is
Acme's 40,000 vectors: brute force it. At a billion vectors, the same 0.04% is 400,000 vectors, and
in-filtering is the better choice. At pass rates that low, plain in-filtering is where
disconnection bites, so prefer ACORN-style traversal (Ninja notes) or a partition.

The practical procedure: **measure your filters' pass-rate distribution.** Most teams have never
done this and are surprised by the result. Commonly, a handful of filters let through almost
nothing and the rest barely filter at all. So one strategy serves 95% of traffic, and a special
case handles the rest.

---

## The best answer: partition instead of filter

If a filter appears in every single query, why filter at all?

If a filter appears in **every query** and has a moderate number of distinct values (tenant ID,
workspace, language, region), we should not filter. **We build a separate index per value.**

```
index_acme      ← 40k vectors, brute force or a small HNSW
index_globex    ← 2M vectors, its own HNSW
index_tinyco    ← 800 vectors, brute force
```

For our question, the Librarian simply opens `index_acme` and searches it normally. Every result
is an Acme page. Pages 212 and 1,140 come back. No filter was involved.

Within this pattern, the advantages are large:

- The filter becomes **routing**, not filtering. It costs nothing.
- Each index is small, so search is fast and recall is high.
- Small tenants use brute force and get exact results.
- **Hard isolation.** Acme's vectors are not in Globex's index. That is a security property, not
  just a speed one (Chapter 62).
- Deleting a tenant is deleting an index.

The costs are real but manageable: many small indexes to operate, memory overhead per index, and
awkwardness if there are enormous numbers of values (a million tenants) or if queries must span
partitions. The standard fix is a **hybrid**: dedicated indexes for large tenants, and one shared
filtered index for the long tail of small ones.

> **If you take one thing from this chapter:** most filtered-search pain comes from trying to
> solve with an index what should have been solved with a partition.

---

## When to use which one

| | Post-filter | Pre-filter | In-filter | Partition |
|---|---|---|---|---|
| Works best at | pass rate > 30% | allowed set < ~50k | pass rate < ~10%, set too big to brute force | filter in every query |
| Exact? | no | yes | no | depends on index |
| Needs special support? | no | a metadata index | a filter-aware index | many indexes |
| Fails by | returning too few | slow scans | disconnected subgraph | too many tiny indexes |

We must use **post-filtering** when the filter lets most things through, like "not archived".

We must use **pre-filtering** when the allowed set is small, like Acme's 40,000 vectors inside a
shared index.

We must use **in-filtering** when the allowed set is too big to scan but too small to find by
post-filtering, like Globex's 2 million.

We must use **partitions** when every query carries the same kind of filter, like a tenant ID.

Many strong systems use all four: partition by tenant, then route each remaining filter by its
pass rate.

---

### Under the hood

Pass-rate-aware routing is roughly what a good vector database does internally, and we can build
it in a service layer.

- **Step 1:** Estimate how many vectors pass the filter.
- **Step 2:** If fewer than 50,000 pass, look up their IDs and brute force them.
- **Step 3:** Otherwise, compute the pass rate.
- **Step 4:** If the pass rate is above 0.3, post-filter, fetching `2k / pass_rate` candidates.
- **Step 5:** Otherwise, run an in-filter search.

```python
def filtered_search(q, predicate, k=10):
    n_matching = metadata_index.estimate_count(predicate)   # Step 1

    if n_matching < 50_000:                                 # Step 2: small set
        ids = metadata_index.lookup(predicate)
        return brute_force(q, ids, k)                       # exact

    pass_rate = n_matching / corpus_size                    # Step 3
    if pass_rate > 0.3:                                     # Step 4: weak filter
        fetch = int(2 * k / pass_rate)                      # enough survivors, reliably
        return [c for c in ann.search(q, fetch) if predicate(c)][:k]

    return ann.search_filtered(q, k, predicate)             # Step 5: in-filter traversal
```

For Acme's question, `estimate_count` returns 40,000, so Step 2 brute-forces Acme's pages and
returns both 212 and 1,140. For Globex, it returns 2 million with a pass rate of 0.02, so the
query goes to Step 5.

Note `estimate_count`. We need these counts to route at all. A small inverted index (Chapter 7) or
a set of **bitmaps** over the metadata gives them cheaply. A bitmap is one bit per vector, set to 1
if the vector passes. **Roaring bitmaps** are the standard compressed version. They intersect
several filters fast and give exact counts, which is precisely what this decision needs.

In simple words, count first, then pick the strategy that fits the count.

---

### What people get wrong

**Assuming filtered search is free.** It is the most common source of "the demo was fast,
production is slow".

**Post-filtering with selective filters.** Returns too few results, or none. The symptom is a
result list that is mysteriously short.

**Not measuring recall *under filters*.** Your unfiltered recall@10 may be 0.98 while your recall
for a filter with a 1% pass rate is 0.55. These are different measurements, and most teams only
make the first. **Build filtered queries into your eval set** (Chapter 19).

**Using in-filtering with a selective, scattered filter.** The filtered subgraph may be
disconnected. Recall drops and nothing reports an error.

**Over-partitioning.** Ten thousand indexes of 200 vectors each is worse than one index with a
filter. Partition on high-value attributes with a moderate number of values only.

---

### Ninja notes

**ACORN** (Patel et al., SIGMOD 2024) is the notable recent advance and worth knowing by name.
The core idea: during filtered traversal, instead of only expanding a node's direct neighbours,
expand its **two-hop** neighbourhood when the direct neighbours fail the predicate. This
effectively simulates a graph built over the filtered subset. The filtered subgraph is far more
likely to stay connected, because you step over the non-matching nodes instead of being blocked
by them. Its main variant also builds a denser graph at construction time, so there are enough
neighbours to step over. It substantially improves recall on selective filters at modest extra
cost, and implementations are appearing in mainstream libraries.

**Filter-aware graph construction** is the other direction: build the index so that edges
preserve connectivity within likely filter values. If you know most queries filter by tenant,
ensure each node has edges to same-tenant neighbours *as well as* global ones. This has been
published and shipped. Filtered-DiskANN (Gollapudi et al., WWW 2023) builds edges from both the
vectors and their labels, and Qdrant adds extra HNSW edges for each indexed payload value. You pay
for the extra edges in memory and build time, and get much better filtered recall. It is often a
large win for multi-tenant products.

**Time-based filters deserve special handling.** "Last 30 days" is extremely common and extremely
awkward as a predicate. Partition by time instead, with daily or weekly indexes queried as a set.
Old partitions become immutable, so they can be compressed harder, moved to cheaper storage, or
dropped entirely. This single architectural choice removes a whole class of filtering pain, and
Chapter 59 shows how it also simplifies index maintenance.

---

### Key takeaways

- **Filtered search = vector search that only returns items whose metadata passes a filter.**
- **Pass rate = the fraction of the corpus a filter lets through.** A selective filter has a low
  pass rate.
- Pre-filter and brute force when the allowed set is small (< ~50k vectors). Post-filter when the
  pass rate is high (> 30%), fetching about $2k$ / pass rate. In-filter for the middle.
- Check the absolute size of the allowed set before the pass rate.
- Route by *estimated pass rate* instead of committing to one strategy.
- In-filtering can silently disconnect the filtered subgraph and collapse recall.
- Partitioning beats filtering whenever a filter appears in every query with a moderate number of
  values. It turns filtering into routing.
- Measure recall *under filters*. Unfiltered recall tells you nothing about it.
- ACORN's two-hop expansion and filter-aware construction (Filtered-DiskANN, Qdrant's payload
  edges) keep filtered graph walks far better connected.

### What's next

Part IV is complete. We can index and search single vectors at any scale.
[Part V](./34-single-vector-bottleneck.md) asks whether one vector per document was ever a good
idea, and what becomes possible when we stop insisting on it.

We now understand why a filter makes vector search hard, how the pass rate decides between post-,
pre- and in-filtering, and why a partition is often the best filter of all.
