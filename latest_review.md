# latest_review.md — full review of "Thinking in Vectors"

Review started 2026-09-16 (after FIXES.md was applied). Each chapter's findings were written to this file as soon as that chapter
was finished, then the file was sorted into chapter order and this summary was added. Severity: **[MUST]** = wrong fact, broken code/maths, contradiction between chapters, or something a
novice reader cannot follow. **[SHOULD]** = inconsistency, style-rule miss, unclear passage. **[NIT]** = minor.

Each finding quotes the text so it can be located after line numbers drift.

---

## Summary (read this first)

**Status: all fixes applied on 2026-09-16.** Every finding below now has a "Fix status" record under its chapter.
A copy of the book before these fixes is in `_backup_before_review_fixes_2026-09-16.tar.gz`.

**Post-fix verification:** every change was then re-read by independent reviewers (last section of this file). No fix
introduced a MUST-level error and none was incomplete. The 7 SHOULD-level and 35 NIT-level items it found were then
all fixed and re-verified (see the end of that section).

| Severity | Fixed as suggested | Fixed differently | Skipped |
|---|---|---|---|
| MUST | 12 | 0 | 0 |
| SHOULD | 53 | 10 | 2 |
| NIT | 178 | 18 | 11 |

Counts come from the fixers' records and include a few small code notes that reviewers raised outside the numbered
findings. The 2 skipped SHOULDs are a readingTime note (superseded, see Cross-chapter checks) and the OUTLINE "no fifth
idea" line, which stays because Ch 20 now agrees with it. Skipped NITs are readingTime notes, source line-wrapping, and
items the reviewer marked as needing no change.

**Verdict before fixing: the book was in good shape, with 12 real errors, all now fixed.** Structure, links, the running scenario and the numbers
shared between chapters are consistent across all 66 chapters. Every FIXES.md item was applied. No problem is widespread.
Each MUST is a local fix to one sentence, one code line or one table row. The lead reviewer re-checked all 12 MUST findings
against the chapter text, and ran the code for Ch 26.

| Severity | Count |
|---|---|
| MUST | 12 |
| SHOULD | 63 |
| NIT | 186 |

Chapters with no MUST and no SHOULD finding: 1, 2, 3, 7, 8, 9, 10, 13, 15, 16, 17, 21, 22, 24, 29, 34, 35, 42, 49, 57.

### The 12 MUST fixes, in suggested order

**Code that gives wrong results**
1. **Ch 26.** `encode_sq` never clips, so values outside the calibrated range wrap around in int8 and flip sign. With a range
   of ±0.15, the value 0.16 encodes as −120 instead of 127. Wrap the expression in `np.clip(..., -128, 127)`.
2. **Ch 31.** The nightly recall check computes ground truth on a sample of the corpus but queries the index over the whole
   corpus, so a perfect index scores about the sample fraction. Sample the queries, and compute their exact top-10 over the
   full corpus.

**Explanations that are wrong**
3. **Ch 38.** "The raw dot product is about `reps` times Chamfer. The constant does not change the ranking" is false. The
   ratio falls as pages get longer (measured 7.8 at 4 tokens, 1.2 at 200), so long pages are scored low. Call the FDE a
   ranking proxy that exact MaxSim reranking corrects.
4. **Ch 39.** log(1+x) is applied per position before pooling. It flattens how strongly the model votes for a word, not how
   often. It is not BM25's k1 rediscovered (body and Key takeaways).
5. **Ch 43.** The SigLIP bias explanation is backwards. The −10 start puts training near the true prior, so the flood of
   "no" cells does not cause huge early corrections (paper §3.2).
6. **Ch 30.** "Stops exactly when more walking cannot help" is false, and the chapter's own page 530 → page 1,140 example
   disproves it. The stopping rule is a bet that a longer shortlist makes less often.
7. **Ch 65.** Residual compression or TurboQuant alone does not bring a token vector to "a few bytes". Ch 37 gives 20–36
   bytes. Only the 32-d, 2-bit combination in the next paragraph reaches 8 bytes.

**Worked examples and tables that contradict other chapters**
8. **Ch 53.** The worked example silently reorders the reranker's list. With the order the chapter gave earlier,
   `order_for_attention` puts page 1,140 in the middle, not last. Say that expanding the section lifts page 1,140 to second.
9. **Ch 61.** Page 3,507 contradicts Ch 44–46: different table columns, "SS0" instead of "S5O", and a different page for the
   120 seats. Two reviewers found this independently.
10. **Ch 58.** `hash(doc_id) % N` is not "simple to add shards". Going from 40 to 41 shards moves 97.6% of vectors. Name
    consistent hashing or fixed virtual buckets.
11. **Ch 66.** The "Why is this answer wrong?" tree checks the unanswerable case after the retrieval/generation split.
    Ch 55 checks it first.
12. **Ch 66.** The IVF `nprobe` default "16–32" matches neither Ch 23 ("16–64") nor Ch 25 ("8–64").

### Issues that span several chapters (fix them together)

- **"Fifth idea."** Ch 20's Ninja note says "There is a fifth idea… learned indexes". Ch 20's body, its Key takeaways and
  OUTLINE say there is none, and Ch 65 cites Ch 20 for the fifth idea.
- **"Biggest win."** Ch 40 and Ch 41 both claim to be the largest quality win. Ch 40's own numbers favour fusion.
- **Keyword rescue of page 1,140.** Ch 11 and Ch 35 rely on "single sign-on" matching "SSO", which BM25 cannot do.
- **Cost levers.** Ch 60's lever table ("4–30× RAM", quantization ranked second) contradicts Ch 60's own worked example
  (under 5% saving), and is copied into Ch 66.
- **Deletion.** Ch 59 hands the audited deletion path to Ch 62, which never builds it. Ch 62's code hides deleted vectors
  with a filter instead of removing them.
- **ColPali loss.** Ch 46 calls it Ch 12's InfoNCE. The paper uses a hardest-negative softplus loss. Ch 48's "same four lines
  every time" repeats the claim.
- **Padding.** Ch 46 zeroes padding vectors, but Ch 36 says padding must be masked with −∞. This is harmless for fixed-size
  page images, but needs a note.
- **Model axes vs rotation.** Ch 26 says model axes "usually work better than random directions". Ch 27's premise is the
  opposite.
- **Temperature direction.** In Ch 42 it divides the logits, while in Ch 43 it multiplies them. Neither chapter says so.
- **Acme's size.** Ch 37 says Acme one day "grows to 10 million passages". Ch 17, 24–26 and 35 say it already has about
  100 million chunks.
- **SSD sizing.** Ch 32 sizes 1B nodes at 3.5 TB, but one 4 KB block per node is about 4.1 TB. Ch 60's 100M figure has the
  same gap.
- **Binary memory.** Ch 18 gives binary index memory as 0.08× in one table and 0.03× in the Ninja notes.

### Notes on this review

- **Two passes.** Chapters 1–4, 8–10, 14–15, 21–22, 28, 34, 42 and 49 were reviewed in a first pass that stopped at a usage
  limit. The rest were reviewed in a second pass with the same checklist.
- **Verification level.** MUST findings were re-verified by the lead reviewer. SHOULD and NIT findings are as each reviewer
  reported them.
- **Recall-based findings.** A few SHOULD findings rest on a reviewer's memory of a paper: Ch 37 PLAID stages, Ch 39 BGE-M3
  sparse output, Ch 33 filter-aware graphs. Check the paper before editing those.
- **Reading times.** Per-chapter notes that call `readingTime` low counted words inside code blocks. With code excluded,
  every chapter is within one minute, so those notes can be ignored. This includes Ch 04's SHOULD.
- **Ch 38 FDE size.** The SHOULD about 2,560-d vs ~4,000-d is a presentation gap, not an error. The chapter already says its
  example is 2,560-d and ~4,000-d is typical, but it never shows settings that reach 4,000.

### Count per chapter

| Ch | MUST | SHOULD | NIT |
|---|---|---|---|
| 01 | 0 | 0 | 2 |
| 02 | 0 | 0 | 2 |
| 03 | 0 | 0 | 1 |
| 04 | 0 | 1 | 2 |
| 05 | 0 | 1 | 2 |
| 06 | 0 | 2 | 2 |
| 07 | 0 | 0 | 2 |
| 08 | 0 | 0 | 2 |
| 09 | 0 | 0 | 2 |
| 10 | 0 | 0 | 3 |
| 11 | 0 | 1 | 2 |
| 12 | 0 | 1 | 2 |
| 13 | 0 | 0 | 1 |
| 14 | 0 | 1 | 2 |
| 15 | 0 | 0 | 2 |
| 16 | 0 | 0 | 5 |
| 17 | 0 | 0 | 3 |
| 18 | 0 | 1 | 2 |
| 19 | 0 | 1 | 2 |
| 20 | 0 | 1 | 5 |
| 21 | 0 | 0 | 2 |
| 22 | 0 | 0 | 4 |
| 23 | 0 | 1 | 2 |
| 24 | 0 | 0 | 2 |
| 25 | 0 | 1 | 3 |
| 26 | 1 | 1 | 1 |
| 27 | 0 | 2 | 4 |
| 28 | 0 | 1 | 3 |
| 29 | 0 | 0 | 4 |
| 30 | 1 | 1 | 2 |
| 31 | 1 | 1 | 4 |
| 32 | 0 | 2 | 5 |
| 33 | 0 | 1 | 4 |
| 34 | 0 | 0 | 3 |
| 35 | 0 | 0 | 4 |
| 36 | 0 | 1 | 1 |
| 37 | 0 | 2 | 3 |
| 38 | 1 | 2 | 0 |
| 39 | 1 | 1 | 1 |
| 40 | 0 | 2 | 2 |
| 41 | 0 | 2 | 3 |
| 42 | 0 | 0 | 2 |
| 43 | 1 | 1 | 3 |
| 44 | 0 | 2 | 3 |
| 45 | 0 | 1 | 3 |
| 46 | 0 | 3 | 4 |
| 47 | 0 | 1 | 3 |
| 48 | 0 | 1 | 5 |
| 49 | 0 | 0 | 2 |
| 50 | 0 | 1 | 5 |
| 51 | 0 | 2 | 2 |
| 52 | 0 | 2 | 3 |
| 53 | 1 | 0 | 3 |
| 54 | 0 | 3 | 2 |
| 55 | 0 | 1 | 2 |
| 56 | 0 | 1 | 3 |
| 57 | 0 | 0 | 4 |
| 58 | 1 | 2 | 3 |
| 59 | 0 | 1 | 4 |
| 60 | 0 | 3 | 4 |
| 61 | 1 | 0 | 1 |
| 62 | 0 | 2 | 3 |
| 63 | 0 | 1 | 2 |
| 64 | 0 | 1 | 3 |
| 65 | 1 | 0 | 4 |
| 66 | 2 | 1 | 3 |
| README + OUTLINE | 0 | 2 | 7 |

---

## 0. Mechanical checks (whole book)

Ran on all 66 chapters: frontmatter `chapter`/`slug`/`prev`/`next` chain, H1 = title, relative links, "Chapter N" references
in range, the seven standard sections, "In simple words" present, Python code blocks parse, running-scenario constants.

- **Clean:** prev/next chain intact for 1→66; no broken `./` links; every `Chapter N` reference is in 1–66; every Python
  block parses; H1 matches frontmatter title in all 66 files; Part labels match README (I: 1–7, II: 8–16, III: 17–20,
  IV: 21–33, V: 34–41, VI: 42–48, VII: 49–56, VIII: 57–63, IX: 64–66).
- **Clean:** running scenario is consistent everywhere it is mechanically checkable. "page 212" ×255, "page 1,140" ×257
  (no "1140" or "213" variants), "40,000 pages" ×50, "Security add-on" ×149 (2 lower-case "security add-on"),
  "under 50 seats" ×94. Side characters Globex (120 seats, pages 3,507–3,508) and the 30-seat Basic customer are used
  consistently across Ch 44, 45, 52, 61, 64.
- **[SHOULD] Ch 65** has no "Under the hood", "What people get wrong" or "Ninja notes" section. README ("How every chapter
  works") says every chapter has them. Either add short versions or soften the README to "almost every".
- **[SHOULD] Ch 66** has none of the seven standard sections (no one-paragraph version, no "We will cover", no takeaways,
  no "What's next"). That is defensible for a reference manual, but README does not say so. Add one sentence to README
  ("Chapter 66 is a reference and breaks the template") or add a one-paragraph version at the top of Ch 66.
- **[NIT] Semicolons in prose** (the style rule in FIXES §1.1 device 11 drives these to 0): one each in Ch 12, 26, 27,
  35, 36, 38, 59. Em-dashes above three per chapter: Ch 7 (4), Ch 50 (5), Ch 62 (5), Ch 64 (5).
- **[NIT] FIXES §Part 4 verify** `grep "50–100×"` still hits `17-brute-force.md` ("Throughput per query is often tens of
  times higher, and can reach 50–100×."). This is a *different* claim (batch throughput) from the one the check was
  written for (PQ storage price in Ch 24) and is plausible as written. False positive, no change needed.
- **Tooling note:** NumPy is not installed in the system Python, so the book's code blocks cannot be run as-is on this
  machine. Reviewers below used a scratch virtualenv.

#### Fix status (applied 2026-09-16)

- Semicolon counts above were partly false positives. In Ch 35, 36 and 38 the ";" was LaTeX spacing (`\;`) inside a
  formula, now `\,`. In Ch 12, 26, 27 and 59 it was in the frontmatter `summary`, now split into sentences. Three
  semicolons remain inside Ch 59's maintenance-calendar table cells, which Ch 66 copies word for word.
- Em-dashes in teaching sections of Ch 07, 50, 62 and 64 were cut to at most one per section.
- README now says Chapter 65 is an essay and Chapter 66 a reference, so both skip parts of the template.
- The lead aligned the frontmatter `part` of Ch 01–07 to "Part I — Foundations: What a Vector Actually Is" (OUTLINE's heading).
- Verification after fixing: prev/next chain, H1 = title, OUTLINE titles and links, all `./` links, every `Chapter N`
  reference, all seven standard sections (Ch 1–64), every Python block parses, no chapter lost more than 3% of its words,
  and none of the old wrong phrases remain anywhere in the book.

---

## Cross-chapter checks, batch 1 (whole book)

- **Reading times are fine.** Measured on prose only (code blocks excluded) at 230 words per minute, every chapter's `readingTime`
  is within one minute of the estimate. Several per-chapter NITs below say "readingTime is low". Those counts included code
  blocks, so treat them as optional. The stated times add up to 798 minutes, about 13.3 hours, which matches OUTLINE's
  "about 13 hours". Ch 66 uses `readingTime: "Reference — keep open while building"`, which is deliberate.
- **Shared constants agree across chapters.** Checked by grep in every chapter where they appear:
  3,072 bytes for a 768-d float32 vector (18 chapters); 128-d for ColBERT and ColPali token vectors (19 chapters);
  ColPali ~1,030 vectors per page = 1,024 image patches plus the prompt tokens (Ch 45, 46, 47, 66); ColPali ~527 KB per page
  (Ch 45, 47, 66); MUVERA FDE ~4,000-d (Ch 38, 47, 61, 66); ColQwen2 ~750 patches (Ch 47, 61); HNSW M = 16 default with
  M = 32 as the high-recall option and efConstruction = 200 (Ch 29, 30, 31, 66); RRF k = 60 (Ch 40); BM25 k1 = 1.2,
  b = 0.75 (Ch 8); ColBERTv2 36 bytes per token (Ch 34, 47).
- **Deliberate differences, not errors.** Recency half-life is ~180 days for documents (Ch 51, Ch 66) and 30 days for
  conversational memory (Ch 64, which says why). Chunk sizes vary by context: 300-word chunks (Ch 8), 512-token (Ch 50),
  400-token (Ch 65).

## Chapter 01 — What Is a Vector, Really?  (reviewer: chapters 01–07)

- No MUST or SHOULD issues found. All arithmetic recomputed and correct: gaps [3,2,1]/[6,2,2]/[5,-3,5]; norms 3.742 and 6.633 (filter would be 7.681); 768 × 4 = 3,072 bytes. Cross-references (Ch 2, 4, 6, 7, 26, 40, 60) all match OUTLINE.md. readingTime "10 min" matches 2,431 words.
- **[NIT]** "embedding space" is used in the cast section before "embedding" is defined two sections later. Quote: "**The Map Room** is the embedding space." Fix: add a half-sentence gloss, e.g. "the embedding space (the room of learned positions; we define embedding at the end of this chapter)", or reorder so "Why hand-picked numbers are not enough" precedes "Meet the cast".
- **[NIT]** The filter-coffee gap line says "big gaps in two places" but all three gaps (5, −3, 5) are at least as large as the latte's "big gap". Quote: "espresso - filter    = [9-4, 4-7, 8-3] = [5, -3, 5]   → big gaps in two places". Fix: "big gaps everywhere" or "big gaps in all three places".
- Style: no notable violations (no semicolons, no em-dashes in teaching text, "we" used throughout, bold-equation definitions present).
- FIXES.md items for this chapter: all applied (cast introduced after "the Map Room" with all five characters and LLM expanded; "remaining 65 chapters").
- Code: 1 block, runs clean; printed values in comments match (3.742, 6.633).

#### Fix status (applied 2026-09-16)

- [NIT] "embedding space" used before "embedding" is defined — fixed: added a half-sentence gloss and a pointer to the chapter's last section
- [NIT] Filter-coffee line says "big gaps in two places" — fixed: now "big gaps everywhere"
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 02 — From Things to Numbers: The Leap of Embedding  (reviewer: chapters 01–07)

- No MUST issues found. Arithmetic recomputed and correct: 40,000 × 768 × 4 = 122,880,000 bytes (~123 MB); 2,000 words × 6 chars ≈ 12 KB; 384 × 4 = 1,536 bytes. Running scenario (page 212 / page 1,140 / Security add-on / under 50 seats / 40,000 pages) consistent with Ch 1. Cross-references (Ch 7, 9, 10, 12, 40, 50, 63, Parts V and VI) all match OUTLINE.md. readingTime "11 min" matches 2,680 words.
- **[NIT]** The keyword-search example says the rival page shares three words with the question including "company", but "company" appears only in the page *title* ("Closing a company account") while the quoted body has "team" and "accounts" only. A novice counting words in the quoted sentence will find two, not three. Quote: "That page shares three words with the question: \"team\", \"company\" and \"accounts\"." Fix: say "its title and text together share three words" or quote a body sentence that contains "company".
- **[NIT]** Section header says "two properties" but the lead sentence says "two jobs". Quote: "A useful embedding has exactly two jobs." Fix: "two properties" for consistency with the heading and the bold labels.
- Style: no notable violations (no semicolons or em-dashes in teaching text; analogy has explicit mapping list; "Do not worry" callout present for the model-name table).
- FIXES.md items for this chapter: all applied ("gibberish **tokens** (the word-fragments...)", "hundreds of distinct facts", "three whole stretches of this book").
- Code: 1 block (sentence_transformers, not runnable here); read clean — BAAI/bge-small-en-v1.5 is 384-d, so `(384,)` is right.

#### Fix status (applied 2026-09-16)

- [NIT] Rival page "shares three words" but "company" is only in title — fixed: now "Its title and text together share three words"
- [NIT] Heading says "two properties", lead sentence says "two jobs" — fixed
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 03 — The Geometry of Meaning  (reviewer: chapters 01–07)

- No MUST or SHOULD issues found. Toy analogy arithmetic checked ([9−1+1, 8−8+1] = [9, 1] = queen). The Chapter 55 promises ("why that line moves depending on the query", "how to pick the floor", "turns this into a procedure") are now delivered by Ch 55 "Failure 9, part 1: calibrating the distance floor" and "part 2: why the right floor moves with the query". Cross-references (Ch 1, 2, 4, 5, 9, 55, Part IV, Part VII) match OUTLINE.md. "We will cover" bullets match the five section headings. readingTime "11 min" matches 2,626 words.
- **[NIT]** The anisotropy band is stated as a fact for "many models" with no source. Quote: "In many models, almost every score lands somewhere between about 0.55 and 0.95." Unverified rather than wrong; consider softening to "in many sentence-embedding models we have measured" or citing Ethayarajh (2019).
- Style: no notable violations (one em-dash only in frontmatter summary; no semicolons; analogies carry explicit mapping lists; k, ANN, RAG, cosine similarity, anisotropy, distance floor all get bold definitions before use).
- FIXES.md items for this chapter: all applied (§1.2 "empty regions" rewrite present with pet-insurance example, wrong answer then right answer; cosine gloss before the anisotropy measurement and in "Cosine 0.9" item; k defined; code comments "multiply-and-add = the dot product, Chapter 4" and "unit length, Chapter 5" present; Ch 55 fix delivered so the Ch 55 sentences are legitimately kept).
- Code: 1 block (sentence_transformers); the NumPy part was run on synthetic unit vectors and executes clean (randint unpack, masked diagonal, max(1).mean() all correct).

#### Fix status (applied 2026-09-16)

- [NIT] Anisotropy band 0.55–0.95 stated without source — fixed differently: softened "many models" to "some popular models" and cited the E5 model card (scores mostly 0.7–1.0)
- Code: no code changed
- Cross-chapter follow-ups: none
- [Lead follow-up from Ch 11 fixer] "model card" used before Ch 11 defines it — fixed: glossed at first use with a pointer to Chapter 11.

## Chapter 04 — Measuring Similarity: Dot, Cosine, Euclidean  (reviewer: chapters 01–07)

- No MUST issues found. Every worked number recomputed and correct: dot products 50 / 24 / 54; norms 5, 10, 5, 12.166; cosines 1.00 / 0.96 / 0.888; Euclidean 5.00 / 1.414 / 8.062; normalized vectors; normalized dot products 1.00 / 0.96 / 0.888; identity ‖a−b‖² = 2 − 2(a·b) gives 0.00 / 0.283 / 0.474 (chapter's 0.28 / 0.47 correct). The three-rulers-three-winners example (C by dot, A by cosine, B by Euclidean) is genuinely demonstrated by the numbers. Cross-references (Ch 1, 3, 5, 17, 26, 36, Parts V and VI) match OUTLINE.md. "We will cover" bullets match the seven headings.
- **[SHOULD]** Frontmatter `readingTime: "11 min"` understates the length; the chapter is 3,089 words, about 13 minutes at 230 wpm. Fix: "13 min".
- **[NIT]** The "When to use which one" table carries a semicolon and two em-dashes in a teaching section. Quote: "Fastest; identical ranking to cosine", "Count differing bits — Chapter 26", "A different object entirely — Chapter 36". Fix: "Fastest, identical ranking to cosine", "Count differing bits (Chapter 26)", "A different object entirely (Chapter 36)".
- **[NIT]** The claim that a CPU core does "billions of these per second" for multiply-adds is fine, but "A GPU ... does far more" is left unquantified; harmless. No change required.
- Style: otherwise clean (bold-equation definitions for dot, cosine, Euclidean, MIPS, SIMD, Hamming, recommender; "In simple words"/"Means," after every formula; "Do not worry" callout for the last two table rows).
- FIXES.md items for this chapter: all applied (θ gloss "Here θ (theta) is the angle between the two vectors"; argpartition comment "a partial sort: much cheaper than fully sorting a million scores"; SIMD gloss; quantization gloss in What people get wrong).
- Code: 1 block; the toy part runs and matches the comments exactly ([50, 24, 54], [1.0, 0.96, 0.888], [5.0, 1.414, 8.062]); the batched argpartition/argsort part was run on synthetic normalized data and returns the correct top-10 in descending order.

#### Fix status (applied 2026-09-16)

- [SHOULD] readingTime "11 min" should be "13 min" — skipped: superseded by the brief (all chapters within one minute when code is excluded)
- [NIT] Semicolon and two em-dashes in "When to use which one" table — fixed (comma and parentheses); also changed "Most models" to "Many models" before the 0.55–0.95 range to match Ch 03's softened wording
- [NIT] "A GPU ... does far more" left unquantified — skipped: reviewer says no change required
- Code: no code changed
- Cross-chapter follow-ups: optional, chapters/40-hybrid-search.md line 94 says cosine scores are "usually crammed into about [0.55, 0.95] (Chapter 3)"; Ch 03 now says this holds in "some popular models", so "often" would match better

## Chapter 05 — Normalization and the Unit Sphere  (reviewer: chapters 05–07, 11–13)

No MUST issues found. All worked arithmetic recomputed with Python and correct: norms 2, 2 and 12.166 ("about 12.17"), q1/q2 unit length, raw dot products 2.00/1.60/10.80 and 1.60/2.00/5.28, normalized security [0.986, 0.164] ("[0.99, 0.16]"), normalized scores 1.00/0.80/0.888 and 0.80/1.00/0.434. The "normalizing only the question changes nothing" note is right. Random 768-d unit pairs: mean 0.0004, std 0.0362, 94.8% inside ±0.07 (matches "about 0.000 and 0.036", "about 95%"). Cross-references (Ch 3 anisotropy/0.7 floor and calibration, Ch 4 toy 2-number model and identical rankings, Ch 6, 7, 8, 12, 26, 39) all match OUTLINE and the target chapters. The length-carries-signal list (two-tower popularity norms, learned-sparse term weights, confidence in magnitude) and the "name what magnitude encodes" rule are sound.

- **[SHOULD]** The quantization benefit is stated for binary quantization too, but binary quantization (Ch 26: bit = 1 if v_i > 0) keeps only signs, which no per-vector rescaling changes, so it needs no range and no calibration. Quote: "Scalar and binary quantization (Chapter 26) work by mapping a range of values onto a few bits." Fix: "Scalar quantization (Chapter 26) works by mapping a range of values onto a few bits." and, if wanted, add "(Binary quantization keeps only the sign of each number, so length does not affect it.)"
- **[NIT]** Key takeaway is stronger than the body, which only says "Quantization behaves" and gives no size of effect. Quote: "It makes quantization dramatically better behaved." Fix: "It makes scalar quantization better behaved, because one range fits every vector."
- **[NIT]** Whitening subtracts the corpus mean and then re-normalizes, so it is an affine map followed by a normalization, not a "fixed linear transform". Quote: "because it is a fixed linear transform you can fold into the encoder". Fix: "because it is a fixed transform (subtract a mean, multiply by a matrix) you can add as a final step of the encoder".
- Style: clean in teaching sections (no semicolons, no em-dashes in prose, "we" throughout, hiker analogy has explicit mapping, wrong-then-right example present).
- FIXES.md items for this chapter: all applied (quantization glossed in the one-paragraph version).
- Code: 4 blocks (normalize, normalize_batch, ingestion assert, random-pair cosine), all run clean; printed-value comments match; normalize_batch returns zeros, not NaN, for all-zero rows as the prose says.

#### Fix status (applied 2026-09-16)

- [SHOULD] Range/calibration benefit wrongly claimed for binary quantization — fixed: now "Scalar quantization (Chapter 26) works by...", plus a sentence that binary keeps only signs, so length does not affect it
- [NIT] Key takeaway "dramatically better behaved" stronger than body — fixed: "It makes scalar quantization better behaved, because one range fits every vector."
- [NIT] Whitening called a "fixed linear transform" — fixed: "a fixed transform (subtract a mean, multiply by a matrix) you can add as a final step of the encoder"
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 06 — High Dimensions Are a Strange Country  (reviewer: chapters 05–07, 11–13)

No MUST issues found. Recomputed with Python: Experiment 1 (seed 1) reproduces the table exactly (0.003/1.048/379.56, 0.341/2.152/5.31, 3.071/4.886/0.59, 10.283/12.2/0.19), and (12.20 − 10.28) ÷ 10.28 = 0.187 ≈ 0.19. Random 768-d sphere: 1/√768 = 0.0361; simulated best/worst of 40,000 pages 0.150/−0.138, 94.8% inside ±0.07 (matches "about 0.15", "about −0.14", "about 95%"). Hotel: 10¹⁰ = 10 billion, 10⁷⁶⁸ has 769 digits. Acme 768→192: 3,072→768 B, 122.9→30.7 MB. JL growth ln(4×10¹⁰)/ln(4×10⁴) = 2.30. **JL numbers:** the chapter prints O(log n/ε²) and names the Dasgupta–Gupta constants, i.e. k ≥ 4 ln n/(ε²/2 − ε³/3) (identical to 24 ln n/(3ε² − 2ε³)). For n = 40,000 that gives 508.6 at ε = 0.5 ("about 510") and 2,445.4 at ε = 0.2 ("about 2,450"), so both printed numbers follow from the named bound (8 ln n/ε² would give 339 and 2,119; only a log₂ variant gives 734 and 3,528; nothing standard gives ~5,300). TWO-NN estimator checked with a NumPy brute-force stand-in for sklearn: a 10-d Gaussian sheet in 768-d reads 9.7 (uniform 8.7), a 40-d sheet reads 28.7 ("about 30"). Cross-references (Ch 1 "think in three, compute in 768", Ch 3 ANN and page 88, Ch 5, 7, 8, 15, 21, 23, 28, 30, 38; rescoring list 24/26/27/32/37/38) all match OUTLINE and targets.

- **[SHOULD]** The intrinsic-dimension function silently breaks on duplicate vectors, which the chapter itself names as the typical hard corpus ("templated, boilerplate or near-duplicate text"). One duplicate pair (r1 = 0, so mu = r2/1e-12) dropped a true-10 estimate from 9.7 to 7.6 on 2,000 points; three or more identical vectors give r1 = r2 = 0, log(0) = −inf, and the function returns −0.0. Quote: "mu = r2 / np.maximum(r1, 1e-12)". Fix: drop zero-distance points before the ratio, e.g. `keep = r1 > 0; mu = r2[keep] / r1[keep]; return len(mu) / np.log(mu).sum()`, and add "deduplicate first" to Step 1.
- **[SHOULD]** A novice who reads the Ninja number next to the main text will find the JL guarantee for Acme (about 2,450 dimensions at ε = 0.2) is *larger* than the 768 we started with, while the main text presents random projections under "What this means in practice" right after "We can often cut dimensions for almost free". Nothing reconciles the two. Quote: "Random projections work, and that is not obvious." Fix: add one sentence in Ninja notes, e.g. "Note that for Acme this worst-case guarantee is bigger than 768, so JL does not by itself justify shrinking our vectors. Its value is the theory behind LSH and MUVERA. In practice, projections do far better than the bound because ranking only needs nearby distances kept in order."
- **[NIT]** The hotel image adds "continent", "floor" and "wing" without mapping them. Read literally, with ten slots per attribute most pairs of guests share a slot on dozens of attributes, so "Not one pair shares a floor" is not true for any natural reading of "floor". Quote: "**Every page is alone on its own continent.** Not one pair shares a floor. Not one pair shares a wing." Fix: "No two pages share a room. The nearest other guest is many rooms away."
- **[NIT]** The reading guide skips 40–60, and its thresholds are in the estimator's own (low-reading) units, which the text does not say. Quote: "**15–40** is typical for good text embeddings. **Over ~60**, expect to pay for recall". Fix: "**15–40** is typical ... **Over ~40–60**, expect to pay for recall (these are readings from this estimator, which runs low)".
- Style: clean in teaching sections (no semicolons or em-dashes in prose, no "you", hotel and crumpled-paper analogies both mapped, random-dots wrong answer then real-model right answer).
- FIXES.md items for this chapter: all applied. Main text has no efSearch/probes/SIFT1M (moved verbatim to Ninja notes), "a small index searched shallowly will do", PCA glossed, "index library", and per the deliberate deviation the main text does not say "a few hundred dimensions suffice", it says the count depends on point count and tolerance only.
- Code: 2 blocks. Experiment 1 runs clean and prints exactly the commented values. Experiment 2 needs sklearn; read clean for API and indexing (d[:, 1], d[:, 2] after self at column 0) and verified with a NumPy stand-in, apart from the duplicate-vector defect above.

#### Fix status (applied 2026-09-16)

- [SHOULD] intrinsic_dim breaks on duplicate vectors — fixed differently: dedupes the sample with np.unique first and also skips any remaining zero r1, and Step 1 now says to drop exact duplicates and why
- [SHOULD] JL bound for Acme (~2,450) exceeds 768, unreconciled — fixed: Ninja note says JL alone does not justify shrinking, gives its value (LSH, MUVERA), and says real data beats the bound thanks to low intrinsic dimension
- [NIT] Hotel image "continent/floor/wing" unmapped and untrue — fixed: "Every page gets a room to itself. No two pages share a room. The nearest other guest is many rooms away, and almost every room is empty."
- [NIT] Reading guide skips 40–60 and does not say units are estimator's — fixed: says the estimator reads low, adds "Over ~40, expect to pay for recall" and keeps "Over ~60" for encoder-fit check
- Code: Experiment 2 block changed. Extracted verbatim and run with real scikit-learn 1.9.1 (installed into venv): 10-d sheet reads 9.72 clean / 9.72 one duplicate pair / 9.74 with 200 identical / 9.70 with 30% copies (old code: 9.14, 10.15, -0.0), 40-d sheet reads 29.06 clean / 28.23–29.04 with duplicates (old: 24.33, -0.0). The chapter's "about 10" and "about 30" still hold. Experiment 1 unchanged.
- Cross-chapter follow-ups: none

## Chapter 07 — Dense vs Sparse Vectors  (reviewer: chapters 05–07, 11–13)

No MUST or SHOULD issues found. Arithmetic recomputed with Python and correct: 300 of 30,000 = 1% filled; 40,000 × 30,000 × 4 B = 4.8 GB; 10⁶ docs = 120 GB; 300 pairs × 8 B = 2,400 B vs 120,000 B; 250 × 128 × 4 = 128,000 B = 41.7× 3,072 ("roughly 40×"); 200-token passage 102,400 B = 33.3× ("about 30×", matching Ch 34/35 "~30× bytes, ~200× vectors"). SSO-4012 vs SSO-4021 (same digits, different order), page 17,450 for SSO-4012 (matches Ch 8 and Ch 16), page 88 as SSO setup guide (matches Ch 3/6/14/28), and the zero-overlap claim for page 212 vs the paraphrase question all check out. BM25 1994 is right. Lucene sentence is correct (Elasticsearch, OpenSearch and Solr are built on Lucene; Tantivy and Vespa are not). Cross-references (Ch 2, 8, 10, 35, 37, 38, 39, 40, 45, Part IV/V/VI) match OUTLINE; "What's next" links 08.

- **[NIT]** The bold definition and one-paragraph version say "a few hundred", but the chapter's own table gives 128–4,096 and Ch 6 already mentions 1,536-d models, so a novice meets a contradiction within one page. Quote: "**Dense vector = a few hundred numbers, nearly all non-zero, learned by a model.**" Fix: "a few hundred to a few thousand numbers" (same in "A **dense** vector has a few hundred dimensions" and the first key takeaway).
- **[NIT]** Key takeaway drops the SPLADE caveat the body makes: learned sparse vectors do need training. Quote: "Sparse is exact, explainable and needs no training." Fix: "Counting-based sparse (BM25) is exact, explainable and needs no training."
- Style: clean in teaching sections (no semicolons or em-dashes in prose; the two em-dashes are inside table cells; book-index vs reviewer-summary analogy has explicit mapping; each question shows the wrong answer then the right one; plain section titles).
- FIXES.md items for this chapter: all applied (Lucene-family vs independent cousins, posting list defined in bold, "roughly 250 vectors, one per token").
- Code: 1 block (dict sparse score), runs clean and prints 3.9 as commented.

#### Fix status (applied 2026-09-16)

- [NIT] "a few hundred" contradicts the chapter's own 128–4,096 table row — fixed: "a few hundred to a few thousand" in the bold definition, the one-paragraph version and the first key takeaway
- [NIT] Key takeaway drops SPLADE caveat ("needs no training") — fixed: "Counting-based sparse (BM25) is exact, explainable and needs no training."
- [Section 0] Em-dash reduction — fixed: the two em-dashes in the "Dense vs sparse" table became parentheses, so teaching sections now have none (the remaining two are in frontmatter `part` and `summary`, which the brief says not to touch)
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 08 — Before Embeddings: One-Hot, Bag-of-Words, TF-IDF, BM25  (reviewer: chapters 08–13)

No MUST or SHOULD issues found. All worked arithmetic recomputed and correct: log table (0.69 / 2.3 / 4.6 / 13.8), IDF(Pequod) = ln(333,333) = 12.72, IDF(SSO-4012) = ln(40,000) = 10.60, saturation table (1.00 / 1.96 / 2.17 at k1 = 1.2), smoothed BM25 IDF 12.56 ("about 12.6"), BM25 scores 21.20 / 24.68 / 8.31, TF-IDF tie 4 × 12.7 = 50.9. Formula, k1/b ranges, Lucene's "+1 inside the log" and the negative-IDF remark are all correct.

- **[NIT]** The chapter carries two scenarios: Moby-Dick/Pequod for the four teaching sections and Acme (SSO-4012, page 212) in the Note, the "Where counting fails" section and the comparison table. FIXES 1.1 device 5 asks for one scenario per chapter. Quote: "Our example for the whole chapter is *Moby-Dick* and its whaling ship, the **Pequod**." Fix (optional): either keep the Pequod throughout and move the Acme material into a single "Note:", or state up front that the Pequod stands in for a rare identifier like SSO-4012. The current mix is readable, so low priority.
- **[NIT]** Line "Picture a collection of one million documents: novels, articles and encyclopedia entries. A reader types..." is a single unwrapped ~150-character source line while the rest of the file wraps at ~95; cosmetic only.
- Style: no semicolons or em-dashes in teaching sections; bold-equation definitions present for one-hot, bag-of-words, TF-IDF, BM25, log; "we" used consistently. No notable violations.
- FIXES.md items for this chapter: all applied (log base and 12.7/0.7 gloss present; "**stopwords**" defined with examples; "300-word RAG chunks"). Note: "tokens" is in fact already defined in Ch 2 and Ch 7, so the Key takeaways line "exact tokens like `SSO-4012`" is fine.
- Code: 1 block, runs clean; printed comments (0.000001, 21.2, 24.7, 8.3) match actual output (9.98e-7, 21.20, 24.68, 8.31).

#### Fix status (applied 2026-09-16)

- [NIT] Two scenarios (Pequod and Acme) in one chapter — fixed: added one sentence to the intro saying the Pequod stands in for a rare identifier like `SSO-4012` (Chapter 7), which returns later.
- [NIT] Unwrapped ~150-character source line in intro — fixed: the paragraph was rewrapped while applying the previous fix.
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 09 — Word2Vec and the Distributional Hypothesis  (reviewer: chapters 08–13)

No MUST or SHOULD issues found. Softmax table (0.665 / 0.245 / 0.090, sums to 1), sigmoid table (0.98 / 0.88 / 0.50 / 0.12 / 0.02), 1000^0.75 ≈ 178, the ±2 window pairs, the negative-sampling objective and the f(w)^0.75 noise distribution are all correct. Firth 1957, word2vec 2013, king − man + woman ≈ queen (with the input words excluded) are accurate. Cross-references (Ch 2 neural network, Ch 4 dot product, Ch 8 log and e, Ch 10, Ch 12, Ch 14) all point at the right chapters.

- **[NIT]** The code averages the K negatives (`torch.logsigmoid(-neg_score).mean()`) while the formula above sums them (Σ_{k=1}^{K}), so the negative term in the code is 1/K of the formula's. Harmless for a skeleton, but the prose says "`logsigmoid` is log σ from the formula" without noting the difference. Quote: "+ torch.logsigmoid(-neg_score).mean())         # makes numbers smaller". Fix (optional): use `.sum(-1).mean()` for the negative term, or add "(averaged over the K negatives rather than summed)".
- **[NIT]** readingTime "12 min" for ~3,000 words is a touch low (~13 min at 230 wpm).
- Style: no semicolons or em-dashes in teaching sections; every new term (weights, softmax, sigmoid, skip-gram, centre word, hidden layer, pretext task, negatives, polysemous, n-grams) has a bold definition at first use; "NLP" is expanded. No notable violations.
- FIXES.md items for this chapter: all applied (weights gloss, softmax gloss, σ/𝔼 gloss beside the formula, "two lookup tables (two embedding matrices) with no nonlinearity between them").
- Code: 1 block (PyTorch, not runnable here), read clean: shapes (B,D) · (B,K,D) → bmm → (B,K) are correct, no undefined names.

#### Fix status (applied 2026-09-16)

- [NIT] Code averages K negatives, formula sums them — fixed: negative term is now `.sum(-1).mean()`, and one sentence explains `.sum(-1)` (the Σ) and `.mean()` (batch).
- [NIT] readingTime "12 min" slightly low — skipped: readingTime NITs are superseded by the brief.
- Code: Under the hood skip-gram block changed (negative term). Torch not installed, so verified with a line-for-line NumPy port: shape (B, K) and the loss equals minus the batch mean of the per-pair formula exactly.
- Cross-chapter follow-ups: none

## Chapter 10 — Transformers and Contextual Embeddings  (reviewer: chapters 08–13)

No MUST or SHOULD issues found. Attention formula, BERT-base d_k = 64 → √64 = 8, softmax table (0.665 / 0.245 / 0.090), 512² = 262,144 and 1,024² = 1,048,576, "six people" for the six tokens of "we sat on the river bank", twelve layers, BERT 2018 with 15% masking, and the raw-[CLS]-underperforms-averaged-GloVe claim (Sentence-BERT paper) are all correct. Every "Chapter N" reference (7, 8, 9 Problem 3, 11, 12, 49 context window, 50 chunking) points at the right topic; "We will cover" bullets match the eight section headings exactly.

- **[NIT]** "fine-tuning" is used twice in What people get wrong ("without fine-tuning", "a contrastive fine-tune (Chapter 12)") before its bold definition, which only arrives in Chapter 12. It is in a wit section, so allowed by the style rules, but a two-word gloss would not hurt. Quote: "It takes a contrastive fine-tune (Chapter 12) to produce a good sentence encoder." Fix (optional): "a contrastive fine-tune (further training on pairs, Chapter 12)".
- **[NIT]** The code comments use em-dashes ("# lower  — different senses") and the code block calls the model without `torch.no_grad()`. Cosmetic; the code is correct.
- **[NIT]** readingTime "11 min" for ~2,750 words is slightly low (~12 min).
- Style: no semicolons or em-dashes in the teaching sections; the meeting analogy has its full mapping list; every new term (token, tokenizer, transformer, self-attention, feed-forward network, layer, attention, linear projection, Q/K/V, encoder, masked language modelling, decoder-only, causal attention, context window, truncates) gets a bold or inline definition. No notable violations.
- FIXES.md items for this chapter: all applied (decoder-only/causal gloss is now a full bold definition; "averaged GloVe vectors (a word2vec-era static embedding)"; late chunking in Ninja notes reduced to one sentence pointing to Chapter 11).
- Code: 1 block (transformers + torch, not runnable here), read clean: `convert_tokens_to_ids("bank")` is a single WordPiece token in bert-base-uncased, `last_hidden_state[0]` is (tokens, 768), `torch.cosine_similarity(a, b, dim=0)` is valid for 1-D tensors.

#### Fix status (applied 2026-09-16)

- [NIT] "fine-tuning" used before its definition — fixed: glossed both uses inline ("further training for one specific job", "further training on pairs, Chapter 12").
- [NIT] Em-dashes in code comments, no `torch.no_grad()` — fixed: comments use colons, and the model call is wrapped in `with torch.no_grad():`.
- [NIT] readingTime "11 min" slightly low — skipped: readingTime NITs are superseded by the brief.
- Code: Under the hood block changed (no_grad wrapper, comment punctuation). Needs transformers/torch, so checked by careful reading: `out` is still bound before use, shapes and outputs unchanged, and the block parses.
- Cross-chapter follow-ups: none

## Chapter 11 — From Tokens to One Vector: Pooling  (reviewer: chapters 05–07, 11–13)

No MUST issues found. Arithmetic recomputed and correct: 512 × 3,072 B = 1.57 MB/page, × 40,000 = 62.9 GB ("about 63 GB"), 40,000 × 3,072 B = 122.9 MB, 512× smaller; toy padding table averages to [0, 0] unmasked and [0.67, 0.67] masked; 300 − 3 = 297 pad rows; the caveat sentence is 12 words, 12/260 = 4.6% ("under 5%"), 12/100 = 12% ("about an eighth"); page 1,140 at about 260 words matches Ch 34. Pooling facts checked: raw BERT [CLS] trained for next-sentence prediction; BGE v1.5 is CLS-pooled, E5 and all-MiniLM/all-mpnet are mean-pooled, e5-mistral is last-token pooled; SGPT-style position-weighted mean exists; LLM2Vec uses bidirectional attention with mean pooling; late chunking is described correctly. The left-padding explanation ([0, 0, 1, 1, 1] → index 2 = first real token) is right. The "3–8 points of nDCG@10" figure is unverified (no source given). Cross-references (Ch 1, 4, 8, 10 meeting/causal attention/one-vector-per-token ending, 12, 19, 35, 37 single-vector + ColBERT rerank, 40, 50, Part V) match OUTLINE and the target chapters. The nine "We will cover" bullets match the nine headings.

- **[SHOULD]** Two of the three "responses" to the lost caveat do not work on the chapter's own example as written. The question says "single sign-on" and the caveat sentence says "SSO", so BM25 can only match "Pro" (which is on hundreds of pages), and Ch 40 in fact shows page 1,140 coming from dense search, not BM25. The ColBERT illustration also matches "Pro" to "Pro", while Ch 35 explains the rescue as "'single sign-on' matched 'SSO' in the caveat sentence." Quote: "let the query token *Pro* find the document token *Pro* directly, wherever it sits on the page." Fix: for 3, "let the question's *sign-on* tokens find the page's *SSO* token directly, wherever it sits on the page (Chapter 35)". For 2, either say hybrid helps "when the page repeats the question's exact words, such as an error code" or add that page 1,140 would need an SSO ↔ single sign-on synonym for BM25 to catch it.
- **[NIT]** "BGE family" is broader than the claim. The decoder-based BGE models (bge-en-icl, bge-multilingual-gemma2) use last-token pooling, which is the chapter's own "read the model card" lesson. Quote: "BGE uses CLS, E5 uses mean, e5-mistral uses last-token." Fix: "BGE v1.5 and BGE-M3 use CLS" here and "such as BGE v1.5 and BGE-M3" in the body and the table.
- **[NIT]** The `transformers` library is named without a gloss, while `sentence-transformers` gets one. Quote: "If you call `transformers` directly, it is yours to get right." Fix: "If you call Hugging Face's `transformers` library (the lower-level library that runs the model itself) directly, …".
- Style: clean in teaching sections (no semicolons or em-dashes in prose, "you" only in advice, meeting analogy has explicit mapping, wrong answer then right answer with the Scholar's reasoning quoted).
- FIXES.md items for this chapter: all applied. The BGE/E5/e5-mistral correction and the right-padding comment are in. Padding is defined in bold in its own section, with a forward pointer from the first use ("The next section defines both properly") rather than an inline gloss. nDCG@10 is glossed with Ch 19, and the Option C different-meeting sentence is present. The [NICE] "nearly 400 words" was superseded by the 260-word page, which is consistent with Ch 34.
- Code: 1 block (torch, not runnable here). Ported line-for-line to NumPy and run: mean_pool shapes (B,T,1)/(B,D)/(B,1) are as commented and return [0.667, 0.667] for the toy row; cls_pool is correct; last_pool is correct for right padding and returns the first real token for left padding, as the prose says.

#### Fix status (applied 2026-09-16)

- [SHOULD] Hybrid and ColBERT rescues of page 1,140 do not work — fixed: the intro now says only responses 1 and 3 rescue this page. Response 2 says BM25 helps when the page repeats the question's exact words (such as an error code), but cannot connect "single sign-on" to "SSO" without a synonym list, which agrees with Ch 40 (page 1,140 comes from dense search). Response 3 now reads "let the question's *single sign-on* tokens find the page's *SSO* token", matching Ch 35 and Ch 13.
- [NIT] "BGE family" broader than the CLS claim — fixed: "BGE v1.5 and BGE-M3" in the body, the table and Key takeaways.
- [NIT] `transformers` library named without a gloss — fixed: "Hugging Face's `transformers` library (the lower-level library that runs the model itself)".
- Extra (lead note and Ch 14/15 findings): "model card" is now a bold definition at its first use in Ch 11, and the bare "single-item inference" became "embedding one text at a time", so Ch 11 no longer uses "inference" undefined.
- Code: no code changed
- Cross-chapter follow-ups: 03-geometry-of-meaning.md line 222 uses "The model card for the E5 family" bare, before Ch 11's definition. Suggest a short gloss there: "The model card (the documentation page a model's authors publish with it) for the E5 family…".

## Chapter 12 — Contrastive Learning: How Embedding Models Are Trained  (reviewer: chapters 05–07, 11–13)

No MUST issues found. Every worked number recomputed with Python and correct: −log p table (0.105 / 0.693 / 2.303 / 4.605); easy quiz logits 16/6/4 → p = 0.999948 / 0.0000454 / 0.0000061, loss 5.15 × 10⁻⁵ ("≈ 0.00005"); hard quiz logits 16/15/6 → p = 0.7310 / 0.2689 / 0.0000332, loss 0.3133, ratio 6,078 ("about 6,000"); the Enterprise page's share of the push on negatives (∂L/∂logit_j = p_j) is 99.988% at τ = 0.05 and 61.06% at τ = 1 (matches "99.99%" and "about 61%"). InfoNCE formula (positive included in the denominator sum, N − 1 negatives) and the in-batch count (1,024 pairs → 1,023 negatives) are right. The loss code was ported to NumPy and matches the explicit per-query formula to 1e-15. Denoised supervision (ColBERTv2, teacher scores) vs RocketQA (discard teacher-approved negatives), Margin-MSE and KL distillation, GradCache and the Wang–Isola alignment/uniformity split are described correctly. Cross-references (Ch 3, 4, 5, 8, 9 softmax/negative sampling, 13, 16 MTEB, 19, 36 distillation, 41) match OUTLINE and targets. The nine "We will cover" bullets match the nine headings.

- **[SHOULD]** The Ninja note reads a high random-pair similarity as a sign of weak contrastive training, but the best-known counterexample is a heavily contrastive-trained family: the E5 model cards explain that their cosine scores sit around 0.7–1.0 *because* they train InfoNCE at a low temperature (0.01), which is inside this chapter's own "typically 0.01–0.07" range. A novice who checks E5 will conclude it was badly trained. Quote: "A model with a random-pair similarity of 0.85 has usually had little uniformity pressure during training." Fix: "A high random-pair similarity can come from weak uniformity pressure or simply from a very low training temperature (E5's cards note scores of 0.7–1.0 at τ = 0.01). Either way, only the ordering matters."
- **[NIT]** The collapse diagnostic sits awkwardly next to Ch 6, which tells the reader that good text embeddings have an intrinsic dimension of 15–40 and that PCA can remove half to three-quarters of 768 dimensions with tiny loss. The "95% of variance in 50 components" threshold is unverified and does not say how it differs from normal low intrinsic dimension. Quote: "if 95% of variance sits in the first 50 components, you have collapse". Fix: add "(PCA measures flat directions only, so this is a stronger symptom than Chapter 6's curved low intrinsic dimension)" and mark the threshold as a rough rule of thumb.
- **[NIT]** "Any" overstates the gap. Large LLM-based bi-encoders can beat small cross-encoders, and Ch 13 frames the trade as same-size accuracy vs speed. Quote: "A cross-encoder is far more accurate than any bi-encoder". Fix: "A cross-encoder is usually far more accurate than a bi-encoder of similar size".
- Style: clean in teaching sections (no semicolons or em-dashes in prose; the one semicolon is in the frontmatter summary; dog analogy has explicit mapping; Phase/Step lines precede the quiz; wrong answer then right answer with the Scholar's reasoning quoted).
- FIXES.md items for this chapter: all applied (loss and gradient defined before the quiz, in a new "How training works" section; denoised supervision rewritten to teacher *scores* with RocketQA as the discard variant; MTEB glossed with Ch 16; logits comment "raw, unbounded scores").
- Code: 1 block (torch, not runnable here). Read clean (normalize → Q @ D.T / tau → arange labels → cross_entropy is the standard in-batch InfoNCE), and a NumPy port reproduces the formula exactly.

#### Fix status (applied 2026-09-16)

- [Section 0 NIT] Prose semicolon (frontmatter summary) — fixed: split into two sentences.
- [SHOULD] High random-pair similarity blamed on weak training, E5 counterexample — fixed: the Ninja note now names both causes (weak uniformity pressure, or a very low training temperature, citing E5's cards: 0.7–1.0 at τ = 0.01), then says a high score alone does not mean bad ranking.
- [NIT] Collapse diagnostic vs Ch 6 low intrinsic dimension — fixed: threshold marked "a rough rule of thumb" ("suspect collapse"), plus two sentences saying intrinsic dimension follows a curved surface while PCA finds only flat directions, so a healthy model spreads variance over many more components.
- [NIT] "far more accurate than any bi-encoder" overstates — fixed: "usually far more accurate than a bi-encoder of similar size".
- Extra (D3 consistency): "Add a reranker… often the single biggest quality jump available" became "Once retrieval works, it is often the largest remaining quality jump (Chapter 41)".
- Code: no code changed
- Cross-chapter follow-ups: 41-rerankers.md line 16 has the same overstatement ("It is far more accurate than any bi-encoder"). Suggest "usually far more accurate than a bi-encoder of similar size".
- [Lead, verification pass] one overlong paragraph rewrapped, no words changed.

## Chapter 13 — Bi-Encoders vs Cross-Encoders  (reviewer: chapters 05–07, 11–13)

No MUST or SHOULD issues found. Cost arithmetic recomputed and internally consistent: 1M pairs = 10,000 × 100 pairs, so ~20–100 ms per 100 pairs scales to 200–1,000 s = 3.3–16.7 min ("~3–17 minutes"). The fixed ~20–100 ms rerank cost is independent of corpus size, as stated. Late-interaction storage: 200 × 128 × 4 = 102,400 B = 33.3× 3,072 B ("roughly 30×"), 200× the vectors, matching Ch 7, 34 and 35. **Page 212 / page 1,140 rescue:** first-stage ranks (212 at 1, 88 at 2, 2,301 at 3, 45 at 4, 9,012 at 5, 1,140 at 23) and reranked scores (9.1 / 8.4 / 5.2 / 3.9 / 3.1, with 1,140 moving 23 → 2) match Ch 41 exactly, "rank 23" matches Ch 15, 17, 18 and 34, and "pages 212 and 1,140 now on top" after reranking the top 100 is consistent everywhere. (Ch 37's rank 2,300 is a separate 10M-passage illustration, not a contradiction.) The +5–15 nDCG@10 and 20–100 ms figures match Ch 41. The "1 dot product / token vs token / full attention" table, the two-tower note (matches Ch 14 lines 382–391) and the recall-early/precision-late rule are correct. Cross-references (Ch 1, 4, 8, 10, 11, 12, 14, 17, 19, 35, 37, 38, 40, 41, 50, Part IV/V) match OUTLINE. The nine "We will cover" bullets match the nine headings.

- **[NIT]** The code's reranker output will not look like the chapter's raw scores. sentence-transformers' `CrossEncoder` applies a sigmoid by default to single-output models (checked in the installed v6.0.1 source, `get_default_activation_fn`: `nn.Sigmoid()` when `num_labels == 1` unless the model config overrides it), so `cross.predict` returns 0–1 values, not 9.1 and 8.4. The order is the same and the "not comparable across queries" warning still holds. Quote: "These are raw model scores, not probabilities (and toy numbers too)." Fix: add to Under the hood "(By default `predict` squashes each score through a sigmoid into 0–1. That keeps the order but does not calibrate the scores across queries.)" or pass `activation_fn=torch.nn.Identity()` to get raw logits.
- Style: clean in teaching sections (no semicolons or em-dashes in prose, no "you" in teaching text, hiring analogy has explicit mapping, Phase/Step lines before the diagram, wrong half-answer then correct answer with the Scholar's reasoning quoted, Advantages/Disadvantages and "When to use which one" present).
- FIXES.md items for this chapter: all applied ("The Scholar (the LLM, Chapter 1) reads what survived."; "roughly **30× the bytes** and **200× the number of vectors**, before any compression"; "raw, unbounded scores (logits, Chapter 12)").
- Code: 1 block (sentence-transformers, not runnable here; `index` is an explicitly named placeholder). Read clean: `encode(..., normalize_embeddings=True)`, `predict` on a list of (query, text) pairs returns a NumPy array by default, and `argsort()[::-1][:10]` is the correct descending top 10. The only issue is the score-scale NIT above.

#### Fix status (applied 2026-09-16)

- [NIT] `CrossEncoder.predict` applies a sigmoid, so output is not raw 9.1/8.4 — fixed: added a short paragraph to Under the hood (default sigmoid into 0–1, same order, still not comparable across questions), and "What people get wrong" now says the scores are logits underneath "even when a library squashes them into 0–1".
- Code: no code changed (prose note only)
- Cross-chapter follow-ups: none

## Chapter 14 — Asymmetric Search and Instruction-Tuned Embeddings  (reviewer: chapters 14–20)

- **[SHOULD]** "Model card" is used five times, including as the bolded advice "**Read the model card.**", but is never defined here or in any earlier chapter (Ch 11 also uses it bare). A novice does not know it is the documentation page a model's authors publish alongside the weights. Quote: "Model cards are sometimes unclear, and code gets copied between projects." Fix: at first use write "the **model card** (the documentation page a model's authors publish with it, listing the prefixes, pooling and intended use)".
- **[NIT]** The claim that a stray prefix "injects two meaningless tokens" is tokenizer-dependent and unverified (`"query: "` may be 2 or 3 tokens depending on the tokenizer). Quote: "just injects two meaningless tokens and slightly degrades the output". Fix: "a couple of meaningless tokens".
- **[NIT]** Frontmatter `readingTime: "11 min"` for 2,971 words (~13 min at 230 wpm). Fix: "13 min".
- Prefix conventions checked: E5 `query:`/`passage:` (and `query:` on both sides for symmetric tasks), BGE "Represent this sentence for searching relevant passages: " (query side only), Nomic `search_query:`/`search_document:`/`clustering:`/`classification:`, e5-mistral `Instruct: {task}\nQuery: {q}` all match the published model cards. "seven missing characters" for `"query: "` is correct. "nine words long" is correct. Cross-references (Ch 10 attention, Ch 12 training, Ch 13 reranker/two-tower and the page 1,140 rescue, Ch 19 nDCG, Ch 42 CLIP, Ch 45 ColPali, Ch 63 migration) all point at the right chapters; What's next links to 15-matryoshka.md.
- Style: no notable violations (no semicolons or em-dashes in teaching text; "you" only in advice).
- FIXES.md items for this chapter: all applied (nDCG@10 is glossed as "a ranking-quality score, Chapter 19").
- Code: 6 blocks. `prefix_check` and `Embedder` read clean (need a sentence-transformers model, cannot run); argmax-vs-arange hit test is correct. The E5/BGE/Nomic snippets are illustrative fragments and read correctly.

#### Fix status (applied 2026-09-16)

- [SHOULD] "Model card" never defined — fixed differently: Ch 11 now gives the bold definition at its first use there (I own Ch 11), and Ch 14's first use points back with a short gloss: "Model cards (the documentation pages published with each model, listing its prefixes, pooling and intended use, Chapter 11)".
- [NIT] "injects two meaningless tokens" depends on the tokenizer — fixed: "a couple of meaningless tokens".
- [NIT] readingTime "11 min" low — skipped: readingTime NITs are superseded by the brief.
- Code: no code changed
- Cross-chapter follow-ups: 03-geometry-of-meaning.md line 222 uses "model card" bare, before Ch 11 defines it (already reported in 11.md).

## Chapter 15 — Matryoshka Embeddings  (reviewer: chapters 14–20)

- **[NIT]** "Inference" is used bare and is not defined here or earlier (Ch 11 also uses it bare). Quote: "It costs a little extra compute during training. It costs nothing at inference." Fix: "It costs nothing at inference (when we run the trained model to embed a text)."
- **[NIT]** Frontmatter `readingTime: "11 min"` for 2,931 words (~13 min at 230 wpm). Fix: "13 min".
- Arithmetic recomputed and correct: 40,000 × 768 × 4 = 122.9 MB, 64-d = 10.2 MB (12×), 100M × 768 × 4 = 307 GB, 64-d = 25.6 GB, binary 768-d = 96 B → 9.6 GB, int8 256-d = 256 B → 25.6 GB; `total / len(dims)` matches "every w_m equal to 1/5". The Chapter 13 hand-off (page 212 rank 1, page 1,140 rank 23, top-100 reranked to ranks 1 and 2) matches Ch 13. The "about 25 GB" RAM figure matches Ch 60's table ("Binary + rescore from SSD | ~25 GB RAM"). Ch 6 does define PCA and the coarse-then-rescore pattern; Ch 12 defines gradient and its `infonce` does re-normalize (`F.normalize`), so the comment "(it re-normalizes each prefix)" is correct. MRL model list (text-embedding-3, Nomic v1.5, mxbai-embed-large, Jina v3) agrees with the published model cards. What's next links to 16-choosing-a-model.md and glosses MTEB.
- Style: no notable violations.
- FIXES.md items for this chapter: all applied (Stage 1 now "768 dims → 96 bytes/vector"; "dot-product scores (which we were treating as cosines)"; PCA glossed with a Chapter 6 pointer; gradient pointer to Ch 12).
- Code: 4 blocks. `eval_at_dims` and the re-normalization snippet run clean on synthetic data; `mrl_loss` reads clean given Ch 12's `infonce`.

#### Fix status (applied 2026-09-16)

- [NIT] "Inference" used bare — fixed: "It costs nothing at inference (when we run the trained model to embed a text)." Ch 11's earlier bare use was reworded ("embedding one text at a time"), so this gloss is now the first use in Chapters 1–15.
- [NIT] readingTime "11 min" low — skipped: readingTime NITs are superseded by the brief.
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 16 — Choosing and Benchmarking an Embedding Model  (reviewer: chapters 16–20)

No MUST or SHOULD issues found. Verified: the Acme recall@10 table's "All 50" column is the correct query-weighted average of the four groups (20/10/10/10) in every row (0.752→0.75, 0.73, 0.67, 0.61, 0.87); 1536-d = 4× 384-d; 7B / 335M ≈ 20.9 ("about 20 times", used twice); 512 tokens ≈ 384 words (fits "350–400"); a 2-point gap at n = 50 is well inside one standard error (√(0.75·0.25/50) ≈ 6 points), so "well within the noise" holds. BGE-M3 dense+sparse+multi-vector with 8k context, OpenAI text-embedding-3 MRL-capable, and the e5 ("query: "/"passage: ") and bge-en-v1.5 query-instruction prefixes are all correct. The SSO-4012 / SSO-4021 / page 17,450 / 17,452 scenario matches Ch 2, 7 and 8. Cross-references (Ch 6 intrinsic dimension, 8, 10, 14, 15, 19, 40, 55, 63, Parts V and VI) all match OUTLINE. We-will-cover bullets match the nine H2 headings exactly. The chapter cites no real leaderboard scores (all table values are labelled toy numbers), so there is nothing unverifiable there.

- **[NIT]** The "All 50" column does not, taken alone, say "pick Model A": the same column shows the fused row at 0.87, twelve points clear. Quote: "Taken alone, that column would still say \"pick Model A\"." Fix: "Taken alone, and looking only at the three models, that column would still say \"pick Model A\"."
- **[NIT]** Caution 1's list of what MTEB averages leaves out pair classification (and bitext mining in the multilingual board), so a reader checking the leaderboard will see columns the book did not mention. Quote: "MTEB averages classification,
clustering, reranking, STS, summarisation and retrieval." Fix: add "pair classification" (and optionally "bitext mining"), or say "averages task types such as …".
- **[NIT]** Original MTEB (Muennighoff et al., 2022) already spanned 112 languages across 58 datasets. Later versions (MMTEB) widened this a lot, but "in later versions" suggests the first release was English-only. Quote: "many tasks, many datasets, and in later versions many languages." Fix: "many tasks, many datasets and many languages (far more in the newer MMTEB)".
- **[NIT]** Caution 4 says to treat the "top 10–15" as equivalent, but Key takeaways says "treat the top 10 as a tie". Quote: "treat the top 10 as a tie". Fix: "treat the top 10–15 as a tie".
- **[NIT]** `BM25Okapi` in rank_bm25 defaults to k1 = 1.5, not the k1 = 1.2 that Chapter 8 teaches and that the code comment points to. Quote: "bm25 = BM25Okapi([p.lower().split() for p in pages])". Fix: `BM25Okapi([...], k1=1.2, b=0.75)`, or note that the library default differs.
- Style: clean. No semicolons or em-dashes in the prose, the wine-critic analogy has explicit mapping lines, the steps are laid out as Phase/Step, and the chapter shows a wrong answer before the right one.
- FIXES.md items for this chapter: all applied (takeaways say "dimensions", shortlist says "Jina v3/v4").
- Code: 1 block, run end-to-end with small stub `rank_bm25` / `sentence_transformers` modules on a toy 15-page corpus. `dense_top`, `bm25_top`, `rrf`, `recall_at_k` and `report` all behave as the prose describes, and the prefixes unpack the right way round (qp, dp).

#### Fix status (applied 2026-09-16)

- [NIT] "All 50" column ignores the fused 0.87 row — fixed
- [NIT] Caution 1 MTEB task list incomplete — fixed differently: "MTEB averages task types such as …" so the list is explicitly partial, without adding more jargon.
- [NIT] "in later versions many languages" wrong for original MTEB — fixed differently: "many tasks, many datasets and many languages, with more added in newer versions" (no new MMTEB acronym for novices).
- [NIT] Key takeaways "top 10" vs Caution 4 "top 10–15" — fixed
- [NIT] BM25Okapi default k1 = 1.5, not Chapter 8's 1.2 — fixed: passed k1=1.2, b=0.75 and commented the library default.
- Code: bm25_top line changed. Ran the whole block with real rank_bm25 (installed in venv) and a stub SentenceTransformer on a 15-page toy corpus. All five rows report correctly.
- Cross-chapter follow-ups: none

## Chapter 17 — Brute Force and the Flat Index  (reviewer: chapters 16–20)

No MUST or SHOULD issues found. Verified with Python:
- **Sizes.** 40,000 × 768 × 4 = 122.88 MB ("122.9 MB"). 1M × 768 × 4 = 3.07 GB, with 768 million multiply-adds. Every size in the scaling table is right: 30.7 MB, 307 MB, 3.07 GB, 30.7 GB, 307 GB. Binary 100M × 96 B = 9.6 GB. 100M / 5,000 tenants = 20,000 chunks (matches Ch 57/58). 768/256 = 3×.
- **Latency arithmetic.** It hangs together: 3.07 GB at 20–50 GB/s is 61–154 ms ("60–150 ms"). 122.9 MB at the same rate is 2.5–6.1 ms ("2.5–6 ms"). 3.07 GB at 1–3 TB/s is 1.0–3.1 ms. The multi-core and GPU columns are consistent with those rates. The bandwidth-bound framing is also self-consistent: 768M multiply-adds in 60–150 ms is only 5–13 billion per second, well below what a 16-wide AVX-512 core can compute, so the belt, not the cashier, sets the pace. The 20–50 GB/s per-core and 1–3 TB/s GPU bandwidth figures are unverified but plausible.
- **Big-O.** Sort O(n log n) vs argpartition O(n) is correct.
- **Batching.** "Tens of times higher, can reach 50–100×" is plausible. On this reviewer's machine (Apple Accelerate BLAS, 500k × 768, batch of 200) batched throughput per query was 18.5× single-threaded and 19.3× multi-threaded. 50–100× needs compute far above the bandwidth-limited single-query rate, which is realistic on a GPU or large batches.
- **Deduplication.** Pairwise dedup of 10M is 3.8e16 multiply-adds for the upper triangle, about 20 minutes at ~3e13 per second, so "tens of minutes" on one GPU is plausible.
- **Scenario.** The "who lost page 1,140" table (212 / 88 / 2,301 / 45 / 9,012, then 1,140 at rank 23 → rank 2 after reranking top 100) matches Ch 13 exactly.
- **Cross-references.** Ch 1, 3, 4 (SIMD), 5, 13, 15, 18, 26, 28–31, 33, 58, 62, and Part VIII's 5,000 tenants / 100M chunks all check out. The We-will-cover bullets match the nine H2s.

- **[NIT]** The HNSW callout says brute force over 40,000 items takes "about 2 ms", while the one-paragraph version, the numbers section, the example, the Note and Key takeaways all say about 1 ms for Acme's 40,000 pages. Quote: "It buys a speedup from about 2 ms to 0.4 ms". Fix: "from about 1 ms to 0.4 ms" (or name single-core, "2.5 ms").
- **[NIT]** The 50–100× batching ceiling is only reachable where compute far outruns bandwidth. At the chapter's own single-core numbers, one CPU core gives roughly 10–20×. Quote: "Throughput per query is often tens of times higher, and can reach 50–100×." Fix: "…often tens of times higher, and on a GPU can reach 50–100×."
- **[NIT]** "Updates are instantaneous" points to Chapter 31 for compaction, but Ch 31 itself defers compaction to Chapter 59. Quote: "no compaction (the periodic clean-up some indexes need, Chapter 31)". Fix: "Chapters 31 and 59".
- Style: clean. No semicolons or em-dashes in the prose, the supermarket analogy has explicit mapping lines, the steps are laid out as Phase/Step, and the chapter shows a wrong diagnosis before the right one. "you" appears only in What people get wrong and Ninja notes.
- FIXES.md items for this chapter: all applied. The Card Catalog sentence is present (adapted to "40,000 cards", which fits Acme and the later "a million cards instead of 40,000"). The HNSW gloss "(the graph index of Chapters 28–31)", the O(n) plain restatement, "a vector (SIMD) unit" and "reuses each block of memory already pulled into the CPU's cache" are all in.
- Code: 1 block, run on 40,000 random 768-d vectors. It returns the same top-5 as a full argsort, `V` stays C-contiguous float32 (122.88 MB), k > n works (returns all 3 of 3), and search takes about 0.56 ms/query here, which fits "about a millisecond".

#### Fix status (applied 2026-09-16)

- [NIT] HNSW callout "about 2 ms" vs "about 1 ms" elsewhere — fixed
- [NIT] 50–100× batching ceiling needs "on a GPU" — fixed by the lead after verification: "…often tens of times higher, and on a GPU can reach 50–100×" (the reviewer measured about 19× on a CPU).
- [NIT] Compaction pointer should name Chapter 59 too — fixed ("Chapters 31 and 59")
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 18 — The Recall–Latency–Memory Triangle  (reviewer: chapters 16–20)

Verified with Python:
- **Recall.** Recall@10 of 0.95 = 10 × 0.05 = 0.5 misses per query ("one every two queries").
- **Speed-up.** 50/4 = 12.5 and 500/4 = 125 ("12–125×"), which supports the "10–100×" rule of thumb in the body and Key takeaways.
- **Memory.** 3,072 B / 3.07 GB / 307 GB are right. HNSW at 1.05–1.1× gives 322.6–337.9 GB ("320–340 GB", and Ch 60 has ~323 GB). IVF-PQ at 96 B is 0.031× and 9.6 GB ("about 10 GB"). DiskANN ~48 B in RAM is 0.016× ("~0.02×", matching Ch 32's 32–64 GB per billion and Ch 60).
- **Percentiles.** The p50/p99 nearest-rank description (500th and 990th of 1,000 timings) is correct.
- **Page 1,140 as a true neighbour.** The SSO chunk of page 1,140 counts as a true top-10 neighbour, and this is consistent. Ch 17 puts the whole-page vector at rank 23. Ch 11 is exactly where page 1,140 is cut into ~100-word chunks and "the Librarian fetches page 212 and the SSO chunk of page 1,140", so the "(Chapter 11)" reference is right. Ch 31's later use of "page 1,140" matches the Note's convention. "Might be the 9th-nearest" stays inside the top 10.
- **Other claims.** The 5–10% HNSW overhead matches Ch 29 (≈5% at M = 16) and Ch 20's "~5%". Cross-encoder "20–100 ms on a GPU" for 100 candidates matches Ch 41.
- **Cross-references.** Ch 1, 3, 6, 11, 17, 19, 23–26, 28–32, 41 and 59 all check out, and the We-will-cover bullets match the eight H2s.
- **Unverified.** The claim that SIFT1M/GloVe are "easy" and that real text embeddings give lower recall at the same `ef` could not be checked. It agrees with Ch 6 and Ch 30.

- **[SHOULD]** The binary index's memory is given two different ways in the same chapter. The "How each index chooses" table says 0.08×, but the Ninja-notes cascade says 0.03× and ~1 ms. A novice who has just read "one bit per number" (= 32×, 0.03×, as Ch 17 and Ch 26 say) cannot see where 0.08× comes from. Ch 60 explains that 0.08× is the 96 B code plus ~150 B of graph links (~250 B). A 0.03× flat binary scan over 100M vectors is 9.6 GB, roughly 200–500 ms per core by Ch 17's bandwidth arithmetic, not ~1 ms. Quote: "binary index (0.03× memory, ~1 ms)  →  top 1,000" and "| Binary + rescore | Very good | Excellent | ~0.08× in RAM + SSD | A little recall |". Fix: change the cascade line to "binary graph index (~0.08× memory, ~1 ms)". In the sentence under the table, add "the 0.08× is the 1-bit codes plus a graph over them (Chapter 60)".
- **[NIT]** The cascade's "float32 rescore of 1,000 vectors (~1 ms)" assumes the full vectors are in RAM, but the table row says the full vectors live on SSD. Ch 26 says fetching those 3 MB from SSD takes "milliseconds". Quote: "float32 rescore of 1,000 vectors (~1 ms)". Fix: "(~1 ms from RAM, a few ms from SSD)".
- **[NIT]** The claim that a reranker would push the missed marginal neighbours *down* is the opposite of the book's own running example, where the reranker lifts page 1,140 from rank 23 to rank 2 (Ch 13, 17). A reranker reorders by relevance, not by vector rank. The Note that follows half-repairs this. Quote: "A reranker (Chapter 41) would probably push them down anyway". Fix: "Often they are not the pages the answer depends on anyway".
- Style: clean. The only em-dash in prose is inside a table cell, and there are no semicolons. The Librarian analogy has explicit mapping sentences, the chapter shows a wrong answer and then the right one, and the steps come before the code.
- FIXES.md items for this chapter: all applied (p50/p99 glossed, "~5–10% HNSW graph overhead at 768 dimensions (Chapter 29 does the arithmetic)", centroids glossed, brute-force row "~50–500× (brute force; scale-dependent)").
- Code: 1 block, run against a mock ANN index that drops one true neighbour on half of the queries. It prints recall@10 = 0.951 plus p50/p99/memory as described. The placeholders (`set_ef`, `search`, `memory_bytes`) are declared as such.

#### Fix status (applied 2026-09-16)

- [SHOULD] Binary memory given as 0.08× and 0.03× — fixed: cascade line now "binary codes + graph (~0.08× memory, a few ms)", matching Ch 60's ~25 GB row and Ch 26's "a few ms" Stage 1. Sentence under the table explains codes alone are 0.03×, the 0.08× adds the graph, full vectors wait on SSD (Chapter 60).
- [NIT] Cascade rescore "~1 ms" assumes RAM — fixed ("~1 ms from RAM, a few ms from SSD", same wording as Ch 26)
- [NIT] Reranker would push missed neighbours down contradicts Ch 13/17 — fixed ("Often they are not the pages the answer depends on")
- Code: no code changed
- Cross-chapter follow-ups: 26-scalar-binary-quantization.md (optional): "95–99% of full-precision quality at roughly 3% of the memory" counts codes only; the same chapter says the cascade needs ~25 GB (≈8%) with the graph. Consider "roughly 3% of the vector bytes" or "under a tenth of the RAM".

## Chapter 19 — Measuring Retrieval Quality  (reviewer: chapters 16–20)

Every worked metric was recomputed with Python and is correct:
- **Recall and precision.** Recall@3 = 0.33, recall@5 = 0.67, recall@10 = 1.00, precision@5 = 0.40.
- **Reciprocal rank and MRR.** RR = 0.5, and the MRR example (0.5 + 1.0 + 0.25)/3 = 0.583 → 0.58.
- **Logs.** log₂3 = 1.585, log₂6 = 2.585, log₂10 = 3.322.
- **nDCG.** DCG@10 = 0.631 + 0.387 + 0.301 = 1.319, IDCG@10 = 2.131, nDCG@10 = 0.6189 → **0.62**, and "~0.79" is gone everywhere. The discount line ("rank 7 counts a third as much") is right.
- **Deleting page 1,140.** RR stays 0.5 while recall@5 drops to 0.33, which demonstrates "MRR does not move".
- **Standard error.** At n = 20 it is √(0.25/20) = 0.112, so **±11 points with a 95% interval of ±22** (28%–72%). n = 50 gives ±7.1 and n = 200 gives ±3.5. This matches Ch 54's table.
- **Index recall.** The example (8 of 10 → 0.80) is right.
- **Cross-references.** Ch 8, 10, 12 (false negatives among hard negatives), 16 ("points of nDCG" out of 100), 18, 53, 54 and Part VII all check out. Page 2,306 "Compare plans" matches Ch 56. We-will-cover bullets match the nine H2s. Key takeaways match the body.

- **[SHOULD]** The worked-example commentary contradicts the chapter's own demonstration. The recall@k section showed that a Scholar reading the top 5 already answers correctly, because pages 212 and 1,140 are both there. Yet the next paragraph says the "real problem" is that page 1,140 must climb, which only matters for a top-3 pipeline. A novice cannot tell which k the "team" is using. Quote: "That is worth nothing to a Scholar that reads the top 5. A team chasing recall@10 would declare victory, since it is already perfect. A team chasing recall@3 would see the real problem: page 1,140 must climb." Fix: "A team that hands the Scholar only the top 3 tracks recall@3 (0.33) and sees the real problem: page 1,140 must climb into the top 3. A team that hands over the top 5 tracks recall@5 and learns the pipeline already works."
- **[NIT]** The graded-relevance scale is introduced with four levels ("perfect / good / marginal / irrelevant", i.e. grades 3/2/1/0), but the Note calls grade 2 "perfect". Quote: "A grade-2 (\"perfect\") page gains 2² − 1 = 3 points". Fix: "A grade-2 (\"good\") page gains 3 points, and a grade-3 (\"perfect\") page gains 7".
- **[NIT]** Overlapping 95% intervals do not by themselves show "no result". Two means can differ significantly while their intervals overlap, and the very next paragraph says a paired test on the same questions can turn such a comparison into a clear answer. Quote: "The intervals visibly overlap, so it is visibly not a result." Fix: "The intervals overlap heavily, so on these numbers alone we cannot claim a win. A paired test (below) is the proper check."
- Style: clean. No semicolons or em-dashes in the prose, the steps are laid out as Step 1–5 for nDCG, the chapter shows a wrong answer (top 3) and then the right one (top 5), and there is a bold-equation definition for every metric. "you" appears only in advice and Key takeaways.
- FIXES.md items for this chapter: all applied (nDCG ~0.62 with the DCG/IDCG shown, the "position 1 / 3 / 7" discount line, and ±11 / ±22 at n = 20 per the deliberate deviation).
- Code: 2 blocks, both run. The metrics block prints 0.33 / 0.67 / 0.4 / 0.5 / 0.62, exactly as commented. `index_recall` returns 0.8 on an 8-of-10 toy, and `bootstrap_ci` runs (it relies on `np` from the first block, which is fine in context).

#### Fix status (applied 2026-09-16)

- [SHOULD] Worked-example commentary contradicts top-5 demonstration — fixed differently: each team is tied to the k it hands the Scholar (top 3 tracking recall@10 declares victory, tracking recall@3 sees page 1,140 must climb into the top 3, top 5 sees both needed pages are already there).
- [NIT] Grade-2 called "perfect" on a four-level scale — fixed (grade 2 "good" = 3 points, grade 3 "perfect" = 7)
- [NIT] Overlapping intervals do not by themselves mean "no result" — fixed
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 20 — The ANN Family Tree  (reviewer: chapters 16–20)

Verified:
- **Choosing table.** The "Your situation → Use → Chapter" table matches FIXES and OUTLINE row for row: 17 / 17, 28 / 28–31 / 25, 26 / 25, 32 / 32, 58 / 33 / 17 / 37, 38. Each cited chapter covers what the row says (25 IVF-PQ, 26 int8, 32 DiskANN, 33 filtering, 37 PLAID, 38 MUVERA, 58 sharding).
- **Compression table.** 3,072 / 1,536 / 768 / 96 / 96 bytes are correct.
- **Graph overhead.** "~5–10% at 768-d, 1.5–2× for low-dimensional vectors with many links" matches Ch 29's Note (1.05× at M = 16, and 1.55× for 128-d at M = 32) and Ch 18.
- **efSearch.** efSearch = 512 → 0.995 matches Ch 30's table.
- **Back-references.** Ch 4 did introduce "quantization", Ch 6 introduces "manifold" and the curse of dimensionality, Ch 3 "clumpy", and Ch 18 glossed centroids.
- **Index names.** IVF-PQ, IVF-Flat, HNSW-SQ, DiskANN and ScaNN (partitioning + anisotropic quantization) are decomposed correctly.
- **Structure.** Every analogy has its "Here, …" mapping sentence, and the We-will-cover bullets match the seven H2s.

- **[SHOULD]** The Ninja notes contradict the chapter's central claim. The summary says "Every approximate nearest neighbour algorithm ever built is one of four ideas", the body says "There are exactly four ways" and "That is the whole taxonomy", Key takeaways say "Everything else is a combination", and OUTLINE's line for this chapter is "There is no fifth idea". Ninja notes then open by announcing a fifth idea. The example given (a trained model that predicts which partition a query belongs in) is really the cluster/partition family with a learned router, so the taxonomy can stand. Quote: "There is a fifth idea that does not fit the taxonomy, and it is becoming important: **learned indexes**." Fix: "**Learned indexes** can look like a fifth idea, but they are not. They keep the cluster or partition family and replace the nearest-centroid rule with a small trained model that predicts which partitions to probe."
- **[NIT]** Hashing is not the oldest of the four ideas as ANN methods. k-d trees (Bentley, 1975) predate LSH (Indyk & Motwani, 1998) by more than twenty years. Quote: "starts with the oldest of the four ideas, hashing". Fix: "starts with the first of the four ideas, hashing".
- **[NIT]** The LSH pseudocode uses one `hash_fn` for every table. Multiple tables only help if each has its own independent hash, which is exactly what Ch 21 says ("each with its own random sheets"). With a single shared function, every table returns the same bucket. Quote: "return scan([d for t in tables for d in t[hash_fn(q)]], q)". Fix: `return scan([d for t in tables for d in t.buckets[t.hash(q)]], q)` or iterate `zip(hash_fns, tables)`.
- **[NIT]** NSG is listed as if Chapters 28–32 cover it, but NSG appears nowhere else in the book. Quote: "→ **HNSW, DiskANN, NSG** (Chapters 28–32)." Fix: "→ **HNSW, DiskANN** (Chapters 28–32), and cousins such as NSG".
- **[NIT]** Acme is placed in the "< 100k vectors" row. But since Ch 18 the book searches ~100-word chunks, and Ch 58 counts Acme's 40,000 pages as about 320,000 chunks, which is the second row. Brute force is still right per Ch 17's "100k–1M: brute force is still viable". Quote: "Its Library has 40,000 pages, so it sits in the first row." Fix: "Its Library has 40,000 pages, a few hundred thousand chunks, so it sits in the first two rows."
- **[NIT]** The side-by-side table puts DiskANN in the Graph row whose "Weak at" column starts with "Memory", yet low RAM is DiskANN's whole purpose (Ch 32; ~0.02× in Ch 18). Quote: "| Graph | HNSW, DiskANN, NSG | Recall at low latency | Memory, deletes, filters |". Fix: "Memory (in-RAM graphs such as HNSW), deletes, filters".
- Style: clean. No semicolons or em-dashes in the prose (the only `;` is inside pseudocode), the chapter shows a wrong answer (partition) and then the right one (graph), there is a bold-equation definition for each family, and "you/your" appears only in advice and the table header.
- FIXES.md items for this chapter: all applied. The decision table is renumbered, "provably moves closer" is replaced by the dead-end/`efSearch` wording, and SOAR is dropped. The graph overhead now reads "~5–10%" rather than "~5%", which agrees with Ch 18 and Ch 29 (fine).
- Code: 1 pseudocode block, run with toy helper implementations (5,000 clustered 32-d vectors). `ivf_search` (nprobe = 5) gave recall@10 = 0.98, and `graph_search` (ef = 64, 16-NN graph plus a few long links) gave 0.88, so both behave as described. The only defect is the shared `hash_fn` above. Minor: `topk` is called on a score array in `ivf_search` but on a node list in `graph_search`, and `improving` / `best_unexpanded` / `keep_best` implicitly need `q`. That is acceptable for pseudocode.

#### Fix status (applied 2026-09-16)

- [SHOULD] Ninja notes announce a fifth idea (learned indexes) — fixed per D2: learned indexes "can look like a fifth idea, but they are not", they keep the partition or cluster family with a trained router, plus an "In simple words" line.
- [NIT] Hashing is not the oldest of the four ideas — fixed ("the first of the four ideas")
- [NIT] LSH pseudocode shares one hash_fn across tables — fixed: `t.buckets[t.hash(q)]`, comment says each table has its own hash function
- [NIT] NSG listed as covered by Chapters 28–32 — fixed ("HNSW, DiskANN (Chapters 28–32), and cousins such as NSG")
- [NIT] Acme placed in the < 100k row despite ~320,000 chunks — fixed: "a few hundred thousand chunks, so it sits in the first two rows"; the follow-on sentence now points to Part VIII's ~100 million chunks instead of "grows into millions" (D10 framing).
- [NIT] Graph row "Weak at: Memory" misfits DiskANN — fixed ("Memory (in-RAM graphs such as HNSW), deletes, filters")
- Code: lsh_search line changed. Ran the pseudocode block with a toy harness (5,000 unit 32-d vectors, 10 tables each with its own 8-bit random-hyperplane hash): runs, returns results, recall@10 ≈ 0.37 (low, as the chapter says for LSH on dense vectors), and the 10 tables give 10 distinct buckets.
- Cross-chapter follow-ups: none beyond D2 (Ch 65 owner cites Ch 20's framing, OUTLINE keeps "There is no fifth idea.")

## Chapter 21 — LSH: Hashing Things That Are Alike  (reviewer: chapters 21–27)

- No MUST or SHOULD issues found. All probabilities recomputed and correct: 0.944^16 = 0.401, 1 − 0.6^8 = 0.983, the k=16 table (10°/30°/60°/90° → 0.40/0.05/0.002/<0.001 for 1 table and 0.98/0.36/0.01/<0.001 for 8 tables), MinHash standard error √(0.25/128) = 0.044 ("at most about 4 points"), 90/110 = 0.82, three 5-word shingles in a 7-word sentence. Cross-references (Ch 3 anisotropy cone and top-k, Ch 4 angle, Ch 6 JL, Ch 18, Ch 19 standard error, Ch 26 Hamming/binary, Ch 38 MUVERA) all point at the right chapters. readingTime 12 min matches 2,930 words.
- **[NIT]** The retrieval formula silently assumes the 16 bits are independent. Quote: "$$ P_{\text{found}} = 1 - (1 - p^k)^L $$". On real data (and even on random data in low d) the bits are correlated, so the empirical rate is a little higher than the formula; a half-sentence "assuming each bit is an independent coin" would make the derivation honest. Optional.
- **[NIT]** "The side test in Step 2 is one line of maths" follows two numbered lists that both have a Step 2. It reads fine in context but "Phase 1, Step 2" would remove the ambiguity.
- Style: no notable violations (no semicolons in prose, one em-dash in the whole file, "you" only in advice sections).
- FIXES.md items for this chapter: all applied ("essentially binary quantization with random axes. Chapter 26 uses the model's own axes instead", bold shingles definition present).
- Code: 1 block, runs clean. Tested on 5,000 random unit vectors in d=64 with queries placed 10° from a stored vector: the stored vector was in the candidate set 99.7% of the time (formula predicts 98%), confirming the AND/OR behaviour the prose describes.

#### Fix status (applied 2026-09-16)

- [NIT] P_found formula silently assumes independent bits — skipped: for a fixed pair of vectors, the sheets are drawn independently, so the bits are exactly independent and the formula is exact. A 200,000-trial simulation at 10° gave 0.401 (one table) and 0.984 (eight tables), matching 0.40 and 0.98. The reviewer's 99.7% came from its own test setup, not from bit correlation.
- [NIT] "Step 2" ambiguous after two step lists — fixed ("Phase 1, Step 2")
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 22 — Trees: KD-Trees and Annoy  (reviewer: chapters 21–27)

- No MUST issues found. Arithmetic checked: 2^20 = 1,048,576; 20 of 100 dimensions = "a fifth"; 2^768 ≈ 10^231, comfortably above the ~10^80 atoms. Cross-references verified: Ch 3 (top-k), Ch 6 (curse of dimensionality), Ch 17 (Big-O is defined there, so $O(\log n)$ here is fair), Ch 18 (introduces "the SSO chunk of page 1,140" as a true nearest neighbour, which is what the "(Chapter 18)" aside points at), Ch 20, Ch 21 (OR amplification), Ch 23 (IVF), Ch 28 (HNSW hierarchy). Annoy's mechanics (two sampled points, bisecting hyperplane, priority-queue search bounded by `search_k`, `n_trees`, immutable mmap file) are described correctly.
- **[NIT]** Heading vs code commentary disagree on what kind of cut Annoy uses. Quote: "**Change 1: Cut on random directions, not axes.**" versus, in Under the hood, "not from a coordinate axis or a purely random direction". Both are true (the direction is random *and* data-derived) but a novice reads them as contradicting. Suggest "Change 1: Cut along the data, not the axes."
- **[NIT]** Table row label does not match its KD-tree cell. Quote: "| Recall at equal speed on embeddings | Slower than brute force | Good | Better |". A speed statement sits in a recall row. Either rename the row ("Speed and recall on embeddings") or write "Exact, but slower than brute force".
- **[NIT]** "The **hierarchical k-means trees** in FAISS (Meta's vector-search library) and in ScaNN's partitioning" — unverified. ScaNN's partitioner is a (possibly hierarchical) k-means tree; FAISS exposes multi-level routing mainly via an IVF or HNSW coarse quantizer over the centroids rather than a product called a "hierarchical k-means tree". Consider "multi-level k-means partitioning in libraries such as ScaNN".
- **[NIT]** Frontmatter says `readingTime: "10 min"`; 2,616 words is closer to 11–12 min at 230 wpm.
- Style: no notable violations (no prose semicolons, one em-dash in the file).
- FIXES.md items for this chapter: all applied ("k-means (a clustering method, defined next chapter)" present in Ninja notes).
- Code: 1 block, runs clean. `RPTree` builds and `forest_search` returns exact-rescored candidates on 20,000 random unit vectors in d=64. Recall@10 was 0.02 with one tree, 0.09 with 10, 0.44 with 50, which is exactly the "one tree loses it, a forest finds it" story; the low absolute numbers are expected because random Gaussian data is the worst case and the block deliberately omits Change 2 (backtracking), which the prose states.

#### Fix status (applied 2026-09-16)

- [NIT] "Change 1" heading vs code note on cut direction — fixed ("Change 1: Cut along the data, not the axes.")
- [NIT] Speed statement in a recall table row — fixed: row renamed "Speed and recall on embeddings", KD-tree cell "Exact, but slower than brute force"
- [NIT] Unverified "hierarchical k-means trees in FAISS" — fixed differently: "Multi-level k-means partitioning, as in ScaNN" plus "FAISS gets a similar effect by putting a small index over the cluster centres".
- [NIT] readingTime "10 min" low — skipped: readingTime NITs are superseded by the brief (prose-only count is within one minute).
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 23 — IVF: Clustering the Space  (reviewer: chapters 23–27)

- No MUST issues found. Every hand calculation was recomputed and is correct: all 15 squared distances in the three k-means rounds, the centre moves (6, 3) → (7, 2) and (2, 7), the query (3, 5) distances 1/5/13/34/52 and centroid distances 5/25, 10M ÷ 1,000 = 10,000 per list, nprobe 10 → 100,000 scanned (100×), the table's scanned counts, √10M ≈ 3,162 ("about 3,000"), 1M ÷ 65,536 ≈ 15 per list, 96 + 8 B vs 3,072 B ≈ 0.03×, 3,072 ÷ 96 = 32×. The "hundred wings" analogy has its full mapping (books = vectors, wing = cluster/inverted list, sign = centroid, number of wings = nlist, wings entered = nprobe). Cross-references checked: Ch 5, 7 (inverted index), 3 (top-k), 17 (flat, memory bandwidth), 18 (centroid, p99, steep curve), 20 (IVF-Flat), 24, 25, 29 (~1.05× graph overhead), 31, 33, 58.
- **[SHOULD]** The √n rule is misapplied for a hundred million vectors. Quote: "That gives 1,000 for a million vectors, and roughly 4,000–16,000 for a hundred million." √(10^8) = 10,000, and the range 4,000–16,000 does not follow from √n (it looks like a garbled version of FAISS's own guideline of 4√n–16√n, which would give 40,000–160,000). Fix: "That gives 1,000 for a million vectors and 10,000 for a hundred million. (FAISS's documentation suggests a larger 4√n–16√n, so treat √n as a starting point.)"
- **[NIT]** "completely predictable" contradicts the next two sentences. Quote: "`nprobe` is the recall–latency dial, and its behaviour is completely predictable." The table is then called illustrative and the reader is told to measure. Fix: "its behaviour is monotonic: more wings, more recall, more latency."
- **[NIT]** "A few hundred thousand vectors is plenty" holds for nlist ≈ 1,000 but not for the nlist = 65,536 used in Ch 25, where FAISS warns below ~39 training points per centroid (≈ 2.5M). Quote: "A few hundred thousand vectors is plenty." Fix: add "(roughly 30–250 per centroid, so more for large `nlist`)".
- Style: no notable violations (no semicolons or em-dashes in prose, no sentence over 32 words, "we" for shared work).
- FIXES.md items for this chapter: all applied (bold k-means Step 1 gloss with "In simple words"; memory claim now "about 1.05–1.1× at 768 dimensions (Chapter 29)" and the PQ-codes point; `train()` creates the lists and `add()` appends).
- Code: 1 block, runs clean. `kmeans(pages, 2, init=pages[[2, 3]])` prints [[7, 2], [2, 7]] as the comment claims. On 50,000 random unit vectors (d = 64, nlist = 224 ≈ √N, 200 queries), recall@10 vs brute force rose monotonically with nprobe (0.10 / 0.25 / 0.51 / 0.84 / 1.00 at nprobe 1 / 4 / 16 / 64 / 224) and was exact at nprobe = nlist. Two `add()` batches kept all 50,000 IDs, confirming the FIXES code fix.

#### Fix status (applied 2026-09-16)

- [SHOULD] √n rule misapplied for 100M vectors — fixed: now "1,000 for a million vectors and 10,000 for a hundred million", plus "Treat √n as a starting point. Chapter 25 goes up to 4√n, and FAISS's own guidelines suggest 4√n–16√n." (keeps Ch 25/66's √n–4√n range consistent). The bad range appeared only in Ch 23.
- [NIT] "completely predictable" contradicts illustrative table — fixed differently: "it always moves both the same way: more wings, more recall, more latency" (avoids the unglossed word "monotonic").
- [NIT] "few hundred thousand is plenty" wrong for large nlist — fixed: "plenty for `nlist = 1000`. The usual guide is roughly 30–250 sample vectors per centroid, so a large `nlist` needs a larger sample."
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 24 — Product Quantization  (reviewer: chapters 23–27)

- No MUST or SHOULD issues found. Recomputed with Python: 768 × 4 = 3,072 B → 96 B (32×), 256^4 = 4,294,967,296 ("≈ 4.3 billion"), 40,000 × 3,072 B = 122.9 MB ("about 123 MB"), 100M × 3,072 B = 307 GB and 100M × 96 B = 9.6 GB, 100M ÷ 256 = 390,625 per code ("about 390,000"), 96 × 256 = 24,576 centroids. The toy example is exact: all four piece-0 distances (0.02 / 0.72 / 1.45 / 0.97), codes [0, 2, 1, 2], [0, 0, 1, 2], [1, 1, 0, 1], the whole 4 × 4 ADC table, ADC sums 3.13 / 3.08 / 1.40 and exact dot products 3.15 / 3.03 / 1.48. The m-table (192 → 16×, 96 → 32×, 48 → 64×, 24 → 128×) is right, and the rescore depth (10–20×, "top 100–200" for k = 10) agrees with Ch 25 and Ch 66. Cross-references checked: Ch 4 (quantization gloss), 6, 17, 19, 23, 25, 32, 37, Part VIII (Ch 57's 5,000 companies and 100 million chunks), SIMD (Ch 17).
- **[NIT]** The m = 24 row breaks the chapter's own rule without saying so. Quote: "| 24 | 24 | 128× | ~0.60 |". That row has d/m = 32, but the text says "Keep $d/m$ between 4 and 16", and the Advantages line repeats "anywhere from 16× to 128×". Fix: label the row "(d/m = 32, outside the safe range)" or change the Advantages line to "16× to 64× in the safe range".
- **[NIT]** The code comment contradicts the running scenario, since Acme has 40,000 pages. Quote: "# e.g. 100,000 Acme page vectors". Fix: "# e.g. 100,000 of Acme's future 100 million chunk vectors" or "# e.g. all 40,000 Acme page vectors".
- Style: no notable violations (no semicolons or em-dashes in prose, one 33-word sentence, bold equation definitions and "In simple words" after the code blocks).
- FIXES.md items for this chapter: all applied ("| — (no PQ) | 3,072 | 1× |", the "96 × 256 table, about 25,000 entries … each entry is an 8-number dot product" wording, and the ColBERTv2 "close cousin … 1–2-bit *scalar* code" wording). The "roughly 30× the storage" price correction in FIXES is a Ch 13 item, not Ch 24. Ch 24 has no price sentence, and the Part 4 `50–100×` grep does not hit it.
- Code: 2 blocks. The ADC snippet reads clean. The `PQ` class needs sklearn, so it was run with a NumPy stand-in for `KMeans` on 20,000 correlated unit vectors (d = 64). Codes are uint8 (n, m), and ADC scores equal q · reconstruction to 1e-4. Distortion and recall move the way the chapter says: d/m = 4/8/16 gave MSE 0.09/0.29/0.49 and recall@10 0.49/0.24/0.12, rising to 0.95/0.74/0.45 after rescoring the top 100. Two small robustness points [NIT]: `np.argpartition(-scores, k)` raises when the collection has ≤ k vectors, and `encode` builds an (n, 256, d/m) temporary with no batching (≈ 330 MB per position for 40,000 float32 pages, impossible at 100M), unlike Ch 23's batched `nearest_centre`.

#### Fix status (applied 2026-09-16)

- [NIT] m = 24 row breaks the d/m 4–16 rule silently — fixed: row labelled "24 ($d/m = 32$, outside the safe range below)" and the Advantages line now says "16× to 64× with $d/m$ in the safe range".
- [NIT] Code comment "100,000 Acme page vectors" contradicts 40,000 pages — fixed differently: "# e.g. 100,000 of the 100 million chunks", and the next comment line now says `all_chunk_vectors` / "96 bytes per chunk" to match.
- [NIT] (Code line) `np.argpartition(-scores, k)` crashes when collection ≤ k — fixed: `min(k, len(scores) - 1)`, same guard as Ch 23.
- [NIT] (Code line) `encode` builds an (n, 256, d/m) temporary with no batching — fixed: `encode(V, batch=20_000)` uses Ch 23's |c|² − 2x·c nearest-centre trick in batches.
- Code: PQ class block changed (encode batching, argpartition guard, comments). Verified with a NumPy stand-in for sklearn KMeans on 5,000 unit vectors (d = 64, m = 16, batch = 777): codes are uint8 (5000, 16) and identical to the old unbatched encode, search works, and search on a 3-vector collection no longer crashes. Code still about thirty lines (27 non-blank).
- Cross-chapter follow-ups: none

## Chapter 25 — IVF-PQ and OPQ in Practice  (reviewer: chapters 23–27)

- No MUST issues found. Recomputed: 1e8 × 16/16,384 = 97,656 ("roughly 98,000"), 1e8 × 96 = 9.6 billion lookups, residual example (levels ±0.25/±0.75, errors 0.13 and 0.20; residuals +0.04/−0.03, errors 0.015 and 0.005), the 45° rotation (variance (4 + 0.01)/2 = 2.005, both blocks 4.01), 768² × 4 B = 2.36 MB ("about 2.4 MB"), both memory ledgers (100M: 9.6 + 0.8 + 0.05 = 10.45 GB. 1B: 3.07 TB raw, 96 + 8 + 0.2 = 104.2 GB), the HNSW column (1e8 × 3,225 B ≈ 322 GB float32, 1e8 × ~921 B ≈ 92 GB int8, consistent with Ch 29), and the nlist examples, which all fall inside √n–4√n (3,162–12,649 / 10,000–40,000 / 31,623–126,491). The FAISS strings `IVF{nlist},PQ{m}`, `OPQ96,IVF4096,PQ96` and `IVF65536_HNSW32,PQ96` are valid factory syntax. "Refine" is FAISS's name for rescoring, and `faiss.extract_index_ivf` is the right way to reach `nprobe` under an `IndexPreTransform`. Rescore depth "10–20× final k" agrees with Ch 24 and Ch 66. A NumPy simulation confirmed the qualitative claims: PQ on IVF residuals had 6.4× lower MSE than PQ on raw clustered vectors, and a rotation evened block variances (71.8/8.0/0.7/0.02 → 21.9/21.8/21.2/15.7) and cut PQ MSE by 27%.
- **[SHOULD]** Phase 1 learns the centroids before the rotation, but Phases 2 and 3 rotate first and then look for the nearest centroid. Followed literally, rotated vectors get compared with centroids trained in unrotated space. Quote: "**Step 2:** Run k-means on the sample to find `nlist` cluster centres, the **centroids**." followed by "**Step 3:** (Optional, recommended) Learn an OPQ rotation, explained below." FAISS's `OPQ…,IVF…,PQ…` (an `IndexPreTransform`) trains the rotation first and runs k-means on rotated vectors. Fix: make Step 2 "(Optional, recommended) Learn an OPQ rotation and rotate the sample", then k-means, residuals and codebooks as Steps 3–5.
- **[NIT]** The `m` rule of thumb excludes a value the chapter later tells us to sweep. Quote: "| `m` (subvectors) | $d/8$ to $d/16$ |" versus "sweep `m` over 48, 96, 192". Here m = 192 is d/4, which Ch 24 allows ("Keep $d/m$ between 4 and 16"). Fix: "$d/4$ to $d/16$".
- **[NIT]** "Residual" now means something different from Ch 24, with no bridge. Ch 24 defined it as "the part PQ got wrong". Here it is "the gap between a vector and its cluster centre". Quote: "We compress the **residual**, the gap between a vector and its cluster centre". Fix: add "(Chapter 24 used the same word for what PQ got wrong. In both cases a residual is whatever a coarser step left over.)"
- **[NIT]** Tuning on a 100k-vector sample with nlist = √n (≈ 316) gives an nprobe knee that does not carry over to the 100M index with nlist = 16,384, because the fraction scanned per probe differs by about 50×. Quote: "Run brute force over a 100k-vector sample with 200 real queries". Fix: "Build the index at full scale, and compute exact top-k for 200 real queries against the full collection (a one-off batch job)", or keep n/nlist the same in the sample.
- Style: no notable violations (semicolons only inside a table, one 34-word sentence, Phase/Step lines before the code).
- FIXES.md items for this chapter: all applied (`IndexIVF_HNSW` is gone, and the "FAISS factory string `IVF65536_HNSW32,PQ96` builds an HNSW graph *over the centroids*" wording is present).
- Code: 1 block (FAISS, not runnable here). It reads clean on API: `index_factory(d, "OPQ96,IVF4096,PQ96", METRIC_INNER_PRODUCT)`, `train`, `add`, `extract_index_ivf(index).nprobe`, and `search(Q, k=100)` are all valid. `train_sample`, `V`, `Q` and `fetch_full_vectors` are placeholders, which is acceptable for a sketch. [NIT] `rescore` does not drop the `-1` IDs FAISS returns when the probed lists hold fewer than 100 codes, so `fetch_full_vectors` could be asked for ID −1. Add `candidate_ids = candidate_ids[candidate_ids >= 0]`.

#### Fix status (applied 2026-09-16)

- [SHOULD] Phase 1 learns centroids before the OPQ rotation — fixed: Step 2 is now "(Optional, recommended) Learn an OPQ rotation … and rotate the sample with it", then k-means (Step 3), residuals (Step 4), codebooks (Step 5). Matches FAISS's `OPQ…,IVF…,PQ…` and the code comment "rotation + centroids + codebooks".
- [NIT] `m` rule "$d/8$ to $d/16$" excludes m = 192 — fixed: "$d/4$ to $d/16$" (matches Ch 24's d/m 4–16 and Ch 66's "4–16 dims each").
- [NIT] "Residual" meaning differs from Ch 24 with no bridge — fixed: added "(Chapter 24 used the word residual for the part PQ got wrong. In both cases, a residual is whatever a coarser step left over.)" in The residual trick section, right after the definition, rather than in the one-paragraph version.
- [NIT] Tuning on a 100k sample gives an nprobe that does not carry over — fixed: Step 1 now computes exact top 10 for 200 real queries over the full collection (one-off batch job). Step 2 builds the index at full size and explains why a small slice's nprobe would not carry over.
- [NIT] (Code line) `rescore` does not drop FAISS's −1 IDs — fixed: `candidate_ids = candidate_ids[candidate_ids >= 0]`, shape comment now "(≤100, 768)".
- Code: FAISS block, only `rescore` changed. Ran `rescore` alone with a stub `fetch_full_vectors` that asserts no negative IDs, on a 100-slot result padded with −1: runs and returns the top 3. The FAISS part is unchanged.
- Cross-chapter follow-ups: none

## Chapter 26 — Scalar and Binary Quantization  (reviewer: chapters 23–27)

- Verified with Python: 1e8 × 768 B = 76.8 GB, 1e8 × 96 B = 9.6 GB, 4× and 32× ratios, and all four int8 encodings (191.25 → 63, 89.25 → −39, 165.75 → 38, 153 → 25) with decodes 0.498 / −0.302 / 0.302 / 0.200. The toy bit strings (10110101, 10100011, 10110100), XORs, popcounts 0 / 3 / 1, exact dot products 0.98 / 0.94 / 0.37 and norms (0.98–1.04, "close to unit length") are all correct. So are 768 bits = 12 × 64-bit words ("12 XORs and 12 popcounts"), "~25 GB of RAM" (9.6 GB codes + ~15 GB graph + 0.8 GB IDs, which matches Ch 60's table), and rescore depth ~100× (1,000 for k = 10, matching Ch 66). Cross-references checked: Ch 1, 3, 5, 12 (uniformity), 15 (binary + MRL), 19, 21, 24, 25, 27, 60.
- **[MUST]** `encode_sq` silently wraps values that fall outside the calibrated range, and the chapter's own procedure produces such values. Phase 1 takes min/max from "a sample", and What people get wrong advises "clip at the 1st and 99th percentiles", but the code never clips, so `astype(np.int8)` wraps around (and float→int8 overflow is undefined behaviour in NumPy). Quote: "return np.round((V - lo) / (hi - lo) * 255 - 128).astype(np.int8)". Measured: min/max from 1,000 of 100,000 unit vectors (d = 768) left 268 values out of range, and 0.176 encoded as −121 while −0.184 encoded as +114. With percentile bounds, 1.5M values overflowed, and the largest coordinate (0.197) became +42 instead of 127. The sign of the largest, most informative coordinates flips. Fix: `return np.clip(np.round((V - lo) / (hi - lo) * 255 - 128), -128, 127).astype(np.int8)`.
- **[SHOULD]** An unsupported claim, in tension with this chapter's own Ninja note and with Ch 27's whole premise. Quote: "Since a well-trained model makes its dimensions informative, this usually works *better* than random directions." Ch 26 then says "Chapter 27 takes the same instinct much further, with a random rotation", and Ch 27's method rests on random rotation *improving* per-coordinate quantization. No source is given (unverified). On synthetic data, sign bits after a random rotation were never worse than raw axes (recall@10 0.37 vs 0.34 with even variances) and far better with uneven per-dimension variance (0.35 vs 0.12). Fix: "On well-trained models the model's own axes usually work about as well as random ones. When dimensions have very uneven spread, a random rotation first (Chapter 27) helps."
- **[NIT]** The rescore function crashes on any collection smaller than `depth`. Quote: "cand   = np.argpartition(dists, depth)[:depth]        # stage 1". On the chapter's own 3-page toy it raises `ValueError: kth(=1000) out of bounds (3)`. Fix: `depth = min(depth, len(dists) - 1)` or fall back to `np.argsort`.
- Style: no notable violations in teaching sections (the one semicolon is in the frontmatter summary, already counted book-wide, and em-dashes appear only in a table cell and a code block).
- FIXES.md items for this chapter: none required ("OK"). The related Ch 66 fix (rescore depth "~100× final k for binary (Ch. 26)") is present and agrees with this chapter.
- Code: 2 blocks, both run. `encode_binary`, `POPCOUNT`, `hamming` and `search_binary_then_rescore` reproduce the toy exactly: Hamming [0, 3, 1], depth 2 returns pages 212 and 87 (the wrong answer), and depth 3 returns 212 and 1,140. On 100,000 structured unit vectors (d = 768), binary recall@10 was 0.35, the true top 10 was inside the binary top 1,000 98.8% of the time (the chapter says "often above 0.98"), and rescoring reached 0.988. int8 recall@10 was 0.986 (~0.99 as tabled). Problems: the int8 overflow (MUST above) and the small-collection crash (NIT above).

#### Fix status (applied 2026-09-16)

- [MUST] `encode_sq` wraps out-of-range values and flips their sign — fixed: `np.clip(np.round(...), -128, 127).astype(np.int8)`, kept as one line so "one line of code" stays true. Also added Phase 2 "Step 4: Clip…" in the prose, saying why (wrap-around flips the sign).
- [SHOULD] "model axes usually work *better* than random directions" (contradicts Ch 27) — fixed: "On well-trained models, the model's own axes usually work about as well as random ones. When some dimensions have far more spread than others, a random rotation first helps (Chapter 27)." Confirmed first with a simulation (d = 256, 20k vectors, sign-bit recall@10: even spread 0.83 axes vs 0.83 rotated, uneven spread 0.82 vs 0.87).
- [NIT] rescore crashes when the collection is smaller than `depth` — fixed: `np.argpartition(dists, min(depth, len(dists) - 1))[:depth]`, same guard as Ch 23.
- [NIT] (Section 0) prose semicolon in the frontmatter summary — fixed: "…for almost nothing. One bit per dimension gives 32×…".
- Code: both blocks changed and run in the venv. `encode_sq` gives 63/−39/38/25 on the worked example. With range ±0.15, 0.16 → 127 and −0.16 → −128 (no wrap). On 20k unit vectors (d = 768) with min/max from 200, all 25 out-of-range values encode as 127. Binary toy: Hamming [0, 3, 1]. depth 2 → pages 212, 87 (0.98, 0.37). depth 3 and depth 1000 on the 3-page toy → pages 212, 1,140 (0.98, 0.94), with no crash.
- Cross-chapter follow-ups: none (Ch 21's "Chapter 26 uses the model's own axes instead" note is still consistent).
- [Lead follow-up from Ch 18 fixer] "roughly 3% of the memory" counted only the 1-bit codes — fixed: "3% of the vector bytes", plus "closer to 8% of the float32 version" with the graph (Chapter 60).

## Chapter 27 — TurboQuant: Near-Optimal Quantization Without Codebooks  (reviewer: chapters 23–27)

- No MUST issues found. Checked against the paper (arXiv 2504.19874, "TurboQuant: Online Vector Quantization with Near-optimal Distortion Rate", Zandieh, Daliri, Hadian, Mirrokni, 28 Apr 2025). All of these match: random rotation from QR of a Gaussian matrix, "concentrated Beta distribution" on coordinates, per-coordinate Lloyd–Max quantizer, the two-stage (b−1 MSE bits + 1-bit QJL on the residual, with the residual norm stored) unbiased inner-product version, upper bound (√3·π/2)/4^b ≈ 2.72/4^b, lower bound 1/4^b, computed MSE 0.36 / 0.117 / 0.03 / 0.009, "≈ 2.7" factor, KV-cache quality neutrality at 3.5 bits per channel, and NN experiments at 2 and 4 bits beating PQ (and RaBitQ) with near-zero indexing time. The Qdrant claim is verified: the Qdrant 1.18 blog (May 11, 2026) and the "TurboQuant in Qdrant" article describe a fast Hadamard rotation, the MSE variant only, a per-coordinate (shift, scale) calibration pre-pass, and a RaBitQ-borrowed length renormalization, exactly as the Note says. Recomputed: Lloyd–Max levels ±0.7979 and ±0.4528/±1.5104 (MSE 0.3634, 0.1175), the whole hand example (rotated vectors [0.8, −0.1, −0.1, 0.6], [0.85, −0.05, −0.15, 0.55], [0.95, 0.25, −0.25, −0.15], bits 1001/1100, scores 1.6 vs 0.2, length² 1.02/1.05/1.05, exact dots 1.03/0.67), 768² = 589,824, 2.36 MB, 1,024 × 10 = 10,240 vs 1,048,576, 192 + 4 = 196 B (19.6 GB), 96 + 4 = 100 B (10 GB), and padding 1,024 × 2 bits = 256 B ("grow by a third"). (The Ch 26 sentence that model axes "usually work *better* than random directions" contradicts this chapter's premise. It is reported under Ch 26.)
- **[SHOULD]** The card-deck analogy's mapping makes a coordinate both a card and a hand. And a plain shuffle is only a permutation, which would not help a per-coordinate quantizer at all. Quote: "The cards are its coordinates. The high cards are coordinates with big values." followed by "Dealing a hand is quantizing a coordinate with a fixed number of bits." The analogy only works if each *hand* is one rotated coordinate made of several cards. Fix: "Each card is one original coordinate. Each player's hand is one rotated coordinate, a mix of several cards. Shuffling and dealing is the rotation. Judging every hand by the same rule is the fixed quantizer."
- **[SHOULD]** The code's scorer contradicts Phase 3 and costs d² per stored vector per query. Quote: "return self.decode(codes, norms) @ q                   # approximate dot products". `decode` rotates every stored vector back (`R @ self.P`), so a query does n × 768 × 768 multiply-adds (≈ 5.9 × 10^13 at 100M vectors) instead of rotating the query once, which Phase 3 Step 1 says to do. Measured: both give the same scores. Fix: `return (self.levels[codes] * norms / np.sqrt(self.d)) @ (self.P @ q)`.
- **[NIT]** The lower bound is stated without its worst-case qualifier. Quote: "the paper proves no quantizer can get the expected squared error below $1/4^b$". The paper's bound is for worst-case unit-vector inputs (via Yao's minimax principle), and structured data can do better, which is how PQ can still win on a given corpus. Fix: "no quantizer can guarantee an expected squared error below $1/4^b$ for every unit vector".
- **[NIT]** The 1-bit two-stage description does not match the paper's construction. Quote: "TurboQuant's 1-bit mode is sign quantization after a random rotation, with an unbiased score estimate available from the two-stage version." At a 1-bit budget the two-stage version spends b − 1 = 0 bits on the first stage, so it is QJL alone (signs of a Gaussian projection), not rotated signs plus a correction. Fix: "…and at the same budget the two-stage version (pure QJL) gives an unbiased score estimate instead."
- **[NIT]** "Lloyd–Max" appears only in a code comment, and the prose never says where the "optimal levels" come from. Quote: "Write down the 4 optimal levels for a standard bell curve". Fix: add "(they come from running 1-D k-means, Chapter 23, on the bell curve itself, called the Lloyd–Max quantizer)".
- **[NIT]** The HNSW run has four chapters, not three (Ch 25 correctly says "Chapters 28–31"). Quote: "begins three chapters on HNSW". Fix: "begins four chapters on HNSW".
- Style: no notable violations (the one semicolon is in the frontmatter summary, already counted book-wide. "you" appears only in advice sections).
- FIXES.md items for this chapter: all applied, and they introduced no new error. The citation is at first mention. The dense QR rotation keeps d = 768, so the table's 192 B + 4 B and 96 B + 4 B match the code, and padding and blocks are explained. The heading reads "In principle it can replace…" with the "could go, not results" Note. The equal-bytes paragraph now says to measure against PQ. The dense-vs-Hadamard caution and the SimHash gloss on QJL are present. The "no mainstream index ships" line was replaced by the Qdrant 1.18 note per the documented deviation, and that note is accurate.
- Code: 2 blocks, both run. `random_rotation(768)` is orthogonal (max error 1e-6) and codes are (n, 768) uint8, so d stays 768. The self-test holds: MSE 0.363 / 0.368 / 0.364 at 1 bit and 0.117 / 0.121 / 0.118 at 2 bits for isotropic, very uneven and one-hot unit vectors ("however uneven the input"). The first-stage inner-product estimate has slope 0.625 at 1 bit (theory 2/π = 0.637), which confirms the bias that motivates QJL. `randomized_hadamard` preserves norm and matches an explicit Sylvester Hadamard matrix on 1,024-d input. Only problem: the scorer cost noted above.

#### Fix status (applied 2026-09-16)

- [SHOULD] Card-deck analogy makes a coordinate both card and hand — fixed: now "Each card is one original coordinate… Each player's hand is one rotated coordinate, a mix of several cards. Shuffling and dealing is the random rotation. Judging every hand by the same rule is the fixed quantizer." The follow-up paragraph now says no single rule for judging hands suits them all, and that TurboQuant "judges every hand with the same well-designed quantizer".
- [SHOULD] Scorer rotates every stored vector back (d² per vector) — fixed: `scores` rotates the query once (`q_rot = self.P @ q`), rebuilds rotated coordinates and returns `R @ q_rot`, following Phase 3 Steps 1–3. `decode` is kept for the self-test.
- [NIT] Lower bound stated without worst-case qualifier — fixed: "no quantizer can guarantee an expected squared error below $1/4^b$ for every unit vector". Also changed "could achieve" to "could guarantee" in the one-paragraph version and the distortion-rate definition so all three agree.
- [NIT] 1-bit two-stage description does not match paper — fixed: "At the same budget, the two-stage version has no bits left for its first stage. It is QJL alone, and it gives an unbiased score estimate instead." (placed after the uneven-coordinates sentence so "it" stays clear). The next sentence now says "Whether TurboQuant matches PQ".
- [NIT] Prose never says where the optimal levels come from — fixed: Phase 1 Step 3 adds "They come from running k-means (Chapter 23) in one dimension on the bell curve itself, which is called the **Lloyd–Max quantizer**."
- [NIT] "begins three chapters on HNSW" — fixed: "four chapters" (Ch 28–31).
- [NIT] (Section 0) prose semicolon in frontmatter summary — fixed: "…cheap but wasteful. Product quantization is efficient…".
- Code: `TurboQuantMSE.scores` changed. Ran both blocks in the venv. On 2,000 vectors with very uneven per-coordinate scales (d = 768), the new scores match the old `decode(codes, norms) @ q` to 1.2e-7 at 1 and 2 bits, with the same top-50 order. The self-test MSE is still 0.363 (1 bit) and 0.117 (2 bits). `randomized_hadamard` still preserves norm.
- Cross-chapter follow-ups: none. Consistency with Ch 26 checked: Ch 26 now says model axes work about as well as random ones on well-trained models, and a random rotation (Ch 27) helps when spread is uneven. That matches Ch 27's premise and its "binary + rescoring when our model's bits are already well balanced" advice.

## Chapter 28 — HNSW I: Small Worlds and Skip Lists  (reviewer: chapters 28–33)

- **[SHOULD]** The Milgram hinge question is not answered by the sentence that follows it, and the real answer then arrives as a stranded "Because…" fragment two sentences later, so a novice reads a non sequitur. Quote: "How does a letter cross a continent in six steps? Each of us knows a few hundred people, and most of them live nearby." followed by "Because **almost everyone has a few unusual connections.**". Fix: move "Each of us knows a few hundred people, and most of them live nearby." before the question, and start the answer "The answer is that almost everyone has a few unusual connections."
- **[NIT]** `mL` appears in What people get wrong ("the parameter that controls it (`mL`) almost never needs tuning") but is never named or defined in this chapter. Fix: "(`mL`, defined in Chapter 29)".
- **[NIT]** In the HNSW worked example every page has a number except the layer-2 stop. Quote: "One hop reaches \"Security and login overview\", on the far side of the Map Room." Fix: give it a page number like the others (e.g. "page 80, \"Security and login overview\"") so the reader can track the walk.
- **[NIT]** Ninja note "The entry point is a bottleneck" argues from CPU cache: "that node's block of CPU cache is read by every query at once". Read-only sharing of a cache line across cores is not a contention point, so the mechanism as stated is doubtful (the throughput benefit of multiple entry points is unverified in the literature). The closing "Measure before relying on it" saves it, but consider softening "bottleneck" to "shared hot spot".
- Style: no notable violations in teaching sections (one em-dash and a "you" inside the pseudocode block; frontmatter summary has an em-dash).
- FIXES.md items for this chapter: all applied ("polylogarithmically" replaces O(log n) for flat NSW; beam search / `efSearch` is glossed at its first use in Problem 1 before the later "wider beam search of Chapter 30").
- Code: 2 blocks, both run clean on a random 2,000-point kNN graph with two layers (`greedy_search_layer` correctly ends each slip pass at the closest listed page as the prose says; single-candidate greedy recall@1 is low, 0.24 in my test, which is exactly the local-minimum problem the chapter describes and Chapter 30 fixes). All Chapter N cross-references (17, 20, 25, 29, 30, 31, 32) point at the right topics per OUTLINE.md; "What's next" links the correct file; readingTime 11 min vs ~12.5 min at 230 wpm is acceptable.

#### Fix status (applied 2026-09-16)

- [SHOULD] Milgram hinge question answered by a stranded "Because" fragment — fixed
- [NIT] `mL` used but never defined in this chapter — fixed
- [NIT] Layer-2 stop in the worked example has no page number — fixed (page 80; no other use of page 80 in the book)
- [NIT] Entry-point "bottleneck" argued from CPU cache sharing — fixed differently: renamed to "shared hot spot", dropped the cache mechanism, said read-only sharing is not by itself contention and the multi-entry-point gain is not well established
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 29 — HNSW II: Building the Graph  (reviewer: chapters 29–33)

No MUST or SHOULD issues found. Verified: m_L = 1/ln 16 = 0.3607; draw 0.5 gives 0.25 (layer 0) and draw 0.03 gives 1.26 (layer 1); P(level ≥ L) = 1/M^L, so 40,000 pages give 2,500 / 156 / 10 at layers 1 / 2 / 3 (a simulated roll of 40,000 gave 2,459 / 154 / 9); M₀ = 2M; the heuristic as defined ("closer to the new node than to every neighbour already accepted") matches hnswlib's `getNeighborsByHeuristic2`; every toy distance in the A–E table and in the query walk (1.00/1.17/1.24/1.51/1.68, 0.41, 0.36, 1.92, 2.65, 2.44, 1.60, 1.94/2.33/1.84, 0.14) is correct to two decimals; 1/(M−1) expected upper layers; 16 × 4 / 15 = 4.27 B; 3,072 + 128 + 4 + 20 = 3,224 B = 1.05× raw; 10 M → 32 GB, 100 M → 322 GB; 128-d at M = 32 gives 792 B = 1.55×, 96-d gives 1.73×; M = 32/48 is 2×/3× the layer-0 edge memory.

- **[NIT]** What people get wrong calls `efSearch` free, which contradicts the chapter's own callout two sections earlier ("`efSearch` (Chapter 30) costs latency per query"). Quote: "It is free until you hit `M`'s ceiling." Fix: "It costs no memory, only latency, until you hit `M`'s ceiling."
- **[NIT]** Taking the floor of an exponential draw gives a geometric (exponentially decaying) distribution over layers, not an exponential distribution. Quote: "The result is an **exponential distribution**". Fix: "The result is an **exponentially decaying distribution**".
- **[NIT]** The memory block says the same thing twice. Quote: "≈     4 B   (≈ 4–5 B)". Fix: keep one, e.g. "≈ 4.3 B".
- **[NIT]** The code calls `dist` with two node IDs (`dist(c, s)`, and `dist(n, x)` while pruning), whereas Chapter 28's `dist` takes `(query, node)`. The prose says both helpers "take the same … `dist` arguments as Chapter 28's code", which only works if the node being inserted is itself an ID that `dist` can look up. Fix: add "Here `dist` takes two node IDs, because the query is the node being inserted."
- Style: no notable violations (semicolons only inside the M and efConstruction table cells; Step lines come before the code; the social-network analogy has its mapping; wrong and right walks are both shown).
- FIXES.md items for this chapter: all applied (upper edges now "≈ 4 B"; the "1.5–2×" note uses 128-d at M = 32 = 792 B ≈ 1.55×; Ch 31 now back-references Chapter 29 for build time).
- Code: 1 block (heuristic + insert), run with Chapter 28's `greedy_search_layer` / `search_entry_point` and Chapter 30's `search_layer` / `hnsw_search` on a dict-based graph. It runs clean and does what the prose says. Recall@10 against brute force: 3,000 × 16-d Gaussian at M = 8, efConstruction = 64 gives 0.77 / 0.91 / 0.98 / 1.00 at efSearch 10 / 20 / 40 / 80+; 10,000 × 32-d at M = 16, efConstruction = 200 gives 0.64 / 0.79 / 0.92 / 0.98 / 0.996 at efSearch 10 / 20 / 40 / 80 / 160. The heuristic claim holds: on 3,000 points in 30 tight 8-d clusters, the heuristic reaches recall 0.99–1.00, but naive top-M selection stays flat at 0.35 whatever efSearch is, which is the "tight cliques with no bridges" failure. Inserting sorted by cluster costs a little, as What people get wrong says (0.97 vs 0.99 at efSearch 10). Layer counts from the code match 1/M per layer.

#### Fix status (applied 2026-09-16)

- [NIT] "efSearch is free" contradicts the latency callout — fixed ("costs no memory, only latency, and it keeps helping until you hit M's ceiling")
- [NIT] Floor of exponential draw is not an exponential distribution — fixed ("exponentially decaying distribution", body and Key takeaways)
- [NIT] Memory block says "≈ 4 B (≈ 4–5 B)" twice — fixed ("≈ 4.3 B"; total corrected to 3,224 B in block and Key takeaways)
- [NIT] `dist` called with two node IDs, unlike Chapter 28 — fixed
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 30 — HNSW III: Searching and Tuning  (reviewer: chapters 29–33)

- **[MUST]** The stopping rule is presented as exact ("nothing left can make the shortlist better"), but it is a heuristic, and the chapter's own example is the counterexample. Page 530 (0.70) is worse than the whole efSearch = 2 shortlist, yet it leads to page 1,140 (0.25). If the rule really stopped only when more walking could not help, a longer shortlist would never find more, and `efSearch` would be pointless. Quote: "nothing left can make the shortlist better. So the search stops when more walking cannot improve the result, not after a fixed number of steps." Also in Key takeaways: "so it quits exactly when more walking cannot help." Fix: "none of the waiting pages can enter the shortlist themselves, so we stop. This is a bet. A worse page can still lead to a better one, as page 530 leads to page 1,140. A longer shortlist makes that bet less often." In the takeaway: "so it stops when no waiting page could join the shortlist; a longer shortlist gives up later." Also soften "stops the moment there is none" to "stops when the shortlist stops improving".
- **[SHOULD]** GloVe is described as easy, but glove-100-angular is one of the harder ann-benchmarks datasets for graph indexes, needing a much higher `ef` than SIFT1M for the same recall. The claim that text embeddings "will typically" do worse at `efSearch = 64` is unverified. Quote: "usually measured on SIFT1M or GloVe, datasets with low intrinsic dimension and friendly structure. Your recall at `efSearch = 64` on real text embeddings will typically be lower." Fix: "usually measured on SIFT1M or GloVe, older datasets whose structure differs from modern text embeddings. Your recall at `efSearch = 64` can be quite different, so always measure on your own data."
- **[NIT]** The Note and What people get wrong disagree about what libraries do when `efSearch < k`. Quote: "Most libraries quietly raise `efSearch` to `k` if we set it lower." vs "Libraries usually clamp silently. Recall suffers and nothing warns you." If the library raises ef to k (hnswlib `max(ef, k)`, which the chapter's own `hnsw_search` also does), recall does not "suffer" relative to the setting. Fix: "Some libraries raise `efSearch` to `k` silently, others return fewer results. Either way nothing warns you."
- **[NIT]** The cost estimate's top end exceeds the maximum stated one line later. Quote: "≈ 1,000–5,000 for efSearch = 128, M = 16" vs "the most the search could compute is 128 × 32 = 4,096 distances". Fix: "≈ 1,000–4,000", and "roughly the most", because nodes expanded can slightly exceed efSearch and the upper-layer descent adds a few dozen. In my run (10,000 × 32-d, M = 16) efSearch = 128 averaged 2,387 distance computations and 128.3 expansions.
- Style: no semicolons or em-dashes. Phase/Step lines come before the code, the house-hunting analogy has its mapping, and a wrong answer (efSearch = 2) and a right answer (efSearch = 4) are both shown. No notable violations.
- FIXES.md items for this chapter: all applied ("Latency grows about 1.6–1.7× each time `efSearch` doubles". Recomputed from the table: 1.56, 1.61, 1.67, 1.73).
- Code: 2 blocks. (1) The hnswlib measurement loop cannot run here (no hnswlib). Read clean: `set_ef`, and `knn_query(q, k=10)[0][0]` is the label row for a single 1-D query; inner-product ground truth is right for normalised vectors. (2) `search_layer` / `hnsw_search` / toy graph run clean and print exactly "efSearch=2: pages [212, 45]" and "efSearch=4: pages [212, 1140]". I traced both walk-throughs step by step and every shortlist in the prose matches the heap state, including 77 being left in "to explore" and triggering the stop. On a 10,000 × 32-d graph built with Chapter 29's code (M = 16, efConstruction = 200), recall@10 was 0.63 / 0.89 / 0.97 / 0.99 / 1.00 / 1.00 at efSearch 10 / 32 / 64 / 128 / 256 / 512, so the illustrative table has a plausible shape. The k ≤ efSearch rule holds in the code via `max(ef_search, k)`. Cross-references (6, 13 reranking and adaptive routing, 17, 18, 26, 28, 29) are correct. "We will cover" matches the eight headings.

#### Fix status (applied 2026-09-16)

- [MUST] Stopping rule presented as exact, page 530 is the counterexample — fixed (body now says no waiting page can join, calls the rule a bet with 530 → 1,140, "In simple words" line rewritten, Key takeaway rewritten, code comment "cannot improve" → "none can join")
- [SHOULD] GloVe wrongly described as easy / text "typically lower" — fixed (also softened "single most common reason" to "a common reason")
- [NIT] Note and What people get wrong disagree on efSearch < k — fixed (both now say some libraries raise to k, others return fewer results)
- [NIT] Cost estimate top end 5,000 exceeds stated max 4,096 — fixed ("1,000–4,000", "roughly … plus a few dozen in the upper layers")
- Code: search_layer block changed only in a comment; extracted and ran it, prints "efSearch=2: pages [212, 45]" and "efSearch=4: pages [212, 1140]" as commented
- Cross-chapter follow-ups: none

## Chapter 31 — HNSW in Production  (reviewer: chapters 29–33)

- **[MUST]** The nightly recall check computes ground truth on a *sample of the corpus*, but queries the index over the *whole* corpus. A perfect index then scores roughly the sample fraction. In my simulation, a perfect top-10 scored 0.09 recall against truth computed on a 10% corpus sample. The two sets also use different ID spaces, since `exact_topk` over `sample_vectors` returns positions within the sample. Quote: "compute exact top-10 answers on a sample of the corpus, and record recall@10" and code "`truth = exact_topk(sample_queries, sample_vectors, k=10)`". The same idea appears in What people get wrong: "Keep a sampled exact index. It is cheap." Fix: sample the *queries*, not the corpus: "Every night, sample 200 queries, compute their exact top-10 over the full corpus with one batched flat scan (Chapter 17), and record recall@10." In the code, pass `all_vectors` (with the index's own labels) and say "Keep a flat copy of the vectors to score sampled queries against." Chapter 18's code, which this block cites, already uses "V: all page vectors".
- **[SHOULD]** FAISS is listed as an index whose deletes keep memory, but FAISS's HNSW index has no delete at all (`remove_ids` is not implemented for `IndexHNSW*`). FAISS's flat and IVF indexes do remove IDs and give the memory back, which matches this chapter's own "IVF's trivial deletes". This was introduced by the FIXES.md wording. Quote: "They do not in hnswlib, FAISS, Lucene or most databases (Vespa is a notable exception)." Fix: "They do not in hnswlib, Lucene (until segments merge) or most databases. FAISS's HNSW index cannot delete at all. Vespa is a notable exception."
- **[NIT]** hnswlib, the chapter's own example library, does support an in-place update: `add_items` with an existing label replaces that element's vector and re-links it (hnswlib README). So the general statement is fine for Lucene, Qdrant and Milvus (delete + insert, reclaimed at segment merge or vacuum/compaction), but it reads as covering hnswlib. Quote: "Most implementations have no true in-place update. The operation is delete-then-insert". Fix: add "(hnswlib can also overwrite an existing label in place, at a similar cost)".
- **[NIT]** The shop analogy introduces "houses" that the mapping list never maps. Quote: "so houses behind it are cut off." Fix: "so the shops behind it (pages like 1,140) are cut off", and the same change in the soft-delete bullet.
- **[NIT]** The two-index ASCII box is misaligned: the top border is 64 characters, the content rows are 65–66, the bottom is 65, and "brute force│" touches the border. Fix: pad all four rows to the same width.
- **[NIT]** "grew by topic" is illustrated with years. Quote: "Suppose Acme's knowledge base grew by topic: all the 2024 pages, then all the 2025 ones." Fix: "grew in order: all the 2024 pages, then all the 2025 ones."
- Style: no semicolons or em-dashes. Phase/Step lines come before both code blocks, the hard-delete wrong answer and soft-delete right answer are both shown, and there is an Advantages/Disadvantages/When to use close. No notable violations.
- FIXES.md items for this chapter: tombstone is defined at the soft-delete definition (applied). The binary "~25 GB" and DiskANN "~5 GB" rows now agree with Ch 60's table (applied, and I verified Ch 60 has int8 ~92 GB, MRL-256+int8 ~42 GB, binary ~25 GB, DiskANN ~5 GB). The Vespa-exception wording was applied but named FAISS incorrectly (see SHOULD above). The Ch 29 back-reference for build time is applied.
- Arithmetic verified: 100 M × 3,072 B = 307 GB; edges 100 M × (128 + 4.3) B = 13 GB; overhead 3 GB (30 B/vector, vs Ch 29's 20 B, which is immaterial); total 323 GB; vectors 95% of the bill; int8 768 + 128 + ~30 = 926 B → 92.6 GB, so "~90 GB" is fine; Matryoshka-256 float32 1,186 B → 119 GB; int8 + MRL-256 418 B → 42 GB; binary 258 B → 26 GB; 128 × 0.7 = 89.6 → "~90"; 1/40 shards = 2.5%.
- Code: 2 blocks, hnswlib-dependent, so not run. Block 1 reads clean: `allow_replace_deleted=True`, `mark_deleted(label)` and `add_items(vec, label, replace_deleted=True)` are the real hnswlib API. Block 2 has the sampling bug above. `exact_topk` / `mean_overlap` do not exist as named functions in Chapter 18 (they are inline steps there), which the comment half-admits. Cross-references (15, 23, 26, 29, 30, 50, 57/Part VIII, 58, 59, 60, 63) point at the right topics.

#### Fix status (applied 2026-09-16)

- [MUST] Nightly recall truth computed on a corpus sample, wrong ID space — fixed (body: exact top-10 over the full corpus with one batched flat scan; Steps and code: truth over every live vector, row numbers mapped to index labels via `live_labels`; What people get wrong: keep a flat copy of the vectors)
- [SHOULD] FAISS listed as keeping memory on delete, but FAISS HNSW cannot delete — fixed
- [NIT] hnswlib does support in-place overwrite of a label — fixed
- [NIT] Shop analogy introduces unmapped "houses" — fixed (both bullets now "shops behind it", first with "(pages like 1,140)")
- [NIT] Two-index ASCII box misaligned — fixed (all four rows padded to the same width, arrows recentred)
- [NIT] "grew by topic" illustrated with years — fixed ("grew in order")
- Code: nightly_index_health block rewritten (signature now `index, live_vectors, live_labels, sample_queries`). Verified with a toy harness: exact_topk/mean_overlap as in Ch 18, a "perfect" exact index with non-row-number labels and 10% tombstones, 200 queries over 5,000 × 32-d. New block scores recall@10 = 1.0. The old block's approach scored 0.0005 on the same index. hnswlib update block unchanged.
- Cross-chapter follow-ups: none

## Chapter 32 — DiskANN: Billion-Scale on SSD  (reviewer: chapters 29–33)

- **[SHOULD]** The chapter says each node takes one 4 KB block, but then sizes the SSD at the unpadded 3.3–3.6 KB per node. One 4,096-byte block per node × 1 B = 4.1 TB (3.73 TiB), which does not fit the "4 TB NVMe drive" in the table (3.64 TiB). DiskANN's disk layout does pad each 768-d node to a full 4,096-byte sector. Quote: "Each node sits on the SSD as one **4 KB block**" vs "**~50 GB + ~3.5 TB on a 4 TB NVMe drive**" and "≈ 3.3–3.6 KB per node, so ~3.5 TB". Fix: "≈ 3.3–3.6 KB of data, padded to one 4 KB block, so ~4.1 TB on SSD", and use an 8 TB drive (or two 4 TB drives) in the table and Key takeaways. Ch 60's "~330 GB SSD" for 100 M has the same omission (≈ 410 GB).
- **[SHOULD]** The chapter overstates what happens without parallel reads, and its own numbers contradict it. With Vamana and PQ navigation, ~100 sequential reads × 100 μs = ~10 ms, which is still inside the chapter's own "2–10 ms" band. The 300 ms collapse came from HNSW's 3,000 reads, not from synchronous I/O. Quote: "Without parallel async reads, every hop costs 100 μs one after another, and the whole design collapses." Also "Parallel reads are the difference between DiskANN being a curiosity and being the standard billion-scale answer" and Key takeaways "Synchronous reads destroy the design." Fix: "Without parallel reads, ~100 hops cost ~10 ms one after another, several times slower than a beam of 4. The big win is Vamana plus PQ navigation cutting 3,000 reads to ~100. Parallel reads then cut the wait by another 3–4×."
- **[NIT]** The round count does not match W = 4. Quote: "That is roughly 100 reads, in about 15–30 rounds." At W = 4, 100 reads need at least 25 rounds. 15 rounds needs W ≈ 7. Running the chapter's `diskann_search` with L = 100 gave 106–118 reads in 28–31 rounds at W = 4, and 16–18 rounds at W = 8. Fix: "roughly 100 reads, in about 25–30 rounds", and "roughly 2.5–3 ms waiting on the SSD".
- **[NIT]** The density comparison sets Vamana's single-layer degree against HNSW's upper-layer `M`. HNSW's layer 0, the fair comparison, holds 2M = 32–64 links (Chapter 29), so Vamana is about 2× denser, not 4×. Quote: "typically 64–128 neighbours per node versus HNSW's 16–32." Fix: "versus 32–64 on HNSW's layer 0". R = 64–128 itself is consistent with DiskANN's README guidance (R roughly 60–150).
- **[NIT]** The build section never names its own search-list size. Step 3 "run a search for it from the medoid" uses a list size (DiskANN's build `L`, the `efConstruction` analogue, typically ~75–200 per the DiskANN README). The chapter only defines `L` for queries. Fix: add "with a search list of size `L` (often 100–200), the `efConstruction` of Vamana."
- **[NIT]** "exact" overstates the rescore. Only the visited nodes are rescored exactly, so the final answer can still miss true neighbours that were never visited. Quote: "The final answer is exact and needs no extra reads." Fix: "The final ranking uses exact distances and needs no extra reads."
- **[NIT]** The reference DiskANN implementation on Linux uses libaio (its build needs `libaio-dev`), not io_uring. Quote: "On Linux this is done with **io_uring**". Fix: "On Linux this is done with asynchronous I/O such as libaio or **io_uring** (Linux's fast asynchronous I/O interface)".
- Verified correct: the alpha-pruning rule "drop c if α · d(s, c) ≤ d(p, c)" matches RobustPrune in Subramanya et al. 2019. α = 1 reduces to Chapter 29's rule. α > 1 keeps longer edges: on 3,000 × 16-d points with 200 candidates, α = 1.2 kept 44.6 edges per node vs 13.5 at α = 1, and the extra edges averaged 3.66 vs 3.30 in length. The two-pass build (α = 1 then α ≈ 1.2), random R-regular start, medoid entry, and back-edge pruning match the paper. 1,000× (100 μs vs 100 ns); 3,000 × 100 μs = 300 ms; float32 HNSW 3,234 B × 1 B = 3.2 TB; int8 926 B → ~900 GB ("~800 GB" is gone); IVF-PQ 96 + 8 = 104 B → ~105 GB; PQ 32–64 B → 32–64 GB; 900 / 50 = 18× (">10×"); 768-d node with R ≤ 128 (3,588 B) fits one 4 KB block; Fresh-DiskANN's in-memory temp index + delete list + streaming merge.
- Style: no semicolons or em-dashes. Phase/Step lines come before the code, the warehouse analogy has its mapping, and a wrong design (300 ms) and a right one (ms) are both shown. No notable violations.
- FIXES.md items for this chapter: all applied (rescore uses vectors already read, as Step 8; W = 2–8 with L ≈ 100 and `beam_width=4`; IOPS and io_uring glossed; "~900 GB"; the log-structured sentence now back-references Chapter 31).
- Code: 1 block (`diskann_search`), run with a small Vamana graph (RobustPrune, two passes), an 8×256 PQ codebook and a mock SSD that counts reads. It runs clean: recall@10 = 1.00 on 4,000 × 32-d clustered points at L = 100 for W = 1/2/4/8. Reads per query were 102–128 and `read_many` calls ≈ reads / W, matching "called about `100 / W` times per query". Every "Chapter N" reference (6 coarse-then-refine, 18 cascade, 20 strategies 3/4, 24, 25 rescoring, 29, 30, 31, 33) is correct.

#### Fix status (applied 2026-09-16)

- [SHOULD] SSD sized unpadded (3.5 TB), but one 4 KB block per node — fixed per D6 (~4.1 TB in the RAM/SSD table, economics table "on an 8 TB NVMe drive, or two 4 TB drives", derivation "padded to one 4,096-byte block", Key takeaways)
- [SHOULD] "Collapses without parallel reads" contradicted by own numbers — fixed (Step 4 paragraph, What people get wrong and Key takeaways now say Vamana + PQ cut 3,000 reads to ~100, parallel reads cut the wait another 3–4×, ~10 ms without them; dropped "most common reason" and "curiosity" claims)
- [NIT] Round count 15–30 does not match W = 4 — fixed ("25–30 rounds", "2.5–3 ms")
- [NIT] Density compares Vamana degree with HNSW upper-layer M — fixed ("versus 32–64 on HNSW's layer 0")
- [NIT] Build search-list size never named — fixed differently: added "search list of size `L` (often 75–200, the `efConstruction` of Vamana)", using the DiskANN docs' 75–200 range rather than 100–200
- [NIT] "Final answer is exact" overstates rescore — fixed ("final ranking uses exact distances")
- [NIT] Reference implementation uses libaio, not io_uring — fixed (libaio or io_uring, body and Key takeaways)
- Code: no code changed
- Cross-chapter follow-ups: none (Ch 60 already shows ~410 GB SSD for 100M. Ch 66 line 92 quotes only "≈ 5 GB RAM + SSD", no TB figure, so nothing to mirror)

## Chapter 33 — Filtered Vector Search  (reviewer: chapters 29–33)

- **[SHOULD]** The chapter calls filter-aware graph construction "under-explored", but it has been published and shipped. Filtered-DiskANN (Gollapudi et al., WWW 2023: FilteredVamana and StitchedVamana) builds label-aware edges. Qdrant's filterable HNSW adds extra links for each indexed payload value to the main graph, so filtered traversal stays connected. The cost stated here is also questionable. Qdrant *adds* edges rather than replacing global ones, so it pays in memory and build time, and a loss of unfiltered recall is unverified. Quote: "You trade a little unfiltered recall for a lot of filtered recall. This is under-explored, and often a large win for multi-tenant products." Fix: "Filtered-DiskANN (2023) and Qdrant's payload-index links do this. You pay extra edges and build time for much better filtered recall. It is often a large win for multi-tenant products."
- **[NIT]** The in-filter band excludes the chapter's own billion-scale example. Quote: "the pass rate is between about 0.1% and 10%" (and the When-to-use row "pass rate 0.1–10%, big set") vs "At a billion vectors, the same 0.04% is 400,000 vectors, and in-filtering is the better choice." The decision table itself ("Pass rate < 10%, allowed set > 50k") is the consistent rule. Fix: change the prose and the When-to-use row to "pass rate below ~10% with an allowed set too big to brute force". Also note that at very low pass rates plain in-filtering is where disconnection bites, which is why ACORN-style traversal or a partition is preferred there.
- **[NIT]** ACORN is overstated, and the construction half is missing. Two-hop expansion improves connectivity of the filtered subgraph but does not guarantee it. ACORN-γ also builds a denser graph (γ·M neighbours) at construction time. Quote: "The filtered subgraph stays connected, because you step over the non-matching nodes". Fix: "The filtered subgraph is far more likely to stay connected…", and name the source once: "ACORN (Patel et al., SIGMOD 2024)".
- **[NIT]** The pre-filter snippet indexes a NumPy array with "a set of IDs", which raises an error, and it returns positions within the subset rather than page IDs. Quote: "`allowed = metadata_index.lookup(tenant=\"acme\")               # a set of IDs`" / "`vectors[allowed]`". Fix: "# an array of IDs", and map the returned positions back through `allowed`.
- **[NIT]** The brute-force timing is quoted without its hardware. Chapter 17 gives 40,000 vectors as "2.5–6 ms on one core, about 1 ms on several", so 50,000 is ~3–8 ms single-core. Quote: "Recall Chapter 17: brute force over 50,000 vectors takes a millisecond or two." Fix: "a millisecond or two on a few cores".
- Verified correct: 40,000 / 100 M = 0.0004. 1,000 × 0.0004 = 0.4 expected. P(zero) = 0.67 ("two queries in three"), and P(one | at least one) = 0.81 ("most of the rest come back with one"). 2k/p = 50,000 (Acme), 20,000 (p = 0.001), 1,000 (Globex, 2 M / 100 M = 2%). The simulation claims are exact binomial results: 30 candidates at p = 0.3 come up short 58.9% of the time ("about 6 in 10"); int(2k/p) = 66 comes up short 0.16% ("fewer than 2 in 1,000"); k/p = 33 comes up short 45% ("nearly half the time"). 0.04% of 1 B = 400,000. Globex as a tenant of Acme's 5,000-company product matches Ch 62. "(Chapter 18)" for "the SSO chunk of page 1,140" is right (Ch 18 defines it). The "estimate the pass rate first, then choose" routing matches Qdrant's query planner (payload-index cardinality vs a full-scan threshold) and Weaviate's flat-search cutoff.
- Style: no semicolons or em-dashes. The unfiltered wrong answer (another tenant's "all team sizes") and the pre-filter right answer are both shown, Step lines come before every code block, and there is a When-to-use close. No notable violations.
- FIXES.md items for this chapter: all applied ("pass rate" defined once at first use with the selectivity clash explained; `pass_rate` variable; `int(2 * k / pass_rate)`; "above ~30% (10–30% also works, with a larger fetch)"; table row "Allowed set < ~50k vectors").
- Code: 4 blocks (post-filter, pre-filter, in-filter sketch, `filtered_search`). `filtered_search` run with mock metadata/ANN objects routes correctly: Acme (40,000) goes to brute force, Globex (2%) to in-filter, "en" (70%) to post-filter. At pass rate 0.35 the `2k / pass_rate` fetch came up short on 0.17% of 20,000 trials. Post-filter snippet reads clean. Pre-filter snippet has the set-indexing NIT above. Cross-references (7 inverted index, 17, 18, 19, 30, 51, 59, 62) are correct, and "What's next" links 34-single-vector-bottleneck.md.

#### Fix status (applied 2026-09-16)

- [SHOULD] Filter-aware construction called "under-explored", cost misstated — fixed (names Filtered-DiskANN, WWW 2023, and Qdrant's extra HNSW edges per indexed payload value, both checked against the ACM record and Qdrant's indexing docs. Cost is now extra edges in memory and build time. Key takeaway updated to match)
- [NIT] In-filter band 0.1–10% excludes the chapter's billion-scale example — fixed (prose and When-to-use row now "below ~10%, set too big to brute force". Added that at very low pass rates plain in-filtering is where disconnection bites, so prefer ACORN-style traversal or a partition)
- [NIT] ACORN overstated, construction half missing — fixed ("far more likely to stay connected", cites Patel et al., SIGMOD 2024, adds the denser graph built at construction time, without the γ·M detail)
- [NIT] Pre-filter snippet indexes a NumPy array with a set, returns subset positions — fixed ("an array of IDs", positions mapped back through `allowed`)
- [NIT] Brute-force timing quoted without hardware — fixed ("on a few cores" for 50,000, and the 40,000 case in the same section changed to "about a millisecond on a few cores", matching Ch 17)
- Code: pre-filter snippet changed. Ran it with a toy harness (2,000 × 16-d, every fifth vector "acme", argsort brute force returning positions). `results` equals the exact filtered top 10 and every ID is an acme ID. Other blocks unchanged.
- Cross-chapter follow-ups: none (Ch 66 line 138 already says "pass rate < 10%, allowed set > 50k ► in-filter traversal (ACORN-style if available)", which is consistent)

## Chapter 34 — The Single-Vector Bottleneck  (reviewer: chapters 34–41)

- No MUST-level issues found. All arithmetic recomputed and correct: 768 × 32 = 24,576 bits; 300 words ≈ 1,800 B ≈ 14,400 bits; 200 × 128 × 4 = 102,400 B (33.3× of 3,072); 200 × 36 = 7,200 B (2.34×); C(4,2) = 6 pairs with 2 reachable in 1-d; C(6,2) = 15 with 6 reachable in 2-d (confirmed by running the code, and also for five random 6-vector arrangements, supporting "The same holds for *any* arrangement"). Weller et al. 2025 claims (sign-rank bound within one, ~1.7 M docs at 768-d, LIMIT = 50 k docs / 1,000 queries / 2 relevant each, < 20 % recall@100) match the paper as I recall it.
- **[NIT]** Sentence count is off by one. Quote: "But twenty other sentences share the same vector" and "Sharing a vector with twenty sentences about audit logs". The `page_1140` string is "Security add-on." + 20 `other_lines` + the caveat = 22 sentences, so 21 others share the vector (the "about 260 words" claim is right: 258). Fix: "twenty-one other sentences" or "about twenty".
- **[NIT]** Cross-reference. Quote: "or per image patch (a small square cut from a page image, Chapter 42)". Chapter 42 (CLIP) defines patches of ordinary images; patches of *page images* are Chapters 45–46 (ColPali). Fix: "Chapter 42 for patches, Chapter 45 for page images" or simply "Chapter 45".
- **[NIT]** Frontmatter `readingTime: "14 min"` for ~3,900 words (~17 min at 230 wpm). Fix: "16 min" or "17 min".
- Style: no notable violations (no semicolons, one em-dash only in frontmatter, analogy has its mapping, "we" used for shared work).
- FIXES.md items for this chapter: all applied (geometric/sign-rank argument replaces the bit-count argument; "about 33× the bytes (call it ~30×)"; "~7,200 (2-bit residual + centroid ID, 36 B/token)"). The dilution experiment replacement is coherent: prose table, code comment (`score=0.63 rank=3`, `score=0.51 rank=6`) and Key takeaways all agree.
- Code: 2 blocks. The 2-d geometric-limit block runs and prints "6 of 15 pairs can ever be returned" as stated. The `bge-base-en-v1.5` dilution block cannot be run here (no sentence_transformers); read clean (shapes `(7,768) @ (768,)` and `(768,) @ (768,)` are right, rank formula and "of 8" = 7 rivals + target are right). Reported scores 0.79/0.66/0.59/0.58/0.54/0.51 and 0.63 are unverified but internally consistent with ranks 6 and 3.

#### Fix status (applied 2026-09-16)

- [NIT] Sentence count "twenty other sentences" off by one — fixed differently: changed to "about twenty" in both places (the extra one is the "Security add-on." heading fragment, not a sentence about audit logs).
- [NIT] Page-image patch cross-reference points to Chapter 42 — fixed differently: "Chapters 42 and 45" (42 defines patches, 45 cuts page images).
- [NIT] readingTime 14 min — skipped: readingTime NITs are superseded by the brief.
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 35 — ColBERT I: Late Interaction  (reviewer: chapters 35–39)

- No MUST or SHOULD issues found. Verified: every tokenization claim, using a WordPiece pass over the local `bert-base-uncased` vocab.txt. "SSO is included on the Pro plan." gives `ss ##o is included on the pro plan .`, so with [CLS], [D] and [SEP] that is 12 tokens, and dropping "." leaves the stated 11 vectors. `SSO-4012` gives `ss ##o - 401 ##2` and `SSO-4021` gives `ss ##o - 402 ##1`, both exactly as printed. I also recomputed the arithmetic. Grid maxima 0.93/0.64/0.86/0.57 sum to 3.00. The page table sums are 3.65 / 3.00 / 2.85 / 1.72, so page 1,140 is 2nd. The cross-encoder over 40,000 pages is 40 × 100–300 ms = 4–12 s, and over 10M docs it is 17–50 min (matches the table). Raw storage is 200 × 128 × 4 = 102,400 B = 33.3×, compressed 200 × 36 = 7,200 B = 2.34×, and 768-d is 614,400 B ≈ 600 KB with 768/128 = 6×. The Ninja example is 32-d × 2-bit = 8 B, and 200 × 8 = 1,600 B = 0.52 of 3,072. Also correct: Nq = 32 with [MASK] padding; [Q]/[D] reusing unused vocab slots; punctuation filtering for documents only; MRR@10 ≈ 0.36 for v1 and ≈ 0.40 for v2 (matches the papers as I recall them). Consistent with Ch 34 (6th place, top 3, page 87) and Ch 13/66 (~30× bytes, ~200× vectors).
- **[NIT]** The `##` subword prefix is shown but never explained, and Ch 10's tokenizer example does not use it. Quote: "splits \"SSO\" into `ss` and `##o`". Fix: add "(the `##` means the piece continues the previous one)".
- **[NIT]** Small internal inconsistency. The rare-terms paragraph says every piece, including `-`, "keeps its own vector", but Phase 1 Step 5 throws away punctuation vectors on the page side. Quote: "`ss`, `##o`, `-`, `401` and `##2`. … Each piece keeps its own vector". Fix: "Each word and number piece keeps its own vector".
- **[NIT]** The three-teams analogy maps the spokesperson, the room and the cards, but never says which team is which. Quote: "The cards are the token vectors." Fix: add "Team A is the question. Team B is the page."
- **[NIT]** Dangling modifier. Quote: "Used only as a reranker over a bi-encoder's candidates, you inherit the bi-encoder's recall ceiling". Fix: "If we use it only as a reranker over a bi-encoder's candidates, we inherit…".
- Style: no semicolons or em-dashes in the body, Phase/Step lines come before the code, and the chapter has a wrong-then-right example and bold-equation definitions. Only the analogy mapping above is missing.
- FIXES.md items for this chapter: all applied (MS MARCO glossed as the 8.8-million-passage benchmark with MRR@10 → Chapter 19; "~30× raw bytes (~200× vector count)"; [D] marker and punctuation-filtering Note added). The cross-encoder latency "100–300 ms per 1,000 passages" agrees with the Ch 37 fix.
- Code: 1 PyTorch block, not runnable here (no torch). It reads clean: the projection has bias=False as in ColBERT, the shapes (B,T,768)→(B,T,128) and (nq,nd) are right, and `max(dim=1).values.sum()` is MaxSim. I replicated the toy grid in numpy float32. The sum is 2.9999998, so `tensor(3.0000)` is the correct printed form.

#### Fix status (applied 2026-09-16)

- [NIT] `##` subword prefix never explained — fixed
- [NIT] "Each piece keeps its own vector" vs punctuation dropping — fixed
- [NIT] Three-teams analogy never says which team is which — fixed
- [NIT] Dangling modifier "Used only as a reranker…, you inherit" — fixed
- [NIT] Section 0: prose semicolon — fixed differently: the only `;` in teaching text was LaTeX spacing `\;` in the MaxSim formula, replaced with `\,` (same rendering, no semicolon).
- Consistency: "twenty sentences" (x2) changed to "twenty or so" / "about twenty" to match the Ch 34 NIT fix (page 1,140 has 21 other sentences).
- Code: no code changed
- Cross-chapter follow-ups: none (the summary's "Keyword rescue of page 1,140" item is a Ch 11 fix; Ch 35 attributes the SSO match to ColBERT, not BM25, so it is already correct)

## Chapter 36 — ColBERT II: MaxSim and Training  (reviewer: chapters 35–39)

- **[SHOULD]** This contradicts Ch 35. Ch 35 (and this chapter's own Note) pads every ColBERT query to a fixed 32 vectors with [MASK] tokens, so every query has the same number of terms, and "normalise by query length" does nothing for ColBERT. Quote: "The sum has one term per query vector, so a query with more vectors can reach a higher total." Fix: "ColBERT pads every query to 32 vectors, but models without fixed padding (ColPali, many PyLate models) sum over the real query tokens, so longer queries reach higher totals. Even at equal length, some questions simply match better than others."
- **[NIT]** "Length-proof" overstates it. Adding tokens can never lower a max, so MaxSim never goes down as a page grows, and very long pages keep a small edge from having more chances at a high match. Quote: "it is length-proof, compositional". Fix: "it is length-robust (extra tokens add nothing unless they match better)".
- Verified: the arithmetic in every worked table, recomputed. Page 212 row maxima give 0.90 + 0.85 = 1.75, and the Enterprise page gives 0.95 + 0.30 = 1.25. With an outer max the Enterprise page's 0.95 beats page 212's 0.90. Row means give (0.90 + 399 × 0.10)/400 = 0.102 → 0.204 against 0.60. Row sums give 81.6 against 24 against 400. Maxima give 1.80 / 0.60 / 0.20. Problem 1 is 32 × 200 = 6,400, and ×1M = 6.4 billion. The training description matches ColBERTv2 as I recall the paper: KL distillation from a MiniLM cross-encoder, 64-way tuples mined by a first-round ColBERT, and in-batch negatives. The RocketQA attribution for discarding is right, and so is v1's BM25-negative triples. The Chamfer definition matches MUVERA's asymmetric Chamfer similarity. Cross-references check out: Ch 4, 8, 9 (softmax), 10, 11 (padding bug), 12 (KL, margin-MSE, page 1,140 false negative), 13, 34, 37, 38, 41, 55 (threshold calibration). The toy numbers match Ch 35's grid (0.93 / 0.64 / 0.57 / 0.86).
- Style: no semicolons or em-dashes in the body, Phase/Step lines come before the code, and every combination shows a wrong-then-right answer. No violations.
- FIXES.md items for this chapter: all applied (KL over 64-way tuples replaces "discard", with RocketQA named as the discarding variant; "three *training* changes (its fourth contribution, residual compression, is Chapter 37)"; the padding paragraph now covers both unmasked unit-norm rows and zeroed rows beating a negative max, with −∞ masking). None introduced an error.
- Code: 1 PyTorch block, not runnable here. I ported `maxsim_batch` to numpy and ran it on random unit vectors (B=3, nq=32, nd=50, real lengths 50/20/5). The masked result equals per-document MaxSim over the real tokens only, and the unmasked version inflates the two padded documents (5.85 → 6.69, 3.49 → 6.71), exactly as the prose warns. `kl_div(log_softmax(student), softmax(teacher), "batchmean")` uses the correct argument order, and the (B, w) stacking is right. The sanity check gives 1.75, so `tensor(1.7500)` is correct.

#### Fix status (applied 2026-09-16)

- [SHOULD] Query-length normalisation contradicts ColBERT's fixed 32-vector padding — fixed differently: says ColBERT always has 32 terms, models without fixed padding such as ColPali (Chapter 45, "about 20 vectors") vary, and normalise "when lengths vary". Dropped "many PyLate models" because PyLate's ColBERT models also pad queries to a fixed length by default.
- [NIT] "Length-proof" overstates it — fixed
- [NIT] Section 0: prose semicolon — fixed differently: the only `;` in teaching text was LaTeX spacing `\;` in the MaxSim formula, replaced with `\,`.
- Code: no code changed
- Cross-chapter follow-ups: none
- Later consistency edit (during Ch 37): Ninja notes "PLAID's centroid pruning" changed to "PLAID's centroid-based search", matching Ch 37's corrected PLAID stages.

## Chapter 37 — ColBERT III: ColBERTv2, PLAID, and Serving  (reviewer: chapters 35–39)

- **[SHOULD]** The PLAID pipeline puts centroid pruning in the wrong place. As I recall the PLAID paper (Santhanam et al. 2022, §4), candidate generation takes each query token's top-`nprobe` centroids (nprobe = 1–4) and gathers their passages. The `t_cs` threshold (0.4–0.5) is then applied during *centroid interaction*: a candidate's tokens whose centroid scores below `t_cs` are ignored while approximate scores pick the top `ndocs`. Those are re-scored without pruning, and only the top `ndocs/4` are decompressed for full MaxSim. The chapter applies one global threshold before candidate generation (Steps 2–3 and `plaid_candidates`). A global top-c ranked by the best token score also lets a few strong query tokens take every slot. Quote: "**Step 2: Prune the centroids.** Give each centroid its best score against any query token, and drop centroids below a threshold." Fix: "Step 2: For each query token, keep its few nearest centroids (nprobe). Step 3: every passage with a token at one of those centroids is a candidate. Step 4: approximate MaxSim from centroid scores, ignoring tokens whose centroid scores below a threshold (centroid pruning), then re-score the best without pruning." In the code, take `np.argsort(-(Q @ centroids.T), axis=1)[:, :nprobe]` per query token.
- **[SHOULD]** Running-scenario slip. Ch 17, 24, 25, 26 and 35 all say Acme *already* hosts about 100 million chunks, but here Acme only one day "grows" to 10 million. Quote: "So let's plan for the day Acme's knowledge-base product grows to **10 million passages**". Fix: "Acme's hosted product already holds about 100 million chunks. Let's price a 10-million-passage slice of it (multiply by ten for the whole thing)", or keep 10M without the word "grows".
- **[NIT]** The build-time row contradicts the chapter's own throughput. At the stated 100–300 ms per 1,000 passages, encoding 10M passages is 10,000 × 0.1–0.3 s ≈ 17–50 min, not ~2 h. ColBERT uses the same BERT-base encoder, so its encode pass should not take twice as long. Quote: "~2 h (encode) + 1 h (HNSW) | ~4 h (encode) + 2 h (cluster + encode)". Fix: "~0.5–1 h (encode)" for both (plus I/O), or say where the extra time goes.
- **[NIT]** The acronym PLAID is never expanded. Quote: "**PLAID = a search engine for ColBERTv2". Fix: add "(Performance-optimized Late Interaction Driver)".
- **[NIT]** Phase 1 Step 4 learns bucket edges per dimension. As I recall the `colbert-ai` code, it computes one set of quantile cutoffs and bucket weights over all residual values and shares it across dimensions. Per-dimension is a fine variant (the code here does it), but not what ColBERTv2 ships. Quote: "For each of the 128 dimensions, split the residual values into 2 equal-sized buckets". Fix: add "(ColBERTv2 itself shares one set of edges across all dimensions)".
- Verified: the arithmetic. 10M × 200 × 128 × 4 = 1.024 TB. 10M × 3,072 = 30.7 GB. 6,400 × 10M = 64 billion. The tiny residual example is right (r = [0.02, −0.04, 0.05, 0.04], bits 1 0 1 1, decoded [0.53, 0.32, −0.42, 0.68], max error 0.05 → 0.02). 128 × 2 bits = 32 B + 4 B = 36 B, and 512/36 = 14.2×. 1-bit gives 20 B = 25.6×. 200 × 36 = 7,200 B. 10M passages come to 72 GB (2-bit) and 40 GB (1-bit). 32 × 2^18 = 8.4 M dot products. 2 × 110M × 200,000 tokens = 44 TFLOP, and 4.4 TFLOP for 100. "About 2× storage" (40–72 vs 30 GB) and "5–10× latency" (50–150 vs 10–20 ms) are fair. PLAID's "up to 7× GPU / 45× CPU" matches the paper's abstract as I recall it, and so does the ~2× token-pooling result (Answer.AI 2024). Cross-references check out (Ch 7 inverted index, 18 cascade, 19, 23–26, 34–36, 38, 41). Ch 41's "+10–30 ms to encode on the fly" for 100 docs agrees.
- Style: the landmark analogy has its mapping, Phase/Step lines come before the code, the example has wrong (Way 2) and right (Way 3), and there are no semicolons or em-dashes in the body. No violations.
- FIXES.md items for this chapter: all applied. "~10 ms" is gone (grep finds no "~10 ms"). It is now "100–300 ms … 10–30 ms for 100" in four places, with the TFLOP derivation. `self.n_centroids` is stored and `self.bin_centres` comes from centre quantiles. "Or smaller than" was dropped. "+3 to +8" is qualified in the table, the prose and Key takeaways. The inverted index now points to Chapter 7, and PyLate is named.
- Code: 2 blocks, both run. I ran `ResidualCompressor` on 20,000 clustered unit 128-d vectors (256 centroids) at nbits = 1 and 2. Packed codes are exactly 16 and 32 bytes, plus 4 B for the uint32 ID, which gives 20 and 36 B as stated. The pack/unpack round-trip is exact. Decoding cuts mean reconstruction error from 0.49 (centroid only) to 0.30 (1-bit) and 0.19 (2-bit). MaxSim on decoded vectors is within 0.1% of exact. `plaid_candidates` runs and returns the source document among its candidates. Its logic differs from PLAID as noted above.

#### Fix status (applied 2026-09-16)

- [SHOULD] PLAID centroid pruning in wrong pipeline stage — fixed differently: kept 5 steps (score centroids, each query token's nprobe nearest centroids, inverted-index candidates, centroid-only approximate MaxSim with centroid pruning at ~0.4–0.5, decompress + exact MaxSim); rewrote the centroid-pruning definition, flow diagram, Way 3, "What people get wrong" (sweep nprobe and threshold), Key takeaway, one-paragraph version and frontmatter summary to match. Omitted PLAID's second no-pruning re-score stage, per the brief's instruction to stay at what the paper clearly supports.
- [SHOULD] Acme "grows" to 10M passages (D10) — fixed: Acme already holds ~100M chunks (Chapter 17), chapter prices a 10M-passage slice, multiply storage by ten for all of Acme.
- [NIT] Build-time row contradicts 100–300 ms per 1,000 passages — fixed: encode is ~0.5–1 h for both columns, ColBERT adds ~2 h (cluster + compress), with a note tying encode time to the same throughput.
- [NIT] PLAID acronym never expanded — fixed
- [NIT] Per-dimension bucket edges vs ColBERTv2's shared edges — fixed
- Code: `plaid_candidates` rewritten (per-token top-nprobe centroids, inverted-index lookup, pruned centroid-only MaxSim over candidates, returns top-ndocs shortlist). Both python blocks extracted and run with the venv on toy data (3,000 docs, 148k clustered 128-d tokens, 512 centroids, 2-bit): packed codes 32 B/token, target doc in shortlist 50/50 and top-1 after decode + exact MaxSim 50/50, recall@10 0.84 vs brute-force exact MaxSim at ndocs=100, all-pruned edge case runs.
- Cross-chapter follow-ups: none required (Ch 20, 38, 65, 66 describe PLAID only generically as centroid-based pruning/inverted index, still accurate). Ch 36 wording aligned (own file).

## Chapter 38 — MUVERA: Multi-Vector Made Single  (reviewer: chapters 35–39)

- **[MUST]** The FDE score is not "about `reps` times Chamfer", and the scale factor is not constant, so it *can* change the ranking. With coarse regions (B = 16) each region averages many unrelated page tokens, so each repetition's dot product falls well below Chamfer, and the shortfall grows with page length. I ran the chapter's `FDE` class (k_sim = 4, d_proj = 16, reps = 10) on clustered unit 128-d toy data with 32-token queries. The mean FDE/Chamfer ratio was 7.8 for 4-token pages, 5.8 for 16, 2.7 for 64 and 1.2 for 200. With few-topic or anisotropic pages (mean cosine 0.27–0.50) it was 3.5–4.6 at 16 tokens and 2.3–3.8 at 200. It is never ≈ 10 at realistic lengths, and long pages are systematically scored lower. Quotes: "# ≈ reps × Chamfer(Q, D); same ranking" and "the raw dot product is about `reps` times Chamfer. The constant does not change the ranking." Fix: "Each repetition gives a rough, usually low estimate of Chamfer. The estimate drops as more page tokens share a region, which happens on longer pages. So the raw FDE score is a ranking *proxy*, not a scaled Chamfer value, and exact MaxSim reranking is what restores the true order." Also soften the main text's "gives nearly the same score as all of ColBERT's token-by-token matching" to "gives a score that ranks pages in roughly the same order".
- **[SHOULD]** The chapter's own parameters give 2,560-d, not the ~4,000-d FDE used throughout the book (Ch 47, 61, 66 and this chapter's pipeline diagram), and no configuration that reaches ~4,000 is ever shown. Quote: "With $B = 16$, $d_{proj} = 16$ and $R_{reps} = 10$ it is 2,560. Around 4,000 dimensions is typical in practice." Fix: add "Raising $R_{reps}$ to 16 gives $16 \times 16 \times 16 = 4{,}096$, the ~4,000-d FDE used in later chapters". Optionally use `reps=16` in the code. (As I recall, the paper's headline experiments use k_sim = 5, d_proj = 16, R_reps = 20, that is 10,240-d.)
- **[SHOULD]** Wrong link to Ch 21. OR-amplification (Ch 21: "matching at least one of L tables") takes a *union*, so one hit is enough. FDE repetitions are concatenated, so their dot products are *summed* (averaged), and a boundary miss in one repetition is diluted rather than rescued. Quote: "This is Chapter 21's OR-amplification once more." Fix: "This is averaging independent estimates. It resembles Chapter 21's many tables, but here the repetitions' scores are added, not unioned."
- Verified: the whole hand example, in numpy. The averages are [0.405, 0.575], [0.398, 0.07] and [−0.058, 0.673]. The dot products are 0.20 and 0.36, and the cosines 0.71 and 0.77. The regions are NE [0.72, 0.605], SE [0.32, −0.93], overview NE [0.675, 0.74] and NW [−0.79, 0.605]. FDE scores are 0.81 + 0.995 = 1.80 and 0.79 + 0.73 = 1.52 (1.51 unrounded), and exact Chamfer is 1.974 and 1.633. All match. Every token's quadrant is right. The B = 8 diagram slots line up. These also match the paper as I recall it: the acronym expansion; sum for queries and average for documents; empty-region fill by nearest Hamming SimHash; the Gaussian/√d_proj inner projection, which is unbiased like the paper's ±1/√d_proj; and the fixed guarantee (±ε on query-normalised Chamfer, with dimension (m/δ)^O(1/ε), polynomial in m). The claims of 10% higher recall and 90% lower latency than PLAID, 2–5× fewer candidates than the single-vector heuristic, and 32× PQ match the abstract as I recall it. Cross-references check out (Ch 4 MIPS, 6 JL, 21 SimHash/hyperplane, 24 PQ, 26 Hamming, 34, 36, 37, 45, 47).
- Style: the post-office analogy is mapped, Phase/Step lines come before the code, the wrong (averaging) and right (FDE) answers are both shown, and there are no semicolons or em-dashes in the body. No violations.
- FIXES.md items for this chapter: all applied (guarantee restated as polynomial-in-m dimension, ±ε on query-length-normalised Chamfer, with the "fixed size is not proven" caveat repeated in What people get wrong and Key takeaways; "2–5× fewer candidates" now explicitly against the single-vector heuristic, with PLAID compared separately).
- Code: 1 block, runs clean. I tested the correlation on 500 toy pages (50–300 tokens drawn from a few of 300 topics) and 30 queries. At 2,560-d the FDE-vs-Chamfer correlation was Pearson 0.64, Spearman 0.47, and FDE top-100 held 94% of the true Chamfer top-10. At 4,096-d (reps = 16) it was 0.69 / 0.51 / 95%, and at 10,240-d (k_sim = 5, reps = 20) 0.75 / 0.56 / 98%. So the FDE is a good shortlister but only a loose score approximator, which supports the MUST above. Shapes, the region hashing (`bool @ pow2`), empty-region fill and the output length B·d_proj·reps are all correct.

#### Fix status (applied 2026-09-16)

- [MUST] FDE score is not reps × Chamfer, ranking can change — fixed: code comment now "a ranking proxy for Chamfer(Q, D)", Under-the-hood prose explains the low, length-dependent estimate (ratio fell about six-fold from 4- to 200-token pages on toy data) and that exact MaxSim reranking restores the order, "nearly the same score" softened to "ranks pages in roughly the same order", Key takeaway added, and the guarantee section's "works well regardless of document length" narrowed to "shortlists well … as long as exact MaxSim reranks".
- [SHOULD] 2,560-d example never reaches the ~4,000-d used later — fixed: prose shows R_reps = 16 gives 16 × 16 × 16 = 4,096, the ~4,000-d FDE of later chapters; code default changed to reps=16 and its sentence now says 4,096; Key takeaway shows 4,096.
- [SHOULD] Wrong OR-amplification analogy to Ch 21 — fixed
- [NIT] Section 0: prose semicolon — fixed differently: the only `;` was LaTeX spacing `\;\approx\;` in the FDE formula, now plain `\approx` (the code-comment semicolon also went with the MUST fix).
- Code: FDE block changed (reps default 10 → 16, comment). Extracted and run with the venv: output length 4,096; FDE/Chamfer ratio on clustered unit 128-d toy data with 32-token queries was 7.9 / 5.1 / 2.3 / 1.2 for 4 / 16 / 64 / 200-token pages (reps=10 gave 4.7 → 0.75, same ~6× fall), confirming the MUST and the "about six-fold" sentence.
- Cross-chapter follow-ups: none required. Ch 47:144 ("dot product approximates MaxSim") and Ch 61:122 ("imitates MaxSim") are loose but not wrong, and both chapters already rerank with exact MaxSim.

## Chapter 39 — SPLADE and Learned Sparse Retrieval  (reviewer: chapters 35–39)

- **[MUST]** The saturation explanation contradicts the formula printed just above it. $\log(1+x)$ is applied *per position, before pooling*, so it compresses how strongly one position votes for a word, not how many times the word is voted for. Under the v1 sum, 20 positions with logit 1 give 20 × 0.693 = 13.9 and 2 positions give 1.39. That is still exactly 10×, so the frequent word *does* drown out the rare one. Under the max pooling that the code and checkpoints use, the count has no effect at all. So it is not BM25's k1 (Ch 8 defines k1 as flattening the *count*). Quotes: "so a word the model thinks of twenty times does not drown out a word it thinks of twice. That is BM25's $k_1$ saturation (Chapter 8), rediscovered as a differentiable function." and the Key takeaway "$\log(1+x)$ saturation is BM25's $k_1$ rediscovered". Fix: "so one position that votes very strongly for a word cannot drown out everything else (a raw score of 20 becomes 3.0, a score of 2 becomes 1.1). It is a cousin of BM25's k1 saturation. Both flatten large values, but BM25 flattens how *often* a word appears, and SPLADE flattens how *strongly* the model votes for it."
- **[SHOULD]** The BGE-M3 "shortcut" is offered in a SPLADE chapter without saying that its sparse output does no expansion. As I recall the BGE-M3 paper, it scores only the tokens actually present (a ReLU weight per input token, matched on shared terms), so it will not rescue the paraphrase twin the way SPLADE does. Quote: "**BGE-M3 is the pragmatic shortcut.** It emits *dense, sparse and multi-vector* representations". Fix: add "Its sparse output weights only words that appear in the text. It does not expand, so its dense side still has to catch paraphrases."
- **[NIT]** The example table lists `saml` as one term, but in bert-base-uncased "SAML" is `sam` + `##l`. The Note covers readable words in general, yet it only names `ss ##o` and `log ##in`. Quote: "| saml | 0.6 | **no, expanded** |". Fix: optional, add "and 'SAML' is `sam` + `##l`" to the Note.
- Verified: vocabulary size 30,522, counted from the local bert-base-uncased vocab.txt. The tokenizations `log ##in`, `ss ##o - 401 ##2` and `ss ##o - 402 ##1` are right, and "authentication" is a single piece. The paraphrase question shares no words with page 212, so BM25 really scores zero. The scores check out: page 212 = 1.98 + 0.91 + 1.26 + 1.82 + 0.72 + 0.72 = 7.41, and password-reset = 3.42 + 0.65 + 1.00 = 5.07. The SPLADE acronym is right. The v1 formula is Σ log(1 + ReLU(w_ij)), with max pooling in v2+ and released checkpoints including `naver/splade-v3`. FLOPS is correctly described as expected posting-list cost with separate λ_q and λ_d, and the code's `(mean |activation| per term)² summed` matches Paris et al.'s relaxation. "~100–300 non-zero" agrees with the Ninja table's "~200". Cross-references check out (Ch 7 posting lists, 8 k1 / car–automobile / WAND, 10, 11, 33, 40), and What's next links 40-hybrid-search.md.
- Style: the index-card analogy is mapped, the chapter has Phase/Step lines and wrong-then-right twins, and there are no semicolons, with em-dashes only in the frontmatter. No violations.
- FIXES.md items for this chapter: all applied ("defeat much of that pruning … expect 2–10× BM25's latency unless … Efficient-SPLADE" in the inverted-index section, and repeated in Problem 1, the table, What people get wrong and Key takeaways; the one-paragraph version now says "We do not get quite BM25's speed"; the checkpoint is `naver/splade-v3`). The 2–10× figure itself is unverified.
- Code: 1 PyTorch/transformers block, not runnable here. It reads clean: logits (B,T,V) → `log1p(relu)` → mask → `max(dim=1)` gives (B,V), and `nonzero(vec).squeeze(-1)` on a (V,) vector gives the indices. `convert_ids_to_tokens(int)` returns a str, and the FLOPS function is correct for (B,V). Nothing implements Phase 2 scoring, and the prose does not claim anything does.

#### Fix status (applied 2026-09-16)

- [MUST] log(1+x) is per-position vote strength, not BM25 k1 — fixed: body now says one strongly voting position cannot drown out everything else (20 → 3.0, 2 → 1.1, checked with log1p), calls it a cousin of k1 (BM25 flattens how often, SPLADE how strongly); dropped "both borrowed from decades of search practice"; Key takeaway rewritten to match.
- [SHOULD] BGE-M3 sparse output does no expansion — fixed (Ninja note, plus a matching clause on the BGE-M3 Key takeaway).
- [NIT] `saml` is `sam` + `##l` in bert-base-uncased — fixed (confirmed against the local bert-base-uncased vocab.txt: no "saml" entry, "sam" and "##l" present).
- Code: no code changed
- Cross-chapter follow-ups: none (no other chapter repeats the k1 claim. Ch 16's BGE-M3 note says "keyword-style" and makes no expansion claim.)

## Chapter 40 — Hybrid Search and Fusion  (reviewer: chapters 40–41, 43–44)

- No MUST issues found. All RRF arithmetic recomputed with k = 60 and correct: SSO-4012 table (1/61 + 1/64 = 0.0320, 2/63 = 0.0317, 1/65 + 1/62 = 0.0315, 1/64 + 1/65 = 0.0310, 1/61 = 0.0164, 1/62 = 0.0161; the order and "fell to fifth" are right); the paraphrase-twin fusion (password reset 0.0320, account settings 0.0315, page 212 0.0164 as the top 3); 1/61 = 0.0164 vs 1/62 = 0.0161; 2/65 ≈ 0.031 beats 1/61; k = 0 gives 1.0 vs 0.5. Min-max definition and the convex-combination formula agree with the code (α on dense, 1 − α on BM25). Cross-references checked: Ch 3 (the 0.55–0.95 cosine band), Ch 8 (lower b for uniform chunks), Ch 13 ("spend on recall early and on precision late"), Ch 19 (golden set), Ch 21 (MinHash), Ch 52, Ch 55 (threshold calibration). Bruch, Gai & Ingber checked against arXiv 2210.11934: tuned CC beats RRF, needs few training queries, and RRF is sensitive to its parameters.
- **[SHOULD]** The table contradicts the paper it cites. Bruch, Gai & Ingber's abstract says "the learning of a CC fusion is generally agnostic to the choice of score normalization", but the chapter lists normalisation choice as something CC is sensitive to. Quote: "| Sensitive to | $k$ | Normalisation choice and $\alpha$ |". Fix: "| Sensitive to | $k$ | Mainly $\alpha$ (the normalisation choice matters little once $\alpha$ is tuned) |". This also resolves the tension with the earlier "Normalising makes them look comparable when they are not": that warning applies to *untuned* mixing.
- **[SHOULD]** The paraphrase-twin example calls an incomplete answer "correct", which goes against the book's running scenario. After fusion, page 1,140 scores 1/63 = 0.0159 and ties with team management at rank 5–6, so it misses the top 3. The Scholar reads page 212 alone and gives exactly the answer the book calls wrong elsewhere: SSO on Pro with no add-on caveat for teams under 50 seats. Quote: "Page 212 is in the top 3, so the Scholar reads it. The answer is correct." Fix: "Page 212 is now in the top 3, which is progress. But page 1,140 (0.0159) is still just outside it, so the Scholar misses the Security add-on caveat. Getting that page in is why we fuse 100 deep and then rerank (Chapter 41)." Alternatively, keep the claim and change the question to one that page 212 fully answers.
- **[NIT]** The consensus claim is stated as universal but holds only for fairly high ranks. With k = 60, a page at rank r in both lists beats a rank-1-only page only when 2/(60 + r) > 1/61, that is r ≤ 61. At the recommended depth of 100, a page at rank 70 in both lists (0.0154) loses to it (0.0164). Quote: "The effect is that **appearing on both lists matters more than being first on either.**" Fix: add "as long as both ranks are reasonably high (above about 60 when k = 60)".
- **[NIT]** `convex_fusion` gives a page missing from one list the value 0.0 on that side. That is the same score as the *lowest* hit that did make the list, so absence and last place are treated the same. Quote: "alpha * v.get(d, 0.0) + (1 - alpha) * b.get(d, 0.0)". Fix: one comment line, e.g. "# missing = treated like the worst retrieved hit; Bruch et al. use the theoretical minimum instead".
- Style: no notable violations (no semicolons or em-dashes in prose, "you" only in advice, analogy has its mapping, wrong-then-right answers shown, bullets match the headings exactly).
- FIXES.md items for this chapter: all applied ("best *untuned* default" in the one-paragraph version; the Bruch et al. comparison and "sensitive to k" appear in the body, the table, What people get wrong and Key takeaways).
- Code: 3 Python blocks plus the one-line "broken" snippet. `rrf` and `HybridRetriever` (with stub retrievers) ran on the chapter's SSO-4012 lists and returned SSO-4012, SSO-4011, SSO-4102, SAML, SSO-4021, overview, exactly as the prose says. `convex_fusion` runs clean on toy data.

#### Fix status (applied 2026-09-16)

- [SHOULD] Table says CC sensitive to normalisation, contradicting Bruch et al. — fixed (table row now "Mainly α (normalisation matters little once α is tuned)"; one matching sentence added to the Bruch paragraph)
- [SHOULD] Paraphrase-twin example calls an answer without page 1,140 "correct" — fixed differently: kept the lists and recomputed scores (page 1,140 = 1/63 ≈ 0.0159, tied fifth); text now says page 212 alone gets in, page 1,140 was third on dense but pushed down by two consensus pages, so the Scholar misses the add-on caveat and the answer is still wrong; Note says fuse 100 deep, then rerank (Ch 41)
- [NIT] Consensus claim holds only for ranks up to ~61 — fixed ("as long as the page ranks in about the top 60 of both")
- [NIT] convex_fusion treats missing like lowest hit — fixed differently: added a factual comment ("A page missing from a list gets 0.0 there, the same as that list's lowest hit") without the unverified attribution to Bruch et al.
- D3: frontmatter summary now "the cheapest, most reliable recall win in this book"; "highest-value thing in this book per hour of work" now "the cheapest and most reliable recall win in this book"
- Code: convex_fusion (comment line only) extracted and run on toy data with the venv; fusion scores for the paraphrase twin recomputed in Python (0.0320, 0.0315, 0.0164, 0.0161, 0.0159, 0.0159); 2/121 > 1/61 and 2/122 = 1/61 checked
- Cross-chapter follow-ups: OUTLINE.md Ch 40 one-liner "the most reliable free win in retrieval" is consistent with D3, no change needed
- [Lead follow-up from Ch 03 fixer] "usually crammed into about [0.55, 0.95]" — fixed to "often", matching Ch 3's narrowed claim.

## Chapter 41 — Rerankers  (reviewer: chapters 40–41, 43–44)

- No MUST issues found. Verified: the first-stage table and the toy reranker scores (9.1 / 8.4 / 5.2 / 3.9 / 3.1, old ranks 1 / 23 / 4 / 3 / 2) match Chapter 13 exactly, as the text claims. The depth table's latency is linear at 0.6 ms per candidate from 50 to 500, with a fixed overhead at 10, and 500 candidates cost 5× as much as 100. The recall@100 ceiling argument is right, since recall@10 after reranking can never exceed recall@100 before it. The ColBERT on-the-fly figure "10–30 ms for 100 passages" matches Chapter 37 ("For 100 passages it is about 10–30 ms"). Pointwise, pairwise and listwise are defined correctly. RankGPT's sliding window checked against arXiv 2304.09542 (window 20, step 10, back to front). Cross-references to Ch 13, 36, 37, 40, 53 and 55 are on topic. Sigmoid (Ch 9), logits (Ch 12), nDCG (Ch 19) and BERT (Ch 10) are all defined earlier. The pipeline is consistent with Ch 40 (top 100 in, top 5–10 out).
- **[SHOULD]** Chapters 40 and 41 each claim to be the book's biggest win, and Ch 40's own numbers favour fusion. Ch 40's table puts fusion at +11 recall@10 (0.68 → 0.79) and reranking at +7 (0.79 → 0.86). Ch 40 also calls fusion "the most reliable quality win available anywhere in this book" and "the highest-value thing in this book per hour of work". Quotes: "Adding one is typically the single largest quality improvement available to a working RAG system", "pound for pound the most valuable component in the pipeline" (frontmatter summary), "Nothing else we can bolt onto a working pipeline reliably does that." Fix: split the claims explicitly. Fusion is the cheapest win and fixes recall. The reranker is the largest *precision* win once retrieval is in place. For example: "Once retrieval is decent (ideally hybrid, Chapter 40), adding a reranker is typically the largest remaining quality improvement." Change the summary to "the highest-precision component in the pipeline", which is what Ch 40's "What's next" already says.
- **[SHOULD]** The listwise sketch contradicts both the prose above it and the chapter's own warning about depth. The prose says RankGPT "slides a window over the candidate list". The code sends only `candidates[:window]` (20) in one call, never slides, and returns only those 20. Fed the chapter's 100 candidates, it never reads page 1,140 at rank 23 and drops it from the output (confirmed with a mock `llm`). That is exactly the "Reranking too few candidates" failure described later. Quote: "for i, c in enumerate(candidates[:window])". Fix: either loop the window from the bottom of the list to the top with step 10 (RankGPT's setting), or rename the function `llm_rerank_top_window`, return `reordered + candidates[window:]`, and add a comment: "one window only; slide it (RankGPT: 20 wide, step 10, back to front) to cover 100". The phrase "reordering a few passages at a time" is also loose, since RankGPT's window holds 20.
- **[NIT]** Problem 1 understates its own table. The table goes from +9 at 100 candidates to +10.5 at 500, a gain of 1.5 points. Quote: "500 candidates cost five times as much as 100, for a gain of about one extra point." Fix: "for about one and a half extra points".
- **[NIT]** The code uses a larger model than the latency figures describe. `BAAI/bge-reranker-v2-m3` is built on BGE-M3 (XLM-RoBERTa-large, about 568M parameters), not a BERT-sized model. Per pair it needs roughly 3–4× the compute of BERT-base, since the non-embedding weights are about 300M vs about 86M. So the "20–100 ms" and "60 ms at 100" figures do not describe the model in the code. Quote: `CrossEncoder("BAAI/bge-reranker-v2-m3", max_length=512)`. Fix: either use a base-size reranker (e.g. `BAAI/bge-reranker-base`) or add "(a larger multilingual model, expect several times the latency in the depth table)".
- **[NIT]** The MMR formula leaves out the candidate set, so as written it could pick an already-selected page again. Quote: "\text{MMR} = \arg\max_{d} \left[". Fix: `\arg\max_{d \in R \setminus S}`, with R = the reranked candidates. The code does restrict to `rest`, so only the formula needs the change.
- Style: no notable violations (no semicolons or em-dashes in prose, "you" only in advice and Ninja notes, hiring analogy mapped, wrong answer then right answer shown, Phase/Step lines before the code, bullets match headings).
- FIXES.md items for this chapter: all applied (OUTLINE now reads "The single largest quality win in a working RAG stack", and the body supports it with "+5–15 nDCG@10". The Chapter 55 reference points at a real threshold-calibration section). The claim itself is disputed by Ch 40, see the SHOULD above.
- Code: 3 blocks. `rerank` ran with a mocked `CrossEncoder` and correctly returns the top 10 by score, highest first. `llm_rerank` ran with a mocked `llm` and is syntactically fine, but it only covers 20 candidates (see above). `mmr` ran on toy vectors (5 near-duplicates plus 2 distinct pages) and picked one duplicate plus the two distinct pages, where plain top-3 picked three duplicates. It does what the prose says.

#### Fix status (applied 2026-09-16)

- [SHOULD] Ch 40 and Ch 41 both claim the biggest win — fixed per D3: body "Once retrieval is decent (ideally hybrid, Chapter 40), adding a reranker is typically the largest remaining quality improvement", summary "the highest-precision component in the pipeline", Key takeaway matched, "Nothing else ... does that" narrowed to precision, "What people get wrong" now "largest remaining win" for decent retrieval
- [SHOULD] Listwise sketch never slides its window — fixed: split into `rank_window` (validates indices, keeps forgotten passages) and `llm_rerank` that slides a 20-wide window back to front with step 10; prose now says window 20, step 10, bottom up; added an "In simple words" paragraph (9 calls for 100, page 1,140 first read in call 7); caution paragraph now points at the validation in the code
- [NIT] Problem 1 understates the 100→500 gain — fixed ("about one and a half extra points")
- [NIT] bge-reranker-v2-m3 is larger than the latency figures — fixed: switched to `BAAI/bge-reranker-base` (BERT-base sized, same model as Ch 13) with a comment
- [NIT] MMR formula omits the candidate set — fixed (`\arg\max_{d \in R \setminus S}`, sentence defines R)
- Code: `llm_rerank` extracted and run with a mock `llm` on 100 candidates (page 1,140 at rank 23): 9 calls, output 100 unique, page 1,140 read in calls 7–9 and ends at rank 2; also tested a malformed/partial LLM reply (no loss or duplicates), 95, 12 and 0 candidates. `rerank` run with a mocked CrossEncoder. `mmr` unchanged
- Cross-chapter follow-ups: OUTLINE.md Ch 41 one-liner should become "The largest precision win once retrieval works." (D3, owned by README/OUTLINE fixer)
- [Lead follow-up from Ch 12 fixer] "far more accurate than any bi-encoder" — fixed to "usually far more accurate than a bi-encoder of similar size". Opening paragraph and the MMR paragraph rewrapped.

## Chapter 42 — Multimodal Embeddings: CLIP and the Shared Space  (reviewer: chapters 42–48)

- No MUST or SHOULD issues found. Facts checked and correct: 400M pairs, batch N = 32,768, ViT-B-32 at 224 px → 7 × 7 = 49 patches, ViT-L-14 → 16 × 16 = 256, shared width 512 for ViT-B-32, learned temperature init 0.07 capped at a 100× logit scale, zero-shot ImageNet ≈ ResNet-50 trained on 1.28M images, 80 prompt templates, 77-token text limit, modality gap as offset "cones".
- **[NIT]** "ResNet-50" is used without a gloss. Quote: "zero-shot CLIP matched the accuracy of a classic ResNet-50 classifier". Fix: add "(an older, widely used image classifier network)". No earlier chapter mentions ResNet.
- **[NIT]** Frontmatter `readingTime: "13 min"` vs ~3,280 words ≈ 14 min at 230 wpm. Harmless.
- Style: no notable violations (bullets match headings; "you" confined to advice; no semicolons in teaching sections).
- FIXES.md items for this chapter: all applied (ViT expanded as "Vision Transformer" with patches defined, and given its own section).
- Code: 2 blocks (torch `clip_loss`, `open_clip` search), both read clean; `create_model_and_transforms("ViT-B-32", pretrained="laion2b_s34b_b79k")` and `get_tokenizer` are the real open_clip API, shapes are consistent ((N,N) logits, `arange` labels, (10000,) score vector → `topk(10)`).

#### Fix status (applied 2026-09-16)

- [NIT] ResNet-50 used without a gloss — fixed (added "(an older, widely used image classifier network)")
- [NIT] readingTime 13 min vs ~14 min — skipped: readingTime NITs are superseded by the brief (rule 5)
- D11: Ch 42 temperature text left unchanged, as the brief requires (the note lives in Ch 43)
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 43 — SigLIP and Modern Vision-Language Encoders  (reviewer: chapters 40–41, 43–44)

- Verified correct. Worked example: softmax at t = 10 gives 0.971 (Batch A) and 0.541 (Batch B); σ(1.2) = 0.77, σ(1.0) = 0.73, σ(−3) = 0.05, σ(−3.5) = 0.03, σ(−4) = 0.02; σ(0) = 0.5, σ(±3) ≈ 0.95 / 0.05. Patches: 384 // 14 = 27 → 729, 448 // 14 = 32 → 1,024, 896 → 4,096 (4×). A4 squeezed to 224 px gives about 0.75 px/mm, so 10 pt text is about 2–3 px tall. Checked against the SigLIP paper (arXiv 2303.15343): the loss formula and 1/N normalisation match Algorithm 1; b initialised to −10; sigmoid well ahead of softmax below 16k (SigLiT) or 32k (SigLIP); "performance for both sigmoid and softmax saturate at around 32 k"; the chunked, no-all-gather implementation passes text vectors between devices. SigLIP 2 (arXiv 2502.14786, 20 Feb 2025) abstract confirms multilingual data, captioning-based pretraining, self-distillation / masked prediction, and a native-aspect-ratio multi-resolution variant. The decoder being training-only is not in the abstract, and the text already hedges it. So400m inside PaliGemma-3B with Gemma-2B, and PaliGemma 2 on Gemma 2 in late 2024, are consistent with Ch 46–47. Ch 47 does contain the resolution numbers promised three times.
- **[MUST]** The explanation of the learned bias is backwards and contradicts itself. The chapter says the imbalance would make the model "answer 'no' to everything", then says the fix is to start with "no" as the default. The paper's reason is different: "the heavy imbalance coming from the many negatives dominates the loss, leading to large initial optimization steps attempting to correct this bias… This makes sure the training starts roughly close to the prior and does not require massive over-correction" (§3.2, confirmed by the Table 4 ablation). Quote: "Without help, the model would learn to answer \"no\" to everything. So the bias starts strongly negative (−10 in the paper), which makes \"no\" the starting guess, and the model learns from there." Fix: "If every cell started at a 50/50 guess, the N² − N 'no' cells would swamp the loss, and the first training steps would be huge corrections just to push them all down. So the bias starts at −10 (and t at 10), which makes 'no' the starting guess, since almost every cell *is* a no. Training then spends its effort pulling the few 'yes' cells up."
- **[SHOULD]** "Temperature" now works in the opposite direction from Chapter 42, and the chapter does not say so. In Ch 42 (and its code, `logits = I @ T.T / temperature`) the temperature is a divisor starting at 0.07. Here it is a *multiplier* t, and the toy example uses t = 10. A novice reading both chapters will think the two contradict each other. Quote: "multiply the dot product by a learned temperature t". Fix: "multiply the dot product by a learned scale t (the SigLIP paper calls it the temperature. It is the *inverse* of Chapter 42's 0.07-style temperature, so multiplying by t = 10 is the same as dividing by 0.1). The paper starts t at 10." This also covers the paper's t initialisation, which the chapter never gives (t = exp(t′), t′ = log 10).
- **[NIT]** The one-paragraph version names the wrong culprit. SigLIP *also* compares every image against every caption (Step 2 builds the full N × N grid). What it removes is the batch-wide normalisation, not the comparisons. Quote: "CLIP's loss compares every image against every caption in the batch, which forces enormous batches". Fix: "CLIP's loss normalises every image's scores over all the captions in the batch, which…".
- **[NIT]** Compute grows faster than pixel count. Doubling the side length gives 4× the visual tokens, so 4× storage and 4× MaxSim cost is right. But the ViT's self-attention inside the encoder grows 16×, so encoding compute rises *at least* 4×. Quote: "doubling the resolution in both directions quadruples compute and storage". Fix: "quadruples storage and scoring cost, and at least quadruples encoding compute".
- **[NIT]** The code uses an undefined placeholder. Quote: "images=[office_dog_photo]". Fix: add `from PIL import Image` and `office_dog_photo = Image.open("office_dog.jpg")`, as Chapter 42's code does with `Image.open(path)`.
- Style: no notable violations (no semicolons or em-dashes in prose, "you" only in advice, exam analogy mapped, Phase/Step lines before the formula, bullets match headings).
- FIXES.md items for this chapter: all applied (sigmoid σ glossed in the one-paragraph version; VLM expanded at first bare use; Gemma glossed; the "Chapter 47 puts numbers on it" promises now hold; SigLIP 2 / PaliGemma 2 paragraph added).
- Code: 1 block (transformers, cannot run here). It reads clean apart from the placeholder: `google/siglip-so400m-patch14-384`, `padding="max_length"`, `out.image_embeds` / `text_embeds` / `logits_per_image`, `torch.sigmoid` on the logits, and `model.vision_model(...).last_hidden_state` with shape (1, 729, 1152) are all correct for the HF SigLIP API (no CLS token, so 27² exactly).

#### Fix status (applied 2026-09-16)

- [MUST] SigLIP bias explanation backwards — fixed: the Note now says a 50/50 start would let the N² − N "no" cells swamp the loss and force huge early corrections, so b starts at −10 (near the prior) and training pulls the few "yes" cells up (paper §3.2)
- [SHOULD] Temperature multiplies here but divides in Ch 42 — fixed (D11): added "The paper starts t at 10" plus a Note that SigLIP's t multiplies and is the inverse of Ch 42's dividing temperature (t = 10 = dividing by 0.1); Ch 42 unchanged
- [NIT] One-paragraph version names the wrong culprit — fixed ("normalises every image's scores over all the captions in the batch")
- [NIT] Encoding compute grows faster than pixel count — fixed ("quadruples storage and scoring cost, and at least quadruples encoding compute")
- [NIT] Undefined `office_dog_photo` placeholder — fixed (added `from PIL import Image` and `office_dog_photo = Image.open("office_dog.jpg")`)
- Code: transformers block changed (two added lines), checked by reading (needs transformers). Toy softmax numbers re-run (0.971 and 0.541 unchanged)
- Cross-chapter follow-ups: none

## Chapter 44 — The Broken Promise of OCR Pipelines  (reviewer: chapters 40–41, 43–44)

- No MUST issues found. The stage count is **six** everywhere: frontmatter, one-paragraph version, "six hands", the numbered diagram (OCR, layout, tables, reading order, chunking, embedding), 0.9⁶ = 0.531 ≈ 53%, "six decent stages", the options table, "seven stages instead of six", Key takeaways, closing line. Book-wide there are no "five"/"seven"/0.9⁵/59% leftovers: Ch 45 says "Two stages instead of six" and "six-stage diagram", Ch 61 says "six-stage" and 0.9⁶ ≈ 53%. Other checks: 0.99⁵ = 0.951, so about 1 word in 20 has an error. The pricing table agrees with the running scenario: 120 Pro seats, "included (Pro, 50+ seats)" is the exact complement of page 1,140's "under 50 seats" rule, and the "Security add-on declined" checkbox fits. The flattened string is a faithful column-by-column reading of the table, including the single `S5O`. IoU is defined correctly. Cross-references to Ch 5, 8, 43 (VLM), 45–47, 49 (hallucination), 50 and 61 are on topic. Ch 47 does contain the promised hybrid page-image + OCR-BM25 design.
- **[SHOULD]** The worked example does not fully show the loss the prose claims. Reading column by column keeps each column's order, so the first chunk still contains `S5O` and `$0` in matching positions: third item, third price. So `$0` *does* reach the Scholar, next to the prices it lines up with. The Scholar's reasoning ignores it and concludes Globex "needs the Security add-on at $4 per seat". A careful novice will object that the chunk says SSO costs $0. What actually severs the fact is the chunk cut, not the flattening. Quotes: "The cut falls right after `$0`." and (Loss 3) "Flatten it, and `$0` and `included` lose their link to `SSO`, as we just saw. Even a human cannot reliably answer from the flat string." Fix: move the cut one cell earlier: "The cut falls right after `$4 / seat`… The words `$0`, `120 seats` and `included (Pro, 50+ seats)` land in the next chunk." Now chunk 1 has three items and only two prices, so the link really is gone. Soften Loss 3 to "Flatten it and cut it, and `$0` and `included` lose their link to `SSO`." Drop "Even a human cannot reliably answer from the flat string", or limit it to multi-word cells running together, as in "120 seats, annual optional included".
- **[SHOULD]** Chapter 61 describes a different page 3,507 table. Ch 44, Ch 45 (line "`SSO: $0, included (Pro, 50+ seats)`, and Globex has 120 Pro seats") and Ch 46 (patches attend "to the row label `SSO` on the same row") all use an Item | Price | Terms table with "120 seats, annual" *on page 3,507* and the OCR misread `S5O`. Ch 61 flattens a Feature | Basic | Pro | Enterprise table with the misread "SS0" (zero), matches "the Pro column header", and needs page 3,508 to learn the seat count. Quotes (Ch 61): "Feature Basic Pro Enterprise SS0 not included $0, included" and "Page 3,508 is the order form, and it says Globex bought 120 Pro seats." Fix: bring Ch 61 in line with Ch 44–46 (three chapters agree). Either reuse the Item | Price | Terms flattening and `S5O`, or say explicitly that Ch 61 shows a *different* table on page 3,507. If page 3,508's neighbour expansion must stay necessary, drop "120 seats, annual" from the Ch 44 table and put the seat count on 3,508, consistent with README's "contract on pages 3,507–3,508".
- **[NIT]** The section "What each stage loses" is organised by kind of loss, not by stage, and has no chunking loss. Yet the chunk boundary is what decided the chapter's own example. Quote: "Is page 3,507 unlucky, or is this normal? Let's go through the losses one kind at a time." Fix: add "**Loss 7: Chunk boundaries.** A fixed-length cut can separate a row label from its value, as it did on page 3,507 (Chapter 50)", or retitle the section "What the pipeline loses" and the matching bullet.
- **[NIT]** The chapter disagrees with itself on the cost of VLM transcription. The table rates its per-page indexing cost "Highest" and the Ninja note says a VLM pass is "slower and more expensive than OCR", but the recommendation calls it cheap. Quote: "VLM transcription into your existing stack is pragmatic and cheap." Fix: "pragmatic and cheap to adopt, though the most expensive per page to index".
- **[NIT]** The audit code and its checklist disagree. Quote: `"empty": len(text.strip()) < 50,` vs "Pages producing under 100 characters that visibly contain text". Fix: use one threshold, e.g. `< 100`. Also, `random.sample(pdf_paths, sample)` raises `ValueError` when there are fewer than 20 files. Use `min(sample, len(pdf_paths))`.
- Style: no notable violations (no semicolons or em-dashes in prose, "you" only in advice, telephone analogy mapped, wrong answer shown with the Scholar's reasoning, Step lines before the audit code, bullets match headings).
- FIXES.md items for this chapter: all applied ("six hands", 0.9⁶ ≈ 53%, "seven stages instead of six" and Key takeaways all consistent. Ch 45 updated to "Two stages instead of six" / "six-stage diagram". IoU expanded as "Intersection over Union… a box-overlap score"). No new error introduced.
- Code: 1 block. `audit_extraction` ran with a stub `pandas` on four toy extractions (the flattened pricing text, whitespace, character-split text, mojibake). Its statistics flag the empty, split and garbage cases, and pass the flattened table as clean, exactly as the prose says ("They will not catch a scrambled table"). The only problems are the threshold mismatch and the small-sample error above.

#### Fix status (applied 2026-09-16)

- [SHOULD] Worked example: `$0` still reaches the Scholar in chunk 1 — fixed (D1): cut moved to right after `$4 / seat`, so `$0`, `120 seats` and `included (Pro, 50+ seats)` land in chunk 2; chunk listing, Scholar's reasoning (three items, two prices, SSO seems tied to the $4 add-on), "The words that decided the question, `$0` and *included*", Loss 3 ("Flatten it and cut it"; the "Even a human cannot..." sentence narrowed to cells running together) and the Key takeaway all updated
- [SHOULD] Ch 61 describes a different page 3,507 table — fixed differently: not in my files; Ch 44 already matches D1's canonical table (Item | Price | Terms, `S5O`, Globex heading), so nothing to change here; Ch 61 owner applies D1
- [NIT] "What each stage loses" has no chunking loss — fixed (added "Loss 7: Chunk boundaries", citing page 3,507 and Chapter 50)
- [NIT] VLM transcription called cheap despite "Highest" cost — fixed ("cheap to adopt, though the most expensive per page to index")
- [NIT] Audit code threshold 50 vs checklist 100, and sample error — fixed (`< 100`, `min(sample, len(pdf_paths))`)
- Code: `audit_extraction` extracted and run with a stub pandas on 4 toy extractions (fewer than 20 files, no ValueError); flags blank, split and garbage, passes the flattened table
- Cross-chapter follow-ups: Ch 61 must use this chapter's Stage 4 flattened text and `S5O` per D1 (already assigned). Ch 45 line 67–69 ("split it across two chunks… never saw the word *included*") stays correct with the new cut, no change needed

## Chapter 45 — ColPali I: Documents as Images  (reviewer: chapters 45–48)

- No MUST issues found. Verified: 448 / 14 = 32, 32 × 32 = 1,024 patches; ~1,030 vectors = 1,024 patches + ~6 prompt tokens (the paper's "Describe the image" tokens); 1,030 × 128 × 4 = 527,360 B ≈ 527 KB; 768-d = 3,072 B ≈ 3 KB; A4 at 448 px = 54.2 DPI across and 38.3 down (chapter: "about 54 ... and 38", and "roughly 40–55 DPI" in the code note); "Two stages instead of six" matches Ch 44's six stages. The Globex recap matches Ch 44 exactly (table rows Pro licence / Security add-on / SSO, heading "Schedule B: Pricing for Globex Corporation", `included (Pro, 50+ seats)` split into the second chunk, wrong "No ... Security add-on" answer), and 120 seats ≥ 50 gives the correct "Yes". ColPali paper (arXiv 2407.01449, final version) checked: 0.39 s/page offline indexing on an L4 vs 7.22 s/page for the PDF-parser pipeline, so "about 0.4 seconds ... against several seconds" is right; ViDoRe average nDCG@5 81.3 for ColPali vs 67.0 for the best parser+captioning baseline supports "clearly beats". Cross-references checked: patch (Ch 42), PaliGemma (Ch 43), heatmap (Ch 46 "Reading the heatmap"), neighbour-expansion fix for cross-page tables and the BM25 + RRF hybrid (both in Ch 47), Chapters 24/26/27/37/38 in Ninja notes.
- **[SHOULD]** The "Why it beats the pipeline" heading re-asserts the absolute claim the FIXES MUST removed from the intro, and then lists fine visual marks as fully kept. Quote: "**Nothing is lost in conversion, because nothing is converted.** Charts, stamps, checkboxes, signatures, highlighting, logos, handwriting and struck-through text are all in the pixels". At ~54 × 38 DPI, small stamps, checkboxes and handwriting are exactly what Problem 1 says "may not" survive. Fix: "**Nothing is lost to text conversion.** Charts, stamps, checkboxes ... stay in the pixels, subject only to the resize in Problem 1."
- **[NIT]** The code-to-steps mapping puts the resize in the wrong call. Quote: "`model(**batch)` is Steps 2 to 5." The 448 × 448 resize (Step 2) happens in `proc.process_images`. The model only cuts patches and runs Steps 3–5. Fix: "`proc.process_images` does the resize in Step 2. `model(**batch)` cuts the patches and does Steps 3 to 5."
- **[NIT]** The architecture diagram makes the linear projection look as if it adds vectors ("1,024 contextualised vectors ↓ linear projection to 128-d ~1,030 vectors"). The ~6 prompt tokens go into Gemma together with the patches. Fix: show "1,024 patch embeddings + ~6 prompt tokens" going into Gemma-2B, and "~1,030 contextualised vectors" coming out.
- **[NIT]** Scale jumps away from the running corpus. Quote: "OCR on 5 pages per query instead of 10 million pages up front." Acme's Library is 40,000 pages. Fix: "instead of all 40,000 pages (or 10 million, at scale) up front", or leave it, since this sits in What people get wrong.
- Style: clean (no semicolons, no em-dashes in prose, "you/your" only in advice, Phase/Step before code, wrong-then-right answer shown, bullets match the eight headings).
- FIXES.md items for this chapter: all applied ("Nothing is *converted*. Something is still lost ... 448 × 448" at the intro; "Two stages instead of six" and "six-stage diagram in Chapter 44"). See the SHOULD above for the one sentence that partly undoes the MUST.
- Code: 1 block (colpali_engine), cannot run here (torch). Read clean: `ColPali`/`ColPaliProcessor.from_pretrained("vidore/colpali-v1.3")`, `process_images`, `process_queries`, `score_multi_vector` are the real API; the shapes (8, ~1030, 128) and (1, ~20, 128) are right; `scores` is (1, 8), so the flattened `argmax` is a valid index into `pages[:8]`.

#### Fix status (applied 2026-09-16)

- [SHOULD] "Nothing is lost in conversion" overclaims fine visual marks — fixed ("Nothing is lost to text conversion." ... "stay in the pixels, subject only to the resize in Problem 1.")
- [NIT] Code-to-steps mapping puts resize in model call — fixed
- [NIT] Diagram makes projection look like it adds vectors — fixed (prompt tokens now enter Gemma in the diagram; Steps 4 and 5 prose moved to match)
- [NIT] "10 million pages" jumps away from the 40,000-page Library — fixed ("all 40,000 pages of the Library (or millions, at scale)")
- Code: no code changed (only the prose line after the block)
- Cross-chapter follow-ups: none (the recap "split it across two chunks. The Scholar never saw the word included" already agrees with Ch 44's new cut after `$4 / seat`)

## Chapter 46 — ColPali II: Inside the Model  (reviewer: chapters 45–48)

- No MUST issues found. Recomputed and correct: 448 / 14 = 32, 32 × 32 = 1,024; one patch on A4 = 0.258 in across × 0.365 in down ("a quarter ... a third of an inch" is fine), about 2–3 characters and ~2 lines of 10–12 pt text; 1,024 × 2,048 × 4 = 8,388,608 B ≈ 8.4 MB; 1,030 × 128 × 4 ≈ 527 KB, ratio 15.9 ("16× smaller"); 128 / 2,048 = 1/16; LoRA 2,048² = 4,194,304 vs 2 × 2,048 × 32 = 131,072, exactly 32×. "Patch 33 sits directly below patch 1" is right for a 32-wide grid. Facts checked: PaliGemma-3B = SigLIP-So400m + linear projector + Gemma-2B (hidden 2,048); ColPali v1.x builds on the 448-px checkpoint (HF `adapter_config.json` base `vidore/colpaligemma-3b-pt-448-base`); the paper's "6 extra text tokens 'Describe the image'" gives the ~1,030; image tokens come before the prompt in PaliGemma's sequence, so `sims[:1024]` really is the grid; LoRA r = 32 (paper and HF config) matches the chapter's "2,048 × 32"; SigLIP frozen; 63% / 37% academic / synthetic split with DocVQA, InfoVQA, TAT-DQA, arXivQA, synthetic queries from Claude-3 Sonnet. Cross-references checked: Ch 5, 10, 12, 19, 42, 43 (Gemma, VLM defined there), 47 (Lever 2 fewer dimensions, ColQwen2 cap), 55 (distance floor). Training wall-clock "hours": unverified (the paper gives 8 GPUs, 1 epoch, batch 32, no duration).
- **[SHOULD]** The training-set size is the first preprint's figure. Quote: "ColPali's training set has about 127,000 pairs in total." (also "about 127,000 (query, page) pairs" in the one-paragraph version and Key takeaways). arXiv 2407.01449v1 says 127,460. The final version (v6, the ICLR 2025 paper, and the released `vidore/colpali_train_set`) says "118,695 query-page pairs", with the same 63% / 37% split. Ch 45 already quotes the final version's latency (0.39 s/page, where v1 said 0.29 s/page), so the book mixes versions. Fix: "about 119,000 pairs (118,695)" in all three places.
- **[SHOULD]** ColPali's loss is not Chapter 12's InfoNCE. Quote: "**The objective** is the contrastive loss of Chapter 12, with MaxSim as the score:" and "**Step 4:** Pull the true pair's score up, push the others down". The paper's loss is L = mean softplus(s⁻ − s⁺), where s⁻ is the *highest-scoring* wrong page in the batch. That is a two-way softmax against the hardest in-batch negative (ColBERTv1-style, Ch 36), with no softmax over all N pages and no temperature. Fix: "**Step 3:** For each query, find the highest-scoring wrong page in the batch. **Step 4:** Push the true page's score above that one: loss = log(1 + exp(s_wrong − s_true))." Then add "In simple words, it only has to beat the strongest wrong page."
- **[SHOULD]** The head zeroes padding, but Chapter 36 teaches that zeroed padding is a bug inside a max. Quote (Ch 46): "**Step 4:** Zero out padding positions with the attention mask." with `return v * mask.unsqueeze(-1)`. Also "The scoring function is byte-for-byte Chapter 36's MaxSim." Quote (Ch 36): "Padding rows zeroed out can still win. ... So we mask with a very large negative number, effectively −∞". Zeroing is what colpali-engine does, and it is safe here only because every page image has the same length (no page-side padding) and a zeroed query row adds exactly 0. Fix: add a **Note:** saying this, and that text documents with padding need Chapter 36's −∞ mask.
- **[NIT]** The projection is LoRA-adapted too, not fully trained. Quote: "only the final projection layer and LoRA adapters on the language model are trained". Paper: "LoRA ... on the transformer layers from the language model, as well as the final randomly initialized projection layer". The HF adapter config targets `custom_text_proj` with `modules_to_save: null`. Fix: "only LoRA adapters train, on Gemma's layers and on the final projection". The same wording appears in "When to fine-tune" and What people get wrong.
- **[NIT]** "byte-for-byte" points at the wrong chapter. The identical function (`sim.max(dim=1).values.sum()`) is Chapter 35's `maxsim`. Chapter 36's code is the masked batch version. Fix: "byte-for-byte Chapter 35's `maxsim` (Chapter 36 explains it)".
- **[NIT]** "VQA" is used as an acronym without expansion. Quote: "two-thirds from open VQA sets (DocVQA, InfoVQA, TAT-DQA, arXivQA)". The body only spells out "visual question-answering". Fix: "**VQA = visual question answering**" at line 163.
- **[NIT]** "throws away 15 of every 16 numbers" reads as picking a subset. A linear projection mixes all 2,048 numbers into 128. Fix: "squeezes every 2,048 numbers into 128".
- Style: clean (no semicolons or em-dashes in prose, bold-equation definitions for PaliGemma and LoRA, Steps before both code blocks, "you" only in advice).
- FIXES.md items for this chapter: all applied (training-set mix in all three places, LoRA gloss, "a few characters across one or two lines", "ColQwen2 and its Qwen2.5-based successors"). The applied 127,000 figure is the v1 number (see SHOULD).
- Code: 2 Python blocks (torch), plus an ASCII grid. I ran NumPy equivalents: `patch_heatmap` returns a (32, 32) grid and puts a planted match at patch 33 → (row 1, col 1) as the prose says; the head gives unit-length rows and zeroes masked rows; `colpali_score` equals a brute-force per-token max-then-sum (6.2703 both). Torch API use (`nn.functional.normalize`, `.max(dim=1).values`) is correct.

#### Fix status (applied 2026-09-16)

- [SHOULD] Training-set size is the v1 preprint figure — fixed (per D9: "about 119,000 (query, page) pairs (118,695)" in the one-paragraph version, body and Key takeaways)
- [SHOULD] ColPali loss is not Chapter 12's InfoNCE — fixed (per D9: Steps 3–4 now use the hardest in-batch wrong page, loss = log(1 + e^(s_wrong − s_true)), named softplus, framed as ColBERTv1's pairwise loss from Ch 36, "no softmax over every page and no temperature", plus "In simple words" line; one-paragraph version and a new Key takeaway match)
- [SHOULD] Head zeroes padding, contradicting Ch 36's −∞ mask — fixed (Note after the code: safe because pages are fixed length and a zeroed query row adds 0, while variable-length documents, including Ch 47's dynamic-resolution pages, need Ch 36's −∞ mask)
- [NIT] Projection is LoRA-adapted, not fully trained — fixed in all four places (training section, When to fine-tune, What people get wrong, Key takeaways) and in the ColPaliHead docstring
- [NIT] "byte-for-byte" points at wrong chapter — fixed differently: "the same computation as Chapter 35's `maxsim` (Chapter 36 explains it)", since the code is not literally byte-identical
- [NIT] "VQA" not expanded — fixed (**VQA = visual question answering.** in the training-data bullet)
- [NIT] "throws away 15 of every 16 numbers" reads as subset — fixed ("squeezes each one's 2,048 numbers into 128")
- Code: only the ColPaliHead docstring changed. Both blocks parse (ast). Checked the new loss and the padding Note with a NumPy harness: hardest-negative softplus computes as described, and appending a zeroed query row leaves the MaxSim sum unchanged.
- Cross-chapter follow-ups: none

## Chapter 47 — ColPali III: Production Reality  (reviewer: chapters 45–48)

- No MUST issues found. All arithmetic recomputed and correct:
  - Storage ladder: 527.36 KB, 527 MB, 21.1 GB, 52.7 GB, 527 GB, 5.27 TB.
  - 527,360 / 3,072 = 171.7 ("about 170×"). 20 × 1,030 = 20,600 dot products per page, 20.6 billion per million pages.
  - Pooling: 1,030 / 3 = 343 → ~340, and 340 × 128 × 4 = 174,080 B ≈ 174 KB (3×). Then 340 × 64 × 4 = 87 KB, and 340 × 64 bits = 2,720 B = 2.7 KB, so 2.7 GB per million pages.
  - 200 × 340 × 20 = 1.36 M dot products.
  - GPU-hours: 1 M pages ÷ 10 pages/s = 27.8 h and ÷ 2 pages/s = 138.9 h, so the stated throughput does give "30–140 GPU-hours". 0.39 s/page (2.6 pages/s, inside the range) gives 108 h ("about 110").
  - 340 × 36 B = 12,240 B ≈ 12 KB → 12 GB. ColPali float16 = 1,030 × 128 × 2 = 263,680 B ≈ 264 KB.
  - ColQwen2: 750 × 128 × 2 = 192 KB (≈190), 750 × 36 = 27 KB, 250 × 36 = 9 KB → 9 GB. Capped A4 is 644 × 896 = 77.9 × 76.6 DPI, with 2.875× the pixels of 448². The raised cap gives 896 × 1,288 = 108 × 110 DPI, 1,472 × 256 B = 377 KB, exactly 2.0× the pixels.
  - Ran the `visual_tokens` block: 736 and 1,472 as commented. Every row of the ColQwen2 table reproduces: A4 450 / 630 / 736, slide 464 / 646 / 720. A4 first hits the cap at 80 DPI (759 tokens at 78–79 DPI, 736 from 80), matching "about 80 DPI".
  - The RRF example holds with k = 60: 1/61 + 1/62 = 0.03252 beats 1/61 + 1/(60+r) whenever `SSO-4021` is below rank 2 in BM25.

  External checks:
  - HF `vidore/colqwen2-v1.0` preprocessor has `max_pixels: 602112` = 768 × 28², patch 14 and merge 2, so the "~768 by default" cap and 28-px tokens are right. Its base is Qwen2-VL-2B.
  - The resize sketch matches Qwen2-VL's `smart_resize` (round to 28, then floor after scaling by √(pixels / max_pixels)).
  - The ColPali paper confirms 0.39 s/page and "pool factor of 3 ... 66.7% [fewer vectors] ... 97.8% of the original performance" with hierarchical mean pooling. "2× pooling is close to free" is unverified for ColPali.

  Cross-references (Ch 8, 24, 26, 36, 37, 38, 40, 43, 44, 50, 54, 61) all match OUTLINE, and the numbers agree with Ch 61 and Ch 66.
- **[SHOULD]** What people get wrong contradicts itself two items later, and Ch 61. Quote: "**Storing float32.** There is no reason to. Quantize." Then: "keep the full-precision vectors there too, for rescoring and future migrations". Lever 3 also says "We would also keep higher-precision vectors somewhere for rescoring". Ch 61's cost table keeps "Full-precision pooled patch vectors (the truth) 1M × 250 × 512 B". Fix: "**Serving float32.** There is no reason to keep float32 in the hot index. Quantize it, and keep the float32 copy in object storage for rescoring."
- **[NIT]** The ColQwen2 token-count steps are out of order. Quote: "**Step 2:** Cut it into 14-pixel patches, then merge each 2 × 2 block ... **Step 3:** Round each side to a multiple of 28 pixels". The image is rounded and capped (Steps 3–4) *before* it is cut, which is also how the `visual_tokens` sketch does it. Fix: move the cutting step after the cap step.
- **[NIT]** Key takeaways drop one item from the "when not to use it" list. Quote: "Do not use it alone for plain prose, exact codes, sub-20 ms budgets, or when you need the text itself." The body also lists "**You have no GPU and no budget for one.**" Fix: add ", no GPU".
- **[NIT]** The heading and its bullet differ slightly: "## Lever 1: token pooling (do this first)" vs "- Lever 1: token pooling". Harmless.
- Style: clean (no semicolons or em-dashes in prose, Phase/Step before the MUVERA diagram and both algorithm blocks, a wrong-then-right answer in the `SSO-4012` hybrid example, "you" only in advice and the "When not to use it" conditions).
- FIXES.md items for this chapter: all applied. The DPI block was rewritten with the ColPali ~60 DPI **Note** and the sweep advice moved under ColQwen2. The ColQwen2 section was added (≈768 cap, ≈750 vectors, ≈190 KB float16, ≈27 KB compressed). The deliberate deviation "30–140 GPU-hours" is consistent between the table and Key takeaways. "~12 GB" is fixed, and "hierarchical clustering" is glossed.
- Code: 3 blocks. `visual_tokens` runs and matches its comments and the table. `pool_page_vectors` could not be run (no scipy) but reads clean: `linkage(V, method="average", metric="cosine")` takes an observation matrix, `fcluster(..., criterion="maxclust")` returns labels 1..k with k ≤ 343 for 1,030 vectors, and the mean-then-renormalise matches Steps 4–5. The `pdf2image` snippet is a fragment (`path` undefined) with a valid `convert_from_path(..., dpi=150, fmt="jpeg")` call.

#### Fix status (applied 2026-09-16)

- [SHOULD] "Storing float32. There is no reason to." contradicts rescoring advice — fixed ("**Serving float32.** There is no reason to keep float32 in the hot index. Quantize it, and keep the float32 copy in object storage for rescoring.", matching Ch 61's object-storage "truth" row)
- [NIT] ColQwen2 token-count steps out of order — fixed (round, then cap, then cut into 14-px patches merged 2×2; Step 2 points ahead to Step 4 for why 28)
- [NIT] Key takeaways drop "no GPU" — fixed
- [NIT] Lever 1 heading and bullet differ — fixed (bullet now "Lever 1: token pooling (do this first)")
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 48 — Audio, Video, Code, Graphs, and Users  (reviewer: chapters 45–48)

- No MUST issues found. Numbers verified:
  - A 20-minute video at 30 fps is 36,000 frames.
  - The 12:38 narration and the 12:40 SSO page are consistent.
  - Five modalities give C(5,2) = 10 pairwise datasets vs 4 with a hub ("ten datasets and four").

  External claims checked:
  - CLAP = Contrastive Language-Audio Pretraining, with a two-tower InfoNCE loss like CLIP.
  - x-vector and ECAPA are speaker-embedding families.
  - node2vec's p is the return parameter and q the in-out parameter (BFS-like walks lean to structural role, DFS-like to community, as the node2vec paper argues). DeepWalk uses uniform walks. struc2vec is the role-based method.
  - GraphSAGE/GAT can embed unseen nodes, and random-walk methods are transductive.
  - ImageBind binds its modalities to images, and audio–text alignment emerges.

  Cross-references checked against the chapters themselves:
  - Ch 2: "encodes the idea of similarity it was trained on".
  - Ch 5: recommenders let popular items grow larger norms.
  - Ch 7: `SSO-4012`.
  - Ch 13: "Two-tower model = another name for a bi-encoder".
  - Ch 41: MMR.
  - Ch 51: page 212's footnote links to page 1,140.
  - Part IV and Part V ranges are right, and What's next links 49.

  Every wrong-then-right example really demonstrates its point: the speaker embedding returns same-agent calls, frame-averaging dilutes the 20-minute tour, and character chunking splits `refresh_token`.
- **[SHOULD]** This overstates how uniform the training objective has been across the book. Quote: "The same objective has now appeared in this book for word2vec, sentence embeddings, CLIP, ColBERT, ColPali and recommenders. **It is the same four lines every time.**"
  - word2vec's negative sampling is a sigmoid loss on sampled negatives (Ch 9).
  - ColBERTv2 trains with KL distillation over 64-way tuples (Ch 36).
  - ColPali's paper loss is softplus(s_hardest_negative − s_positive), not in-batch InfoNCE (see Ch 46 review).
  - CLIP's loss is the symmetric two-direction version.

  Fix: "The same *idea*, pull the pair together and push the crowd apart, has now appeared for word2vec, sentence embeddings, CLIP, ColBERT, ColPali and recommenders. The exact loss varies, but its most common form is these four lines."
- **[NIT]** "never share a walk" is false for the graph the chapter describes. Quote: "two admins at *different* companies never share a walk, so random walks cannot place them close." Acme's graph has company → feature edges, so a walk can go admin → company → feature → company → admin. Fix: "rarely share a walk, and then only several hops apart".
- **[NIT]** The Note's recipe for keeping popularity would break the loss as written. Quote: "would drop the two `F.normalize` calls and score with the raw dot product." With `tau=0.05` still in place, raw dot products are multiplied by 20. On random 128-d rows I measured logits of ~3,000, which saturates the softmax. Fix: add "and set `tau` to 1 (or learn it)".
- **[NIT]** Unverified superlative. Quote: "This is the oldest and largest commercial use of vector search." Vector-space document retrieval (Salton, 1970s; LSI, late 1980s) predates vector recommenders. Fix: "one of the oldest and largest commercial uses".
- **[NIT]** Overclaim on BM25 for code. Quote: "error strings like `SSO-4012` are exactly the exact-match case from Chapter 7, and BM25 handles them perfectly." Identifier matching depends on the tokenizer (`refresh_token`, camelCase, `SSO-4012` split at the hyphen). Fix: "BM25 handles them well, provided the tokenizer keeps identifiers intact".
- **[NIT]** "GAT" is never expanded. Quote: "graph neural networks (GraphSAGE, GAT)". Fix: "GAT (graph attention network)".
- Style: clean (no semicolons, no em-dashes in teaching prose, bold-equation definitions for the recipe, CLAP, ASR and node2vec; Steps before each procedure; "you" only in advice and the article title "Articles you may need").
- FIXES.md items for this chapter: all applied ("**ASR = automatic speech recognition.**" defined before first use at the Steps, and repeated in Key takeaways).
- Code: 1 Python block (torch) plus the recipe card. I ran a NumPy equivalent of `TwoTower.loss`: the (B, B) in-batch logits and `arange` labels are right, the loss is ≈4.4 for random rows and ≈1.6e-7 when each item row equals its user row, so it pulls pairs together as described. The torch API use (`nn.Embedding`, `F.normalize`, `F.cross_entropy(..., torch.arange(len(u), device=u.device))`) is correct.

#### Fix status (applied 2026-09-16)

- [SHOULD] "It is the same four lines every time" overstates loss uniformity — fixed (per D9: "The same *idea*, pull each pair together and push the crowd apart, has now appeared ... **The exact loss varies, but its most common form is these four lines.**")
- [NIT] "never share a walk" false for company → feature edges — fixed ("rarely share a walk, and then only several hops apart, so random walks do not place them close")
- [NIT] Dropping F.normalize with tau=0.05 saturates softmax — fixed (Note adds "set `tau` to 1 (or learn it)" and one sentence on why)
- [NIT] "the oldest and largest commercial use" unverified — fixed ("one of the oldest and largest commercial uses")
- [NIT] "BM25 handles them perfectly" ignores tokenization — fixed ("BM25 handles them well, provided the tokenizer keeps identifiers such as `refresh_token` intact")
- [NIT] GAT never expanded — fixed ("such as GraphSAGE and GAT (graph attention network)")
- Code: no code changed. Checked the Note's saturation claim with NumPy: raw 128-d embedding dot products divided by 0.05 give logits in the hundreds.
- Cross-chapter follow-ups: none

## Chapter 49 — RAG From First Principles  (reviewer: chapters 49–56)

- No MUST or SHOULD issues found. All worked arithmetic reproduces (40,000 pages × 500 words = 20M words ≈ 26.7M tokens ≈ 27 million-token windows; 500,000 / 4,000 = 125; 750,000 words ≈ 8 novels). Every "Chapter N" reference matches OUTLINE.md (Ch 2 embedding model, Ch 3 top-k / empty set, Ch 10 tokens, Ch 19 recall, Ch 33 & 62 access control, Ch 40 RRF, Ch 41 rerank, Ch 45 ColPali, Ch 52–56, Ch 61). "What's next" links to 50-chunking.md. readingTime 13 min vs 3,210 words ≈ 14 min, fine.
- **[NIT]** Small mismatch between prose and code: the worked example says "The Librarian searches the Great Library and brings back two pages", while the `rag()` code and its gloss say "fetch the five closest chunks" (k=5). Quote: "fetch the five closest chunks, give up politely". Fix: add half a sentence, e.g. "the other three are near misses the Scholar is told to ignore", or set the example to k=5 with pages 212 and 1,140 among them (the text already says they "become sources [1] and [2]", which implies they rank first and second, so one clause suffices).
- **[NIT]** The comparison table says long-context cost is "High (up to hundreds of thousands)" of tokens while the section above talks about million-token windows. Quote: "High (up to hundreds of thousands)". Fix: "High (hundreds of thousands to a million)".
- Style: no notable violations (no semicolons or em-dashes in teaching sections; "we" used throughout; bold-equation definitions for LLM, prompt, context window, hallucination, RAG).
- FIXES.md items for this chapter: all applied (2.1 "Now the Scholar, whom we met in Chapter 1, takes centre stage"; 2.2 LLM = Large Language Model bold line; prompt and context-window glosses; hallucination named; "fifty times" replaced by "many times bigger… thousands of windows"; "slow to start answering"; 2.3 "four steps to prepare and six to answer").
- Code: 1 block (`rag()`), not self-contained (embedder/index/llm are external); read clean, matches the Phase 2 steps.

#### Fix status (applied 2026-09-16)

- [NIT] Worked example "two pages" vs k=5 in code — fixed: the code gloss now says pages 212 and 1,140 rank first and second, and the other three are near misses the answer does not need to cite.
- [NIT] Long-context cost "up to hundreds of thousands" vs million windows — fixed
- Code: no code changed
- Cross-chapter follow-ups: none

## Chapter 50 — Chunking  (reviewer: chapters 50–54)

- **[SHOULD]** `split_by_headings` produces the empty chunk that the chapter warns about in "Leaving empty chunks", and it does so on the chapter's own page 1,140. A `# Title` line followed by a blank line and then `## Audit logs` (exactly the outline shown in "Why chunking matters") sets `current = [""]`, which is truthy, so the function emits `('Security add-on', '')`. I ran it on that outline and got this as the first section. Quote: "if current:\n                sections.append((heading, \"\\n\".join(current)))". Fix: test `if "\n".join(current).strip():` in both places, or end with `if body.strip()` in the final comprehension.
- **[NIT]** The "more than a third" claim is only true before the title and heading are pasted on. It is 15/40 = 37.5% for the bare section, but the same paragraph prepends "Security add-on" and "Plans and seats" (about 7 tokens), which gives 15/47 ≈ 32%. Quote: "The SSO sentence is now more than a third of its chunk instead of 4%". Fix: "about a third of its chunk".
- **[NIT]** This overstates the empty-chunk failure and does not match the hedged wording in Ch 5 ("can produce all-zero embeddings") and Ch 44 ("a meaningless vector or a zero vector"). Most dense encoders embed "" as a non-zero vector made from the special tokens alone. Quote: "Whitespace-only chunks become zero vectors (Chapter 5)". Fix: "Whitespace-only chunks become meaningless or all-zero vectors (Chapter 5)".
- **[NIT]** Auto-merging retrieval is a variant, not a synonym. In LlamaIndex, the retrieved children are merged into their parent only when enough of that parent's children are hit. Quote: "sometimes called parent-document or auto-merging retrieval". Fix: "usually called parent-document retrieval (auto-merging retrieval is a close variant)".
- **[NIT]** "Top-20 retrieval failure rate" is not glossed, so a novice cannot tell what halved. Quote: "cut the top-20 retrieval failure rate by about half (~49%)". Fix: add "(the share of questions whose right chunk is missing from the top 20)". The figures themselves are correct: Anthropic, Sept 2024, 5.7% → 2.9% (−49%), and 1.9% (−67%) with reranking.
- **[NIT]** The chunk-size table row has two cells in a three-column table. Quote: "| Pages of scanned documents | Do not chunk — use the page (Ch. 45) |". Fix: "| Pages of scanned documents | Do not chunk, use the page (Ch. 45) | n/a |".
- Verified: 512 − 50 overlap gives a step of 462. A fixed-size chunker on a 350-token page gives one chunk. On 1,200 tokens it gives spans 0–511, 462–973 and 924–1199, which matches "each chunk starts 50 tokens before the previous one ends". Toy page 120+40+100+90 = 350 tokens, and 15/350 = 4.3% ("roughly 4%"). 50% overlap halves the step, so the index doubles. The sweep is 4 sizes × 3 overlaps = 12 settings, and its f-string prints "10%". The Anthropic contextual-retrieval figures are correct. Cross-references Ch 5, 10, 11 (late chunking Ninja note), 19, 34, 45, 49 (Step 2: Chunk), 51 (title prepend) and 53 (prompt caching) all point at the right topics. The "We will cover" list matches the eight headings. Page 88 as the SSO setup guide matches Ch 6, 14 and 28. Sources [1]/[2] match Ch 49.
- Style: no notable violations. There are no semicolons, and the only em-dashes in teaching sections are three in table cells. "You" appears only in advice and What people get wrong.
- FIXES.md items for this chapter: all applied. There is one default rule, stated in the blockquote and repeated in Key takeaways, with no leftover "this is the one that actually works". "The large share of retrieval quality" appears in What's next and in the OUTLINE line for Ch 51. Prompt caching is glossed and points to Chapter 53. "Retrieve small, pass large" in What people get wrong is cut to one line.
- Code: 4 blocks. The sweep loop is correct pseudocode (helpers are external). `split_by_headings` runs with a stubbed `split_if_long` and has the empty-section bug above. It also treats `#` comment lines inside fenced code blocks as headings: "```python\n# install" became a section named "install". That is a NIT, but relevant given the "Code | Per function" advice. `late_chunk` was read but not run (torch). It is correct in outline, with three NIT caveats. The spans must use the tokenizer's own indices, which include the leading [CLS]. Any span past the 8,192-token truncation slices to empty and averages to NaN. And there is no `torch.no_grad()`. The parent-child snippet matches Phase 2 Steps 1–3, provided `dedupe` preserves order.

#### Fix status (applied 2026-09-16)

- [SHOULD] `split_by_headings` emits the empty title chunk on page 1,140 — fixed: always append each section, then drop sections whose body is blank in the final comprehension; gloss now names the page 1,140 title-line case.
- [NIT] "more than a third" false once title and heading are prepended — fixed ("about a third")
- [NIT] "Whitespace-only chunks become zero vectors" overstated — fixed ("meaningless or all-zero vectors")
- [NIT] Auto-merging retrieval is a variant, not a synonym — fixed
- [NIT] "Top-20 retrieval failure rate" not glossed — fixed (added one sentence defining it)
- [NIT] Scanned-documents table row has two cells — fixed (em-dash removed, "n/a" third cell)
- [NIT, from Code note] `#` comment lines inside fenced code treated as headings — fixed: `in_code` flag toggled on ``` lines; gloss sentence added.
- [NIT, from Code note] `late_chunk` caveats (tokenizer indices, truncation NaN, no_grad) — fixed differently: span comment now says "spans in the tokenizer's indices"; truncation is already covered by the prose ("must first be split into large windows"); no_grad skipped (memory only, would need a torch import).
- [Section 0] Em-dashes (5) — fixed: the three in the trade-off table became parentheses, the table-row one became a comma, the code comment one became a comma. None remain outside the frontmatter `part`.
- Code: `split_by_headings` rewritten; extracted from the chapter and run with a stub `split_if_long` on the page 1,140 outline (sections: Audit logs, Plans and seats; no empty chunk) and on a fenced block with a `# install` comment (stays in one section). `late_chunk` only had comment changes.
- Cross-chapter follow-ups: none
- [Lead, verification pass] `split_by_headings` wrote a literal triple backtick inside a fenced block, which some blog renderers treat as the end of the block — fixed: `line.startswith("`" * 3)`. Re-run on the page 1,140 outline with a code block: no empty chunk, `#` inside code is not a heading.

## Chapter 51 — Metadata and Structure  (reviewer: chapters 50–54)

- **[SHOULD]** The first worked example contradicts the chapter's own setup for Chapters 51–56. In "Why we need metadata", the only change is a `status = "active"` filter, yet the Librarian then "brings back pages 212 and 1,140" and the Scholar gives the full caveat. Four sections later the chapter says the examples in Chapters 51 to 56 start from plain fixed-size chunks, with page 1,140 at rank 14 and "The Scholar gets page 212 alone". Dropping page 38,902 moves 1,140 up at most one place, to rank 13. Ch 56 repeats "rank 14, as in Chapter 51". Quote: "Page 38,902 never reaches the Scholar. The Librarian brings back pages 212 and 1,140". Fix: say "The Librarian brings back page 212 first, and the Scholar writes *"Yes, SSO is included on Pro."* The contradiction is gone. The under-50 caveat still needs the links we follow below." Also move the "examples in Chapters 51 to 56 start from…" framing sentence up to this first example.
- **[SHOULD]** The "combining everything above" scorer decays every document, which contradicts the chapter's own Note and Key takeaway ("only where freshness matters"). With now = 2026-09-16, page 1,140 (`updated_at` 2026-01-22, 237 days) keeps 0.5^(237/180) = 40% of its score. A June 2026 Globex contract (Ch 52) is also cut, the exact case the Note calls wrong. In my run with toy scores, page 1,140 fell from 0.78 to 0.31, below a contract chunk. Quote: "s *= 0.5 ** ((now - h.updated_at).days / 180)   # recency half-life". Fix: guard it, e.g. `if h.doc_type in DECAYING_TYPES:`, or use the blended `0.7 + 0.3 * decay` from the Note.
- **[NIT]** The heading and Key takeaway say "title and breadcrumb", but the code and its gloss prepend title and section only. Quote: "text_to_embed = f\"{title} > {section}\\n\\n{chunk_text}\"". Fix: `f"{breadcrumb}\n\n{chunk_text}"`, or change the heading to "title and section heading" (which is also what Ch 50 says).
- **[NIT]** "CMS" is never expanded here or in any earlier chapter. Quote: "| The filesystem / CMS |". Fix: "CMS (content management system)".
- Verified: 0.84 × 0.5^(730/180) = 0.0505 ≈ 0.05. 0.80 × 0.5^(30/180) = 0.713 ≈ 0.71. A two-year-old page keeps 6.0% ("about 6%"). 180 days gives half and 360 days a quarter. The blend at two years is 0.718. The ACL gloss is present. The 1,140 metadata (`updated_at` 2026-01-22, `url` …#plans-and-seats) matches Ch 53's `<source … updated="2026-01-22">`, and `kb_1140#c02` matches Ch 54's `gold_sources`. The chunk "It also records every export and permission change" matches Ch 50. Chapter refs 33 (filtering, partition by tenant), 40, 49 (citations), 50 (contextual retrieval, parent-child) and 62 are right. "We will cover" matches the seven headings. What's next links to 52.
- Style: no notable violations. There are no semicolons or em-dashes in teaching sections, and the analogy has its mapping. "You" appears only in the table header "What you get" and in What people get wrong / Ninja notes.
- FIXES.md items for this chapter: all applied ("ACL = access-control list: who may see this document").
- Code: 4 Python blocks plus 1 text block. `recency_boost` reproduces every number in the prose. As a NIT, it reads an undefined global `now` and has an unused `import math`, so pass `now` in as `score_with_metadata` does. `score_with_metadata` runs on toy hits: it drops superseded pages, keeps at most 2 chunks per `doc_id` and sorts correctly, apart from the decay issue above. As a NIT, it mixes `query_meta.get("preferred_type")` with `query_meta["language"]`, which raises KeyError when language is absent. `retrieve_with_structure` reads clean as pseudocode and matches Steps 1–4. `[:2]` after `fetch` does give "up to 2 referenced pages".

#### Fix status (applied 2026-09-16)

- [SHOULD] First example gets pages 212 and 1,140, contradicting rank 14 — fixed: after the filter the Librarian brings back page 212 first, the Scholar says "Yes, SSO is included on Pro", the caveat is still missing until links are followed; the "examples in Chapters 51 to 56 start from…" framing moved up to the first example, and the Key takeaway no longer says "the answer was right".
- [SHOULD] Combined scorer decays every document, against its own Note — fixed: decay now applies only to `DECAYING_TYPES = {"forum_post", "release_note"}`; gloss says pages 212 and 1,140 keep full scores and the two-year-old forum post sinks.
- [NIT] "title and breadcrumb" heading vs title + section code — fixed differently: heading and Key takeaway now say "title and section heading" (matches the code, its gloss, and Ch 50).
- [NIT] "CMS" never expanded — fixed
- [NIT, from Code note] `recency_boost` uses undefined global `now` and unused `import math` — fixed (`now` is a parameter, import removed)
- [NIT, from Code note] `query_meta["language"]` KeyError when absent — fixed (`query_meta.get("language", h.language)`)
- Code: `recency_boost` and `score_with_metadata` extracted from the chapter and run with toy hits: 0.0505 and 0.7127 reproduce the prose's ≈0.05 and ≈0.71; superseded page dropped, forum post decays to 0.051, help articles and a June 2026 contract keep full scores, cap of 2 chunks per doc works, no KeyError without `language`.
- Note: while fixing Ch 53 (child chunks), the framing sentence gained "unless a chapter says otherwise", and the plain-retrieval paragraph was rewrapped.
- Cross-chapter follow-ups: none (Ch 56's "rank 14, as in Chapter 51" still matches).

## Chapter 52 — Query Understanding  (reviewer: chapters 50–54)

- **[SHOULD]** The Decomposition section describes multi-hop correctly: the second search cannot be written until the first returns, and "the hops must run one after another". The comparison table and the routed code then treat multi-hop as a single one-shot decomposition. Multi-hop needs at least one LLM call per hop to write the next query from the previous results, run in sequence. `decompose(q)` followed by `parallel_map` cannot do the Globex example, because the second hop's "Pro, 120 seats" is not known when `decompose` runs. Quote: "| Decomposition | Multi-part, comparative and multi-hop questions | One LLM call plus N searches |". Fix: "One LLM call plus N searches (multi-hop: one LLM call and one search per hop, in sequence; Chapter 56)". Add a comment in `understand()` that multi-hop questions go to the agent loop of Chapter 56.
- **[SHOULD]** The multi-query latency claim ignores the serial LLM call that writes the paraphrases, and that call is the dominant cost. The chapter's own figure ("Four LLM calls before retrieval turn a 300 ms system into a 3-second one") implies about (3,000 − 300) / 4 ≈ 675 ms per call. That alone more than triples a 300 ms pipeline, however parallel the searches are. Quote: "The searches run in parallel, so the extra waiting time is modest." Fix: "The searches run in parallel, so they add little waiting time. The LLM call that writes the phrasings is the real cost, often more than half a second before the first search starts."
- **[NIT]** This description of page 1,140 contradicts Ch 50, which makes it a page of about 350 tokens and says "The page is short, so we let the whole page be the parent". Quote: "Page 1,140 is a long help article about an add-on, mostly about audit logs." Fix: "Page 1,140 is a help article about an add-on, mostly about audit logs."
- **[NIT]** The multi-query section says "no words", but the HyDE section of this chapter and Ch 14 both say "almost no". Quote: "Take our paraphrase twin, which shares no words with the pages that answer it". Fix: "which shares almost no words with the pages that answer it".
- **[NIT]** The chapter's own advice is "Preserve quoted strings, codes and identifiers verbatim". The `REWRITE` prompt does not say this, and `understand()` rewrites before it classifies. In a stubbed run, a rewrite that normalises "and SSO-4012?" into "SAML login error" wrongly takes the plain lookup path and loses `sparse_weight: 2.0`. Quote: "Resolve all pronouns and references. Output only the query." Fix: add "Keep codes, quoted strings and identifiers exactly as written." to the prompt, or run the identifier regex on the raw message before rewriting.
- Verified: the Globex multi-hop matches the scenario. Pages 3,507–3,508 give Pro with 120 seats (same as Ch 44/45/61), and page 1,140's rule covers only teams under 50, so Globex needs no add-on. The multi-query example counts four lists (original + 3), which matches `[q] + paraphrase(q, n=3)`. SSO-4012 = "SAML assertion expired" and the SSO-4021 near-miss match Ch 7. The "rank 14, as in Chapter 51" framing is honoured. HyDE = Hypothetical Document Embeddings (Gao et al. 2022). The reference interview and step-back prompting are real. Chapter refs 8 (query expansion), 14 (prefixes), 34 (multi-constraint failure), 40 (RRF consensus), 51 (Ninja notes, `status = "active"`) and 56 are right. The multi-hop gloss is present. "We will cover" matches the eight headings. What's next links to 53.
- Style: no notable violations. There are no semicolons or em-dashes in teaching sections, and the support-agent analogy has its mapping. "You" appears only in advice ("If you do only one thing…") and What people get wrong.
- FIXES.md items for this chapter: all applied ("Multi-hop question = a question where one fact is needed before we can even search for the next").
- Code: 2 Python blocks plus 4 text blocks. `REWRITE` is a valid format string. `understand`/`retrieve` run with stubbed helpers. The four intent paths return what the prose says, and `rrf` over decomposed sub-queries interleaves both lists. The only problems are the multi-hop and identifier-rewrite points above.

#### Fix status (applied 2026-09-16)

- [SHOULD] Table and routed code treat multi-hop as one-shot decomposition — fixed: table cell adds "multi-hop: one LLM call and one search per hop, in sequence, Ch. 56"; `understand()` gains a `multi_hop` intent, `retrieve()` sends it to a new `multi_hop()` loop (one LLM call then one search per hop, in order); gloss and Key takeaway say hops run in order.
- [SHOULD] Multi-query latency ignores the paraphrase-writing LLM call — fixed (searches add little waiting time, the LLM call is the real cost, often more than half a second before the first search)
- [NIT] "long help article" contradicts Ch 50's short page — fixed
- [NIT] "shares no words" vs "almost no" elsewhere — fixed
- [NIT] REWRITE prompt does not preserve identifiers — fixed: prompt adds "Keep codes, quoted strings and identifiers exactly as written."; gloss mentions `SSO-4012` left untouched.
- Code: `REWRITE` formatted with `.format()`; `understand`/`retrieve`/`multi_hop` extracted from the chapter and run with stubbed helpers: identifier path gets `sparse_weight: 2.0`, compound decomposes into two searches fused by RRF (212 then 1,140), Globex runs two sequential hops (contract search, then 1,140) and stops, exploratory runs 4 searches, lookup runs 1.
- Cross-chapter follow-ups: none

## Chapter 53 — Context Assembly  (reviewer: chapters 50–54)

- **[MUST]** The worked example only puts page 1,140 last because it silently changes the reranker's order. "Why we need context assembly" gives the rerank order as 212 (1), Okta (4), Azure AD (5), Team roles (6), 1,140 (7). After Steps 1–3 the five survivors in rerank order are therefore 212, Okta, Azure AD, Team roles, 1,140. Running the chapter's own `order_for_attention` on that list gives [212, Azure AD, **1,140**, Team roles, Okta]. Page 1,140 lands at position 3 of 5, the exact middle. The prose instead says the reranker ranks page 1,140 second, and then claims "Retrieval did not change at all. Only the assembly did." Quote: "The reranker ranks them: page 212, page 1,140, the Okta guide, the Azure AD guide, and \"Team roles\"." Fix: make the rerank explicit, e.g. "Step 2 (end): we rerank the expanded sections. With its title and heading now attached, page 1,140's section rises to second." Then change "Retrieval did not change at all" to "The retrieved chunks did not change". Alternatively, keep the original order and show the fix coming from labels and instructions rather than position.
- **[NIT]** The code's default dedupe threshold deletes the exact case that Problem 1 warns about. Two vectors at cosine 0.95 are ≥ 0.92, so `dedupe` keeps only the first. I confirmed this on a constructed 0.95 pair. Quote: "Two versions of a pricing page can be 0.95 similar and still differ in the one number that matters. Filter superseded pages first (Chapter 51), and keep the dedupe threshold high." Fix: "…keep the dedupe threshold above that, e.g. 0.97 or higher, or dedupe only within one `doc_id`".
- **[NIT]** The 800-token example system prompt is below the minimum prefix that major providers cache. OpenAI and Anthropic document a minimum of about 1,024 tokens, and some models need more. The caching saving therefore appears only once system prompt plus history cross that minimum. Quote: "These instructions are the same for every question, so they belong in the system prompt, at the very start. That placement also saves money, through prompt caching." Fix: add "Providers cache only prefixes above a minimum length, often about 1,024 tokens, so check yours."
- **[NIT]** Chunk granularity drifts from the Ch 51 framing that "the examples in Chapters 51 to 56 start from the plain fixed-size chunks at the start of Chapter 50". With 512-token chunks, page 1,140 (about 350 tokens) is one whole-page chunk, title included. Here it arrives as a bare one-sentence fragment that "expands" to its 40-token "Plans and seats" section. Quote: "Page 1,140's caveat chunk expands to the whole \"Plans and seats\" section, heading included." Fix: say this chapter's pipeline uses small child chunks (Chapter 50's parent-child), or narrow the Ch 51 framing sentence.
- Verified: 800 + 3,000 + 2,000 + 12,000 = 17,800 ("under 18,000"). `order_for_attention([1..6])` gives [1,3,5,6,4,2], as the comment says. The five survivors (212, Okta, Azure AD, Team roles, 1,140) are counted right. The lost-in-the-middle definition matches Liu et al. 2023 (U-shaped accuracy, with primacy usually stronger than recency, which supports best-first, second-best-last). Citation format `[n]` and the order system → history → sources → question match Ch 49's prompt ("sources [1] and [2]"). Here 1,140 becomes [5] because of ordering, which the text explains. `updated="2026-01-22"` and the URL match Ch 51's metadata. The FIXES items are in: system prompt glossed, and the prose now says the item that does not fit is "dropped whole", which matches `break`. Chapter refs 5, 41 (MMR), 47 (tables spanning pages), 49 (distractor, cost, waiting), 50, 51 (cap per page, superseded), 54 (faithfulness) and 62 (prompt injection) are right. "We will cover" matches the eight headings. What's next links to 54.
- Style: no notable violations. There are no semicolons or em-dashes in the text, and the briefing analogy has its mapping. "You" appears in the one-paragraph version ("more than you would expect") and in advice.
- FIXES.md items for this chapter: all applied.
- Code: 5 Python blocks plus 3 text/prompt blocks. `dedupe` and `order_for_attention` run and do what the prose says, apart from the threshold point above. `assemble` runs with stubbed helpers and produces system / history / sources / question in the stated order. NITs: the f-string label omits the `section` attribute shown in the Step 5 example. Nothing escapes `title` or `text`, so a page containing `</source>` closes the tag early; I confirmed this in the stub run, and it undercuts the prompt-injection point. `history` sits bare beside the sources with no delimiter, contrary to "History and sources should be clearly separated".

#### Fix status (applied 2026-09-16)

- [MUST] Worked example silently reorders reranker list; 1,140 would land mid — fixed: "Choosing what goes in" Step 2 and the example's Step 2 now rerank the expanded items (page 1,140's section, with title and heading attached, rises from last to second); Step 4 says "In the new rerank order"; `assemble()` gains `rerank(question, items)` after the second dedupe; gloss and Key takeaway mention the rerank; "Retrieval did not change at all" → "The retrieved chunks did not change".
- [NIT] Default dedupe threshold 0.92 deletes the 0.95 pricing-page case — fixed: default is 0.97, Problem 1 says "above that, at 0.97 or higher, or dedupe only within one page".
- [NIT] 800-token system prompt below provider cache minimum — fixed (added "Providers cache only prefixes above a minimum length, often about 1,024 tokens, so check yours. Our 800-token system prompt is too short on its own.")
- [NIT] Chunk granularity drifts from Ch 51 fixed-size framing — fixed: Ch 53 now says this pipeline indexes small child chunks (Chapter 50's parent-child), and Ch 51's framing sentence adds "unless a chapter says otherwise".
- [NIT, from Code note] f-string label omits `section` — fixed
- [NIT, from Code note] `title`/`text` unescaped, `</source>` closes the tag early — fixed with `html.escape` (text escaped with quote=False); gloss sentence added.
- [NIT, from Code note] history sits bare beside sources — fixed: wrapped in `<history>` tags.
- Code: all 3 Python blocks extracted and run. `dedupe` keeps a 0.95 pair and merges a 0.99 pair. `order_for_attention([1..6])` = [1,3,5,6,4,2]. On the post-rerank list [212, 1,140, Okta, Azure AD, Team roles] it gives [212, Okta, Team roles, Azure AD, 1,140], exactly the prose (without the rerank it gives 1,140 in the middle). `assemble` with stubs: 1,140 is source 5, a page containing `</source>` is escaped, history is in its own tags.
- Cross-chapter follow-ups: none

## Chapter 54 — Evaluating RAG  (reviewer: chapters 50–54)

- **[SHOULD]** The claim that a 5-point gain "starts to show" at 200 questions is contradicted by the table just above it. At n = 200 the 95% interval on a single score is ±6.9 points, already wider than 5. The difference between two independently scored runs has a 95% interval of 1.96 × √(2 × 0.25 / 200) = ±9.8 points. A ±5-point 95% interval on one score needs 0.25 × (1.96/0.05)² ≈ 385 questions. A 5-point difference only becomes visible at 200 through a paired comparison, which the next sentence plays down as "narrows the noise somewhat". Ch 19 says paired tests have "far more statistical power". Quote: "So with 20 questions, a 5-point improvement is invisible. With 200, it starts to show. Comparing two versions on the same questions narrows the noise somewhat". Fix: "With 200, one score is still ±7 points, so a 5-point gain shows only when we compare both versions question by question on the same set (a paired test, Chapter 19). Unpaired, we would need roughly 400 questions for a ±5-point interval on each score."
- **[SHOULD]** Gold sources are keyed to chunk IDs, so the eval breaks exactly when the chapter says to run it. That includes the structure-aware re-chunking in its own worked fix and Ch 50's 12-setting chunk-size sweep. After re-chunking, `kb_1140#c02` either does not exist or names a different span, and `recall` silently drops. The prose itself counts pages ("1 of 2 gold pages"), and Ch 19's golden set is page-level. Quote: "gold_sources: [\"kb_212#c01\", \"kb_1140#c02\"]" and "ctx_ids = {c.chunk_id for c in out.context}". Fix: store `gold_sources: ["kb_212", "kb_1140"]` (doc/page IDs, or answer-bearing text spans) and compute `ctx_ids = {c.doc_id for c in out.context}`. Also change "`kb_212#c01` and `kb_1140#c02`" in the Under the hood gloss.
- **[SHOULD]** `evaluate()` silently drops the refusal and correctness scores from its report. `refused` mixes `None` with booleans, and so does `correct` if the judge returns True/False. pandas stores those columns as `object`, and `.mean(numeric_only=True)` discards them. I confirmed this with pandas 3.0.5: the output has only recall, faithfulness, latency_ms and tokens, so the unanswerable segment shows NaN everywhere. Quote: "return df.groupby(\"segment\").mean(numeric_only=True)". Fix: `refused = float(judge_is_refusal(out.answer))` and `correct = float(judge_correctness(...))`, or `df = pd.DataFrame(rows).astype({"correctness": float, "refused": float})`. Also add `import pandas as pd`.
- **[NIT]** The one-paragraph version promises "faithful, relevant and complete", and the frontmatter tags RAGAS. The four measurements drop RAGAS's answer relevancy in favour of answer correctness without saying so, so a reader opening RAGAS will meet a metric the chapter never named. Quote: "was the answer faithful, relevant and complete?". Fix: one sentence under measurement 4: "RAGAS also reports **answer relevancy**, whether the answer addresses the question at all, which needs no reference answer."
- **[NIT]** Raw judge–human agreement can look high on skewed labels. If 90% of claims are SUPPORTED, a judge that always says SUPPORTED agrees 90% of the time. Quote: "**Step 3:** Measure how often the judge agrees with you." Fix: "…agrees with you, separately for each label (or with Cohen's kappa), since most claims are supported and a lazy judge scores well on raw agreement."
- Verified (FIXES target): √(0.5 × 0.5 / 20) = 0.1118, so ±11 points and a 95% interval of 1.96 × 11.2 = ±21.9 ≈ ±22 (28%–72%). Table rows: n = 50 gives ±7.07 / ±13.9, n = 100 gives ±5.0 / ±9.8, n = 200 gives ±3.54 / ±6.9, all correctly rounded. p = 0.5 is the worst case. What people get wrong and Key takeaways repeat ±11 / ±22 and match Ch 19. Also verified: the SSO diagnosis (recall 1/2 = 0.5, faithfulness 1/1, correctness low) maps to the "Low | — | Low → retrieval" row. The "no extra cost" judge example gives 1/2 = 0.5. The `FAITHFULNESS_JUDGE` braces format correctly. ColPali training data is about one-third synthetic, as in Ch 46. The judge biases (length, own-family, fluency) are the documented ones. Chapter refs 19 (golden set, standard error, bootstrap, segments), 46, 50 and 52 are right. "We will cover" matches the eight headings. What's next links to 55.
- Style: no notable violations. There are no semicolons, and the only em-dash is a table cell used as "any". The mechanic analogy has its mapping. "You" appears only in advice (calibration steps) and What people get wrong.
- FIXES.md items for this chapter: applied, as the documented deviation "±11 points, 95% ±22", in both the body and What people get wrong. No new error was introduced.
- Code: 3 blocks (YAML, judge prompt, `evaluate`). The prompt formats cleanly. `evaluate` runs with stubs and pandas, and has the dropped-column problem above. The YAML is valid, apart from the chunk-ID keying point.

#### Fix status (applied 2026-09-16)

- [SHOULD] "5-point gain starts to show at 200" contradicts the table — fixed: 200 gives ±7 on one score; a ±5-point 95% interval needs about 385 questions (0.25 × (1.96/0.05)² ≈ 385, new table row 385 | ±2.5 | ±5); unpaired comparison of two runs needs about twice as many per run; a paired test on the same questions (Chapter 19) often shows a 5-point gain at about 200. Key takeaway updated to match.
- [SHOULD] Gold sources keyed to chunk IDs break on re-chunking — fixed: YAML now stores page IDs (`kb_212`, `kb_1140`, `kb_17450`); prose explains why and mentions an optional short quote for long pages; `evaluate()` compares `c.doc_id`; Under the hood gloss updated.
- [SHOULD] `evaluate()` silently drops refusal and correctness — fixed: `float()` around `judge_correctness` and `judge_is_refusal`, `import pandas as pd` added, gloss explains the dropped-column trap.
- [NIT] RAGAS answer relevancy never named — fixed (one short paragraph under measurement 4)
- [NIT] Raw judge–human agreement misleading on skewed labels — fixed (Step 3: per label, plus Cohen's kappa glossed)
- Code: YAML block parsed with PyYAML; `FAITHFULNESS_JUDGE.format()` checked; `evaluate()` extracted and run with pandas 3.0.5 (installed into the venv) on the chapter's own YAML with a stub system whose chunk IDs differ from any stored ID and judges that return bools: report has recall (SSO 0.5), faithfulness, correctness (SSO 0.0, identifier 1.0) and refused (unanswerable 1.0) columns. Stats re-computed: 384.2 → 385, 1.96 × √(0.25/385) = 0.0499; paired example with 10% flipped questions at n = 200 gives ±4.3 points.
- Cross-chapter follow-ups: chapters/55-rag-failure-modes.md `diagnose()` compares `example.gold_sources` to `c.chunk_id` and `h.chunk_id`; since Ch 54's gold_sources are now page IDs, those should become `c.doc_id` and `h.doc_id` (e.g. `ctx_ids = {c.doc_id for c in output.context}`).

## Chapter 55 — RAG Failure Modes: A Field Guide  (reviewer: chapters 55–59)

- No MUST issues found. Numbering is consistent: there are exactly 12 failures, 1–8 under "Retrieval failures" and 9–12 under "Generation failures". That matches the OUTLINE ("Twelve ways it breaks"), the one-paragraph version, the cover list, the split ("Go to failures 1–8" / "9–12"), the Key takeaways, the example ("So look at failures 1–8", "That is failure 8"), What people get wrong (failure 2, failure 8), Ninja notes (F1, F4, F9), What's next (failure 8) and every label in `diagnose()` (F1; "F2–F7 (chunking, rewrite, stale, hubs, OCR, filters)" maps one-to-one onto failures 2–7; F8; F9; F10; "F11, F12"). Other chapters that cite Ch 55 are all correct: Ch 56:86 "failure 8 from Chapter 55: only part of the evidence arrived". Ch 3's promises land on the exact headings "Failure 9, part 1: calibrating the distance floor" and "Failure 9, part 2: why the right floor moves with the query", and Ch 3's "gap between the top result and the tenth … Chapter 55 turns this into a procedure" is delivered by "Add the gap". The references in Ch 16, 36, 40, 41, 46, 49, 63 and 66 also match. I checked the 20-seat example: 20 < 50, so the customer really does need the Security add-on, and "It is half an answer" is right. Arithmetic checked: SE at n = 20 is √(0.25/20) = 0.112, so "±11 points" is right. From the sweep table, the 2% cap picks 0.60, and 100 − 55 = 45% of unanswerable questions get through. The gaps are 0.83 − 0.81 = 0.02 and 0.71 − 0.46 = 0.25. Pet insurance at 0.41 < 0.60 is refused. A at 0.12 < 0.45 is refused and B at 0.88 is answered. Every "Chapter N" reference matches OUTLINE and the cited chapter really has the idea: Ch 7 and 10 on shredded identifiers, 33 on pass rate, pre/in-filtering and partitioning, 50 on contextual retrieval and retrieve small / pass large, 51 on status, 52 on routing identifiers, 53 on "say what is missing", lost in the middle and the question after the context, 54 on `answerable: false` and segments, 41 on scores not being comparable across models, and 46 on per-token best-patch scores. VLM is defined in Ch 43.
- **[SHOULD]** In the flagship example, the stated cause of failure 8 is failure 2's cause, so the example cannot tell the two apart. Failure 2 already uses page 1,140 as its example ("states the SSO seat rule in one sentence among paragraphs on audit logs and data retention", i.e. dilution). Failure 8 then explains the SSO miss the same way. Quote: "Page 1,140 is mostly about audit logs and data retention, so it misses the top results." The worked example jumps from "one part answered" to failure 8 and never runs failure 2's test. By the chapter's own wording, that test would also come back positive. `diagnose()` also sends every partial-evidence case to F8. A reader cannot see why decomposition, not smaller chunks, is the fix. Fix: make failure 8's cause about the question, not the page. For example: "One query vector sits in one place. It lands next to page 212, which answers 'is it included?'. The 'under what conditions?' part is never said in the question, so nothing pulls the search towards page 1,140." Then add one line after Step 4: "If the second search still misses page 1,140, run failure 2's test too. The two often come together."
- **[NIT]** The bold equation defines one name, and the chapter then uses a different one. Quote: "**Relevance floor = the lowest score the best result must reach before the Scholar may answer.**" followed by "Chapter 3 called it the **distance floor** … and we use that name here". Chapters 46, 49 (`RELEVANCE_FLOOR`), 63 and 66 say "relevance floor", while Ch 3 and the rest of Ch 55 say "distance floor". A novice now has three nearby terms: relevance floor, distance floor and relevance check. Fix: lead with "**Distance floor = …**" and add "(Chapters 49, 63 and 66 call it the relevance floor)".
- **[NIT]** Failure 3's test clue does not appear in its own example. Quote: "Unresolved words like \"it\" or \"that one\" confirm it." The example follow-up is "And what about Basic?", which has neither word. Fix: "Words that only make sense inside the conversation, like \"what about\", \"it\" or \"that one\", confirm it."
- Style: clean. There are no semicolons or em-dashes in the prose (only in the frontmatter summary and the code strings). Every failure has the symptom/cause/test/fix shape, the mechanic analogy maps all four parts, the example shows a wrong answer then a right one, and calibration is written as Phases and Steps before the code.
- FIXES.md items for this chapter: all applied. The calibration procedure is present and more detailed than the spec: `answerable: false` sweep, 2% cap, re-run on model change (Ch 41), and the crowded-neighbourhood reason for the moving floor (Ch 3). The old failure 11 now sits in the retrieval group as failure 8, with 8 retrieval and 4 generation failures, and `diagnose()` was updated ("check F11, F12" now names numbers-wrong and injection correctly). The renumbering introduced no stale number anywhere in the book.
- Code: 3 blocks. `diagnose()` was run with stub helpers on six cases: partial → F8, ungrounded → F10, grounded → "F11, F12", fabricated unanswerable → F9, BM25-only → F1, and cut-by-rerank. Every case returned the label the prose predicts. `not gold & ctx_ids` parses as `not (gold & ctx_ids)`, as intended. `calibrate_floor()` was run on 150 answerable and 50 unanswerable toy scores. It picked 0.657 (78% correct refusals, 1.3% wrong refusals), the same answer as a brute-force 0.001 grid. `answer_or_refuse()` needs a reranker. On reading, it is correct and matches Part 3's steps 1–3.

#### Fix status (applied 2026-09-16)

- [SHOULD] Failure 8 cause duplicates failure 2's dilution cause — fixed: cause is now about the question (one query vector, "under what conditions?" never said), plus a line after Step 4 to run failure 2's test if the second search still misses page 1,140. Numbering unchanged.
- [NIT] Bold definition names "relevance floor", chapter uses "distance floor" — fixed: bold definition now "Distance floor = …", with "Chapters 49, 63 and 66 call it the relevance floor". Also changed failure 9's cause "no relevance floor" to "no distance floor" for one term inside the chapter.
- [NIT] Failure 3 test clue ("it", "that one") absent from its example — fixed
- Code: no code changed
- Cross-chapter follow-ups: none (Ch 46, 49, 56, 63, 66 keep "relevance floor"; Ch 55 now names that alias explicitly)
- [Lead follow-up from Ch 54 fixer] `diagnose()` matched gold sources against chunk ids, but Ch 54's gold sources are now page ids — fixed: uses `doc_id`. Tested on full, partial, cut-by-rerank and unanswerable cases.

## Chapter 56 — Agentic Retrieval  (reviewer: chapters 55–59)

- No MUST issues found. The loop arithmetic is right. In "It took two searches and three model calls instead of one search and one call", the calls are: call 1 → search, call 2 → search, call 3 → answer. Running the orchestrator with a scripted stub gives exactly 3 LLM calls and 2 searches, and the table row "Several (our SSO example used three)" agrees. The Step 1–6 list matches the loop diagram and `agentic_answer()`: answer or call a tool, run it, append the result, go back to Step 2, force an answer at the step cap. Scenario checks pass. "Page 1,140 … SSO chunk at rank 14, as in Chapter 51" matches Ch 51:202 and Ch 52:144. Page 212's footnote "Seat limits apply. See Security add-on." matches Ch 51:196. Page 88 is the SSO setup guide and page 2,306 is "Compare plans", as in Ch 3/14/19. Globex's contract is page 3,507. "It is failure 8 from Chapter 55" is correct after the renumbering. Every term is defined before use: agent, tool call and orchestrator here, multi-hop and HyDE in Ch 52, neighbour expansion in Ch 47/51, system prompt in Ch 53. Cross-references (Ch 50, 51, 52, 53, 54, 55, 64, Parts IV–V) match OUTLINE. What's next links 57-architecture-at-scale.md, and Ch 57 is indeed framed at "a hundred million chunks". The cover list matches all eight section headings.
- **[SHOULD]** The comparison table says the agent's own judgement replaces the relevance floor, and it shows only the easy case. This undercuts the chapter just before it. Quote: "| Relevance floor | The model judges whether results are relevant |" and "A well-designed agent handles **the empty set** gracefully. Ask it *\"Does Acme offer pet insurance?\"*". Ch 55 says pet insurance is "the easy case", that the hard case is a crowded neighbourhood ("Does the Pro plan include phone support?" returns plausible Pro pages at 0.83), and that "Failure 9 deserves engineering, not just a prompt". An agent that reads those Pro pages can still invent phone support. Fix: after the pet-insurance paragraph, add "This is Chapter 55's LLM relevance check, run by the agent itself. It still needs measuring on the golden set's `answerable: false` questions, and the hard case is a question like phone support on Pro, where every result looks close."
- **[NIT]** Token cost grows faster than the number of loops, but the chapter says only "multiplies". Quote: "| Token cost | Lowest | Multiplies with every loop |" and "Token costs multiply with every loop." Each call re-reads the whole growing conversation, so call k reads k − 1 result sets. The SSO example reads 0 + 1 + 2 = 3 result sets against 1 for single-shot. At the `max_steps=6` cap plus the forced answer it reads 0 + 1 + … + 6 = 21, while retrieval latency grows only 6×. Fix: one sentence with that sum, plus "prompt caching (Chapter 53) softens it". The same passage says "Cap iterations and total tokens", but the code caps only steps. Add a token counter or drop "and total tokens".
- **[NIT]** "Trajectory" is used four times and never defined (table "A whole trajectory to inspect", "Not logging the trajectory", Ninja notes, Key takeaways). Fix: at first use, "a whole **trajectory** (every query, result and model step, in order) to inspect".
- **[NIT]** The tool's default result count does not match the example. Quote: "\"limit\": {\"type\": \"integer\", \"default\": 8, \"maximum\": 20}" versus "The orchestrator returns the same five pages as before". The example call passes no `limit`, so it would return 8. Page 1,140 is at rank 14 either way, so the lesson stands. Fix: "default": 5, or "returns the same top pages as before".
- Style: clean. There are no semicolons or em-dashes in the prose (the one semicolon is inside the tool-description string). The folder / Library-card analogy has its mapping, and the chapter shows the wrong answer then the right one. It ends with a comparison table and "We must use … Many strong systems use both."
- FIXES.md items for this chapter: all applied (the tool call / orchestrator sentence is present, word for word, before the loop, under "What is agentic retrieval").
- Code: 2 blocks, both run clean. The tool definitions are valid JSON-Schema `input_schema` dicts. `agentic_answer()` was run with a scripted stub LLM. The SSO script ended after 3 calls and 2 searches. A stub that never answers stopped after 6 calls and then called `force_final_answer`. The block format (`tool_use` blocks with `.id/.name/.input`, and `tool_result` with `tool_use_id` in a user turn) is the right Messages-API shape. Minor: the comment "parallel calls allowed" sits on a serial `for` loop, while Ninja notes say "Execute them concurrently". A thread pool, or a comment "run these concurrently in production", would match.

#### Fix status (applied 2026-09-16)

- [SHOULD] Agent judgement shown as replacing the relevance floor, easy case only — fixed: added paragraph after pet insurance (Chapter 55's LLM relevance check run by the agent, still measured on `answerable: false`, phone-support-on-Pro is the hard case).
- [NIT] Token cost grows faster than loops, and "total tokens" cap not in code — fixed: sum sentence (3 vs 1 result sets, 21 at the cap, prompt caching softens it), table row reworded, and `agentic_answer()` now has a `max_tokens` budget checked at the top of the loop (Step 6 and the prose below the code mention it).
- [NIT] "Trajectory" never defined — fixed (defined in the table cell at first use)
- [NIT] Tool default `limit` 8 vs "same five pages" — fixed: default is now 5.
- [NIT] (Code note) "parallel calls allowed" comment on a serial loop — fixed: comment now "run concurrently in production".
- Code: orchestrator block changed (token budget, comments). Extracted and run with a stub LLM: SSO script gives 3 model calls and 2 searches, a never-answering stub stops after 6 calls and forces an answer, a 30k-tokens-per-call stub stops after 2 calls on the token budget with the conversation ending on a valid tool_result turn. SEARCH_TOOL dict still parses.
- Cross-chapter follow-ups: none

## Chapter 57 — Architecture for Millions of Documents  (reviewer: chapters 55–59)

- No MUST or SHOULD issues found. Every sizing number was recomputed and is right. 100M × 3,072 B = 307.2 GB ("about 307 GB"), and 1M × 3,072 B = 3.07 GB ("about 3 GB per million chunks"). The 323 GB HNSW figure is 307.2 GB of vectors plus about 12.8 GB of layer-0 links at M = 16, and it matches Ch 31:218 and Ch 60:101. 12.5M documents × 8 chunks = 100M. Small-model embedding: 100M / 5,000 per s = 5.6 h and 100M / 2,000 per s = 13.9 h, so "~6–14 hours" holds. A 7B model: 100M / 300 per s = 3.9 days and 100M / 100 per s = 11.6 days, so "~4–12 days" holds. Extraction at 50–200 docs/s per worker takes 17–69 h per worker, which fits "hours, parallelisable". 40,000 pages × 8 = 320,000 chunks, so "a few hundred thousand", which at 2,000/s is 2.7 minutes ("a few minutes"). That also agrees with Ch 58:183 ("about 320,000 chunks"). The three sections (ingestion, source of truth, serving) and their jobs are clear and match the diagram. The ID formulas match the code: `vector_id(ch.chunk_id, model.name, model_version, model.prefix_scheme)`. The Ninja note on content-keyed vector ids is logically right, because `chunk_id` includes `chunker_version`. The cover list matches all six headings. Cross-references (Ch 5, 24–27, 31, 40, 41, 52, 53, 58, 60, 62, 63) match OUTLINE. What's next links 58.
- **[NIT]** The chapter says both that embedding is the most expensive stage and that other stages dominate, without saying it means money in one place and time in the other. Quote: "Embedding is usually the most expensive stage." and "The idempotent embedding stage, which is where most of the cost sits". Compare "parsing is often the slowest step", "No fetching and no extraction, which were the slowest stages" and "**OCR and LLM enrichment are often the true long poles**, not embedding". LLM contextual enrichment of 100M chunks usually costs more in money than embedding them. Fix: "Embedding is usually the most expensive stage in GPU money, unless we enrich chunks with an LLM. Extraction and OCR are usually the slowest in wall-clock time."
- **[NIT]** Two terms in teaching sections are never explained. "enrich" appears in the bold pipeline definition ("fetch, extract, chunk, enrich, embed") and "long pole" in "Throughput arithmetic". Fix: "enrich (add metadata and, optionally, an LLM-written context line, Chapters 50–51)" and "the long pole (the slowest stage, which sets the total time)".
- **[NIT]** The build row contradicts the chapter's own claim that the index outgrows one machine. Quote: "| HNSW build, 100M × 768-d | many-core machine | hours |". Building one 100M × 768-d HNSW graph needs the 323 GB of RAM from the first section on a single box, and "hours" is unverified at 768-d with efConstruction = 200. Fix: "| HNSW build, 100M × 768-d | one machine per shard, in parallel (Chapter 58) | hours |".
- **[NIT]** Wrong chapter for int8. Quote: "HNSW with int8 or TurboQuant compression (Chapter 27)". int8 scalar quantization is Chapter 26. Fix: "int8 (Chapter 26) or TurboQuant (Chapter 27) compression".
- Style: clean. There are no semicolons or em-dashes in prose, the restaurant analogy has its mapping lines, the bold-equation definitions cover ingestion pipeline, CDC, reference design and idempotent, and the example goes "without the principles → with the principles".
- FIXES.md items for this chapter: all applied. CDC is now expanded as a bold equation ("**CDC = change-data-capture.**") placed just before the diagram. That is better than the parenthetical the spec asked for, and it covers the diagram's bare "(CDC, webhooks, crawls)". The replicas gloss "(copies of an index that share the query load, Chapter 58)" is present word for word.
- Code: 1 block, runs clean with a stub store and model. First call embeds 10 and a repeat call embeds 0 (idempotent). One new chunk embeds 1. Bumping `model_version` re-embeds 10. The stored dtype is float32. Minor: the "not decoration" checks use `assert`, which Python strips under `python -O`. Production code should use `if …: raise ValueError(...)`, and one clause in the prose would say so.

#### Fix status (applied 2026-09-16)

- [NIT] "Most expensive stage" vs "slowest stages" ambiguity — fixed: "most expensive stage in GPU money, unless we enrich chunks with an LLM. Extraction and OCR are usually the slowest in wall-clock time."; Under the hood lead-in now says "most of the GPU cost".
- [NIT] "enrich" and "long pole" undefined — fixed: bold "enrich" sentence after the pipeline definition (Chapters 50–51), gloss on "long poles" in Throughput arithmetic.
- [NIT] HNSW build row implies one machine — fixed ("one machine per shard, in parallel (Chapter 58)")
- [NIT] int8 cited as Chapter 27 — fixed (int8 Chapter 26, TurboQuant Chapter 27)
- [NIT] (Code note) `assert` stripped under `python -O` — fixed: one sentence in the prose after the code says to raise ValueError in production.
- Code: no code changed
- Cross-chapter follow-ups: none
- [Lead, verification pass] one overlong paragraph rewrapped, no words changed.

## Chapter 58 — Sharding, Replication, and Routing  (reviewer: chapters 55–59)

- Fan-out and sizing arithmetic recomputed and all correct. For 1 − 0.99^N at N = 1/10/20/40/70/100 I get 1.0/9.6/18.2/33.1/50.5/63.4%, which matches the table. 0.99^40 = 0.669 ("0.67"). The median of the maximum of 40 shards is the shard's 0.5^(1/40) = 98.3rd percentile ("98th percentile"). The system median equals the shard p99 at N = ln 0.5 / ln 0.99 = 69.0 ("about 70", used in both places). 40 × 2.5M = 100M. int8: 76.8 GB of codes + 12.8 GB of links ≈ 90 GB, and Ch 60 says ~92 GB. Acme: 320,000 / 100M = 0.32%, and 320,000 / 40 = 8,000 per shard. A random post-filter keeps about 0.0032 × 100 × 40 ≈ 13 Acme chunks in expectation ("a handful"). Hedging after p95 duplicates about 5% of requests. Doubling replicas roughly doubles QPS. p99 is defined in Ch 18 and k-means in Ch 23. "IVF as a shard router in front of graph indexes" exists at Ch 23:525, and time-partitioned indexes at Ch 59:353. The cover list matches all eight headings. What's next links 59.
- **[MUST]** Plain modulo hashing makes adding shards hard, not simple. Quote: "Assign each vector to `hash(doc_id) % N`" and, under Strengths, "simple to add shards". When N changes, `hash % N` sends almost every document to a different shard. Going from 40 to 41 shards moves 97.6% of vectors (checked on 2M random hashes: only 1/41 stay put). Going from 40 to 80 moves 50%. Fix: change the strength to "easy to add shards *if* we use consistent hashing or a fixed set of virtual buckets (for example 4,096 buckets mapped to shards), so adding a shard moves only about 1/N of the data". Also add a sentence to Strategy 1: "A plain `hash % N` would move almost every vector when N changes."
- **[SHOULD]** "Rarely" understates how often a missing shard changes the result. Quote: "missing one shard of forty rarely changes the top 10." Under random sharding each top-10 item sits on any given shard with probability 1/40. So a missing shard holds at least one of the top 10 in 1 − (39/40)^10 = 22% of queries, about one in five. On average it loses 0.25 of the 10 items, a 2.5% dent in recall@10. Fix: "missing one shard of forty drops one of the top 10 in about one query in five, and costs about 2.5% of recall@10 on average. Log coverage and alert on it."
- **[SHOULD]** In the Under-the-hood code, cancelling a slow shard does not stop its replica searches. Quote: "p.cancel()                                   # accept partial results". Cancelling the outer `search_shard` task raises `CancelledError` inside `asyncio.wait`, but `asyncio.wait` never cancels the `primary`/`backup` tasks it was waiting on. I ran it with a shard whose replicas each take 300 ms against the 80 ms deadline: after `scatter_gather` had returned, 2 replica searches were still running, and both later ran to completion. The load the deadline is meant to shed keeps hitting the slow shard. Fix: in `search_shard`, wrap the body in `try: … finally: for t in (primary, backup): t.cancel()` (initialise `backup = None` and skip it when it is None), or use `asyncio.TaskGroup`.
- **[NIT]** The analogy's numbers do not fit together. Quote: "We open twenty branches" and "A patron from organisation 42 only ever phones branch 42." There is no branch 42 among twenty. Fix: "organisation 7 only ever phones branch 7", or "phones its own branch".
- **[NIT]** An overbroad sentence could be misread as contradicting "Boundary misses" in Strategy 2. Quote: "So the global top-k is always inside the merged lists, up to each shard's ANN approximation. This holds for any way of splitting." This holds only when every shard is asked. Fix: "This holds for any way of splitting, as long as we ask every shard."
- **[NIT]** The rescore depth disagrees with earlier chapters. Quote: "Collect ~2–5× the final `k` across shards". Ch 25's table says "Rescore depth | 10–20 × final k" for PQ, and Ch 66 gives ~100× for binary. 2–5× suits int8 only. Fix: "~2–5× for int8, 10–20× for PQ (Chapter 25), ~100× for binary (Chapter 26)".
- Style: clean. There are no semicolons or em-dashes in the prose, the branch analogy has explicit mapping sentences, the example goes wrong answer (random + post-filter) then right answer (tenant shard), and a "When to use which one" table is present.
- FIXES.md items for this chapter: all applied ("hits at least one shard's p99 on about a third of requests" is now present and agrees with the 0.67 line, the table and Key takeaways).
- Code: 1 block, runs clean with stub replicas. A fast shard is answered by its primary. A shard with a 200 ms primary and a 10 ms backup is won by the hedge at about 25 ms. A shard at 300 ms/300 ms is dropped at the 80 ms deadline, so coverage = 0.67. Scatter-gather and hedging do what the prose says, apart from the orphaned-task leak above. Minor: `shard.replicas[1]` assumes at least 2 replicas, and there is no de-duplication, which matters if boundary vectors are copied into two shards as Strategy 2 suggests.

#### Fix status (applied 2026-09-16)

- [MUST] `hash % N` called simple to add shards — fixed: Strategy 1 now says a plain `hash % N` moves almost every vector when N changes (about 98% going 40 → 41), names fixed virtual buckets (4,096, bucket-to-shard table, about 1/N moves) and consistent hashing; Strengths line and comparison-table row updated to match.
- [SHOULD] "Missing one shard rarely changes the top 10" — fixed: about one query in five loses one of its top 10, about 2.5% of recall@10 on average, log coverage and alert.
- [SHOULD] Deadline cancel leaves replica searches running — fixed: `search_shard` cancels primary/backup in a `finally`, and `scatter_gather` awaits the cancelled shard tasks so nothing is still running when it returns. Prose after the code explains it.
- [NIT] Branch 42 among twenty branches — fixed (organisation 7 / branch 7)
- [NIT] "Holds for any way of splitting" overbroad — fixed ("as long as we ask every shard")
- [NIT] Rescore depth 2–5× only fits int8 — fixed (2–5× int8, 10–20× PQ Ch 25, about 100× binary Ch 26)
- [NIT] (Code note) no de-duplication of boundary vectors copied into two shards — fixed: merge keys hits by id. `replicas[1]` assumption left as is (every shard in the chapter has replicas for hedging).
- Code: scatter-gather block changed. Reproduced the bug first with asyncio and stub replicas (original: after return, slow0 and slow1 still running and both later finished). Fixed block extracted from the chapter and run: 81 ms, coverage 0.67, fast shard served by primary, hedge wins on the slow-primary shard, nothing running after return, slow0/slow1/hedge0 cancelled, duplicate id merged once. Also simulated: 40 → 41 with `% N` moves 97.6%, with 4,096 buckets 2.4%; missing shard hits top 10 in 22.4% of queries, loses 2.5% on average.
- Cross-chapter follow-ups: 66-field-manual.md line ~219 sharding table row "Random (hash) | `hash(doc_id) % N`" should become "`hash(doc_id)` → virtual bucket → shard" (or mention consistent hashing). 66-field-manual.md line ~205 "Over-fetch across shards before rescoring | ~2–5× final k | 58" should say "~2–5× for int8, 10–20× for PQ, ~100× for binary".
- [Lead follow-up from Ch 60 fixer] int8 at 100M is ~928 B → ~93 GB in Ch 60 — fixed "about 92 GB" → "about 93 GB" in two places.

## Chapter 59 — Freshness and Index Maintenance  (reviewer: chapters 55–59)

- No MUST issues found. The delta + tombstone + compaction pattern is described correctly and matches Ch 31 (main HNSW + flat delta, tombstone set over both, nightly fold-in) and Ch 32 (Fresh-DiskANN). The write path tombstones first and then inserts, the read path over-fetches, drops tombstones and merges, and compaction rebuilds from the source of truth, verifies, replays after the snapshot and swaps atomically. That is consistent across the diagram, Phases 1–3, the code and Key takeaways. Numbers checked. "Twenty thousand vectors scan in about a millisecond": 20,000 × 3,072 B = 61 MB, which at Ch 17's 20–50 GB/s is 1.2–3 ms on one core and under 1 ms on several, and I measured 0.24 ms with NumPy. "Brute force over a million-vector delta is no longer cheap" fits Ch 17's ~20 ms. The staleness-budget and maintenance-calendar tables are copied word for word into Ch 66 (lines 241–260). The scenario reads correctly: the example is clearly labelled imaginary ("Everywhere else in the book, SSO stays on Pro"), tombstoning drops the 0.91 and 0.80 hits so the new page 212 (0.84) ranks first, and the wrong answer comes before the right one. Cross-references (Ch 17, 23, 25, 31, 32, 33, 54, 57, 62, 63) match OUTLINE. The cover list matches all eight headings. What's next links 60.
- **[SHOULD]** `FreshIndex.upsert` makes a reverted page vanish from search until the next compaction. Quote: "self.tombstones.update(chunk_ids_to_remove)\n        self.delta.add(new_chunks.vectors, new_chunks.ids)". The chapter says "Chunk ids include the content hash", following Ch 57's `chunk_id = hash(doc_id, doc_content_hash, chunker_version, chunk_index)`. If a writer undoes an edit, the restored text hashes to the same ids that are already tombstoned. `upsert` adds them to the delta, and `search` then filters them out. I ran it: after changing page 212 on 1 October and reverting it on 2 October, neither version of page 212 could be found. Fix: add `self.tombstones.difference_update(new_chunks.ids)` after the tombstone update, and de-duplicate ids when merging main and delta hits. Alternatively, key tombstones by (id, version).
- **[NIT]** The over-fetch reduces the risk of returning fewer than k results but does not "make sure" k survive. Quote: "so we request more than `k` to make sure `k` survive." `over = k + min(len(self.tombstones), 4 * k)` fetches at most 5k. Tombstones cluster exactly where queries land, because an updated document's old chunks sit next to questions about that document. With 45 tombstoned ids at the top of the ranking and k = 10, `search` returned 5 results. Fix: "to make it likely that `k` survive. If fewer do, search again with a larger `over`."
- **[NIT]** Deleting page 1,140 outright does not fit what the page is. Quote: "**Page 1,140**, the rule about the Security add-on on Pro, no longer applies, so it is deleted." Everywhere else (for example Ch 55 and Ch 52), page 1,140 is the long Security add-on page, "mostly about audit logs and data retention", with one SSO sentence. Removing the SSO rule would be an update, not a deletion. Fix: "Acme retires the Security add-on, so page 1,140 is deleted", or keep the page and make it an update.
- **[NIT]** The chapter credits the whole product's corpus to Acme. Quote: "A crawl of Acme's 12.5 million unchanged documents does 12.5 million hash comparisons and no embedding." 12.5 million is all 5,000 tenants' documents (Ch 57). Ch 58 carefully says Acme's own library is 40,000 pages. The crawl also still re-fetches every document. Fix: "A crawl of the product's 12.5 million documents, if none changed, re-fetches and hashes each one but embeds nothing."
- **[NIT]** The maintenance calendar lists the recall check at two frequencies. Quote: "| Continuous | Monitor delta size, tombstone ratio, index recall vs brute force |" and "| Nightly | Recall check on sampled queries; eval-set regression run |". The prose then says "The nightly check is what turns an invisible trend into a graph". Fix: drop "index recall vs brute force" from the Continuous row (and from Ch 66:259), or make the Continuous row "recall on a small live sample".
- Style: teaching prose is clean (newspaper analogy with explicit mapping, Phase/Step lines before the code, wrong answer then right answer, "We must use …" close). The remaining semicolons are in the frontmatter summary ("Documents change every minute; indexes are expensive to modify") and in three calendar table cells.
- FIXES.md items for this chapter: all applied. The [NICE] LSM gloss is present ("log-structured merge trees: storage where new data lands in small fresh segments that are merged into bigger ones later"), and Lucene is glossed as "the search library inside Elasticsearch".
- Code: 1 block, runs with a flat stub main index and a content-hash id scheme. After the 1 October upsert and delete, the new page 212 ranks first and both stale ids are gone. `compact` resets the delta and tombstones as described, with the missing replay called out in the prose. Two edge cases fail as reported above: the revert, and fewer than k results.

#### Fix status (applied 2026-09-16)

- [SHOULD] `FreshIndex.upsert` makes a reverted page vanish until compaction — fixed: `upsert` now calls `self.tombstones.difference_update(new_chunks.ids)`, `search` keeps one hit per id, Phase 1 Step 5 and Phase 2 Step 3 say the same, and a paragraph after the code walks through the 2 October revert.
- [NIT] Over-fetch does not "make sure" k survive — fixed ("make it likely", "If fewer do, search again with a larger `over`")
- [NIT] Deleting page 1,140 does not fit the long add-on page — fixed: "Page 1,140, the Security add-on page, is deleted, because Acme retires the add-on."
- [NIT] 12.5 million documents credited to Acme, crawl still re-fetches — fixed (product's 12.5 million documents, re-fetches and hashes each, embeds nothing)
- [NIT] Recall check listed at two frequencies — fixed: Continuous row is now "Monitor delta size and tombstone ratio"
- [NIT] (Section 0) prose semicolon — fixed: frontmatter summary "Documents change every minute, and indexes are…". Three semicolons inside maintenance-calendar table cells left as is (table, not prose, and copied word for word in Ch 66).
- Code: FreshIndex block changed. Reproduced first with a NumPy flat main index, flat delta and content-hash chunk ids: original code, after the 1 October change and a 2 October revert, page 212 was not findable. Fixed block extracted from the chapter and run: after the revert page 212 ranks first (0.91), no duplicate ids in the top 10 (upsert fix without the dedup gave 1 duplicate), old page 1,140 stays gone, compaction clears delta and tombstones and keeps the reverted page. Clustered-tombstone edge case (45 tombstones, k = 10) still returns 5, matching the new "make it likely / search again" wording.
- Cross-chapter follow-ups: 66-field-manual.md line ~259 maintenance-calendar Continuous row should drop "index recall vs brute force" to match Ch 59 ("Monitor delta size and tombstone ratio").

## Chapter 60 — Cost Engineering  (reviewer: chapters 60–63)

- No MUST issues found. I recomputed every figure in Python and all are correct. Memory table: 3,072 + 160 = 3,232 B → 323.2 GB; fp16 1,696 B → 170 GB (0.52×); MRL-256 + int8 416 B → 42 GB; TurboQuant 2-bit 192 + 4 (norm) + 160 = 356 B → 36 GB; **binary + rescore 96 + 160 = 256 B → 25.6 GB RAM, plus 307.2 GB SSD, so the "~25 GB RAM" row that Ch 15, 26, 31 and 66 rely on holds**; DiskANN 48 B → 4.8 GB (+ 3,072 + 256 B on SSD ≈ 333 GB); IVF-PQ 96 + 8 B → 10.4 GB. Other checks: ×3 replicas = 969.6 GB; 1B vectors ×3 at $5 = $48,480/month and $581,760/year; int8 at 1B = 928 GB. GPU-hours 6.9 / 18.5 / 138.9, and 140/7 = 20×. Acme bill $4,848 + $6.14 + $62,500 + $4,000 = $71,354. int8 fix $67,898, saving $3,456 = 4.84%. Context fix saves $45,000 = 63.1%. Both together $22,898, 67.9% lower, 1.43 ¢ → 0.458 ¢ per query. Index share 6.8% vs LLM 93.2%. Break-even ≈ 364,500 queries/month. All match the prose. Ch 66's copy of the formula, GPU-hours and lever table is word-for-word the same as this chapter. Ch 57's 323 GB / 307 GB and Ch 32's ~900 GB int8 / ~50 GB DiskANN at 1B match too.
- **[SHOULD]** The lever table says quantizing the index saves "4–30× RAM", but the chapter's own memory table shows that the whole HNSW index shrinks only 3.5× (int8), 9× (TurboQuant 2-bit) or 13× (binary). The graph links are not compressed. 4–32× applies to the vector bytes alone. Quote: "| **Quantize the index** (int8 → TurboQuant/binary + rescore) | 4–30× RAM |". Fix: "3.5–13× HNSW RAM (4–32× on the vector bytes alone)". Make the same change in Ch 66's copy of the table.
- **[SHOULD]** The ranking "by typical impact, largest first" puts quantization second, above prompt caching and routing. But the chapter's worked example shows quantization saving under 5%, and the next paragraph says the LLM levers come first "when query volume is moderate or high, which is most RAG products". The table and the prose contradict each other. Quote: "By typical impact, largest first:". Fix: move "Quantize the index" below "Prompt caching" and "Route by query complexity", or retitle the list "largest impact for the least work" and say the order flips for large-corpus, low-traffic systems. Apply the same fix to Ch 66 ("The levers, largest typical impact first").
- **[SHOULD]** The price ladder "ten times at each step" does not match list prices, and it does not match the chapter's own toy prices. RAM at $5 against object storage at $0.02 is 250×, not 100×. At AWS us-east-1 list prices (unverified for other providers), instance RAM costs about $4–6 per GB-month, EBS gp3 SSD about $0.08 and S3 Standard about $0.023. That makes RAM:SSD roughly 50× and SSD:object roughly 3–4×. The chapter calls this "The single most important relationship in this chapter". Quote: "**RAM costs roughly ten times more per GB than NVMe SSD.**". Fix: "RAM costs tens of times more per GB than SSD, and SSD a few times more than object storage. End to end, RAM is a hundred or more times the price of object storage." Update the Key takeaways line "roughly ten times at each step" and Ch 66's "(each roughly an order of magnitude cheaper per GB)" to match.
- **[NIT]** The formula is labelled "cost per query", but it covers only the LLM tokens. The chapter's own definition also includes the index, embedding and GPU share, and Ch 66 labels it "Cost per query (LLM)". The worked line also drops the division and then says "per million". Quote: "= 13,000 × P_in + 400 × P_out    per million, per query". Fix: label it "LLM cost per query ≈" and write "= (13,000 × P_in + 400 × P_out) / 1,000,000 per query".
- **[NIT]** The int8 row says "~920" bytes, but 768 + 160 = 928 B (0.29×). The worked example uses 92.8 GB (278 GB for three replicas), and so does the code output. Quote: "| HNSW, int8 | ~920 | ~92 GB | 0.28× |". Fix: "~930 | ~93 GB | 0.29×", or leave the rounding and accept the 0.01 gap.
- **[NIT]** Ch 15 and Ch 26 both say "Chapter 60 itemises" the ~25 GB binary figure, but the row gives only a total ("~250 RAM"). Quote: "| Binary + rescore from SSD | ~250 RAM + 307 GB SSD |". Fix: add to "How to read one row": "Binary is 96 bytes of code plus the same ~160 bytes of graph, about 256 bytes, so about 25 GB."
- **[NIT]** "~20×" is measured against the small model, but the sentence reads as if it compares against any upgrade. Against the base model it is ~7×. Quote: "It is ~20× the re-embedding bill, every time." Fix: "It is ~20× a small model's re-embedding bill (~7× a base model's), every time."
- Style: clean. No semicolons or em-dashes in the teaching sections. Section titles are plain. The chapter shows a wrong answer ("obvious fix") and then the right one ("measured fix"). "you" is used only for advice.
- FIXES.md items for this chapter: all applied. The only item was "OK arithmetically, align Ch 31", and Ch 31 now says its rows "roughly match Chapter 60's cost table" (~90 / ~25 GB).
- Code: 1 block (`monthly_cost`). It runs, and the output matches the printed comments exactly: 323.2/4854/66500/0.93, 92.8/1398/0.98, 35.2/534/0.99. The only difference is whitespace alignment in the comments.

#### Fix status (applied 2026-09-16)

- [SHOULD] Quantize lever "4–30× RAM" overstates whole-index saving — fixed (D4 row: "3.5–13× HNSW RAM (4–32× on vector bytes alone)")
- [SHOULD] Lever ranking contradicts worked example and prose — fixed (D4 order and intro line; "top of that table" paragraph, line on quantization in memory section, and Key takeaway rewritten to name the LLM levers as largest)
- [SHOULD] Price ladder "ten times at each step" wrong — fixed (D5 wording in body and Key takeaways)
- [NIT] Formula labelled "cost per query" but LLM-only, division dropped — fixed
- [NIT] int8 row ~920 / ~92 GB / 0.28× — fixed ("~930 | ~93 GB | 0.29×", Key takeaway now ~93 GB)
- [NIT] Binary ~25 GB not itemised despite Ch 15/26 pointers — fixed (added 96 + ~160 = ~256 bytes sentence to "How to read one row")
- [NIT] "~20×" re-embedding comparison ambiguous — fixed
- [D6] DiskANN SSD ~330 GB → ~410 GB (one 4,096-byte block per node) — fixed
- Code: no code changed
- Cross-chapter follow-ups:
  - 66-field-manual.md: mirror D4 table/intro and D5 ladder line; formula block (lines ~276–279) should read "LLM cost per query ≈ ..." and "= (13,000 × P_in + 400 × P_out) / 1,000,000    per query"; "~92 GB" (lines ~90, ~239) → "~93 GB"; any DiskANN 100M SSD figure → ~410 GB.
  - 58-sharding-and-routing.md lines ~70 and ~310: "about 92 GB, Chapter 60" → "about 93 GB, Chapter 60".

## Chapter 61 — Image-Heavy RAG, End to End  (reviewer: chapters 60–63)

- **[MUST]** Page 3,507 in this chapter contradicts page 3,507 in Ch 44, 45 and 46, and the contradiction breaks the point of the worked example. In Ch 44 the page is headed *"Schedule B: Pricing for Globex Corporation"*, and its table has the columns Item | Price | Terms. One row reads "Pro licence | $15 / seat | 120 seats, annual", and OCR misreads SSO as `S5O`. Ch 61 differs in four ways.
  (a) Its OCR output shows a different table: a Feature | Basic | Pro | Enterprise plan comparison with an "Audit log" row, no seat count, and SSO misread as `SS0`.
  (b) It claims the seat count exists only on page 3,508. Quote: "Without it, the Scholar would have known the rule but not whether Globex met it". In Ch 44, page 3,507 already says 120 seats.
  (c) It says BM25 cannot see "Globex" on 3,507 ("Page 3,507 appears only in the FDE list"). But the page's heading contains "Globex", and Ch 61 itself says OCR reads clean printed text well. So BM25 would also list the page.
  Quote: "Feature Basic Pro Enterprise SS0 not included $0, included". Fix:
  - Replace the OCR block with Ch 44's Stage 4 output ("Item Pro licence Security add-on S5O Price $15 / seat $4 / seat $0 / Terms 120 seats, annual optional included (Pro, 50+ seats)"), and change "SS0" to "S5O" three times (the OCR block, Step 4, and "Why still run OCR?").
  - In Step 4, have BM25 also return 3,507 through its heading, for example at rank 9. The RRF score then becomes 1/74 + 1/69 ≈ 0.0280, still easily in the top 150.
  - Give page 3,508 something 3,507 lacks, such as the signature page with the June 2026 effective date, or drop the claim that neighbour expansion was what supplied the seat count.
  The worked answer itself ("SSO is included from 50 seats", 120 ≥ 50) is correct and agrees with page 1,140's rule (the add-on is needed only under 50 seats).
- **[NIT]** The last takeaway contradicts the body and is wrong on its own terms. A new encoder makes the stored vectors obsolete, so every index must be rebuilt. The body correctly says "a re-encode job, not a re-architecture". Quote: "so the next model upgrade is a re-encode, not a rebuild." Fix: "Keep page images, so the next model upgrade is a re-encode, not a re-architecture. Keep full-precision vectors, so a new index format is a rebuild, not a re-encode."
- I recomputed and confirmed every number. 1M × 750 × 128 × 4 B = 384 GB → pooled 1M × 250 × 512 B = 128 GB. Codes 1M × 750 × 36 B = 27 GB → 9 GB. TurboQuant 32 + 4 B = 36 B → 9 GB. FDE 4,000 × 4 B = 16 KB → PQ 1,000 × 1 B = 1 KB (16×) → ~1 GB + graph. Hot set 1–2 + 9 + 5–15 = 15–26 GB. Cold 100 + 128 ≈ 230 GB. Exhaustive MaxSim 1M × 250 × 20 = 5 × 10⁹. GPU-hours 1M ÷ 10/s = 27.8 h and 1M ÷ 2/s = 138.9 h, so "30–140" agrees with Ch 47. Pages per hour 7,200–36,000. RRF 1/74 = 0.0135, and the worst case for a rank-14 FDE-only page is fused rank ≤ 114, so "comfortably inside the top 150" holds. Latency 20+10+0+10 = 40 to 50+30+1+40 ≈ 120 ms, and with a rewrite 140–420 ms. 0.9⁶ = 0.531. Cap 768 × 28² = 602,112 px → A4 at 78.9 DPI ≈ 80 DPI, matching Ch 47's 644 × 896 / 736-token table. The ColQwen2 ~750 figure, the pooled 250 at 36 B and the 9 GB all match Ch 47. The cross-references I checked all point at the right content: Ch 7 exact terms, 18 p99, 36, 38, 40, 43 VLM, 44 six stages, 46 448 × 448 and heatmaps, Ch 47 neighbour expansion for tables that span two pages (line 457), 52, 55 floor, 59 delta index.
- Style: clean. The chapter uses Phase/Step lines before the diagram and the code, shows a wrong answer and then the right one, and maps every part of the law-firm analogy. There are no semicolons or em-dashes in the teaching sections, and "we" is used throughout.
- FIXES.md items for this chapter: all applied. "The 150 ms SLO holds only if query rewrite is skipped on the hot path" is present, with the two options (route, or background rerank), and the retrieval total is now 40–120 ms, which matches its components. SLO is defined as "service-level objective" in a bold equation with an "In simple words" gloss. The FDE is "~4,000-d" in Step 6, the table and the takeaways. "~750 patch vectors" now matches Ch 47's ColQwen2 section.
- Code: 1 block (`answer`), read only (async pseudo-code over external services). The names, shapes and flow are consistent. Rewriting happens only for `router.is_hard` queries, `gather` awaits two search coroutines, MaxSim is normalised by `len(q_vecs)` (n_tokens) as the prose explains, both returns are `(text, pages)` tuples, and the LLM receives the original `question`.

#### Fix status (applied 2026-09-16)

- [MUST] Page 3,507 contradicts Ch 44–46 (table, seats, BM25, SS0) — fixed (per D1: OCR block is Ch 44's Stage 4 text, every "SS0" → `S5O`, BM25 also returns 3,507 at rank 9 via "Globex" in its heading, RRF recomputed 1/74 + 1/69 ≈ 0.0280 with a note that FDE alone (0.0135) suffices, MaxSim step no longer cites a "Pro column header", page 3,508 is the order form / signature page confirming 120 Pro seats with the June 2026 effective date, Scholar reasoning and "Two details" paragraph now credit 3,508 with showing the contract is signed and current, answer kept as "because Globex has 120 seats and SSO is included from 50 seats [pp. 3,507 and 3,508]"; also "words are unreadable to OCR" → "OCR has scrambled its table")
- [NIT] Last takeaway says re-encode "not a rebuild" — fixed
- Code: no code changed (no latency figures changed; RRF arithmetic checked in Python)
- Cross-chapter follow-ups: none (Ch 52 "pages 3,507 and 3,508 … Pro with 120 seats" and Ch 64 "June 2026" already agree)

## Chapter 62 — Multi-Tenancy, Privacy, and Embedding Inversion  (reviewer: chapters 60–63)

- **[SHOULD]** The chapter names "noisy neighbours are contained" as a benefit of namespaces. But its own recommended layout packs small tenants "onto shared machines, each still in its own namespace", and there they still share CPU, RAM and disk. The takeaway also attributes containment to dedicated shards, not namespaces. Quote: "1. **Noisy neighbours are contained.** Globex's re-import touches Globex's index, not Initech's." Fix: "1. **Noisy neighbours are partly contained.** Globex's re-import rebuilds only Globex's index, so Initech's index is never locked or rebuilt. On a shared machine they still share CPU and memory, so large tenants get their own shards (Chapter 58) and small ones get per-tenant rate limits." Soften the matching "Noisy neighbours are contained." bullet under Advantages the same way.
- **[SHOULD]** No chapter actually designs deletion, and the code hides deleted vectors instead of removing them. Ch 59 hands off to this chapter: "Design deletion as a first-class, audited path (Chapter 62)". This chapter has one bullet pointing back to Ch 59 and never uses the word tombstone. Meanwhile `SecureRetriever` treats deletion as a `"status": {"neq": "deleted"}` filter, which leaves the vector physically in the index, where the inversion section says it is still readable. A novice will take the filter as deletion. Quote: "**Deleting a document means deleting its vectors** from every index, delta, replica, cache, backup and the source-of-truth vector store (Chapter 59)." Fix: add a short Phase/Step list for a deletion request:
  1. Tombstone the ids synchronously, so they are hidden within minutes (Chapter 59).
  2. Purge the result and embedding caches.
  3. Delete from the delta and the source-of-truth store.
  4. Run compaction (or a rebuild) before the legal deadline, so the vectors physically leave the main index on every replica.
  5. Let backups expire on a documented retention period.
  6. Write an audit entry.
  Also comment the `status` filter "hides tombstoned items; compaction removes them".
- **[NIT]** The vec2text passage is accurate but vague, and it leaves out the attacker's precondition. Morris et al., "Text Embeddings Reveal (Almost) As Much As Text" (EMNLP 2023), recover **92% of 32-token inputs exactly** from GTR-base embeddings (BLEU 97.3), 60.9% exactly from OpenAI ada-002, and recover full names from clinical notes. The method re-embeds each guess, so the attacker needs query access to the same embedding model. Quote: "On short texts, published reconstructions recover a large share of the original wording exactly." Fix: "On 32-token texts, vec2text (Morris et al., 2023) recovered 92% of inputs word for word. It needs query access to the same embedding model, which is easy when that model is a public API."
- **[NIT]** The mitigations sentence says every mitigation costs retrieval quality, which is wrong for a secret orthogonal rotation. A rotation preserves every dot product, so retrieval is unchanged. Its weakness is that enough known text–vector pairs let an attacker solve for it. Quote: "They raise the cost of inversion, but they also hurt retrieval quality". Fix: "Noise costs retrieval quality. A secret rotation costs none, but it stops protecting once an attacker collects enough known text–vector pairs to undo it. Neither replaces access control."
- **[NIT]** The cache-key expression looks like Python but would raise: `hash()` takes one argument, and a list is unhashable. Quote: "`hash(query, tenant_id, sorted(user_groups), corpus_version)`". Fix: "`sha256` of `(query, tenant_id, tuple(sorted(user_groups)), corpus_version)`". Python's built-in `hash` is also salted per process, so it is unsuitable for a shared cache.
- Verified: 100M ÷ 5,000 = 20,000 chunks per average tenant, so Initech's "roughly 20,000" is plausible. Pass rate 20,000/100M = 0.02%. 20,000 × 3,072 B = 61.4 MB, and Ch 59 says "Twenty thousand vectors scan in about a millisecond". Ch 33's table sends an allowed set under ~50k to "Pre-filter + brute force", which matches "fall back to pre-filtering". "The widget bug lived in the fourth row" is correct ("Filter passed by application code"). The injection example's correct answer matches pages 212 and 1,140. Ch 51 defines ACL. Ch 53 covers delimiters against prompt injection. Ch 58 has tenant sharding. Ch 59 has the synchronous deletion path and the "Hourly incremental crawl" row. Ch 16 and Ch 56 are cited correctly. All seven "We will cover" bullets match the headings.
- Style: clean in the teaching prose (no semicolons, and the "In simple words" glosses are present). The 5 em-dashes are all in the isolation table and the code comments, and are already counted in §0.
- FIXES.md items for this chapter: all applied ("noisy neighbour" is defined in bold: "one tenant's heavy load slowing down another tenant's queries", with the Globex 9 a.m. re-import example).
- Code: 2 blocks. The post-filter vs pre-filter snippet reads clean. I ran `SecureRetriever` with mock auth, router and index: it routes to the principal's tenant index, and a caller passing `acl={"any_of": ["*"]}, status={"eq": "deleted"}` is overridden by the mandatory filter while the harmless `lang` filter is kept. So "mandatory wins on conflict" is correct.

#### Fix status (applied 2026-09-16)

- [SHOULD] "Noisy neighbours are contained" overclaims on shared machines — fixed (benefit 1 uses the suggested "partly contained" text, Advantages bullet softened, Key takeaway adds per-tenant rate limits for small tenants)
- [SHOULD] No deletion design, status filter only hides vectors — fixed (per D8: six-step deletion list with tombstone gloss and Chapter 59 citations added after the mitigations paragraph, plus an "In simple words"; `status` filter commented "hides tombstoned items; compaction removes them" and Under-the-hood Step 3 says the same; Key takeaway adds "A filter only hides them. Tombstone at once, then compact before the legal deadline"; benefit 2 notes caches and backups still follow the deletion path)
- [NIT] vec2text passage vague, omits attacker precondition — fixed (92% of 32-token inputs word for word, Morris et al. 2023, needs query access to the same model)
- [NIT] Mitigations sentence wrong for secret rotation — fixed ("secret transformation" → "secret rotation", noise costs quality, rotation costs none but falls to known text–vector pairs)
- [NIT] Cache-key expression is invalid Python — fixed differently: described as "a stable hash, such as SHA-256, of `(query, tenant_id, tuple(sorted(user_groups)), corpus_version)`", and the first mention became "a hash of the query text alone"
- [§0 NIT] Em-dashes above limit — fixed (the two in the WRONG/RIGHT code comments and the three in the isolation table replaced; teaching sections now have none, only frontmatter keeps its own)
- Code: SecureRetriever block changed only by a comment line; both Python blocks re-parsed with ast (the reviewer's mock-harness run of the logic still applies, no logic changed)
- Cross-chapter follow-ups: none

## Chapter 63 — Model Migration and Drift  (reviewer: chapters 60–63)

- **[SHOULD]** The opening calls old and new vectors "unrelated", and the analogy says an old shelf number "tells us nothing" about the new one. The Ninja note then says a small linear or MLP adapter can "project old vectors approximately into the new space", and the comparison table recommends that adapter as a bridge. If the spaces were truly unrelated, no such mapping could work, so the chapter contradicts itself. Quote: "The new coordinates are not a shifted or rotated copy of the old ones. They are unrelated." Fix: "The new coordinates are not a simple shifted or rotated copy of the old ones, so comparing them directly is meaningless. A mapping learned from thousands of documents embedded by both models can only approximate the translation (see Ninja notes)." Change the last analogy line to: "Knowing a book's old shelf number tells us nothing about its new shelf, unless someone builds a translation table by looking up many books in both libraries."
- **[NIT]** The adapter note mixes up two directions. Projecting *old document vectors* into the new space produces a new index of projected vectors. "Serve new-model queries against the old index" needs the opposite map, from new-model queries into the old space. The query-side map is the one that rescues an unplanned deprecation, when model A can no longer embed queries. Quote: "can project old vectors approximately into the new space ... But it lets us serve new-model queries against the old index". Fix: "can map new-model query vectors approximately into the old space, so the old index keeps serving while the backfill runs (or map old document vectors forward into a temporary new index)."
- **[NIT]** "Rank correlation" is used in Phase 3 Step 2 but defined 35 lines later. The Phase 3 code, which the text calls "the shadow comparison from Phase 3", logs overlap, top ids and latency, but not the rank correlation or errors that Step 2 lists. Quote: "log top-10 overlap, rank correlation, latency and errors." Fix: move the one-line definition of rank correlation into "Three terms first" (it becomes four terms), and either add a comment such as `# rank correlation and errors omitted for brevity` or log them.
- Verified: the six phases are named identically (Prepare, Build, Evaluate, Shadow, Shift, Retire, numbered 0–5) in the prose Phase/Step lists, the pinned code-style block, the Key takeaways ("six phases: prepare, build, evaluate, shadow, shift, retire") and Ch 66's "Model migration, six phases" table. Ch 66's row contents match step for step: dual-write and backfill in Build, recalibration in Evaluate, 1% → 10% → 50% → 100% in Shift. Figures: 19 / ~140 GPU-hours match Ch 60, "about 90 GB" int8 matches Ch 31, and floor 0.60 matches Ch 55's sweep. Toy scores 0.55 and 0.52 fall below 0.60 and pass 0.45. Overlap is (2 + 5)/10 = 0.7. Ch 66's canary tolerance 1e-4 matches. Cross-references all correct: Ch 2 "Vectors from different models are never comparable"; Ch 57 `vector_id = hash(chunk_id, model_name, model_version, prefix_scheme)`; Ch 19 and 54 bootstrap confidence intervals; Ch 23 and 25 stale centroids and codebooks; Ch 31 nightly recall against brute force; Ch 59 quarterly model review in the maintenance calendar. The wrong-then-right answers (mixed index, then stale floor) both use the running SSO question and page 212 / 1,140 rule correctly.
- Style: clean. Blue-green, backfill and dual-write are all defined as bold equations, and every phase is written as Step lines before the code. There are no semicolons in the prose (the two inside the pinned plan block are code), and every "Think of it like" has an explicit mapping.
- FIXES.md items for this chapter: all applied, correctly. [MUST] `latency_ms`: `timed_search` now returns a `SearchResult` dataclass with `.hits` and `.latency_ms`, and every use in `shadow_compare` is consistent (`a.hits`, `a.hits[0].id if a.hits else None`, `a.latency_ms`, and the same for `b`). No list attribute access remains. [SHOULD] CI is expanded as "CI (continuous integration: the automated test run on every code change)".
- Code: 1 block, run with mocks (stub `load`, `log_shadow`, embedder, systems). `check_embedding_path` passes on an identical path and raises "max abs diff 2.70e-01" when normalization is dropped (vectors ×1.3), exactly the failure the prose describes. `shadow_compare` returns 0.7 when the two top-10s share pages 212, 1,140 and five others, logs both latencies, and handles an empty result list (`old_top=None`, overlap 0.0).

#### Fix status (applied 2026-09-16)

- [SHOULD] "Unrelated" old/new vectors contradicts the adapter Ninja note — fixed (opening now says "not a simple shifted or rotated copy… comparing them directly is meaningless", a learned mapping "can only approximate the translation (see Ninja notes)", analogy's last line adds the translation table, and "dot product between unrelated vectors" → "between vectors from two different models")
- [NIT] Adapter note mixes up mapping directions — fixed (query-side map from new-model queries into the old space keeps the old index serving and rescues an unplanned deprecation; forward map of old document vectors fills a temporary new index)
- [NIT] Rank correlation used before defined, and not logged in code — fixed ("Three terms first" → "Four terms first" with a bold definition and an "In simple words" line, later duplicate definition removed, code gets a `# rank correlation and errors (Phase 3, Step 2) omitted for brevity` comment, and the prose says a production version logs them)
- Code: shadow_compare block gained one comment line; extracted and re-run with the mock harness (scratchpad/ch63/run.py) in the venv: canary passes on identical path, raises "max abs diff 2.70e-01" when unnormalized, overlap 0.7, empty-list case returns 0.0 / None
- Cross-chapter follow-ups: none

## Chapter 64 — Vectors as Agent Memory  (reviewer: chapters 64–66 + README/OUTLINE)

No MUST issues found. Scoring table recomputed row by row with weights 0.6 / 0.25 / 0.15 and a 30-day half-life, and all three rows are right: 0.5^(90/30) = 0.125 → 0.432 + 0.031 + 0.135 = **0.598**; 0.5^(30/30) = 0.500 → 0.372 + 0.125 + 0.045 = **0.542**; 0.5^(220/30) = 0.0062 → 0.480 + 0.0016 + 0.135 = **0.617**. The claims built on the table hold: without the importance term the August memory would win (0.497 vs 0.463), so "its high importance lifts it to the top" is true, and the superseded Basic memory would beat the Pro memory if it were active. The code computes the same thing (`0.5 ** ((now - m.last_accessed).days / 30)`, same weights). Run on the table's values, it reproduces 0.598 / 0.542 in the same order. The ages match a mid-September question (June to September ≈ 92 days, early February to September ≈ 220). The Globex story matches README (Basic, 30 seats in February, then Pro, 120 seats since June 2026, and 120 is not under 50, so no add-on). Supersession logic is consistent across prose, diagram, code and table: the `"status": "active"` filter is applied in both `remember` and `recall`, UPDATE supersedes and keeps history, IGNORE stores nothing. k = 5 candidates, 50 retrieval candidates and top 8 match the Phase/Step text. Park et al. 2023 (Generative Agents) does score recency + importance + relevance, with exponential recency decay since last access. Its weights are equal, and the chapter's 0.6 / 0.25 / 0.15 is presented as its own toy choice, which is fine. Semantic / episodic / procedural memory types are standard. Cross-references check out: Ch 4 cosine, Ch 51 superseded pages, 180-day half-life and reference links, Ch 52 conversational rewriting, Ch 56 agents, Ch 60 cost, Ch 62 tenancy (Initech is Ch 62's other tenant). "What's next" links to 65-the-frontier.md.

- **[SHOULD]** The toy similarity values contradict the chapter's own explanation of why the diary failed. The diary section says a turn ranks high because "It mentions SSO". Yet in the table the plan memory, which the text says "does not mention SSO either", gets similarity 0.72, and the superseded "Globex is on Basic, 30 seats" gets 0.80. Both beat the one memory that does mention SSO ("how long SSO setup takes", 0.62). A careful novice will ask why. Quote: "| Globex is on Basic, 30 seats (superseded) | 0.80 | 220 |". Fix: keep the numbers, because the lesson needs them, but add one sentence after the table, e.g. "The question is really about what Globex's plan requires, so plan memories sit closer to it than the SSO-setup memory does. The old Basic memory scores highest because 'Basic, 30 seats' is exactly the situation that needs an add-on."
- **[NIT]** The UPDATE and MERGE branches drop the extracted importance. Only ADD passes `importance=fact.importance`, but the Globex example goes down the UPDATE path, and the table then relies on that memory having importance 0.9. Quote: "memory_store.supersede(decision[\"id\"], new_text=decision[\"text\"])   # keep history". Fix: pass `importance=fact.importance` in the `supersede` and `merge` calls too.
- **[NIT]** The Store step lists the metadata as "created, updated, source, importance, subject and status", but the recall code decays on `m.last_accessed`, which Phase 2 Step 4 refreshes. That field is not in the list. Quote: "metadata: created, updated, source, importance, subject and status (active or superseded)". Fix: add "last used" to the Step 4 list and to the diagram's STORE line.
- **[NIT]** The graph example states the page-212 half-truth the book spends 60 chapters warning about. Quote: "which *uses* the Pro plan, which *includes* SSO". Fix: "which *includes* SSO at 50 seats or more", or "which *includes* SSO (with the Security add-on below 50 seats)".
- Style: no violations in teaching sections. There are no semicolons in prose. The em-dashes counted in section 0 are in the frontmatter and the Ninja-notes table. "Think of it like" has an explicit mapping list, a wrong answer and a right answer are shown, and "We will cover the following" matches all seven H2 headings.
- FIXES.md items for this chapter: none (FIXES has no Chapter 64 block, and no Part 2 item touches it).
- Code: 1 block (external `llm_*`, `memory_index`, `memory_store`, `embed`). The `CONSOLIDATE.format(...)` template runs clean (the escaped braces render as JSON), and `score()` runs on stub memories and reproduces the table. Otherwise it reads clean apart from the importance NIT above.

#### Fix status (applied 2026-09-16)

- [SHOULD] Toy similarity values contradict diary explanation — fixed differently: instead of explaining the old numbers away, changed the table so the SSO-mentioning August memory is most similar (0.70) and both plan memories are equal (0.62), gave the August memory 45 days / importance 0.1, recomputed scores (0.538 / 0.523 / excluded 0.509), added one sentence saying importance lifts the plan memory, and rewrote the superseded-row lesson (0.03 below the correct memory, still in the top 8, so recency alone does not save us).
- [NIT] UPDATE and MERGE drop extracted importance — fixed
- [NIT] "last used" missing from Store metadata list and diagram — fixed
- [NIT] Graph example states the page-212 half-truth — fixed ("includes SSO at 50 seats or more")
- [NIT] Section 0: em-dashes above limit — fixed: the four in the Ninja-notes table became parentheses. The frontmatter `part` dash stays (protected field).
- Code: the `remember` block now passes `importance=fact.importance` to `supersede` and `merge`. Extracted it and ran it with stub LLM/index/store objects: all four ops dispatch correctly, and `recall` scores reproduce the table (0.538, 0.523, 0.509).
- Cross-chapter follow-ups: none (the table values appear nowhere else in the book)
- [Lead, verification pass] Step 4 bullet rewrapped, no words changed.

## Chapter 65 — The Frontier  (reviewer: chapters 64–66 + README/OUTLINE)

Verified against the chapters Ch 65 summarises. The multi-vector cost figures match Ch 35: "~30× raw bytes (~200× vector count)"; 64- or 32-d per token "with modest quality loss"; 32 × 2 bits = 8 B, 200 tokens = 1,600 B ≈ 0.52 of 3,072 B; the crossover "reached in research settings". Pooling 2–3× for a small cost matches Ch 47 ("3× fewer vectors cost a small amount of quality, and 2× pooling is close to free"). MUVERA needing no special index matches Ch 38 ("every index from Part IV works unchanged"). Page 3,507's pricing table, lost by OCR, found on pixels, with images in object storage, matches Ch 61. DiskANN ~5 GB RAM at 100M matches Ch 60's table and Ch 32's ~50 GB at 1B. The RAM ≫ SSD ≫ object storage ladder matches Ch 60 (~10× per step). Corpus drift staling PQ codebooks matches Ch 63. ImageBind-style single anchor matches Ch 48, learned routing as the "fifth idea" matches Ch 20, and HyDE matches Ch 52. Every Chapter N reference matches OUTLINE, and "What's next" links to 66-field-manual.md. The six trends match between the one-paragraph version, the body and the takeaways, as do the four open questions. External claims check out as far as I can tell. MUVERA's paper does report beating PLAID on BEIR. TurboQuant's paper reports beating PQ in its nearest-neighbour experiments. BRIGHT (Su et al., 2024) is a real reasoning-intensive retrieval benchmark on which standard retrievers score low. Generative retrieval's update and delete problems are well documented. "Learned routing is promising in specific settings" is too vague to verify, and is harmless.

- **[MUST]** The chapter says low-bit quantization alone brings a token vector down to "a few bytes", but Ch 37, which it cites, gives 20–36 bytes per token (and 128 × 2 bits of TurboQuant is 32 bytes). Only the next paragraph's 32-d-plus-2-bit combination reaches 8 bytes. Quote: "through residual compression (Chapter 37) or TurboQuant (Chapter 27),\n  brings each token vector down to a few bytes." Fix: "brings each token vector from 512 bytes down to about 20–36 bytes (Chapter 37)", and let the "Put those together" paragraph deliver the 8-byte figure.
- **[NIT]** The trend-5 paragraph puts DiskANN's figure right before "a few hundred milliseconds is acceptable", which reads as though DiskANN is that slow. Ch 32 gives DiskANN 2–10 ms. The hundreds of milliseconds belong to object-storage-backed indexes. Quote: "DiskANN needs about 5 GB of RAM plus SSD. For archives and rarely\nread content, a few hundred milliseconds is acceptable". Fix: "DiskANN needs about 5 GB of RAM plus SSD and answers in a few milliseconds (Chapter 32). Object-storage-backed indexes cut further, at a few hundred milliseconds, which is acceptable for archives and rarely read content."
- **[NIT]** The FIXES rewording calls LSH only "the ancestor", but Ch 38 says MUVERA's regions "come from SimHash. This is exactly the LSH of Chapter 21." LSH is a working part of MUVERA, not just its forebear. Quote: "LSH was the ancestor of the idea. It lost on dense vectors". Fix: "LSH lives on inside MUVERA (its SimHash partitions, Chapter 38). As a stand-alone index it lost on dense vectors (Chapter 21), though its MinHash cousin still wins at deduplication." Make the same change in the takeaway "LSH was their ancestor, and it lost on dense vectors."
- **[NIT]** The takeaways list seven of the eight "What will not change" principles and drop #8. Quote: "The durable principles are compression awareness, coarse-then-refine, hybrid matching,\n  first-stage recall, training-defined similarity, measurement on your own data, and the index as\n  a cache." Fix: append ", and preferring the well-built simple system over the shiny one".
- **[NIT]** Line 19 of the one-paragraph version is an unwrapped ~150-character source line ("belong there too. The second pile holds genuinely open questions, such as…"), while the rest of the file wraps at ~100. Cosmetic only.
- Style: no violations in teaching sections. There are no semicolons or em-dashes, the weather analogy and the Great Library analogy both have explicit mappings, and headings are plain. The "We will cover" list has only four items, which is below FIXES' 5–8 guideline but matches the four H2 headings exactly.
- FIXES.md items for this chapter: applied. The "LSH, MUVERA and TurboQuant" sentence now reads "MUVERA and TurboQuant are randomised, training-free constructions with guarantees", with LSH as the ancestor that lost on dense vectors and MinHash still winning at deduplication. It is consistent with Ch 21's "Why LSH lost for dense retrieval" and no new factual error was introduced (see the LSH-in-MUVERA NIT for a precision improvement). The missing Under the hood / What people get wrong / Ninja notes are already recorded in section 0.
- Code: none (0 blocks). The arithmetic in prose recomputed: 32 × 2 / 8 = 8 B, × 200 = 1,600 B, 1,600 / 3,072 = 0.52 ("about half").

#### Fix status (applied 2026-09-16)

- [MUST] "a few bytes" per token contradicts Ch 37 — fixed ("from 512 bytes down to about 20–36 bytes (Chapter 37)"; the 8-byte figure stays in "Put those together")
- [NIT] DiskANN figure reads as hundreds of milliseconds — fixed (DiskANN "a few milliseconds (Chapter 32)", object storage "tens to hundreds of milliseconds", matching Ch 32)
- [NIT] LSH called only "the ancestor" — fixed in body and Key takeaways (lives on inside MUVERA via SimHash, Ch 38; lost as a stand-alone index, Ch 21)
- [NIT] Takeaways drop principle #8 — fixed
- [NIT] Unwrapped ~150-character line in one-paragraph version — skipped: source line-wrapping only (brief excludes it)
- [SHOULD, section 0] No Under the hood / What people get wrong / Ninja notes — fixed differently: per D12, README now says Chapter 65 is an essay that skips parts of the template (README is in my assignment), so no sections were added.
- [D2] Learned-routing sentence — fixed: now says it is not a fifth index idea and cites Chapter 20 for the partition or cluster family with a trained router.
- [Consistency, D5] "SSD far more than object storage" — fixed to "RAM tens of times more than SSD, SSD a few times more than object storage", matching Ch 60's new ladder.
- [Consistency, D12] "What's next" said Ch 66 "compresses the whole book" — fixed to "collects the book's key decision trees, formulas and defaults in one place".
- Code: no code changed (chapter has no code)
- Cross-chapter follow-ups: none

## Chapter 66 — The Ninja's Field Manual  (reviewer: chapters 64–66 + README/OUTLINE)

I checked every entry against the chapter it cites by opening that chapter.

**Formulas table (all 26 rows agree):**
- **Similarity and IDF.** The dot product, cosine and ‖a−b‖² = 2 − 2a·b match Ch 4. â = a/‖a‖ matches Ch 5. std ≈ 1/√d matches Ch 6. IDF = natural-log log(N/n_t) and the BM25 term match Ch 8.
- **Training and evaluation.** InfoNCE matches Ch 12. System recall, index recall, MRR and nDCG with (2^rel − 1)/log₂(i+1) match Ch 19. SE √(p(1−p)/n) = 0.112 gives ±11, 95% ±22, matching Ch 19 and 54.
- **Hashing and quantization.** 1 − θ/π and 1 − (1 − p^k)^L match Ch 21. PQ m = 96 → 96 B matches Ch 24, and binary 768/8 = 96 B matches Ch 26.
- **Index and multi-vector.** ⌊−ln U · m_L⌋ with m_L = 1/ln M matches Ch 29. MaxSim/Chamfer matches Ch 35 and 36. The FDE dimension B × d_proj × R_reps with B = 2^k_sim matches Ch 38.
- **Fusion, ranking and cost.** Weighted RRF with k = 60 matches Ch 40. MMR matches Ch 41. The recency boost matches Ch 51. 0.99^40 = 0.669 matches Ch 58. The LLM cost formula matches Ch 60. The memory score matches Ch 64.

**Memory arithmetic (all rows agree):**
- **Per-vector table.** 3.072 / 307 GB, 1.5 / 154 GB and 0.77 / 77 GB. TurboQuant 192 + 4 = 196 B → 19.6 GB matches Ch 27's own table. 96 B → 9.6 GB.
- **HNSW figures.** The HNSW overhead of ~1.05× and "1.5–2× only for low-d" match Ch 29. int8 HNSW ~90 GB matches Ch 31, ~92 GB matches Ch 60, 900 GB at 1B matches Ch 32, and the 256 GB machine matches Ch 58.
- **Compressed and on-disk.** Binary ≈ 25 GB and DiskANN ≈ 5 GB match Ch 60.
- **Multi-vector and storage.** ColBERTv2 36 B → 7,200 B matches Ch 37. ColQwen2 750 × 128 × 2 = 192 KB (≈190) and 750 × 36 = 27 KB match Ch 47. FDE ~4,000-d matches Ch 38 and 61. 3 GB per million in object storage matches Ch 57.

**Defaults (all match):**
- τ 0.01–0.07 (Ch 12).
- M / efC / efSearch 16 / 200 / 64–128 (Ch 29, 30).
- PQ d/m 4–16 (Ch 24).
- Tombstones: alert at 15%, rebuild at 20% (Ch 31).
- BM25 k1 1.2–2.0, b 0.75, with b lowered for uniform chunks (Ch 8).
- RRF 60, fusing 100 candidates per retriever (Ch 40).
- Rerank 100, MMR λ 0.7 (Ch 41).
- Chunks of 400–600 tokens with 10–15% overlap (Ch 50).
- 180-day half-life (Ch 51).
- 4k / 8k / 16k / 32k budget sweep, with 12k as the example (Ch 53).
- Golden set 50 / 200 (Ch 19, 54).
- 2% wrong-refusal cap, re-run on model change (Ch 55).
- DPI: ~60 for ColPali, ~80 for the A4 cap (Ch 47), 150–200 for storage (Ch 61).
- Shard over-fetch 2–5× (Ch 58).
- Canary tol 1e-4 (Ch 63).
- GPU-hours ~7 / ~19 / ~140 (Ch 60).

**Scale and operations:** the Ch 58 strategy and query-pattern tables, the Ch 59 staleness budget and maintenance calendar, and the Ch 60 lever table are copied verbatim. The Ch 63 six phases agree step by step, and the ACL sync path matches Ch 62. The filtering tree matches Ch 33's table row for row.

**Other checks:** the manual announces seven parts and has seven. The eight laws are eight and match Ch 65's eight principles. The "Why is this answer wrong?" failure list maps exactly onto Ch 55's 1–8 (identifiers, dilution, follow-up, stale, hubs, tables, filters, multi-part) and 9–12 (unanswerable, ignored context, altered numbers, injection).

- **[MUST]** The "Why is this answer wrong?" tree puts the unanswerable case under the YES branch, but Ch 55 says it must be checked *before* the split. Ch 55: "Failure 9, a question the corpus cannot answer, is a special case. There is no page to find… We check for it before the split." Its `diagnose()` code also tests `answerable is False` first. Following Ch 66, a reader asks "Was the needed information in the assembled context?" about a pet-insurance question, answers NO (there is none), and is sent to the retrieval branch. Quote: "└─ YES → generation: unanswerable-but-answered? ignored context? altered numbers?". Fix: add a first line above the split, e.g. "Is the question answerable from the corpus at all? NO → failure 9: distance floor + relevance check (Ch. 55)", and remove "unanswerable-but-answered?" from the YES branch.
- **[MUST]** The IVF `nprobe` default range appears in neither cited chapter. Ch 66 says "16–32 (sweep)", Ch 25's defaults table says "| `nprobe` | 8–64 |", and Ch 23 says "IVF often needs `nprobe` of 16–64". Quote: "| IVF `nlist` / `nprobe` | $\sqrt{n}$–$4\sqrt{n}$ / 16–32 (sweep) | 23, 25 |". Fix: "8–64, start at 16 (sweep)" to match Ch 25, or change Ch 25 and Ch 23 if 16–32 is the intended starting band.
- **[SHOULD]** "Which index?" does not reproduce Ch 20's table, although it cites Ch 20. It drops Ch 20's "> 1B | DiskANN, or sharded IVF-PQ | 32, 58" and "Batch/offline workload | Flat on GPU, batched | 17" rows. It also adds two branches Ch 20 does not have, and cites neither: "sharded HNSW + compression" at > 100M (that is Ch 58) and "object-storage index" for cold data (Ch 57/60). Quote: "├─ latency ≤ 10 ms, budget OK ─────► sharded HNSW + compression". Fix: add a "> 1B → DiskANN, or sharded IVF-PQ (Ch. 32, 58)" branch and a "Batch/offline → Flat on GPU (Ch. 17)" line, and add Ch. 58 and 60 to the heading's citation list.
- **[NIT]** The ColPali row calls the 1,030 vectors "patches". Ch 46/47 say 1,024 patches plus the prompt tokens (Ch 47: "1,030 vectors × 128 dims × 4 bytes"). Quote: "**ColPali page:** 1,030 patches × 128-d × 4 B". Fix: "1,030 vectors (1,024 patches + prompt tokens) × 128-d × 4 B".
- **[NIT]** The Retrieval-quality checklist has two overlapping relevance-floor items. Quote: "- [ ] Relevance floor → honest \"I don't know\"". Fix: merge it into the preceding "Relevance floor calibrated (Ch. 55)…" item, e.g. append "; below it, answer an honest 'I don't know'".
- **[NIT]** The frontmatter summary still says "Every decision tree, formula, default and checklist in the book". The manual is now much fuller (the FIXES Scale block was added), but some decision content is still absent: Ch 20's > 1B and batch rows, Ch 44's three-way OCR / VLM-transcription / page-image choice, and Ch 50's chunk sizes for other content types. Quote: "summary: \"Every decision tree, formula, default and checklist in the book". Fix: "The book's key decision trees, formulas, defaults and checklists" (see the README block for the matching README sentence).
- Style: reference chapter, so the teaching-section rules largely do not apply. There are no semicolons in prose ("In simple words" appears after the Scale-and-operations intro and the cost block). The missing seven standard sections are already recorded in section 0.
- FIXES.md items for this chapter: all applied. The BM25 k1 range is "1.2–2.0". Rescore depth reads "10–20× final k for PQ (Ch. 25). ~100× final k for binary (Ch. 26)". The half-life is "~180 days". The "Scale and operations" block reproduces all four requested tables (Ch 58 sharding, Ch 59 staleness budget + maintenance calendar, Ch 60 cost-per-query formula + lever ranking, Ch 63 six phases), each verified against its source, and none introduced an error.
- Code: no Python blocks. Arithmetic in the text blocks recomputed (0.99^40, SE at n = 20, all byte and GB conversions, 2^4 × 16 × 10 = 2,560, GPU-hours) and correct.

#### Fix status (applied 2026-09-16)

- [MUST] "Why is this answer wrong?" checks unanswerable case after split — fixed (new first question "Can the corpus answer the question at all? (check this first)" → failure 9, calibrated relevance floor + relevance check. "unanswerable-but-answered?" removed from the YES branch)
- [MUST] IVF `nprobe` default 16–32 matches no chapter — fixed per D7 ("8–64, start at 16 (sweep)")
- [SHOULD] "Which index?" drops Ch 20's > 1B and batch rows, uncited branches — fixed (">100M" split into "~100M – 1B" and "> 1B → DiskANN, or sharded IVF-PQ (Ch. 32, 58)", added "Batch/offline? → Flat on GPU, batched (Ch. 17)", heading now cites Ch. 17, 20, 25, 32, 58, 60)
- [NIT] ColPali 1,030 vectors called "patches" — fixed ("1,030 vectors (1,024 patches + prompt tokens)", cites Ch. 46, 47)
- [NIT] Duplicate relevance-floor checklist items — fixed (merged: "Below the floor, answer with an honest 'I don't know'")
- [NIT] Frontmatter summary says "Every decision tree…" — fixed ("The book's key decision trees, formulas, defaults and checklists, in one place, …")
- [SHOULD, section 0] No standard chapter sections — fixed differently: kept as a reference with no standard sections, and README now says Chapter 66 is a reference that skips parts of the template (D12).
- [D4] Lever table and intro — fixed: table copied from Ch 60 and checked identical by script, intro is the D4 Ch 66 line.
- [D5] Price ladder line — fixed (`RAM tens of times SSD, SSD a few times object storage`)
- [D6] DiskANN SSD figure — no change: Ch 66 quotes only "DiskANN ≈ 5 GB RAM + SSD", no SSD size.
- [Follow-up from Ch 60] LLM cost formula block — fixed ("LLM cost per query ≈ …", "= (13,000 × P_in + 400 × P_out) / 1,000,000 per query", monthly bill line uses "LLM cost per query"). "~92 GB" → "~93 GB" in both places.
- [Follow-up from Ch 58] Sharding table row — fixed ("`hash(doc_id)` → virtual bucket → shard", table checked identical to Ch 58). Over-fetch default — fixed ("~2–5× final k for int8, 10–20× for PQ, ~100× for binary").
- [Follow-up from Ch 59] Maintenance calendar Continuous row — fixed ("Monitor delta size and tombstone ratio", table checked identical to Ch 59; staleness table also identical)
- Code: no code changed (no Python blocks). Text-block arithmetic unchanged apart from the "/ 1,000,000" division and the realigned "(agents: perhaps 4–8)" note.
- Cross-chapter follow-ups: none (Ch 25 Step 4's "Usually 16–32" knee is consistent with "8–64, start at 16"; OUTLINE's Ch 66 one-liner is updated under `readme`)

## README.md and OUTLINE.md  (reviewer: chapters 64–66 + README/OUTLINE)

No MUST issues found. What I verified:
- **Titles, numbers and links (scripted).** OUTLINE has 66 rows. Every title is byte-identical to its chapter's frontmatter `title`, every row number equals the frontmatter `chapter`, and every link targets the right `NN-slug.md`. All `./` links in README and OUTLINE resolve.
- **Part ranges.** They are I 1–7, II 8–16, III 17–20, IV 21–33, V 34–41, VI 42–48, VII 49–56, VIII 57–63 and IX 64–66, identical in README and OUTLINE. OUTLINE's headings for Parts II–IX equal the frontmatter `part` exactly.
- **README counts.** "Sixty-six chapters" and "9 parts" are right. "HNSW in three chapters plus one on production" is Ch 28–31, "ColBERT … in three chapters" is Ch 35–37, and "ColPali in three chapters" is Ch 45–47. "About 13 hours" matches the 798 minutes of stated reading time.
- **Cast table.** "Ch. 1" is right for all five characters. Ch 1 introduces each in bold: Great Library, Map Room, Card Catalog, Librarian, and the Scholar as the LLM.
- **Reading paths.** The three paths name real chapters and Parts (Ch 1, Ch 17, Parts II, IV–IX).
- **One-liners.** I read the one-paragraph version of all 66 chapters beside OUTLINE, and the one-liners agree except for the items below.
- **FIXES items.** The FIXES 2.1 README items are applied: all five show "Ch. 1", and the "one running metaphor" sentence is replaced. The FIXES OUTLINE items are applied verbatim and match their chapters. The Ch 41 line is "The single largest quality win in a working RAG stack.", matching Ch 41's "typically the single largest quality improvement". The Ch 51 line is "The large share of retrieval quality that has nothing to do with vectors.", matching "we fix more problems than another embedding model would".

- **[SHOULD]** OUTLINE's Ch 20 one-liner flatly contradicts Ch 20's own Ninja notes, which say "There is a fifth idea that does not fit the taxonomy, and it is becoming important: **learned indexes**". Ch 65 cites Chapter 20 for exactly that idea ("Learned indexes and learned routing … (Chapter 20)"). Quote: "Hash, partition, cluster, or walk a graph. There is no fifth idea." Fix: "Hash, partition, cluster, or walk a graph: four ideas behind every classic index."
- **[SHOULD]** README's builder path skips Part V, yet the reader it addresses ("It demos well, fails in production, and you do not know why") most needs Ch 40 and Ch 41. The book itself calls hybrid search "the most reliable free win in retrieval" and rerankers "the single largest quality win in a working RAG stack", and Ch 66's default architecture is "hybrid + cross-encoder rerank". Quote: "Start at Chapter 17, then read Part VII (RAG) and Part VIII (Scale). Come back to\nParts II and IV when something breaks." Fix: "Start at Chapter 17, read Chapters 40–41 (hybrid search and rerankers), then Part VII (RAG) and Part VIII (Scale)."
- **[NIT]** The Part I heading differs between OUTLINE/README and the chapters. OUTLINE says "Part I — Foundations: What a Vector Actually Is", while the frontmatter `part` of Ch 1–7 is "Part I — Foundations". Parts II–IX match exactly. Fix: extend the seven frontmatter `part` values, or shorten the OUTLINE heading, so all three agree. README's sentence-case "Foundations: what a vector actually is" is fine as prose.
- **[NIT]** The OUTLINE Ch 2 one-liner names "a face", which Ch 2 never mentions. Coffee appears only in passing, as a callback to Ch 1. Its one-paragraph version says "a sentence, a photo or a product". Quote: "How you turn a coffee, a sentence, or a face into coordinates." Fix: "How you turn a sentence, a photo, or a product into coordinates."
- **[NIT]** The OUTLINE Ch 60 one-liner says "a billion vectors", but the chapter's paragraph and summary say it "works it through for a hundred million chunks". A billion appears only as "multiply every line by ten". Quote: "The arithmetic of a billion vectors, in dollars per month." Fix: "The arithmetic of a hundred million vectors, in dollars per month."
- **[NIT]** README and OUTLINE both say every chapter is a 10–15 minute read, but four chapters fall outside that: Ch 5 is "9 min", Ch 55 "16 min", Ch 65 "8 min", and Ch 66 is "Reference — keep open while building". Quote (README): "Each one is a ten-to-fifteen-minute read"; (OUTLINE): "each chapter a 10–15 minute read". Fix: "most chapters are a 10–15 minute read".
- **[NIT]** README never mentions the recurring side scenario, Globex: Pro with 120 seats since June 2026, previously Basic with 30, contract on pages 3,507–3,508. That scenario carries Ch 44, 45, 52, 56, 61, 62, 64 and 65, and README still calls the SSO question "one running example". Fix: add one sentence under the running example, e.g. "A second customer, Globex (Pro, 120 seats since June 2026, previously Basic with 30), and its scanned contract on pages 3,507–3,508 return in Parts VI–IX."
- **[NIT]** The README subtitle says "a hundred million documents", but Part VIII's scale is "about **100 million chunks**" across 5,000 customer companies (Ch 57, 60). Quote: "shipping retrieval over a hundred million documents." Fix: "…over a hundred million chunks", or "…over millions of documents", matching Ch 57's title.
- **[NIT]** OUTLINE's Ch 66 line "Every decision tree, formula, and default in one place." overclaims slightly. See the Ch 66 block: Ch 20's > 1B and batch rows and Ch 44's three-way choice are not reproduced. Fix: "The book's key decision trees, formulas and defaults in one place."
- Style: not teaching text, so only a light check. Two OUTLINE one-liners use semicolons (Ch 7 "…document; both are still in production.", Ch 35 "…vector; compare them at the end."), against FIXES §1.1 device 11. Optional.
- FIXES.md items for this unit: all applied (§2.1 README cast column and intro sentence; Part 3 OUTLINE Ch 41 and Ch 51 one-liners).
- Code: none.

#### Fix status (applied 2026-09-16)

- [SHOULD] OUTLINE Ch 20 "There is no fifth idea" contradicts Ch 20 Ninja notes — skipped per D2: OUTLINE keeps "There is no fifth idea." Ch 20's Ninja note now says learned indexes only look like a fifth idea, and Ch 65 cites Ch 20 in that framing.
- [SHOULD] README builder path skips Chapters 40–41 — fixed ("Start at Chapter 17, read Chapters 40–41 (hybrid search and rerankers), then Part VII (RAG) and Part VIII (Scale).")
- [NIT] Part I heading differs between OUTLINE and chapter frontmatter — skipped: the lead already aligned all seven Part I frontmatter `part` values with OUTLINE (checked), and D12 says not to touch it.
- [NIT] OUTLINE Ch 2 one-liner names "a face" — fixed ("How you turn a sentence, a photo, or a product into coordinates.")
- [NIT] OUTLINE Ch 60 one-liner says "a billion vectors" — fixed ("The arithmetic of a hundred million vectors, in dollars per month.")
- [NIT] "Every chapter a 10–15 minute read" — fixed in both files (README "Most are a 10–15 minute read, and each one can stand alone", OUTLINE "most chapters a 10–15 minute read")
- [NIT] README never mentions Globex — fixed (two sentences under the running example: Globex returns in Parts VI–IX, on Pro with 120 seats since June 2026, previously Basic with 30, contract on pages 3,507–3,508)
- [NIT] README subtitle says "a hundred million documents" — fixed ("…a hundred million chunks.")
- [NIT] OUTLINE Ch 66 line overclaims "Every decision tree" — fixed ("The book's key decision trees, formulas and defaults in one place.")
- [NIT] OUTLINE semicolons in Ch 7 and Ch 35 one-liners — fixed ("…document, and both are still in production.", "Keep every token's vector, and compare them at the end.")
- [D3] OUTLINE Ch 41 one-liner — fixed ("The largest precision win once retrieval works."). The Ch 40 line stays (Ch 40 fixer confirmed it agrees with D3).
- [D12 / section 0 SHOULDs for Ch 65 and 66] README template exception — fixed: a Note after "How every chapter works" says Chapter 65 is an essay and Chapter 66 is a reference ("the book's key decision trees, formulas and defaults in one place"), so both skip parts of the template.
- Code: no code changed
- Cross-chapter follow-ups: none

---

# Post-fix verification (2026-09-16)

After all fixes were applied, independent reviewers read every change (a diff against the pre-fix backup) in the context
of its chapter, looking for errors the fixes introduced and for fixes that did not fully resolve their finding. Findings
are written here chapter by chapter as they are found. Severity: **[NEW-MUST]**, **[NEW-SHOULD]**, **[NEW-NIT]** for
problems in the new text, **[INCOMPLETE]** for a finding whose fix does not fully resolve it. All items in this section
were then fixed. Each block ends with a "Resolution" record.

**Result.** No fix introduced a MUST-level error, and no fix was left incomplete. Every original MUST finding is confirmed
resolved. The checks found 7 SHOULD-level and 35 NIT-level problems, listed below by chapter. Most are in
new sentences or code the fixes added. Two table-rendering problems in Ch 35 and Ch 66 predate the fixes.

| Severity | Count |
|---|---|
| NEW-MUST | 0 |
| INCOMPLETE | 0 |
| NEW-SHOULD | 7 |
| NEW-NIT | 35 |

**Resolution.** All 42 items were fixed on 2026-09-16, a few with different wording than suggested (12 of them),
and none was skipped. A backup of the book before this round is in `_backup_before_postfix_fixes_2026-09-16.tar.gz`. The fixers
also raised four older mismatches outside their files, which the lead fixed: Ch 7's paraphrase answer, Ch 31's rescoring
tier, Ch 51's parent id, and Ch 55's list of chapters that say "relevance floor".

**Final verification by the lead.** Every change in this round was read in context against the backup. Structural checks
pass on all 66 chapters: chapter chain, headings, OUTLINE titles and links, every link, the seven standard sections, every
Python block parses, and every table row has the right number of cells. Changed code was rerun: Ch 6's estimator reads
9.8 on a 10-d sheet and 29.6 on a 40-d sheet, and ignores 5% near-copies at a tenth of the typical gap. Ch 37's
PLAID shortlist returns a quarter of `ndocs` and kept and ranked the target first in 40 of 40 toy queries. Ch 50's
splitter gives no empty chunk and ignores `#` inside code. Ch 55's `diagnose()` labels full, partial, cut and unanswerable
cases correctly.

### Post-fix check: Chapter 01  (verifier: chapters 01–20)

- No new issues.
- Verified: both NIT fixes resolve their findings. "big gaps everywhere" matches [5, -3, 5]. The new Map Room gloss points to "the end of this chapter", and "embedding" is indeed defined in the last main section ("Why hand-picked numbers are not enough"). The frontmatter `part` change matches OUTLINE's "Part I — Foundations: What a Vector Actually Is" and all seven Part I chapters. No code changed.

### Post-fix check: Chapter 02  (verifier: chapters 01–20)

- No new issues.
- Verified: title "Closing a company account" plus body "When a team cancels, we delete its user accounts" does contain "team", "company" and "accounts", so "Its title and text together share three words" is now true. "exactly two properties" matches the heading, the intro sentence and the "We will cover" bullet, and no leftover "two jobs" remains. `part` matches OUTLINE. No code changed.

### Post-fix check: Chapter 03  (verifier: chapters 01–20)

- No new issues.
- Verified: the E5 model cards on Hugging Face do state that cosine scores distribute around 0.7 to 1.0 (a result of the τ = 0.01 InfoNCE temperature), which matches Ch 12 line 394 ("its scores sit around 0.7–1.0 because it trains at τ = 0.01"). The new "model card" gloss ("the documentation page its authors publish with the model") agrees with Ch 11's definition ("the documentation page a model's authors publish alongside its weights") and Ch 14's gloss. No other 0.55–0.95 statement in the chapter needed updating. `part` matches OUTLINE. No code changed.

### Post-fix check: Chapter 04  (verifier: chapters 01–20)

- No new issues.
- Verified: the "When to use which one" table still has 3 columns in every row, and the semicolon and both em-dashes are gone. "Many models place nearly everything in a narrow cone ... 0.55 to 0.95" agrees with Ch 03's softened "some popular models" and with Ch 40 line 94 ("often crammed into about [0.55, 0.95]"). The readingTime skip follows the brief. `part` matches OUTLINE. No code changed.

### Post-fix check: Chapter 05  (verifier: chapters 01–20)

- No new issues.
- Verified: all three findings are resolved. Scaling a vector by a positive length cannot change any sign, so the new binary note is correct, and it matches Ch 26's sign-bit rule. The takeaway now matches the body. The whitening wording ("subtract a mean, multiply by a matrix") describes an affine map correctly. The one-paragraph version's general "makes quantization better behaved" line is not contradicted. `part` matches OUTLINE. No code changed.

### Post-fix check: Chapter 06  (verifier: chapters 01–20)

- **[NEW-SHOULD]** The dedup fix removes only *exact* duplicates, but near-duplicates still drag the TWO-NN reading down, and this chapter tells readers with near-duplicate corpora to trust the reading. With scikit-learn 1.9.1, a true 40-d sheet reads 28.8 clean. It reads 3.8 when 5% of points are near-copies at 0.1× the typical NN distance, 6.5 at 0.3×, and 1.1 when 10% are near-copies at 0.01×. The guide calls all of these "easy". Meanwhile "What people get wrong" blames bad recall on "near-duplicates, boilerplate" and says "Measure intrinsic dimension". Quote: "**Step 1:** Take a random sample of up to 5,000 vectors, and drop exact duplicates." Fix: add one sentence after the reading guide, e.g. "Near-duplicates (pages that differ by a word) are not removed by this and pull the reading far down. On text, a reading under about 5 usually means boilerplate, not an easy corpus."
- **[NEW-NIT]** The new JL sentence says "this worst-case number is bigger than the 768", but the paragraph gives two numbers, and only the ε = 0.2 one is bigger (about 510 at ε = 0.5 is smaller). "JL" is also never introduced as an abbreviation. Quote: "Note that for Acme this worst-case number is bigger than the 768 we started with, so JL alone does not justify shrinking our vectors." Fix: "Note that for Acme the ε = 0.2 figure, about 2,450, is bigger than the 768 we started with, so the Johnson–Lindenstrauss (JL) bound alone does not justify shrinking our vectors."
- Verified: extracted the new `intrinsic_dim` verbatim and ran it with real scikit-learn 1.9.1 (float32 and float64). 10-d sheet: 9.72/9.94 clean, 9.72/9.94 with one duplicate pair (old code 9.14/8.93), 9.74/10.02 with 200 identical rows (old 10.15/−0.0), 9.95/10.02 with 30% exact copies (old 0.09/0.08). 40-d sheet: 28.83/28.16 clean and 28.3–28.8 with duplicates (old 24.1, −0.0, 0.08). The text's "about 10" and "about 30" still hold, and the exact-duplicate finding is resolved. JL figures recomputed with the Dasgupta–Gupta bound: 508.6 (ε = 0.5) and 2,445 (ε = 0.2). The new hotel wording is consistent with its mapping list. The reading guide (15–40 typical, over ~40 pay for recall, over ~60 check the encoder) agrees with the main text's "about 40" example and the Key takeaways.

#### Resolution (applied 2026-09-16)

- [NEW-SHOULD] Near-duplicates still drag intrinsic-dimension reading down — fixed differently: fixed in code and prose, not by a caveat sentence alone. `intrinsic_dim` gains a Step 2 (`near=0.5`) that drops both vectors of any pair whose nearest neighbour is closer than half the median NN distance. Steps renumbered 1–5 with matching code comments. New paragraph says plainly that near-copies make a corpus harder to search but make the reading lower, gives the before/after readings, and admits that copies half the typical gap apart still slip through (reading about 9). The reading guide now says to give an "under ~15" text reading a second look (print the closest pairs, deduplicate or raise `near`, measure again). "What people get wrong" now says to check the NN-similarity distribution for a pile-up near 1 and to measure with near-copies removed.
- [NEW-NIT] JL sentence: only the eps = 0.2 figure exceeds 768 — fixed differently: "the eps = 0.2 figure is bigger than the 768 ..., and the eps = 0.5 one lets distances drift by up to half. So the bound alone does not justify shrinking our vectors." This dropped the undefined "JL" instead of introducing it, since it was not used again.
- Code: took the new `intrinsic_dim` verbatim from the chapter and ran it with scikit-learn 1.9.1, float32 and float64 (same results). Test data: a Gaussian sheet (5,000 points) inside 768-d. Near-copies replace 5% of points with copies of other points plus off-sheet noise at a fraction of the median NN distance. Old code → new code: 40-d clean 28.13 → 28.13. 5% near-copies at 0.1x: 3.82 → 28.35. At 0.3x: 6.47 → 28.35. At 0.5x: 9.33 → 9.33 (slips through, as the text says). 10% at 0.01x: 1.06 → 28.60. 10-d clean 9.84 → 9.87. 5% at 0.1x: 3.12 → 9.81. At 0.3x: 4.67 → 9.81. At 0.5x: 5.94 → 5.95. 40-d with 20,000 points (5,000 sampled), 5% at 0.1x: 11.27 → 27.92. Exact duplicates still handled: one pair 28.14, 200 identical rows 28.20, 30% exact copies 28.48 (40-d). The cost: on very low-dimensional clean data the filter also removes some real points and nudges the reading up (2-d 2.00 → 2.57, 5-d 5.15 → 5.57), well inside the "easy" band. Text numbers ("about 10", "about 30", "about 4", "about 28", "near 9") match.
- Cross-chapter follow-ups: none (Ch 12:400, 16:381, 18:325, 29:208, 30:229 mention intrinsic dimension only generally, no conflict)

### Post-fix check: Chapter 07  (verifier: chapters 01–20)

- No new issues.
- Verified: "a few hundred to a few thousand" now appears in the one-paragraph version, the bold definition and the first Key takeaway. It agrees with the table's "128 – 4,096" row. The analogy's "a few hundred scores" (line 71) is only illustrative. The BM25 qualifier in the takeaway matches the table's "No (BM25)" row. The comparison table still has 3 columns in every row. "cheapest large quality win" is consistent with D3. `part` matches OUTLINE. No code changed.

#### Resolution (applied 2026-09-16)

- [Lead, follow-up from Ch 40 fixer] Ch 7 called page 212 alone a correct answer to the paraphrase twin — fixed: "Dense search found the right page. The answer is still incomplete, because page 1,140's catch for teams under 50 seats did not come back."

### Post-fix check: Chapter 08  (verifier: chapters 01–20)

- No new issues.
- Verified: the new bridge sentence is accurate. Ch 7 introduces `SSO-4012` as Acme's error-code page, and Ch 8 uses the code again at line 182 (Note), line 320 (table) and line 450 (Key takeaways), so "we will meet that code again along the way" holds. The paragraph is rewrapped to the file's line width. No code changed.

### Post-fix check: Chapter 09  (verifier: chapters 01–20)

- No new issues.
- Verified: the new negative term `torch.logsigmoid(-neg_score).sum(-1).mean()` matches the formula log σ(v_c·v_w) + Σ_{k=1}^{K} log σ(−v_c·v_{w_k}). A NumPy port (B = 4, K = 5, D = 8) gives neg_score of shape (4, 5), and the code's loss equals minus the batch mean of the per-pair formula exactly (7.76887 both ways). The new explanatory sentence (`.sum(-1)` is the Σ, `.mean()` is the batch average) is correct. The inline comments still align (column 65). The readingTime skip follows the brief.

### Post-fix check: Chapter 10  (verifier: chapters 01–20)

- No new issues.
- Verified: the changed Under-the-hood block parses (ast), `import torch` is present for the new `torch.no_grad()`, and `out` is bound inside the `with` block before use. The comment punctuation no longer uses em-dashes. Both new fine-tuning glosses agree with Ch 12's definition ("taking an existing embedding model and training it further on our own pairs"). The readingTime skip follows the brief. Torch and transformers are not installed, so the block was not executed.

### Post-fix check: Chapter 11  (verifier: chapters 01–20)

- No new issues.
- Verified: the rewritten rescue list resolves the SHOULD. Response 2 now says BM25 needs the exact words, and "single sign-on" vs "SSO" needs a synonym list. This matches Ch 40, where page 1,140 is absent from BM25's paraphrase list and reaches the fused list only through dense search. Response 3's "question's *single sign-on* tokens find the page's *SSO* token" matches Ch 35's toy grid (single sign-on × SSO = 0.86) and its explanation "'single sign-on' matched 'SSO' in the caveat sentence". "Only the first and the third rescue this particular page" agrees with Ch 34 (chunking) and Ch 35 (ColBERT). The new bold "model card" definition matches Ch 3's gloss (which points to Ch 11), Ch 14's first use at line 231 ("the documentation pages published with each model, listing its prefixes, pooling and intended use, Chapter 11") and the bare uses in Ch 15, 16, 39 and 66. "BGE v1.5 and BGE-M3" as CLS-pooled is correct (both use the [CLS] hidden state for dense vectors) and is consistent across the body, the table (still 4 columns) and Key takeaways. No stale "BGE family … CLS" wording remains in the book. Ch 14's "BGE family uses an instruction on the query side" is about instructions, not pooling. No code changed.

### Post-fix check: Chapter 12  (verifier: chapters 01–20)

- No new issues.
- Verified: the reranker line "Once retrieval works, it is often the largest remaining quality jump (Chapter 41)" matches D3. It agrees with Ch 41 line 19 ("typically the largest remaining quality improvement"), Ch 41's takeaway at line 389, OUTLINE's "The largest precision win once retrieval works." and Ch 40's "highest-precision component". The Key takeaway ordering ("Fine-tune fourth, after chunking and hybrid search, reranking, and model selection") still matches the numbered list. The E5 statement (cards attribute 0.7–1.0 scores to τ = 0.01) is accurate and matches Ch 3's new sentence. The collapse note is now hedged as a rule of thumb, and it is consistent with Ch 6: PCA's linear dimension is at least the manifold's intrinsic dimension, and Ch 6's PCA cut to 192 dims does not conflict with needing more than 50 components. The "bi-encoder of similar size" wording matches Ch 13, and Ch 41 no longer says "any bi-encoder". The frontmatter summary semicolon is gone. No code changed.

### Post-fix check: Chapter 13  (verifier: chapters 01–20)

- **[NEW-NIT]** The new note reads as if every CrossEncoder squashes its output, but the sigmoid is only the library fallback. A model's config can override it. In sentence-transformers' `get_default_activation_fn`, the config's `activation_fn` (or the legacy `sbert_ce_default_activation_function`) wins, and only then does `num_labels == 1` give `nn.Sigmoid()`. `cross-encoder/ms-marco-MiniLM-L6-v2` sets `Identity` and returns raw logits (its card shows 8.6 and −4.3). `BAAI/bge-reranker-base`, the model in this code, sets nothing, so it does get the sigmoid. A reader who swaps models will see the opposite of what the note promises. Quote: "By default, `CrossEncoder.predict` squashes each score through a sigmoid (Chapter 9) into the range 0–1, so real output will not look like the 9.1 and 8.4 of our example." Fix: "For this model, `CrossEncoder.predict` squashes each score through a sigmoid (Chapter 9) into the range 0–1 (some models' configs switch this off), so real output will not look like the 9.1 and 8.4 of our example."
- Verified: the note is correct for the code's `BAAI/bge-reranker-base`. Its Hub config.json has no activation override, and the installed sentence-transformers source falls back to `nn.Sigmoid()` for one-label models. The rewritten "What people get wrong" item agrees with Ch 41 line 326 ("raw logits, or logits squashed through a sigmoid"). The sigmoid cross-reference to Chapter 9 is right. The finding is resolved. No code changed.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Sigmoid is only the fallback, and configs can override it — fixed differently: "For this model, `CrossEncoder.predict` squashes each score through a sigmoid (Chapter 9) into the range 0–1 ... Some models' configs switch the squashing off and return raw scores instead. The order stays the same either way." The override moved into its own sentence rather than a mid-sentence parenthetical.
- Code: no code changed
- Cross-chapter follow-ups: none (Ch 13 "What people get wrong" and Ch 41:326 already allow both raw logits and sigmoid output)

### Post-fix check: Chapter 14  (verifier: chapters 01–20)

- No new issues.
- Verified: line 231 is the chapter's first use of "model card", and it now carries a gloss plus a pointer to Chapter 11. The gloss agrees with Ch 11's bold definition and Ch 3's gloss. The later bare uses (lines 284, 319, 372, 375) come after it. Ch 13's "this model's card" (line 402) comes after Ch 11's definition. "a couple of meaningless tokens" resolves the tokenizer-dependence NIT. The readingTime skip follows the brief. No code changed.

### Post-fix check: Chapter 15  (verifier: chapters 01–20)

- No new issues.
- Verified: grep shows line 142 is now the only use of "inference" in Chapters 1–15, so the gloss sits at the first use. The gloss is accurate. The readingTime skip follows the brief. No code changed.

### Post-fix check: Chapter 16  (verifier: chapters 01–20)

- No new issues.
- Verified: in the venv, `rank_bm25.BM25Okapi.__init__` has the signature `(corpus, tokenizer=None, k1=1.5, b=0.75, epsilon=0.25)`, so `k1=1.2, b=0.75` is accepted (the instance reports 1.2 and 0.75) and the comment "library default k1 is 1.5" is correct. Key takeaways' "top 10–15" now matches Caution 4 (line 147). The "All 50" sentence is now true (Model A 0.75 is the best of the three embedding models, and the fused row is 0.87). The MTEB wording changes are accurate hedges. No table was touched.

### Post-fix check: Chapter 17  (verifier: chapters 01–20)

- No new issues.
- Verified: "about 1 ms to 0.4 ms" now matches the one-paragraph version (line 18), the numbers section ("about 1 ms on several" cores, line 141) and the example (line 203). "Chapters 31 and 59" is correct, since Ch 31 line 52 defers compaction to Ch 59, which defines it (line 56). "on a GPU can reach 50–100×" is consistent with the reviewer's roughly 19× CPU measurement. No code changed.

### Post-fix check: Chapter 18  (verifier: chapters 01–20)

- No new issues.
- Verified: recomputed the binary numbers for 100M × 768-d. Codes are 96 B × 100M = 9.6 GB = 0.031× of 307.2 GB ("The 1-bit codes alone are 0.03×"). Codes plus ~160 B of graph ≈ 256 B → ~25.6 GB = 0.083× of raw float32 (0.079× of Ch 60's 323 GB HNSW baseline), so "~0.08×" holds against both bases. These match Ch 60's row ("~250 RAM + 307 GB SSD | ~25 GB RAM | 0.08× RAM" and "96 bytes of code … plus the same ~160 bytes of graph"). They also match Ch 26 lines 246–266 (Stage 1 "a few ms", Stage 2 "~1 ms in RAM, a few ms from SSD", "about 25 GB of RAM", "3% of the vector bytes … closer to 8%"), Ch 15 line 367 and Ch 31 line 232. The Ninja cascade now agrees with the table. The reranker sentence no longer contradicts the page 1,140 rescue in Ch 13 and 17. The table still has 5 columns. No code changed.

### Post-fix check: Chapter 19  (verifier: chapters 01–20)

- **[NEW-NIT]** The last new sentence credits recall@5 with showing something the number cannot show. Recall@5 on this list is 0.67, which says one relevant page is missing. Only by looking at the list does the team learn that the missing page is the optional page 2,306 ("The answer does not strictly need it"). Quote: "A team that hands over the top 5 tracks recall@5 and sees that both pages the answer needs are already there." Fix: "A team that hands over the top 5 tracks recall@5 (0.67), and a look at the list shows that the one page missing is the optional page 2,306."
- Verified: the rewritten team paragraph now ties each team to the k it passes, which resolves the SHOULD. Recall@3 = 1/3 = 0.33 and recall@10 = 1.00 on hits at ranks 2, 5 and 9 of 3 relevant pages. Grade gains 2²−1 = 3 and 2³−1 = 7 are correct, and "good" = 2, "perfect" = 3 fits the four-level "perfect / good / marginal / irrelevant" scale at line 226. "A paired test (below)" does point to the paired-test paragraph that follows the code. Key takeaways and the worked-example table were not touched and still agree. No code changed.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Recall@5 cannot show that the missing page is optional — fixed
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 20  (verifier: chapters 01–20)

- **[NEW-NIT]** The rewritten Ninja note's closing sentence narrows its own claim. The note opens "They keep the partition or cluster family", and Ch 65 line 196 says "Chapter 20 places it in the partition or cluster family, with a trained router in place of computed signposts". The closing line says only "partition", and "signposts" matches neither of this chapter's images (cuts in the floor plan, signs at each wing's door). Quote: "A learned index is a partition index whose signposts were trained rather than computed." Fix: "A learned index is a partition or cluster index whose signs were trained rather than computed."
- **[NEW-NIT]** Acme's index holds chunks, and "a few hundred thousand chunks" (Ch 58: about 320,000) is past the "< 100k vectors" first row. The index therefore sits in the second row, not "the first two rows". The conclusion (brute force) is unchanged. Quote: "Its Library has 40,000 pages, a few hundred thousand chunks, so it sits in the first two rows." Fix: "Its Library has 40,000 pages, a few hundred thousand chunks, so it sits in the second row."
- Verified: D2 is satisfied. The Ninja note now says learned indexes "can look like a fifth idea, but they are not" and that "the families stay four". This agrees with the body ("That is the whole taxonomy", "exactly four ways"), the summary ("one of four ideas, or a combination"), Key takeaways ("There are four"), line 167 ("Compression is not a fifth strategy"), OUTLINE line 40 ("There is no fifth idea.") and Ch 65 lines 193–198 ("It is not a fifth index idea. Chapter 20 places it…"). Ch 21 line 22 ("the first of the four ANN families") matches the new "first of the four ideas". The new LSH pseudocode line was run in a harness (5,000 unit 32-d vectors, 10 tables, each with its own 8-bit hyperplane hash). It executes, one query hits 10 distinct buckets, and recall@10 = 0.385, low as the chapter says. The side-by-side table still has 4 columns. The Part VIII 100M-chunk framing matches Ch 17 line 261 and does not say Acme "grows" (D10).

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Learned-index closing line says only "partition" and "signposts" — fixed differently: "A learned index is a partition or cluster index whose routing rule is trained rather than written by hand." This reuses the note's own "hand-written routing rule" instead of "signs", which fits only the cluster image.
- [NEW-NIT] A few hundred thousand chunks sit in the second row — fixed
- Code: no code changed
- Cross-chapter follow-ups: none (Ch 65:197 "trained router in place of computed signposts" stands on its own and does not conflict)

### Post-fix check: Chapter 21  (verifier: chapters 21–33)

No new issues.

- Verified: the one change ("The side test in Phase 1, Step 2 is one line of maths") matches the bold "Phase 1: Building the table." / "Phase 2: Answering a query." labels and the later "`_sig` does Phase 1, Steps 1–3" sentence. The skipped independence NIT is correctly reasoned: for a fixed pair of vectors the random hyperplanes are drawn independently, so each bit collides with probability 1 − θ/π independently and $P_{\text{found}} = 1-(1-p^k)^L$ is exact.

### Post-fix check: Chapter 22  (verifier: chapters 21–33)

No new issues.

- Verified: "Change 1: Cut along the data, not the axes." now agrees with the Under-the-hood note ("not from a coordinate axis or a purely random direction"), the one-paragraph "random cuts" and the takeaway "data-driven random cuts". The comparison table still has 4 cells per row. The new routing sentence is accurate: ScaNN's partitioner is a k-means tree that supports several levels, and FAISS gets the same effect with a small coarse index (for example HNSW) over the IVF centroids. No other chapter still uses "hierarchical k-means trees".

### Post-fix check: Chapter 23  (verifier: chapters 21–33)

- **[NEW-NIT]** The new training-sample guide says 250 where Ch 25 (and FAISS's `max_points_per_centroid = 256`) say 256. Quote: "The usual guide is roughly 30–250 sample vectors per centroid". Ch 25 has "| Training sample | 30–256 × `nlist` vectors |" and "Use at least 30 × `nlist` vectors, ideally 256 ×." Fix: "roughly 30–256 sample vectors per centroid".
- Verified: √(10^6) = 1,000, √(10^8) = 10,000, √(10^7) ≈ 3,162 ("about 3,000") are correct. The new √n-to-4√n wording matches Ch 25's table ("$\sqrt{n}$ to $4\sqrt{n}$ | 4,096 for 10M; 16,384 for 100M; 65,536 for 1B", each value inside its range: 3,162–12,649, 10,000–40,000, 31,623–126,491) and Ch 66 ("$\sqrt{n}$–$4\sqrt{n}$ / 8–64, start at 16"). FAISS's 4√N–16√N guideline and its 30×K–256×K training-size rule are stated correctly. The code's default `sample=200_000` gives 200 per centroid at `nlist = 1000`, inside the new guide. The nprobe rewording resolves the "completely predictable" NIT. Takeaway "`nlist` ≈ √n" and the pitfall "Keep roughly $\sqrt{n}$" still agree with "starting point".

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Training-sample guide says 250, Ch 25 and FAISS say 256 — fixed
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 24  (verifier: chapters 21–33)

No new issues.

- Verified: the m-table label "24 ($d/m = 32$, outside the safe range below)" keeps 4 cells per row and points at "Keep $d/m$ between 4 and 16", which is indeed below the table. "16× to 64× with $d/m$ in the safe range" matches m = 192/96/48 (d/m = 4/8/16). The new code comment "100,000 of the 100 million chunks" fits D10 and the chapter's 100M-chunk framing. Ran the changed `PQ` class with sklearn KMeans on 3,000 clustered unit vectors (d = 32, m = 8, batch = 777): batched `encode` gives uint8 (3000, 8) codes identical to the old unbatched version, ADC scores equal q · reconstruction to 1e-7, and `search` on 3-vector and 1-vector collections returns [0 2 1] and [0] without crashing. The `c_sq - 2 * sub @ C.T` trick matches Ch 23's `nearest_centre`.

### Post-fix check: Chapter 25  (verifier: chapters 21–33)

- **[NEW-NIT]** The new tuning procedure builds the index at full size (100 million chunks) and still sweeps `m` over three values, so "an afternoon" now understates the work: each full build re-encodes 100M vectors against 16,384 centroids (hours on CPU). Quote: "**The tuning procedure.** Do this in order. It takes an afternoon and removes the guesswork." and, in the intro, "how to tune it in an afternoon". Fix: "It takes a day or two, mostly waiting on batch builds, and removes the guesswork." and "how to tune it without guesswork".
- Verified: the reordered Phase 1 (Step 2 learn the OPQ rotation and rotate the sample → Step 3 k-means → Step 4 residuals → Step 5 codebooks) now matches Phase 2 and Phase 3 (rotate first, then nearest centroid), the code comment "learns rotation + centroids + codebooks (Phase 1)", and FAISS's `OPQ…,IVF…,PQ…` `IndexPreTransform`, which trains the OPQ matrix on the raw sample and then trains the IVF quantizer and residual PQ on the transformed vectors. No text refers to the old step numbers. "$d/4$ to $d/16$" gives 192–48 for d = 768, matching the `m` sweep, Ch 24's "d/m between 4 and 16" and Ch 66's "4–16 dims each". The residual bridge matches Ch 24's "the residual is the part PQ got wrong". The full-collection ground truth plus full-size build resolves the nprobe carry-over NIT, and the explanation (a small slice with `nlist` = √n gives each list a larger share of the data) is correct. Ran `rescore` with a stub `fetch_full_vectors` that rejects negative IDs, on a 100-slot result with 63 `-1` pads: it returns 10 results with the query's own ID first.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] "An afternoon" understates full-size builds swept over three m values — fixed
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 26  (verifier: chapters 21–33)

- **[NEW-NIT]** The new 8% sentence says "the whole index", but the cascade still keeps the 307 GB of full vectors on SSD for rescoring. The 8% is RAM only, which is how Ch 60 ("0.08× RAM") and Ch 18 ("~0.08× memory") label it. Quote: "With an HNSW graph on top, the whole index is closer to 8% of the `float32` version (Chapter 60)." Fix: "With an HNSW graph on top, the index needs closer to 8% of the RAM of the `float32` version, with the full vectors on SSD (Chapter 60)."
- Verified: 96 / 3,072 = 3.1% ("3% of the vector bytes") and 25 GB / 323 GB = 7.7% ("closer to 8%") agree with Ch 60's table ("~323 GB | 1.00×" and "~25 GB RAM | 0.08× RAM"), Ch 60's lever "3.5–13× HNSW RAM (4–32× on vector bytes alone)" and Ch 18's "binary codes + graph (~0.08× memory)". Ran both changed blocks. `encode_sq` gives [63 −39 38 25] on the worked example. With range ±0.15, inputs ±0.16 encode as 127/−128 (no wrap). On 20,000 random unit vectors (d = 768) with min/max from 200 of them, all 95 out-of-range values kept the right sign. The binary toy gives Hamming [0 3 1]. depth 2 → pages 212 and 87 (0.98, 0.37). depth 3 and depth 1000 → pages 212 and 1,140 (0.98, 0.94), with no crash. The new Phase 2 Step 4 matches the code comment "Steps 1-4". The softened axes-vs-random sentence now agrees with the Ninja note "Chapter 27 takes the same instinct much further, with a random rotation". The frontmatter semicolon is gone.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] "Whole index" 8% is RAM only, full vectors stay on SSD — fixed differently: same meaning, reordered for flow: "With an HNSW graph on top and the full vectors on SSD, the index needs closer to 8% of the RAM of the `float32` version (Chapter 60)."
- Code: no code changed
- Cross-chapter follow-ups: none (Ch 60 "0.08× RAM" and Ch 18 "~0.08× memory" agree)

### Post-fix check: Chapter 27  (verifier: chapters 21–33)

No new issues.

- Verified: ran the whole Under-the-hood code on 2,000 unit vectors with very uneven per-coordinate scales (d = 768). The new query-rotating `scores` (`q_rot = self.P @ q`, then `R @ q_rot`) matches the old `decode(codes, norms) @ q` to 1.1e-8 at 1 and 2 bits, returns shape (2000,), and gives the same top-50 order. The self-test MSE is still 0.362 (1 bit) and 0.117 (2 bits), and `randomized_hadamard` still preserves norm. The step comments match Phase 3 Steps 1–3 (rotate the query, rebuild rotated coordinates, dot product), and `P v` in `encode` versus `P q` in `scores` use the same rotation. The rewritten card analogy is now self-consistent (card = original coordinate, hand = rotated coordinate, one judging rule = fixed quantizer). "could guarantee" is used consistently in the one-paragraph version, the distortion-rate definition and the Ninja note ("for every unit vector"). The 1-bit two-stage sentence (b − 1 = 0 bits, so pure QJL, unbiased) matches the paper. "Lloyd–Max = 1-D k-means on the bell curve" is correct and matches the ±0.7979 / ±0.4528 / ±1.5104 constants. "four chapters" matches Ch 28–31. The frontmatter semicolon is gone.

### Post-fix check: Chapter 28  (verifier: chapters 21–33)

No new issues.

- Verified: the Milgram passage now reads premise → question → "The answer is that **almost everyone has a few unusual connections.**", which resolves the SHOULD. "page 80" is used nowhere else in the book, and no other sentence names the layer-2 stop. `mL` is defined in Ch 29 ($m_L = 1/\ln(M)$), so "(`mL`, defined in Chapter 29)" is accurate. The rewritten entry-point Ninja note no longer claims a cache mechanism, and no other chapter (including Ch 31) calls the entry point a bottleneck. No code changed.

### Post-fix check: Chapter 29  (verifier: chapters 21–33)

No new issues.

- Verified: 16 × 4 / 15 = 4.27 ("≈ 4.3 B"), 3,072 + 128 + 4.27 + 20 = 3,224.3 ("≈ 3,224 B") and 3,224 / 3,072 = 1.05×. The block and Key takeaways both say 3,224, and no other file uses the old 3,225 (Ch 60 rounds to "~3,230", unchanged). "exponentially decaying distribution" is used in both the body and the takeaway, and no chapter still says "exponential distribution". The `efSearch` sentence ("costs no memory, only latency") now agrees with the latency callout. The two-node-ID `dist` note matches the code's `dist(c, s)` and pruning calls. No code changed.

### Post-fix check: Chapter 30  (verifier: chapters 21–33)

- **[NEW-NIT]** The new "bet" sentence ties the 530 → 1,140 miss to the Step 3 *stopping* rule, but in the efSearch = 2 walk the stop comes because "Nothing is left to explore". Page 530 was never waiting: Step 5's identical "worse than the worst" test skipped it ("Page 530 (0.70) is worse than the worst on the shortlist (0.60). We skip it."). Quote: "This stopping rule is a bet. A worse page can still lead to a better one, as page 530 leads to page 1,140 in the example below." Fix: "This rule is a bet, and Step 5 makes the same bet when it skips a neighbour. A worse page can still lead to a better one, as page 530 leads to page 1,140 in the example below."
- **[NEW-NIT]** The rewritten takeaway now repeats the next bullet. Both say that 530 leads to 1,140 and that a longer shortlist helps. Quote: "A worse page can still lead to a better one (530 leads to 1,140), and a longer shortlist gives up later." followed by "- A longer shortlist can step through a less relevant page (530) to reach the one we need (1,140)." Fix: end the stopping bullet at "That is a bet, not a guarantee, and a longer shortlist gives up later." and keep the 530 bullet as is.
- Verified: the stopping rule now agrees everywhere. The body says "none of the waiting pages can enter the shortlist themselves", the bet paragraph follows, the simple-words line reads "keeps walking while the waiting pages could still join the shortlist", the code comment reads "Step 3: none can join, stop", and the takeaway reads "no waiting page could join it. That is a bet, not a guarantee". No "exactly when more walking cannot help" or "cannot improve" wording is left in any chapter. The one remaining "Page 530 cannot help me" is the search's own mistaken thought, which is intended. Ran the changed search block: it prints "efSearch=2: pages [212, 45]" and "efSearch=4: pages [212, 1140]", matching the comments. The `efSearch < k` Note and What-people-get-wrong now agree. "≈ 1,000–4,000" is consistent with 128 × 32 = 4,096 "plus a few dozen in the upper layers". The benchmark-transfer note no longer calls GloVe friendly.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Bet sentence ties the 530 → 1,140 miss to the stop, not Step 5 — fixed differently: "This stopping rule is a bet, and Step 5 makes the same bet each time it skips a neighbour. A worse page can still lead to a better one. In the example below, the answer is lost at Step 5, not at the stop. The search skips page 530, the only road to page 1,140. A longer shortlist makes both bets less often." This says outright where the miss happens.
- [NEW-NIT] Two takeaways repeat 530 → 1,140 — fixed differently: merged into one bullet instead of only trimming the first: "Once the shortlist is full, the search skips any neighbour worse than everything on it, and stops when the closest unexplored page is that bad. Both are bets, not guarantees. A longer shortlist bets less often, so it can step through a less relevant page (530) to reach the one we need (1,140)." The takeaway now names both the skip and the stop as bets, matching the body.
- Code: no code changed
- Cross-chapter follow-ups: none (Ch 31:61–71 tombstone story uses 530 → 1,140 as a road only and does not conflict)

### Post-fix check: Chapter 31  (verifier: chapters 21–33)

- **[NEW-NIT]** The rewritten health check indexes `live_labels` with a whole matrix of row numbers, so it must be a NumPy array. A plain Python list of labels (the natural thing to keep) raises `TypeError: only integer scalar arrays can be converted to a scalar index`. Quote: "live_labels[i] is the index\n# label of row i." and "truth = live_labels[rows]". Fix: "# label of row i (a NumPy array)." or `truth = np.asarray(live_labels)[rows]`.
- Verified: ran `nightly_index_health` exactly as printed, using Ch 18's brute-force (`argsort(-(Q @ V.T))[:, :10]`) and overlap steps and an hnswlib-like stub (uint64 labels, `mark_deleted`, `knn_query` returning shape (1, k)). The toy had 5,000 × 32-d unit vectors, labels that are not row numbers, 10% tombstoned, and 200 noisy queries. A perfect index scores recall@10 = 1.0 with tombstones 0.1, and a noisy index scores 0.62. The uint64-vs-int64 label sets intersect correctly. Body item 1, Steps 1–3, the code comments and What people get wrong ("Keep a flat copy of the vectors to score sampled queries against") all describe the same full-corpus truth, and Ch 66 ("Recall check on sampled queries") agrees. The two-index ASCII box rows are now all 69 characters. The FAISS/Lucene/Vespa and hnswlib in-place wording is accurate, and the analogy no longer mentions houses.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] `live_labels[rows]` fails when labels are a plain list — fixed: `truth = np.asarray(live_labels)[rows]`, added `import numpy as np` to the block, comment now says "label of row i (a list or an array)"
- Code: took `nightly_index_health` verbatim from the chapter and exec'd it with Ch 18's brute-force (`argsort(-(Q @ V.T))[:, :10]`) and overlap steps and an hnswlib-like stub (uint64 labels, knn_query returning (1, k)). Data: 5,000 × 32-d unit vectors, labels that are not row numbers, about 10% tombstoned, 200 noisy queries. `live_labels` passed as a list of Python ints, a uint64 array, an int64 array and a list of np.uint64: every case gives recall@10 = 1.000 for an exact index and about 0.40 for a noisy one, with tombstones 0.096. The old `live_labels[rows]` on a list raises "TypeError: only integer scalar arrays can be converted to a scalar index", as reported.
- Cross-chapter follow-ups: none
- [Lead, follow-up from Ch 47 fixer] "Keep full-precision vectors in object storage" read as rescoring from object storage — fixed: object storage is the source of truth, and query-time rescoring reads a copy on SSD or RAM (Chapter 60).

### Post-fix check: Chapter 32  (verifier: chapters 21–33)

No new issues.

- Verified: 3,072 + 4 + 64×4 = 3,332 B and 3,072 + 4 + 128×4 = 3,588 B ("≈ 3.3–3.6 KB of data"), each padded to one 4,096-byte block. 10^9 × 4,096 B = 4.10 TB (3.73 TiB), which does not fit a nominal 4 TB drive (3.64 TiB) but fits 8 TB or two 4 TB drives. So the D6 wording is correct, and 10^8 × 4,096 = 410 GB matches Ch 60's "~410 GB SSD". "~4.1 TB" now appears in the RAM/SSD table, the economics table (still 3 cells per row), the derivation and Key takeaways. No "3.5 TB" or "4 TB NVMe drive" is left anywhere in the book. The softened parallel-read claim is self-consistent: 100 sequential reads × 100 μs = 10 ms versus 25–30 rounds × 100 μs = 2.5–3 ms at W = 4, a 3.3–4× gain ("another 3–4×", "several times slower"). The body, What people get wrong and Key takeaways all say this, and no "mandatory", "collapses" or "curiosity" wording remains. "25–30 rounds" matches W = 4. The build `L` of 75–200 and R of 64–128 fit DiskANN's documented ranges, and "libaio or io_uring" is accurate.

### Post-fix check: Chapter 33  (verifier: chapters 21–33)

No new issues.

- Verified: the new claims hold. Filtered-DiskANN (Gollapudi et al., WWW 2023) builds label-aware Vamana graphs. Qdrant's indexing docs say it "extend[s] the HNSW graph with additional edges based on indexed payload values", so "adds extra HNSW edges for each indexed payload value" and the cost "extra edges in memory and build time" are accurate. ACORN is Patel, Kraft, Guestrin & Zaharia, PACMMOD 2(2) / SIGMOD 2024, and its main variant (ACORN-γ) does build a denser graph at construction. "far more likely to stay connected" correctly drops the guarantee. The in-filter band ("pass rate is below about 10%", table row "pass rate < ~10%, set too big to brute force") now matches the decision table, the billion-scale 400,000-vector example and Ch 66 ("pass rate < 10%, allowed set > 50k ► in-filter traversal (ACORN-style if available)"). The timing wording ("about a millisecond on a few cores" for 40,000, "a millisecond or two on a few cores" for 50,000) matches Ch 17. Ran the changed pre-filter snippet on 2,000 × 16-d vectors with every fifth row "acme" and IDs equal to row numbers: `results = allowed[top]` equals the exact filtered top 10, and every returned ID is an Acme ID. No "under-explored" or "current frontier" wording remains.

### Post-fix check: Chapter 34  (verifier: chapters 34–41)

No new issues.
- Verified: both "about twenty other sentences" / "about twenty sentences on audit logs, allow-lists and retention" edits match the code (`other_lines` has exactly 20 entries plus the "Security add-on." heading fragment), so "about twenty" is accurate in both places. "Chapters 42 and 45" is correct: Ch 42 defines patches (ViT, 224 ÷ 32 = 7 × 7) and Ch 45 cuts page images into a 32 × 32 patch grid. No code or tables changed; markdown intact.

### Post-fix check: Chapter 35  (verifier: chapters 34–41)

No new issues.
- Verified: all four NIT fixes read correctly in place (`##` gloss is accurate for WordPiece; "Each word and number piece keeps its own vector" now agrees with page-side punctuation filtering; "Team A is the question. Team B is the page." completes the analogy mapping; the dangling modifier is gone). The MaxSim LaTeX `\,` renders cleanly and `$$` fences are balanced. "twenty or so" / "about twenty" agree with Ch 34 (`other_lines` has 20 entries). No code or numbers changed.

#### Resolution (applied 2026-09-16)

(Item from the "table rendering, whole book" block. The Ch 66 half belongs to another fixer.)
- [NEW-SHOULD] Ch 35 table cell `$|q| \times |d|$` splits the row — fixed ($\lvert q\rvert \times \lvert d\rvert$). Rescanned all table rows in chapters 35, 37, 38, 40, 41, 44, 45, 47, 50: no other `|` inside math or inline code.
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 36  (verifier: chapters 34–41)

No new issues.
- Verified: The new "comparable across queries" paragraph is correct. ColBERT pads or truncates queries to 32 [MASK]-expanded vectors, matching Ch 35 and Ch 37's "Queries padded to 32 tokens". ColPali's query length still varies with the text, and Ch 45 says "about 20 vectors". Dropping PyLate was right, since PyLate ColBERT models also pad queries by default. "length-robust (extra tokens add nothing unless they match better)" is accurate and agrees with the Key takeaway "length alone cannot win". "PLAID's centroid-based search" matches Ch 37's rewritten pipeline. "What's next" still names "centroid pruning", which remains a named sub-step (Step 4) in Ch 37. The `\,` LaTeX spacing renders, `$$` is balanced, and no code changed.

### Post-fix check: Chapter 37  (verifier: chapters 34–41)

- **[NEW-NIT]** The rewrite drops PLAID's second, unpruned centroid-interaction pass, so the code and Step 5 decompress the whole `ndocs` shortlist. The paper (and the ColBERT library's k ≤ 100 preset, which these defaults copy: nprobe 2, t_cs 0.45, ndocs 1,024) re-scores those 1,024 without pruning and decompresses only the best ndocs/4 = 256. Quote: "return sorted(rough, key=rough.get, reverse=True)[:ndocs]   # shortlist for Step 5". Fix: add one sentence to Step 5: "(PLAID first re-scores this shortlist from centroids once more, without pruning, and decompresses only the best quarter.)" Or change the comment to "# PLAID re-scores these without pruning and decompresses the top ndocs/4". "A few hundred to a thousand" still covers PLAID's k = 100 and k = 1,000 settings (256 and 1,024), so no number needs to change.
- **[NEW-NIT]** The centroid count is loose for the stated nprobe range. Quote: "32 query tokens keep at most about a hundred of the $2^{18}$ centroids". At nprobe 4 the maximum is 32 × 4 = 128. Fix: "keep at most 128 (32 × 4) of the $2^{18}$ centroids". The "more than 99.9% are never visited" claim is still correct (128 / 262,144 = 0.05%).
- Verified: I checked every stage against PLAID (Santhanam et al. 2022). Step 2 takes each query token's top-nprobe centroids (1–4). Step 3 gathers candidates from the inverted lists. Step 4 scores from centroids only and skips tokens whose centroid's best query-token score is below t_cs (0.4–0.5). That is where PLAID puts centroid pruning, and the expansion "Performance-optimized Late Interaction Driver" is right. The shared-across-dimensions bucket note matches `colbert-ai` (global `quantile` over the residual matrix). D10 is applied ("already holds about 100 million chunks (Chapter 17) … multiply the storage figures by ten"). "Grows to" no longer appears for Acme at 10M anywhere in the book. I recomputed the numbers: 1.024 TB, 30.7 GB, 72 GB (2-bit) and 40 GB (1-bit), 8.39M dot products, and 512/36 = 14.2×, 512/20 = 25.6×. Encoding 10M passages at 100–300 ms per 1,000 takes 17–50 min, which fits "~0.5–1 h". 44 and 4.4 TFLOP are right. ×10 for all of Acme gives 307 GB single-vector, which agrees with Ch 24's 307 GB. I ran both Python blocks on toy data (3,000 docs, 148k clustered 128-d tokens, 512 centroids). Packed codes are 16 and 32 B plus a 4-B uint32 ID. For `plaid_candidates` (nprobe 2, t 0.45, ndocs 100), the target doc was in the shortlist 40/40 and top-1 after decode + exact MaxSim 40/40. Recall@10 against brute-force MaxSim was 0.76, and the all-pruned edge case runs. Ch 36 ("centroid-based search"), Ch 20, 38, 47 (36 B/vector), 61 (1M × 250 × 36 B ≈ 9 GB) and 66 (36 B/token, ~7,200 B/passage, PLAID pointer) still agree. The "~50,000 candidate passages" figure was not changed and remains unverified.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Missing PLAID second unpruned centroid pass before decompression — fixed differently: one sentence added at the end of Step 4 ("keeps the top `ndocs` (about a thousand), scores them once more from centroids without pruning, and keeps only the best quarter"). The code now does that pass too (re-scores the top `ndocs` on all tokens, returns `ndocs // 4`), so code and prose agree. The diagram gains a "Step 4, second pass … ~250 passages" line. Step 5 now reads "the few hundred to a thousand passages left". For consistency, the summary and one-paragraph version say "a few hundred to a thousand" (was "about a thousand"), the Key takeaway says "rough centroid-only scoring (pruned, then unpruned)", and the closing line says "the right few hundred passages".
- [NEW-NIT] "at most about a hundred" centroids — fixed ("at most 128 (32 × 4)")
- Code: extracted both Python blocks verbatim and ran them on toy data (3,000 docs, 148k clustered unit 128-d tokens, 512 centroids). Packed codes are 16 B (1-bit) and 32 B (2-bit) plus a 4-B uint32 ID, as the text says. `plaid_candidates` (nprobe 2, t 0.45, ndocs 400 → 100): target doc in shortlist 40/40, top-1 after decode + exact MaxSim 40/40, recall@10 vs brute force 0.85. Defaults return 256 = 1,024 / 4. The all-pruned edge case (threshold 9) runs.
- Cross-chapter follow-ups: none (Ch 20, 36, 38, 66 mention PLAID only in general terms and still agree)

### Post-fix check: Chapter 38  (verifier: chapters 34–41)

- **[NEW-NIT]** The Key takeaway still carries the wording the MUST fix narrowed in the body. Quote: "In practice a fixed few-thousand-d FDE works well." The body now says "shortlists well regardless of document length, as long as exact MaxSim reranks the shortlist". Fix: "In practice a fixed few-thousand-d FDE shortlists well, with exact MaxSim reranking."
- **[NEW-NIT]** The "six-fold" figure depends on the toy data used, and a single number overstates how general it is. Quote: "the ratio of FDE score to Chamfer fell about six-fold from 4-token pages to 200-token pages". I reran the chapter's `FDE` (reps = 16) on my own clustered unit 128-d data with 32-token queries. The ratio fell 4.85 → 4.13 → 2.39 → 1.61 for 4/16/64/200-token pages, about three-fold (reps = 10: 2.83 → 0.98). The direction is confirmed, but not the magnitude. Fix: "fell several-fold".
- Verified: The `FDE` block runs, and `reps=16` gives output length 4,096 (reps = 10 gives 2,560). "16 × 16 × 16 = 4,096" is stated consistently in the body, the code paragraph and the Key takeaway. The Ch 47 ("about 4,000 dimensions", "~4,000-d"), Ch 61 ("~4,000-number", "A 4,000-number FDE in float32 is 16 KB") and Ch 66 ("~4,000-d in practice") wording still agrees with 4,096. So does Ch 61's `fde.encode(q_vecs, is_query=True)`. The new Ch 21 wording is accurate: Ch 21's OR step is a union over L tables, while FDE repetitions are summed. "Estimate usually runs low" is correct. Without projection, each query token meets an average of page tokens or a single fill token, and neither can exceed its max match. This agrees with the hand example (1.80 vs 1.97). `$$` blocks are balanced.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Key takeaway "few-thousand-d FDE works well" too broad — fixed ("In practice a fixed few-thousand-d FDE shortlists well when exact MaxSim reranks."). The same claim in What people get wrong ("is what works in practice") is aligned too: "shortlists well in practice when exact MaxSim reranks".
- [NEW-NIT] "about six-fold" FDE/Chamfer ratio fall overstated — fixed ("fell several-fold")
- Code: no code changed
- Cross-chapter follow-ups: none (Ch 47, 61, 66 FDE wording does not repeat the "works well" or "six-fold" claim)

### Post-fix check: Chapter 39  (verifier: chapters 34–41)

No new issues.
- Verified: log1p(20) = 3.04 → "3.0" and log1p(2) = 1.10 → "1.1" are correct. The new explanation (per-position vote strength, a "cousin" of BM25's k1 that flattens how often) agrees with the formula, with Phase 1 Step 3 ("so big scores grow slowly") and with Ch 8's k1 definition, and the Key takeaway now matches the body. The BGE-M3 note is correct: its lexical weights are ReLU weights on the input's own tokens, matched on shared terms, with no expansion. It appears in both the Ninja note and the takeaway, and Ch 16's "keyword-style" BGE-M3 mention does not contradict it. "SAML" → `sam` + `##l` checked with a WordPiece pass over the local bert-base-uncased vocab.txt (`sso` → `ss ##o`, `login` → `log ##in` also confirmed). No code changed, and `$$` blocks are balanced.

### Post-fix check: Chapter 40  (verifier: chapters 34–41)

- **[NEW-SHOULD]** The rewritten paraphrase-twin paragraph calls the fused result "progress", but by the chapter's own lists it is a regression from dense search alone, and the text never says so. Dense's top 3 (page 212, SAML setup guide, page 1,140) holds both pages the answer needs, which is why the chapter says earlier "Dense search got the paraphrase right". Fusion pushes page 1,140 out of the top 3, so the answer goes from correct (dense) to wrong (fused). That is left unacknowledged, and it contradicts "So running both should cover both" (Why we need hybrid search) and the Key takeaway "Running both covers both." Quote: "Page 212 is now in the top 3, which is progress. But page 1,140 (1/63 ≈ 0.0159) ties with team management for fifth." Fix: "Compared with BM25 alone, page 212 is now in the top 3. But dense search alone had both page 212 and page 1,140 in its top 3, and fusion pushed page 1,140 down to a tie for fifth (1/63 ≈ 0.0159)." In the Note, add "At a cut of 3, fusion can cost us a page that one retriever ranked well." Change the takeaway to "Running both covers both in the fused shortlist of ~100, which a reranker then orders."
- **[NEW-NIT]** The consensus qualifier was added to the body but not to the matching Key takeaway. Quote: "$k = 60$ makes appearing on *both* lists matter more than topping *one*." Fix: append ", for pages in about the top 60 of both".
- Verified: I recomputed the paraphrase-twin RRF (k = 60): password reset 1/61 + 1/64 = 0.0320, account settings 1/62 + 1/65 = 0.0315, page 212 1/61 = 0.0164, SAML guide 0.0161, page 1,140 and team management both 1/63 = 0.0159 (tied fifth), company profile 0.0156, session timeout 0.0154. "Two pages that both consultants listed pushed it down" is correct (dense rank 3 → fused 5). The top-60 qualifier holds: 2/120 and 2/121 beat 1/61, and 2/122 ties it. D3 is applied in the summary ("cheapest, most reliable recall win in this book") and in What people get wrong. It agrees with the OUTLINE ("the most reliable free win in retrieval"), Ch 7 ("the cheapest large quality win"), Ch 41 and the What's next line ("highest-precision component"). The convex-fusion table row matches Bruch, Gai & Ingber ("agnostic to the choice of score normalization"). `convex_fusion`'s new comment is accurate (min-max maps the lowest hit to 0.0). Tables have 3 columns, and fences and `$$` are balanced.

#### Resolution (applied 2026-09-16)

- [NEW-SHOULD] Paraphrase-twin fusion is a regression from dense, left unacknowledged — fixed. The paragraph now says "Compared with BM25 alone, page 212 is now in the top 3. But dense search alone had both page 212 and page 1,140 in its top 3, and fusion pushed page 1,140 down to a tie for fifth … The answer is wrong. On this question, plain fusion did worse than dense search alone." The Note now says "At a cut of 3, fusion can cost us a page that one retriever ranked well" and explains the fix: fuse 100 deep so page 1,140 stays in the shortlist, then a reranker reads the question with each page and moves the right ones up (Chapter 41). "Dense search got the paraphrase right" now names its top 3 (pages 212 and 1,140) and adds "that holds for the fused shortlist, but not always for its top 3". The Key takeaway now reads "Running both covers both in the fused shortlist of ~100. Its top few can still be worse than one retriever alone, so a reranker orders it."
- [NEW-NIT] k = 60 takeaway missing the top-60 qualifier — fixed (", for pages in about the top 60 of both")
- Code: no code changed. Recomputed the paraphrase-twin fusion with the chapter's own `rrf`: password reset 0.0320, account settings 0.0315, page 212 0.0164, SAML guide 0.0161, team management and page 1,140 0.0159 (tied fifth), company profile 0.0156, session timeout 0.0154. These match the text.
- Cross-chapter follow-ups: Ch 7 (07-dense-vs-sparse.md, lines ~116-119) says dense search brings back page 212 alone and "The answer is correct" for the paraphrase twin. Ch 40 treats page 212 without page 1,140 as wrong (no Security add-on caveat). This mismatch predates the fixes. A possible fix in Ch 7 is to say "brings back page 212 (and page 1,140 with its Security add-on catch)", or to leave it and accept that Ch 7 uses the simpler version.

### Post-fix check: Chapter 41  (verifier: chapters 34–41)

- **[NEW-NIT]** The lead's hedge from the Ch 12 follow-up changed the one-paragraph version but not the matching Key takeaway. Quote: "Far more accurate than a bi-encoder, far too slow for corpus-scale search." The body now says "usually far more accurate than a bi-encoder of similar size". Fix: "Usually far more accurate than a bi-encoder of similar size, far too slow for corpus-scale search."
- Verified: I ran `rank_window` + `llm_rerank`, extracted verbatim, with a mock `llm`/`parse_ranking` on 100 candidates, with page 1,140 at rank 23 and a slightly lower true relevance than page 212. It made 9 calls and returned 100 unique items, the same set as the input. Page 1,140 was first read in call 7, read again in calls 8–9, and ended at rank 2. So "9 calls", "first read in the seventh call" and "climbs toward the top in the last two" are correct. With a malformed reply (duplicates, 99, −1, half the numbers missing) there was still no loss or duplication. 95, 21, 20 and 12 candidates also work. With 0 candidates it still makes one empty LLM call, which is harmless for a sketch. The window-20, step-10, back-to-front description matches RankGPT. `rerank` ran with a mocked `CrossEncoder` (top 10, highest first). `bge-reranker-base` matches Ch 13. D3 wording is exact in the body ("Once retrieval is decent (ideally hybrid, Chapter 40), adding a reranker is typically the largest remaining quality improvement"), the summary ("the highest-precision component in the pipeline"), the Key takeaway and What people get wrong. The OUTLINE Ch 41 one-liner reads "The largest precision win once retrieval works." No other chapter still says "single largest" for rerankers, and Ch 12 says "largest remaining quality jump". "About one and a half extra points" matches the +9 → +10.5 table. The MMR formula (`d \in R \setminus S`) matches the code's `rest`. Tables and fences are intact.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Key takeaway lacks the "usually … of similar size" hedge — fixed
- Code: no code changed
- Cross-chapter follow-ups: none (Ch 12 already says "usually far more accurate than a bi-encoder of similar size")

### Post-fix check: Chapter 42  (verifier: chapters 42–50)

No new issues.

- Verified: the only change is the ResNet-50 gloss, which reads cleanly and resolves the NIT. The temperature text (divide by a learned temperature starting at 0.07, logit scale capped at 100) is unchanged as D11 requires and is consistent with the new Ch 43 note.

### Post-fix check: Chapter 43  (verifier: chapters 42–50)

No new issues.

- Verified: the rewritten bias Note now matches SigLIP §3.2 (many negatives dominate the loss at a 50/50 start and force large early corrections, so b = −10 starts training near the prior), and σ(0·10 − 10) ≈ 4.5e-5 confirms "no" is the starting guess. The new temperature Note is correct and matches Ch 42 (Ch 42 divides by a temperature starting at 0.07; SigLIP multiplies by t, t′ = log 10 so t starts at 10; ×10 = ÷0.1), and the worked example's "multiplies by t" for the CLIP softmax is consistent with it. Re-ran the toy numbers (softmax 0.971 / 0.541, σ values 0.77, 0.73, 0.05, 0.03, 0.02 unchanged). The one-paragraph wording, the "at least quadruples encoding compute" line (MLP linear, attention quadratic in tokens, so 4–16×), and the added `from PIL import Image` / `Image.open("office_dog.jpg")` lines are correct. The transformers block cannot run here (no torch). Fences and `$$` are balanced.

### Post-fix check: Chapter 44  (verifier: chapters 42–50)

- **[NEW-NIT]** With the new Loss 7, the section headed as a per-stage list now has seven losses for a six-stage pipeline, and Ch 61 says "six small losses multiplied (0.9⁶ ≈ 53%, Chapter 44)". A careful reader counting will see 7 vs 6. Quote: "## What each stage loses" (and the intro "exactly what each one throws away", plus the "We will cover" bullet "What each stage loses"). Fix: retitle the section and its bullet "What the pipeline loses", as the original NIT offered. The intro already says "Let's go through the losses one kind at a time". Leave the 0.9⁶ stage arithmetic as it is.
- **[NEW-NIT]** (Wording predates the fix, but the lead asked whether the chunk listing matches the OCR text.) Stage 5 says chunk 1 "starts with the page heading, *Schedule B: Pricing for Globex Corporation*", but the Stage 4 block it cuts from has no heading line. The block is only the table's text, and D1 has Ch 61 copy it word for word. Quote: "The first chunk starts with the page heading, *Schedule B: Pricing for Globex Corporation*, and ends". Fix: "The first chunk starts with the page heading above the table, *Schedule B: Pricing for Globex Corporation*, and ends", so the Stage 4 block (and the Ch 61 copy) can stay unchanged.
- Verified: the chunk listing now matches the Stage 4 text. Cutting after `$4 / seat` leaves "Item Pro licence Security add-on S5O Price $15 / seat $4 / seat" (three items, two prices) in chunk 1, and puts "$0 Terms 120 seats, annual optional included (Pro, 50+ seats)" in chunk 2. The Scholar's new reasoning (three items but two prices, so SSO seems tied to the $4 add-on, page 1,140 agrees, seat count unseen) follows from those chunks. "The words that decided the question, `$0` and *included*", Loss 3 ("Flatten it and cut it", with `120 seats, annual optional included` really present in the flat string), Loss 7 and the Key takeaway all agree with it. "Even a human cannot…" is gone. Stage counts stay six everywhere: Ch 44 (frontmatter, diagram, 0.9⁶, table, "seven stages instead of six", takeaways), Ch 45 ("Two stages instead of six", "six-stage diagram"), Ch 61 ("six-stage pipeline", 0.9⁶ ≈ 53%, and its OCR block is identical to Ch 44 Stage 4). Ch 46 and Ch 66 have no stage or loss counts. Ch 45 lines 67–69 ("split it across two chunks… never saw the word *included*") still hold. The VLM cost wording now matches the table's "Highest". `audit_extraction` ran with real pandas on 4 files (fewer than 20, no ValueError): it flags the blank page (`empty` True at < 100), the character-split page (avg_word_len 1.04) and the mojibake page (nonascii 0.9), and passes the flattened table.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Seven losses under a "What each stage loses" heading — fixed. The section heading and its "We will cover" bullet now read "What the pipeline loses", and the intro says "exactly what the pipeline throws away" (was "what each one throws away"). The 0.9⁶ stage arithmetic is unchanged.
- [NEW-NIT] Stage 5 heading not in the Stage 4 text block — fixed ("starts with the page heading above the table, *Schedule B: …*"). The Stage 4 block is unchanged, so Ch 61's copy still matches.
- Code: no code changed
- Cross-chapter follow-ups: none (no other chapter cites the section title. Ch 45, 46 and 61 describe the heading as a separate part of the page, which agrees.)

### Post-fix check: Chapter 45  (verifier: chapters 42–50)

- **[NEW-NIT]** The new sentence points ahead to "Problem 1" without saying where it is. "Problems with ColPali" comes two sections later, so a reader meets the label before the list it belongs to. Quote: "stay in the pixels, subject only to the resize in Problem 1." Fix: "stay in the pixels, subject only to the 448 × 448 resize (see Problems with ColPali below)."
- Verified: the diagram now has "1,024 patch embeddings + ~6 prompt tokens" going into Gemma-2B and "~1,030 contextualised vectors" coming out. That agrees with Steps 4–5 and with Ch 46's "Gemma's final vectors for every patch and prompt token", and 1,024 + 6 = 1,030. The code-to-steps mapping (`process_images` resizes, `model(**batch)` cuts patches and runs Steps 3–5) is correct for colpali_engine. "40,000 pages of the Library" matches Ch 1–8. Stage counts ("Two stages instead of six", "six-stage diagram") agree with Ch 44, and the recap at lines 67–69 still fits Ch 44's new cut after `$4 / seat`. No code changed. Fences are balanced.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Forward reference to "Problem 1" before that list appears — fixed ("subject only to the 448 × 448 resize (see Problems with ColPali below)")
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 46  (verifier: chapters 42–50)

No new issues.

- Verified: "about 119,000 (query, page) pairs (118,695)" appears in the one-paragraph version, the body and Key takeaways, and no 127,000 is left anywhere in the book. The new loss is correct. $\log(1+e^{s_\text{wrong}-s_\text{true}})$ is softplus, and a NumPy harness (4 queries, 1,030×128 unit pages, MaxSim, hardest in-batch wrong page) gives the same value as a 2-way softmax cross-entropy (0.12066 both). "No softmax over every page… no temperature" matches D9 and the ColPali paper. Steps 3–4, the one-paragraph version and the new takeaway agree, and "ColBERTv1's pairwise loss (Chapter 36)" matches Ch 36 lines 233–235. The padding Note is correct: a zeroed query row leaves the MaxSim sum unchanged (3.5009 → 3.5009), fixed-length pages have no page-side padding, and it cites Ch 36's warning at lines 403–406. The LoRA wording is consistent in all four places and in the docstring. The "Chapter 35's `maxsim`" reference is right (Ch 35 line 320). Fences and `$$` are balanced. Only the docstring changed in code.

### Post-fix check: Chapter 47  (verifier: chapters 42–50)

- **[NEW-SHOULD]** The new "Serving float32" item puts the rescoring copy on the wrong storage tier compared with Ch 60 and Ch 61. Ch 60 keeps query-time rescoring vectors on SSD ("Binary + rescore from SSD | ~250 RAM + 307 GB SSD"). It keeps the object-storage float32 copy as the source of truth for rebuilds ("the vectors let you rebuild without re-embedding"). Ch 61's object-storage row is "Full-precision pooled patch vectors (the truth)", and Ch 61 reranks with MaxSim over the compressed multi-vector store inside a 150 ms budget, not by fetching from object storage. The item also now repeats the next-but-one item ("keep the full-precision vectors there too, for rescoring and future migrations"). Quote: "Quantize it, and keep the float32 copy in object storage for rescoring." Fix: "**Serving float32.** There is no reason to keep float32 in the hot index. Quantize it. Keep the float32 copy in object storage as the source of truth for rebuilds, and keep any higher-precision vectors that query-time rescoring needs on SSD (Chapter 60)." Change line 445 to match: "keep the full-precision vectors there too, for rebuilds and future migrations".
- **[NEW-NIT]** The added "no GPU" does not fit the sentence. "Do not use it alone for … no GPU" is ungrammatical. It is also wrong in substance: without a GPU the body says not to use ColPali at all ("Encoding a million pages on CPU is not practical"), and the hybrid does not rescue that. Quote: "Do not use it alone for plain prose, exact codes, sub-20 ms budgets, no GPU, or when you need the text itself." Fix: "Do not use it alone for plain prose, exact codes, sub-20 ms budgets, or when you need the text itself. Without a GPU, do not use it at all."
- Verified: the reordered ColQwen2 steps (round to 28 → cap → cut into 14-px patches merged 2×2) now match `visual_tokens` and Qwen2-VL's `smart_resize`. No sentence refers to the old step numbers (the only later "Step 3" reference, at line 387, is to the pooling steps). Re-ran `visual_tokens`: 736 and 1,472 as commented. The "We will cover" bullet now matches the Lever 1 heading. No code changed.

#### Resolution (applied 2026-09-16)

- [NEW-SHOULD] Float32 rescoring copy on the wrong tier, and the advice duplicated — fixed differently. "Serving float32" now says "Quantize it. Keep the float32 copy in object storage as the source of truth for rebuilds and index migrations. If query-time rescoring needs higher-precision vectors, serve them from SSD or RAM, not from object storage (Chapter 60)." The duplicate went out of "Discarding the page images", which now only says to keep the images in object storage. That is cleaner than restating it as "for rebuilds and future migrations". Lever 3's "keep higher-precision vectors somewhere for rescoring" still agrees.
- [NEW-NIT] Ungrammatical "no GPU" in the Key takeaway — fixed ("… sub-20 ms budgets, or when you need the text itself. Without a GPU, do not use it at all.")
- Code: no code changed
- Cross-chapter follow-ups: optional, low priority. Ch 31 (31-hnsw-production.md:327-328) lists "rescore" among the reasons to keep full-precision vectors, then says "Keep full-precision vectors in object storage". That can read as query-time rescoring from object storage. A possible fix: "you cannot rebuild, migrate models, or change parameters … (rescoring copies belong on SSD, Chapter 60)". Ch 60 and Ch 61 agree with the new Ch 47 wording.

### Post-fix check: Chapter 48  (verifier: chapters 42–50)

No new issues.

- Verified: the "same *idea*… **The exact loss varies, but its most common form is these four lines.**" wording matches D9, and it no longer conflicts with Ch 46's hardest-negative softplus loss, Ch 36's KL distillation or Ch 9's sigmoid negative sampling. "Four lines" still matches the `loss` body and Ch 12's "Four lines". The Note's saturation claim holds: N(0,1) 128-d rows divided by 0.05 give mean |logit| ≈ 180 and a max of ≈ 970, and `tau` = 1 or a learned `tau` is the right fix. The BM25/tokenizer, "rarely share a walk", "one of the oldest", and GAT-expansion edits are correct and read cleanly. No code changed.

### Post-fix check: Chapter 49  (verifier: chapters 42–50)

No new issues.

- Verified: the code gloss now explains that `k=5` fetches five chunks, pages 212 and 1,140 rank first and second, and the other three are uncited near misses. That fits the prompt's "Cite sources as [1], [2]" and the worked example's "brings back two pages". The table cell "High (hundreds of thousands to a million)" matches the million-token-window discussion. The table still has 3 columns. No code changed.

### Post-fix check: Chapter 50  (verifier: chapters 42–50)

- **[NEW-NIT]** The new fence check only recognises backtick fences at the start of a line. A `~~~` fence, or a fenced block indented inside a list item, is still read as prose, so its `#` comment lines become section headings. That contradicts the new gloss, which states the rule generally. Run results: `"# A\n\n~~~python\n# comment in tilde fence\n~~~"` gives the sections `('A', '\n~~~python')` and `('comment in tilde fence', '~~~')`. Quote: "if line.startswith(\"`\" * 3):            # a code fence opens or closes". Fix: `if line.lstrip().startswith(("`" * 3, "~~~")):`. Alternatively, keep the code and narrow the gloss to "Inside a ``` code block, a `#` line is a comment".
- Verified: extracted `split_by_headings` from the chapter and ran it with a stub `split_if_long`. On the chapter's page 1,140 outline, raw and with the ← annotations stripped, it returns only "Audit logs" and "Plans and seats". There is no empty "Security add-on" title chunk, exactly as the new gloss says. The elided "IP allow-lists" / "Data retention" stubs are dropped too, which is correct for empty bodies. On a page with a ```bash block holding `# install the CLI` and `## not a heading either`, the block stays inside "Install guide" and "Configure" is split correctly. "About a third" holds (15/47 ≈ 32% with title and heading prepended). The top-20 failure-rate gloss matches Anthropic's metric (share of queries whose relevant chunk is not in the top 20). The scanned-pages row now has 3 cells, matching the 3-column header. No em-dashes remain outside the frontmatter. Fences are balanced (12 fence lines). The `late_chunk` changes are comment-only.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Fence check misses `~~~` and indented fences — fixed (`if line.lstrip().startswith(("`" * 3, "~~~")):`). The gloss "Inside a code block, a `#` line is a comment" now holds as written.
- Code: extracted `split_by_headings` verbatim and ran it with a stub `split_if_long`. On page 1,140's outline (raw and with the ← notes stripped) it returns only "Audit logs" and "Plans and seats". The tilde fence case `# A\n\n~~~python\n# comment…\n~~~` now gives one section ('A'). A ```bash block holding `# install the CLI` / `## not a heading either` stays inside "Install guide", and "Configure" still splits. A fence indented inside a list item (with a flush-left `#` comment) stays in "Steps", and "Next" still splits.
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 51  (verifier: chapters 51–58)

- **[NEW-NIT]** Ch 51 now says "title and section heading" in its heading and Key takeaway, but the Ch 66 ingestion checklist still says "breadcrumb". Quote (66-field-manual.md): "- [ ] Title/breadcrumb prepended before embedding". Fix: "- [ ] Title and section heading prepended before embedding" (this also matches Ch 50's "title prepend").
- Verified: I extracted `recency_boost` and `score_with_metadata` and ran them with dates relative to 2026-09-16. `recency_boost` gives 0.0505 and 0.7127, matching the prose's ≈0.05 and ≈0.71, and 0.5^(730/180) = 6.0% matches "about 6%". In the scorer, only `forum_post` and `release_note` decay. The two-year-old forum post falls to 0.0505, while kb_212 and kb_1140 (`updated_at` 2026-01-22) and a June 2026 contract keep their full scores. The superseded page is dropped, and the cap of two chunks per doc works. With no `language` in `query_meta` there is no KeyError. So the gloss "Pages 212 and 1,140 are not decaying types, so they keep their full scores" is correct. The new first example (page 212 first, caveat missing because 1,140 misses the top 5) is consistent with rank 14 in "Structure between documents", with Ch 52's "14th, as in Chapter 51, outside the top 5" and with Ch 56's "rank 14, as in Chapter 51". The framing sentence now appears once, before the first example. The Key takeaway was updated. The CMS expansion and the "title and section heading" wording agree with the code and with Ch 50 (lines 127, 316). The table column counts are intact, and the fences are balanced.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Ch 66 checklist still said "Title/breadcrumb" — fixed in Ch 66: "Title and section heading prepended before embedding".
- [Lead, follow-up from Ch 53 fixer] page 1,140's chunk metadata had a section as its parent — fixed: `"parent": "kb_1140"`, matching Ch 50 and Ch 53 (a short page is its own parent).

### Post-fix check: Chapter 52  (verifier: chapters 51–58)

- **[NEW-SHOULD]** The new `multi_hop()` applies the filters extracted from the original question to every hop. By the chapter's own rule ("Extract dates, page types and names into exact filters", and Ch 51's `"who signed the Globex contract" → doc_type: contract, entity: Globex`), the Globex question yields a Globex entity filter. That filter would then block hop 2 from finding page 1,140, which is not a Globex document. So the code as written fails its own worked example once `extract_filters` does what the chapter says. Quote: "        hits += hybrid_search(next_q, filters)    # one search, then the next hop". Fix: let each hop choose its own filters, e.g. `next_q, hop_filters = write_next_search(q, hits, filters)   # one LLM call: what is still missing, and where?` and then `hits += hybrid_search(next_q, hop_filters)`. Alternatively, apply `filters` only on the first hop.
- **[NEW-NIT]** The cost in the table undercounts LLM calls. `multi_hop` makes one more LLM call after the last search to decide that the hits already answer the question. In a stubbed run, the Globex question made 3 LLM calls and 2 searches. Quote: "(multi-hop: one LLM call and one search per hop, in sequence, Ch. 56)". Fix: "(multi-hop: one LLM call and one search per hop, in sequence, plus a final LLM call to stop, Ch. 56)". Or leave the table as it is and change the gloss to "one LLM call and one search per hop, then one last call to stop".
- Verified: I ran `REWRITE.format()` and it formats cleanly, with the new identifier-preservation line. I extracted `understand`/`retrieve`/`multi_hop` and ran them with stubs. The identifier path gets `sparse_weight: 2.0`. The compound path runs two searches fused by RRF (kb_212, then kb_1140). Exploratory runs 4 searches, and lookup runs 1. Globex runs strictly in order: LLM → "Globex contract" search → LLM sees p3507/p3508 → second search → LLM stops. "one of five kinds" matches the five intents. The latency line "often more than half a second" agrees with the review's ≈675 ms derivation and with Ch 60's "LLM generation | 500–5,000 ms". The Key takeaway ("so their hops run in order") matches the body. "help article" (not "long") and "almost no words" now match Ch 50 and this chapter's HyDE section. The table still has 4 columns.

#### Resolution (applied 2026-09-16)

- [NEW-SHOULD] multi_hop reuses question filters, blocking hop 2 — fixed (write_next_search(q, hits, filters) now returns next_q and hop_filters, and hybrid_search uses hop_filters; the gloss says hop 1 keeps the Globex filter and hop 2 drops it, because page 1,140 is not a Globex page)
- [NEW-NIT] Cost gloss undercounts the final stop LLM call — fixed (table: "plus a final LLM call to stop"; gloss: "then one last call to stop")
- Code: extracted the Under-the-hood block and ran it with a stub LLM and a stub hybrid_search that enforces metadata filters exactly (pf5260/ch52.py). Globex question: hop 1 "Globex contract" with {doc_type: contract, entity: Globex} → [3507, 3508]; hop 2 with {} → [1140, 3507, 212]; then stop. 3 LLM calls, 2 searches. The same run through retrieve() gives the same result. Control: when every hop reuses the question's filters, hop 2 returns only [3507] and never reaches 1,140.
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 53  (verifier: chapters 51–58)

- **[NEW-NIT]** The new framing cites "Chapter 50's parent-child chunks", but Ch 50 applies that technique to page 1,140 with the *whole page* as the parent ("The page is short, so we let the whole page be the parent"). Here the caveat child expands only to the 40-token "Plans and seats" section. A reader following the citation gets a different parent for the same page. Quote: "this pipeline indexes small child chunks of up to about 200 tokens (Chapter 50's parent-child chunks)". Fix: "this pipeline indexes small child chunks of up to about 200 tokens, with each section as the parent (Chapter 50's parent-child chunks)". Also acceptable: leave Ch 53 as it is and drop "The page is short, so we let the whole page be the parent" from Ch 50.
- Verified: I extracted all 3 Python blocks and ran them. `order_for_attention([1..6])` = [1,3,5,6,4,2]. On the stated post-rerank list [212, 1,140, Okta, Azure AD, Team roles] it gives [212, Okta, Team roles, Azure AD, 1,140], exactly as the prose says. Without the rerank it gives 1,140 in the middle, so the MUST is resolved. `dedupe` at 0.97 keeps a 0.95 pair and merges a 0.99 pair, which matches Problem 1. `assemble` with stubs (rerank lifts the expanded 1,140 section to second) makes 1,140 source 5. It emits `section=` as in the Step 5 example, escapes `</source>` in page text to `&lt;/source&gt;`, and wraps history in `<history>` tags. The rerank-after-expansion is stated consistently in "Choosing what goes in" Step 2, the example's Steps 2 and 4, the code comment, the gloss and the Key takeaway. "In rerank order" (budget fill) still holds. "The retrieved chunks did not change" is accurate. The 1,024-token cache minimum is right for OpenAI and many Anthropic models (some need more, which "so check yours" covers). The "small child chunks" framing agrees with Ch 51's new "unless a chapter says otherwise" and with Ch 50's child size ("up to about 200 tokens"). No other chapter quotes the old 0.92 threshold. Fences and tables render.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Page 1,140's parent differs from Ch 50 (section vs whole page) — fixed differently: rather than only adding "with each section as the parent" (which still gives page 1,140 a different parent from Ch 50), Ch 53 now states Ch 50's full rule ("Each child's parent is its section, or the whole page when the page is short") and page 1,140's caveat chunk expands to the whole page, as in Ch 50. Made consistent: Step 2 bullet ("To the parent"), the dedupe-again sentence ("chunks with one parent"), the budget example (caveat now sits mid-page after the audit-log section, so a cut could keep audit logs and lose it), the Step 5 source label (whole page, page URL, no section attribute), example Step 2, and the code (expand_to_parents; the label no longer emits section=, matching "ID, title, date and URL" in Step 5 and Key takeaways)
- Code: extracted all 3 Python blocks from the edited chapter and ran them with stubs (pf5260/ch53.py). order_for_attention([1..6]) = [1,3,5,6,4,2]. dedupe keeps a 0.95 pair and merges a 0.99 pair. assemble: two children of page 1,140 collapse into one whole-page parent, rerank lifts it to second, and the attention order is [212, Okta, Team roles, Azure AD, 1,140], so 1,140 is source 5. The label matches the Step 5 example (title, updated, url, whole page body), and `</source>` in page text is escaped. With a budget 1 token short, 1,140 is dropped whole.
- Cross-chapter follow-ups: chapters/51-metadata-and-structure.md line 69 gives page 1,140's chunk `"parent": "kb_1140#s02"` (a section). Ch 50 (line 286–287) and now Ch 53 make the whole short page the parent, so it should be `"parent": "kb_1140"` (or Ch 50 should drop "The page is short, so we let the whole page be the parent" and Ch 53 revert to section parents). Pre-existing mismatch, not caused by this fix.

### Post-fix check: Chapter 54  (verifier: chapters 51–58)

No new issues.

- Verified: 0.25 × (1.96/0.05)² = 384.2, which rounds up to 385. At n = 385 the SE is √(0.25/385) = 0.0255 (±2.5) and the 95% interval is ±4.99 (±5), so the new table row is right. Two unpaired runs need 768 per run for a ±5-point interval on the difference ("about twice as many per run" holds). With the paired test at n = 200, a net 5-point gain is significant whenever at most about 13% of questions flip (10% flipped gives ±4.3 points, 20% gives ±6.2), so "often shows a 5-point gain with about 200 questions" is fair, and it matches Ch 19's paired-test advice (lines 443–454). The Key takeaway (200 + paired, 385 for ±5 on one score) matches the body. I parsed the YAML with PyYAML. Gold sources are page IDs (`kb_212`, `kb_1140`, `kb_17450`), which agree with Ch 51's `doc_id` and with Ch 57's `chunk_id = hash(doc_id, …, chunker_version, chunk_index)`, which does change on re-chunking. I ran `evaluate()` with pandas 3.0.5 on the chapter's YAML, using a stub system whose chunk IDs match nothing stored and judges that return bools. The report keeps all columns: multi-page recall 0.5 and correctness 0.0, identifier 1.0/1.0, unanswerable refused 1.0. The answer-relevancy description (no reference answer, cannot tell a relevant wrong answer from a right one) and the Cohen's kappa gloss are accurate. The one-paragraph version's "relevant" is now covered. The table column counts and `$$` blocks are intact.

### Post-fix check: Chapter 55  (verifier: chapters 51–58)

- **[NEW-NIT]** The new alias sentence lists only three of the five chapters that say "relevance floor". Ch 46 (line 365, "a calibrated relevance floor") and Ch 56 (line 140, table row "Relevance floor") use it too. Quote: "Chapters 49, 63 and 66 call it the relevance floor." Fix: "Chapters 46, 49, 56, 63 and 66 call it the relevance floor." Alternatively: "Other chapters, such as 49 and 63, call it the relevance floor."
- Verified: failure 8's new cause (one query vector in one place, and "under what conditions?" never said) is now about the question and is distinct from failure 2's dilution cause. The added line after Step 4 ("run failure 2's test too") keeps the example honest, given that Ch 51–56 start from whole-page fixed-size chunks. Step 3's test and the "F8" gloss still match. I extracted `diagnose()` and ran it with `doc_id`-keyed stubs whose chunk IDs match nothing in the gold set: partial → F8, full+grounded → "F11, F12", full+ungrounded → F10, gold only in candidates → "found but cut", BM25-only → F1, nothing → F2–F7, fabricated unanswerable → F9, refusal → OK. Gold sources are now page IDs, the same format as Ch 54's YAML (`kb_212`, `kb_1140`) and `evaluate()` (`c.doc_id`). "Distance floor" is now used consistently inside the chapter (bold definition, failure 9 cause, parts 1–3, takeaways). Failure 3's test clue now includes "what about", which matches its example. No code-fence or table breakage.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Alias sentence omits Ch 46 and Ch 56 — fixed ("Chapters 46, 49, 56, 63 and 66 call it the relevance floor."; confirmed by grep that exactly these five other chapters use the term; also rejoined the paragraph's broken line wrap)
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 56  (verifier: chapters 51–58)

- **[NEW-NIT]** The token budget is soft, but the gloss describes a hard stop. `used > max_tokens` is checked only before each call, so the call that crosses the budget still runs and its tool results are appended, and then `force_final_answer` makes one more call. In a stub run at 30,000 tokens per call, the loop stopped after 60,000 tokens plus the forced answer, against `max_tokens=50_000`. Quote: "The loop stops after `max_steps` calls or `max_tokens` tokens, whichever comes first, and forces an answer." Fix: "The loop stops once it has made `max_steps` calls or passed `max_tokens` tokens, whichever comes first, and forces an answer. The last call can overshoot the token budget, so set it with some headroom."
- Verified: I extracted `agentic_answer()` and ran it with a scripted stub LLM. The SSO script gives 3 model calls and 2 searches, which matches "two searches and three model calls" and the table. A never-answering stub stops after 6 calls and forces an answer. A 30k-per-call stub stops on the token budget after 2 calls, and the conversation ends on a `user` tool_result turn, so `force_final_answer` gets a valid history. The result-set sums are right: calls read 0, 1, 2 → 3 (vs 1 single-shot), and at 6 steps plus the forced answer they read 0 + 1 + … + 6 = 21, against 6× retrieval. Step 6, the prose below the code, "Cap iterations and total tokens" and What people get wrong ("Cap steps and tokens") all now match the code. The tool `limit` default of 5 matches "the same five pages". The new relevance-check paragraph matches Ch 55 (LLM relevance check, `answerable: false` sweep, phone-support-on-Pro as the hard case). The "trajectory" definition is in the table's third column with no extra pipe (3 columns intact). Ch 60's looser "Agentic loops multiply tokens" does not contradict it.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Token budget is soft but gloss says hard stop — fixed (chose the soft-limit prose over enforcing: a "likely cost" pre-check still cannot bound the tool results or the forced-answer call, so it would not make the cap hard. Gloss now: stops once it has made `max_steps` calls or used more than `max_tokens` tokens; the budget is a soft limit, checked only before each call, so the crossing call still runs and the forced answer is one more call; set it with headroom. Code comment now "# token budget passed (a soft limit)")
- Code: extracted agentic_answer() from the edited chapter and ran it with a scripted stub LLM (pf5260/ch56.py). SSO script: 3 model calls, 2 searches, answered. Never-answering stub: 6 calls, forced answer on a `user` tool_result turn. 30k-tokens-per-call stub: 2 loop calls (60,000 tokens, past the 50,000 budget), then the forced answer, 90,000 in total. This matches the new soft-limit wording.
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 57  (verifier: chapters 51–58)

No new issues.

- Verified: the "enrich" definition (Chapters 50–51) sits right after the pipeline definition. The "long poles" gloss is in place. The GPU-money vs wall-clock split ("most expensive stage in GPU money, unless we enrich chunks with an LLM. Extraction and OCR are usually the slowest in wall-clock time") agrees with the table note "parsing is often the slowest step", with "OCR and LLM enrichment are often the true long poles" and with the Under the hood lead-in "most of the GPU cost". The build row "one machine per shard, in parallel (Chapter 58)" matches Ch 58:71–72 and :360 (forty shards rebuild in parallel). The table still has 3 columns. int8 → Chapter 26 ("Scalar and Binary Quantization") and TurboQuant → Chapter 27 are correct. The `assert`/`python -O` sentence is accurate. No code changed, and 307 GB / 3 GB per million were left untouched and are still right (100M × 3,072 B = 307.2 GB).

### Post-fix check: Chapter 58  (verifier: chapters 51–58)

- **[NEW-NIT]** "A few whole buckets" understates the move. With 4,096 buckets, going from 40 to 41 shards gives the new shard about 4,096 / 41 ≈ 100 buckets. That is only two or three from each existing shard, but about 100 in total. Quote: "Adding a shard moves a few whole buckets onto it, about 1/N of the data." Fix: "Adding a shard moves two or three whole buckets from each existing shard onto it, about 1/N of the data."
- Verified: plain `hash % N` from 40 to 41 moves 97.6% on 1M random 64-bit hashes (exactly 1/41 stay, so "about 98%" and "only one document in 41" are right). With 4,096 virtual buckets, reassigning 99 buckets moves 2.4% of documents (≈ 1/41). The consistent-hashing description is accurate. 1 − (39/40)^10 = 0.224 ("about one query in five"), and 10 × 1/40 = 0.25 items = 2.5% of recall@10. I extracted the scatter-gather block and ran it with asyncio stub replicas (a fast shard, a slow-primary shard with a fast hedge, and a 300 ms/300 ms shard). It returns in 82 ms with coverage 0.67. The hedge wins and the slow primary is cancelled. Both slow replicas are already cancelled when `scatter_gather` returns and never finish later. A duplicate id from two shards appears once. The comparison-table row, the Strengths line and "as long as we ask every shard" are consistent with each other. "About 93 GB" matches Ch 60 (table "~930 | ~93 GB", takeaway "int8 ~93 GB") and Ch 66:90/245. The rescore depths (2–5× int8, 10–20× PQ per Ch 25:255/271, ~100× binary per Ch 26:251 "rescore those 1,000") match. The Ch 66 follow-ups were applied: 66:211 has the per-scheme over-fetch and 66:225 has "`hash(doc_id)` → virtual bucket → shard". Branch 7 among twenty is fine. The tables keep 6 and 2 columns.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] "A few whole buckets" understates the move — fixed ("Adding a shard moves two or three whole buckets from each existing shard onto it, about 1/N of the data."; checked 4,096 / 41 ≈ 100 buckets for the new shard, 100 / 40 ≈ 2.5 from each existing shard)
- Code: no code changed
- Cross-chapter follow-ups: none (no other file repeats the bucket-move claim)

### Post-fix check: Chapter 59  (verifier: chapters 59–66 + README/OUTLINE)

- **[NEW-SHOULD]** The new revert walkthrough happens "on 2 October", but the chapter's own nightly compaction has run by then, so the bug it describes cannot occur and "They now sit in both the main index and the delta" is false. Quote: "Now say a writer undoes the 1 October edit on 2 October." and "They now sit in both the main index and the delta, which is why `search` keeps one hit per id." Line 206 says "Tonight's compaction will build a main index without the old vectors at all", and `compact` sets `self.tombstones = set()`. After that compaction the old ids are neither tombstoned nor in the main index. I ran the ORIGINAL code (no `difference_update`, no dedup) with a compaction between the edit and the revert, and the reverted page 212 ranked first. The collision only happens when the revert arrives before the next compaction. Fix: "Now say a writer undoes the edit an hour later, before tonight's compaction." (and keep the rest of the paragraph).
- Verified: extracted the `FreshIndex` block and ran it with a NumPy flat main index, a flat delta and content-hash chunk ids. After the 1 October edit and delete, old page 212 and old page 1,140 are gone. After a same-day revert (no compaction), page 212 ranks first (score 1.0) with 0 duplicate ids in the top 10. After compaction it is still first, and 1,140 stays gone. The original code in the same scenario loses both versions of page 212, so the fix resolves the finding. Phase 1 Step 5 and Phase 2 Step 3 match the code. "make it likely that `k` survive" wording, page 1,140 retirement wording, "the product's 12.5 million documents" (matches Ch 57 line 238) and the Continuous calendar row (now identical in Ch 66 line 265) are all correct. The frontmatter semicolon is gone. Tables and fences render.

#### Resolution (applied 2026-09-16)

- [NEW-SHOULD] Revert on 2 October comes after nightly compaction — fixed ("Now say a writer undoes the 1 October edit an hour later, before tonight's compaction."; also "until the next compaction" → "until tonight's compaction" so every sentence names the same night. No other sentence in the book mentions the revert date)
- Code: prose only, the FreshIndex code is unchanged. Reran the scenario with the FreshIndex block from the edited chapter (pf5260/ch59.py: flat main and delta indexes, content-hash ids, exact cosines). 1 Oct edit + delete: new 212 first (0.84), old 1,140 gone. An hour later, revert with no compaction: old 212's id is in both main and delta (2 raw copies), and search returns it first (0.91) with 0 duplicate ids in the top 10. After tonight's compaction: still first, 0 tombstones, empty delta. Control without the difference_update line: after the revert no version of 212 is in the top 10, and after compaction 212 is back. This matches "could be found until tonight's compaction".
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 60  (verifier: chapters 59–66 + README/OUTLINE)

- **[NEW-NIT]** The new base-model ratio rounds 7.5 down. Quote: "It is ~20× a small model's re-embedding bill (~7× a base model's)". The chapter's own speeds give 1,500 ÷ 200 = 7.5 (138.9 ÷ 18.5 GPU-hours). Fix: "(~7.5× a base model's)".
- Verified: the lever table matches D4 row for row, character for character (script comparison), and so does Ch 66's copy. The intro line matches D4. The prose after the table, the memory-section sentence, the Key takeaway and the `llm_share` paragraph now all put the LLM levers first, with index levers first for huge, low-traffic corpora. D5 ladder wording is exact in the body and the Key takeaway, and "a hundred or more times" agrees with the toy prices ($5 ÷ $0.02 = 250×). Ch 65 line 132 and Ch 66 line 279 use the same ladder. Recomputed: int8 768 + 160 = 928 B → ~930, 92.8 GB → ~93 GB, 0.287 → 0.29×. Binary 96 + 160 = 256 B → 25.6 GB (the book's "~25 GB"). DiskANN 100M × 4,096 B = 409.6 GB → ~410 GB, matching Ch 32's 4.1 TB at 1B, with 48 B → 4.8 GB RAM. Quantize lever: whole index 3,232 ÷ 928 = 3.5×, ÷ 356 = 9.1×, ÷ 256 = 12.6× → "3.5–13×". Vector bytes alone 4× to 32×. The context cut is $45,000 ÷ $71,354 = 63%, so "more than half" holds. The LLM formula now divides by 1,000,000 in both lines. Ch 58 lines 70 and 330 now say ~93 GB. No leftover "92 GB", "330 GB", "4–30×" or "ten times at each step" anywhere in the book. Tables render.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Base-model ratio "~7×" should be ~7.5× — fixed ("~7.5× a base model's"; 1,500 ÷ 200 = 7.5, and 4,000 ÷ 200 = 20 confirms "~20×"; rewrapped the line)
- Code: no code changed
- Cross-chapter follow-ups: none. Ch 66 (lines 289–291) and Ch 63 (line 107) repeat the GPU-hour figures (~7 / ~19 / ~140) but not the ~7× ratio, and no file in the book contains "~7×".

### Post-fix check: Chapter 61  (verifier: chapters 59–66 + README/OUTLINE)

- **[NEW-NIT]** The new credit for page 3,508 says the Scholar learns from it that the contract is signed, but Ch 44 already puts a signature block on page 3,507, and the seat count and 50-seat rule both come from 3,507. Quote: "Without it, the Scholar would have known the terms but not that they belong to a signed contract that is still in force." Ch 44 line 157 says the second chunk of 3,507 lands "next to the signature block", and line 161 says that chunk "is about 'terms and signatures'". What only 3,508 adds (per D1) is the effective date. Fix: "Without it, the Scholar would have known the terms but not when the contract took effect, so it could not tell that this is the current contract." Optionally soften "Two details made the difference" to "Two details made the answer correct and trustworthy", because the answer would still be "yes" without 3,508.
- Verified: the OCR block is byte-identical to Ch 44's Stage 4 output (script comparison), and no "SS0" is left anywhere in the book (three `S5O` uses in Ch 61). Step 4 now puts 3,507 at BM25 rank 9 through its *Schedule B: Pricing for Globex Corporation* heading, which is the heading Ch 44 line 155 and Ch 45 line 157 use. RRF: 1/74 + 1/69 = 0.013514 + 0.014493 = 0.028006 ≈ 0.0280. A page in only one list can score at most 1/61 = 0.0164, so the page ranks at worst about 101st of 150 and "comfortably inside the top 150" holds. The FDE-only 0.0135 claim is also true. Step 6 no longer mentions a Pro column. Page 3,508 is the order form and signature page, confirming 120 Pro seats and a June 2026 effective date. That agrees with Ch 52 line 173 ("pages 3,507 and 3,508 … Pro with 120 seats") and Ch 64 ("Pro (120 seats) in June 2026"). The final answer matches D1 word for word. The new takeaway on re-encode vs rebuild agrees with the body (lines 438, 464). Code unchanged. Fences and tables render.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Page 3,508 credit: effective date, not signature — fixed (last paragraph now says without 3,508 the Scholar would not know when the contract took effect, so could not tell it is current; "Two details made the difference" → "Two details made the answer correct and trustworthy"). Step 9 reasoning and the final answer with citations to pp. 3,507 and 3,508 still agree (3,508 confirms the seats and carries the June 2026 date, per D1). No other "signed contract"/"in force" claim for 3,508 in the book.
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 62  (verifier: chapters 59–66 + README/OUTLINE)

- **[NEW-NIT]** Deletion Step 2 purges a cache that never holds the deleted document and misses caches that can. Quote: "**Step 2:** Purge the result cache and the query-embedding cache, so no cached answer quotes it." A query-embedding cache stores vectors of users' questions, not of the document, so purging it does nothing for "so no cached answer quotes it". Ch 57 line 98 lists the serving caches as "query embeddings · results · rerank scores · LLM prefix". The rerank-score cache and the LLM prefix cache are the ones that can still carry the document's ids or text. Fix: "**Step 2:** Purge every cache that can hold the document, such as the result cache and the rerank-score cache (Chapter 57), so no cached answer quotes it."
- Verified: the six-step deletion list follows D8 in order: tombstone, caches, delta and source store, compaction or rebuild on every replica before the legal deadline, backups on a documented retention period, audit entry. It cites Chapter 59 twice. Its tombstone gloss is word for word Ch 59 line 49. It covers exactly the places Ch 59 lines 262–263 list (caches, deltas, replicas, backups, source-of-truth store). "within minutes" matches Ch 59's "Minutes, guaranteed" row. The noisy-neighbour wording (dedicated shards for large tenants, packed small tenants) agrees with Ch 58 line 186, and the benefit, Advantages bullet and Key takeaway now agree with each other. vec2text: 92% exact recovery of 32-token inputs and the need to re-embed guesses match Morris et al. (EMNLP 2023). An orthogonal rotation preserves dot products and can be solved from known pairs, so the new mitigation sentence is correct. `SecureRetriever` still parses with the inserted comment line. A mock run routes to the principal's tenant, and a caller's `acl`/`status` override is replaced by the mandatory filter while `lang` is kept. The isolation table has 3 columns in every row.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Deletion Step 2 purges the wrong cache — fixed ("Purge every cache that can hold the document's text or ids, such as the result cache and the rerank-score cache (Chapter 57), so no cached answer quotes it."). Used "text or ids" instead of "hold the document" to say why those caches matter. Query-embedding cache dropped (holds only question vectors). Consequence bullet (line 129 "every index, delta, replica, cache, backup") and benefit 2 ("Caches and backups still follow the deletion path above") still agree. No other "query-embedding cache" mention in the book.
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 63  (verifier: chapters 59–66 + README/OUTLINE)

No new issues.

- Verified: the "unrelated" wording is fully reconciled. The opening, the analogy's last line and the dot-product sentence all say vectors from different models cannot be compared directly, and that only a learned mapping approximates the translation. That now agrees with the adapter Ninja note and table row, and with Ch 2 line 327 and Ch 66 line 324 ("never comparable"). No "unrelated" is left in the chapter. The adapter note uses the right directions: a query-side map from new model into old space, or a forward map of old document vectors into a temporary new index. "Four terms first" now lists four bold definitions. Rank correlation is defined before Phase 3 Step 2 uses it, the later duplicate is gone, and "1 … near 0" is correct. The code comment and the closing prose say rank correlation and errors are omitted. I re-ran the fixer's mock harness on the current block: the canary passes on an identical path and raises "max abs diff 2.70e-01" when unnormalized, overlap is 0.7, and the empty list gives 0.0 / None. Fences balance.

### Post-fix check: Chapter 64  (verifier: chapters 59–66 + README/OUTLINE)

No new issues.

- Verified: I recomputed every table row with weights 0.6 / 0.25 / 0.15 and a 30-day half-life. Pro: 0.372 + 0.25 × 0.125 + 0.135 = 0.5383 → 0.538. August: 0.5^(45/30) = 0.3536 → 0.354, and 0.42 + 0.0884 + 0.015 = 0.5234 → 0.523. Basic: 0.5^(220/30) = 0.0062 → 0.006, and 0.372 + 0.0016 + 0.135 = 0.5086 → 0.509. The gap is 0.5383 − 0.5086 = 0.0297 ≈ 0.03. The prose claims hold. August is the most similar (0.70). Importance (0.15 × 0.8 = 0.12) outweighs the similarity gap (0.048) plus the recency gap (0.057), so Pro wins by 0.015. Without importance, August would win, which matches "its high importance lifts it to the top" (line 211). 45 days before a mid-September question falls on about 1 August. I extracted the code and ran it with stub index and store: `recall` returns [pro, aug], filters out the superseded memory, and its `score` reproduces 0.538 / 0.523 / 0.509. `remember` dispatches ADD / UPDATE / MERGE / IGNORE, and `supersede` and `merge` now receive `importance`. `CONSOLIDATE.format` renders. "last used" is in Step 4 and in the diagram. "includes SSO at 50 seats or more" matches pages 212 / 1,140. No old values (0.598, 0.542, 0.617) remain anywhere in the book. The memory-type table has 4 columns in every row.

### Post-fix check: Chapter 65  (verifier: chapters 59–66 + README/OUTLINE)

- **[NEW-NIT]** The new claim that LSH was "the first" randomised, training-free construction with guarantees is doubtful, and the book itself points to an older one. Quote: "LSH was the first of this kind, and it lives on inside MUVERA". Random projection under the Johnson–Lindenstrauss lemma (1984) predates LSH (Indyk and Motwani, 1998). Ch 6 teaches it, and Ch 21 line 365 calls JL "the licence for every" such trick. Fix: "LSH is the classic example of this kind, and it lives on inside MUVERA".
- **[NEW-NIT]** The rewritten quantization bullet cites Chapter 37 twice in one sentence. Quote: "through residual compression (Chapter 37) or TurboQuant (Chapter 27),\n  brings each 128-d token vector from 512 bytes down to about 20–36 bytes (Chapter 37)." Fix: drop the trailing "(Chapter 37)".
- Verified: 20–36 bytes per token matches Ch 37 (lines 17, 146, 150, 471), and TurboQuant 2-bit at 128-d is 32 + 4 = 36 B (Ch 61 table). 128 × 4 = 512 B. The 8-byte and 1,600-byte arithmetic is unchanged and correct. MUVERA's SimHash regions match Ch 38 line 168 ("exactly the LSH of Chapter 21"), and the takeaway matches the body. DiskANN "a few milliseconds" matches Ch 32's 2–10 ms, and object storage "tens to hundreds of milliseconds" matches Ch 32 line 376. The D5 ladder sentence matches Ch 60. The learned-routing paragraph follows D2 and agrees with Ch 20's Ninja note ("a partition index whose signposts were trained rather than computed") and OUTLINE's "There is no fifth idea". Principle #8 is now in Key takeaways. "What's next" uses the D12 wording, and no "compresses the whole book" or "ancestor" wording is left for Ch 65/66.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] "LSH was the first of this kind" contradicts older JL projections — fixed differently: "LSH was an early member of this kind" (random projection under JL, Ch 6, is older, so "early" not "classic example"). Paragraph rewrapped. Key takeaway ("LSH lives on inside MUVERA") makes no priority claim, so unchanged. No other "first of this kind" claim in the book (Ch 20/21 "first" refers to the order of the four families).
- [NEW-NIT] Quantization bullet cites Chapter 37 twice — fixed (dropped the trailing "(Chapter 37)").
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: Chapter 66  (verifier: chapters 59–66 + README/OUTLINE)

- **[NEW-NIT]** The rewritten frontmatter summary now has a plural subject but keeps the singular pronoun. Quote: "The book's key decision trees, formulas, defaults and checklists, in one place, with a pointer back to the chapter that explains it." Fix: "…in one place, each with a pointer back to the chapter that explains it."
- Verified by script against the CURRENT source chapters. These are identical line for line: the sharding strategy table and the query-pattern table (Ch 58), the staleness budget and maintenance calendar (Ch 59, including the new Continuous row), and the lever table (Ch 60, and also identical to D4). The LLM cost formula, the worked 13,000-token line, and the 7 / 19 / 140 GPU-hour lines appear verbatim in Ch 60. The D4 intro, the D5 ladder line and the D7 nprobe "8–64, start at 16 (sweep)" match the brief exactly. nprobe agrees with Ch 25 line 269 (8–64), its knee "Usually 16–32" (line 308) and Ch 23's "16–64". The failure tree now checks answerability first, as Ch 55 does (line 79 "We check for it before the split" and `diagnose()` testing `answerable is False` first). Failures 1–8 and 10–12 map correctly, and "relevance floor" is the name Ch 55 line 279 says Ch 66 uses. The "Which index?" tree now contains Ch 20's "> 1B → DiskANN, or sharded IVF-PQ (32, 58)" and "Batch/offline → Flat on GPU, batched (17)" rows. The over-fetch default matches Ch 58 line 350 (2–5× int8, 10–20× PQ, ~100× binary). ColPali 1,030 vectors (1,024 patches + prompt tokens) matches Ch 46 lines 70 and 88, and 1,030 × 128 × 4 = 527,360 B ≈ 527 KB. ~93 GB appears in both places. The merged relevance-floor checklist item reads correctly. All 9 tables have consistent column counts, and fences balance.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Summary plural subject with singular "it" — fixed ("…in one place, each with a pointer back to the chapter that explains it."). OUTLINE's Ch 66 one-liner has no pointer clause, so unchanged.
- [NEW-SHOULD] (table rendering block, Ch 66 half) BM25 cell `|D|` splits the row — fixed ($\lvert D\rvert$). Rescanned every table row in Ch 66: no other `|` inside math or inline code.
- [NEW-NIT] (Chapter 51 block) "Title/breadcrumb prepended" checklist item — fixed ("Title and section heading prepended before embedding"). Remaining "breadcrumb" uses are in Ch 51 metadata fields, not the prepend step.
- Ch 60 "~7×" base-model ratio: not copied in Ch 66 (it only lists ~7 / ~19 / ~140 GPU-hours, which are Ch 60's absolute figures), so no change.
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: README.md and OUTLINE.md  (verifier: chapters 59–66 + README/OUTLINE)

- **[NEW-NIT]** The new Globex sentence says Globex "returns in Parts VI–IX", but Globex first appears in Part IV, and "A second customer" reads as if Acme were the first customer. Quote: "A second customer, Globex, returns in Parts VI–IX." Ch 33 line 202 introduces it: "Globex, one of Acme's largest customers, has 2 million chunks". The running example is Acme's own knowledge base, and Globex is Acme's customer. Fix: "One of Acme's customers, Globex, appears from Chapter 33 on and carries many examples in Parts VI–IX."
- Verified: every README and OUTLINE change matches D12 and D3 word for word. Subtitle "a hundred million chunks". "Most are a 10–15 minute read" and "most chapters a 10–15 minute read": the stated reading times sum to 798 min ≈ 13.3 h, so "about 13 hours" still holds. The builder path now includes Chapters 40–41. The template Note is accurate: a script over all 66 chapters shows that only Ch 65 (no Under the hood / What people get wrong / Ninja notes) and Ch 66 (none of the standard sections) skip template parts. The Globex facts (Pro, 120 seats since June 2026, previously Basic with 30, pages 3,507–3,508) match Ch 64 lines 64 and 206, Ch 52 line 173 and Ch 61's revised Step 7. OUTLINE one-liners checked against the chapters. Ch 2 matches its one-paragraph version ("a sentence, a photo or a product"). Ch 41 "largest precision win once retrieval works" matches Ch 41 lines 19 and 389 and its summary. Ch 60 "a hundred million vectors" matches its summary. Ch 66 matches its new summary. The Ch 7 and Ch 35 semicolons are gone. Ch 20 keeps "There is no fifth idea" per D2, and that now agrees with Ch 20's Ninja note. OUTLINE still has 66 rows, all with 3 columns.

#### Resolution (applied 2026-09-16)

- [NEW-NIT] Globex "returns in Parts VI–IX" but first appears Ch 33 — fixed differently: "Globex, one of Acme's largest customers, first appears in Chapter 33 and returns in many chapters of Parts VI–IX." Grep shows Globex in Ch 33 (Part IV), none in Part V, then Ch 44–47, 51, 52, 55, 56, 61, 62, 64, 65, so "many chapters of" is more accurate than "from Chapter 33 on", and "one of Acme's largest customers" matches Ch 33 line 202 wording. OUTLINE.md has no Globex mention, so unchanged.
- Code: no code changed
- Cross-chapter follow-ups: none

### Post-fix check: table rendering, whole book  (verifier: lead)

Found while verifying Ch 35 and then scanned across all 66 chapters. Both problems predate the fixes.

- **[NEW-SHOULD]** A `|` inside math in a table cell splits the cell, so the row renders with too many columns. Quote
  (Ch 35, comparison table): "| Interaction | 1 scalar | $|q| \times |d|$ grid | full attention |". Fix:
  "$\lvert q\rvert \times \lvert d\rvert$ grid", or escape the bars as `\|`.
- **[NEW-SHOULD]** Same problem in Ch 66's formulas table. Quote: "| BM25 term | $\text{IDF}\cdot\frac{f(k_1+1)}{f + k_1(1-b+b\frac{|D|}{\text{avgdl}})}$ | 8 |".
  Fix: "\lvert D\rvert" in place of "|D|".
- Verified: scanned every table row in every chapter for `|` inside `$…$` or inline code. Only these two are real. The
  hits in Ch 60's cost table are dollar amounts in neighbouring cells, not math.

#### Resolution (applied 2026-09-16)

- [NEW-SHOULD] Ch 35 table cell `$|q| \times |d|$` — fixed: `$\lvert q\rvert \times \lvert d\rvert$`.
- [NEW-SHOULD] Ch 66 BM25 cell `|D|` — fixed: `\lvert D\rvert`.
- A column-count scan of every table row in the book now finds no mismatches.

