---
title: "From Things to Numbers: The Leap of Embedding"
chapter: 2
part: "Part I — Foundations: What a Vector Actually Is"
slug: the-leap-of-embedding
readingTime: "11 min"
summary: "An embedding is a learned function from a thing to a place. Here is what 'learned' actually buys you, and what it quietly costs."
tags: [foundations, embeddings, intuition]
prev: 01-what-is-a-vector
next: 03-geometry-of-meaning
---

# From Things to Numbers: The Leap of Embedding

**The one-paragraph version.** An embedding is a list of numbers that a model gives to a thing,
such as a sentence, a photo or a product, so that *similar things come back as nearby lists*.
Nobody writes the rules for this by hand. A model learns them from millions of examples. The
arrangement it learns is richer than anything a person would design, and the price is that
nobody can read it directly. Every embedding also squeezes a whole page into a few hundred
numbers, so something is always lost.

In this chapter, we will learn what an embedding is and how a model learns where to put each
page in the Map Room. We will see why that lets a search find a page that shares no words with
the question. We will also see what can be embedded, the two properties every embedding needs,
and what an embedding quietly throws away.

We will cover the following:

- What is an embedding
- How a model learns where to put things
- An example: a question with no matching words
- What we can embed
- The two properties every embedding needs
- What an embedding throws away

---

## What is an embedding

In Chapter 1 we chose the numbers ourselves: bitterness, acidity and body. That worked for
coffee and failed for sentences. So now we hand the job to a model.

Before we go further, we must know what a neural network is.

**Neural network = a model with a huge number of adjustable numbers, learned from data.**

A *model* here is simply a program whose behaviour was learned from examples instead of written
out rule by rule. In simple words, a neural network is one big calculation with millions of
knobs. Text goes in, numbers come out, and the knobs decide which numbers. Nobody turns the knobs
by hand. Training turns them. (People call these knobs **weights**, and Chapter 9 looks at them
closely.)

Now we can define the main idea of this chapter.

**Embedding = the list of numbers a model gives to a thing, arranged so that similar things get
nearby lists.**

Let's break it down:

- **The thing:** a sentence, a page, a photo, a song, a customer.
- **The model:** a neural network trained for exactly this job. It is called an **embedding
  model**.
- **The list:** a vector with a fixed number of positions, often 384, 768 or 1,024.
- **The promise:** things that are similar come out as dots near each other in the Map Room.

**Note:** People use the word "embedding" in three ways. The vector itself is an embedding. The
model that makes it is an embedding model. The Map Room where all the vectors live is the
*embedding space*. When the meaning is not clear from context, we will say which one we mean.

---

## How a model learns where to put things

Imagine we must seat ten thousand guests in an enormous banquet hall. The only instruction is
this: *people who would enjoy talking to each other should sit near each other.*

We cannot ask everyone. So we do something clever. We collect records of who has talked to whom
in the past. Then we shuffle the seats over and over. People who talk a lot get pulled slightly
closer. Strangers get nudged slightly apart. We keep going until the whole hall settles.

When it settles, something remarkable has happened. The chemists are in one corner. The jazz
musicians are in another. And *between* them, in a spot nobody planned, sit the four people who
play saxophone and run a lab. The seating chart learned a structure nobody asked for.

Think of it like this:

- The banquet hall is the Map Room.
- Each guest is one page of the Great Library.
- The records of who talked to whom are the training data: pairs of texts we know belong
  together.
- Shuffling the seats is training.
- A guest's final seat is that page's embedding.

Now let's say the same thing in real terms. Training an embedding model goes like this.

**Step 1:** Take a pair of texts that we know mean nearly the same thing, such as a question and
the page that answers it.

**Step 2:** Pass both texts through the model and get two vectors.

**Step 3:** Measure how far apart the two dots are, and how far each is from unrelated texts.

**Step 4:** Nudge the model's adjustable numbers a tiny amount, so the pair moves a little closer
and the unrelated texts move a little farther away.

**Step 5:** Repeat millions of times, with millions of different pairs.

When training ends, the adjustable numbers are frozen. From then on, the model is a fixed recipe:
text in, vector out.

Put simply, the model is never told what any position in the list means. It is only told which
things belong together, and it finds its own way to arrange them. Chapter 12 shows the real
training loop, and it is simpler than it sounds.

---

## An example: a question with no matching words

Let's put an embedding model to work in the Great Library, Acme's 40,000-page knowledge base.

A user types the everyday version of our running question:

> *"Can our team log in with our company accounts?"*

This is the single sign-on question in plain words. The answer lives on page 212, *"Single
sign-on is included on the Pro plan"*, with the catch on page 1,140: teams under 50 seats also
need the Security add-on.

Let's first see what a search that matches words does. It looks for pages that share words with
the question. Page 212 shares none. It has no "team", no "log", no "company" and no "accounts".

But a page called *"Closing a company account"* says: "When a team cancels, we delete its user
accounts after 30 days." Its title and text together share three words with the question:
"team", "company" and "accounts". So the word-matching search ranks it first.

The Librarian hands that page to the Scholar. The Scholar writes: "Your team's accounts are
deleted 30 days after you cancel." The answer is wrong. The user asked how to log in, not how to
leave.

Now let's see what an embedding search does. Every page was turned into a vector in advance. The
question is turned into a vector now. The Librarian then looks for the nearest dots.

```
embed("Can our team log in with our company accounts?")
    lands near  embed("Single sign-on is included on the Pro plan.")
    and far from embed("Refunds are processed within 14 days.")
```

Why does that happen? During training, the model saw phrases like "log in with your company
account" and "single sign-on" used again and again in the same kinds of places, for the same
purpose. So it learned to seat them together, even though they share no words.

The Librarian brings back page 212. The Scholar answers: "Yes. Single sign-on lets your team log
in with your company accounts, and it is included on the Pro plan." The answer is correct, as far
as it goes. The catch on page 1,140 is harder to find, and "What an embedding throws away" below
shows why.

That is the leap: **from matching words to matching meaning.**

**Note:** Matching meaning does not always win. When a user searches for an exact error code
such as `SSO-4012`, matching the exact characters wins. Chapter 7 shows that case, and Chapter 40
shows how to use both kinds of search together.

---

## What we can embed

Almost anything, as long as we can collect examples of what "similar" should mean.

| Thing | What similarity means | Typical model family |
|---|---|---|
| Sentence, paragraph | Same topic, same intent | Sentence transformers |
| Whole document | Same subject matter | Long-context encoders |
| Image | Same content or style | CLIP, SigLIP, DINOv2 |
| Page of a PDF (as a picture) | Answers the same question | ColPali, ColQwen |
| Audio clip | Same speaker, same sound | CLAP, wav2vec |
| Source code | Same functionality | Code embedding models |
| A user | Same taste | Recommender two-tower models |
| A node in a graph | Same role in the network | node2vec, GNNs |

Do not worry about the names in the last column. The ones that matter get their own chapters later
in the book. CLIP, SigLIP, ColPali and ColQwen, for example, are model families for pictures and
page images that we meet in Part VI.

The recipe is the same every time:

1. Choose what "similar" should mean.
2. Collect examples of it.
3. Train a model to place those examples close together.

---

## The two properties every embedding needs

A useful embedding has exactly two properties.

**Property 1: Fixed size.** A three-word question and a two-thousand-word page both come out as
the same number of numbers, say 768.

This is what makes everything after it possible. We can store all the vectors in one big table.
We can compare any two with the same arithmetic. We can build one Card Catalog over all of them.
If every page got a list of a different length, fast search would be close to impossible.

Here is the storage bill for the whole Library. 40,000 pages × 768 numbers × 4 bytes =
122,880,000 bytes, about 123 MB. That fits comfortably in a laptop's memory.

**Property 2: Geometry that follows meaning.** Distance in the Map Room should match how
different two things are. This is the part that is learned, and the part that can fail.

A model trained on general web text is excellent at telling whether two sentences share a topic.
It can be mediocre at telling whether two contract clauses create the same obligation, because
nobody trained it on that difference. Acme's scanned contracts are exactly where this bites.

> **A model's embedding space encodes the idea of similarity it was trained on, not the one in
> your head.** More retrieval bugs trace back to this single sentence than to any index setting.

---

## What an embedding throws away

Now, the question is, how much can 768 numbers really hold?

Let's do the arithmetic on one page. Acme's *Plans and pricing* page has 2,000 words. At roughly
6 characters per word, that is about 12 KB of text. It holds hundreds of distinct facts: prices,
seat limits, which features come with which plan, and the exceptions.

The embedding of that page is 768 numbers at 4 bytes each: 3,072 bytes, about 3 KB.

The vector is not a copy of the text. It is a summary, trained to capture what makes pages
similar or different in general. The model decided what to keep, and nobody told it which
details we would later care about.

What survives is the *gist*: the topic, the domain, the tone, the main claims. What is routinely
lost:

- a single number buried in paragraph nine,
- an exact code such as `SSO-4012`, or a product **SKU** (stock-keeping unit, a product's
  catalogue code),
- a one-sentence exception, such as "for teams under 50 seats",
- the difference between "shall" and "may" in a contract.

Look at the third item. That is exactly what happens to page 1,140. It is a long page about the
Security add-on, mostly about audit logs, IP allow-lists and data retention. It mentions SSO in one
sentence: "On Pro, SSO needs the Security add-on for teams under 50 seats." The page's vector says
"security add-on features", and that one sentence barely moves it. That is the half of our running
question that a careless search drops.

This is not a flaw in one particular model. Every embedding that squeezes a page into one vector
makes this trade. And it is the reason for three whole stretches of this book:

- **Chunking** (Chapter 50): cut long pages into smaller pieces before embedding, so each vector
  has less to summarise.
- **Hybrid search** (Chapter 40): keep a keyword search running alongside, because keyword search
  never loses the SKU.
- **Multi-vector retrieval** (Part V): stop insisting on one vector per page, and keep many.

In simple words, one vector per page is a summary, and summaries skip details.

---

### Under the hood

Using an embedding model takes three lines. Understanding what it did is the rest of this book.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-small-en-v1.5")   # a small model: 384 dimensions
v = model.encode("Can our team log in with our company accounts?")
v.shape    # → (384,)
```

Means, we loaded an embedding model that someone has already trained, gave it our question, and
got back one vector of 384 numbers. At 4 bytes each, that is 1,536 bytes.

Two properties of this function are worth learning right away:

- It is **deterministic**. The same text always gives the same vector for a given model version,
  apart from tiny rounding differences between machines. This is why we can embed all 40,000
  pages once, store the vectors, and reuse them.
- It is **model-specific**. Vectors from model A and model B live in different Map Rooms that
  cannot be compared. Mixing them produces nonsense that *looks* like numbers. Chapter 63 is about
  the pain this creates when we switch models.

---

### What people get wrong

**"I will embed the whole 40-page contract as one vector."** You will get a vector that means
"this is a software contract", which retrieves well for no useful question. Chunk it.

**"Embeddings understand my domain."** They understand what they were trained on. If your corpus
is full of internal project codenames, a general model sees them as gibberish **tokens** (the
word-fragments a model actually reads, explained in Chapter 10). Hybrid search or fine-tuning
(further training on your own examples) fixes this. Hope does not.

**"I can compare vectors from two different models."** You cannot. Not even if both have 768
dimensions. Different Map Rooms, different floor plans.

---

### Ninja notes

Embedding models are not neutral. They inherit the biases, the vocabulary and the recency of
their training data. A model trained before Acme named its "Security add-on" has never seen that
phrase used as a product name. It will seat it near generic security text instead of near the
pricing pages. Before you blame your index, check whether your model has ever seen your domain's
words.

The cheapest diagnostic in all of retrieval: embed twenty domain terms, find each one's nearest
neighbours, and read them. If the neighbours are nonsense, no amount of index tuning will save
you.

---

### Key takeaways

- **Embedding = the list of numbers a model gives to a thing, arranged so that similar things get
  nearby lists.**
- **Neural network = a model with adjustable numbers learned from data.** Training nudges those
  numbers until pairs that belong together sit close.
- Embeddings match meaning, not words. That is how "log in with our company accounts" finds the
  single sign-on page.
- The two required properties: fixed size, and geometry that follows meaning.
- "Similar" means whatever the model was trained to think it means. Check it on your own data.
- Every embedding compresses. What it discards is what your retrieval will miss.
- Vectors from different models are never comparable.

### What's next

We now have every page placed in a room. [Chapter 3](./03-geometry-of-meaning.md) walks around
that room and looks at what its neighbourhoods, directions and empty spaces actually mean.

Now we know what an embedding is, how a model learns one, and why it finds meaning but can lose
the details.
