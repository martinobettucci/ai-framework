# Agent Operating Guide

## Always check the docs first
- Read and follow `doc/DAT.md`, `doc/SPECIFICATIONS.md`, and `doc/API.md` before starting any work.
- Treat these documents as the source of truth; update them whenever decisions change scope, behavior, data, or interfaces.
- If information is missing or unclear, capture the open questions in the docs and avoid shipping changes that conflict with them.

## Working principles
- Keep the Python API backend and JavaScript UI aligned; record cross-cutting contracts and expectations in the docs as you add or change them.
- Update relevant doc sections in the same change set as code changes so the docs stay current.
- Document rationale for key choices to make future iterations traceable.
- Prefer small, incremental updates and avoid untracked assumptions.

## Updating the docs
- `doc/DAT.md` — domain language, core entities, data flows, constraints, and assumptions.
- `doc/SPECIFICIATIONS.md` — product/feature requirements, scope boundaries, constraints, and risks.
- `doc/API.md` — Python backend contracts, endpoints, payloads, auth, and error conventions.
- Reflect UI/backend coupling in the specs and API docs; keep examples and payloads synchronized with the implemented behavior.
