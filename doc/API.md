# API (Python backend)

## Overview
- Serves question-centric Entity Knowledge Queries: declare questions/templates (with schemas and activation rules), manage entities, ingest sources, build versioned context, run knowledge queries, and return structured answers with confidence and provenance.
- All artifacts (indexes/embeddings/graphs/summaries) and pipelines exist to resolve declared questions; recomputation happens when questions or context change.
- Consumed by the JavaScript UI; UI/backend contracts must stay aligned with `doc/SPECIFICATIONS.md` and data terminology in `doc/DAT.md`.
- Base URLs: TBD for local/staging/prod.

## Tech stack
- Framework: FastAPI + Pydantic; ORM: SQLAlchemy.
- Persistence: Postgres for relational tables; pgvector for embeddings; S3-compatible object storage for raw payloads and artifacts; optional graph store (e.g., Neo4j) when graph-heavy features are enabled.
- Background jobs: Celery + Redis (or equivalent) for ingestion/context rebuild and re-eval pipelines.

## Authentication and authorization
- API issues bearer tokens; use `Authorization: Bearer <token>` and `X-Tenant-ID` on all requests.
- Token scope defaults to the issuing user and the entities they created; entity owners can grant access to other users at the entity level via roles.
- Roles: `admin` (reserved), `owner` (entity creator), and custom roles per tenant/group. Custom roles are defined by admins; users may request to join but require owner approval. `admin` may not be customized; `owner` is implicit and non-transferable without reassignment rules (TBD).
- Webhooks signed with shared-secret HMAC; receivers must verify signatures before processing events.
- Service tokens may be issued for background jobs; scoped to tenant and role.

## Endpoints
| Path | Method | Description | Request shape | Response shape | Notes |
| ---- | ------ | ----------- | ------------- | -------------- | ----- |
| `/entities` | POST | Create entity | metadata minimal | `entity_id`, full entity | |
| `/entities/{entity_id}` | GET | Fetch entity | path `entity_id` | entity details + current context + questions (optional) | |
| `/entities/{entity_id}` | PATCH | Update entity | partial `display_name`/`metadata` | updated entity | |
| `/entities/{entity_id}/sources` | POST | Ingest/reference a source | upload or URI reference | `source_id`, `ingestion_status` | triggers ingestion pipeline |
| `/entities/{entity_id}/rebuild_context` | POST | Trigger context rebuild | options TBD | `context_id`, `job_id` | async |
| `/contexts/{context_id}` | GET | Context status | path `context_id` | state, artifacts, version | |
| `/entities/{entity_id}/questions` | POST | Create entity-specific question | question definition | created question | |
| `/question_templates` | POST | Create generic question template | template definition (entity type scoped) | template id | |
| `/entities/{entity_id}/questions` | GET | List questions for entity | filters TBD | list | |
| `/entities/{entity_id}/query` | POST | Run knowledge query | `question_ids` list or inline questions; options: `force_regeneration`, `max_latency`, `strategy_profile` | snapshot of answers existing or generating | recompute rules below |
| `/entities/{entity_id}/answers` | GET | List answers | query filters: `status`, `question_type`, `tags` | list | |
| `/answers/{answer_id}` | GET | Fetch answer | path `answer_id` | detailed answer with explanations | |
| `/answers/{answer_id}/reeval` | POST | Re-evaluate answer | options: `force_latest_context`, `strategy_override` | re-eval job/result | |
| `/entities/{entity_id}/questions/{question_id}/reeval` | POST | Re-evaluate question for entity | options TBD | re-eval job/result | |
| `/auth/tokens` | POST | Issue API token | admin-only; token attributes/scopes | token id, redacted token | managed in UI by admins |
| `/auth/tokens` | GET | List tokens | admin-only | list (no secret values) | filtering by user/role |
| `/auth/tokens/{token_id}` | DELETE | Revoke token | admin-only | status | immediate revocation |

## Default strategies and profiles
- Ingestion profiles:
  - `light`: minimal preprocessing, BM25 only; use for low-volume entities/tests.
  - `standard` (default): preprocessing + semantic chunking + embeddings + BM25.
  - `rich`: hierarchical chunking + embeddings + knowledge graph + BM25; for content bundles/complex projects.
- Default mapping by entity type: `person` -> `standard`; `project` -> `standard` (upgrade to `rich` for complex dossiers); `content_bundle` -> `rich`; fallback to `standard` when unspecified.
- Retrieval/answer defaults by `question_type`:
  - `scalar`/`categorical`: `rerank_retrieval` + `deterministic_first` (rules then LLM).
  - `free_text`/`document`: `rerank_retrieval` with HyDE fallback + `ia_first`.
  - `scorecard`: `graph_heavy` when graph artifacts exist, else `standard`.
- Recompute rules: only when missing, `unresolved`/`stale`, or `force_regeneration=true`.

## Data models
- Entities: `entity_id`, `entity_type`, `display_name`, `tenant_id`, `metadata`, timestamps.
- Sources: `source_id`, `entity_id`, `source_type`, `raw_payload_location`, `schema_version`, `ingestion_status`, `origin`, `ingested_at`.
- Contexts: `context_id`, `entity_id`, `version`, `build_strategy`, `artifacts` references (indexes/graphs/summaries), timestamps.
- Questions: `question_id`, `entity_id` (nullable for templates), `question_type`, `prompt_template`, `expected_output_schema`, `min_confidence`, `retrieval_strategy`, `answer_strategy`, `auto_reeval_policy`, `tags`, timestamps.
- Answers: `answer_id`, `question_id`, `entity_id`, `context_id`, `status`, `payload`, `confidence`, `explanations` (citations/chunk ids/provenance), `retrieval_strategy_used`, `answer_strategy_used`, timestamps.
- Answer events: `event_type`, `reason`, `actor_type`, `actor_id`, `created_at`.
- Serialization: JSON; ensure schemas stay aligned with UI contracts.

## Errors
- Standard JSON error envelope (code, message, details, trace id). Status codes: 4xx client validation/auth errors; 5xx server/pipeline failures.
- Async operations should expose job status endpoints or polling guidance.
- UI must surface provenance and retry guidance when retrieval/answer strategies fail.

## Versioning and compatibility
- Versioned API (v0 sketch). Prefer additive changes; document breaking changes with deprecation windows. Contexts versioned per entity.

## Testing and observability
- Tests: unit (models/strategies), contract tests for endpoints, integration for pipelines and storage backends.
- Observability: request logs with tenant scoping, metrics (latency per endpoint, resolved rate, ingestion throughput), tracing for ingestion/retrieval/answer stages.

## Changelog
- v0: initial spec import (entities, sources, contexts, questions, answers, re-eval, events).
