# Thinking in Vectors

### From "what is a vector?" to shipping retrieval over a hundred million chunks.

---

This is a book written as a series of blog posts.

There are sixty-six chapters. Most are a 10–15 minute read, and each one can stand alone. They are
arranged so that if you start at Chapter 1 knowing nothing and finish at Chapter 66, you will
understand modern vector search the way a practitioner does: not as a list of library names, but
as a small set of ideas that fit together.

You will not find "and then the magic happens" anywhere in here. Every idea is built from
something you already understand. It is explained with an everyday example, and then pushed until
the example breaks. Knowing *where the example breaks* is what separates someone who has read
about vectors from someone who can fix a broken search system at 2 a.m.

---

## Who this is for

**The novice.** You have heard "embeddings" and "vector database" and nodded along. You do not
know what a dot product is, or you knew once and it has gone. Start at Chapter 1 and skip nothing.
Every term is defined the first time it appears, and every formula comes with plain words and a
small worked example you can check with a calculator.

**The builder.** You have shipped a RAG prototype. It demos well, fails in production, and you do
not know why. Start at Chapter 17, read Chapters 40–41 (hybrid search and rerankers), then Part
VII (RAG) and Part VIII (Scale). Come back to Parts II and IV when something breaks.

**The ninja-in-training.** You know HNSW and you have tuned `efSearch`. You want ColBERT, ColPali,
MUVERA, learned sparse retrieval, and the honest economics of billion-scale systems. Read Parts V,
VI, VIII and IX.

---

## The cast

This book explains its ideas with five characters and one running example. Meet them once, in
Chapter 1, and every later chapter gets easier.

| Character | What it really is | First appears |
|---|---|---|
| **The Great Library** | Your corpus, every document you want to search | Ch. 1 |
| **The Map Room** | The embedding space, where meaning becomes location | Ch. 1 |
| **The Card Catalog** | The index, the structure that makes search fast | Ch. 1 |
| **The Librarian** | The retriever, which fetches candidates | Ch. 1 |
| **The Scholar** | The LLM, which reads what the Librarian brings and answers | Ch. 1 |

The running example is **Acme's support knowledge base**: 40,000 pages of help articles, pricing
pages, release notes and scanned contracts. We keep asking it one question:

> *"Does the Pro plan include single sign-on?"*

The answer is split across two pages. That is exactly the kind of question that breaks naive
retrieval. We watch it fail in many different ways, and fix it one chapter at a time.

Globex, one of Acme's largest customers, first appears in Chapter 33 and returns in many chapters
of Parts VI–IX. It has been on Pro with 120 seats since June 2026 (previously Basic with 30), and
its scanned contract sits on pages 3,507–3,508.

The whole book, in one sentence: *we teach a Librarian to find the right books in a Library too
big to read, by giving every book a place in a Map Room and building a Card Catalog over those
places, so that the Scholar can answer from the right pages.*

---

## How every chapter works

- **"The one-paragraph version"** opens every chapter. If that paragraph is all you need, close
  the tab. No hard feelings.
- **"We will cover the following"** comes next, so you always know where you are.
- **The teaching sections** follow one rhythm: what the idea is, why we need it, how it works step
  by step, a worked example where the old way gets the answer wrong and the new way gets it right,
  and when to use it.
- **"Under the hood"** has the code. It is Python with NumPy, written to be *read*, not pasted.
  Where a real library matters, it is named.
- **"What people get wrong"** exists because the fastest way to learn a field is to inherit other
  people's scar tissue.
- **"Ninja notes"** are the parts that only matter once you have real traffic. Novices can skip
  them on the first pass and should come back on the second.
- **"Key takeaways" and "What's next"** close every chapter. Read only those and you still get a
  correct, shallow map of the whole book.
- **Maths** appears, but never without a plain-English translation and a small worked example
  right beside it. Skip every equation and you will still finish the book understanding vector
  search. The equations are there to make that understanding *precise*.

**Note:** Two chapters skip parts of this template on purpose. Chapter 65 is an essay about where
the field is heading. Chapter 66 is a reference: the book's key decision trees, formulas and
defaults in one place.

---

## The map

**Part I — Foundations: what a vector actually is** (Ch. 1–7)
The leap from things to numbers, the geometry of meaning, similarity, normalization, high
dimensions, and dense versus sparse.

**Part II — Where embeddings come from** (Ch. 8–16)
TF-IDF and BM25, word2vec, transformers, pooling, contrastive training, bi-encoders vs
cross-encoders, asymmetry, Matryoshka, and how to choose a model.

**Part III — Finding neighbours** (Ch. 17–20)
Brute force, the recall–latency–memory triangle, how retrieval quality is measured, and the family
tree of approximate nearest neighbour algorithms.

**Part IV — Index structures** (Ch. 21–33)
LSH, trees, IVF, product quantization, scalar and binary quantization, TurboQuant, HNSW in three
chapters plus one on production, DiskANN, and filtered search.

**Part V — Beyond one vector** (Ch. 34–41)
The single-vector bottleneck, ColBERT and late interaction in three chapters, MUVERA, SPLADE,
hybrid fusion, and rerankers.

**Part VI — Vectors beyond text** (Ch. 42–48)
CLIP, SigLIP, why OCR pipelines quietly lose, ColPali in three chapters, and a tour of audio, video,
code and graph embeddings.

**Part VII — RAG** (Ch. 49–56)
First principles, chunking, metadata, query understanding, context assembly, evaluation, a field
guide to failure, and agentic retrieval.

**Part VIII — Scale and production** (Ch. 57–63)
Architecture for millions of documents, sharding, freshness, cost engineering, image-heavy RAG end
to end, multi-tenancy and security, and model migration.

**Part IX — Ninja tier** (Ch. 64–66)
Vectors as agent memory, the frontier, and the field manual.

Full chapter list with one-line summaries: [OUTLINE.md](./OUTLINE.md)

---

## A promise and a warning

**The promise.** Nothing in this book is hand-waved. When we say HNSW is fast, you will see the
mechanism that makes it fast and the memory bill that pays for it. When we say ColPali beats an
OCR pipeline, you will see exactly what OCR throws away.

**The warning.** Vector search is a field where the shiny thing is often not the right thing. A
well-tuned BM25 index beats a badly chunked embedding pipeline more often than the industry
admits. This book will tell you when *not* to use the technique it just spent three chapters
teaching you.

Begin: [Chapter 1 — What Is a Vector, Really?](./chapters/01-what-is-a-vector.md)
