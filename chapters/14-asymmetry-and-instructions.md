---
title: "Asymmetric Search and Instruction-Tuned Embeddings"
chapter: 14
part: "Part II — Where Embeddings Come From"
slug: asymmetry-and-instructions
readingTime: "11 min"
summary: "A question is not a short answer. Modern embedding models want a prefix telling them which side you are on — and forgetting it is one of the most common silent bugs in RAG."
tags: [asymmetric-search, prefixes, instructions, e5, bge, nomic]
prev: 13-bi-vs-cross-encoders
next: 15-matryoshka
---

# Asymmetric Search and Instruction-Tuned Embeddings

**The one-paragraph version.** Most search is **asymmetric**: a short question has to find a
long page that answers it, and those two texts do not look alike. Embedding models handle
this by being trained with **prefixes**, such as `"query: "` and `"passage: "`, or with a
plain-language instruction. The prefix tells the encoder which role a text is playing. Leave
the prefix out, or use the wrong one, and quality drops without a single error message.

In this chapter, we will learn about asymmetric search, the reason a question and its answer
need different treatment. We will see what prefixes and instructions are, and watch a missing
prefix send one of our Acme questions to the wrong pages. We will also see how to check our
prefixes in five minutes, and when search is symmetric instead.

We will cover the following:

- What is asymmetric search
- What is an instruction prefix
- An example: a missing prefix, before and after
- Instruction-tuned embeddings
- How to check your prefixes in five minutes
- When search is symmetric
- Asymmetric vs symmetric search
- When to use which one

---

## What is asymmetric search

There are two kinds of search, and they want different things from the Map Room.

**Symmetric search = both sides are the same kind of thing, and "similar" means "alike".**

It asks, "find things like this thing." Two paraphrases. Two duplicate support tickets. Two
help pages that say the same thing.

**Asymmetric search = a question on one side, the page that answers it on the other.**

It asks, "find the thing that answers this."

Let's take a question from Acme's knowledge base:

> *"Can our team log in with our company accounts?"*

The page that answers it is page 88, a 300-word guide called "Setting up single sign-on". It
talks about SSO, certificates and identity providers, the services such as Okta that companies use
to manage staff logins. The question is nine words long. The page is three hundred. They share
almost no vocabulary. They are not *alike* at all. They **fit**.

Think of it like a key and a lock:

- The question is the key.
- The page is the lock.
- A key does not look like a lock. One is a small flat piece of metal, the other a heavy box.
- But the right key fits the right lock, and that fit is what we are searching for.

In simple words, the model's job is to put a question and its answer near each other in the
Map Room, even though the two look nothing alike. That is a different arrangement from "put
similar things together". Models trained only on symmetric data, pairs of look-alike texts, do
it poorly.

---

## What is an instruction prefix

So how does one model know whether a text is a question or a page?

We tell it.

**Instruction prefix = a short, fixed piece of text we put in front of the input to tell the
model what role that text plays.**

The E5 family of open embedding models popularised the simplest convention:

```python
doc_vec   = model.encode("passage: Single sign-on (SSO) lets your staff sign in to Acme through your identity provider...")
query_vec = model.encode("query: Can our team log in with our company accounts?")
```

Means, the same model reads `"query: ..."` as a question looking for an answer, and
`"passage: ..."` as a page waiting to be found.

The BGE family uses an instruction on the query side only:

```python
INSTRUCTION = "Represent this sentence for searching relevant passages: "
query_vec = model.encode(INSTRUCTION + "Can our team log in with our company accounts?")
doc_vec   = model.encode("Single sign-on (SSO) lets your staff sign in to Acme...")   # no prefix
```

The Nomic family uses a different prefix for each task:

```python
model.encode("search_query: ...")       # querying
model.encode("search_document: ...")    # indexing
model.encode("clustering: ...")         # grouping
model.encode("classification: ...")     # feature extraction
```

These are not decoration. The prefix tokens were present on every training example
(Chapter 12), and the model learned to shape its entire output around them.

In simple words, a prefix switches the model into a mode. Embed a question with the page
prefix, or with no prefix, and we get a vector built for the wrong job.

---

## An example: a missing prefix, before and after

Let's go back to our question:

> *"Can our team log in with our company accounts?"*

This is our paraphrase question. It never says "single sign-on", yet that is exactly what it
means. Keyword search struggles with it, because the pages that answer it say "SSO",
"identity provider" and "sign in", not "company accounts". This is the kind of question dense
search is supposed to win.

Acme uses an E5 model. All 40,000 pages were embedded with the `"passage: "` prefix, which is
correct. But the engineer who wrote the search box forgot to add `"query: "` to the question.

Let's first see what happens without the query prefix. The model now treats the question as
an ordinary sentence, and ordinary sentences land near texts that *look like* them. Here is
the top of the list (toy scores, to show the shape of the failure):

| Rank | Page | What the page is | Score |
|---|---|---|---|
| 1 | 31,207 | Forum post: "Can our team share one login?" | 0.86 |
| 2 | 5,530 | FAQ: "Can I change the email on my account?" | 0.85 |
| 3 | 14,002 | How to invite teammates to your workspace | 0.84 |
| … | … | … | … |
| 9 | 88 | Setting up single sign-on (SSO) | 0.81 |

The top three are all short, question-shaped texts about teams, logins and accounts. They
resemble the question. None of them answers it.

The Scholar (the LLM) reads the top 3. It thinks like this: "The user asks about team logins.
Page 31,207 says each person needs their own login and logins cannot be shared. Page 14,002
explains how to invite teammates." It answers:

> *"No. Each team member needs their own Acme account. You can invite them by email."*

The answer is wrong. Acme does let teams log in with their company accounts. It is called
single sign-on.

Now, let's add the seven missing characters, `"query: "`, and search again:

| Rank | Page | What the page is | Score |
|---|---|---|---|
| 1 | 88 | Setting up single sign-on (SSO) | 0.87 |
| 2 | 212 | Pro plan page: SSO included on Pro | 0.85 |
| 3 | 31,207 | Forum post: "Can our team share one login?" | 0.82 |

With the prefix, the model knows this text is a question. It places the question where the
*answers* to such questions live, not where look-alike questions live.

The Scholar reads the new top 3. It thinks like this: "The user wants staff to log in with
the accounts they already have at work. Page 88 calls that single sign-on and explains it.
Page 212 says SSO is included on the Pro plan." It answers:

> *"Yes. Acme supports single sign-on (SSO) through your identity provider, and it is
> included on the Pro plan."*

The answer is correct. Whether page 1,140's add-on caveat also reaches the Scholar is a
separate problem, the one the reranker solved in Chapter 13.

**The failure is silent.** Nothing crashed. Results came back. They were simply worse than
they should have been, in a way that looks like "embeddings aren't great for our domain."

---

## Instruction-tuned embeddings

The newer generation of models takes this further. Instead of a fixed prefix, we write the
task in plain language.

**Instruction-tuned embedding model = a model trained on many tasks, each described by a
sentence, so that we can describe our own task at query time.**

A few examples of such instructions:

```
"Given a web search query, retrieve relevant passages that answer the query"
"Given a support ticket, find similar previously resolved tickets"
"Given a legal clause, retrieve clauses with conflicting obligations"
"Given a code snippet, find its documentation"
```

One common format, used by e5-mistral for example, puts the instruction in front of the
query, and leaves the pages alone:

```python
task  = "Given a customer question, retrieve knowledge-base pages that answer it"
query = f"Instruct: {task}\nQuery: Can our team log in with our company accounts?"
```

Means, the instruction and the question travel into the model together. The instruction
tokens take part in attention (Chapter 10), so they reshape the vector that comes out. The
same encoder produces genuinely different geometry for each instruction.

This helps when one library must serve several tasks with different ideas of relevance.
Acme's support staff search the same knowledge base in two ways. Sometimes they want *"past
tickets similar to this ticket"*. Sometimes they want *"the policy page that governs this
ticket"*. Those are two different similarity functions, and one instruction-tuned model can
provide both.

There are two caveats.

First, instructions help, but only so much. Rewriting an instruction will shift results, but
it does not replace a model that has actually seen our kind of data.

Second, an instruction on the page side is baked into every stored vector. Changing it means
re-embedding all 40,000 pages. So in practice, most systems vary the instruction only on the
query side.

---

## How to check your prefixes in five minutes

Model cards (the documentation pages published with each model, listing its prefixes, pooling
and intended use, Chapter 11) are sometimes unclear, and code gets copied between projects. So
do not trust your prefixes. Test them.

**Step 1:** Take 20 (question, page) pairs we know belong together. For Acme, one of them is
"Can our team log in with our company accounts?" with page 88.

**Step 2:** Embed them four ways: both sides prefixed, neither side, query side only, page
side only.

**Step 3:** For each setup, rank the pages for every question, and check whether the
question's own page comes first.

**Step 4:** Count the hits for each setup and compare.

```python
import numpy as np

SETUPS = {"both":       ("query: ", "passage: "),
          "neither":    ("",        ""),
          "query only": ("query: ", ""),
          "page only":  ("",        "passage: ")}

def prefix_check(model, pairs):
    """pairs: 20 (question, page_text) pairs we know belong together."""
    for name, (qp, dp) in SETUPS.items():
        Q = model.encode([qp + q for q, _ in pairs], normalize_embeddings=True)
        D = model.encode([dp + d for _, d in pairs], normalize_embeddings=True)
        hits = (np.argmax(Q @ D.T, axis=1) == np.arange(len(pairs))).mean()
        print(f"{name:10s}  own page ranked first: {hits:.0%}")
```

In simple words, we let the model take the same small exam four times, once per setup, and
see which setup scores best. The correct configuration usually wins visibly, and now we know
for certain rather than by faith.

**Note:** Compare *ranks*, not raw similarity scores. A prefix can shift every score up or
down together, so "true pairs averaged 0.84 instead of 0.81" tells us nothing on its own. And
if every setup scores 100%, the exam is too easy: add a few hundred other pages as
distractors.

---

## When search is symmetric

Not everything is asymmetric. We use symmetric treatment, the same prefix on both sides or
none at all, when both sides are the same kind of thing:

- Deduplication and near-duplicate detection
- "More like this" and related-article recommendations
- Clustering and topic discovery
- Matching a support ticket to previously resolved tickets

Which symmetric prefix to use depends on the model. E5's model card, for example, says to put
`"query: "` on both sides for symmetric tasks. Nomic has its own `"clustering: "` prefix.

Getting this wrong in the other direction is possible too. Suppose Acme wants to find
duplicate help pages. If one copy gets `"query: "` and the other gets `"passage: "`, we push
two nearly identical pages into different parts of the Map Room, when we wanted them side by
side.

**The question to ask: are the two sides the same kind of thing?** Same kind → symmetric.
Question versus answer, short versus long, intent versus content → asymmetric.

---

## Asymmetric vs symmetric search

| | Asymmetric search | Symmetric search |
|---|---|---|
| The two sides | A question and its answer | Two things of the same kind |
| Typical lengths | Short vs long | Similar |
| "Similar" means | Fits | Looks alike |
| Acme example | "Can our team log in with our company accounts?" → page 88 | A help page → its near-duplicate |
| Prefixes | A different prefix per side, or an instruction on the query | The same prefix on both sides, or none |
| What goes wrong | Missing query prefix finds look-alike questions | Mixed prefixes push twins apart |

---

## When to use which one

We must use **asymmetric** treatment when a question is looking for its answer. That covers
the search box, question answering, and the retrieval step of RAG.

We must use **symmetric** treatment when both sides are the same kind of thing. That covers
deduplication, related articles, clustering and ticket-to-ticket matching.

Many strong systems use both, from the same model: asymmetric prefixes for the search box, and
symmetric ones for the "related articles" panel. Either way, read the model card, because the
exact strings differ from model to model.

---

### Under the hood

The cleanest defence is to make prefixes structurally impossible to forget. Wrap the model
once, at the boundary of the system, and never call `encode` directly again:

```python
class Embedder:
    QUERY_PREFIX = "query: "
    DOC_PREFIX   = "passage: "

    def __init__(self, model):
        self.model = model

    def embed_query(self, text):
        return self.model.encode(self.QUERY_PREFIX + text, normalize_embeddings=True)

    def embed_documents(self, texts):
        return self.model.encode([self.DOC_PREFIX + t for t in texts],
                                 normalize_embeddings=True)

embedder = Embedder(model)
page_vecs = embedder.embed_documents(acme_pages)     # all 40,000 pages, once
q = embedder.embed_query("Can our team log in with our company accounts?")
```

Two methods, two prefixes, no way to mix them up. In simple words, the search box can only
ever call `embed_query`, so it can never forget `"query: "` again.

Store the model name **and** the prefix scheme in the index's metadata (the extra facts stored
alongside it). When we migrate to a new model (Chapter 63), the prefix scheme changes too. A
mismatch between an old index and a new encoder is the kind of bug that survives a week of
debugging.

---

### What people get wrong

**Skipping the prefix entirely.** The most common mistake. It costs a few points of nDCG@10
(a ranking-quality score, Chapter 19), silently.

**Using the same prefix on both sides for a question-answer search.** It collapses the
asymmetry the model learned.

**Prefixing at index time but not query time (or vice versa).** Quietly guarantees a mismatch
between the two geometries you are comparing.

**Assuming all models use prefixes.** Many do not. OpenAI's embedding endpoints, and several
open models, expect raw text. Adding `"query: "` to a model that never saw it just injects a
couple of meaningless tokens and slightly degrades the output. **Read the model card.** There is no
universal convention.

**Translating a prefix.** Use the exact string from the model card, including capitalisation,
spacing and the trailing space. These are literal tokens the model was trained on, not
suggestions.

---

### Ninja notes

Chapter 13 called a bi-encoder a two-tower model, with both towers usually sharing weights.
Asymmetry has a deeper implication: the two towers can be *different encoders entirely*, with
separate weights for queries and documents. That is standard in recommendation systems, where
a user tower and an item tower encode fundamentally different objects into one shared space.

For text retrieval, shared weights plus prefixes usually win, because both sides are natural
language and sharing weights is a strong regulariser. But separate towers become attractive
when the sides are genuinely different modalities: text queries against structured records,
natural language against SQL, questions against page images. CLIP (Chapter 42) is exactly
this, a text tower and an image tower trained to meet in the middle. ColPali (Chapter 45) also
puts text questions and page images into one shared space.

The unifying idea is worth holding: **a shared embedding space does not require a shared
encoder.** It requires only a shared training objective.

---

### Key takeaways

- **Asymmetric search = a short question must *fit* a long page that answers it, not resemble
  it.**
- **Instruction prefix = a fixed string that tells the model which role a text plays**, such as
  `"query: "` and `"passage: "`.
- A missing or mismatched prefix degrades quality silently. Check it with a 20-pair exam, and
  compare ranks, not raw scores.
- Instruction-tuned models take a plain-language task description, usually on the query side
  only.
- Symmetric tasks (deduplication, similar items, clustering) need symmetric treatment.
- Wrap the encoder in `embed_query` / `embed_documents` so prefixes cannot be forgotten.
- A shared space needs a shared objective, not a shared encoder, and that is what makes
  multimodal retrieval possible.

### What's next

[Chapter 15](./15-matryoshka.md) covers a training trick that lets one model produce 768-,
256- or 64-dimensional vectors from a single forward pass. It makes cheap two-stage search
possible without a second model.

We now know why a question is not just a short page, how prefixes and instructions tell the
model which is which, and how to catch the silent bug when they go missing.
