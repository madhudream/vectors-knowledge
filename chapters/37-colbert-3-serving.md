---
title: "ColBERT III: ColBERTv2, PLAID, and Serving"
chapter: 37
part: "Part V — Beyond One Vector"
slug: colbert-3-serving
readingTime: "13 min"
summary: "Two hundred vectors per passage sounds unaffordable. Residual compression cuts storage 14–25x, and PLAID's centroid-based search means full MaxSim runs on a few hundred to a thousand passages out of ten million. Here is how a multi-vector index actually runs."
tags: [colbertv2, plaid, compression, residual, serving, production]
prev: 36-colbert-2-maxsim
next: 38-muvera
---

# ColBERT III: ColBERTv2, PLAID, and Serving

**The one-paragraph version.** Multi-vector retrieval becomes practical through two ideas. The
first is **residual compression**. We group all token vectors into clusters, then store each token
as `(centroid ID, a few bits describing the leftover)`. That is roughly 20–36 bytes per token
instead of 512. The second is the **PLAID** engine. It uses those same centroids to cheaply rule
out almost every document, so full MaxSim runs on a few hundred to a thousand candidates instead
of millions. Together they turn a research curiosity into an index we can deploy.

In this chapter, we will learn how ColBERTv2 shrinks two hundred vectors per passage into a few
kilobytes, and how PLAID searches them quickly. We will also see what the whole thing costs next
to a single-vector index, and when running ColBERT only as a reranker is the smarter choice.

We will cover the following:

- Why serving ColBERT is hard
- What is residual compression
- How residual compression works
- What is PLAID
- How PLAID answers a query
- An example: the SSO question, served three ways
- What it actually costs
- Full ColBERT index vs ColBERT as a reranker
- When to use which one

---

## Why serving ColBERT is hard

Acme's 40,000 pages are a small Library. But Acme's hosted product already holds about 100
million chunks (Chapter 17). Let's price a slice of it: **10 million passages**, each about 200
tokens long. For all of Acme, multiply the storage figures by ten.

Here is the bill for plain ColBERT from Chapter 35, one vector per token:

10,000,000 passages × 200 tokens × 128 dimensions × 4 bytes = **1.02 TB**.

A single-vector index over the same passages, one 768-d float32 vector each, is about 30 GB.
That ratio is why ColBERTv1 was admired and rarely deployed.

There are two separate costs to attack.

- **Storage.** 1 TB of token vectors.
- **Compute.** Full MaxSim is 6,400 dot products per passage (Chapter 36). Over 10 million
  passages, that is 64 billion dot products for a single query.

In simple words, plain ColBERT is too big to hold and too slow to search. ColBERTv2's residual
compression fixes the first problem. PLAID fixes the second.

---

## What is residual compression

Before jumping in, we must recall three words. A centroid is the average point of a cluster
of vectors. k-means is the algorithm that finds those clusters (Chapter 23). A residual
is what is left after subtracting the centroid: vector minus centroid (Chapters 24–25).

**Residual compression = store each token vector as the ID of its nearest centroid, plus a few
bits that roughly describe the residual.**

Think of it like giving directions in a city.

- The city's well-known landmarks are the centroids.
- "Meet me at the town hall" is the centroid ID. A few characters get us close.
- "Then a little north and a little east" is the residual.
- Saying "north-ish, east-ish" instead of exact metres is the 1–2 bits per dimension.

The directions are short, and they land us very close to the right spot.

So why does this work so well for tokens? Because **token vectors cluster
tightly.** The contextual vector for "sso" on page 212 sits very near the vector for "sso" on
page 1,140, and near "sso" on thousands of other pages. The space of things a token vector can
be is far smaller than 128 dimensions suggests. So one shared set of landmarks covers it well,
and the leftovers are small.

---

## How residual compression works

**Phase 1: Building the codebook (once, for the whole index).**

**Step 1:** Take a sample of token vectors from across the corpus.

**Step 2:** Run k-means on the sample to get one shared set of centroids, typically $2^{16}$ to
$2^{18}$ of them.

**Step 3:** Compute the residual of every sampled vector from its nearest centroid.

**Step 4:** For each of the 128 dimensions, split the residual values into 2 equal-sized buckets
(1 bit) or 4 (2 bits). Remember each bucket's edges and a typical value from its middle.
(ColBERTv2 itself shares one set of edges across all dimensions. Per-dimension edges, as in our
code below, are a small variant.)

**Phase 2: Encoding a token.**

**Step 1:** Find the token vector's nearest centroid, and store its ID.

**Step 2:** Subtract the centroid to get the residual.

**Step 3:** For each dimension, record which bucket the residual falls in. That is 1 or 2 bits.

**Step 4:** Pack the bits into bytes and store them next to the ID.

**Phase 3: Decoding a token (only when we need it).**

**Step 1:** Look up the centroid by its ID.

**Step 2:** For each dimension, add the typical value of the recorded bucket.

As a formula, $\hat{v} = c_{id} + \text{dequant}(r)$. Means, a table lookup and an add. We do it
lazily, only for the tokens we actually score.

Let's run it on a tiny example: 4 dimensions instead of 128, and 1 bit per dimension. The token
is "sso" from page 212. The two buckets are "below 0" and "above 0", with typical values −0.03
and +0.03.

```
token vector v       = [ 0.52,  0.31, -0.40,  0.69]
nearest centroid c   = [ 0.50,  0.35, -0.45,  0.65]
residual r = v − c   = [ 0.02, -0.04,  0.05,  0.04]
1 bit: above 0?      = [    1,     0,     1,     1]
decoded c ± 0.03     = [ 0.53,  0.32, -0.42,  0.68]
```

The centroid alone was off by up to 0.05 in some dimension. With just 4 extra bits, the decoded
vector is off by at most 0.02. In simple words, the centroid gets us to the right neighbourhood
and the bits get us to the right street.

Now the real byte count for one 128-d token:

```
centroid ID:               ~4 bytes  (an 18-bit index, padded to 32 bits)
residual, 128-d @ 2 bits:  32 bytes
                    total: ~36 bytes per token      (vs 512 uncompressed)
```

**A 14× reduction.** A 200-token passage drops to about 7,200 bytes, and the terabyte becomes
about 72 GB. With 1-bit residuals it is about 20 bytes per token and about 40 GB. That is
comparable to the 30 GB single-vector index over the same corpus.

**Note:** This is the residual trick from IVF-PQ (Chapters 24–25), applied one token at a time.
Residuals are small and centred on zero, so one global codebook works everywhere, and the bits
go on local detail rather than on re-describing where the cluster is. The difference is that
ColBERTv2 gives each residual coordinate its own simple 1–2-bit scalar code (Chapter 26), not a
PQ sub-vector codebook.

---

## What is PLAID

Compression fixed storage. Now the compute problem: 64 billion dot products per query.

**PLAID (Performance-optimized Late Interaction Driver) = a search engine for ColBERTv2 that uses
the stored centroid IDs to discard almost every passage before running any real MaxSim.**

PLAID's insight is that the **centroid IDs alone are a usable rough sketch** of each passage,
and we already stored them. A passage whose tokens all sit at landmarks far from every query
token cannot score well. We can rule it out without decompressing anything.

**Centroid pruning = skipping every token whose centroid no query token is close to, while we
score passages roughly from their centroids.**

It is Chapter 18's cascade once more. Each stage narrows the set, so the next stage can afford
more precision. PLAID is simply a Card Catalog built for token vectors.

---

## How PLAID answers a query

Let's follow our running question, *"Does the Pro plan include single sign-on?"*, through PLAID.

**Step 1: Score the centroids.** Compute the similarity of each of the 32 query token vectors
with every centroid. With $2^{18}$ centroids that is 8.4 million dot products. It is one dense
matrix multiply, done once per query, in about a millisecond on a GPU.

**Step 2: Pick each query token's nearest centroids.** For each query token, keep only its few
best-scoring centroids, typically 1 to 4. PLAID calls this number `nprobe`, as IVF does (Chapter
23). The kept centroids are the landmarks where tokens like "sso", "sign", "saml", "pro" and
"plan" live. 32 query tokens keep at most 128 (32 × 4) of the $2^{18}$ centroids, so more than
99.9% are never visited.

**Step 3: Gather candidates.** Look up an index that maps each centroid to the passages with a
token assigned to it. This is the inverted index of Chapter 7, with learned clusters playing
the part of words. Every passage with a token at one of the kept centroids becomes a candidate.
Pages 212 and 1,140 both have "sso" tokens, so both are in.

**Step 4: Approximate MaxSim, with centroid pruning.** Score the candidates using centroid
similarities only, without decompressing residuals. Each token stands in as its centroid, and
Step 1 already scored every centroid against every query token, so this is only table lookups.
Tokens whose centroid scores below a threshold (about 0.4–0.5) against every query token are
skipped. That is centroid pruning. Most of a passage's tokens are far from the question, so it
skips most of the lookups. The result is good enough to rank roughly. PLAID keeps the top
`ndocs` (about a thousand), scores them once more from centroids without pruning, and keeps only
the best quarter.

**Step 5: Exact MaxSim.** Decompress the residuals for the few hundred to a thousand passages
left and run full MaxSim. Return the top 10.

```
10,000,000 passages
   ↓ Steps 1–3: nearest centroids per query token + inverted index
   ~50,000 candidate passages
   ↓ Step 4: approximate MaxSim (centroids only, with pruning)
   ~1,000 passages
   ↓ Step 4, second pass: approximate MaxSim again (centroids only, no pruning)
   ~250 passages
   ↓ Step 5: exact MaxSim (decompressed residuals)
   top 10
```

In simple words, cheap arithmetic on landmarks decides which passages get the expensive
arithmetic on tokens.

The PLAID paper reports end-to-end speedups of up to about 7× on GPU and up to about 45× on CPU
versus vanilla ColBERTv2, with essentially unchanged quality.

---

## An example: the SSO question, served three ways

Our question needs two pages. Page 212 says SSO is included on Pro. Page 1,140 adds that teams
under 50 seats need the Security add-on. Page 1,140 is mostly about the Security add-on itself
(audit logs, IP allow-lists, data retention), so the catch is one sentence in a long page.

**Way 1: plain ColBERT, no compression and no pruning.** MaxSim over all 10 million passages
finds both pages. The answer is correct. But the index needs 1 TB, and every query runs 64
billion dot products. This does not ship.

**Way 2: a single-vector index, with ColBERT reordering only its top 1,000.** This is cheap. But
the Librarian's first step uses one vector per passage, and page 1,140's single vector mostly
says "Security add-on". In this illustration it lands at rank 2,300 for this question. ColBERT
reorders the shortlist perfectly, but page 1,140 is not on it. The Scholar reads page 212 alone
and replies, "Yes, SSO is included on Pro." The answer is wrong. It is half an answer, and the
half it misses is the catch.

**Way 3: ColBERTv2 with PLAID.** Page 1,140's "sso" token sits at one of the centroids nearest to
the query's `sign` token, so it becomes a candidate in Step 3. Its "sso" and "pro" tokens survive
pruning, so its rough score in Step 4 keeps it on the shortlist. In Step 5, exact MaxSim lets
`sign` find its "sso" token and `pro` find its "pro" token, and ranks it in the top 10 beside page
212. The Scholar reads both pages and replies, "Yes, but teams under 50 seats also need the
Security add-on." The answer is correct, from a 72 GB index, in about 50–150 ms.

That is the whole chapter in one comparison. Way 1 is right and unaffordable. Way 2 is
affordable and wrong on this question. Way 3 is right and affordable.

---

## What it actually costs

Honest accounting for **10 million passages** of about 200 tokens each:

| | Single-vector (768-d) | ColBERTv2 + PLAID |
|---|---|---|
| Storage | 30 GB | 72 GB (2-bit) / 40 GB (1-bit) |
| Index build | ~0.5–1 h (encode) + ~1 h (HNSW) | ~0.5–1 h (encode) + ~2 h (cluster + compress) |
| Query latency (p50) | 10–20 ms | 50–150 ms |
| Quality (nDCG@10, typical) | baseline | +3 to +8 points over a same-generation bi-encoder, often 0–3 over the strongest modern dense models |
| Operational complexity | Low | Moderate |

The build and latency rows are rough estimates for one GPU machine. Encoding uses the same
100–300 ms per 1,000 passages as reranking below, plus reading and writing the data. Measure them
on your own hardware.

Roughly: **about 2× the storage, 5–10× the latency, and a real quality gain.** That gain is +3
to +8 nDCG points over a bi-encoder of the same generation. Against the strongest modern dense
models the gap is often 0–3 points. It is largest on exactly the queries of Chapter 34: needles
in long pages and questions with several constraints.

Whether that trade is right depends on the application. In a chat product where the LLM call
takes 2 seconds, an extra 100 ms is invisible and the quality gain is real. For an autocomplete
box with a 20 ms budget, it is not viable.

---

## Full ColBERT index vs ColBERT as a reranker

There is a cheaper way to get part of ColBERT's benefit. Retrieve candidates with an ordinary
single-vector index, then run exact MaxSim over just those. We need token vectors only for the
passages we rerank, and we can even compute them on the fly.

On the fly is not free, though. Running 1,000 passages of ~200 tokens through BERT-base takes
roughly **100–300 ms** on a modern GPU. For 100 passages it is about 10–30 ms.

| | ColBERTv2 + PLAID index | Single-vector + ColBERT rerank |
|---|---|---|
| Storage (10M passages) | 40–72 GB of compressed token vectors | 30 GB single-vector index, and no token vectors if computed on the fly |
| Recall | ColBERT's own, high | Capped by the single-vector shortlist |
| Precision in the top 10 | High | High |
| Query latency | 50–150 ms (p50) | Single-vector search + ~100–300 ms to encode 1,000 passages (~10–30 ms for 100) |
| Special index needed | Yes | No |

In simple words, the reranker keeps ColBERT's *precision* and gives up its *recall*. Way 2 in
our example shows exactly what that loss looks like.

---

## When to use which one

We must use **a single-vector index alone** when the latency budget is tight and questions are
short and topical.

We must use **ColBERT as a reranker** when storage is the blocker, when we want to try late
interaction without building a new index, or when the shortlist already has good recall. If we
encode on the fly, 100 candidates is a far better fit than 1,000.

We must use **a full ColBERTv2 + PLAID index** when recall on needles and multi-constraint
questions matters, and we can afford about 2× the storage and a 50–150 ms search.

Many teams start with the reranker, measure what it misses on their eval set (Chapter 19), and
build the full index only when that missing recall shows up.

---

### Under the hood

The compression and decompression, which is the part worth being able to picture. The helpers
at the top are minimal numpy versions, so the whole block runs as written.

```python
import numpy as np

# --- minimal helpers ------------------------------------------------------------
def nearest(X, C, chunk=4096):
    """Index of the closest centroid (Euclidean) for every row of X."""
    half_sq = 0.5 * (C ** 2).sum(1)
    return np.concatenate([np.argmax(X[i:i + chunk] @ C.T - half_sq, axis=1)
                           for i in range(0, len(X), chunk)])

def kmeans(X, k, iters=10, seed=0):
    rng = np.random.default_rng(seed)
    C = X[rng.choice(len(X), k, replace=False)].copy()
    for _ in range(iters):
        a = nearest(X, C)
        sums = np.zeros_like(C)
        np.add.at(sums, a, X)
        counts = np.bincount(a, minlength=k)[:, None]
        C = np.where(counts > 0, sums / np.maximum(counts, 1), C)
    return C

def pack_bits(codes, nbits):              # (n, dim) small ints → (n, dim·nbits/8) bytes
    bits = (codes[..., None] >> np.arange(nbits)) & 1
    return np.packbits(bits.reshape(len(codes), -1).astype(np.uint8), axis=1)

def unpack_bits(packed, nbits, dim):
    bits = np.unpackbits(packed, axis=1)[:, :dim * nbits].reshape(len(packed), dim, nbits)
    return (bits << np.arange(nbits)).sum(-1)

# --- the compressor -------------------------------------------------------------
class ResidualCompressor:
    def __init__(self, n_centroids=2**16, nbits=2):
        self.n_centroids = n_centroids
        self.nbits, self.levels = nbits, 2 ** nbits

    def train(self, token_vectors, sample=2_000_000, seed=0):          # Phase 1
        rng = np.random.default_rng(seed)
        n = min(sample, len(token_vectors))
        S = token_vectors[rng.choice(len(token_vectors), n, replace=False)]
        self.centroids = kmeans(S, self.n_centroids)                    # (C, dim)
        residuals = S - self.centroids[nearest(S, self.centroids)]
        # per-dimension buckets from the residual distribution
        edge_q   = np.arange(1, self.levels) / self.levels              # 2-bit: .25 .50 .75
        centre_q = (np.arange(self.levels) + 0.5) / self.levels         # 2-bit: .125 .375 .625 .875
        self.bins        = np.quantile(residuals, edge_q,   axis=0)     # (levels-1, dim) edges
        self.bin_centres = np.quantile(residuals, centre_q, axis=0)     # (levels, dim) typical values

    def encode(self, V):                                                # Phase 2
        cid = nearest(V, self.centroids)                                # (n,)
        res = V - self.centroids[cid]
        codes = np.stack([np.digitize(res[:, d], self.bins[:, d])
                          for d in range(res.shape[1])], axis=1)        # (n, dim), each < levels
        return cid.astype(np.uint32), pack_bits(codes, self.nbits)

    def decode(self, cid, packed):                                      # Phase 3
        dim = self.centroids.shape[1]
        codes = unpack_bits(packed, self.nbits, dim)                    # (n, dim)
        return self.centroids[cid] + self.bin_centres[codes, np.arange(dim)]
```

`train` is Phase 1, `encode` is Phase 2 and `decode` is Phase 3. On random clustered 128-d test
vectors, the packed codes come out at exactly 16 bytes (1-bit) and 32 bytes (2-bit) per token.
Add the 4-byte centroid ID and we get the 20 and 36 bytes from the byte count above.

And the candidate search that makes PLAID fast, Steps 1–4:

```python
def plaid_candidates(Q, centroids, centroid_to_docs, doc_cids,
                     nprobe=2, threshold=0.45, ndocs=1024):
    S = Q @ centroids.T                             # Step 1: (nq, C) query tokens vs centroids
    near = np.argsort(-S, axis=1)[:, :nprobe]       # Step 2: each query token's nearest centroids
    cands = set()
    for c in np.unique(near):                       # Step 3: inverted-index lookup
        cands.update(centroid_to_docs[c])
    keep = S.max(axis=0) >= threshold               # Step 4: centroid pruning, (C,) True/False
    rough = {}
    for d in cands:
        cids = doc_cids[d][keep[doc_cids[d]]]       # skip tokens at pruned centroids
        rough[d] = S[:, cids].max(axis=1).sum() if len(cids) else 0.0  # MaxSim on centroids
    top = sorted(rough, key=rough.get, reverse=True)[:ndocs]
    full = {d: S[:, doc_cids[d]].max(axis=1).sum() for d in top}   # again, no pruning
    return sorted(full, key=full.get, reverse=True)[:ndocs // 4]    # shortlist for Step 5
```

`centroid_to_docs` is an inverted index, and `doc_cids[d]` holds the stored centroid ID of every
token in passage `d`. Step 5 then calls `decode` on the shortlist and runs exact MaxSim. Put
simply, the whole first stage of a state-of-the-art neural retriever is a posting-list lookup: the
structure from 1970s information retrieval, with a learned vocabulary.

---

### What people get wrong

**Underestimating index build time.** We encode every token and then cluster hundreds of
millions of vectors. Budget hours, not minutes, and remember that it repeats on every model
migration.

**Using too few centroids.** With $2^{12}$ centroids over billions of token vectors, residuals
are large and quantize badly. Scale centroids with corpus size. $2^{16}$–$2^{18}$ is the usual
range.

**Tuning `nprobe` and the pruning threshold blindly.** Too aggressive and recall collapses. Too
loose and the speedup disappears. Sweep both against your eval set, exactly as we sweep `nprobe`
for IVF (Chapter 25).

**Not using an existing implementation.** The Stanford ColBERT library, PyLate (the actively
maintained library for training and serving ColBERT models), RAGatouille, and several vector
databases with native multi-vector support implement all of this correctly. Reimplementing PLAID
is a months-long project.

**Ignoring the `[MASK]` expansion cost.** Queries padded to 32 tokens mean 32 query vectors, even
for a three-word query. Shortening the pad length reduces latency. It also reduces the learned
query expansion of Chapter 35. Measure before shortening.

**"Reranking with ColBERT is free because MaxSim is fast."** MaxSim over stored vectors is fast.
Encoding 1,000 passages on the fly is a 100–300 ms forward pass. Store the token vectors, or
rerank fewer candidates.

---

### Ninja notes

**Token pooling is the quiet, large win.** Neighbouring tokens in a passage often have nearly
identical vectors: function words, repeated phrases, boilerplate. Clustering a document's own
token vectors and keeping the cluster centroids cuts vectors per document by 50–75%. Published
results report that pooling by about 2× (half the vectors) is essentially free, and that going
further, to a third or a quarter of the vectors, costs a little more quality. If you
deploy multi-vector retrieval, this is the first optimisation to apply, and it is far simpler
than anything else in this chapter.

**Where 100–300 ms comes from.** 1,000 passages × ~200 tokens = 200,000 tokens. A transformer's
forward pass costs about 2 × (parameters) floating-point operations per token, and BERT-base has
~110M parameters. That is ~44 trillion operations (~44 TFLOP) per rerank call. At the effective
throughput of a current data-centre GPU, that lands at roughly 100–300 ms. At 100 passages it is
~4.4 TFLOP and ~10–30 ms. Chapter 41's latency table for ColBERT reranking uses the same
arithmetic.

**The frontier is Chapter 38.** PLAID makes multi-vector search fast by building specialised
machinery. MUVERA takes the opposite approach: transform the problem so that *ordinary*
single-vector machinery works. That is a more portable answer, and it is where the field is
heading.

---

### Key takeaways

- **Residual compression = nearest-centroid ID + 1–2 bits per dimension of residual.** ~20–36
  bytes instead of 512, a 14–25× reduction, ~7,200 bytes for a 200-token passage at 2 bits.
- **PLAID = each query token's nearest centroids + an inverted index over centroid IDs + rough
  centroid-only scoring (pruned, then unpruned) + exact MaxSim** on a few hundred to a thousand
  passages.
- Realistic cost versus single-vector: ~2× storage and ~5–10× latency, for +3 to +8 nDCG points
  over a same-generation bi-encoder (often 0–3 against the strongest modern dense models),
  largest on needle and multi-constraint queries.
- Build time is substantial: hours for 10M passages, repeated on every migration.
- Token pooling cuts vector count by half almost for free. Do it first.
- ColBERT as a reranker keeps precision and loses recall. Encoding 1,000 passages on the fly
  costs ~100–300 ms, so rerank ~100 or store the vectors.

### What's next

[Chapter 38](./38-muvera.md) asks a bolder question: what if we turned a set of 200 vectors into
a *single* vector whose dot product approximates MaxSim, and used any index we already have?

We now know how ColBERTv2 fits token vectors into a few kilobytes, how PLAID finds the right
few hundred passages among millions, and what that costs against one vector per passage.
