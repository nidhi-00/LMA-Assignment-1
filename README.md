# LMA Assignment 1: Baseline Retrieval vs. Cross-Encoder Re-Ranking

A retrieval-augmented generation (RAG) experiment comparing dense retrieval with Cross-Encoder re-ranking on the **FIDE Laws of Chess (2023)** PDF.

## Approach

- **Document:** 41-page PDF, split into **91** overlapping, page-tracked chunks.
- **Baseline:** `sentence-transformers/all-MiniLM-L6-v2` embeddings + FAISS cosine-similarity search. Retrieve the top 10 chunks and send the top 3 to Gemini.
- **Re-ranking:** `cross-encoder/ms-marco-MiniLM-L6-v2` rescoring the **same top 10** candidates. Send the new top 3 to Gemini using identical generation settings.
- **Evaluation:** 24 document-specific questions; 3 manually reviewed answer pairs (Q02, Q14, Q23). Re-ranking recovered additional rules or qualifications, without evidence that the baseline hallucinated in these cases.

## Results

| Retrieval stage | Median latency (CPU) |
|---|---:|
| Baseline retrieval | 14.23 ms |
| Cross-Encoder scoring overhead | 593.17 ms |
| Baseline + re-ranking | 607.13 ms |

Measurements exclude document indexing, model loading, and Gemini API latency. Detailed rankings, answers, manual evaluations, and timing records are in [`results/`](results/). See [`report.pdf`](report.pdf) for the full analysis.

## Repository contents

- `rag_assignment.ipynb`: complete executable notebook and saved experiment outputs.
- `data/`: source PDF.
- `results/`: retrieval rankings, paired answers, evaluations, and latency profiling.
- `report.pdf` / `report.md`: assignment report.
- `requirements.txt`: Python dependencies.

**Document reference:** [FIDE Laws of Chess (effective 1 January 2023)](https://handbook.fide.com/chapter/E012023).
