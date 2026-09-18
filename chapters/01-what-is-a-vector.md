---
title: "What Is a Vector, Really?"
chapter: 1
part: "Part I — Foundations: What a Vector Actually Is"
slug: what-is-a-vector
readingTime: "10 min"
summary: "A vector is a list of numbers. It is also a place. Once you see both at once, everything else in this book follows."
tags: [foundations, intuition, geometry]
prev: null
next: 02-the-leap-of-embedding
---

# What Is a Vector, Really?

**The one-paragraph version.** A vector is an ordered list of numbers, like `[3, 7]`, or
`[0.12, -0.44, 0.98, ...]` with 768 entries. That is the whole definition. What makes it
powerful is that the same list can also be read as a *location*: `[3, 7]` is a point three
steps east and seven steps north. If similar things get similar numbers, then finding similar
things becomes finding nearby points. Every technique in this book comes from holding those two
readings in our head at once: a description, and a place.

In this chapter, we will learn what a vector is, why turning things into numbers lets a computer
calculate "similar", and how one list of numbers can be both a description and a place. We will
also meet the five characters who carry this book from the first page to the last.

We will cover the following:

- What is a vector
- Why we turn things into numbers
- An example: which coffee is most like an espresso?
- Two ways to read the same list
- What "dimensions" means
- Meet the cast of this book
- Why hand-picked numbers are not enough

---

## What is a vector

**Vector = an ordered list of numbers.**

`[9, 4, 8]` is a vector. `[0.12, -0.44, 0.98]` is a vector. A list of 768 numbers is a vector
too.

The word *ordered* matters. `[9, 4, 8]` and `[8, 4, 9]` are different vectors, because each
position has its own job. The first number always means the same kind of thing. So does the
second, and the third.

That is it. Nothing more mysterious is hiding in the definition.

---

## Why we turn things into numbers

Now, the question is, why would anyone describe a sentence, a photo or a song as a list of
numbers?

Because a computer cannot compare opinions, but it can compare numbers extremely fast.

"Is this document similar to that one?" is a matter of opinion. "Is this point close to that
point?" is arithmetic. A modern computer does that arithmetic billions of times per second.

So the whole field rests on one trick.

> **The core trick:** if we can describe things as numbers so that similar things get similar
> numbers, then *searching for similar things becomes searching for nearby points*. The
> remaining 65 chapters are about doing this well, and doing it fast.

Let's watch the trick work on something small.

---

## An example: which coffee is most like an espresso?

Suppose we run a coffee shop, and we want to describe every coffee we sell. We could write a
paragraph for each one, but paragraphs are hard to compare. So we pick three measurements and
score each coffee from 0 to 10:

| Coffee | Bitterness | Acidity | Body |
|---|---|---|---|
| Espresso | 9 | 4 | 8 |
| Filter | 4 | 7 | 3 |
| Cold brew | 6 | 2 | 7 |
| Latte | 3 | 2 | 6 |

Espresso is now `[9, 4, 8]`. That is a vector.

A customer walks in. "Your espresso machine is broken. What tastes closest to an espresso?"

Let's first answer the way most of us would, by reading the words. The menu describes a latte
as "espresso with steamed milk". The word *espresso* is right there. So the latte must be the
closest.

The answer is wrong. A latte is mostly milk. It tastes nothing like a straight espresso.

Now, let's answer with the numbers. We subtract each coffee from espresso, one position at a
time:

```
espresso - cold brew = [9-6, 4-2, 8-7] = [3,  2, 1]   → small gaps everywhere
espresso - latte     = [9-3, 4-2, 8-6] = [6,  2, 2]   → a big gap in bitterness
espresso - filter    = [9-4, 4-7, 8-3] = [5, -3, 5]   → big gaps everywhere
```

Cold brew has the smallest gaps, so cold brew is the closest match. The answer is correct, and
nobody had to taste anything.

In simple words, we turned *"similar"* into *"close"*. Matching the words picked the latte.
Matching the numbers picked the cold brew.

**Note:** This is, in miniature, the difference between keyword search and vector search.
Keyword search matches the words on the page. Vector search matches what those words describe.
Neither one always wins. Chapters 7 and 40 explain when each one does, and how to use both.

---

## Two ways to read the same list

The list `[9, 4, 8]` can be read in two ways.

**As a description.** "This drink scores 9 on bitterness, 4 on acidity and 8 on body." Each
number is an *attribute*, and its position in the list tells us which attribute it is.

**As a place.** Think of it like a room with three directions in it:

- Bitterness runs along the floor, from the left wall to the right wall.
- Acidity runs from the door to the back wall.
- Body runs from the floor up to the ceiling.

Now `[9, 4, 8]` is one exact spot in that room: 9 steps to the right, 4 steps in from the door,
8 steps up. Each coffee is a dot floating in the room. The distance between two dots is how
differently two coffees taste. Similar coffees float close together. An unusual coffee floats
alone in a corner.

We will call that room **the Map Room**. Everything we ever want to search will end up as a dot
in a Map Room: a help article, a photograph, a song, a customer, a page of a scanned contract.
Finding relevant things becomes finding nearby dots.

**Note:** Mathematicians add a third reading, where a vector is an *arrow* from the corner of
the room to the dot. That is where the word comes from (Latin *vehere*, to carry). The arrow
reading matters when we add vectors together or measure the angle between them, which Chapter 4
does. For search, the *place* reading does the heavy lifting, so it is the one we will use.

---

## What "dimensions" means

**Dimensions = how many numbers are in the list.**

Our coffee vector has three numbers, so it is **3-dimensional**. People also write "3-d", or
`d = 3`. We can picture three dimensions easily, because we live in three.

The vectors this book is really about have 384, 768, 1024, 1536 or 3072 numbers.

Now, the question is, how do we picture a 768-dimensional room?

We don't, and we should stop trying. The attempt is what makes people feel this topic is beyond
them. Here is the good news: **we never need to picture it.** Every operation we will ever do
on a 768-dimensional vector is the arithmetic we just did on the coffee. We subtract position by
position, 768 times instead of 3.

So the mental habit that works is this: *think in three dimensions, compute in 768, and stay
alert for the few places where high dimensions behave strangely.* Chapter 6 is all about those
few places.

---

## Meet the cast of this book

This book explains its ideas with the help of five characters. Let's meet all of them now, so
that every later chapter feels familiar.

To keep things concrete, our collection of documents for the whole book is **Acme's support
knowledge base**: 40,000 pages of help articles, pricing pages, release notes and scanned
contracts. Acme sells software to businesses.

**The Great Library** is our corpus, the full collection of things we want to search. Here, it
is Acme's 40,000 pages. Each page is one book on a shelf.

**The Map Room** is the embedding space, the room of positions a model learns for our pages. We
define *embedding* properly at the end of this chapter. Every book in the Library gets one dot in
the Map Room, placed so that books about similar things sit near each other.

**The Card Catalog** is the index. It is the structure that lets us find nearby dots *without*
checking every single dot. We build it in Parts III and IV.

**The Librarian** is the retriever. We hand the Librarian a question, and the Librarian uses the
Card Catalog to fetch the few books whose dots sit closest to the question's dot.

**The Scholar** is the LLM.

**LLM = Large Language Model.** An LLM is a program trained on a vast amount of text until it
can read a passage and write fluent language about it. Ask it a question and it writes an
answer, piece by piece. Assistants such as ChatGPT, Claude and Gemini are built on LLMs. In our
story, the Scholar reads the books the Librarian brings back, and writes the answer.

Here is the whole book in one sentence: *we teach a Librarian to find the right books in a
Library too big to read, by giving every book a place in a Map Room and building a Card Catalog
over those places, so that the Scholar can answer from the right pages.*

And here is the question we will keep handing to the Librarian, chapter after chapter:

> *"Does the Pro plan include single sign-on?"*

**Single sign-on (SSO)** means logging in to Acme with your company account instead of a
separate Acme password. The answer lives in two places. Page 212 of the Library says SSO is
included on the Pro plan. Page 1,140 adds a catch: on Pro, SSO needs the Security add-on for
teams under 50 seats.

A good Librarian brings back both pages. A careless one brings back only page 212, and the
Scholar confidently gives half an answer. We will watch this question go wrong in many different
ways, and fix it one chapter at a time.

---

## Why hand-picked numbers are not enough

Our coffee vector works because we, as humans, chose three meaningful measurements. Let's try
the same thing on a sentence from page 212 of the Library:

> *"Single sign-on is included on the Pro plan."*

What are its three measurements? Bitterness? Obviously not. Topic? Formality? Length? None of
those separates it from *"Two-factor login is included on the Basic plan"*, which has a similar
topic, the same formality and the same length, but states a completely different fact. We would
need thousands of hand-picked measurements to tell every sentence apart from every other one,
and we would still miss things.

This is exactly the problem machine learning solved. Instead of a human deciding what each
position in the list means, a model **learns** hundreds of positions from data. It arranges them
so that things people consider similar end up close together.

Nobody can tell us what position 412 of a modern embedding model "means". Usually it means
nothing we could put a name to. What matters is that the arrangement *as a whole* puts similar
things near each other.

That learned arrangement is called an **embedding**, and it is the subject of Chapter 2.

---

### Under the hood

In code, a vector is a flat array of `float32` values. A `float32` is the standard way a
computer stores a number with a decimal point, using 4 bytes.

```python
import numpy as np

espresso  = np.array([9.0, 4.0, 8.0], dtype=np.float32)
cold_brew = np.array([6.0, 2.0, 7.0], dtype=np.float32)
latte     = np.array([3.0, 2.0, 6.0], dtype=np.float32)

# Straight-line distance between two dots in the Map Room
np.linalg.norm(espresso - cold_brew)   # → 3.742
np.linalg.norm(espresso - latte)       # → 6.633
```

`np.linalg.norm` squares each gap, adds the squares up and takes the square root. In simple
words, it lays a ruler between two dots. Cold brew is 3.7 steps from espresso, and latte is 6.6
steps away. The code agrees with our table.

Now the storage bill. A 768-dimensional `float32` vector takes `768 × 4 = 3,072 bytes`, about
3 KB.

Remember that number. In Chapter 60 we multiply it by a hundred million, and it turns into a
real line on the monthly bill.

---

### What people get wrong

**"A vector is an arrow."** It can be, but for search it is far more useful as a point. Nobody
draws arrows in a 768-dimensional room.

**"More dimensions is more accurate."** No. More dimensions is more *capacity*, and capacity you
do not need is pure cost, paid in RAM, in latency and in bandwidth. A well-trained
384-dimensional model routinely beats a badly trained 1536-dimensional one.

**"Each number means something on its own."** Almost never. In a learned embedding, meaning
lives in the *relationships between* vectors, not in any single position. Hunting for the
meaning of dimension 412 is the vector-search version of looking for a whole sentence inside a
single neuron.

---

### Ninja notes

The number format of a vector is an engineering decision, not a detail. `float32` is the default.
`float16` halves memory with almost no quality loss for most models. `int8` gives 4× compression
and needs calibration. A single *bit* per dimension (Chapter 26) gives 32× compression and still
keeps enough structure to shortlist candidates. A large part of production vector search is
choosing how few bits you can get away with, then rescoring the survivors at full precision.

---

### Key takeaways

- **Vector = an ordered list of numbers.** The same list is also a point in a space.
- If similar things get similar numbers, *similarity search becomes proximity search*, and
  proximity is cheap to compute.
- **Dimensions = how many numbers are in the list.** Reason in 3-d, compute in 768-d.
- The cast: the **Great Library** (the corpus), the **Map Room** (embedding space), the **Card
  Catalog** (the index), the **Librarian** (the retriever) and the **Scholar** (the LLM).
- Our running question is *"Does the Pro plan include single sign-on?"*, and its answer is
  split across two pages.
- Hand-picked dimensions do not scale beyond toy problems. Learned ones do.
- A 768-d `float32` vector is about 3 KB. That number will matter later.

### What's next

So far we have chosen the numbers by hand. In [Chapter 2](./02-the-leap-of-embedding.md) we hand
that job to a model, and see what it means for a machine to decide where a sentence lives in the
Map Room.

We now know what a vector is, why it is both a description and a place, and who will be with us
for the rest of the book.
