---
title: "Image-Heavy RAG, End to End"
chapter: 61
part: "Part VIII — Scale and Production"
slug: image-heavy-rag
readingTime: "15 min"
summary: "A complete, costed design for question answering over a million scanned, chart-filled, table-heavy pages — assembled from ColQwen, MUVERA, hybrid search and a vision LLM."
tags: [colpali, colqwen, muvera, multimodal-rag, architecture, case-study]
prev: 60-cost-engineering
next: 62-security-and-tenancy
---

# Image-Heavy RAG, End to End

**The one-paragraph version.** Question answering over a million scanned pages full of tables
and charts needs no new ideas, only the right assembly of earlier ones. We render every page to
an image and encode it with ColQwen2 into patch vectors. We squash each page into one MUVERA
vector for fast search, and run OCR only to feed a keyword index for exact terms. We fuse the
two result lists with RRF, rerank with exact MaxSim, and add the neighbouring pages. Then a
vision LLM reads the page images and answers with page-level citations.

In this chapter, we will design that complete system piece by piece, watch it answer a question a
text pipeline gets wrong, price it, and check it against a latency target.

We will cover the following:

- What we are building
- Why a text pipeline gets it wrong
- How the design works
- An example: page 3,507, end to end
- Why each piece is there
- Storage and cost
- The latency budget and the SLO
- When to use which one

---

## What we are building

Acme's support knowledge base is 40,000 pages. Its contract archive is far bigger: over the
years, sales and legal have scanned **1,000,000 pages** of signed contracts, order forms, price
lists and renewal decks. About 40% of those pages contain a table or a chart. About 15% are
poor-quality scans.

The archive also holds the contracts already in the Great Library, including page 3,507, the
scanned Acme–Globex contract from Part VI. We keep calling it page 3,507. Its Schedule B pricing
table has a row that says "SSO: $0, included (Pro, 50+ seats)".

Account managers ask questions like these:

- *"Does Globex's contract include SSO on the Pro plan?"* The answer sits in a table.
- *"Which customer contracts contain a change-of-control clause?"* The answer sits in clause text.
- *"Show the renewal-deck slide where we projected Pro-plan growth."* The answer is a chart.
- *"What does order form OF-7 require for the seat count?"* The query is an exact identifier.

The first one is our main question. The requirements are:

- **Page-level citations.** Every answer points to the document and page it came from.
- **p95 retrieval latency under 150 ms.**
- **Region isolation.** An account manager sees only their own sales region's contracts.
- **Freshness.** New documents are searchable within an hour.

**p95** works like the p99 of Chapter 18: 95 of every 100 queries must finish faster than
this time.

**SLO = service-level objective.** It is a measurable target the team commits to, such as
"p95 retrieval latency under 150 ms". In simple words, an SLO is the speed limit we promise to
stay under. We will check the finished design against it.

A **vision LLM** is an LLM that can read images as well as text (the VLMs of Chapter 43). It
plays the Scholar in this chapter.

---

## Why a text pipeline gets it wrong

Let's first see what a classic text pipeline does with *"Does Globex's contract include SSO on
the Pro plan?"* This is Chapter 44's six-stage pipeline: OCR, layout, tables, reading order,
chunking, embedding.

OCR reads page 3,507. The scan is faint and the table lines are thin. The text layer of its
pricing table comes out like this:

```
Item Pro licence Security add-on S5O Price $15 / seat $4 / seat $0
Terms 120 seats, annual optional included (Pro, 50+ seats)
```

The table has been flattened into two lines, one column after another. "SSO" became `S5O`, with
a five. Nobody can tell which price or term belongs to which item.

The chunker cuts that mess into chunks and embeds them. The question's vector lands closer to a
different page: page 3,512, a scanned cover letter from the negotiation. It says "SSO pricing
to be confirmed."

The Scholar reads page 3,512 and writes: *"Globex's contract does not settle SSO. Pricing was
still to be confirmed."*

The answer is wrong. The signed table says SSO is included on Pro for teams of 50 or more.

Nothing crashed. Each stage did its job a little badly, and six small losses multiplied
(0.9⁶ ≈ 53%, Chapter 44).

Now, the question is, how do we find page 3,507 when OCR has scrambled its table? We stop
searching the text layer, and search the pixels instead.

---

## How the design works

**Image-heavy RAG = page images + multi-vector search + a keyword safety net + a Scholar that
reads pictures.**

**Phase 1: Preparing the archive.**

- **Step 1:** Render every page of every PDF or scan to an image, at 150–200 DPI.
- **Step 2:** Store each image in object storage. The Scholar and the user interface need it later.
- **Step 3:** Run each image through the ColQwen2 encoder. A dense page gives roughly 750 patch
  vectors of 128 numbers each (Chapter 47).
- **Step 4:** Pool those vectors about 3×, down to roughly 250 per page (token pooling, Chapter 47).
- **Step 5:** Compress the pooled vectors and save them in a multi-vector store, for exact MaxSim later.
- **Step 6:** Turn each page's patch vectors into one MUVERA FDE: a single ~4,000-number vector whose dot product imitates MaxSim (Chapter 38).
- **Step 7:** Add every FDE to an ordinary ANN index, such as HNSW or DiskANN.
- **Step 8:** Run OCR on the page and put the text in a BM25 index. We use it only for exact words.
- **Step 9:** Store metadata: document id, page number, title, type, date, region, previous and next page.

**Phase 2: Answering a question.**

- **Step 1:** Read the user's region from their login. It becomes a filter.
- **Step 2:** Encode the question with ColQwen2's query encoder. We get one vector per query token.
- **Step 3:** Turn those vectors into a query FDE and search the ANN index for the top 200 pages, inside the filter.
- **Step 4:** At the same time, search BM25 over the OCR text for the top 100 pages, inside the filter.
- **Step 5:** Fuse the two lists with RRF (Chapter 40) and keep the top 150 pages.
- **Step 6:** Score those 150 with exact MaxSim against their stored patch vectors. Keep the top 5.
- **Step 7:** If even the best score is below a floor, say "not found" instead of answering.
- **Step 8:** Add the page before and after each hit, and remove duplicates.
- **Step 9:** Hand the page images to the vision LLM. It answers and cites `[document, page]`.

Here is the same design as a diagram:

```
INGESTION
  PDF / scan
     │
     ├─► render pages @150–200 DPI ─► object store (page images)          ── for the LLM + UI
     │
     ├─► ColQwen2 encoder ─► patch vectors (~750 on a dense page)
     │        │
     │        ├─► token pooling (~3×) ─► quantize ─► multi-vector store     ── for exact MaxSim
     │        └─► MUVERA FDE (one vector/page) ─► ANN index (HNSW/DiskANN)  ── for fast recall
     │
     ├─► OCR (text layer only) ─► BM25 index                               ── for exact terms
     │
     └─► metadata: doc_id, page_no, title, doc_type, date, region, prev/next page

QUERY (hot path: no LLM rewrite)
  question
     ├─► region filter from the login
     ├─► ColQwen2 query encoder ─► FDE ─► ANN search (filtered)     ─► top 200 pages ─┐
     ├─► BM25 over OCR text (filtered)                             ─► top 100 pages ─┤
     │                                                                                ▼
     │                                                       RRF fusion ─► top 150 pages
     │                                                                                ▼
     │                                         exact MaxSim on pooled patch vectors ─► top 5
     │                                                                                ▼
     │                                                    expand to page n−1, n+1 (dedupe)
     │                                                                                ▼
     └────────────────────────────────────── vision LLM reads page images ─► answer + [doc, page]
```

Think of it like a law firm answering a client's question from a warehouse of paper contracts.

- A junior clerk flips through thumbnails of every page and pulls a few hundred that look
  right. That is the FDE search.
- A second clerk checks the index of contract numbers and names. That is BM25.
- A senior paralegal compares each pulled page closely against the question. That is MaxSim.
- The lawyer reads the final handful of pages, plus the pages on either side, and writes the
  answer. That is the vision LLM.

In simple words, Phase 2's Steps 1 to 8 are the Librarian's work and Step 9 is the Scholar's. The new
part is that the Scholar reads photographs of pages, not extracted text.

**Note:** Chapter 52's query rewriting is not in Phase 2. That is deliberate, and the latency
section explains why.

---

## An example: page 3,507, end to end

Let's now run *"Does Globex's contract include SSO on the Pro plan?"* through Phase 2. The
ranks below are toy numbers, chosen to show the mechanics.

**Step 1.** The account manager works in the Americas region. Globex is an Americas customer,
so page 3,507 is inside the filter.

**Steps 2 and 3.** The FDE search returns 200 pages. Page 3,507 is at rank 14, among many
look-alike pricing tables from other contracts.

**Step 4.** BM25 finds "Globex" on the contract's cover page, 3,505, and on the cover letter,
3,512. It also finds it in page 3,507's heading, *Schedule B: Pricing for Globex Corporation*,
at rank 9. It does not match "SSO" on page 3,507, because OCR wrote `S5O`.

**Step 5.** RRF fuses the lists. Page 3,507 appears in both, so its score is
1 / (60 + 14) + 1 / (60 + 9) ≈ 0.0135 + 0.0145 ≈ 0.0280. That is comfortably inside the top 150.
Even from the FDE list alone, 0.0135 would have been enough.

**Step 6.** MaxSim now compares every query token with every patch on each page. The token
"SSO" finds its best match in the patch covering the "SSO" cell. "Pro" matches "Pro licence" and
the "(Pro, 50+ seats)" note. "included" matches the word in the SSO row. Page 3,507 rises to
rank 1. The cover letter drops to rank 4.

**Steps 7 and 8.** The best score clears the floor. We add pages 3,506 and 3,508. Page 3,508
is the order form and signature page. It confirms the 120 Pro seats and carries the contract's
effective date, June 2026.

**Step 9.** The vision LLM reads the page images. It thinks like this: *"The user asks whether
Globex's contract includes SSO on Pro. Page 3,507's table says Globex has 120 Pro seats, and SSO
is included on Pro from 50 seats. 120 is more than 50, so the condition is met. Page 3,508 is
the signed order form. It confirms the 120 seats and took effect in June 2026, so this contract
is the current one."*

It answers: *"Yes. SSO is included on Globex's Pro plan, because Globex has 120 seats and SSO is
included from 50 seats [Acme–Globex contract, pp. 3,507 and 3,508]."*

The answer is correct.

Two details made the answer correct and trustworthy. First, MaxSim matched the question to the
pixels of the table cell, so the OCR typo never mattered. Second, neighbour expansion brought in
page 3,508. Without it, the Scholar would have known the terms but not when the contract took
effect, so it could not tell that this is the current contract.

---

## Why each piece is there

**Why page images, not OCR text, for meaning?** About 40% of this archive is tables and charts,
exactly what OCR pipelines lose (Chapters 44–45).

**Why ColQwen2 rather than the original ColPali?** The original squeezes every page into a fixed
448×448 square, distorting A4 pages and wasting vectors on sparse slides. ColQwen2's dynamic
resolution keeps the page's shape and gives a dense page up to roughly 768 patch vectors
(Chapters 46–47).

**Why render at 150–200 DPI?** The vision LLM reads fine print, so the stored images need it.
The retriever is different. ColQwen2 resizes each page to fit its token cap, so extra DPI helps
retrieval only if we raise that cap, which also raises vectors per page (Chapter 47). Sweep DPI
and the cap together on the eval set, per document type.

**Why token pooling?** It gives about 3× fewer vectors per page for a small quality cost
(Chapter 47). At a million pages, float32 patch vectors shrink from about 384 GB to 128 GB, and
compressed codes from about 27 GB to 9 GB.

**Why MUVERA FDEs for the first search?** Exact MaxSim over the whole archive would mean about
5 billion 128-number dot products for every 20-token question. That does not fit in 150 ms. An
FDE turns each page into one ordinary vector, so the filtering, sharding and compression of
Part IV all apply unchanged (Chapter 38).

**Why still run OCR?** For exact identifiers. *"Order form OF-7"*, contract numbers and clause
titles are keyword queries (Chapter 7). OCR reads clean printed identifiers well, even when it
scrambles tables. When it misses, as with `S5O`, the FDE side still finds the page.

**Why RRF?** FDE scores and BM25 scores live on different scales (Chapter 40). RRF uses only
ranks.

**Why exact MaxSim to rerank, rather than a cross-encoder?** The patch vectors already exist,
and MaxSim over 150 pages is cheap. It also gives per-token heatmaps that show which part of the
page matched (Chapter 46). A text cross-encoder cannot read a page image anyway.

**Why add neighbouring pages?** Tables and clauses cross page breaks (Chapter 47).

**Why a vision LLM for the answer?** No extraction step sits between retrieval and answer, so no
extraction error can reach it. The Scholar reads the same pixels the Librarian matched.

---

## Storage and cost

Our assumptions: about 750 patch vectors per dense page after ColQwen2's dynamic resolution
(Chapter 47), pooled 3× to 250, each with 128 numbers.

| Store | Calculation | Size |
|---|---|---|
| Page images (JPEG, ~150 DPI) | 1M × ~100 KB | ~100 GB object storage |
| Multi-vector store, 2-bit residual + centroid ID (Ch. 37) | 1M × 250 × 36 B | ~9 GB |
| — or TurboQuant 2-bit (Ch. 27) | 1M × 250 × (32 B codes + 4 B scale) | ~9 GB, no codebooks |
| Full-precision pooled patch vectors (the truth) | 1M × 250 × 512 B | ~128 GB object storage |
| FDE index (~4,000-d, PQ-compressed to ~1 KB) | 1M × ~1 KB + graph | ~1–2 GB RAM |
| BM25 index over OCR text | corpus-dependent | ~5–15 GB |

A 4,000-number FDE in float32 is 16 KB. PQ with 1,000 subvectors of 4 numbers each (Chapter 24)
stores it in 1,000 bytes, which is 16× smaller. That is where the "~1 KB" comes from.

**Everything that must be fast fits on one modest machine per replica.** The FDE index, the
multi-vector store and BM25 add up to roughly 15–26 GB. The bulk, about 230 GB of images and
float32 truth, sits in object storage, the cheapest tier (Chapter 60).

**Index-time compute.** At 2–10 pages per second per GPU, encoding a million pages takes
roughly 30–140 GPU-hours. OCR for the BM25 side runs on CPUs in parallel. That is a one-time
cost. New documents flow through Phase 1 into a small delta index (Chapter 59). One GPU handles
7,000–36,000 new pages an hour, so the one-hour freshness target is comfortable.

---

## The latency budget and the SLO

Here is where the time goes for one question:

| Stage | Budget |
|---|---|
| ColQwen2 query encoding | 20–50 ms |
| FDE ANN search + BM25, in parallel | 10–30 ms |
| RRF | < 1 ms |
| Exact MaxSim over 150 pages | 10–40 ms |
| **Retrieval total, hot path (no rewrite)** | **~40–120 ms** |
| Query rewrite with a small LLM, if we ran it | 100–300 ms |
| Vision LLM answer (5–7 page images) | seconds |

Now, the question is, does this design meet the 150 ms SLO?

The **hot path** is the route that ordinary queries take. Without a rewrite, retrieval takes
about 40–120 ms, which is under 150 ms. Add a rewrite and retrieval becomes 140–420 ms. Most of
that range breaks the target.

**The 150 ms SLO holds only if query rewrite is skipped on the hot path.**

We can still rewrite some queries without paying for it on every one. There are two ways:

1. **Route.** A cheap router flags hard queries, such as long, vague or multi-part ones. Only
   those take the slow path with a rewrite.
2. **Run it in the background.** Answer from the hot path first. When the rewritten query's
   results return, rerank with them.

**Note:** The SLO covers retrieval, not the whole answer. The vision LLM takes seconds, and it
dominates the time the user waits (Chapter 60). The number of page images is the main lever on
it, because each image costs many input tokens.

---

## When to use which one

Here is the image pipeline next to the text pipeline of Chapter 44:

| | Text pipeline (Ch. 44) | Image pipeline (this chapter) |
|---|---|---|
| Tables, charts, scans | Often lost in extraction | Matched on pixels |
| Exact identifiers | Good (BM25) | Good, through the OCR → BM25 side index |
| Index-time compute | CPU OCR plus a text embedder | ~30–140 GPU-hours per million pages |
| Moving parts | Fewer | Two indexes, a multi-vector store, page images |
| Answer model | Any LLM | A vision LLM, with many tokens per page |
| Evidence shown to users | Extracted text | The page itself, with the match highlighted |

**Advantages of the image pipeline:** answers in tables, charts and bad scans become findable,
no extraction error reaches the Scholar, and users see the real page as evidence.

**Disadvantages of the image pipeline:**

- **Problem 1: GPU cost at ingestion.** A million pages takes roughly 30–140 GPU-hours, and every
  model upgrade repeats it.
- **Problem 2: More moving parts.** Two indexes, a multi-vector store and an image store must all
  stay in sync when page 3,507 is replaced.
- **Problem 3: Slower, pricier answers.** Seven page images usually cost far more input tokens
  than seven text chunks.
- **Problem 4: The token cap still limits fine print.** A tiny footnote can be unreadable to
  the retriever even when the stored image is sharp.

We must use the **image pipeline** when a large share of answers live in tables, charts, forms or
scans, and OCR quality is poor or uneven. We must use the **text pipeline** when documents are
mostly born-digital prose, where extraction loses little. A sensible middle ground uses both:
route prose documents through text, route visual ones through images, and fuse the results
with RRF.

---

### Under the hood

The query path, condensed. It follows Phase 2, and the rewrite sits off the hot path:

```python
from asyncio import gather

async def answer(question, user):
    filters = {"region": user.region}                      # from the login, Ch. 62
    q = question                                           # hot path: no LLM rewrite
    if router.is_hard(question):                           # rare: long, vague, multi-part
        q = await rewrite(question)                        # slow path, Ch. 52, 100–300 ms

    q_vecs = colqwen.encode_query(q)                       # (n_tokens, 128)
    fde_task  = fde_index.search(fde.encode(q_vecs, is_query=True), k=200, filters=filters)
    bm25_task = bm25.search(q, k=100, filters=filters)
    fde_hits, bm25_hits = await gather(fde_task, bm25_task)

    candidates = rrf([fde_hits, bm25_hits])[:150]          # Ch. 40

    page_vecs = mv_store.fetch_many([c.page_id for c in candidates])
    scored = sorted(((c, maxsim(q_vecs, page_vecs[c.page_id])) for c in candidates),
                    key=lambda x: -x[1])[:5]               # Ch. 36

    if not scored or scored[0][1] / len(q_vecs) < NO_EVIDENCE_FLOOR:   # Ch. 46, 55
        return "I couldn't find this in the documents you have access to.", []

    pages = expand_neighbours([c for c, _ in scored], radius=1, max_pages=7)
    images = [object_store.get(p.image_key) for p in pages]
    labels = [f"[{p.doc_title}, p.{p.page_no}, {p.date}]" for p in pages]

    return vision_llm.answer(question, images, labels, instructions=GROUNDED_CITE), pages
```

Look at the no-evidence check. `scored[0][1] / len(q_vecs)` divides the best MaxSim score by the
number of query tokens. In simple words, it turns a total into an average per token. Raw MaxSim
grows with the number of query tokens (Chapter 36), so the average is what lets one floor work
for short and long questions alike.

The vision LLM still receives the original `question`. The rewrite helps retrieval only.

---

### What people get wrong

**Running OCR as the semantic retrieval layer "because we already have it".** It brings back
every failure of Chapter 44, on exactly the pages that matter.

**Skipping BM25 because "ColPali handles text".** It handles meaning. It does not guarantee an
exact match on *OF-7*.

**Exhaustive MaxSim over the whole corpus.** It works in a demo on 5,000 pages. It misses the
SLO at a million.

**Putting query rewrite on the hot path.** A 100–300 ms LLM call inside a 150 ms budget is
arithmetic that cannot work, however fast the index is.

**One DPI or token cap for all document types.** Slides and price lists with footnotes have
very different text densities.

**Passing too many page images to the LLM.** Each one is expensive in tokens and latency. Five to
seven, including neighbours, is typically enough. Measure it.

**Not keeping page images.** Without them we cannot show the evidence, cannot feed a vision LLM,
and cannot re-encode when a better model arrives.

---

### Ninja notes

**Highlight the evidence.** Per-token MaxSim heatmaps (Chapter 46) map query terms onto page
regions. Showing page 3,507 with the SSO cell highlighted builds a lot of user trust. It also
makes wrong answers obvious at a glance, which is exactly what regulated domains want.

**The default token cap is lower than the render DPI.** In the reference implementation,
ColQwen2 caps a page at roughly 768 visual tokens, and each token covers a 28×28-pixel square.
That is about 600,000 pixels, so an A4 page hits the cap at roughly 80 DPI (Chapter 47). Renders
above that are resized down for retrieval unless we raise the cap. The stored 150–200 DPI image
is for the Scholar.

**Routing and p95 fit together.** A p95 target ignores the slowest 5 queries in 100. If fewer
than 5% of queries take the rewrite path and every hot-path query stays under 150 ms, the p95
holds. So watch the router's flag rate as closely as latency.

**Evaluate the hard segment separately.** Segment the golden set (Chapter 54) into table, chart,
scanned-page, identifier and multi-page questions, and measure the OCR-text baseline on it too.
The gap on tables and charts justifies this architecture. The gap on prose will be small. Show
both numbers to whoever funds the GPUs.

**Plan the migration on day one.** Vision-document retrievers are improving quickly. Because we
keep page images and the ingestion pipeline (Chapter 57), a better encoder is a re-encode job,
not a re-architecture (Chapter 63).

---

### Key takeaways

- **Image-heavy RAG = page images + multi-vector search + a keyword safety net + a Scholar that
  reads pictures.**
- Render pages to images, encode with ColQwen2, then pool and quantize the patch vectors.
- Use MUVERA FDEs (~4,000-d) for the fast first search over a million pages.
- Run OCR only to feed BM25 for exact identifiers, and fuse the two lists with RRF.
- Rerank with exact MaxSim, expand to neighbouring pages, and let a vision LLM cite
  `[document, page]`.
- The hot data for a million pages fits in roughly 15–26 GB per replica. Retrieval takes about
  40–120 ms.
- **The 150 ms SLO holds only if query rewrite stays off the hot path.** Route hard queries to a
  slow path, or rewrite in the background.
- Keep page images, so the next model upgrade is a re-encode, not a re-architecture. Keep
  full-precision vectors, so a new index format is a rebuild, not a re-encode.

### What's next

This chapter assumed that the region filter cannot be bypassed. [Chapter 62](./62-security-and-tenancy.md)
makes sure of it, and shows that embeddings themselves are not as anonymous as they look.

Now we have seen how the pieces of Parts V, VI and VIII fit into one working system, what it
costs, and the one shortcut we must refuse to keep it fast.
