
# LMA Assignment 1: Baseline Retrieval vs. Cross-Encoder Re-Ranking
*Link to the repository: https://github.com/nidhi-00/LMA-Assignment-1.git*

## 1. Setup and architecture comparison

### Dataset and preprocessing

- Dataset: FIDE Laws of Chess (2023).
- 41-page PDF with formal rules, restrictions, and exceptions.
- Extracted text using PyMuPDF.
- Divided text into page-local chunks.
- Maximum chunk size: 170 words.
- Chunk overlap: 40 words.
- Total corpus: 91 chunks.
- Each chunk retained:
  - A stable identifier.
  - Its original PDF page number.
- This allowed retrieved evidence to be traced back to the document.

### Pipeline 1: Baseline (Bi-Encoder)

- Model: `sentence-transformers/all-MiniLM-L6-v2`.
- Encoded document chunks and queries independently.
- Embedding dimension: 384.
- Normalized the embedding vectors.
- Used FAISS `IndexFlatIP` for cosine-similarity retrieval.
- Retrieved the top 10 highest-scoring chunks per query.
- Selected the top 3 chunks directly for answer generation.

### Pipeline 2: Cross-Encoder Re-Ranking

- Used the exact same corpus and initial top 10 candidates.
- Model: `cross-encoder/ms-marco-MiniLM-L6-v2`.
- Scored each query–chunk pair jointly.
- Reordered candidates by Cross-Encoder relevance scores.
- Selected the new top 3 chunks for answer generation.

### Architectural difference

**Bi-Encoder:**
- Processes queries and passages separately.
- Allows document embeddings to be precomputed.
- Supports fast similarity-based retrieval.

**Cross-Encoder:**
- Processes queries and passages jointly.
- Uses token-level attention across both texts.
- Captures more detailed interactions between words.
- Requires fresh computation for every query–passage pair.

### LLM answer generation

- Model: Gemini 3.8 Flash.
- Temperature: 0.
- Identical prompt structure and output settings for both pipelines.
- Both prompts instructed the model to:
  - Use only the retrieved chunks.
  - Include important exceptions.
  - Cite chunk IDs and PDF page numbers.
- Only the selected context chunks differed between the two pipelines.

## 2. Qualitative results and rank-shift analysis

### Evaluation setup

- Tested 24 document-specific questions.
- Questions focused on:
  - Exceptions.
  - Prohibited actions.
  - Corner cases.
  - Potentially misleading wording.
- Cross-Encoder promoted 26 candidate chunks from initial ranks 4–10 into the reranked top 3.
- Rank promotion alone does not guarantee answer improvement.
- Therefore, I examined three representative questions.
- Compared both generated answers against the source rules.

| Question | Critical PDF chunk | Bi-Encoder rank | Cross-Encoder rank | Observed difference |
|---|---|---:|---:|---|
| Q14: Can a mobile phone be used in the playing venue? | `P20_C02` (p. 20) | 4 | 1 | Adds Article 12.8, applying to anybody and designated contiguous areas. |
| Q23: Can differently ordered moves lead to a repetition draw claim? | `P17_C01` (p. 17) | 5 | 1 | Adds Article 9.2.3, defining when positions count as identical. |
| Q02: Is the third occurrence of a position automatically a draw? | `P17_C03` (p. 17) | 6 | 1 | Adds the automatic fivefold repetition rule, Article 9.6.1. |

### Q14: Mobile-phone restriction (scope/negation)

**Baseline retrieval and Answer 1**

- Retrieved passages from Article 11.3.2.
- Correctly explained:
  - Restrictions on players possessing electronic devices.
  - The exception for switched-off devices stored in a bag.
  - The possible loss-of-game penalty.
- However, Article 12.8 was initially ranked 4th.
- It was therefore excluded from the baseline's top 3 context chunks.
- Answer 1 consequently omitted a broader restriction:
  - Mobile-phone and communication-device use is restricted for anybody in the playing venue.
  - The restriction also covers designated contiguous areas.
  - Use requires authorisation from the arbiter.

**Cross-Encoder and Answer 2**

- Promoted chunk `P20_C02` from rank 4 to rank 1.
- Recovered the relevant Article 12.8 passage.
- Answer 2 included the wider restriction.
- It clarified that the rule applies to everybody, not only players.

**Observation**

- Re-ranking recovered a missing scope limitation.
- It improved answer completeness.
- The baseline answer was not completely wrong.

### Q23: Repeated positions reached by different moves (corner case)

**Baseline retrieval and Answer 1**

- Correctly stated that repeated positions do not require the same move sequence.
- Also explained:
  - When a player must make a draw claim.
  - What happens after an incorrect claim.
- However, it omitted the main conditions for determining whether two positions are identical.

**Cross-Encoder and Answer 2**

- Promoted chunk `P17_C01` from rank 5 to rank 1.
- Retrieved the relevant Article 9.2.3 passage.
- Answer 2 included the requirements for identical positions:
  - The same player must have the move.
  - The same types and colours of pieces must occupy the same squares.
  - The possible moves must be the same.
- It also mentioned important exceptions:
  - En passant capture availability.
  - Changes in castling rights.
- These conditions can make apparently similar positions different.

**Observation**

- Re-ranking recovered important conditions for a valid repetition claim.
- Answer 2 provided a more complete explanation.
- The additional details materially qualify when a repetition claim is valid.

### Q02: Threefold versus fivefold repetition (supplementary case)

**Baseline retrieval and Answer 1**

- Question: Does the third occurrence of a position automatically end the game?
- Answer 1 correctly stated that three occurrences do not automatically produce a draw.
- Under Article 9.2, a correct claim is required.
- The claim must be made by the player having the move.
- The baseline answer was already sufficient for the question.

**Cross-Encoder and Answer 2**

- Promoted chunk `P17_C03` from rank 6 to rank 1.
- Retrieved the relevant Article 9.6.1 passage.
- Answer 2 additionally distinguished:
  - Threefold repetition: draw requires a valid claim.
  - Fivefold repetition: draw occurs automatically.

**Observation**

- Re-ranking added a relevant exception or related rule.
- It improved answer completeness.
- It did not correct a factual error in the baseline answer.

### Overall observations

- Re-ranking recovered important evidence initially outside the baseline's top 3.
- Semantic similarity alone sometimes ranked essential qualifications too low.
- Q14 demonstrated recovery of a wider rule restriction.
- Q23 demonstrated recovery of precise conditions and exceptions.
- Q02 demonstrated improved completeness through an additional rule.
- Neither pipeline produced an obvious factual hallucination in these comparisons.
- The clearest improvement was the inclusion of previously omitted qualifications.

## 3. Latency profiling

### Experimental setup

- Environment: Windows 11.
- Hardware: CPU inference.
- PyTorch CPU threads: 4.
- Models were warmed up before measurement.
- Queries tested: 6.
- Repetitions per query: 5.
- Total measured runs: 30.
- Timing function: `time.perf_counter()`.

### Operations included

- Bi-Encoder query embedding.
- FAISS similarity search.
- Cross-Encoder scoring of the initial top 10 candidates.

### Operations excluded

- Model downloads.
- Corpus indexing.
- Gemini API answer generation.

### Results

| Stage | Median latency |
|---|---:|
| Baseline: query encoding and FAISS retrieval | 11.52 ms |
| Additional Cross-Encoder scoring | 477.59 ms |
| Advanced pipeline total | 488.65 ms |

### Analysis

- Advanced retrieval was approximately **42.4 times slower** than baseline retrieval.
- The comparison uses the median total times of both pipelines.
- Most additional latency came from the Cross-Encoder.
- It evaluated 10 query–passage pairs for each query.
- This demonstrates a speed–relevance trade-off:
  - Baseline retrieval required substantially less CPU time.
  - Advanced retrieval provided more relevant context in selected failure cases.
- Median times of individual stages need not add up exactly to the median of their per-run totals.

## 4. Scaling Question

### Cross-Encoder limitations

- Cross-Encoders usually provide more detailed relevance estimates.
- They cannot precompute reusable document representations independently of queries.
- Every new query requires joint evaluation with every candidate passage.
- Scoring an entire collection would therefore be computationally expensive.
- Latency increases approximately with the number of passages evaluated.
- This becomes impractical for large document collections.

### Bi-Encoder advantages

- Passage embeddings can be precomputed once.
- Embeddings can be stored in a vector search index.
- New query embeddings can be compared efficiently against stored vectors.
- This avoids repeated joint inference over the entire collection.

### Benefits of two-stage retrieval

- Stage 1: Bi-Encoder efficiently narrows the collection to a small candidate set.
- Stage 2: Cross-Encoder performs detailed relevance scoring on those candidates.
- Combines fast retrieval with more precise ranking.
- Avoids running the Cross-Encoder across the complete corpus.
- In this experiment:
  - Total document chunks: 91.
  - Cross-Encoder candidates per query: 10.
- This preserves the strengths of both approaches while reducing computational cost.

## 5. Conclusion and limitations

### Key findings

- Cross-Encoder re-ranking improved the relevance of the final three retrieved chunks for several carefully chosen questions.
- The strongest examples were:
  - Q14: Mobile-phone restrictions.
  - Q23: Repeated-position definitions.
- In both cases, baseline retrieval omitted important qualifications.
- Re-ranking promoted the missing passages into the LLM context.
- This led to more complete answers.

### Interpretation

- Baseline answers were mostly factually correct.
- The main improvement was answer completeness.
- Results do not establish that re-ranking eliminates hallucinations.
- The experiment demonstrates improved retrieval of important restrictions and exceptions.

### Limitations

- Used only one PDF document.
- Questions were manually designed.
- Only three questions received paired LLM answer analysis.
- Latency measurements came from one CPU system.
- Results are illustrative, not a general accuracy benchmark.
- Cross-Encoder re-ranking has an important dependency:
  - The initial Bi-Encoder must retrieve the relevant passage within its top 10.
  - If the relevant passage is absent from those candidates, re-ranking cannot recover it.

### Source document

FIDE, *Laws of Chess taking effect from 1 January 2023*.

https://handbook.fide.com/chapter/E012023

Key rules checked: Articles 9.2, 9.2.3, 9.6.1, 11.3.2, and 12.8.
