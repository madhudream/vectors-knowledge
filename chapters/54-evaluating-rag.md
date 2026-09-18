---
title: "Evaluating RAG"
chapter: 54
part: "Part VII — RAG"
slug: evaluating-rag
readingTime: "11 min"
summary: "Evaluate retrieval and generation separately, or you will never know which one broke. The four measurements that matter, how to build a golden set in a day, and how far to trust an LLM judge."
tags: [evaluation, rag, faithfulness, golden-set, llm-as-judge, ragas]
prev: 53-context-assembly
next: 55-rag-failure-modes
---

# Evaluating RAG

**The one-paragraph version.** A RAG system has two parts that fail independently, so we evaluate
them independently. **Retrieval**: did the right pages reach the context? **Generation**: given
that context, was the answer faithful, relevant and complete? We build a golden set of 50–200 real
questions with known answers and known source pages. We automate the scoring with an LLM judge,
but only after checking the judge against human labels. An unchecked judge is just a more
expensive way of guessing.

In this chapter, we will learn how to tell whether a RAG system works and, when it does not, which
part broke. We will watch a team trust an SSO answer that looks right and is wrong, then see four
measurements point straight at the fault. We will build a golden set around our running question,
learn how far to trust an LLM judge, and work out how many questions we need before the numbers
mean anything.

We will cover the following:

- What is RAG evaluation
- Why "the answer looks good" is not a metric
- The four measurements
- An example: diagnosing the SSO answer
- Building a golden set
- LLM-as-judge
- How many questions we need
- When to use which check

---

## What is RAG evaluation

**RAG evaluation = measuring retrieval and generation separately, on a fixed set of questions with
known answers, so we know which part to fix.**

Two terms carry this whole chapter.

**Golden set = a fixed list of real questions, each stored with its correct answer and the pages
that contain that answer.** Chapter 19 used a small one to score retrieval. Here we build a full
one.

**Faithfulness = the share of claims in the answer that the retrieved context actually supports.**
In simple words, did the Scholar stick to the pages, or add things from memory?

Think of it like a mechanic with a car that will not start.

- The car is our RAG system.
- The fuel line is retrieval. Did the right pages arrive?
- The spark is generation. Did the Scholar use them correctly?
- "The car will not start" is end-to-end answer quality. It tells us something is wrong, not what.

A good mechanic tests the fuel and the spark separately. We do the same.

---

## Why "the answer looks good" is not a metric

Acme's team has just built its support assistant. To test it, they ask our running question:
*"Does the Pro plan include single sign-on?"*

It answers: *"Yes, single sign-on is included on the Pro plan [1]."* It even cites page 212. The
team reads it, nods, and ships.

Two weeks later, a customer with a 20-seat team complains. They upgraded to Pro for single
sign-on, followed the setup guide, and hit a paywall. Page 1,140 says teams under 50 seats need
the Security add-on. The answer was wrong for them, and the eyeball test could not see it.

Now, the question is, what went wrong? When a user reports a wrong answer, there are seven
possible causes:

1. The page with the answer does not exist.
2. It exists but was not retrieved.
3. It was retrieved but ranked too low, and fell outside the context budget.
4. It was in the context and the model ignored it.
5. It was in the context and the model misread it.
6. The model answered from pretraining instead of the context.
7. The answer was right and the user was wrong.

Seven causes, seven different fixes, and **end-to-end answer quality alone cannot tell any of them
apart.** That is why the first rule of RAG evaluation is to measure the stages separately.

---

## The four measurements

```
                          ┌──────────────────────────┐
question ──► retrieval ──►│ 1. CONTEXT RECALL        │  was the needed info retrieved?
                          │ 2. CONTEXT PRECISION     │  how much of it was relevant?
                          └──────────────────────────┘
                                       │
                                       ▼
                          ┌──────────────────────────┐
             generation ─►│ 3. FAITHFULNESS          │  is every claim supported by context?
                          │ 4. ANSWER CORRECTNESS    │  does it actually answer, correctly?
                          └──────────────────────────┘
```

**1. Context recall: did retrieval find what was needed?** It is the fraction of the gold evidence
that made it into the assembled context. This is Chapter 19's recall@k, and it is the ceiling for
everything downstream. **Measure it first.**

**2. Context precision: how much of the context was relevant?** Low precision wastes tokens and
distracts the model. It matters less than recall, until it matters a lot.

**3. Faithfulness, also called groundedness: is every claim in the answer supported by the
context?** This measures hallucination relative to the sources, and it is the metric most directly
tied to user trust. An answer can be *correct* and *unfaithful*, right by luck from pretraining.
That is still a failure, because next time the pretraining will be wrong.

**4. Answer correctness: is it the right answer?** We compare it against a reference answer. It
includes completeness, so an answer that is faithful but leaves out half of what was asked is not
correct.

Tools such as RAGAS (below) also report **answer relevancy**: whether the answer addresses the
question at all. It needs no reference answer, but it cannot tell a relevant wrong answer from a
right one.

The diagnosis falls out of the combination:

| Context recall | Faithfulness | Correctness | Diagnosis |
|---|---|---|---|
| Low | — | Low | **Retrieval problem.** Fix chunking, hybrid, reranking. |
| High | Low | Low | **Model ignoring context.** Fix prompt and instructions. |
| High | High | Low | **Context insufficient or misread.** Fix assembly, expansion. |
| Low | Low | High | **Lucky pretraining.** Dangerous. Will fail on your private data. |
| High | High | High | Working. |

---

## An example: diagnosing the SSO answer

Let's score the answer that went wrong. The golden set says the SSO question needs two pages,
212 and 1,140.

**Context recall.** The context held page 212 but not page 1,140. That is 1 of 2 gold pages, so
recall is 0.5.

**Faithfulness.** The answer makes one claim: "SSO is included on Pro". Page 212 supports it.
Faithfulness is 1.0.

**Correctness.** The reference answer mentions the Security add-on for teams under 50 seats. The
answer leaves that out, so it is marked incorrect.

Low recall, low correctness. The table says: **retrieval problem.** The Scholar did its job
perfectly with what it was given. Page 1,140 simply never arrived.

Without this split, the team's first instinct was to rewrite the prompt, adding "always mention
conditions and add-ons". That fix is wrong, and it could never have worked. No instruction can
make the Scholar cite a page it never saw.

With the split, the team fixes retrieval instead. They switch to structure-aware chunking
(Chapter 50) and add decomposition (Chapter 52). They re-run the golden set. Recall goes to 1.0,
faithfulness stays at 1.0, and the answer now includes the add-on. The fix is correct, and the
numbers prove it.

---

## Building a golden set

A useful golden set takes about a day to build.

**Morning: collect questions.** 50 is the minimum, and 200 is comfortable.

- **Real queries from logs**, if we have them. Always prefer these.
- **Ask domain experts**, here Acme's support agents, to write the questions they actually get.
- **Stratify deliberately.** Include simple lookups, multi-hop questions, comparisons, identifier
  queries like *"What does SSO-4012 mean?"*, negations like *"Which plans do not include single
  sign-on?"*, and questions about recent changes.
- **Include questions the corpus cannot answer.** Unanswerable questions are how we test the "I
  don't know" behaviour, and most eval sets forget them entirely.

**Afternoon: annotate each question.**

```yaml
- id: q_001
  question: "Does the Pro plan include single sign-on?"
  segment: multi-page
  reference_answer: "Yes. SSO is included on Pro, but teams under 50 seats
                     also need the Security add-on."
  gold_sources: ["kb_212", "kb_1140"]
  answerable: true

- id: q_017
  question: "What does error SSO-4012 mean?"
  segment: identifier
  reference_answer: "The SAML assertion has expired."
  gold_sources: ["kb_17450"]
  answerable: true

- id: q_044
  question: "Does Acme offer pet insurance?"
  segment: unanswerable
  reference_answer: null
  gold_sources: []
  answerable: false
```

Means, each question carries four labels, and each label unlocks one measurement. `gold_sources`
enables the retrieval metrics. It lists page IDs, not chunk IDs, because chunk IDs change every
time we re-chunk, and re-chunking is exactly when we need the eval. For a long page, we can also
store a short quote of the sentence that holds the answer, and check that the quote reached the
context. `reference_answer` enables correctness. `answerable: false` tests refusal. For the pet
insurance question, the only right answer is some version of "Acme does not offer that". `segment`
lets us report by question type, which Chapter 19 showed is where the real decisions hide.

**Synthetic questions help, carefully.** We can prompt an LLM to write questions from our pages.
The same idea produced part of ColPali's training data (Chapter 46). It scales well. It also
produces questions that are cleaner, more literal and easier to answer than the ones real users
write, so synthetic sets systematically overestimate quality. Use them to widen coverage, never as
the only set.

---

## LLM-as-judge

Faithfulness and correctness are hard to score by matching strings. So we use a model.

**LLM-as-judge = using an LLM to score answers, for example by checking each claim in an answer
against the sources.**

```python
FAITHFULNESS_JUDGE = """You are checking whether an answer is supported by sources.

Sources:
{context}

Answer:
{answer}

Step 1: List each factual claim in the answer.
Step 2: For each claim, state SUPPORTED, CONTRADICTED, or NOT_FOUND in the sources,
        quoting the supporting text if any.
Step 3: Output JSON: {{"claims": [...], "faithfulness": supported / total}}"""
```

In simple words, the judge splits the answer into single claims, checks each one against the
sources, and reports the fraction that were supported.

Suppose the assistant answers *"Yes, SSO is included on Pro, at no extra cost."* The judge finds
two claims. "SSO is included on Pro" is SUPPORTED by page 212. "At no extra cost" is NOT_FOUND, or
CONTRADICTED if page 1,140 is in the context. Faithfulness is 1 of 2, which is 0.5.

Splitting the answer into claims before scoring is the important design choice. It makes the
judgement auditable, and much more reliable than a single overall score.

**But calibrate the judge before trusting it.**

**Step 1:** Hand-label 50 answers yourself for faithfulness and correctness.

**Step 2:** Run the judge on the same 50.

**Step 3:** Measure how often the judge agrees with you, separately for each label. Most claims
are supported, so a lazy judge that always says SUPPORTED would score well on overall agreement.
Cohen's kappa, an agreement score that removes the agreement expected by chance, avoids the same
trap.

If agreement is poor, fix the judge prompt, use a stronger judge model, or add worked examples,
then measure again. **An uncalibrated LLM judge produces confident numbers that may not track
reality at all.**

Judges have known biases. They tend to prefer longer answers, prefer the style of their own model
family, and go easy on fluent text.

Frameworks such as RAGAS, TruLens and DeepEval implement these metrics. They are useful starting
points. The calibration step is still yours.

---

## How many questions we need

Now, the question is, how many golden questions are enough to trust a score?

Chapter 19 introduced the **standard error**, the typical amount a score moves just because of
which questions happened to be in the set. For a score that is a fraction $p$ of $n$ questions, it
is $\sqrt{p(1-p)/n}$.

Take the worst case, a system that gets half the questions right, with $n = 20$:

$$ \sqrt{0.5 \times 0.5 / 20} \approx 0.11 $$

That is a standard error of about ±11 points. A 95% interval is about two standard errors wide on
each side, so about ±22 points. In simple words, a score of 50% on 20 questions is consistent with
a true score anywhere from about 28% to 72%.

| Questions | Standard error | 95% interval |
|---|---|---|
| 20 | ±11 points | ±22 points |
| 50 | ±7 points | ±14 points |
| 100 | ±5 points | ±10 points |
| 200 | ±3.5 points | ±7 points |
| 385 | ±2.5 points | ±5 points |

So with 20 questions, a 5-point improvement is invisible. With 200, one score is still ±7 points,
wider than the 5-point gain we hope to see. A ±5-point 95% interval on one score needs about 385
questions, since 0.5 × 0.5 × (1.96 / 0.05)² ≈ 385. Comparing the averages of two separately scored
runs needs about twice as many per run, because both runs add noise.

The better way is a **paired test** (Chapter 19): we score both versions on the same questions and
compare them question by question. Only the questions whose result changed add noise, so a paired
test often shows a 5-point gain with about 200 questions. It still does not turn 20 questions into
a trustworthy test.

---

## When to use which check

Not every check needs to run all the time. Cheap checks run often, and expensive ones run less
often.

| Check | Needs | Cost | When to run |
|---|---|---|---|
| Context recall | `gold_sources` only | Seconds, no LLM calls | Every change |
| Refusal on unanswerable questions | `answerable: false` questions | A few LLM calls | Every change |
| Faithfulness and correctness | LLM judge plus reference answers | Many LLM calls | Nightly, and before releases |
| Judge calibration | 50 human-labelled answers | Human time | When the judge or its prompt changes |
| Online signals | Production logs | Cheap, but noisy | All the time, reviewed weekly |

We must run **context recall** on every change, because it is nearly free and it is the ceiling on
everything else. We must run the **judge-based metrics** before every release. We must
**recalibrate the judge** whenever we change it. And we must watch **online signals** to find the
questions our golden set is missing. Many strong teams do all four.

---

### Under the hood

```python
import pandas as pd

def evaluate(system, golden):
    rows = []
    for ex in golden:
        out = system.answer(ex.question)              # returns answer + context used
        ctx_pages = {c.doc_id for c in out.context}   # page IDs survive re-chunking

        recall = (len(ctx_pages & set(ex.gold_sources)) / len(ex.gold_sources)
                  if ex.gold_sources else None)

        # float() turns True/False into 1.0/0.0, so pandas can average these columns
        if ex.answerable:
            faith = judge_faithfulness(out.context, out.answer)
            correct = float(judge_correctness(ex.question, ex.reference_answer, out.answer))
            refused = None
        else:
            faith = correct = None
            refused = float(judge_is_refusal(out.answer))   # did it say "I don't know"?

        rows.append(dict(id=ex.id, segment=ex.segment, recall=recall,
                         faithfulness=faith, correctness=correct, refused=refused,
                         latency_ms=out.latency_ms, tokens=out.tokens))
    df = pd.DataFrame(rows)
    return df.groupby("segment").mean(numeric_only=True)
```

In simple words, for every golden question we run the system and score each stage. For the SSO
question, recall comes from comparing the pages in the context to `kb_212` and `kb_1140`. For the
pet insurance question, we skip recall and check only whether the system refused. Then we average
the scores within each segment, so the report shows recall, faithfulness, correctness and refusal
rate side by side. Without `float()`, pandas would store the True/False judgements as plain
objects, and `mean(numeric_only=True)` would silently leave out correctness and refusal.

Report **by segment**, with **bootstrapped confidence intervals**. To bootstrap, resample the golden
questions with replacement many times, re-average each time, and see how far the average moves
(Chapter 19). Run it on every change (chunking, model, prompt, reranker, index parameters) and keep
the history. This is your regression suite, the set of tests that catches things getting worse.
Without it, every "improvement" is an opinion.

---

### What people get wrong

**Evaluating only end-to-end.** You cannot tell which stage failed.

**No unanswerable questions.** Then you never measure whether the system fabricates, which is the
failure users punish hardest.

**Only synthetic questions.** Systematically optimistic.

**Trusting an uncalibrated judge.** Measure agreement with humans first.

**Twenty questions.** With 20 examples the standard error is roughly ±11 points, and a 95%
interval is about ±22. You cannot detect realistic improvements. Chapter 19 has the arithmetic.

**Evaluating once.** Corpora change, models get deprecated, prompts drift. Evaluation is
continuous or it is theatre.

**Ignoring cost and latency in the eval.** A change that adds two points of correctness and
doubles latency is a product decision, and the eval should surface both numbers together.

---

### Ninja notes

**Online signals complement offline evals.** Thumbs up or down, copy events, follow-up questions
that rephrase the original (a strong signal of failure), and citation click-through all carry
information about real performance on real traffic. They are noisy and biased toward engaged
users. They are also the only measurement of the questions you actually serve. Log them, and look
at the worst-rated conversations weekly. That is where new golden-set questions come from.

**Grow the golden set from failures.** Every production failure a user reports becomes a new test
case, with its segment tagged. The 20-seat customer's SSO complaint becomes question q_001's
reason for existing. Over months, the set comes to represent the hard parts of your real traffic
rather than the easy parts you imagined at the start. This is the single habit most correlated
with RAG systems that improve over time rather than plateau.

**Evaluate retrieval with the cheapest signal first.** Context recall needs only `gold_sources`
and no LLM calls, so it runs in seconds on hundreds of questions. Run it on every commit. Keep the
expensive judge-based metrics for nightly runs or release candidates.

---

### Key takeaways

- **RAG evaluation = measuring retrieval and generation separately, on a fixed set of questions
  with known answers, so we know which part to fix.**
- The SSO answer looked right and was wrong. Context recall of 0.5 showed page 1,140 never
  arrived, so the fix was retrieval, not the prompt.
- Four measurements: context recall, context precision, **faithfulness**, answer correctness.
- A **golden set** is 50–200 real, stratified questions with answers and source pages, **including
  unanswerable ones** like pet insurance.
- Synthetic questions widen coverage but overestimate quality.
- **LLM-as-judge** scales scoring. Split answers into claims, and calibrate against human labels
  before trusting it.
- At 20 questions the standard error is about ±11 points and the 95% interval about ±22. Aim for
  200, and compare versions with a paired test on the same questions. A ±5-point interval on one
  score needs about 385.
- Report by segment with confidence intervals, run on every change, and track cost and latency.
  Grow the golden set from production failures.

### What's next

[Chapter 55](./55-rag-failure-modes.md) is a field guide: the recurring ways RAG systems break, and
the symptom that identifies each one.

We now know how to measure a RAG system stage by stage, how to build the questions that measure
it, and how many of them we need before the numbers can be trusted.
