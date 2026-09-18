---
title: "ColPali III: Production Reality"
chapter: 47
part: "Part VI — Vectors Beyond Text"
slug: colpali-3-production
readingTime: "14 min"
summary: "527 KB per page times a million pages is 527 GB. Here is the full cost model, the four compression levers, and an honest account of when not to use this."
tags: [colpali, colqwen, production, storage, pooling, cost, muvera]
prev: 46-colpali-2-inside
next: 48-other-modalities
---

# ColPali III: Production Reality

**The one-paragraph version.** A ColPali page is about 1,030 vectors of 128 dimensions, roughly
527 KB in float32. A million pages is 527 GB, which is not a prototype problem. Four levers fix
it: **token pooling** (fewer vectors per page), **fewer dimensions**, **quantization** (fewer
bits per number), and **MUVERA** for the first retrieval stage. ColQwen2 changes the picture
again: it keeps each page's shape and spends a capped, variable number of vectors per page.
Together, these bring a million pages down to tens of gigabytes, at which point this is an
ordinary system.

In this chapter, we will learn what ColPali really costs to store, index and query, and the four
levers that shrink the bill. We will meet ColQwen2 and its dynamic resolution, and see when
rendering resolution matters and when it is thrown away. We will also see when *not* to use any
of this, and the hybrid design most production systems end up with.

We will cover the following:

- What ColPali costs
- Lever 1: token pooling (do this first)
- Lever 2: fewer dimensions
- Lever 3: quantization
- Lever 4: MUVERA for the retrieval stage
- ColQwen2: dynamic resolution and the token cap
- The full cost model
- When not to use it, and the hybrid design

---

## What ColPali costs

Acme's own Library is 40,000 pages. At 527 KB a page, ColPali needs about 21 GB for it. That is fine.

But Acme's product team wants to offer the same search to customers, who will upload their own
scanned contracts, reports and screenshots. The plan is one million pages in the first year. Here is
the bill, before any tricks:

```
per page:    1,030 vectors × 128 dims × 4 bytes  =  527 KB
1,000 pages:                                     =  527 MB
40,000 pages (Acme's Library):                   =   21 GB
100,000 pages:                                   =   53 GB
1,000,000 pages:                                 =  527 GB
10,000,000 pages:                                =  5.3 TB
```

For comparison, one 768-d float32 vector per page is 3 KB. ColPali stores **about 170× more**.

Scoring grows the same way. MaxSim between a 20-token query and a 1,030-patch page is 20,600 dot
products *per page*. Over a million pages that is about 20 billion dot products for a single query.

In simple words, both the storage and the scoring are tractable, but neither is tractable naively.
Let's pull the levers one at a time.

---

## Lever 1: token pooling (do this first)

Look at page 3,507, the Globex contract. Most of it is white margin, blank space between clauses and
repeated table lines. Many of its 1,024 patches are nearly identical, and carry almost no information.

**Token pooling = merging a page's near-duplicate patch vectors into their average.**

The usual tool is **hierarchical clustering**. It starts with every patch vector as its own group,
then repeatedly merges the two most similar groups, until only the number of groups we asked for is
left.

**Step 1:** Take the page's patch vectors.

**Step 2:** Decide how many to keep, for example a third of them.

**Step 3:** Cluster them hierarchically until that many groups remain.

**Step 4:** Average each group into one vector.

**Step 5:** Normalize each average back to length 1, and store those instead.

On page 3,507, hundreds of blank-margin patches collapse into a few vectors. The patches of the pricing
table stay distinct, because they are not alike.

Published results on this are encouraging. Roughly **3× fewer vectors cost a small amount of quality**,
and 2× pooling is close to free. Blank regions dominate many real pages, so this is the
highest-return-per-effort optimisation available.

```
1,030 vectors → ~340 vectors        527 KB → 174 KB
```

---

## Lever 2: fewer dimensions

128 dimensions per patch is inherited from ColBERT's text setting. Patch vectors may not need that many,
because each describes a small, fairly constrained region.

Training or distilling (Chapter 36) a 64-dimensional projection head halves storage. Some work explores
32. The quality cost is often modest, and the saving is linear. Measure it on your own pages.

```
340 vectors × 64 dims × 4 bytes = 87 KB per page
```

---

## Lever 3: quantization

Chapter 26 applies directly and unchanged.

- **int8**: 4× smaller, typically around 1% quality loss. Close to free.
- **Binary (1 bit per number)**: 32× smaller. With a rescoring pass, the loss is small.
- **2-bit residual (ColBERTv2 style, Chapter 37)**: the standard multi-vector choice. At 128-d it is
  32 bytes of residual plus a 4-byte centroid ID, 36 bytes per vector.

Stack binary on top of Levers 1 and 2:

```
340 vectors × 64 dims × 1 bit = 2.7 KB per page
```

From 527 KB to 2.7 KB. **A million pages is now 2.7 GB**, which fits in a laptop's RAM.

Realistically, we would not stack every lever at full strength without measuring. We would also keep
higher-precision vectors somewhere for rescoring. But the order of magnitude is the point: **the storage
objection to ColPali is an engineering problem with known solutions, not a fundamental barrier.**

---

## Lever 4: MUVERA for the retrieval stage

Compression fixes storage. It does not fix the 20 billion dot products per query.

**MUVERA (Chapter 38) does.** It turns each page's bag of patch vectors into one Fixed Dimensional
Encoding (FDE), a single long vector whose dot product approximates MaxSim.

**Phase 1: Indexing.**

**Step 1:** Build one FDE per page, about 4,000 dimensions.

**Step 2:** Compress the FDEs with PQ (Chapter 24) to about 1 KB each.

**Step 3:** Index them with ordinary HNSW or IVF-PQ.

**Phase 2: Answering a query.**

**Step 1:** Build the query's FDE and run an ANN search. Keep the top 200 pages.

**Step 2:** Run exact MaxSim on only those 200 pages.

**Step 3:** Keep the top 5 pages.

**Step 4:** Hand the page images to a Scholar that can read images.

```
1M pages
   ↓ FDE index (one ~4,000-d vector per page, PQ-compressed: ~1 KB/page = 1 GB)
   ↓ ANN search                                    ~5 ms
top 200 pages
   ↓ exact MaxSim on 200 pages × 340 vectors       ~20 ms
top 5 pages
   ↓ vision LLM reads the page images              ~2 s
answer
```

In simple words, a cheap single-vector search finds 200 candidates, and the expensive multi-vector score
runs only on those. Exact MaxSim now needs 200 × 340 × 20 = 1.36 million dot products instead of 20
billion.

This is the coarse-then-refine cascade one final time. It is the recommended architecture for
image-document RAG at scale, and Chapter 61 assembles it end to end.

---

## ColQwen2: dynamic resolution and the token cap

Everything so far assumed the original ColPali, which squeezes every page into 448 × 448 pixels. What
if the page did not have to be squeezed?

**ColQwen2 = ColPali's recipe (late interaction, 128-d vectors, MaxSim) on top of Qwen2-VL.** Qwen2-VL
is the vision-language model from Alibaba we met in Chapter 43. The widely used ColQwen2 release is
built on its 2-billion-parameter version.

**Dynamic resolution = the model takes an image at roughly its own size and shape, and the number of
visual tokens grows with the number of pixels.**

Here is how the reference implementation counts tokens, as we understand it:

**Step 1:** Render the page at some DPI.

**Step 2:** Round each side to a multiple of 28 pixels, keeping the page's shape. (Step 4 shows why 28.)

**Step 3:** If that would give more tokens than the cap, shrink the image, still keeping its shape, until
it fits.

**Step 4:** Cut it into 14-pixel patches, then merge each 2 × 2 block of patches into one visual token.
So each token covers 28 × 28 pixels.

**Step 5:** Each visual token becomes one 128-d vector, plus a few vectors for the text prompt (the
fixed instruction text the model reads alongside every page image).

The cap is roughly **768 visual tokens by default** in the reference implementation, and it can be
changed. (The setting is expressed as a maximum pixel count or token count, and its name varies between
library versions.)

Here is what that means for real pages, using our own reimplementation of the resize rule:

| Page, rendered at | Rendered pixels | Visual tokens |
|---|---|---|
| A4, 60 DPI | 496 × 701 | 450 |
| A4, 72 DPI | 595 × 842 | 630 |
| A4, 100, 150 or 200 DPI | resized to 644 × 896 | 736 (capped) |
| Slide (13.33 × 7.5 in), 60 DPI | 800 × 450 | 464 |
| Slide, 72 DPI | 960 × 540 | 646 |
| Slide, 100 DPI or more | resized to 1,008 × 560 | 720 (capped) |

Three things stand out.

**First, pages hit the cap early.** An A4 page reaches it at about 80 DPI. After that, a denser render
changes nothing, because the resize throws the extra pixels away, just as ColPali's resize does.

**Second, capped ColQwen2 still sees much more.** At the cap, an A4 page keeps 644 × 896 pixels, about
77 DPI in both directions, with its shape kept. ColPali keeps 448 × 448, about 54 DPI across and 38 down,
squashed. That is 2.9× the pixels, carried by *fewer* vectors, because each ColQwen2 token covers a
28-pixel square instead of a 14-pixel one.

**Third, the model counts pixels, not ink.** A dense filing and a blank page rendered at the same size
cost the same. In practice, dense pages are the ones we render large enough to hit the cap. Slides, with
big text, are the ones we can render small, so they use fewer tokens.

**The storage line.** A capped A4 page is about 750 vectors (736 visual tokens plus the prompt):

```
750 vectors × 128 dims × 2 bytes (float16)     ≈ 190 KB per page
750 vectors × 36 bytes (2-bit residual + ID)   ≈  27 KB per page
1M pages: ≈ 190 GB in float16, ≈ 27 GB compressed
```

Token pooling works on ColQwen2 exactly as on ColPali. Pooled 3× to 250 vectors at 36 bytes each, a page
is 9 KB, and a million pages is about 9 GB.

**Where DPI matters.**

**Note:** For the original ColPali (PaliGemma, fixed 448 × 448 input), rendering DPI does not matter
beyond about 60. The resize throws the extra pixels away. DPI only matters for dynamic-resolution models
like ColQwen2.

For ColQwen2, DPI matters until the page hits the cap. Beyond that, the real knob is the cap itself.
Raise the cap to about 1,536 tokens and a 200-DPI A4 render keeps about 108 DPI, at 1,472 tokens, which
is roughly 377 KB per page in float16. That is twice the storage for twice the pixels.

So we sweep **DPI and the token cap together** against the eval set, exactly as we would sweep
`efSearch`. On corpora with small fonts, such as financial footnotes and legal documents, try a higher cap
with a render large enough to fill it. On slides, try a lower DPI, which uses fewer tokens and less
storage. Let the golden set (Chapter 54) decide.

---

## The full cost model

For **one million pages**, here is a realistic ColPali configuration: pooled to ~340 vectors, 128-d,
2-bit residual compression, plus an FDE index. A **GPU-hour** is one GPU running for one hour.

| Component | Cost |
|---|---|
| One-time encoding (GPU) | ~1M pages ÷ 2–10 pages/s ≈ 30–140 GPU-hours, depending on GPU and batching |
| Multi-vector storage | 340 × 36 B ≈ 12 KB/page ≈ 12 GB |
| FDE index | ~1 GB |
| Page images (for the Scholar to read) | object storage, ~100 GB at 150 DPI JPEG |
| Query latency (retrieval only) | 25–80 ms |
| Query GPU cost | one encoder pass per query |

The ColPali paper reports about 0.39 seconds per page to encode, which is about 110 GPU-hours per million
pages. Good batching on a fast GPU does better. With ColQwen2 at the default cap, pooled 3×, storage drops
to about 9 GB.

Compare an OCR pipeline over the same corpus. OCR plus layout plus table extraction is typically **slower
per page** than one VLM forward pass. It needs several components. And, per Chapter 44, it loses much of
the information on the pages that matter.

**The honest summary: ColPali costs more storage and gives better quality on visual pages, and its indexing
cost is comparable or lower.** The decision is about quality requirements, not about affordability.

---

## When not to use it, and the hybrid design

This is the section most write-ups leave out, and it matters.

**Your documents are plain prose.** Markdown files, articles, clean text with no tables or figures. A text
embedding model is cheaper, faster and just as good.

**You need exact-term search.** ColPali has no inverted index. Let's see what that does.

A support agent searches the customer screenshots for the error code *"SSO-4012"* (SAML assertion
expired). Let's first see what ColPali alone does. It ranks a screenshot of error `SSO-4021` first. The two
error dialogs have the same layout, the same colours and nearly the same characters, so their patch vectors
are almost the same.

The answer is wrong. The agent reads the fix for a different error.

Now, let's see what the hybrid design does. OCR text from every screenshot also sits in a BM25 index
(Chapter 8). BM25 matches the exact characters `SSO-4012`, and only one screenshot contains them. RRF
(Chapter 40) merges the two ranked lists. The true `SSO-4012` dialog ranks first in BM25 and second in
ColPali, so it ranks high in *both* lists and comes out on top. The `SSO-4021` dialog is first in
ColPali, but in BM25 it is only one of many screenshots that share the letters "SSO", so it sinks.

The answer is correct.

**Your latency budget is under about 20 ms.** Multi-vector retrieval plus a VLM encoder pass for the query
will not fit.

**You have no GPU and no budget for one.** Encoding a million pages on CPU is not practical.

**You need the text itself downstream**, for redaction, compliance extraction or structured data
pipelines. Retrieval and extraction are different jobs. ColPali does the first and not the second.

**The best real architecture is usually hybrid**, and it is worth stating plainly:

```
Index:  page images → ColPali vectors        (visual/semantic retrieval)
        page images → OCR text → BM25        (exact terms, identifiers)

Query:  ColPali search ─┐
                        ├─ RRF fusion (Ch. 40) → top pages
        BM25 search ────┘
                        → vision LLM reads the retrieved page images → answer
```

We pay for OCR *once*, at index time, only to feed a keyword index. We never rely on it for meaning, so its
errors on tables and figures do not matter. Each component does what it is good at.

| | OCR + text embeddings | ColPali | ColQwen2 |
|---|---|---|---|
| Input size | Any, via text | Fixed 448 × 448, squashed | Near native size and shape, capped |
| Vectors per A4 page | A few chunks | ~1,030 | ~750 at the default cap |
| Raw float16 per page | A few KB | ~264 KB | ~190 KB |
| Rendering DPI matters | For OCR accuracy | Not above ~60 | Up to the cap, then the cap matters |
| Tables, charts, stamps | Damaged or lost | Kept | Kept, with finer detail |
| Exact codes | Strong with BM25 | Weak | Weak |

**Advantages of ColQwen2 over ColPali.** More pixels, little distortion, fewer vectors per page at the default
cap, and a tunable budget.

**Disadvantages of ColQwen2.** The budget is one more setting to tune, token counts vary by page size so
capacity planning needs care, and it is still weak on exact codes.

We must use **ColQwen2** (or its newer successors) for new visually rich corpora, especially A4 documents
with small print.

We must use **the original ColPali** when an existing index and evaluation already depend on it, and the
golden set shows no gap.

We must use **OCR plus BM25** alongside either one, whenever users search for codes, IDs or exact names.

Many strong systems use all three pieces: a ColQwen2-style model for meaning, BM25 over OCR text for exact
terms, and RRF to fuse them.

---

### Under the hood

Token pooling is the first thing to implement. It follows the five steps of Lever 1:

```python
import numpy as np
from scipy.cluster.hierarchy import linkage, fcluster

def pool_page_vectors(V, target_ratio=3.0):
    """V: (n_patches, dim), L2-normalized. Returns fewer, pooled vectors."""
    n_clusters = max(1, int(len(V) / target_ratio))
    Z = linkage(V, method="average", metric="cosine")
    labels = fcluster(Z, t=n_clusters, criterion="maxclust")
    pooled = np.stack([V[labels == c].mean(0) for c in np.unique(labels)])
    return pooled / np.linalg.norm(pooled, axis=1, keepdims=True)
```

`linkage` does the repeated merging of Step 3. `fcluster` stops at the number of groups we asked for. The
last two lines are Steps 4 and 5. In simple words, similar patches are grouped, each group is averaged, and
the averages are stored.

Next, the resolution budget. This is a sketch of the ColQwen2-style resize rule from Steps 1 to 4 above,
for capacity planning. Check it against the library version you deploy:

```python
import math

def visual_tokens(width_px, height_px, cap=768, block=28):
    """Budgeting sketch of Qwen2-VL-style resizing: 14-px patches merged 2×2 → 28-px tokens."""
    w = max(block, round(width_px / block) * block)
    h = max(block, round(height_px / block) * block)
    if w * h > cap * block * block:                       # too big: shrink, keep the shape
        scale = math.sqrt((width_px * height_px) / (cap * block * block))
        w = math.floor(width_px / scale / block) * block
        h = math.floor(height_px / scale / block) * block
    return (w // block) * (h // block)

visual_tokens(1240, 1754)               # A4 at 150 DPI → 736 (capped)
visual_tokens(1654, 2338, cap=1536)     # A4 at 200 DPI, raised cap → 1472
```

And the rendering decision, which people set once and never revisit:

```python
from pdf2image import convert_from_path

# Rendering DPI: what actually reaches the model
#   ColPali (fixed 448×448): detail above ~60 DPI is resized away.
#       Render for your UI and the Scholar instead (150 DPI is a good default).
#   ColQwen2 (dynamic, ~768 visual tokens by default):
#       A4 hits the cap at ~80 DPI. To keep finer print, raise the cap AND the DPI,
#       and pay for it in vectors per page. Sweep both against your eval set.
pages = convert_from_path(path, dpi=150, fmt="jpeg")
```

---

### What people get wrong

**Not pooling.** 3× storage reduction for very little quality loss, left on the table.

**Rendering at the "right" DPI for the wrong model.** For fixed-448 ColPali, 300 DPI buys nothing over 60.
For ColQwen2, a high DPI buys nothing once the page hits the token cap. Too little resolution and the model
cannot read. Too much and you burn money. Measure DPI and the cap together.

**"ColQwen2 spends fewer tokens on sparse pages."** It spends fewer tokens on *smaller images*. It counts
pixels, not ink.

**Serving float32.** There is no reason to keep float32 in the hot index. Quantize it. Keep the float32
copy in object storage as the source of truth for rebuilds and index migrations. If query-time rescoring
needs higher-precision vectors, serve them from SSD or RAM, not from object storage (Chapter 60).

**Running MaxSim over the whole corpus.** Use FDEs or a two-stage retrieval. Exhaustive MaxSim does not scale
past a few hundred thousand pages.

**Discarding the page images.** You need them to show the user and to feed a vision LLM. Keep them in object
storage.

**Not measuring against an OCR baseline.** On *your* corpus, the gap might be 30 points or 3. That number
decides whether the complexity is justified, and you can measure it in a day with Chapter 54's golden set.

---

### Ninja notes

**Page-level retrieval changes what your chunks are.** In text RAG, you agonise over chunk boundaries
(Chapter 50). With ColPali, the unit is a page: a natural, human-designed boundary that authors already used
to organise information. Many chunking problems simply disappear.

But new ones appear. A table spanning two pages is split, and neither page alone answers the question. The
standard fix is **neighbour expansion**: when page $n$ is retrieved, also pass pages $n-1$ and $n+1$ to the
LLM. Cheap, and it fixes the most common failure. For documents with heavy cross-page structure, also
consider indexing two-page spreads as single images. With ColQwen2, remember that a spread is a wider image
under the same token cap, so each page in it gets fewer tokens.

**Query-side cost is real and often overlooked.** Every query needs a forward pass through a 2–3B parameter
model to produce query vectors. That is 10–50 ms on a GPU and much worse on CPU, and it is paid per query,
not per document. At high query volume, this becomes the dominant serving cost, not storage. Cache
aggressively for repeated queries, and consider whether a smaller query encoder can be distilled.

---

### Key takeaways

- **Raw ColPali = ~1,030 vectors × 128-d × 4 B ≈ 527 KB per page**, 527 GB per million pages, and 20,600 dot
  products per page per query.
- Four levers: **token pooling** (~3×), fewer dimensions (~2×), quantization (up to 32×), and MUVERA FDEs
  for the retrieval stage.
- **ColQwen2 = ColPali's recipe on Qwen2-VL, with dynamic resolution.** Pages keep their shape. Tokens grow
  with pixels, up to a cap of roughly 768 by default. A capped A4 page is ~750 vectors: ≈190 KB in
  float16, or ≈27 KB at 36 B/vector.
- Fixed-448 ColPali discards detail above ~60 DPI. For ColQwen2, DPI matters until the cap, then the cap
  matters. Sweep both.
- One million pages: 30–140 GPU-hours to encode, ~12 GB of pooled, compressed vectors, ~1 GB of FDEs,
  and tens of milliseconds per query for retrieval.
- Indexing costs about the same as, or less than, a full OCR pipeline. Storage is higher.
- Do not use it alone for plain prose, exact codes, sub-20 ms budgets, or when you need the text itself.
  Without a GPU, do not use it at all.
- The best production design is hybrid: a ColPali-family model for meaning, OCR + BM25 for exact terms, fused
  with RRF. Retrieve pages, expand to neighbours, and let a vision LLM read the images.

### What's next

[Chapter 48](./48-other-modalities.md) closes Part VI with a tour of everything else that has been mapped into
vector space: audio, video, code, graphs, and users.

Now we understand what page-image retrieval really costs, the levers that make it affordable, how ColQwen2's
token cap changes the budget, and when a simpler design is the better one.
