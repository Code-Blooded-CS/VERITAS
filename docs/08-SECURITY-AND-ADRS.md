# Security Checklist & Architecture Decision Records (ADRs)

**Related Docs:** [TRD](./02-TRD.md) | [API & Data Contracts](./04-API-AND-DATA-CONTRACTS.md)

---

## 1. Practical Security & Guardrail Checklist

### 1.1 SSRF Protection (Agent 5: Link Verifier & PDF Downloader)
Because Agent 5 pings URLs returned by external indices:
- Allow **only** `http://` and `https://` schemes.
- Resolve hostname DNS before connecting and reject private, loopback, link-local, and reserved IP ranges (`127.0.0.0/8`, `10.0.0.0/8`, `169.254.0.0/16`, `172.16.0.0/12`, `192.168.0.0/16`, `::1`).
- Enforce timeouts (`10s` connect/read), max `5` redirects (re-checking IP on each redirect), and a max PDF download size (`25 MB`).

### 1.2 Indirect Prompt Injection Hygiene
External abstracts and PDFs are **untrusted input**:
- Wrap all retrieved chunks in explicit XML/data delimiters (e.g., `<source_chunk id="...">...</source_chunk>`) and instruct Synthesizer/Critic system prompts to treat text inside as inert evidence only.
- Never let text extracted from a PDF trigger tool calls or override PRISMA inclusion rules.

### 1.3 Safe LaTeX Compilation
- Escape LaTeX special characters (`\`, `{`, `}`, `$`, `&`, `#`, `_`, `%`, `~`, `^`) on all titles, authors, and synthesized prose before rendering Jinja2 templates.
- Run `tectonic` or `pdflatex` in a sandboxed temp directory with `--no-shell-escape` (never allow `\write18`) and a `30s` process timeout.

### 1.4 Secrets & Configuration
- Keep all API keys (`GEMINI_API_KEY`, `OPENAI_API_KEY`, `S2_API_KEY`, `USPTO_API_KEY`, `CROSSREF_MAILTO`) in `.env` (already in `.gitignore`) and document placeholders in `.env.example`.

---

## 2. Architecture Decision Records (ADRs)

| ID | Decision | Rationale | Alternative Not Chosen |
| :--- | :--- | :--- | :--- |
| **ADR-01** | **SSE over WebSockets** for primary telemetry | Unidirectional server-to-client progress fits HTTP/SSE cleanly, auto-reconnects in browsers, and needs zero WS handshake state. Human-in-the-loop approvals use simple `POST /resume`. | Bidirectional WebSockets (more boilerplate for little gain) |
| **ADR-02** | **SQLite (dev) → PostgreSQL (prod)** via SQLAlchemy | Zero-setup local development for 5 team members while staying 100% schema-compatible with Postgres in Docker/production. | Raw JSON files or MongoDB |
| **ADR-03** | **LangGraph + Redis Checkpointing + Celery** | Research reviews run 2–4 minutes; Celery prevents HTTP timeouts while LangGraph + Redis preserves state across Critic retries and pause/resume. | Blocking FastAPI async tasks |
| **ADR-04** | **8 Core Agents + Domain Capability Router** | Aligns directly with `synopsis.tex` (`Table 1`); optional Tavily whitepapers act as a toggleable connector under Discovery instead of complicating the graph. | 9+ separate agent nodes |
| **ADR-05** | **Hybrid BM25 + ChromaDB + PyMuPDF `bbox`** | Combining sparse (BM25) and dense (ChromaDB) retrieval catches exact acronyms/patent IDs alongside semantic concepts, while `bbox` enables click-to-highlight in the Cockpit. | Dense vector search alone |
