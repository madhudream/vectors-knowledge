---
title: "Hybrid Search and Fusion"
chapter: 40
part: "Part V — Beyond One Vector"
slug: hybrid-search
readingTime: "11 min"
summary: "Run keyword search and vector search, then merge the two ranked lists. A few lines of code, no training, and the cheapest, most reliable recall win in this book."
tags: [hybrid, rrf, fusion, bm25, dense, reciprocal-rank-fusion]
prev: 39-splade
next: 41-rerankers
---

# Hybrid Search and Fusion

**The one-paragraph version.** Dense retrieval misses exact terms. Sparse retrieval misses
paraphrases. So we run both and merge the two ranked lists. **Reciprocal Rank Fusion** (RRF)
merges by *rank* rather than by score, which sidesteps the fact that BM25 scores and cosine
similarities are incomparable quantities. It takes a handful of lines, needs no training, and
often delivers a larger gain than switching to a better embedding model. RRF is the best *untuned*
default. With an eval set, a carefully tuned score fusion can beat it.

In this chapter, we will learn how hybrid search combines a keyword retriever and a vector
retriever, and why merging by rank works when adding scores does not. We will also work through
RRF by hand, see what it gains, and learn when a tuned alternative is worth the effort.

We will cover the following:

- What is hybrid search
- Why we need hybrid search
- Why we cannot just add the scores
- What is Reciprocal Rank Fusion
- An example of RRF, by hand
- The hybrid architecture
- What we actually gain
- RRF vs tuned score fusion
- When to use which one

---

## What is hybrid search

**Hybrid search = run keyword (sparse) search and vector (dense) search on the same query, then
merge their ranked lists into one.**

In simple words, we ask two different Librarians and combine their answers.

Think of it like a detective with two consultants.

- The **archivist** is BM25 or SPLADE. The archivist finds any exact phrase, name or error code
  instantly, but shrugs at a paraphrase.
- The **analyst** is dense retrieval. The analyst understands meaning and finds the right page
  even in different words, but asked for error `SSO-4012`, offers something that merely looks
  similar.
- The **detective** is the fusion step. The detective's job is not to pick the better
  consultant. It is to combine their answers.

A detective who consults only one is working with half a brain.

---

## Why we need hybrid search

Let's give each consultant one of our twin questions from Acme's knowledge base. In both cases,
the Librarian hands the Scholar the top 3 pages.

**The exact-token twin goes to dense search.** A user asks, "What does error SSO-4012 mean?"
Acme has dozens of SSO error pages: `SSO-4011`, `SSO-4021`, `SSO-4102` and more. To a dense
embedding they all look nearly identical. Dense search returns `SSO-4021`, `SSO-4102` and
`SSO-4011` as its top 3. The right page sits at rank 4. The Scholar explains an error the user
does not have. The answer is wrong.

**The paraphrase twin goes to BM25.** A user asks whether their team can log in with their
company accounts. The answer starts at page 212, "SSO is included on the Pro plan." But the
question and page 212 share no words. BM25 returns a password-reset page, an account-settings
page and a team-management page. Page 212 is nowhere in its list. The answer is wrong.

Notice that the two retrievers fail in *opposite* directions. Dense search got the paraphrase
right: its top 3 holds both page 212 and page 1,140, the page with the Security add-on catch. BM25
nails the error code. So running both should cover both. As we will see, that holds for the fused
shortlist, but not always for its top 3.

Now, the question is, how do we merge two lists into one?

---

## Why we cannot just add the scores

The obvious approach fails, and understanding why leads straight to the right one.

```python
final = 0.5 * bm25_score + 0.5 * cosine_score     # broken
```

**BM25 scores are unbounded.** They depend on IDF, term frequency and document length, and can
run from 0 to 40 or beyond. **Cosine similarities are bounded** in $[-1, 1]$ and, for real models,
often crammed into about $[0.55, 0.95]$ (Chapter 3). Add them, and BM25 dominates completely.

Normalising each list per query helps a little and brings new problems. Min-max normalisation
(rescaling so the lowest score is 0 and the highest is 1) is dominated by outliers. And the
*scale* of scores varies enormously by query. A query with one very rare term, like `SSO-4012`,
produces high BM25 scores. A query of common words produces low ones. Normalising makes them look
comparable when they are not.

The insight that resolves this: **ranks are comparable even when scores are not.** Being 3rd on
the archivist's list and 3rd on the analyst's list means the same thing, whatever the numbers
behind them.

---

## What is Reciprocal Rank Fusion

**Reciprocal Rank Fusion (RRF) = give each page 1 / (k + its rank) from every list it appears in,
add those up, and sort.**

$$ \text{RRF}(d) = \sum_{r \in \text{retrievers}} \frac{1}{k + \text{rank}_r(d)} $$

with $k = 60$ by convention. Means, a page ranked 1st earns 1/61, a page ranked 2nd earns 1/62,
and a page missing from a list earns nothing from it.

Here is the algorithm as steps.

**Step 1:** Run every retriever on the query, to a depth of about 100 each.

**Step 2:** For each page in each list, compute 1 / (60 + its rank in that list).

**Step 3:** Add up each page's contributions across all the lists.

**Step 4:** Sort pages by their total, highest first.

```python
def rrf(result_lists, k=60, top_n=100):
    scores = {}
    for results in result_lists:
        for rank, doc_id in enumerate(results, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)[:top_n]
```

That is the entire algorithm. Six lines, no training, and no score normalisation.

**Why $k = 60$?** It damps the influence of the very top ranks. Without it ($k = 0$), rank 1 scores
1.0 and rank 2 scores 0.5, so one retriever's top hit would dominate. With $k = 60$, rank 1 scores
0.0164 and rank 2 scores 0.0161, nearly equal. The effect is that **appearing on both lists
matters more than being first on either**, as long as the page ranks in about the top 60 of both.
That is exactly what we want from a consensus mechanism. A page at rank 5 in both lists
(2/65 ≈ 0.031) beats a page at rank 1 in one list and absent from the other (1/61 ≈ 0.016).

**Weighting.** When one retriever is genuinely stronger on our data, we can weight it:

$$ \text{RRF}(d) = \sum_r \frac{w_r}{k + \text{rank}_r(d)} $$

Tune $w_r$ on an eval set. Modest weights, say 1.0 for dense and 0.7 for sparse, are typical.
Large weights defeat the purpose.

---

## An example of RRF, by hand

Let's fuse the two lists for "What does error SSO-4012 mean?" To keep the arithmetic short, we
look only at the top 5 of each list.

- **BM25:** 1. `SSO-4012` page, 2. SSO troubleshooting overview, 3. `SSO-4011`, 4. SAML setup
  guide, 5. `SSO-4102`.
- **Dense:** 1. `SSO-4021`, 2. `SSO-4102`, 3. `SSO-4011`, 4. `SSO-4012` page, 5. SAML setup
  guide.

Now we apply Steps 2 and 3 to every page:

| Page | BM25 rank | Dense rank | RRF score |
|---|---|---|---|
| `SSO-4012` | 1 | 4 | 1/61 + 1/64 = **0.0320** |
| `SSO-4011` | 3 | 3 | 1/63 + 1/63 = 0.0317 |
| `SSO-4102` | 5 | 2 | 1/65 + 1/62 = 0.0315 |
| SAML setup guide | 4 | 5 | 1/64 + 1/65 = 0.0310 |
| `SSO-4021` | not in list | 1 | 1/61 = 0.0164 |
| SSO troubleshooting overview | 2 | not in list | 1/62 = 0.0161 |

Step 4 sorts them. The `SSO-4012` page comes first. The answer is correct.

Look at what happened to `SSO-4021`. It was dense search's number one, yet it fell to fifth,
because the archivist never vouched for it. And `SSO-4012` won without topping the dense list,
because *both* consultants had it in their top 5. That is the consensus property doing its job.

The paraphrase twin shows the limit. BM25's list is password reset, account settings, team
management, company profile and a session-timeout page. Dense search's list is page 212, the SAML
setup guide, page 1,140, password reset and account settings. After fusion, the top 3 are
password reset (0.0320), account settings (0.0315) and page 212 (0.0164). Compared with BM25
alone, page 212 is now in the top 3. But dense search alone had both page 212 and page 1,140 in
its top 3, and fusion pushed page 1,140 down to a tie for fifth with team management
(1/63 ≈ 0.0159). Two pages that *both* consultants listed, and neither of them relevant, outranked
it. So the only relevant page the Scholar reads is page 212. It says SSO is included on Pro, with
no Security add-on caveat for teams under 50 seats. The answer is wrong. On this question, plain
fusion did worse than dense search alone.

**Note:** Both pages are in the fused list, just not in its top 3. At a cut of 3, fusion can cost
us a page that one retriever ranked well. So we do not cut there. We fuse 100 deep, which keeps
page 1,140 in the shortlist, and a reranker then reads the question with each page and moves the
right ones to the top (Chapter 41). Fusion's job is to get the right pages *into* the shortlist.
Putting them *first* is the reranker's job.

---

## The hybrid architecture

```
                 query
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     BM25 /     dense      (optional)
     SPLADE      ANN        ColBERT
     top 100    top 100     top 100
        └──────────┼──────────┘
                   ▼
            RRF fusion → top 100
                   ▼
            cross-encoder rerank → top 10
                   ▼
                  LLM
```

Three notes on practice.

**Retrieve 100 from each, not 10.** Fusion needs depth to find consensus. In our example we used
5 only to keep the arithmetic short. The top 10 from each list gives the fusion almost nothing to
work with.

**Run the retrievers in parallel.** They are independent. Latency is the slower of the two, not
the sum, typically 20–40 ms in total.

**Fusion and reranking are complements, not alternatives.** Fusion maximises *recall* into the
candidate set. The reranker maximises *precision* within it. This is Chapter 13's principle:
spend recall early, precision late.

---

## What we actually gain

Illustrative numbers, but consistent with what teams typically see when they measure:

| System | recall@10 | Notes |
|---|---|---|
| BM25 only | 0.61 | Strong on identifiers, weak on paraphrase |
| Dense only | 0.68 | Strong on paraphrase, fails on identifiers |
| **RRF(BM25, dense)** | **0.79** | **Often a larger gain than a single-model upgrade** |
| RRF + cross-encoder rerank | 0.86 | The standard production stack |

The jump from 0.68 to 0.79 is the point. **That is often a bigger improvement than upgrading to a
better embedding model, and it costs an afternoon rather than a re-embed of the whole corpus.**

Split the results by query type and the mechanism becomes visible. Error-code queries like
`SSO-4012` go from poor under dense-only to near-perfect. Paraphrase queries stay high. Put
simply, hybrid is not a uniform lift. It is the union of two coverage sets.

---

## RRF vs tuned score fusion

RRF is not the last word. Its rival is a **convex combination**: normalise each retriever's
scores, then mix them with one weight.

$$ \text{score}(d) = \alpha \cdot \text{norm}(\text{dense}(d)) + (1 - \alpha) \cdot \text{norm}(\text{bm25}(d)) $$

Here $\alpha$ is a number between 0 and 1 that we tune. In simple words, it is the broken
"add the scores" idea from earlier, done carefully: normalise first, then *learn* the mix from
data instead of guessing 0.5.

Bruch, Gai and Ingber (2023) compared the two directly. A tuned convex combination of normalised
scores can beat RRF, and it needs only a modest number of labelled queries to tune. Once $\alpha$
is tuned, the choice of normalisation matters little. They also found that RRF is sensitive to its
$k$, so "$k = 60$ always" is a convention, not a law.

| | RRF | Tuned convex combination |
|---|---|---|
| Needs labelled queries | No | Yes, a modest eval set |
| Uses score magnitudes | No, ranks only | Yes |
| Sensitive to | $k$ | Mainly $\alpha$ (normalisation matters little once $\alpha$ is tuned) |
| Quality with no tuning | Best default | Risky, with a guessed $\alpha$ |
| Quality after tuning | Good | Can be better |

So the rule is simple. **Start with RRF. Graduate to tuned fusion when you can measure.**

---

## When to use which one

We must use **dense search alone** when queries are conceptual prose with no identifiers, and an
eval set confirms hybrid adds nothing.

We must use **BM25 alone** when queries are almost all exact codes, names and part numbers.

We must use **hybrid with RRF** by default, whenever queries mix both kinds, and especially before
we have an eval set.

We must use **a tuned convex combination** once we have a golden set (Chapter 19) and can show it
beats RRF on our own queries.

Many strong systems use hybrid retrieval and a reranker together, because fusion fills the
shortlist and the reranker orders it.

---

### Under the hood

A complete hybrid retriever, with the four RRF steps marked:

```python
import concurrent.futures as cf

class HybridRetriever:
    def __init__(self, bm25, dense, k_rrf=60, weights=(1.0, 1.0)):
        self.bm25, self.dense, self.k, self.w = bm25, dense, k_rrf, weights

    def search(self, query, depth=100, top_n=100):
        with cf.ThreadPoolExecutor(2) as pool:                        # Step 1, in parallel
            f_sparse = pool.submit(self.bm25.search, query, depth)
            f_dense  = pool.submit(self.dense.search, query, depth)
            lists = [f_sparse.result(), f_dense.result()]

        scores = {}
        for w, results in zip(self.w, lists):                         # Steps 2–3
            for rank, doc_id in enumerate(results, start=1):
                scores[doc_id] = scores.get(doc_id, 0.0) + w / (self.k + rank)
        return sorted(scores, key=scores.get, reverse=True)[:top_n]   # Step 4
```

Fed the two `SSO-4012` lists from our example, it returns `SSO-4012`, `SSO-4011`, `SSO-4102`,
the SAML guide, `SSO-4021` and the overview, in that order. It is under twenty lines, including
the parallelism. There is no training step and no model to maintain. If one retriever degrades, the
other still contributes.

And the tuned alternative, for when we have an eval set:

```python
def convex_fusion(bm25_hits, dense_hits, alpha=0.6, top_n=100):
    """bm25_hits, dense_hits: {doc_id: raw score}. Tune alpha on your eval set."""
    def minmax(h):
        lo, hi = min(h.values()), max(h.values())
        return {d: (s - lo) / (hi - lo + 1e-9) for d, s in h.items()}
    b, v = minmax(bm25_hits), minmax(dense_hits)
    # A page missing from a list gets 0.0 there, the same as that list's lowest hit.
    fused = {d: alpha * v.get(d, 0.0) + (1 - alpha) * b.get(d, 0.0) for d in b.keys() | v.keys()}
    return sorted(fused, key=fused.get, reverse=True)[:top_n]
```

That is, sweep `alpha` from 0 to 1 on the golden set and keep the best value. Never ship the
default 0.6 untested.

---

### What people get wrong

**Guessing weights for score fusion.** Adding raw or hand-weighted scores is fragile and
query-dependent, and usually worse than RRF. A *tuned* convex combination, fitted on an eval set,
can beat RRF (Bruch, Gai & Ingber, 2023). Start with RRF, and graduate when you can measure.

**Fusing shallow lists.** The top 10 from each gives the fusion little to fuse. Use 100.

**Running retrievers sequentially.** It doubles latency for no reason.

**Treating $k = 60$ as sacred.** It is a sensible default. RRF is sensitive to $k$, so sweep it if
you are tuning anything else.

**Assuming hybrid always wins.** On a corpus of clean natural-language prose with no identifiers,
dense alone may match it. Measure, and measure *by segment*, because the average can hide a
doubling on the segment you care about.

**Forgetting that the sparse side needs its own tuning.** BM25's $b$ parameter should usually be
lowered for uniform RAG chunks (Chapter 8). A badly tuned BM25 contributes a badly ranked list to
the fusion.

**Skipping it because it feels unsophisticated.** It is the cheapest and most reliable recall win
in this book.

---

### Ninja notes

**Fusion generalises beyond two retrievers.** Any ranked list can join: BM25, dense, SPLADE,
ColBERT, a title-only field search, a recency-ordered list, a popularity-ordered list, results
from a rewritten query (Chapter 52), results from each sub-question of a decomposed query. RRF
takes them all with no modification.

This makes it the natural aggregation point for **multi-query retrieval**. Generate three
paraphrases of the user's question, retrieve for each, and fuse all the lists. The consensus
property means documents relevant to the underlying intent, rather than to one particular
phrasing, rise to the top.

**Consider the distribution-based alternative when you need scores, not just ranks.** RRF
discards score magnitude, so you lose the ability to say "nothing here is relevant" (Chapter 55's
confidence signal). If you need that, fit a per-retriever score normalisation on a held-out set,
for example converting scores to percentiles of their empirical distribution, and fuse the
normalised scores. It is more work and more fragile, and it preserves calibration. Use it only
when you specifically need the signal.

**Deduplicate before fusing.** If the same content exists as two near-identical chunks, both may
appear in both lists, and their fused scores will crowd out genuine diversity. MinHash
deduplication at ingestion time (Chapter 21) prevents this, and it is one more reason to run it.

---

### Key takeaways

- **Hybrid search = sparse search + dense search on the same query, merged into one ranked
  list.**
- Dense and sparse fail in opposite directions: dense blurs `SSO-4012` with `SSO-4021`, and BM25
  misses the paraphrase. Running both covers both in the fused shortlist of ~100. Its top few can
  still be worse than one retriever alone, so a reranker orders it.
- Scores from different retrievers are incomparable, so fuse by **rank**, not raw score.
- **RRF** = $\sum_r 1/(k + \text{rank}_r)$ with $k = 60$. Six lines, no training.
- $k = 60$ makes appearing on *both* lists matter more than topping *one*, for pages in about the
  top 60 of both.
- RRF is the best *untuned* default. A convex combination tuned on an eval set can beat it, and
  RRF itself is sensitive to $k$.
- Retrieve ~100 per retriever, run them in parallel, then rerank the fused list.
- RRF accepts any number of ranked lists, including multi-query and multi-field.

### What's next

Fusion maximises what reaches the shortlist. [Chapter 41](./41-rerankers.md) maximises what
happens to it, with the single highest-precision component in the pipeline.

We now know why keyword and vector search fail in opposite directions, how RRF merges their lists
by rank, and when a tuned fusion is worth the extra work.
