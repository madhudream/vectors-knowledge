---
title: "Before Embeddings: One-Hot, Bag-of-Words, TF-IDF, BM25"
chapter: 8
part: "Part II — Where Embeddings Come From"
slug: before-embeddings
readingTime: "14 min"
summary: "The counting-based ancestors of every modern retrieval system — and why BM25, a formula from 1994, is still the baseline you must beat."
tags: [sparse, bm25, tfidf, history, baselines]
prev: 07-dense-vs-sparse
next: 09-word2vec
---

# Before Embeddings: One-Hot, Bag-of-Words, TF-IDF, BM25

**The one-paragraph version.** Long before neural networks, people turned documents into
vectors by counting words. Each method fixed a flaw in the one before it. One-hot gave every
word its own dimension. Bag-of-words counted the words. TF-IDF gave less weight to words that
appear everywhere. BM25 added two more fixes: repeated words stop helping after a while, and
long documents stop winning just for being long. BM25 is about thirty years old, needs no
training, runs on a laptop, and still beats many neural retrievers on many real query streams.
It is the baseline, and beating it is the bar.

In this chapter, we will learn how search worked before embeddings, by counting words and
weighting the counts. We will build the idea in four small steps, each one fixing the problem
the last one left behind. We will also see exactly where counting breaks, because that break is
the reason the rest of Part II exists.

We will cover the following:

- What is one-hot encoding
- What is bag-of-words
- What is TF-IDF
- What is BM25
- An example: finding the Pequod
- Where counting fails
- BM25 vs embeddings
- When to use which one

Our example for the whole chapter is *Moby-Dick* and its whaling ship, the **Pequod**. Picture a
collection of one million documents: novels, articles and encyclopedia entries. A reader types
*"the Pequod"*. Only three documents in the whole collection mention that ship. We want those
three at the top. The Pequod stands in for a rare identifier, like the error code `SSO-4012` in
Acme's knowledge base (Chapter 7), and we will meet that code again along the way.

---

## What is one-hot encoding

**One-hot = one dimension for every word in the vocabulary, with a 1 for this word and 0
everywhere else.**

A **vocabulary** is the list of every word we have agreed to track. Suppose ours has 30,000
words. Then every word becomes a vector with 30,000 positions, and exactly one of them is 1.

```
vocabulary = [a, aardvark, ..., whale, ..., zebra]      # 30,000 words
"whale"    = [0, 0, ..., 1, ..., 0]
```

In simple words, the position reserved for "whale" is switched on, and every other position is
switched off.

This encoding is faithful and lossless. It records exactly *which word this is*. It is also
useless for similarity, and the reason is worth seeing.

Let's take the dot product (Chapter 4) of `whale` and `dolphin`. We multiply position by
position and add. Wherever `whale` has its 1, `dolphin` has a 0. Wherever `dolphin` has its 1,
`whale` has a 0. Every product is zero, so the sum is zero. The dot product of `whale` and
`bureaucracy` is zero too.

**Every pair of different one-hot vectors is orthogonal.** Orthogonal means "at right angles",
and here it means the dot product is exactly 0: the two vectors share nothing at all. The
representation has no idea that a whale and a dolphin are related, because we never put that
idea in.

Fixing exactly this is what Chapter 9 is about. But first, let's see how far counting alone can
take us.

---

## What is bag-of-words

**Bag-of-words = add up the one-hot vectors of every word in a document.**

The result is simply a count of each word.

```
"the whale ate the fish"  →  {the: 2, whale: 1, ate: 1, fish: 1}
```

Means, a document becomes a 30,000-position vector that is almost all zeros, with a count
wherever a word appears. This is the sparse vector of Chapter 7.

The name is honest about the flaw. It is a *bag*, so word order is gone. *"Dog bites man"* and
*"man bites dog"* give identical vectors. For finding documents on a topic, this matters less
than we might think. For anything that depends on who did what to whom, it is fatal.

Now we can score a document against a query. We multiply the query's count and the document's
count for each word, and add. Shared words raise the score.

Now, the question is, does that find the Pequod?

Let's try. The query is *"the Pequod"*. Here are two documents from our collection:

- **Document A:** *"the whale swam past the ship in the storm"*
- **Document B:** *"Ahab commanded the Pequod"*

| | the | pequod | Bag-of-words score |
|---|---|---|---|
| Query | 1 | 1 | |
| Document A | 3 | 0 | 1×3 + 1×0 = **3** |
| Document B | 1 | 1 | 1×1 + 1×1 = **2** |

Document A wins. The answer is wrong. Document A never mentions the Pequod. It won because it
says "the" three times.

The word `the` appears in almost every document ever written. It dominates the count and tells
us nothing.

---

## What is TF-IDF

The fix is one of the great ideas in information retrieval, and it is intuitive. **A word that
appears in every document distinguishes nothing. A word that appears in three documents is a
fingerprint.**

**TF-IDF = Term Frequency × Inverse Document Frequency.**

- **TF (term frequency)** is how often the term appears *in this document*. More mentions of
  "whale" means the document is more about whales.
- **IDF (inverse document frequency)** is how rare the term is *across the whole collection*.

We score each term by multiplying the two. IDF is computed like this:

$$ \text{IDF}(t) = \log\frac{N}{n_t} $$

Here $N$ is the number of documents in the collection, and $n_t$ is the number of documents
that contain the term $t$.

Before we can use this, we must know what "log" does.

**log (the natural logarithm) = a function that turns a big ratio into a small number.** Every
time the ratio gets 10 times bigger, the log goes up by only about 2.3.

| Ratio $N / n_t$ | log of the ratio |
|---|---|
| 1 | 0 |
| 2 | 0.69 |
| 10 | 2.3 |
| 100 | 4.6 |
| 1,000,000 | 13.8 |

"Natural" names the base of the logarithm, the number e ≈ 2.718. It is the default in maths
libraries, including Python's `math.log`. Put simply, the log keeps a huge range of ratios on a
small, usable scale, so one rare word adds a strong signal without swamping everything else.

Now let's work out the two words of our query, in our collection of 1,000,000 documents.

`the` appears in all of them. So $n_t = N$, the ratio is 1, and its IDF is log 1 = **0**.

`Pequod` appears in 3 of them. The ratio is 1,000,000 / 3 ≈ 333,333, and its IDF is
log 333,333 ≈ **12.7**.

In simple words, the log squashes a big ratio into a small number. A term in 3 of a million
documents gets about 12.7. A term in half of them gets log 2 ≈ 0.7. A term in all of them gets
0.

Now we score our two documents again. Each word's count is multiplied by that word's IDF:

| | the (IDF 0) | pequod (IDF 12.7) | TF-IDF score |
|---|---|---|---|
| Document A | 3 × 0 | 0 | **0** |
| Document B | 1 × 0 | 1 × 12.7 | **12.7** |

Document B wins. The answer is correct.

Notice what happened to `the`. Its contribution vanished automatically. We never needed a list
of **stopwords**, the very common words like "the", "of" and "and" that carry almost no meaning.
The maths deletes them for free. Meanwhile a single mention of `Pequod` is a strong signal.

**Note:** Acme's knowledge base has fingerprints of exactly this kind. The error code `SSO-4012`
("SAML assertion expired") appears on just one of Acme's 40,000 pages, page 17,450. Its IDF is
log(40,000 / 1) ≈ 10.6. A word like "plan", on about half the pages, gets 0.7. So
counting nails the code. As Chapter 7 showed, a dense embedding tends to treat `SSO-4012` and
`SSO-4021` as nearly the same thing. To a counting method they are two completely different
words, and only one of them is in the query.

TF-IDF is still the right answer for a surprising number of small problems: tagging,
near-duplicate detection, keyword extraction, and quick search over ten thousand documents. It
has no training step, no GPU, and no model version to manage.

---

## What is BM25

**BM25 = TF-IDF + saturation for repeated words + a penalty for long documents.**

The name means Best Matching 25. It came out of the Okapi project in the 1990s, and it is still
the default ranking function in the most widely used keyword search engines. TF-IDF has two
flaws left, and BM25 fixes both.

**Flaw 1: term frequency should saturate.** Under plain TF, a document that mentions "Pequod"
100 times scores ten times higher than one that mentions it 10 times. But it is not ten times
more about the Pequod. After the first several mentions, more mentions add almost nothing. BM25
replaces the raw count with a curve that flattens out. A setting called **k1** (typically
1.2–2.0) controls how quickly it flattens.

Here is that curve with k1 = 1.2, for a document of average length:

| Mentions of "Pequod" | TF-IDF counts it as | BM25 counts it as |
|---|---|---|
| 1 | 1 | 1.00 |
| 10 | 10 | 1.96 |
| 100 | 100 | 2.17 |

The BM25 column can never pass k1 + 1 = 2.2. That is it. The hundredth mention is worth almost
nothing.

**Flaw 2: long documents cheat.** A 50-page document contains more of every word, so it wins on
raw counts. BM25 compares each document's length with the average length in the collection. A
setting called **b** (typically 0.75) controls how hard long documents are penalised. With
b = 0, length is ignored. With b = 1, the penalty is at full strength.

Put together:

$$ \text{BM25}(q,D) = \sum_{t \in q} \text{IDF}(t) \cdot \frac{f(t,D)\,(k_1+1)}{f(t,D) + k_1\left(1 - b + b\,\frac{|D|}{\text{avgdl}}\right)} $$

Do not memorise it. Read what it *says*, one piece at a time:

- $\sum_{t \in q}$ means: go through the words $t$ of the query $q$, and add up their scores.
- $\text{IDF}(t)$ weights each word by how rare it is.
- $f(t,D)$ is how often the word appears in document $D$.
- The fraction with $k_1$ in it gives diminishing returns for repeated mentions.
- $|D| / \text{avgdl}$ is this document's length divided by the average document length. The
  $b$ next to it decides how much a long document is penalised.

Every piece corresponds to a sentence of common sense.

---

## An example: finding the Pequod

Let's run BM25 on our query, *"the Pequod"*. Our collection still has 1,000,000 documents, and
the average document is 1,000 words long.

**Step 1:** Split the query into words: `the` and `pequod`.

**Step 2:** Look up how many documents contain each word: all 1,000,000 for `the`, and 3 for
`pequod`.

**Step 3:** Turn each of those counts into an IDF weight: about 0 for `the`, and about 12.6 for
`pequod`. (BM25 uses a slightly smoothed IDF. The code below shows it.)

**Step 4:** For each document, count each query word, and pass that count through the
flattening curve.

**Step 5:** Adjust each count for the document's length.

**Step 6:** Multiply by the IDF, add up across the query words, and sort the documents by score.

First, our two documents from before. Bag-of-words put Document A first, which was wrong. BM25
gives Document A about 0, because only `the` matched. It gives Document B about 21.2. Document B
wins, just as it did with TF-IDF. (B scores higher here than its TF-IDF 12.7 because it is only
4 words long, far shorter than average, so its one mention counts for more.)

Now a harder pair. Both of these documents mention the Pequod exactly 4 times:

- **Document E:** a 200-word encyclopedia entry about the Pequod.
- **Document F:** a 10,000-word history of whaling that names the Pequod 4 times in passing.

TF-IDF gives both the same score, 4 × 12.7 ≈ 50.9. It cannot tell them apart. But a reader
looking for the Pequod clearly wants Document E.

BM25 sees that Document F is ten times longer than average, and Document E is five times
shorter. It gives E about 24.7 and F about 8.3. Document E wins. The answer is correct.

Put into words, the arithmetic says: *"Four mentions in 200 words means the Pequod is the
subject. Four mentions in 10,000 words means it is a passing reference."*

---

## Where counting fails

Everything above shares one blind spot: **it can only match terms that literally appear.** This
is called the **vocabulary mismatch problem**, and it is the reason the rest of Part II exists.

Let's see it on the question Acme's users actually ask. A user types *"Can our team log in with
our company accounts?"* Page 212 of Acme's knowledge base answers it: "SSO is included on the
Pro plan." The question and the page share no useful words. BM25 scores page 212 near zero and
returns something else. The answer is wrong, and BM25 cannot even tell that it missed.

| Query | Document says | BM25 |
|---|---|---|
| "car" | "automobile" | No match |
| "how to fix a leaking tap" | "repairing dripping faucets" | Almost no match |
| "¿cómo reinicio?" | "how to restart" | No match |
| "ML model deployment" | "shipping neural networks to production" | Weak |
| "Can our team log in with our company accounts?" | "SSO is included on the Pro plan." | No match |

Classical information retrieval spent decades attacking this. It tried query expansion (adding
related words to the query), synonym dictionaries, stemming (cutting words to their root, so
"repairing" matches "repair") and relevance feedback (using the top results to find more search
words). Each helped a little. Each needed hand-maintained resources that went stale.

Embeddings solve it by construction. "car" and "automobile" simply land near each other in the
Map Room, and nobody wrote a synonym list.

In simple words, counting gives us exactness without understanding. Embeddings give us
understanding without exactness.

---

## BM25 vs embeddings

| | BM25 | Dense embeddings |
|---|---|---|
| Training data needed | None | A lot, already spent by the model maker |
| Hardware to index | A laptop CPU | Usually a GPU |
| Exact codes, names, jargon (`SSO-4012`) | Excellent | Unreliable |
| Paraphrases ("log in with company accounts") | Misses | Finds |
| Other languages | Misses | Finds, with a multilingual model |
| Explains why a result matched | Yes: "matched on *Pequod*" | No |
| Re-index when a model is replaced | No model to replace | Every time |

**Advantages of BM25**

- It needs no training data, no GPU, no embedding model, and no re-indexing when a model is
  deprecated.
- It handles identifiers, error codes, names and jargon perfectly.
- It is exact, explainable, and fast.

**Disadvantages of BM25**

- It cannot match a word the document does not contain.
- It ignores word order, so it cannot tell "dog bites man" from "man bites dog".
- Its two settings, k1 and b, need checking for your kind of documents.

> **Why this matters in 2026.** Any dense retrieval system that has not been benchmarked against
> BM25 on the real query distribution is a system whose quality nobody actually knows.

---

## When to use which one

We must use **BM25** when queries contain identifiers, error codes, product names or rare jargon,
when we have no training data, or when we need to explain why a result matched.

We must use **embeddings** when users describe what they want in their own words, or ask in a
different language from the documents.

Many strong systems use both, and Chapter 40 shows how to combine them. Whatever we build, we
measure it against BM25 on our own queries first.

---

### Under the hood

```python
import math
from collections import Counter

def bm25_score(query_terms, doc_terms, df, N, avgdl, k1=1.2, b=0.75):
    tf, dl, score = Counter(doc_terms), len(doc_terms), 0.0
    for t in query_terms:
        if t not in tf:
            continue
        idf = math.log(1 + (N - df[t] + 0.5) / (df[t] + 0.5))   # smoothed IDF, never negative
        num = tf[t] * (k1 + 1)
        den = tf[t] + k1 * (1 - b + b * dl / avgdl)
        score += idf * num / den
    return score

# Our collection: 1,000,000 documents, 1,000 words long on average
N, avgdl = 1_000_000, 1_000
df = {"the": 1_000_000, "pequod": 3}              # how many documents contain each word
query = ["the", "pequod"]

doc_a = "the whale swam past the ship in the storm".split()
doc_b = "ahab commanded the pequod".split()
bm25_score(query, doc_a, df, N, avgdl)            # → 0.000001  (only "the" matched)
bm25_score(query, doc_b, df, N, avgdl)            # → 21.2

doc_e = ["pequod"] * 4 + ["other"] * 196          # 200-word encyclopedia entry
doc_f = ["pequod"] * 4 + ["other"] * 9_996        # 10,000-word history of whaling
bm25_score(["pequod"], doc_e, df, N, avgdl)       # → 24.7
bm25_score(["pequod"], doc_f, df, N, avgdl)       # → 8.3
```

The function is the six steps from our example. `df` holds the document counts, `idf` turns them
into weights, `num / den` is the flattening curve with the length adjustment, and `score` adds up
across the query words.

Two production notes.

First, the IDF line. The classic BM25 IDF is log((N − n_t + 0.5) / (n_t + 0.5)). It goes
*negative* for any term in more than half the documents, so a common query word would lower a
document's score. Adding 1 inside the log, as Lucene does and as our code does, keeps every
weight positive. (The plain log(N / n_t) from the TF-IDF section never goes negative. It just
lacks the 0.5 smoothing.)

Second, tuning. Raise $b$ toward 1.0 when documents vary wildly in length. Lower it toward 0 for
chunks of uniform size. Many teams run BM25 over RAG chunks (the short passages a document is cut
into before search, Chapter 50) that are all about the same size. They often find $b \approx 0.3$
beats the default, because length normalization is solving a problem they no longer have.

---

### What people get wrong

**"BM25 is obsolete."** It is the strongest zero-training baseline in retrieval, and it is a
component of many of the best production systems.

**"BM25 has no parameters to tune."** It has two that matter, k1 and b. Defaults tuned for
1990s full-length documents are often wrong for 300-word RAG chunks.

**"TF-IDF and BM25 are basically the same."** The saturation curve makes a real difference on
documents with repeated terms. That means spam, transcripts, and anything auto-generated.

**"I need stopword removal."** IDF handles it. Aggressive stopword lists actively break phrase
queries like *"to be or not to be"*, which is made entirely of stopwords.

---

### Ninja notes

Two refinements you will meet in serious systems.

**BM25F** extends BM25 across weighted fields, so a match in the title counts more than one in
the body. For structured content such as product catalogues, documentation with headings and
support tickets, this is often a bigger win than swapping in a neural model. It costs nothing at
query time.

**Impact-ordered posting lists with WAND / block-max WAND** let the engine skip documents that
provably cannot make the top-k. This is how a single machine serves BM25 over a hundred million
documents in single-digit milliseconds. The same machinery is what makes learned sparse retrieval
(SPLADE, Chapter 39) practical: SPLADE produces weights that plug straight into this decades-old,
hyper-optimised infrastructure, though its flatter weights defeat some of the skipping.

---

### Key takeaways

- **BM25 = TF-IDF + saturation (k1) + length normalization (b).** It remains the baseline to
  beat.
- One-hot makes every word orthogonal to every other, so it has no notion of similarity at all.
- Bag-of-words counts terms and throws away order.
- TF-IDF weights rare terms higher. IDF = log(N / n_t) with the natural log, so a term in 3 of a
  million documents gets 12.7 and stopwords get 0 automatically.
- Counting nails exact tokens like `SSO-4012` that dense embeddings blur.
- All of these methods fail on vocabulary mismatch. That failure is what embeddings fix.
- Tune $b$ down for uniform RAG chunks, and consider BM25F for structured content.

### What's next

[Chapter 9](./09-word2vec.md) takes the idea that a word's meaning comes from the company it
keeps, and turns it into the first genuinely useful dense embeddings.

We now know how search worked by counting, why rare words carry the signal, and the one wall
that no amount of counting can climb.
