---
title: "Dense vs Sparse Vectors"
chapter: 7
part: "Part I — Foundations: What a Vector Actually Is"
slug: dense-vs-sparse
readingTime: "11 min"
summary: "Two completely different shapes of vector, two completely different index structures, two different failure modes — and the reason serious systems run both."
tags: [foundations, sparse, dense, inverted-index, hybrid]
prev: 06-high-dimensions
next: 08-before-embeddings
---

# Dense vs Sparse Vectors

**The one-paragraph version.** A **dense** vector has a few hundred to a few thousand dimensions,
all non-zero, and none of them meaningful on its own. It captures *meaning*. A **sparse** vector
has tens of thousands of dimensions, almost all zero, and each one stands for a specific word. It
captures *exact terms*. They fail in opposite directions: dense search misses the error code, and
sparse search misses the paraphrase. Production systems run both and merge the results.

In this chapter, we will learn the two shapes a vector can take. We will watch each shape get one
of Acme's questions wrong while the other gets it right. We will see the inverted index, the
structure that makes sparse search fast. Then we will compare the two side by side and decide
when to use which one.

We will cover the following:

- What are dense and sparse vectors
- Two ways to describe the same page
- An example: two questions, two winners
- How sparse search works: the inverted index
- Dense vs sparse
- When to use which one

---

## What are dense and sparse vectors

Every embedding in Chapters 2 to 6 has been one shape of vector. Now we meet the other shape.

**Dense vector = a few hundred to a few thousand numbers, nearly all non-zero, learned by a model.**

**Sparse vector = one position for every word in a vocabulary, with almost every position zero.**

Before we go on, one small term. Search systems and models do not read whole words exactly. They
split text into **tokens**, which are words or pieces of words. For this chapter, it is fine to
read "token" as "word". Chapter 10 explains tokens fully.

Let's see the difference with numbers. Suppose our vocabulary has 30,000 words. An Acme help page
uses 300 distinct words. Its sparse vector has 30,000 positions: 300 of them non-zero and 29,700
of them zero. That is 1% filled. Its dense vector has 768 positions, and essentially all 768 hold
a non-zero number.

In simple words, a dense vector is a short list where every number is busy. A sparse vector is a
very long list where almost every number is zero.

---

## Two ways to describe the same page

Take Acme's troubleshooting page for error code `SSO-4012`, *"SAML assertion expired"*. SAML is
one of the standards single sign-on runs on. The page explains that the login message from the
company's identity system was too old by the time Acme checked it, and that the usual cause is a
wrong clock on that system.

**The sparse description** is an index card listing every word on the page and how often it
appears: `sso-4012: 3`, `saml: 5`, `assertion: 4`, `expired: 2`, `clock: 2`, `the: 41`...
Most of the 30,000 words in the vocabulary appear zero times, so the card is mostly blank. But the
words that *are* there are precise and human-readable.

**The dense description** is a few hundred scores, something like `login-troubleshooting: 0.91`,
`enterprise-security: 0.84`, `error-message: 0.88`... In a real model no score has a name, and
nobody could tell you what score #57 measures. But together, the scores place the page among
pages like it.

Think of it like the two ways a book can be described:

- The sparse card is like the index at the back of a book. It lists the exact words and where
  they appear.
- The dense card is like a reviewer's summary. It says what the book is about, in no particular
  words.
- The index finds exact words. The summary finds the subject.

Now, the question is, which description finds this page, and when?

---

## An example: two questions, two winners

Let's hand the Librarian two questions. Each one is a twin of our running question.

**Question 1, the exact-token twin:** *"What does error SSO-4012 mean?"*

Let's first see what dense search does. The dense model breaks `SSO-4012` into fragments that
carry almost no meaning. Acme also has a page for error `SSO-4021`, "SAML signature invalid".
To the dense model the two codes look nearly identical: the same letters, the same digits in a
different order. The two pages also share a topic, login errors. So their dots sit almost on
top of each other in the Map Room.

The Librarian is as likely to bring back the `SSO-4021` page first as the right one, and this time
it does. The Scholar explains how to replace the SAML certificate. The answer is wrong.

Now sparse search. The token `sso-4012` appears on exactly one page in the whole Library. That
page's card has it. The `SSO-4021` page's card does not. Sparse search returns the right page
immediately. The Scholar explains that the login message expired and says to check the identity
system's clock. The answer is correct.

**Question 2, the paraphrase twin:** *"Can our team log in with our company accounts?"*

This time let's start with sparse search. The answer is on page 212, *"Single sign-on is included
on the Pro plan."* Page 212's card has `single`, `sign-on`, `included`, `pro` and `plan`. The
question's words are `team`, `log`, `company` and `accounts`. The overlap is zero, so page 212
scores zero. Instead, sparse search ranks *"Closing a company account"* first, because it shares
`team`, `company` and `accounts`. The answer is wrong.

Now dense search. As we saw in Chapter 2, the model learned that "log in with our company
accounts" and "single sign-on" serve the same purpose. Their dots are neighbours. The Librarian
brings back page 212, and the Scholar explains that single sign-on is included on the Pro plan.
Dense search found the right page. The answer is still incomplete, because page 1,140's catch
for teams under 50 seats did not come back. Chapter 40 returns to this question.

| Question | Sparse search | Dense search |
|---|---|---|
| "What does error SSO-4012 mean?" | Right page | Wrong page (`SSO-4021`) |
| "Can our team log in with our company accounts?" | Wrong page | Right page (212) |

In simple words, sparse search matches the characters on the page. Dense search matches what the
page is about. Each one wins exactly where the other one loses.

The same split shows up well beyond these two questions:

- **Every identifier behaves like the error code.** Invoice numbers such as `#A-2291-B`, product
  SKUs and part numbers all get an exact hit from sparse search. Dense search often confuses them
  with look-alikes. Identifiers are the single clearest case where dense retrieval fails, and most
  RAG systems over business documents hit this soon after launch.
- **Other languages behave like the paraphrase.** A Spanish-speaking admin asks, *"¿Nuestro equipo
  puede iniciar sesión con las cuentas de la empresa?"* No Spanish word appears on page 212, so
  sparse search finds nothing. Dense search with a multilingual model (one trained on many
  languages) finds page 212.

---

## How sparse search works: the inverted index

Why do sparse vectors need their own index? Let's do the arithmetic.

We cannot store a 30,000-dimensional vector as a plain array. For Acme's 40,000 pages, that is
40,000 × 30,000 × 4 bytes = 4.8 GB, almost entirely zeros. For a million documents it would be
120 GB.

So sparse vectors are stored as a list of `(position, value)` pairs, keeping only the non-zeros.
Our page with 300 distinct words becomes 300 pairs. At 8 bytes per pair, that is 2,400 bytes
instead of 120,000.

Then they are searched with a structure that has been refined for fifty years.

**Inverted index = a table from each word to the list of documents that contain it.**

It flips the usual relationship. Instead of storing *page → its words*, we store *word → the
pages containing it*:

```
"sso-4012"  → [page 17,450]
"saml"      → [page 88, page 17,450, page 17,452, ... ]
"company"   → [page 88, page 907, page 17,450, ... ]
```

Each of those lists is a **posting list**: the list of documents that contain that term.

Here is how a search runs on it.

**Step 1:** Split the question into tokens: `what`, `does`, `error`, `sso-4012`, `mean`.

**Step 2:** For each token, fetch its posting list.

**Step 3:** Add up a score for every page that appears in those lists. Rare tokens get large
weights and common ones get tiny weights (Chapter 8 shows how). Pages in none of the lists are
never touched.

**Step 4:** Return the top-k pages.

The search touches only the posting lists of the question's tokens, never the whole Library. The
rare token `sso-4012` has a posting list of length one, and it carries most of the score. With a
query built on a rare term, we might examine three documents out of a hundred million. No ANN
index comes close to that efficiency for such selective queries. And it is *exact*, not
approximate.

In simple words, the inverted index lets sparse search skip every page that shares no words with
the question.

This is why "vector search replaced keyword search" was never true. Lucene-family engines
(Elasticsearch, OpenSearch, Solr) and their independent cousins (Tantivy, Vespa) run enormous
production workloads on this structure, and they are extremely good at it. (Lucene is the
open-source search library at the heart of the first three.)

---

## Dense vs sparse

| | Sparse | Dense |
|---|---|---|
| Dimensions | 30,000 – 1,000,000+ (vocabulary size) | 128 – 4,096 |
| Non-zero entries | Tens to hundreds | All of them |
| What a dimension means | One specific term | Nothing nameable |
| Built by | Counting + weighting (or a neural net, Ch. 39) | A neural network |
| Index structure | Inverted index | ANN index (HNSW, IVF) |
| Storage per doc | Proportional to unique terms | Fixed: `d × 4` bytes |
| Finds exact terms (`SSO-4012`) | Yes, perfectly | Unreliably |
| Finds paraphrases ("log in with company accounts") | No | Yes |
| Handles typos | No | Somewhat |
| Cross-lingual | No | Yes, with the right model |
| Out-of-domain jargon | Works fine (it is just a token) | Often fails |
| Explainable | Yes ("matched on `sso-4012`") | No |
| Needs training data | No (BM25) | Yes |

Read that table twice. The two columns are almost perfect complements. The things sparse is bad
at are exactly the things dense is good at, and the other way round.

That symmetry is why Chapter 40 on hybrid search, which runs both and merges the results,
describes what is probably the highest return-on-effort technique in the whole book. (HNSW and IVF
are two kinds of Card Catalog for dense vectors, covered in Part IV. BM25 is the classic formula
for scoring sparse matches, covered in Chapter 8.)

---

## When to use which one

> **The heuristic.** If a human would find the answer with Ctrl-F, use sparse. If a human would
> need to understand the question, use dense. Most real query streams contain both kinds, in
> proportions you should actually measure.

We must use **sparse** search when questions contain exact identifiers: error codes like
`SSO-4012`, invoice numbers, SKUs, and product names or jargon the model never saw.

We must use **dense** search when users describe what they want in their own words, like "log in
with our company accounts", or ask in another language.

Acme's users do both, often within the same hour. Many strong systems use both: run the two
searches side by side and merge the two ranked lists (Chapter 40).

---

### Under the hood

Sparse vectors in code are dictionaries, and scoring is an intersection:

```python
page = {"sso-4012": 3.9, "saml": 2.1, "expired": 1.4}   # term → weight
qry  = {"sso-4012": 1.0, "fix": 1.0}

score = sum(w * page.get(t, 0.0) for t, w in qry.items())   # → 3.9
```

Means, we multiply the weights of the terms the question and the page share, and add them up.
Look at what happened to `"fix"`: it contributed exactly zero, because the page's card has no
entry for it. Sparse scoring only sums over *shared* terms. A dense dot product, by contrast, sums
over all 768 dimensions, and every one contributes something. That is the mathematical shape of
the difference between exact matching and fuzzy matching. (Chapter 8 shows where weights like 3.9
come from.)

Modern sparse vectors need not be produced by counting. **SPLADE** (Chapter 39) uses a transformer,
the kind of neural network behind modern language models (Chapter 10), to *learn* sparse term
weights. That includes terms that never appear in the document. It can put weight on `login` for
a page that only ever says "SAML assertion expired". That blend of neural understanding with
inverted-index efficiency is one of the most elegant ideas in retrieval. It is why the sparse
column of that table is less fixed than it looks.

---

### What people get wrong

**"Sparse is the old way."** BM25 from 1994 remains a brutally strong baseline. A dense-only system
that has not been compared against BM25 on its own data has not been evaluated.

**"I'll just use dense and add more context."** More context does not create information the
embedding discarded. If the SKU is not in the vector, no `top_k` will find it.

**"Hybrid is complicated."** It is a few dozen lines: run both searches, merge with Reciprocal
Rank Fusion (a simple way to combine two ranked lists), return. Chapter 40 gives the code.

**"Sparse vectors need a vector database."** They need an inverted index. Some vector databases
now support both in one system, which is convenient, but the underlying structures remain
completely different.

---

### Ninja notes

There is a third shape worth knowing about now, because Part V is built on it: **multi-vector**
representations. Instead of one dense vector per document, store one per *token*. A 200-word
passage becomes roughly 250 vectors, one per token, of 128 dimensions each.

This buys back much of what single-vector compression destroys. Individual terms keep their own
representation, so exact-ish matching and semantic matching happen in the same mechanism. ColBERT
(Chapter 35) does this for text. ColPali (Chapter 45), one of the page-image model families of
Part VI, does it for page images.

The cost is hundreds of times more vectors to store and score, about 250 instead of 1 for that
passage. In bytes, 250 × 128 × 4 = 128,000 instead of 3,072, roughly 40× before compression.
Part V uses a shorter 200-token passage as its standard example, which works out to about 30×.
Either way, that is why Chapters 37 and 38 are entirely about making it affordable.

So the real taxonomy is three-way: sparse (one dimension per term), dense (one vector per
document), multi-vector (one vector per token). Each trades storage for precision differently,
and a sophisticated system may use all three.

---

### Key takeaways

- **Dense vector = a few hundred to a few thousand learned numbers, all non-zero. Sparse vector =
  one position per word, almost all zero.**
- Counting-based sparse (BM25) is exact, explainable and needs no training. Dense is semantic and
  fuzzy.
- They fail in opposite directions: identifiers like `SSO-4012` defeat dense search, paraphrases
  like "log in with our company accounts" defeat sparse search.
- Sparse uses inverted indexes. Dense uses ANN indexes. Different structures entirely.
- Run both and merge. It is the cheapest large quality win available.
- Multi-vector is the third shape, and Part V is devoted to it.

### What's next

Part I is done. We know what a vector is, how to measure similarity, and the two shapes vectors
come in. Next, [Part II](./08-before-embeddings.md) opens the factory where embeddings actually
come from, starting with the counting-based ancestors that still beat neural models on a
surprising number of queries.

Two shapes, two kinds of index, two opposite failures, and one lesson: the strongest systems run
both.
