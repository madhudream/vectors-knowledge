---
title: "Context Assembly"
chapter: 53
part: "Part VII — RAG"
slug: context-assembly
readingTime: "11 min"
summary: "Retrieval found the right material. Now decide how much of it to send, in what order, with what labels, and how to stop the Scholar answering from its own memory instead."
tags: [context, prompt, lost-in-the-middle, citations, token-budget]
prev: 52-query-understanding
next: 54-evaluating-rag
---

# Context Assembly

**The one-paragraph version.** Between the reranker and the LLM sits a step that most pipelines
treat as gluing strings together. It deserves more. We remove duplicates, expand chunks into
useful units, and keep to a token budget. We order the sources deliberately and label every source
so it can be cited. And we tell the model exactly what to do when the sources are not enough. Small
changes here move answer quality more than you would expect.

In this chapter, we will learn what actually goes into the prompt, in what order, and why. We will
watch a carelessly glued prompt bury page 1,140 in the middle and lose its caveat. Then we will
assemble the same material properly and get the full answer. Along the way we will meet the
system prompt, the "lost in the middle" effect, and prompt caching.

We will cover the following:

- What is context assembly
- Why we need context assembly
- How context assembly works
- Choosing what goes in
- Arranging and labelling it
- Instructing the Scholar
- An example of context assembly
- Problems with context assembly

---

## What is context assembly

**Context assembly = choosing, cleaning, ordering and labelling the retrieved passages before they
go into the prompt.**

Think of it like preparing a briefing for a busy executive before a meeting.

- The executive is the Scholar.
- The forty pages of material on our desk are what the Librarian retrieved.
- The ten minutes the executive has are the token budget.
- The briefing pack we hand over is the assembled context.

We would not staple forty pages together. We would remove duplicates and put the most important
material where it will be read. We would label each source so claims can be checked, and make each
excerpt understandable on its own. And we would flag clearly if a likely question is not covered.

That is context assembly. One more term before we start.

**System prompt = the standing instructions sent with every request.** It comes first, before
anything specific to one question. For Acme, it says things like "You are Acme's support
assistant. Answer only from the sources provided." In simple words, it is the part of the prompt
that never changes from one customer to the next.

---

## Why we need context assembly

A customer asks: *"Does the Pro plan include single sign-on?"*

The reranker (Chapter 41) hands us its top 12 chunks. Unlike the fixed-size chunks of Chapters 51
and 52, this pipeline indexes small child chunks of up to about 200 tokens. Each child's parent is
its section, or the whole page when the page is short (Chapter 50's parent-child chunks). Let's
first see what happens if we simply glue them together in rank order, with no labels and no
instructions.

```
 1. page 212             "SSO is included on the Pro plan."
 2. pricing FAQ          quotes page 212 word for word
 3. page 212             an overlapping chunk, same sentence again
 4. Okta setup guide
 5. Azure AD setup guide
 6. "Team roles and permissions"
 7. page 1,140           "...On Pro, SSO needs the Security add-on for teams
                          under 50 seats."
 8–12. near-copies of the Okta and Azure AD guides
```

Look at what the Scholar receives. The sentence "SSO is included on the Pro plan" appears three
times, right at the top. The caveat appears once, in position 7 of 12, in the middle of the pile.
It arrives as a bare fragment with no title or date, so nothing tells the Scholar that it is a
separate page with its own rule.

The Scholar thinks like this: *"The sources say, again and again, that Pro includes SSO. Two of
them explain setup."* It writes: *"Yes, SSO is included on the Pro plan. Follow the Okta or Azure
AD guide to set it up."*

The answer is wrong for every team under 50 seats. The right page *was* retrieved. It simply got
drowned out, and with no labels the customer cannot even check where the answer came from.

Put simply, retrieval finding the page is not enough. The page also has to arrive in a form the
Scholar will actually use.

---

## How context assembly works

Context assembly is six steps, run in this order:

**Step 1:** Remove duplicates.

**Step 2:** Expand chunks into useful units.

**Step 3:** Keep to a token budget.

**Step 4:** Order the sources deliberately.

**Step 5:** Label every source.

**Step 6:** Instruct the Scholar precisely.

The next three sections walk through them.

---

## Choosing what goes in

**Step 1: Remove duplicates.**

Retrieval returns near-duplicates all the time: the same paragraph in two versions of a page,
overlapping chunks (Chapter 50), a policy quoted in three places. In our pile, positions 1, 2 and
3 all say the same thing.

Each duplicate costs tokens and **crowds out a distinct source.** Five copies of one fact is still
one fact.

```python
def dedupe(hits, threshold=0.97):
    kept = []
    for h in hits:
        if all(float(h.vec @ k.vec) < threshold for k in kept):
            kept.append(h)
    return kept
```

In simple words, we walk down the list and keep a hit only if its vector is not too close to any
hit we already kept. The vectors are normalized, so the dot product is the cosine similarity
(Chapter 5).

We also cap the number of chunks per page (Chapter 51). If the corpus is highly repetitive, we can
apply **MMR** (maximal marginal relevance, a re-ordering that skips results too similar to ones
already picked, Chapter 41).

**Step 2: Expand chunks into useful units.**

A 200-token chunk retrieved precisely may be missing what makes it understandable. So we expand:

- **To the parent**, which is retrieve small, pass large (Chapter 50). Page 1,140 is short, so its
  caveat chunk expands to the whole page, title and headings included.
- **To neighbouring chunks**, because the paragraph before often defines the terms.
- **To neighbouring pages**, for page-image retrieval, because tables span pages (Chapter 47).

Then we deduplicate again, since expansion creates overlaps. Three chunks with one parent
collapse into that parent, once. Finally we rerank the expanded items. A parent that now carries
its title and headings can score very differently from the bare fragment it grew from.

**Step 3: Keep to a token budget.**

We decide a budget and stick to it. Everything else competes for the same context window: the
system prompt, the conversation history, the instructions, and the answer itself.

```
context window:              200,000 tokens available
system prompt + instructions:    800
conversation history:          3,000
reserved for the answer:       2,000
retrieved context budget:     12,000   ← a deliberate choice, not "whatever fits"
```

Means, of 200,000 tokens available, we plan to use under 18,000. The retrieved sources get
12,000 of them, on purpose.

Now, the question is, why not fill the window? Chapter 49 gave the reasons. Irrelevant context is
a distractor, cost grows with every token, and the time before the answer starts grows with input
length. **Choose the budget by measuring answer quality at several sizes**, such as 4k, 8k, 16k
and 32k tokens, on the eval set. Quality typically levels off well before the window is full, and
often declines after that.

We fill the budget greedily, in rerank order. When the next item does not fit, we stop, and that
item is dropped whole. We do not cut an item in half. Page 1,140 shows why. Its caveat comes after
the audit-log section, in the middle of the page, so cutting at the budget line could keep the
audit logs and remove exactly the sentence that matters.

---

## Arranging and labelling it

**Step 4: Order the sources deliberately.**

Models do not pay equal attention across a long context.

**Lost in the middle = the finding that models use information at the start and the end of a long
context more reliably than information in the middle.** It is a widely replicated result. Modern
long-context models have reduced the effect considerably, but not eliminated it, and it is cheap
to account for:

```python
def order_for_attention(ranked):
    """Best items at both ends, weakest in the middle."""
    front, back = [], []
    for i, item in enumerate(ranked):
        (front if i % 2 == 0 else back).append(item)
    return front + back[::-1]
    # ranks [1,2,3,4,5,6] → [1,3,5,6,4,2]
```

In simple words, the best source goes first, the second-best goes last, and the weakest sources end
up in the middle, where they matter least.

**Place the question after the context**, not before it. The model then reads the sources and meets
the question right at the point of answering. For long contexts this measurably helps.

**Step 5: Label every source.**

Every source needs a stable ID it can be cited by, and enough detail to be checked:

```
<source id="5" title="Security add-on" updated="2026-01-22"
        url="https://help.acme.example/security-add-on">
## Audit logs
Every sign-in ...
## Plans and seats
Customers add it from the Billing page.
On Pro, SSO needs the Security add-on for teams under 50 seats.
## IP allow-lists
...
</source>
```

XML-style tags work well. They are unambiguous boundaries that models handle reliably. They also
keep source text visibly separate from our instructions, which helps against **prompt injection**
(instructions hidden inside a retrieved page that try to take over the model, Chapter 62).

Include the **date**. A model that can see two sources with different dates can reason about which
one is current. Without dates it cannot.

---

## Instructing the Scholar

**Step 6: Instruct precisely.**

```
Answer using only the information in the <source> tags.
- Cite every factual claim with its source id, like [3].
- If sources conflict, say so and prefer the most recently updated one.
- If the sources do not contain enough information to answer, say what is
  missing. Do not answer from general knowledge.
- Quote exact figures, dates and names as they appear in the sources.
```

Each line prevents a specific failure:

| Instruction | Failure it prevents |
|---|---|
| "only the information in the sources" | Blending retrieval with pretraining |
| "cite every claim" | Unverifiable answers |
| "if sources conflict…" | Silently picking the stale version |
| "say what is missing" | Confident fabrication on insufficient context |
| "quote exact figures" | Paraphrasing 14.5% into "about 15%" |

These instructions are the same for every question, so they belong in the system prompt, at the
very start. That placement also saves money, through prompt caching.

**Prompt caching = a provider feature that stores the start of a prompt for a while, so the next
request that begins with the same text costs less and answers sooner.**

Means, anything identical across requests should come first, and anything that changes should come
after it. Providers cache only prefixes above a minimum length, often about 1,024 tokens, so check
yours. Our 800-token system prompt is too short on its own.

So the full prompt runs in this order: the system prompt with its instructions, then the
conversation history, then the labelled sources, then the question.

---

## An example of context assembly

Let's take the same 12 chunks for our SSO question and run all six steps.

**Step 1:** Positions 1, 2 and 3 say the same thing. Deduplication keeps page 212 and drops the
other two. Positions 8 to 12 repeat the two setup guides, so they go too.

**Step 2:** Page 1,140 is short, so its caveat chunk expands to the whole page, under the title
"Security add-on" and the heading "Plans and seats". Then we rerank the expanded items. With its
title and headings now attached, page 1,140 rises from last to second, right behind page 212.

**Step 3:** After dedupe and a cap per page, five sources remain, about 3,900 tokens. They fit
inside the 12,000-token budget.

**Step 4:** In the new rerank order, the five sources are page 212, page 1,140, the Okta guide, the
Azure AD guide, and "Team roles". Ordering for attention gives: page 212, Okta guide, Team roles,
Azure AD guide, page 1,140. Page 1,140 now sits last, right before the question, instead of lost
in the middle.

**Step 5:** Each source gets an ID, title, date and URL. Page 212 becomes source 1. Page 1,140
becomes source 5.

**Step 6:** The system prompt carries the instructions.

The Scholar thinks like this: *"Source 1 says Pro includes SSO. Source 5, updated January 2026 and
titled 'Security add-on', adds a condition for teams under 50 seats. I must cite both."*

It writes: *"Yes, single sign-on is included on the Pro plan [1]. Teams under 50 seats also need the
Security add-on [5]."*

The answer is correct. The retrieved chunks did not change. Only the assembly did.

---

## Problems with context assembly

**Problem 1: Deduplication can delete a real difference.** Two versions of a pricing page can be
0.95 similar and still differ in the one number that matters. Filter superseded pages first
(Chapter 51), and keep the dedupe threshold above that, at 0.97 or higher, or dedupe only within
one page.

**Problem 2: Squeezing sources can drop a caveat.** Imagine a tool that keeps only the sentences
containing the question's key phrase, "single sign-on". It would keep the setup guides, which spell
the phrase out. It would drop "On Pro, SSO needs the Security add-on for teams under 50 seats",
because that sentence only says "SSO". Measure faithfulness (Chapter 54) before and after any
compression.

**Problem 3: Ordering tricks matter less on newer models.** The lost-in-the-middle effect is
shrinking. Ordering is still cheap, but check on the eval set how much it helps before relying on
it.

---

### Under the hood

The whole stage:

```python
from html import escape            # so a page cannot close its <source> tag early

def assemble(question, reranked, history, budget_tokens=12_000):
    items = dedupe(reranked)
    items = dedupe(expand_to_parents(items))    # section, or whole page if short
    items = rerank(question, items)             # expanded parents score differently
    items = cap_per_document(items, max_per_doc=2)

    chosen, used = [], 0
    for it in items:
        cost = count_tokens(it.text)
        if used + cost > budget_tokens:
            break                              # stop: drop the item that does not fit
        chosen.append(it); used += cost

    chosen = order_for_attention(chosen)
    sources = "\n\n".join(
        f'<source id="{i+1}" title="{escape(c.title)}" '
        f'updated="{c.updated_at}" url="{escape(c.url)}">\n'
        f'{escape(c.text, quote=False)}\n</source>' for i, c in enumerate(chosen))

    system = SYSTEM_PROMPT + "\n\n" + INSTRUCTIONS          # identical every time: cacheable
    user = (f"<history>\n{history}\n</history>\n\n"      # changes per question
            f"{sources}\n\nQuestion: {question}")
    return system, user, chosen
```

In simple words, this is Steps 1 to 6 in order. We dedupe, expand, dedupe again, rerank the
expanded items and cap per page. We fill the budget and stop at the first item that does not fit.
Then we order, label, and put the fixed instructions first and the question last. Escaping the
page text means a page that contains `</source>` cannot break out of its tag, and the history sits
in its own tags, apart from the sources.

Return `chosen` alongside the prompt. You need it to show citations to the user. Critically, you
also need it **to log exactly what context produced each answer.** When an answer is wrong, the
first question is always "was the right information in the context?". You can only answer it if
you logged the context.

---

### What people get wrong

**Concatenating raw chunks.** No labels, no dates, no dedup. Citations become impossible and
duplicates waste the budget.

**Filling the context window because it is there.** More context is not better past a point you
can measure.

**Question before context in long prompts.** Put it after.

**No "insufficient information" instruction.** The model will answer anyway, from pretraining,
fluently.

**Not logging assembled context.** It makes every quality investigation guesswork.

**Mixing conversation history into the source block.** History and sources should be clearly
separated. Otherwise the model may cite its own previous answer as a source, which is how a single
hallucination becomes a self-reinforcing one.

---

### Ninja notes

**Consider compressing context rather than dropping it.** Instead of dropping the eighth source
when the budget runs out, extract only the sentences from each source that are relevant to the
question. A small model, or a cross-encoder scoring individual sentences, can often cut context by
half or more while keeping the relevant evidence. You get more *distinct* sources into the same
budget. The risk is dropping a sentence that looked irrelevant but carried a caveat, exactly as in
Problem 2. Measure faithfulness (Chapter 54) before and after.

**Put stable content first to exploit prompt caching.** If your system prompt and instructions are
identical across requests, place them at the very beginning so providers can cache that prefix.
Retrieved sources vary per query and belong after it. At volume, this ordering alone is a
significant cost and latency saving, and it costs nothing.

**For image-based RAG, the same rules apply to pages.** Label each image with its document title,
page number and date. Deduplicate near-identical pages. Expand to neighbouring pages. Budget by
image count, since each page image uses a substantial number of tokens in a vision model.

---

### Key takeaways

- **Context assembly = choosing, cleaning, ordering and labelling the retrieved passages before
  they go into the prompt.**
- Glued together raw, the SSO chunks buried page 1,140 in the middle, and the Scholar left out the
  caveat. Assembled properly, the same chunks gave the full, cited answer.
- Deduplicate, expand to understandable units, deduplicate again, then rerank the expanded units.
- Set an explicit token budget, chosen by measuring answer quality, not by filling the window. When
  an item does not fit, drop it whole.
- Order the best items at the start and end (**lost in the middle**), and put the question after
  the context.
- Label every source with ID, title, date and URL, using clear delimiters.
- Instruct explicitly in the **system prompt**: sources only, cite everything, handle conflicts,
  admit gaps, quote exactly. Keeping it identical lets **prompt caching** cut cost.
- Log the assembled context for every answer.

### What's next

[Chapter 54](./54-evaluating-rag.md) builds the evaluation that tells us whether any of this is
actually working.

We now know that finding the right page is only half the job, and how six small steps make sure
the Scholar actually reads it.
