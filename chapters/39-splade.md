---
title: "SPLADE and Learned Sparse Retrieval"
chapter: 39
part: "Part V — Beyond One Vector"
slug: splade
readingTime: "12 min"
summary: "A transformer that outputs term weights instead of a dense vector — including weights for words the document never contained. Neural semantics, classical inverted index."
tags: [splade, learned-sparse, inverted-index, expansion, hybrid]
prev: 38-muvera
next: 40-hybrid-search
---

# SPLADE and Learned Sparse Retrieval

**The one-paragraph version.** **SPLADE** uses a transformer's masked-language-model head to give
a weight to every word in its vocabulary for a given document, then forces most of those weights
to zero. The result is a sparse vector that plugs straight into a classical inverted index. But
it also contains *learned* terms the document never used, such as "login" and "SAML" for a page
that only says "SSO". We get semantic matching with the exactness, explainability and familiar
infrastructure of BM25. We do not get quite BM25's speed, because the expanded queries are
heavier to serve.

In this chapter, we will learn how SPLADE turns a transformer into a writer of index cards, and
why those cards find pages that plain keyword search misses. We will also see why it runs on the
search engines many teams already operate, where it is slower than BM25, and when to use it.

We will cover the following:

- What is SPLADE
- Why we need SPLADE
- How SPLADE works
- An example of SPLADE
- Why the inverted index matters
- Problems with SPLADE
- BM25 vs dense vs SPLADE
- When to use which one

---

## What is SPLADE

Before jumping in, recall from Chapter 7 that a **sparse** vector has one slot per vocabulary
word, and almost all slots are zero. It lives in an inverted index, where each word points to
the pages that contain it.

**Learned sparse retrieval = a neural network decides which words go on a page's sparse vector,
and how much each one weighs.** BM25 counts words. A learned sparse model *chooses* them.

**SPLADE = SParse Lexical AnD Expansion model.** Let's break it down.

- **Sparse:** the output is a sparse vector over the vocabulary, mostly zeros.
- **Lexical:** each slot is a real word (or word piece) we can read.
- **Expansion:** the model may add words the page never contained.

**Expansion = putting weight on words that are not in the text, but that a searcher might use to
find it.**

Think of it like a librarian writing an index card for page 212 of the Great Library, which says
"SSO is included on the Pro plan."

- The card is the sparse vector.
- A lazy librarian lists only the words on the page: "SSO", "Pro", "plan", "included".
- A good librarian also writes "single sign-on", "login" and "SAML", because that is how people
  will ask. Those extra words are the expansion.
- How firmly each word is written on the card is its weight.

That is SPLADE. It produces an index card with weights on terms that are present *and* on terms
that ought to be.

---

## Why we need SPLADE

Acme's Librarian already has two tools. Let's give each one a question it gets wrong.

**BM25 and the paraphrase twin.** A user asks whether their team can log in with their company
accounts. The answer is page 212, "SSO is included on the Pro plan." But the question and page
212 share no words at all. BM25 gives page 212 a score of zero. It ranks a password-reset page
first, because that page says "log in". The answer is wrong.

**Dense search and the exact-token twin.** A user pastes an error code, `SSO-4012` ("SAML
assertion expired"). A dense embedding sees `SSO-4012` and `SSO-4021` as almost the same string.
It returns the page for the other error just as happily. The answer is wrong.

This is the vocabulary mismatch problem of Chapter 8, "car" not matching "automobile", plus its
mirror image. BM25 is exact but literal. Dense search understands meaning but blurs exact tokens.

SPLADE fixes the first problem without giving up the exactness that fixes the second. It does not
make the representation dense. It **expands the sparse representation**, so the matching words
are there to be matched.

---

## How SPLADE works

Every masked language model already has the machinery. BERT was trained to guess hidden words.
Its **masked-language-model (MLM) head** is the final layer that does the guessing. For every
position in the input, it gives a score to each of the 30,522 entries in BERT's vocabulary
(Chapter 10). SPLADE repurposes that head.

**Phase 1: Encoding a page.**

**Step 1:** Run the page through BERT and its MLM head. For input position $i$ and vocabulary
entry $j$, we get a raw score $w_{ij}$.

**Step 2:** Clamp negative scores to zero with **ReLU** (rectified linear unit), which turns every
negative number into 0 and leaves positive numbers alone.

**Step 3:** Squash each score with $\log(1 + x)$, so big scores grow slowly.

**Step 4:** Combine each vocabulary entry's scores across all positions into one weight.

**Step 5:** Keep only the non-zero weights. That is the page's sparse vector.

**Step 6:** Write each non-zero term into the inverted index, with its weight.

In the original SPLADE paper, Steps 2–4 are one formula:

$$ s_j = \sum_{i \in \text{tokens}} \log\left(1 + \text{ReLU}(w_{ij})\right) $$

Put simply, a vocabulary word gets weight if some position in the page makes the model think of
that word. Two design choices hide in the formula. **ReLU creates sparsity**, because most
vocabulary words score negative for most pages. **$\log(1+x)$ saturates**, so one position that
votes very strongly for a word cannot drown out everything else. A raw score of 20 becomes 3.0,
and a score of 2 becomes 1.1. It is a cousin of BM25's $k_1$ saturation (Chapter 8). Both flatten
large values, but BM25 flattens how *often* a word appears, and SPLADE flattens how *strongly* the
model votes for it at each position.

**Phase 2: Answering a query.**

**Step 1:** Encode the query the same way, into its own sparse vector.

**Step 2:** For each non-zero query term, walk that term's posting list.

**Step 3:** Add up query weight × page weight over the shared terms. That sum is the page's score.

**Phase 3: Training for sparsity.** Left alone, the model would put small non-zero weights on
thousands of words, and the posting lists would explode. So SPLADE adds a **FLOPS regulariser**.

**FLOPS = an estimate of how many floating-point operations the inverted index will need to score
a query.** The regulariser adds that estimated serving cost to the loss:

$$ \mathcal{L} = \mathcal{L}_{\text{rank}} + \lambda_q \, \ell_{\text{FLOPS}}(q) + \lambda_d \, \ell_{\text{FLOPS}}(d) $$

In simple words, the model is punished for being expensive to search. This is unusual and worth
pausing on: **the training objective includes the retrieval system's serving cost.** $\lambda$ is
a dial trading quality against speed. Turn it up and posting lists shorten, queries get faster,
and quality drifts down. Very few models let us tune that trade-off at *training* time.

The result, for a typical passage, is 100–300 non-zero terms out of 30,522.

---

## An example of SPLADE

Here is what a SPLADE vector for page 212 might look like. The weights are illustrative.

| Term | Weight | In page 212? |
|---|---|---|
| sso | 2.6 | yes |
| pro | 2.2 | yes |
| plan | 1.6 | yes |
| sign | 1.4 | **no, expanded** |
| single | 1.2 | **no, expanded** |
| included | 1.1 | yes |
| log | 1.1 | **no, expanded** |
| authentication | 0.9 | **no, expanded** |
| accounts | 0.7 | **no, expanded** |
| saml | 0.6 | **no, expanded** |

**Note:** SPLADE works on BERT's word pieces, not whole words. "SSO" is really two pieces, `ss` and
`##o`, "login" is `log` + `##in`, and "SAML" is `sam` + `##l`. We show readable words where we can.

Now let's run the paraphrase twin again, *"Can our team log in with our company accounts?"*
SPLADE encodes the question too, and expands it. Its non-zero terms include `log` 1.8,
`accounts` 1.3, `company` 1.0, `sign` 0.9, `authentication` 0.8, `sso` 0.7 and `single` 0.6,
plus a small weight on `password`.

The score is query weight × page weight, summed over shared terms:

```
page 212:             log 1.8×1.1  + accounts 1.3×0.7 + sign 0.9×1.4
                    + sso 0.7×2.6  + single 0.6×1.2   + authentication 0.8×0.9   = 7.41

password-reset page:  log 1.8×1.9  + accounts 1.3×0.5 + password 0.4×2.5           = 5.07
```

Page 212 comes first. The answer is correct. BM25 found no shared words, but SPLADE put `log`,
`sign` and `authentication` on both cards, so they meet in the inverted index.

And the exact-token twin? The tokenizer splits `SSO-4012` into `ss`, `##o`, `-`, `401` and `##2`.
It splits `SSO-4021` into `ss`, `##o`, `-`, `402` and `##1`. Those pieces stay exact terms in
separate posting lists. A query for `SSO-4012` matches `401` and `##2` together only on the right
page, so the two codes no longer blur together. The answer is correct. BM25 over whole tokens is
still the more airtight tool for codes, because SPLADE's pieces are shared with many other
numbers.

And notice: **the whole thing is readable.** We can look at a SPLADE vector and see what the model
thinks a page is about. We can explain a result by naming the terms that matched. Dense retrieval
offers nothing comparable.

---

## Why the inverted index matters

SPLADE's output is a sparse vector, so it runs on infrastructure that has been optimised for
fifty years. That brings real benefits, and one honest caveat about speed.

- **Pruned posting-list traversal.** Impact-ordered posting lists and block-max WAND let the engine
  skip documents that provably cannot reach the top-k. This is why BM25 serves hundreds of millions
  of documents in single-digit milliseconds. SPLADE runs on the same index. But its expanded
  queries and flatter weight distributions defeat much of that pruning. We should expect 2–10×
  BM25's latency, unless we use the Efficient-SPLADE variants.
- **Exact retrieval.** There is no approximation, no recall parameter and no `efSearch`.
- **Easy updates and deletes.** Posting lists are append-and-filter structures.
- **Natural filtering.** The same engine handles metadata predicates, so Chapter 33's hardest
  problem becomes easy.
- **Familiar systems.** It runs in Elasticsearch, OpenSearch, Vespa and Lucene, which many
  organisations already operate.

That last point is strategically significant. SPLADE can often be deployed into existing search
infrastructure without introducing a vector database at all.

---

## Problems with SPLADE

Why not use SPLADE for everything? There are five reasons.

**Problem 1: Queries are slower than BM25.** A SPLADE query expands to 20–50 terms instead of 3,
so more posting lists are walked. Query latency is typically 2–10× BM25's and can exceed a dense
ANN search. The Efficient-SPLADE variants attack this directly. They keep queries very sparse,
often with a lighter query encoder, while still letting documents expand.

**Problem 2: The index grows.** More non-zero terms per page means longer posting lists, commonly
2–5× the size of a BM25 index.

**Problem 3: It is still vocabulary-bound.** SPLADE expands *within* the model's vocabulary. It
handles synonyms superbly, and cross-lingual retrieval not at all, unless the model was built to
be multilingual. A question asked in Spanish is unlikely to reach page 212.

**Problem 4: Every page needs a transformer pass at index time.** Unlike BM25, we must run a model
over all 40,000 pages. That costs as much as dense embedding, and a model upgrade means
re-indexing everything.

**Problem 5: Expansions can be wrong on jargon.** Expansion is learned from the training corpus.
On specialised vocabulary the model may add misleading terms, which can be worse than adding
nothing.

---

## BM25 vs dense vs SPLADE

| | BM25 | Dense | SPLADE |
|---|---|---|---|
| Paraphrase twin ("log in with company accounts") | Misses page 212 | Finds it | Finds it |
| Exact-token twin (`SSO-4012`) | Nails it | Blurs with `SSO-4021` | Keeps exact pieces |
| Cross-lingual questions | No | Yes, with a multilingual model | No, unless multilingual |
| Index | Inverted | ANN (HNSW, IVF, ...) | Inverted |
| Query speed | Fastest | Fast | Typically 2–10× BM25 |
| Explainable | Yes | No | Yes |
| Model inference at index time | None | Yes | Yes |

---

## When to use which one

We must use **BM25** when queries are mostly exact identifiers, names and codes, and latency or
cost must be as low as possible.

We must use **dense retrieval** when questions are conceptual or cross-lingual, and exact tokens
rarely decide the answer.

We must use **SPLADE** when we want meaning-aware matching inside an existing Elasticsearch,
OpenSearch or Vespa deployment, or when results must be explainable term by term.

Many strong systems use more than one. SPLADE or BM25 on one side and dense on the other, fused
together, is the subject of Chapter 40.

---

### Under the hood

```python
import torch
from transformers import AutoModelForMaskedLM, AutoTokenizer

class Splade(torch.nn.Module):
    def __init__(self, name="naver/splade-v3"):
        super().__init__()
        self.mlm = AutoModelForMaskedLM.from_pretrained(name)

    def forward(self, ids, mask):
        logits = self.mlm(ids, attention_mask=mask).logits          # Step 1: (B, T, V)
        act = torch.log1p(torch.relu(logits))                        # Steps 2–3: ReLU, saturate
        act = act * mask.unsqueeze(-1)                               # drop padding
        return act.max(dim=1).values                                 # Step 4: (B, V) pooled

def to_sparse(vec, tokenizer, top_k=None):                           # Step 5: readable card
    idx = torch.nonzero(vec).squeeze(-1)
    pairs = [(tokenizer.convert_ids_to_tokens(int(i)), float(vec[i])) for i in idx]
    pairs.sort(key=lambda p: -p[1])
    return dict(pairs[:top_k] if top_k else pairs)

def flops_regulariser(batch_vecs):                                   # Phase 3
    """Penalises expected posting-list cost: mean activation per term, squared."""
    return (batch_vecs.abs().mean(dim=0) ** 2).sum()
```

`to_sparse` prints the index card, like the page 212 table above. `flops_regulariser` is the
serving-cost term from Phase 3.

Note that this implementation uses `max` pooling over positions, which is what the released
SPLADE checkpoints use, including `naver/splade-v3`. The `sum` form in the equation above is the
original v1 formulation. Both appear in the literature, and the model cards say which one a
checkpoint expects. In simple words, match the pooling to the model's training. It is the same
class of issue as Chapter 11's pooling.

---

### What people get wrong

**"Sparse means old-fashioned."** SPLADE is a transformer. The *representation* is sparse. The
model producing it is not.

**"It replaces dense retrieval."** It is complementary. SPLADE handles exact terms and
in-vocabulary synonyms. Dense handles paraphrase, cross-lingual and conceptual similarity. The
best systems run both (Chapter 40).

**Ignoring the efficiency regulariser.** Train or select a checkpoint without attention to
$\lambda$, and your posting lists can become so long that queries are slower than brute-force
dense search.

**"It inherits BM25's speed because it uses the same index."** It uses the same index, not the
same workload. Expanded queries touch many more lists, and flat weights defeat WAND-style
skipping. Measure before assuming it is free.

**Using it on out-of-domain jargon without evaluation.** Expansion is learned from the training
corpus. On specialised vocabulary the expansions may be wrong in ways that are worse than no
expansion.

---

### Ninja notes

**BGE-M3 is the pragmatic shortcut.** It emits *dense, sparse and multi-vector* representations
from a single forward pass, with an 8,192-token context window. One model, one inference cost,
three representations you can fuse (Chapter 40). Its sparse output weights only words that appear
in the text. It does not expand, so its dense side still has to catch paraphrases. If you want
hybrid retrieval without operating three pipelines, this is an unusually practical starting point,
and it collapses a lot of the architecture in this Part.

**The three-way taxonomy is now complete**, and it is worth seeing all at once:

| | Vectors per doc | Dimensions | Index | Strength |
|---|---|---|---|---|
| **Sparse (SPLADE)** | 1 | ~30k, ~200 non-zero | Inverted | Exact terms, expansion, explainable |
| **Dense** | 1 | 768, all non-zero | ANN | Paraphrase, cross-lingual |
| **Multi-vector (ColBERT)** | ~200 | 128 each | PLAID / FDE | Precision, compositional queries |

They are not competitors so much as three points on a spectrum of *how much you refuse to
compress*. Sparse refuses to compress the vocabulary. Multi-vector refuses to compress the
sequence. Dense compresses both, and is cheapest.

Knowing which compression is hurting you is the diagnostic skill. If you are losing identifiers
and jargon, you compressed the vocabulary too hard, so add sparse. If you are losing needles in
long documents and multi-constraint queries, you compressed the sequence too hard, so add
multi-vector or smaller chunks.

---

### Key takeaways

- **SPLADE = a transformer's MLM head writing weights over the whole vocabulary, made sparse with
  ReLU and a FLOPS regulariser.**
- $\log(1+x)$ flattens how strongly each position votes for a word. It is a cousin of BM25's
  $k_1$ saturation, which flattens how often a word appears.
- The output includes **expansion terms** the document never contained, such as "login" and
  "saml" for a page that says "SSO". That solves vocabulary mismatch inside a sparse
  representation.
- It runs on classical inverted indexes: exact, updatable, filterable, explainable, and deployable
  in infrastructure you may already run.
- It does not inherit BM25's speed. Expanded queries and flat weights defeat much of WAND's
  pruning, so expect 2–10× BM25's latency unless you use Efficient-SPLADE variants.
- Costs: larger indexes, transformer inference at index time, and no cross-lingual ability.
- BGE-M3 gives dense, sparse and multi-vector output from one model. Its sparse output does not
  expand.

### What's next

Three representations, three strengths. [Chapter 40](./40-hybrid-search.md) combines them, and
delivers the highest return-on-effort technique in this entire book.

We now know how SPLADE writes expanded index cards, why they catch paraphrases that BM25 misses
without blurring exact codes, and what that costs at query time.
