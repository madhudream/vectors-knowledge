---
title: "Contrastive Learning: How Embedding Models Are Trained"
chapter: 12
part: "Part II — Where Embeddings Come From"
slug: contrastive-learning
readingTime: "15 min"
summary: "Pull the pair together, push the crowd apart. The objective is four lines of code. Everything that makes a model good or bad lives in how you choose the negatives."
tags: [training, contrastive, infonce, hard-negatives, fine-tuning]
prev: 11-pooling
next: 13-bi-vs-cross-encoders
---

# Contrastive Learning: How Embedding Models Are Trained

**The one-paragraph version.** Show the model a query and a document that answers it, plus a batch
of documents that do not. Adjust the weights so the right pair scores higher than the wrong ones.
Repeat over hundreds of millions of pairs. That is the whole recipe, and it is word2vec's negative
sampling applied to sentences. Model quality is determined less by architecture than by **what we
use as negatives**, and that is where all the craft lives.

In this chapter, we will learn how an embedding model is trained to put a question near its answer.
First we will meet the vocabulary of training: loss, gradient, batch and logits. Then we will see
the standard objective, InfoNCE, and the kinds of negatives it learns from. Finally, we will
watch a single mislabelled negative teach a model to miss half of our running answer, and see how
the best recipes prevent it.

We will cover the following:

- What is contrastive learning
- How training works: loss, gradient and batch
- The training data: pairs that belong together
- What is InfoNCE
- In-batch negatives
- Hard negatives
- An example: the negative that was actually right
- Distillation: learning from a better teacher
- When to fine-tune

---

## What is contrastive learning

**Contrastive learning = training by comparison: pull a matching pair together, and push
non-matching items apart.**

Think of it like teaching a child to recognise dogs. We could describe fur and snouts and tails. Or
we could do what actually works: hold up two pictures and say *"this is a dog, that is not."* Do it
ten thousand times with cats, wolves, foxes and mops. The child never receives a definition, and
ends up with a better classifier than any definition would produce.

Now the mapping:

- The child is the embedding model.
- "This is a dog" is a **positive**, an item that belongs with the thing we are asking about.
- "That is not" is a **negative**, an item that does not.
- Each comparison is one small training step.

Crucially, the *hard* comparisons teach the most. Dog-versus-teapot teaches almost nothing, because
the child got it right immediately. Dog-versus-wolf teaches a great deal, because that is where the
boundary actually is.

Hold onto that. It is the single most important practical fact in this chapter.

In our world, the question is *"Does the Pro plan include single sign-on?"* and the dog is Acme's
page 212: "SSO is included on the Pro plan." We will follow this one question all the way through.

---

## How training works: loss, gradient and batch

Chapter 9 said training "nudges the weights until the guesses improve". Now, the question is, how
does the model measure "improve", and how does it know which way to nudge?

**Loss = a single number that says how wrong the model currently is.** Zero means perfect. Bigger
means worse. Training means nudging the weights to make the loss smaller.

Many losses in this book follow one pattern. Take the probability p that the model gives to the
right answer, and compute −log p (the natural log from Chapter 8):

| Probability on the right answer | Loss = −log p |
|---|---|
| 0.9 | 0.105 |
| 0.5 | 0.693 |
| 0.1 | 2.303 |
| 0.01 | 4.605 |

In simple words, confident and right costs almost nothing. Confident and wrong costs a lot.

**Gradient = for every weight, how much the loss would rise if that weight grew a little.** Training
nudges each weight the opposite way, so the loss falls. Put simply, the gradient gives the
direction of the nudge: it points uphill, and training steps the other way, downhill.

A toy example. One weight is 0.50, and the loss is 0.69. If raising that weight to 0.51 would push
the loss up to 0.70, the gradient is positive. So training lowers the weight a little instead, and
the loss drops.

**Batch = the group of training examples the model processes together in one step.** For example,
1,024 (question, page) pairs. The model computes one loss for the whole batch, then takes one
nudge.

---

## The training data: pairs that belong together

Training data is a pile of pairs that *belong together*. Each pair has an **anchor** (usually the
query) and a **positive**:

| Anchor | Positive |
|---|---|
| "Does the Pro plan include single sign-on?" | Page 212: "SSO is included on the Pro plan." |
| "how do I reset my password" | "Password reset: click Forgot Password…" |
| A photo of a beach | "sunset over the ocean" |
| A question from Stack Overflow | Its accepted answer |
| A paper's title | Its abstract |

These pairs come from places where "belongs together" is already recorded: question and accepted
answer, title and abstract, caption and image, query and clicked result, citation and cited paper.
For Acme, it is a support ticket and the help article that resolved it. Nobody hand-labels hundreds
of millions of pairs. Finding naturally paired data at scale *is* the dataset engineering.

Then, for each anchor, we need **negatives**: documents that do not answer it.

---

## What is InfoNCE

The standard loss treats training as a multiple-choice quiz.

**InfoNCE = a quiz loss: given the query, pick the right document out of all the candidates.** (NCE
stands for noise-contrastive estimation.)

Given an anchor $q$, one positive $d^+$, and $N-1$ negatives, the model must pick the positive:

$$ \mathcal{L} = -\log \frac{\exp(\text{sim}(q, d^+)/\tau)}{\sum_{j=1}^{N}\exp(\text{sim}(q, d_j)/\tau)} $$

Let's read it piece by piece:

- $\text{sim}(q, d)$ is the cosine similarity (Chapter 4) between the query and a document.
- $\tau$ is the temperature, explained below. Dividing by it turns similarities into logits.
- $\exp$ raises e to that power. Dividing by the sum over all $N$ documents is exactly the softmax
  from Chapter 9.
- $-\log$ of the positive's probability is the loss pattern from the table above.

**Logits = raw, unbounded scores that go into a softmax.** They can be any size, positive or
negative. The softmax turns them into probabilities.

In English: *the positive's similarity should dominate the sum of all similarities.* Push it up,
push everything else down. Word2vec did this with a handful of random words. Now the items are
sentences, and the negatives are documents.

Here is how one quiz question is scored.

**Step 1:** Embed the query and every candidate document.

**Step 2:** Compute the cosine similarity of the query with each document.

**Step 3:** Divide each similarity by the temperature τ. These are the logits.

**Step 4:** Apply a softmax, so the logits become probabilities that add up to 1.

**Step 5:** Take the probability p on the positive. The loss is −log p.

**Step 6:** Compute the gradient, and nudge the weights so that p rises.

Let's run it on our question with three candidates and τ = 0.05:

| Candidate | Similarity | Logit (÷ 0.05) | Probability |
|---|---|---|---|
| Page 212, "SSO is included on the Pro plan." (positive) | 0.80 | 16 | 0.99995 |
| Refund policy page | 0.30 | 6 | 0.00005 |
| Password reset page | 0.20 | 4 | 0.000006 |

The loss is −log 0.99995 ≈ 0.00005. The model already gets this right. The loss is almost zero, so
the gradient is almost zero, and the model learns almost nothing. Dog versus teapot.

**Temperature = τ, the number that controls how sharply the softmax separates close scores.** It is
typically 0.01–0.07. In simple words, it sets how harshly the model treats near-misses.

Low temperature sharpens the softmax, so the hardest negative dominates the gradient: the model
obsesses over the closest wrong answer. High temperature spreads attention across all negatives.
Too low, and training gets unstable and over-separates. Too high, and the model never learns fine
distinctions. We will see real numbers for this in the hard negatives section.

---

## In-batch negatives

Here is the trick that makes this economical.

**In-batch negatives = using the documents of the *other* pairs in the same batch as negatives, for
free.**

Take a batch of 1,024 (query, document) pairs. For query #7, the positive is document #7, and
documents #1–6 and #8–1,024 are, with high probability, negatives.

We get 1,023 negatives per query for zero extra compute. One matrix multiply produces the full
N × N similarity grid. The diagonal holds the positives, and everything off the diagonal is a
negative. Here is a batch of three:

| | Doc 1: page 212 (SSO on Pro) | Doc 2: refund policy | Doc 3: password reset |
|---|---|---|---|
| Query 1: "Does the Pro plan include SSO?" | **positive** | negative | negative |
| Query 2: "How do refunds work?" | negative | **positive** | negative |
| Query 3: "How do I reset my password?" | negative | negative | **positive** |

In simple words, every query's right answer is every other query's wrong answer. The four-line code
for this is in Under the hood.

This is also why **batch size matters enormously** for embedding models. More items in the batch
means more negatives, which means a harder quiz and a better model. Training runs with batch sizes
of 16k–64k are normal. They are achieved through GradCache (splitting a huge batch into pieces that
fit in GPU memory) or cross-device gathering (collecting negatives from many GPUs). It is one of the
few places where raw hardware budget translates directly into model quality.

---

## Hard negatives

In-batch negatives are mostly *easy*. In a batch of a thousand random Acme pages, our SSO question
is competing against pages on invoices, refunds and API rate limits. The model gets those right
immediately and learns nothing. That is the dog-versus-teapot problem, and we just watched it
happen: a loss of 0.00005.

**Hard negatives = documents that look right and are not.**

| Query | Positive | Hard negative |
|---|---|---|
| "Does the Pro plan include single sign-on?" | "SSO is included on the Pro plan." | "SSO is included on the Enterprise plan." |
| "how to reset password" | "Password reset instructions…" | "How to change your password while logged in" |
| "What does error SSO-4012 mean?" | "SSO-4012: SAML assertion expired." | The help page for error `SSO-4021` |

These are where the decision boundary lives. Mining them is standard practice: retrieve the top 50
with an existing model, remove known positives, and use the rest.

Let's put the first hard negative into our quiz, still with τ = 0.05:

| Candidate | Similarity | Logit | Probability |
|---|---|---|---|
| Page 212 (positive) | 0.80 | 16 | 0.73 |
| "SSO is included on the Enterprise plan." (hard negative) | 0.75 | 15 | 0.27 |
| Refund policy page | 0.30 | 6 | 0.00003 |

The loss is −log 0.73 ≈ 0.31, about 6,000 times the easy batch's loss. Now there is something to
learn.

Look at where the push goes. Of all the push applied to the negatives, the Enterprise page gets
99.99%. If we raised τ to 1, it would get only about 61%, and the refund page would soak up the
rest. That is temperature at work: low τ makes the model focus on the closest wrong answer.

---

## An example: the negative that was actually right

But there is a trap, and it is the most important subtlety in modern embedding training. **Some
mined "hard negatives" are actually unlabelled positives.**

Let's watch it happen to our question. Our training data says page 212 answers *"Does the Pro plan
include single sign-on?"*. We mine hard negatives: the old model's top 50 pages, minus page 212.
Page 1,140 is in that list, because it is about SSO on Pro. But page 1,140 is not a wrong answer. It
holds the other half of the right one: on Pro, SSO needs the Security add-on for teams under 50
seats. Nobody labelled it.

Let's first see what naive training does. It treats page 1,140 as a negative. Every time page 1,140
shows up, the gradient pushes its vector *away* from the question. After enough steps, the
Librarian reliably brings back page 212 and reliably leaves page 1,140 behind. The Scholar answers
*"Yes, SSO is included on Pro."* The answer is wrong for every Pro team under 50 seats. Training
taught the model that a correct answer is wrong.

Now, the question is, how do we stop it? There are three standard mitigations:

1. Skip negatives that score above a similarity threshold, because a "negative" that similar is
   suspicious.
2. Sample negatives from ranks 30–100 rather than 1–10, where unlabelled positives are rarer.
3. Use a cross-encoder (Chapter 13) to score every candidate and train against those *scores*
   rather than yes/no labels.

A cross-encoder is a slower, more accurate model that reads the question and the page together
(Chapter 13 explains it). Here it plays the teacher.

Let's run the third mitigation. The teacher reads our question alongside page 1,140 and gives it a
high score. The model is now trained to match the teacher's scores, not the label "negative". So a
mislabelled "negative" the teacher likes is no longer pushed away. Page 1,140 stays close to the
question.

The Librarian now brings back both pages. The Scholar thinks: *"Page 212 says SSO is on Pro. Page
1,140 says small Pro teams need an add-on. So the answer is yes, with a condition."* It writes:
*"SSO is included on Pro, but teams under 50 seats need the Security add-on."* The answer is
correct.

This is ColBERTv2's **denoised supervision**. (Outright discarding teacher-approved negatives is
RocketQA's variant.) The idea appears throughout state-of-the-art training recipes.

---

## Distillation: learning from a better teacher

The strongest current recipes go one step further. They do not train against binary labels at all.
They train against the *scores* of a cross-encoder. This is *distillation* in the broad sense: a
small, fast model learns to copy a big, slow model's judgements (Chapter 36 goes deeper).

A cross-encoder is usually far more accurate than a bi-encoder of similar size (a model that
embeds the query and the document separately, like every model in this chapter, Chapter 13). But
it is far too slow to serve. So we run the teacher over millions of (query, document) pairs
offline. Then we train the fast student to match the teacher's score distribution, usually with a
margin-MSE or KL objective. MSE is mean squared error, the average squared gap between two sets
of numbers. Margin-MSE teaches the student to reproduce the *gap* between the teacher's score for
a positive and for a negative. KL (Kullback–Leibler divergence) measures how different two
probability distributions are, so minimising it makes the student's probabilities look like the
teacher's.

The student inherits much of the teacher's judgement at a small fraction of the serving cost. This
teacher-student pattern is behind many of the models at the top of the leaderboards. It also tells
us something practical: if we are going to fine-tune, generating teacher scores is often a better
use of a day than hand-labelling.

---

## When to fine-tune

**Fine-tuning = taking an existing embedding model and training it further on our own pairs.** For
Acme, those would be support tickets and the help articles that resolved them.

Should you fine-tune? Usually not first. In rough order of return on effort:

1. **Fix chunking and add hybrid search.** Often larger gains than fine-tuning, and often in a day.
2. **Add a reranker** (a slower second-pass model that re-orders the top results). Once retrieval
   works, it is often the largest remaining quality jump (Chapter 41).
3. **Try three off-the-shelf models on your own eval set** (real questions with known right answers,
   Chapter 19). It is free, and the spread between models on *your* data is frequently larger than
   the spread on MTEB (the public embedding leaderboard, Chapter 16).
4. **Then fine-tune**, if you have a genuinely specialised vocabulary and at least a few thousand
   real query-document pairs, ideally from click logs.

Fine-tuning a bi-encoder on a few hundred synthetic pairs generally produces a model that is worse
in ways your eval set is too small to detect.

We must fine-tune when our words mean something special (for Acme, "Pro" and "Security add-on" are
product names, not plain English), steps 1–3 are done, and we have thousands of real pairs.

We must not fine-tune as a first fix for a problem that chunking, hybrid search or a reranker would
solve more cheaply.

---

### Under the hood

```python
import torch, torch.nn.functional as F

def infonce(Q, D, tau=0.05):
    Q, D = F.normalize(Q, dim=-1), F.normalize(D, dim=-1)
    logits = Q @ D.T / tau                       # (N, N) similarity grid: raw, unbounded scores
    labels = torch.arange(len(Q), device=Q.device)   # diagonal = correct
    return F.cross_entropy(logits, labels)
```

That is a complete, working contrastive loss. Four lines.

In simple words, `Q` holds the batch's query vectors and `D` holds their documents' vectors, in
matching order. `F.normalize` makes every vector unit length (Chapter 5), so `Q @ D.T` is the full
grid of cosine similarities. Dividing by `tau` turns them into logits. `labels` says that the right
answer for query i is document i, which is the diagonal. `F.cross_entropy` does the softmax and the
−log p of Steps 4 and 5, then averages over the batch.

---

### What people get wrong

**"I need labelled data to fine-tune."** You need *pairs*. Support tickets and their resolutions,
questions and accepted answers, search queries and clicked results: you probably already have
thousands.

**"More negatives is always better."** More *hard* negatives is better, up to the point where false
negatives start poisoning the signal. Filter them.

**"Small batches are fine, I'll just train longer."** Batch size directly determines the number of
in-batch negatives. Small batches are a weaker objective, not merely a slower one.

**"I fine-tuned and MTEB scores dropped."** Expected. You specialised the model. What matters is
your own eval set. But do keep a general eval to detect catastrophic forgetting (losing general
skill while learning the new one), which is real and can be severe.

---

### Ninja notes

Two properties of the objective explain behaviour you will observe in the wild.

**Alignment and uniformity.** The contrastive loss can be decomposed into pulling positives together
(alignment) and spreading everything over the sphere (uniformity). The uniformity term is what
fights the anisotropy of Chapter 3. It is *why* trained embedding models have a wider usable dynamic
range than raw BERT. A high random-pair similarity, such as 0.85, can come from weak uniformity
pressure during training. It can also come simply from a very low training temperature. E5's model
cards, for example, explain that its scores sit around 0.7–1.0 because it trains at τ = 0.01.
Either way, a high score alone does not mean the model ranks badly, since only the ordering matters.

**Dimension collapse.** If training is misconfigured (temperature too high, too few negatives, batch
too small), the model can converge to using only a fraction of its dimensions, with the rest
carrying near-zero variance. You have paid for 768 dimensions and are using 60. Diagnose it by
running PCA on a corpus sample. As a rough rule of thumb, if 95% of the variance sits in the first
50 components, suspect collapse. This is a far stronger symptom than Chapter 6's low intrinsic
dimension. Intrinsic dimension follows a curved surface, while PCA only finds flat directions, so
a healthy model spreads its variance over many more PCA components. Once collapse is real, no
index tuning will recover the lost capacity.

---

### Key takeaways

- **Contrastive learning = pull matching pairs together, push negatives apart.** InfoNCE does it in
  four lines.
- **Loss** is how wrong the model is. The **gradient** says which way each weight must move to
  shrink it. **Logits** are the raw scores that go into the softmax.
- In-batch negatives are free, so batch size is a first-class quality lever.
- Temperature decides how much the hardest negative dominates the gradient.
- Hard negatives are where learning happens, and false negatives among them are the main way
  training goes wrong. A mislabelled page 1,140 teaches the model to miss half the answer.
- Denoised supervision trains against a teacher's scores, so a "negative" the teacher likes is not
  pushed away. Distillation from a cross-encoder teacher is the current state-of-the-art recipe.
- Fine-tune fourth, after chunking and hybrid search, reranking, and model selection.

### What's next

We just used a cross-encoder as a teacher without fully explaining it.
[Chapter 13](./13-bi-vs-cross-encoders.md) covers the two fundamental architectures of neural
retrieval, and the speed-versus-accuracy trade that defines every pipeline in this book.

We now know how an embedding model learns, why the negatives matter more than the architecture, and
how one wrong label can quietly teach a model to be wrong.
