---
title: "Freshness and Index Maintenance"
chapter: 59
part: "Part VIII — Scale and Production"
slug: freshness
readingTime: "11 min"
summary: "Documents change every minute, and indexes are expensive to modify. The log-structured pattern that reconciles them — deltas, tombstones, compaction — and the maintenance calendar that keeps it healthy."
tags: [freshness, updates, deletes, compaction, lsm, maintenance, cdc]
prev: 58-sharding-and-routing
next: 60-cost-engineering
---

# Freshness and Index Maintenance

**The one-paragraph version.** High-quality ANN indexes are expensive to modify in place
(Chapter 31), but documents change constantly. We resolve the conflict the way databases have
for decades. New data goes into a **small, fast, mutable** index. Deletions are recorded as
**tombstones**. Every query searches both the big index and the small one. And every so often,
we **compact** them into a fresh big index. Then we run a maintenance calendar, because an index
nobody maintains quietly decays.

In this chapter, we will learn what freshness means, why an updated page can keep returning its
old text, and how the delta-plus-tombstone pattern fixes that. We will also see how to detect
changes, how fresh each kind of content needs to be, and what to schedule so the index stays
healthy.

We will cover the following:

- What is freshness
- Why we need freshness: page 212 changes
- How the log-structured pattern works
- An example: making the old page 212 disappear
- Delta pattern vs the alternatives
- Detecting change
- Freshness budgets
- The maintenance calendar

---

## What is freshness

**Freshness = how quickly a change in the source shows up in search results.**

Its opposite is **staleness**: the time during which search can still return something that is
no longer true.

Before going further, we must recall one term and learn three more.

A **tombstone** (Chapter 31) is a mark that says "this id is deleted, skip it". We record the
mark instead of actually removing the item from the index, because removing it is expensive.

**Delta index = a small, mutable index that holds everything added since the last rebuild.**

**Main index = the large, optimised index, which we never modify.** Immutable, in other words.

**Compaction = building a fresh main index that includes the delta and leaves out every
tombstoned item, then swapping it in.**

Think of it like a newspaper and its corrections. A newspaper prints once a day. It cannot
reprint the whole edition when a fact changes at noon. So it publishes a **correction** in
tomorrow's edition, and readers apply the correction to what they read yesterday. Every so
often, a compiled **annual edition** folds all the corrections in.

- **Yesterday's paper** = the main index. Large, optimised, immutable.
- **Today's corrections** = the delta index and the tombstone set. Small, mutable, recent.
- **The annual edition** = compaction. Everything folded into a new main index.

This is the **log-structured pattern**. It underlies **LSM trees** (log-structured merge trees:
storage where new data lands in small fresh segments that are merged into bigger ones later). It
also underlies Lucene's segments (Lucene is the search library inside Elasticsearch),
Fresh-DiskANN (Chapter 32), and the delta-index pattern of Chapter 31.

---

## Why we need freshness: page 212 changes

Suppose Acme changes its pricing on 1 October. Single sign-on moves from the Pro plan to a new
Business plan. This change is imaginary, made up for this chapter only. Everywhere else in the
book, SSO stays on Pro. Two pages change that day:

- **Page 212** is rewritten: "From 1 October, SSO is included on the Business plan. It is no
  longer part of Pro."
- **Page 1,140**, the Security add-on page, is deleted, because Acme retires the add-on.

The ingestion pipeline from Chapter 57 sees the change. It chunks and embeds the new page 212,
and inserts the new vector. Chunk ids include the content hash, so the new text gets a new id.

And that is the problem. **Nothing removed the old vectors.** The old page 212 and the old page
1,140 are still in the index.

A user asks *"Does the Pro plan include single sign-on?"* The search returns:

| Rank | Chunk | Text | Score |
|---|---|---|---|
| 1 | old page 212 | "SSO is included on the Pro plan." | 0.91 |
| 2 | new page 212 | "From 1 October, SSO is included on the Business plan…" | 0.84 |
| 3 | old page 1,140 | "On Pro, SSO needs the Security add-on for teams under 50 seats." | 0.80 |

The old page 212 wins, because it repeats the question's words. The Scholar writes: "Yes, SSO is
included on the Pro plan [page 212]. Teams under 50 seats also need the Security add-on [page
1,140]."

The answer is wrong. Worse, the citations look right, because the stale chunks carry the same
page numbers as the real ones.

Now, the question is, how do we make the old vectors stop coming back, right now, without
rebuilding a 100-million-vector index every time a page changes?

---

## How the log-structured pattern works

```
                         ┌──────────────────────────┐
  writes ──────────────► │ DELTA INDEX              │  small, flat or small HNSW
  (new/updated chunks)   │ minutes-to-hours of data │  instantly searchable
                         └──────────────────────────┘
                         ┌──────────────────────────┐
  deletes / updates ───► │ TOMBSTONE SET            │  ids to exclude everywhere
                         └──────────────────────────┘
                         ┌──────────────────────────┐
                         │ MAIN INDEX (per shard)   │  large, optimised, immutable
                         │ rebuilt on a schedule    │
                         └──────────────────────────┘

  query ──► search MAIN and DELTA in parallel ──► drop tombstoned ids ──► merge ──► top-k
```

**Phase 1: A page changes (the write path).**

**Step 1:** A change event arrives, from CDC or a webhook (Chapter 57).

**Step 2:** Look up the chunk ids the page produced last time.

**Step 3:** Add those ids to the tombstone set. From this moment, they are skipped everywhere.

**Step 4:** If the page was updated rather than deleted, chunk and embed the new version.

**Step 5:** Add the new chunks to the delta index. They are searchable immediately. If a new
chunk's id is already a tombstone (a writer undid an edit, so the old text is back), take it off
the tombstone set.

**Phase 2: A query arrives (the read path).**

**Step 1:** Search the main index and the delta index in parallel, asking each for a few more
than k.

**Step 2:** Drop every result whose id is in the tombstone set.

**Step 3:** Merge by score, keep one hit per id, and keep the top k.

**Phase 3: Compaction (on a schedule, or when the delta grows too big).**

**Step 1:** Record the time, and take a snapshot of the source of truth (Chapter 57). It already
contains every current chunk and none of the deleted ones.

**Step 2:** Build a new main index from that snapshot.

**Step 3:** Verify it on the eval set.

**Step 4:** Replay the writes and deletes that arrived after the snapshot.

**Step 5:** Swap it in atomically (in one step, so no query ever sees half of each). Clear the
delta and the tombstones the new index now covers.

Three rules fall out of this.

**Updates are delete-plus-insert.** Tombstone the old chunk ids and write the new chunks to the
delta. Never modify a chunk in place.

**The delta stays small**, so brute force over it is fast and exact (Chapter 17). Twenty
thousand vectors scan in about a millisecond.

**Compaction builds from the source of truth**, never by patching the old index. That is why
Chapter 57 insisted the index must never be the only copy.

In simple words, we never edit the big index. We add to a small one, cross things out in a list,
and every so often we print a clean new big index.

---

## An example: making the old page 212 disappear

Let's replay our imagined 1 October with the pattern in place.

**Step 1:** The webhook for page 212 fires. Its old chunk id goes into the tombstone set. The new
page 212 is embedded and added to the delta index.

**Step 2:** The webhook for page 1,140 fires. It is a deletion, so its chunk id goes into the
tombstone set, and nothing is added.

**Step 3:** A user asks *"Does the Pro plan include single sign-on?"* The main index still
returns the old page 212 (0.91) and the old page 1,140 (0.80). The delta returns the new page
212 (0.84).

**Step 4:** Both old ids are in the tombstone set, so both are dropped. The new page 212 is now
the top result.

The Scholar reasons: "Page 212, updated on 1 October, says SSO is on the Business plan and no
longer part of Pro." It answers:

> "No. Since 1 October, single sign-on is included on the Business plan, not on Pro
> [page 212]."

The answer is correct, seconds after the change, and the 100-million-vector main index was never
touched. Tonight's compaction will build a main index without the old vectors at all.

**Note:** The index can only be as fresh as the pages. If Acme's writers had forgotten to update
or delete page 1,140, the Librarian would still, correctly, return it.

---

## Delta pattern vs the alternatives

| | Modify HNSW in place | Full rebuild on every change | Delta + tombstones + compaction |
|---|---|---|---|
| Time until a change is visible | Immediate | Hours | Immediate |
| Deletes | Tombstones pile up inside the graph | Clean | Tombstone set, cleared at compaction |
| Recall over time | Decays with churn (Ch. 31) | Stable | Stable |
| Query cost | One index | One index | Two indexes plus a merge |
| Operational effort | Low at first, rebuilds later anyway | Very high | Moderate, and scheduled |

We must use **in-place updates** only when churn is light and a periodic rebuild is already
planned. We must use **full rebuilds** only for small corpora or rarely changing archives. For
everything else, especially large indexes with steady churn, we use **the delta pattern**. Many
strong systems add time partitions on top (see Ninja notes).

---

## Detecting change

Freshness starts upstream. There are three sources of change events, in order of preference.

**1. Change-data-capture (CDC) and webhooks.** The source system tells us when something changed.
Lowest latency, lowest cost. Use it wherever it exists.

**2. Incremental crawls with content hashing.** Re-fetch periodically, hash the content, and
reprocess only if the hash changed. The idempotency of Chapter 57 makes this cheap. A crawl of
the product's 12.5 million documents, if none changed, re-fetches and hashes each one but embeds
nothing.

**3. Full re-crawls.** Expensive, but necessary occasionally to catch deletions that no event
reported. **Deletions are the change type most often missed**, because a deleted document
generates no content to notice. So we reconcile periodically. List what exists at the source,
compare it with what is indexed, and tombstone the difference.

---

## Freshness budgets

"Real-time" is rarely the real requirement. Decide explicitly, per content type:

| Content | Acceptable staleness | Approach |
|---|---|---|
| Support tickets, chat | Seconds–minutes | CDC → delta index |
| Wiki, documentation, pricing pages | Minutes–hours | Webhooks → delta, nightly compaction |
| Policies, contracts | Hours | Hourly incremental crawl |
| Archives, historical reports | Days–weeks | Weekly batch |
| **Deletions with legal or privacy implications** | **Minutes, guaranteed** | **Tombstone synchronously** |

The last row is not like the others. When a user exercises a deletion right, or a document is
withdrawn for legal reasons, it must stop appearing *now*. And it must be gone from caches,
deltas, replicas, backups and the source-of-truth vector store, not merely hidden. Design
deletion as a first-class, audited path (Chapter 62).

---

## The maintenance calendar

An index is infrastructure. Schedule its upkeep.

| Frequency | Task | Why |
|---|---|---|
| Continuous | Monitor delta size and tombstone ratio | Early warning (Ch. 31) |
| Hourly–daily | Compact deltas into main index | Bound delta size and query cost |
| Nightly | Recall check on sampled queries; eval-set regression run | Catch silent decay (Ch. 54) |
| Weekly | Reconcile against source to catch unreported deletions | Missed deletes accumulate |
| Monthly | Retrain IVF centroids / PQ codebooks if used; check shard balance | Drift (Ch. 23, 25) |
| Quarterly | Review embedding model options; re-run model comparison on eval set | Chapter 63 |

The recall check deserves emphasis: **index decay is silent.** Nothing errors. Recall drops a
point a month, and users describe the product as "getting worse" without being able to say why.
The nightly check is what turns an invisible trend into a graph.

---

### Under the hood

The query side of the pattern, following the Steps above:

```python
class FreshIndex:
    def __init__(self, main, dim):
        self.main = main                      # large immutable ANN index
        self.delta = FlatIndex(dim)           # Ch. 17
        self.tombstones = set()

    def upsert(self, chunk_ids_to_remove, new_chunks):
        self.tombstones.update(chunk_ids_to_remove)
        self.tombstones.difference_update(new_chunks.ids)    # a revert brings old ids back
        self.delta.add(new_chunks.vectors, new_chunks.ids)

    def delete(self, chunk_ids):
        self.tombstones.update(chunk_ids)     # synchronous: effective immediately

    def search(self, q, k=10):
        over = k + min(len(self.tombstones), 4 * k)          # over-fetch
        hits = self.main.search(q, over) + self.delta.search(q, over)
        live = {h.id: h for h in hits if h.id not in self.tombstones}   # one hit per id
        return sorted(live.values(), key=lambda h: -h.score)[:k]

    def compact(self, source_of_truth):
        new_main = build_index(source_of_truth.current_vectors())   # excludes deleted
        verify(new_main, eval_set)                                 # never skip
        self.main = new_main                                       # atomic swap
        self.delta = FlatIndex(self.delta.dim)
        self.tombstones = set()
```

On our imagined 1 October, `upsert` handles page 212 and `delete` handles page 1,140. Note the
over-fetch in `search`. If tombstones exist, some results from the main index will be filtered out,
so we request more than `k` to make it likely that `k` survive. If fewer do, search again with a
larger `over`.

In simple words, `search` asks for extra, crosses out the dead, and keeps the best of what is
left.

Now say a writer undoes the 1 October edit an hour later, before tonight's compaction. Chunk ids
come from the content hash, so the restored text gets back the very ids we tombstoned. Without the
`difference_update` line, the restored page 212 would be added to the delta and then crossed out
by `search`, so no version of page 212 could be found until tonight's compaction. With it, the old
ids are live again. They now sit in both the main index and the delta, which is why `search` keeps
one hit per id.

In a real system, tombstones and delta writes that arrive *during* compaction must be replayed
onto the new index before the swap (Phase 3, Step 4). Record the timestamp at which the
compaction snapshot was taken, and replay everything after it. This sketch skips that step for
brevity.

---

### What people get wrong

**Modifying HNSW in place under heavy churn.** Tombstones accumulate, memory grows, recall decays
(Chapter 31). Use the delta pattern.

**Letting the delta grow unbounded.** Brute force over a million-vector delta is no longer cheap.
Compact on size as well as on schedule.

**Missing deletions.** No event fires when a file disappears. Reconcile.

**Forgetting caches.** A deleted document can survive in a result cache for hours. Key caches
with a corpus version, or invalidate on delete.

**Compacting without verification.** A buggy build that silently drops a shard of documents looks
exactly like a successful one until users notice.

**"Real-time everything."** Expensive and usually unnecessary. Set staleness budgets per content
type.

---

### Ninja notes

**Time-partitioned indexes simplify all of this** for corpora dominated by appends: tickets,
logs, messages, news. Build one index per day or week. Old partitions become **immutable
forever**. They never need compaction, can be compressed aggressively, can move to cheaper
storage tiers (Chapter 57), and can be deleted wholesale when retention expires. Only the current
partition is mutable. Recency filters become partition selection rather than predicate filtering
(Chapter 33).

**Version the corpus, not just the index.** Stamp every query log entry with the corpus version
that served it. When quality changes, you can answer "was it the data or the code?" immediately.
You can also replay historical queries against historical corpus versions to reproduce a reported
problem exactly.

**Tombstone by document, not only by chunk.** Keep a map from each document id to the chunk ids
it produced. Then a page update or deletion is one lookup, and no orphaned chunk from an older
chunker version survives by accident.

---

### Key takeaways

- **Freshness = how quickly a change in the source shows up in search.** The log-structured
  pattern delivers it: immutable main index, small mutable delta, tombstones, periodic
  compaction.
- Updates are delete-plus-insert. Deletions are tombstones that take effect immediately.
- Without tombstones, an updated page keeps returning its old text, with a citation that looks
  right.
- Detect change via CDC or webhooks first, content-hashed crawls second, and reconcile
  periodically, because deletions often go unreported.
- Set explicit staleness budgets per content type. Treat legal and privacy deletions as
  synchronous and complete.
- Compact from the source of truth, verify before swapping, and replay writes that arrived
  during the build.
- Keep a maintenance calendar with a nightly recall check. Index decay is silent.
- Time-partitioned indexes eliminate most maintenance for append-heavy corpora.

### What's next

[Chapter 60](./60-cost-engineering.md) puts a price on everything in this book: per million
vectors, per query, per month.

We now know how to keep a huge index truthful while its pages change underneath it, and which
upkeep to put on the calendar so it stays that way.
