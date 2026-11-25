# Technical Requirements Document (TRD): VERITAS

**Version:** 1.0  
**Related Docs:** [PRD](./01-PRD.md) | [Workflow Diagrams](./03-WORKFLOW-DIAGRAMS.md) | [API & Data Contracts](./04-API-AND-DATA-CONTRACTS.md)

---

## 1. System Architecture Overview

VERITAS follows an **asynchronous, event-driven multi-agent architecture** separated into four logical tiers:

1. **Presentation Tier (Next.js 14 Cockpit):** TypeScript/React UI consuming REST endpoints and SSE telemetry streams; renders the PRISMA dashboard, split-pane PDF provenance reader (`react-pdf` / PDF.js), and interactive D3.js citation/patent graph.
2. **API & Streaming Tier (FastAPI + Uvicorn):** Validates requests with Pydantic v2, enqueues deep research runs to Celery, and streams LangGraph state events published via Redis Pub/Sub over Server-Sent Events (SSE) / WebSockets.
3. **Orchestration & Agent Tier (Celery + LangGraph `StateGraph`):** Executes the cyclical 8-agent research workflow with Redis state checkpointing, human-in-the-loop interrupts, and bounded Critic reflection loops.
4. **Data, Retrieval & Verification Tier:**
   - **Redis:** Task broker, result backend, Pub/Sub event bus, 24h external API metadata cache, and LangGraph checkpoint store.
   - **PostgreSQL / SQLite (Dev):** Persistent job metadata, PRISMA audit logs, verified bibliography records, and user exports.
   - **ChromaDB + BM25 (`rank_bm25`):** Hybrid vector + lexical chunk index for full-text evidence retrieval.
   - **Local / Object Filesystem (`storage/pdfs/`):** Downloaded open-access PDFs and generated LaTeX/PDF export bundles.

---

## 2. Technology Stack Specification

| Layer | Technology | Rationale |
| :--- | :--- | :--- |
| **Frontend** | Next.js 14 (App Router), React 18, TypeScript, TailwindCSS, Radix UI, Lucide Icons, D3.js | SSR + rich interactive cockpit, force-directed citation graph, clean SSE hooks |
| **Backend API** | Python 3.10+, FastAPI, Uvicorn, Pydantic v2 | Async I/O native, strict schema validation, effortless SSE streaming |
| **Orchestration** | LangGraph (`StateGraph`), LangChain Core | Cyclic DAG support for Critic backtrack loops and human-in-the-loop interrupts |
| **Queue & Cache** | Celery, Redis 7 | Isolates 2–4 min research jobs from web workers; provides Pub/Sub telemetry |
| **LLMs (Tiered)** | Reasoning/Critic: Gemini 1.5 Pro / GPT-4o; Extraction/Screening: lightweight/local Ollama (Llama 3) or Gemini Flash | Balances deep reasoning quality against high-volume abstract screening cost/latency |
| **Retrieval** | ChromaDB, `rank_bm25`, `sentence-transformers`, NetworkX | Hybrid semantic + exact-term matching plus graph centrality/citation metrics |
| **PDF & NLP** | PyMuPDF (`fitz`), spaCy | Fast PDF text + bounding-box coordinate extraction; n-gram & syntactic overlap checks |
| **Verification** | `aiohttp`, Crossref REST API, `doi.org` resolver | Non-blocking concurrent HTTP 200/302 pinging and bibliographic registry verification |
| **Evaluation & Export** | RAGAS, Jinja2 LaTeX templates, `tectonic` / `pdflatex` | Automated faithfulness/recall benchmarking and deterministic IEEE/ACM compilation |

---

## 3. Detailed Multi-Agent Specifications

All agents operate over a shared typed `ResearchState` inside a LangGraph `StateGraph`.

### Agent 1: Plausibility & Sanity Gatekeeper
- **Goal:** Reject nonsensical, physically impossible, or biologically contradictory prompts before spending retrieval budget.
- **Pipeline:**
  1. Extract `(subject, predicate, object)` triples and domain classification from the user query.
  2. Query **Wikidata SPARQL** (`https://query.wikidata.org/sparql`) and **NCBI Taxonomy** (`esearch`/`efetch`) for biological taxa or physical entity properties.
  3. Run an NLI/reasoning plausibility check combining retrieved ontology facts with scientific constraints.
- **Output:** `plausibility_verdict: Literal["plausible", "refuted"]`, `confidence: float`, `ontological_evidence: list[EvidenceRef]`, `refutation_summary: str | None`.
- **Routing:** If `"refuted"`, transition directly to `END` and emit a `refutation` SSE event.

### Agent 2: Methodology & PRISMA 2020 Agent
- **Goal:** Formalize the search strategy according to PRISMA 2020 guidelines.
- **Pipeline:**
  1. Decompose the research topic into 3–6 research sub-questions (`RQ1..RQk`).
  2. Build Boolean search strings (`AND` / `OR` / synonym expansion) tailored per database (arXiv syntax, PubMed MeSH, S2AG, USPTO).
  3. Define `InclusionCriteria` (year range, document types, empirical requirement) and `ExclusionCriteria`.
- **Checkpoint:** Optionally pauses (`interrupt_before` or `pause_for_review` flag) so the user can edit criteria in the Cockpit before retrieval begins.

### Agent 3: Academic Search Agent
- **Goal:** Retrieve candidate scholarly metadata and open-access PDF URLs concurrently.
- **Connectors:**
  - **arXiv API** (`export.arxiv.org/api/query`) — Atom XML feed, open PDF URLs.
  - **Semantic Scholar Graph API (`S2AG`)** — citations, references, openAccessPdf, fieldsOfStudy.
  - **PubMed E-utilities** (`esearch` + `efetch`) — biomedical abstracts, PMIDs, PMCIDs, DOIs.
  - **Crossref REST API** — canonical DOI metadata & license links.
  - **Tavily Web Connector (Optional):** Used only when `include_web_whitepapers=true` to capture technical reports/pre-standards (matches the optional Web Explorer stream in the project synopsis without adding a separate 9th core agent).
- **Caching & Resilience:** Redis 24h cache keyed by `(provider, normalized_query)`; exponential backoff on HTTP 429.

### Dynamic Capability Router (Lightweight Skill Injection)
Instead of running every database on every query, Agent 2 (PRISMA) classifies the topic into a domain tag (`biomedical`, `cs_ai`, `hardware_ip`, `general`) and activates only the matching connector set and prompt rubric:
- `biomedical` → enables PubMed + NCBI Taxonomy + arXiv `q-bio`
- `cs_ai` → enables arXiv + Semantic Scholar + USPTO
- `hardware_ip` → prioritizes USPTO PatentsView + Google Patents + Crossref
- `general` → arXiv + Semantic Scholar + Crossref + optional USPTO

### Agent 4: Patent & Prior-Art Agent
- **Goal:** Surface registered patents and commercial assignees in the same technical domain.
- **Connectors:**
  - **USPTO PatentsView API** — patent IDs, titles, abstracts, CPC classifications, assignees, filing dates.
  - **Google Patents / CDP Browser Transport** — fallback Playwright/CDP extractor with readiness checks for JS-hydrated patent pages when structured APIs lack full claim text.
- **Output:** `PatentRecord` items normalized into the unified `SourceRecord` model (`source_type="patent"`).

### Agent 5: Source & Link Verifier Agent (Zero-Trust Gate)
- **Goal:** Guarantee 0% dead links and 0% fabricated citations.
- **Verification Algorithm:**
  1. **HTTP Liveness:** Concurrent `aiohttp` `HEAD` (fallback `GET` with browser-safe headers, 10s timeout, max 5 redirects) to target URL and `https://doi.org/{doi}`. Accept only final HTTP `200` (or `302` resolving to `200` on publisher domain).
  2. **Crossref Registry Match:** Fetch `/works/{doi}` from Crossref (with polite `mailto` User-Agent). Compute normalized token-set similarity between cited title/first-author/year and Crossref canonical metadata; require `title_similarity >= 0.85` and author overlap.
  3. **Purge Policy:** Any candidate failing HTTP liveness OR Crossref validation is marked `verification_status="purged"`, logged in the PRISMA/Audit trail, and blocked from entering the synthesis corpus.

### Agent 6: IP & Copyright Compliance Agent
- **Goal:** Prevent copyright infringement and flag commercial IP overlaps.
- **Checks:**
  1. **License Audit:** Inspect Crossref `license` field, arXiv license, or OpenAlex/Unpaywall OA status to tag each source (`CC-BY`, `CC0`, `MIT`, `Open-Access-Restricted`, `Proprietary-Abstract-Only`).
  2. **Verbatim Overlap Guard:** Restrict ingestion of `Proprietary-Abstract-Only` items to metadata + abstract; for synthesized prose, compute 6-gram / 8-gram sliding overlap against source chunks and rewrite any passage exceeding the fair-use verbatim threshold (`> 15%` contiguous n-gram overlap without explicit short block-quote attribution).

### Ingestion & PRISMA Screening Service (Shared Subsystem)
- **Deduplication:** Exact match on DOI / arXiv ID / patent ID, followed by normalized title similarity (`>= 0.92`).
- **PRISMA Funnel Execution:**
  1. **Identification:** Total raw records per connector.
  2. **Screening:** Abstract-level LLM/classifier evaluation against `InclusionCriteria`.
  3. **Eligibility:** Full-text PDF download + PyMuPDF (`fitz`) parsing (storing `page_number` and `bbox=[x0, y0, x1, y1]` per chunk) + empirical eligibility check.
  4. **Included:** Verified chunks indexed into **ChromaDB** (dense embeddings via `sentence-transformers`) and **BM25** (sparse lexical index). Build **NetworkX** citation/co-citation + patent-citation graph.

### Agent 7: Academic Synthesizer Agent
- **Goal:** Author a rigorous, structured literature & patent review with granular claim-level provenance.
- **Output Structure:**
  - Executive Abstract & PRISMA Summary Table
  - Conceptual Taxonomy (hierarchical classification of approaches)
  - Comparative Literature & Patent Prior-Art Matrix
  - Detailed Thematic Synthesis (every factual claim bound to `[chunk_id, source_id, page, bbox]`)
  - Research Gaps & Commercial White Spaces

### Agent 8: Peer-Review Critic Agent (Closed-Loop Reflection)
- **Goal:** Enforce empirical grounding and prevent unsupported generalizations.
- **Loop Engineering:**
  1. Decompose draft sections into atomic `(claim_text, cited_chunk_ids)` pairs.
  2. Compute NLI entailment score $e_i \in [0, 1]$ between cited chunks and each claim (using a lightweight local cross-encoder such as `cross-encoder/nli-deberta-v3-small`, with structured LLM scoring as fallback when GPU/model weights are unavailable).
  3. Compute rubric scores: coverage of RQs, patent integration, logical coherence.
  4. **Decision:** If $\min_i e_i < 0.90$ or coverage gaps exist (and `loop_count < max_loops`, default `max_loops = 2`), generate targeted follow-up search queries and route back to **Academic/Patent Search → Verification → Synthesizer**. Otherwise approve and proceed to Export.

### Model Routing Policy (Simple 2-Tier Split)
| Task Tier | Agents | Recommended Model |
| :--- | :--- | :--- |
| **Reasoning & Synthesis** | Plausibility, PRISMA, Synthesizer, Critic | Gemini 1.5 Pro / GPT-4o |
| **High-Volume Screening** | Abstract Screening, Metadata Extraction | Gemini Flash / Local Llama 3 (Ollama) |

---

## 4. Export Engine Technical Spec

- **BibTeX Generator:** Deterministic citation keys (`AuthorYearFirstWord`), escaped LaTeX special characters, verified `doi`, `url`, `note = {Verified HTTP 200}`.
- **LaTeX Templates (Jinja2):**
  - `ieee_tran` (`IEEEtran.cls` two-column conference/journal layout)
  - `acm_sigconf` (`acmart.cls` layout)
  - Auto-generated PRISMA 2020 counts table (`\begin{table}`) and taxonomy comparison table.
- **Bundling:** Outputs `veritas_export_{job_id}.zip` containing `main.tex`, `references.bib`, and optional `main.pdf` (compiled via `tectonic` or `pdflatex` + `bibtex` inside an isolated worker subprocess with timeout).

---

## 5. Scalability, Resilience & Deployment

- **Async Queue:** FastAPI never runs LangGraph in-process during an HTTP request; it dispatches `run_research_task.delay(job_id)` to Celery.
- **Telemetry Bus:** LangGraph node callbacks publish JSON events to Redis channel `veritas:events:{job_id}`; FastAPI SSE endpoint subscribes and streams to the browser with heartbeat keepalives every 15s.
- **Containerization:** `docker-compose.yml` defines `frontend` (Next.js), `backend` (FastAPI), `worker` (Celery), `redis`, and `chromadb` (or persistent volume) for one-command local & staging startup.
