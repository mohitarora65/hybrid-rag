# Hybrid Search RAG

Production-shaped retrieval-augmented generation with dense retrieval, BM25, reciprocal-rank fusion,
reranking, grounded generation, and claim-level citation checks.

On the included 11-question seed set, hybrid retrieval reaches **100% Recall@3 and 100% verified-citation
accuracy**; BM25 alone reaches 90.9% Recall@3. The corpus is deliberately small, so these numbers are a
reproducible smoke benchmark, not a claim about an external production dataset.

## Result first

| retrieval mode | Recall@3 | MRR | nDCG | verified citations |
|---|---:|---:|---:|---:|
| dense + reranker | 100.0% | 1.000 | 1.000 | 100.0% |
| BM25 + reranker | 90.9% | 0.909 | 0.909 | 90.9% |
| **hybrid RRF + reranker** | **100.0%** | **1.000** | **1.000** | **100.0%** |

Regenerate the checked-in report with `make eval`. A real portfolio deployment should replace the seed
documents with a domain corpus and grow the human-reviewed golden set before quoting results.

## Architecture

```text
Markdown/text documents
        |
        v
fixed / recursive / semantic chunking
        |
        +--------------------+
        |                    |
        v                    v
hashing/real embeddings     BM25 index
        |                    |
        +---------+----------+
                  v
       reciprocal-rank fusion
                  |
                  v
       relevance reranker (top 5)
                  |
                  v
      grounded answer + citations
                  |
                  v
      claim/source verification
```

The default hashing embedder and extractive generator make tests deterministic and free. Set
`LLM_API_KEY`, `LLM_BASE_URL`, and `LLM_MODEL` to use any OpenAI-compatible model for answer generation.
The interfaces are intentionally narrow so a learned embedding model or external vector database can be
added without changing the API or evaluation harness.

## Run it

```bash
make venv
make test
make eval
make run
```

The service seeds the sample operations handbook on first startup. OpenAPI documentation is at
`http://localhost:8000/docs`.

Ask a question:

```bash
curl -s http://localhost:8000/v1/ask \
  -H 'Content-Type: application/json' \
  -d '{"question":"Which thresholds control a production canary?","mode":"hybrid","top_k":5}'
```

Ingest a document:

```bash
curl -s http://localhost:8000/v1/ingest \
  -H 'Content-Type: application/json' \
  -d '{"title":"Support policy","source":"support.md","strategy":"recursive","text":"Priority support responds within two hours."}'
```

The service also exposes `POST /v1/chat/completions`, so the semantic-cache project can use it as a normal
OpenAI-compatible upstream.

## What is implemented

- Idempotent multi-document ingestion with stable chunk IDs and source metadata.
- Fixed-overlap, structure-aware recursive, and adjacency-based semantic chunking.
- Dense retrieval and a from-scratch BM25 implementation over the same chunks.
- Weighted reciprocal-rank fusion followed by a transparent local reranker seam.
- Grounded generation that refuses unrelated questions instead of inventing an answer.
- Citation parsing, claim-to-source support checks, and a composite confidence score.
- Dense, sparse, and hybrid evaluation with Recall@k, MRR, nDCG, category slices, and case-level output.
- FastAPI ingestion/query/document APIs, OpenAI-compatible chat, health checks, and Prometheus metrics.
- SQLite persistence, Docker packaging, sample documents, and offline tests.

## Design decisions and tradeoffs

**Measure retrieval separately from generation.** The golden set names expected sources. That makes it
possible to identify whether a poor answer began with missed evidence or failed synthesis.

**Use RRF instead of comparing incompatible scores.** Cosine similarity and BM25 have different scales.
Rank fusion combines them without pretending their raw values are calibrated.

**Keep an offline path.** CI should not become flaky or expensive because an API is unavailable. The local
embedder and generator exercise the complete control flow, while production adapters can improve quality.

**Reject weak evidence.** A top result always exists in nearest-neighbor search, even for an unrelated
question. The absolute reranker threshold is therefore a required abstention control.

## What did not work / limitations

- Pure dense and hybrid retrieval tie on this tiny seed corpus. The honest conclusion is that the corpus is
  too small to justify added hybrid complexity from these results alone; the BM25 failure cases still prove
  why the second retrieval path needs evaluation.
- SQLite scans embeddings in-process. It is convenient for a reviewer and correct for a small corpus, but a
  large deployment should use Qdrant, pgvector, or another ANN index.
- The local reranker is lexical and transparent, not a cross-encoder. The class boundary is where a learned
  reranker belongs.
- Citation verification checks textual support. High-risk use cases should calibrate a separate judge against
  human labels rather than treating lexical agreement as factual proof.

## Repository map

```text
app/                 ingestion, retrieval, generation, API, and metrics
data/sample_docs/    reproducible example corpus
data/golden.json     versioned human-written retrieval cases
eval/run_eval.py     dense vs sparse vs hybrid comparison
scripts/demo.py      API-free terminal walkthrough
tests/               unit and end-to-end HTTP tests
results/             generated evaluation report
```

## Interview discussion

Be ready to explain why exact identifiers favor BM25, why RRF is safer than adding raw scores, how an
abstention threshold is selected, and why citation correctness is not the same as answer correctness. The
most useful next experiment is expanding the golden set to at least 50 questions, then comparing chunking
strategies and reranker choices on the same cases.
