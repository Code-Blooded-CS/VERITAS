# VERITAS: Verified Evidence-Based Research, IP & Taxonomy Autonomous System

> **A Closed-Loop Multi-Agent Framework for Scientific Literature Discovery, Patent Landscaping, and Automated Academic Synthesis**

---

## 📌 Project Overview
**VERITAS** is an autonomous, multi-agent AI web platform engineered for Computer Science and academic research. Unlike standard AI tools that hallucinate fake citations, provide dead links, and uncritically accept scientifically impossible topics, VERITAS guarantees verifiable research integrity through:
1. **Ontological Plausibility Filtering:** Intercepts and refutes biologically/physically impossible queries (e.g., *"an elephant hatching an egg"*) using Wikidata and NCBI Taxonomy before searching.
2. **PRISMA 2020 Systematic Methodology:** Applies official academic review standards (identification, deduplication, screening funnel, eligibility).
3. **Dual Literature & Patent Landscaping:** Queries academic papers (arXiv, Semantic Scholar, PubMed) and registered patents (Google Patents, USPTO) concurrently.
4. **Live Cryptographic Source Verification:** Real-time HTTP 200 status checks and Crossref DOI metadata matching to ensure zero dead links or fake authors.
5. **Interactive Web Research Cockpit:** Next.js dual-pane claim inspector with highlighted PDF source provenance, D3.js citation graph, and 1-click IEEE/ACM LaTeX export.

---

## 👥 Project Team & Faculty Details

### **Department of Computer Science and Engineering**
* **Team Name:** `Code Blooded`
* **Academic Session:** 2026--2027

| # | Student Name | Role | University Roll No |
| :---: | :--- | :--- | :---: |
| 1 | **Pritam Jana** | Team Leader & Agent Orchestration Lead | `25500123099` |
| 2 | **Sachin Kumar Singh** | Full-Stack Web & Interface Lead | `25500123122` |
| 3 | **Rohit Kumar Ojha** | Academic & Patent Data Ingestion Lead | `25500123118` |
| 4 | **Sagar Jha** | Plausibility & PRISMA Review Lead | `25500123124` |
| 5 | **Bhaskar Raj** | Verification, IP & Export Pipeline Lead | `25500123061` |

* **Project Supervisor / Guide:** **Prof. (Dr.) Amitava Halder** (Dept. of Computer Science & Engineering)
* **Head of Department (HoD):** **Prof. (Dr.) Amrut Ranjan Jena** (Dept. of Computer Science & Engineering)

---

## 🛠️ Work Division & Module Ownership

To ensure smooth collaboration, the project is divided into 5 independent yet interconnected modules:

### 1. Pritam Jana — Agent Orchestration & State Machine
* **Module:** `backend/agents/orchestrator`
* **Branch:** `feature/agent-orchestration`
* **Tasks:**
  - Build the LangGraph cyclical StateGraph orchestrator.
  - Implement the **Academic Synthesizer Agent** (generating survey draft sections).
  - Implement the **Peer-Review Critic Agent** (evaluating claims with entailment threshold $\tau \ge 0.90$).
  - Coordinate pull request reviews and state machine transitions.

### 2. Sachin Kumar Singh — Web Research Cockpit & Telemetry
* **Module:** `frontend/`
* **Branch:** `feature/web-frontend`
* **Tasks:**
  - Build the Next.js 14 + TailwindCSS research cockpit.
  - Implement WebSocket / Server-Sent Events (SSE) streaming live agent execution steps.
  - Develop the synchronized split-pane claim inspector (highlighting source PDF passages).
  - Create the interactive D3.js force-directed citation and patent network graph.

### 3. Rohit Kumar Ojha — Academic & Patent Data Ingestion
* **Module:** `backend/ingestion/`
* **Branch:** `feature/academic-patent-connectors`
* **Tasks:**
  - Build asynchronous API connectors for Semantic Scholar (S2AG), arXiv, PubMed, and Crossref.
  - Build patent exploration engine for Google Patents Public Data and USPTO PatentsView API.
  - Implement PDF downloading, caching with Redis, and PyMuPDF text parsing.

### 4. Sagar Jha — Plausibility Gatekeeper & PRISMA Engine
* **Module:** `backend/agents/methodology/`
* **Branch:** `feature/prisma-plausibility-engine`
* **Tasks:**
  - Build the **Ontological Plausibility Gatekeeper** using Wikidata SPARQL and NCBI Taxonomy APIs.
  - Implement the **PRISMA 2020 Protocol Engine**: query decomposition, Boolean query generation, and inclusion/exclusion criteria.
  - Generate the 4-stage PRISMA screening funnel audit metrics ($527 \rightarrow 400 \rightarrow 120 \rightarrow 34$).

### 5. Bhaskar Raj — Live Link Verifier, IP Audit & LaTeX Exporter
* **Module:** `backend/verification/` & `backend/export/`
* **Branch:** `feature/verification-export-pipeline`
* **Tasks:**
  - Implement asynchronous `aiohttp` pinger verifying `HTTP 200 OK` for every URL.
  - Cross-match paper titles, authors, and years against the official Crossref REST API.
  - Implement the **IP & Copyright Auditor** (checking CC-BY vs. proprietary licenses and n-gram overlap).
  - Build the 1-click IEEE/ACM LaTeX ZIP and authenticated `.bib` compilation gateway.

---

## 🌿 Git Collaboration & Workflow Guide

### 1. Clone the Repository
```bash
git clone https://github.com/sachinn-alt/VERITAS.git
cd VERITAS
```

### 2. Create Your Feature Branch
Never push directly to `main`. Always create a branch for your module:
```bash
# Example for Sachin
git checkout -b feature/web-frontend

# Example for Rohit
git checkout -b feature/academic-patent-connectors

# Example for Sagar
git checkout -b feature/prisma-plausibility-engine

# Example for Bhaskar
git checkout -b feature/verification-export-pipeline

# Example for Pritam
git checkout -b feature/agent-orchestration
```

### 3. Commit Your Work Regularly
```bash
git add .
git commit -m "feat(module-name): describe what you built"
```

### 4. Push to GitHub
```bash
git push -u origin feature/your-branch-name
```

### 5. Open a Pull Request (PR)
1. Go to the repository on GitHub: `https://github.com/sachinn-alt/VERITAS`.
2. Click **Compare & pull request**.
3. Add a clear description of your changes.
4. Request review from team members (Team Leader **Pritam Jana**).
5. Once approved, merge into `main`!

---

## 💻 Tech Stack
* **Frontend:** Next.js 14, React, TailwindCSS, D3.js, Lucide Icons
* **Backend:** FastAPI (Python 3.10+), WebSockets / SSE, Uvicorn
* **Task Queue & Cache:** Celery, Redis
* **Agent Engine:** LangGraph (Cyclical StateGraph DAG)
* **LLMs:** Gemini 1.5 Pro / GPT-4o (Reasoning), Ollama / Llama 3 (Extraction)
* **Vector Store:** ChromaDB / BM25 Hybrid Retriever
* **APIs:** Semantic Scholar, arXiv, PubMed, Crossref, Google Patents, USPTO
* **Document Engine:** PyMuPDF, LaTeX / BibTeX Engine

---

## 📄 Documentation & Synopsis
* **Academic Synopsis (LaTeX Source):** [`synopsis.tex`](synopsis.tex)
* **Comprehensive Project Markdown:** [`synopsis_multi_agent_research_assistant.md`](synopsis_multi_agent_research_assistant.md)

---
*Developed by Team Code Blooded | Department of Computer Science & Engineering*
