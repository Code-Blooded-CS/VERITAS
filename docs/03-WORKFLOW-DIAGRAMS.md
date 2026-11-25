# VERITAS Workflow & Architecture Diagrams

This document defines the canonical system flows before implementation. All agent transitions, data flows, and UI synchronization protocols adhere to these diagrams.

---

## 1. End-to-End Multi-Agent LangGraph StateGraph

```mermaid
flowchart TD
    Start([User Submits Research Query]) --> Plausibility[1. Plausibility & Sanity Agent<br/>Wikidata SPARQL + NCBI Taxonomy + NLI]

    Plausibility -->|Contradiction Detected| Refute([Emit Refutation Report<br/>with Ontological Evidence])
    Plausibility -->|Scientifically Plausible| PRISMA[2. PRISMA 2020 Protocol Agent<br/>Sub-questions, Boolean Matrix & Criteria]

    PRISMA -->|Optional Checkpoint| Pause{Human-in-the-Loop<br/>Review Criteria?}
    Pause -->|User Edits / Approves| FanOut{Concurrent Retrieval}
    Pause -->|Auto-Run| FanOut

    FanOut -->|Papers| Academic[3. Academic Search Agent<br/>arXiv, S2AG, PubMed, Crossref]
    FanOut -->|Patents| Patent[4. Patent Prior-Art Agent<br/>USPTO PatentsView, Google Patents/CDP]

    Academic & Patent --> Verifier[5. Zero-Trust Link & DOI Verifier<br/>Async HTTP 200 + Crossref Match]
    Verifier -->|Failed Check| Purge[(Purged Citations Log)]
    Verifier -->|Verified| DedupeScreen[PRISMA Deduplication &<br/>Abstract Screening]

    DedupeScreen --> Ingest[PDF Ingestion & Hybrid Index<br/>PyMuPDF BBox + BM25 + ChromaDB + NetworkX]
    Ingest --> IPAudit[6. IP & Copyright Auditor<br/>License Tagging & Fair-Use Rules]
    IPAudit --> Synthesizer[7. Academic Synthesizer Agent<br/>Draft Claims with Chunk Provenance]

    Synthesizer --> Critic{8. Peer-Review Critic<br/>NLI Entailment >= 0.90 &<br/>Rubric Pass?}
    Critic -->|Gaps / Score < 0.90 & loops < max| TargetedQuery[Generate Targeted Follow-Up Queries]
    TargetedQuery --> FanOut
    Critic -->|Approved / Max Loops Reached| OverlapCheck[Syntactic N-Gram Overlap Guard]

    OverlapCheck --> Finalize([Persist Final Report, Citation Graph &<br/>IEEE/ACM LaTeX Export Bundle])
```

---

## 2. Real-Time Platform Sequence Diagram (Cockpit ↔ Backend ↔ Worker)

```mermaid
sequenceDiagram
    actor Researcher
    participant Cockpit as Next.js 14 Cockpit
    participant API as FastAPI Gateway
    participant Redis as Redis (Broker + PubSub)
    participant Worker as Celery + LangGraph Worker
    participant ExtAPIs as Academic/Patent/Crossref APIs

    Researcher->>Cockpit: Enter topic & launch review
    Cockpit->>API: POST /api/v1/research
    API->>Redis: Store initial job record + enqueue task
    API-->>Cockpit: 202 Accepted { job_id }

    Cockpit->>API: GET /api/v1/research/{job_id}/stream (SSE)
    API->>Redis: SUBSCRIBE veritas:events:{job_id}

    Worker->>Redis: Dequeue job
    Worker->>ExtAPIs: Plausibility check (Wikidata / NCBI)
    Worker->>Redis: PUBLISH stage_started / plausibility_result
    Redis-->>API: Forward event
    API-->>Cockpit: SSE event: plausibility_passed

    Worker->>Redis: PUBLISH prisma_criteria_ready
    API-->>Cockpit: SSE event: prisma_updated
    Note over Cockpit,Worker: If human_checkpoint=true, Worker pauses until POST /resume

    par Concurrent Discovery
        Worker->>ExtAPIs: Query arXiv, S2AG, PubMed, Crossref
    and
        Worker->>ExtAPIs: Query USPTO PatentsView / Google Patents
    end

    Worker->>ExtAPIs: Async HTTP 200 & Crossref validation
    Worker->>Redis: PUBLISH source_verified / source_purged (live badges)
    API-->>Cockpit: SSE stream updates live source table

    Worker->>Worker: PyMuPDF BBox chunking + ChromaDB/BM25 + Synthesizer + Critic loop
    Worker->>Redis: Persist final state + PUBLISH completed
    API-->>Cockpit: SSE event: completed

    Researcher->>Cockpit: Click claim sentence
    Cockpit->>API: GET /api/v1/research/{job_id}/pdf/{source_id}
    API-->>Cockpit: Stream PDF + scroll/highlight exact bbox
```

---

## 3. PRISMA 2020 4-Stage Screening & Deduplication Funnel

```mermaid
flowchart TB
    subgraph Identification
        A1[arXiv Results: n_arxiv]
        A2[Semantic Scholar Results: n_s2]
        A3[PubMed Results: n_pubmed]
        A4[USPTO / Google Patents: n_pat]
        A1 & A2 & A3 & A4 --> TotalRaw[Total Records Identified: N_raw]
        TotalRaw --> LinkGate[Zero-Trust Link & DOI Verification Gate]
        LinkGate -->|Purged Dead/Fake| ExDead[Excluded - Failed HTTP 200 / Crossref]
        LinkGate --> Dedupe[Deduplication by DOI / Patent ID / Title Similarity]
        Dedupe -->|Duplicates Removed| ExDup[Excluded - Duplicates]
    end

    subgraph Screening
        Dedupe --> Screen[Title & Abstract Screening vs. Inclusion Criteria]
        Screen -->|Off-Topic / Out of Date| ExAbs[Excluded - Abstract Screening]
    end

    subgraph Eligibility
        Screen --> FullText[Full-Text PDF Retrieval & PyMuPDF Parsing]
        FullText -->|Paywalled / Unparseable| ExNoText[Excluded - Full Text Unavailable]
        FullText --> EligAssess[Empirical & Methodological Eligibility Check]
        EligAssess -->|Insufficient Rigor| ExMeth[Excluded - Methodological Criteria]
    end

    subgraph Included
        EligAssess --> FinalCorpus[Final Verified Corpus: N_included<br/>Indexed in ChromaDB + BM25 + NetworkX]
    end
```

---

## 4. Zero-Trust Link, DOI & IP Verification Pipeline

```mermaid
flowchart LR
    Candidate[Candidate Source Record] --> HasDOI{Has DOI / Patent ID?}
    HasDOI -->|Academic w/ DOI| Crossref[Query Crossref REST API<br/>/works/DOI]
    HasDOI -->|arXiv / Preprint / Patent| DirectURL[Check Official Registry<br/>arXiv / PubMed / USPTO]

    Crossref --> MetaMatch{Title Sim >= 0.85<br/>& Author Match?}
    MetaMatch -->|No| Purge[PURGE: Fabricated or Mismatched Metadata]
    MetaMatch -->|Yes| HTTPPing

    DirectURL --> HTTPPing[Async aiohttp GET/HEAD<br/>Resolve doi.org or Source URL]
    HTTPPing --> StatusOK{Final HTTP 200<br/>after <= 5 redirects?}
    StatusOK -->|No / Timeout / 404 / 403| Purge
    StatusOK -->|Yes| LicenseCheck[IP & License Classifier<br/>CC-BY / MIT / OA / Proprietary]

    LicenseCheck -->|Open Access| FullPDF[Allow Full-Text PDF Extraction]
    LicenseCheck -->|Proprietary| AbstractOnly[Restrict to Metadata + Abstract Only]
```

---

## 5. Document Ingestion & Claim Provenance Binding

```mermaid
flowchart LR
    PDF[Verified Open-Access PDF] --> PyMuPDF[PyMuPDF fitz Parser]
    PyMuPDF --> Chunks[Text Chunks + Metadata<br/>source_id, page_no, bbox x0 y0 x1 y1]
    Chunks --> Dense[Sentence-Transformers<br/>ChromaDB Vector Index]
    Chunks --> Sparse[Tokenized Corpus<br/>BM25 Index]
    Dense & Sparse --> Hybrid[Reciprocal Rank Fusion<br/>Top-K Evidence Chunks]
    Hybrid --> Synth[Synthesizer Drafts Claim<br/>attaching chunk_id + page + bbox]
    Synth --> Critic[Critic Verifies NLI >= 0.90]
    Critic --> UI[Cockpit Split-Pane:<br/>Click Claim -> Highlight PDF BBox]
```
