# ai-framework

Question-centric Entity Knowledge Queries: a Python API backend and JavaScript UI built to make questions the contract, not the byproduct. Every artifact (indexes, embeddings, graphs, summaries) and pipeline step exists to resolve explicit questions with structured answers, confidence, and provenance.

Project status: bootstrap phase. See `AGENT.md` for how to work, `RATIONALE.md` for the mental model, and `doc/` for evolving requirements, domain notes, and API contracts.

## Stack overview
- Python: API backend (default: FastAPI) that issues tokens, enforces tenant/role scoping, and orchestrates ingestion/context/query pipelines.
- JavaScript: UI that manages tokens (admins) and consumes the API to ask/view answers.

## Repository layout
- `AGENT.md` — operating instructions for contributors and automation.
- `RATIONALE.md` — why this exists and the question-first schema.
- `doc/` — living documentation (`DAT.md`, `SPECIFICATIONS.md`, `API.md`).

## Next steps
- Scaffold the Python API skeleton (auth/tokens, entities, sources, contexts, questions, answers).
- Scaffold the JavaScript UI shell with admin token management and basic query/answers views.
- Keep contracts and defaults in sync across `doc/DAT.md`, `doc/SPECIFICATIONS.md`, and `doc/API.md`.
