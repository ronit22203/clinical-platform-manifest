# System Architecture

## Platform Overview

The platform is organized into four vertically integrated layers. Each layer is independently deployable, observable, and replaceable. No layer has hard dependencies on specific cloud providers or API vendors.

```
+------------------------------------------------------------------+
|  ACQUISITION LAYER                                               |
|  bioRxiv  medRxiv  ClinicalTrials.gov  PubMed                    |
|  Multi-cloud storage: S3 (primary) -> Azure Blob -> Local        |
+------------------------------------------------------------------+
|  INGESTION LAYER                                                 |
|  SHA256 Dedup -> Surya OCR -> PII Redaction -> Chunking          |
|  -> Embeddings (BGE-small) -> Qdrant + Neo4j                     |
|  -> Byte range tracking -> Tiered entity extraction              |
+------------------------------------------------------------------+
|  REASONING LAYER                                                 |
|  LangGraph ReAct Agent  /  Temporal Workflow Engine              |
|  Tools: PubMed  FDA  ClinicalTrials  GraphRAG  MCP FS            |
|  -> Reranker (BGE base) -> Hybrid retrieval with graph context   |
+------------------------------------------------------------------+
|  INTERFACE LAYER                                                 |
|  CLI (Click, Rich)  Structured JSON logs  Web UI (Alpine.js)     |
|  AI synthesis panel  Evidence cards  Debug visualizations        |
+------------------------------------------------------------------+
```

---

## Design Principles

**Build for failure.** The acquisition layer runs a primary-through-fallback chain: S3 fails over to Azure Blob, which fails over to local disk. In 100+ production runs, the fallback chain activated twice, both handled without operator intervention or data loss.

**Design for observability.** Every agent execution produces a structured JSON log anchored to a UUID, a git commit hash, and a timestamp. Token counts, tool call sequences, and tool responses are all captured. No query is unaccountable.

**Configure with YAML, not code changes.** Agent behavior including models, prompts, tool selection, inference parameters, retry policy, and HITL gate is declared in YAML and validated by Pydantic at startup. A clinician can change the system prompt without writing Python.

**Local-first, privacy by default.** The default inference path uses locally running open-weight models. No patient data, document content, or query text leaves the machine unless explicitly configured. API keys are optional acceleration, not requirements.

---

## Dual Runtime Agent Model

The platform exposes two execution paths over the same tool registry and YAML configuration.

**LangGraph ReAct** is an autonomous tool selection loop where the LLM decides which tools to call based on query content. Designed for interactive, streaming, low-latency responses. The model receives tool results incrementally and can chain multiple retrieval steps before synthesizing an answer.

**Temporal Workflow** is a deterministic activity orchestration engine where all five tools fan out in parallel, results are persisted as workflow history, and an optional Human-in-the-Loop signal gates synthesis before any response is returned. A worker crash mid-execution resumes from the last completed activity checkpoint, not from the beginning. This path is required for regulated research workflows.

Both paths share identical tool implementations, system prompts, and execution logging. The choice of runtime is a YAML flag (`runtime: langgraph` or `runtime: temporal`).

---

## Stack Decisions

| Decision | Choice | Rationale | Rejected alternative |
|---|---|---|---|
| Vector + graph storage | Qdrant + Neo4j | Pure vector search cannot resolve multi-hop clinical queries (Drug TREATS Condition CONTRAINDICATED_WITH Drug). Graph traversal closes this gap. | Qdrant alone; Pinecone |
| Agent orchestration | LangGraph + Temporal | LangGraph for low-latency streaming; Temporal for durable, replayable, auditable workflows where a worker crash must not lose execution state. | Celery; bare asyncio tasks |
| OCR engine | Surya (local) | AWS Textract and Google Document AI send documents to remote services. Surya runs on-device, preserves multi-column medical layouts, and costs zero per page. | AWS Textract; pdfminer (scrambles column order) |
| LLM inference | Ollama/LiteLLM local, then SGLang on RTX spot | Zero data leakage, zero API cost for local dev. SGLang with RadixAttention for production-grade throughput. Model name is a single YAML value. | OpenAI API; Anthropic API |
| Reranker | BAAI/bge-reranker-base (cross-encoder) | Improves Precision@5 by 6 points and HitRate@5 by 5 points over vector-only retrieval. | No reranker; Cohere Rerank (sends data to API) |
| PII handling | Microsoft Presidio + custom recognizers as a structural firewall | PII redaction is applied before any text reaches the embedding model or knowledge graph. It is a pipeline boundary, not a post-processing step. | Regex only; post-storage redaction |

---

## End-to-End Data Flow

```
[Source APIs] -> [SHA-256 hash + metadata] -> [Storage chain: S3 / Azure / local]
                                                            |
                                          [Surya OCR -> Markdown -> TextCleaner]
                                                            |
                                     [500-token chunks with byte-range offsets]
                                                     /            \
                              [BGE-small embeddings]            [Qwen3-8b entity extraction]
                                        |                                |
                                   [Qdrant]                         [Neo4j]
                                        \                               /
                                   [Hybrid retrieval + BGE reranker]
                                                    |
                               [LangGraph ReAct / Temporal Workflow]
                                                    |
                            [FastAPI /api/query -> UI or CLI output]
```

---

## Hardware Spectrum

| Metric | MacBook Air M4 (16GB) | AWS g4dn.xlarge (T4) | RTX 5080 (VAST.ai spot) |
|---|---|---|---|
| GPU | Apple M4 (unified) | NVIDIA T4 (16GB GDDR6) | NVIDIA RTX 5080 (16GB GDDR7) |
| Cost | Free (local dev) | $0.526 per hour | $0.017 per hour |
| Inference throughput | 13.25 tok/s | not measured | 62.62 tok/s |
| TTFT (mean) | 307.6 ms | not measured | 81.4 ms |
| Memory bandwidth utilization | ~15% | not measured | 91.35% |
| Batch concurrency | 1 sequence | not measured | 16 sequences |
| Ingestion p50 per PDF | ~22 minutes | 133.7 seconds | not applicable |
| Failure modes | silent timeout, hallucination | OOM on very large PDFs | spot preemption (cold restart) |

**Decision framework:**

- M4 local: development iteration, single-document validation, deterministic benchmarking. Not suitable for bulk ingestion or production inference.
- g4dn.xlarge T4: batch ingestion where GPU-accelerated OCR is the bottleneck. 2-3 parallel spot instances would complete the 1,036-document corpus in under 20 hours at p50 133.7s/doc.
- RTX 5080 spot: production inference and real-time agent queries requiring sub-100ms TTFT or concurrent batching.

---

## Critical Environment Variables

```
NEO4J_URI
NEO4J_USER
NEO4J_PASSWORD
QDRANT_URL
LM_STUDIO_BASE_URL
NEXT_PUBLIC_API_URL   (UI layer only)
```

All YAML config references use `${ENV_VAR}` substitution. Secrets are never committed to YAML.
