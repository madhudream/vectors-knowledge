---
title: "Word2Vec and the Distributional Hypothesis"
chapter: 9
part: "Part II — Where Embeddings Come From"
slug: word2vec
readingTime: "12 min"
summary: "One idea — you shall know a word by the company it keeps — turned into the training trick that made dense embeddings practical, and that every modern model still uses."
tags: [word2vec, embeddings, history, negative-sampling]
prev: 08-before-embeddings
next: 10-transformers-contextual
---

# Word2Vec and the Distributional Hypothesis

**The one-paragraph version.** In 2013, word2vec showed that we can learn useful word vectors by
training a tiny neural network on a fake task: *given this word, predict the words around it*.
Nobody wanted the predictions. The researchers wanted the network's internal **weights**, which
turned out to place similar words near each other. The core trick, **negative sampling**, is
the direct ancestor of how every modern embedding model is trained. Word2vec itself is obsolete.
Its ideas are load-bearing.

In this chapter, we will learn how word2vec turns "the company a word keeps" into a vector. On
the way we will meet three terms the rest of the book uses constantly: weights, softmax and
sigmoid. We will also see the ceiling word2vec hit, which the next chapter breaks through.

We will cover the following:

- What is the distributional hypothesis
- What is word2vec
- What are weights
- How word2vec learns
- Why a softmax is too slow
- What is negative sampling
- An example: which word is most like *tesgar*?
- What word2vec could not do
- When word2vec still makes sense

---

## What is the distributional hypothesis

Let's read three sentences:

> The *tesgar* was rich and dark, and I drank two cups before noon.
>
> She ground fresh *tesgar* beans every morning.
>
> Too much *tesgar* keeps me awake.

We now know roughly what *tesgar* means. We have never seen the word, and nobody defined it. We
worked it out entirely from the words *around* it: ground, beans, cups, drank, awake. It is
coffee, or something very like it.

That is the **distributional hypothesis**, summed up by the linguist J.R. Firth in 1957:

> *"You shall know a word by the company it keeps."*

**Distributional hypothesis = words that appear in similar contexts have similar meanings.**

It is not a perfect rule. It famously mixes up opposites, because "hot" and "cold" appear in
near-identical contexts ("the coffee is too hot", "the coffee is too cold"). But it is
astonishingly productive.

Crucially, it is **self-supervised**. In simple words, raw text already contains the answers we
train on, so nobody has to label anything. No annotation, no humans, no cost beyond compute.
That property is why this idea scaled and dictionary-based approaches did not.

*Tesgar* is our example for the whole chapter.

---

## What is word2vec

**Word2vec = a tiny neural network trained to predict a word's neighbours, kept only for the word
vectors it learns along the way.**

Chapter 8 left us with one-hot vectors, where "coffee" and "tea" share nothing at all. Word2vec
gives each word a short dense vector instead, typically 300 numbers, and it learns those numbers
from text.

To see how, we must first know what a neural network's weights are.

---

## What are weights

Chapter 2 described a neural network as a model with adjustable numbers learned from data. Those
adjustable numbers have a name.

**Weights = the adjustable numbers inside a neural network.**

In simple words, a neural network is a big table of adjustable numbers. Training nudges those
numbers, a tiny bit at a time, until the network's guesses improve. In word2vec, the numbers
themselves turned out to be the useful product.

Word2vec's main table of weights has one row per word in the vocabulary. Each row holds 300
numbers:

```
              dim 1   dim 2   dim 3  ...  dim 300
"coffee"   [  0.21,  -0.40,   0.05, ...,   0.33 ]
"tea"      [  0.19,  -0.37,   0.11, ...,   0.29 ]
"tungsten" [ -0.52,   0.08,  -0.61, ...,   0.02 ]      # toy numbers
```

Before training, every row is random. After training, the row for "coffee" *is* the vector for
"coffee".

---

## How word2vec learns

Word2vec turns the distributional hypothesis into a prediction game. Its **skip-gram** variant
slides a window across the text. The word in the middle of the window is the **centre word**,
and the network tries to predict the words around it.

```
"she ground fresh coffee beans every morning"
                    ▲
              centre word

window ±2  →  training pairs:
  (coffee, ground) (coffee, fresh) (coffee, beans) (coffee, every)
```

**Phase 1: Making the training pairs.**

**Step 1:** Slide a window of ±2 words across billions of words of text.

**Step 2:** At each position, pair the centre word with every word inside the window.

**Phase 2: Training.**

**Step 3:** Look up the centre word's row in the table. That row is its current vector.

**Step 4:** Use that vector to guess which word is the neighbour.

**Step 5:** Nudge the weights so the true neighbour becomes a little more likely.

**Step 6:** Repeat for every pair, billions of times.

**Phase 3: Keeping the table.**

**Step 7:** Throw away the predictions. Keep the table.

The row the network looks up in Step 3 is called the **hidden layer**: the middle step between the
word going in and the guess coming out. So the move is this: **throw away the predictions and
keep the hidden layer.** The table of weights that maps each word to its hidden layer *is* the
embedding table.

Now, the question is, why does this give similar words similar vectors?

Because to predict "beans", "cups" and "drank" from *tesgar*, the network must give *tesgar* a
vector resembling the vector for *coffee*, which predicts the same neighbours. Words that keep
the same company end up with the same internal code. The fake task forced a useful representation
into existence.

This pattern, inventing a task nobody wants in order to get a representation everybody wants, is
called a **pretext task**. It underlies essentially all of modern self-supervised learning.

---

## Why a softmax is too slow

There was a computational catch in Step 4. Properly predicting a word means a **softmax** over
the whole vocabulary.

**Softmax = a function that turns a list of raw scores into probabilities that add up to 1.**

It raises the number e (Chapter 8) to the power of each score, then divides each result by their
total. Let's take a toy vocabulary of three words, and the network's raw scores for *"which word
sits next to coffee?"*:

| Word | Raw score | After softmax |
|---|---|---|
| beans | 2 | 0.665 |
| cups | 1 | 0.245 |
| tungsten | 0 | 0.090 |

In simple words, the biggest score gets the biggest share, every share lies between 0 and 1, and
the shares add up to 1 (0.665 + 0.245 + 0.090 = 1).

Now the problem. The total we divide by includes *every* word in the vocabulary. With a million
words, a softmax turns a million raw scores into probabilities, which means touching every word
in the vocabulary for every single training pair. There are billions of pairs. Hopeless.

---

## What is negative sampling

**Negative sampling = replace "pick the right word out of a million" with a few cheap yes/no
questions: is this pair real, or did I make it up?**

The fix is so simple it feels like cheating. For each true pair, like `(coffee, beans)`, we pick
a handful of random words and pair each one with the centre word: `(coffee, democracy)`,
`(coffee, tungsten)`, `(coffee, jellyfish)`, and so on. We label those pairs fake. The fake pairs
are called **negatives**.

With five negatives, one true pair becomes 6 yes/no questions instead of a softmax over 1,000,000
words.

The objective becomes: **make real pairs score high, make random pairs score low.**

Each yes/no answer needs a probability. For that we need one more function.

**Sigmoid = a function that squashes any score into a number between 0 and 1.** It is written σ,
the Greek letter sigma.

| Score | σ(score) |
|---|---|
| 4 | 0.98 |
| 2 | 0.88 |
| 0 | 0.50 |
| −2 | 0.12 |
| −4 | 0.02 |

Put simply, a big positive score means "almost certainly real", zero means "no idea", and a big
negative score means "almost certainly fake". The score itself is the dot product of the two
words' vectors (Chapter 4).

**Note:** Softmax makes all the words compete for one shared total of 1. Sigmoid judges each pair
on its own. That difference is the whole speed-up.

Now the full objective, for one true pair:

$$ \log \sigma(v_c \cdot v_w) + \sum_{k=1}^{K} \mathbb{E}_{w_k \sim P_n} \left[ \log \sigma(-v_c \cdot v_{w_k}) \right] $$

Let's read it piece by piece:

- $v_c$ is the centre word's vector, and $v_w$ is the true neighbour's vector.
- $\sigma(v_c \cdot v_w)$ is the probability that the true pair is real. We want it near 1.
- $w_k$ is one of $K$ random negative words, drawn from the word list $P_n$ described below.
- $\sigma(-v_c \cdot v_{w_k})$ is the probability that the fake pair is fake. We want that near 1
  too.
- σ is the sigmoid, which squashes any score into 0–1. $\mathbb{E}$ means "on average over the
  random negatives".
- log is the natural log from Chapter 8. The log of a probability near 1 is near 0, and the log
  of a small probability is a large negative number. Training pushes the whole sum up toward 0.

In simple words: **pull the true pair together, push the sampled negatives apart.**

**Remember that sentence.** It is the core idea behind how almost every modern embedding model is
trained, including the one you will use this week. Chapter 12 shows the modern form. The pairs become
(query, relevant document) instead of (word, neighbour), and the negatives become deliberately
hard rather than random. But the shape is unchanged. Word2vec helped make contrastive learning
(training by comparing right pairs with wrong ones) mainstream in NLP, natural language
processing.

One detail is worth keeping. The negatives are not drawn in plain proportion to how common each
word is. They come from a *modified* word-frequency distribution:

$$ P(w) \propto f(w)^{0.75} $$

Here $f(w)$ is how often word $w$ appears in the text, and ∝ means "in proportion to". Raising to
the power 0.75 flattens the gaps. If "the" is 1,000 times as common as "tungsten", it is chosen
as a negative only about 178 times as often. So common words are not always the ones picked.
Small hack, significant quality gain, and a foreshadowing of how much the choice of negatives
matters later.

---

## An example: which word is most like *tesgar*?

Let's ask one question of two methods: *which word in the vocabulary is most similar to
tesgar?*

First, the one-hot vectors of Chapter 8. The dot product of *tesgar* with every other word is 0.
"coffee", "tungsten" and "democracy" all tie at 0. The method cannot answer. Any word it returns
is a guess.

Now, word2vec. Suppose our training text contains the three *tesgar* sentences, plus many
ordinary sentences about coffee.

**Before training**, the row for *tesgar* is random. It is as close to "tungsten" as to "coffee".

**During training**, the true pair `(tesgar, beans)` pulls the vector for *tesgar* a little
toward "beans". So do `(tesgar, cups)` and `(tesgar, drank)`. Meanwhile, fake pairs like
`(tesgar, tungsten)` push it away from random words. "coffee" gets exactly the same pulls, because
it keeps the same company.

**After training**, *tesgar* and "coffee" sit close together in the Map Room, and "tungsten" is
far away. The most similar word to *tesgar* is "coffee". The answer is correct, and nobody wrote a
dictionary entry.

The vectors learn more than neighbourhoods. The famous demonstration from the word2vec authors
used arithmetic:

```
vector("king") − vector("man") + vector("woman")  ≈  vector("queen")
```

In simple words, the step from "man" to "king" points in roughly the same direction as the step
from "woman" to "queen". The space has consistent directions for some relationships.

**Note:** That search for "queen" leaves out the three input words. Without that rule, the
nearest vector is often "king" itself. The What people get wrong section below has more on how
far to trust this.

---

## What word2vec could not do

Word2vec produces **one vector per word, forever**. That is a hard ceiling.

**Problem 1: One word, one meaning.** *Bank* gets a single vector, an incoherent average of river
banks and savings banks. *Apple* blends the fruit and the company. *Lead* the metal and *lead* the
verb collapse into one point. A word with several meanings is called **polysemous**, and polysemy
is unrepresentable here.

**Problem 2: Context cannot change anything.** In "I deposited money at the bank" and "we sat on
the river bank", the vector for *bank* is byte-identical. All the information needed to tell them
apart is right there in the sentence, and the model structurally cannot use it.

**Problem 3: Unknown words break it.** A word absent from the training text has no row in the
table. FastText patched this by building vectors from character n-grams (short runs of letters,
such as "tes", "esg" and "sga"), which also helped with word endings and typos.

**Problem 4: Sentence meaning is crude.** Averaging a sentence's word vectors gives a usable
sentence vector, which is remarkable. But "the dog bit the man" and "the man bit the dog" average
to exactly the same thing. Acme's negation question suffers too. *"Which plans do not include
single sign-on?"* differs from *"Which plans include single sign-on?"* by one small word, so the
two averages sit almost on top of each other.

Every one of these limits has the same root cause: *the vector is computed before the context is
known*. Fixing that is precisely what transformers did, and it is Chapter 10.

---

## When word2vec still makes sense

We should not use word2vec for text search in 2026. A sentence embedding model does that job
better in every way that matters.

We may still use word2vec's recipe when we have sequences of things that are not text, such as
products viewed together or songs that share playlists. It is cheap, it needs no labels, and it
works where no text encoder exists. The Ninja notes below list the common versions.

---

### Under the hood

The skip-gram model with negative sampling, stripped to its skeleton:

```python
import torch, torch.nn as nn

class SkipGramNS(nn.Module):
    def __init__(self, vocab, dim=300):
        super().__init__()
        self.centre  = nn.Embedding(vocab, dim)   # the table you keep
        self.context = nn.Embedding(vocab, dim)   # the table you discard

    def forward(self, c, pos, neg):               # neg: (batch, K)
        vc = self.centre(c)                                     # (B, D)
        pos_score = (vc * self.context(pos)).sum(-1)            # (B,)   true pair
        neg_score = torch.bmm(self.context(neg), vc.unsqueeze(-1)).squeeze(-1)  # (B, K) fakes
        return -(torch.logsigmoid(pos_score).mean()             # minus: training code
                 + torch.logsigmoid(-neg_score).sum(-1).mean()) # makes numbers smaller
```

Reading it line by line: `self.centre` and `self.context` are the two tables of weights. `pos_score` is
the dot product for each true pair, `neg_score` holds the dot products for the K fake pairs, and
`logsigmoid` is log σ from the formula. `.sum(-1)` adds up the K negatives, like the Σ does, and
`.mean()` averages over the batch. The minus sign at the end turns "make this big" into "make
this small", which is the form training code expects. Chapter 12 calls that number the loss.

Note the two separate embedding tables. A word has one vector when it is the centre and a
different one when it is the context, and only the centre table survives. This asymmetry is an
early echo of the query/document asymmetry in Chapter 14. The same text plays different roles and
deserves different representations.

---

### What people get wrong

**"Word2vec is deep learning."** It is two lookup tables (two embedding matrices) with no
nonlinearity between them. Its power came from scale and the objective, not depth. That is a
lesson the field keeps relearning.

**"The analogy arithmetic proves it understands concepts."** It shows consistent linear structure
in the space, which is real and useful. It is not comprehension. And the headline examples quietly
exclude the input words from the answer.

**"I should use word2vec for my search engine."** Not in 2026. Use a sentence embedding model.
Word2vec's value now is conceptual, and occasionally practical for embedding non-linguistic
sequences, below.

---

### Ninja notes

The skip-gram trick generalises far past words, and this is genuinely useful. Any sequence of
discrete items can be fed to it, treating co-occurrence as context:

- **item2vec**: products co-viewed in a session become "words in a sentence", yielding product
  embeddings for recommendation.
- **node2vec**: random walks on a graph produce node sequences, and the nodes get embedded.
- **Session and playlist embeddings**: songs that co-occur in playlists.
- **Log and trace embeddings**: events that co-occur in an incident.

These remain live techniques because they are cheap, need no labels, and work on data that has no
text at all. If you have co-occurrence data and no obvious encoder, skip-gram with negative
sampling is still a strong, fast first move.

---

### Key takeaways

- **Word2vec = a tiny network trained to predict a word's neighbours, kept for its weights.** The
  table of weights is the embedding table.
- The distributional hypothesis: similar contexts imply similar meanings.
- **Weights** are a network's adjustable numbers. **Softmax** turns scores into probabilities that
  add up to 1. **Sigmoid** squashes one score into 0–1.
- Negative sampling (pull true pairs together, push random pairs apart) is the direct ancestor of
  modern contrastive training.
- One vector per word means polysemy and context are structurally impossible.
- The technique still works for any co-occurring discrete items: products, nodes, events.

### What's next

[Chapter 10](./10-transformers-contextual.md) removes the ceiling: representations computed
*after* seeing the context, so that "bank" can finally mean two things.

A fake prediction game produced real word vectors, negative sampling made it fast, and one fixed
vector per word was always going to be the ceiling.
