# UML & Data Flow Diagrams (DFD): VERITAS

> **Purpose:** Academic and software-engineering design diagrams prepared for project guide reviews, SRS/design checkpoints, and dissertation documentation.  
> **Companion Workflow & Sequence Diagrams:** See [03-WORKFLOW-DIAGRAMS.md](./03-WORKFLOW-DIAGRAMS.md) for the End-to-End LangGraph Flowchart, Runtime Sequence Diagram, PRISMA Funnel, Link/IP Verification Gate, and Ingestion Pipeline.

---

## 1. Data Flow Diagram — Level 0 (Context Diagram)

Shows VERITAS as a single system interacting with the Researcher and external scientific/IP registries.

```mermaid
flowchart LR
    Researcher["Researcher / User"]
    Ontologies["Wikidata & NCBI Taxonomy"]
    ScholarAPIs["Academic APIs (arXiv, S2AG, PubMed)"]
    PatentAPIs["Patent APIs (USPTO, Google Patents)"]
    Registries["Crossref & Publisher Endpoints"]

    VERITAS(("0.0 VERITAS Multi-Agent System"))

    Researcher -->|"Research Query & PRISMA Config"| VERITAS
    VERITAS -->|"Refutation / Live Telemetry / PDF Provenance / LaTeX ZIP"| Researcher

    VERITAS -->|"Entity & Taxonomy Queries"| Ontologies
    Ontologies -->|"Ontological Facts"| VERITAS

    VERITAS -->|"Boolean Literature Queries"| ScholarAPIs
    ScholarAPIs -->|"Papers, Abstracts & Open PDFs"| VERITAS

    VERITAS -->|"Prior-Art Queries"| PatentAPIs
    PatentAPIs -->|"Patent Claims & Assignees"| VERITAS

    VERITAS -->|"DOI Metadata & HTTP 200 Checks"| Registries
    Registries -->|"Canonical Metadata, Licenses & HTTP Status"| VERITAS
```

---

## 2. Data Flow Diagram — Level 1

Decomposes `0.0 VERITAS` into six core processes and four persistent/cached data stores (`D1`–`D4`).

```mermaid
flowchart TD
    User["Researcher"]
    ExtOnt["Wikidata / NCBI"]
    ExtSearch["arXiv / S2AG / PubMed / USPTO"]
    ExtVerify["Crossref / doi.org / Publishers"]

    P1(("1.0 Plausibility Gatekeeper"))
    P2(("2.0 PRISMA Protocol & Screening"))
    P3(("3.0 Dual Academic & Patent Retrieval"))
    P4(("4.0 Zero-Trust Link & IP Verifier"))
    P5(("5.0 PDF Parsing & Hybrid Indexing"))
    P6(("6.0 Synthesis, Critic Loop & Export"))

    D1[("D1: Jobs & PRISMA Logs")]
    D2[("D2: Sources & Audit Ledger")]
    D3[("D3: PDFs, ChromaDB & BM25")]
    D4[("D4: Redis Cache & State")]

    User -->|"Topic & Filters"| P1
    P1 <-->|"Taxonomy Check"| ExtOnt
    P1 -->|"Refutation Report"| User
    P1 -->|"Validated Topic"| P2

    P2 <-->|"Checkpoint / Criteria"| User
    P2 -->|"PRISMA Counts"| D1
    P2 -->|"Boolean Queries"| P3

    P3 <-->|"Query / Metadata"| ExtSearch
    P3 <-->|"24h Metadata Cache"| D4
    P3 -->|"Candidate Records"| P4

    P4 <-->|"HTTP 200 & DOI Check"| ExtVerify
    P4 -->|"Verified & Purged Logs"| D2
    P4 -->|"Verified Open-Access Sources"| P5

    P5 -->|"PyMuPDF Chunks + BBox"| D3
    D3 -->|"Hybrid Evidence Chunks"| P6
    D2 -->|"Verified Citations & Licenses"| P6
    P6 -->|"Critic Re-Query Request"| P3
    P6 -->|"Claims & Report"| D1
    P6 -->|"Telemetry, Provenance Reader & IEEE/ACM Export"| User
```

---

## 3. State Machine Diagram (`stateDiagram-v2`)

Models the LangGraph job lifecycle, human-in-the-loop PRISMA checkpoint, and bounded Critic reflection loop.

```mermaid
stateDiagram-v2
    [*] --> Queued
    Queued --> PlausibilityCheck : Worker picks job
    PlausibilityCheck --> Refuted : Contradiction detected
    PlausibilityCheck --> PRISMACheckpoint : Plausible
    Refuted --> [*]

    PRISMACheckpoint --> Retrieving : Auto or User Resumes
    Retrieving --> Verifying : Raw papers & patents fetched
    Verifying --> ScreeningAndIndexing : HTTP 200 & Crossref passed
    ScreeningAndIndexing --> Synthesizing : Chunks indexed (Chroma + BM25)
    Synthesizing --> Critiquing : Draft with chunk provenance

    Critiquing --> Retrieving : NLI < 0.90 & loop_count < max_loops
    Critiquing --> Completed : Approved or max_loops reached
    Completed --> [*]
```

---

## 4. Class Diagram (`classDiagram`)

Represents the core domain models and multi-agent orchestrator defined in [04-API-AND-DATA-CONTRACTS.md](./04-API-AND-DATA-CONTRACTS.md).

```mermaid
classDiagram
    class ResearchJob {
        +str job_id
        +str query
        +str status
        +str domain_tag
        +str plausibility_verdict
        +int critic_loops_used
    }

    class PRISMAFunnelMetrics {
        +int total_identified
        +int purged_unverified
        +int duplicates_removed
        +int records_screened
        +int full_text_assessed
        +int total_included
    }

    class SourceRecord {
        +str source_id
        +str source_type
        +str provider
        +str title
        +str doi
        +str patent_number
        +int http_status
        +bool crossref_matched
        +str verification_status
        +str license_tag
        +str prisma_stage
    }

    class PDFChunk {
        +str chunk_id
        +str source_id
        +int page_number
        +tuple bbox
        +str text
    }

    class VerifiedClaim {
        +str claim_id
        +str section_id
        +str text
        +float min_entailment
        +float ngram_overlap_ratio
    }

    class LangGraphOrchestrator {
        +run_plausibility()
        +run_prisma_protocol()
        +run_dual_discovery()
        +run_zero_trust_verifier()
        +run_ip_auditor()
        +run_synthesizer()
        +run_critic_loop()
    }

    ResearchJob "1" *-- "1" PRISMAFunnelMetrics
    ResearchJob "1" *-- "0..*" SourceRecord
    ResearchJob "1" *-- "0..*" VerifiedClaim
    SourceRecord "1" *-- "0..*" PDFChunk
    VerifiedClaim "1" --> "1..*" PDFChunk : cites provenance
    LangGraphOrchestrator ..> ResearchJob : updates state
```

---

## 5. Entity-Relationship Diagram (`erDiagram`)

Maps the relational database schema (`research_jobs`, `sources`, `claims`, `audit_events`) used by SQLite (dev) and PostgreSQL (production).

```mermaid
erDiagram
    RESEARCH_JOBS ||--o{ SOURCES : collects
    RESEARCH_JOBS ||--o{ CLAIMS : generates
    RESEARCH_JOBS ||--o{ AUDIT_EVENTS : logs
    SOURCES ||--o{ CLAIMS : supports

    RESEARCH_JOBS {
        string id PK
        string query
        json config_json
        string status
        string plausibility_verdict
        json prisma_json
        text report_md
        datetime created_at
    }

    SOURCES {
        string id PK
        string job_id FK
        string source_type
        string provider
        string doi_or_patent_id
        string title
        string url
        int http_status
        boolean crossref_matched
        string verification_status
        string license_tag
        string prisma_stage
        string pdf_local_path
    }

    CLAIMS {
        string id PK
        string job_id FK
        string section_id
        text claim_text
        json citations_json
        float min_entailment
        float ngram_overlap
    }

    AUDIT_EVENTS {
        string id PK
        string job_id FK
        string stage
        string event_type
        json payload_json
        datetime created_at
    }
```
