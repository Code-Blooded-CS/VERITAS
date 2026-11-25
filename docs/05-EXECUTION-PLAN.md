# Project Execution Plan & Team Roadmap: VERITAS

**Team:** Code Blooded  
**Related Docs:** [PRD](./01-PRD.md) | [TRD](./02-TRD.md) | [Synopsis](../synopsis.tex)

---

## 1. Alignment with 6-Month Academic Schedule

This execution plan translates the 6-month milestone table in [synopsis.tex](../synopsis.tex#L240-L260) into concrete engineering phases and verifiable deliverables.

| Academic Month | Synopsis Milestone | Engineering Phase | Exit Criteria |
| :--- | :--- | :--- | :--- |
| **Month 1** | Requirements & Ontology Setup | **Phase 0 & Phase 1A:** Documentation freeze, repo scaffold, Plausibility Gatekeeper (Wikidata + NCBI) | Docs approved; impossible-premise test suite passes 100% |
| **Month 2** | Retrieval & Ingestion Pipelines | **Phase 1B & Phase 2:** Academic & Patent connectors, PyMuPDF bbox parser, ChromaDB + BM25 | Live queries return normalized `SourceRecord` + indexed `PDFChunk`s |
| **Month 3** | Multi-Agent Graph Implementation | **Phase 1C & Phase 2B:** LangGraph `StateGraph`, PRISMA 2020 agent, Synthesizer + Critic loop | End-to-end review loop runs with SSE events & bounded Critic retries |
| **Month 4** | Verification & IP Guardrails | **Phase 3:** Async HTTP 200 + Crossref auditor, license classifier, n-gram overlap guard | 0% dead links/fake DOIs in output; IP guardrails verified |
| **Month 5** | Interface & IEEE/ACM Export | **Phase 4:** Next.js 14 Cockpit, split-pane PDF highlighter, D3.js graph, LaTeX/PDF exporter | Click-to-highlight provenance works; IEEE & ACM ZIP compiles cleanly |
| **Month 6** | Benchmarking & Final Dissertation | **Phase 5:** RAGAS benchmarks, PRISMA 2020 compliance audit, user study, thesis submission | RAGAS faithfulness $\ge 0.88$; final demo & dissertation ready |

---

## 2. Detailed Engineering Phases

### Phase 0: Repository & Infrastructure Scaffold (Week 1)
- Create `backend/` (FastAPI, LangGraph, Celery, pytest), `frontend/` (Next.js 14, TailwindCSS, TypeScript, D3.js), `docker-compose.yml` (Redis, backend, worker, frontend), and `.env.example`.
- Implement canonical Pydantic models from [04-API-AND-DATA-CONTRACTS.md](./04-API-AND-DATA-CONTRACTS.md).
- Configure linting (`ruff`, `eslint`, `tsc --noEmit`) and CI workflow.

### Phase 1: Vertical Slice MVP (Weeks 2–4)
- **Backend:** FastAPI `POST /research` + SSE stream; 4-node LangGraph (`Plausibility` → `PRISMA` → `arXiv + Crossref` → `Synthesizer + 1x Critic`).
- **Frontend:** Query form, live SSE timeline, Markdown synthesis viewer with verified HTTP 200 badges.
- **Milestone Demo:** Live run showing (a) an impossible query refuted and (b) a real query returning verified arXiv citations.

### Phase 2: Full Retrieval, Patents & Hybrid Ingestion (Weeks 5–8)
- Add Semantic Scholar (S2AG), PubMed E-utilities, USPTO PatentsView, and Google Patents / CDP fallback.
- Implement Redis 24h response caching and per-API rate limiters.
- Implement PyMuPDF (`fitz`) page + `bbox` extraction, deduplication, 4-stage PRISMA funnel tracking, and hybrid BM25 + ChromaDB + NetworkX indexing.

### Phase 3: Zero-Trust Verification, IP Audit & LaTeX Export (Weeks 9–12)
- Build concurrent `aiohttp` + Crossref metadata matcher with automatic reference purging and audit logging.
- Build IP/License tagger and syntactic n-gram overlap rewriter.
- Implement full NLI ($\tau = 0.90$) Peer-Review Critic loop with targeted re-retrieval.
- Build Jinja2 IEEE/ACM LaTeX + `.bib` ZIP exporter and optional `tectonic` PDF compiler.

### Phase 4: Interactive Cockpit & Async Scaling (Weeks 13–16)
- Move execution fully onto Celery workers with Redis LangGraph checkpoints and human-in-the-loop pause/resume.
- Build Split-Pane PDF Provenance Inspector (`bbox` highlight overlay) and D3.js force-directed Citation & Patent Network.

### Phase 5: Evaluation, Hardening & Thesis (Weeks 17–20)
- Execute full test plan ([07-TEST-AND-EVAL-PLAN.md](./07-TEST-AND-EVAL-PLAN.md)) including RAGAS benchmarks and adversarial impossible-query testing.
- Finalize `CONTRIBUTING.md`, `LICENSE` (MIT), deployment guide, and university dissertation.

---

## 3. Team Ownership Matrix (`Code Blooded`)

| Member | Role | Owned Modules | First Deliverables |
| :--- | :--- | :--- | :--- |
| **Pritam Jana** | Project Lead & Agent Architect | `backend/app/graph/`, `backend/app/agents/synthesizer.py`, `critic.py`, `workers/` | `ResearchState`, LangGraph `StateGraph`, Celery + Redis SSE pub/sub, Critic loop |
| **Sachin Kumar Singh** | Full-Stack & Interface Lead | `frontend/` (Next.js 14, SSE hooks, PDF bbox viewer, D3.js graph) | Cockpit layout, live SSE telemetry feed, split-pane provenance reader |
| **Rohit Kumar Ojha** | Data Ingestion & Pipeline Lead | `backend/app/connectors/`, `backend/app/ingestion/` | arXiv, S2AG, PubMed, USPTO/Google Patents connectors, PyMuPDF + ChromaDB + BM25 |
| **Sagar Jha** | Review Protocol Lead | `backend/app/agents/plausibility.py`, `prisma.py`, `screening/` | Wikidata SPARQL + NCBI Taxonomy gatekeeper, PRISMA 2020 query & screening funnel |
| **Bhaskar Raj** | Verification & Export Lead | `backend/app/verification/`, `backend/app/export/` | Async `aiohttp` + Crossref auditor, license/overlap guard, IEEE/ACM LaTeX & `.bib` |

---

## 4. Engineering Workflow & Definition of Done (DoD)

- **Branching:** Trunk-based workflow (`feature/<module>-<short-desc>` → PR into `main`).
- **PR Checklist:**
  1. Matches schemas in `04-API-AND-DATA-CONTRACTS.md`.
  2. Unit tests pass (`pytest` / `vitest`) with mocked external APIs.
  3. Linters clean (`ruff check`, `npm run lint`, `tsc --noEmit`).
  4. At least 1 peer review approval before merge.

---

## 5. Risk Register & Mitigations

| Risk | Impact | Likelihood | Mitigation |
| :--- | :--- | :--- | :--- |
| **External API Rate Limits (S2AG / Crossref / USPTO)** | Slow or failing retrieval | High | Redis 24h caching, polite `mailto` headers, exponential backoff, graceful connector degradation |
| **Google Patents JS Hydration / Scraping Blocks** | Missing patent full text | Medium | Primary reliance on USPTO PatentsView REST API; CDP fallback only for needed pages; no CAPTCHA bypassing |
| **LLM Hallucinated Claims Bypassing Critic** | Violates core accuracy promise | Medium | Strict chunk-ID citation constraint + NLI $\ge 0.90$ verification + deterministic URL/DOI gate |
| **2–4 Min Research Timeout in Browser** | Failed UX | High | Celery background worker + Redis checkpointing + SSE heartbeats |
| **PDF Parsing Failure on Scanned/Malformed PDFs** | Missing provenance bboxes | Medium | Fallback to abstract-only indexing with explicit `excluded_eligibility` logging |
