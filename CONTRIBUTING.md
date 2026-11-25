# Contributing to VERITAS

Thank you for contributing to **VERITAS**! Please read the specifications in [`docs/`](./docs/README.md) before opening a Pull Request.

## Development Workflow

1. **Branch from `main`:**
   ```bash
   git checkout -b feature/<module>-<short-description>
   ```
2. **Follow the Data Contracts:** Any backend/frontend interface change must match [`docs/04-API-AND-DATA-CONTRACTS.md`](./docs/04-API-AND-DATA-CONTRACTS.md).
3. **Commit Style:** Use Conventional Commits (e.g., `feat(verifier): add Crossref title similarity check`, `docs(trd): clarify capability router`).
4. **Before Opening a PR:**
   - Backend: `pytest` and `ruff check` pass cleanly.
   - Frontend: `npm run lint`, `npx tsc --noEmit`, and tests pass cleanly.
   - At least **1 peer review** approval is required before merging to `main`.

## Team Ownership

See [`docs/05-EXECUTION-PLAN.md`](./docs/05-EXECUTION-PLAN.md) for module ownership across Team **Code Blooded**.
