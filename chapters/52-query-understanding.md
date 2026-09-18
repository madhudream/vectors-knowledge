---
title: "Query Understanding"
chapter: 52
part: "Part VII — RAG"
slug: query-understanding
readingTime: "11 min"
summary: "Users write short, ambiguous, context-dependent questions. Rewriting, decomposition, HyDE and multi-query turn them into something a retriever can actually use."
tags: [query-rewriting, hyde, multi-query, decomposition, rag]
prev: 51-metadata-and-structure
next: 53-context-assembly
---

# Query Understanding

**The one-paragraph version.** The question is usually the worst-written text in the whole system:
three words, a typo, or a pronoun pointing at something said five messages ago. So before we
retrieve, we transform it. **Rewriting** turns a follow-up message into a standalone question.
**Decomposition** splits a question into parts that each need their own pages. **Multi-query
expansion** searches with several phrasings and fuses the results. **HyDE** embeds a made-up
answer instead of the question. Each one fixes a specific failure, and each one costs an LLM
call, so we apply them where they help, not everywhere.

In this chapter, we will learn how to turn the message a user types into the searches the
Librarian should actually run. We will watch a follow-up question retrieve the wrong pages, then
the right one after rewriting. We will split our SSO question into two sub-questions, one for
each page it needs. We will also see when to skip all of this.

We will cover the following:

- What is query understanding
- Why we need query understanding
- Conversational rewriting
- Decomposition
- Multi-query expansion
- HyDE
- Metadata and intent extraction
- When to use which one

---

## What is query understanding

**Query understanding = turning the user's message into one or more better search queries,
before the Librarian searches.**

Think of it like a good support agent at Acme. A customer writes just *"SSO?"*. A good agent does
not type "SSO?" into the search box. They ask back: *"Do you want to know whether your plan
includes it, how to set it up, or why your login is failing?"*

- The customer's message is the raw query.
- The agent's clarifying thought is query understanding.
- The search the agent finally runs is the transformed query.

Librarians have a name for that clarifying step. They call it the **reference interview**. The
principle behind it is simple: **the question as asked is rarely the question as meant.** Query
understanding is the automated reference interview.

---

## Why we need query understanding

Here is a real-looking conversation with Acme's support assistant:

```
User:  Does the Pro plan include single sign-on?
Bot:   Yes, single sign-on is included on the Pro plan [1].
User:  what about for small teams?
```

Let's first see what happens if we search with the last message exactly as typed.

We embed *"what about for small teams?"*. It has no topic. It does not say SSO, Pro or plan. Its
dot lands near pages about team size in general, such as "Adding seats to your team" and "Team
roles and permissions". The Librarian returns those.

The Scholar thinks like this: *"The sources explain how small teams add seats."* It writes:
*"Small teams can add seats at any time from the Billing page."*

The answer is wrong. The customer was asking about SSO for small teams, and page 1,140 was never
found. The meaning of the question lived entirely in the conversation, and the search never saw
the conversation.

Now let's rewrite the message before searching. A small LLM reads the conversation and produces:

```
"what about for small teams?"
   →  "Does single sign-on on the Pro plan require anything extra for small teams?"
```

Now the query mentions single sign-on, Pro and small teams. Page 1,140 is the top hit. The Scholar
writes: *"On the Pro plan, teams under 50 seats need the Security add-on to use single sign-on
[1]."* The answer is correct.

In simple words, the search can only use the words it is given. Query understanding makes sure
those words carry the whole meaning.

---

## Conversational rewriting

**Query rewriting = using an LLM to turn a message that depends on the conversation into a
standalone search query.**

**Step 1:** Take the conversation history and the latest message.

**Step 2:** Ask a small, fast LLM to rewrite the message so it makes sense on its own, resolving
every pronoun and reference.

**Step 3:** Search with the rewritten query.

**Step 4:** Log both the original and the rewritten query.

Here is a prompt that does Step 2:

```python
REWRITE = """Given the conversation, rewrite the final user message as a
standalone search query. Resolve all pronouns and references. Keep codes, quoted
strings and identifiers exactly as written. Output only the query.

Conversation:
{history}

Final message: {message}"""
```

Means, we hand the model the chat so far and the new message, and ask for one self-contained
search query back, with codes such as `SSO-4012` left untouched. Nothing else.

**This is the highest-value transformation in this chapter** for any chat system, and it is
cheap, because a small, fast model handles it well. If you do only one thing from this chapter,
do this.

---

## Decomposition

Some questions need more than one page, and the pages talk about different things.

**Decomposition = splitting one question into smaller sub-questions, and retrieving for each one
separately.**

Our running question is exactly this kind. Page 212 is the Pro plan page. Page 1,140 is a help
article about an add-on, mostly about audit logs. They sit in different parts of the Map Room.

Let's first see a single search. The one vector for *"Does the Pro plan include single sign-on?"*
lands closest to page 212 and the other plan pages. Page 1,140's SSO chunk comes 14th, as in
Chapter 51, outside the top 5. The Scholar writes *"Yes, SSO is included on Pro."* That is half an
answer, and wrong for every team under 50 seats.

Now let's decompose. An LLM splits the question into two parts:

```
"Does the Pro plan include single sign-on?"
   →  1. "Which plans include single sign-on?"
      2. "Are there conditions or extra requirements for single sign-on on the Pro plan?"
```

Sub-question 1 retrieves page 212. Sub-question 2 retrieves page 1,140, because it asks about
conditions and extra requirements, and page 1,140's one SSO sentence states exactly such a
condition. We give the Scholar both result lists. It writes: *"Yes, SSO is on Pro [1], but teams
under 50 seats need the Security add-on [2]."* The answer is correct.

In simple words, one vector cannot point at two different places at once (Chapter 34's
multi-constraint failure). Two sub-questions can. Each one is a clean, single-topic search, which
is exactly what embeddings are good at.

Decomposition matters most for one special kind of question.

**Multi-hop question = a question where one fact is needed before we can even search for the
next.**

Take *"Does Globex need the Security add-on for SSO?"*. The first hop finds Globex's contract,
pages 3,507 and 3,508, which show Globex is on Pro with 120 seats. Only then can the second hop ask
the right thing: does a Pro team of 50 or more seats need the add-on? Page 1,140 says no. We
cannot write the second search until the first one comes back, so the hops must run one after
another. Chapter 56 turns that loop into an agent.

Decomposition helps most on comparisons, multi-hop questions and questions about several things
at once. It helps not at all on simple lookups, where it just wastes a call. So we **route**:
classify the question first, and decompose only when needed.

---

## Multi-query expansion

**Multi-query expansion = asking an LLM for several phrasings of the question, searching with
each one, and fusing the result lists with RRF (Chapter 40).**

Take our paraphrase twin, which shares almost no words with the pages that answer it:

```
"Can our team log in with our company accounts?"
   →  "single sign-on setup"
      "SAML login with an identity provider such as Okta"
      "log in with a company Google or Microsoft account"
```

The consensus property of RRF does the work. Pages about single sign-on rank well in all four
lists, so they rise to the top. The original words "company accounts" also match a billing page
called "Managing your company account details". That page appears near the top of only one list,
so fusion pushes it down.

Put simply, pages that match the *intent* show up under many phrasings. Pages that matched one
phrasing's wording by accident do not. It is a direct attack on vocabulary mismatch, and it is
essentially Chapter 8's classical query expansion with an LLM writing the synonyms.

The cost is N searches plus one LLM call. The searches run in parallel, so they add little waiting
time. The LLM call that writes the phrasings is the real cost, often more than half a second before
the first search starts.

---

## HyDE

This is the cleverest idea in this chapter, and the one that most needs care.

**HyDE = Hypothetical Document Embeddings: ask an LLM to write a plausible answer, and search with
the embedding of that answer instead of the question.**

The problem it solves is that a question and its answer often do not look alike (Chapter 14).
*"Can our team log in with our company accounts?"* shares almost no surface form with *"SSO is
included on the Pro plan."*

**Step 1:** Ask an LLM to write a short, plausible answer, with no retrieval. It may be wrong.

**Step 2:** Embed that hypothetical answer.

**Step 3:** Search with its vector.

```
query:          "Can our team log in with our company accounts?"
hypothetical:   "Yes. Single sign-on (SSO) lets your team log in with your company's
                 identity provider, such as Okta or Azure AD. It is usually available
                 on business plans and is set up by an admin in the security settings..."
embed the hypothetical → search
```

The hypothetical answer *looks like* a help page. Answer-shaped text lands near answer-shaped
pages. The specific facts may be wrong ("usually available on business plans"), and that does
not matter. Retrieval only needs the vector to land in the right neighbourhood. The real page
supplies the real facts.

**Where HyDE helps:** asymmetric retrieval with a weak or symmetric embedding model, short queries
against long documents, and new domains with no training data.

**Where it hurts:**

- **Out-of-knowledge domains.** The LLM has never heard of Acme's Security add-on. Its
  hypothetical answer is generic, and it pulls retrieval toward generic SSO content.
- **Identifier queries.** Ask *"What does SSO-4012 mean?"* and the LLM may invent *"SSO-4012
  means the certificate is invalid"*. That pulls retrieval toward certificate pages. The real
  meaning, "SAML assertion expired", is on a page the hypothetical never resembled.
- **Modern asymmetric models.** Models trained with query and passage prefixes (Chapter 14)
  already solve the problem HyDE addresses, so the gain shrinks or disappears.

**Measure it.** HyDE is the query technique most likely to help a naive system and least likely to
help a well-built one.

---

## Metadata and intent extraction

Chapter 51's Ninja notes covered this. It belongs in the same stage, so here it is again:

```
"Pro plan SSO pricing in 2024"  →  query:  "Pro plan single sign-on pricing"
                                   filter: { year: 2024, doc_type: "pricing" }
```

Means, we pull the date and the page type out of the words and turn them into exact filters. The
rest of the text becomes the search query.

Converting plain-language conditions into exact filters beats hoping the embedding notices them.
It is often the single biggest precision gain for business-document corpora.

**Note:** This question asks about 2024, so it *wants* the old, superseded pricing page. The
extracted filter must also switch off the usual `status = "active"` default from Chapter 51.

---

## When to use which one

| Technique | What it fixes | Extra cost | Skip it when |
|---|---|---|---|
| Conversational rewriting | Follow-ups that depend on the chat | One small LLM call | There is no chat history |
| Decomposition | Multi-part, comparative and multi-hop questions | One LLM call plus N searches (multi-hop: one LLM call and one search per hop, in sequence, plus a final LLM call to stop, Ch. 56) | The question is a simple lookup |
| Multi-query expansion | Users' words differ from the pages' words | One LLM call plus N parallel searches | The query is an identifier |
| HyDE | Questions that look nothing like answers | One LLM call | The model is asymmetric, or the topic is private jargon |
| Metadata extraction | Dates, page types and names hidden in the question | A regex or a small model | Pages carry no usable metadata |

We must use **rewriting** in every chat product. We must use **decomposition** when the answer
lives on pages about different things, as our SSO answer does. We must use **multi-query
expansion** when users describe things in their own words. We must use **HyDE** only after
measuring that it helps. We must use **metadata extraction** whenever questions mention dates,
page types or names.

Many strong systems use all of these, but never all at once. They classify each question first and
run only the transformations it needs.

---

### Under the hood

A routed query-understanding stage, so each technique runs only where it helps:

```python
def understand(message, history):
    q = rewrite_standalone(message, history) if history else message

    intent = classify(q)   # lookup, compound, multi_hop, exploratory or identifier
    filters = extract_filters(q)

    if intent == "identifier":
        return [q], filters, {"sparse_weight": 2.0}      # lean on BM25
    if intent == "compound":
        return decompose(q), filters, {}                 # independent sub-questions
    if intent == "multi_hop":
        return [q], filters, {"multi_hop": True}         # hops run in order, below
    if intent == "exploratory":
        return [q] + paraphrase(q, n=3), filters, {}
    return [q], filters, {}                              # simple lookup: do nothing

def retrieve(message, history):
    queries, filters, opts = understand(message, history)
    if opts.get("multi_hop"):
        return multi_hop(queries[0], filters)
    lists = parallel_map(lambda s: hybrid_search(s, filters, **opts), queries)
    return rrf(lists)

def multi_hop(q, filters, max_hops=3):
    hits = []
    for _ in range(max_hops):
        # one LLM call: what is still missing, and which filters fit that search?
        next_q, hop_filters = write_next_search(q, hits, filters)
        if next_q is None:                        # the hits already answer q
            break
        hits += hybrid_search(next_q, hop_filters)   # one search, then the next hop
    return hits
```

In simple words, we rewrite the message if there is a conversation, sort it into one of five
kinds, and pick a path. *"What does SSO-4012 mean?"* is an identifier, so we lean on keyword
search. Our SSO question is compound, so it gets decomposed, and its two independent sub-questions
are searched in parallel. *"Does Globex need the Security add-on for SSO?"* is multi-hop, so
`multi_hop` runs one LLM call and one search per hop, in order, then one last call to stop. The
contract search must come back before the second search can be written. Each hop also picks its own
filters. The first hop searches with the Globex filter from the question. Page 1,140 is not a
Globex page, so the second hop drops that filter. The other paths end in hybrid search and RRF.

The classification step can be a small model, a fine-tuned classifier, or a few heuristics.
Identifier detection is a regex. What matters is that **simple queries skip the expensive
paths.** Most traffic is simple.

---

### What people get wrong

**Applying every technique to every query.** Four LLM calls before retrieval turn a 300 ms system
into a 3-second one, mostly to rephrase *"password reset"*.

**No conversational rewriting in a chat product.** Follow-up questions fail silently, and users
conclude the system "forgets".

**Trusting HyDE blindly.** On internal jargon it actively misleads.

**Rewriting identifiers.** An LLM that "helpfully" normalises `SSO-4012` into *"SAML login error"*
has destroyed the exact-match signal, and now `SSO-4021` matches just as well. Preserve quoted
strings, codes and identifiers verbatim.

**Not logging the rewritten query.** When retrieval fails, you need to know what was *actually*
searched. Log original and transformed queries together, always.

---

### Ninja notes

**Search with the rewrite, but also with the original.** Rewriting occasionally distorts meaning.
Retrieving for both and fusing with RRF costs one extra search and protects you from the rewrite's
failures. Cheap insurance.

**Step-back prompting** is a useful complement to decomposition. Generate a *more general* version
of the question, such as *"how do Acme's plans and add-ons fit together?"* for our specific SSO
question, and retrieve for that too. It pulls in background and principles that the specific
query misses, and it helps the Scholar reason rather than just look things up.

**The logical end of this chapter is Chapter 56.** Every technique here is a fixed transformation
applied before a single retrieval. An agent does the same things adaptively. It searches, reads
the results, notices what is missing, and reformulates. Decomposition becomes a plan. Multi-query
becomes iteration. HyDE becomes "search, read, search again with better terms". If you find
yourself building an elaborate fixed query-understanding pipeline, that is usually the signal to
consider letting the model drive retrieval instead.

---

### Key takeaways

- **Query understanding = turning the user's message into one or more better search queries,
  before the Librarian searches.** The question as asked is rarely the question as meant.
- **Conversational rewriting** turned *"what about for small teams?"* into a standalone SSO
  question and found page 1,140. It is the highest-value technique for any chat product.
- **Decomposition** split our SSO question into two sub-questions, one per page. **Multi-hop**
  questions need one fact before the next search can even be written, so their hops run in order.
- **Multi-query expansion** plus RRF attacks vocabulary mismatch through consensus.
- **HyDE** embeds a hypothetical answer. It helps weak or symmetric models, and hurts on private
  jargon and identifiers. Measure it.
- Extract dates, page types and names into exact filters.
- Route by intent so simple queries skip expensive steps. Preserve identifiers verbatim, and log
  the rewritten query.

### What's next

Retrieval has returned its candidates. [Chapter 53](./53-context-assembly.md) decides what actually
goes into the prompt, in what order, and within what budget.

We now know why the typed question is rarely the right search, and how rewriting, decomposition,
expansion, HyDE and filters each close a different part of that gap.
