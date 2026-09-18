---
title: "Chunking"
chapter: 50
part: "Part VII — RAG"
slug: chunking
readingTime: "14 min"
summary: "How you cut documents into pieces can matter more for retrieval quality than which embedding model you pick. The trade-off, the strategies, and the techniques that escape it."
tags: [chunking, rag, late-chunking, contextual-retrieval, preprocessing]
prev: 49-rag-first-principles
next: 51-metadata-and-structure
---

# Chunking

**The one-paragraph version.** A whole page is too much to squeeze into one vector without
losing detail (Chapter 34's bottleneck), so we cut pages into chunks before embedding them.
Small chunks retrieve precisely but lose their context. Large chunks keep context but blur the
signal. Every fixed-size strategy is a compromise between the two. We do better by cutting where
the document's own sections begin and end, and then by using one of two techniques that escape
the trade-off. **Contextual retrieval** adds the missing context back before embedding. **Late
chunking** embeds the whole document first and cuts afterwards.

In this chapter, we will learn what a chunk is and why the way we cut pages matters so much. We
will watch a simple cut bury a crucial sentence on page 1,140 in unrelated text, and see the
Scholar give a wrong answer because of it. Then we will fix it, and settle on one clear default.

We will cover the following:

- What is a chunk
- Why chunking matters
- Chunking strategies, from simple to smart
- Contextual retrieval
- Late chunking
- Parent-child chunks: retrieve small, pass large
- When to use which one
- Finding the right chunk size

---

## What is a chunk

**Chunk = a piece of a document that we embed, store and retrieve as one unit.**

In Chapter 49, chunking was Step 2 of indexing. Now we look at it closely.

We cut pages for two reasons. First, one vector cannot hold every detail of a long page, so the
details blur together (Chapter 34). Second, embedding models can only read a limited number of
tokens at once. So we measure chunk size in tokens (Chapter 10), and a typical chunk is a few
hundred of them.

**Overlap = the number of tokens two neighbouring chunks share.** With 512-token chunks and 50
tokens of overlap, each chunk starts 50 tokens before the previous one ends. The idea is that a
sentence sitting right on a cut still appears whole in at least one chunk.

Think of it like cutting a cookbook into index cards, where each card will be found on its own.

- The cookbook is a document, and each card is a chunk.
- Cut one line per card, and a card says *"bake for 25 minutes"*. Bake what? At what temperature?
  The card is precise and useless.
- Cut one card per chapter, and the dessert card holds forty recipes. Its embedding is an average
  of forty recipes, so it matches "chocolate" only weakly.
- Cut one card per recipe, and each card has a title, ingredients and a method. It is
  self-contained, easy to find and easy to read.

**That is the whole principle: chunk at meaningful boundaries, so that each chunk is a complete
thought.** The rest of this chapter is about finding those boundaries.

---

## Why chunking matters

Here is the trade-off laid out:

| | Small chunks (100–200 tokens) | Large chunks (800–1500 tokens) |
|---|---|---|
| Retrieval precision | High (the needle dominates its vector) | Low (dilution, Ch. 34) |
| Context for the Scholar | Poor (dangling pronouns, missing setup) | Good |
| Chunks per document | Many | Few |
| Index size and cost | Larger | Smaller |
| Risk of splitting an answer | High | Low |
| Best for | Fact lookup, FAQs, specifications | Narrative, reasoning, explanation |

There is no universally correct setting. **There is a correct setting for our questions**, and we
find it by measuring, not by copying a blog post.

Now, let's see a bad cut in action, with our running question: *"Does the Pro plan include single
sign-on?"*

Page 1,140 of the Great Library is Acme's help article "Security add-on". Almost all of it covers
audit logs, IP allow-lists and data retention. It mentions single sign-on in exactly one
sentence, in a short section in the middle. Here is its outline. The token counts are toy
numbers, chosen to keep the arithmetic simple.

```
# Security add-on                                        ← page title

## Audit logs                                            ← about 120 tokens
Every sign-in ... It also records every export and
permission change ...

## Plans and seats                                       ← about 40 tokens
Customers add it from the Billing page.
On Pro, SSO needs the Security add-on for teams under    ← the one SSO sentence
50 seats.                                                   (about 15 tokens)

## IP allow-lists                                        ← about 100 tokens
## Data retention                                        ← about 90 tokens
```

Put simply, the sentence we need is about 15 tokens, in the middle of a page of about 350.

Let's first see what fixed-size chunking does. We cut every 512 tokens with 50 tokens of overlap.
The whole page is shorter than one chunk, so all of it lands in a single chunk. The SSO sentence
is about 15 of its 350 tokens, roughly 4%. The chunk's vector is an average of all of them
(Chapter 11), so it is mostly about audit logs, IP allow-lists and data retention. In the Map
Room it lands far from our question.

The Librarian returns page 212 and page 88, the SSO setup guide. The chunk holding the SSO sentence
does not make the top 5. The Scholar thinks like this: *"Both sources say Pro includes SSO, and
one explains how to set it up."* It writes: *"Yes, SSO is included on Pro. Follow the setup guide
to turn it on."*

The answer is wrong for every team under 50 seats. They will follow the steps and hit a paywall.

Now, let's cut at the document's own structure instead. We split at headings, and allow a
section up to 800 tokens before splitting it further. "Plans and seats" becomes its own
40-token chunk. We also paste the page title and section heading in front of it before
embedding (Chapter 51).

The SSO sentence is now about a third of its chunk instead of 4%, and the rest of the chunk
is about plans, seats and the add-on. It matches the question strongly. The Librarian returns it
together with page 212. The Scholar writes: *"Yes, SSO is included on Pro [1]. Teams under 50
seats also need the Security add-on [2]."*

The answer is correct. Nothing about the embedding model changed. Only the cut did.

---

## Chunking strategies, from simple to smart

**1. Fixed-size with overlap.** Every $N$ tokens, with $M$ tokens of overlap. Typical: 512 with
50–100 overlap. It is crude and fast, and a reasonable baseline.

**2. Recursive splitting.** Try to split on paragraph breaks. If a piece is still too long, split
it on sentences, then on words. This is the default in most frameworks. It is meaningfully better
than fixed-size, because it respects natural boundaries when they exist.

**3. Structure-aware splitting.** Use the document's own markup: markdown headings, HTML
sections, PDF bookmarks, code functions, slide boundaries. **This is the biggest available win,
and the most commonly skipped.** Our documents already contain the author's own boundaries. We
should use them. This is the strategy that fixed page 1,140.

**4. Semantic chunking.** Embed each sentence, then cut wherever two neighbouring sentences are
very dissimilar, which suggests the topic changed. It is elegant. In practice, the gain over good
structure-aware splitting is often small, and it costs an embedding pass over every sentence. Try
it, but try structure first.

**5. Contextual retrieval and late chunking.** These do not change *where* we cut. They change
what each chunk knows about the rest of its document.

Structure-aware splitting has one weakness. It needs structure. Some of Acme's older pages came
from a wiki export that flattened everything into plain text. Take the flattened copy of page
1,140. The heading words are still there, but nothing marks them as headings, so the splitter
cannot see where "Plans and seats" starts or ends. It falls back to fixed-size cuts, the whole
page becomes one chunk again, and the SSO sentence is buried in audit-log text. The next two
techniques help a chunk carry its document with it.

---

## Contextual retrieval

The trouble with a blindly cut chunk is often not its size. It is its **missing context**: the
chunk does not say what it is about, or which page it came from. So we add the context back.

**Contextual retrieval = before embedding each chunk, an LLM writes one or two sentences that
place the chunk within its document, and we paste them in front of the chunk.**

**Step 1:** Give a cheap LLM the whole document and one chunk from it.

**Step 2:** Ask it for one or two sentences that say where this chunk sits and what it is about.

**Step 3:** Paste those sentences in front of the chunk.

**Step 4:** Embed the result, and also index it for keyword search.

Here is the one chunk of the flattened page 1,140, before and after (shortened):

```
Original chunk:
  "Security add-on Audit logs Every sign-in ... Plans and seats Customers
   add it from the Billing page. On Pro, SSO needs the Security add-on
   for teams under 50 seats. IP allow-lists ... Data retention ..."

Contextualised chunk:
  "This chunk is Acme's help article 'Security add-on'. It covers audit
   logs, IP allow-lists and data retention, and gives the rule for
   single sign-on (SSO) on the Pro plan: teams under 50 seats need
   the add-on.
   Security add-on Audit logs Every sign-in ... Plans and seats Customers
   add it from the Billing page. On Pro, SSO needs the Security add-on
   for teams under 50 seats. IP allow-lists ... Data retention ..."
```

Now the chunk opens by naming the SSO rule for the Pro plan, in the question's own words. Its
vector moves toward our question, and keyword search on "single sign-on" matches it too, where
the original only said "SSO". In our example, it now makes the top 5. In simple words, the LLM
writes the label that the cut tore off.

The reported results for this technique are striking. In Anthropic's published experiments,
contextual embeddings combined with contextual BM25 (BM25 run over the contextualised chunks) cut
the top-20 retrieval failure rate by about half (~49%). That rate is the share of questions whose
right chunk is missing from the top 20. Adding a reranker on top brought the
reduction to about two-thirds (~67%). That is a larger improvement than most model upgrades
deliver, though, as always, we should measure it on our own corpus.

**The costs are real and manageable.** It takes one LLM call per chunk at index time. That is
expensive in total, but the calls can be cached and batched, and they run once per document
version. **Prompt caching** makes it much cheaper. Providers charge less for the start of a prompt
they have already seen recently (Chapter 53). Every chunk from one page shares the same document
text, so we put the document first and only the short chunk at the end changes.

---

## Late chunking

Late chunking escapes the same trap in a different way, and it costs no LLM calls. We met it
briefly in Chapter 11.

Normally we cut first and embed each chunk on its own. Each chunk's token vectors are then shaped
only by the tokens inside that chunk.

**Late chunking = embed the whole document first, then cut the token vectors into chunks.**

**Step 1:** Run the *entire* document through a long-context embedding model in one pass. Every
token's vector is now shaped by the whole document.

**Step 2:** Slice the token vectors into the chunk ranges we chose.

**Step 3:** Mean-pool each range into one chunk vector (Chapter 11).

Back to the flattened page 1,140, cut this time into chunks of a few sentences. Our SSO sentence
already names SSO, Pro and the add-on, so it needs little help. Other chunks do. One begins *"It
also records every export and permission change."* Cut first, that chunk never says what "It" is.
With late chunking, the model read the page title "Security add-on" and the words "Audit logs" in
the same pass. So the chunk's vector carries "the Security add-on's audit log", even though those
words are not inside the chunk. Means, the pronoun was resolved before any cutting happened.

**Requirements:** a genuine long-context embedding model (8,000 tokens or more), and a framework
that gives us each token's output vector. A page longer than the model's window must first be
split into large windows.

**Contextual retrieval vs late chunking.** Contextual retrieval is more powerful. The LLM writes
real words, such as "single sign-on" and the page title, so the added context helps keyword search
as well as the dense vector. It costs an LLM call per chunk. Late chunking is nearly free, but it
only changes the dense vector, and it can only use context that is already in the document. They
compose, so we can use both.

---

## Parent-child chunks: retrieve small, pass large

There is one more escape from the trade-off, and it needs no model at all. The unit we search
does not have to be the unit we hand to the Scholar.

**Parent-child chunking = search over small child chunks for precision, then hand the Scholar the
larger parent section each child belongs to.**

**Phase 1: Indexing.**

**Step 1:** Split each page into sections. These are the parents.

**Step 2:** Split each section into small chunks of a few sentences, up to about 200 tokens.
These are the children.

**Step 3:** Embed and index only the children, and store each child's parent ID with it.

**Phase 2: Answering.**

**Step 1:** Search the children and take the top 20.

**Step 2:** Look up each hit's parent, and drop repeated parents.

**Step 3:** Fetch the top 5 parents and put them in the prompt.

On page 1,140, the child chunk holding the SSO sentence is a sentence or two long, so the rule
dominates its vector and it matches our question precisely. The page is short, so we let the
whole page be the parent. The Scholar sees the rule together with the line about adding the
add-on from the Billing page. It can now tell a 20-seat team both that it needs the add-on and
how to get it.

We get small-chunk precision *and* large-chunk context. We also get free deduplication, since
three hits within one section collapse into one passage instead of taking three slots. This is
usually called parent-document retrieval (auto-merging retrieval is a close variant), and it is
often the highest-value chunking technique that costs nothing.

---

## When to use which one

| Technique | Extra cost | Needs | What it fixes |
|---|---|---|---|
| Fixed-size + overlap | None | Nothing | Baseline only |
| Recursive splitting | None | Nothing | Respects paragraphs when present |
| Structure-aware splitting | None | Headings or other markup | Keeps sections whole |
| Semantic chunking | One embedding per sentence | Nothing | Finds topic shifts without markup |
| Late chunking | Nearly free | Long-context model with token outputs | Pronouns and references in the dense vector |
| Contextual retrieval | One LLM call per chunk (cacheable) | An LLM pass at index time | Missing context, for dense and keyword search |
| Parent-child | A parent lookup | Section boundaries | Precision and context together |

Here is the one default rule for this chapter:

> **Start with structure-aware chunking plus title prepend (Chapter 51). Add late chunking if
> your embedding model supports it. Add contextual retrieval where your eval says quality matters
> most and you can afford the LLM pass.**

Title prepend means pasting the page title and section heading in front of each chunk before
embedding it.

We must use **structure-aware chunking** whenever pages have headings, sections, functions or
slides. We must add **late chunking** when the model allows it, because it is nearly free. We must
add **contextual retrieval** where the eval still shows failures and the LLM bill is acceptable.
**Parent-child** sits on top of any of these. All of them are cheaper than fine-tuning an
embedding model, so try them first.

---

## Finding the right chunk size

Now, the question is, how big should each chunk be? We answer it by measuring.

**Step 1:** Pick a few chunk sizes and a few overlaps.

**Step 2:** For each pair, chunk the corpus and build an index.

**Step 3:** Run the eval questions and record recall@5 (Chapter 19).

**Step 4:** Keep the best setting, and check it again whenever the corpus changes a lot.

```python
for size in [128, 256, 512, 1024]:
    for overlap_pct in [0, 0.1, 0.25]:
        chunks = chunk_corpus(docs, size, int(size * overlap_pct))
        idx = build_index(chunks)
        r = evaluate(idx, eval_queries, gold)           # Ch. 19
        print(f"{size:5d} tokens, {overlap_pct:.0%} overlap → recall@5={r:.3f}")
```

In simple words, we try twelve settings and let the numbers pick. **Run this before touching
anything else.** It takes about an hour, and the gap between the best and worst setting is often
larger than the gap between embedding models.

Reasonable starting points, to be verified:

| Content | Size | Overlap |
|---|---|---|
| FAQs, short answers | 128–256 | 0–10% |
| Documentation, articles | 400–600 | 10–15% |
| Legal, contracts | 300–500, at clause boundaries | 15% |
| Narrative, reports | 600–1000 | 10% |
| Code | Per function | Include signature + imports |
| Pages of scanned documents | Do not chunk, use the page (Ch. 45) | n/a |

---

### Under the hood

Structure-aware splitting, which cuts a markdown page at its headings:

```python
def split_by_headings(markdown, max_tokens=800):
    sections, current, heading, in_code = [], [], "", False
    for line in markdown.splitlines():
        if line.lstrip().startswith(("`" * 3, "~~~")):  # a code fence opens or closes
            in_code = not in_code                         # a "#" inside code is a comment
        if line.startswith("#") and not in_code:
            sections.append((heading, "\n".join(current)))
            heading, current = line.lstrip("# ").strip(), []
        else:
            current.append(line)
    sections.append((heading, "\n".join(current)))
    return [s for h, body in sections if body.strip()     # drop empty sections
            for s in split_if_long(h, body, max_tokens)]
```

In simple words, every line starting with `#` opens a new section, and every other line joins the
current one. Inside a code block, a `#` line is a comment, so it does not open a section. Sections
with no text are dropped. On page 1,140, the title line `# Security add-on` is followed only by a
blank line, so without that check it would become an empty chunk. `split_if_long` falls back to
recursive splitting for any section over the limit.

Late chunking, which embeds the page once and then slices:

```python
def late_chunk(doc_text, boundaries, model, tokenizer):
    enc = tokenizer(doc_text, return_tensors="pt", truncation=True, max_length=8192)
    hidden = model(**enc).last_hidden_state[0]          # (T, D), full-doc context
    vecs = []
    for start, end in boundaries:                        # spans in the tokenizer's indices
        v = hidden[start:end].mean(0)
        vecs.append(v / v.norm())
    return vecs
```

That is it. One forward pass gives a vector for every token (`hidden`). Each chunk's vector is
the mean of its tokens' vectors, normalized to length 1 (Chapter 5).

Parent-child retrieval, the answering phase:

```python
hits = index.search(q, k=20)                       # precise, small chunks
sections = dedupe([h.parent_section_id for h in hits])
context = [fetch_section(s) for s in sections[:5]]  # rich, large context
```

---

### What people get wrong

**Ignoring document structure.** Splitting a well-structured markdown file by character count
throws away boundaries the author already gave you.

**Trusting overlap to repair a bad cut.** Overlap only helps a sentence that sits right on a cut.
Fifty tokens of overlap cannot rescue one sentence buried among hundreds of tokens of unrelated
text.

**Retrieving and passing the same unit.** They need not match. Retrieve small, pass large (see
above).

**Excessive overlap.** 50% overlap doubles your index and fills your top-k with near-duplicates of
the same passage, crowding out diversity.

**Chunking tables and code by character count.** Split at row or function boundaries, or keep them
whole.

**Never revisiting it.** Chunking is tuned once, at the start, when nobody has an eval set. Revisit
it after you have one.

**Leaving empty chunks.** Whitespace-only chunks become meaningless or all-zero vectors (Chapter 5)
and either crash your index build or silently poison it. Filter them at ingestion.

---

### Ninja notes

**Hierarchical indexing is the extension of parent-child.** Build summaries of sections and of
whole documents, index those too, and search all levels. A broad query (*"what does the Security
add-on cover?"*) matches the document summary. A specific one matches a single paragraph. It is
more machinery to maintain, and for large mixed corpora it is worth it.

**For document images, this entire chapter mostly dissolves.** The page is the chunk
(Chapter 45), a boundary the author already chose, with no arbitrary cutting at all. The leftover
problem is content that spans pages, handled by retrieving neighbouring pages.

---

### Key takeaways

- **Chunk = a piece of a document that we embed, store and retrieve as one unit.** Small chunks
  retrieve precisely and lose context. Large chunks keep context and blur the signal.
- Fixed-size chunking buried page 1,140's one SSO sentence in audit-log text, and the Scholar
  answered wrongly. Cutting at headings gave that sentence its own small chunk, and the answer
  was right.
- **The default:** structure-aware chunking plus title prepend, then late chunking if the model
  supports it, then contextual retrieval where the eval says it matters and the LLM pass is
  affordable.
- **Contextual retrieval** pastes LLM-written context in front of each chunk and substantially
  reduces retrieval failures. Prompt caching keeps its cost down.
- **Late chunking** embeds the whole document first, then slices the token vectors. It is nearly
  free and resolves pronouns and references.
- **Parent-child chunks**: search small children, hand the Scholar the parent section.
- Sweep chunk size against the eval set before tuning anything else.
- Filter empty chunks. Do not split tables or functions by character count.

### What's next

[Chapter 51](./51-metadata-and-structure.md) covers the large share of retrieval quality that has
nothing to do with vectors at all: titles, dates, permissions, and the links between pages.

We now know what a chunk is, how a careless cut can hide the one sentence that matters, and which
techniques keep every chunk a complete thought.
