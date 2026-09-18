---
title: "The Geometry of Meaning"
chapter: 3
part: "Part I — Foundations: What a Vector Actually Is"
slug: geometry-of-meaning
readingTime: "11 min"
summary: "A tour of the Map Room: neighbourhoods, directions, clusters, and the famous king − man + woman trick — including why it works less well than you have been told."
tags: [foundations, geometry, intuition]
prev: 02-the-leap-of-embedding
next: 04-measuring-similarity
---

# The Geometry of Meaning

**The one-paragraph version.** Once every page is a dot in the Map Room, the room itself has a
shape we can read. *Neighbourhoods* group similar pages. *Directions* can stand for
relationships. *Density* shows where our pages actually live, and *empty parts* show what the
Library cannot answer. The whole room is also *squashed*, so raw similarity scores mean less than
they seem. Learning to read this shape is how we build an intuition for why a retrieval system
behaves the way it does.

In this chapter, we will take a walk through the Map Room built from Acme's knowledge base. We
will see why the room is clumpy and why that makes fast search possible. We will test the famous
"king − man + woman" trick with real arithmetic. We will watch a question land where no page
lives, and learn why a similarity score of 0.8 might mean very little.

We will cover the following:

- A walk through the Map Room
- Neighbourhoods: why the room is clumpy
- Directions: relationships as arrows
- Empty parts of the room still return results
- The squashed room: anisotropy

---

## A walk through the Map Room

First, a quick reminder of the cast from Chapter 1. The **Great Library** is Acme's 40,000-page
support knowledge base. The **Map Room** holds one dot per page, placed by the embedding model
from Chapter 2. The **Librarian** fetches the pages whose dots sit nearest to a question. The
**Scholar**, our LLM, reads those pages and writes the answer.

Let's stand on the dot for page 212, *"Single sign-on is included on the Pro plan"*, and start
walking.

**Take one step.** We land on pages about the very same thing. There is page 88, the guide to
setting up single sign-on, an older copy of the pricing page, and page 212 translated into German.

**Walk for a minute.** We are now among other login pages: two-factor login, password rules, SAML
error codes (SAML is one of the standards single sign-on runs on). Same neighbourhood, different
streets.

**Walk for ten minutes.** We reach the edge of the accounts-and-admin district. Around us are
pages about adding seats, the Security add-on and its audit logs, and downloading invoices.

**Walk for an hour.** We are in a completely different part of the city: release notes for the
mobile app's dark mode, and the office holiday schedule. Nothing here has anything to do with
where we started.

That walk is the single most useful mental model in this book.

**Distance is dissimilarity, continuously.** There is no wall between "related" and "unrelated".
There is a slope.

In simple words, the farther we walk from page 212, the less a page has to do with it, and
nothing in the room marks the point where "relevant" stops.

This matters as soon as we build a real system. Many systems use a **similarity threshold**, a
cut-off score below which results are thrown away. Setting one means drawing a line somewhere on
that slope. The line is our choice, not something the room gives us, and Chapter 55 will explain
why that line moves depending on the query.

---

## Neighbourhoods: why the room is clumpy

The dots in the Map Room are not spread evenly. They are *clumpy*: dense clusters, with sparse
gaps between them.

Why? Because real content is clumpy. Acme has hundreds of pages about billing and logins, and
one lonely page about the office recycling scheme. Most of any knowledge base is about a handful
of topics.

Now, the question is, why should we care about clumps?

Because clumps are what make fast search possible. The simplest way to find the nearest pages is
to check every single dot. That is fine for 40,000 dots and far too slow for hundreds of
millions. So real systems use a shortcut.

**ANN = Approximate Nearest Neighbour search.**

- **Nearest neighbour:** the dots closest to the question's dot.
- **Approximate:** allowed to miss one now and then, in exchange for being far faster.

Think of it like finding a book in a well-run library:

- The shelves are grouped by topic, just as the dots are grouped into clumps.
- A librarian walks straight to the right section instead of reading every spine. That is the
  shortcut.
- Now and then a book sits on an odd shelf and gets missed. That is the "approximate".

This clumpiness is the reason approximate nearest neighbour search (ANN) works at all. If ten
million vectors were spread perfectly evenly across 768 dimensions, no Card Catalog could help.
We would have to check all of them. Because they are clustered, an algorithm can learn where the
clusters are and skip 99% of the room. Every Card Catalog design in Part IV is, at bottom, a way
of exploiting clumpiness.

**Note:** If a vector index (a Card Catalog) is ever *slow* despite correct settings, one likely
cause is that many of the embeddings are nearly identical. Often the text being embedded is
templated boilerplate that the model cannot tell apart, such as 5,000 release notes that share the
same long legal footer. Look at the data, not the index.

---

## Directions: relationships as arrows

The most famous demonstration in the history of embeddings is about words:

```
vec("king") − vec("man") + vec("woman") ≈ vec("queen")
```

Here `vec("king")` means "the embedding of the word *king*". Means, start at the dot for *king*,
subtract the vector for *man*, add the vector for *woman*, and we land near *queen*.

Adding and subtracting vectors works position by position, exactly like the coffee subtraction in
Chapter 1. Let's try it on toy numbers. Imagine a tiny 2-dimensional word space where, by luck,
position 1 tracks "royal" and position 2 tracks "male":

```
king  = [9, 8]        man   = [1, 8]
woman = [1, 1]        queen = [9, 1]

king − man + woman = [9 − 1 + 1,  8 − 8 + 1] = [9, 1] = queen
```

Subtracting *man* took away the "male" amount. Adding *woman* put back a person without it. The
"royal" amount was never touched, so we land on *queen*.

The claim behind the trick is that consistent relationships correspond to consistent
*directions* in the space. There would be a "gender direction", a "plural direction", a
"past-tense direction".

This is genuinely real and genuinely important. It is part of why embeddings generalise at all,
and why a model that has never seen our exact sentence can still place it sensibly. But there are
three things that rarely make it into the blog posts:

1. **The original demonstrations excluded the input words from the answer.** Without that rule,
   the nearest word to king − man + woman is very often *king* itself. The trick is real, but it
   was presented generously.
2. **It works far better for word embeddings than for sentence embeddings.** In modern sentence
   models, the room is arranged mostly by topic, and clean analogy arithmetic mostly stops
   working. Do not expect "Pro pricing page − Pro + Basic" to land on the Basic pricing page.
3. **We will almost never do vector arithmetic in production.** It is a superb teaching device and
   a poor engineering tool.

Keep the intuition, *relationships are directions*, and let go of the party trick. Chapter 9
shows where this word arithmetic came from.

---

## Empty parts of the room still return results

Some parts of the Map Room are empty. No page in the Library lives there.

Now, the question is, what happens when a question lands in an empty part?

Let's take an example. A user asks, *"Does Acme offer pet insurance?"* Acme sells software. No
page mentions pets. The question's vector lands in a part of the Map Room where no page lives.

Before we follow the Librarian, we need one piece of vocabulary. Every search asks for a fixed
number of results.

**k = the number of results we ask the search for.** The results themselves are called the
**top-k**.

In simple words, k is how many results we asked for, usually 5 or 10.

The Librarian does not say "nothing found". It cannot. We asked for the 5 nearest pages (k = 5),
and there are always 5 nearest pages, even when the nearest one is very far away. So the
Librarian comes back with 5 pages about billing, refunds and account deletion.

**Vector search always returns something.** There is no built-in "I found nothing."

Think of it like asking for a book the library does not own. A good human librarian says, "We
do not have that." Our Librarian is a nearest-dot machine, so it never says that. It hands over
the five least-unrelated books it can find.

Then the Scholar reads those five pages and writes a confident paragraph: "Pet cover can be added
from the Billing page and refunded within 14 days." The answer is wrong, and nothing in the system
knows it is wrong.

This is the single most common cause of confidently wrong answers in **RAG**. RAG stands for
retrieval-augmented generation: finding the right pages and handing them to a language model to
answer from. It is the subject of Part VII.

The fix has a simple shape: a **distance floor**. It is a minimum closeness that a page must reach
before we trust it. If even the nearest page is too far away, the system says so instead of
answering.

Let's run the same question again with a floor in place. Even the nearest page is farther away
than the floor allows. So the Librarian returns nothing, and the Scholar replies, "Acme's
knowledge base has no information about pet insurance." The answer is correct.

**Note:** Where exactly to put the floor is harder than it looks, because raw scores are not
calibrated, as the next section shows. Chapter 55 shows how to pick the floor, and adds a second
check on whether the pages actually answer the question.

---

## The squashed room: anisotropy

Here is a structural fact about real embedding spaces that surprises people.

We need one term first. Chapter 4 defines **cosine similarity** properly. For now: it is a score
where 1 means same direction (very similar) and 0 means unrelated. Higher means more alike.

Suppose we take a large sample of Acme pages and measure the cosine similarity between random,
unrelated pairs. We might expect something near zero. We will often get 0.6, 0.75, sometimes
higher. In some popular models, almost every score lands somewhere between about 0.55 and 0.95.
The model card for the E5 family (the documentation page its authors publish with the model, see
Chapter 11) says its scores mostly fall between 0.7 and 1.0.

The dots are not spread over all directions. They are crowded into a narrow cone.

**Anisotropy = the dots crowd into a narrow cone instead of spreading out in every direction.**

Think of it like a torch beam in a dark hall:

- The hall is the whole Map Room.
- The beam is the narrow cone.
- Every dot sits somewhere inside the beam.
- So even two unrelated dots point in roughly the same direction, and score high.

That means raw similarity scores are not calibrated. A cosine similarity of 0.82 does not mean
"82% similar". It might be an excellent match in one model and background noise in another.

Let's see it with toy numbers, made up for illustration. In one model, page 212 and page 88
score 0.86 with each other. Page 212 and the office holiday schedule score 0.71. That 0.71 looks
high, but it is just the floor of this squashed room. The meaning is in the gap: 0.86 against
0.71. In a different model, the same two pairs might score 0.62 and 0.18. Same order, completely
different numbers.

The correct response is not to panic. It is to *never use an absolute threshold we have not
calibrated on our own data*. What matters is the ranking, and the *gap* between the top result
and the tenth. Chapter 55 turns this into a procedure.

---

### Under the hood

Here is a five-minute experiment that teaches more than an hour of reading. It measures the
squash and the clumps on our own pages.

**Step 1:** Embed 2,000 pages, each normalized to length 1 (Chapter 5 explains why).

**Step 2:** Score 5,000 random pairs and average the scores. That is the floor of the room.

**Step 3:** For every page, find its single most similar other page, and average those scores.
That is how close real neighbours get.

**Step 4:** Compare the two numbers.

```python
import numpy as np
from sentence_transformers import SentenceTransformer

m = SentenceTransformer("BAAI/bge-small-en-v1.5")
docs = [...]                      # 2,000 pages from Acme's knowledge base (or YOUR corpus)
V = m.encode(docs, normalize_embeddings=True)   # unit length, Chapter 5

# 1. How similar are random pairs? (the anisotropy floor)
i, j = np.random.randint(0, len(V), (2, 5000))
# multiply-and-add = the dot product, Chapter 4
print("random-pair similarity:", (V[i] * V[j]).sum(1).mean())   # often 0.5–0.8

# 2. How clumpy is the space?
sims = V @ V.T                    # every page against every page: the dot product again, Chapter 4
np.fill_diagonal(sims, -1)        # ignore each page's score against itself
print("nearest-neighbour similarity:", sims.max(1).mean())
```

In simple words, the first number says how alike two random pages look. The second says how
alike true neighbours look. With the toy numbers from the last section, that would be roughly 0.71
and 0.86.

The gap between those two numbers is the model's **usable dynamic range** on our data. If it is
small, that model is a poor fit for the corpus, and no index tuning will rescue it. This one
diagnostic has saved more projects than any tuning knob.

---

### What people get wrong

**"Cosine 0.9 means very similar."** Only relative to that model's floor. (Cosine is the Chapter
4 score where 1 means "same direction".) Measure the floor first.

**"Clusters in my embedding space correspond to my categories."** Sometimes. Often the dominant
clusters correspond to *language*, *document length* or *formatting* instead. Visualise before
you assume.

**"If nothing relevant exists, search returns nothing."** It never does. It returns the k
nearest, however far away they are.

---

### Ninja notes

Dimensionality-reduction plots (UMAP, t-SNE), which squash many dimensions into a 2-d picture,
are wonderful for building intuition and dangerous for drawing conclusions. They preserve local
neighbourhoods and systematically distort global distances and cluster sizes. Two clusters that
look far apart in a UMAP plot may be adjacent in the real space.

Use these plots to generate hypotheses, such as "why is this blob separate?". Then always confirm
the hypothesis with actual distances in the full-dimensional space.

---

### Key takeaways

- **Distance in the Map Room = dissimilarity, on a continuous slope.** There is no natural
  boundary between related and unrelated.
- Real Map Rooms are clumpy, and **ANN** (approximate nearest neighbour search) depends on that
  clumpiness.
- Relationships can be directions, but analogy arithmetic is a teaching device, not an
  engineering technique.
- Vector search always returns the top-k, even when nothing relevant exists. A distance floor is
  the fix.
- **Anisotropy** squashes scores into a narrow band, so scores are not calibrated across models.
  Measure your own floor.

### What's next

We have been saying "close" and "far" loosely. [Chapter 4](./04-measuring-similarity.md) makes it
precise, with the three rulers everyone uses and the situations in which each one lies.

That is the shape of the Map Room: its slopes, its clumps, its directions, its empty parts and
its squash.
