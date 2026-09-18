---
title: "Vectors as Agent Memory"
chapter: 64
part: "Part IX — Ninja Tier"
slug: agent-memory
readingTime: "12 min"
summary: "Embedding every message and retrieving the nearest ones is the obvious way to give an agent memory, and it fails in predictable ways. What memory actually requires: extraction, consolidation, recency, and forgetting."
tags: [agents, memory, long-term-memory, consolidation, retrieval]
prev: 63-migration-and-drift
next: 65-the-frontier
---

# Vectors as Agent Memory

**The one-paragraph version.** The obvious way to give an agent a memory that lasts across
conversations is to embed every conversation turn and retrieve the most similar ones later. It
fails because memory is not a document corpus. Memory is **mutable** (facts change),
**contradictory** (old beliefs must be replaced), **sparse** (most turns are not worth keeping)
and **time-sensitive**. Good memory
systems *extract* durable facts and *consolidate* them against what is already known. They
retrieve with similarity, recency and importance together, and they deliberately *forget*.
Vectors are one component, not the design.

In this chapter, we will learn what agent memory is, watch the obvious approach give Globex's IT
admin a wrong answer, and build the pipeline that gets it right.

We will cover the following:

- What agent memory is
- Why "embed every turn" fails
- How a memory pipeline works
- An example: Dana's SSO question
- How to score memories at retrieval
- Problems with memory systems
- When to use which one

---

## What agent memory is

Acme's support assistant is an agent (Chapter 56). It answers customers' questions, searches the
knowledge base, and calls tools. Dana is the IT admin at Globex, and she talks to the assistant
every few weeks.

Every conversation starts from nothing unless we give the assistant a memory.

**Agent memory = what an agent keeps from past conversations and brings back into future ones.**

There are two broad kinds.

**Semantic memory** holds facts: *"Globex is on the Pro plan with 120 seats."* **Episodic memory**
holds events: *"On 4 July we helped Dana download Globex's June invoice."* In simple words,
semantic memory is what the assistant knows, and episodic memory is what happened.

Now, the question is, how should the assistant store these? There are two very different ways.

Think of it like two ways to remember a year.

**The diary** records everything, in order. *"February: Dana said Globex is on Basic with 30
seats. She also asked how to change her avatar."* Months later, a search for *"What plan is Globex
on?"* returns that February entry. It also returns a June entry that says *"We moved to Pro, 120
seats now."* Nothing tells us which one is current.

**The notebook** is curated. It has a page for Globex: *"Pro plan, 120 seats (since June 2026,
previously Basic with 30). Dana prefers numbered steps."* The avatar question is not in it. When a
fact changes, the page is updated, not appended.

Here is the mapping:

- The diary is storing every conversation turn as a vector.
- The notebook is a memory system that extracts and curates facts.
- Updating the notebook page is **consolidation**, which we define below.
- Looking something up is retrieval, in both cases.

Naive vector memory is a diary with a search box. What agents need is the notebook.

---

## Why "embed every turn" fails

Here is the diary approach answering a real question.

In September, Dana asks: *"Do we need the Security add-on for SSO?"*

The assistant embeds her question and retrieves the five most similar past turns. The top one is
from February: *"We're on Basic with 30 seats, and SSO isn't set up yet."* It mentions SSO, so it
is very similar. The June turn, *"We moved to Pro, 120 seats now"*, never mentions SSO, so it ranks
lower and misses the cut. A leftover fragment, *"Yes, the second one"*, takes a slot too.

The Librarian also fetches page 1,140: *"On Pro, SSO needs the Security add-on for teams under 50
seats."*

The Scholar reads all of it and writes: *"You're on Basic with 30 seats. You'll need to upgrade to
Pro, and with 30 seats you'll also need the Security add-on."*

The answer is wrong. Globex has been on Pro with 120 seats since June. It needs no add-on.

Now, why did it fail? There are five reasons, and this example shows most of them.

**Problem 1: Contradiction without resolution.** *"Basic, 30 seats"* in February and *"Pro, 120
seats"* in June can both be retrieved. Similarity cannot tell which is current. This is Chapter
51's superseded-page failure in its purest form.

**Problem 2: Noise drowns signal.** Most turns are passing remarks: *"thanks"*, *"can you make that
shorter?"*, *"try again"*. Embedded, they clutter the index and take retrieval slots.

**Problem 3: Fragments lack context.** *"Yes, the second one"* is meaningless outside the
conversation it came from. It is Chapter 52's conversational problem, stored permanently.

**Problem 4: Similarity alone is the wrong signal.** The most *similar* memory is often not the
most *useful*. Dana said once, months ago, *"Always give me numbered steps."* That may matter more
than a similar-sounding remark from yesterday.

**Problem 5: Nothing is ever forgotten.** Memory grows without limit, retrieval gets noisier, and
stale or sensitive information stays forever. That is a privacy problem as well as a quality one
(Chapter 62).

---

## How a memory pipeline works

**Consolidation = comparing a new fact with what the agent already remembers, and deciding whether
to add it, update an old memory, merge duplicates, or ignore it.**

The pipeline has three phases.

**Phase 1: Writing memories (after each conversation).**

- **Step 1: Extract.** An LLM reads the conversation and pulls out durable facts, preferences, decisions and commitments.
- **Step 2: Find candidates.** Vector search finds the 5 most similar existing memories for this customer.
- **Step 3: Consolidate.** An LLM looks at the new fact and those 5 memories, and picks one operation: ADD, UPDATE, MERGE or IGNORE.
- **Step 4: Store.** Save the memory's text, its embedding, and metadata: created, updated, last
  used, source, importance, subject and status (active or superseded).

**Phase 2: Reading memories (before each reply).**

- **Step 1:** Search this customer's *active* memories for about 50 candidates.
- **Step 2:** Score each candidate on similarity, recency and importance together.
- **Step 3:** Keep the top 8 and place them in the Scholar's context.
- **Step 4:** Refresh the recency of the memories just used.

**Phase 3: Forgetting (on a schedule).**

- **Step 1:** Let unused, low-importance memories decay and expire.
- **Step 2:** Honour explicit deletion requests, everywhere the memory is stored.

The same pipeline as a diagram:

```
conversation
   │
   ▼
EXTRACT       an LLM pulls durable facts, preferences, decisions, commitments
   │            "Globex moved from Basic (30 seats) to Pro (120 seats) in June 2026."
   ▼
CONSOLIDATE   compare with existing memories (vector search finds candidates)
   │            → ADD new memory
   │            → UPDATE existing (the Basic fact is superseded, history kept)
   │            → MERGE near-duplicates
   │            → IGNORE (already known, or not durable)
   ▼
STORE         text + embedding + metadata: created, updated, last used, source,
   │            importance, subject/entity, status (active | superseded)
   ▼
RETRIEVE      score = f(similarity, recency, importance), active memories only
   │
   ▼
FORGET        decay unused low-importance memories; honour explicit deletion
```

Three things stand out.

**Vector search appears twice.** Once in consolidation (*"do I already know something about
this?"*) and once in retrieval. It is essential both times, and it is not enough either time.

**Consolidation is the step that makes it memory rather than logging.** It is where contradictions
get resolved. It needs an LLM's judgement: given the new fact and the most similar existing
memories, which operation applies?

**Memories are text first, vectors second.** The stored thing is a clean, self-contained
statement, such as *"Dana prefers numbered steps"*, written by an LLM. The embedding is just an
index over it. This is the reverse of naive memory, where the raw turn is stored and embedded as
it is.

---

## An example: Dana's SSO question

Let's run the same September question through the pipeline.

**In June,** Dana's conversation ends. Phase 1 runs.

- Extract finds one durable fact: *"Globex moved from Basic (30 seats) to Pro (120 seats) in June 2026."*
- Candidate search finds the February memory: *"Globex is on Basic, 30 seats."*
- Consolidation thinks like this: *"The new fact changes the plan and the seat count of an existing
  memory. This is an UPDATE, not an ADD."*
- Store marks the February memory as superseded, and saves the new one as active, with the old
  value kept as history.

The *"Yes, the second one"* and *"thanks"* turns produce no durable facts, so nothing is stored for
them.

**In September,** Dana asks *"Do we need the Security add-on for SSO?"* Phase 2 runs. The search
only looks at active memories, so the February fact cannot come back. Retrieval returns:

- *"Globex: Pro plan, 120 seats (since June 2026, previously Basic with 30)."*
- *"In August, Dana asked how long SSO setup takes."*

The plan memory does not mention SSO either. But Dana's store now holds a few dozen clean
statements instead of hundreds of raw turns, so the plan memory easily makes the 50 candidates,
and its high importance lifts it to the top.

Dana's small profile is always loaded, without any search: *"IT admin at Globex. Prefers numbered
steps."*

The Librarian fetches page 1,140 as before.

The Scholar thinks like this: *"Globex is on Pro with 120 seats. Page 1,140 says Pro needs the
add-on only for teams under 50 seats. 120 is not under 50. Her profile says she likes numbered
steps."*

It answers: *"No add-on needed. Globex is on Pro with 120 seats, and the Security add-on is only
required below 50. To turn on SSO: 1. Open Admin settings. 2. ..."*

The answer is correct.

---

## How to score memories at retrieval

The classic formulation comes from generative-agents research (Park et al., 2023). It combines
three signals:

$$ \text{score}(m) = \alpha \cdot \text{similarity}(q, m) + \beta \cdot \text{recency}(m) + \gamma \cdot \text{importance}(m) $$

In simple words, a memory's score is a weighted mix of how related it is, how fresh it is, and how
much it matters. α, β and γ are the weights.

- **Similarity** is Chapter 4's cosine, between the current context and the memory.
- **Recency** decays exponentially since the memory was last *used or updated*, not just since it
  was created. Memories that keep proving useful stay fresh.
- **Importance** is set at extraction time (*"customer's plan" high, "prefers dark mode" medium,
  "was in a hurry on Tuesday" low*), or learned from how often a memory gets retrieved and used.

Here is the scoring from Dana's example, with weights 0.6, 0.25 and 0.15 and a 30-day half-life
for recency. The numbers are toy values.

| Memory | Similarity | Days since used | Recency | Importance | Score |
|---|---|---|---|---|---|
| Globex: Pro, 120 seats | 0.62 | 90 | 0.125 | 0.9 | **0.538** |
| In August, Dana asked how long SSO setup takes | 0.70 | 45 | 0.354 | 0.1 | **0.523** |
| Globex is on Basic, 30 seats (superseded) | 0.62 | 220 | 0.006 | 0.9 | *(0.509, but excluded)* |

The August memory mentions SSO, so it is the most similar, just as in the diary. The plan memory
still wins, because its importance (0.9 against 0.1) outweighs that gap.

Look at the last row. The two plan memories are equally similar, because neither mentions SSO. If
the February memory were still active, it would score 0.509, only 0.03 below the correct memory,
even with almost no recency left. In a store of a few dozen memories, that easily makes the top 8,
and the Scholar would see Basic and Pro side by side again. Recency did not save us.
Consolidation did, by marking it superseded so retrieval never sees it.

Normalise each signal to a comparable range before weighting, and tune the weights on real
conversations. The right balance differs a lot between a coding assistant, where similarity
dominates, and a personal assistant, where importance and recency matter more.

**Note:** Our code uses a 30-day half-life, because conversational memory goes stale quickly.
Chapter 51's recency boost for documents uses about 180 days. Tune both on real data.

---

## Problems with memory systems

The notebook fixes the diary's failures, but it brings its own.

**Problem 1: Extraction errors become permanent.** If the LLM writes "12 seats" instead of "120",
that wrong fact now looks authoritative. Keep a link to the source conversation, and let users
see and correct their memories.

**Problem 2: Every fact costs LLM calls.** Extraction and consolidation each need a model call.
At Acme's scale, with thousands of conversations a day, that is a real line on the bill
(Chapter 60).

**Problem 3: A wrong UPDATE can erase the truth.** If consolidation wrongly decides that a new
fact replaces an old one, the old one disappears from retrieval. This is why we supersede instead
of overwrite: a bad update can be reverted.

**Problem 4: Memory is personal data.** It needs the same isolation, deletion and access rules as
any tenant data (Chapter 62).

---

## When to use which one

| | Raw turn log (the diary) | Curated memory (the notebook) | Always-loaded profile |
|---|---|---|---|
| What is stored | Every turn, as-is | Extracted, consolidated statements | A handful of core facts |
| Handles changing facts | No | Yes, by UPDATE | Yes, by editing the record |
| Retrieval | Similarity only | Similarity + recency + importance | None: always in context |
| Cost | Cheap to write | LLM calls per fact | Tiny |
| Best for | Audit, re-extraction later | The long tail of facts | Facts that must never be missed |

We must use **curated memory** when an assistant talks to the same person many times and facts
about them change. We must use an **always-loaded profile** for the few facts that every reply
depends on, because retrieval can miss them. We must use a **raw log** only as an archive, never
as the thing the Scholar reads. Many strong systems use all three: the profile in context, curated
memories retrieved, and the raw log kept for audit and for re-extracting with a better model.

---

### Under the hood

The code follows Phase 1 (`remember`) and Phase 2 (`recall`):

```python
from datetime import datetime, timezone

CONSOLIDATE = """Existing memories:
{existing}

New candidate fact: {fact}

Choose ONE operation and output JSON:
- {{"op": "ADD"}}                                   if genuinely new
- {{"op": "UPDATE", "id": "...", "text": "..."}}     if it changes an existing memory
- {{"op": "MERGE", "ids": ["...", "..."], "text": "..."}}  if it duplicates existing memories
- {{"op": "IGNORE"}}                                if already known or not durable"""

def remember(conversation, user_id):
    for fact in llm_extract_durable_facts(conversation):              # Step 1: extract
        similar = memory_index.search(embed(fact.text), k=5,           # Step 2: candidates
                                      filter={"user": user_id, "status": "active"})
        decision = llm_json(CONSOLIDATE.format(existing=fmt(similar),  # Step 3: consolidate
                                               fact=fact.text))
        if decision["op"] == "ADD":                                    # Step 4: store
            memory_store.add(user_id, fact.text, importance=fact.importance)
        elif decision["op"] == "UPDATE":
            memory_store.supersede(decision["id"], new_text=decision["text"],   # keep history
                                   importance=fact.importance)
        elif decision["op"] == "MERGE":
            memory_store.merge(decision["ids"], new_text=decision["text"],
                               importance=fact.importance)

def recall(query, user_id, k=8, now=None):
    now = now or datetime.now(timezone.utc)
    cands = memory_index.search(embed(query), k=50,
                                filter={"user": user_id, "status": "active"})
    def score(m):
        rec = 0.5 ** ((now - m.last_accessed).days / 30)              # 30-day half-life
        return 0.6 * m.similarity + 0.25 * rec + 0.15 * m.importance
    top = sorted(cands, key=score, reverse=True)[:k]
    memory_store.touch([m.id for m in top], now)                      # refresh recency
    return top
```

`0.5 ** (days / 30)` is the recency term. In simple words, a memory untouched for 30 days counts
half as fresh, and one untouched for 60 days counts a quarter. The `"status": "active"` filter is
what keeps superseded facts, like Globex's old Basic plan, out of every search.

Notice `supersede` rather than overwrite. Keeping the history with a status flag means the system
can answer *"what plan were we on before?"*, and an erroneous update can be reverted. An IGNORE
decision simply stores nothing.

---

### What people get wrong

**Storing raw turns.** Noise, fragments and contradictions, retrieved by similarity alone.

**No consolidation.** Contradictory facts pile up, and both get retrieved.

**Similarity-only retrieval.** It ignores recency and importance, which are often decisive.

**Never forgetting.** Unbounded growth, rising noise, and personal data kept indefinitely.

**Invisible memory.** Users should be able to see, correct and delete what the agent remembers
about them. It is a trust feature and, in many jurisdictions, a legal one.

**One memory pool for everyone.** Memory is per-user or per-tenant data, with the same isolation
requirements as Chapter 62. Dana's memories must never surface for someone at Initech.

---

### Ninja notes

**Separate memory types, because they behave differently:**

| Type | Example | Storage | Retrieval |
|---|---|---|---|
| **Semantic** (facts) | "Globex is on Pro, 120 seats" | Consolidated statements | Similarity + importance |
| **Episodic** (events) | "Helped Dana download the June invoice on 4 July" | Summaries with timestamps | Time and similarity |
| **Procedural** (how-to) | "For Globex, attach the SAML trace before escalating" | Instructions | Task-type match |
| **Profile** (stable preferences) | "Dana prefers numbered steps" | Small structured record | Always loaded |

The profile tier deserves emphasis. A small set of core facts should not be *retrieved* at all. It
should be **always in context**, because retrieval can miss it. Vector retrieval is for the long
tail that does not fit.

**Graph-shaped memory helps with entities.** Facts about people, companies and systems form a
graph: Dana *works at* Globex, which *uses* the Pro plan, which *includes* SSO at 50 seats or
more. Storing entity links alongside vectors lets retrieval walk from a mentioned entity to related
facts, the same way Chapter 51 follows document references.

---

### Key takeaways

- **Agent memory = what an agent keeps from past conversations and brings back into future
  ones.** Semantic memory holds facts. Episodic memory holds events.
- Naive "embed every turn" memory is a diary with a search box: contradictory, noisy, fragmentary
  and unbounded.
- **Consolidation = deciding whether a new fact adds, updates, merges or is ignored.** It is what
  turns logging into memory.
- Vector search finds consolidation candidates and retrieval candidates. It is not the design.
- Store clean, self-contained memory statements, and let the embedding index them.
- Retrieve on similarity, recency and importance together, over active memories only. Keep
  superseded history.
- Keep a small always-loaded profile, and retrieve the long tail.
- Memory is personal data: make it visible, correctable, deletable and isolated.

### What's next

[Chapter 65](./65-the-frontier.md) looks ahead: which directions in this field look durable, and
which bets are safe to make today.

Now we have understood why a pile of embedded messages is not a memory, and how extraction,
consolidation, scoring and forgetting turn it into one.
