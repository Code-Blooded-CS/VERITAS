# Product Requirements Document (PRD): VERITAS

**Product Name:** VERITAS (*Verified Evidence-Based Research, IP & Autonomous Synthesis System*)  
**Version:** 1.0 (Pre-Implementation Baseline)  
**Team:** Code Blooded  
**Related Docs:** [TRD](./02-TRD.md) | [Workflow Diagrams](./03-WORKFLOW-DIAGRAMS.md) | [Execution Plan](./05-EXECUTION-PLAN.md)

---

## 1. Executive Summary & Problem Statement

Modern generative AI assistants produce fluent literature summaries but fail four core requirements of rigorous academic and enterprise R&D:
1. **Premise Gullibility:** Single-agent LLMs uncritically hallucinate pseudoscientific reviews for biologically or physically impossible prompts (e.g., *"an elephant hatching an egg"*).
2. **Fabricated Citations & Link Rot:** Generic LLMs invent DOIs, mismatch authors/venues, and emit URLs that return HTTP 404/403.
3. **Absence of Systematic Methodology:** Standard RAG wrappers perform ad-hoc keyword search without adhering to **PRISMA 2020** (Preferred Reporting Items for Systematic Reviews and Meta-Analyses) screening and audit trails.
4. **Siloed Literature & IP:** Academic papers and patent prior art (USPTO, Google Patents) are treated separately, and open-access/fair-use licensing is ignored.

**VERITAS** solves these failures through a **zero-trust, closed-loop multi-agent research platform** combining ontological plausibility gatekeeping, PRISMA 2020 screening, dual academic + patent discovery, live HTTP/Crossref verification, split-pane PDF provenance inspection, and one-click IEEE/ACM LaTeX export.

---

## 2. Target Users & Personas

| Persona | Profile | Primary Pain Point | Value Delivered by VERITAS |
| :--- | :--- | :--- | :--- |
| **Academic Researcher / Grad Student** | M.Tech / Ph.D. / B.Tech thesis student | Weeks spent manually screening papers, checking DOIs, and formatting IEEE/ACM `.bib` & `.tex` | Automated PRISMA review, 100% live-verified citations, and one-click LaTeX/BibTeX export |
| **Enterprise R&D / Patent Analyst** | Applied scientist or innovation engineer | Needs to map academic state-of-the-art against active commercial patents to find white space | Concurrent literature + USPTO/Google Patents prior-art landscaping and assignee graph |
| **Thesis Advisor / Peer Reviewer** | Faculty guide or journal reviewer | Cannot trust AI-written surveys without checking every claim against the original PDF | Split-pane claim-to-PDF passage highlighter with HTTP 200 & Crossref verification badges |

---

## 3. Product Goals & Success Metrics (KPIs)

| Metric | Target | Measurement Method |
| :--- | :--- | :--- |
| **Live Citation Validity Rate** | **100%** | Every citation in the final output resolves HTTP `200`/`302` and matches Crossref title + authors |
| **Fabricated DOI / Hallucinated Paper Rate** | **0%** | Automated audit against Crossref/arXiv/PubMed/USPTO registries |
| **Impossible-Premise Interception Rate** | **100%** on benchmark suite | Adversarial test set of impossible biological/physical queries refuted before retrieval |
| **Claim Entailment Grounding (NLI)** | **$\ge 0.90$** average | Peer-Review Critic NLI scoring against retrieved source chunks |
| **RAGAS Faithfulness & Context Recall** | **$\ge 0.88$** / **$\ge 0.85$** | Evaluated via RAGAS suite in Phase 5 |
| **PRISMA 2020 Audit Completeness** | **100%** of search/screen counts logged | Structured Identification → Screening → Eligibility → Included funnel output |
| **End-to-End Deep Review Latency** | **2–4 minutes** (async, non-blocking) | Measured via Celery job telemetry with real-time SSE updates |

---

## 4. Functional Requirements

### FR-1: Ontological Plausibility Gatekeeper
- **FR-1.1:** Intercept every user prompt prior to external literature retrieval.
- **FR-1.2:** Extract semantic subject–predicate–object triples and validate against Wikidata SPARQL and NCBI Taxonomy (plus LLM scientific commonsense reasoning).
- **FR-1.3:** If a fundamental physical, chemical, or biological contradiction is detected, halt the pipeline and return a structured **Refutation Report** citing the ontological contradiction (e.g., *Loxodonta* $\in$ *Mammalia* / viviparous vs. oviparous reproduction).

### FR-2: PRISMA 2020 Systematic Review Engine
- **FR-2.1:** Decompose a valid research topic into 3–6 structured sub-questions.
- **FR-2.2:** Generate database-specific Boolean search strings (synonyms, MeSH/domain terms) and explicit inclusion/exclusion rules (date bounds, study type, language, peer-review status).
- **FR-2.3:** Maintain an immutable **PRISMA Flow Log** recording counts at Identification, Deduplication, Title/Abstract Screening, Full-Text Eligibility, and Final Included stages, along with exclusion reasons.

### FR-3: Dual Academic & Patent Prior-Art Discovery
- **FR-3.1:** Query academic indices concurrently: **arXiv**, **Semantic Scholar (S2AG)**, **PubMed**, and **Crossref** (plus optional Tavily web whitepapers).
- **FR-3.2:** Query patent registries concurrently: **USPTO PatentsView** and **Google Patents** (with CDP browser transport fallback for JS-hydrated portals).
- **FR-3.3:** Deduplicate records across sources by DOI, arXiv ID, patent number, and normalized title similarity.

### FR-4: Document Ingestion & Hybrid Provenance Indexing
- **FR-4.1:** Fetch open-access PDFs and parse text with **PyMuPDF (`fitz`)**, preserving exact page numbers and bounding-box coordinates (`bbox`) for every extracted chunk.
- **FR-4.2:** Index chunks in a hybrid **BM25 + ChromaDB** dense/sparse retriever.

### FR-5: Zero-Trust Source, Link & IP Auditing
- **FR-5.1:** Perform asynchronous `aiohttp` HTTP status verification (`200 OK` or valid `302` redirect) for every candidate source URL and `https://doi.org/<DOI>`.
- **FR-5.2:** Cross-match title, authors, and publication year against **Crossref REST API**; purge any record failing either check.
- **FR-5.3:** Audit source license metadata (e.g., CC-BY, MIT, arXiv perpetual non-exclusive, vs. All Rights Reserved) and enforce n-gram syntactic overlap thresholds to guarantee fair-use compliance.

### FR-6: Closed-Loop Academic Synthesis & Peer-Review Critic
- **FR-6.1:** Generate structured academic survey sections: Abstract, PRISMA Methodology Summary, Conceptual Taxonomy, Comparative Literature & Patent Matrix, Synthesis, and Open Research Gaps.
- **FR-6.2:** Attach exact source chunk IDs and page provenance to every factual claim.
- **FR-6.3:** Route drafts through the **Peer-Review Critic Agent**; claims with entailment confidence $< 0.90$ or missing evidence trigger targeted secondary retrieval loops (up to a bounded max iteration count).

### FR-7: Interactive Research Cockpit (Web UI)
- **FR-7.1:** Stream live agent telemetry, executed queries, PRISMA counts, and link audit results via **SSE / WebSockets**.
- **FR-7.2:** Support human-in-the-loop pause/resume checkpoints after PRISMA query generation so users can refine date bounds or inclusion criteria.
- **FR-7.3:** Provide a **Split-Pane Provenance Inspector**: clicking any claim highlights the exact source passage inside the embedded PDF viewer alongside its live HTTP 200 & Crossref badge.
- **FR-7.4:** Render an interactive **D3.js Citation & Patent Network** showing papers, seminal authors, co-citations, and related patents.

### FR-8: Publication-Grade Export
- **FR-8.1:** Export validated reports as **IEEE** or **ACM** LaTeX project ZIP bundles containing `main.tex`, `references.bib`, and PRISMA tables/figures.
- **FR-8.2:** Provide optional compiled PDF export when a TeX engine (`tectonic` / `pdflatex`) is available, plus Markdown/JSON export.

---

## 5. Non-Functional Requirements

| Category | Requirement |
| :--- | :--- |
| **Performance & Async Execution** | Long-running research jobs (2–4 min) must execute asynchronously via Celery + Redis without HTTP request timeouts; initial SSE ack < 500 ms. |
| **Reliability & Checkpointing** | LangGraph state checkpoints persisted in Redis so transient external API failures retry with exponential backoff without losing progress. |
| **Determinism & Auditability** | Every included/excluded paper, HTTP status code, and Critic loop decision is logged with timestamps for full reproducibility. |
| **Security & Secrets** | All API keys stored via environment variables; no arbitrary code execution from downloaded PDFs; strict SSRF allowlist/validation on URL pinging where applicable. |
| **Rate-Limit Respect** | Per-provider token-bucket rate limiting and Redis response caching (24h TTL for bibliographic metadata). |
| **Usability & Accessibility** | Responsive Next.js 14 UI, clear verification status color + icon indicators, keyboard navigation in split-pane reader. |

---

## 6. Core User Stories & Acceptance Criteria

1. **US-1 (Impossible Premise):** *As a user*, if I submit `"Systematic review of how elephants hatch eggs"`, *then* the Plausibility Gatekeeper intercepts the query within 10 seconds, skips literature scraping, and displays a Refutation Report citing mammalian viviparity from Wikidata/NCBI Taxonomy.
2. **US-2 (PRISMA Review):** *As a researcher*, I can submit `"Federated Learning in Healthcare Imaging (2021–2026)"`, inspect/edit the generated Boolean queries and inclusion rules, and watch live SSE progress across Identification, Screening, Eligibility, and Inclusion.
3. **US-3 (Zero Dead Links):** *As a researcher*, every citation in my final report displays a verified badge (`HTTP 200` + `Crossref Matched`), and any hallucinated or broken reference is automatically excluded and logged in the audit trail.
4. **US-4 (Provenance Inspection):** *As a thesis advisor*, clicking a sentence in the generated review scrolls the right-hand PDF pane to the exact page and highlights the supporting paragraph.
5. **US-5 (Patent White-Space Discovery):** *As an R&D engineer*, I can view a unified table and D3.js graph contrasting academic papers with USPTO/Google Patents assignees to spot unpatented research opportunities.
6. **US-6 (LaTeX Export):** *As an author*, clicking **Export IEEE LaTeX** downloads a `.zip` containing a clean `main.tex` and verified `references.bib` that compiles without missing citation keys.

---

## 7. Scope Boundaries: MVP vs. Full v1.0

| Capability | Phase 1 (Vertical Slice MVP) | Full v1.0 Release |
| :--- | :--- | :--- |
| **Plausibility Gatekeeper** | LLM + Wikidata SPARQL check | Wikidata + NCBI Taxonomy + NLI contradiction scorer |
| **Retrieval Connectors** | arXiv + Crossref | arXiv + Semantic Scholar + PubMed + Crossref + USPTO PatentsView + Google Patents/CDP |
| **PRISMA Funnel** | Basic query decomposition + counts | Full 4-stage screening, deduplication, exclusion reasons, interactive checkpoint |
| **Ingestion & Retrieval** | Abstract + PDF text extraction | PyMuPDF bbox extraction + BM25 + ChromaDB hybrid index |
| **Verification & IP** | Async HTTP 200 + Crossref check | + License classifier (CC-BY/MIT) + n-gram copyright overlap audit |
| **Synthesis & Critic** | 1-pass Synthesizer + 1 Critic retry | NLI $\ge 0.90$ convergence loop with targeted re-retrieval |
| **Frontend Cockpit** | Query box, SSE timeline, Markdown report | + Split-pane PDF bbox highlighter, D3.js citation/patent graph, pause/resume |
| **Export** | Markdown + `.bib` | IEEE & ACM LaTeX ZIP + compiled PDF + PRISMA diagram |
