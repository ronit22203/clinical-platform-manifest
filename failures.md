# Failure Register

This is an append-only incident register. Entries are sourced directly from build session histories and production run logs. Every entry includes a hard signal: a metric, test outcome, error text, or reproducer.

## Enforcement

- New entries are appended with a new ID. Existing entries are never rewritten.
- Every entry must include at least one recorded piece of evidence.
- If two incidents look similar but occurred in different contexts, keep both.

---

## Incident Register (F-001 to F-062)

| ID | Source context | Specific failure | Recorded evidence |
|---|---|---|---|
| F-001 | Ingestion run | PubMed open-access filter returned no usable PDFs | 50 requests -> 0 PDFs |
| F-002 | Ingestion run | ClinicalTrials protocol links returned no valid protocol PDFs | 10 studies -> 0 valid PDFs |
| F-003 | Ingestion run | CT.gov redirect chain degraded to dead targets | 301 -> 404 for many links |
| F-004 | Storage integration | S3 `upload_fileobj()` call used wrong ContentType placement | `ContentType` passed directly, not inside `ExtraArgs`; 3 retry failures then Azure fallback |
| F-005 | OCR cost control | Azure Document Intelligence burst spend was uncontrolled | One complex batch hit $50/day |
| F-006 | Spot resilience | Spot termination detection was too slow before hardening | 2 min detection reduced to 30 sec after chaos tests |
| F-007 | OCR performance | Small OCR tail dominated end-to-end latency | <2% of PDFs caused 40% of processing latency |
| F-008 | Metadata integrity | Schema evolution broke hash compatibility | SHA256 version mismatches after schema changes |
| F-009 | Corpus quality | Duplicate corpus volume was unexpectedly high | 43% duplicate detection rate |
| F-010 | Vector infra | Qdrant ran out of memory under load on small node | 12GB vector store on `t4g.small` (4GB RAM) |
| F-011 | Graph infra | Neo4j connection pool settings became a bottleneck under spikes | 0-3 pool worked for batch, spike tuning required |
| F-012 | Observability | Silent pipeline loss existed without alerting | 0.4% document loss before counters/alerts were added |
| F-013 | Reliability baseline | Core platform had recurring operational failure cadence | MTBF 11.3 days |
| F-014 | AWS quotas | Spot quota approval was not usable immediately | Approved limit took 18 days to propagate |
| F-015 | AWS quotas | On-demand quota default prevented launch | vCPU quota defaulted to 0, separate request required |
| F-016 | AWS UX | Quota request path was hard to discover | Service Quotas links/UI buried |
| F-017 | AMI bootstrap | AL2023 package conflict broke user-data bootstrap | `curl-minimal` conflict; explicit curl install removed |
| F-018 | GPU bootstrap | Manual NVIDIA driver path failed on AL2023 | Switched to Deep Learning AMI |
| F-019 | Container runtime | `nvidia-docker` repo unavailable on AL2023 path | Repository 404 |
| F-020 | Access control | SSH failed due to SG scope errors | Missing `My IP` or VPC CIDR rules |
| F-021 | Instance access | EC2 Instance Connect failed when bootstrap damaged sshd | User-data breakage affected SSH daemon |
| F-022 | Networking | DB connectivity failed from network placement mismatch | Cross-AZ/VPC mismatches blocked connectivity |
| F-023 | Spot lifecycle | One-time spot requests could not be stopped cleanly | Must terminate instead of stop |
| F-024 | Spot lifecycle | Spot termination protection unavailable | Platform limitation noted during ops |
| F-025 | Quota telemetry | Quota utilization reporting was stale after termination | Console lag/inconsistency |
| F-026 | Console UX | Form state was lost on launch/config errors | Config reset on error |
| F-027 | Console UX | Limits location was non-obvious | Hidden under Auto Scaling -> Limits |
| F-028 | Console UX | Error messages were inconsistent across limit failures | Spot count vs vCPU messaging mismatch |
| F-029 | VPC routing | Timeouts caused by layered network misconfig | IGW/route table/SG/NACL issues combined |
| F-030 | Client networking | Mobile hotspot and Private Relay caused IP churn lockouts | SG required repeated manual updates |
| F-031 | Endpoint stability | Public IP drift after stop/start broke scripts | Hardcoded endpoints invalidated |
| F-032 | IAM bootstrap | Missing instance profile produced SSM/metadata errors | Not root SSH cause, but operational noise |
| F-033 | Dual-stack ops | IPv4/IPv6 rule visibility in CLI output was misleading | `--output table` hid CIDRs |
| F-034 | Ops complexity | Troubleshooting required cross-layer deep inspection | VPC, subnet, IGW, routes, NACL, SG, host firewall, Docker |
| F-035 | Network asymmetry | IPv6-only client to IPv4-only EC2 stalled requests | `curl` required `-4`; CGNAT blocked high inbound ports |
| F-036 | Host networking | Open SG rules were insufficient without host bind/firewall fixes | Needed host firewall correction and IPv4-only binding |
| F-037 | Data persistence | Anonymous Docker volumes lost data on recreate | Switched to named volumes + `restart: always` policy |
| F-038 | Neo4j runtime | Plugin stack failed without strict env and permissions | Required explicit env vars and `chown` on volume |
| F-039 | Local LLM runtime | Ollama timed out on heavy chunk processing | 2 min timeout required retry/backoff |
| F-040 | Agent behavior | Tool selection policy failed consistency tests | 7/20 tests failed; over-trigger GraphRAG, under-trigger MCP/PubMed |
| F-041 | Dependency mgmt | NLP stack deps caused long cold starts and failures | sentence-transformers/torch/HF token issues; first call 90 sec delays |
| F-042 | Service coupling | Multi-service runtime fragile to single-service outages | One down service broke hybrid search |
| F-043 | API resiliency | FDA tool timed out without resilience policy | Test 2 timed out at 120 sec; no retry/backoff at time of failure |
| F-044 | Runtime drift | Worker used wrong Python interpreter | Local venv vs global pip mismatch; fixed with explicit `.venv/bin/python` |
| F-045 | Embedding cold start | Model load repeated across sessions | SentenceTransformer cold start 30-90 sec |
| F-046 | Retrieval fusion | Hybrid graph+vector merge incomplete | Routing remained rule-based, not learned |
| F-047 | Inference stability | Silent model failure returned empty output | Q18: 0 tokens after 5 min, `error: null` |
| F-048 | Clinical factuality | Answer synthesis inverted trial conclusion | TOPAZ-1 hallucination (inverted result) |
| F-049 | Inference utilization | Local M4 utilization was poor for target workload | 15% memory bandwidth utilization baseline |
| F-050 | GPU startup | Serving cold start cost remained material | 6-10 sec model/server init on GPU |
| F-051 | OCR bottleneck | OCR dominated ingestion runtime share | 94-97% of pipeline time |
| F-052 | Duplicate pressure | Cross-source duplication remained severe in larger run | 67.6% duplicate PDFs |
| F-053 | Agent reproducibility | Agent behavior lacked deterministic replay initially | Non-reproducible runs before deterministic logs/audit trail |
| F-054 | GPU economics | Spot preemption and idle windows wasted budget | Required auto-stop + queue controls |
| F-055 | Ollama robustness | Sustained inference unstable under real runs | Timeouts, 404s, JSON parse errors |
| F-056 | KG extraction prompt contract | BioMistral template rejected `system` role | Silent KG extraction failure until prompt merged into `user` message |
| F-057 | Pipeline orchestration | Graph build executed twice from Makefile flow | Stage-6 + separate `ingestion-neo4j-build` target both fired |
| F-058 | Retrieval contamination | Watermark boilerplate polluted chunk set | 12 of 24 chunks were boilerplate lines (medRxiv watermark) |
| F-059 | Async runtime | FastAPI loop conflict from nested event loop call | `asyncio.run()` called inside active loop |
| F-060 | LLM API contract | LM Studio rejected `response_format=json_object` in agent path | Directive removed for agent calls; only used in KG extraction |
| F-061 | Temporal local stack | Health checks failed due to slow auto-setup startup | Startup >2 min triggered false health check failures |
| F-062 | Postgres local stack | Bind-mounted data dir corrupted by host artifacts | `.DS_Store` blocked `initdb` |

---

## Known Current Limitations

These are open, not yet resolved. They are distinct from the above incidents, which all have corrective controls in place.

| Limitation | Detail |
|---|---|
| OCR is the throughput bottleneck | Surya table detection consumes 94-97% of total ingestion runtime on all hardware. GPU acceleration for this stage is the single highest-leverage optimization not yet deployed. |
| Inference failure modes on local hardware | M4 baseline produced silent failure (Q18: 5 min, 0 tokens, no error) and factual hallucination (Q82: inverted TOPAZ-1 trial outcome). Mitigations defined but not yet deployed. Production workloads must route through SGLang stack. |
| Full Docker stack is heavy for local dev | Temporal + PostgreSQL + Qdrant + Neo4j together consume significant local resources. A lightweight local profile (Qdrant-only, no Temporal/Neo4j) that still runs full ingestion is planned. |
| UI is functional but not feature-complete | Supports upload, query, evidence display, verification. Does not support multi-user auth, saved queries, or FHIR integration. |
| Comparative query retrieval regressed with reranker | Per-category breakdown shows -16.7 R@5 for comparative queries. Root cause under investigation. |

## Failure Register – Entries F-063 to F-067

Below are the five new failures, documented from the recent integration work and RunPod deployment attempts.

---

### F-063: Ingestion API port mismatch (8000 vs 8001)

**Context:** React UI attempted to call ingestion endpoints on port 8000 (reasoning API) instead of 8001.

**Failure Evidence:**

- `POST http://localhost:8000/api/ingest → 404`
- SSE connection to `http://localhost:8000/api/ingest/stream` never opened

**Root Cause:** `dev` target launches reasoning API on :8000 and ingestion API on :8001, but React `.env` pointed all API calls to a single base URL.

**Fix:**

- Split API clients: `REACT_APP_REASONING_API=http://localhost:8000`, `REACT_APP_INGESTION_API=http://localhost:8001`
- Update fetch calls to use correct client per endpoint

**Prevention:** `validate` target now checks both ports. Add to CI.

---

### F-064: Wrong endpoint for trial matching (`/api/query` vs `/api/match`)

**Context:** React UI called `POST /api/query` with JSON body; actual endpoint is `POST /api/match` with form data.

**Failure Evidence:**

- `POST /api/query → 404`
- `POST /api/match` with `{"query": "..."}` → 422 (expects form data, not JSON)

**Root Cause:** FastAPI `server.py` defines `async def match(request: Request)` that reads form data (`await request.form()`), not JSON body. React sent `Content-Type: application/json`.

**Fix:**

- Change FastAPI to accept JSON via Pydantic `MatchRequest` model
- Or change React to send `FormData`

**Decision:** Use JSON (cleaner). Add Pydantic model.

**Prevention:** OpenAPI schema generation + React client types.

---

### F-065: SSE event parser assumed `event:` prefix, but backend sends raw `data:` lines

**Context:** React UI used `EventSource` which routes on `event:` field. Ingestion API emits `data: {"stage": "ocr", "status": "running"}` with no `event:` line.

**Failure Evidence:**

- `EventSource` `currentEvent` always empty → all events routed to default handler
- No per‑stage UI updates; only "done" showed

**Root Cause:** `EventSource` spec expects:

```
event: stage_update
data: {"stage": "ocr", "status": "done"}
```

But backend sent only `data:` lines.

**Fix:** Backend: add `event: stage_update` before each `data:` line. Or frontend: ignore `currentEvent` and treat all events as stage updates.

**Decision:** Fix backend to be spec‑compliant.

**Prevention:** Unit test for SSE event shape. Postman collection for manual verification.

---

### F-066: `add-apt-repository` crashed due to Python apt_pkg binding mismatch

**Context:** Running `pre_requisites.sh` on RunPod Ubuntu 22.04 container to upgrade to Python 3.12.

**Failure Evidence:**

```
Traceback (most recent call last):
  File "/usr/bin/add-apt-repository", line 5, in <module>
    import apt_pkg
ModuleNotFoundError: No module named 'apt_pkg'
```

**Root Cause:** `add-apt-repository` is a Python script that depends on `python3-apt` – native C++ bindings compiled for the system's default Python version (3.10). Partial system updates broke symlinks.

**Fix:**

- Use `apt install python3.12 python3.12-venv` directly (no deadsnakes PPA)
- Or use RunPod's built‑in PyTorch image (already has Python 3.11+)
- Or skip `pre_requisites.sh` entirely – pre‑bake an image

**Prevention:** Test `pre_requisites.sh` on a fresh Ubuntu container before remote runs.

---

### F-067: Docker not found in RunPod secure cloud pod

**Context:** `make bootstrap` validation on RunPod Secure Cloud Pod.

**Failure Evidence:**

```
Checking dependencies…
✗  Docker not found. Install from https://docs.docker.com/get-docker/
make: *** [Makefile:97: bootstrap] Error 1
```

**Root Cause:** RunPod Secure Cloud Pods are minimal execution containers – they do not ship with a nested Docker engine. The monorepo expects `docker-compose` to run Neo4j and Qdrant as child containers.

**Fix (choose one):**

1. Use RunPod's "Docker" template (has Docker pre‑installed)
2. Run services natively: install Neo4j + Qdrant directly on the host (no Docker)
3. Modify the platform to use Podman (daemonless, works in containers)
4. Pre‑bake a custom RunPod image with Docker + your repo

**Prevention:** Always check `docker --version` first. Use RunPod community images labelled with `docker` or `podman`. For pure inference (no Neo4j/Qdrant), skip Docker entirely.

---

### F-068: Spot instance root filesystem (20GB) too small for full GPU dependency install

**Context:** Attempting production startup on a spot instance with 20GB root partition.

**Failure Evidence:**

- `No space left on device` during `pip install torch`
- `Disk quota exceeded` when installing `sglang[all]`
- `data-ingestion` attempted to pull CUDA‑13 packages, exhausting remaining space

**Root Cause:**
Startup scripts assume large root volume or no heavy GPU libraries. On constrained instances, pip cache, temporary build directories, and venvs fill the root partition before services complete.

**Fix:**

- Redirect `PIP_CACHE_DIR` and `TMPDIR` to `/workspace`
- Place venvs on `/workspace`
- Split bootstrap into light (API) and heavy (inference/OCR) phases
- Document minimum storage requirements (≥50GB root or dedicated `/workspace`)

**Prevention:**
Add `df -h` check to `make bootstrap`. Warn if <30GB free. Use `--target` to install only required subsets.

### Summary of new failures

| ID | Problem | Status |
|----|---------|--------|
| F-063 | Port mismatch (8000 vs 8001) | Fixed |
| F-064 | Wrong endpoint `/api/query` | Fixed |
| F-065 | SSE missing `event:` prefix | Fixed |
| F-066 | `add-apt-repository` Python binding crash | Workaround: use PyTorch image |
| F-067 | Docker missing in RunPod pod | Workaround: use native services or Docker template |

All have corrective actions documented. The platform continues to harden.
