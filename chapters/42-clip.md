---
title: "Multimodal Embeddings: CLIP and the Shared Space"
chapter: 42
part: "Part VI — Vectors Beyond Text"
slug: clip
readingTime: "13 min"
summary: "Train two encoders — one for images, one for text — to place matching pairs at the same point. Suddenly you can search photographs with sentences."
tags: [clip, multimodal, contrastive, two-tower, zero-shot]
prev: 41-rerankers
next: 43-siglip
---

# Multimodal Embeddings: CLIP and the Shared Space

**The one-paragraph version.** **CLIP** trains an image encoder and a text encoder
*together*, with one goal: a photo and its caption should land at the same point. Train it on
400 million image–caption pairs from the web and we get one Map Room that holds both pictures
and words. Then we can search photos with a sentence, find captions for a photo, and sort
pictures into categories nobody trained it on. All of it uses the same dot product from
Chapter 4.

In this chapter, we will learn what CLIP is, how it turns a picture into a vector, and how two
encoders learn to meet in one shared space. We will also see what that shared space is good
at, where it quietly fails, and when to use it instead of plain tags or a text model.

We will cover the following:

- What is CLIP
- Why we need a shared space
- How CLIP sees a picture: ViT and patches
- How CLIP is trained
- An example of CLIP
- Problems with CLIP
- CLIP vs tags and captions
- When to use which one

---

## What is CLIP

Until now, every vector in this book came from text. A **modality** is a kind of data: text is
one modality, images are another, audio is a third. A **multimodal** model handles more than
one of them.

**CLIP = Contrastive Language–Image Pre-training.**

- **Contrastive:** it learns by pulling matching pairs together and pushing mismatched pairs
  apart, exactly as in Chapter 12.
- **Language–Image:** the pairs are a picture and a piece of text.
- **Pre-training:** it learns general skills once, on a huge pile of data, before anyone uses
  it for a specific job.

CLIP was published by OpenAI in 2021. It has two encoders. The **image encoder** turns a
picture into a vector. The **text encoder** turns a sentence into a vector. Both vectors have
the same length and live in the same Map Room.

In simple words, CLIP puts a photograph of a dog and the words *"a photo of a dog"* at nearly
the same dot.

---

## Why we need a shared space

Let's take a concrete case. Acme's marketing team keeps a library of 10,000 photos from
offices, conferences and team events. The files have names like `IMG_4471.jpg`. A hurried
intern once tagged a few hundred of them by hand. Nobody tagged the rest.

A designer needs a picture for a blog post and types:

> *"a dog sitting under a desk"*

Let's first see what keyword search does. It can only match words, and photos have no words
except their file names and tags. The only tag containing "dog" is *"hot dog stand, Acme
conference 2025"*. So keyword search returns a photo of a food stall.

The answer is wrong. Acme's office dog sleeps under a desk in at least forty photos, and not
one of them is tagged.

Now, the question is, how can a search find a photo that contains no words at all?

We need the sentence and the photo to become comparable. Suppose the photo of the office dog gets
a dot, and the sentence *"a dog sitting under a desk"* gets a dot nearby. Then the search is just
the nearest-neighbour search we already know.

Think of it like a museum where every painting hangs next to its description card:

- The paintings are the photos.
- The description cards are the sentences.
- Hanging each card beside its painting is what training does.
- Walking to a card and looking at the wall beside it is the search.

Once that is true, four jobs become one nearest-neighbour lookup:

- **Text → image:** embed the sentence, find nearby photo vectors.
- **Image → text:** embed the photo, find nearby caption vectors. That is captioning by retrieval.
- **Image → image:** embed a photo, find visually similar photos.
- **Zero-shot classification:** sort photos into categories nobody trained a classifier for.
  We explain this one in the example below.

---

## How CLIP sees a picture: ViT and patches

Before we look at training, we must know how an image becomes a vector at all.

CLIP's image encoder is usually a **ViT**.

**ViT = Vision Transformer.** It is the transformer of Chapter 10, reading an image instead of a
sentence.

A transformer reads tokens. An image has no tokens. So the ViT makes some:

**Step 1:** Resize the photo to a fixed square, for example 224 × 224 pixels.

**Step 2:** Cut the square into a grid of small squares called **patches**. With 32-pixel
patches, that is 224 ÷ 32 = 7 patches across and 7 down, so 49 patches.

**Step 3:** Flatten each patch's pixels into a list of numbers and map it to a vector with one
learned layer. Each patch is now treated like a token.

**Step 4:** Add a position to each patch vector, so the model knows where the patch sits in the
grid.

**Step 5:** Run the transformer. Every patch attends to every other patch, exactly as words
attend to each other in a sentence.

**Step 6:** Keep one summary vector for the whole image, the same idea as the CLS token in
Chapter 11.

In simple words, a ViT cuts a picture into tiles and reads the tiles the way a text model reads
words.

**Note:** The model name `ViT-B-32`, which appears in the code later, decodes as "Vision
Transformer, Base size, 32-pixel patches". Smaller patches mean more of them. A `ViT-L-14` at
224 pixels makes 16 × 16 = 256 patches, and sees finer detail at a higher cost. Patches come
back throughout this Part.

---

## How CLIP is trained

CLIP uses the **two-tower** design we met in Chapter 13: two separate encoders, one per side,
that only meet at the end when their vectors are compared. Here the two towers handle two
different modalities.

```
image ──► [image encoder: ViT] ──────────► projection ──► normalize ──┐
                                                                       ├─► shared space
text  ──► [text encoder: transformer] ───► projection ──► normalize ──┘
```

A **projection** here is one learned layer that resizes each tower's output to the shared
length. For `ViT-B-32` that length is 512.

**Phase 1: Collecting the data.**

**Step 1:** Harvest images from the web together with the text people wrote beside them, such
as captions and alt text (the short description a web page attaches to an image).

**Step 2:** Keep 400 million (image, caption) pairs. Nobody labels them by hand.

**Phase 2: One training step.**

**Step 1:** Take a **batch** of N pairs. CLIP used N = 32,768.

**Step 2:** The image tower turns every image into a vector. The text tower turns every caption
into a vector.

**Step 3:** Project and normalize both, so every vector has length 1 (Chapter 5).

**Step 4:** Compute the N × N grid of dot products: every image against every caption.

**Step 5:** The diagonal holds the true pairs. Everything else is a negative. For each image,
ask the model to pick its own caption out of the row. For each caption, ask it to pick its own
image out of the column.

**Step 6:** Average the two losses and nudge the weights of both towers.

Here is a toy batch of three pairs from Acme's photo library. A ✓ marks the pair the model must
score highest:

```
                     "a dog under a desk"   "a conference stage"   "a team dinner"
photo of office dog           ✓
photo of keynote stage                              ✓
photo of team dinner                                                     ✓
```

This is Chapter 12's InfoNCE, applied in both directions:

```python
import torch, torch.nn.functional as F

def clip_loss(I, T, temperature):
    I, T = F.normalize(I, dim=-1), F.normalize(T, dim=-1)
    logits = I @ T.T / temperature                       # (N, N)
    labels = torch.arange(len(I), device=I.device)
    return (F.cross_entropy(logits, labels) +            # image → text
            F.cross_entropy(logits.T, labels)) / 2       # text → image
```

`I @ T.T` builds the grid of Step 4. `F.cross_entropy` is the softmax-based "pick the right
one" loss of Chapter 12. In simple words, each photo must pick its caption, each caption must
pick its photo, and the loss is the average of how badly both did.

Eight lines. It is the same contrastive loop as word2vec and sentence embeddings, now bridging
two modalities.

Three things made it work at the scale it did.

**Scale of data.** 400 million pairs. Alt text is free, noisy and abundant. It is the same
"naturally paired data" idea as Chapter 12's question–answer pairs.

**Enormous batches.** 32,768 pairs per batch. As Chapter 12 explained, the batch size *is* the
number of negatives, and more negatives make a harder quiz and a better model.

**A learned temperature.** The **temperature** of Chapter 12 is a number the model learns rather
than a setting we pick. It starts at 0.07 and is capped so it never scales the scores by more
than 100, which keeps training stable.

---

## An example of CLIP

Let's go back to the designer and the query *"a dog sitting under a desk"*.

**Phase 1: Indexing the photo library.**

**Step 1:** Run all 10,000 photos through the image tower once.

**Step 2:** Store the 10,000 normalized vectors in any index from Parts III and IV.

**Phase 2: Answering the query.**

**Step 1:** Run the sentence through the text tower. We get one vector.

**Step 2:** Find the nearest photo vectors by dot product.

**Step 3:** Return the top results.

The top results are photos of the office dog under desks. No file name mattered. No tag
mattered. The hot dog stand is nowhere near, because its photo shows food and a crowd, and
the sentence describes an animal in an office. The answer is correct.

Now the second trick, **zero-shot classification**. **Zero-shot** means doing a task with no
training examples for that task.

The marketing team wants to sort all 10,000 photos into four folders. Nobody wants to label
examples. So we write the folder names as captions:

- *"a photo of an office"*
- *"a photo of a conference stage"*
- *"a screenshot of software"*
- *"a photo of a team dinner"*

We embed the four captions with the text tower. For each photo, we pick the caption whose vector
is nearest. That caption is the folder.

In simple words, the labels are just more sentences, and classification is just search over a
very short list.

This was the result that made CLIP famous. On ImageNet, a standard test with 1,000 categories,
zero-shot CLIP matched the accuracy of a classic ResNet-50 classifier (an older, widely used
image classifier network). That classifier had trained on 1.28 million labelled ImageNet photos.
CLIP had seen none of them.

---

## Problems with CLIP

CLIP is excellent at the gist of a natural photograph: objects, scenes, styles, colours and broad
activities. *"A dog on a beach at sunset"* works beautifully. So what does it get wrong? There are
six problems, and they matter more than people expect.

**Problem 1: Counting.** CLIP struggles to tell *"two dogs under a desk"* from *"three dogs
under a desk"*. Captions rarely hinge on exact counts, so the training signal was never there.

**Problem 2: Spatial relations.** *"A dog sitting on a desk"* and *"a dog sitting under a
desk"* get very close vectors. The designer searches for one and gets both. CLIP behaves like
a bag of concepts, much like Chapter 8's bag of words.

**Problem 3: Text inside images.** One photo shows a whiteboard with the Q3 roadmap written on
it. Searching *"Q3 roadmap"* rarely finds it. CLIP can sometimes read a big, bold word, but it
cannot read a paragraph in a screenshot. **This is the most important limitation for document
search**, and it is why ColPali (Chapter 45) needed a different foundation.

**Problem 4: Fine-grained distinctions.** The 2024 and 2025 conference stages look almost the
same. So do two versions of one chart. Web captions are too coarse to teach such small
differences.

**Problem 5: Negation.** *"An office with no dog"* returns photos of the office dog. The word
"no" barely moves the vector, the same problem text-only models have (Chapter 34).

**Problem 6: The modality gap.** The **modality gap** is a measurable offset between where image
vectors sit and where text vectors sit, even after training. Images cluster near images and
texts near texts. If one index holds both photos and their captions, a text query prefers the
captions, simply because they are text. The Ninja notes give two fixes.

> **The rule for all of Part VI:** a model's embedding space encodes the kind of similarity its
> training data rewarded. CLIP learned *"what would someone write as a caption for this?"* That is
> an excellent guide to scene content and a poor guide to anything a caption would not mention.

---

## CLIP vs tags and captions

There are three common ways to make photos searchable by text.

| | Keyword tags | Caption each photo, then text embeddings | CLIP |
|---|---|---|---|
| Needs human effort per photo | Yes, a lot | No, a captioning model writes them | No |
| Finds untagged photos | No | Yes | Yes |
| Handles paraphrase ("pup" vs "dog") | No | Yes | Yes |
| Exact names, codes, printed words | Yes, if tagged | Only if the caption copies them | Weak |
| Counting, spatial relations | Only if tagged | Depends on caption detail | Weak |
| Search by example photo | No | Indirect | Yes |
| Cost to index | People's time | One captioning pass plus one text-embedding pass | One image-encoder pass |

**Advantages of CLIP.** No labelling. One index serves text queries and photo queries. Zero-shot
classification comes free.

**Disadvantages of CLIP.** Weak at counting, spatial words, negation, fine detail and reading
text. Its text tower is a poor general sentence embedder. The modality gap needs care in mixed
indexes.

---

## When to use which one

We must use **CLIP** when the collection is natural photos, product shots or artwork, when most
items have no useful text, and when people search by describing what they want to see.

We must use **tags or a keyword field** when people search by exact names, codes or IDs, such as a
product SKU or an event name.

We must use **a document model** (Chapters 45 to 47) when the images are pages, screenshots,
slides or scans, and the answer lives in the text on them.

Many strong photo-search systems use both: CLIP for the description, a keyword field for the exact
names, and the fusion of Chapter 40 to merge the two lists.

---

### Under the hood

The same photo search, with the open-source `open_clip` library:

```python
import torch, open_clip
from PIL import Image

model, _, preprocess = open_clip.create_model_and_transforms(
    "ViT-B-32", pretrained="laion2b_s34b_b79k")
tokenizer = open_clip.get_tokenizer("ViT-B-32")

def embed_image(path):
    x = preprocess(Image.open(path)).unsqueeze(0)
    with torch.no_grad():
        v = model.encode_image(x)
    return torch.nn.functional.normalize(v, dim=-1)[0]

def embed_text(s):
    with torch.no_grad():
        v = model.encode_text(tokenizer([s]))
    return torch.nn.functional.normalize(v, dim=-1)[0]

# image_matrix: the 10,000 photo vectors from embed_image, stacked row by row
scores = image_matrix @ embed_text("a dog sitting under a desk")
best   = scores.topk(10).indices
```

`embed_image` is Phase 1 for one photo. `embed_text` plus the dot product is Phase 2. In simple
words, the whole search is one matrix multiplication.

Once both modalities are normalized vectors in one space, **everything in Parts III and IV
applies unchanged.** HNSW does not know or care that half its vectors came from pixels. That is
the practical payoff of a shared space: the whole retrieval stack stops caring about modality.

One detail that catches people: the wording of the text matters. That wording is often called the
**prompt**, the text we hand a model (Chapter 49 uses the word for the text we give an LLM). CLIP
was trained on caption-like text, so `"a photo of a dog"` embeds differently from `"dog"`, and
usually works better for classification. **Prompt ensembling** means averaging the vectors of
several wordings, such as "a photo of a dog" and "a picture of a dog". The original CLIP paper
used 80 such templates for ImageNet. It is a standard, free improvement.

---

### What people get wrong

**"CLIP can read documents."** It cannot. Its ability to read text in images is rudimentary at
best. Do not build document search on it.

**"CLIP embeddings are comparable to my text model's."** They live in a different space entirely
(Chapter 2). Only CLIP's own text tower shares CLIP's image space.

**"CLIP's text encoder is a good sentence embedder."** It is weak at that job. It was trained on
short captions with a 77-token limit, and optimised for matching images, not for judging whether
two sentences mean the same thing. Use a proper text embedding model for text.

**"Zero-shot means no work."** Label wording matters enormously. Test several templates.

**Ignoring the modality gap.** Image and text vectors occupy neighbouring but separate cones in
the shared space. They are not fully mixed. So image-to-image scores and text-to-image scores sit
on different scales, and a single similarity threshold for both is wrong. Calibrate each
separately.

---

### Ninja notes

**The modality gap deserves more attention than it gets.** Even after training, images cluster
with images and texts with texts, with a measurable offset between the two groups. A text query's
nearest *text* neighbours score much higher than its nearest *image* neighbours, purely because of
that offset.

So a mixed index of photos and captions will systematically prefer the captions. Two fixes. Keep a
separate index per modality and fuse the result lists with RRF (Chapter 40). Or centre each
modality by subtracting its own mean vector before indexing, then renormalize. The second is one
line of code and often removes much of the offset. Measure on your own data either way.

**Fine-tuning CLIP on a domain is unusually effective.** The base model already knows general
visual structure. Adapting it to product photos, medical images or satellite imagery often takes
thousands of pairs, not millions. If your product already has image–text pairs, such as catalogue
descriptions, this is a high-yield project.

**CLIP shipped with ResNet image towers as well as ViTs.** The ViT versions are the ones most
people use today.

---

### Key takeaways

- **CLIP = two encoders, one for images and one for text, trained so matching pairs land at the
  same point.** The loss is Chapter 12's InfoNCE, applied in both directions.
- **ViT = a transformer that cuts an image into patches and reads each patch like a token.**
- The shared space turns text→image search, image→text search and **zero-shot** classification
  into ordinary nearest-neighbour lookups.
- CLIP learned "what caption would this get". It is excellent at scene gist and poor at counting,
  spatial relations, negation, fine detail and text inside images.
- Once vectors share a space, all of Parts III and IV apply unchanged.
- Mind the **modality gap**: centre each modality, or index them separately and fuse.
- Domain fine-tuning is cheap and effective.

### What's next

[Chapter 43](./43-siglip.md) covers what changed after CLIP: a simpler loss, higher
resolution, and the models that finally learned to read a page.

Now we understand CLIP, why a shared space for pictures and words is useful, how it is trained,
and why it still cannot read the text in a picture.
