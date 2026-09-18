---
title: "The Single-Vector Bottleneck"
chapter: 34
part: "Part V — Beyond One Vector"
slug: single-vector-bottleneck
readingTime: "14 min"
summary: "768 numbers cannot hold everything a page contains. Here is precisely what gets lost, why part of it is a geometric limit rather than a model failure, and what the alternatives are."
tags: [multi-vector, limitations, geometry, colbert, motivation]
prev: 33-filtered-search
next: 35-colbert-1-late-interaction
---

# The Single-Vector Bottleneck

**The one-paragraph version.** Squeezing a whole page into one vector loses information, and
better models will not fully fix it. There are two reasons. First, the model gives most of the
vector's room to the page's main topic, so a single number, exception or name gets almost none.
Second, there is a hard geometric limit. A single dot product can only ever return a limited set
of page combinations, however good the model is. What survives is the main topic. What is lost is
the specific detail. Part V is about what we can do once we stop insisting on one vector per page.

In this chapter, we will learn what the single-vector bottleneck is, watch it hide half of our SSO
answer, and see the geometric reason it can never fully go away. We will also look at the four
ways to respond, and what "more vectors" really costs.

We will cover the following:

- What is the single-vector bottleneck
- Why one vector loses the details
- An example: the buried SSO caveat
- The geometric limit
- What this looks like in production
- The four responses
- What "more vectors" costs
- When to use which one

---

## What is the single-vector bottleneck

**Single-vector bottleneck = the information a page loses because all of it must squeeze through
one fixed-size vector before it can be searched.**

A bottleneck is the narrow neck of a bottle. However much is inside, everything must pass through
that neck. In our case, the neck is 768 numbers.

Think of it like writing a one-sentence summary of every book in a library, and then answering
questions using only those sentences:

- The books are the pages in the Great Library.
- Each one-sentence summary is a page's vector.
- The questions are the user's queries.
- The Librarian can only read the summaries, never the books.

For *"what is this book about?"*, the summaries work beautifully.

For *"which book mentions the Security add-on for small teams?"*, they are useless, unless that
happened to be the book's main subject. A passing but crucial mention cannot survive a
one-sentence summary.

In simple words, a single vector is a superb summary and a poor index of specifics.

---

## Why one vector loses the details

Let's be concrete. A 300-word page might contain:

- A main topic
- Three or four subtopics
- A dozen named things (plans, products, features)
- Several numbers
- Two or three relationships between those things
- Dates and version information
- A tone and register
- Perhaps one important exception or caveat

Call it thirty separately retrievable pieces of information. They must all be expressed through
768 numbers. The same 768 numbers must *also* place the page correctly relative to every other
page in the corpus.

The model does the only sensible thing. It shares out the vector's room according to what was
useful during training (Chapter 12). The main topic gets a lot of room. The single number in
paragraph nine gets almost none, because across the training data, passing numbers rarely decided
whether two texts were a match.

**Note:** This is *not* a shortage of raw bits. A 768-d float32 vector holds 768 × 32 = 24,576
bits. The raw text of a 300-word page is only about 1,800 bytes, roughly 14,400 bits. So the
vector could store the text itself with room to spare. The loss comes from what the vector must
*do*: put similar pages near each other so that one dot product ranks them. That job has limits of
its own, as we will see.

A 4,096-dimensional model loses less. It still loses.

---

## An example: the buried SSO caveat

Let's use our running question on Acme's 40,000-page knowledge base:

> *"Does the Pro plan include single sign-on?"*

Page 212 says "SSO is included on the Pro plan." The catch is on page 1,140, the Security add-on
page. It is a long page, about 260 words, and almost all of it is about three other things: audit
logs, IP allow-lists and data retention. Exactly one sentence, in the middle, mentions SSO: "On
Pro, SSO needs the Security add-on for teams under 50 seats."

The Librarian returns the top 3 pages. We ran this with a real 768-d embedding model
(`bge-base-en-v1.5`) against page 1,140 and seven other short Acme pages. The code is in "Under
the hood".

Let's first see what happens with one vector for the whole of page 1,140.

| Page | Score |
|---|---|
| 212: SSO is included on the Pro plan | 0.79 |
| 87: Two-factor login is included on the Basic plan | 0.66 |
| How to configure SAML single sign-on | 0.59 |
| Plans overview | 0.58 |
| Connect Okta to log in with company accounts | 0.54 |
| **1,140: Security add-on, one vector for the whole page** | **0.51** |

Page 1,140 comes 6th. The top 3 is page 212, a Basic-plan login page, and a setup guide. The
Scholar reads them and answers: "Yes, SSO is included on the Pro plan." The answer is wrong for
every team under 50 seats.

The page contains the caveat. The caveat matches the question well. But about twenty other
sentences share the same vector, and they pull it towards "audit logs, allow-lists and retention".

Now, let's give the caveat sentence its own vector, separate from the rest of the page.

| Page | Score |
|---|---|
| 212: SSO is included on the Pro plan | 0.79 |
| 87: Two-factor login is included on the Basic plan | 0.66 |
| **1,140: the caveat sentence, its own vector** | **0.63** |

The caveat climbs from 6th place to 3rd, inside the top 3. Page 87 still sits above it, but that
does no harm. The Librarian now brings back pages 212 and 1,140 together. The Scholar answers:
"SSO is on Pro, but teams under 50 seats need the Security add-on." The answer is correct.

**Nothing about the sentence changed except how much else shared its vector.** That is the
bottleneck, measured. Across the full 40,000-page Library, the same blending is what left page
1,140 at rank 23 in Chapter 13.

---

## The geometric limit

Now, the question is, if we had a perfect model, would one vector per page be enough for every
question?

It would not, and the reason is geometry, not bit counts.

A search returns the `k` pages with the highest dot product with the question. Let's see how few
combinations that can produce in a tiny Map Room.

**One dimension.** Each page is a single number, say 0.1, 0.4, 0.7 and 0.9. A question is also a
single number. If it is positive, bigger pages score higher. If it is negative, smaller pages
score higher. So the top 2 is always {0.7, 0.9} or {0.1, 0.4}. There are 6 possible pairs of
pages, and only 2 can ever come back.

**Two dimensions.** Put six pages as unit vectors, evenly around a circle. For any question, the
two closest pages are always next-door neighbours on the circle. So at most 6 of the 15 possible
pairs can ever be a top-2 answer. The same holds for *any* arrangement of six unit vectors in two
dimensions.

More dimensions allow many more combinations. But the pattern never disappears. For any fixed
number of dimensions, there are patterns of "these pages answer this question" that no
arrangement of vectors can reproduce.

In simple words, each dimension buys a limited number of ways to cut the Map Room, and every
answer set must be one of those cuts.

Weller and colleagues made this precise in 2025, in *On the Theoretical Limitations of
Embedding-Based Retrieval*. They showed that the top-k sets a d-dimensional embedding can return
are limited by d. Picture a table with one row per question and one column per page, marking which
pages are relevant. The smallest dimension that can reproduce that table is within one of its
**sign rank**. Sign rank is the smallest number of dimensions in which dot products can land on the
correct side of zero for every yes/no entry. Some tables have a sign rank that grows with their
size, so no fixed dimension serves them all.

They also tested the best possible case. They skipped language models and optimised the vectors
directly for the test questions. Their extrapolation says 768 dimensions stop being able to
represent every pair of documents at around 1.7 million documents. Real trained models are far
from that best case.

Then they built a dataset called **LIMIT** to test real models. It has 50,000 documents and 1,000
simple questions like *"Who likes Hawaiian pizza?"*, each with exactly two relevant documents.
State-of-the-art single-vector models struggled to reach even 20% recall@100. BM25 keyword search
came close to perfect. A multi-vector ColBERT-style model (one vector per token, Chapter 35) did
much better than the single-vector ones.

What does this mean for Acme? Our SSO question needs pages 212 and 1,140 *together*. "Which plans
need an add-on for SSO?" needs a different pair. "How long are audit logs kept?" needs another.
Every question asks for its own combination of pages. 768 dimensions leave plenty of room for any
single pair. The limit says that as the number of distinct combinations grows, some questions must
get the wrong set, no matter how well the model is trained.

---

## What this looks like in production

The failures are easy to recognise once we know the shape. Here they are, on Acme's knowledge
base.

**The needle-in-a-haystack failure.** A long page has one sentence that matters. That is the
buried SSO caveat above. The page's vector says "audit logs, allow-lists and retention", which is
not close enough to beat shorter pages about SSO.

**The multi-constraint failure.** *"Does SSO on Pro need an add-on for a 30-seat team?"* has
three constraints: the plan, the feature, and the team size. The single query vector blends all
three, and lands somewhere between them. Pages matching two of the three, like "SSO is included
on Pro", outrank the one page matching all three.

**The rare-term failure.** A user searches for error code `SSO-4012` ("SAML assertion expired").
The tokenizer chops it into pieces (Chapter 10). Even if it did not, one code among hundreds of
words is a tiny fraction of a mean-pooled vector (Chapter 11). The page for `SSO-4021` looks
nearly identical.

**The negation failure.** *"Which plans do not include single sign-on?"* The vector for this
question sits extremely close to the vector for plans that *do* include it, because the two share
almost every word. The word "not" barely moves it.

All four have the same root cause. All four are what Part V addresses.

---

## The four responses

**1. Chunk smaller** (Chapter 50). A **chunk** is a piece of a page that gets its own vector. If
page 1,140 is cut into short chunks of a few sentences, the SSO caveat gets a far bigger share of
its chunk's vector, as in our example. This is effective, cheap, and the first thing to try. Its
limit: chunks lose their surrounding context, and the vector count multiplies.

**2. Hybrid with sparse** (Chapter 40). **Hybrid search** runs keyword search (BM25, Chapter 8)
next to vector search and merges the results. BM25 never loses `SSO-4012`, because the code is
literally in its index. This covers the rare-term failure completely and the multi-constraint one
partly. It is also cheap, and should be the default.

**3. Rerank** (Chapter 41). A cross-encoder (Chapter 13) reads the question and a page together,
so it can check each constraint one by one. It solves multi-constraint questions and negation
well. But it **only reorders what retrieval already found.** If page 1,140 sits at rank 800, it
never gets a chance.

**4. Use more vectors** (Chapters 35–38). Stop compressing each page to one vector. Keep a vector
per token (a word or word piece), or per image patch (a small square cut from a page image,
Chapters 42 and 45). Now the question's `SSO-4012` tokens can find the matching tokens on the page
directly. And each of the three constraints can match its own part of the page.

Responses 1–3 are mitigations everyone should already use. Response 4 is a different architecture,
and it is the only one that attacks the bottleneck itself.

---

## What "more vectors" costs

Honest numbers, for a 200-token passage:

| Approach | Vectors/passage | Bytes |
|---|---|---|
| Single dense (768-d, float32) | 1 | 3,072 |
| Multi-vector, 128-d/token, float32 | ~200 | 102,400 |
| Multi-vector, compressed (2-bit residual + centroid ID, 36 B/token) | ~200 | ~7,200 |

Raw, that is **about 33× the bytes** (call it ~30×) and 200× the number of vectors. That is why
ColBERT was interesting in 2020 and impractical for most teams. Chapter 37's compression
(ColBERTv2's residual scheme, 36 bytes per token) brings it to about 7,200 bytes, roughly 2.3× a
single-vector index. That is why it is practical now.

Scoring cost rises too. Instead of one dot product per page, we compute a grid: every question
token against every page token. Chapters 37 and 38 are entirely about making that affordable, and
MUVERA (Chapter 38) is the most elegant answer.

---

## When to use which one

| Response | Cost | Fixes needle | Fixes multi-constraint | Fixes rare terms | Fixes geometric limit |
|---|---|---|---|---|---|
| Smaller chunks | low | yes | partly | no | no |
| Hybrid with BM25 | low | partly | partly | yes | partly |
| Reranking | medium | only if retrieved | yes | partly | only within the candidates |
| Multi-vector | high | yes | yes | yes | much better |

**Advantages of one vector per page:** tiny storage, one dot product per page, and every index in
Part IV works on it.

**Disadvantages of one vector per page:** buried details get lost, constraints get blended, rare
codes get averaged away, and some combinations of pages can never be returned together.

We must use **smaller chunks, hybrid search and reranking** first, on every system. They are
cheap and fix most failures.

We must use **multi-vector retrieval** when we have measured that those three are not enough, or
when the pages are images (Chapter 45), where the calculation is different.

Many strong systems use all four.

---

### Under the hood

We can watch the bottleneck happen in about a minute. This is the exact code behind our example.

- **Step 1:** Embed the question, the caveat sentence on its own, the whole Security add-on page,
  and seven other short Acme pages.
- **Step 2:** Score every text against the question with a dot product.
- **Step 3:** Rank the caveat alone, then the whole page, against the seven other pages.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-base-en-v1.5")        # 768-d
embed = lambda texts: model.encode(texts, normalize_embeddings=True)

question = "Does the Pro plan include single sign-on?"
caveat = "On Pro, SSO needs the Security add-on for teams under 50 seats."
other_lines = [
    "Audit logs record every admin action, permission change and data export.",
    "Each audit log entry shows who acted, what changed, when, and from which IP address.",
    "Audit logs can be filtered by user, action and date, and exported as CSV.",
    "Logs can also be streamed to Splunk, Datadog or any HTTPS endpoint.",
    "Audit log entries are kept for one year unless a retention rule says otherwise.",
    "IP allow-lists limit workspace access to the office and VPN addresses you choose.",
    "An allow-list accepts single addresses and CIDR ranges, up to 200 entries.",
    "Requests from addresses outside the allow-list are blocked and logged.",
    "Admins can test a new allow-list in report-only mode before enforcing it.",
    "Mobile apps and API tokens follow the same allow-list rules.",
    "Data retention rules delete tickets, attachments and chat transcripts after a set period.",
    "Retention periods range from 30 days to 7 years, and can differ by data type.",
    "Deleted data is purged from backups within 35 days.",
    "Legal holds pause deletion for selected users during an investigation.",
    "A retention report lists what will be deleted in the next 30 days.",
    "Encryption keys can be rotated on demand from the Security tab.",
    "Admins receive an email whenever a retention rule or allow-list changes.",
    "The add-on is billed per seat on the same invoice as your plan.",
    "It can be turned on or off at any time from the Billing tab.",
    "Turning it off keeps existing audit logs but stops recording new ones.",
]
# Page 1,140: the Security add-on page, with the SSO caveat buried in the middle
page_1140 = " ".join(["Security add-on."] + other_lines[:10]
                     + [caveat] + other_lines[10:])
rivals = [                                                     # seven other pages
    "SSO is included on the Pro plan.",                        # page 212
    "Two-factor login is included on the Basic plan.",         # page 87
    "To configure SAML single sign-on, upload your identity provider's metadata file.",
    "Acme offers three plans: Basic, Pro and Enterprise. Compare features and prices.",
    "Connect Okta to Acme to let employees log in with their company accounts.",
    "Admins can require strong passwords and set session timeouts in Security settings.",
    "The Enterprise plan includes SCIM provisioning, audit logs and a dedicated manager.",
]
q = embed(question)
rival_scores = embed(rivals) @ q

for label, text in [("caveat on its own   ", caveat), ("whole page, 1 vector", page_1140)]:
    score = float(embed(text) @ q)
    rank = 1 + int((rival_scores > score).sum())
    print(f"{label}: score={score:.2f}  rank={rank} of 8")
# caveat on its own   : score=0.63  rank=3 of 8
# whole page, 1 vector: score=0.51  rank=6 of 8
```

Same sentence, same model, same question. Sharing a vector with about twenty sentences on audit
logs, allow-lists and retention dropped the caveat from 3rd place to 6th, out of the top 3.

Exact scores vary a little between library versions, and other models give different numbers.
On the smaller `bge-small-en-v1.5`, this page did not show the drop. The sentence on its own
scored 0.54 and the whole page 0.56, and both missed the top 3. That model matched the
abbreviation "SSO" weakly: spelled out as "single sign-on", the lone sentence scored 0.71. So
dilution is a strong tendency, not a law. Measure it on your own data and your own model.

That result is also the strongest argument for small chunks anywhere in this book.

The geometric limit is just as easy to check. Here are six pages as 2-d unit vectors, and 100,000
random questions:

```python
import numpy as np

rng = np.random.default_rng(0)
angles = np.arange(6) * np.pi / 3                        # six pages, evenly round a circle
pages = np.c_[np.cos(angles), np.sin(angles)]            # 2-d unit vectors
questions = rng.normal(size=(100_000, 2))                # 100,000 random questions
top2 = np.argsort(-(questions @ pages.T), axis=1)[:, :2]  # each question's top-2 pages
pairs = {tuple(sorted(p)) for p in top2.tolist()}
print(len(pairs), "of 15 pairs can ever be returned")    # 6 of 15
```

In simple words, 100,000 different questions, and only 6 different answers.

---

### What people get wrong

**"A bigger embedding model will fix this."** It may lose somewhat less. The mechanism does not
change, and the geometric limit only moves further out. Run the experiment above on a larger model
and see for yourself.

**"768 floats is simply too few bits."** A 768-d float32 vector has 24,576 bits, more than the raw
text of a 300-word page. The limit is about what dot products can rank, not about storage.

**"More context in the prompt will fix this."** The page was never retrieved. The Scholar cannot
use what it never saw.

**"Multi-vector is only for research."** ColBERT variants run in production today, and for
document images (Chapter 45) they are the *standard* approach, not the exotic one.

**"Just use multi-vector for everything."** It costs more storage, more compute, and more
operational complexity. Chunking plus hybrid plus reranking solves a large fraction of cases at a
fraction of the cost. Reach for multi-vector when you have measured that those are not enough, or
when your documents are images, where the calculation is different.

---

### Ninja notes

There is a formal way to see why late interaction (Chapter 35) recovers so much.

A single-vector bi-encoder computes a rank-1 interaction: one query vector, one document vector,
one scalar. Whatever the model wants to express about the relationship must pass through that
single number.

A cross-encoder (Chapter 13) computes full attention between all query and document tokens at
every layer. That is an extremely high-capacity interaction, and it cannot be indexed.

Late interaction sits between. It builds a $|q| \times |d|$ matrix of token-level similarities,
reduced by MaxSim (each query token keeps only its best match, then the matches are added). The
interaction is now $O(|q| \cdot |d|)$ scalars rather than one, so it can
express *which query term matched where*. That is a far richer channel than rank-1. It is still
decomposable: each document's vectors are computed independently, so they can be precomputed and
indexed.

**Capacity of interaction, not capacity of representation, is the axis that matters.** Once you
see retrieval architectures this way, the whole design space (bi-encoder, late interaction,
cross-encoder) becomes one dimension with three points on it.

The sign-rank result makes the same point from the other side. Weller et al. bound the minimum
embedding dimension for a binary relevance matrix $A$ within one of the sign rank of $2A - 1$. No
training trick escapes a bound on the representation itself. Escaping it needs far more dimensions
or a richer interaction. BM25's sparse vectors have one dimension per vocabulary term, which is why
it nearly aces LIMIT. The multi-vector model has the richer interaction, which is why it does much
better there too.

---

### Key takeaways

- **Single-vector bottleneck = the information a page loses because all of it must squeeze
  through one fixed-size vector.**
- What survives is the main topic. What is lost is specifics: numbers, names, exceptions, rare
  codes, and the combination of several constraints.
- It is not about bit counts. It comes from training priorities, and from a geometric limit on
  which top-k sets a d-dimensional dot product can return (sign rank, Weller et al. 2025, LIMIT).
- Burying Acme's SSO caveat in the long Security add-on page dropped it from 3rd to 6th place
  with `bge-base-en-v1.5`, out of the top 3. Measure this on your own data and model.
- Four responses: smaller chunks, hybrid with sparse, reranking, and more vectors.
- The first three are cheap mitigations everyone should use. The fourth is a different
  architecture.
- Raw multi-vector storage is ~30× the bytes and ~200× the vectors. Compressed, it is ~7,200 bytes
  per passage, about 2.3× a single vector.
- The real axis is *interaction capacity*: rank-1 (bi-encoder), token grid (late interaction), full
  attention (cross-encoder).

### What's next

[Chapter 35](./35-colbert-1-late-interaction.md) introduces ColBERT, which keeps every token's
vector and compares them at the last possible moment.

We now understand what the single-vector bottleneck is, why part of it is a hard geometric limit
and not a weak model, and which of the four responses to reach for first.
