---
title: "Rerankers"
chapter: 41
part: "Part V — Beyond One Vector"
slug: rerankers
readingTime: "12 min"
summary: "The last stage before the LLM, and the highest-precision component in the pipeline. Cross-encoders, LLM rerankers, and how to decide how deep to go."
tags: [reranking, cross-encoder, llm-reranker, precision, latency]
prev: 40-hybrid-search
next: 42-clip
---

# Rerankers

**The one-paragraph version.** A reranker reads the query together with each candidate document
and scores the pair directly. It is usually far more accurate than a bi-encoder of similar size,
and far too slow to run over a whole corpus. So it runs over the 50–200 candidates that retrieval
already produced. Once retrieval is decent (ideally hybrid, Chapter 40), adding a reranker is
typically the largest remaining quality improvement, and it needs no changes to the index.

In this chapter, we will learn what a reranker does, why reading the question and the page
together catches what retrieval missed, and how to wire one into a pipeline. We will also compare
the three kinds of reranker, find out how many candidates to rerank, and see when to skip
reranking altogether.

We will cover the following:

- What is a reranker
- Why we need a reranker
- How reranking works
- An example: lifting page 1,140
- The three kinds of reranker
- How deep to rerank
- Problems with rerankers
- When to use which one

---

## What is a reranker

Before jumping in, recall two models from Chapter 13. A bi-encoder turns the question and
each page into separate vectors, so page vectors can be computed in advance. A cross-encoder
reads the question and a page together, as one input, and outputs a single relevance score.

**Reranker = a model that scores each (question, candidate page) pair jointly, used to reorder a
short list that retrieval already found.**

In simple words, retrieval finds a hundred plausible pages quickly. The reranker reads each one
carefully against the question and puts them in the right order.

Think of it like hiring.

- Retrieval is the CV screen: fast, shallow and generous. It looks at a summary and decides
  whether someone is plausibly worth considering.
- Reranking is the interview: slow, deep and specific. It puts the candidate and the job
  description in the same room.
- The job description is our question. The candidates are the pages.

Nobody hires from CVs alone. Nobody interviews ten thousand people either. The two-stage shape
is not a compromise. It is how careful selection works at scale, which is why the same shape
appears in web search, recommendation and advertising.

---

## Why we need a reranker

A bi-encoder compressed each page into a vector *before knowing the question* (Chapter 13).
Whatever it discarded is gone, and the retrieval order is only a rough guess.

Let's watch that guess fail on our running question, *"Does the Pro plan include single
sign-on?"* This is the same first-stage run we saw in Chapter 13. The retriever searches Acme's
40,000 pages and the Librarian hands the top 5 to the Scholar.

| Retrieval rank | Page | What the page is about |
|---|---|---|
| 1 | 212 | Pro plan page: SSO included on Pro |
| 2 | 88 | Setting up single sign-on (how-to guide) |
| 3 | 2,301 | SSO on the Enterprise plan |
| 4 | 45 | Pro plan overview |
| 5 | 9,012 | Troubleshooting SSO login errors |
| ... | ... | ... |
| 23 | 1,140 | The Security add-on |

Page 1,140 is mostly about the Security add-on: audit logs, IP allow-lists, data retention. The
catch, "On Pro, SSO needs the Security add-on for teams under 50 seats", is one sentence among
many, so the page lands at rank 23. The Scholar reads the top 5 and replies, *"Yes, the Pro plan
includes single sign-on."* The answer is wrong for every Pro team under 50 seats.

A cross-encoder sees the question and the page at once, so at every layer every question token
can attend to every page token. That lets it:

- **Check each constraint separately.** "Pro" and "single sign-on" must both be addressed.
- **Handle negation far better.** For the negation twin, *"Which plans do not include single
  sign-on?"*, a single vector barely notices "not". A cross-encoder reads it in context.
- **Tell "mentions the topic" from "answers the question".** Cosine similarity cannot express
  that difference.
- **Weigh evidence.** A definitive one-line answer beats three paragraphs of nearby discussion.

Published results vary with the data, but a cross-encoder reranker commonly adds roughly **5–15
points of nDCG@10** over the retrieval it reranks. No other stage we can add to a working
pipeline reliably adds that much precision.

---

## How reranking works

**Phase 1: Retrieve (fast, generous).**

**Step 1:** Run retrieval (dense, sparse or hybrid) and keep the top 100 candidates.

**Phase 2: Rerank (slow, precise).**

**Step 2:** Pair the question with each candidate's text: 100 pairs.

**Step 3:** Run the pairs through the cross-encoder in batches. Each pair gets one score.

**Step 4:** Sort the candidates by that score, highest first.

**Step 5:** Hand the top 5–10 to the Scholar.

That is it. The index does not change, and the retriever does not change. The reranker slots in
between the Librarian and the Scholar.

---

## An example: lifting page 1,140

Now let's rerank the top 100 candidates, which include page 1,140 at rank 23. The scores below
are toy raw scores from a cross-encoder, the same ones as in Chapter 13.

| New rank | Page | Reranker score | Old rank |
|---|---|---|---|
| 1 | 212: Pro plan page, SSO included on Pro | 9.1 | 1 |
| 2 | **1,140: The Security add-on** | **8.4** | **23** |
| 3 | 45: Pro plan overview | 5.2 | 4 |
| 4 | 2,301: SSO on the Enterprise plan | 3.9 | 3 |
| 5 | 88: Setting up single sign-on | 3.1 | 2 |

Why did page 1,140 jump from 23 to 2? The cross-encoder read the question and the page together.
The page's sentence "On Pro, SSO needs the Security add-on" addresses both "Pro" and "single
sign-on" directly. A page that answers the question, with a condition, outranks pages that are
merely *about* SSO. Page 88 drops to rank 5, because a setup guide never says which plans include
SSO.

The Scholar now reads both pages. It thinks like this: *"Page 212 says SSO is included on Pro.
Page 1,140 says Pro teams under 50 seats also need the Security add-on. So the answer is yes, with
a condition."* It writes, *"Yes. SSO is included on Pro, but teams under 50 seats need the
Security add-on."* The answer is correct.

**Note:** This only worked because page 1,140 was among the 100 candidates. Had we reranked only
the top 10, rank 23 would never have been read. We return to this in "How deep to rerank".

---

## The three kinds of reranker

**1. Cross-encoder rerankers.** A BERT-sized model trained to score (question, passage) pairs.
Latency is about 20–100 ms for 100 candidates on a GPU, more on CPU. Open options include the BGE
reranker family, mxbai rerankers and Jina rerankers. Hosted options include Cohere Rerank and
Voyage. **This is the default, and where you should start.**

**2. LLM rerankers.** We prompt a general language model to judge relevance. There are three
styles.

- **Pointwise:** score one passage at a time, for example "rate this passage 0–10".
- **Pairwise:** compare two passages, "which of these two is more relevant?"
- **Listwise = show the model several passages at once and ask it to reorder the whole list.**

LLM rerankers are more accurate on subtle or reasoning-heavy relevance, and dramatically more
expensive and slower. RankGPT-style listwise prompting is among the strongest published variants. It
slides a window of 20 passages over the candidate list, from the bottom up, 10 places at a time.

**3. Late-interaction rerankers.** ColBERT-style MaxSim over the candidates (Chapter 36). They sit
between the other two in cost and quality, and they give a per-token explanation. They are a good
choice when token vectors are already stored. Computing them on the fly for 100 passages adds
about 10–30 ms of encoding (Chapter 37).

| | Cross-encoder | LLM reranker | ColBERT rerank |
|---|---|---|---|
| Latency (100 docs) | 20–100 ms | 1–10 s | 5–20 ms with stored token vectors (+10–30 ms to encode on the fly) |
| Cost | Low | High | Low |
| Quality | Excellent | Excellent+ on hard queries | Very good |
| Explainable | No | Yes, if asked | Yes, per token |

---

## How deep to rerank

The parameter that matters most is how many candidates we pass in. The curve has a knee, and it is
worth finding. Illustrative numbers for a BERT-sized cross-encoder on one GPU:

| Candidates reranked | nDCG@10 gain | Latency |
|---|---|---|
| 10 | +2 | 10 ms |
| 50 | +7 | 30 ms |
| 100 | +9 | 60 ms |
| 200 | +10 | 120 ms |
| 500 | +10.5 | 300 ms |

Gains flatten around 100–200, while latency grows linearly. **Start at 100.**

The hard ceiling to remember: **the reranker can only reorder what retrieval found.** If the right
page is at rank 400 and we rerank 100, no reranker helps. Page 1,140 was rescued at rank 23, not
at rank 400. This is why Chapter 40's fusion matters: it is what gets the right page into the
window.

A useful diagnostic is to measure two numbers. `recall@100` is how often the right pages reached
the reranker. `recall@10` after reranking is how often they reached the Scholar. If recall@100 is
0.75, the ceiling is 0.75 no matter how good the reranker is. In simple words, the effort then
belongs in retrieval, not in reranking.

---

## Problems with rerankers

Why not rerank everything, deeply, with the best model? There are five reasons.

**Problem 1: Latency and cost grow with depth.** Every candidate is a full model pass. 500
candidates cost five times as much as 100, for about one and a half extra points.

**Problem 2: It cannot see what retrieval missed.** If page 1,140 had been at rank 400, the
reranker would never have read it.

**Problem 3: Its scores are not calibrated.** A score of 8.4 means "better than 5.2 for this
question". It does not mean "relevant" in any absolute sense, and it is not comparable across
questions or across reranker models.

**Problem 4: It truncates long pages.** A reranker with a 512-token limit silently reads only the
start of a 1,500-token chunk. If page 1,140's add-on sentence were near the end, the reranker
would never see it.

**Problem 5: It ignores redundancy.** It scores each page independently, so it will happily
return five near-identical pages about SSO on Pro. The Scholar then gets one fact five times.

---

## When to use which one

We must use **a cross-encoder** by default, on about 100 candidates, in any RAG system that has a
working retriever.

We must use **an LLM reranker** only for genuinely hard, reasoning-heavy questions, routed there
selectively, when a few seconds of extra latency is acceptable.

We must use **a ColBERT reranker** when token vectors are already stored, or when we want a
per-token explanation of every result.

We must **skip reranking** for easy navigational queries, where retrieval's top result already
scores far above the rest.

Many strong systems use hybrid retrieval, a cross-encoder on the top 100, and a diversity step
before handing pages to the Scholar.

---

### Under the hood

A cross-encoder reranker, Steps 2–5:

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("BAAI/bge-reranker-base", max_length=512)    # BERT-base sized

def rerank(query, candidates, top_k=10, batch_size=32):
    pairs = [(query, c.text) for c in candidates]                  # Step 2
    scores = reranker.predict(pairs, batch_size=batch_size)        # Step 3
    order = scores.argsort()[::-1][:top_k]                         # Steps 4–5
    return [(candidates[i], float(scores[i])) for i in order]
```

In simple words, we build 100 (question, page) pairs, score them in batches of 32, and keep the
best 10.

An LLM listwise reranker, sketched:

```python
PROMPT = """Rank these passages by how well they answer the question.
Return only a comma-separated list of passage numbers, best first.

Question: {query}

{passages}"""

# llm() calls your language model. parse_ranking() turns "3, 0, 7" into [3, 0, 7].
def rank_window(query, passages):
    body = "\n\n".join(f"[{i}] {p.text[:600]}" for i, p in enumerate(passages))
    order = parse_ranking(llm(PROMPT.format(query=query, passages=body)))
    order = [i for i in dict.fromkeys(order) if 0 <= i < len(passages)]   # drop bad numbers
    order += [i for i in range(len(passages)) if i not in order]          # keep forgotten ones
    return [passages[i] for i in order]

def llm_rerank(query, candidates, window=20, step=10):
    ranked, end = list(candidates), len(candidates)
    while True:                                          # slide from the bottom to the top
        start = max(end - window, 0)
        ranked[start:end] = rank_window(query, ranked[start:end])
        if start == 0:
            return ranked
        end -= step
```

In simple words, the model sorts 20 passages at a time, starting at the bottom of the list. Each
window shares 10 passages with the next one up, so a strong page can climb all the way to the top.
For 100 candidates that is 9 calls. Page 1,140 at rank 23 is first read in the seventh call, and
it climbs toward the top in the last two.

Two cautions on the LLM version. It is sensitive to the *order* in which passages are presented,
because models show position bias. A common mitigation is to run it twice with shuffled input and
combine the results. And parsing failures are routine, which is why `rank_window` drops invalid
numbers and keeps any passage the model forgot in its input order, rather than crashing.

---

### What people get wrong

**Skipping it.** This is the most common and most costly omission in RAG systems. If your
retrieval is decent and your pipeline has no reranker, that is almost certainly your largest
remaining win.

**Reranking too few candidates.** Reranking 10 gets you very little. The value comes from the
reranker's ability to promote something retrieval ranked 23rd, or 47th.

**Assuming the reranker fixes retrieval.** It cannot see what was not retrieved.

**Treating reranker scores as calibrated.** They are raw logits, or logits squashed through a
sigmoid so they look like probabilities. Either way, use them to order pages within one query,
not to threshold across queries, unless you calibrate on held-out data. That is worth doing if
you need a "nothing relevant" signal (Chapter 55). Raw scores are not comparable across reranker
models either, so re-calibrate whenever the model changes.

**Ignoring the truncation limit.** A reranker with a 512-token limit silently truncates a
1,500-token chunk and scores only the beginning. If your chunks are long, either use a
long-context reranker or score passages within the chunk.

**Using an LLM reranker by default.** Ten to a hundred times the cost and latency of a
cross-encoder, for a gain that is usually small, except on genuinely hard, reasoning-heavy
queries. Route selectively.

---

### Ninja notes

**Adaptive reranking is the optimisation most teams have not made.** Not every query needs it. A
navigational lookup where the top result scores far above the second is already solved. The
reranker will not change the order, and you just spent 60 ms confirming it.

A workable policy: compute the **score gap** between retrieval's 1st and 5th results. A large gap
with a high top score means skip reranking. A small gap, or a low top score, means rerank, and
possibly rerank deeper. On typical query mixes this cuts average latency substantially while
leaving quality unchanged or better, because the saved budget can be spent on the hard tail.

**Diversity matters more than you think.** A reranker optimises relevance per document,
independently. It will happily return five near-identical passages. For RAG that is a real loss:
your Scholar gets one fact five times instead of five facts.

**Maximal Marginal Relevance (MMR)** fixes this by trading relevance against novelty:

$$ \text{MMR} = \arg\max_{d \in R \setminus S} \left[ \lambda \cdot \text{rel}(d, q) - (1-\lambda) \max_{s \in S} \text{sim}(d, s) \right] $$

Select greedily from the reranked candidates ($R$) not yet chosen, penalising each by its
similarity to what you have already selected ($S$). With $\lambda \approx 0.7$ you keep relevance
dominant while breaking up duplicate clusters. Apply it *after* reranking, as the final selection
step before context assembly (Chapter 53). It is cheap, it is about ten lines, and on corpora
with redundancy it noticeably improves answer completeness.

```python
def mmr(cand_vecs, rel, k=5, lam=0.7):
    """cand_vecs: (n, d) unit vectors. rel: (n,) reranker scores rescaled to 0–1."""
    selected, rest = [], list(range(len(cand_vecs)))
    while rest and len(selected) < k:
        def value(i):
            redundancy = max((float(cand_vecs[i] @ cand_vecs[j]) for j in selected), default=0.0)
            return lam * rel[i] - (1 - lam) * redundancy
        best = max(rest, key=value)
        selected.append(best)
        rest.remove(best)
    return selected
```

Rescale the reranker scores to 0–1 first, so relevance and similarity live on the same scale.

---

### Key takeaways

- **Reranker = a model that scores (question, page) pairs jointly, to reorder a short list.**
  Usually far more accurate than a bi-encoder of similar size, far too slow for corpus-scale
  search.
- Once retrieval is decent (ideally hybrid), adding one is typically the largest remaining
  quality win, about +5–15 nDCG@10.
- In our example, it lifted page 1,140 from rank 23 to rank 2, and the Scholar's half answer
  became a full one.
- Rerank ~100 candidates. Gains flatten beyond 100–200 while latency grows linearly.
- The reranker's ceiling is retrieval's recall@100. Measure both.
- Cross-encoders are the default. LLM rerankers (pointwise, pairwise, listwise) are for hard
  queries only. ColBERT MaxSim fits when token vectors are stored.
- Scores are not calibrated, and not comparable across queries or models.
- Watch the reranker's own token limit, route adaptively by score gap, and apply MMR afterwards
  for diversity.

### What's next

Part V is complete. [Part VI](./42-clip.md) leaves text behind entirely: shared embedding spaces
for images, the failure of OCR pipelines (which turn scans into text), and ColPali.

We now know how a reranker reads each candidate against the question, why that rescues pages like
1,140 that retrieval ranked too low, and how deep to rerank before the cost stops paying off.
