# Decisions

This is a log of the real choices made while building this platform solo over four months on an M4 MacBook Air. Each entry explains what I picked, what I seriously considered dropping it for, and why I did not. Some of these worked well. Some were mistakes I had to reverse. Both kinds are documented.

---

## ADR-001: Qdrant + Neo4j for storage

**Status:** Accepted

Vector search alone does not cut it for clinical queries. If someone asks whether Drug A is safe with Condition B given a patient's current medications, that is a chain of typed relationships: Drug TREATS Condition, Condition CONTRAINDICATED_WITH Drug. A dot-product score against a chunk embedding will not traverse that chain. So the storage layer is two things: Qdrant for semantic retrieval, Neo4j for structured relationship traversal. They merge results at query time through HybridRetriever.

Pinecone was out immediately because it sends data to an external API and that is a non-starter for documents that may contain patient information. Qdrant alone was the tempting simple path but fails on multi-hop queries. Weaviate has a graph module but the operational surface is larger for no real gain here.

The cost is running two stateful services. On anything smaller than a t4g.medium, memory becomes a real issue (F-010, F-011).

---

## ADR-002: LangGraph ReAct + Temporal dual runtime

**Status:** SUPERSEDED by ADR-008 (2026-05-10, commit `4c1bb17e`)

The original idea was two runtimes sharing a tool registry, switchable by a single YAML flag. LangGraph for interactive low-latency sessions, Temporal for durable auditable workflows with a human-in-the-loop gate.

It did not survive contact with the actual model. See ADR-008.

---

## ADR-003: Surya for OCR, everything local

**Status:** Accepted

The alternative was paying AWS Textract or Google Document AI to process documents that may contain protected health information. That is a non-starter. Surya runs entirely on device, handles multi-column medical layouts and tables, and costs nothing per page.

The price is speed. Surya is responsible for 94-97% of ingestion runtime (F-051). That is the single biggest unsolved bottleneck in this build. GPU acceleration for its table detection model is the next thing to fix.

pypdf and pdfminer were tested early and discarded. Both scramble multi-column reading order and drop table structure entirely.

---

## ADR-004: Local inference first, VAST.ai spot for production

**Status:** Accepted

Sending clinical query content plus retrieved document chunks to OpenAI or Anthropic has two problems: the data leaves the machine, and at scale GPT-4 costs $30 per million tokens. An RTX 5080 spot on VAST.ai costs $0.017 per hour and runs at 62.6 tokens per second. The break-even against GPT-4o mini is under 15 minutes of actual usage.

Development happens on Ollama or LM Studio on the M4. The model name is a single YAML value so switching requires no code changes.

M4 local inference has two known failure modes: silent timeout with empty output (F-047) and factual hallucination when the context window is insufficient (F-048). Anything production-grade runs the SGLang path on the GPU.

---

## ADR-005: Presidio as a mandatory PII firewall, not optional

**Status:** Accepted

If PII redaction is a post-processing step, patient data can enter the vector index or knowledge graph before it is cleaned. The only safe design is to run redaction before any text touches the embedding model or Neo4j. TextCleaner with Microsoft Presidio is a mandatory pipeline stage, not a toggle.

Standard Presidio does not cover Singapore NRIC/FIN or Medical Registration Numbers so custom recognizers were added for both. Regex-only approaches were tested and failed on edge cases. Making redaction optional was never a real option: a compliance guarantee that can be turned off is not a guarantee.

Overhead is about 15% per document.

---

## ADR-006: YAML configuration for all agent behavior

**Status:** Accepted (partially superseded in practice by ADR-008)

The goal was that a clinician or ops person could change agent behavior, swap models, or toggle tool selection without touching Python. All agent config was declared in YAML and validated by Pydantic at startup.

This worked well in the ingestion pipeline and still applies there. In the reasoning layer it became irrelevant when the entire LangGraph + YAML-driven agent architecture was replaced with `agent.py` (ADR-008). The flat Pydantic config in `config.py` is now the source of truth for reasoning configuration.

The failure mode for YAML-driven config is always startup: Pydantic catches bad schemas before any queries run.

---

## ADR-007: BGE cross-encoder reranker after vector retrieval

**Status:** Accepted

Dense vector retrieval ranks by dot-product which is fast but imprecise. Adding `BAAI/bge-reranker-base` as a cross-encoder after Qdrant retrieval and before graph enrichment gave concrete measured improvements:

- Recall@5 up 11.6 points
- Precision@5 up 6.0 points
- HitRate@5 up 5.0 points

Benchmarked against deterministic run IDs `det_20260425_113342_e14da50` (with reranker) vs `det_20260424_144844_fcb7c39` (baseline). Full numbers in eval.md.

The open issue is that comparative query types regressed 16.7 points on Recall@5 with the reranker applied. Root cause is not resolved yet.

Cohere Rerank API was the alternative. It is good but sends document content externally. Not acceptable here.

---

## Roadmap (as of April 2026)

| What | Why | Priority |
|---|---|---|
| GPU-accelerated OCR table detection | Surya is 94-97% of ingestion time | High |
| Empty output detection + query timeout | M4 silent failure (F-047) | High |
| RAG grounding for clinical trials queries | M4 hallucination failure (F-048) | High |
| Fine-tuned clinical LLM | BioMistral or OpenBioLLM for better extraction F1 | High |
| HIPAA compliance audit prep | Required before any commercial use | High |
| Lightweight local dev profile | Docker on M4 is heavy | Medium |
| REST API / FastAPI gateway | EHR integration path | Medium |
| RAGAS evaluation framework | Continuous synthesis quality measurement | Medium |
| Additional data sources | PubMed Central, Europe PMC | Medium |
| Streaming HITL via WebSocket | Replace CLI approval loop with browser | Medium |

---

## ADR-008: Nuke LangGraph, ship a deterministic two-phase pipeline

**Status:** Accepted
**Commit:** `4c1bb17e` (2026-05-10), branch `rebuild/agent-tool-routing`
**Supersedes:** ADR-002

This was the biggest architectural reversal of the project.

The problem was not the model and not the prompts. The problem was that LangGraph ReAct gives the model the option to call tools. On clinical queries where Qwen3-8B already had a confident parametric answer, it answered directly and skipped retrieval entirely. For a RAG system that is not a latency trade-off, it is a correctness failure. The whole point is that answers come from the documents, not from model weights.

Tried stricter prompting. Tried structured output constraints. Neither worked reliably. Tool execution in LangGraph is always advisory.

The fix was to stop giving the model a choice. The new pipeline in `agent.py` is two phases with no branching:

- Phase 1: retrieval tools run unconditionally. The model does not touch this step. Every query goes through vector search and graph enrichment.
- Phase 2: synthesis runs over whatever phase 1 returned. If phase 1 came back empty, a hardcoded no-evidence response goes out. No hallucinated answer.

While pulling out LangGraph I also removed Temporal. It was overkill for a single-user CLI tool and the operational overhead was not worth it (F-061, F-062). `app.yaml` went from 280 lines to 35 lines. A lot of infrastructure that felt important turned out to be scaffolding for complexity I did not need yet.

What is left is `agent.py`, `config.py`, `cli.py`, `tools/graphrag.py`, and 25 tests. The entry point is `make serve` for the interactive CLI or `make reasoning-run-query QUERY="..."` for a single shot.

The trade-off is that the pipeline is no longer autonomous. Multi-step tool chaining based on intermediate results requires explicit code changes now. That is fine. Predictable grounded answers matter more than agentic flexibility when the domain is clinical.

---

## ADR-009: Palantir Blueprint UI + Solarized Light 4-Pane HUD

**Status:** Accepted

The CLI works for development, but an ICU clinician under high cognitive load cannot chase text logs or jump between tabs. They need a high-density, low-latency cockpit. The UI is structured exactly like an IDE layout using Palantir's Blueprint UI components, bathed entirely in a Solarized Light color palette.

The screen is pinned into a rigid 4-pane system to eliminate overlapping windows:

- **Left Pane:** Three horizontal stripes tracking session history ("Previous Chats") and immediate clinical context ("Active Patient Record").
- **Right Pane (Telemetry Layer):** A multi-modal drop zone streaming real-time Docker ingestion over Server-Sent Events (SSE). It visualizes the raw pipeline stages (Surya OCR → Presidio Redaction → Embeddings) via debug bounding boxes.
- **Center-Bottom Pane (Provenance & Graph):** Shows byte-level source PDF coordinates alongside the interactive Neo4j entity-relationship diagram.
- **Center-Top Pane (Synthesis Arena):** Renders the grounded output from the deterministic `agent.py` pipeline, wired to live external APIs like clinicaltrials.gov.

Dark mode was rejected immediately. It fails under the brutal fluorescent glare of an ICU ward, causing severe pupillary fatigue. Solarized Light maintains sharp contrast without washing out. Standard whitespace-heavy SaaS design was also dropped — high-acuity environments require maximum information density. Blueprint UI handles complex, data-dense engineering layouts out of the box.

The main trade-off is front-end state management. High-frequency SSE streaming of OCR bounding boxes and graph vectors can lock up the React main thread. The UI must aggressively throttle incoming telemetry frames to ensure typing and rendering stay smooth.
