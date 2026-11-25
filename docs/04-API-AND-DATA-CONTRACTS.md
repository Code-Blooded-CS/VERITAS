# API Specification & Data Contracts: VERITAS

**Version:** 1.0  
**Base URL:** `/api/v1`  
**Transport:** REST (JSON) + Server-Sent Events (SSE)  
**Related Docs:** [TRD](./02-TRD.md) | [UI/UX Spec](./06-UI-UX-SPEC.md)

---

## 1. REST Endpoints

### 1.1 Launch a Research Job
- **Endpoint:** `POST /api/v1/research`
- **Request Body:**
```json
{
  "query": "Federated learning for medical image segmentation",
  "year_from": 2021,
  "year_to": 2026,
  "include_patents": true,
  "sources": ["arxiv", "semantic_scholar", "pubmed", "uspto"],
  "human_checkpoint": true,
  "max_papers": 30,
  "max_patents": 15,
  "max_critic_loops": 2
}
```
- **Response (`202 Accepted`):**
```json
{
  "job_id": "job_01J9X8K2M3N4P5Q6",
  "status": "queued",
  "stream_url": "/api/v1/research/job_01J9X8K2M3N4P5Q6/stream"
}
```

### 1.2 Get Full Research Job State & Report
- **Endpoint:** `GET /api/v1/research/{job_id}`
- **Response (`200 OK`):** Returns full `ResearchJobResponse` including status, plausibility verdict, PRISMA funnel counts, verified/purged sources, synthesized sections with provenance claims, and critic scores.

### 1.3 Resume Paused PRISMA Checkpoint (Human-in-the-Loop)
- **Endpoint:** `POST /api/v1/research/{job_id}/resume`
- **Request Body:**
```json
{
  "approved": true,
  "boolean_queries": [
    "(\"federated learning\" OR \"distributed learning\") AND (\"medical imaging\" OR \"segmentation\")"
  ],
  "inclusion_criteria": [
    "Published between 2021 and 2026",
    "Reports quantitative segmentation metrics (Dice / IoU)"
  ],
  "exclusion_criteria": [
    "Non-English publications",
    "Position papers without empirical evaluation"
  ]
}
```
- **Response (`200 OK`):** `{ "job_id": "...", "status": "running" }`

### 1.4 Get Citation & Patent Network Graph
- **Endpoint:** `GET /api/v1/research/{job_id}/graph`
- **Response (`200 OK`):** `{ "nodes": [...], "edges": [...] }` consumed directly by the D3.js force-directed graph.

### 1.5 Stream Verified PDF for Split-Pane Viewer
- **Endpoint:** `GET /api/v1/research/{job_id}/pdf/{source_id}`
- **Response (`200 OK`):** `application/pdf` binary stream with `Content-Disposition: inline`.

### 1.6 Export Publication Bundle
- **Endpoint:** `GET /api/v1/research/{job_id}/export/{format}`
- **Path Param `format`:** `ieee_latex` | `acm_latex` | `bibtex` | `markdown` | `pdf`
- **Response (`200 OK`):** `application/zip` (for LaTeX bundles), `application/x-bibtex`, `text/markdown`, or `application/pdf`.

### 1.7 Health & Provider Diagnostics
- **Endpoint:** `GET /api/v1/health`
- **Response (`200 OK`):** Redis connectivity, Celery worker count, ChromaDB status, and configured LLM/API providers.

---

## 2. Real-Time SSE Telemetry Stream

- **Endpoint:** `GET /api/v1/research/{job_id}/stream`
- **Media Type:** `text/event-stream`

Every SSE frame carries a standard envelope:
```json
{
  "event_id": "evt_0042",
  "job_id": "job_01J9X8K2M3N4P5Q6",
  "timestamp": "2026-11-25T10:42:15Z",
  "type": "source_verified",
  "stage": "verification",
  "payload": {}
}
```

### Event Types
| Event `type` | Trigger | Key `payload` Fields |
| :--- | :--- | :--- |
| `stage_started` | Any LangGraph node entry | `stage`, `agent_name`, `message` |
| `plausibility_result` | Agent 1 completes | `verdict` (`plausible` / `refuted`), `confidence`, `evidence[]`, `refutation_summary` |
| `prisma_ready` | Agent 2 completes (or pauses) | `sub_questions[]`, `boolean_queries[]`, `inclusion_criteria[]`, `awaiting_approval` |
| `query_executed` | Search connector returns | `provider`, `query`, `raw_count` |
| `source_verified` | Agent 5 approves source | `source_id`, `title`, `http_status`, `crossref_matched`, `license` |
| `source_purged` | Agent 5 rejects source | `title`, `url`, `reason` (`http_404`, `doi_mismatch`, `timeout`) |
| `prisma_updated` | Funnel stage count changes | `identified`, `deduplicated`, `screened`, `eligible`, `included` |
| `claim_drafted` | Agent 7 emits section/claims | `section_id`, `title`, `claim_count` |
| `critic_feedback` | Agent 8 evaluates draft | `loop_index`, `min_entailment`, `avg_entailment`, `action` (`retry` / `approve`), `gap_queries[]` |
| `completed` | Workflow finishes | `summary_stats`, `export_formats[]` |
| `error` | Unrecoverable failure | `code`, `message`, `recoverable` |

---

## 3. Canonical Pydantic / JSON Data Models

### 3.1 `SourceRecord`
```python
class SourceRecord(BaseModel):
    source_id: str
    source_type: Literal["academic", "patent"]
    provider: Literal["arxiv", "semantic_scholar", "pubmed", "crossref", "uspto", "google_patents"]
    title: str
    authors_or_assignees: list[str]
    year: int | None
    venue_or_office: str | None
    doi: str | None = None
    patent_number: str | None = None
    url: HttpUrl
    pdf_url: HttpUrl | None = None
    abstract: str
    # Verification & IP fields
    http_status: int | None = None
    crossref_matched: bool = False
    title_similarity: float | None = None
    verification_status: Literal["pending", "verified", "purged"] = "pending"
    purge_reason: str | None = None
    license_tag: Literal["CC-BY", "CC0", "MIT", "OA-Restricted", "Proprietary-Abstract-Only", "Unknown"] = "Unknown"
    # PRISMA tracking
    prisma_stage: Literal["identified", "deduplicated", "screened", "eligible", "included", "excluded"]
    exclusion_reason: str | None = None
```

### 3.2 `PDFChunk` (Provenance Unit)
```python
class PDFChunk(BaseModel):
    chunk_id: str
    source_id: str
    page_number: int          # 1-based page index
    bbox: tuple[float, float, float, float]  # [x0, y0, x1, y1] in PDF points
    text: str
    section_heading: str | None = None
```

### 3.3 `VerifiedClaim`
```python
class CitationProvenance(BaseModel):
    source_id: str
    chunk_id: str
    page_number: int
    bbox: tuple[float, float, float, float]
    quote_snippet: str
    entailment_score: float

class VerifiedClaim(BaseModel):
    claim_id: str
    section_id: str
    text: str
    citations: list[CitationProvenance]
    min_entailment: float
    ngram_overlap_ratio: float
```

### 3.4 `PRISMAFunnelMetrics`
```python
class PRISMAFunnelMetrics(BaseModel):
    identified_by_source: dict[str, int]
    total_identified: int
    purged_unverified: int
    duplicates_removed: int
    records_screened: int
    excluded_screening: int
    full_text_assessed: int
    excluded_eligibility: dict[str, int]
    total_included: int
```

### 3.5 `ResearchJobResponse` & `GraphResponse` (API Payloads)
```python
class ResearchJobResponse(BaseModel):
    job_id: str
    query: str
    status: Literal["queued", "running", "awaiting_approval", "refuted", "completed", "failed"]
    domain_tag: str | None = None
    plausibility_verdict: Literal["plausible", "refuted"] | None = None
    refutation_summary: str | None = None
    prisma_metrics: PRISMAFunnelMetrics | None = None
    sources: list[SourceRecord] = []
    claims: list[VerifiedClaim] = []
    report_markdown: str | None = None
    critic_loops_used: int = 0

class GraphNode(BaseModel):
    id: str
    label: str
    node_type: Literal["academic", "patent", "author"]
    year: int | None = None
    verified: bool = True

class GraphEdge(BaseModel):
    source: str
    target: str
    relation: Literal["cites", "authored_by", "prior_art_overlap"]

class GraphResponse(BaseModel):
    nodes: list[GraphNode]
    edges: list[GraphEdge]
```

### 3.6 LangGraph `ResearchState` (Internal State Machine)
```python
from typing import Annotated, TypedDict
from operator import add

class ResearchState(TypedDict):
    job_id: str
    query: str
    config: dict
    domain_tag: str
    plausibility_verdict: str
    refutation_summary: str | None
    sub_questions: list[str]
    boolean_queries: list[str]
    inclusion_criteria: list[str]
    exclusion_criteria: list[str]
    raw_sources: Annotated[list[dict], add]
    verified_sources: list[dict]
    purged_sources: Annotated[list[dict], add]
    prisma_metrics: dict
    draft_claims: list[dict]
    report_markdown: str
    critic_feedback: dict
    loop_count: int
```

---

## 4. Minimal Relational Database Schema (SQLite Dev / Postgres Prod)

| Table | Key Columns | Purpose |
| :--- | :--- | :--- |
| **`research_jobs`** | `id (PK)`, `query`, `config_json`, `status`, `plausibility_verdict`, `prisma_json`, `report_md`, `created_at`, `updated_at` | Stores job lifecycle, PRISMA counts, and final synthesis |
| **`sources`** | `id (PK)`, `job_id (FK)`, `source_type`, `provider`, `doi_or_patent_id`, `title`, `url`, `http_status`, `crossref_matched`, `verification_status`, `license_tag`, `prisma_stage`, `pdf_local_path` | Full audit ledger of verified and purged references |
| **`claims`** | `id (PK)`, `job_id (FK)`, `section_id`, `claim_text`, `citations_json`, `min_entailment`, `ngram_overlap` | Stores provenance-linked claims (`page`, `bbox`, `chunk_id`) |
| **`audit_events`** | `id (PK)`, `job_id (FK)`, `stage`, `event_type`, `payload_json`, `created_at` | Immutable timeline for debugging & telemetry replay |
