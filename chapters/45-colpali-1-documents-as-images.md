---
title: "ColPali I: Documents as Images"
chapter: 45
part: "Part VI — Vectors Beyond Text"
slug: colpali-1-documents-as-images
readingTime: "12 min"
summary: "What is ColPali? Delete the entire parsing pipeline. Take a screenshot of the page, embed the pixels with a vision-language model, and search that. It works better than the pipeline it replaces."
tags: [colpali, document-retrieval, vision, vidore, late-interaction]
prev: 44-ocr-broken-promise
next: 46-colpali-2-inside
---

# ColPali I: Documents as Images

**The one-paragraph version.** **ColPali** indexes a page by taking a picture of it. The page
image goes through a vision-language model, which produces about a thousand small vectors, one
per region of the page. We store all of them, and search them with ColBERT's MaxSim. No OCR, no
layout detection, no table extraction, no reading-order rules. On visually rich documents it
clearly beats the pipeline it replaces, and it is simpler and faster to index.

In this chapter, we will learn what ColPali is, why searching a picture of a page can beat
searching its text, and how the whole idea fits in two stages. We will send the scanned Globex
contract from Chapter 44 through ColPali and watch it get the right answer. We will also see its
limits, and when the old pipeline is still the better choice.

We will cover the following:

- What is ColPali
- Why we need it
- How ColPali works
- An example: page 3,507 again
- Why it beats the pipeline
- The evidence
- Problems with ColPali
- ColPali vs the OCR pipeline, and when to use which one

---

## What is ColPali

**ColPali = Contextualized Late Interaction over PaliGemma.**

- **Col** comes from ColBERT (Chapter 35). It means **late interaction**: keep many small vectors
  per document and compare them with the query's vectors only at search time.
- **Pali** comes from **PaliGemma**, Google's open vision-language model (Chapter 43). It is the
  model that turns the page picture into vectors. Chapter 46 opens it up.

In simple words, ColPali is ColBERT where the document's "words" are small squares of a page image.

A quick recap of the two ideas it borrows. A **patch** (Chapter 42) is one small square of an image,
treated like a token. **MaxSim** (Chapters 35 and 36) scores a document like this: for each query
token, find its best-matching document vector, then add up those best scores.

---

## Why we need it

When a person looks for a fact in a contract, they do not run OCR in their head. They *look at the
page*. They see a table, see that it is about pricing, and find the row they want. Layout,
typography and position are not obstacles to understanding. They carry meaning.

Chapter 44 described a pipeline that begins by destroying exactly that information. Let's recall
what happened. A support agent asked:

> *"Does the Globex contract include SSO?"*

Page 3,507 has the answer in its pricing table: `SSO: $0, included (Pro, 50+ seats)`, and Globex has
120 Pro seats. The OCR pipeline read the table column by column and split it across two chunks. The
Scholar never saw the word *included*, and answered "No, Globex needs the Security add-on."

The answer was wrong, and no stage of the pipeline noticed.

So why convert the page to text at all?

```
Traditional:  PDF → OCR → layout → tables → order → chunk → embed → search
ColPali:      PDF → page image → embed → search
```

Two stages instead of six. Nothing is *converted*. Something is still lost: the page is resized to a
fixed 448 × 448 grid of pixels, and that resize is itself lossy. Chapters 46 and 47 return to exactly
this.

The idea looks obvious now. It became possible only when vision-language models got good enough to
actually *read* a page. As Chapter 43 traced, that needed SigLIP-class encoders at high enough
resolution, joined to language models trained on documents.

---

## How ColPali works

Here is the whole architecture in outline:

```
page image (448×448)
    ↓ SigLIP vision encoder
1,024 patch embeddings            (32 × 32 grid of image patches)
  + ~6 prompt tokens              (a short fixed text instruction)
    ↓ Gemma-2B language model
~1,030 contextualised vectors     (each patch now aware of the whole page)
    ↓ linear projection to 128-d
~1,030 vectors × 128 dims         ← stored per page

query text ("Does the Globex contract include SSO?")
    ↓ same model, text path
~20 vectors × 128 dims

score = MaxSim(query vectors, page vectors)          ← ColBERT, Chapter 36
```

**Phase 1: Indexing a page.**

**Step 1:** Render the PDF page as an image.

**Step 2:** Resize it to 448 × 448 pixels and cut it into a 32 × 32 grid of patches, 1,024 in all.

**Step 3:** The SigLIP encoder turns each patch into a vector.

**Step 4:** The Gemma language model reads all the patch vectors together, along with a short text
prompt (a fixed instruction the model always reads alongside the image). Each patch's vector now
reflects the whole page around it.

**Step 5:** A learned layer shrinks each vector to 128 numbers. With the few extra vectors from the
prompt, that is about 1,030 vectors.

**Step 6:** Store all ~1,030 vectors for the page.

**Phase 2: Answering a query.**

**Step 1:** Run the query text through the same model. We get about 20 vectors of 128 numbers, roughly
one per query token.

**Step 2:** For each query vector, find its best-matching patch vector on a page.

**Step 3:** Add up those best matches. That sum is the page's score.

**Step 4:** Return the pages with the highest scores.

The crucial choice, and the reason this sits in Part V's family tree: **patch vectors are not
pooled.** The page is not squeezed into one vector. Each patch keeps its own vector, so a query token
can match the exact region of the page that answers it.

That means the pricing table in the middle of page 3,507 can be matched on its own. This is
Chapter 35's late interaction, applied to pixels. It is why ColPali handles tables, the thing OCR
pipelines fail at most badly.

---

## An example: page 3,507 again

Let's send the same question through ColPali.

> *"Does the Globex contract include SSO?"*

The query becomes about 20 vectors. Let's follow four of them, one query token at a time.

**"Globex"** finds its best match in the patches of the page heading, *Schedule B: Pricing for Globex
Corporation*.

**"contract"** finds its best match in the contract's running header at the top of the page.

**"SSO"** finds its best match in the patch holding "SSO" at the start of the table's last row.

**"include"** finds its best match among the patches holding "included (Pro, 50+ seats)". Those
patches sit in the same row as "SSO". During Step 4 of indexing, Gemma let them look at their
neighbours, so their vectors already reflect "the SSO row" and not just the word "included".

Every important query token found strong evidence on one page. Page 3,507 gets the top score.

Now the page image, not a text chunk, goes to a Scholar that can read images. It reads the table the
way a person does. It thinks like this: *"The last row is SSO. Its price is $0 and its terms say
included for Pro at 50 or more seats. The first row says Globex has 120 Pro seats. 120 is more than
50."*

It answers: *"Yes. SSO is included in the Globex contract at no charge, because Globex has 120 Pro
seats."*

The answer is correct. The table was never flattened, so `SSO`, `$0`, `included` and `120 seats` never
lost their places.

**Note:** The patch-by-patch story above is a simplification of what the vectors do, not a printout of
the model's internals. But it is not a fairy tale either. Chapter 46 shows how to draw a heatmap of
which patches each query token matched, and on real pages it looks very much like this.

---

## Why it beats the pipeline

**Nothing is lost to text conversion.** Charts, stamps, checkboxes, signatures, highlighting, logos,
handwriting and struck-through text stay in the pixels, subject only to the 448 × 448 resize (see
Problems with ColPali below). The model was trained on documents full of them.

**Layout is signal, not noise.** A patch in the header is contextualised differently from a patch in a
footnote. A label and its value stay linked, because the patches keep their places in the grid.

**Tables work.** The cell "included" sits in the same row as "SSO", and the transformer's attention can
relate them. This is the single largest practical advantage.

**No error compounding.** One model, one forward pass, one place to measure quality.

**Indexing is much simpler.** No OCR engine, no layout model, no table parser, no per-document-type
tuning, no PDF library quirks. Render to an image and encode. The ColPali paper reports about 0.4
seconds per page to encode on a GPU, against several seconds per page for the OCR-based pipeline it
compared with. A 3-billion-parameter model is slow, but OCR plus layout plus table extraction is
slower.

**It is interpretable.** Because it is late interaction, we can see *which patches matched which query
token*: a heatmap over the page showing where the model found its evidence. For document search this
is remarkably useful. We can hand a user the page with the relevant row highlighted.

---

## The evidence

ColPali was introduced alongside **ViDoRe**.

**ViDoRe = Visual Document Retrieval Benchmark.** It was built because no earlier benchmark measured
this properly. It spans academic figures, infographics, financial reports, government documents,
healthcare and industrial PDFs. It is deliberately weighted toward visually complex material.

The published pattern is consistent:

- On **text-heavy, clean pages**, OCR pipelines and ColPali are comparable.
- On **visually rich pages** (charts, tables, infographics, forms), ColPali wins, by a wide nDCG margin
  (Chapter 19) on several subsets.
- **ColQwen2**, a successor built on the Qwen2-VL model with dynamic resolution, improves further.
  Chapter 47 covers it.

Let's read that honestly. This is not a universal replacement for text retrieval. It is a decisive
improvement on one specific, common and badly served class of documents. If your corpus is plain
prose, the OCR pipeline is fine and cheaper. If your corpus is reports, slides, forms, scans or
scientific papers, this is the architecture.

---

## Problems with ColPali

**Problem 1: The resize loses detail.** Every page is squeezed to 448 × 448 pixels. An A4 page becomes
about 54 dots per inch (DPI) across and 38 down, and its shape is squashed. Normal body text usually
survives. The fine print in a contract footnote may not. Chapter 47 shows how dynamic-resolution models ease this.

**Problem 2: Storage.** About 1,030 vectors of 128 numbers per page is roughly 527 KB in float32. One
768-d text vector is 3 KB. Chapter 47 does the arithmetic and brings it down.

**Problem 3: Exact codes.** There is no inverted index. A screenshot showing error `SSO-4012` and one
showing `SSO-4021` can look almost identical to the model. Keyword search still wins for exact codes.

**Problem 4: Every query runs a big model.** Each query goes through a 3-billion-parameter model to
make its vectors. That costs GPU time per query, not just per page.

**Problem 5: The page is the unit.** A table that runs across two pages is split, and neither page
alone answers the question. Chapter 47 gives the standard fix.

---

## ColPali vs the OCR pipeline, and when to use which one

| | OCR pipeline (Chapter 44) | ColPali |
|---|---|---|
| Stages before search | Six | Two |
| What is stored per page | A few text chunks, one vector each | ~1,030 vectors × 128-d |
| Tables and forms | Often scrambled | Kept in place |
| Charts, stamps, checkboxes | Lost | In the pixels |
| Fine print | Readable if OCR gets it | May be lost in the 448 × 448 resize |
| Exact codes like `SSO-4012` | Strong, with BM25 | Weak |
| Indexing | Several components, slow per page | One model, faster per page |
| Storage per page | Small | Large until compressed (Chapter 47) |
| Interpretability | Hard to trace failures | Heatmap per query token |

**Advantages of ColPali.** Tables, charts and visual marks survive. One model replaces a chain of
fragile stages. Results can be explained with a heatmap.

**Disadvantages of ColPali.** Lossy resize, large storage, weak exact-token matching, and a big model
on every query.

We must use **ColPali** when the documents are visually rich (reports, slides, forms, scans,
scientific papers) and the answers live in tables, figures and layout.

We must use **the OCR pipeline** (or plain text embeddings) when the documents are clean prose, when we
need exact codes, or when we need the extracted text itself for other jobs.

Many strong systems use both: ColPali for meaning, OCR text in a BM25 index for exact codes, fused with
RRF. Chapter 47 shows that design.

---

### Under the hood

```python
import torch
from colpali_engine.models import ColPali, ColPaliProcessor
from pdf2image import convert_from_path

model = ColPali.from_pretrained("vidore/colpali-v1.3", torch_dtype=torch.bfloat16,
                                device_map="cuda").eval()
proc  = ColPaliProcessor.from_pretrained("vidore/colpali-v1.3")

# Index: a page is just an image
pages = convert_from_path("globex_contract.pdf", dpi=150)
with torch.no_grad():
    batch = proc.process_images(pages[:8]).to(model.device)
    page_vecs = model(**batch)          # (8, ~1030, 128)

# Query
with torch.no_grad():
    qbatch = proc.process_queries(["Does the Globex contract include SSO?"]).to(model.device)
    q_vecs = model(**qbatch)            # (1, ~20, 128)

scores = proc.score_multi_vector(q_vecs, page_vecs)   # MaxSim
best_page = pages[int(scores.argmax())]
```

`convert_from_path` is Phase 1, Step 1. `proc.process_images` does the resize in Step 2.
`model(**batch)` cuts the patches and runs Steps 3 to 5. `score_multi_vector` is Phase 2, Steps 2
and 3. In simple words, the entire indexing pipeline is "render, then one forward
pass". Compare it to the six-stage diagram in Chapter 44, and the operational argument makes itself.

**Note:** Look at `dpi=150`. For ColPali, the processor squeezes every page to 448 × 448 anyway. For an
A4 page that is roughly 40–55 DPI, so any detail rendered above about 60 DPI is thrown away. 150 DPI
does not help the model read. It is still a sensible default, because the same image is what we store,
show to users and hand to the Scholar. DPI does matter for dynamic-resolution models, which Chapter 47
covers.

---

### What people get wrong

**"This replaces all text retrieval."** It replaces the *document parsing* pipeline for visually rich
documents. For a corpus of markdown files or clean articles, a text embedding model is cheaper and just
as good.

**"It's too expensive."** Indexing is comparable to or cheaper than a full OCR pipeline. Query time
costs more, because every query runs the model and scores many vectors. The storage is the real cost,
and it is manageable. Chapter 47 does the arithmetic.

**"I still need OCR for the answer."** Often not. You can pass the page *image* to a vision-capable LLM
to write the answer, keeping the whole pipeline in pixels. If you do need text, run OCR on the handful
of retrieved pages rather than on the whole corpus. That inverts the cost structure entirely: OCR on 5
pages per query instead of all 40,000 pages of the Library (or millions, at scale) up front.

**"One vector per page would be simpler."** It would, and it would bring back Chapter 34's bottleneck in
its most extreme form. A page holds far more distinct information than a paragraph. Squeezing it into
128 numbers loses almost everything.

---

### Ninja notes

**The deeper shift here is worth naming.** For thirty years, document retrieval meant *converting
documents into text and searching the text*. That assumption was so basic it was rarely stated. ColPali
discards it.

The consequence reaches past documents. Anything with meaningful visual structure can now be searched
by what it *looks like* rather than by a lossy text stand-in: dashboards, screenshots, UI recordings,
engineering diagrams, maps, medical scans, handwritten notes, whiteboard photos. Screenshot search over
a user's own history, retrieval over CAD drawings and search across slide decks all become
straightforward uses of the same architecture.

**Practically, this also changes where you spend engineering effort.** The old pipeline put most of the
work, and most of the failures, in extraction. The new one puts most of the work in storage and serving.
Those are problems this book has already solved, in Chapters 24, 26, 27, 37 and 38. Trading an
ill-defined problem (parse arbitrary documents correctly) for a well-defined one (store and search many
vectors efficiently) is a very good trade.

---

### Key takeaways

- **ColPali = Contextualized Late Interaction over PaliGemma.** Render the page, encode it with a
  vision-language model, store ~1,030 patch vectors, search with MaxSim.
- Two stages instead of six. It deletes OCR, layout detection, table extraction, reading order and
  chunking.
- Nothing is converted, but the fixed 448 × 448 resize still loses detail.
- Patch vectors are not pooled, so query tokens match specific regions of the page. That is why tables
  and figures finally work, and why page 3,507 now gets the right answer.
- On visually rich documents it clearly beats OCR pipelines. On plain prose it is comparable and more
  expensive.
- It is interpretable: we can see which regions matched which query token.
- We can keep the whole pipeline in pixels by passing retrieved page images to a vision-capable LLM.

### What's next

[Chapter 46](./46-colpali-2-inside.md) opens the model: how patches become vectors, how it was trained,
and why the projection to 128 dimensions is doing so much work.

Now we understand ColPali, why a picture of a page can be searched better than its text, how it works
in two stages, and where it still falls short.
