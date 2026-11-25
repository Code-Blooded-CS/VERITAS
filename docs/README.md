# VERITAS Documentation Index

> **Verified Evidence-Based Research, IP & Autonomous Synthesis System**  
> Maintained by Team **Code Blooded**

This directory contains the complete product, technical, architectural, and operational specifications created prior to implementation. All documents are aligned with the project overview in [README.md](../README.md) and the academic specification in [synopsis.tex](../synopsis.tex).

## Document Map

| Document | Purpose | Primary Audience |
| :--- | :--- | :--- |
| [01-PRD.md](./01-PRD.md) | Product vision, personas, functional/non-functional requirements, user stories, KPIs, and MVP vs. v1.0 scope | Full team, Project Guide |
| [02-TRD.md](./02-TRD.md) | System architecture, 8-agent LangGraph design, retrieval/verification algorithms, tech stack, and deployment | Backend & Agent Engineers |
| [03-WORKFLOW-DIAGRAMS.md](./03-WORKFLOW-DIAGRAMS.md) | Mermaid architecture, StateGraph, sequence, PRISMA funnel, ingestion, and verification flow diagrams | Full team |
| [04-API-AND-DATA-CONTRACTS.md](./04-API-AND-DATA-CONTRACTS.md) | REST endpoints, SSE/WebSocket telemetry events, and canonical Pydantic/JSON schemas | Frontend & Backend Leads |
| [05-EXECUTION-PLAN.md](./05-EXECUTION-PLAN.md) | Phased build roadmap, 6-month academic milestone alignment, 5-member sprint split, and risk register | Full team |
| [06-UI-UX-SPEC.md](./06-UI-UX-SPEC.md) | Next.js 14 Research Cockpit views, split-pane PDF provenance UX, D3.js graph spec, and verification badges | Frontend Lead |
| [07-TEST-AND-EVAL-PLAN.md](./07-TEST-AND-EVAL-PLAN.md) | Unit/integration test strategy, adversarial plausibility suite, link integrity tests, and RAGAS evaluation | Verification & QA |
| [08-SECURITY-AND-ADRS.md](./08-SECURITY-AND-ADRS.md) | Practical SSRF/prompt-injection/LaTeX guardrails and Architecture Decision Records (ADRs) | Full team |

## Recommended Reading Order

1. Start with **[01-PRD.md](./01-PRD.md)** to understand *what* we are building and why.
2. Review **[03-WORKFLOW-DIAGRAMS.md](./03-WORKFLOW-DIAGRAMS.md)** to visualize the end-to-end closed-loop pipeline.
3. Study **[02-TRD.md](./02-TRD.md)** and **[04-API-AND-DATA-CONTRACTS.md](./04-API-AND-DATA-CONTRACTS.md)** before writing any code.
4. Use **[05-EXECUTION-PLAN.md](./05-EXECUTION-PLAN.md)** to assign Phase 0 & Phase 1 tasks across the team.
