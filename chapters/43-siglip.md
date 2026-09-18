---
title: "SigLIP and Modern Vision-Language Encoders"
chapter: 43
part: "Part VI — Vectors Beyond Text"
slug: siglip
readingTime: "13 min"
summary: "Replace the softmax with a sigmoid and the batch-size tyranny disappears. Plus the other changes that turned vision encoders into something that can read a page."
tags: [siglip, vision-encoders, sigmoid-loss, vit, dinov2]
prev: 42-clip
next: 44-ocr-broken-promise
---

# SigLIP and Modern Vision-Language Encoders

**The one-paragraph version.** CLIP's loss normalises every image's scores over all the captions
in the batch, which forces enormous batches and heavy communication between GPUs. **SigLIP** swaps
CLIP's softmax for a **sigmoid** on each pair. The sigmoid σ squashes any number into a 0–1
probability, so each image–caption pair gets its own independent yes/no score instead of
competing with the whole batch. That change trains well at modest batch sizes and scores
better. SigLIP is also the vision encoder inside PaliGemma, Google's open vision-language model,
which is the foundation of the document retriever ColPali (Chapter 45).

In this chapter, we will learn what SigLIP changes about CLIP's training and why one small swap
from softmax to sigmoid matters. We will see how SigLIP became the "eyes" of the document
retrievers in Chapters 45 to 47. We will also see the other changes that separate a 2021 vision
encoder from a modern one.

We will cover the following:

- What is SigLIP
- Softmax vs sigmoid
- Why CLIP's softmax is a problem
- How SigLIP is trained
- An example: one photo, two batches
- Why SigLIP matters for documents
- What else changed after CLIP
- SigLIP vs CLIP, and when to use which one

---

## What is SigLIP

**SigLIP = Sigmoid Loss for Language–Image Pre-training.**

- **Sigmoid Loss:** a training loss built from the sigmoid function, which we define in the next
  section.
- **Language–Image Pre-training:** the same job as CLIP (Chapter 42). Two towers, one for images
  and one for text, learn to put matching pairs at the same point.

SigLIP came from Google in 2023. The architecture is CLIP's. The data is similar. The one big
change is the loss.

In simple words, CLIP asks *"which of these captions is right?"*, and SigLIP asks *"is this
caption right, yes or no?"*, once for every pair.

---

## Softmax vs sigmoid

Before we see why the swap matters, we must know the two functions. Both turn raw scores into
probabilities. They differ in what they look at.

**Softmax** (Chapter 9) takes a whole *list* of scores and turns them into probabilities that
add up to 1. Raise one score and every other probability must go down. The scores compete.

**Sigmoid** takes *one* score at a time:

$$ \sigma(x) = \frac{1}{1 + e^{-x}} $$

It squashes any number into the range 0 to 1. A score of 0 becomes 0.5. A score of 3 becomes
about 0.95. A score of −3 becomes about 0.05.

Means, a sigmoid is a yes/no judge for a single score. It never looks at any other score.

Think of it like two kinds of exam:

- Softmax is a multiple-choice question. Picking one answer means rejecting all the others.
- Sigmoid is a stack of true/false questions. Each one is marked on its own.
- The captions in a batch are the answer options.
- The photo is the question being asked.

---

## Why CLIP's softmax is a problem

CLIP's loss is a multiple-choice question: *given this photo, which of the 32,768 captions in the
batch is correct?* The softmax divides each pair's score by a total taken over every caption in
the batch. That denominator ties every pair to every other pair.

This causes three problems.

**Problem 1: A pair's score depends on who else is in the batch.** The same photo and the same
caption get a different probability when the other captions change. We see this with numbers
in the example below.

**Problem 2: Quality depends on batch size.** More captions in the batch make a harder quiz,
so CLIP-style training needs huge batches to learn well.

**Problem 3: The whole grid must be gathered.** To compute each row's denominator, every GPU
needs every other GPU's vectors. Collecting them is a **global gather**, and at 32,768 pairs it
is a serious communication and memory cost.

---

## How SigLIP is trained

Let's take the same setup as CLIP: a batch of N (image, caption) pairs, two towers, normalized
vectors.

**Step 1:** Encode every image and every caption, exactly as CLIP does.

**Step 2:** Compute the N × N grid of dot products.

**Step 3:** Label each cell. The diagonal (a photo with its own caption) is "yes". Every other
cell is "no".

**Step 4:** For each cell, multiply the dot product by a learned temperature t, then add a
learned bias b.

**Step 5:** Put that number through a sigmoid to get a yes-probability.

**Step 6:** Penalise each cell on its own: a "yes" cell with a low probability, or a "no" cell
with a high one.

**Step 7:** Average the penalties over the batch and nudge the weights of both towers.

As a formula:

$$ \mathcal{L} = -\frac{1}{N}\sum_{i}\sum_{j} \log \sigma\!\left(z_{ij}\,(t \cdot x_i \cdot y_j + b)\right) $$

Here $x_i$ is image i's vector and $y_j$ is caption j's vector. $z_{ij} = +1$ for a matching pair
and $-1$ otherwise. $t$ is the learned temperature and $b$ the learned bias. The paper starts $t$
at 10.

In simple words, every cell of the grid is its own true/false question. No softmax. No
denominator. **Each pair is an independent binary classification.**

**Note:** This temperature works the other way round from Chapter 42's. CLIP *divides* the dot
products by its temperature, which starts at 0.07. SigLIP *multiplies* them by $t$. So $t$ is the
inverse of CLIP's temperature: multiplying by $t = 10$ is the same as dividing by 0.1.

**Note:** The learned bias b matters more than it looks. In a batch of N pairs, only N cells are
"yes" and N² − N are "no". If every cell started as a 50/50 guess, the "no" cells would swamp the
loss, and the first training steps would be huge corrections just to push them all down. So the
bias starts strongly negative (−10 in the paper), which makes "no" the starting guess, since
almost every cell *is* a no. Training then spends its effort pulling the few "yes" cells up.

The practical effects:

- **Batch size stops being a quality bottleneck.** The paper found the sigmoid loss clearly ahead
  of softmax at smaller batches, such as 4k and 16k. For both losses, gains from bigger batches
  levelled off around 32k.
- **No global gather.** Each GPU computes the loss for its own images against the captions it
  holds, then passes those captions on to the next GPU. No GPU ever needs the full grid in memory.
- **Better results at equal compute**, as reported in the SigLIP paper across model sizes.

---

## An example: one photo, two batches

Let's use Acme's photo library from Chapter 42. The photo is the office dog under a desk. These
are toy numbers, with t = 10 and b = −5, chosen to make the arithmetic easy to follow.

**Batch A** has four captions. The photo's dot products with them are:

```
"a dog sitting under a desk"   0.62   ← its own caption
"a conference stage"           0.20
"a team dinner"                0.15
"a laptop on a table"          0.10
```

Start with CLIP's softmax. It multiplies by t and takes the softmax across the row. The correct
caption gets probability **0.97**.

**Batch B** is the same batch plus one more caption, which belongs to a different photo of the
same dog:

```
"a dog lying under a desk"     0.60
```

Nothing about our photo changed. Nothing about its own caption changed. The softmax now gives the
correct caption only **0.54**, because a strong rival appeared in the denominator.

In simple words, CLIP's verdict on this one pair moved from "almost certainly" to "a coin flip",
purely because of a stranger in the batch.

Now, let's see what SigLIP's sigmoid does. For the correct pair, it computes
σ(10 × 0.62 − 5) = σ(1.2) = **0.77**. In Batch A the answer is 0.77. In Batch B it is still
0.77. The sigmoid never looked at the other captions.

The new caption is judged on its own too. σ(10 × 0.60 − 5) = σ(1.0) = 0.73, and its label is
"no", so only that one cell is penalised. The other three captions score 0.05, 0.03 and 0.02,
which are comfortably "no".

**Note:** Neither loss knows that "a dog lying under a desk" also describes our photo. Both call
it a negative. That problem, a **false negative**, is the same one Chapter 12 described. SigLIP
does not fix it. It only stops one false negative from dragging down the score of every other
pair in the row.

---

## Why SigLIP matters for documents

SigLIP is a better CLIP, which is mildly interesting. What makes it matter for this book is that
**SigLIP-So400m is the vision encoder inside PaliGemma**, and PaliGemma is the backbone of ColPali
(Chapter 45). ("So400m" is the name of a shape-optimised ViT with about 400 million parameters.)

Before the chain, two names.

**VLM = Vision-Language Model.** It is a language model that can also take images as input, and
answer questions about them in text.

**Gemma** is Google's family of small, open language models. Gemma-2B has about 2 billion
parameters.

Now the chain:

```
SigLIP-So400m            →  a strong ViT that produces patch-level features
        ↓ combined with Gemma-2B
PaliGemma-3B             →  a vision-language model that can READ pages
        ↓ + 128-d projection + late-interaction training
ColPali                  →  a document retriever that needs no OCR
```

Each step adds one capability:

- **SigLIP** supplies visual features good enough for a language model to interpret.
- **PaliGemma** is SigLIP joined to Gemma (Chapter 46 opens it up). It supplies the language
  model, trained on tasks including document understanding, chart reading and transcribing text
  from images. That is where the ability to *read* comes from.
- **ColPali** supplies the retrieval objective. The result searches pages without OCR (optical
  character recognition, the usual step that turns a picture of text into characters, Chapter 44).

**CLIP alone could not have been the foundation.** As Chapter 42 showed, CLIP cannot read text in
images. The reading ability came from PaliGemma's training mixture, not from the contrastive loss.

---

## What else changed after CLIP

Beyond the loss, several developments separate a 2021 vision encoder from a modern one.

**Higher and dynamic resolution.** CLIP usually ran at 224 × 224 pixels. That is enough to see a
dog. Squeeze an A4 page into it and a letter of body text is only two or three pixels tall, which
no model can read. Modern encoders run at 384, 448 or 896 pixels, or accept **variable
resolution** entirely. Qwen2-VL, a VLM from Alibaba, uses what its authors call "naive dynamic
resolution". It processes an image near its native size and aspect ratio, and turns it into a
variable number of visual tokens. For pages of different shapes and sizes, that is exactly what we
want. Chapter 47 puts numbers on it.

**Patch-level outputs are kept, not pooled.** CLIP pools its patches into one image vector. For
document search we want the patches, so the region holding the relevant table can be matched on
its own. This is Chapter 11's pooling argument, arriving in pictures. It is what makes late
interaction over images possible.

**Document-heavy pretraining.** Modern VLMs train on charts, tables, screenshots, receipts,
scientific figures and forms. That mixture is why they can answer *"what was Q3 revenue?"* from a
bar chart.

**Self-supervised alternatives.** DINOv2, from Meta, learns strong visual features with no text at
all, through self-distillation (a student network learns to match a teacher network's view of the
same image). It gives excellent image-to-image similarity, usually better than CLIP for pure visual
search. But it has no shared text space. Use it for "find visually similar images", not "find
images matching this sentence".

**SigLIP 2 and PaliGemma 2.** Google released **SigLIP 2** in early 2025. It keeps the sigmoid
loss and adds more. Its training data is multilingual. During training, it also attaches a small
text decoder that learns captioning and localisation (saying *where* in an image something is),
plus self-distillation-style objectives that improve the per-patch features. A variant called
**NaFlex** accepts several resolutions and keeps an image's native aspect ratio. We understand the
decoder to be a training aid only, so the finished encoder is used the way SigLIP is. Treat the
details as the paper's claims, and check them against your own evaluation. Shortly before, in late
2024, **PaliGemma 2** paired the same SigLIP-So400m encoder with the newer Gemma 2 language
models, at several sizes.

---

## SigLIP vs CLIP, and when to use which one

| | CLIP | SigLIP |
|---|---|---|
| Loss | Softmax over the batch (multiple choice) | Sigmoid per pair (true/false) |
| Pair score depends on the rest of the batch | Yes | No |
| Needs a global gather across GPUs | Yes | No |
| Behaviour at modest batch sizes | Degrades | Trains well |
| Output of the model is a per-pair probability | No, only a ranking within a list | Yes, roughly |
| Typical resolution | 224 px | 224 to 512 px, plus larger variants |
| Role in document retrieval | None, it cannot read | Vision encoder inside PaliGemma and ColPali |

**Advantages of SigLIP.** Cheaper training at a given quality. Scores that mean something on their
own. A direct line to document models.

**Disadvantages of SigLIP.** On its own, it still cannot read a page. It needs a language model
and document training on top. It is also sensitive to how text is padded, as the code below shows.

We must use **SigLIP-family encoders** for new image–text search, zero-shot classification, and as
the vision half of any document model we build or fine-tune.

We must use **CLIP** when an existing system, index or fine-tuned model already depends on it, and
re-embedding everything is not worth it yet.

We must use **DINOv2-style encoders** when the job is image-to-image similarity with no text.

Many strong systems use two: a SigLIP-family model for text-to-image search, and a
self-supervised encoder for near-duplicate detection.

---

### Under the hood

The same office-dog photo, through SigLIP with Hugging Face `transformers`:

```python
from transformers import AutoProcessor, AutoModel
from PIL import Image
import torch

proc  = AutoProcessor.from_pretrained("google/siglip-so400m-patch14-384")
model = AutoModel.from_pretrained("google/siglip-so400m-patch14-384")
office_dog_photo = Image.open("office_dog.jpg")

inputs = proc(text=["a dog sitting under a desk", "a conference stage"],
              images=[office_dog_photo], return_tensors="pt", padding="max_length")
with torch.no_grad():
    out = model(**inputs)

# Pooled embeddings (CLIP-style use)
img_vec, txt_vec = out.image_embeds, out.text_embeds

# Independent yes-probabilities: sigmoid, not softmax
probs = torch.sigmoid(out.logits_per_image)     # each pair judged on its own

# Patch-level features — what ColPali actually builds on
patches = model.vision_model(inputs["pixel_values"]).last_hidden_state
patches.shape     # (1, 729, 1152) for 384px with patch size 14
```

`torch.sigmoid` is Step 5 in code. The two probabilities do not have to add up to 1, because each
pair was judged alone.

The last line is the one to remember. 384 ÷ 14 rounds down to 27, and 27 × 27 = **729 patch
vectors**, each describing a small region of the image. Run a page image through the same call
and each patch describes a small region of the page. A single pooled vector throws all of that
away. Keeping it is the entire idea of Chapter 45.

Note also `padding="max_length"`. SigLIP was trained with text padded to a fixed length, and it
behaves differently without that padding. In simple words, feed the model text in the same shape
it saw in training. That is Chapter 11's rule again.

---

### What people get wrong

**"SigLIP is just CLIP with a different loss."** The loss change is what made efficient training
possible at the scales and resolutions that document understanding needs. It is an enabling
change, not a cosmetic one.

**"Any vision encoder can read documents."** Reading needs resolution *and* training on
text-bearing images. A 224px CLIP has neither.

**"I should pool patch features into one vector."** For scene-level similarity, yes. For documents,
that deliberately brings back the bottleneck of Chapter 34.

**"Higher resolution is strictly better."** Visual tokens grow roughly with pixel count, so doubling
the resolution in both directions quadruples storage and scoring cost, and at least quadruples
encoding compute. Resolution is a budget, not a free setting. Chapter 47 puts numbers on it.

---

### Ninja notes

**Aspect ratio is a real, underappreciated failure mode for documents.** A model that resizes
everything to a square distorts an A4 page, and distortion hurts text legibility more than it hurts
scene recognition. Models with dynamic resolution (Qwen2-VL and its descendants, hence ColQwen2 in Chapter 47)
preserve aspect ratio, which is part of why they do better on documents.

**Know your patch budget, because it drives the cost.** A 448 × 448 image with patch size 14 gives
$32 \times 32 = 1{,}024$ patches. At 896 × 896 it gives 4,096. In a late-interaction system, that
is *four times the vectors per page*: four times the storage, four times the scoring cost.
Chapter 47 shows the trade-off with real numbers, but internalise the relationship now: **visual
tokens scale with pixel count, and everything downstream scales with visual tokens.**

**For pure visual similarity, try DINOv2 before CLIP.** If you are deduplicating images, finding
visually similar products or detecting near-duplicate scans, a self-supervised encoder usually beats
a caption-trained one. The caption objective teaches "what would someone say about this", which is
a lossy proxy for "what does this look like".

**SigLIP's probabilities are usable as-is, within reason.** Each pair is scored alone. So a
threshold on SigLIP's sigmoid output means more than a threshold on CLIP's softmax output, which
only ranks captions against each other. It is still a model's calibration, not the truth, so
check it on labelled examples.

---

### Key takeaways

- **SigLIP = CLIP's two towers + a sigmoid on each pair instead of a softmax over the batch.**
- **Sigmoid = a function that squashes one score into 0–1.** Softmax spreads probability across a
  whole list, so scores compete. Sigmoid judges each score alone.
- A softmax score for one pair changes when the rest of the batch changes. A sigmoid score does not.
- SigLIP trains well at modest batch sizes, avoids a global gather across GPUs, and scores higher at
  equal compute.
- SigLIP-So400m → PaliGemma → ColPali: each step adds a capability, and the ability to *read* comes
  from PaliGemma's training mixture, not from the contrastive loss.
- Modern encoders keep patch-level outputs and support high or dynamic resolution. SigLIP 2 adds
  multilingual data, better per-patch features and a native-aspect-ratio variant.
- Aspect ratio matters for documents, and visual token count scales with pixels.
- DINOv2-style self-supervised encoders are better for pure image-to-image similarity.

### What's next

[Chapter 44](./44-ocr-broken-promise.md) examines the conventional document pipeline (OCR, layout,
tables, reading order, chunking, embedding) and everything it throws away before a vector is ever
computed.

Now we understand SigLIP, why a per-pair sigmoid beats a batch-wide softmax, and how it became the
vision half of the models that can read a page.
