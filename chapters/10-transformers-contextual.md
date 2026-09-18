---
title: "Transformers and Contextual Embeddings"
chapter: 10
part: "Part II — Where Embeddings Come From"
slug: transformers-contextual
readingTime: "11 min"
summary: "Attention lets every token's vector be rewritten by its neighbours. That single change is why 'bank' finally has two meanings — and why nearly every embedding model since 2018 is a transformer."
tags: [transformers, attention, bert, contextual-embeddings, tokenization]
prev: 09-word2vec
next: 11-pooling
---

# Transformers and Contextual Embeddings

**The one-paragraph version.** A transformer starts with a fixed vector for each token, then
repeatedly lets every token look at every other token and rewrite its own vector based on what it
sees. After a dozen such rounds, a token's vector no longer describes the word in isolation. It
describes *this word, in this sentence, in this role*. Retrieval quality jumped when embeddings
became contextual, because queries and documents are sentences, not bags of words.

In this chapter, we will learn how a transformer gives the same word different vectors in
different sentences. We will see what a token is, how attention works, and how BERT-style and
GPT-style models differ. We will also see the four problems that come with transformers, each of
which shapes a later chapter.

We will cover the following:

- What is a token
- What is a transformer
- How attention works
- An example: two meanings of "bank"
- Encoders and decoder-only models
- Problems with transformers
- Word2vec vs transformers
- When this matters

Our example for the whole chapter is the word **bank**. In *"I deposited money at the bank"* it
means a place that holds money. In *"we sat on the river bank"* it means the edge of a river.
Chapter 9 showed that word2vec gives both the same vector. This chapter fixes that.

---

## What is a token

Chapter 7 used the word *token* loosely. Now we need it properly, because a transformer never
sees words. It sees tokens.

**Token = a piece of text that the model reads as one unit: a whole word, a part of a word, or a
punctuation mark.**

Before anything else happens, a **tokenizer** splits the text into tokens. Modern tokenizers work
on subwords (the common families are called BPE, WordPiece and SentencePiece). Common words stay
whole. Rare words break into fragments.

```
"bank"              → ["bank"]
"unbelievable"      → ["un", "believ", "able"]
"ORA-01555"         → ["OR", "A", "-", "015", "55"]
"Kubernetes"        → ["Kub", "ernetes"]
# illustrative: the exact pieces depend on the tokenizer
```

In simple words, the tokenizer chops text into the pieces the model has a vector for. Each token
starts with a fixed vector from a lookup table, exactly like word2vec's rows in Chapter 9.

Two consequences follow, and we will actually feel both.

**No out-of-vocabulary problem.** Any string can be represented by falling back to smaller pieces.
So new words and typos degrade gracefully instead of failing. This fixes Problem 3 from Chapter 9.

**Identifiers are shredded.** Look at `ORA-01555` above. It becomes five fragments with no
coherent meaning. We come back to this in the problems below.

---

## What is a transformer

**Transformer = a neural network that repeatedly lets every token look at every other token, then
rewrite its own vector using what it found.**

Think of it like a meeting.

Picture six people around a table, one for each token of *"we sat on the river bank"*. Each person
starts with a note card holding their own initial opinion.

**Round one.** Everyone reads everyone else's card. Then each person rewrites their own card, adding
what they found relevant. The person holding *bank* sees *river* and *sat* on nearby cards, and
edits toward the riverside meaning. In a longer sentence, a person holding *it* would look around
for what *it* refers to, and edit toward that.

**Round two.** Same again, on the updated cards. Now second-hand effects spread. Information reaches
a card through intermediaries, even from cards that were not directly relevant.

Repeat twelve times. Every card is now a rich, context-dependent description. The card that started
as "the generic word *bank*" now reads "the river-edge sense of *bank*, the place where the sitting
happened".

Now the mapping, one part at a time:

- Each person is one token.
- Each note card is that token's vector. The starting card is the fixed, word2vec-style vector.
- Reading everyone else's card is **self-attention**.
- Each person rewriting their own card is a **feed-forward network**: a small neural network that updates each
  token's vector on its own.
- One round is one **layer**. Twelve rounds is twelve layers.

That meeting is a transformer.

---

## How attention works

**Attention = the step where each token decides how much to borrow from every other token, and
mixes in their information in those amounts.**

Each token makes three vectors. Each one is produced by multiplying the token's vector by its own
learned table of weights (a **linear projection**, in the jargon):

- **Query (Q)**: "what am I looking for?"
- **Key (K)**: "what do I offer?"
- **Value (V)**: "what will I contribute if you pick me?"

A token's new vector is a weighted average of everyone's Values. The weights come from how well
its Query matches each Key:

$$ \text{Attention}(Q,K,V) = \text{softmax}\!\left(\frac{QK^\top}{\sqrt{d_k}}\right)V $$

Let's read it left to right.

- $QK^\top$ is every token's Query dotted with every token's Key: a full grid of relevance scores.
- $d_k$ is the number of dimensions in each Key. In BERT-base, $d_k = 64$, so we divide by
  $\sqrt{64} = 8$. This keeps the scores from growing with dimension and saturating the softmax.
- The softmax (Chapter 9) turns each row of scores into weights that add up to 1.
- Multiplying by $V$ mixes everyone's contribution in those proportions.

Let's do it for *bank* in *"we sat on the river bank"*. To keep the numbers small, we let *bank*
look at only three tokens. Suppose its Query, dotted with their Keys and divided by $\sqrt{d_k}$,
gives these raw scores:

| Token | Raw score | After softmax |
|---|---|---|
| river | 2 | 0.665 |
| sat | 1 | 0.245 |
| the | 0 | 0.090 |

In simple words, *bank*'s new vector is about two-thirds what *river* offers, a quarter what *sat*
offers, and a sliver of *the*. In *"I deposited money at the bank"*, *bank*'s Query would match
*money* and *deposited* instead, and *bank* would lean the other way.

Notice the dot product at the centre. **Attention is similarity search, run inside the model, at
every layer.** The operation we use to find documents is the operation the model uses to decide
which words matter to which. We never need to act on that coincidence, but it is a satisfying one.
It also explains why the hardware that makes transformers fast also makes vector search fast.

"Multi-head" attention simply runs several of these side by side, each with its own Query, Key and
Value tables. One head can track grammar while another tracks what a pronoun refers to and a third
tracks topic.

---

## An example: two meanings of "bank"

Let's take three sentences:

- **a:** *"I deposited money at the bank"*
- **b:** *"we sat on the river bank"*
- **c:** *"the bank approved my loan"*

Sentences a and c use the money meaning. Sentence b uses the river meaning. We ask one question:
*which uses of "bank" mean the same thing?*

Let's first see what word2vec does. It looks up the row for *bank* three times and gets the same
vector three times. Every pair scores a perfect cosine similarity of 1. By its numbers, the river
bank means exactly the same as the bank that approved a loan. The answer is wrong.

Now, let's see what a transformer does. It runs all three sentences through its layers. In
sentence b, attention pulls *bank* toward *river* and *sat*. In a and c, it pulls *bank* toward
*money*, *deposited*, *approved* and *loan*. We read off the three final vectors for *bank* and
compare them. The pair a–c scores higher, and a–b scores lower. The answer is correct.

Put into words, the layers are saying: *"Same spelling. But one bank has a river and people sitting on it.
The other two have money and loans. Those are different things."*

The Under the hood section below runs exactly this test.

---

## Encoders and decoder-only models

Transformers come in two families that matter for embeddings. They differ in which direction each
token may look.

**BERT (2018)** is an **encoder**: every token may look at every other token, before and after it.
Its pretext task (Chapter 9) was **masked language modelling**: hide 15% of the tokens and predict
them from both sides at once.

```
"The [MASK] was rich and dark, and I drank two cups."  →  predict "coffee"
```

Seeing both sides is the key difference from a generative language model such as GPT. Those models
write text one token at a time, so each token may only look *backwards*.

**Decoder-only model = a GPT-style model, where each token may only look at the tokens before
it.**

**Causal attention = attention restricted to the tokens before the current one.** In simple words,
no peeking ahead. The name comes from cause and effect: earlier text can affect later text, never
the reverse.

For *encoding*, we want both sides. The words after a token disambiguate it as much as the words
before. In *"the bank approved my loan"*, the clue *loan* comes after *bank*. A causal model reading
*bank* has not seen it yet.

BERT outputs one vector per token, each one contextual. That is exactly what we wanted. It is why
almost every text embedding model is one of two things. Either it is a BERT-style encoder, or it is
a decoder-only model (the GPT-style kind, with causal attention) adapted to encoding.

---

## Problems with transformers

**Problem 1: The cost grows with the square of the length.** Attention compares every token with
every other token, so it is $O(n^2)$ in the number of tokens $n$. Read that as "order n squared":
the work grows with the square of n. Means, 512 tokens make 262,144 pairs, and 1,024 tokens make
1,048,576. Doubling the length of a chunk quadruples the attention work. This is why context
windows (the most tokens a model can read at once) were historically short. It is also why
long-context models need architectural tricks, and why "just embed the whole document" is
expensive as well as lossy.

**Problem 2: Text past the limit is silently cut off.** Every model has a **context window**: the
maximum number of tokens it can read in one go (Chapter 49 covers it fully). An embedding model with
a 512-token limit silently **truncates** longer input in many libraries. A 2,000-word document
becomes a vector describing roughly its first 400 words, with no warning. If a long Acme page, such
as the Security add-on page, mentions the SSO catch only in its last paragraph, that sentence never
reaches the vector. Check your model's `max_seq_length`, and verify what your framework does
when you exceed it. This one-line check is cheap, and it catches a surprisingly common bug. Token
limits are also why chunking, cutting documents into smaller pieces (Chapter 50), exists.

**Problem 3: Identifiers are shredded.** `ORA-01555` becomes five fragments with no coherent
meaning. This is the mechanical reason dense retrieval fails on error codes, SKUs and part numbers
(Chapter 7). The model literally never sees the identifier as a unit. Acme's error code `SSO-4012`
gets the same treatment, and its neighbour `SSO-4021` breaks into almost the same pieces. No change
to the wording of the question fixes this. A sparse index, like the BM25 of Chapter 8, does.

**Problem 4: We get one vector per token, not one per passage.** A 512-token passage produces 512
vectors. Search usually wants one. That is Chapter 11.

---

## Word2vec vs transformers

| | Word2vec | Transformer |
|---|---|---|
| Vectors per word | One, forever | One per use, shaped by the sentence |
| "bank" in two senses | Identical | Different |
| Unknown words | No row, fails | Falls back to subword pieces |
| Cost to compute | One table lookup | Many layers of attention, $O(n^2)$ |
| Long text | Any length | Cut off at the context window |
| Error codes like `SSO-4012` | Usually no row at all | Shredded into fragments |

---

## When this matters

It matters every time we choose an embedding model, because every modern one is a transformer.
Three habits follow from this chapter.

We must check the context window against our chunk size, and confirm what happens on overflow.

We must tokenize our most important terms (product names, error codes) and look at the pieces.

We must keep a sparse index beside the dense one for anything that is really an identifier.

---

### Under the hood

Contextuality is easy to verify, and doing so makes the concept concrete:

```python
from transformers import AutoTokenizer, AutoModel
import torch

tok = AutoTokenizer.from_pretrained("bert-base-uncased")
mdl = AutoModel.from_pretrained("bert-base-uncased")

def bank_vector(sentence):
    enc = tok(sentence, return_tensors="pt")
    with torch.no_grad():                              # we only read, never train
        out = mdl(**enc).last_hidden_state[0]          # (tokens, 768)
    idx = enc.input_ids[0].tolist().index(tok.convert_tokens_to_ids("bank"))
    return out[idx]

a = bank_vector("i deposited money at the bank")
b = bank_vector("we sat on the river bank")
c = bank_vector("the bank approved my loan")

# (values illustrative: exact numbers vary by model and sentence)
torch.cosine_similarity(a, b, dim=0)   # lower:  different senses
torch.cosine_similarity(a, c, dim=0)   # higher: same sense
```

In simple words, `bank_vector` runs a sentence through BERT, finds the position of the token
`bank`, and returns the vector the last layer wrote there. `last_hidden_state` is the final set of
note cards from our meeting, one per token.

Three different vectors for the same string. Word2vec would have returned one vector three times.
That gap is the whole chapter.

---

### What people get wrong

**"BERT gives me sentence embeddings."** Raw BERT is famously poor at it. Its `[CLS]` vector (a
special token BERT adds at the start of every input, Chapter 11), without fine-tuning (further
training for one specific job), underperforms averaged GloVe vectors (a word2vec-era static
embedding) on similarity tasks. It takes a contrastive fine-tune (further training on pairs,
Chapter 12) to produce a good sentence encoder. Use a model trained for embedding, not a raw
language model.

**"More layers is more semantic."** The final layer is specialised for the pretraining objective.
Intermediate layers often carry more transferable semantics, which is why some models pool (combine token
vectors into one, Chapter 11) from layer $n-1$ or blend layers.

**"Context length equals quality."** A model advertising 8,192 tokens was usually *trained* on much
shorter sequences. Quality often degrades well before the limit. Test at your actual chunk sizes.

**"The tokenizer is an implementation detail."** It determines whether your domain's critical
strings survive as units. Tokenize twenty of your most important terms and look at the output. It
takes five minutes and predicts entire classes of failure.

---

### Ninja notes

Attention's $O(n^2)$ cost has spawned an ecosystem: FlashAttention (exact, but memory-aware and far
faster), sliding-window and local attention, and linear-attention variants. For embedding models
the practical upshot is that **long-context encoders are now viable**. Jina, Nomic, ModernBERT and
others handle 8K tokens.

Long context also enables late chunking, which embeds a whole document once and only then splits
its token vectors into chunks (Chapter 11 explains it properly).

---

### Key takeaways

- **Transformer = a network that lets every token look at every other token and rewrite its own
  vector, layer after layer.** Stacking layers makes representations deeply contextual.
- A **token** is a word, a piece of a word, or a punctuation mark. Subword tokenization removes
  unknown words but shreds identifiers, which is why dense retrieval fails on error codes.
- Attention is a dot-product similarity search, followed by a softmax, performed inside the model.
- Encoders (BERT) see both sides of each token. Decoder-only models (GPT-style) use causal
  attention and see only the tokens before.
- Cost is quadratic in sequence length. That is the reason context windows and chunking matter.
- Exceeding `max_seq_length` often truncates silently. Verify this for your model.
- A transformer gives one vector *per token*. Turning those into one vector is next.

### What's next

[Chapter 11](./11-pooling.md) confronts the decision hiding in every embedding model: how do 512
token vectors become the single vector we store?

We now know what a transformer does to a sentence, why "bank" finally gets two meanings, and which
four problems come with it.
