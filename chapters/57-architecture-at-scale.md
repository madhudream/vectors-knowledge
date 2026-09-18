---
title: "Architecture for Millions of Documents"
chapter: 57
part: "Part VIII — Scale and Production"
slug: architecture-at-scale
readingTime: "10 min"
summary: "The reference design for RAG over millions of documents: separate ingestion from serving, keep a source of truth outside the index, and treat every index as rebuildable."
tags: [architecture, scale, rag, ingestion, serving, system-design]
prev: 56-agentic-retrieval
next: 58-sharding-and-routing
---

# Architecture for Millions of Documents

**The one-paragraph version.** At scale, design principles matter more than any single
component choice. **Separate ingestion from serving**, so a slow embedding job never touches
query latency. **Keep a durable source of truth** (raw documents, extracted text and
full-precision vectors) outside the index, so every index is disposable and rebuildable. **Make
every stage idempotent and versioned**, so reprocessing is routine rather than an incident.
Everything else is detail.

In this chapter, we will learn the reference design for a RAG system over a hundred million
chunks: the ingestion side, the serving side, and the boundary between them. We will also see
the four principles behind it, what they save when something goes wrong, and the throughput
numbers that decide capacity.

We will cover the following:

- What changes at scale
- What is the reference design
- The four principles
- An example: fixing a chunking bug at 100 million chunks
- Throughput arithmetic
- When to use which one

---

## What changes at scale

Part VIII uses a bigger version of our running example. Acme has turned its knowledge base into
a product. It now hosts knowledge bases for **5,000 customer companies**, holding about **100
million chunks** in total. Acme's own 40,000-page support library is just one of them.

Each customer is a **tenant**: a customer whose pages must stay separate from every other
customer's. One customer must never see another's pages (Chapter 62 covers how to guarantee
that).

At 40,000 pages, a RAG system is a script. Read the files, embed them, insert them, query. That
is a few hundred thousand chunks, which one GPU embeds in a few minutes. Everything lives in one
process, and rebuilding is a coffee break.

At 100 million chunks, that script becomes a liability in five specific ways:

- Embedding the corpus takes **hours to days**, not minutes.
- A bug in chunking means **reprocessing everything**, and you will find such bugs.
- The embedding model **will be deprecated or superseded** (Chapter 63).
- Documents **change continuously**. Five thousand customers edit pages all day.
- The index **outgrows one machine**. A float32 HNSW index over 100 million 768-d vectors is
  about 323 GB of RAM (Chapter 60), before a single spare copy.

So the architecture must assume, from the start, that every derived artefact will be rebuilt
more than once. That one assumption drives every decision below.

---

## What is the reference design

Before the diagram, we must know a few terms.

**Ingestion pipeline = the chain of steps that turns a raw document into searchable vectors:
fetch, extract, chunk, enrich, embed, and load into the index.**

To **enrich** a chunk is to add metadata and, optionally, an LLM-written context line
(Chapters 50–51).

**CDC = change-data-capture.** The source system emits an event every time a record is created,
changed or deleted, so we hear about changes instead of hunting for them. A **webhook** is the
simplest form: the source system calls a web address we gave it whenever something happens.

```
┌─────────────────────────── INGESTION (async, batch + stream) ────────────────────────────┐
│                                                                                          │
│  sources ─► change feed ─► fetch raw ─► extract ─► chunk ─► enrich ─► embed              │
│  (CMS, S3,  (CDC, webhooks,    │           │          │        │         │               │
│   drives,    crawls)           ▼           ▼          ▼        ▼         ▼               │
│   tickets)                ┌─────────── SOURCE OF TRUTH (object store + DB) ────────────┐ │
│                           │ raw docs │ extracted text │ chunks+metadata │ fp32 vectors │ │
│                           │   versioned by content hash and pipeline version           │ │
│                           └──────────────────────────────┬─────────────────────────────┘ │
└──────────────────────────────────────────────────────────┼───────────────────────────────┘
                                                           │  index build / incremental load
┌──────────────────── SERVING (sync, latency-bound) ───────▼───────────────────────────────┐
│                                                                                          │
│  query ─► understand ─► ┌─ dense ANN (sharded) ─┐                                        │
│           (Ch. 52)      ├─ sparse / BM25 ───────┼─► fuse ─► rerank ─► assemble ─► LLM    │
│                         └─ metadata filters ────┘   (Ch. 40)  (Ch. 41)  (Ch. 53)         │
│                                                                                          │
│  caches: query embeddings · results · rerank scores · LLM prefix                         │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

**Reference design = two planes with one boundary.** The top plane, **ingestion**, runs in the
background and writes everything it produces into a durable **source of truth**. The bottom
plane, **serving**, answers queries within a latency budget, from indexes built out of that
source of truth.

In simple words, one side prepares the Library and the other side answers questions from it,
and neither side waits for the other.

Think of it like a restaurant.

- Ingestion is the prep kitchen. It chops and portions all day, at its own pace.
- The source of truth is the walk-in fridge where the prepared food is stored.
- Serving is the line cooks. They plate orders in minutes from what is already prepared.
- A slow delivery to the prep kitchen never makes a diner wait.

A few labels in the diagram. A CMS (content management system) is where help articles are
written. An **object store** is cheap storage for large files, such as Amazon S3. **Sharded**
means the index is split across several machines (Chapter 58). The LLM prefix cache is prompt
caching (Chapter 53).

Let's walk through why each piece exists.

---

## The four principles

### Principle 1: the source of truth is not the index

The index is a **derived, disposable cache**. It is optimised for query speed, compressed
lossily (Chapters 24–27), and structured in ways that make changes expensive (Chapter 31). It is
the wrong place to keep anything we cannot regenerate.

We store these durably, outside the index:

| Artefact | Why keep it |
|---|---|
| Raw documents | Re-extract when the parser improves |
| Extracted text | Re-chunk without re-parsing (parsing is often the slowest step) |
| Chunks + metadata | Rebuild sparse indexes, change enrichment |
| **Full-precision vectors** | Rebuild ANN indexes, change quantization, rescore, **without re-embedding** |

The last row is worth pausing on. Embedding is usually the most expensive stage in GPU money,
unless we enrich chunks with an LLM. Extraction and OCR are usually the slowest in wall-clock
time. Keeping float32 vectors in object storage costs very little: about 3 GB per million chunks,
so about 307 GB for Acme's 100 million. With them, changing the index type, the HNSW parameters,
the quantization scheme or the shard count is a rebuild measured in hours. Without them, it is a
re-embed measured in days and GPU budget.

Every chunk's metadata also includes its tenant, so any index rebuilt from this store can still
keep customers apart.

### Principle 2: idempotent, versioned stages

**Idempotent = running a step twice has exactly the same effect as running it once.**

We get that by making every stage a function of its inputs plus a version:

```
chunk_id  = hash(doc_id, doc_content_hash, chunker_version, chunk_index)
vector_id = hash(chunk_id, model_name, model_version, prefix_scheme)
```

A **hash** here is a short fingerprint computed from its inputs. The same inputs always give the
same fingerprint, and any change gives a different one. Means, a vector's id records exactly
which document content, chunker and model produced it.

Consequences:

- **Reprocessing skips unchanged work.** If a document's content hash did not change and the
  chunker version did not change, its chunks are already correct.
- **Partial failures are safe.** Rerun the job. Completed items are no-ops.
- **Upgrades are incremental.** Bump `chunker_version`, and only chunking and the stages after
  it rerun. Extraction is reused.
- **We always know what produced a vector.** This is essential for migration (Chapter 63) and
  for debugging.

### Principle 3: ingestion is asynchronous and elastic

Embedding throughput and query latency are different problems with different hardware. Couple
them, and a backfill (reprocessing a large batch of old content) will slow down live search.

- **Queue between stages.** Extraction, chunking and embedding then scale independently.
- **Batch the embedding stage.** GPU throughput rises sharply with batch size.
- **Separate backfill from live updates** with priority queues. A page one customer published
  five minutes ago should not wait behind a re-embed of all 100 million chunks.
- **Rate-limit writes toward the index**, so bulk loads do not starve serving.

### Principle 4: serving is stateless over shared indexes

Query services should hold no state beyond caches. Indexes are loaded read-mostly. **Replicas**
(copies of an index that share the query load, Chapter 58) scale with traffic. Any serving node
can answer any query.

In simple words, if a serving machine dies, we start another and lose nothing, because nothing
lived only on that machine.

---

## An example: fixing a chunking bug at 100 million chunks

Let's see the principles pay for themselves.

Acme discovers that its chunker has been splitting pricing tables in half. Header rows land in
one chunk and prices land in the next. Across 5,000 customers, table questions like *"Does the
Pro plan include single sign-on?"* start returning the wrong half of the table.

**Without the principles.** The vector database is the only copy of anything. To fix the bug,
the team must re-fetch every document from 5,000 customers' source systems. That is about 12.5
million documents, at 50–200 documents per second per extraction worker, with scanned pages far
slower. Then it must re-chunk, re-embed all 100 million chunks, and rebuild the index. Live
search degrades while all of this competes for the same machines. The honest answer to "how long
until it is fixed?" is likely days.

**With the principles.**

**Step 1:** Fix the chunker and bump `chunker_version`.

**Step 2:** Re-chunk from the **extracted text** already in the source of truth. No fetching and
no extraction, which were the slowest stages.

**Step 3:** New chunk ids mean new vector ids, so the embedding stage re-embeds. At 2,000–5,000
chunks per second, one GPU takes about 6–14 hours, and several GPUs divide that.

**Step 4:** Load the new vectors through the low-priority backfill queue. Live updates keep
flowing ahead of it.

**Step 5:** Build the new index alongside the old one. Live search keeps using the old index
until the new one passes its checks, then switches.

The fix takes hours of GPU time instead of days of everything, and no customer sees search slow
down. Same bug, very different week.

---

## Throughput arithmetic

For Acme's **100 million chunks** (about 12.5 million documents at 8 chunks each):

| Stage | Rough throughput | Time for full corpus |
|---|---|---|
| Fetch + extract (text PDFs) | ~50–200 docs/s per worker | hours, parallelisable |
| Extract with OCR (scans) | ~1–5 pages/s per worker | days unless heavily parallel |
| Chunking | very fast | minutes |
| Contextual enrichment (LLM) | bounded by API rate limits | the long pole if used |
| Embedding, small model, 1 GPU | ~2,000–5,000 chunks/s | ~6–14 hours |
| Embedding, 7B model, 1 GPU | ~100–300 chunks/s | ~4–12 days |
| HNSW build, 100M × 768-d | one machine per shard, in parallel (Chapter 58) | hours |

Two lessons jump out. **Model size dominates embedding time.** A 7B embedder is not a small
upgrade at this scale. It is roughly 4 to 12 days of single-GPU time per re-embed. And **OCR and LLM
enrichment are often the true long poles** (the slowest stages, which set the total time), not
embedding. Plan capacity around those.

---

## When to use which one

We must use **a simple script** when the corpus is small enough to rebuild from scratch in
minutes, such as a prototype or a single 40,000-page library. The reference design would be
pure overhead there.

We must use **the reference design** once a full rebuild takes hours, once many customers share
the system, or once documents change faster than we can rebuild.

Many strong systems start as a script and adopt the principles one at a time as they grow.
Keeping the full-precision vectors is the cheapest one to start with, because object storage is
the cheapest tier (Chapter 60).

---

### Under the hood

The idempotent embedding stage, which is where most of the GPU cost sits:

```python
def embed_stage(chunk_batch, model, model_version, store):
    todo = []
    for ch in chunk_batch:
        vid = vector_id(ch.chunk_id, model.name, model_version, model.prefix_scheme)
        if not store.exists(vid):                  # idempotent: skip done work
            todo.append((vid, ch))
    if not todo:
        return 0

    vecs = model.embed_documents([c.text_for_embedding for _, c in todo])
    assert np.allclose(np.linalg.norm(vecs, axis=1), 1, atol=1e-3)   # Ch. 5
    assert not np.isnan(vecs).any()

    store.put_many([
        dict(vector_id=vid, chunk_id=c.chunk_id, model=model.name,
             model_version=model_version, vector=v.astype(np.float32))
        for (vid, c), v in zip(todo, vecs)])
    return len(todo)
```

The stage first skips every vector that already exists. It embeds only what is left, checks the
results, and writes full-precision vectors to the source of truth.

The two assertions are not decoration. At this scale, a single `NaN` vector ("not a number", the
value a broken calculation produces) or a normalization regression will corrupt an index build
hours later. The error message will point nowhere near the cause. In production, write them as
`if ...: raise ValueError(...)`, because Python skips `assert` under `python -O`.

---

### What people get wrong

**Treating the vector database as the source of truth.** Then every parameter change, model
migration, or corruption means re-embedding everything.

**Synchronous ingestion.** Uploading a document blocks on embedding, and bulk loads degrade
search latency.

**No content hashing.** Every re-run reprocesses the entire corpus.

**Discarding float32 vectors after quantizing.** It saves little money and forfeits cheap
rebuilds.

**Planning for embedding time and ignoring extraction time.** For scanned corpora, OCR is
frequently the slowest stage by an order of magnitude.

**Coupling the chunker to the embedder.** Changing one should not require redeploying both.

---

### Ninja notes

**Tiered indexes are the natural shape at this scale.** Not all documents deserve equal
treatment:

- **Hot tier:** recent and frequently accessed. HNSW in RAM, full-precision rescoring, low
  latency.
- **Warm tier:** the bulk. HNSW with int8 (Chapter 26) or TurboQuant (Chapter 27) compression,
  or DiskANN (Chapter 32).
- **Cold tier:** archival. IVF-PQ or object-storage-backed indexes, where latency in the hundreds
  of milliseconds is acceptable.

Query all tiers in parallel and fuse, or query hot first and fall through when results are weak.
Access patterns in document corpora are heavily skewed toward recent content, so a hot tier
holding a small fraction of documents can serve most traffic.

**Shadow indexes make changes safe.** Build the new index, mirror a fraction of live queries to
it, compare results and latency against production, and switch only when the numbers hold. The
same machinery serves parameter changes, quantization changes and model migrations. Build it
once.

**Key vectors by content, not only by position.** With `vector_id` derived from `chunk_id`, a
chunker bump re-embeds every chunk, even chunks whose text came out identical. Deriving the id
from a hash of `text_for_embedding` instead lets unchanged chunks keep their vectors across
chunker versions. That can turn a full re-embed into a partial one.

---

### Key takeaways

- **Reference design = asynchronous ingestion into a durable source of truth, plus stateless
  serving over disposable indexes.**
- At scale, assume every derived artefact will be rebuilt repeatedly.
- Keep raw documents, extracted text, chunks and **full-precision vectors** in durable storage.
  The index is a disposable cache.
- Make every stage idempotent with content hashes and pipeline versions.
- Use CDC or webhooks to hear about changes, and replicas to scale with traffic.
- Model size dominates embedding time. OCR and LLM enrichment are often the true long poles.
- Assert normalization and the absence of NaNs at ingestion.
- Tier indexes by access pattern, and validate changes with shadow indexes.

### What's next

The index no longer fits on one machine. [Chapter 58](./58-sharding-and-routing.md) splits it,
and decides which pieces each query should ask.

We now have the shape of a system that survives a hundred million chunks: two planes, one source
of truth, and stages we can rerun without fear.
