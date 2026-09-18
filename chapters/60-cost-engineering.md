---
title: "Cost Engineering"
chapter: 60
part: "Part VIII — Scale and Production"
slug: cost-engineering
readingTime: "11 min"
summary: "The arithmetic of vector search at scale — bytes, GPU-hours, and tokens — worked through for a hundred million chunks, with the levers ranked by how much each one saves."
tags: [cost, memory, capacity-planning, economics, optimization]
prev: 59-freshness
next: 61-image-heavy-rag
---

# Cost Engineering

**The one-paragraph version.** A RAG system has four cost centres: **memory** for the index,
**compute** for embedding, **compute** for query-time models, and **tokens** for the LLM. At
scale, the index's RAM bill and the LLM's token bill usually dominate. Both are governed by a
handful of multipliers we can compute on a napkin. This chapter gives us the napkin, works it
through for a hundred million chunks, then ranks the levers by what they save.

In this chapter, we will learn how to estimate what a vector search and RAG system costs, in
units that do not go out of date. We will also see where Acme's money actually goes, why the
obvious fix is often the wrong one, and which levers save the most.

We will cover the following:

- What is cost engineering
- A note on prices
- Memory for the index
- Embedding compute
- Query-time compute
- The LLM bill
- An example: cutting Acme's monthly bill
- The levers, ranked

---

## What is cost engineering

**Cost engineering = working out what each part of the system costs, in units, and changing
the parts that cost the most for the least quality.**

Two units carry most of this chapter.

**GPU-hour = one GPU running for one hour.** Cloud providers bill GPUs this way, and we measure
embedding jobs this way.

**Cost per query = what it costs to answer one question:** the LLM tokens it uses, plus its share
of the index, embedding and GPU bills.

The four cost centres, one line each:

- **Index memory.** The RAM (and SSD) that holds the Card Catalog, every hour of every month.
- **Embedding compute.** GPU-hours to turn every chunk into a vector, paid again on every
  re-embed.
- **Query-time compute.** Query embedding, search, reranking, for every question.
- **LLM tokens.** Every token the Scholar reads and writes, for every question.

Our scenario is Acme's knowledge-base product from Chapter 57: 5,000 customer companies, **100
million chunks, 768 dimensions**.

---

## A note on prices

Cloud prices change constantly and vary by provider, region and commitment. So this chapter
works in **units** (gigabytes, GPU-hours, tokens), and uses clearly labelled placeholder prices
only to show proportions. **Substitute your own prices.** The multipliers are what matter, and
they do not change.

```
PLACEHOLDER UNIT PRICES (replace with yours)
  RAM (in an instance):   P_ram   per GB-month
  NVMe SSD:               P_ssd   per GB-month     typically ≪ P_ram
  Object storage:         P_obj   per GB-month     typically ≪ P_ssd
  GPU:                    P_gpu   per GPU-hour
  LLM input tokens:       P_in    per million tokens
  LLM output tokens:      P_out   per million tokens
```

A GB-month is one gigabyte kept for one month. The `≪` sign means "much smaller than".

The single most important relationship in this chapter:

**RAM costs tens of times more per GB than NVMe SSD.**

**NVMe SSD costs a few times more per GB than object storage.**

In simple words, bytes get much cheaper at each step down that ladder. End to end, RAM costs a
hundred or more times what object storage does. Most of vector-search cost engineering is moving
bytes down the ladder without losing quality.

---

## Memory for the index

Here is what 100 million 768-d vectors cost in memory, for each index configuration in this
book. This table is the reference that other chapters point to.

| Configuration | Bytes/vector (incl. graph) | Total RAM | Relative |
|---|---|---|---|
| HNSW, float32 | ~3,230 | ~323 GB | **1.00×** |
| HNSW, float16 | ~1,700 | ~170 GB | 0.52× |
| HNSW, int8 | ~930 | ~93 GB | 0.29× |
| HNSW, TurboQuant 2-bit (Ch. 27) | ~356 | ~36 GB | 0.11× |
| MRL-256 + int8 | ~416 | ~42 GB | 0.13× |
| Binary + rescore from SSD | ~250 RAM + 307 GB SSD | ~25 GB RAM | 0.08× RAM |
| DiskANN | ~48 RAM + ~410 GB SSD | ~5 GB RAM | 0.02× RAM |
| IVF-PQ (m=96) | ~105 | ~10.5 GB | 0.03× |

How to read one row. HNSW float32 stores 768 × 4 = 3,072 bytes of vector plus about 160 bytes of
graph links and ids per vector. That is about 3,230 bytes. Times 100 million, that is about 323
GB. Binary keeps 96 bytes of code in RAM plus the same ~160 bytes of graph, about 256 bytes, so
about 25 GB. MRL-256 is Matryoshka truncation to 256 dimensions (Chapter 15).

The table's message is blunt: **the default configuration costs roughly 10–40× more RAM than a
well-compressed one**, for a recall difference that, with rescoring, is usually a point or two.
Quantization is the largest single lever on the index's RAM bill.

Add the source of truth (Chapter 57): 100M × 3,072 bytes ≈ **307 GB in object storage**, the
cheapest tier. Keep it.

**Replication multiplies memory.** Everything above is *per replica*. Three replicas, for
availability and throughput (Chapter 58), triple the RAM line. For float32 HNSW that is about 970
GB. **This is why compression matters so much at scale.** It multiplies through replication and
through sharding overhead.

**At a billion vectors, multiply every line by ten.** Float32 HNSW then needs about 3.2 TB of RAM
per replica, and about 9.7 TB across three replicas. At the toy RAM price used later in this
chapter ($5 per GB-month), that is about $48,000 a month, or roughly $580,000 a year, just to keep
uncompressed numbers in memory. That is in the range of what a small engineering team costs. The
same billion vectors need about 900 GB per replica in int8, and about 50 GB of RAM plus SSD with
DiskANN (Chapter 32).

---

## Embedding compute

```
chunks × (seconds per chunk) = GPU-seconds

100M chunks, small model (~4,000 chunks/s per GPU):   ~7 GPU-hours
100M chunks, base model  (~1,500 chunks/s):           ~19 GPU-hours
100M chunks, 7B model    (~200 chunks/s):             ~140 GPU-hours
```

Means, divide the chunk count by the model's speed to get seconds, then divide by 3,600 to get
GPU-hours. 100 million ÷ 4,000 = 25,000 seconds, about 7 hours.

That is a **one-time** cost per model version, until we migrate (Chapter 63), change chunking, or
add contextual enrichment. Multiply by how often you expect to re-embed per year. A 7B embedder
is not a small quality upgrade. It is ~20× a small model's re-embedding bill (~7.5× a base
model's), every time.

**Contextual retrieval** (Chapter 50) adds an LLM call per chunk. At 100M chunks this is
frequently the largest *index-time* cost in the whole system, often larger than embedding. Prompt
caching over the parent document reduces it substantially, because every chunk from one document
shares the same prefix.

---

## Query-time compute

Per query, roughly:

| Component | Typical cost |
|---|---|
| Query embedding (small model) | < 5 ms GPU, or ~20 ms CPU |
| ANN search per shard | 1–10 ms CPU |
| BM25 | 1–10 ms CPU |
| Cross-encoder rerank, 100 candidates | 20–100 ms GPU |
| ColPali query encoding (Ch. 47) | 10–50 ms GPU |
| LLM generation | 500–5,000 ms, **plus tokens** |

The reranker and any vision-language query encoder are the GPU consumers at query time.
Retrieval itself is cheap CPU work.

---

## The LLM bill

It is usually the biggest.

```
LLM cost per query ≈ (input_tokens × P_in + output_tokens × P_out) / 1,000,000

context 12,000 tokens + prompt 1,000 + output 400:
    = (13,000 × P_in + 400 × P_out) / 1,000,000    per query
```

In simple words, count the tokens the Scholar reads and writes, and multiply each by its price.
Prices are quoted per million tokens, hence the division.

Now multiply by queries per month, and by the number of LLM calls per query. In an agentic system
(Chapter 56) that might be four to eight.

**Context size is a direct multiplier on this bill.** Halving the context budget nearly halves the
input-token cost of every query, forever. This is why Chapter 53 insists on choosing a budget by
measurement. If quality plateaus at 6,000 tokens and we are sending 24,000, we are paying 4× for
nothing.

---

## An example: cutting Acme's monthly bill

Let's put numbers on Acme. The prices below are **toy prices, for arithmetic only**. They are not
quotes from any provider.

- RAM: $5 per GB-month. Object storage: $0.02 per GB-month.
- LLM input: $0.50 per million tokens. LLM output: $2 per million tokens.

Acme answers **5 million questions a month**. Today it runs float32 HNSW with 3 replicas. Every
question sends 24,000 tokens of context plus a 1,000-token prompt, and gets a 400-token answer.

| Line | Arithmetic | Per month |
|---|---|---|
| Index RAM | 323.2 GB × 3 replicas × $5 | $4,848 |
| Source of truth | 307 GB × $0.02 | $6 |
| LLM input | 5M × 25,000 tokens = 125B tokens × $0.50/M | $62,500 |
| LLM output | 5M × 400 tokens = 2B tokens × $2/M | $4,000 |
| **Total** | | **$71,354** |

Finance asks the team to cut the bill.

**The obvious fix.** The team has read Part IV, so it goes after the index and moves HNSW from
float32 to int8. RAM drops from about 970 GB to about 278 GB, and the RAM line from $4,848 to
$1,392. The new total is $67,898. That is a saving of $3,456, **under 5%**. As an answer to
finance, it is wrong. The team optimised the wrong end of the bill.

**The measured fix.** The team runs the golden set (Chapter 54) at several context budgets and
finds that quality stops improving at 6,000 tokens. It cuts the context from 24,000 to 6,000
tokens. Input tokens fall from 125 billion to 35 billion a month, and the LLM input line from
$62,500 to $17,500. That is a saving of $45,000, **about 63%**, for an afternoon of measurement.
This answer is correct.

With both changes, the bill is $22,898, about 68% lower. Cost per query falls from about 1.4
cents to about 0.46 cents.

Put simply, the index was 7% of Acme's bill and the LLM was 93%. Look at the split before
choosing what to optimise.

**Note:** This is not always the split. At these toy prices, the float32 index costs as much as
the LLM calls of about 365,000 queries a month. A small-traffic system over a huge corpus can
have the index as its biggest line. Run the calculator with your own numbers.

---

## The levers, ranked

By typical impact for most RAG products, largest first. For a large corpus with little traffic,
the index levers move to the top:

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

The top of that table is striking: **the largest levers are also among the cheapest to
implement.** Choosing a context budget by measurement is an afternoon of work, and prompt caching
mostly means putting the stable part of the prompt first. In Acme's example, the context budget
alone cut the bill by more than half.

We must pull **the LLM levers first** (context budget, prompt caching, routing) when query volume
is moderate or high, which is most RAG products. We must pull **the index levers first**
(quantization, MRL, tiering) when the corpus is huge and traffic is light, or when the index no
longer fits the machines we have. Many strong systems do both, in that order of payoff.

---

### Under the hood

A capacity calculator worth keeping in your repository:

```python
def monthly_cost(n_vectors, dim, bytes_per_dim, graph_bytes, replicas,
                 queries_per_month, ctx_tokens, out_tokens, llm_calls_per_query,
                 P_ram, P_obj, P_in, P_out):
    index_gb  = n_vectors * (dim * bytes_per_dim + graph_bytes) / 1e9
    truth_gb  = n_vectors * dim * 4 / 1e9                         # float32 in object store
    memory    = index_gb * replicas * P_ram + truth_gb * P_obj
    tokens_in = queries_per_month * llm_calls_per_query * ctx_tokens
    tokens_out= queries_per_month * llm_calls_per_query * out_tokens
    llm       = tokens_in / 1e6 * P_in + tokens_out / 1e6 * P_out
    return {"index_gb": round(index_gb, 1), "memory": round(memory), "llm": round(llm),
            "llm_share": round(llm / (llm + memory), 2)}

P_ram, P_obj, P_in, P_out = 5.00, 0.02, 0.50, 2.00      # toy prices from the example

# Compare float32 vs int8 vs 2-bit for Acme (25,000 input tokens = context + prompt):
for label, bpd in [("fp32", 4), ("int8", 1), ("2-bit", 0.25)]:
    print(label, monthly_cost(100e6, 768, bpd, 160, 3, 5e6, 25_000, 400, 1,
                              P_ram, P_obj, P_in, P_out))
# fp32  {'index_gb': 323.2, 'memory': 4854, 'llm': 66500, 'llm_share': 0.93}
# int8  {'index_gb': 92.8,  'memory': 1398, 'llm': 66500, 'llm_share': 0.98}
# 2-bit {'index_gb': 35.2,  'memory': 534,  'llm': 66500, 'llm_share': 0.99}
```

In simple words, the function turns bytes into GB and tokens into money, and reports what share
of the bill is the LLM. Notice that changing the index format never touches the `llm` column.

Run it with your real prices. The `llm_share` output is usually the eye-opener. For most RAG
products at moderate query volume, **the LLM bill exceeds the index bill**. That means context
budgeting and routing deserve more engineering attention than index tuning.

---

### What people get wrong

**Optimising the index while sending 30,000-token contexts.** Wrong end of the bill.

**Forgetting replication in capacity plans.** RAM per replica, times replicas.

**Choosing a 7B embedder without pricing re-embeds.** The cost recurs on every migration.

**Discarding float32 vectors to save storage.** Object storage is the cheapest tier, and the
vectors let you rebuild without re-embedding. Saving a small storage cost forfeits a large
compute cost.

**Ignoring agentic multipliers.** An agent issuing six LLM calls per question multiplies the
token bill six-fold. Route.

**Not measuring quality alongside cost.** Every lever above has a quality cost of zero to a few
percent, *if* you verify it on your eval set. Unmeasured, a "cheap" configuration can quietly
cost more in failed answers than it saves.

---

### Ninja notes

**Cost per successful answer is the metric that matters.** Divide total monthly cost by the
number of queries that produced a correct, faithful answer (Chapter 54). A configuration that is
30% cheaper and 10% less accurate may be *more* expensive per successful answer, once you account
for users retrying, escalating to humans, or giving up. This single ratio aligns cost engineering
with quality engineering, which otherwise tend to fight.

**GPU utilisation is usually terrible.** Query-time GPUs serving rerankers and query encoders
often sit at low utilisation, because traffic is bursty and requests arrive one at a time.
**Dynamic batching** means waiting a few milliseconds to group concurrent requests into one GPU
batch. It can raise throughput several-fold on the same hardware, for a small latency cost. It is
frequently the cheapest way to halve a GPU bill.

---

### Key takeaways

- **Cost per query = the tokens, compute and memory one answered question consumes.** Work it
  out in units, then substitute your own prices.
- Four cost centres: index memory, embedding compute, query-time models, LLM tokens.
- RAM ≫ SSD ≫ object storage per GB: tens of times from RAM to SSD, a few times from SSD to object
  storage.
- For 100M vectors: float32 HNSW ~323 GB RAM, int8 ~93 GB, binary + rescore ~25 GB, DiskANN ~5
  GB plus SSD. Well-compressed configurations use 10–40× less RAM, for a small recall difference
  with rescoring.
- Replication multiplies memory. Agentic loops multiply tokens. Embedding model size multiplies
  every re-embed.
- The LLM bill usually exceeds the index bill. In Acme's example, trimming context saved 63% and
  quantizing saved under 5%.
- For most RAG products the largest levers are the LLM ones: context budgeting, prompt caching
  and routing. Index levers lead for huge corpora with light traffic.
- Optimise **cost per successful answer**, not cost per query.

### What's next

[Chapter 61](./61-image-heavy-rag.md) assembles Parts V, VI and VIII into one complete, costed
design: RAG over a million scanned, chart-filled pages.

We now know where the money in a RAG system goes, how to estimate it on a napkin, and which
levers to pull first.
