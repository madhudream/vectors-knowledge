---
title: "From Tokens to One Vector: Pooling"
chapter: 11
part: "Part II — Where Embeddings Come From"
slug: pooling
readingTime: "12 min"
summary: "A transformer hands you 512 vectors. You need one. The way you collapse them is a real design decision — and the road not taken here becomes ColBERT."
tags: [pooling, cls, mean-pooling, embeddings]
prev: 10-transformers-contextual
next: 12-contrastive-learning
---

# From Tokens to One Vector: Pooling

**The one-paragraph version.** After a transformer runs, we have one vector per token. Storing
them all is expensive, so nearly every embedding model collapses them into one vector by
**pooling**. Usually it takes the mean, sometimes a dedicated `[CLS]` token, sometimes the last
token. The choice must match how the model was trained, and getting it wrong silently degrades
quality. Refusing to pool at all is the road that leads to Part V.

In this chapter, we will learn how a model turns hundreds of token vectors into the single vector
we store. We will see the three standard ways to do it, the padding bug that quietly breaks two of
them, and what pooling throws away. That last point explains how the Librarian can miss half of
the answer to our running question.

We will cover the following:

- What is pooling
- Why we need pooling
- How pooling works: four ways to summarise a meeting
- Mean, CLS and last-token pooling
- Padding and the attention mask
- An example: the sentence that disappears
- Less common pooling methods
- Mean vs CLS vs last-token
- When to use which one

---

## What is pooling

**Pooling = collapsing a transformer's many token vectors into one vector for the whole text.**

In simple words, the model writes one note card per token, and pooling files a single summary
card.

---

## Why we need pooling

Chapter 10 ended with a problem. We wanted one vector for a passage, and the transformer gave us one
vector per token.

Now, the question is, why not just keep them all?

Let's do the arithmetic for Acme's knowledge base. A page of 512 tokens gives 512 vectors of 768
numbers each. At 4 bytes per number (Chapter 1), one vector is 3,072 bytes, so the page takes
512 × 3,072 bytes, about 1.5 MB. Across all 40,000 pages, that is about 63 GB.

One pooled vector per page is 3,072 bytes. All 40,000 pages then fit in about 123 MB. That is 512
times less.

Search gets simpler too. One dot product per page (Chapter 4) instead of hundreds.

**Note:** Keeping every token's vector is not crazy. It is exactly what ColBERT does in Part V. But
ColBERT shrinks each token vector to 128 numbers and compresses it hard, and Chapter 37 is devoted
to making that affordable.

---

## How pooling works: four ways to summarise a meeting

Think of it like the meeting from Chapter 10. Everyone around the table has finished rewriting
their note card, one card per token. We must file *one* summary.

**Option A: appoint a note-taker.** One person's job, from the start, was to listen to everything
and write the summary. We file their card. (This is `[CLS]` pooling.)

**Option B: average all the cards.** Every voice contributes equally. (Mean pooling.)

**Option C: file the last person's card.** This is a different meeting from Chapter 10's: here
people speak one at a time and hear only those before them. That is how GPT-style models work.
Since everyone spoke in order and the last speaker heard everything, their card is the most
informed. (Last-token pooling.)

**Option D: file every card.** Nothing is lost. The filing cabinet is now hundreds of times larger.
(No pooling at all: ColBERT, Chapter 35.)

Now the mapping:

- The note-taker is a special token added to the start of every input.
- The average is the position-by-position mean of all the token vectors.
- The last speaker is the final token, in a model with causal attention (Chapter 10).
- The filing cabinet is our vector storage.

Each option is defensible. Each is used in production today. The dangerous move is not picking the
wrong one. It is *not knowing which one our model expects*.

---

## Mean, CLS and last-token pooling

**Mean pooling** averages all the token vectors, ignoring padding:

$$ v = \frac{\sum_{i} m_i \, h_i}{\sum_{i} m_i} $$

Here $h_i$ is token $i$'s final hidden state, meaning the vector the last layer wrote for token $i$.
$m_i$ is its attention mask: 1 for a real token and 0 for padding. The next section defines both
properly. Means, add up the real tokens' vectors, then divide by how many real tokens there are.

Mean pooling is the most common choice and the strongest default. It is robust, it uses all the
information, and it degrades gracefully. Its weakness is dilution. In a long passage, one crucial
sentence is averaged against forty unremarkable ones, and its signal fades. This is a major
argument for smaller chunks.

**CLS pooling** takes the vector at a special token called **`[CLS]`**, added to the front of every
input. (CLS is short for "classification", its original job.) Nothing about `[CLS]` is inherently
special. It is just a token whose vector the model was *trained* to make a whole-text summary. In
raw BERT it is trained for next-sentence prediction (guessing whether one sentence really followed
another), which is why raw BERT's `[CLS]` is a poor similarity vector.

In models fine-tuned for retrieval with a `[CLS]` objective, such as BGE v1.5 and BGE-M3, it is
excellent. (Fine-tuned means given extra training for one job. Chapter 12 shows how.) E5, by
contrast, is mean-pooled, and e5-mistral is last-token pooled. This is exactly why you must read
the **model card**: the documentation page a model's authors publish alongside its weights, saying
how the model was trained and how to use it, including its pooling.

**Last-token pooling** takes the final real token's vector. This is standard for decoder-only
embedding models, such as e5-mistral and several recent top-of-leaderboard models. Causal attention
(Chapter 10) means only the last position has seen everything.

> **The rule that prevents the bug:** use the pooling the model was trained with. It is in the
> model card. In `sentence-transformers` (the most widely used Python library for running embedding
> models), it is in the `Pooling` module config. Mean-pooling a CLS-trained model does not throw an
> error. It quietly costs several points of retrieval quality, and you will spend a week blaming
> your chunking.

---

## Padding and the attention mask

Mean pooling and last-token pooling share one trap. To see it, we need two more terms.

Models embed texts in groups, called batches, because that is much faster than one at a time. Every
text in a batch must have the same number of tokens. So shorter texts get filler added, usually at the end.

**Padding = the filler tokens added so every text in a batch has the same length.**

**Attention mask = a list of 1s and 0s, one per token, where 1 marks a real token and 0 marks
padding.**

Let's take a tiny example. We embed two Acme snippets in one batch. The longer one has 4 tokens. The
shorter one, *"SSO on Pro"*, has 3 real tokens (pretend each word is one token), so it gets one
padding token. Its token vectors are toy 2-d vectors:

| Position | 1 | 2 | 3 | 4 |
|---|---|---|---|---|
| Token | SSO | on | Pro | [PAD] |
| Attention mask | 1 | 1 | 1 | 0 |
| Token vector (toy) | [1, 0] | [0, 1] | [1, 1] | [−2, −2] |

Without the mask, we average all four rows and get **[0, 0]**.

With the mask, we average only the three real rows and get **[0.67, 0.67]**.

In simple words, the padding row is junk, and averaging junk drags the vector somewhere meaningless.
The mask tells mean pooling to skip it.

Worse, the amount of padding depends on the *longest* text in the batch. Put *"SSO on Pro"* next to
a 300-token page, and it gets 297 padding rows. So an unmasked vector changes depending on what else
happened to be in the batch.

---

## An example: the sentence that disappears

Back to our running question from Chapter 1: *"Does the Pro plan include single sign-on?"* The full
answer needs two pages. Page 212 says SSO is included on Pro. Page 1,140 adds the catch.

Page 1,140 is a long page about the Security add-on, about 260 words. Almost all of it covers
audit logs, data retention and IP allow-lists. It mentions SSO in exactly one sentence:

> *"On Pro, SSO needs the Security add-on for teams under 50 seats."*

Let's first see what happens when the whole page is embedded as one mean-pooled vector.

That sentence is 12 words. Mean-pooled into 768 numbers alongside about 250 other words about
audit logs and retention, it contributes under 5% of the final vector. The page's vector says
"security features". The query's vector says "single sign-on on the Pro plan". They are not close.

The Librarian fetches page 212, whose whole vector is about SSO on Pro, and misses page 1,140. The
Scholar writes: *"Yes, SSO is included on the Pro plan."* The answer is wrong for every Pro team
under 50 seats. A human reading page 1,140 would have found the catch instantly.

Now, the question is, how do we get that sentence back? Three responses exist. All three are
legitimate, but only the first and the third rescue this particular page:

1. **Chunk smaller** (Chapter 50), so the SSO sentence dominates its own vector.
2. **Hybrid search** (Chapter 40). Keyword matching with BM25 (Chapter 8) scores each matching
   word on its own, so a single mention is not averaged away the way it is in a pooled vector.
   That helps when the page repeats the question's exact words, such as an error code. Here the
   question says "single sign-on" and the page says "SSO", so BM25 cannot connect them without a
   synonym list.
3. **Do not pool** (Chapter 35). Keep a vector per token, and let the question's *single sign-on*
   tokens find the page's *SSO* token directly, wherever it sits on the page.

Let's run the first one. We cut page 1,140 into chunks of about 100 words each. In its chunk, the
12-word SSO sentence is now about an eighth of the vector instead of under 5%. That chunk's vector lands
near the query.

The Librarian fetches page 212 *and* the SSO chunk of page 1,140. The Scholar reads both and thinks:
*"Page 212 says SSO is on Pro. The other passage says Pro teams under 50 seats need an add-on. So
the honest answer is yes, with a condition."* It writes: *"SSO is included on Pro, but teams under
50 seats need the Security add-on."* The answer is correct.

ColBERT is, quite literally, the answer to "what if we skipped this chapter?"

---

## Less common pooling methods

**Max pooling** takes the element-wise maximum across tokens. That is, for each of the 768
positions, it keeps the largest value any token has there. It is rarely used for text because it
discards too much. But it is the conceptual cousin of ColBERT's MaxSim (Chapter 35).

**Weighted mean** weights later tokens more heavily. Some decoder-based models use it as a softened
last-token pooling.

**Attention pooling** learns a small attention layer over the token vectors, letting the model
decide what to keep. It brings more capacity and more parameters, and it must be trained together
with the rest of the model.

**Multi-vector (no pooling)** keeps everything. It is not a pooling strategy so much as a refusal,
and it is the most important entry in this list, because Part V is built on it.

---

## Mean vs CLS vs last-token

| | Mean | CLS | Last-token |
|---|---|---|---|
| Takes | The average of the real tokens | The `[CLS]` token's vector | The last real token's vector |
| Example models | E5, all-MiniLM, all-mpnet | BGE v1.5, BGE-M3 | e5-mistral and other decoder-only embedders |
| Needs the attention mask | Yes, to skip padding | No (`[CLS]` sits first in BERT-style inputs) | Yes, to find the last real token |
| Main weakness | Dilution in long texts | Poor unless trained for it | Breaks if the padding side is wrong |

---

## When to use which one

We must use **the pooling the model was trained with**. That is the whole rule. We read it from the
model card or the `sentence-transformers` config, never from habit.

We must use **mean pooling** when we train our own model from a BERT-style encoder and have no
reason to choose otherwise.

We must use **no pooling** (multi-vector, Part V) when single sentences inside long texts must stay
findable and we can afford the storage.

Some strong systems do both: a pooled vector finds a shortlist fast, and per-token vectors re-rank
that shortlist (Chapter 37).

---

### Under the hood

Mean pooling with a correct attention mask, the version people get wrong:

```python
import torch

def mean_pool(last_hidden, attention_mask):
    mask = attention_mask.unsqueeze(-1).float()        # (B, T, 1)
    summed = (last_hidden * mask).sum(dim=1)           # (B, D)
    counts = mask.sum(dim=1).clamp(min=1e-9)           # (B, 1)
    return summed / counts

def cls_pool(last_hidden):
    return last_hidden[:, 0]                           # first token

def last_pool(last_hidden, attention_mask):            # right-padding; for left padding use last_hidden[:, -1]
    idx = attention_mask.sum(dim=1) - 1
    return last_hidden[torch.arange(last_hidden.size(0)), idx]
```

In simple words, `mask` turns the 1s and 0s into multipliers, so padding rows become zero before
the sum. `counts` is how many real tokens each text has, and the `clamp` only prevents a division by
zero. `last_pool` finds the last real token by counting the 1s.

That counting only works when padding sits on the *right*. Decoder-only models are often run with
padding on the *left*, so every text ends at the final position. Take the mask `[0, 0, 1, 1, 1]`.
Counting gives index 2, which is the *first* real token, not the last. With left padding the answer
is simply `last_hidden[:, -1]`. Check which side your tokenizer pads before trusting either line.

The classic bug is omitting the mask and averaging over padding tokens. Short texts in a batch get
their vectors dragged toward the padding representation, so **your embeddings change depending on
what else was in the batch**. It is maddening to debug, because embedding one text at a time works
perfectly. If you use `sentence-transformers`, this is handled. If you call Hugging Face's
`transformers` library (the lower-level library that runs the model itself) directly, it is yours
to get right.

---

### What people get wrong

**"Pooling barely matters."** Mismatched pooling can cost 3–8 points of nDCG@10 (a
ranking-quality score from 0 to 1, usually quoted as points out of 100, Chapter 19). That is the
difference between a good system and a mediocre one.

**"I'll average the chunk embeddings to get a document embedding."** An average of averages is very
blurry. It produces a vector near the centroid (the average point) of your entire corpus, which
retrieves for everything and means nothing. If you need a document vector, either embed a summary,
or store chunk vectors and aggregate at *score* time. The max of the chunk scores is a far better
document score than the mean of the chunk vectors.

**"Padding tokens are masked automatically."** In raw `transformers`, they are masked inside
attention, but they are still present in `last_hidden_state`. You must exclude them yourself when
pooling.

---

### Ninja notes

**Late chunking = embed the whole document once, then pool token vectors into per-chunk vectors.**
It inverts the usual order, and it is worth knowing about now that you understand pooling. Normally
we split the document, then embed each chunk independently, so each chunk's vectors are
contextualised only by that chunk. Late chunking instead runs the full document through a
long-context encoder *once*. Every token is contextualised by the entire document. Only *then* does
it pool the token vectors into per-chunk embeddings, by slicing the token range.

The payoff shows up on a chunk that reads *"It needs the Security add-on for teams under 50
seats"*. Its vector encodes what "it" is (SSO on Pro), because the pronoun was resolved during the
forward pass. It is pure pooling arithmetic, with no extra model, and it brings real gains on
documents with heavy coreference (pronouns and short names pointing back to earlier text). That
describes most reports, contracts and papers. The requirements are a genuine long-context encoder
and a framework that gives you token-level output.

Not every decoder-based embedder pools the last token. LLM2Vec, for example, switches on
bidirectional attention and mean-pools by default. The model card, again, is the only authority.

---

### Key takeaways

- **Pooling = collapsing a transformer's token vectors into one vector for the whole text.**
- Mean, `[CLS]`, and last-token are the three standard choices.
- **Always use the pooling the model was trained with.** Mismatches fail silently. BGE v1.5 and
  BGE-M3 use CLS, E5 uses mean, e5-mistral uses last-token.
- Pooling dilutes, and mean pooling does it most visibly: one important sentence in a long passage
  nearly disappears. That is how page 1,140's SSO catch goes missing.
- Mask your padding, or batch composition will change your vectors. Last-token pooling must also
  know which side the padding is on.
- Not pooling at all is ColBERT, and it exists precisely because pooling loses things.

### What's next

We keep saying models are "trained for retrieval". [Chapter 12](./12-contrastive-learning.md) opens
the training loop and shows the objective, which is word2vec's negative sampling, grown up.

Pooling looks like a detail, but it decides what a single vector can remember, and the model card
decides which kind we use.
