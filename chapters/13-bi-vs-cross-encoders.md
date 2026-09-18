---
title: "Bi-Encoders vs Cross-Encoders"
chapter: 13
part: "Part II — Where Embeddings Come From"
slug: bi-vs-cross-encoders
readingTime: "15 min"
summary: "Encode separately and be fast, or read together and be right. This single trade-off is the skeleton of every serious retrieval pipeline — and late interaction is the compromise between them."
tags: [architecture, bi-encoder, cross-encoder, reranking, latency]
prev: 12-contrastive-learning
next: 14-asymmetry-and-instructions
---

# Bi-Encoders vs Cross-Encoders

**The one-paragraph version.** There are two basic ways for a model to judge how well a page
answers a question. A **bi-encoder** turns the question and the page into vectors
*separately*, so every page's vector can be computed ahead of time and searched in
milliseconds. A **cross-encoder** reads the question and the page *together* and gives back
one relevance score. It is far more accurate, but it needs a full run of the model for every
(question, page) pair, so it can never search a whole library. Real systems use both: the
bi-encoder fetches about 100 candidates, and the cross-encoder puts them in the right order.
Much of Part V is an attempt to get cross-encoder quality at bi-encoder speed.

In this chapter, we will learn about bi-encoders and cross-encoders, the two ways a model can
compare a question with a page. We will see why one is fast and the other is accurate. We
will watch a bi-encoder give half an answer to our Pro plan question, and watch a
cross-encoder fix it. We will also see the two-stage pipeline that nearly every production
system uses, and the third option that sits between the two.

We will cover the following:

- What is a bi-encoder
- What is a cross-encoder
- An analogy: two ways to hire
- Why a cross-encoder cannot search the whole library
- An example: the Pro plan question, before and after
- How the two-stage pipeline works
- Late interaction: the third option
- Bi-encoder vs cross-encoder
- When to use which one

---

## What is a bi-encoder

Every embedding model we have met so far, including Chapter 12's, works the same way. Text
goes in, one vector comes out.

**Bi-encoder = encode the question and the page separately, then compare the two vectors.**

"Bi" means two. There are two encoding runs: one for the question, one for the page.

```
question → [encoder] → q  (768 numbers)
page     → [encoder] → d  (768 numbers)     ← computed once, when we build the index
score    = q · d
```

Means, the page never sees the question. We embed all 40,000 pages of Acme's knowledge base
once, when we build the index, and store the vectors in the Card Catalog. The vector for page
212 was fixed weeks before anyone asked about single sign-on.

When a question arrives, we run the encoder once, on the question. Then we compare its vector
with the stored ones using the dot product (Chapter 4). We can score all of them in one matrix
multiplication (Chapter 17), or only a small part of them through an index (Part IV).

In simple words, a bi-encoder does the hard work in advance and leaves only arithmetic for
question time.

**Two-tower model = another name for a bi-encoder: one tower encodes questions, the other
encodes pages.**

The two towers are simply the two encoding paths. In most text search both towers share the
same weights, so it is really one model used twice. Chapter 14 looks at when the two towers should be
different models.

---

## What is a cross-encoder

**Cross-encoder = feed the question and the page into the model together, and get back one
score.**

```
[CLS] question [SEP] page [SEP] → [encoder] → score  (one number)
```

`[CLS]` is the special summary token from Chapter 11. `[SEP]` is a separator token that marks
where the question ends and the page begins. In simple words, we glue the question and the
page into one piece of text and ask the model, "How well does this page answer this
question?"

Because the question and the page sit in the same sequence, **every question token can attend
to every page token, at every layer**. Attention (Chapter 10) is the step where each token
looks at all the other tokens and updates itself based on what it finds. So the word "Pro" in
the question can look straight at "On Pro" in the page. The words "single sign-on" can look
straight at "SSO".

Notice what does *not* come out: a vector. So nothing about the page can be computed in
advance, and nothing can be stored in a Card Catalog. Every (question, page) pair costs one
full run of the model, which is called a **forward pass**.

A cross-encoder is built for one job: reranking. **Reranking** means taking a short list of
candidates that something cheaper has already found, and putting them in a better order. A model
used for this job is called a *reranker*. Chapter 41 covers rerankers in depth, including ones
built on LLMs.

---

## An analogy: two ways to hire

Think of it like hiring for one job that has 10,000 applicants.

**The bi-encoder way.** Each applicant writes a one-page summary of themselves before they
know which job they are applying for. We write a one-page summary of the job. Then we compare
the summaries.

**The cross-encoder way.** For each applicant, we read their full CV *next to* the job
description and think carefully about the fit.

Here is the mapping:

- The job description is the question.
- Each applicant is a page in the Great Library.
- A one-page summary is a vector.
- Reading a CV next to the job description is the cross-encoder's joint reading.

The summaries are fast, because they are written once and reused for every job. But they are
lossy. A summary written without knowing the job cannot highlight the one detail
this job needs.

The careful reading is accurate. We might spot a side project that is exactly what this team
is missing. But 10,000 careful readings for every job does not scale.

So what does a sensible recruiter do? Screen all 10,000 by summary, shortlist 50, and read
those 50 properly. Two stages, each doing what it is good at.

That is the design of most serious production search systems.

---

## Why a cross-encoder cannot search the whole library

Now, the question is, if the cross-encoder is so much more accurate, why not use it for
everything?

Let's put numbers on it. These are rough figures for a base-sized model on one modern GPU (the
graphics chip that runs neural networks quickly):

| | Bi-encoder | Cross-encoder |
|---|---|---|
| Work done ahead of time | Encode every page once | None possible |
| Work per question | 1 encode + a vector search | 1 forward pass **per page** |
| 1M pages, one question | ~10 ms | ~3–17 minutes |
| 100 pages, one question | ~10 ms | ~20–100 ms |
| Typical nDCG@10 gain | baseline | +5 to +15 points |
| Scales to | Billions of pages | Hundreds of pages |

nDCG@10 is a score for how many good pages land in the top 10 and how well they are ordered. It
runs from 0 to 1 and is usually quoted as points out of 100. Chapter 19 defines it properly.

Look at the "1M pages" row. A million pairs is 10,000 times the work of 100 pairs, so one
question takes minutes. No tuning fixes that. The cross-encoder produces nothing to index, so
it can only ever be a second stage.

Now look at the "100 pages" row next to the nDCG row. If the bi-encoder hands over 100
candidates, the cross-encoder adds most of its accuracy for a fixed ~20–100 ms. That cost is
the same whether the library holds 40,000 pages or 40 million.

So the two-stage design is not a compromise. It is the correct design.

---

## An example: the Pro plan question, before and after

Let's watch both approaches on our running question:

> *"Does the Pro plan include single sign-on?"*

Remember from Chapter 1: page 212 says SSO is included on the Pro plan. Page 1,140 adds the
catch: on Pro, SSO needs the Security add-on for teams under 50 seats. A correct answer needs
both pages.

Let's first see what a bi-encoder alone does. We embed the question, search Acme's 40,000
page vectors, and hand the top 5 to the Scholar. Here is what comes back. The scores are toy
numbers, chosen to show the shape of the problem.

| Rank | Page | What the page is about | Bi-encoder score |
|---|---|---|---|
| 1 | 212 | Pro plan page: SSO included on Pro | 0.82 |
| 2 | 88 | Setting up single sign-on (how-to guide) | 0.80 |
| 3 | 2,301 | SSO on the Enterprise plan | 0.79 |
| 4 | 45 | Pro plan overview | 0.77 |
| 5 | 9,012 | Troubleshooting SSO login errors | 0.76 |
| … | … | … | … |
| 23 | 1,140 | The Security add-on | 0.68 |

Why is page 1,140 all the way down at rank 23?

Page 1,140 is mostly about the Security add-on: audit logs, IP allow-lists, data retention.
The SSO caveat is one sentence among many. Its single vector is a blend of all those topics,
so its dot sits in the "security add-on" part of the Map Room, not the "Pro plan SSO" part.
The page was embedded long before anyone asked a question, so its vector could not know which
sentence would matter.

The Scholar reads the top 5. It thinks like this: "Page 212 says SSO is included on Pro.
Page 88 explains how to set it up, and page 9,012 how to fix login errors. Nothing says
otherwise." It answers:

> *"Yes, the Pro plan includes single sign-on."*

The answer is wrong. It is half right, which is worse, because it sounds complete. A 30-person
team on Pro will run straight into the add-on.

Now, let's see what a two-stage system does. The bi-encoder fetches the top 100 instead of
the top 5. Page 1,140 is in that list, at rank 23. Then the cross-encoder reads the question
next to each of the 100 pages, one pair at a time, and we sort by its scores:

| New rank | Page | Cross-encoder score | Old rank |
|---|---|---|---|
| 1 | 212 | 9.1 | 1 |
| 2 | 1,140 | 8.4 | 23 |
| 3 | 45 | 5.2 | 4 |
| 4 | 2,301 | 3.9 | 3 |
| 5 | 88 | 3.1 | 2 |

These are raw model scores, not probabilities (and toy numbers too). Only their order
matters.

Why did page 1,140 jump from rank 23 to rank 2? When the cross-encoder reads the pair, it sees
the sentence "On Pro, SSO needs the Security add-on for teams under 50 seats." The question's
tokens for "Pro plan" and "single sign-on" attend directly to it. The model does not have to
hope that a summary made in advance kept that sentence. It simply reads it. Meanwhile, page 88
drops to rank 5, because a setup guide never says which plans include SSO.

The Scholar reads the new top 5. It thinks like this: "The user is asking two things. One, is
SSO on the Pro plan? Page 212 says yes. Two, is there any condition? Page 1,140 says teams
under 50 seats also need the Security add-on." It answers:

> *"Yes, SSO is included on Pro, but teams under 50 seats also need the Security add-on."*

The answer is correct.

**Note:** The cross-encoder could rescue page 1,140 only because page 1,140 was among the 100
candidates. If the bi-encoder had put it at rank 500, no reranker could have helped. The first
stage decides what is *possible*. The second stage decides what the Scholar *sees*.

---

## How the two-stage pipeline works

**Phase 1: Preparing the library (once, ahead of time).**

**Step 1:** Split Acme's knowledge base into pages, or smaller chunks (Chapter 50).

**Step 2:** Run the bi-encoder on every page to get one vector per page.

**Step 3:** Store the vectors in the Card Catalog.

**Step 4 (optional):** Also build a keyword index, so we can run BM25 (Chapter 8).

**Phase 2: Answering a question (every time).**

**Step 1:** Embed the question with the same bi-encoder.

**Step 2:** Fetch the top 100 pages by vector score. Optionally fetch the top 100 by BM25 too,
and merge the two lists into one.

**Step 3:** Run the cross-encoder on each of the 100 (question, page) pairs.

**Step 4:** Sort by cross-encoder score and keep the top 10.

**Step 5:** Hand those 10 pages to the Scholar, which writes the answer.

As a diagram:

```
question
  │
  ├─► BM25         top 100 ─┐
  │                          ├─► merge (RRF) ─► top 100 ─► cross-encoder ─► top 10 ─► LLM
  └─► bi-encoder   top 100 ─┘
      (ANN index)
```

Means, two cheap searches cast a wide net, one expensive model sorts the catch, and the
Scholar reads only the best of it. RRF (reciprocal rank fusion) is a simple rule for merging
two ranked lists. Chapter 40 covers it.

There are three stages, and each one is tuned to its scale:

1. **Retrieval** (millions → 100s). Cheap. It needs high *recall*, meaning the right pages are
   somewhere in the list. Wrong pages mixed in are fine. A page left out here can never come
   back.
2. **Reranking** (100s → 10). Expensive. It needs high *precision*, meaning the pages at the
   top are the right ones. Now that the list is short, we can afford a careful reading.
3. **Generation** (10 → 1 answer). The Scholar (the LLM, Chapter 1) reads what survived.

Chapter 19 defines both. For now, recall asks "did we find it
at all?" and precision asks "is the top of the list clean?"

**The rule behind this design: each stage should be as cheap as it can be while keeping the
right answer in the list it passes on**. A retrieval stage that misses the answer cannot be
fixed later. A reranker that is slightly worse costs us some ordering, not the answer itself.
Put simply, spend on recall early and on precision late.

---

## Late interaction: the third option

There is a middle position, and it is the subject of Part V.

A bi-encoder squeezes a page into one vector *before* it sees the question. The two meet only
at the very end, through a single dot product. A cross-encoder lets them interact at every
layer of the model. *Late interaction*, which Chapter 35 defines properly, also waits until
the end, but it compares *many* vectors instead of one.

| | How question and page interact | Precomputable? | Stored per page |
|---|---|---|---|
| Bi-encoder | 1 dot product | Yes | 1 vector |
| Late interaction (ColBERT) | Every question token vs every page token | Yes | ~200 vectors (one per token) |
| Cross-encoder | Full attention at every layer | No | Nothing (1 forward pass per pair) |

ColBERT (Chapter 35) is the best-known late-interaction model. It keeps one small vector for
every token of the page. At question time, each question token finds its best-matching page
token, and those best matches are added up. Chapter 35 calls this MaxSim.

In simple words, the question's tokens for "single sign-on" can find the token "SSO" inside
page 1,140 directly. One pooled vector cannot do that. Yet all the page-side work still
happens in advance. So late interaction recovers a large share of cross-encoder quality while
remaining indexable.

The price is storage. One 768-d vector is 3,072 bytes. ColBERT stores one 128-number vector
per token, so a 200-token page costs `200 × 128 × 4 = 102,400` bytes. That is roughly **30×
the bytes** and **200× the number of vectors**, before any compression. Chapters 37 and 38 are
about paying that bill down.

---

## Bi-encoder vs cross-encoder

| | Bi-encoder | Cross-encoder |
|---|---|---|
| Input | One text | A (question, page) pair |
| Output | A vector | A single score |
| Page work done ahead of time? | Yes | No |
| Can live in a Card Catalog? | Yes | No |
| One question over 1M pages | ~10 ms | minutes |
| Accuracy | Good, but lossy | Much better |
| Job in the pipeline | Retrieval (stage 1) | Reranking (stage 2) |

**Advantages of a bi-encoder.**

- Page vectors are computed once, and with an index, search scales to billions of pages.

**Disadvantages of a bi-encoder.**

- Each page is squeezed into one vector without knowing the question, so details like page
  1,140's caveat get blurred.
- A question with several conditions gets averaged into a vague topic.

**Advantages of a cross-encoder.**

- It reads the question and the page together, so it can check each part of the question
  against the actual text.
- On top of a bi-encoder, it typically adds 5–15 points of nDCG@10.

**Disadvantages of a cross-encoder.**

- One forward pass per pair, and nothing to index, so it can never be the first stage.
- Its scores are raw numbers that are not comparable across questions unless we calibrate them.

---

## When to use which one

We must use a **bi-encoder** whenever we search the whole library. That means every first
stage, and every corpus bigger than a few hundred pages.

We must use a **cross-encoder** when we already have a short list and the order matters. That
means reranking the top 50–200 candidates before the Scholar reads them.

We must consider **late interaction** when one vector per page loses too much, a
cross-encoder is too slow, and we can afford the extra storage.

Most strong systems use both: a bi-encoder, often alongside BM25, to retrieve, and a
cross-encoder to rerank.

---

### Under the hood

```python
from sentence_transformers import SentenceTransformer, CrossEncoder

bi    = SentenceTransformer("BAAI/bge-small-en-v1.5")
cross = CrossEncoder("BAAI/bge-reranker-base")

query = "Does the Pro plan include single sign-on?"

# Stage 1: cheap, over all 40,000 pages (page vectors were computed at index time).
# (Chapter 14: this model's card also recommends an instruction in front of the query.)
q = bi.encode(query, normalize_embeddings=True)
candidates = index.search(q, k=100)          # index = our Card Catalog; page 1,140 is at rank 23

# Stage 2: expensive, over 100 (question, page) pairs
scores = cross.predict([(query, c.text) for c in candidates])
top10  = [candidates[i] for i in scores.argsort()[::-1][:10]]   # pages 212 and 1,140 now on top
```

Look at the shape of the inputs. The bi-encoder takes *one* text and returns a vector. The
cross-encoder takes a *pair* and returns a single number. That difference in signature is the
entire architectural difference, visible right in the API.

In simple words, if a model needs the question before it can process the page, the page can
never be processed in advance, and so it can never be indexed.

One practical detail. For this model, `CrossEncoder.predict` squashes each score through a
sigmoid (Chapter 9) into the range 0–1, so real output will not look like the 9.1 and 8.4 of our
example. Some models' configs switch the squashing off and return raw scores instead. The order
stays the same either way. The squashing does not make scores comparable across questions.

---

### What people get wrong

**"I'll use a cross-encoder for search."** Only over a shortlist. No index structure can serve
a cross-encoder over a whole corpus, because there is nothing to index.

**"Reranking 1,000 candidates is better than 100."** Usually not worth it. The gains flatten
quickly past ~100, while latency (the time one query takes) keeps growing in step with the
number of candidates. Measure the curve on your own data. The knee is typically between 50
and 200.

**"The reranker will fix bad retrieval."** It reorders what it is given. If the right page is
at rank 500 and you rerank the top 100, no reranker on earth will help. **Recall at stage one
is the ceiling for the entire pipeline.**

**"Cross-encoder scores are comparable across queries."** Underneath, they are raw, unbounded
scores (logits, Chapter 12), even when a library squashes them into 0–1. Use them to rank within
one query, not to set a threshold across queries, unless you calibrate them deliberately.

---

### Ninja notes

Cross-encoders shine brightest on exactly the queries a bi-encoder finds hardest:
multi-constraint questions. *"Can a 30-seat team on the Pro plan use SSO with Okta without
buying an add-on?"* has four constraints: the seat count, the plan, the identity provider and
the add-on. A single 768-dimensional vector cannot hold all four crisply. Pooling blends them
into a vague topical average of "SSO on Pro". A cross-encoder can check each constraint against
the page text on its own.

This suggests a genuinely useful routing strategy: **classify query complexity and spend
accordingly.** Simple lookups skip the reranker and save its 20–100 ms. Multi-constraint
queries get the reranker, or a larger candidate set, or an LLM reranker. Many production query
streams are dominated by simple queries, so adaptive routing often cuts average latency
substantially while *improving* quality on the hard tail. It is one of the better-value
optimisations available, and almost nobody does it.

---

### Key takeaways

- **Bi-encoder = encode the question and the page separately, then compare vectors.**
  Precomputable, indexable, fast, lossy.
- **Cross-encoder = read the question and the page together, and output one score.** Far more
  accurate, impossible to index.
- Retrieve about 100 candidates with the bi-encoder, then rerank with the cross-encoder. That is
  the standard design.
- Stage-one recall is the hard ceiling. A page the first stage misses can never be reranked
  into view.
- Late interaction (Part V) is the structural compromise: precomputable, but ~200 vectors per
  page, roughly 30× the bytes before compression.

### What's next

So far we have treated questions and pages as the same kind of text.
[Chapter 14](./14-asymmetry-and-instructions.md) explains why they are not, and why modern
models want us to tell them which is which.

Fast to fetch, careful to order: that two-stage shape is the skeleton that the rest of this book
builds on.
