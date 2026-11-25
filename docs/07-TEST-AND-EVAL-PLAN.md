# Testing, Verification & Evaluation Plan: VERITAS

**Related Docs:** [PRD](./01-PRD.md) | [TRD](./02-TRD.md) | [Execution Plan](./05-EXECUTION-PLAN.md)

---

## 1. Testing Pyramid & CI Gates

| Level | Scope | Tools | Run Cadence |
| :--- | :--- | :--- | :--- |
| **Unit Tests** | Pure functions: Boolean query builder, deduplicator, Crossref title similarity, n-gram overlap, BibTeX escaping, Pydantic schemas | `pytest`, `vitest` | Every commit / PR |
| **Agent & Graph Integration** | LangGraph `StateGraph` transitions with mocked LLM + mocked arXiv/S2AG/Crossref/Wikidata HTTP fixtures (`respx` / `aioresponses`) | `pytest-asyncio` | Every PR |
| **API & SSE Contract Tests** | FastAPI endpoints, 202 enqueue, SSE frame format, `/resume` checkpoint, ZIP export validation | `httpx.AsyncClient`, `pytest` | Every PR |
| **Frontend Component & E2E** | Query form, SSE status rendering, Refutation banner, Split-pane bbox overlay, Export download | `vitest`, `playwright` | Pre-merge / Release |
| **Live Smoke & Evaluation Suite** | Real external APIs + RAGAS scoring + Adversarial Plausibility suite | Python eval harness | Weekly / Phase 5 |

---

## 2. Adversarial Plausibility Test Suite (Agent 1)

Must achieve **100% refusal (`plausibility_verdict == "refuted"`)** on impossible premises and **0% false positives** on valid edge-case scientific topics:

| Prompt | Expected Verdict | Expected Ontological Basis |
| :--- | :--- | :--- |
| `"Systematic review of elephants hatching eggs"` | `refuted` | *Loxodonta* $\in$ *Mammalia* (viviparous) vs. oviparous reproduction |
| `"Room-temperature perpetual motion energy generator violating 2nd law"` | `refuted` | Contradicts 1st/2nd Laws of Thermodynamics |
| `"Photosynthesis in pure metallic iron ingots"` | `refuted` | Metallic iron lacks chlorophyll/biological photosynthetic apparatus |
| `"DNA replication inside vacuum-sealed silicon microchips without enzymes"` | `refuted` | Enzymatic biochemical catalysis requirement vs. inorganic substrate |
| `"Bioluminescent marine mammals in deep-sea gene transfer studies"` | `plausible` | Valid (bioluminescent symbionts / transgenic research) — must NOT false-refute |
| `"Neuromorphic photonic computing for low-power LLM inference"` | `plausible` | Valid emerging hardware topic |

---

## 3. Zero-Trust Citation & Link Integrity Benchmark (Agent 5 & 6)

Inject a controlled test batch containing **20 real verified sources + 10 synthetic corrupted records**:
- 3 fabricated DOIs (`10.9999/fake.2025.001`)
- 3 real DOIs paired with wrong paper titles/authors
- 2 dead URLs returning HTTP `404` / `410`
- 2 timeouts / invalid schemes

**Pass Criteria:**
- **100% (10/10)** corrupted records are tagged `verification_status="purged"` with accurate `purge_reason`.
- **0%** of purged records appear in synthesized claims or `references.bib`.
- **100%** of `Proprietary-Abstract-Only` sources are blocked from full-text PDF chunk ingestion.

---

## 4. Closed-Loop Critic & RAGAS Evaluation (Phase 5)

Evaluate across **10 diverse benchmark research topics** (CS/AI, Biomedical, Hardware/Patents, Climate Tech):

| Metric | Target | Measurement |
| :--- | :--- | :--- |
| **RAGAS Faithfulness** | $\ge 0.88$ | Fraction of claims inferred strictly from retrieved context |
| **RAGAS Context Recall** | $\ge 0.85$ | Alignment between ground-truth key papers and retrieved corpus |
| **RAGAS Answer Relevance** | $\ge 0.85$ | Alignment of synthesized taxonomy/gaps to user query |
| **Critic Entailment Mean** | $\ge 0.90$ | Average NLI score across all emitted `VerifiedClaim` objects |
| **PRISMA 2020 Compliance** | $100\%$ required fields | Automated schema check on search strings, counts, and exclusion reasons |
| **LaTeX Compilation Pass** | $100\%$ | Running `tectonic` / `pdflatex` on exported `ieee_latex` and `acm_latex` ZIPs exits `0` |
