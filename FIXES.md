# FIXES.md — executable fix spec for "Thinking in Vectors"

> **Status: APPLIED on 2026-09-16.** Every item below has been applied to all 66 chapters, followed by a
> full consistency pass. This file is kept as a record of what changed and why. Where the finished book
> deliberately differs from this spec, the book is right and this file is not. Those cases are:
>
> - **Running scenario.** Page 1,140 is the *Security add-on* page. On Pro, SSO needs the add-on for teams
>   under 50 seats.
> - **Ch 6.** The main text does *not* say "a few hundred dimensions suffice". By the Johnson–Lindenstrauss
>   bound, 40,000 points at ε = 0.2 need about 2,450. The chapter now says the count depends on the
>   number of points and the tolerance, never on the starting dimension.
> - **Ch 27.** "No mainstream index ships TurboQuant" was out of date. Qdrant 1.18 (May 2026) ships a
>   TurboQuant mode built on a Hadamard rotation (qdrant.tech/blog/qdrant-1.18.x). The chapter's reference
>   code uses a dense random rotation, so d stays 768.
> - **Ch 19 and Ch 54.** The standard error at n = 20 is written as ±11 points, with a 95% interval of ±22.
> - **Ch 47.** Encoding a million pages takes 30–140 GPU-hours, not 30–100.
> - **Ch 55.** The failures were renumbered after moving incomplete multi-part answers into retrieval. It is
>   now failure 8, with 1–8 retrieval and 9–12 generation.
> - **Ch 34.** The original's embedding-dilution numbers were not reproducible. They were replaced with a
>   measured experiment.
>
> A pristine copy of the pre-fix book is in `_backup_before_fixes_2026-09-16.tar.gz`.

Written 2026-09-16 after a full top-to-bottom review by eight independent readers (one per Part) plus
mechanical checks. This file supersedes REVIEW.md. It is written for a model (or person) to execute.

**Read this whole file before editing anything.** Part 1 changes the *voice* of every chapter; Parts 2–4 are
specific corrections. Do Part 2 (global) first, then Part 3 (per-chapter corrections), then Part 1 (style
pass), then Part 4 (verify). Doing the style pass last means you are not rewriting text you then correct.

Line numbers were accurate on 2026-09-16 and will drift. **Always locate by the quoted text, not the line.**

---

## Part 1 — The voice: what the Outcome School blog does that this book does not

Reference: https://outcomeschool.com/blog/vectorless-rag (Amit Shekhar, Sept 2026, ~3,800 words).
The author of this book wants "the same magic". Here is what that magic actually consists of, as rules you
can check a paragraph against. The book's current voice is literary and clever (Latin etymology, "pays
rent", "scar tissue", "2 a.m."), with long em-dash sentences and several jargon terms per paragraph. The
blog's voice is plain, patient, and concrete. Keep the book's wit inside **What people get wrong** and
**Ninja notes**. Make every *teaching* section follow the rules below.

### 1.1 The twelve devices (each with the blog's own example)

1. **Open with a contract.** First paragraph: "In this chapter, we will learn about X, a way of … We will
   also see A, B, C, and when to use which one." Then a bullet list **"We will cover the following:"** with
   5–8 items that are the section headings. The book's existing *one-paragraph version* stays; the bullet
   list goes directly under it.

2. **Define prerequisites inline, right before use, never assume.** "Before jumping into Vectorless RAG,
   we must know what an LLM is." Then define it in one sentence. Every term that a Chapter-1 reader has
   not met gets defined the sentence before it is used, in bold at the definition:
   `A **vector** is just a long list of numbers, something like [0.12, -0.88, 0.45, ...].`

3. **Definitions as bold equations.** `**RAG = Retrieval + Augmented + Generation.**` then break each word
   down on its own line. `**LLM = Large Language Model.**` `**Vectorless RAG = RAG without the vectors.**`
   Use this for every named technique: `**HNSW = a graph where every vector links to its nearest
   neighbours, stacked in layers so you can skip.**`

4. **"In simple words, …"** Every technical sentence is immediately followed by a plain restatement.
   "In simple words, we replace mathematics with reading." This is the single most important device. If a
   paragraph has an equation, code, or a term in backticks, the next sentence must start with "In simple
   words," or "Means," or "That is it."

5. **One running scenario per chapter, concrete and small, carried all the way through.** The blog uses
   *one* 300-page insurance policy and *one* question ("Is dental treatment covered in the first year?")
   in the intro, in the Vector RAG walk-through, in Problem 1, and in the worked example. The reader never
   loads a second scenario. Numbers are small and specific: 300 pages, two pages, Chapter 5, 12 months,
   page 154. The book currently switches scenario mid-chapter (coffee → mitochondrion sentence → password
   reset). Pick one per chapter and reuse it in every section including the code block.
   **Recommendation for the whole book:** make the Great Library concrete. It is *Acme's support knowledge
   base: 40,000 pages*. The recurring question is *"Does the Pro plan include single sign-on?"* and its
   trap twin *"Which plans do NOT include single sign-on?"* (negation). The twist maps to *"SSO is
   included on Pro" (page 212) but "On Pro, SSO needs the Security add-on for teams under 50 seats"
   (page 1,140)*. Every chapter that needs a query uses this one.

6. **"Think of it like …" followed by an explicit mapping, one sentence per part.**
   "Think of it like an open book exam. The LLM is the student. Our document is the book. Retrieval is
   the act of finding the correct page." The mapping sentences are mandatory. An analogy without the
   mapping is decoration; with it, it is a definition.

7. **Rhetorical question as the hinge between sections.** "Now, the question is, how do we find the
   correct page? This is exactly where the two approaches differ." "Why do we not simply hand over all the
   300 pages every single time? There are three reasons. First… Second… Third, and the most important
   one…" Use a question to open every problem, and enumerate the answer.

8. **Phases and Steps, one action per line.** "**Phase 1: Preparing the document.** … **Phase 2:
   Answering the question.** **Step 1:** The user asks a question. **Step 2:** We show the LLM only the
   top level…" Every algorithm in the book (IVF build, HNSW insert, HNSW search, PQ encode, PLAID, MUVERA
   FDE, RRF, RAG pipeline, migration) gets this treatment *before* the code block.

9. **Problems as a numbered bold list, each with a mini-example from the running scenario.**
   "**Problem 2: Similar is not the same as correct.** … If we ask 'Which policy does *not* cover dental
   treatment?', the word 'not' is a tiny word. It barely moves the numbers." Three sentences max each.

10. **Before/after with a wrong answer and a right answer.** "Let's first see what a Vector RAG system
    does. … The answer is wrong. Now, let's see what a Vectorless RAG system does. … The answer is
    correct. Problem Solved!" Show the model's reasoning as a quote: `It thinks like this: "The user is
    asking two things. One… Two…"` Every technique chapter should show the baseline getting the running
    question *wrong* and the technique getting it *right*.

11. **One-sentence contrast punchlines, then move on.** "In Vector RAG, we search with mathematics. In
    Vectorless RAG, we search with reading." "That is it. That is the whole idea." "The answer is wrong."
    Sentences average 12–18 words. No semicolons. Em-dashes at most one per section.

12. **Honest close: Advantages, Disadvantages, a comparison table, "When to use which one", then a
    hybrid.** "We must use **Vector RAG** when … We must use **Vectorless RAG** when … Many strong systems
    use both." Ends with a one-sentence recap: "Now we must have understood X, why it exists, how it
    works, and where it fits."

Also: **"we"** not "you" for the shared work ("we cut the document", "we store the vectors"); "you" only
for advice. **"Note:"** callouts to pre-empt a misconception. **"Do not worry, we will learn about each of
them in detail."** where a list of unknowns appears. Section titles are plain nouns or questions ("What is
an LLM", "How X works", "Problems with X", "An example of X", "X vs Y", "When to use which one"), not
metaphors ("Start with a coffee", "The seating chart", "The game of telephone").

### 1.2 Before / after, on a real passage (Chapter 3, "empty regions")

**Before (current text, 03-geometry-of-meaning.md, "Every embedding space has regions that are empty"):**

> Every embedding space has regions that are empty because nothing in your corpus lives there. This has a
> direct, daily consequence. When a user asks a question whose vector lands in an empty region, the search
> still returns results — vector search *always* returns the k nearest neighbours, even when the nearest
> neighbour is a thousand miles away. There is no built-in "I found nothing." The Librarian always comes
> back with an armful of books. This is the single most common cause of confident nonsense in RAG systems:
> the user asks about something you have no documentation for, retrieval returns the five least irrelevant
> documents anyway, and the Scholar dutifully synthesises an answer from them. Chapter 55 gives the fix (a
> distance floor plus a relevance check); for now, just internalise the shape of the failure.

Problems: "k", "RAG", "the Librarian", "the Scholar" all undefined at this point; three scenarios implied
and none shown; no wrong answer shown; the fix is deferred without even naming its shape.

**After:**

> ## Empty parts of the room still return results
>
> Some parts of the Map Room are empty. Nothing in our corpus lives there.
>
> Now, the question is, what happens when a question lands in an empty part?
>
> Let's take an example. Our corpus is Acme's 40,000-page knowledge base. A user asks, "Does Acme offer
> pet insurance?" Acme sells software. No page mentions pets. The question's vector lands in a part of the
> Map Room where no page lives.
>
> The search does not say "nothing found". It cannot. We asked for the 5 nearest pages, and there are
> always 5 nearest pages, even when the nearest one is very far away. So we get back 5 pages about
> billing, refunds, and account deletion.
>
> **Vector search always returns something.** There is no built-in "I found nothing."
>
> Think of it like asking a librarian for a book the library does not own. A good librarian says, "We do
> not have that." This search says nothing, and hands us the five least-unrelated books it could find.
>
> Then the answer-writing model reads those five pages and writes a confident paragraph about Acme's pet
> insurance. The answer is wrong, and nothing in the system knows it is wrong.
>
> **Note:** This is the single most common cause of confidently wrong answers in RAG (retrieval-augmented
> generation: finding the right pages and handing them to a language model, the subject of Part VII). The
> fix has a simple shape: a **distance floor**. If even the nearest page is too far away, say so instead
> of answering. Chapter 55 shows how to pick the floor.

Same facts. Every term defined before use. One scenario. One wrong answer. One named fix. Question hinge,
bold claim, think-of-it-like with mapping, Note callout.

### 1.3 Chapter template (keep the existing skeleton, add the blog's devices)

```
---frontmatter (unchanged)---
# Title

**The one-paragraph version.** (unchanged in role; rewrite in the plain voice: "In this chapter, we will
learn about X …")

We will cover the following:
- (5–8 bullets = the H2 headings below)

## What is X                      ← bold equation definition; "In simple words, …"
## Why we need X                  ← the running scenario; the baseline gets the question wrong
## How X works                    ← Phase 1 / Phase 2, Step 1..N, one action per line
## An example of X                ← same scenario; the LLM's/algorithm's reasoning quoted; "The answer is correct."
## Problems with X / Disadvantages← Problem 1..N, bold one-liners, mini-example each
## X vs Y                         ← comparison table (where a rival exists)
## When to use which one          ← "We must use X when … We must use Y when … Many strong systems use both."

### Under the hood                ← existing; the code implements the Steps above, same scenario
### What people get wrong         ← existing; the book's wit lives here
### Ninja notes                   ← existing; jargon allowed here without gloss
### Key takeaways                 ← existing; first bullet is the bold-equation definition
### What's next                   ← existing; end with "Now we must have understood X, why it exists, how it works, and where it fits."
```

### 1.4 Style checklist to run on every chapter after rewriting

- [ ] A "We will cover the following:" list directly under the one-paragraph version.
- [ ] Every term not defined in an earlier chapter is defined in bold the sentence before first use.
- [ ] Every equation / code / backtick term is followed by "In simple words, …".
- [ ] Exactly one running scenario; it appears in the intro, the how-it-works, the example, and the code.
- [ ] At least one wrong answer shown, then the right answer.
- [ ] Every "Think of it like" has explicit mapping sentences.
- [ ] Every algorithm is written as Phase/Step lines before its code.
- [ ] Section titles are plain nouns/questions.
- [ ] No sentence over ~30 words; no semicolons; at most one em-dash per section.
- [ ] "we" for shared work; "you" only for advice.
- [ ] Ends with Advantages / Disadvantages / When to use which one (or a clear equivalent).
- [ ] Wit and jargon confined to What people get wrong + Ninja notes.

---

## Part 2 — Global fixes (do these first; they touch many files)

### 2.1 The five characters (README.md "The five characters" table)

Reality vs README:

| Character | README says first appears | Actual |
|---|---|---|
| Great Library | Ch. 1 | 23-ivf.md ("The Great Library is organised into a hundred wings"). 6 chapters total. |
| Map Room | Ch. 3 | 01-what-is-a-vector.md ("We will call that room **the Map Room**"). 6 chapters total. |
| Card Catalog | Ch. 17 | Never appears as a named character. 17-brute-force.md says only "a card catalogue with a million cards". |
| Librarian | Ch. 20 | 03-geometry-of-meaning.md ("The Librarian always comes back with an armful of books"), unglossed. |
| Scholar | Ch. 49 | 03:111, 13-bi-vs-cross-encoders.md ("The Scholar reads what survived"), 19:56, 21:111, 41. |

README also says "Almost every idea in this book is explained against one running metaphor." The Map Room
appears in 6 of 66 chapters.

**Fix (do this, not the soft option):** Introduce all five in Chapter 1, concretely, and use them.

In `01-what-is-a-vector.md`, immediately after the sentence "We will call that room **the Map Room**."
add:

> Let's meet the whole cast now, because we will use them in every chapter.
>
> **The Great Library** is our corpus. To keep it concrete, it is Acme's support knowledge base: 40,000
> pages. Every page is a book in the Library.
> **The Map Room** is the embedding space. Every book gets a dot in the Map Room. Similar books, nearby dots.
> **The Card Catalog** is the index. It is the structure that lets us find nearby dots without checking
> every one. We build it in Part IV.
> **The Librarian** is the retriever. Given a question, the Librarian uses the Card Catalog to fetch a
> handful of books.
> **The Scholar** is the LLM (the large language model, the kind of system behind ChatGPT or Claude). The
> Scholar reads what the Librarian brings and writes the answer.
>
> In one sentence: we teach a Librarian to find books in a Library too big to read, by giving every book a
> place in a Map Room and building a Card Catalog over those places, so the Scholar can answer.

Then update the README table's "First appears" column to "Ch. 1" for all five, and change "Almost every
idea in this book is explained against one running metaphor" to "The book uses one running cast of five
characters. Meet them in Chapter 1 and every later chapter gets easier."

Then in `17-brute-force.md` change "The Librarian has a question and a card catalogue with a million
cards." to "The Librarian has a question. The **Card Catalog**, our index, has a million cards, one per
book." In `20-ann-family-tree.md` and `49-rag-first-principles.md` keep the existing introductions but
change "Our last character arrives." (49:26) to "Now the Scholar, whom we met in Chapter 1, takes centre
stage."

Finally, thread the cast: every chapter's running scenario should be phrased in terms of the Library /
Map Room / Card Catalog / Librarian / Scholar where natural. Target: each character appears in at least
half the chapters of the Parts where it is relevant (Map Room: Parts I–II; Card Catalog: Parts III–IV;
Librarian: everywhere; Scholar: Parts VII–IX).

### 2.2 Acronyms that are never expanded

- **RAG.** First use `03-geometry-of-meaning.md` "confident nonsense in RAG systems"; also 07:119, 08:147,
  08:183, 08:220, 10:107, 14:7. Only expanded at 49:15. At the Ch 3 first use write: "RAG (retrieval-
  augmented generation: finding the right pages and handing them to a language model; Part VII)". Every
  later pre-49 use may stay bare.
- **LLM.** Never expanded in any chapter. With fix 2.1 it is expanded in Ch 1. Also in
  `49-rag-first-principles.md` after "**The Scholar** is brilliant, articulate, and has read enormously —
  but not *your* documents." add: "In real terms the Scholar is the **LLM**, the large language model.
  In simple words, it is a program that has read a huge amount of text, and when we ask it something it
  writes an answer one word at a time." Add "**LLM = Large Language Model.**" as a bold equation line.
- **ANN.** `03-geometry-of-meaning.md` "This clumpiness is the reason approximate nearest neighbour
  search works at all." → "…the reason approximate nearest neighbour search (**ANN** for short) works at
  all." Bare "ANN" at 03:192, 06:21, 06:108, 06:168, 07:55, 07:89, 07:195 then resolves.

### 2.3 Chapter-count and Part-count slips

- `01-what-is-a-vector.md` "The remaining 64 chapters are about doing this well" → "The remaining 65
  chapters".
- `02-the-leap-of-embedding.md` "it is the reason for three entire parts of this book:" followed by
  Chunking (Ch 50), Hybrid (Ch 40), Multi-vector (Part V) → "it is the reason for three whole stretches
  of this book:".
- `49-rag-first-principles.md` "six steps each side" → "roughly half a dozen steps each side" (index side
  has four).

---

## Part 3 — Per-chapter corrections

Format: **file** → *locate by* "quoted current text" → **change to** / **do**. Severity tag: [MUST] a novice
is misled or a number is wrong; [SHOULD] undefined term or inconsistency; [NICE] polish/staleness.

### Chapter 1 — 01-what-is-a-vector.md
- [MUST] See 2.1 (cast introduction) and 2.3 ("64 chapters").

### Chapter 2 — 02-the-leap-of-embedding.md
- [SHOULD] "a general model sees them as gibberish tokens" → "…as gibberish **tokens** (the word-fragments a
  model actually reads; Chapter 10 explains them)".
- [SHOULD] "contains, at a conservative estimate, thousands of distinct facts" → "hundreds of distinct
  facts" (a 2,000-word article cannot hold thousands).
- [SHOULD] See 2.3 ("three entire parts").

### Chapter 3 — 03-geometry-of-meaning.md
- [MUST] Rewrite the "empty regions" passage as in §1.2 (defines k, RAG, Librarian, Scholar, names the
  fix).
- [MUST] "measure the cosine similarity between random, unrelated pairs, you might expect something near
  zero" → insert before it: "Chapter 4 defines **cosine similarity** properly. For now: it is a score
  from 1 (same direction, very similar) down to 0 (unrelated)." Also at "Cosine 0.9 means very similar".
- [MUST] "and Chapter 55 will explain why that line moves depending on the query" and "Chapter 55 turns
  this into a procedure" — these promises are currently *not delivered* in Ch 55. Keep the sentences
  only after doing the Ch 55 fix below.
- [SHOULD] "vector search *always* returns the k nearest neighbours" → "…the k nearest neighbours (k is
  simply how many results we asked for, usually 5 or 10)". (Superseded by the §1.2 rewrite if applied.)
- [SHOULD] Code block "(V[i] * V[j]).sum(1)" and "V @ V.T" — add comment `# multiply-and-add = the dot
  product, Chapter 4`; on `normalize_embeddings=True` add `# unit length, Chapter 5`.

### Chapter 4 — 04-measuring-similarity.md
- [SHOULD] "cos(θ)" → add "where θ (theta) is the angle between the two vectors".
- [SHOULD] "# O(n), not O(n log n)" → "# a partial sort: much cheaper than fully sorting a million scores".
- [SHOULD] "billions per second with SIMD" → "…with SIMD (vector instructions that do several multiplies
  at once)".
- [SHOULD] "due to quantization" → "due to **quantization** (storing each number in fewer bits to save
  memory; Chapter 26)".

### Chapter 5 — 05-normalization.md
- [SHOULD] One-paragraph version "makes quantization better behaved" → add the same gloss as Ch 4 if Ch 4
  did not get it first.

### Chapter 6 — 06-high-dimensions.md
- [MUST] Main text uses Part III/IV vocabulary. "need higher `efSearch`, more probes, more memory — the
  same recall costs more. Benchmarks like SIFT1M are easy; some real corpora are much harder." → "need to
  search a bigger slice of the index to find the same true neighbours, so they cost more time and
  memory. Public benchmark datasets are often easy; real corpora are often harder." Move the `efSearch`
  / probes / SIFT1M sentence verbatim into Ninja notes.
- [MUST] "under ~15 — easy, small indexes and low `efSearch` will do" → "under ~15 — easy; a small index
  searched shallowly will do".
- [SHOULD] "PCA, or better, a Matryoshka-trained model" → "PCA (a classical method that re-orders
  dimensions by how much they vary), or better, …".
- [SHOULD] "$O(\log n / \epsilon^2)$ dimensions … within a factor of $1 \pm \epsilon$" — move to Ninja
  notes; in main text say "roughly: a few hundred dimensions suffice no matter how many you started with".
- [SHOULD] "so the library is broken" → "so the index library is broken" (avoid clash with Great Library).

### Chapter 7 — 07-dense-vs-sparse.md
- [MUST] "Lucene-family engines (Elasticsearch, OpenSearch, Solr, Tantivy, Vespa)" → "Lucene-family
  engines (Elasticsearch, OpenSearch, Solr) and their independent cousins (Tantivy, Vespa)".
- [SHOULD] "One rare token, one posting list" → "One rare token, one **posting list** (the list of
  documents that contain that term)".
- [SHOULD] "a 200-word passage becomes 200 vectors" → "a 200-word passage becomes roughly 250 vectors,
  one per token".

### Chapter 8 — 08-before-embeddings.md
- [MUST] "$$ \text{IDF}(t) = \log\frac{N}{n_t} $$ … its IDF is about 12.7" — state the base and gloss:
  after the formula add "Here log is the natural logarithm. In simple words, it squashes a big ratio into
  a small number: a term in 3 of a million documents gets about 12.7, a term in half of them gets 0.7."
- [SHOULD] "No stopword list required" → "No list of **stopwords** (the, of, and…) required".
- [SHOULD] "300-token RAG chunks" → "300-word RAG chunks" (tokens undefined until Ch 10).

### Chapter 9 — 09-word2vec.md
- [MUST] "they wanted the internal weights" → "they wanted the internal **weights**. In simple words, a
  neural network is a big table of adjustable numbers; training nudges those numbers until the guesses
  improve, and the numbers themselves turned out to be the useful product."
- [MUST] "Properly predicting a word means a softmax over the whole vocabulary — a normalization across a
  million words" → "Properly predicting a word means a **softmax** over the whole vocabulary. In simple
  words, a softmax turns a million raw scores into probabilities that add up to 1, which means touching
  every word in the vocabulary for every single training pair."
- [SHOULD] "σ … 𝔼 … P_n" → add beside it "σ is the sigmoid, which squashes any score into 0–1; 𝔼 means
  'on average over the random negatives'".
- [SHOULD] "It is a single linear layer" → "It is two lookup tables (two embedding matrices) with no
  nonlinearity between them".

### Chapter 10 — 10-transformers-contextual.md
- [SHOULD] "a decoder-only model adapted to encoding" → "a **decoder-only** model (the GPT-style kind,
  where each token may only look at the tokens *before* it, called 'causal' attention) adapted to
  encoding".
- [SHOULD] "averaged GloVe vectors" → "averaged GloVe vectors (a word2vec-era static embedding)".
- [NICE] Late chunking is fully explained here (Ninja notes) AND in Ch 11 Ninja notes with the same
  pronoun example. Cut this one to one sentence pointing to Ch 11.

### Chapter 11 — 11-pooling.md
- [MUST] "In models fine-tuned for retrieval with a `[CLS]` objective — the BGE and E5 families among them
  — it is excellent." → "In models fine-tuned for retrieval with a `[CLS]` objective, such as the BGE
  family, it is excellent. (E5, by contrast, is mean-pooled, and e5-mistral is last-token pooled. This is
  exactly why you must read the model card.)"
- [MUST] Code: `def last_pool(last_hidden, attention_mask):            # left-padding aware` → comment
  `# right-padding; for left padding use last_hidden[:, -1]`.
- [SHOULD] "ignoring padding" first use → "ignoring **padding** (the filler tokens added so every text in
  a batch has the same length)".
- [SHOULD] "3–8 points of nDCG@10" → "3–8 points of nDCG@10 (a 0–100 ranking-quality score; Chapter 19)".
- [SHOULD] Option C "Since everyone spoke in order and the last speaker heard everything" → add "(a
  different meeting from Chapter 10's: here people speak one at a time and hear only those before them.
  That is how GPT-style models work.)"
- [NICE] "alongside 380 words" → "alongside nearly 400 words".

### Chapter 12 — 12-contrastive-learning.md
- [MUST] "The standard loss treats it as a multiple-choice quiz." → precede with "A **loss** is a single
  number that says how wrong the model currently is. Training means nudging the weights to make it
  smaller, and the **gradient** is the direction of that nudge. The standard loss treats it as…"
- [MUST] "use a cross-encoder (Chapter 13) to score candidates and discard any it rates as relevant. This
  last approach — **denoised supervision** — is a core contribution of ColBERTv2" → "use a cross-encoder
  (Chapter 13) to score every candidate and train against those *scores* rather than yes/no labels, so a
  mislabelled 'negative' the teacher likes is no longer pushed away. This is ColBERTv2's **denoised
  supervision**. (Outright discarding teacher-approved negatives is RocketQA's variant.)"
- [SHOULD] "MTEB" first use → "MTEB (the public embedding leaderboard; Chapter 16)".
- [SHOULD] `logits` in code → comment `# raw, unbounded scores`.

### Chapter 13 — 13-bi-vs-cross-encoders.md
- [MUST] "The Scholar reads what survived." → "The Scholar (the LLM, Chapter 1) reads what survived."
- [MUST] "The price is 50–100× the storage." → "The price is roughly 30× the storage in bytes (100–200
  tokens × 128 dims vs one 768-d vector), and 100–200× the vector *count*, before compression."
- [SHOULD] "uncalibrated logits" → "raw, unbounded scores (**logits**)".

### Chapter 14 — 14-asymmetry-and-instructions.md
- [SHOULD] "nDCG@10" — gloss if Ch 11 did not already.

### Chapter 15 — 15-matryoshka.md
- [MUST] "Stage 1: binary vectors, 1024 dims → 128 bytes/vector" but "300 GB of RAM and 10 GB" are 768-d
  numbers. Change Stage 1 to "768 dims → 96 bytes/vector" (or change the RAM to ~410 GB and ~13 GB).
- [SHOULD] "your cosine similarities are scaled by an arbitrary per-vector constant" → "your dot-product
  scores (which you were treating as cosines) are scaled…".
- [SHOULD] "Gradients flow from all of them" — fine once Ch 12 defines gradient.
- [SHOULD] "conceptually similar to what PCA gives you" → gloss PCA if Ch 6 did not.

### Chapter 16 — 16-choosing-a-model.md
- [SHOULD] Key takeaways list "size" for dimension #2; body says "Dimensions". Use "dimensions".
- [NICE] "Jina v3" → "Jina v3/v4".

### Chapter 17 — 17-brute-force.md
- [MUST] See 2.1 (Card Catalog sentence).
- [SHOULD] "building an HNSW index over 40,000 chunks" (first HNSW as-if-known) → "an HNSW index (the
  graph index of Chapters 28–31)".
- [SHOULD] "$O(n \log n)$ … $O(n)$" → add "(time grows in step with n, rather than n times log n; a real
  difference at ten million scores)".
- [SHOULD] "hand-tuned SIMD kernels" and "reuses each cache line" → at the earlier "eight or sixteen
  multiply-adds per cycle" write "a vector (SIMD) unit that performs…"; "reuses each block of memory
  already pulled into the CPU's cache".

### Chapter 18 — 18-recall-latency-memory.md
- [MUST] "usually quoted at p50 and p99. The p99 is what your users feel." → "usually quoted at **p50**
  (the time a typical query takes; half are faster) and **p99** (the time the slowest 1-in-100 queries
  take). The p99 is what your users feel."
- [MUST] "plus ~50% HNSW graph overhead" → "plus ~5–10% HNSW graph overhead at 768 dimensions (Chapter 29
  does the arithmetic)".
- [SHOULD] "IVF needs its centroids trained on a sample" → "IVF needs its cluster centres (**centroids**)
  trained on a sample".
- [SHOULD] Speed-up table: brute force "~1000×" vs text "10–100× speedup" and Ch 17's table → "~50–500×
  (brute force; scale-dependent)".

### Chapter 19 — 19-measuring-retrieval-quality.md
- [MUST] Worked example "| nDCG@10 | ~0.79 | correct docs present but poorly positioned |" → "~0.62".
  (DCG = 1/log2(3)+1/log2(6)+1/log2(10) = 1.319; IDCG = 1+1/log2(3)+1/log2(4) = 2.131; 0.619.)
- [SHOULD] nDCG formula: add one arithmetic line "position 1 is divided by 1, position 3 by 2, position 7
  by 3, so a hit at rank 7 counts a third as much as one at rank 1".

### Chapter 20 — 20-ann-family-tree.md
- [MUST] Decision table "Your situation | Use | Chapter" is off by one (written before Ch 27 was inserted).
  Change the Chapter column to: Flat "17"; Flat/HNSW "17, 28"; HNSW "28–31"; HNSW+int8/IVF-PQ "25, 26";
  IVF-PQ/DiskANN "25, 32"; DiskANN/sharded "32, 58"; filtering "33"; batch "17"; multi-vector "37, 38".
- [MUST] "Every hop provably moves closer to the query" → "Every hop moves to a closer neighbour, so you
  can stop whenever you are satisfied. The walk can get stuck in a dead end, which is exactly what the
  `ef` shortlist in Chapter 30 is for."
- [SHOULD] "Google's SOAR and various learned-routing approaches" → drop SOAR (it is a multi-assignment
  trick for partition indexes, not a learned index).
- [SHOULD] "a modest ~5% on top" — correct; keep, and make Ch 18/23 agree (above/below).

### Chapter 21 — 21-lsh.md
- [SHOULD] "taking the sign of a random projection is exactly binary quantization" → "…is essentially
  binary quantization with random axes (Chapter 26 uses the model's own axes instead)".
- [SHOULD] "word shingles" → "word shingles (overlapping runs of, say, five consecutive words)".
- [SHOULD] "Scholar" at 21:111 — fine once Ch 1 introduces the cast.

### Chapter 22 — 22-trees-and-annoy.md
- [SHOULD] "k-means" first appears in Ninja notes / What's next with no definition anywhere in the book.
  The definition goes in Ch 23 (below); here write "k-means (a clustering method, defined next chapter)".

### Chapter 23 — 23-ivf.md
- [MUST] "**Step 1: train.** Run k-means on a sample of your vectors" → "**Step 1: train.** Run
  **k-means** on a sample of your vectors. In simple words: pick `nlist` starting centres at random,
  assign every vector to its nearest centre, move each centre to the average of its vectors, and repeat
  until nothing moves. The final centres are the **centroids**."
- [MUST] "HNSW stores the vectors *plus a graph*, at 1.5–2×. At a billion vectors that difference is
  hundreds of gigabytes." → "HNSW stores the vectors *plus a graph*, about 1.05–1.1× at 768 dimensions
  (Chapter 29). The real memory gap is that IVF's lists compress to PQ codes (Chapter 25) while HNSW must
  keep something close to full vectors in RAM."
- [MUST] Code `IVFIndex.add`: the two lines `self.lists = {c: [] for c in range(self.nlist)}` and
  `self.vecs  = {c: [] for c in range(self.nlist)}` re-initialise on every call, so a second `add()`
  drops the first batch. Move both into `train()` and make `add` append.

### Chapter 24 — 24-product-quantization.md
- [SHOULD] Table row "| 768 (no PQ) | 3,072 | 1× |" → "| — (no PQ) | 3,072 | 1× |" (m=768 with 8-bit
  codes would be 768 B, not 3,072).
- [SHOULD] "about 25,000 operations" → "a 96 × 256 table, about 25,000 entries, computed once per query
  (each entry is an 8-dimension dot product)".
- [SHOULD] "encodes what PQ got wrong with a second codebook … this is exactly how ColBERTv2 compresses" →
  "ColBERTv2 uses a close cousin: a centroid ID plus a 1–2-bit *scalar* code for each residual
  coordinate (Chapter 37)".

### Chapter 25 — 25-ivfpq-and-opq.md
- [SHOULD] "`IndexIVF_HNSW`" is not a FAISS class → "the FAISS factory string `IVF65536_HNSW32,PQ96`
  builds an HNSW graph over the centroids".

### Chapter 26 — 26-scalar-binary-quantization.md
- OK. (Depth=1000 for k=10 here is 100×; Ch 66 must reflect that, see below.)

### Chapter 27 — 27-turboquant.md  (weakest chapter; four fixes)
- [MUST] No citation anywhere. At first mention add "(Zandieh, Daliri, Hadian & Mirrokni, Google, 2025;
  arXiv 2504.19874)".
- [MUST] Table rows "**TurboQuant, 2-bit** | **192**" and "**TurboQuant, 1-bit** | **96**" contradict the
  code, which pads 768 → 1024 (`self.d = 1 << (dim - 1).bit_length()`), giving 256 B and 128 B. Either
  (a) change the code to a block Hadamard (768 = 3 × 256) or a dense random rotation and keep the table,
  or (b) state the padding cost in the table. Also add "+4 B scale" to both TurboQuant rows since the
  text says the per-vector scale is stored.
- [MUST] "**3. It fits into every other structure in this book.** It can replace the compression step
  anywhere:" + the five bullets read as shipped facts. Change the heading sentence to "**3. In principle
  it can replace the compression step in every other structure in this book.**" and add after the list:
  "**Note:** these integrations are where the method *could* go, not published results. As of 2026 no
  mainstream index ships TurboQuant. Treat it as something to benchmark against PQ on your own data."
- [MUST] "At 96 bytes, TurboQuant distorts less than binary and gets comparable accuracy to PQ, without
  PQ's codebooks." → "At 96 bytes, TurboQuant's 1-bit mode is rotated sign quantization plus an unbiased
  score estimator, so it distorts less than plain binary. Whether it matches PQ at equal bytes on *your*
  data is something to measure, not assume."
- [SHOULD] "applies a fast random rotation" / "the standard trick is a randomized Hadamard transform" → add
  one sentence: "The paper's guarantee is for a dense uniformly random rotation; the randomized Hadamard
  transform (a fixed matrix of ±1 entries, so multiplying needs only additions) is the fast stand-in
  used in practice."
- [SHOULD] "1-bit quantized Johnson–Lindenstrauss (QJL) transform" → add "(project the residual onto
  random directions and keep only the signs: the SimHash trick from Chapter 21)".

### Chapter 28 — 28-hnsw-1-small-worlds.md
- [SHOULD] "reaches the target in about O(log n) hops" (flat NSW) contradicts "polylogarithmically" later;
  Kleinberg is O(log² n). → "in a polylogarithmic number of hops".
- [SHOULD] "the wider beam search of Chapter 30" first use → "(a greedy walk that keeps a shortlist of
  the best `ef` candidates instead of just one; Chapter 30)".

### Chapter 29 — 29-hnsw-2-building.md
- [SHOULD] Upper-layer edge formula gives 4.3 B for M=16 but text writes "≈ 10 B" → "≈ 4–5 B".
- [SHOULD] "128 bytes of edges sits against 512 bytes of vector" is 1.25×, not "1.5–2×". Use M=32 (256 B
  of edges) for the low-dimensional example.
- [NICE] "100 million vectors at efConstruction = 200 is hours on a many-core machine" duplicated in
  Ch 31; make Ch 31's a back-reference.

### Chapter 30 — 30-hnsw-3-searching.md
- [SHOULD] "latency grows close to linearly" but the table above grows 1.56–1.73× per doubling →
  "latency grows about 1.6–1.7× each time `efSearch` doubles".

### Chapter 31 — 31-hnsw-production.md
- [SHOULD] Where "soft delete" is defined, add "(a **tombstone**)" so the later "30% of the nodes in your
  beam are tombstones" resolves.
- [SHOULD] "deletes do not free memory in any mainstream implementation" → "in hnswlib, FAISS, Lucene and
  most databases (Vespa is the exception)".
- [SHOULD] 100M memory table rows: binary "~20 GB + rescore store", DiskANN "~15 GB RAM + SSD" disagree
  with Ch 60's better-derived "~25 GB" and "~5 GB". Align to Ch 60.
- [SHOULD] int8 HNSW at 1B: this chapter implies ~925 GB; Ch 32 says "~800 GB". Fix Ch 32 to "~900 GB".

### Chapter 32 — 32-diskann.md
- [MUST] Query path step "4. Read their full float32 vectors from SSD (batched read)" contradicts the next
  paragraph ("a single 4 KB read returns both") and the later "already fetched during traversal … nearly
  free". → "4. Rescore with the full vectors already read alongside each visited node's neighbour list".
- [MUST] "A query issuing 64 parallel reads" and `beam_width=64` — DiskANN's beam width W is 2–8; the
  ~100 search list L is the `efSearch` analogue. → "A query with L=100 and beam width 4–8 does ~100 4 KB
  reads in ~15–25 parallel rounds, ≈2–3 ms on NVMe"; set `beam_width=4`.
- [SHOULD] "IOPS", "io_uring" → "IOPS (random reads per second)"; "io_uring (Linux's fast asynchronous
  I/O interface)".
- [SHOULD] "~800 GB" → "~900 GB" (see Ch 31).
- [NICE] LSM/Lucene-segments sentence duplicated from Ch 31; back-reference.

### Chapter 33 — 33-filtered-search.md
- [MUST] "selectivity" is used in two opposite senses. Prose: "It breaks when the filter is selective"
  (= passes few). Code/table: `selectivity > 0.3  # weak filter` (= fraction passing). Define once at the
  prose first use: "A filter's **pass rate** is the fraction of the corpus it lets through. A *selective*
  filter has a low pass rate." Rename the code variable `selectivity` → `pass_rate` and the table header
  "Choosing, by selectivity" → "Choosing, by pass rate".
- [MUST] Code `return [c for c in ann.search(q, k * 3) if predicate(c)][:k]` under-returns at pass rate
  0.3 (expected 0.9k survivors). → `ann.search(q, int(2 * k / pass_rate))`.
- [SHOULD] Thresholds disagree: "Use when: the filter passes more than ~10%" vs table ">30%" vs takeaways
  ">30%". Make the first ">30% (10–30% with a larger k)".
- [SHOULD] Table row "< 0.1%" is relative while code and takeaways use absolute "<50k". At 1B, 0.1% is
  1M vectors. → "matching set < ~50k vectors".

### Chapter 34 — 34-single-vector-bottleneck.md
- [MUST] "The information-theoretic argument is unavoidable: you cannot encode $n$ bits in fewer than $n$
  bits." A 768-d float32 vector is 24,576 bits, so this argument fails for the "thirty facts" example. →
  "The limit is geometric, not about bit counts: a d-dimensional dot product can only carve out a
  bounded number of distinct 'top-k sets' (a sign-rank argument; see Weller et al. 2025, the LIMIT
  result). A corpus with more query-specific facts than that must lose some, no matter how good the
  model."
- [SHOULD] "33× storage increase" — this is the correct byte multiplier; make Ch 13/35/37 agree (~30×
  bytes, ~200× vectors).
- [SHOULD] "~6,400 bytes" compressed omits the 4 B centroid ID Ch 37 counts → "~7,200 (2-bit residual +
  centroid ID)".

### Chapter 35 — 35-colbert-1-late-interaction.md
- [SHOULD] "MS MARCO dev MRR@10" → "MS MARCO (the standard 8.8-million-passage web-search benchmark) dev
  MRR@10 (Chapter 19)".
- [SHOULD] "20–100× raw" storage → "~30× raw bytes (~200× vector count)".
- [SHOULD] Text implies only queries get a marker token; ColBERT prepends `[D]` to documents too and
  filters document punctuation. Add one sentence.

### Chapter 36 — 36-colbert-2-maxsim.md
- [MUST] "**2. Denoised hard negatives.** … have the teacher score them and **discard any the teacher
  rates as relevant**." → "have the teacher score every candidate and train against those scores with a
  KL loss over 64-way tuples, so a mislabelled 'negative' the teacher likes is no longer pushed away.
  (Outright discarding is RocketQA's variant.)" Keep the rest of the paragraph.
- [SHOULD] "**ColBERTv2** made three changes" → "made three *training* changes (its fourth contribution,
  residual compression, is Chapter 37)".
- [SHOULD] Padding bug: "Padding tokens produce near-zero vectors … a `max` can select one, silently
  inflating scores." → "Unmasked padding rows are unit-norm garbage vectors and can win the `max`;
  masked-to-zero rows can still lift a negative maximum to 0. Mask with −∞ either way."

### Chapter 37 — 37-colbert-3-serving.md
- [MUST] "compute them on the fly for 1,000 passages, which is one batched GPU forward pass of ~10 ms" →
  "~100–300 ms on a modern GPU (1,000 passages × ~200 tokens through BERT-base is ~44 TFLOP), or ~10–30
  ms for 100 passages". Cross-check with Ch 41's "ColBERT rerank (100 docs) 5–20 ms".
- [MUST] Code: `self.n_centroids` is never stored and `self.bin_centres` is never defined. Add
  `self.n_centroids = n_centroids` in `__init__` and compute `self.bin_centres` from the quantile edges in
  `train`.
- [SHOULD] "comparable to, or smaller than" single-vector storage — 1-bit residuals give 40 GB vs 30 GB.
  Drop "or smaller than".
- [SHOULD] "+3 to +8 nDCG points" (three places) → qualify "over a same-generation bi-encoder; against
  the strongest modern dense models the gap is often 0–3 points, largest on needle and multi-constraint
  queries". Reconciles with Ch 35's "closed much of the gap".
- [SHOULD] "credits the inverted index to Chapter 8" → Chapter 7 (where it is bolded and introduced).
- [NICE] RAGatouille → also name PyLate (the maintained ColBERT library).

### Chapter 38 — 38-muvera.md
- [MUST] "for any $\epsilon > 0$, FDEs of dimension depending on $\epsilon$ (and not on the number of
  vectors) give an $\epsilon$-additive approximation of Chamfer similarity." → "for any ε > 0, FDEs whose
  dimension grows polynomially with the number of vectors give a ±ε approximation of the
  *query-length-normalised* Chamfer score. In practice a fixed few-thousand-dimensional FDE works well
  regardless of document length, which is the property we actually use."
- [SHOULD] "2–5× fewer candidates" is versus the paper's single-vector heuristic, not PLAID. Say so.

### Chapter 39 — 39-splade.md
- [MUST] "This is why BM25 serves hundreds of millions of documents in single-digit milliseconds, and
  SPLADE inherits it." → "…single-digit milliseconds. SPLADE runs on the same index, but its expanded
  queries and flatter weight distributions defeat much of that pruning, so expect 2–10× BM25's latency
  unless you use the Efficient-SPLADE variants." Also soften the one-paragraph version's "efficiency of
  BM25".
- [NICE] `naver/splade-cocondenser-ensembledistil` → `naver/splade-v3`.

### Chapter 40 — 40-hybrid-search.md
- [MUST] "it consistently outperforms carefully tuned score-based fusion in published comparisons" → "it
  is the best *untuned* default. With an eval set, a tuned convex combination of normalised scores can
  beat it (Bruch, Gai & Ingber, 2023), and RRF is sensitive to k. Start with RRF; graduate when you can
  measure." Same for "Weighted score fusion … usually worse" in What people get wrong.

### Chapter 41 — 41-rerankers.md
- [SHOULD] OUTLINE one-liner "The 10% of the pipeline that delivers 40% of the quality" is unsupported
  (chapter says +5–15 nDCG). Change OUTLINE to "The single largest quality win in a working RAG stack".
- [SHOULD] "Chapter 55" promise about thresholds — see Ch 55.

### Chapter 42 — 42-clip.md
- [MUST] "[vision encoder: ViT]" first use → "ViT, a **Vision Transformer**: the image is cut into small
  squares called **patches**, each patch is treated like a token, and the transformer of Chapter 10 reads
  them."

### Chapter 43 — 43-siglip.md
- [MUST] σ / "sigmoid" in the one-paragraph version and formula → "(the **sigmoid** σ squashes any number
  into a 0–1 probability, so each image–text pair gets its own independent yes/no score instead of
  competing with the whole batch)".
- [SHOULD] "VLM" → expand at first bare use; "Gemma-2B" → "(Google's small open language model)".
- [SHOULD] Text says "Chapter 47 puts numbers on it / shows the trade-off explicitly" about resolution
  budget. Only true after the Ch 47 fix below.
- [NICE] Title says "Modern" but never mentions SigLIP 2 (Feb 2025) or PaliGemma 2. Add one paragraph.

### Chapter 44 — 44-ocr-broken-promise.md
- [MUST] "A document passes through five hands:" but the diagram lists six numbered stages, then
  "$0.9^5 \approx 59\%$", then "six stages instead of five", "five lossy stages"; Ch 45 says "Two stages
  instead of seven" and "the seven-stage diagram in Chapter 44". Settle on **six** (OCR, layout, tables,
  reading order, chunking, embedding): "six hands", $0.9^6 \approx 53\%$, and in Ch 45 "Two stages
  instead of six" / "the six-stage diagram".
- [SHOULD] "IoU" → "IoU (intersection-over-union, a box-overlap score)".

### Chapter 45 — 45-colpali-1-documents-as-images.md
- [MUST] "Nothing is thrown away, because nothing is converted." → "Nothing is *converted*. Something is
  still lost: the page is resized to a fixed 448×448 grid, and that resize is itself lossy. Chapters 46
  and 47 return to exactly this." (The chapter later admits the resize at "illegible to the model".)
- [SHOULD] Stage counts, see Ch 44.

### Chapter 46 — 46-colpali-2-inside.md
- [MUST] "ColPali's training set was roughly 127,000 such pairs, mixed with what open visual-QA data
  existed." and "most of them synthetically generated" and "~127k synthetic (query, page) pairs" → "about
  127,000 pairs in total, roughly a third generated synthetically by a VLM and two-thirds drawn from open
  visual-QA training sets (DocVQA, InfoVQA, TAT-DQA, arXivQA)". Fix all three places.
- [SHOULD] "LoRA adapter" first use → "LoRA adapter (a small set of add-on weights trained in place of the
  full model)".
- [SHOULD] "roughly a character or two" per 14×14 patch → "a few characters across one or two lines".
- [NICE] "ColQwen2 is usually the better choice today" → "ColQwen2 and its Qwen2.5-based successors".

### Chapter 47 — 47-colpali-3-production.md
- [MUST] The DPI block ("#   100 DPI → small text becomes illegible … Sweep DPI against your eval set …
  200 DPI can be worth several points of recall") gives advice that cannot affect the model shown. For
  PaliGemma-based ColPali at fixed 448×448, an A4 page is ~50 DPI after resize, so anything above ~60 DPI
  is discarded. Rewrite: "**Note:** for the original ColPali (PaliGemma, fixed 448×448 input) rendering
  DPI does not matter beyond ~60: the resize throws the extra pixels away. DPI only matters for
  dynamic-resolution models like ColQwen2, below." Then keep the sweep advice under a ColQwen2 heading.
- [MUST] Add the missing ColQwen2 section. OUTLINE promises "ColQwen2, storage maths"; the file's own tags
  include `colqwen`; Ch 43 says "Chapter 47 puts numbers on it"; Ch 61 assumes "~750 patch vectors per
  page after ColQwen's dynamic resolution" which the reader has never seen. Write ~150 words: ColQwen2
  (Qwen2-VL backbone) keeps the page's aspect ratio and emits a variable number of visual tokens up to a
  cap set by `max_pixels` (default ≈ 768 tokens); dense pages hit the cap, sparse slides use fewer; give
  the per-page storage line (≈750 × 128 × 2 B ≈ 190 KB float16, ≈27 KB at 2-bit + centroid) and the
  DPI/patch-budget trade-off.
- [SHOULD] "~10 pages/s ≈ 28 GPU-hours" → "2–10 pages/s depending on GPU and batching, i.e. ~30–100
  GPU-hours" (paper reports ~0.39 s/page).
- [SHOULD] "~11 GB" for 340 vectors × 128-d × 2-bit uses 32 B/vector; Ch 37 says 36 B with centroid ID →
  "~12 GB".
- [SHOULD] "Hierarchical clustering" → gloss (k-means is the only clustering taught).

### Chapter 48 — 48-other-modalities.md
- [SHOULD] "ASR" → "ASR (automatic speech recognition)".

### Chapter 49 — 49-rag-first-principles.md
- [MUST] See 2.1 and 2.2 (Scholar = LLM; "Our last character arrives").
- [MUST] "putting them in the model's **prompt**" and "**context windows** hold a million tokens" → gloss
  both: "prompt (the text we hand the model)"; "context window (the maximum amount of text a model can
  read in one go)".
- [SHOULD] "produce something plausible" → "produce something plausible, what the field calls a
  **hallucination**".
- [SHOULD] "A modest company wiki is fifty times that" (= 75,000 pages) → "A mid-sized company's document
  store is easily ten times that; an enterprise's, thousands of times".
- [SHOULD] "slow to first token" → "slow to start answering".
- [SHOULD] See 2.3 ("six steps each side").

### Chapter 50 — 50-chunking.md
- [MUST] Two conflicting defaults: "Contextual retrieval… this is the one that actually works. Do this
  before you consider fine-tuning anything" vs "**Use late chunking by default**; add contextual retrieval
  where quality matters most." State one rule in the comparison paragraph: "Start with structure-aware
  chunking plus title prepend (Chapter 51). Add late chunking if your embedding model supports it. Add
  contextual retrieval where your eval says quality matters most and you can afford the LLM pass."
- [SHOULD] "covers the roughly one-third of retrieval quality" → "covers the large share of retrieval
  quality" (unsupported number; also fix OUTLINE Ch 51 one-liner "The 30% of retrieval quality…" → "The
  large share of retrieval quality that has nothing to do with vectors").
- [SHOULD] "Prompt caching over the parent document" first use → "(providers charge less for a prompt
  prefix they have already seen; Chapter 53)".
- [NICE] The "retrieve small, pass large" case is made twice (Ninja notes and What people get wrong);
  cut the second to one line.

### Chapter 51 — 51-metadata-and-structure.md
- [SHOULD] "acl" → "acl (access-control list: who may see this document)".

### Chapter 52 — 52-query-understanding.md
- [SHOULD] "multi-hop" → "multi-hop (questions where one fact is needed to find the next)".

### Chapter 53 — 53-context-assembly.md
- [SHOULD] "system prompt" first use → "system prompt (the standing instructions sent with every request)".
- [SHOULD] Prose says truncate the last item "at a sentence boundary" but `assemble()` just `break`s and
  drops it. Change prose to "drop the item that does not fit" or add a `truncate_to_sentence()` call.

### Chapter 54 — 54-evaluating-rag.md
- [MUST] "With 20 examples the confidence interval is roughly ±10 points" → "With 20 examples the standard
  error is roughly ±10 points (a 95% interval of about ±20)". Ch 19 says standard error.

### Chapter 55 — 55-rag-failure-modes.md
- [MUST] Deliver the promised calibration procedure. Under failure 8's "**Fix:** A calibrated relevance
  threshold on top retrieval or rerank scores" add: "**How to calibrate it:** take the golden set from
  Chapter 54, including its `answerable: false` questions. Sweep the floor from low to high. For each
  value record two numbers: how often the system correctly refuses the unanswerable questions, and how
  often it wrongly refuses answerable ones. Pick the floor that maximises the first while keeping the
  second under a limit you choose (say 2%). Re-run this whenever you change the embedding or reranker
  model, because raw scores are not comparable across models (Chapter 41). This is also why the line
  'moves with the query' (Chapter 3): a query in a dense part of the Map Room has many near neighbours,
  so its top score is high even when none is relevant."
- [MUST] Failure 11 ("Incomplete answers to multi-part questions") sits under *Generation* but its Cause
  ("Retrieval found evidence for only one part") and Test ("context recall per sub-question") make it a
  retrieval failure by the chapter's own split ("No → failures 1–7; Yes → failures 8–12"). Move it into
  the retrieval group (8 retrieval / 4 generation, renumber) and update the `diagnose()` code that
  currently returns "check F10, F11" for the grounded-but-incorrect branch.

### Chapter 56 — 56-agentic-retrieval.md
- [SHOULD] "search as a *tool*" and "orchestrator" → one sentence before the loop diagram: "A **tool call**
  is when the model, instead of answering, emits a structured request such as `search("SSO Pro plan")`.
  Our code, the **orchestrator**, runs it and feeds the result back. The model may then call again or
  answer."

### Chapter 57 — 57-architecture-at-scale.md
- [SHOULD] "(CDC, webhooks, crawls)" → "(change-data-capture (CDC), webhooks, crawls)".
- [SHOULD] "replicas scale with traffic" → "replicas (copies of an index that share the query load;
  Chapter 58) scale with traffic".

### Chapter 58 — 58-sharding-and-routing.md
- [MUST] "a query fanning out to 40 shards hits at least one shard's p99 on most requests" → "…on about a
  third of requests" (the next line computes 1 − 0.99^40 = 0.33).

### Chapter 59 — 59-freshness.md
- [NICE] "LSM trees, Lucene segments" → "(log-structured storage, where new data lands in fresh segments
  that are merged later)".

### Chapter 60 — 60-cost-engineering.md
- OK arithmetically. Its 100M memory table (binary ~25 GB, DiskANN ~5 GB) is the reference; align Ch 31.

### Chapter 61 — 61-image-heavy-rag.md
- [MUST] Latency table declares "The retrieval SLO is met" on "**Retrieval total (excluding rewrite)**
  ~50–120 ms" while "Query rewrite … 100–300 ms" alone exceeds the 150 ms p95 requirement. Add: "The
  150 ms SLO holds only if rewrite is skipped on the hot path. Run it only for queries the router flags
  as hard, or run it asynchronously and re-rank when it returns."
- [SHOULD] "SLO" → "service-level objective (SLO), the latency target from the brief".
- [SHOULD] "~5,000-d" FDE → "~4,000-d" (Ch 38 and Ch 47 say ~4,000).
- [SHOULD] "~750 patch vectors per page after ColQwen's dynamic resolution" — resolves once Ch 47 gets its
  ColQwen2 section.

### Chapter 62 — 62-security-and-tenancy.md
- [SHOULD] "noisy neighbours are contained" → "noisy neighbours (one tenant's load slowing another's
  queries) are contained".

### Chapter 63 — 63-migration-and-drift.md
- [MUST] Code: `a.latency_ms` where `a` is the list returned by `search()` (the previous line uses
  `a[0].id`). Return a result object with `.hits` and `.latency_ms`, or time the calls separately.
- [SHOULD] "in CI" → "in CI (the automated test run on every code change)".

### Chapter 65 — 65-the-frontier.md
- [MUST] "**LSH, MUVERA and TurboQuant are randomised, training-free constructions with guarantees**, and
  each competes with or beats learned alternatives in practice." contradicts Ch 21's section "Why LSH
  lost for dense retrieval". → "**MUVERA and TurboQuant are randomised, training-free constructions with
  guarantees** (LSH was the ancestor of the idea; it lost on dense vectors but its MinHash cousin still
  wins at deduplication), and each…".

### Chapter 66 — 66-field-manual.md
- [SHOULD] BM25 k1 "1.2–1.5" → "1.2–2.0" (Ch 8).
- [SHOULD] Rescore depth "10–20× final k | Ch 24, 26" → "10–20× final k for PQ (Ch 25); ~100× for binary
  (Ch 26)".
- [SHOULD] Recency half-life "90–180 days" → "~180 days" (Ch 51 only ever uses 180).
- [SHOULD] README promises "every decision tree, formula, and default". Ch 66 has nothing from Ch 58
  (sharding-strategy table), Ch 60 (cost-per-query formula, lever ranking), Ch 63 (six migration phases),
  Ch 59 (staleness budget, maintenance calendar). Add a "Scale and operations" block reproducing those
  four tables, or change README/Ch 66 to "the most-used".

### OUTLINE.md
- Ch 41 one-liner → "The single largest quality win in a working RAG stack."
- Ch 51 one-liner → "The large share of retrieval quality that has nothing to do with vectors."

---

## Part 4 — Verify after fixing

Run all of these and fix anything they surface.

```bash
cd chapters
# 1. prev/next chain still intact, no broken links
for f in $(ls *.md|sort); do grep -m1 '^next:' $f; done | awk '{print $2}' | while read n; do [ "$n" = null ] || ls $n.md >/dev/null || echo "BAD next: $n"; done
grep -oE '\]\(\./[^)#]+' *.md | sed 's/.*(\.\///' | sort -u | while read t; do [ -f "$t" ] || echo "BROKEN $t"; done

# 2. cast is introduced in Ch 1 and used
grep -c "Great Library\|Map Room\|Card Catalog\|Librarian\|Scholar" 01-what-is-a-vector.md   # expect >= 5
for c in "Great Library" "Map Room" "Card Catalog" "Librarian" "Scholar"; do echo "$c: $(grep -l "$c" *.md | wc -l) chapters"; done

# 3. acronyms expanded before/at first use
grep -n "RAG" 0[1-9]*.md | head -1        # must be the glossed one
grep -rn "Large Language Model\|large language model" *.md | head -2   # must exist, in Ch 1 and Ch 49
grep -n "(ANN" 03-geometry-of-meaning.md   # must exist

# 4. numbers
python3 -c "import math; d=sum(1/math.log2(r+1) for r in [2,5,9]); i=sum(1/math.log2(r+1) for r in [1,2,3]); print(round(d/i,2))"   # 0.62, must match Ch 19
grep -n "0.9^\|0\.9\^" 44-ocr-broken-promise.md  # must be ^6 and match "six"
grep -n "50–100×\|50-100×" *.md            # must be empty
grep -n "~800 GB" 32-diskann.md             # must be empty
grep -n "~10 ms" 37-colbert-3-serving.md    # must be empty
grep -n "~0.79" 19-measuring-retrieval-quality.md   # must be empty

# 5. Ch 20 table matches OUTLINE
sed -n '/Your situation/,/^$/p' 20-ann-family-tree.md

# 6. terms defined before use: for each term, the first file that mentions it must also define it (bold)
for t in "k-means" "p99" "softmax" "gradient" "tombstone" "beam" "patch" "sigmoid" "LoRA" "prompt" "context window" "SLO" "CDC" "pass rate"; do
  f=$(grep -li "$t" *.md | sort | head -1); echo "$t -> first in $f; bolded there: $(grep -ci "\*\*$t" $f)"; done

# 7. style pass
grep -c "We will cover the following" *.md | grep ":0"     # must be empty (Ch 66 excepted)
grep -c "In simple words" *.md | grep ":0"                # should be empty
grep -n ";" *.md | grep -v '^\S*:\s*[`#|]' | grep -v "&[a-z]*;" | wc -l   # semicolons in prose; drive toward 0
```

Then read Chapters 1, 3, 9, 12, 20, 27, 33, 47, 49, 55 top to bottom once more as a novice.
