---
title: "Model Migration and Drift"
chapter: 63
part: "Part VIII — Scale and Production"
slug: migration-and-drift
readingTime: "13 min"
summary: "Vectors from different models live in different spaces, so changing models means re-embedding everything. How to do it without downtime, how to know when you should, and how to detect drift before users do."
tags: [migration, drift, re-embedding, versioning, blue-green, operations]
prev: 62-security-and-tenancy
next: 64-agent-memory
---

# Model Migration and Drift

**The one-paragraph version.** Vectors from two different models cannot be compared (Chapter
2), so changing the embedding model means re-embedding the entire corpus and rebuilding every
index. We do it as a **blue-green migration**: build the new index beside the old one, evaluate
offline, shadow live traffic, shift traffic gradually, and keep rollback instant. Separately, we
watch for **drift**, because the corpus and the queries change even when the model does not.

In this chapter, we will learn how to change the model behind a live search system without taking
search down, and how to notice drift before users do.

We will cover the following:

- What a model migration is
- Why old and new vectors cannot be mixed
- When to migrate
- How a migration works, phase by phase
- An example: the relevance floor after the switch
- What drift is, and how to detect it
- When to use which approach

---

## What a model migration is

**Migration = replacing the embedding model behind a live search system, together with every
vector that model produced.**

Every vector in our index is a coordinate in one particular Map Room, drawn by one particular
model. A new model draws a *different* Map Room. The new coordinates are not a simple shifted or
rotated copy of the old ones, so comparing them directly is meaningless. A mapping learned from
thousands of documents embedded by both models can only approximate the translation (see Ninja
notes).

Think of it like two libraries that number their shelves differently.

- Each library's numbering scheme is an embedding model.
- A book's shelf number is its vector.
- Shelf 42 in the old library holds cookery. Shelf 42 in the new library holds tax law.
- Knowing a book's old shelf number tells us nothing about its new shelf, unless someone builds a
  translation table by looking up many books in both libraries.

In simple words, vectors only make sense next to other vectors from the same model.

Our setting for this chapter is Acme's knowledge-base product at Part VIII scale: 5,000 customer
companies and about 100 million chunks. The team wants to move from the current model, which we
will call model A, to a newer model B.

---

## Why old and new vectors cannot be mixed

Let's first see the shortcut that looks sensible.

The team is in a hurry. They start embedding *new* pages with model B and adding them to model
A's index. They also switch the query encoder to model B, so new pages will match well. The old
pages can be re-embedded "gradually, over the next month".

A user asks: *"Does the Pro plan include single sign-on?"*

The question is embedded with model B. The Librarian compares it with page 212 and page 1,140,
which still hold model A vectors. Those numbers mean nothing in model B's Map Room. The nearest
pages turn out to be a refund policy and a page about annual invoices.

The Scholar reads them and writes: *"Acme's plans do not mention single sign-on."*

The answer is wrong. And nothing warned anyone. The scores looked normal, because a dot product
between vectors from two different models is still a perfectly ordinary number.

So we cannot:

- Compare a query embedded with model B against documents embedded with model A.
- Embed new documents with model B and add them to model A's index.
- Migrate "gradually" by re-embedding a few documents at a time into the same index.

Any of these produces rankings that look plausible and are nonsense, **without errors**.

**Note:** If the two models have different dimensions, say 768 and 1024, the database at least
refuses to compare them. The dangerous case is two models with the *same* dimension. Then every
comparison runs, and every result is meaningless.

Now, let's see the correct way. The team re-embeds all 100 million chunks with model B into a new
index. Queries embedded with model B go only to that index. The Librarian brings back pages 212
and 1,140. The Scholar answers: *"SSO is included on Pro, but teams under 50 seats need the
Security add-on."* The answer is correct.

**A migration is always a full re-embed and a full rebuild, run in parallel with the system it
replaces.**

---

## When to migrate

Migrations are expensive. At Acme's scale, re-embedding 100 million chunks is about 19
GPU-hours with a base-sized model, and about 140 with a 7B model (Chapter 60). Evaluation and a
careful rollout add weeks. So we need a reason:

- **Forced.** The provider deprecates our API model. Schedule the migration well before the
  deadline, not at it.
- **Measured quality gain.** A candidate model beats the current one on *our* golden set
  (Chapter 54), by a margin outside the confidence interval, on the segments that matter.
- **Capability.** We need multilingual support, longer inputs, Matryoshka truncation
  (Chapter 15), or multi-vector output.
- **Cost.** A smaller model matches quality at a fraction of the embedding and serving cost.

Not a reason: a leaderboard moved (Chapter 16).

---

## How a migration works, phase by phase

Four terms first.

**Blue-green = running the old system (blue) and the new system (green) side by side, then
moving traffic from blue to green with the option to move it straight back.**

**Backfill = re-embedding every existing document into the new vector store.**

**Dual-write = writing every new or changed document to both the old and the new pipelines
while the migration runs.**

**Rank correlation = a score for how similarly two ranked lists are ordered.** It is 1 when both
lists put the same pages in the same order, and near 0 when their orders have nothing in common.

In simple words, the backfill handles the past, dual-write handles the present, and blue-green
lets us switch without betting everything on one moment. Rank correlation tells us whether the
two systems agree on the order of results, not just on which pages come back.

The migration has six phases.

**Phase 0: Prepare.**

- **Step 1:** Record, for every vector, the model name, version, prefix scheme, pooling and dimensions.
- **Step 2:** Make each vector's id include the model version, as Chapter 57's `vector_id` does.

**Phase 1: Build.**

- **Step 1:** Turn on dual-write, so new documents reach both pipelines from now on.
- **Step 2:** Backfill: re-embed the whole corpus from the stored text into a new vector store.
- **Step 3:** Build the new indexes from those vectors.
- **Step 4:** Keep the old vectors. Never overwrite them.

**Phase 2: Evaluate.**

- **Step 1:** Run the golden set against old and new, broken down by segment.
- **Step 2:** Compare the scores with confidence intervals (Chapter 19).
- **Step 3:** Recalibrate every threshold for the new model: relevance floors (Chapter 55) and reranker cut-offs.

**Phase 3: Shadow.**

- **Step 1:** Copy a share of live queries to the new system. Users still see the old results.
- **Step 2:** For each query, log top-10 overlap, rank correlation, latency and errors.
- **Step 3:** Pull out the queries where the two systems disagree strongly, and read a sample by hand.

**Phase 4: Shift.**

- **Step 1:** Route 1% of real traffic to the new system.
- **Step 2:** Watch online signals: thumbs up and down, reformulated queries, escalations.
- **Step 3:** If they hold, move to 10%, then 50%, then 100%. If they worsen, roll back instantly.

**Phase 5: Retire.**

- **Step 1:** Keep the old system warm for a rollback window.
- **Step 2:** When the window closes, decommission the old indexes and vectors.

Here is the same plan in one block, for pinning to the wall:

```
Phase 0  PREPARE   version everything: model name, version, prefix scheme, pooling, dims
Phase 1  BUILD     re-embed corpus from source-of-truth text into NEW vector store
                   build NEW indexes; keep writing new docs to BOTH pipelines
Phase 2  EVALUATE  golden set on OLD vs NEW, by segment, with confidence intervals
Phase 3  SHADOW    mirror a % of live queries to NEW; compare results, latency, errors
Phase 4  SHIFT     route 1% → 10% → 50% → 100% of traffic; watch online signals
Phase 5  RETIRE    keep OLD warm for a rollback window; then decommission
```

**Phase 1 is where most of the cost and time goes.** Three details matter:

- **Re-embed from stored text, not from source systems.** Chapter 57's source of truth makes the
  backfill a batch job rather than a re-crawl of 5,000 customers' systems.
- **Dual-write during the build.** Documents created or changed while the backfill runs must land
  in both indexes, or the new index is stale on arrival.
- **Store the new vectors beside the old,** keyed by model version. Never overwrite.

**Phase 3 is the one teams skip and regret.** **Shadow traffic** means copies of real queries sent
to the new system, whose results are logged but never shown. Offline evaluation measures our
golden set. Shadowing measures the real mix of questions. The queries where the systems
*disagree strongly*, with low overlap or low rank correlation, are where regressions hide.

---

## An example: the relevance floor after the switch

The last step of Phase 2 is easy to skip. Let's see why it matters. The scores below are toy
numbers.

With model A, page 212 scored 0.83 for *"Does the Pro plan include single sign-on?"* The
relevance floor, calibrated with Chapter 55's sweep, was 0.60. Below that, the system says "I
couldn't find this."

Model B spreads its scores differently. For the same question, page 212 scores 0.55 and page
1,140 scores 0.52. Model B still ranks them first and second. Only the scale has moved.

Let's first see what happens when the team switches the encoder but keeps the old floor of 0.60.
Both pages fall below it. The system replies: *"I couldn't find any information about single
sign-on."*

The answer is wrong. The right pages were retrieved, ranked first, and then thrown away.

Now, let's see what happens after recalibration. The team re-runs Chapter 55's sweep on the
golden set with model B. The new floor comes out at 0.45. Pages 212 and 1,140 both pass. The
Scholar answers: *"SSO is included on Pro, but teams under 50 seats need the Security add-on."*
The answer is correct.

The same mistake can go the other way. If the new model scores *higher* across the board, an old
floor lets irrelevant pages through, and the Scholar fabricates more.

**Everything must switch together:** query encoder, document index, prefixes, pooling,
normalization, reranker thresholds, relevance floors and cache keys. Calibrated similarity values
belong to one model only (Chapters 2 and 55).

---

## What drift is, and how to detect it

**Drift = retrieval quality changing over time even though nobody changed the model.**

It comes in four kinds.

**Corpus drift.** New product lines, new terms, a new language, a new document type. Say Acme
launches passkey login. The model may represent the new content poorly, because it knows only
what it was trained on (Chapter 2). IVF centroids and PQ codebooks fitted to old data also go
stale (Chapters 23, 25).

**Query drift.** Users start asking different things, because of a launch, a policy change, a
seasonal pattern, or a new group of users.

**Index drift.** HNSW quality decays under heavy churn (Chapter 31), and shards drift out of
balance (Chapter 58).

**Dependency drift.** A library update changes tokenization, default pooling or normalization.
This one is sneaky. The model name is unchanged, so nobody suspects it.

Here is how to catch each kind:

| Signal | What it catches | How |
|---|---|---|
| Golden-set scores over time | Everything, on known queries | Nightly run (Ch. 54) |
| Index recall vs brute force | Index drift | Sampled nightly check (Ch. 31) |
| Top-1 score distribution | Corpus/query drift, "nothing relevant" rate | Track percentiles daily |
| Embedding distribution stats | Dependency drift, domain shift | Mean vector, norm, random-pair similarity |
| Online signals | Real user impact | Thumbs, reformulations, escalations |
| Unmatched query clusters | New topics users want | Cluster low-score queries weekly |

The cheapest detector of all is the **canary vector**. It works in four steps:

- **Step 1:** Pick about 100 fixed sentences, such as *"Does the Pro plan include single sign-on?"* and *"SSO-4012: SAML assertion expired"*.
- **Step 2:** At release, embed them and store the vectors as the reference.
- **Step 3:** On every deploy, and at every service start-up, embed them again.
- **Step 4:** Compare. Any change beyond floating-point noise means something in the embedding path changed.

We run this in **CI (continuous integration: the automated test run on every code change)**. If
a library upgrade quietly stops normalizing vectors, the canary vectors come out with a different
length, the comparison fails, and the upgrade never reaches production.

---

## When to use which approach

A full blue-green migration is not the only way to change what sits behind search. Here are the
three options:

| Approach | Quality | Cost | Use it when |
|---|---|---|---|
| **Full re-embed, blue-green** | Best | GPU-hours, double infrastructure for a while, weeks of rollout | We are changing the model |
| **Embedding adapter** | Below a true re-embed | A small model trained on a sample | We need a bridge during a long backfill or an unplanned deprecation |
| **Matryoshka truncation** | Same model, fewer dimensions | Rebuild only, no re-embed | We adopted a Matryoshka model and want smaller vectors |

**Advantages of a blue-green migration:**

- No downtime, and rollback is one routing change.
- Every decision is measured: offline, in shadow, then on a slice of real traffic.
- The old system stays intact until the new one has proven itself.

**Disadvantages:**

- **Problem 1: Double infrastructure.** While both systems run, we pay for both. An int8 HNSW
  index over 100 million vectors is about 90 GB per replica, so the overlap doubles that.
- **Problem 2: Weeks of calendar time.** Backfill, evaluation, shadowing and a gradual shift do
  not fit in a sprint at scale.
- **Problem 3: Dual-write complexity.** Every ingestion path must write to both systems, and a
  missed path leaves a gap.

We must use **a full blue-green migration** whenever the embedding model changes. We must use **an
adapter** only as a temporary bridge, never as the final state. We must use **Matryoshka
truncation** when the model stays the same and only the dimensions change. An adapter and a
blue-green migration also work well together: the adapter keeps search working while the backfill
runs.

---

### Under the hood

Two tools from this chapter: the canary check, and the shadow comparison from Phase 3.

```python
import time
from dataclasses import dataclass

import numpy as np

CANARY_TEXTS = load("canary_sentences.txt")                  # fixed, versioned
CANARY_REF   = np.load("canary_vectors_model_v3.npy")        # stored at release

def check_embedding_path(embedder, tol=1e-4):
    now = embedder.embed_documents(CANARY_TEXTS)
    drift = np.abs(now - CANARY_REF).max()
    if drift > tol:
        raise RuntimeError(f"embedding path changed: max abs diff {drift:.2e}. "
                           "Check tokenizer, pooling, normalization, dtype, library versions.")

@dataclass
class SearchResult:
    hits: list              # ranked hits, each with .id and .score
    latency_ms: float

def timed_search(system, query, k):
    t0 = time.perf_counter()
    hits = system.search(query, k)
    return SearchResult(hits=hits, latency_ms=(time.perf_counter() - t0) * 1000)

def shadow_compare(query, old_sys, new_sys, k=10):
    a = timed_search(old_sys, query, k)
    b = timed_search(new_sys, query, k)
    overlap = len({h.id for h in a.hits} & {h.id for h in b.hits}) / k
    # rank correlation and errors (Phase 3, Step 2) omitted for brevity
    log_shadow(query=query, overlap=overlap,
               old_top=a.hits[0].id if a.hits else None,
               new_top=b.hits[0].id if b.hits else None,
               old_ms=a.latency_ms, new_ms=b.latency_ms)
    return overlap
```

`check_embedding_path` embeds the canary sentences and finds the largest difference from the
stored reference. In simple words, if any number in any canary vector moved by more than 0.0001,
the build stops. Run it in CI and at service start-up. It takes a second and catches the whole
class of "we upgraded a dependency and search quietly got worse" incidents.

`shadow_compare` sends one query to both systems. `timed_search` wraps each call in a
`SearchResult`, which holds the ranked `hits` and the `latency_ms` it took. The overlap is the
share of the top 10 that both systems agree on. If both return pages 212 and 1,140 plus five
more in common, the overlap is 0.7. A production version also logs rank correlation and errors.

---

### What people get wrong

**Mixing vectors from two models in one index.** Silent nonsense, delivered with confidence.

**Re-embedding in place.** No rollback, and a window where the index is half one model and half
another.

**Forgetting to recalibrate thresholds.** Similarity values are model-specific. The old floor
either refuses good answers or waves bad ones through.

**Skipping shadow traffic.** The golden set cannot represent everything users ask.

**No dual-write during the build.** The new index launches missing a week of documents.

**Treating library upgrades as safe.** Tokenizer and pooling defaults do change, and they do not
announce it.

**Waiting for the deprecation deadline.** A full re-embed plus evaluation plus gradual rollout
takes weeks at scale. Start early.

---

### Ninja notes

**Embedding adapters can postpone a migration.** A small linear or MLP mapping (a tiny neural
network), trained on a sample of documents embedded by both models, can map new-model query
vectors approximately into the old space. The old index then keeps serving while a long backfill
runs. The same query-side map bridges an unplanned deprecation, when model A can no longer embed
queries. We can also map old document vectors forward into a temporary new index. Either way,
quality is below a true re-embed, so an adapter is never a permanent substitute.

**MRL models make some migrations free.** If we adopted a Matryoshka (MRL) model (Chapter 15) and
want to change dimensions, no re-embed is needed. Truncate the stored vectors, re-normalize, and
rebuild the index. That flexibility is a reason to prefer MRL models in its own right.

**Treat the model as a versioned dependency with a lifecycle.** Record for each deployed model:
adoption date, golden-set baseline, cost per million chunks, expected deprecation, and the date
of the last candidate comparison. Revisit quarterly (Chapter 59's maintenance calendar). Teams
that do this migrate on their own schedule. Teams that do not migrate under deadline pressure.

---

### Key takeaways

- **Migration = replacing the embedding model, together with every vector it produced.** Vectors
  from different models cannot be compared, so it is always a full re-embed and rebuild.
- Migrate for forced deprecations, measured gains on our own data, needed capabilities, or cost.
  Never for leaderboard movement.
- Run it blue-green in six phases: prepare, build, evaluate, shadow, shift, retire.
- Backfill the past, dual-write the present, and never overwrite the old vectors.
- Switch encoder, index, prefixes, thresholds and caches together. An old relevance floor on a
  new model gives wrong answers.
- **Drift = quality changing without a model change**: corpus, query, index and dependency drift.
- Canary vectors in CI catch dependency drift in seconds.
- Adapters can bridge a migration. MRL models make dimension changes free.

### What's next

Part VIII is complete. [Part IX](./64-agent-memory.md) begins with a problem that looks like
retrieval and behaves quite differently: giving agents long-term memory.

Now we have understood why a new model means a new Map Room, and how to move a live system into
it without downtime. We also know how to notice when the old Map Room slowly stops fitting the
world.
