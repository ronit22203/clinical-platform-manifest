# Architectural Decision Records

Each ADR documents a key design choice, the context and constraints that drove it, the alternatives that were seriously considered, and the current status.

---

## ADR-001: Hybrid vector + graph storage (Qdrant + Neo4j)

**Status:** Accepted  
**Context:** Clinical queries routinely require multi-hop reasoning across entity relationships (Drug TREATS Condition CONTRAINDICATED_WITH Drug). Dense vector similarity search returns semantically close chunks but cannot traverse typed relationships.  
**Decision:** Run Qdrant for semantic vector search and Neo4j for structured entity-relationship traversal. Results from both are merged at query time via the HybridRetriever.  
**Rejected alternatives:**
- Qdrant alone: cannot resolve multi-hop clinical relationship chains
- Pinecone: sends data to external API, violates local-first constraint
- Weaviate with graph module: more complex operational surface than Qdrant + Neo4j separately  
**Trade-off:** Two stateful services to operate and maintain. On small instances (t4g.small), memory management is non-trivial (see F-010, F-011).

---

## ADR-002: Dual agent runtime (LangGraph ReAct + Temporal Workflow)

**Status:** Accepted  
**Context:** Interactive clinical queries need low-latency streaming responses. Regulated research workflows need fault-tolerant, replayable, auditable execution with an optional Human-in-the-Loop gate.  
**Decision:** Expose both runtimes from the same tool registry and YAML configuration. The runtime choice is a single YAML flag. LangGraph handles interactive sessions; Temporal handles durable, auditable workflows.  
**Rejected alternatives:**
- LangGraph only: no durable checkpoint, mid-run worker crash loses execution state
- Temporal only: too much overhead for simple interactive queries, no streaming
- Celery: no LLM-native tool-calling support, no HITL primitives  
**Trade-off:** Two orchestration runtimes with different failure modes. Temporal requires PostgreSQL to be running (see F-061, F-062).

---

## ADR-003: Local OCR engine (Surya) over cloud document APIs

**Status:** Accepted  
**Context:** Medical PDFs contain Protected Health Information. Sending documents to remote OCR APIs (AWS Textract, Google Document AI, Azure Document Intelligence) creates HIPAA/GDPR compliance risk. Multi-column medical layouts are structurally complex and require layout-aware extraction.  
**Decision:** Surya runs on-device and handles multi-column layouts, table linearization, and reading order reconstruction. Cost per page is zero.  
**Rejected alternatives:**
- AWS Textract: sends raw document content to AWS, not acceptable for PHI documents
- Google Document AI: same data egress concern
- pypdf/pdfminer: scrambles column order and discards table structure (verified in testing)
- Azure Document Intelligence: retained as fallback only, not primary path  
**Trade-off:** Surya is the single largest bottleneck; 94-97% of ingestion pipeline runtime (F-051). GPU acceleration for Surya's table detection model is the highest-leverage remaining optimization.

---

## ADR-004: Local-first inference (Ollama/LiteLLM then SGLang)

**Status:** Accepted  
**Context:** Clinical query content is sensitive. API-based inference sends query text and retrieved document content to third-party providers. Cost at scale with GPT-4 class models is prohibitive.  
**Decision:** Default inference path is local (Ollama or LM Studio). Production path is SGLang on VAST.ai RTX 5080 spot at $0.017/hr. Model name is a single YAML value, swappable without code changes.  
**Rejected alternatives:**
- OpenAI API: $30/1M tokens for GPT-4, 333x the spot GPU cost; query content leaves machine
- Anthropic API: $0.25/1M tokens (Claude 3 Haiku); data egress concern  
**Trade-off:** Local inference has known failure modes on M4 (silent timeout F-047, factual hallucination F-048). Production workloads require the SGLang path.

---

## ADR-005: PII redaction as a pipeline structural firewall (Presidio)

**Status:** Accepted  
**Context:** Clinical documents may contain Protected Health Information. Redacting PII as a post-processing step allows PHI to temporarily enter the vector database or knowledge graph.  
**Decision:** TextCleaner with Microsoft Presidio runs as a mandatory pipeline stage before any text reaches the embedding model or Neo4j. Custom recognizers added for Singapore NRIC/FIN and Medical Registration Numbers (not covered by standard Presidio models). Redaction is not configurable off.  
**Rejected alternatives:**
- Regex only: insufficient coverage for regional PII formats
- Post-storage redaction: PHI already in vector index before redaction runs
- Optional toggle: compliance guarantee requires structural enforcement, not configuration  
**Trade-off:** Approximately 15% processing overhead per document.

---

## ADR-006: YAML-driven agent configuration (no code deploys for behavior changes)

**Status:** Accepted  
**Context:** Clinical agent behavior (system prompts, tool selection, model selection, retry policy, HITL gates) needs to be modifiable without code deployments. Clinicians and ops staff are not Python developers.  
**Decision:** All agent behavior is declared in YAML and validated by Pydantic at startup. Tool implementations are registered via a plugin model (class + YAML entry). Model, prompt, and tool changes require no Python changes.  
**Rejected alternatives:**
- Hardcoded Python configuration: every behavior change requires a code review and deploy
- Environment variables only: insufficient structure for nested agent configuration  
**Trade-off:** YAML validation errors at startup are the failure mode; Pydantic catches schema violations before any queries run.

---

## ADR-007: Cross-encoder reranker (BAAI/bge-reranker-base) after vector retrieval

**Status:** Accepted  
**Context:** Dense vector dot-product ranking is fast but imprecise. Clinical queries benefit from a more accurate relevance signal, especially on quantitative and methodology question types.  
**Decision:** Apply BGE cross-encoder reranker after Qdrant vector retrieval, before graph enrichment. Top-K chunks after reranking are enriched with Neo4j graph facts.  
**Measured impact:** Recall@5 +11.6 points, Precision@5 +6.0 points, HitRate@5 +5.0 points (run `det_20260425_113342_e14da50` vs `det_20260424_144844_fcb7c39`).  
**Rejected alternatives:**
- Cohere Rerank API: sends document content to external API
- No reranker: leaves Recall@5 at 88.8% vs 100.4%  
**Trade-off:** Comparative query category regressed by 16.7 R@5 points with reranker applied. Root cause not yet resolved.

---

## Roadmap (from weight_2, April 2026)

| Feature | Rationale | Priority |
|---|---|---|
| GPU-accelerated OCR table detection | Eliminate the 94-97% ingestion bottleneck | High |
| Empty output detection + query timeout | Fix M4 silent failure mode (F-047) | High |
| RAG grounding for clinical trials queries | Fix M4 hallucination mode (F-048) | High |
| Fine-tuned clinical LLM | BioMistral or OpenBioLLM for improved extraction F1 | High |
| Formal HIPAA compliance audit preparation | Pre-commercialisation requirement | High |
| Lightweight local dev profile | Reduce Docker resource consumption on M4 | Medium |
| REST API / FastAPI gateway | EHR integration | Medium |
| RAGAS evaluation framework | Continuous synthesis quality measurement | Medium |
| Additional data sources | PubMed Central, Europe PMC | Medium |
| Streaming HITL via WebSocket | Replace CLI approval with browser-based review | Medium |
