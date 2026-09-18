---
title: "RAG From First Principles"
chapter: 49
part: "Part VII — RAG"
slug: rag-first-principles
readingTime: "13 min"
summary: "Why retrieve at all when context windows are enormous? Because attention is not retrieval, tokens cost money, and your data changes. The honest case for RAG, and its exact shape."
tags: [rag, architecture, first-principles, context-window, long-context]
prev: 48-other-modalities
next: 50-chunking
---

# RAG From First Principles

**The one-paragraph version.** RAG, short for retrieval-augmented generation, means finding the
right pages and putting them in front of a language model before it answers. People now ask why
we should bother, when some models can read a million tokens at once. There are five answers.
Our collection is bigger than any model can read at once. Answer quality drops long before that
limit. Every extra token costs money and time. Retrieval can pick up a page edited a minute ago.
And retrieval can respect permissions and give citations, which a prompt stuffed with everything
cannot.

In this chapter, we will learn what RAG is, starting from the model that writes the answer. We
will watch that model get Acme's single sign-on question wrong on its own, and then get it right
once retrieval hands it the correct pages. We will also see why "just put everything in the
prompt" is not enough, and when long context is the better tool.

We will cover the following:

- What is an LLM
- What is RAG
- Why we need RAG
- How RAG works
- Why not put everything in the context window
- What the Scholar actually needs
- RAG vs long context
- When to use which one

---

## What is an LLM

Now the Scholar, whom we met in Chapter 1, takes centre stage.

Everything in the first 48 chapters was about making the Librarian good: turning pages into
vectors, placing them in the Map Room, and building a Card Catalog to find them fast. Part VII is
about what happens next, when the Librarian hands the books to the Scholar.

The Scholar is brilliant and articulate, and has read an enormous amount, but not *our*
documents. In real terms, the Scholar is the LLM.

**LLM = Large Language Model.**

**Large:** it has learned billions of numbers from a huge amount of text.

**Language:** text goes in, and text comes out.

**Model:** its behaviour was learned from data, not written by hand (Chapter 2).

In simple words, an LLM is a program that has read a huge amount of text, and when we ask it
something, it writes an answer one small piece at a time. Those pieces are tokens (Chapter 10),
and one token is roughly three-quarters of an English word.

We need two more terms before we can talk about RAG.

**Prompt = the text we hand the model.** It is everything the model sees for one request: our
instructions, any pages we attach, and the question.

**Context window = the maximum amount of text a model can read in one go.** It is measured in
tokens. For most models, the prompt and the answer together must fit inside it. Today's windows
range from tens of thousands of tokens up to a million or more.

Think of it like the Scholar working at a desk.

- The Scholar is the LLM.
- The desk is the context window. Only so many pages fit on it at once.
- The pages we lay on the desk, plus our note with the question, are the prompt.
- Anything not on the desk, the Scholar can only answer from memory.

And here is the catch. The Scholar's memory has never seen Acme's knowledge base. It does not
know Acme's prices, last Tuesday's outage, or what a customer's contract says. Ask about those,
and it will either admit it does not know, or produce something that sounds right. The second
one has a name.

**Hallucination = a fluent, confident answer that no source supports.** The model is not lying
on purpose. It writes the most likely-sounding text, and likely-sounding is not the same as true.

---

## What is RAG

**RAG = Retrieval + Augmented + Generation.**

**Retrieval:** the Librarian finds the few pages most likely to hold the answer.

**Augmented:** we add those pages to the prompt, right next to the question.

**Generation:** the Scholar writes the answer from the pages in front of it.

In simple words, we find the right pages first, put them on the Scholar's desk, and only then ask
the Scholar to answer.

```
question → [Librarian: retrieve] → relevant documents → [Scholar: read + answer] → answer
                                                                  ↑
                                                          with citations
```

Neither character can do the job alone. The Librarian can find any page in the Great Library in
milliseconds, but cannot write an answer. The Scholar writes beautifully, but has never read our
pages. RAG is the handoff between them.

---

## Why we need RAG

A customer types a question into Acme's support assistant:

> *"Does the Pro plan include single sign-on?"*

Let's first see what the Scholar does alone, with no retrieval. The prompt contains only the
question.

The Scholar has never seen Acme's pages. It has seen thousands of other software pricing pages,
though, and on many of them single sign-on is kept for the most expensive plan. It thinks like
this: *"Many companies reserve SSO for their top plan. 'Pro' sounds like a middle plan. So SSO is
probably not on Pro."*

It writes: *"No. Single sign-on is only available on Acme's Enterprise plan. Upgrade to
Enterprise to turn it on."*

The answer is wrong. Page 212 of the Library says SSO *is* included on Pro. The Scholar invented a
policy that sounds normal for the industry. That is a hallucination, and nothing in the answer
warns the customer.

Now, let's see what RAG does with the same question.

The Librarian searches the Great Library and brings back two pages:

- **Page 212:** "SSO is included on the Pro plan."
- **Page 1,140:** "On Pro, SSO needs the Security add-on for teams under 50 seats."

We put both pages into the prompt, labelled [1] and [2], with an instruction to answer only from
them. The Scholar thinks like this: *"Source 1 says Pro includes SSO. Source 2 adds a condition
for small teams. So the honest answer is yes, with a condition."*

It writes: *"Yes, single sign-on is included on the Pro plan [1]. If your team has fewer than 50
seats, you also need the Security add-on [2]."*

The answer is correct. It also comes with citations, so the customer can open pages 212 and 1,140
and check it.

Put simply: without retrieval, the Scholar answers from memory. With retrieval, the Scholar
answers from the right pages.

**Note:** RAG is only as good as what the Librarian brings. If the Librarian returns page 212
alone, the Scholar says "Yes" with no condition, which is half an answer. Making sure both pages
arrive is the work of Chapters 50 to 52.

---

## How RAG works

A RAG system runs in two phases. The first prepares the Library. The second answers each
question.

**Phase 1: Indexing.** We do this once, and again whenever pages change.

**Step 1: Parse.** Pull clean text out of every page, whether it is HTML, a PDF or a scan.

**Step 2: Chunk.** Cut each page into passages of a few hundred words. Each passage is a
**chunk**, and Chapter 50 is all about how to cut them well.

**Step 3: Embed.** Turn each chunk into a vector with an embedding model (Chapter 2).

**Step 4: Store.** Save each vector with its text and its metadata, such as title and URL, in
the Card Catalog.

**Phase 2: Answering.** We do this for every question.

**Step 1: Ask.** The user asks a question.

**Step 2: Embed.** Turn the question into a vector with the same embedding model.

**Step 3: Search.** The Librarian asks the Card Catalog for the top-k closest chunks (Chapter 3).

**Step 4: Build the prompt.** Put the numbered chunks, the instructions and the question together.

**Step 5: Generate.** Send the prompt to the LLM.

**Step 6: Return.** Give the user the answer and its citations.

Here is the same pipeline in two lines:

```
INDEX TIME
  documents → parse → chunk → embed → store (vectors + text + metadata)

QUERY TIME
  question → embed → search → top-k chunks → prompt → LLM → answer + citations
```

That is four steps to prepare and six to answer. We can build it in an afternoon. The afternoon
version will demo well and disappoint in production, for reasons that are entirely predictable
and that Chapters 50 to 55 list one by one.

The production version differs in five specific places:

```
question
  → query understanding        (rewrite, decompose, expand)      [Ch. 52]
  → hybrid retrieval           (dense + sparse, fused with RRF)  [Ch. 40]
  → rerank                     (cross-encoder over ~100)         [Ch. 41]
  → context assembly           (dedupe, order, expand, budget)   [Ch. 53]
  → generate with citations
  → evaluate continuously                                        [Ch. 54]
```

Means, we improve the question before searching, and search with both meaning and keywords.
Then we re-score the top hundred results with a slower, more accurate model, arrange the prompt
with care, and measure the whole thing all the time. Each of those five is often worth more than
upgrading the embedding model. That is the central practical claim of Part VII.

---

## Why not put everything in the context window

Now, the question is, why retrieve at all? Some models can read a million tokens. Why not put the
whole Library on the desk every time?

This is the serious objection, and it deserves a serious answer. There are five reasons.

**First, the Library does not fit.** A million-token window holds roughly 750,000 words, about
eight novels. Acme's knowledge base has 40,000 pages. At a modest 500 words a page, that is 20
million words, or about 27 million tokens. Acme's support pages alone would fill about 27 full
windows. A whole company's document store is usually many times bigger, and a large enterprise's
runs to thousands of windows. Collections tend to grow at least as fast as windows do, so the next
model generation does not close this gap.

**Second, quality drops well before the limit.** Models use facts in the middle of a long prompt
less reliably than facts near the start or the end. This is often called "lost in the middle"
(Chapter 53). A model given pages 212 and 1,140 usually answers better than the same model given
those two pages buried among 500 unrelated ones. **Irrelevant context is not neutral. It is a
distractor.**

**Third, cost and waiting time grow with every token.** Sending 500,000 tokens per question is
expensive and slow to start answering. Sending 4,000 is neither, and it is 125 times fewer tokens.
At any real number of questions per day, this decides the economics.

**Fourth, freshness.** A page edited two minutes ago can be in the index two minutes later.
Retraining a model is not a path to real time, and re-sending every page on every question runs
straight back into the third reason.

**Fifth, access control.** Retrieval can filter out pages a user may not see *before* the model
ever reads them (Chapters 33 and 62). If every page goes into the prompt, the model has already
read pages meant for someone else. The leak has already happened.

---

## What the Scholar actually needs

This list drives every decision in the rest of Part VII, so let's state it plainly.

**Sufficiency.** The answer must be *present* in what we retrieved. For the SSO question, that
means both page 212 and page 1,140. This is recall (Chapter 19), and it is the hard ceiling on
everything that follows.

**Enough context to interpret it.** Each chunk must make sense on its own. The chunk *"It also
records every export and permission change"* is useless if the Scholar cannot tell what "It" is.
Here, "It" means the Security add-on's audit log. This is why Chapter 50 spends so
long on chunking, and why Chapter 51 is about metadata.

**Not too much else.** Every irrelevant chunk is a distractor and a cost.

**Attribution.** The Scholar must be able to say *which* page each claim came from. If our chunks
do not carry stable IDs and titles, we cannot produce citations. Without citations nobody can
check the answer, and for most business uses that makes the system unusable, however good it is.

**An honest empty set.** Sometimes nothing relevant exists. A customer asks, *"Does Acme offer pet
insurance?"* Acme sells software, so no page mentions pets, but the Librarian still returns the
five least unrelated pages, about billing and refunds (Chapter 3). The Scholar must be told
"nothing relevant was found" instead of being handed those five pages. This is the single most
important reliability property, and Chapter 55 shows how to detect it.

---

## RAG vs long context

| | RAG | Long context (put everything in) |
|---|---|---|
| What goes in the prompt | A few relevant chunks | Everything, or one whole document |
| Collection size it can handle | Any size | Up to one context window |
| Cost per question | Low (thousands of tokens) | High (hundreds of thousands to a million) |
| Time before the answer starts | Short | Long |
| Main risk | Retrieval misses a needed page | Needed facts get lost in the middle |
| Citations, freshness, permissions | Built into the design | Hard to add |
| Best for | Lookups over a large, changing collection | Deep reading of one long thing |

---

## When to use which one

We must use **RAG** when the collection is bigger than a context window, when it changes often,
when different users may see different pages, or when answers need citations. Acme's support
assistant is all four.

We must use **long context** when the relevant material is one large thing and we cannot tell in
advance which part matters. Examples are analysing one long contract end to end, reasoning across
a whole codebase, or summarising a full report.

Many strong systems use both. The right framing is not RAG *versus* long context. It is
**retrieval selects, long context absorbs.** Because the window is large, we can retrieve
generously: twenty full pages instead of five small fragments.

---

### Under the hood

The whole thing, honestly:

```python
# embedder, index, llm and RELEVANCE_FLOOR are set up elsewhere in the application.
def rag(question, k=5):
    q_vec = embedder.embed_query(question)
    hits = index.search(q_vec, k=k)

    if not hits or hits[0].score < RELEVANCE_FLOOR:     # calibrated, Ch. 55
        return "I don't have information about that.", []

    context = "\n\n".join(
        f"[{i+1}] (source: {h.title}, {h.url})\n{h.text}" for i, h in enumerate(hits))

    prompt = f"""Answer the question using ONLY the sources below.
Cite sources as [1], [2]. If the sources do not contain the answer, say so.

Sources:
{context}

Question: {question}"""
    return llm(prompt), hits
```

In simple words, this is Phase 2 from above. We embed the question, fetch the five closest chunks,
give up politely if even the best one is a poor match, number the chunks, wrap them in
instructions, and send everything to the LLM. For our SSO question, pages 212 and 1,140 rank
first and second, so they become sources [1] and [2]. The other three are near misses that the
answer does not need to cite.

Three things in that snippet do real work, and they are often left out:

- **The relevance floor**, which turns "always returns something" into "says I don't know". This
  is what saves the pet insurance question. Calibrate it on your own data, because it is
  model-specific.
- **The numbered source labels**, which make citations possible at all.
- **"ONLY the sources below"**, the instruction that keeps the answer tied to our pages instead of
  blending them with whatever the model remembers from training.

---

### What people get wrong

**Treating RAG as a library call.** Frameworks give you the prototype pipeline in about ten lines.
The ten lines are the prototype. The production system is the five refinements.

**Optimising generation before retrieval.** If the answer was not retrieved, no prompt
engineering recovers it. **Measure recall first.** If recall@5 is 0.6, your ceiling is 0.6 and
prompt work is wasted.

**No citations.** Unverifiable answers are unusable in most business settings, and citations are
also your cheapest debugging tool.

**Never saying "I don't know."** Without a relevance floor, an out-of-scope question produces a
confident fabrication built from irrelevant context. This is the failure that destroys trust in a
deployed system.

**Evaluating by vibes.** Ten demo questions that you chose yourself. Build the eval set
(Chapter 54).

---

### Ninja notes

**Retrieval is becoming a tool call rather than a pipeline stage**, and this is the most
significant shift in the field since RAG was named. A tool call is the model itself asking our
system to run a search. In the classic design, retrieval happens
once, before generation. In the agentic design, the model decides *when* to search, *what* to
search for, and whether to search *again* after reading what came back.

That changes what your retrieval system must provide. A single-shot pipeline needs high recall at
k=5. An agentic system needs **fast, cheap, repeatable** search, because it may issue ten queries
to answer one question. It also needs good behaviour on the empty set, since the model needs a
reliable signal to decide whether to reformulate. Chapter 56 covers this properly.

**The other shift worth preparing for is retrieval that returns images.** With ColPali
(Chapter 45) and vision-capable models, the retrieved artefact can be a page image. It goes
straight to the Scholar, with no text extraction anywhere in the loop. Your "context" becomes a
set of pictures. Chapter 61 builds that system end to end.

---

### Key takeaways

- **RAG = Retrieval + Augmented + Generation.** The Librarian finds the pages, we add them to the
  prompt, and the Scholar answers from them with citations.
- **LLM = Large Language Model**, our Scholar. The **prompt** is the text we hand it, the
  **context window** is the most it can read at once, and a **hallucination** is a confident
  answer no source supports.
- Alone, the Scholar says SSO is Enterprise-only, which is wrong. With pages 212 and 1,140, it
  says yes, with the Security add-on for teams under 50 seats, which is right.
- Long context does not remove the need for retrieval. Collections are bigger than windows,
  irrelevant context distracts, tokens cost money, and freshness and permissions need filtering.
- **Retrieval selects, long context absorbs.** Retrieve generously.
- The prototype is four indexing steps and six answering steps. Production adds query
  understanding, hybrid retrieval, reranking, context assembly and continuous evaluation.
- The Scholar needs sufficiency, interpretable chunks, little noise, attribution, and an honest
  empty set.
- Measure retrieval recall before touching the prompt.

### What's next

The prototype cuts every page into chunks without much thought. [Chapter 50](./50-chunking.md)
shows why that one decision shapes retrieval quality more than the choice of embedding model does.

We now know who the Scholar is, why it needs the Librarian, how the two phases of RAG fit
together, and where long context fits alongside it.
