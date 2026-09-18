---
title: "The Frontier"
chapter: 65
part: "Part IX — Ninja Tier"
slug: the-frontier
readingTime: "8 min"
summary: "Where vector search appears to be heading, separated into trends with strong evidence, open questions worth watching, and the principles that will outlast all of them."
tags: [future, trends, research, multi-vector, compression, reasoning-retrieval]
prev: 64-agent-memory
next: 66-field-manual
---

# The Frontier

**The one-paragraph version.** What we can see of vector search's future sorts into three piles.
The first pile holds trends with enough evidence to plan around. They include affordable
multi-vector retrieval, compression that needs no training, vision-first document retrieval, and
retrieval as a tool that agents call again and again. Storage moving to cheaper tiers, and long
context changing what retrieval selects, belong there too. The second pile holds genuinely open questions, such as whether retrievers learn to
reason and whether generative models replace indexes. The third pile holds a handful of
principles from this book that have held for decades and will keep holding.

In this chapter, we will look at where the field appears to be heading and separate the evidence
from the guesses. Then we will collect the ideas from this book that are safe to build on.

We will cover the following:

- How to read this chapter
- Trends with strong evidence
- Open questions worth watching
- What will not change

---

## How to read this chapter

Predictions in a fast-moving field age badly. So this chapter keeps a strict line between **what
the evidence supports** and **what is worth watching**, and it is deliberately modest about the
second. It makes no promises about dates.

Think of it like a weather forecast.

- Tomorrow's forecast is usually right. Those are the trends with strong evidence.
- Next month's forecast is an educated guess. Those are the open questions.
- The climate barely moves from year to year. Those are the principles that will not change.

In simple words, plan around the first pile, keep an eye on the second, and build everything on
the third.

Each trend below ends with a **planning implication**: one thing worth doing now, whichever way
the details turn out.

---

## Trends with strong evidence

### 1. Multi-vector is becoming affordable

Part V's big objection to multi-vector retrieval was cost. One 768-d float32 vector for a passage
is 3,072 bytes. ColBERT keeps one 128-d vector per token instead, which is roughly 30× the bytes
and 200× the vector count before compression (Chapter 35).

That objection is eroding from several directions at once:

- **Token pooling** cuts vectors per document by 2–3× for a small quality cost (Chapter 47).
- **Smaller per-token dimensions**, such as 64 or 32, cost only modest quality (Chapter 35).
- **Low-bit quantization**, through residual compression (Chapter 37) or TurboQuant (Chapter 27),
  brings each 128-d token vector from 512 bytes down to about 20–36 bytes.
- **MUVERA** removes the need for special serving infrastructure (Chapter 38).

Put those together and something striking happens. A token vector at 32 dimensions with 2 bits per
number is 8 bytes. A 200-token passage is then 1,600 bytes, about *half* of one uncompressed 768-d
float32 vector. Chapter 35 noted that this crossover has been reached in research settings. It is
not yet routine in production.

In simple words, "many vectors per page" is heading toward the price of "one vector per page".
That matters for a page like Acme's page 1,140. One sentence on it carries the catch about seats,
and a single vector can blur that sentence away (Chapter 34). Many vectors keep it.

**Planning implication:** keep token-level output available from the encoder path, and prefer
platforms with native multi-vector or FDE support. Then adopting multi-vector later does not mean
rebuilding ingestion.

### 2. Randomised constructions that need no training

A **data-oblivious** method is built from randomness, not from our data. It needs no training, so
there is nothing to refit when the data changes.

A striking pattern in recent results: **MUVERA and TurboQuant are randomised, training-free
constructions with guarantees.** LSH was an early member of this kind, and it lives on inside
MUVERA, whose regions come from SimHash (Chapter 38). As a stand-alone index it lost on dense
vectors (Chapter 21), though its MinHash cousin still wins at deduplication. In their published
experiments, MUVERA and TurboQuant each compete with or beat learned alternatives. Chapter 27's
advice still stands: benchmark on your own data before switching.

The operational argument is strong. There are no codebooks to fit, nothing goes stale, nothing
needs retraining when the corpus drifts, and new vectors can stream in at any time.

Here is why that matters. Suppose Acme trained IVF-PQ codebooks on last year's pages, and then
launched a new product with new vocabulary. The codebooks now describe a Map Room that has moved
(Chapter 63's corpus drift). A random rotation has no such memory of last year.

**Planning implication:** when a learned component and a data-oblivious one have similar quality,
the data-oblivious one usually has the lower lifetime cost.

### 3. Vision-first document retrieval

The ColPali line of models (Chapters 45–47) moved document retrieval from "convert to text, then
search" to "search the page". Vision-language models keep getting better at reading dense pages,
and retrievers built on them inherit the gains.

Page 3,507, the scanned Acme–Globex contract, is the example. A text pipeline lost its pricing table.
An image pipeline found it (Chapter 61).

**Planning implication:** keep page images for every document you ingest (Chapter 61). They are
cheap to store, and they are the input to whatever the next retriever turns out to be.

### 4. Retrieval as a tool

Agentic retrieval (Chapter 56) changes what retrieval systems are built for. An agent does not
make one big high-recall call per question. It makes many cheap, fast, filtered calls per task,
and it needs sensible behaviour when a call finds nothing.

When Dana asks Acme's assistant about SSO (Chapter 64), the agent might search the plan pages,
then Globex's contract, then the release notes. Three calls, each small, each scoped.

**Planning implication:** invest in retrieval latency, clean tool interfaces and structured
metadata filters. Those are what agents consume.

### 5. Storage moves down the cost ladder

RAM costs tens of times more per gigabyte than SSD, and SSD a few times more than object storage
(Chapter 60). Disk-based graphs (Chapter 32) and object-storage-backed indexes trade latency for
large cost cuts. At 100 million vectors, DiskANN needs about 5 GB of RAM plus SSD and answers in a
few milliseconds (Chapter 32). Object-storage-backed indexes cut cost further, at tens to hundreds
of milliseconds. For archives and rarely read content, that is acceptable, and the savings are
compelling.

**Planning implication:** tier the corpus by how often it is read. Do not pay RAM prices for pages
nobody queries.

### 6. Retrieval selects, long context absorbs

Longer context windows have not removed the need for retrieval (Chapter 49). They have changed its
role. Retrieval increasingly selects *whole relevant documents* for the Scholar to read, rather
than small fragments for the Scholar to stitch together.

For our running question, that could mean handing the Scholar the whole Pro plan page and the
whole Security add-on page, rather than two 400-token chunks.

**Planning implication:** retrieve at document or section granularity where the budget allows,
and revisit chunk sizes as the price of context changes.

---

## Open questions worth watching

These are real research directions. None of them is yet something to build a production system
around.

### Retrieval that reasons

Most retrievers, including everything in this book, match a query to documents by some kind of
semantic relatedness. Some questions need *reasoning* before we even know which document is
relevant.

Suppose an Acme admin asks why their logins broke the day after they renewed a certificate. The
page that answers it may be a release note about signing keys. It shares almost no words, and
barely a topic, with the question. Benchmarks built for reasoning-intensive retrieval, such as
BRIGHT, show that standard retrievers struggle with exactly this.

Approaches under exploration include retrievers trained on reasoning traces, LLMs that write out
intermediate reasoning before retrieving (a relative of HyDE, Chapter 52), and agentic loops
(Chapter 56). Which one wins is unresolved. So is whether this stays the agent's job rather than
the retriever's.

### Generative retrieval

One research direction trains a model to *generate document identifiers directly*. The model
itself becomes the index. In simple words, instead of looking up page 212, the model has memorised
that the answer lives at "page 212".

It is elegant, and it faces hard practical problems. Adding documents means updating model
weights. Deleting documents is difficult. Scaling to large, fast-changing corpora remains
challenging. It is worth understanding, but not yet worth planning around.

### Unified any-to-any spaces

Models that align many modalities through one shared anchor (Chapter 48) point to a future where
one index serves text, images, audio and structured data together. Today, quality across all the
pairs is uneven.

### Learned indexes and learned routing

This means training a small model to pick which partitions to search, instead of a hand-written
rule such as "nearest centroid". It is not a fifth index idea. Chapter 20 places it in the
partition or cluster family, with a trained router in place of computed signposts. It is
promising in specific settings. It is not yet a general replacement for HNSW or IVF.

---

## What will not change

These principles have held from BM25 through HNSW to ColPali. They are the parts of this book most
worth remembering.

**1. Every representation compresses, and we should know what it discards.** We met it three
times: the embedding (Chapter 2), the single-vector page (Chapter 34) and OCR text (Chapter 44).

**2. Coarse-then-refine beats any single stage.** Cheap and broad first, exact and narrow last.
It is IVF, rescoring, cascades, PLAID, MUVERA, DiskANN and the reranker. One idea, many forms.
Think of it like finding a book in the Great Library. Walking to the right wing is the coarse
stage. Scanning the right shelf is the shortlist. Reading the few candidate books closely is the
exact stage. Nobody reads every book.

**3. Exact and semantic matching fail in opposite directions.** Keyword search finds `SSO-4012`
and misses *"log in with our company accounts"*. Dense search does the reverse. Run both.

**4. Recall at the first stage is the ceiling on everything downstream.** If page 1,140 never
enters the candidate list, no reranker and no Scholar can bring it back.

**5. A model encodes the similarity it was trained on, not the one we have in mind.** "Similar"
to the model may mean "same topic" when we needed "same answer".

**6. Measure on your own data, by segment, with confidence intervals.** Benchmarks measure someone
else's problem.

**7. The index is a cache. Keep the source of truth somewhere else.** Then every model change,
parameter change and bug is a rebuild, not a disaster.

**8. The shiny thing is often not the right thing.** A well-tuned BM25 with good chunking and a
reranker beats a poorly built system that uses every technique in Part V.

---

### Key takeaways

- **The frontier = trends with strong evidence + open questions worth watching + principles that
  will outlast both.** Plan around the first, watch the second, build on the third.
- Strong-evidence trends: affordable multi-vector, training-free compression with guarantees,
  vision-first document retrieval, retrieval as an agent tool, cheaper storage tiers, and
  retrieval feeding long context.
- MUVERA and TurboQuant are the randomised, training-free constructions to watch. LSH lives on
  inside MUVERA, but as a stand-alone index it lost on dense vectors.
- Open questions: reasoning-intensive retrieval, generative retrieval, unified modality spaces and
  learned routing.
- Plan around the trends: keep token-level output, page images, rich metadata and a source of
  truth. Tier storage. Invest in retrieval latency.
- The durable principles are compression awareness, coarse-then-refine, hybrid matching,
  first-stage recall, training-defined similarity, measurement on your own data, the index as a
  cache, and preferring a well-built simple system over the shiny one.

### What's next

[Chapter 66](./66-field-manual.md) collects the book's key decision trees, formulas and defaults
in one place, to keep open while building.

Now we have seen which bets look safe, which questions are still open, and which ideas from this
book will still be true whatever comes next.
