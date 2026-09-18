---
title: "Agentic Retrieval"
chapter: 56
part: "Part VII — RAG"
slug: agentic-retrieval
readingTime: "10 min"
summary: "Stop retrieving once, up front. Give the model a search tool and let it decide when to search, what to search for, and whether to search again after reading."
tags: [agents, tool-use, iterative-retrieval, multi-hop, rag]
prev: 55-rag-failure-modes
next: 57-architecture-at-scale
---

# Agentic Retrieval

**The one-paragraph version.** Classic RAG retrieves once, before the Scholar thinks.
**Agentic retrieval** gives the Scholar search as a *tool*. It reads the question, issues a
query, reads the results, notices what is missing, searches again with better terms, and answers
when it has enough. This handles multi-hop questions, vague questions and exploratory research
far better. The price is more latency, more tokens, and a retrieval system that must stay fast,
cheap and predictable under repeated calls.

In this chapter, we will learn what an agent is, what a tool call is, and how the Scholar can
run its own searches in a loop. We will also see where agents win, where they waste money, how
to design search as a tool, and when to use which approach.

We will cover the following:

- What is agentic retrieval
- Why we need agentic retrieval
- How agentic retrieval works
- An example: the SSO question, searched twice
- Where it wins, and where it does not
- Designing search as a tool
- Single-shot RAG vs agentic retrieval
- When to use which one

---

## What is agentic retrieval

Before jumping into agentic retrieval, we must know three terms: agent, tool call and
orchestrator.

**Agent = an LLM running in a loop, choosing its next action until it decides it is done.**

A **tool call** is when the model, instead of answering, emits a structured request such as
`search("SSO Pro plan")`. Our code, the **orchestrator**, runs it and feeds the result back. The
model may then call again or answer.

In simple words, the Scholar decides what to look up, and our code does the looking.

**Agentic retrieval = RAG in which the Scholar decides when, what and how often to search.**

Think of it like the difference between being handed a folder and being let into the Library.

- **Classic RAG** is a folder of printouts someone else chose. The folder is the top few pages
  the Librarian fetched. If it is missing something, the Scholar cannot fix that.
- **Agentic retrieval** is a Library card. The Scholar searches, skims, notices a reference,
  searches for that too, and then answers.

The second is how every competent human researcher works. RAG did not start this way only
because models were not yet reliable at deciding what to do next. Current models are usually
reliable enough to do it well.

---

## Why we need agentic retrieval

Let's first see what classic, single-shot RAG does with our running question: *"Does the Pro
plan include single sign-on?"*

The Librarian runs one search and returns the top five pages:

1. Page 212, the Pro plan page: "SSO is included on the Pro plan.\*" At the bottom, a
   footnote: "\*Seat limits apply. See Security add-on."
2. Page 88, the SSO setup guide.
3. A SAML troubleshooting page.
4. Page 2,306, "Compare plans", which ticks SSO in the Pro column.
5. A release note announcing SSO.

Page 1,140, the one that says Pro teams under 50 seats need the Security add-on, has its SSO
chunk at rank 14, as in Chapter 51. It is not in the folder.

The Scholar reads the five pages and answers: "Yes, single sign-on is included on the Pro plan."

The answer is wrong. It is failure 8 from Chapter 55: only part of the evidence arrived. Worse,
the Scholar *saw* the footnote pointing to the missing page. It simply had no way to go and get
it.

Chapter 52's decomposition could have helped, but only if someone decided to split the question
*before* retrieval. Nobody knew about the footnote before retrieval. The information that tells
us what to search next only appears *after* the first search.

---

## How agentic retrieval works

**Step 1:** The orchestrator sends the question to the Scholar, together with a description of
each available tool.

**Step 2:** The Scholar decides: answer now, or call a tool.

**Step 3:** To call a tool, the Scholar emits a tool call, such as
`search_documents(query="Pro plan single sign-on")`.

**Step 4:** The orchestrator runs that search on our retrieval stack and appends the results to
the conversation.

**Step 5:** The Scholar reads the results and returns to Step 2.

**Step 6:** When the Scholar has enough, it answers with citations. If the orchestrator's step
or token limit is reached first, it forces a final answer.

```
question
   ↓
┌─────────────────────────────────────────────┐
│  model thinks: what do I need to know?      │
│        ↓                                    │
│  calls search(query, filters)               │ ◄─┐
│        ↓                                    │   │
│  reads results                              │   │
│        ↓                                    │   │
│  enough to answer?  ── no ──────────────────┼───┘
│        │ yes                                │
└────────┼────────────────────────────────────┘
         ↓
   answer with citations
```

The Scholar now supplies, on demand, what fixed pipelines hard-code:

| Fixed pipeline technique | Agentic equivalent |
|---|---|
| Query rewriting (Ch. 52) | The model writes its own queries |
| Decomposition | The model plans sub-questions and searches each |
| Multi-query expansion | The model tries a different phrasing when results are poor |
| HyDE | The model searches using terms it learned from earlier results |
| Neighbour expansion | The model calls `fetch_document` on a promising hit |
| Relevance floor | The model judges whether results are relevant |

That last row matters. A well-designed agent handles **the empty set** gracefully. Ask it *"Does
Acme offer pet insurance?"* and the first search returns billing pages. A search that returns
nothing useful is information. The agent reformulates once, finds nothing again, and concludes
that the corpus does not cover it.

This is Chapter 55's LLM relevance check, run by the agent itself. It still needs measuring on
the golden set's `answerable: false` questions. Pet insurance is the easy case. The hard case is
a question like *"Does the Pro plan include phone support?"*, where every result looks close and
the agent can still invent an answer.

---

## An example: the SSO question, searched twice

Now, let's see what an agent does with the same question.

**Step 1:** The Scholar thinks: "I need the Pro plan's SSO details." It calls
`search_documents(query="Pro plan single sign-on")`.

**Step 2:** The orchestrator returns the same five pages as before, including page 212 and its
footnote.

**Step 3:** The Scholar thinks: "Page 212 says yes. But its footnote says seat limits apply and
points to the Security add-on. I have not seen that page. I should not answer yet." It calls
`search_documents(query="Security add-on single sign-on seat limits Pro")`.

**Step 4:** The orchestrator returns page 1,140: "On Pro, SSO needs the Security add-on for
teams under 50 seats."

**Step 5:** The Scholar thinks: "Now I have both halves. The answer is yes, with a condition."
It answers:

> "Yes, the Pro plan includes single sign-on [page 212]. Teams under 50 seats also need the
> Security add-on [page 1,140]."

The answer is correct. It took two searches and three model calls instead of one search and one
call. That trade is the whole chapter in miniature.

---

## Where it wins, and where it does not

**It wins on multi-hop questions.** *"Does the plan in Globex's contract include audit logs?"*
First the agent must read Globex's contract (page 3,507) to learn which plan Globex bought. Only
then can it search that plan's pricing page. No single query retrieves both, and no fixed
decomposition knows Globex's plan in advance.

**It wins on vague or exploratory questions.** Take a question like *what goes wrong when
customers set up SSO*. The agent searches, reads, and then runs follow-up searches on the
specific problems the first results mention, such as error `SSO-4012`.

**It wins when the right terms are only learnable from the corpus.** A user asks about *"the
security bundle"*. Acme's pages call it *"the Security add-on"*. The first search finds a
release note that mentions the rename. The second search uses the new name.

**It wins on mixed sources.** An agent with separate tools (document search, a billing database,
a ticket system) can pick the right one per sub-question. *"Does our team need the Security
add-on for SSO?"* needs the team's seat count from billing and the rule from page 1,140. A
single retrieval pipeline cannot do that.

**It loses on simple lookups.** *"What does error SSO-4012 mean?"* One search is enough. An
agent adds seconds of latency and several model calls to reach the same page.

**It loses under strict latency budgets.** Each loop is a model call plus a search. Three loops
can easily take several seconds.

**It loses on high-volume, low-margin traffic.** Token costs grow faster than the number of
loops. Each model call re-reads the whole conversation so far, including every earlier result
set. Our SSO example read 0 + 1 + 2 = 3 result sets, against 1 for single-shot. At a cap of six
steps plus the forced answer, it reads 0 + 1 + … + 6 = 21, while retrieval latency grows only six
times. Prompt caching (Chapter 53) softens this.

---

## Designing search as a tool

This is the part that changes for the retrieval engineer. The tool definition is an API that
the model reads, and its design shapes the model's behaviour.

```python
SEARCH_TOOL = {
  "name": "search_documents",
  "description": (
    "Search Acme's support knowledge base. Returns up to `limit` passages with "
    "title, date, source id and a relevance score. Use specific terms; for "
    "exact identifiers (error codes, ticket IDs) put them in quotes. If results "
    "are poor, try different terminology rather than repeating the same query."),
  "input_schema": {
    "type": "object",
    "properties": {
      "query":     {"type": "string"},
      "doc_type":  {"type": "string",
                    "enum": ["pricing", "help", "release_note", "contract", "any"]},
      "after":     {"type": "string", "description": "ISO date; only newer documents"},
      "limit":     {"type": "integer", "default": 5, "maximum": 20}
    },
    "required": ["query"]
  }
}

FETCH_TOOL = {
  "name": "fetch_document",
  "description": "Retrieve the full text of a document by source id when a passage "
                 "looks relevant but lacks surrounding context.",
  "input_schema": {"type": "object",
                   "properties": {"source_id": {"type": "string"}},
                   "required": ["source_id"]}
}
```

In simple words, each tool is a name, a plain-English description the model reads, and a list
of the inputs it may fill in. The model sees nothing else, so the description is the manual.

Principles that emerge from practice:

- **Return compact results with metadata.** Titles, dates and ids let the model decide what to
  open without reading everything.
- **Separate search from fetch.** A cheap skim, then selective deep reads. This is the agent
  version of retrieve small, pass large (Chapter 50).
- **Expose filters as parameters.** Models use them well when they are named clearly.
- **Return scores.** They help the model judge whether to reformulate.
- **Say something useful on empty results.** "No results. Try broader terms." beats an empty
  list.
- **Cap iterations** and total tokens in the orchestrator. Agents occasionally loop.

---

## Single-shot RAG vs agentic retrieval

| | Single-shot RAG | Agentic retrieval |
|---|---|---|
| Searches per question | One (or a fixed few) | As many as the model decides, up to a cap |
| LLM calls per question | One | Several (our SSO example used three) |
| Latency | Predictable, lowest | Variable, often seconds |
| Token cost | Lowest | Grows faster than the number of loops |
| Multi-hop and vague questions | Weak | Strong |
| Uses what it learns mid-search | No | Yes |
| Debugging | One context to inspect | A whole **trajectory** (every query, result and model step, in order) to inspect |

---

## When to use which one

We must use **single-shot RAG** when questions are simple lookups, when latency budgets are
tight, or when traffic is high-volume and low-margin.

We must use **agentic retrieval** when questions are multi-hop, vague, or depend on terms only
the corpus knows, or when answers need several kinds of source.

Many strong systems use both. **The practical answer is routing again.** Send simple queries
down the single-shot path and complex ones to the agent. A cheap classifier works. So does
letting the agent answer immediately when its first search is clearly enough. Either captures
most of the benefit at a fraction of the cost.

---

### Under the hood

The orchestrator, which implements the Steps above:

```python
def agentic_answer(question, max_steps=6, max_tokens=50_000):
    messages = [{"role": "user", "content": question}]
    used = 0
    for step in range(max_steps):
        if used > max_tokens:
            break                                        # token budget passed (a soft limit)
        resp = llm(messages, tools=[SEARCH_TOOL, FETCH_TOOL], system=SYSTEM)
        used += resp.usage.input_tokens + resp.usage.output_tokens
        messages.append({"role": "assistant", "content": resp.content})

        calls = [b for b in resp.content if b.type == "tool_use"]
        if not calls:
            return resp                                  # model chose to answer

        results = []
        for c in calls:                                  # run concurrently in production
            out = (hybrid_search(**c.input) if c.name == "search_documents"
                   else fetch_document(**c.input))
            log_tool_call(step, c, out)                  # essential for debugging
            results.append({"type": "tool_result", "tool_use_id": c.id,
                            "content": format_results(out)})
        messages.append({"role": "user", "content": results})

    return force_final_answer(messages)                  # step or token budget exhausted
```

`SYSTEM` is the system prompt (Chapter 53). Each pass through the loop is one model call. If
the reply contains no tool calls, the model has answered. Otherwise we run every call, append
the results, and go round again. The loop stops once it has made `max_steps` calls or used more
than `max_tokens` tokens, whichever comes first, and forces an answer. The token budget is a soft
limit. We check it only before each call, so the call that crosses it still runs, and the forced
answer is one more call. Set the budget with some headroom.

Notice that `hybrid_search` is exactly the retriever from Parts IV and V. **Agentic retrieval
does not replace your retrieval stack. It calls it repeatedly.** Everything earlier in this book
still decides the quality of each individual search.

---

### What people get wrong

**Using an agent for everything.** Most queries do not need it. Route.

**Weak tool descriptions.** The model knows only what the description says. Vague descriptions
produce vague queries.

**Returning full documents from search.** Burns the context on the first call. Return passages
and let the model fetch.

**No iteration cap.** Occasionally an agent searches in circles. Cap steps and tokens.

**Not logging the trajectory.** When an agentic answer is wrong, you need every query issued and
every result returned to understand why. The diagnostic from Chapter 55 now applies per step.

**Assuming slow retrieval is fine because the LLM is slow.** An agent calling search six times
multiplies retrieval latency by six. Retrieval speed matters *more* in agentic systems, not
less.

---

### Ninja notes

**Evaluation changes shape.** You now care about the trajectory as well as the final answer.
How many searches were issued? Did queries improve across iterations? Did the agent stop too
early (answering from thin evidence) or too late (searching after it had the answer)? Add these
to your eval (Chapter 54). "Average searches per correct answer" is a particularly useful
efficiency metric.

**Parallel tool calls are a large latency lever.** Modern models can issue several searches in
one step, one per sub-question. Execute them concurrently. A three-part comparison then costs
one round-trip rather than three.

**Retrieval quality compounds in agents.** In single-shot RAG, a mediocre retriever produces a
mediocre answer. In an agent, poor first results produce poorly informed follow-up queries,
which produce worse results. Good retrieval makes agents efficient. Bad retrieval makes them
expensive *and* wrong. The investment in Parts IV and V pays off more here, not less.

**Memory is the natural next step.** An agent that researches a topic produces useful
intermediate knowledge: which documents mattered, what the codenames mean, which sources were
stale. Persisting that across sessions is the subject of Chapter 64.

---

### Key takeaways

- **Agentic retrieval = RAG in which the Scholar decides when, what and how often to search.**
  Search becomes a tool the model calls, and the orchestrator runs it.
- It adapts rewriting, decomposition, expansion and relevance judgement to what each search
  actually returned.
- It wins on multi-hop, exploratory and terminology-discovery questions, and on mixed sources.
- It costs latency and tokens, so route: single-shot for simple queries, the agent for complex
  ones.
- Design tools deliberately: compact results with metadata, separate search from fetch,
  explicit filters, useful empty-result messages, iteration caps.
- Agents call your retrieval stack repeatedly, so retrieval speed and quality matter more.
- Log trajectories and evaluate them, not just final answers.

### What's next

Part VII is complete. [Part VIII](./57-architecture-at-scale.md) scales everything up, starting
with the reference architecture for Acme's knowledge-base product at a hundred million chunks.

We now know what an agent is, how the Scholar runs its own searches, where that pays for itself,
and how to route between the two approaches.
