---
title: "Audio, Video, Code, Graphs, and Users"
chapter: 48
part: "Part VI — Vectors Beyond Text"
slug: other-modalities
readingTime: "13 min"
summary: "A tour of everything else that has been mapped into vector space — and the single recipe that keeps working across all of them."
tags: [audio, video, code, graphs, recommenders, embeddings, tour]
prev: 47-colpali-3-production
next: 49-rag-first-principles
---

# Audio, Video, Code, Graphs, and Users

**The one-paragraph version.** Every modality in this chapter goes into the Map Room with the
same recipe. We decide what "similar" means, find pairs of things that show that kind of
similarity, train an encoder with a contrastive loss, and index the vectors with anything from
Part IV. People have applied this recipe to audio, video, code, molecules, graphs, users and
products. The one thing that really changes from modality to modality is *what counts as a
pair*. That single choice decides what the system can find, and what it can never find.

In this chapter, we will take a short tour of the other things people put in the Map Room:
sound, video, source code, graphs, and users. For each one we will see what "similar" means,
what the training pairs are, and where it shows up at Acme. We will also see the one recipe
underneath all of them, and the one place where the usual advice about normalizing vectors
flips.

We will cover the following:

- The one recipe behind every modality
- Audio
- Video
- Code
- Graphs
- Users, items and recommendation
- All the modalities side by side

---

## The one recipe behind every modality

So far, Part VI has been about pictures and scanned pages. What about everything else a company
owns?

Acme's support team has more than the 40,000 pages of the Great Library. It has recorded support
calls, screen-recording tutorials, code samples for its software development kit (SDK), and a
record of which customers use which features. Each can get a Map Room of its own, and we already
know how to build one.

**Embedding a new modality = choose what "similar" means + find pairs that show it + train with a
contrastive loss + index the vectors.**

**Step 1:** Decide what "similar" means for our task.

**Step 2:** Find data where similar things come naturally in pairs. A docstring (the short
description at the top of a function) and the function itself. A caption and its image. A
question and its answer. A user and the item they clicked.

**Step 3:** Train two encoders (or one shared encoder) with a contrastive loss, so each pair
lands close together and unrelated things land far apart (Chapter 12).

**Step 4:** Normalize the vectors, unless their length carries meaning (Chapter 5).

**Step 5:** Index them with anything from Part IV. The Card Catalog does not care what the dots
mean.

In simple words, the machinery is shared, and only the pairs are new.

Step 1 is the only step that needs real judgement, and it is where projects succeed or fail.
Steps 3 to 5 are commodity work. Let's visit each modality and watch Step 1 being decided.

---

## Audio

In audio, "similar" can mean more different things than anywhere else. Choosing the wrong
meaning is the main way audio search fails.

| Task | "Similar" means | Typical model |
|---|---|---|
| Speaker identification | Same person speaking | Speaker embedding (x-vector, ECAPA) |
| Music recommendation | Same genre, mood, instrumentation | Audio encoders on music corpora |
| Sound search by text | Matches this description | CLAP (contrastive language-audio) |
| Spoken content search | Same words spoken | Usually transcribe first, then text search |

The names in the right-hand column are model families. We do not need to know their insides.

**CLAP = Contrastive Language-Audio Pretraining.** It is CLIP (Chapter 42) for sound. An audio
encoder and a text encoder are trained on (sound, caption) pairs, so they share one Map Room. The
text *"a dog barking in the rain"* lands near a recording of a dog barking in the rain. It is the
same two-tower design (Chapter 13) and the same InfoNCE loss (Chapter 12), with a different
modality.

Now let's see why picking the meaning matters. Acme records its support calls. A support lead
wants every call where a customer asked whether the Pro plan includes single sign-on.

Let's first try the model the team already has. It is a speaker embedding, left over from a
call-quality project. They take one call about SSO, embed it, and ask for the 50 nearest calls.
Back come 50 calls handled by the same support agent, about refunds, invoices and password
resets.

The answer is wrong. A speaker embedding places two recordings of the same voice close together,
whatever was said. We asked "what is being said?" and the model answered "who is speaking?".

Now let's use the approach that works for spoken words. It starts with one new term.

**ASR = automatic speech recognition.** It is software that listens to speech and writes down
the words.

**Step 1:** Run ASR over every call to get a transcript.

**Step 2:** Chunk and embed the transcripts like any other text.

**Step 3:** Search them with the full text stack: hybrid search, reranking, everything from
Parts IV and V.

Now the query *"Does the Pro plan include single sign-on?"* finds the calls where those words,
or a paraphrase of them, were spoken. It no longer matters which agent took the call. The answer
is correct.

**For searching spoken words, transcribing first usually wins.** One transcription pass buys
every text technique in this book, including exact-term search. Embed the audio directly only
when the *sound itself* is the point: a cough, an engine fault, a musical texture.

**Note:** Chapter 2's rule bites hardest here. *A model encodes the similarity it was trained
on.* A speaker model and a content model give opposite answers about the same recording. Name
the task before picking the model.

---

## Video

Video is expensive because it is large, and most of it repeats itself.

Acme publishes screen-recording tutorials. One of them is a 20-minute tour of the admin
settings. At 12:40 it opens the single sign-on page. At 30 frames per second, that one video has
36,000 frames.

Start with the simple approach. We embed the frames and average them into one vector for the whole
video. A user searches *"where do I turn on single sign-on?"*. The admin tour is only a weak
match, because 19 of its 20 minutes are about other settings. If it ranks at all, the user gets a
20-minute video and no idea where to look.

The standard approach avoids embedding all of it. It needs one small term first. A **shot** is a
stretch of video with no cut in it. In a screen recording, a new shot usually means a new page
or dialog.

**Step 1:** **Shot detection** splits the video wherever the picture changes sharply.

**Step 2:** **Keyframe extraction** takes one or a few representative frames from each shot.

**Step 3:** **Frame embedding** turns each keyframe into a vector with CLIP or SigLIP
(Chapters 42 and 43).

**Step 4:** **Aggregation.** Either pool the frames into one vector per clip, or, better, keep
them all and treat the video as a multi-vector document (Part V).

That last choice should look familiar. A video is a sequence of frames, the way a document is a
sequence of tokens, and pooling loses the exact moment we wanted. MaxSim (Chapter 35) scores the
video by its best-matching frame instead, so the frame of the SSO settings page can carry the
whole video to the top.

Then we add the audio track. We run ASR on it, index the transcript as a parallel text index, and
fuse the two result lists with RRF (Chapter 40). The narrator says "now let's set up single
sign-on" at 12:38. Both indexes now point at the same moment, and we can send the user straight
to 12:40.

In simple words, users rarely want "this video is relevant". They want "here is the timestamp",
and keyframes plus a transcript give them that.

---

## Code

Acme also publishes code samples for its SDK. General text embedding models are noticeably weak
on code, and the reason is worth understanding.

**Code similarity is about what the code does, not the words it uses.** Two quicksort functions
in different languages, with different variable names, are "the same". Two sort functions that
differ only by `<` versus `>` sort in opposite orders.

Dedicated code embedding models are trained on pairs that capture behaviour:

- A docstring and its function body (the biggest source, because it is naturally paired and
  abundant)
- A commit message and its diff (the lines of code that changed)
- An issue and the commit that fixed it
- Duplicate questions on developer forums

A developer asks, *"How do I refresh an SSO session token with the Python SDK?"* Suppose we
chunk the SDK by character count. The chunk holding `def refresh_token(` ends halfway through
the function, and the chunk with its retry loop has no function name at all. Neither piece
answers the question.

**Practical notes for code.**

- Chunk by syntax: functions, classes, modules. A function split across two chunks is useless.
- Put the file path and the enclosing name in the text we embed. `auth/session.py :: refresh_token`
  carries real signal.
- Run hybrid search. Function names, identifiers and error strings like `SSO-4012` are exactly
  the exact-match case from Chapter 7. BM25 handles them well, provided the tokenizer keeps
  identifiers such as `refresh_token` intact.

With those three changes, the whole `refresh_token` function is one chunk that starts with its
path. The query lands on it.

---

## Graphs

A **graph** is a set of things, called nodes, joined by links, called edges. Acme's customer
graph has nodes for customer companies, users and features. Its edges say "this user belongs to
this company" and "this company uses this feature".

**Graph embeddings place nodes so that nodes related in the graph sit close in the Map Room.**
There are two families.

**Family 1: shallow random-walk methods (node2vec, DeepWalk).**

**node2vec = random walks over the graph + word2vec's skip-gram.**

**Step 1:** Start at a node and hop along edges at random for a while.

**Step 2:** Write the walk down as a "sentence", where each node is a "word".

**Step 3:** Train skip-gram (Chapter 9) on many such sentences.

Nodes that keep turning up in the same walks end up close together, just as words that share
contexts do. It is cheap, needs no information about the nodes themselves, and it remains a
strong baseline.

node2vec has two knobs, $p$ and $q$. Means, $p$ controls how often the walk steps straight back
to where it came from, and $q$ controls whether it stays near its start or wanders outward.
Walks that stay close to home, like breadth-first search, lean toward a node's local
*structural role*, such as "an admin at the centre of many users". Walks that wander far, like
depth-first search, tend to capture *community*. All the users inside one customer company end up
close. One caution: two admins at *different* companies rarely share a walk, and then only several
hops apart, so random walks do not place them close. Methods built for roles, such as struc2vec,
handle that case. These are two different meanings of "similar node", and we must choose between
them on purpose.

**Family 2: graph neural networks, such as GraphSAGE and GAT (graph attention network).** A
**graph neural network (GNN)** builds each node's vector by repeatedly mixing in information from
its neighbours. It can use node attributes, such as a company's plan and seat count. It can also
embed a node it never saw during training, which random-walk methods cannot.

Graph embeddings are used for fraud rings, recommendation, knowledge-graph completion, and
spotting unusual access patterns. In RAG, the edges themselves are useful too. We find a page by
its content, then follow its links to the pages it depends on. Page 212 links to page 1,140, and
Chapter 51 shows how to follow that link.

---

## Users, items and recommendation

This is one of the oldest and largest commercial uses of vector search. At Acme, the help center
shows each logged-in admin a short list called "Articles you may need".

**Two-tower models** (Chapter 13) encode a user and an item into one shared Map Room. Training
pulls each user's vector close to the items that user engaged with. Serving a recommendation is
then an ANN query. We embed the user and fetch the nearest items. Every technique in Part IV
applies directly, at enormous scale.

An admin who has just read page 212 and the SSO setup guide gets a user vector near the SSO part
of the Map Room. The nearest item they have not read is page 1,140, about the Security add-on.
That is exactly the page they need next.

**Difference 1: length often carries meaning here** (Chapter 5). Popular items may be allowed to
grow larger norms, so that the raw dot product favours them. "How to reset your password" is
read by almost every admin, and its long vector keeps it near the top. If we normalize by
reflex, that popularity signal is deleted. This is the classic case of when *not* to normalize.
It is worth remembering precisely because it contradicts the default advice everywhere else in
this book.

**Difference 2: diversity and freshness are first-class goals.** Ten near-identical SSO articles
is a failure even if all ten are relevant. That is where **MMR** (maximal marginal relevance, a
re-ordering that skips results too similar to ones already picked, Chapter 41) and deliberate
exploration come in.

---

## All the modalities side by side

| Modality | "Similar" means | Training pairs | Usual approach | At Acme |
|---|---|---|---|---|
| Spoken words | Same words said | (not needed) | ASR, then text search | Support calls |
| Sounds | Matches a description | Sound ↔ caption | CLAP | (rare) |
| Voices | Same speaker | Clips of one speaker | Speaker embedding | Call-quality checks |
| Video | Same moment | Frame ↔ caption (via CLIP/SigLIP) | Shots, keyframes, multi-vector, plus transcript | Screen-recording tutorials |
| Code | Same behaviour | Docstring ↔ function, commit ↔ diff | Code model, syntax chunks, hybrid | SDK samples |
| Graphs | Same role, or same community | Nodes in the same walk | node2vec, or a GNN | Customer graph |
| Users and items | User would engage with item | User ↔ click | Two-tower | Article recommendations |

**When to use which one.** We must transcribe first when the *words* in audio or video matter.
We must embed the raw signal when the *sound or picture itself* matters. We must use a code model
with syntax-aware chunks and hybrid search for code. For graphs, we start with node2vec, and move
to a GNN when nodes have useful attributes or new nodes arrive all the time. Many strong systems
index several of these side by side and fuse the results.

---

### Under the hood

The recipe, on one card:

```
1. Decide what "similar" means for YOUR task.
2. Find naturally-paired data that demonstrates it.
   (docstring↔code, caption↔image, question↔answer, user↔click, walk↔node)
3. Train two encoders (or one shared) with a contrastive loss.   [Ch. 12]
4. Normalize — unless magnitude carries meaning.                 [Ch. 5]
5. Index with anything from Part IV. It does not care what the vectors mean.
```

And a two-tower recommender, which is the shape underneath most of this chapter:

```python
import torch, torch.nn as nn, torch.nn.functional as F

class TwoTower(nn.Module):
    def __init__(self, n_users, n_items, dim=128):
        super().__init__()
        self.user = nn.Embedding(n_users, dim)
        self.item = nn.Embedding(n_items, dim)

    def loss(self, user_ids, pos_item_ids, tau=0.05):
        u = F.normalize(self.user(user_ids), dim=-1)
        i = F.normalize(self.item(pos_item_ids), dim=-1)
        logits = u @ i.T / tau                                   # in-batch negatives
        return F.cross_entropy(logits, torch.arange(len(u), device=u.device))
```

In simple words, each user and each article is a row in a table of learned vectors. For a batch
of (user, article they read) pairs, the loss pulls each user toward their own article and pushes
them away from everyone else's article in the batch.

That is Chapter 12's loss, unchanged, with lookup tables instead of transformers. The same
*idea*, pull each pair together and push the crowd apart, has now appeared in this book for
word2vec, sentence embeddings, CLIP, ColBERT, ColPali and recommenders. **The exact loss varies,
but its most common form is these four lines.**

**Note:** This version normalizes inside the loss, which is the common default. A recommender
that should keep popularity in vector length would drop the two `F.normalize` calls, score with
the raw dot product, and set `tau` to 1 (or learn it). Raw dot products divided by 0.05 would be
huge, and the softmax would saturate.

---

### What people get wrong

**Using a text model for code.** Measurably worse. Use a code model, and run hybrid.

**Embedding every video frame.** Redundant and expensive. Detect shots, take keyframes.

**Choosing the wrong audio model.** A speaker embedding cannot search spoken content, and a
content model cannot identify speakers. Name the task before picking the model.

**Normalizing recommender embeddings by reflex.** You may be deleting your popularity signal.

**Ignoring the cold-start problem** (a new user or item with no history yet). Random-walk graph embeddings and ID-based recommender
towers have no vector for a node or item they never saw. Content-based towers and GNNs do. If
new items arrive constantly, that one fact decides your architecture.

---

### Ninja notes

**"Any-to-any" models are worth watching.** ImageBind and its successors train several
modalities against a single anchor, usually images. The other modalities then become aligned to
each other *without ever being trained on those pairs*. Audio and text end up comparable because
both were separately aligned to images.

If that generalises robustly, it changes the data problem a lot. You would need pairs only
between each new modality and one hub, rather than between every pair of modalities. For an
organisation with text, images, audio, telemetry and structured records, that is the difference
between ten datasets and four.

**The more immediate ninja move is simpler.** Most teams treat their modalities as separate
systems with separate search boxes. Put them in one shared space, or run separate indexes and fuse
with RRF (Chapter 40). Then one query returns the relevant help page, call recording, code change
and ticket. The infrastructure is identical. Only the encoders differ. That is usually a bigger
product win than any model upgrade.

---

### Key takeaways

- **Embedding a new modality = choose what "similar" means + find pairs that show it + train with
  a contrastive loss + index the vectors.** Only the definition of a pair changes.
- Audio: pick the model by task (speaker, content, or text-described sound). For spoken words,
  run **ASR** (automatic speech recognition) and use text retrieval. **CLAP** is CLIP for sound.
- Video: detect shots, embed keyframes, and treat the video as multi-vector so we can return a
  timestamp. Add the transcript and fuse.
- Code: similarity is about behaviour, not words. Chunk by syntax, prefix the path, always run
  hybrid.
- Graphs: **node2vec** (random walks plus skip-gram) is still a strong baseline. GNNs handle node
  attributes and unseen nodes.
- Recommenders: two towers, and **do not normalize** if vector length encodes popularity.
- One shared space, or RRF across modality indexes, turns separate search boxes into one.

### What's next

Part VI is complete. [Part VII](./49-rag-first-principles.md) puts everything so far to work:
retrieval-augmented generation, where the Librarian fetches and the Scholar writes the answer.

We now know that sound, video, code, graphs and users all reach the Map Room by the same recipe,
and that the real decision each time is what counts as a pair.
