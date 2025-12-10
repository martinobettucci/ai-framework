# ai-framework

A lightweight framework that pairs a Python API backend with a JavaScript UI to explore and query knowledge-centric data. The backend owns the API surface; the UI consumes it to deliver interactive workflows.

Project status: bootstrap phase. Consult `AGENT.md` for working practices and the `doc/` folder for evolving requirements, domain notes, and API contracts.

## Stack overview
- Python: API backend (framework to be finalized).
- JavaScript: UI that consumes the backend API.

## Repository layout
- `AGENT.md` — operating instructions for contributors and automation.
- `doc/` — living documentation (`DAT.md`, `SPECIFICATIONS.md`, `API.md`).

## Next steps
- Choose and scaffold the Python API framework (e.g., FastAPI/Flask) and define the initial routes.
- Scaffold the JavaScript UI that consumes the API and document UI/backend contracts in `doc/API.md`.
