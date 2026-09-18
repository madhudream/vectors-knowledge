---
title: "The Ninja's Field Manual"
chapter: 66
part: "Part IX — Ninja Tier"
slug: field-manual
readingTime: "Reference — keep open while building"
summary: "The book's key decision trees, formulas, defaults and checklists, in one place, each with a pointer back to the chapter that explains it."
tags: [reference, cheatsheet, decision-trees, defaults, checklists]
prev: 65-the-frontier
next: null
---

# The Ninja's Field Manual

## How to use this manual

This chapter is not meant to be read front to back. It is the page we keep open while building.
Every entry points to the chapter that explains *why*, so when a number looks surprising, follow
the pointer.

The manual has seven parts:

1. **Formulas**, for when we need to compute something.
2. **Memory arithmetic**, for sizing an index before we buy machines.
3. **Decision trees**, for choosing an index, a filter strategy or an architecture.
4. **Defaults**, for the first value of every knob.
5. **Scale and operations**, for sharding, freshness, cost and model migration.
6. **Checklists**, for before we ship.
7. **The eight laws**, for when we are unsure what matters.

Let's take an example. Say we are sizing Acme's knowledge-base product at 100 million chunks.
Part 2 gives the RAM for each compression choice. Part 5 gives the sharding strategy, the cost
per query and the migration plan. Part 4 gives the first values to try.

**Note:** Every default here is a starting point, not an answer. Measure it on your own golden
set (Chapter 54) before trusting it.

---

## 1. Formulas

| What | Formula | Ch. |
|---|---|---|
| Dot product | $a \cdot b = \sum_i a_i b_i$ | 4 |
| Cosine similarity | $\frac{a \cdot b}{\lVert a\rVert\lVert b\rVert}$ | 4 |
| Euclidean ↔ dot (unit vectors) | $\lVert a-b\rVert^2 = 2 - 2\,a\cdot b$ | 4 |
| Normalize | $\hat a = a / \lVert a\rVert$ | 5 |
| Random unit-vector cosine spread | std $\approx 1/\sqrt{d}$ | 6 |
| IDF | $\log\frac{N}{n_t}$ (natural log) | 8 |
| BM25 term | $\text{IDF}\cdot\frac{f(k_1+1)}{f + k_1(1-b+b\frac{\lvert D\rvert}{\text{avgdl}})}$ | 8 |
| InfoNCE | $-\log\frac{e^{s(q,d^+)/\tau}}{\sum_j e^{s(q,d_j)/\tau}}$ | 12 |
| Recall@k (system) | relevant in top-k / all relevant | 19 |
| Recall@k (index) | overlap with brute-force top-k / k | 19 |
| MRR | mean of $1/\text{rank}_{\text{first hit}}$ | 19 |
| nDCG@k | $\frac{\sum (2^{rel_i}-1)/\log_2(i+1)}{\text{IDCG}}$ | 19 |
| Standard error of a score | $\sqrt{p(1-p)/n}$. At $n=20$, $p=0.5$: ±11 points (95% interval ≈ ±22) | 19, 54 |
| SimHash collision | $1 - \theta/\pi$ | 21 |
| LSH amplification | $1-(1-p^k)^L$ | 21 |
| PQ bytes per vector | $m \times \text{bits}/8$. $m=96$, 8-bit: 96 B | 24 |
| Binary bytes per vector | $d/8$. 768-d: 96 B | 26 |
| HNSW level | $\lfloor -\ln U \cdot m_L \rfloor$, $m_L = 1/\ln M$ | 29 |
| MaxSim / Chamfer | $\sum_{i}\max_j q_i\cdot d_j$ | 35, 36 |
| MUVERA FDE dimension | $2^{k_{sim}} \times d_{proj} \times R_{reps}$ | 38 |
| RRF | $\sum_r \frac{w_r}{60 + \text{rank}_r}$ | 40 |
| MMR | $\lambda\,\text{rel} - (1-\lambda)\max_{s\in S}\text{sim}$ | 41 |
| Recency boost | $\text{score} \times 0.5^{\,\text{age}/\text{half-life}}$ | 51 |
| Fan-out: no shard hits its p99 | $0.99^N$. At $N=40$: 0.67, so a third of queries hit a slow shard | 58 |
| Cost per query (LLM) | $(\text{input tokens} \times P_{in} + \text{output tokens} \times P_{out}) / 10^6$ | 60 |
| Memory score | $\alpha\,\text{sim} + \beta\,\text{recency} + \gamma\,\text{importance}$ | 64 |

---

## 2. Memory arithmetic

**Bytes per vector** $= d \times$ bytes per dimension, plus index overhead.

| 768-d vector as | Bytes | 1M vectors | 100M vectors |
|---|---|---|---|
| float32 | 3,072 | 3.1 GB | 307 GB |
| float16 | 1,536 | 1.5 GB | 154 GB |
| int8 | 768 | 0.77 GB | 77 GB |
| TurboQuant 2-bit (+ 4 B scale) | ~196 | 0.20 GB | 19.6 GB |
| PQ m=96 / binary | 96 | 0.10 GB | 9.6 GB |

These are raw vectors. The rules below add the index and the copies.

- **HNSW graph overhead:** layer 0 stores $2M \times 4$ bytes of neighbour ids per vector. At 768-d
  with $M=16$ that is only about 5–10% on top of the vectors (≈1.05–1.1×). The "1.5–2×" figure
  applies only to low-dimensional vectors with $M=32$ (Ch. 29).
- **int8 HNSW:** about 90 GB per 100M vectors (Ch. 31, and Ch. 60 itemises ~93 GB), about 900 GB per
  1B (Ch. 32). One 256 GB machine holds the 100M case (Ch. 58).
- **100M vectors, heavily compressed:** binary + rescore ≈ 25 GB RAM. DiskANN ≈ 5 GB RAM + SSD
  (Ch. 60).
- **Multiply by replicas** (Ch. 58). Everything above is per replica.
- **ColBERT passage:** ~200 tokens × 128-d. That is ~30× the bytes and ~200× the vector count of
  one 768-d vector, before compression (Ch. 35).
- **ColBERTv2 compressed:** 2-bit residuals (32 B) + centroid ID (4 B) = 36 B per token, so
  ~7,200 B per 200-token passage (Ch. 37).
- **ColPali page:** 1,030 vectors (1,024 patches + prompt tokens) × 128-d × 4 B ≈ **527 KB raw**
  (Ch. 46, 47).
- **ColQwen2 page:** up to ~768 visual tokens by default. ~750 × 128 × 2 B ≈ **190 KB float16**,
  ~27 KB at 36 B per vector (Ch. 47, 61).
- **MUVERA FDE:** ~4,000-d in practice (Ch. 38, 61).
- **Keep float32 truth in object storage:** 3 GB per million vectors (Ch. 57).

---

## 3. Decision trees

### Which index? (Ch. 17, 20, 25, 32, 58, 60)

```
How many vectors?
├─ < 100k ─────────────────────────────► Flat (brute force). Done.
├─ 100k – 1M ──────────────────────────► Flat, or HNSW defaults
├─ 1M – ~100M
│   ├─ RAM available ──────────────────► HNSW (M=16, efC=200)
│   └─ RAM tight ──────────────────────► HNSW + int8 or binary (rescore), or IVF-PQ
│                                         (TurboQuant: benchmark first, Ch. 27)
├─ ~100M – 1B
│   ├─ latency ≤ 10 ms, budget OK ─────► sharded HNSW + compression
│   ├─ memory is the constraint ───────► DiskANN, or IVF-PQ + rescore
│   └─ archival / cold ────────────────► IVF-PQ or object-storage index
└─ > 1B ───────────────────────────────► DiskANN, or sharded IVF-PQ (Ch. 32, 58)

Batch/offline?   → Flat on GPU, batched (Ch. 17).
Heavy filtering? → read Ch. 33 first.
Tenant-scoped?   → namespace or shard per tenant (Ch. 58, 62).
Multi-vector?    → MUVERA FDE + any index above, rerank with MaxSim (Ch. 38),
                   or PLAID (Ch. 37).
```

### Filtering strategy (Ch. 33)

```
Filter present in every query, moderate cardinality? ──► PARTITION (index per value)
Otherwise check the allowed set's size first, then the pass rate:
├─ allowed set < ~50k vectors ────────► pre-filter + brute force (exact)
├─ pass rate > 30% ───────────────────► post-filter, fetch 2k / pass rate
├─ pass rate 10–30% ──────────────────► in-filter, or post-filter with a larger fetch
└─ pass rate < 10%, allowed set > 50k ► in-filter traversal (ACORN-style if available)
```

### Which retrieval architecture? (Ch. 7, 34, 40, 41, 44)

```
Documents mostly visual (tables, charts, scans, slides)?
├─ yes ─► ColQwen/ColPali page retrieval + OCR→BM25 for identifiers   (Ch. 61)
└─ no
    ├─ identifiers / codes / names common? ─► hybrid: BM25 (or SPLADE) + dense, RRF
    ├─ long heterogeneous docs, multi-constraint queries, precision critical?
    │                                         ─► add multi-vector (ColBERT) or rerank deeper
    └─ otherwise ─────────────────────────────► hybrid + cross-encoder rerank (default)
```

### Single-shot RAG or agent? (Ch. 56, 61)

```
Simple lookup ─────────────────────► single-shot RAG, maybe skip rerank
Multi-hop / comparative / vague ───► agentic retrieval (cap steps)
Heterogeneous sources (docs+SQL+tickets) ─► agent with one tool per source
Strict latency SLO ─────────────────► single-shot, no LLM rewrite on the hot path,
                                      route hard queries to a slow path (Ch. 61)
```

### Why is this answer wrong? (Ch. 55)

```
Can the corpus answer the question at all?   (check this first)
└─ NO  → failure 9: the right answer is a refusal. Answered anyway?
          → calibrated relevance floor + relevance check

Answerable? Then: was the needed information in the assembled context?
├─ NO  → retrieval: identifiers? dilution? follow-up? stale? hub pages? lost table? filter?
│         incomplete multi-part (evidence found for only one part)?
│         check: BM25 finds it? → hybrid. Small chunk finds it? → chunking.
│                Found pre-rerank but cut? → rerank depth / context budget.
│                Multi-part? → decompose, check context recall per sub-question.
└─ YES → generation: ignored context? altered numbers? injected instructions?
```

---

## 4. Defaults

Starting points only. Measure before trusting.

| Parameter | Default | Ch. |
|---|---|---|
| Normalize vectors | Yes, unless magnitude encodes something, such as popularity or confidence | 5, 48 |
| Metric on normalized vectors | Dot product | 4 |
| Pooling | Whatever the model was trained with | 11 |
| Contrastive temperature $\tau$ | 0.01–0.07 | 12 |
| Query/document prefixes | Exactly as the model card specifies | 14 |
| HNSW `M` / `efConstruction` / `efSearch` | 16 / 200 / 64–128 | 29, 30 |
| IVF `nlist` / `nprobe` | $\sqrt{n}$–$4\sqrt{n}$ / 8–64, start at 16 (sweep) | 23, 25 |
| PQ subvector size | 4–16 dims each | 24 |
| Rescore depth | 10–20× final k for PQ (Ch. 25). ~100× final k for binary (Ch. 26) | 25, 26 |
| Tombstone ratio | Alert at 15%, rebuild at ~20% | 31 |
| BM25 $k_1$ / $b$ | 1.2–2.0 / 0.75 (lower $b$ for uniform chunks) | 8 |
| RRF $k$ | 60 | 40 |
| Candidates per retriever before fusion | 100 | 40 |
| Rerank depth | 100 | 41 |
| MMR $\lambda$ | 0.7 | 41 |
| Chunk size (docs/articles) | 400–600 tokens, 10–15% overlap | 50 |
| Recency half-life | ~180 days for operational content | 51 |
| Context budget | Sweep it on the golden set (e.g. 4k, 8k, 16k, 32k). Ch. 53's example budgets 12k | 53, 60 |
| Golden set size | 50 minimum, 200 comfortable | 19, 54 |
| Relevance floor | Sweep it on the golden set, incl. `answerable: false` questions. Cap wrong refusals at ~2%. Re-run on every model change | 55 |
| Page render DPI | Store at 150 (200 for fine print). Original ColPali discards detail above ~60 DPI. For ColQwen2, sweep DPI together with the token cap (an A4 page hits the default cap at ~80 DPI) | 47, 61 |
| Over-fetch across shards before rescoring | ~2–5× final k for int8, 10–20× for PQ, ~100× for binary | 58 |
| Canary tolerance (max abs diff) | 1e-4 | 63 |

---

## 5. Scale and operations

This part collects the tables that keep a large system healthy after launch. In simple words:
how to split it, how fresh to keep it, what it costs, and how to change its model safely.

### Sharding strategy (Ch. 58)

| Strategy | Where a chunk goes | Shards per query | Load balance | Main risk | Best for |
|---|---|---|---|---|---|
| **Random (hash)** | `hash(doc_id)` → virtual bucket → shard | All N | Even | Tail latency from fan-out | Global search, simplest operations |
| **Semantic (cluster)** | Nearest centroid | A few nearest | Uneven, drifts | Imbalance, boundary misses, rebalancing | Global search over very large corpora, compute-sensitive |
| **Tenant** | Its owner's shard | Exactly one | As uneven as tenants | Large tenants need their own shards | Queries always scoped to one customer |
| **Time-partitioned** (Ch. 59) | Its date's partition | Partitions in the time range | Recent partition is busiest | Queries that ignore time ask everything | Time-scoped queries dominate |

| Query pattern | Strategy |
|---|---|
| Always scoped to one tenant/workspace | **Tenant sharding** |
| Global search, even load, simplest ops | **Random sharding** + hedged requests |
| Global search, very large corpus, compute-sensitive | **Semantic sharding** with multi-probe |
| Time-scoped queries dominate | **Time-partitioned** shards (Chapter 59) |
| Mixed | Tenant or time at the top level, random within |

- Composite schemes are normal: partition by tenant, then shard large tenants randomly.
- **Shards scale with data. Replicas scale with traffic.** Size them independently.
- **Tail latency:** 40 shards, each with a p99 of 20 ms, means $1 - 0.99^{40} \approx 33\%$ of
  queries hit at least one slow shard. Use hedged requests, deadlines with partial results, or
  fewer, larger shards.
- Keep shard configurations identical, over-fetch per shard, and rescore after merging.
- **Compress before you shard** (Ch. 31). One machine with 256 GB of RAM holds 100 million 768-d
  vectors in int8 HNSW (about 93 GB, Ch. 60), or more with TurboQuant (Ch. 27).

### Staleness budgets (Ch. 59)

| Content | Acceptable staleness | Approach |
|---|---|---|
| Support tickets, chat | Seconds–minutes | CDC → delta index |
| Wiki, documentation, pricing pages | Minutes–hours | Webhooks → delta, nightly compaction |
| Policies, contracts | Hours | Hourly incremental crawl |
| Archives, historical reports | Days–weeks | Weekly batch |
| **Deletions with legal or privacy implications** | **Minutes, guaranteed** | **Tombstone synchronously** |

The last row is not like the others. A legal or privacy deletion must disappear from caches,
deltas, replicas, backups and the source-of-truth vector store, not merely be hidden. ACL changes
take the same synchronous path (Ch. 62).

### Maintenance calendar (Ch. 59)

| Frequency | Task | Why |
|---|---|---|
| Continuous | Monitor delta size and tombstone ratio | Early warning (Ch. 31) |
| Hourly–daily | Compact deltas into main index | Bound delta size and query cost |
| Nightly | Recall check on sampled queries; eval-set regression run | Catch silent decay (Ch. 54) |
| Weekly | Reconcile against source to catch unreported deletions | Missed deletes accumulate |
| Monthly | Retrain IVF centroids / PQ codebooks if used; check shard balance | Drift (Ch. 23, 25) |
| Quarterly | Review embedding model options; re-run model comparison on eval set | Chapter 63 |

**Index decay is silent.** Nothing errors. The nightly recall check is what turns an invisible
trend into a graph.

### Cost per query and the levers (Ch. 60)

```
PLACEHOLDER UNIT PRICES (replace with yours)
  RAM ≫ NVMe SSD ≫ object storage     (RAM tens of times SSD, SSD a few times object storage)

LLM cost per query ≈ (input_tokens × P_in + output_tokens × P_out) / 1,000,000

context 12,000 tokens + prompt 1,000 + output 400:
    = (13,000 × P_in + 400 × P_out) / 1,000,000    per query

monthly LLM bill = LLM cost per query × queries per month × LLM calls per query
                                                            (agents: perhaps 4–8)

embedding compute, 100M chunks:  small model ~7 GPU-hours
                                 base model  ~19 GPU-hours
                                 7B model    ~140 GPU-hours   (repeats on every migration)
```

In simple words, the LLM bill usually exceeds the index bill, and context size multiplies it
directly. Optimise **cost per successful answer**, not cost per query.

The levers, largest typical impact first for most RAG products (index levers rise to the top for
large, low-traffic corpora):

| Lever | Saves | Quality cost | Chapter |
|---|---|---|---|
| **Right-size context budget** | LLM input tokens, directly | None if measured | 53 |
| **Prompt caching** (stable prefix first) | Large share of repeated input tokens | None | 53 |
| **Route by query complexity** (skip rerank/agent for simple queries) | GPU and LLM calls | Often positive | 13, 41, 56 |
| **Quantize the index** (int8 → TurboQuant/binary + rescore) | 3.5–13× HNSW RAM (4–32× on vector bytes alone) | ~1–3% with rescoring | 26, 27 |
| **Smaller embedding model** (after eval) | Embedding GPU, query latency, dims | Measure | 16 |
| **MRL truncation** | 2–4× RAM | ~1–2% | 15 |
| **Tier cold data** to SSD / object storage | RAM for the long tail | Latency on cold queries | 32, 57 |
| **Cache query embeddings and results** | Repeated work | Staleness risk | 59 |
| **Token pooling for multi-vector** | 2–3× multi-vector storage | Small | 37, 47 |
| **Fewer replicas via better p99** | RAM × replica count | None | 58 |

### Model migration, six phases (Ch. 63)

| Phase | Name | What happens |
|---|---|---|
| 0 | **Prepare** | Version everything: model name, version, prefix scheme, pooling, dims. Put the model version in every vector id |
| 1 | **Build** | Turn on dual-write. Backfill: re-embed the corpus from source-of-truth text into a NEW vector store. Build NEW indexes. Never overwrite old vectors |
| 2 | **Evaluate** | Golden set on OLD vs NEW, by segment, with confidence intervals. Recalibrate relevance floors and reranker thresholds |
| 3 | **Shadow** | Mirror a share of live queries to NEW. Compare top-10 overlap, rank correlation, latency, errors. Read strong disagreements by hand |
| 4 | **Shift** | Route 1% → 10% → 50% → 100% of traffic. Watch online signals. Roll back instantly if they worsen |
| 5 | **Retire** | Keep OLD warm for a rollback window, then decommission |

- Vectors from two models are never comparable. A migration is always a full re-embed and rebuild.
- Switch encoder, index, prefixes, pooling, normalization, thresholds and cache keys together.
- Run canary vectors in CI to catch dependency drift between migrations.

---

## 6. Checklists

### Before choosing an embedding model (Ch. 16, 54, 62)
- [ ] 50+ real queries with gold documents, segmented by type, incl. unanswerable
- [ ] BM25 baseline measured
- [ ] 3–4 candidates, each with correct prefixes and pooling
- [ ] Hybrid (best model + BM25, RRF) row measured
- [ ] Results reported per segment with confidence intervals
- [ ] Index-time throughput and re-embed cost estimated
- [ ] Licence and data-residency constraints checked

### Ingestion pipeline (Ch. 5, 21, 50, 51, 57, 61)
- [ ] Content hashing and pipeline versions on every artefact
- [ ] Raw docs, extracted text, chunks, float32 vectors in durable storage
- [ ] Structure-aware chunking, empty chunks filtered
- [ ] Title and section heading prepended before embedding
- [ ] Metadata: ids, dates, version, status, language, ACL, neighbours
- [ ] Assert unit norm and no NaNs
- [ ] Near-duplicate removal (MinHash)
- [ ] Page images kept for visual documents

### Retrieval quality (Ch. 40, 41, 50, 52, 53, 55)
- [ ] Hybrid dense + sparse with RRF
- [ ] Cross-encoder rerank over ~100
- [ ] Conversational rewrite for chat products (off the hot path if there is a strict SLO)
- [ ] Metadata filters extracted from queries
- [ ] Retrieve small, pass large
- [ ] Dedupe + per-document cap + MMR
- [ ] Relevance floor calibrated (Ch. 55): sweep it over the golden set's `answerable: false`
  questions, keep wrong refusals of answerable ones under ~2%, and recalibrate on every model change.
  Below the floor, answer with an honest "I don't know"
- [ ] Sources labelled with id/title/date/url, and answers cite them

### Production operations (Ch. 31, 58, 59, 62, 63)
- [ ] Nightly index recall vs brute force
- [ ] Nightly golden-set regression run
- [ ] Tombstone ratio, delta size, p50/p99, shard coverage dashboards
- [ ] Delta index + compaction schedule
- [ ] Deletion reconciliation against sources
- [ ] ACL enforced before retrieval, server-side, with permission-aware cache keys
- [ ] Tenants isolated by namespace or shard, never by a caller-supplied filter
- [ ] Embeddings classified at source-data sensitivity
- [ ] Retrieved content treated as untrusted: delimited, sanitised, least-privilege tools
- [ ] Canary vectors in CI
- [ ] Blue-green rebuilds with verification and rollback
- [ ] Log rewritten query, candidates, assembled context, answer

---

## 7. The eight laws

1. **Every representation compresses. Know what yours discards.**
2. **Coarse and broad first. Exact and narrow last.**
3. **Exact and semantic matching fail in opposite directions. Run both.**
4. **First-stage recall is the ceiling on everything downstream.**
5. **A model encodes the similarity it was trained on, not the one you meant.**
6. **Measure on your own data, by segment, with confidence intervals.**
7. **The index is a cache. The source of truth lives elsewhere.**
8. **The shiny thing is often not the right thing.**

---

We started this book asking what a vector is. Now we know how to find one needle among a billion
vectors, or inside a picture of a page, in milliseconds, for a price we can defend. We also know
how to tell when it has gone wrong.

Go build something.
