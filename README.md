# Clinical Intelligence Platform: Manifest Repository

This repository is the engineering manifest for a local-first, deterministic clinical intelligence platform built across four months of production development and validation. It is a reference companion to the active codebase at https://github.com/ronit22203/clinical-trials-matching-platform.

The manifest answers four questions a reader should be able to resolve in under 10 minutes:

1. What runs?
2. How does it run?
3. What does it depend on?
4. How do I know it works?

---

## What This Platform Is

A complete end-to-end system for medical PDF ingestion, clinical knowledge graph construction, hybrid retrieval, and agentic clinical reasoning. Every component runs locally by default. No patient data, document content, or query text leaves the machine unless explicitly configured. Every run is fingerprintable.

Three production-verified results define the platform:

| Result | Value |
|---|---|
| Acquisition cost | 1,036 medical PDFs at $0.10 total ($0.000096 per document) |
| Retrieval quality | 100.4% Recall@5 on a 20-query golden set with cross-encoder reranking |
| Inference throughput | 62.6 tokens/second at $0.09 per 1M tokens (RTX 5080 spot, SGLang) |

---

## System Components

| Component | What it does |
|---|---|
| data-acquisition | Config-driven source fetchers (bioRxiv, medRxiv, PubMed, ClinicalTrials.gov). Multi-cloud storage chain: S3 primary, Azure Blob fallback, local disk final. SHA-256 dedup before any processing. |
| data-ingestion | Seven-stage pipeline: OCR (Surya), PII redaction (Presidio), chunking, BGE-small embeddings into Qdrant, entity-relation extraction into Neo4j with byte-range provenance and tiered trust labels. |
| agentic-reasoning | LangGraph ReAct agent and Temporal workflow engine sharing the same tool registry. Tools: GraphRAG, PubMed, ClinicalTrials.gov, openFDA, MCP filesystem. YAML-controlled configuration. |
| platform-ui | Static web UI (Alpine.js, Tailwind) over FastAPI. Three panels: AI synthesis, evidence cards with Verify buttons, debug visualizations. |

---

## Manifest File Map

| File | Question answered |
|---|---|
| system.md | Architecture, design principles, stack decisions, runtime boundaries, hardware spectrum |
| run.md | How to start, verify, benchmark, and scale the system |
| eval.md | Retrieval metrics, inference benchmarks, KG extraction quality, deterministic run fingerprints |
| failures.md | 62-entry failure register from production (incidents, root causes, corrective controls) |
| decisions.md | Architectural decision records with rationale and rejected alternatives |
| demo/script.md | 5-minute OCR bottleneck walkthrough companion to the demo video |

---

## Internal Reference Files

The following files are internal working documents and are not part of the public-facing manifest:

| File | Contents |
|---|---|
| weight_0_canonical_code_docs.md | Canonical code documentation, module notes, command surface |
| weight_1_chat_deepseek.md | Failure register sourced from build session histories |
| weight_2_portfolio_docs_live_production.md | Full production engineering report with verified benchmarks |

---

## Active Codebase

https://github.com/ronit22203/clinical-trials-matching-platform
