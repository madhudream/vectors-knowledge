# Thinking in Vectors — Full Outline

66 chapters · 9 parts · most chapters a 10–15 minute read (about 13 hours in total).

---

## Part I — Foundations: What a Vector Actually Is

| # | Chapter | In one line |
|---|---|---|
| 1 | [What Is a Vector, Really?](./chapters/01-what-is-a-vector.md) | A vector is a list of numbers that is also a place. |
| 2 | [From Things to Numbers: The Leap of Embedding](./chapters/02-the-leap-of-embedding.md) | How you turn a sentence, a photo, or a product into coordinates. |
| 3 | [The Geometry of Meaning](./chapters/03-geometry-of-meaning.md) | Why "nearby" can mean "similar", and what the Map Room looks like. |
| 4 | [Measuring Similarity: Dot, Cosine, Euclidean](./chapters/04-measuring-similarity.md) | Three rulers, and when each one lies to you. |
| 5 | [Normalization and the Unit Sphere](./chapters/05-normalization.md) | Why almost everyone throws away vector length, and what it costs. |
| 6 | [High Dimensions Are a Strange Country](./chapters/06-high-dimensions.md) | Distance concentration, empty space, and why ANN works anyway. |
| 7 | [Dense vs Sparse Vectors](./chapters/07-dense-vs-sparse.md) | Two ways to describe the same document, and both are still in production. |

## Part II — Where Embeddings Come From

| # | Chapter | In one line |
|---|---|---|
| 8 | [Before Embeddings: One-Hot, Bag-of-Words, TF-IDF, BM25](./chapters/08-before-embeddings.md) | The ancestors that still win benchmarks. |
| 9 | [Word2Vec and the Distributional Hypothesis](./chapters/09-word2vec.md) | "You shall know a word by the company it keeps." |
| 10 | [Transformers and Contextual Embeddings](./chapters/10-transformers-contextual.md) | Why "bank" finally got two meanings. |
| 11 | [From Tokens to One Vector: Pooling](./chapters/11-pooling.md) | CLS, mean, last-token — the quiet decision that changes everything. |
| 12 | [Contrastive Learning: How Embedding Models Are Trained](./chapters/12-contrastive-learning.md) | Pull the pair together, push the crowd apart. |
| 13 | [Bi-Encoders vs Cross-Encoders](./chapters/13-bi-vs-cross-encoders.md) | Precompute and be fast, or read together and be right. |
| 14 | [Asymmetric Search and Instruction-Tuned Embeddings](./chapters/14-asymmetry-and-instructions.md) | A question is not a shorter answer. |
| 15 | [Matryoshka Embeddings](./chapters/15-matryoshka.md) | One model, many dimensions, nested like dolls. |
| 16 | [Choosing and Benchmarking an Embedding Model](./chapters/16-choosing-a-model.md) | How to read MTEB without being fooled by it. |

## Part III — Finding Neighbours

| # | Chapter | In one line |
|---|---|---|
| 17 | [Brute Force and the Flat Index](./chapters/17-brute-force.md) | The honest baseline you should always measure against. |
| 18 | [The Recall–Latency–Memory Triangle](./chapters/18-recall-latency-memory.md) | You may pick two. Everything in Part IV is a way of picking. |
| 19 | [Measuring Retrieval Quality](./chapters/19-measuring-retrieval-quality.md) | Recall@k, MRR, nDCG, and the difference between recall and *recall*. |
| 20 | [The ANN Family Tree](./chapters/20-ann-family-tree.md) | Hash, partition, cluster, or walk a graph. There is no fifth idea. |

## Part IV — Index Structures

| # | Chapter | In one line |
|---|---|---|
| 21 | [LSH: Hashing Things That Are Alike](./chapters/21-lsh.md) | Deliberately bad hash functions, and why that is genius. |
| 22 | [Trees: KD-Trees and Annoy](./chapters/22-trees-and-annoy.md) | Splitting space, and why it stops working above ~20 dimensions. |
| 23 | [IVF: Clustering the Space](./chapters/23-ivf.md) | The library-wing method: only search the wings that matter. |
| 24 | [Product Quantization](./chapters/24-product-quantization.md) | 3 KB per vector becomes 96 bytes, and search still works. |
| 25 | [IVF-PQ and OPQ in Practice](./chapters/25-ivfpq-and-opq.md) | The workhorse of billion-scale search, tuned. |
| 26 | [Scalar and Binary Quantization](./chapters/26-scalar-binary-quantization.md) | 32× compression with one `> 0`, then rescore. |
| 27 | [TurboQuant: Near-Optimal Quantization Without Codebooks](./chapters/27-turboquant.md) | Rotate the problem away: near-optimal bits, no training, no staleness. |
| 28 | [HNSW I: Small Worlds and Skip Lists](./chapters/28-hnsw-1-small-worlds.md) | Six degrees of separation, built on purpose. |
| 29 | [HNSW II: Building the Graph](./chapters/29-hnsw-2-building.md) | M, efConstruction, and the pruning heuristic that matters most. |
| 30 | [HNSW III: Searching and Tuning](./chapters/30-hnsw-3-searching.md) | efSearch, the recall curve, and how to pick a point on it. |
| 31 | [HNSW in Production](./chapters/31-hnsw-production.md) | Deletes, updates, memory bills, and the rebuild you will eventually run. |
| 32 | [DiskANN: Billion-Scale on SSD](./chapters/32-diskann.md) | When RAM is the budget, not the bottleneck. |
| 33 | [Filtered Vector Search](./chapters/33-filtered-search.md) | The hardest easy problem in the whole field. |

## Part V — Beyond One Vector

| # | Chapter | In one line |
|---|---|---|
| 34 | [The Single-Vector Bottleneck](./chapters/34-single-vector-bottleneck.md) | You cannot compress a page into 768 numbers without losing something. |
| 35 | [ColBERT I: Late Interaction](./chapters/35-colbert-1-late-interaction.md) | Keep every token's vector, and compare them at the end. |
| 36 | [ColBERT II: MaxSim and Training](./chapters/36-colbert-2-maxsim.md) | The scoring function, and why it is so hard to beat. |
| 37 | [ColBERT III: ColBERTv2, PLAID, and Serving](./chapters/37-colbert-3-serving.md) | Making hundreds of times more vectors affordable. |
| 38 | [MUVERA: Multi-Vector Made Single](./chapters/38-muvera.md) | Turn a bag of vectors into one vector whose dot product mimics MaxSim. |
| 39 | [SPLADE and Learned Sparse Retrieval](./chapters/39-splade.md) | A neural network that writes into an inverted index. |
| 40 | [Hybrid Search and Fusion](./chapters/40-hybrid-search.md) | Dense plus sparse plus RRF: the most reliable free win in retrieval. |
| 41 | [Rerankers](./chapters/41-rerankers.md) | The largest precision win once retrieval works. |

## Part VI — Vectors Beyond Text

| # | Chapter | In one line |
|---|---|---|
| 42 | [Multimodal Embeddings: CLIP and the Shared Space](./chapters/42-clip.md) | Put pictures and words in the same Map Room. |
| 43 | [SigLIP and Modern Vision-Language Encoders](./chapters/43-siglip.md) | What changed after CLIP, and why it matters for documents. |
| 44 | [The Broken Promise of OCR Pipelines](./chapters/44-ocr-broken-promise.md) | Everything a parser throws away before you ever embed it. |
| 45 | [ColPali I: Documents as Images](./chapters/45-colpali-1-documents-as-images.md) | Delete the parser. Screenshot the page. Search that. |
| 46 | [ColPali II: Inside the Model](./chapters/46-colpali-2-inside.md) | PaliGemma, patches, projection, and late interaction over pixels. |
| 47 | [ColPali III: Production Reality](./chapters/47-colpali-3-production.md) | ColQwen2, storage maths, pooling tricks, and when not to use it. |
| 48 | [Audio, Video, Code, Graphs, and Users](./chapters/48-other-modalities.md) | Everything else that has been squeezed into the Map Room. |

## Part VII — RAG

| # | Chapter | In one line |
|---|---|---|
| 49 | [RAG From First Principles](./chapters/49-rag-first-principles.md) | Why retrieve at all, and what the Scholar actually needs. |
| 50 | [Chunking](./chapters/50-chunking.md) | The most underrated decision in the entire pipeline. |
| 51 | [Metadata and Structure](./chapters/51-metadata-and-structure.md) | The large share of retrieval quality that has nothing to do with vectors. |
| 52 | [Query Understanding](./chapters/52-query-understanding.md) | Rewriting, decomposition, HyDE, and multi-query. |
| 53 | [Context Assembly](./chapters/53-context-assembly.md) | What actually goes in the prompt, in what order, and why. |
| 54 | [Evaluating RAG](./chapters/54-evaluating-rag.md) | Golden sets, faithfulness, and the eval you can build in a day. |
| 55 | [RAG Failure Modes: A Field Guide](./chapters/55-rag-failure-modes.md) | Twelve ways it breaks, and the symptom that identifies each. |
| 56 | [Agentic Retrieval](./chapters/56-agentic-retrieval.md) | Search as a tool the model calls, not a step in a pipeline. |

## Part VIII — Scale and Production

| # | Chapter | In one line |
|---|---|---|
| 57 | [Architecture for Millions of Documents](./chapters/57-architecture-at-scale.md) | The reference design, and where it bends. |
| 58 | [Sharding, Replication, and Routing](./chapters/58-sharding-and-routing.md) | How one index becomes forty, and who decides which to ask. |
| 59 | [Freshness and Index Maintenance](./chapters/59-freshness.md) | Streaming updates, tombstones, and the compaction you must schedule. |
| 60 | [Cost Engineering](./chapters/60-cost-engineering.md) | The arithmetic of a hundred million vectors, in dollars per month. |
| 61 | [Image-Heavy RAG, End to End](./chapters/61-image-heavy-rag.md) | A complete design for a million scanned, chart-filled pages. |
| 62 | [Multi-Tenancy, Privacy, and Embedding Inversion](./chapters/62-security-and-tenancy.md) | Embeddings are not anonymised. Plan accordingly. |
| 63 | [Model Migration and Drift](./chapters/63-migration-and-drift.md) | Changing the embedding model without taking search down. |

## Part IX — Ninja Tier

| # | Chapter | In one line |
|---|---|---|
| 64 | [Vectors as Agent Memory](./chapters/64-agent-memory.md) | Why "just embed the conversation" fails, and what works. |
| 65 | [The Frontier](./chapters/65-the-frontier.md) | Where this field is going, and which bets look safe. |
| 66 | [The Ninja's Field Manual](./chapters/66-field-manual.md) | The book's key decision trees, formulas and defaults in one place. |
