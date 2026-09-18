---
title: "The Broken Promise of OCR Pipelines"
chapter: 44
part: "Part VI — Vectors Beyond Text"
slug: ocr-broken-promise
readingTime: "13 min"
summary: "OCR, layout, tables, reading order, chunking, embedding. Six stages, six places to lose information, and errors that compound silently. Here is exactly what you lose."
tags: [ocr, document-parsing, layout, tables, pipeline, failure-modes]
prev: 43-siglip
next: 45-colpali-1-documents-as-images
---

# The Broken Promise of OCR Pipelines

**The one-paragraph version.** The usual way to search PDFs and scans is to turn them into text
first. The page passes through six stages: OCR, layout detection, table extraction,
reading-order reconstruction, chunking and embedding. Every stage is lossy, every stage can
fail, and the failures multiply. Charts become nothing. Tables get scrambled. Reading order gets
shuffled. And because each stage is tuned on its own, nobody measures the one thing we care
about: whether the right page comes back.

In this chapter, we will learn what an OCR pipeline is, what each of its six stages does, and
exactly what the pipeline throws away. We will follow one scanned contract page through the whole
pipeline, and watch a correct fact turn into a wrong answer. We will also see how to audit a
pipeline, and what the alternatives are.

We will cover the following:

- What is an OCR pipeline
- Why teams build one
- How the six stages work
- An example: page 3,507 through the pipeline
- What the pipeline loses
- Why the errors compound
- The operational cost
- Three ways to search scanned pages, and when to use which one

---

## What is an OCR pipeline

**OCR = Optical Character Recognition.**

- **Optical:** it works from an image, such as a scan, a photo or a rendered PDF page.
- **Character Recognition:** it finds the letters and digits in that image and writes them out
  as text.

In simple words, OCR turns a picture of text into text a computer can search.

**OCR pipeline = OCR + a chain of clean-up stages + a text embedding model.** Every page in the
Great Library is converted to plain text, cut into passages, and embedded like any other text.
After that, the Librarian searches it exactly as in Parts III to V.

---

## Why teams build one

Every tool we have built so far works on text. BM25 (Chapter 8) needs words. Embedding models
need sentences. The Scholar reads text.

Acme's knowledge base is not all text. Among its 40,000 pages are scanned contracts, faxed order
forms and PDF slide decks. Some are photos of paper. A scan has no words in it at all, only
pixels.

So the obvious plan is: turn every page into text, then reuse the whole text stack. For clean,
simple pages this works well. The trouble starts on the pages that matter most.

---

## How the six stages work

A page passes through six hands:

```
PDF/scan
   ↓  1. OCR                    → characters, with errors
   ↓  2. layout detection       → columns, headers, figure boxes
   ↓  3. table extraction       → rows and cells, sometimes
   ↓  4. reading order          → a linear sequence, hopefully correct
   ↓  5. chunking               → passages
   ↓  6. embedding              → vectors
```

**Step 1: OCR.** Find the characters on the page and write them out.

**Step 2: Layout detection.** Draw boxes around regions: columns, headings, tables, figures,
footers.

**Step 3: Table extraction.** For each box marked "table", rebuild the rows and cells.

**Step 4: Reading order.** Decide which box comes first, second and third, and join all the text
into one long string.

**Step 5: Chunking.** Cut that string into **chunks**, passages short enough to embed (Chapter 50
covers chunking in depth).

**Step 6: Embedding.** Turn each chunk into a vector and store it in the Card Catalog.

Each stage takes the previous stage's output as the truth. None of them can see the original page
again.

Think of it like a game of telephone:

- The original page is the message whispered at the start.
- Each stage is one person passing it on.
- Each person hears only the person before them, never the original.
- The Scholar is the last person, who says the message out loud.

A small mishearing early on is never corrected. It is passed on, and built upon.

---

## An example: page 3,507 through the pipeline

Page 3,507 of the Library is a scanned Acme–Globex contract. Its pricing table looks like this on
paper:

```
┌───────────────────┬──────────────┬────────────────────────────┐
│ Item              │ Price        │ Terms                      │
├───────────────────┼──────────────┼────────────────────────────┤
│ Pro licence       │ $15 / seat   │ 120 seats, annual          │
│ Security add-on   │ $4 / seat    │ optional                   │
│ SSO               │ $0           │ included (Pro, 50+ seats)  │
└───────────────────┴──────────────┴────────────────────────────┘
```

A support agent asks the Librarian:

> *"Does the Globex contract include SSO?"*

A person reads the table in five seconds. Globex bought 120 Pro seats. SSO costs $0 and is
"included (Pro, 50+ seats)". So the answer is yes.

Now, let's see what the pipeline does with this page.

**Stage 1, OCR.** The scan is slightly tilted and a little faint. OCR gets 99% of the characters
right. One of the few it misreads is in "SSO", which comes out as `S5O`.

**Stage 2, layout detection.** The table's grid lines are faint on the scan. The layout model does
not see a table. It sees three narrow columns of text.

**Stage 3, table extraction.** No box was marked "table", so this stage has nothing to do. The
rows are never rebuilt.

**Stage 4, reading order.** Three columns are read the way we read a newspaper: all of column one,
then all of column two, then all of column three.

```
Item Pro licence Security add-on S5O Price $15 / seat $4 / seat $0
Terms 120 seats, annual optional included (Pro, 50+ seats)
```

**Stage 5, chunking.** The chunker cuts every few dozen words. The cut falls right after
`$4 / seat`. The first chunk starts with the page heading above the table, *Schedule B: Pricing
for Globex Corporation*, and ends with the three item names and only two of their prices. The
words `$0`, `120 seats` and `included (Pro, 50+ seats)` land in the next chunk, next to the
signature block.

**Stage 6, embedding.** The first chunk's vector is about "Globex pricing, licences, security
add-on". The second chunk's vector is about "terms and signatures".

The Librarian brings back the first chunk, because it mentions Globex, pricing and the Security
add-on. It also brings back page 1,140: *"On Pro, SSO needs the Security add-on for teams under 50
seats."*

The Scholar reads them. It thinks like this: *"The contract lists three items, Pro licence,
Security add-on and S5O, which must be SSO, but only two prices. So SSO seems to come with the $4
Security add-on. Page 1,140 agrees: SSO on Pro needs the Security add-on for teams under 50 seats.
I cannot see how many seats Globex has. So Globex must pay for the add-on to get SSO."*

It answers: *"No. SSO is not included. Globex needs the Security add-on at $4 per seat."*

The answer is wrong. The words that decided the question, `$0` and *included*, never reached the
Scholar.

Look at what went wrong. No single stage failed badly. OCR was 99% accurate. The layout model
found real text blocks. The chunker cut at a sensible length. Yet the fact `SSO: included (Pro,
50+ seats)` was split into pieces that no longer mean anything together.

---

## What the pipeline loses

Is page 3,507 unlucky, or is this normal? Let's go through the losses one kind at a time.

**Loss 1: OCR errors.** Modern OCR is good on clean printed text, often 98–99% of characters
correct. It degrades sharply on scans, faxes, handwriting, stamps, low contrast and unusual fonts.
And 99% of characters is *not* 99% of words. With about 5 characters per word, 0.99⁵ ≈ 0.95, so
roughly 1 word in 20 has an error. One wrong character in `SSO-4012` makes that error code
unfindable by keyword search.

**Loss 2: Reading order.** A two-column page read straight across the full width mixes two
unrelated columns, line by line. The text is nonsense, and it embeds to a vector that means
nothing. Sidebars, footnotes, pull quotes, headers and page numbers get spliced into the middle of
paragraphs.

**Loss 3: Tables.** This is the worst one. A table's meaning lives in the *alignment* of each cell
with its row and column headers. Flatten it and cut it, and `$0` and `included` lose their link to
`SSO`, as we just saw. Even before the cut, the flat string runs cells together, as in
`120 seats, annual optional included`, so a reader must guess where one cell ends.

**Loss 4: Charts and figures.** Usually lost entirely. If the contract had a chart of seat growth by
quarter, the pipeline would keep, at best, its caption: *"Figure 2: Seats by quarter"*. The actual
values, the trend and the outlier quarter are gone. If a document's key finding is in a figure, the
pipeline never saw it.

**Loss 5: Everything visual.** Page 3,507 has a red "CONFIDENTIAL" stamp, two signatures, a ticked
checkbox beside "Security add-on declined", and one clause struck through with a handwritten
correction. Each one carries meaning. Each converts to nothing, or to a confusing fragment.

**Loss 6: Forms.** On a form, a label and its value are linked by *position*. After flattening,
`"Effective date"` and `"2026-03-01"` may end up paragraphs apart.

**Loss 7: Chunk boundaries.** A fixed-length cut can separate a row label from its value, as it did
on page 3,507. Chapter 50 covers smarter ways to cut.

---

## Why the errors compound

Each stage is trained and evaluated on its own. The OCR team measures character accuracy. The layout
team measures box **IoU**.

**IoU = Intersection over Union.** It is a box-overlap score: the area where the predicted box and
the correct box overlap, divided by the area they cover together. A score of 1 means a perfect
match. A score of 0 means no overlap.

The chunker is a rule nobody evaluated at all.

**Nobody measures end-to-end retrieval quality**, which is the only number that matters.

Let's put a rough number on it. Suppose each of the six stages keeps 90% of the information that
matters. The pipeline keeps

$$ 0.9^6 \approx 53\% $$

In simple words, six decent stages in a row lose nearly half. This is a toy model, since real stages
are not independent. But the real picture is worse, for two reasons.

First, the failures are not random noise. They are systematic, and **nothing reports them.** We get
text out. It looks like text. It is quietly wrong.

Second, the failures follow value. A plain page of prose survives nearly intact. A dense pricing
table, a scanned contract with handwritten changes, or a figure with an embedded legend gets mangled.
Those are exactly the pages people search for.

> **The diagnostic question for any document pipeline:** *"What fraction of my corpus is visually
> complex, and what happens to those pages?"* Most teams have never looked. Take twenty of your most
> important pages, run them through your parser, and read the output beside the original. It is a
> sobering twenty minutes and it will decide your architecture.

---

## The operational cost

Even when it works, the pipeline is expensive to run and to own.

- **Latency and throughput.** OCR plus layout detection can take seconds per page. A million pages
  is a real compute job.
- **Many components, many licences.** Tesseract (a popular open-source OCR engine) or a commercial
  one, a layout model, a table extractor and a reading-order rule. Each has its own versions,
  failure modes and costs.
- **Per-document-type tuning.** The settings that work for invoices do not work for scientific
  papers.
- **Silent failure.** A malformed PDF yields empty text. An empty chunk still gets embedded, into a
  meaningless vector or a zero vector that breaks normalization (Chapter 5). Either it poisons the
  index or it is quietly dropped. That document is now invisible, and nothing alerted anyone.

---

## Three ways to search scanned pages, and when to use which one

There are three serious options today.

**Option 1: The OCR pipeline.** The six stages in this chapter.

**Option 2: VLM transcription.** A vision-language model (Chapter 43) reads each page image and
writes it out as clean, structured text in markdown, a simple plain-text format for headings and
tables. Tables become markdown tables. Figures become written descriptions. Then chunking and
embedding run as usual. Six stages become three: transcribe, chunk, embed. The new risk is
**hallucination**: the model confidently writing something the page does not say (Chapter 49
covers it in depth).

**Option 3: Search the page image directly.** Do not convert to text at all. Embed the page picture
and let the retriever match queries against it. This is ColPali, the subject of Chapters 45 to 47.

| | OCR pipeline | VLM transcription | Page images (ColPali) |
|---|---|---|---|
| Stages before search | Six | Three | Two (render, embed) |
| Tables | Often scrambled | Usually good | Good |
| Charts, stamps, checkboxes | Lost | Described in words | Kept in the pixels |
| New failure mode | Silent garbling | Hallucination | Detail lost in resizing |
| Exact codes like `SSO-4012` | Yes, if OCR reads them | Yes, if transcribed | Weak on its own |
| Reuses the existing text stack | Yes | Yes | No, needs multi-vector search |
| Cost per page to index | Low to medium | Highest | Medium, GPU |

**Advantages of the OCR pipeline.** Mature tools. Cheap on clean pages. Produces text for keyword
search, compliance and extraction.

**Disadvantages of the OCR pipeline.** Six lossy stages. Tables, charts and visual marks are damaged
or lost. Failures are silent, and they cluster on the most valuable pages.

We must use **the OCR pipeline** when the corpus is mostly clean, digital prose with few tables, or
when we need exact text downstream for other jobs.

We must use **VLM transcription** when the corpus is mostly prose with some tables and figures, and
we want to keep our text stack.

We must use **page images** when the documents are visually rich (slides, financial reports,
scientific papers, forms, scanned archives) and retrieval quality on those pages is the point.

Many strong systems use two: page images for meaning, plus OCR text feeding a keyword index for
exact codes and names. Chapter 47 shows that design.

---

### Under the hood

Audit your own pipeline. This is the most useful hour in Part VI.

**Step 1:** Pick 20 PDFs at random.

**Step 2:** Run each through the parser.

**Step 3:** Compute a few cheap statistics on the output.

**Step 4:** Flag anything that looks broken.

**Step 5:** Read every extraction side by side with its original page.

```python
import random
import numpy as np
import pandas as pd

def audit_extraction(pdf_paths, parser, sample=20):
    rows = []
    for p in random.sample(pdf_paths, min(sample, len(pdf_paths))):
        text = parser.extract(p)
        rows.append({
            "file": p,
            "chars": len(text),
            "empty": len(text.strip()) < 100,
            "digit_ratio": sum(c.isdigit() for c in text) / max(len(text), 1),
            "avg_word_len": np.mean([len(w) for w in text.split()] or [0]),
            "nonascii_ratio": sum(ord(c) > 127 for c in text) / max(len(text), 1),
        })
    return pd.DataFrame(rows)
```

`audit_extraction` does Steps 1 to 3. In simple words, it measures how much text came out and what
that text looks like. The statistics catch gross failures: empty output, OCR garbage raising the
non-ASCII ratio, or a suspiciously short average word length from characters split apart. They will
not catch a scrambled table or a reading-order error. Only your eyes will, in Step 5.

Signals that your pipeline is failing:

- Pages producing under 100 characters that visibly contain text
- Average word length below 3 or above 12 (character-level splitting, or words merged)
- High non-ASCII ratio on English documents (OCR noise)
- Digit density far below what the page visibly contains (table extraction failing)

---

### What people get wrong

**"OCR is a solved problem."** Character recognition on clean print largely is. *Document
understanding* is not: layout, tables, figures and reading order.

**"I'll use a better PDF library."** Library choice moves the numbers a little. The architecture is
the problem: you are throwing away two-dimensional structure to get a one-dimensional string.

**"I'll just add a vision model to caption the figures."** Better, and now you have seven stages
instead of six, plus captions that summarise rather than preserve.

**"Digital PDFs don't need OCR, so they're fine."** A digital PDF gives exact characters and *no*
reading order. The text layer is stored in creation order, which for a two-column layout is often
interleaved. You have solved stage 1 and none of stages 2–4.

**Not measuring.** The single most common error. Teams ship a pipeline, see mediocre retrieval, and
tune the embedding model, because the extraction failure is invisible in every dashboard they have.

---

### Ninja notes

**There is an honest middle path, and for many corpora it is the right answer.** Use a
vision-language model to *transcribe* pages into structured markdown. Tables become markdown
tables, figures become detailed descriptions, and reading order is corrected. Then embed that text
with your existing pipeline.

This replaces OCR, layout, tables and reading order with one model pass, handles layout and tables
far better than traditional parsers, and keeps your whole text retrieval stack unchanged. The
costs are real. One VLM forward pass per page is slower and more expensive than OCR. And the model
may hallucinate, writing content that was not on the page, which is a new and unpleasant failure
mode. Spot-check transcription quality on a sample, always.

**Chapter 45's approach is more radical:** do not convert to text at all. Embed the page image
directly, and let the retriever match queries against pixels. No transcription, no hallucination
risk, nothing converted. (Something is still lost: the page is resized to a fixed grid of pixels
before encoding. Chapters 46 and 47 measure that loss.)

**Which to choose?** If your documents are mostly prose with occasional tables, VLM transcription
into your existing stack is pragmatic and cheap to adopt, though the most expensive per page to
index. If your documents are visually rich (slides, financial reports, scientific papers, forms,
scanned archives) and retrieval quality on those pages is the point of the project, go direct to
pixels. Chapter 47 gives the full decision criteria. Chapter 61 gives a complete architecture.

---

### Key takeaways

- **OCR pipeline = OCR → layout detection → table extraction → reading order → chunking →
  embedding.** Six lossy stages whose losses multiply: 0.9⁶ ≈ 53%.
- Tables lose the cell-to-header link that carries their meaning. On page 3,507, flattening and
  one chunk cut split `SSO: $0, included (Pro, 50+ seats)` apart, and the Scholar answered wrongly.
- Charts, stamps, checkboxes and signatures are usually lost entirely.
- Reading-order errors turn multi-column pages into nonsense.
- Failures concentrate on visually complex pages, the ones that matter most.
- Nothing in the pipeline measures end-to-end retrieval quality. You must.
- Audit by reading twenty extractions beside their original pages. Today.

### What's next

[Chapter 45](./45-colpali-1-documents-as-images.md) deletes the parser entirely: take a picture of
the page, embed the pixels, and search that.

Now we understand the OCR pipeline, what each of its six stages throws away, and why a correct page
can still produce a wrong answer.
