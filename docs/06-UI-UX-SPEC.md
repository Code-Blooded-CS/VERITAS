# UI/UX Specification: VERITAS Research Cockpit

**Framework:** Next.js 14 (App Router), React 18, TypeScript, TailwindCSS, Radix UI, Lucide Icons, D3.js  
**Related Docs:** [PRD](./01-PRD.md) | [API & Data Contracts](./04-API-AND-DATA-CONTRACTS.md)

---

## 1. Information Architecture & Views

The Research Cockpit uses a clean academic-grade workspace layout with a collapsible left navigation rail and four primary views:

1. **New Research / Query Studio (`/`)**
   - Topic / hypothesis prompt box with example chips (including 1 adversarial impossible prompt demo).
   - Configuration drawer: year range (`year_from`–`year_to`), source toggles (`arXiv`, `Semantic Scholar`, `PubMed`, `USPTO`, `Google Patents`), `Pause for PRISMA Criteria Review` toggle, `Max Papers/Patents`.
2. **Live Run Cockpit (`/research/[jobId]`)**
   - **Top Status Stepper:** `Plausibility` → `PRISMA Protocol` → `Discovery` → `Link & IP Audit` → `Synthesis & Critic` → `Ready`.
   - **Tab 1 — Live Telemetry & PRISMA Funnel:**
     - Real-time SSE event log (queries, API counts, Critic NLI loops).
     - Interactive PRISMA Checkpoint Drawer when `awaiting_approval=true` (edit Boolean queries & inclusion/exclusion chips, then click **Approve & Retrieve**).
     - Live 4-stage PRISMA funnel counter cards (`Identified` → `Deduplicated` → `Screened` → `Eligible` → `Included`) + Purged Citations log.
   - **Tab 2 — Split-Pane Provenance Reader (Primary View on Completion):**
     - **Left Pane (60%):** Synthesized review (Taxonomy, Comparative Literature & Patent Matrix, Thematic Sections, Research Gaps). Each claim sentence has inline citation pills `[1]`, `[P2]` and an NLI entailment indicator (`0.94`).
     - **Right Pane (40%):** Embedded PDF Viewer (`PDF.js` / `react-pdf`) + Verification Card (`HTTP 200 OK`, `Crossref Matched`, License Badge `CC-BY`). Clicking any claim pill scrolls the PDF to `page_number` and renders a translucent highlight box over `bbox=[x0, y0, x1, y1]`.
   - **Tab 3 — Citation & Patent Network (D3.js):**
     - Force-directed graph distinguishing Academic Papers (blue circles), Patents (amber hexagons/squares), and Seminal Authors/Assignees.
     - Node click opens metadata, DOI/patent link, and shared citation paths; filters by year, source type, and license.
   - **Tab 4 — Audit & Verification Ledger:**
     - Filterable table of all **Verified** and **Purged** sources with exact HTTP status codes, Crossref similarity scores, license tags, and exclusion reasons.
3. **Refutation View (Inline State in `/research/[jobId]`)**
   - Triggered automatically on `plausibility_result.verdict === "refuted"`.
   - Displays a clear scientific warning banner, extracted `(subject, predicate, object)` contradiction, Wikidata/NCBI Taxonomy references, and why downstream synthesis was halted.
4. **Export Modal**
   - Format picker: **IEEE LaTeX (.zip)**, **ACM LaTeX (.zip)**, **BibTeX (.bib)**, **Markdown (.md)**, **Compiled PDF (.pdf)**.
   - Shows checklist confirmation: *"100% of included references passed live HTTP 200 & Crossref validation."*

---

## 2. Verification & Status Badge System

| State | Visual Badge | Tooltip / Details |
| :--- | :--- | :--- |
| **Verified Source** | Green pill + ShieldCheck | `HTTP 200 OK · Crossref Matched (0.97) · CC-BY` |
| **Patent Prior Art** | Amber pill + FileBadge | `USPTO Verified · CPC G06N · Assignee: ...` |
| **Abstract-Only (Proprietary)** | Slate pill + Lock | `HTTP 200 OK · Paywalled — Abstract & Metadata Only` |
| **Purged Source** | Red strikethrough + Ban | `Purged: HTTP 404 / Crossref Author Mismatch` |
| **Claim Entailment** | Emerald (`>= 0.90`) / Amber (`0.80–0.89`) | `Critic NLI Score: 0.93 against Chunk #14 (p. 4)` |

---

## 3. Split-Pane Provenance Interaction Spec

```text
+-----------------------------------------------------------------------+
| VERITAS  |  Query: Federated Learning in Healthcare   [Export v]      |
| [Telemetry & PRISMA]  [Split-Pane Reader*]  [D3 Graph]  [Audit Log]   |
+------------------------------------+----------------------------------+
| SYNTHESIZED REVIEW                 | PDF PROVENANCE INSPECTOR         |
|                                    | Source: FedAvg-Med (2024)        |
| 3.2 Privacy-Preserving Aggregation | [HTTP 200] [Crossref OK] [CC-BY] |
| Differential privacy adds calibrated |----------------------------------|
| Gaussian noise prior to server     | Page 4 of 12                     |
| aggregation, bounding epsilon [1].  | ...section 3.2...                |
|  ^ clicked! (NLI: 0.94)            | +------------------------------+ |
|                                    | | HIGHLIGHTED BBOX PASSAGE     | |
| Patent US11488012B2 discloses a    | | "We inject Gaussian noise... | |
| hardware enclave for weight... [P1]| |  achieving (eps, delta)-DP"  | |
|                                    | +------------------------------+ |
+------------------------------------+----------------------------------+
```

- Clicking `[1]` sets `activeCitation = { sourceId, pageNumber, bbox }`.
- The right pane fetches `/api/v1/research/{jobId}/pdf/{sourceId}` (cached in browser memory), jumps to `pageNumber`, and draws an overlay `<div />` scaled from PDF points to viewport pixels using `page.getViewport({ scale })`.
- If the selected source is `Proprietary-Abstract-Only` (no open-access PDF), the right pane cleanly displays the verified **Abstract + Crossref Metadata Card** with a direct publisher DOI link instead of an empty PDF frame.
