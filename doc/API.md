# API (Python backend)

## Overview
- Serves Entity Knowledge Queries: manage entities, ingest sources, build versioned context, define questions/templates, run knowledge queries, and return structured answers with confidence and provenance.
- Consumed by the JavaScript UI; UI/backend contracts must stay aligned with `doc/SPECIFICATIONS.md` and data terminology in `doc/DAT.md`.
- Base URLs: TBD for local/staging/prod.

## Tech stack
- Framework: TBD (candidate: FastAPI). ORM/persistence: RDBMS + vector store + file/blob storage; graph DB optional for knowledge graph artifacts.
- Background jobs: TBD (e.g., Celery/RQ) for ingestion and context rebuilds.

## Authentication and authorization
- Approach TBD. Must support multi-tenant isolation (`tenant_id`) and minimal RBAC roles: `admin`, `integrator`, `viewer`.
- Webhook auth/signature model TBD.

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
