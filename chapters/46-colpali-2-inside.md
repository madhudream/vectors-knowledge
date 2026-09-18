---
title: "ColPali II: Inside the Model"
chapter: 46
part: "Part VI — Vectors Beyond Text"
slug: colpali-2-inside
readingTime: "12 min"
summary: "Patches, projection, and late interaction over pixels. How a page becomes a thousand vectors, how the model was trained, and how to read the similarity heatmaps it gives you for free."
tags: [colpali, paligemma, patches, training, interpretability, vidore]
prev: 45-colpali-1-documents-as-images
next: 47-colpali-3-production
---

# ColPali II: Inside the Model

**The one-paragraph version.** A page image is cut into a grid of patches. SigLIP turns each patch
into a vector. Gemma, a language model, lets every patch look at every other patch, so each one
"knows" the whole page. A learned layer then shrinks each vector to 128 numbers. Training pushes
each query's true page above the strongest wrong page in its batch, using about 119,000 (query,
page) pairs (118,695). Roughly a third were generated synthetically by a vision-language model,
and two-thirds came from open visual question-answering datasets. The result is a retriever whose
matches we can see.

In this chapter, we will open ColPali and follow page 3,507, the scanned Globex contract, from
pixels to stored vectors. We will learn what PaliGemma is, why a language model sits inside a
retriever, what the 128-dimensional projection buys, and how the model was trained cheaply. We will
also draw a heatmap that shows where on the page each query word found its evidence.

We will cover the following:

- What is PaliGemma
- From pixels to patches
- Contextualisation: the step that matters
- The 128-dimensional projection
- How ColPali was trained
- Reading the heatmap
- When to fine-tune ColPali

---

## What is PaliGemma

**PaliGemma = SigLIP (the eyes) + a projection + Gemma (the reader).**

- **SigLIP** (Chapter 43) turns an image into patch vectors.
- **A projection** is one learned layer that maps each vector from one length to another. Here it
  resizes SigLIP's patch vectors to the size Gemma expects.
- **Gemma** is Google's small open language model (Chapter 43). It reads the patch vectors as if
  they were word tokens, alongside any text we give it.

PaliGemma-3B uses the 2-billion-parameter Gemma, so the whole model has about 3 billion parameters.
Google trained it on a broad mix of image tasks, including captioning, reading text in images,
answering questions about charts and documents, and locating objects.

In simple words, PaliGemma is a language model that has been given eyes and taught to read what
they see. ColPali keeps PaliGemma's eyes and reader, and swaps its "write an answer" job for a
"make search vectors" job.

---

## From pixels to patches

A vision transformer does not process pixels one by one. It cuts the image into a grid of
fixed-size squares, the **patches** of Chapter 42, and treats each one as a token, exactly as a text
transformer treats a word piece.

ColPali uses PaliGemma's 448 × 448 version, with 14-pixel patches:

$$ \frac{448}{14} = 32 \quad\Rightarrow\quad 32 \times 32 = 1{,}024 \text{ patches} $$

Means, 32 patches across, 32 down, 1,024 in all.

```
┌────┬────┬────┬ ... ┬────┐
│ p0 │ p1 │ p2 │     │p31 │     32 patches across
├────┼────┼────┼ ... ┼────┤
│p32 │p33 │    │     │    │     32 patches down
├────┼────┼────┼ ... ┼────┤
│    │    │    │     │    │     = 1,024 patches
└────┴────┴────┴ ... ┴────┘
```

How much of page 3,507 does one patch cover? The A4 page is squeezed into the 448 × 448 square, so
one 14 × 14 patch covers about a quarter of an inch across and a third of an inch down. For normal
body text, that is a few characters across one or two lines. For a chart, it is a small piece of one
bar or one label.

Add a handful of vectors for the short text prompt (the fixed instruction text read alongside every
page image) and we get the ~1,030 vectors per page from Chapter 45.

**Patches form a grid, so position is kept.** Patch 33 sits directly below patch 1. The model
receives each patch's position, so it knows whether a patch is in the heading, the left margin or
the pricing table. This is exactly the structure the OCR pipeline of Chapter 44 destroyed in its
reading-order stage.

---

## Contextualisation: the step that matters

SigLIP's patch vectors already carry some context, because inside the ViT the patches attend to each
other. But SigLIP learned to match photos with captions, not to read. Its vectors tend to say more
about how a region *looks* than about what the words in it *mean*.

Now, the question is, how does a patch holding the word "included" come to mean "SSO is included"?

The patches pass through **Gemma**, with full attention across all 1,024 of them. Every patch attends
to every other patch. Let's follow page 3,507.

**Step 1:** The patches holding `included (Pro, 50+ seats)` attend to the row label `SSO` on the same
line.

**Step 2:** They also attend to the column header `Terms` above them.

**Step 3:** Their vectors now encode something like *"the SSO row's terms: included for Pro at 50+
seats"*, not merely *"the word included"*.

The same happens across the page. The patches of the signature block attend to *"Signed for Globex
Corporation"*. The patches of a clause attend to the heading *"Schedule B: Pricing"* above it.

**This is Chapter 10's contextualisation, applied across space instead of along a sentence.** It is
why the model can answer questions about tables. The link between a cell and its headers is rebuilt
by attention, from the layout alone, without anyone writing a table parser.

It is also the reason a language model is in the stack at all. A vision encoder alone gives mostly
appearance. The language model adds *document understanding*.

---

## The 128-dimensional projection

Gemma's internal vectors are large: 2,048 numbers each for Gemma-2B. Storing 1,024 of them per page
in float32 would take

$$ 1{,}024 \times 2{,}048 \times 4 \text{ bytes} \approx 8.4 \text{ MB per page} $$

For a million pages that is far too much.

So ColPali adds a **projection** at the very end: one learned linear layer that maps each patch
vector from 2,048 numbers to **128**. That is the same width ColBERT uses for text tokens
(Chapter 35). A page becomes

$$ 1{,}030 \times 128 \times 4 \text{ bytes} \approx 527 \text{ KB} $$

In simple words, the projection keeps the page's patches and squeezes each one's 2,048 numbers into
128.

That is still large, and Chapter 47 attacks it. But it is 16× smaller, and the layer is trained
together with the retrieval objective. So it learns to keep exactly what MaxSim needs, not what
rebuilding the image would need.

The vectors are also normalized to length 1 (Chapter 5). So every query-token-to-patch dot product
lies between −1 and 1. The MaxSim total stays readable as "how many parts of the query found good
evidence" (Chapter 36).

---

## How ColPali was trained

**The data problem.** Training a retriever needs (query, relevant page) pairs. Almost no such dataset
existed for visually rich documents, which is part of why this approach arrived late.

**The solution was a mix of borrowed and generated data.** ColPali's training set has about 119,000
pairs in total (118,695).

- **About two-thirds were borrowed** from open **VQA** training sets. **VQA = visual question
  answering.** These are collections of document images, each with questions written about it.
  They include **DocVQA** (scanned documents), **InfoVQA** (infographics), **TAT-DQA** (financial
  reports with tables) and **arXivQA** (figures from scientific papers). Each question, paired with
  its page, is a ready-made (query, page) pair.
- **About a third were generated synthetically.** The team collected PDF pages from the web across
  many topics. For each page, they prompted a strong vision-language model: *"Write a question that
  this page answers."* That gives a query whose relevant page is known by construction.

This is a pattern worth stealing. **When you lack retrieval training data for your own domain,
generate queries from your own documents with a capable model.** It is the same "naturally paired
data" logic as Chapter 12, manufactured instead of found. The data is imperfect, because synthetic
queries are cleaner and more literal than real ones. But it is vastly better than nothing, and it is
how you would fine-tune ColPali on your own corpus.

**The objective** is ColBERTv1's pairwise loss (Chapter 36), with MaxSim as the score and the
negative picked from the batch:

**Step 1:** Take a batch of (query, page) pairs.

**Step 2:** Score every query against every page in the batch with MaxSim.

**Step 3:** For each query, find the highest-scoring wrong page in the batch.

**Step 4:** Push the true page's score above that one, and update the trainable weights. The loss
for one query is

$$ \text{loss} = \log\left(1 + e^{\,s_{\text{wrong}} - s_{\text{true}}}\right) $$

Means, the loss is close to 0 when the true page clearly beats the strongest wrong page, and it grows
as the wrong page catches up. This curve is called **softplus**. The batch loss is the average over
all queries. Unlike Chapter 12's InfoNCE, there is no softmax over every page in the batch and no
temperature.

In simple words, the true page only has to beat the strongest wrong page.

**The efficiency trick is LoRA.** **LoRA = Low-Rank Adaptation.** It is a small set of add-on weights
trained in place of the full model. Take one of Gemma's big weight matrices, 2,048 × 2,048, about 4.2
million numbers. LoRA freezes it, and trains two thin matrices beside it, say 2,048 × 32 and
32 × 2,048, whose product is added on. That is 131,072 trainable numbers instead of 4.2 million, 32×
fewer.

In simple words, LoRA leaves the big model alone and trains a small correction next to it.

In ColPali, only LoRA adapters are trained, on Gemma's layers and on the final projection layer. The
SigLIP encoder stays frozen, and so do the original weights underneath the adapters. So training is
cheap, on the order of hours on a small number of GPUs rather than weeks. That is also excellent news
if you want to adapt it to your own documents.

---

## Reading the heatmap

Because scoring happens per query token, we can see *where on the page* each query token found its
best match. No single-vector system can offer that.

**Step 1:** Pick one query token, for example "SSO".

**Step 2:** Compute its dot product with each of the 1,024 image patches.

**Step 3:** Arrange the 1,024 scores back into the 32 × 32 grid.

**Step 4:** Colour the grid and lay it over the page image.

```python
def patch_heatmap(q_vecs, page_vecs, token_index, grid=(32, 32)):
    sims = (q_vecs[token_index] @ page_vecs.T)      # (n_patches,)
    return sims[:grid[0] * grid[1]].reshape(grid)   # → overlay on the page image
```

The image patches come first in the stored sequence, so the first 1,024 scores are the grid. In simple
words, the heatmap is one query token's similarity to every square of the page, drawn on the page.

Overlay it on page 3,507 and we see what we would hope. For the query *"Does the Globex contract
include SSO?"*, the token "SSO" lights up the row label in the pricing table. "include" lights up the
"included (Pro, 50+ seats)" cell. "Globex" lights up the page heading.

**Note:** This is a similarity map, not a picture of the model's internal attention. It shows which
stored patch vectors matched, which is exactly the evidence the score was built from.

Three genuine uses:

**Debugging.** A query that returns the wrong page becomes diagnosable. We might see that "Globex"
matched a footer on every page, not the pricing table.

**User experience.** Return the page with the matched row highlighted. Users trust a result when they
can see the evidence.

**Compliance and audit.** In regulated settings, "why was this document retrieved?" has a visual
answer.

---

## When to fine-tune ColPali

Now we know what is inside, we can decide whether to adapt it.

**Advantages of fine-tuning.** Only LoRA adapters train, on Gemma and the projection, so it takes
hours, not weeks. Synthetic queries from your own pages remove the need for hand labels. Domain terms
and layouts the model rarely saw, such as Acme's order forms, can improve noticeably.

**Disadvantages of fine-tuning.** Synthetic queries are cleaner than real ones, so gains measured on
them can be optimistic. Every fine-tune means re-embedding the whole corpus. And it does not fix the
fixed 448 × 448 resize.

We must **use ColPali as it is** when the documents look like its training data (reports, forms,
slides, papers) and a golden set (Chapter 19) shows it already does well.

We must **fine-tune** when the documents have unusual layouts or vocabulary, and the golden set shows a
clear gap.

We must **switch to a dynamic-resolution model** (Chapter 47) when the problem is fine print or unusual
page shapes, because no amount of fine-tuning restores pixels the resize threw away.

---

### Under the hood

Here is the retrieval head ColPali adds on top of PaliGemma, plus the scoring function.

**Step 1:** Take Gemma's final vectors for every patch and prompt token.

**Step 2:** Project each one from 2,048 numbers to 128.

**Step 3:** Normalize each to length 1.

**Step 4:** Zero out padding positions with the attention mask.

**Step 5:** To score, take all query-token-to-patch dot products, keep each query token's maximum, and
sum.

```python
import torch.nn as nn

class ColPaliHead(nn.Module):
    """The retrieval head on top of PaliGemma (in training, LoRA adapts Gemma and proj)."""
    def __init__(self, hidden=2048, dim=128):
        super().__init__()
        self.proj = nn.Linear(hidden, dim, bias=False)

    def forward(self, hidden_states, mask):
        v = nn.functional.normalize(self.proj(hidden_states), dim=-1)
        return v * mask.unsqueeze(-1)

def colpali_score(q_vecs, page_vecs):
    """q: (nq, 128)  page: (n_patches, 128)  → scalar"""
    return (q_vecs @ page_vecs.T).max(dim=1).values.sum()
```

`ColPaliHead.forward` is Steps 2 to 4. `colpali_score` is Step 5. The scoring function is the same
computation as Chapter 35's `maxsim` (Chapter 36 explains it). **ColPali is ColBERT where the
document tokens happen to be image patches.** That is the whole conceptual leap, and it is why Part V
had to come before Part VI.

**Note:** Step 4 zeroes padding, and Chapter 36 warned that a zeroed row can still win a `max`. It is
safe here, and it is what colpali-engine does. Every ColPali page image has the same length, so page
vectors carry no padding. A zeroed query row scores 0 against every patch, so it adds exactly 0 to
the sum. Documents of different lengths batched together, such as text passages or the
dynamic-resolution pages of Chapter 47, need Chapter 36's −∞ mask on the document side.

---

### What people get wrong

**"The model reads the text like OCR."** It does not transcribe. It builds vectors in which text,
visual and spatial evidence are mixed together. That is why a stamp or a handwritten amendment still
shapes the page's vectors, instead of vanishing as it does in an OCR pipeline.

**"More patches is strictly better."** Patch count grows with pixel count, and everything downstream
grows with patch count: storage, scoring cost, latency. Doubling the resolution in both directions
quadruples the bill for a modest quality gain on most pages.

**"It works on any image."** It was trained on *document* pages. For natural photographs, use CLIP or
SigLIP. For documents, use this.

**"I can't fine-tune it."** You can, cheaply: only LoRA adapters train, on Gemma and the
projection. Generate synthetic queries from your own pages and adapt it in hours.

**"The 128 dimensions are arbitrary."** They are inherited from ColBERT, and they are a carefully chosen
point on the quality-versus-storage curve. Going lower is being actively explored (Chapter 47).

**"The training data was all synthetic."** About a third was. The rest came from open visual QA
datasets. The mix matters when you judge how well it transfers to your pages.

---

### Ninja notes

**ColQwen2 and its Qwen2.5-based successors are usually the better choice today**, and the reason is
architectural rather than incremental. ColQwen2 replaces PaliGemma with Qwen2-VL, which supports
**dynamic resolution**. Pages keep roughly their native aspect ratio. The number of patches depends on
the rendered image's size, up to a cap, instead of being fixed at 1,024.

Two consequences. A dense A4 page rendered at a good resolution gets more of its detail kept, without
being squashed into a square. A slide rendered smaller gets fewer patches and costs less to store. Note
that the model counts pixels, not ink: a blank page rendered large costs as much as a dense one. The
fixed 448 × 448 square of PaliGemma both distorts A4's shape and spends the same budget on every page.
Chapter 47 puts numbers on the cap.

**The heatmaps are also a quality signal you can automate.** If every query token's best patch score is
low, the model found no evidence anywhere. That is a genuine "nothing relevant here" signal, the kind
Chapter 55 has to build by hand with a calibrated relevance floor. Multi-vector retrieval gives some calibration for free,
because "no query token found a good match" is meaningfully different from "the overall similarity is
moderate".

---

### Key takeaways

- **PaliGemma = SigLIP (eyes) + projection + Gemma (reader).** ColPali adds a 128-d projection head and
  late-interaction training.
- A page becomes a 32 × 32 grid of patches. Each patch is a token with a position, covering a few
  characters across one or two lines.
- Gemma contextualises every patch against every other, rebuilding cell-to-header links in tables from
  layout alone.
- A learned linear layer projects each patch to 128 normalized dimensions: ~527 KB per page before
  compression.
- Scoring is exactly ColBERT's MaxSim. ColPali is ColBERT over image patches.
- Training used about 119,000 (query, page) pairs (118,695): roughly a third synthetic, generated by a
  VLM, and two-thirds from open VQA sets (DocVQA, InfoVQA, TAT-DQA, arXivQA). The synthetic recipe is
  reusable for your own corpus.
- The loss pushes each true page above the highest-scoring wrong page in its batch. There is no
  softmax over all pages and no temperature.
- **LoRA = small add-on weights trained beside a frozen model.** Only LoRA adapters train, on Gemma and
  the projection, so adaptation is cheap.
- Per-token heatmaps give free interpretability and a usable "nothing found" signal.

### What's next

[Chapter 47](./47-colpali-3-production.md) faces the bill: 527 KB per page, a million pages, and what to
do about it, including ColQwen2's dynamic resolution.

Now we understand what sits inside ColPali, how a scanned page becomes about a thousand small vectors,
how the model was trained, and how to see what it matched.
