# Run Reference

## Prerequisites

Clone the active codebase:

```bash
git clone https://github.com/ronit22203/clinical-trials-matching-platform
cd clinical-trials-matching-platform
```

Required environment variables (set in shell or `.env`):

```
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=<password>
QDRANT_URL=http://localhost:6333
LM_STUDIO_BASE_URL=http://localhost:1234/v1
```

---

## Quickstart

Reproduce the verified deterministic run `det_20260430_011150_451864b`:

```bash
git checkout 451864b6
make bootstrap
make up
make benchmark-all
```

`make bootstrap` installs Python dependencies and pulls Ollama models.  
`make up` starts Qdrant, Neo4j, Temporal, and the Temporal worker.  
`make benchmark-all` runs the full deterministic suite: ingest, graph construction, retrieval benchmark, inference benchmark, extraction evaluation, and report generation.

---

## Full Command Surface

### Infrastructure

| Command | What it does |
|---|---|
| `make up` | Start all Docker services (Qdrant, Neo4j, Temporal, Postgres) |
| `make down` | Stop all services |
| `make status` | Check service health |
| `make validate` | Validate config and env vars |

### Bootstrap

| Command | What it does |
|---|---|
| `make bootstrap` | Install all Python deps, pull Ollama models |

### Acquisition

| Command | What it does |
|---|---|
| `make acquisition-install` | Install acquisition module deps |
| `make fetch SOURCE=medrxiv MAX_PDFS=10` | Fetch PDFs from a source |
| `make acquisition-test` | Run acquisition tests |

Supported SOURCE values: `medrxiv`, `biorxiv`, `pubmed`, `clinicaltrials`

### Ingestion

| Command | What it does |
|---|---|
| `make ingestion-install` | Install ingestion module deps |
| `make ingestion-run` | Run full 7-stage ingestion pipeline |
| `make ingestion-run SKIP=ocr` | Run pipeline skipping OCR stage |
| `make ingestion-test` | Run ingestion tests |
| `make ingestion-neo4j-build` | Rebuild Neo4j graph from existing chunks |

### Reasoning

| Command | What it does |
|---|---|
| `make reasoning-install` | Install reasoning module deps |
| `make reasoning-run` | Start interactive CLI agent |
| `make reasoning-run-query QUERY="..."` | Single-shot query |
| `make reasoning-serve-api` | Start FastAPI server on port 8000 |
| `make reasoning-test` | Run reasoning tests |

### UI

| Command | What it does |
|---|---|
| `make ui-install` | Install UI deps |
| `make ui-dev` | Start UI dev server |
| `make ui-build` | Build UI for production |

### Benchmarking

| Command | What it does |
|---|---|
| `make benchmark-all` | Full deterministic suite (all phases) |
| `make deterministic-run` | 9-phase run: validate, reset, ingest, graph, provenance, retrieval, extraction, inference, report |
| `make benchmark-retrieval COLLECTION=medical_papers BOOTSTRAP=1000 RERANKER=BAAI/bge-reranker-base` | Retrieval benchmark with cross-encoder reranker |

---

## Temporal Worker (Durable Workflows)

Run the background worker for auditable, fault-tolerant workflow execution:

```bash
python -m src.temporal.worker
```

The worker must be running for `runtime: temporal` agent queries to execute. If the worker crashes mid-run, the workflow resumes from the last completed activity checkpoint on restart.

---

## Scale Decision Framework

| Workload | Hardware | Rationale |
|---|---|---|
| Development iteration, single-doc validation, benchmarking | M4 local | Zero cost, zero data egress, deterministic |
| Bulk ingestion (>10 PDFs) | AWS g4dn.xlarge T4 ($0.526/hr) or spot | GPU-accelerated OCR at p50 133.7s/doc. 2-3 parallel instances complete 1,036 docs in under 20 hours. |
| Production inference, real-time queries | RTX 5080 spot (VAST.ai, $0.017/hr) | 62.6 tok/s, 81ms TTFT cold, 37-45ms TTFT with RadixAttention cache. Break-even vs GPT-4o mini in under 15 minutes of usage. |

---

## Verifying a Run

Every deterministic run produces a fingerprint file with:
- Run ID (timestamp + git commit short hash)
- PDF SHA-256
- Config SHA-256
- Model identifiers
- Qdrant point count

To confirm a run matches a known baseline, compare the fingerprint against entries in `eval.md`.

Execution logs are written to:
- `agentic-reasoning/log/{execution_id}.json` (per query)
- `agentic-reasoning/log/summary.jsonl` (all queries, append-only)
