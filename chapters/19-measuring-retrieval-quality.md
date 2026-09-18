---
title: "Measuring Retrieval Quality"
chapter: 19
part: "Part III — Finding Neighbours"
slug: measuring-retrieval-quality
readingTime: "14 min"
summary: "Recall@k, MRR, nDCG, and the crucial distinction between system recall and index recall — two different numbers that share a name and get confused constantly."
tags: [evaluation, metrics, recall, ndcg, mrr, measurement]
prev: 18-recall-latency-memory
next: 20-ann-family-tree
---

# Measuring Retrieval Quality

**The one-paragraph version.** **Recall@k** asks whether the right pages are in the top k at
all. **MRR** asks how high the first right page ranks. **nDCG@k** asks whether the whole ordering
is good, giving the top positions the most weight. For RAG, start with recall@k: if the answer is
not retrieved, nothing downstream can save it. And be careful. "Recall" means two completely
different things, depending on whether we are grading our *system* or our *index*.

In this chapter, we will learn how to turn "did the Librarian bring back the right pages?" into a
number we can trust. We will work out every common metric by hand on our SSO question, see how
differently they judge the same list, and learn which one to use for which job.

We will cover the following:

- Why we need to measure retrieval
- The two recalls
- What is recall@k
- What is precision@k
- What is MRR
- What is nDCG
- A worked example: every metric on one list
- How many test questions we need
- When to use which metric

---

## Why we need to measure retrieval

Our running question is *"Does the Pro plan include single sign-on?"*

The full answer needs two pages. Page 212 says "SSO is included on the Pro plan." Page 1,140 says
"On Pro, SSO needs the Security add-on for teams under 50 seats."

Suppose we change something: a new embedding model, a new chunk size, a new index setting. Did
the Librarian get better or worse?

"It looks better" is not an answer. We try five questions, three look better, two look worse,
and we have learned nothing. We need a number, computed the same way every time, on the same
questions.

To compute that number, we must know the right answers in advance.

**Golden set = a list of test questions, each paired with the pages a person marked as right.**

Here is the golden-set entry for our question:

```
question:        "Does the Pro plan include single sign-on?"
relevant pages:  212, 1,140, 2,306
```

Page 2,306 is "Compare plans", the plan-comparison table, which ticks SSO in the Pro column. The
answer does not strictly need it, but it helps, so the person marking answers counted it. Chapter 54
builds a full golden set step by step.

In simple words, the golden set is the answer key, and a metric is how we mark the test.

Now we ask the Librarian our question. It returns 10 pages, best first. We will use this one list
for every metric in the chapter:

| Rank | Page returned | Relevant? |
|---|---|---|
| 1 | Enterprise plan overview | no |
| 2 | **Page 212:** SSO is included on the Pro plan | **yes** |
| 3 | Setting up two-factor login | no |
| 4 | SSO error codes | no |
| 5 | **Page 1,140:** On Pro, SSO needs the Security add-on under 50 seats | **yes** |
| 6 | Resetting your password | no |
| 7 | Basic plan overview | no |
| 8 | March release notes | no |
| 9 | **Page 2,306:** Compare plans (SSO ticked for Pro) | **yes** |
| 10 | Inviting teammates | no |

---

## The two recalls

Before any formula, one warning. This trips up nearly everyone, including people who have shipped
retrieval systems.

**System recall** (also called retrieval quality or end-to-end recall) asks: *of the pages that
genuinely answer this question, how many did we return?* The answer key is a person's judgement,
our golden set. It measures the embeddings, the chunking, the query handling: the whole pipeline.

**Index recall** (also called ANN recall, the recall of Chapter 18) asks: *of the k nearest
vectors by exact distance, how many did the approximate index return?* The answer key is brute
force. It measures only the index's shortcuts. It knows nothing about relevance.

They are unrelated numbers. Let's see both extremes on our question.

> Suppose our embedding model places the question near the Enterprise plan pages. The index
> faithfully returns the exact 10 nearest vectors. **Index recall is 1.00.** But none of pages
> 212, 1,140 or 2,306 is among them. **System recall is 0.** The index is perfect, and the model
> is wrong for our domain.
>
> Now suppose the index skips two of the true 10 nearest vectors, and both were irrelevant pages.
> **Index recall is 0.80**, yet all three relevant pages came back. The system is excellent.

When someone says "our recall is 95%", the first question is always *which one*. Chapter 18's
tuning curves are index recall. Everything in Part VII is system recall. Optimising the wrong one
is a classic way to spend three months achieving nothing.

From here on, "recall@k" means system recall, graded against the golden set.

---

## What is recall@k

**Recall@k = relevant pages in the top k ÷ all relevant pages.**

$$ \text{Recall@}k = \frac{|\{\text{relevant docs in top } k\}|}{|\{\text{all relevant docs}\}|} $$

In simple words, of all the pages that should come back, what fraction made it into the first k?

Our golden set has 3 relevant pages. From the list above:

- **Recall@3** = 1 / 3 = 0.33. Only page 212 is in the top 3.
- **Recall@5** = 2 / 3 = 0.67. Pages 212 and 1,140 are in the top 5.
- **Recall@10** = 3 / 3 = 1.00. All three are in the top 10.

Now, let's see why the choice of k is not a detail.

Suppose we hand the Scholar the top 3 pages. It reads the Enterprise overview, page 212 and the
two-factor page. It thinks like this: "Page 212 says SSO is included on Pro. Nothing here
mentions a condition." It answers, "Yes, the Pro plan includes SSO."

That answer fails every Pro team under 50 seats.

Now we hand it the top 5 pages. It thinks like this: "Page 212 says SSO is included on Pro. Page
1,140 adds that teams under 50 seats need the Security add-on. So the answer is yes, with a
condition." It answers, "Yes, but teams under 50 seats need the Security add-on."

This time the answer is right. Same Librarian, same list. Recall@3 was 0.33 and recall@5 was 0.67.

That is why recall@k is the right primary metric for RAG. Retrieval in a RAG pipeline is a
**filter feeding a reader**. The Scholar reads all k pages we hand it. It does not much care
whether page 1,140 sat at position 2 or position 5. It cares whether the page is *there*.

Position does matter somewhat. Language models pay less attention to pages buried in the middle
of a long prompt (the text we send the model). That is the "lost in the middle" effect of
Chapter 53. But presence matters far more. **Recall@k is the ceiling on the whole system's
quality.** Nothing downstream can use a page that retrieval never returned.

So we measure at the k we actually pass to the next stage:

- `recall@100` for the first-stage retriever, if we rerank 100 pages.
- `recall@5` for the final context we give the Scholar.

The gap between those two numbers shows how much of the ceiling the reranker manages to keep.

---

## What is precision@k

**Precision@k = relevant pages in the top k ÷ k.**

$$ \text{Precision@}k = \frac{|\{\text{relevant docs in top } k\}|}{k} $$

Means, of what we returned, how much was relevant?

On our list, **precision@5** = 2 / 5 = 0.40. Two of the five pages we returned were relevant.

Precision matters when a person reads the list. A web search page with three junk results in the
top five is a bad experience.

For RAG it matters less than we might think, because the Scholar can ignore some irrelevant
pages. But it is not free. Irrelevant pages use up **tokens** (the word-pieces a language model
reads, Chapter 10), add latency, cost money, and can actively distract the model. "Precision does
not matter" is roughly true up to about 5–10 pages and false after that.

---

## What is MRR

First, one smaller idea.

**Reciprocal rank = 1 ÷ the rank of the first relevant page.**

On our list, the first relevant page is page 212, at rank 2. Its reciprocal rank is 1 / 2 = 0.5.
First result relevant gives 1.0. Third gives 0.33. Tenth gives 0.1.

**MRR = Mean Reciprocal Rank = the average reciprocal rank over all questions in the golden set.**

$$ \text{MRR} = \frac{1}{|Q|}\sum_{i=1}^{|Q|} \frac{1}{\text{rank}_i} $$

In simple words, add up 1/rank for each question's first hit, then divide by the number of
questions. Suppose our golden set has three questions. Ours scores 0.5, a second question's first
hit is at rank 1 (1.0), and a third's is at rank 4 (0.25). Then MRR = (0.5 + 1.0 + 0.25) / 3 =
0.58.

MRR is the right metric when there is essentially **one** right answer and the user wants it
immediately: "take me to the SSO setup page", a known-item lookup, a navigational search.

It is a poor fit when many pages are relevant, because it ignores everything after the first hit.
Watch what that means for us. Delete page 1,140 from the list entirely. The first hit is still
page 212 at rank 2, so the reciprocal rank is still 0.5. **MRR does not move**, yet the Scholar's
answer just became wrong.

---

## What is nDCG

**nDCG = normalized Discounted Cumulative Gain = a score from 0 to 1 for how well the whole
ranking puts relevant pages near the top.**

Let's break it down:

- **Gain**: each relevant page earns points.
- **Cumulative**: we add the points up over the top k.
- **Discounted**: a page earns fewer points the lower it sits.
- **normalized**: we divide by the best score possible, so a perfect ranking scores exactly 1.

nDCG is the standard metric in **information retrieval (IR)**, the field that studies search. It
does two things the other metrics do not. It handles **graded relevance** (perfect / good /
marginal / irrelevant, not just yes or no), and it discounts by position smoothly.

$$ \text{DCG@}k = \sum_{i=1}^{k} \frac{2^{rel_i}-1}{\log_2(i+1)} $$

$$ \text{nDCG@}k = \frac{\text{DCG@}k}{\text{IDCG@}k} $$

In simple words, each page's gain is divided by a number that grows slowly with its rank. Here
i is the rank and rel_i is the page's relevance grade. The top of the fraction gives more
relevant pages many more points. The bottom shrinks the reward for sitting low in the list.
**IDCG** (the Ideal DCG) is the DCG of the perfect ordering, so nDCG lands between 0 and 1 and can
be compared across questions.

The bottom uses **log₂**, the base-2 cousin of Chapter 8's natural log. It simply counts how many
times we double 1 to reach a number, so log₂(8) = 3 because 1 → 2 → 4 → 8.

The discount is gentle. Position 1 is divided by log₂(2) = 1, position 3 by log₂(4) = 2, and
position 7 by log₂(8) = 3. So a hit at rank 7 counts a third as much as one at rank 1.

Now let's compute nDCG@10 for our list, one step at a time. Our golden set marks pages simply as
relevant (grade 1) or not (grade 0). A grade-1 page gains 2¹ − 1 = 1 point. A grade-0 page gains
nothing.

**Step 1: Find the relevant ranks.** They are 2, 5 and 9.

**Step 2: Discount each gain by its rank.**

```
rank 2:  1 / log2(2+1)  = 1 / log2(3)  = 1 / 1.585 = 0.631
rank 5:  1 / log2(5+1)  = 1 / log2(6)  = 1 / 2.585 = 0.387
rank 9:  1 / log2(9+1)  = 1 / log2(10) = 1 / 3.322 = 0.301
```

**Step 3: Add them up.** DCG@10 = 0.631 + 0.387 + 0.301 = **1.319**.

**Step 4: Compute the best possible score.** In the ideal list, our 3 relevant pages sit at ranks
1, 2 and 3.

```
rank 1:  1 / log2(2) = 1 / 1.000 = 1.000
rank 2:  1 / log2(3) = 1 / 1.585 = 0.631
rank 3:  1 / log2(4) = 1 / 2.000 = 0.500
```

IDCG@10 = 1.000 + 0.631 + 0.500 = **2.131**.

**Step 5: Divide.** nDCG@10 = 1.319 / 2.131 = 0.619, which rounds to **0.62**.

That is it. The right pages are all present, but they sit lower than they could, so the ordering
earns 62% of the best possible score. The code in Under the hood prints the same 0.62.

Earlier chapters spoke of "points of nDCG@10" on a 0–100 scale. That is the same score multiplied
by 100. Our 0.62 is 62 points, and "+5 points" means +0.05.

**Note:** With graded relevance, the exponent starts to matter. A grade-2 ("good") page gains
2² − 1 = 3 points, three times a grade-1 page, and a grade-3 ("perfect") page gains 2³ − 1 = 7.
That is how nDCG rewards putting the most useful page first.

We use nDCG when relevance is genuinely graded and ordering matters: search result pages and
recommendations. For a RAG first stage, recall@k is more actionable and far cheaper to label.

---

## A worked example: every metric on one list

Here is our one list again, with every metric side by side. Relevant pages sit at ranks 2, 5 and
9, and 3 relevant pages exist in total.

| Metric | Value | Calculation |
|---|---|---|
| Recall@3 | 0.33 | 1 of 3 found in top 3 (only page 212) |
| Recall@5 | 0.67 | 2 of 3 found in top 5 |
| Recall@10 | 1.00 | 3 of 3 found |
| Precision@5 | 0.40 | 2 of 5 returned are relevant |
| MRR | 0.50 | first hit at rank 2 → 1/2 |
| nDCG@10 | ~0.62 | DCG 1.319 ÷ IDCG 2.131: correct pages present but poorly positioned |

Notice how differently they behave.

A team chasing MRR would work to move page 212 from rank 2 to rank 1. That is worth nothing to a
Scholar that reads the top 5. A team that hands the Scholar only the top 3 but tracks recall@10
would declare victory, since recall@10 is already perfect. If it tracks recall@3 (0.33) instead,
it sees the real problem: page 1,140 must climb into the top 3. A team that hands over the top 5
tracks recall@5 (0.67), and a look at the list shows that the one page missing is the optional
page 2,306.

**The metric we choose defines the work we do.** So we choose it to match what the system
actually needs.

---

## How many test questions we need

How many questions must the golden set hold before a score means anything?

Every score from a golden set is an estimate. Ask a different 20 questions and the score would
come out a little different.

**Standard error = how much a score would typically wobble if we re-ran the evaluation on a fresh
set of similar questions.**

For a score that is a proportion p (such as "fraction of questions where the answer was
retrieved") over n questions, the standard error is √(p × (1 − p) / n).

Let's take an example. We have 20 questions, and the score is about 0.5.

```
standard error = √(0.5 × 0.5 / 20) = √0.0125 = 0.112   → about ±11 points
95% interval   ≈ 2 × standard error                    → about ±22 points
```

In simple words, a measured 50% could plausibly be anywhere from about 28% to 72%. A 3-point
improvement is invisible inside that. With 50 questions, the standard error falls to about ±7
points. With 200, about ±3.5 points.

So fifty questions is a working minimum, and 200 is comfortable. Ninja notes show how comparing
two systems on the *same* questions squeezes more out of a small set.

---

## When to use which metric

We must use **recall@k** when retrieval feeds a reader, which is every RAG system. Measure it at
the k we actually pass along.

We must use **precision@k** when a person reads the results list, or when the context we send is
long enough that junk pages start to cost money and attention.

We must use **MRR** when each question has one right answer and the user wants it first, such as
a "take me to that page" search.

We must use **nDCG** when relevance is graded and order matters, such as a search results page.

Many strong teams track two: recall@k as the headline number, and nDCG@10 to watch the ordering.

---

### Under the hood

The metrics, run on our SSO list:

```python
import numpy as np

def recall_at_k(ranked_ids, relevant_ids, k):
    return len(set(ranked_ids[:k]) & set(relevant_ids)) / max(len(relevant_ids), 1)

def precision_at_k(ranked_ids, relevant_ids, k):
    return len(set(ranked_ids[:k]) & set(relevant_ids)) / k

def reciprocal_rank(ranked_ids, relevant_ids):    # MRR = mean of this over all questions
    for i, doc in enumerate(ranked_ids, start=1):
        if doc in relevant_ids:
            return 1.0 / i
    return 0.0

def ndcg_at_k(ranked_ids, relevance, k):          # relevance: id -> grade
    gains = [(2 ** relevance.get(d, 0) - 1) / np.log2(i + 2)
             for i, d in enumerate(ranked_ids[:k])]
    ideal = sorted(relevance.values(), reverse=True)[:k]
    idcg = [(2 ** g - 1) / np.log2(i + 2) for i, g in enumerate(ideal)]
    return sum(gains) / sum(idcg) if sum(idcg) else 0.0

def index_recall(approx_ids, exact_ids, k):       # the OTHER recall
    return len(set(approx_ids[:k]) & set(exact_ids[:k])) / k

ranked = ["enterprise-plan", "page-212", "two-factor", "saml-errors", "page-1140",
          "password-reset", "basic-plan", "release-notes", "page-2306", "invite-team"]
relevant = {"page-212", "page-1140", "page-2306"}
grades = {page: 1 for page in relevant}

print(round(recall_at_k(ranked, relevant, 3), 2))    # 0.33
print(round(recall_at_k(ranked, relevant, 5), 2))    # 0.67
print(precision_at_k(ranked, relevant, 5))           # 0.4
print(reciprocal_rank(ranked, relevant))             # 0.5
print(round(ndcg_at_k(ranked, grades, 10), 2))       # 0.62
```

`enumerate` counts from 0, so rank i + 1 becomes `np.log2(i + 2)`. That is the same log₂(rank + 1)
we used by hand.

Note that `index_recall` divides by `k`, not by the number of relevant pages. Put simply, its
answer key is exactly the k nearest vectors found by brute force. Different denominator,
different meaning, same word.

---

### What people get wrong

**Reporting one number.** A single average hides everything. Report recall@k by question
*segment*: identifier lookups like the error code `SSO-4012`, natural-language questions,
multi-condition questions. The average can look fine while a whole segment sits at zero.

**Tiny eval sets.** With 20 questions, the standard error on a proportion is roughly ±11 points,
and the 95% interval is about ±22. You cannot detect a 3-point improvement. Fifty is a working
minimum, and 200 is comfortable.

**Confusing index recall with system recall.** Covered above, and worth checking whenever a
number surprises you.

**Forgetting unjudged documents.** Suppose the golden set marks 3 pages relevant, and the system
returns a 4th that is *also* relevant but was never labelled. The system is penalised for being
right. This is the same false-negative problem as hard-negative mining (Chapter 12). It is why
academic IR uses pooled judgements, where the top results of many systems are pooled and all of
them get judged.

**Optimising a metric that does not match the product.** Improving MRR from 0.62 to 0.71 while
recall@10 stays flat means the Scholar is reading much the same pages as before.

---

### Ninja notes

Two practices make evaluation genuinely useful rather than ceremonial.

**Bootstrap your confidence intervals.** Resample the question set with replacement a thousand
times, recompute the metric each time, and take the 2.5th and 97.5th percentiles. Now "0.81 vs
0.84" becomes "0.81 [0.74–0.87] vs 0.84 [0.78–0.90]". The intervals overlap heavily, so on these
numbers alone we cannot claim a win. A paired test (below) is the proper check. This one habit
prevents an enormous amount of wasted work chasing noise.

```python
def bootstrap_ci(per_query_scores, n=1000):
    s = np.array(per_query_scores)
    means = [np.mean(np.random.choice(s, len(s), replace=True)) for _ in range(n)]
    return np.percentile(means, [2.5, 97.5])
```

**Use paired tests for comparisons.** When comparing two systems on the same questions, compare
*per-question differences*, not the two averages. A paired bootstrap or a randomisation test has
far more statistical power, because it removes the variance caused by some questions simply
being harder than others. This routinely turns an apparently inconclusive comparison into a clear
answer with the same 50 questions.

---

### Key takeaways

- **Recall@k = relevant pages in the top k ÷ all relevant pages.** It is the primary metric for
  RAG, because it is the ceiling on everything downstream.
- System recall (did we return the relevant pages?) and index recall (did the approximation find
  the true nearest vectors?) are different numbers with the same name.
- On our SSO list (hits at ranks 2, 5, 9 of 3 relevant): recall@3 = 0.33, recall@5 = 0.67,
  precision@5 = 0.40, MRR = 0.50, nDCG@10 = 0.62.
- MRR fits one-right-answer search. nDCG fits graded, ordered results.
- Measure at the k you actually use, and report by question segment.
- With 20 questions the standard error is about ±11 points. Use at least 50, bootstrap the
  confidence intervals, and compare systems with paired tests.

### What's next

[Chapter 20](./20-ann-family-tree.md) closes Part III by sorting every ANN algorithm ever invented
into four ideas, after which Part IV is details.

We now know how to put a trustworthy number on retrieval, which recall a number refers to, and
which metric fits which job.
