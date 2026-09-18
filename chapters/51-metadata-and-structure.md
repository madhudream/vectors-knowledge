---
title: "Metadata and Structure"
chapter: 51
part: "Part VII — RAG"
slug: metadata-and-structure
readingTime: "11 min"
summary: "A large share of retrieval quality comes from things that are not vectors at all: titles, dates, permissions, document type, and the links between documents."
tags: [metadata, structure, filtering, freshness, permissions, graph]
prev: 50-chunking
next: 52-query-understanding
---

# Metadata and Structure

**The one-paragraph version.** Vectors capture what a chunk *says*. Metadata captures what a chunk
*is*: which page it came from, which version, when it was written and by whom, and who may see it.
Many real questions are partly questions about metadata, and many retrieval failures are really
metadata failures. Get this layer right and we fix more problems than another embedding model
would.

In this chapter, we will learn what metadata is, what it can do that vectors cannot, and how the
links between pages help the Librarian find the second half of an answer. We will watch an
out-of-date pricing page fool vector search, and watch a single metadata field fix it.

We will cover the following:

- What is metadata
- Why we need metadata
- What metadata does for us
- Structure between documents
- Where metadata comes from
- Metadata vs vectors
- When to use which one

---

## What is metadata

**Metadata = facts about a chunk that are not part of its text, such as where it came from, when
it changed, and who may see it.**

Think of it like the record card that sits behind every book in a real library.

- The book itself is the chunk's text.
- The card lists the title, the shelf, the edition and the date. That is the metadata.
- A stamp on the card saying "staff only" is the permission.
- A note saying "replaced by the 2025 edition" is the version status.

Every chunk in our Card Catalog should carry a record like this one, for a chunk of page 1,140:

```python
{
  "chunk_id":    "kb_1140#c02",
  "doc_id":      "kb_1140",
  "title":       "Security add-on",
  "section":     "Plans and seats",
  "breadcrumb":  "Help Center > Plans and billing > Security add-on > Plans and seats",
  "url":         "https://help.acme.example/security-add-on#plans-and-seats",
  "doc_type":    "help_article",
  "author":      "billing-docs",
  "created_at":  "2024-03-11",
  "updated_at":  "2026-01-22",
  "version":     7,
  "status":      "active",            # active | superseded | draft | archived
  "language":    "en",
  "acl":         ["group:all-customers"],
  "prev_chunk":  "kb_1140#c01",
  "next_chunk":  "kb_1140#c03",
  "parent":      "kb_1140",           # a short page is its own parent (Chapter 50)
}
```

Most fields explain themselves. Two need a word.

**ACL = access-control list: who may see this document.** Here, any Acme customer may see it. An
internal sales playbook would carry `["group:acme-staff"]` instead.

The **breadcrumb** is the path of headings from the top of the help center down to this chunk.

In simple words, the text tells the Librarian what a chunk is about. The metadata tells the
Librarian whether the chunk is current, allowed, citable, and connected to anything else. Every
field earns its place, and we will see how.

---

## Why we need metadata

Acme changed its plans in 2025. Before that, single sign-on was Enterprise-only. The old pricing
page was never deleted. It is still in the Great Library as page 38,902, and it says:

> "Single sign-on is available on the Enterprise plan only. It is not included on Pro."

Now a customer asks our running question: *"Does the Pro plan include single sign-on?"*

To show what metadata adds on its own, the examples in Chapters 51 to 56 start from the plain
fixed-size chunks at the start of Chapter 50, before any of its fixes, unless a chapter says
otherwise. The ranks are illustrative.

Let's first see what vector search does on its own. Page 38,902 is about exactly the same topic
as page 212, in almost the same words. It even mentions "Pro" and "single sign-on" together. In
the Map Room, its dot sits right next to page 212's dot. The Librarian brings back page 38,902 in
first place, with page 212 just behind it.

The Scholar sees two sources that disagree. It thinks like this: *"One source says SSO is
included on Pro. The other says, very explicitly, that it is not. The explicit one sounds more
authoritative."* It writes: *"No. Single sign-on is only available on the Enterprise plan."*

The answer is wrong. And no better embedding model would have fixed it. **The superseded page is
semantically almost identical to the current one.** Whether a page is still true is not written
anywhere in its text.

Now let's add one field. At index time, page 38,902 was marked `status = "superseded"` when the
new pricing page replaced it. At query time, we filter on `status = "active"` before ranking.

Page 38,902 never reaches the Scholar. The Librarian brings back page 212 first, and the Scholar
writes: *"Yes, SSO is included on Pro."* The contradiction is gone, and the Scholar no longer says
SSO is Enterprise-only. The caveat for teams under 50 seats is still missing, because page 1,140
did not make the top 5. We fetch it by following links, later in this chapter.

Put simply, similarity cannot tell old from current. One metadata field can.

---

## What metadata does for us

**1. Filtering (Chapter 33).** This is what we just did. `status = "active"` removes a whole class
of confident wrong answers that no amount of semantic similarity will prevent.

**2. Access control (Chapter 62).** Filter by `acl` before retrieval. If the filter runs after
retrieval, restricted pages have already been fetched, and one bug or one skipped check puts them
in front of the wrong customer. So this must be a pre-filter or a partition, never a
post-filter. At Acme, the internal sales playbook about Security add-on discounts must never
reach a customer's prompt.

**3. Recency.** For anything operational, newer is usually better. So we boost by age instead of
cutting old pages off:

```python
def recency_boost(score, updated_at, now, half_life_days=180):
    age = (now - updated_at).days
    return score * (0.5 ** (age / half_life_days))
```

In simple words, every 180 days a page's score is halved. A page 180 days old keeps half its
score, and a page 360 days old keeps a quarter.

Here is why it matters. A two-year-old community forum post says *"SSO is Enterprise-only"*.
Forum posts have no status field, so the filter above cannot catch it. Its similarity to our
question is 0.84, higher than page 212's 0.80. After the boost, the post drops to
0.84 × 0.5^(730/180) ≈ 0.05. Page 212, updated 30 days ago, keeps 0.80 × 0.5^(30/180) ≈ 0.71.
Page 212 wins.

A half-life beats a hard cutoff, because a slightly older but far more relevant page can still
win.

**Note:** Multiplying the whole score is strong medicine. A two-year-old page keeps only about 6%
of its score. That is right for release notes and forum posts, and wrong for a contract that is
still in force. Apply the decay only to document types where freshness matters, or soften it by
blending, for example `score * (0.7 + 0.3 * decay)`.

**4. Citations.** `title` and `url` are what make an answer checkable. Without them the system is
a black box that nobody can audit (Chapter 49).

**5. Better embeddings, the underrated one.** **Prepend the title and section heading to the chunk
text before embedding it.**

```python
text_to_embed = f"{title} > {section}\n\n{chunk_text}"
```

Means, we paste the page title and section heading in front of the chunk, and embed the result.
A chunk from page 1,140 that begins *"It also records every export and permission change"* is
ambiguous on its own. With `"Security add-on > Audit logs"` in front, its vector now encodes what
"It" is. This is the cheapest possible version of contextual retrieval
(Chapter 50). It needs zero LLM calls and one line of code, and it works.

**6. Deduplication and diversity.** `doc_id` lets us cap how many chunks come from one page. Five
chunks from a single page cannot then crowd out four other relevant sources.

**7. Navigation.** `prev_chunk`, `next_chunk` and `parent` make parent-child retrieval
(Chapter 50) and neighbour expansion possible.

---

## Structure between documents

Pages are not independent, and the links between them are a retrievable signal too.

- A new pricing page **supersedes** an older one.
- A help article **references** another article ("see Security add-on").
- A release note **links to** the help article it changed.
- A support ticket **links to** the article that solved it, the way a ticket links to the commit
  that fixed it.
- A contract **cites** the price list it was signed against, the way a paper cites its
  predecessors.
- A slide deck for customers **summarises** a longer report.

Now, the question is, how do these links help our SSO question?

Page 212 carries a footnote: *"Seat limits apply. See Security add-on."* That is a reference link
to page 1,140.

Let's first see plain retrieval, on the same fixed-size chunks as before. Page 212 is the top hit,
and page 1,140's chunk sits at rank 14. That is still outside the top 5. The Scholar gets page 212
alone and writes *"Yes, SSO is included on Pro."* That is half an answer, and wrong for every team
under 50 seats.

Now let's follow the structure.

**Step 1:** At index time, store the links: supersedes, references, and neighbouring chunks.

**Step 2:** At query time, run the normal hybrid search.

**Step 3:** For each of the top 3 hits, follow its links. Fetch the latest version, up to 2
referenced pages, and the neighbouring chunks.

**Step 4:** Remove duplicates and pass everything on.

Page 212 is a top hit, so we follow its reference link and fetch page 1,140. Now the Scholar has
both pages, and the answer includes the Security add-on caveat. The answer is correct.

```python
def retrieve_with_structure(query, k=10):
    hits = hybrid_search(query, k)
    expanded = list(hits)
    for h in hits[:3]:
        expanded += fetch(h.supersedes_chain_latest)     # always the current version
        expanded += fetch(h.referenced_docs)[:2]         # what it depends on
        expanded += fetch(h.neighbours)                  # adjacent chunks
    return dedupe(expanded)
```

In simple words, vector search finds the *entry point*, and structure finds what that entry point
depends on. This is how a human researcher works: find one relevant thing, then follow its
references. It is dramatically underused in RAG systems.

It is also the honest, low-ceremony version of GraphRAG. **GraphRAG** usually means having an LLM
extract a graph of entities and relationships from the documents, then retrieving over that
graph. We do not need an extracted graph to benefit from links our documents already contain.

---

## Where metadata comes from

| Source | What you get | Effort |
|---|---|---|
| The filesystem / CMS (content management system) | path, dates, author, permissions | Free |
| Document structure | title, headings, breadcrumb, page number | Cheap parsing |
| The link graph | references, supersession, citations | Cheap, often free |
| An LLM at index time | doc_type, summary, entities, topics | One call per document |
| Human curation | authority, canonical-ness, deprecation | Expensive, highest value |

The first three are nearly free, and most teams use only the first. Start there.

The LLM-extracted layer is worth its cost for one thing in particular: **classifying document type
and status**. Knowing that a page is a draft, an outdated internal how-to or a meeting note, rather
than a published help article, lets us weight and filter in ways that measurably improve answers.

---

## Metadata vs vectors

| Question the Librarian needs answered | Vectors | Metadata |
|---|---|---|
| What is this chunk about? | Yes | No |
| Does it match a paraphrase of the question? | Yes | No |
| Is it the current version? | No | Yes (`status`, `version`) |
| Who may see it? | No | Yes (`acl`) |
| How old is it? | No | Yes (`updated_at`) |
| How do we cite it? | No | Yes (`title`, `url`) |
| What other pages does it depend on? | No | Yes (links) |

---

## When to use which one

We must use **vectors** to find what a chunk says, especially when the question uses different
words from the page.

We must use **metadata** to decide whether a chunk is current, allowed, recent enough and
citable, and to follow it to the pages it depends on.

Every strong system uses both. Metadata filters and boosts, vectors rank, and structure expands.

---

### Under the hood

Metadata-aware scoring, combining everything above:

```python
DECAYING_TYPES = {"forum_post", "release_note"}         # where freshness matters

def score_with_metadata(hits, query_meta, now):
    out = []
    for h in hits:
        s = h.score
        if h.status == "superseded":  continue          # hard filter
        if h.status == "draft":       s *= 0.5
        if h.doc_type in DECAYING_TYPES:                # recency half-life
            s *= 0.5 ** ((now - h.updated_at).days / 180)
        if h.doc_type == query_meta.get("preferred_type"): s *= 1.2
        if h.language != query_meta.get("language", h.language): s *= 0.8
        out.append((h, s))

    # cap chunks per document for diversity
    seen, final = {}, []
    for h, s in sorted(out, key=lambda x: -x[1]):
        if seen.get(h.doc_id, 0) >= 2:  continue
        seen[h.doc_id] = seen.get(h.doc_id, 0) + 1
        final.append((h, s))
    return final
```

In simple words, we drop superseded pages, halve drafts, decay forum posts and release notes by
age, nudge the preferred page type up and other languages down, then keep at most two chunks per
page. For our question, page 38,902 is dropped on the first line, and the two-year-old forum post
decays to almost nothing. Pages 212 and 1,140 are not decaying types, so they keep their full
scores.

None of that is machine learning. All of it is measurable, explainable, and adjustable without
retraining anything, which is exactly why it is such good engineering leverage.

---

### What people get wrong

**Storing only the vector and the text.** Then you cannot filter, cite, deduplicate, expand, or
apply access control. Everything above becomes impossible.

**Post-filtering on permissions.** A correctness *and* security bug. Filter before, or partition by
tenant (Chapter 33).

**Not embedding the title.** One line, free, and it meaningfully improves retrieval on ambiguous
chunks.

**Hard recency cutoffs.** *"Only documents from the last 90 days"* silently excludes the
authoritative document written two years ago. Decay, do not cut.

**Ignoring superseded content.** The most dangerous failure in the whole of Part VII, because the
wrong answer is indistinguishable from the right one in embedding space.

**Letting metadata go stale.** If `updated_at` is set at ingestion rather than taken from the
source, your recency logic is measuring your pipeline, not your documents.

---

### Ninja notes

**Query-time metadata inference is worth building.** Many questions contain metadata conditions in
plain language that nobody extracts:

```
"what did the Pro plan include in 2024"   → time filter: 2024
"the setup guide for SSO"                 → doc_type: guide, topic: sso
"who signed the Globex contract"          → doc_type: contract, entity: Globex
```

A small model, or a handful of regexes for the common cases, can turn these into real filters. The
gain is large because we convert a fuzzy semantic problem into an exact structured one, and exact
beats fuzzy whenever it is available. This is the same principle as hybrid search (Chapter 40),
applied to the metadata layer.

**Keep an authority signal.** In any corpus of real size, some documents are canonical and some
are someone's notes from 2021. If you have any signal for this (page views, inbound links, an
explicit "official" flag, the folder it lives in), encode it and use it as a multiplier. In
practice this is often the difference between a system that answers from the handbook and one
that answers from a stale Slack export, and no embedding model can tell those apart.

---

### Key takeaways

- **Metadata = facts about a chunk that are not part of its text**: where it came from, when it
  changed, and who may see it. An **ACL** (access-control list) says who may see a document.
- Vector search ranked the superseded pricing page first, and the Scholar said SSO is
  Enterprise-only. A `status = "active"` filter removed it, and that wrong answer went away.
- Store rich metadata with every chunk: identity, title, breadcrumb, dates, version, status,
  language, ACL and neighbour pointers.
- Metadata enables filtering, access control, recency, citations, dedup and navigation, none of
  which vectors can do.
- Prepend the title and section heading to chunk text before embedding. One line, real gain.
- Filter permissions *before* retrieval, always.
- Decay by recency with a half-life of about 180 days, rather than applying hard cutoffs, and only
  where freshness matters.
- Store and follow links between pages. Retrieval finds the entry point, structure finds the rest.
- Extract metadata conditions from the question, and keep an authority signal.

### What's next

So far we have improved the pages. [Chapter 52](./52-query-understanding.md) turns to the other
end: the user's question, which is often the worst-written text in the entire system.

We now know what metadata is, why similarity alone cannot tell a current page from a superseded
one, and how filters, boosts and links turn a good Librarian into a reliable one.
