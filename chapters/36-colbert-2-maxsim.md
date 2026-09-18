---
title: "ColBERT II: MaxSim and Training"
chapter: 36
part: "Part V — Beyond One Vector"
slug: colbert-2-maxsim
readingTime: "15 min"
summary: "Why max and not mean, what the scoring function is really computing, why it is asymmetric, and how ColBERT is trained. The details that decide whether it works."
tags: [colbert, maxsim, chamfer, training, distillation, scoring]
prev: 35-colbert-1-late-interaction
next: 37-colbert-3-serving
---

# ColBERT II: MaxSim and Training

**The one-paragraph version.** MaxSim is the scoring rule inside ColBERT. For each query
token, it finds the best-matching token anywhere in the document, and then it adds up those
best matches. Taking the **max** over the document, and not the mean, means a long page is
never punished for talking about other things. Adding up over the query means every part of
the question has to find its own evidence. The rule is deliberately **asymmetric**: it never
checks whether every document token was used. Training matters as much as the rule. ColBERTv2
learns from a cross-encoder teacher's graded scores, and that cleaner supervision is worth as
much as the architecture.

In this chapter, we will learn how MaxSim turns a grid of token similarities into one score, and
why its two choices (max, then sum) are the right ones. We will also see its older mathematical
name, how ColBERT is trained, and how to read a MaxSim table.

We will cover the following:

- What is MaxSim
- A tiny MaxSim example, by hand
- Why max over the document, not mean or sum
- Why sum over the query, not max
- MaxSim is Chamfer similarity
- How ColBERT is trained
- Reading a MaxSim table
- Advantages and disadvantages of MaxSim

---

## What is MaxSim

In Chapter 35 we met ColBERT. It keeps one small vector, 128 numbers long, for every token of
the query and for every token of every document. Now we need a rule that turns two piles of
token vectors into one relevance score.

**MaxSim = for each query token, its best match anywhere in the document, added up over all
query tokens.**

Written as a formula:

$$ S(q,d) = \sum_{i=1}^{|q|} \, \max_{1 \le j \le |d|} \, q_i \cdot d_j $$

Let's break it down, one piece per line:

- $q_i$ is the vector of the $i$-th query token. $d_j$ is the vector of the $j$-th document token.
- $q_i \cdot d_j$ is their dot product (Chapter 4). ColBERT normalises every token vector, so
  this is a cosine similarity between −1 and 1.
- $\max_j$ keeps only the single best document token for query token $i$.
- $\sum_i$ adds those best scores up over all the query tokens.

In simple words, every word of the question goes looking for its best partner on the page, and
the page's score is how well all the words did in total.

**Note:** The query tokens include the `[MASK]` padding tokens from Chapter 35. They are real
query vectors and take part in the sum like any other.

---

## A tiny MaxSim example, by hand

Let's score one page of the Great Library (Acme's 40,000-page support knowledge base) against
our running question, *"Does the Pro plan include single sign-on?"* Page 212 says "SSO is
included on the Pro plan."

A real query has 32 token vectors and a real page has a couple of hundred. To keep the
arithmetic small, we keep just two query tokens, `sign-on` and `pro`, and three page tokens,
`sso`, `included` and `pro`. ColBERT lowercases everything, so we do too. (A real tokenizer
also splits some words into pieces, such as `ss` + `##o`, but whole words keep the arithmetic
readable.) The similarities are toy numbers, chosen to look like what a trained model produces.

**Step 1:** Compute the similarity of every query token with every page token. That gives a
grid with 2 rows and 3 columns.

| Query token ↓ · page 212 token → | `sso` | `included` | `pro` |
|---|---|---|---|
| `sign-on` | **0.90** | 0.20 | 0.10 |
| `pro` | 0.15 | 0.25 | **0.85** |

**Step 2:** In each row, keep the largest number. `sign-on` keeps 0.90 (it found `sso`). `pro`
keeps 0.85 (it found `pro`).

**Step 3:** Add up the row winners. 0.90 + 0.85 = **1.75**.

That is it. Page 212 scores 1.75.

Look at Step 2 again. The query said "sign-on" and the page said "SSO", yet they matched at
0.90, because contextual token vectors carry meaning (Chapter 10). And `included` never won a
row, but nothing penalised the page for it.

Now let's score a rival, page 2,301, about SSO on the Enterprise plan, with the tokens `sso`,
`enterprise` and `saml`:

| Query token ↓ · Enterprise page token → | `sso` | `enterprise` | `saml` |
|---|---|---|---|
| `sign-on` | **0.95** | 0.10 | 0.60 |
| `pro` | 0.10 | **0.30** | 0.05 |

The row winners are 0.95 and 0.30, so MaxSim = **1.25**. Page 212 (1.75) beats the Enterprise
page (1.25). That is the right order for a question about the Pro plan.

We will reuse these two tables to see why the rule has exactly this shape.

---

## Why max over the document, not mean or sum

Now, the question is, why keep the *max* of each row? We could take the mean of the row, or its
sum. Let's try both on a harder case and watch them fail.

Remember the catch in our running question. Page 1,140 says "On Pro, SSO needs the Security
add-on for teams under 50 seats." That sentence sits inside a long page about the Security
add-on. In this toy, page 1,140 is 400 tokens long. One token matches `sign-on` at 0.90. One
token matches `pro` at 0.90. The other 399 tokens in each row score 0.10.

Its rival is a short "Plans overview" page. It has 40 tokens, and every one of them is mildly
related to both query tokens, at 0.30. It never mentions the add-on.

**First, the mean of each row.** Page 1,140 gets (0.90 + 399 × 0.10) / 400 = 0.102 per row, so
0.204 in total. The overview page gets 0.30 per row, so 0.60 in total. The mean ranks the
overview page first. The answer is wrong. The one sentence that holds the catch is drowned by
399 unrelated tokens. This is the dilution that sank single vectors in Chapter 34, the very
thing we stopped pooling to escape.

**Second, the sum of each row.** Page 1,140 scores 2 × (0.90 + 39.9) = 81.6. The overview page
scores 2 × 12 = 24. That looks fine until a third, unrelated page arrives: a 2,000-token page of
release notes that never mentions SSO, with every token at 0.10. It scores 2 × 200 = 400 and beats
both. The answer is wrong again, this time because of length alone. This is BM25's problem before
length normalisation (Chapter 8).

**Now, the max of each row.** Page 1,140 scores 0.90 + 0.90 = 1.80. The overview page scores
0.30 + 0.30 = 0.60. The release notes score 0.10 + 0.10 = 0.20. Page 1,140 comes first. The
answer is correct.

In simple words, the max asks *"is the evidence for this word anywhere on the page?"* One
strong match is enough. The other 399 tokens contribute nothing, good or bad.

> **Max over the document is evidence detection.** It asks whether support exists, not how
> much of the document is support.

---

## Why sum over the query, not max

The outer step is a **sum**, and that choice is just as deliberate.

Let's go back to our two small tables. Suppose we took the max over the query as well, so a
page's score is its single best row.

- Page 212: its best row is 0.90.
- The Enterprise page: its best row is 0.95.

The Enterprise page wins. The answer is wrong. It matched `sign-on` very well and `pro` hardly
at all, and an outer max never noticed that `pro` was missing.

With the sum, page 212 scores 1.75 and the Enterprise page scores 1.25. Page 212 wins. The
answer is correct.

A question with four content words collects four row winners. A page that supports all four
scores about four units. A page that supports three scores about three. **The score adds up
over the question's requirements.** That is the compositional behaviour Chapter 34 said one
pooled vector could never have.

Put simply, the asymmetry is the whole design: **max over the document, sum over the query.**
One sentence to remember it by: *every requirement must find evidence, and documents are not
penalised for containing other things.*

Here are the four combinations side by side:

| Inside a row | Across rows | What happens on the SSO question |
|---|---|---|
| mean | sum | Long page 1,140 is diluted, the short overview page wins |
| sum | sum | The longest page wins, whatever it says |
| max | max | The Enterprise page wins on one strong word |
| **max** | **sum** | **The right page wins in both tests** |

---

## MaxSim is Chamfer similarity

MaxSim has an older name. Computer vision has used it for decades to compare two shapes, each
drawn as a cloud of points.

**Chamfer similarity = for each point in set A, its similarity to the closest point in set B,
added up over all of A.**

$$ \text{Chamfer}(Q, D) = \sum_{q \in Q} \max_{d \in D} \langle q, d\rangle $$

Here $\langle q, d\rangle$ is another way to write the dot product. Put the query's token
vectors in $Q$ and the document's in $D$, and this is exactly MaxSim. Means, ColBERT's score is
a set-to-set similarity with a long history.

Knowing the name gives us three useful facts.

**1. It compares two sets, not two vectors.** Every kind of Card Catalog in Part IV was built to
find the single vector nearest to another single vector. A set-to-set score is a different
mathematical object, so those indexes do not apply directly. Chapters 37 and 38 are two ways to
force it back into a shape an index can handle.

**2. It is not symmetric.** Chamfer(Q, D) is not the same as Chamfer(D, Q). Swap the roles and
every page token would need a partner in the question, which would punish long pages again. The
asymmetry is deliberate.

**3. It has known approximation theory.** Because Chamfer similarity is well studied, there are
constructions that turn a whole set of points into one vector whose dot product approximates
it, with proven error bounds. MUVERA (Chapter 38) uses exactly that.

---

## How ColBERT is trained

Before jumping into the recipe, we must know two terms.

**Distillation = training a small, fast student model to copy the scores of a big, slow teacher
model.** Here the teacher is a cross-encoder (Chapter 13), which reads the question and the page
together. It is accurate but far too slow to search a corpus. The student is ColBERT.

**KL loss = a number that says how far one probability distribution is from another.** KL
stands for Kullback–Leibler, and Chapter 12 met it briefly. It is zero when the student spreads
its belief over the candidate pages exactly as the teacher does, and it grows as the two
disagree.

The goal is Chapter 12's contrastive one, with MaxSim in place of the dot product. **ColBERTv1**
trained on MS MARCO's ready-made triples: a query, a passage labelled relevant, and a negative
mined with BM25. Its loss pushed the positive's score above the negative's. It worked, and it
left quality on the table.

**ColBERTv2 made three *training* changes** (its fourth contribution, residual compression, is
Chapter 37). These three matter as much as the architecture.

**1. Distillation from a cross-encoder.** Instead of yes/no labels, the student learns to match
the teacher's *graded* scores over many candidates, using a KL or margin-MSE loss (ColBERTv2
uses KL). It inherits the teacher's sense of "very relevant", "somewhat relevant" and "not
relevant", not just a single yes.

**2. Denoised supervision for hard negatives.** A first-round ColBERT retrieves hard candidates
for each training query. The teacher scores every one of them. The student is then trained
against those scores with a KL loss over **64-way tuples**: one query, 64 passages, and the
teacher's score for each passage.

We saw why this matters in Chapter 12. Page 1,140 was mined as a "negative" for our SSO question,
although it holds half the answer. A yes/no label pushes it away, and the model learns that a
correct answer is wrong. The teacher scores it high, so the student is no longer pushed away from
it. On MS MARCO this happens constantly, because usually only one passage per query is labelled.

**Note:** Outright *discarding* the negatives the teacher likes is a different variant, from a
system called RocketQA. ColBERTv2 keeps them and learns from their scores.

**3. In-batch negatives at scale**, exactly as in Chapter 12. The passages belonging to the
other queries in the same batch act as extra negatives, for free.

Here is the whole recipe as steps.

**Phase 1: Preparing the training data.**

**Step 1:** Take a training query from MS MARCO.

**Step 2:** Retrieve its top candidate passages with a first-round ColBERT.

**Step 3:** Have the cross-encoder teacher score every candidate.

**Step 4:** Keep 64 passages for the query: one highly ranked or labelled positive, plus 63
lower-ranked passages, each with its teacher score.

**Phase 2: One training step.**

**Step 1:** Score all 64 passages with MaxSim.

**Step 2:** Turn the student's 64 scores into probabilities with a softmax (Chapter 9).

**Step 3:** Turn the teacher's 64 scores into probabilities the same way.

**Step 4:** Compute the KL loss between the two sets of probabilities.

**Step 5:** Add an ordinary contrastive loss that uses the other queries' passages in the batch
as negatives.

**Step 6:** Nudge the weights to make the total loss smaller.

**Modern retrieval quality comes as much from supervision quality as from architecture.** A
well-supervised bi-encoder can beat a poorly supervised late-interaction model.

---

## Reading a MaxSim table

MaxSim gives us something a single vector never can: a readable reason for every score.

For each query token, we can print the page token that won its row. Here is what that might
look like when the Librarian brings back page 1,140 for our running question. The numbers are
illustrative.

| Query token | Best token on page 1,140 | Score |
|---|---|---|
| `pro` | `pro` | 0.93 |
| `plan` | `pro` | 0.64 |
| `include` | `needs` | 0.57 |
| `single` | `sso` | 0.82 |
| `sign` | `sso` | 0.86 |
| `on` | `on` | 0.71 |
| `[MASK]` | `security` | 0.61 |
| `[MASK]` | `add-on` | 0.55 |

Let's read it.

- `single` and `sign` both matched `sso`. The model bridged a spelled-out phrase and its
  abbreviation.
- `include` matched `needs`, a weaker but sensible pairing.
- Two `[MASK]` padding tokens reached for `security` and `add-on`, words nobody typed. That is
  the learned query expansion from Chapter 35 at work.

Failures are just as visible. A query token whose best score is 0.2 found no evidence on the
page.

**This is a debugging tool that single-vector retrieval cannot offer.** Wherever someone will
eventually ask "why did it return that?", it is one of the strongest arguments for late
interaction.

---

## Advantages and disadvantages of MaxSim

**Advantages:** it is length-robust (extra tokens add nothing unless they match better),
compositional, explainable per token, and the page side is precomputed once at index time.

**Disadvantages:** why not use MaxSim everywhere? There are four reasons.

**Problem 1: It is expensive to compute.** A 32-token query against a 200-token page is 6,400
dot products for that one page.

**Problem 2: It stores many vectors.** Page 212 needs about 200 token vectors instead of one.

**Problem 3: Ordinary indexes cannot search it.** HNSW or IVF cannot run a set-to-set score
directly.

**Problem 4: Scores do not compare across queries.** A score of 20 on one question and 18 on
another says nothing about which answer is better.

**When to use which one**

We must use a plain dot product when speed matters most and questions are short and topical. We
must use MaxSim when questions carry several requirements, when the evidence is one sentence in a
long page, or when results must be explained. We must use a cross-encoder to reorder a short list
at the highest accuracy (Chapter 41). Many strong systems use all three.

---

### Under the hood

Batched MaxSim, followed by the ColBERTv2 distillation loss. The comments point back to the
steps above.

```python
import torch

def maxsim_batch(Q, D, d_mask):
    """
    Q      : (B, nq, dim)   query token vectors, L2-normalised
    D      : (B, nd, dim)   document token vectors, L2-normalised
    d_mask : (B, nd)        1 for real tokens, 0 for padding
    """
    sim = torch.einsum('bqd,bnd->bqn', Q, D)             # Step 1: the (B, nq, nd) grid
    pad = ~d_mask.bool().unsqueeze(1)                     # (B, 1, nd)
    sim = sim.masked_fill(pad, -1e4)                      # padding can never win (≈ −∞ here)
    return sim.max(dim=-1).values.sum(dim=-1)             # Step 2: max per row, Step 3: sum → (B,)

def colbert_distill_loss(Q, docs, teacher_scores, tau=1.0):
    """
    docs           : list of w (D, d_mask) pairs; w = 64 in ColBERTv2
    teacher_scores : (B, w) cross-encoder scores for the same passages
    """
    student = torch.stack([maxsim_batch(Q, D, m) for D, m in docs], dim=1)   # (B, w)
    return torch.nn.functional.kl_div(
        torch.log_softmax(student / tau, dim=1),          # student's probabilities (as logs)
        torch.softmax(teacher_scores / tau, dim=1),       # teacher's probabilities
        reduction="batchmean")

# Sanity check with the page-212 table from earlier
S = torch.tensor([[0.90, 0.20, 0.10],
                  [0.15, 0.25, 0.85]])
S.max(dim=1).values.sum()                                 # → tensor(1.7500)
```

`einsum` builds the grid for a whole batch at once, `.max` keeps each row's winner and `.sum`
adds them. In simple words, three lines do what we did by hand, and the last line checks it.

That `masked_fill` line is not cosmetic. Documents in a batch have different lengths, so short
ones are topped up with padding rows (Chapter 11). Those rows can win the `max` in two ways.

- **Unmasked padding rows are unit-length garbage vectors.** ColBERT normalises every row, so a
  padding row points in some arbitrary direction with full length. It can beat a real token and
  win the max. Short documents carry the most padding, so their scores inflate silently.
- **Padding rows zeroed out can still win.** A zero row scores exactly 0. If a query token's
  best real match is negative, say −0.6, the max picks the 0 instead and lifts the score.

So we mask with a very large negative number, effectively −∞, either way. It is the
multi-vector version of Chapter 11's padding bug, and just as easy to miss.

---

### What people get wrong

**"MaxSim is just cosine similarity done many times."** The aggregation *is* the model.
Max-then-sum encodes an asymmetric, compositional idea of relevance that no single dot product
expresses.

**Forgetting to mask padding.** It inflates short documents. Always mask, and mask with −∞, not
with zero.

**Not normalising token vectors.** ColBERT L2-normalises every token vector. That bounds each
dot product to $[-1, 1]$ and keeps the sum readable as "how many requirements found evidence".
Skip it and no term's contribution is comparable to another's.

**Training without distillation and expecting v2 results.** The published gains come
substantially from supervision, not from architecture.

**Assuming MaxSim scores are comparable across queries.** ColBERT pads every query to 32
vectors, so every ColBERT score has 32 terms. Models without that fixed padding, such as ColPali
(Chapter 45), sum over the real query tokens, so longer queries reach higher totals. Even at equal
length, some questions simply match better than others. For thresholding, normalise by query
length when lengths vary, then calibrate on real data (Chapter 55).

---

### Ninja notes

**MaxSim's cost is the real engineering problem.** 6,400 dot products per document becomes 6.4
billion over a million documents. The saving grace is that the computation is one dense matrix
multiply, which GPUs perform at enormous throughput, and it batches trivially across documents. As a *reranker* over 1,000
candidates whose token vectors are already stored, MaxSim itself takes a few milliseconds.
Computing those token vectors on the fly costs far more, and Chapter 37 puts numbers on it. As a
*first-stage retriever* over millions of documents, it needs PLAID's centroid-based search (Chapter
37) or MUVERA's transformation (Chapter 38).

**Variants worth knowing.** Some models learn a weight per query token instead of a plain sum,
letting the model decide that `pro` matters more than `the`. Others replace the hard max with a
soft maximum, which gives smoother gradients and sometimes steadier training. Neither has
displaced plain MaxSim, which says a lot about how well the original function was chosen.

---

### Key takeaways

- **MaxSim = for each query token, its best match anywhere in the document, added up over the
  query.**
- **Max over the document** is evidence detection. Long pages are not diluted, and length alone
  cannot win.
- **Sum over the query** makes the score compositional: every requirement must find its own
  evidence.
- MaxSim is Chamfer similarity, a set-to-set measure. That is why ordinary ANN indexes do not
  apply directly, and why MUVERA can approximate it.
- ColBERTv2 made three training changes: cross-encoder distillation, denoised supervision of hard
  negatives through a KL loss over 64-way tuples, and in-batch negatives.
- The per-token best matches are a real, readable explanation of why a page scored well.
- Mask padding with −∞. Zeroed padding can still win a negative max.
- Cost is ~6,400 dot products per document. Chapters 37 and 38 deal with it.

### What's next

[Chapter 37](./37-colbert-3-serving.md) makes MaxSim affordable at scale: residual compression,
centroid pruning, and the PLAID engine that turns it into a real index.

We now know what MaxSim computes, why it takes the max over the page and the sum over the
question, and how ColBERT learns token vectors worth scoring.
