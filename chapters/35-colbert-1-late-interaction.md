---
title: "ColBERT I: Late Interaction"
chapter: 35
part: "Part V — Beyond One Vector"
slug: colbert-1-late-interaction
readingTime: "13 min"
summary: "What is ColBERT? Keep one vector per token instead of one per document, and compare them at the last moment. The result recovers most of a cross-encoder's accuracy while staying fully precomputable."
tags: [colbert, late-interaction, multi-vector, maxsim, retrieval]
prev: 34-single-vector-bottleneck
next: 36-colbert-2-maxsim
---

# ColBERT I: Late Interaction

**The one-paragraph version.** **ColBERT** refuses to squash a page into one vector. It encodes
a page into one small 128-dimensional vector **per token**, and stores all of them. At query time,
it encodes the question the same way. Then, for each question token, it finds the best-matching
page token, and adds up those best matches. The page's vectors never depend on the question. So
everything can still be computed in advance and indexed, which a cross-encoder cannot do.

In this chapter, we will learn about late interaction, the idea behind ColBERT. We will see how
ColBERT encodes pages and questions, how its MaxSim score works, and how it finds the SSO caveat
that a single vector buried. We will also see where it sits between a bi-encoder and a
cross-encoder, and when to use which one.

We will cover the following:

- What is late interaction
- Why we need late interaction
- How ColBERT encodes pages and questions
- How MaxSim scores a page
- An example of late interaction
- Why it beats one vector
- Bi-encoder vs ColBERT vs cross-encoder
- When to use which one

---

## What is late interaction

Before jumping into ColBERT, we must know what BERT is. **BERT** is a well-known transformer
model (Chapter 10) released by Google in 2018. It reads a piece of text and produces one
contextual vector for every token.

**ColBERT = Contextualized Late Interaction over BERT.**

- **Contextualized:** each token's vector depends on the words around it (Chapter 10).
- **Late interaction:** the question and the page meet only at the very end, when we score.
- **over BERT:** the encoder underneath is BERT.

**Late interaction = encode the question and the page separately, keep one vector per token, and
let the two sets of vectors interact only at scoring time.**

In simple words, nobody summarises anything. Every token keeps its own vector, and the comparison
happens token by token at the end.

Think of it like three ways for two teams to decide how well they match:

- **The bi-encoder way.** Each team elects a spokesperson, who sums up the whole team in one
  statement. The two statements are compared. That is fast, but enormous detail is lost. The
  spokesperson is the single pooled vector (Chapter 11).
- **The cross-encoder way.** All twenty people sit in one room and talk for an hour. That gives
  maximum understanding, but it cannot be done for a thousand teams. The room is the cross-encoder
  (Chapter 13), reading question and page together.
- **The late-interaction way.** Every person on team A writes a card saying what they are looking
  for. Every person on team B writes a card saying what they offer, *before ever meeting team A*.
  Then we lay out the cards, find each A-person's single best match on team B, and add up the
  scores. Team A is the question. Team B is the page. The cards are the token vectors.

No spokesperson. No meeting. Every card was written independently, so all of team B's cards can be
written in advance. But the comparison still happens person by person, not summary by summary.

That third way is ColBERT.

---

## Why we need late interaction

Let's go back to Chapter 34. Acme's knowledge base has 40,000 pages, and the question is:

> *"Does the Pro plan include single sign-on?"*

Page 212 says "SSO is included on the Pro plan." Page 1,140 is the long Security add-on page. It
is mostly about audit logs, IP allow-lists and data retention, and it holds one crucial sentence:
"On Pro, SSO needs the Security add-on for teams under 50 seats."

Start with a bi-encoder. It squeezes page 1,140 into one vector. The twenty or so sentences about
audit logs, allow-lists and retention pull that vector away from SSO. In Chapter 34's measurement,
the page came 6th. The Librarian returned the top 3, without page 1,140. The Scholar answered "Yes,
SSO is included on Pro." The answer is wrong for every team under 50 seats.

Now, what about a cross-encoder? It reads the question and page 1,140 together, sees the caveat,
and scores the page highly. But it has to run BERT on every question-page pair, at query time.
Running 1,000 passages through BERT-base takes about 100–300 ms on a modern GPU. All 40,000 of
Acme's pages would take 4–12 seconds, for *every* question. At the 100 million chunks of Acme's
hosted product, it is hopeless.

So we need two things at once:

- **Precomputed page vectors**, like a bi-encoder, so search is fast.
- **Token-level detail**, like a cross-encoder, so a buried line still counts.

Late interaction gives us both.

---

## How ColBERT encodes pages and questions

**Phase 1: Encoding a page (at index time).**

- **Step 1:** Add a special `[D]` marker token at the start, so the model knows this text is a
  document.
- **Step 2:** Run the text through BERT and take the contextual 768-d vector for every token.
- **Step 3:** Shrink each vector to 128 dimensions with a learned **linear projection** (one
  learned matrix multiplication).
- **Step 4:** Normalize each vector to unit length (Chapter 5).
- **Step 5:** Throw away the vectors for punctuation tokens. They carry little meaning and cost
  storage.
- **Step 6:** Store every remaining vector.

```
"[D] SSO is included on the Pro plan."
    ↓ BERT
  one 768-d vector per token: [CLS] [D] ss ##o is included on the pro plan . [SEP]
    ↓ linear projection to 128-d, L2-normalized, punctuation dropped
  11 vectors × 128 dims  →  stored
```

In simple words, one sentence becomes about a dozen small vectors. Notice that BERT's tokenizer
splits "SSO" into `ss` and `##o` (the `##` means the piece continues the previous one). A 200-token
passage becomes about 200 vectors. This is the storage cost Chapter 34 warned about. The shrink to
128 dimensions is the first of several measures that make it bearable.

**Phase 2: Encoding a question (at query time).**

- **Step 1:** Add a special `[Q]` marker token at the start, so the model knows this text is a
  query. That is Chapter 14's asymmetry, built into the architecture.
- **Step 2:** Pad the question to a fixed length, typically 32 tokens, using `[MASK]` tokens.
- **Step 3:** Run it through BERT, project each vector to 128-d, and normalize, exactly as for
  pages.
- **Step 4:** Keep every vector, *including* the ones at the `[MASK]` positions.

That padding detail is stranger and more important than it looks. The `[MASK]` tokens are not
ignored. BERT gives them meaning from the rest of the question, so they become extra, learned
question vectors.

In effect, the model performs **query expansion**: adding related terms the user did not type. The
mask positions drift toward meanings related to the question but absent from it. For our question,
they can learn to match words like *"SSO"*, *"SAML"* or *"log in with company accounts"*. This is a
genuine, learned expansion mechanism, and it is part of why ColBERT is so strong when the question
and the page use different words.

**Note:** Both sides get a marker. Pages get `[D]` and questions get `[Q]`. Only questions get the
`[MASK]` padding, and only pages lose their punctuation vectors.

---

## How MaxSim scores a page

ColBERT scores a page with **MaxSim**, short for "maximum similarity".

$$ S(q, d) = \sum_{i=1}^{|q|} \max_{j=1}^{|d|} \, q_i \cdot d_j $$

Here $q_i$ is the $i$-th question token vector, $d_j$ is the $j$-th page token vector, and $|q|$
and $|d|$ are how many of each there are.

- **Step 1:** Take one question token vector.
- **Step 2:** Compute its dot product with every token vector on the page.
- **Step 3:** Keep only the largest one. That is this token's best match.
- **Step 4:** Repeat for every question token.
- **Step 5:** Add up all the best matches. That sum is the page's score.

In simple words, every question token goes looking for its best partner on the page, and the page
scores well if every token finds a good one.

Chapter 36 unpacks why this particular function, and not another.

---

## An example of late interaction

Let's score page 1,140 for our SSO question. To keep the numbers small, we use toy similarities
and only four question tokens: *pro*, *plan*, *single sign-on* and *include*. (Really,
"single sign-on" is several tokens, and there are 32 question vectors.)

Here is part of page 1,140's similarity grid. Each cell is a dot product between a question token
and a page token. The first four page tokens come from the caveat sentence. The last two come from
the rest of the page.

| Question token ↓ / page token → | Pro | SSO | needs | add-on | audit | retention | **best** |
|---|---|---|---|---|---|---|---|
| pro | **0.93** | 0.20 | 0.10 | 0.15 | 0.05 | 0.02 | 0.93 |
| plan | **0.64** | 0.15 | 0.12 | 0.30 | 0.08 | 0.10 | 0.64 |
| single sign-on | 0.25 | **0.86** | 0.20 | 0.30 | 0.12 | 0.04 | 0.86 |
| include | 0.12 | 0.18 | **0.57** | 0.35 | 0.08 | 0.06 | 0.57 |

The best matches add up to 0.93 + 0.64 + 0.86 + 0.57 = **3.00**.

Notice what the rest of the page did to this score. Nothing. The "audit" and "retention" tokens
are in the grid, but they never win a `max`, so they add nothing and take nothing away.

The model thinks like this: *"The question wants 'Pro', 'plan' and 'single sign-on'. This page has
'Pro' and 'SSO' side by side in one sentence, and 'Pro' covers 'plan' fairly well. The rest of the
page is irrelevant, and I do not care."*

Now the same four question tokens against the other pages, with toy best matches:

| Page | pro | plan | single sign-on | include | MaxSim |
|---|---|---|---|---|---|
| 212: SSO is included on the Pro plan | 0.95 | 0.95 | 0.85 | 0.90 | **3.65** |
| 1,140: Security add-on page, with the caveat | 0.93 | 0.64 | 0.86 | 0.57 | **3.00** |
| 87: Two-factor login is included on the Basic plan | 0.40 | 0.95 | 0.60 | 0.90 | 2.85 |
| How to configure SAML single sign-on | 0.20 | 0.25 | 0.97 | 0.30 | 1.72 |

Page 1,140 comes 2nd, behind page 212. The Librarian brings back both. The Scholar answers:
"SSO is included on Pro, but teams under 50 seats need the Security add-on." The answer is correct.

With one vector, the rest of the page dragged page 1,140 down to 6th. With late interaction, the
rest of the page could not touch the caveat.

---

## Why it beats one vector

Let's go back to Chapter 34's failures and watch them dissolve.

**The needle in the haystack.** The question token *"single sign-on"* finds the matching `SSO`
token in page 1,140's one relevant sentence. It does not matter that about twenty sentences on
audit logs and retention surround it. There is no averaging step to dilute it. Each token stands on
its own.

**Multi-constraint questions.** *"Does SSO on Pro need an add-on for a 30-seat team?"* has several
content tokens. Each one finds its own best match somewhere on the page. A page satisfying every
constraint collects a strong maximum for each one. A page satisfying fewer collects fewer. **The
score is naturally compositional**, which a single averaged vector can never be.

**Rare terms.** BERT's tokenizer turns `SSO-4012` into `ss`, `##o`, `-`, `401` and `##2`. The
near-identical `SSO-4021` becomes `ss`, `##o`, `-`, `402` and `##1`. Each word and number piece
keeps its own vector, so the pieces that differ can still find, or fail to find, their match.
Nothing is averaged into oblivion. ColBERT behaves partly like an exact-match system and partly
like a semantic one, which is the combination Chapter 7 said we wanted.

**Interpretability, and this is underrated.** We can see *which* question token matched *which*
page token. That is a genuine explanation: "page 1,140 was retrieved because 'single sign-on'
matched 'SSO' in the caveat sentence." Single-vector retrieval offers no such account. For
regulated or high-stakes applications, this alone can justify the architecture.

---

## Bi-encoder vs ColBERT vs cross-encoder

Before the table, one benchmark name. **MS MARCO** is the standard web-search benchmark from
Microsoft: about 8.8 million passages and real search-engine questions. Its usual score is MRR@10
on the "dev" question set (Chapter 19).

| | Bi-encoder | **ColBERT** | Cross-encoder |
|---|---|---|---|
| Page vectors | 1 | ~100–200 | none |
| Precomputable | Yes | **Yes** | No |
| Interaction | 1 scalar | $\lvert q\rvert \times \lvert d\rvert$ grid | full attention |
| Quality (MS MARCO dev MRR@10, approx.) | ~0.31–0.38 | **~0.36 (v1) – 0.40 (v2)** | ~0.37–0.41 |
| Latency over 10M docs | ~10 ms | ~50–200 ms | ~15–50 min |
| Storage | 1× | ~30× raw bytes (~200× vector count), ~2.3× compressed | 0 |

The quality figures are approximate and depend heavily on the model generation and training
recipe. Modern single-vector models have closed much of the gap. But the ordering is the durable
lesson.

Read the quality row against the latency row. ColBERT gets within a point or two of a cross-encoder
while staying searchable over millions of documents. That is the result that made late interaction
matter.

**Advantages of ColBERT:** token-level detail, precomputed and indexable page vectors, learned
query expansion, and explanations for free.

**Disadvantages of ColBERT:** far more vectors to store, more compute per scored page, and a more
complex index (Chapters 37 and 38).

---

## When to use which one

We must use a **bi-encoder** when storage and speed matter most, the pages are short and focused,
and hybrid search plus reranking already give good enough answers.

We must use **ColBERT** when precision genuinely matters and pages are long and cover many topics,
like the Security add-on page. It also fits when answers hinge on buried details or rare codes, or
when we need to explain why a page was retrieved.

We must use a **cross-encoder** only on a short candidate list, a few dozen to a few hundred
pages, never on the whole corpus.

Many strong systems combine them. A fast first stage (a bi-encoder with BM25, or ColBERT itself)
finds candidates, and a cross-encoder reranks the top few dozen.

---

### Under the hood

Here are both phases in PyTorch. `encode` is Steps 2–4 of both phases, and `maxsim` is the scoring
Steps. The `[Q]`/`[D]` markers, `[MASK]` padding and punctuation filtering happen in the tokenizer
wrapper and are left out to keep this short.

```python
import torch
from transformers import AutoModel, AutoTokenizer

class ColBERT(torch.nn.Module):
    def __init__(self, name="bert-base-uncased", dim=128):
        super().__init__()
        self.bert = AutoModel.from_pretrained(name)
        self.proj = torch.nn.Linear(self.bert.config.hidden_size, dim, bias=False)

    def encode(self, ids, mask):
        h = self.bert(ids, attention_mask=mask).last_hidden_state   # (B, T, 768)
        v = torch.nn.functional.normalize(self.proj(h), dim=-1)     # (B, T, 128)
        return v * mask.unsqueeze(-1)                               # zero the padding
        # (for questions, the wrapper marks [MASK] positions as real: mask = 1)

def maxsim(Q, D):
    """Q: (nq, 128) query token vectors. D: (nd, 128) document token vectors."""
    sim = Q @ D.T                    # (nq, nd): every query token vs every doc token
    return sim.max(dim=1).values.sum()    # best match per query token, summed

# The toy grid for page 1,140 from our example (rows: question tokens)
sim_1140 = torch.tensor([[0.93, 0.20, 0.10, 0.15, 0.05, 0.02],
                         [0.64, 0.15, 0.12, 0.30, 0.08, 0.10],
                         [0.25, 0.86, 0.20, 0.30, 0.12, 0.04],
                         [0.12, 0.18, 0.57, 0.35, 0.08, 0.06]])
print(sim_1140.max(dim=1).values.sum())   # tensor(3.0000)
```

Two lines of scoring. The `max` is where "find my best match" happens. The `sum` is where the
compositionality comes from.

Also note that `D` was computed at index time. Nothing in `maxsim` needs the page encoder at all.
In simple words, the expensive BERT work on pages happens once, and every question after that only
pays for dot products.

---

### What people get wrong

**"ColBERT is a reranker."** It can be used as one, but its point is that it is **indexable**. If we
use it only as a reranker over a bi-encoder's candidates, we inherit the bi-encoder's recall
ceiling and give up the main advantage.

**"The storage cost is prohibitive."** It was, in 2020. ColBERTv2's residual compression (Chapter
37) brings a 200-token passage to about 7 KB, roughly 2.3× a single-vector index, not 30×.

**"Late interaction means interaction happens late in the model."** It means interaction happens
late in the *pipeline*: after all encoding, at scoring time. The model itself never sees question
and page together.

**"I should always use ColBERT."** It costs more storage and more compute. Use it when precision
genuinely matters, when documents are long and heterogeneous, or when interpretability is required.
And use it when your documents are images, where Chapter 45 shows it is the natural fit rather than
the expensive one.

---

### Ninja notes

**The 128-dimensional projection is not a detail. It is what makes this possible.** BERT's native
768 dimensions per token would cost 3 KB per token, about 600 KB for a 200-token passage. The
learned down-projection to 128 dimensions cuts that by 6× at essentially no quality cost. Token
vectors do not need the capacity of a document-level summary. Each one only has to represent *one
token in context*, which is a much smaller job.

**The markers are cheap.** In the reference implementation, `[Q]` and `[D]` reuse spare, unused
slots in BERT's vocabulary, so no new embeddings have to be added to the model.

Recent work pushes the projection further, to 64 or even 32 dimensions per token with modest
quality loss, and combines it with binary quantization. The direction of travel is clear: **many
small vectors beat one large vector**, and each of the many can be extremely small. A token vector
at 32 dimensions with 2-bit quantization is 8 bytes. 200 of those is 1.6 KB, about *half* a single
768-d float32 vector.

That crossover point, where multi-vector becomes cheaper than single-vector in raw bytes, has
already been reached in research settings. When it becomes routine in production, the argument for
pooling largely disappears.

---

### Key takeaways

- **Late interaction = encode question and page separately, one vector per token, and compare them
  only at scoring time.** ColBERT = Contextualized Late Interaction over BERT.
- Pages get a `[D]` marker and lose punctuation vectors. Questions get a `[Q]` marker and `[MASK]`
  padding to 32 tokens.
- Each token vector is projected to ~128-d and normalized.
- Scoring is **MaxSim**: for each question token take its best-matching page token, then sum.
- Page vectors do not depend on the question, so everything stays precomputable and indexable. That
  is the crucial difference from a cross-encoder.
- Query `[MASK]` padding acts as learned query expansion.
- It fixes the needle, multi-constraint and rare-term failures of single vectors, and it explains
  its matches. Page 1,140's caveat is found no matter what else shares the page.
- Storage is ~30× the bytes and ~200× the vectors raw, about 2.3× compressed. Quality approaches a
  cross-encoder at a fraction of the cost.

### What's next

[Chapter 36](./36-colbert-2-maxsim.md) examines MaxSim itself: why *max* and not *mean*, what it is
really computing, and how ColBERT is trained.

We now understand what late interaction is, how ColBERT encodes and scores, why it finds details a
single vector buries, and where it fits between a bi-encoder and a cross-encoder.
