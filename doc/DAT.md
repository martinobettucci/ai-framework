# Domain, Assumptions, and Terminology (DAT)

## Purpose
- Common foundation for question-centric Entity Knowledge Queries: declare questions (with schemas and activation rules), create entities, ingest heterogeneous sources, build versioned context (indexes/graphs/summaries), and answer those questions with confidence and provenance. Product-agnostic and reused across tenants.

## Domain terminology
- Entity: business object (person, project, content bundle) with `entity_id`, `entity_type`, `display_name`, `tenant_id`, metadata; anchors questions and answers.
- Source: raw information tied to an entity; types include `text`, `form_response`, `file`, `media`, `external_ref` with provenance (`origin`) and storage reference (`raw_payload_location`); ingested to improve answers to pending questions.
- Context: derived artifacts per entity and version (BM25 index, embedding index, knowledge graph, hierarchical summaries) built via a `build_strategy`; answers reference the context version used.
- Question: first-class contract linked to an entity (types: scalar, categorical, free_text, document, scorecard) with `expected_output_schema` and `min_confidence`; drives retrieval/answer strategy choice.
- Answer: resolved snapshot for a question/context with `status`, `confidence`, `payload`, and `explanations` (citations/provenance); recomputed when context or question changes.
- Strategies: pluggable compositions for ingestion (preprocess/segment/enrich), encoding (chunking variants), augmentation (indexes/embeddings/graphs), storage (persistence/versioning), retrieval, and answering—selected to satisfy question contracts.
- Activation function: rule set determining when an answer becomes `resolved` (e.g., `confidence >= min_confidence` plus evidence rules).

## Data sources and ownership
- Sources ingested via connectors, user uploads, or system feeds; stored in object storage referenced by `raw_payload_location`.
- Ownership isolated by `tenant_id`; RBAC profiles: `admin`, `integrator`, `viewer`.
- Artifact storage may span RDBMS, vector stores, and file systems; naming/versioning controlled by storage strategies.

## Data flows
- UI (JavaScript) calls Python API to: create entities, ingest sources, trigger context rebuilds, manage questions/templates, run queries, fetch answers.
- Background pipelines handle ingestion -> encoding -> augmentation -> storage; context rebuilds can be async jobs.
- Webhooks emit events (`answer.resolved`, `answer.invalidated`, `context.rebuilt`) to external systems.
- Admin UI flows include managing API tokens issued by the backend; tokens are scoped to tenant/user and default to entities created by the token owner.

## Defaults and strategy profiles
- Ingestion profiles:
  - `light`: minimal preprocessing, BM25 only; for low-volume entities and tests.
  - `standard` (default): preprocessing + semantic chunking + embeddings + BM25.
  - `rich`: hierarchical chunking + embeddings + knowledge graph + BM25; for content bundles or complex projects.
- Default profile by entity type: `person` -> `standard`; `project` -> `standard` (upgrade to `rich` for complex dossiers); `content_bundle` -> `rich`; fallback to `standard` if unspecified.
- Retrieval/answer defaults:
  - `scalar`/`categorical`: `rerank_retrieval` + `deterministic_first` (rules then LLM).
  - `free_text`/`document`: `rerank_retrieval` with HyDE fallback + `ia_first`.
  - `scorecard`: `graph_heavy` when graph artifacts exist, otherwise `standard`.
- Storage defaults: Postgres + SQLAlchemy for relational data; pgvector for embeddings; S3-compatible object storage for raw payloads and artifacts; optional graph store (e.g., Neo4j) when graph-heavy features are enabled; Redis + Celery (or equivalent) for background jobs.
- Auth defaults: backend issues bearer tokens; `Authorization: Bearer <token>` plus `X-Tenant-ID` required. Tokens are scoped to the issuing user and, by default, only grant access to entities they created. Entity owners can grant roles to other users at the entity level. Roles: `admin` (reserved), `owner` (creator of the entity), and custom roles defined per tenant/group; users may request to join a custom role but require owner approval. Webhook signatures via shared-secret HMAC.

## Constraints and assumptions
- Multi-tenant isolation by default; no cross-tenant leakage.
- Entities, sources, and questions can arrive in any order; auto-eval stops once activation criteria met.
- Answers become `stale` when context changes per `auto_reeval_policy`; re-evaluation endpoints available.
- Strategies are configurable per entity type, question, tenant, or ingestion profile (`light`, `standard`, `rich`).
- Observability: track latency, resolved rate, volume of sources/context size per tenant/entity type.
- Default stack choice: Python API with FastAPI, Postgres/pgvector, S3-compatible blobs, Redis + Celery; can be swapped per deployment if documented.

## Open questions
- [ ] Confirm graph store choice and rollout plan for graph-heavy features.
- [ ] Clarify rollout order for use cases (Grants, content catalog, CRM) to prioritize features.
