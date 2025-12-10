# Domain, Assumptions, and Terminology (DAT)

## Purpose
- Common foundation for Entity Knowledge Queries: create entities, ingest heterogeneous sources, build versioned context (indexes/graphs/summaries), and answer knowledge queries with confidence scoring. The module is product-agnostic and reused across tenants.

## Domain terminology
- Entity: business object (person, project, content bundle) with `entity_id`, `entity_type`, `display_name`, `tenant_id`, metadata.
- Source: raw information tied to an entity; types include `text`, `form_response`, `file`, `media`, `external_ref` with provenance (`origin`) and storage reference (`raw_payload_location`).
- Context: derived artifacts per entity and version (BM25 index, embedding index, knowledge graph, hierarchical summaries) built via a `build_strategy`.
- Question: knowledge query linked to an entity (types: scalar, categorical, free_text, document, scorecard) with `expected_output_schema` and `min_confidence`.
- Answer: resolved snapshot for a question/context with `status`, `confidence`, `payload`, and `explanations` (citations/provenance).
- Strategies: pluggable compositions for ingestion (preprocess/segment/enrich), encoding (chunking variants), augmentation (indexes/embeddings/graphs), storage (persistence/versioning), retrieval, and answering.
- Activation function: rule set determining when an answer becomes `resolved` (e.g., `confidence >= min_confidence` plus evidence rules).

## Data sources and ownership
- Sources ingested via connectors, user uploads, or system feeds; stored in object storage referenced by `raw_payload_location`.
- Ownership isolated by `tenant_id`; RBAC profiles: `admin`, `integrator`, `viewer`.
- Artifact storage may span RDBMS, vector stores, and file systems; naming/versioning controlled by storage strategies.

## Data flows
- UI (JavaScript) calls Python API to: create entities, ingest sources, trigger context rebuilds, manage questions/templates, run queries, fetch answers.
- Background pipelines handle ingestion -> encoding -> augmentation -> storage; context rebuilds can be async jobs.
- Webhooks emit events (`answer.resolved`, `answer.invalidated`, `context.rebuilt`) to external systems.

## Constraints and assumptions
- Multi-tenant isolation by default; no cross-tenant leakage.
- Entities, sources, and questions can arrive in any order; auto-eval stops once activation criteria met.
- Answers become `stale` when context changes per `auto_reeval_policy`; re-evaluation endpoints available.
- Strategies are configurable per entity type, question, tenant, or ingestion profile (`light`, `standard`, `rich`).
- Observability: track latency, resolved rate, volume of sources/context size per tenant/entity type.

## Open questions
- [ ] Finalize Python framework choice for the API (e.g., FastAPI vs Flask) and persistence backends (vector store, graph DB, blob storage).
- [ ] Define default ingestion/retrieval/answer strategy profiles per entity type and tenant.
- [ ] Decide authn/z mechanism (tokens/headers/session) and webhook auth model.
- [ ] Clarify rollout order for use cases (Grants, content catalog, CRM) to prioritize features.
