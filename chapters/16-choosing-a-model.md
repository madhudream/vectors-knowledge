---
title: "Choosing and Benchmarking an Embedding Model"
chapter: 16
part: "Part II — Where Embeddings Come From"
slug: choosing-a-model
readingTime: "13 min"
summary: "Leaderboards measure someone else's problem. Here is the decision framework, the seven criteria that actually matter, and the one-day evaluation that beats any ranking."
tags: [model-selection, mteb, benchmarking, evaluation, practical]
prev: 15-matryoshka
next: 17-brute-force
---

# Choosing and Benchmarking an Embedding Model

**The one-paragraph version.** A leaderboard tells us how models perform on someone else's
data. The gap between models on *our* data is often larger than the gap on MTEB, and
sometimes in the opposite order. So we use the leaderboard to shortlist three or four
candidates, using the criteria below. Then we spend one day building a 50-query evaluation
set from our own corpus. That day will save months.

In this chapter, we will learn how to choose an embedding model. We will see what MTEB is
and how to read it without being fooled, the seven criteria that actually matter, and the
one-day evaluation that beats any ranking. We will also watch Acme pick the leaderboard's top
model, get an error-code question wrong, and then fix the choice with its own data.

We will cover the following:

- What is MTEB
- Why the leaderboard is not enough
- The seven criteria
- How to read MTEB honestly
- How to run the one-day evaluation
- An example: choosing Acme's model, before and after
- Sensible starting shortlists
- Leaderboard vs our own evaluation
- When to use which one

---

## What is MTEB

Before we can read a leaderboard, we must know what a benchmark is.

**Benchmark = a fixed set of tasks with known right answers, used to score and compare
models.**

**MTEB = Massive Text Embedding Benchmark.** Let's break it down:

- **Massive:** many tasks, many datasets and many languages, with more added in newer versions.
- **Text Embedding:** it tests models that turn text into vectors.
- **Benchmark:** every model takes the same exam and gets a score.

MTEB groups its tasks into types. Retrieval means finding the passages that answer a query.
Other types include reranking, classification, clustering, summarisation and STS (semantic
textual similarity: scoring how alike two sentences are). Each model gets a score per type
and an overall average, and the public leaderboard ranks the models.

In simple words, MTEB is a big public exam for embedding models, and the leaderboard is the
class ranking.

---

## Why the leaderboard is not enough

Think of it like a wine critic's ranking:

- The critic is MTEB.
- The wines are the embedding models.
- The critic's own meals are MTEB's datasets: public collections such as web search
  questions, Wikipedia passages and scientific abstracts.
- Our dinner is our own corpus and our own users' questions.

A critic's ranking is real information. It tells us which wines are well made. But it ranks
how the wines taste *to that critic*, with their food. It cannot tell us how a wine tastes
with our dinner.

MTEB works the same way. It is genuinely valuable, because it tells us a model is broadly
competent rather than a toy. But it cannot tell us whether a model handles Acme's product
names and error codes. Nor can it tell us how the model copes with Acme's customers' phrasing,
Acme's page lengths, or the three query types that make up 70% of Acme's traffic.

**Use the leaderboard to build a shortlist. Use your own data to choose.**

---

## The seven criteria

We will call these *criteria*, so they do not get mixed up with the dimensions of a vector.
(One of them *is* the vector's dimensions.)

**1. Retrieval quality on *our* domain.** The only measurement that matters. Everything below
is a constraint on how we get there.

**2. Dimensions.** They drive memory, index size, latency and cost, in straight proportion.
Often 384-d is enough. 1536-d is four times the storage of 384-d, sometimes for
two points of nDCG (a ranking-quality score quoted out of 100, Chapter 19). If the model is MRL-trained
(Chapter 15), the number of dimensions becomes a dial rather than a commitment.

**3. Maximum sequence length.** This is how many tokens (Chapter 10) the model reads before
it stops. 512 tokens is roughly 350–400 English words. If our natural chunk is longer, we
need a long-context model or smaller chunks, and we must know which. Exceeding the limit
usually truncates **silently**. Suppose Acme embeds a 1,200-word pricing page with a
512-token model. Only the first ~400 words count, so if the SSO row sits near the bottom of
the pricing table, it is simply gone.

**4. Speed.** Speed matters twice. First, index throughput: how long it takes to embed 10
million documents, which can be days. Second, query latency: how many milliseconds pass
before the search even starts. A 7B model (one with 7 billion weights) may add 100 ms per
query.

**5. Multilingual capability.** If we have any non-English content, we must test it
explicitly. "Multilingual" covers everything from 5 languages to more than 100, at very
different quality levels per language.

**6. Licence and deployment.** Apache-2.0 or MIT (permissive open-source licences) means we
can self-host freely. Some open-weight models (models whose weights anyone can download) carry
non-commercial licences, so read the licence as well as the model card. API models, hosted by a vendor and called over the
network, mean no GPU operations to run. They also mean a per-token bill, a network hop, and a
vendor who can deprecate the model under us, which forces a full re-embed (Chapter 63).

**7. Stability and provenance.** Is this model maintained? Is the training data documented?
Was the benchmark data in the training set? A model that tops a leaderboard and disappears in
six months is a liability.

---

## How to read MTEB honestly

There are four cautions, in order of how often they bite.

**Caution 1: Look at the retrieval column, not the average.** MTEB averages task types such as
classification, clustering, reranking, STS, summarisation and retrieval. If we are building
search, only retrieval matters. Models are routinely tuned to lift the average.

**Caution 2: Benchmark contamination is real.** **Contamination** means a model was trained
on data that overlaps the test. Think of a student who saw the exam paper in advance. MTEB's
test sets are public, and some models train on data that overlaps them, directly or through
paraphrase. A model far above its peers on MTEB and unremarkable on our data may be a
contamination story.

**Caution 3: Check the model size column.** A 7B model scoring 1.5 points higher than a 335M
model is not obviously the better choice. It has about 20 times the weights, and we pay for
every one of them on a GPU.

**Caution 4: Small score differences are noise.** The gap between rank 3 and rank 11 is often
within run-to-run variance, the wobble a score shows from small changes in setup. Treat the
top 10–15 as a pool of roughly equivalent candidates, and pick on the other six criteria.

---

## How to run the one-day evaluation

This is the most valuable day of work in any retrieval project. It is not sophisticated, and
it does not need to be.

**Phase 1 (morning): build the set.**

**Step 1:** Collect 50 real queries. Real ones: from search logs, from support tickets, from
asking five colleagues what they would type. Not queries we invented, which are always cleaner
than reality.

**Step 2:** Cover the real mix. Include the boring lookups, the identifier queries such as
error codes, the paraphrases, the multi-constraint questions and the misspellings.

**Step 3:** For each query, find the pages that genuinely answer it. By hand. These are the
*gold* pages. A spreadsheet with `query, page_id, relevant (0/1)` is a perfectly good format.

Fifty queries with carefully chosen gold pages beat five hundred sloppy ones.

**Phase 2 (afternoon): run the matrix.**

**Step 4:** Embed all pages and queries with each shortlisted model, using each model's
correct prefixes (Chapter 14).

**Step 5:** For each query, take the top 10 pages and measure recall@10: the share of its gold
pages that made the top 10. Chapter 19 defines it fully.

**Step 6:** Add two rows that are not models at all.

**Step 7:** Report every row twice: once over all 50 queries, and once per query type.

In simple words, for every model we search with every query, and count how many of the right
pages show up in the top 10. That is the whole evaluation. The code is in *Under the hood*
below.

The two extra rows from Step 6 tell us where we stand:

- **BM25**, the keyword baseline from Chapter 8. If it wins, that is enormously useful news.
- **Best model + BM25, fused with RRF.** RRF is a simple rule for merging two ranked lists
  (Chapter 40). This row is usually the best number on the page.

**Phase 3 (evening): read the table.**

By evening we have a table that answers the question with evidence. We also have a permanent
regression test, a check we re-run after every change to catch anything that got worse. It
covers every future change: new chunking, new index, new reranker, model migration.
Everything downstream in this book gets easier because this exists.

---

## An example: choosing Acme's model, before and after

Let's first see what happens when Acme picks by leaderboard.

Acme's engineers open MTEB, sort by the average score, and take the top model. It has 7
billion weights, and we will call it Model A. They embed all 40,000 pages and ship.

A customer's login breaks. They type the error code from their screen into the search box:

> *"SSO-4012"*

Page 17,450 explains `SSO-4012`, "SAML assertion expired". It is usually caused by a wrong
clock on the customer's server. Page 17,452 explains `SSO-4021`, "SAML signature invalid",
which is usually a wrong certificate.

To a dense embedding, "SSO-4012" and "SSO-4021" are nearly the same handful of tokens. Their
pages sit almost on top of each other in the Map Room. Model A ranks page 17,452 first.

The Scholar (the LLM) reads it and thinks like this: "The user has an SSO error. This page
says the fix is a new certificate." It answers:

> *"Upload a new SAML certificate to your identity provider."*

The answer is wrong. The customer's certificate is fine. Their server clock is off, and they
lose an afternoon.

Now, let's see what the one-day evaluation shows. Acme collects 50 real queries in four
groups and runs the matrix. Here is recall@10 for each group. These are toy numbers, chosen
to show the shape a real result often takes:

| Row | Plain questions (20) | Paraphrases (10) | Error codes (10) | Multi-constraint (10) | All 50 |
|---|---|---|---|---|---|
| Model A: #1 on MTEB average, 7B | 0.88 | 0.90 | 0.40 | 0.70 | 0.75 |
| Model B: mid-size, 335M | 0.85 | 0.85 | 0.45 | 0.65 | 0.73 |
| Model C: small, 110M | 0.80 | 0.80 | 0.40 | 0.55 | 0.67 |
| BM25 | 0.70 | 0.20 | 0.95 | 0.50 | 0.61 |
| Model B + BM25 (RRF) | 0.90 | 0.85 | 0.95 | 0.75 | **0.87** |

Look at the "All 50" column first. Model A beats Model B by two points, which is well within
the noise for 50 queries. Taken alone, and looking only at the three models, that column would
still say "pick Model A".

Now look at the groups. All three embedding models fail on error codes. BM25, which matches
exact tokens, gets them almost all right, and does badly on paraphrases. The fused row takes
the best of both, and wins overall by a wide margin. And Model B is about 20 times smaller
than Model A, so it is far cheaper and faster to run.

Acme ships Model B + BM25. The customer searches "SSO-4012" again. BM25 puts page 17,450
first, because it is the only page that contains that exact token. RRF keeps it on top. The
Scholar reads it and answers:

> *"SSO-4012 means the SAML assertion expired. Check that your server's clock is correct."*

The answer is correct. It came from a smaller, cheaper model, chosen with evidence instead of
a leaderboard position.

---

## Sensible starting shortlists

This is not a ranking. It is a place to begin, subject to the evaluation above.

| Situation | Start with |
|---|---|
| General English, self-hosted, cheap | A small BGE / E5 / GTE model (~384-d) |
| General English, quality first | A base or large BGE / E5 / mxbai model |
| Long documents (2k–8k tokens) | Jina v3/v4, Nomic Embed, ModernBERT-based encoders |
| Multilingual | multilingual-E5, BGE-M3, Qwen3-Embedding, Cohere multilingual |
| No ML infrastructure | OpenAI `text-embedding-3` (MRL-capable), Cohere, Voyage |
| Code search | A dedicated code embedding model, because general models are poor at code |
| Maximum precision, budget available | ColBERT-family models, which keep one vector per token (Part V) |
| Documents with charts, tables, scans | ColPali and ColQwen, which search page images directly (Part VI), instead of running OCR (optical character recognition: turning page images into text) first |

**BGE-M3 deserves a special mention.** It produces dense, sparse *and* multi-vector
representations from a single model, with an 8k-token context window (the most text the
model reads in one go). If we want keyword-style and dense search together without running
three pipelines, it is an unusually pragmatic starting point.

---

## Leaderboard vs our own evaluation

| | MTEB leaderboard | Our one-day evaluation |
|---|---|---|
| Data | Public datasets | Our pages and our users' queries |
| Cost | Free, instant | One day of work |
| Good for | Shortlisting, ruling out toys | Choosing |
| Blind spots | Our domain, contamination, tuning to the average | Only 50 queries, so small gaps are noise here too |
| Reusable afterwards | No | Yes, as a regression test |

**Advantages of the leaderboard.** It is free, broad, and instantly narrows hundreds of models
to a handful.

**Disadvantages of the leaderboard.** It measures someone else's problem, it can be gamed, and
it cannot see Acme's error codes.

**Advantages of our own evaluation.** It measures exactly our problem, it can be broken down
by query type, and it keeps paying off as a regression test.

**Disadvantages of our own evaluation.** It costs a day, and with only 50 queries it can
separate large differences but not small ones. Chapter 19 shows how to tell which is which.

---

## When to use which one

We must use the **leaderboard** when we have hundreds of candidates and need a shortlist of
three or four.

We must use **our own evaluation** when we choose from that shortlist, and again every time
we change anything afterwards.

Many strong teams use both: the leaderboard to shortlist, their own data to decide.

---

### Under the hood

The whole one-day evaluation, including the BM25 row, the fused row and the per-group report:

```python
import numpy as np
from rank_bm25 import BM25Okapi
from sentence_transformers import SentenceTransformer

def dense_top(model, queries, pages, prefixes=("", ""), k=100):
    qp, dp = prefixes
    D = model.encode([dp + p for p in pages], normalize_embeddings=True)
    Q = model.encode([qp + q for q in queries], normalize_embeddings=True)
    return np.argsort(-(Q @ D.T), axis=1)[:, :k]            # top-k page indices per query

def bm25_top(queries, pages, k=100):                         # Chapter 8
    bm25 = BM25Okapi([p.lower().split() for p in pages], k1=1.2, b=0.75)  # library default k1 is 1.5
    return np.array([np.argsort(-bm25.get_scores(q.lower().split()))[:k] for q in queries])

def rrf(a, b, k=100, c=60):                                  # Chapter 40
    fused = []
    for ra, rb in zip(a, b):
        score = {}
        for ranking in (ra, rb):
            for rank, page in enumerate(ranking, start=1):
                score[page] = score.get(page, 0) + 1 / (c + rank)
        fused.append(sorted(score, key=score.get, reverse=True)[:k])
    return np.array(fused)

def recall_at_k(top, gold, k=10):
    """gold[i] = the set of page indices that truly answer query i."""
    return np.array([len(set(row[:k]) & set(g)) / len(g) for row, g in zip(top, gold)])

def report(name, top, gold, groups):                         # groups: np.array of labels
    r = recall_at_k(top, gold)
    cells = "  ".join(f"{g}={r[groups == g].mean():.2f}" for g in np.unique(groups))
    print(f"{name:26s} all={r.mean():.2f}  {cells}")

CANDIDATES = [                                   # 3–4 models, correct prefixes each (Chapter 14)
    ("intfloat/e5-base-v2",   ("query: ", "passage: ")),
    ("BAAI/bge-base-en-v1.5", ("Represent this sentence for searching relevant passages: ", "")),
    ("thenlper/gte-base",     ("", "")),
]

rows = {name: dense_top(SentenceTransformer(name), queries, acme_pages, prefixes)
        for name, prefixes in CANDIDATES}
rows["BM25"] = bm25_top(queries, acme_pages)
rows["best + BM25 (RRF)"] = rrf(rows["BAAI/bge-base-en-v1.5"], rows["BM25"])  # use your best dense row
for name, top in rows.items():
    report(name, top, gold, groups)             # groups[i] = "plain", "paraphrase", "code" or "multi"
```

Means, each row produces a ranked list of pages for every query, and the same `report`
scores every row, so models, BM25 and the fused row are compared on exactly equal terms. The
BM25 row splits text on spaces, so `sso-4012` stays one token and matches only itself. That
is why a BM25 row so often wins the error-code group.

---

### What people get wrong

**"I'll use the #1 model on MTEB."** You will often get a large, slow model tuned for a
benchmark average, and no evidence about your domain.

**"Bigger dimensions, better quality."** Only up to your data's intrinsic dimension
(Chapter 6). Beyond that you are buying cost.

**"I'll evaluate later, after it's built."** You will then be evaluating a system with five
coupled variables and no way to isolate any of them. Build the eval set first.

**"Our data is too specialised to evaluate."** That makes evaluation more necessary, not
less. Specialised domains are exactly where leaderboard rankings invert.

**Changing two things at once.** New model *and* new chunking. Now you cannot attribute the
result. Change one variable per measurement.

---

### Ninja notes

Three things separate a careful evaluation from a superficial one.

**Segment your eval set by query type.** Report recall separately for lookups,
natural-language questions, identifier queries and multi-constraint queries. Models differ
wildly by segment. A model that wins overall may be the worst on the segment that drives your
revenue, as Acme's error codes showed. This single change turns an eval from a number into a
diagnosis.

**Measure index-time cost, not just quality.** Embedding 50 million chunks with a 7B model
versus a 300M model can be the difference between hours and days of GPU time, repeated every
time you re-embed. That cost recurs. The quality difference may not justify it.

**Track the score distribution, not just the ranking.** Suppose a model puts the correct page
at rank 1 with a similarity of 0.91, while rank 2 sits at 0.62. That model has given you a
usable confidence signal. A model that returns 0.88 and 0.87 has not. You will want that
signal in Chapter 55, to detect "we have nothing relevant."

---

### Key takeaways

- **MTEB = Massive Text Embedding Benchmark**, a public leaderboard that ranks embedding
  models on someone else's data. Leaderboards produce shortlists. Your own data makes the
  decision.
- Read the retrieval column, watch for contamination, and treat the top 10–15 as a tie.
- Seven criteria: domain quality, dimensions, context length, speed, languages, licence,
  provenance.
- Build a 50-query gold set in one day. It pays for itself immediately and forever.
- Always include BM25 and a hybrid row as baselines.
- Segment results by query type. That is where the real decisions hide.

### What's next

Part II is complete: we can now obtain good vectors. [Part III](./17-brute-force.md) starts
searching them, beginning with the brute-force baseline that is faster than almost everyone
assumes.

We now know how to read a leaderboard without being fooled by it, which criteria actually
decide the choice, and how one day with our own data turns a guess into evidence.
