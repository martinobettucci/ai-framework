# Entity Knowledge Queries - Technical Specifications v0

## 1. Objective and scope
- Provide a reusable module to create business entities, ingest and normalize heterogeneous sources, build/maintain queryable context (indexes, knowledge graphs, summaries), run knowledge queries over entities, and return structured answers with confidence and resolution status.
- Must stay agnostic to specific products (Grants, content catalogs, CRM) and expose a stable API for reuse.

## 2. Key concepts
### 2.1 Entity
- High-level business object (examples: Person, Project, ContentBundle).
- Minimal fields: `entity_id` (UUID), `entity_type` (normalized string), `display_name`, `created_at`, `updated_at`, `tenant_id`, `metadata` (extensible key/value).

### 2.2 Source
- Any raw piece of information tied to an entity.
- Types: `text`, `form_response`, `file`, `media`, `external_ref`.
- Minimal fields: `source_id`, `entity_id`, `source_type`, `raw_payload_location` (object storage URI), `schema_version`, `ingestion_status`, `ingested_at`, `origin` (connector/system/user).

### 2.3 Context
- Derived structures to optimize queries: full-text index (BM25), embedding index, knowledge graph, hierarchical summaries.
- Versioned per entity.
- Conceptual fields: `context_id`, `entity_id`, `version` (int), `build_strategy`, `artifacts` (references to indexes/graphs/summaries).

### 2.4 Question
- Knowledge query attached to an entity.
- Example types: `scalar`, `categorical`, `free_text`, `document`, `scorecard`.
- Minimal fields: `question_id`, `entity_id`, `question_type`, `prompt_template`, `expected_output_schema`, `min_confidence`.

### 2.5 Answer
- Result of a knowledge query for a given entity/context.
- Minimal fields: `answer_id`, `question_id`, `entity_id`, `context_version`, `status` (`unresolved`, `in_progress`, `resolved`, `stale`), `payload` (JSON per `expected_output_schema`), `confidence` (0-1), `explanations` (trace/citations/provenance), timestamps.

### 2.6 Activation function
- Rules that allow an answer to transition to `resolved` (e.g., `confidence >= min_confidence`, minimum supporting evidence, deterministic rule + LLM score combos).

### 2.7 Ingestion strategies
- Compositions of steps implemented by dedicated classes:
  - Preprocessing (markdownize, OCR, vision, audio extraction, etc.).
  - Initial segmentation (blocks/sections/pages).
  - Low-level metadata enrichment (doc type, language, length, author).

### 2.8 Encoding strategies
- Control how normalized content is chunked/encoded:
  - `simple_chunking`, `semantic_chunking`, `paragraph_chunking`, `hierarchical_document_chunking`.
- Each strategy composes walkers that produce encodable units.

### 2.9 Augmentation strategies
- Define derived structures built from encoded chunks: BM25 index, embeddings, derived metadata, knowledge graphs.

### 2.10 Storage strategies
- Persistence rules for generated artifacts: backend choice (RDBMS, KV, vector store, file system), naming/versioning conventions, durability/cleanup. Configurable per ingestion profile or entity type.

## 3. Logical data model (relational view)
### 3.1 Tables
- `entities`: `id`, `entity_type`, `display_name`, `tenant_id`, `metadata`, `created_at`, `updated_at`.
- `sources`: `id`, `entity_id` (FK), `source_type`, `location`, `origin`, `schema_version`, `ingestion_status`, `ingested_at`.
- `contexts`: `id`, `entity_id` (FK), `version`, `build_strategy`, `artifacts` (JSONB), `created_at`.
- `questions`: `id`, `entity_id` (FK, nullable for generic questions by entity type), `question_type`, `prompt_template`, `expected_output_schema` (JSONB), `min_confidence`, `retrieval_strategy`, `answer_strategy`, `auto_reeval_policy` (e.g., `on_context_change`, `manual_only`), `tags`, `created_at`, `updated_at`.
- `answers`: `id`, `question_id` (FK), `entity_id` (FK), `context_id` (FK), `status`, `payload` (JSONB), `confidence`, `explanations` (JSONB with citations/chunk ids), `retrieval_strategy_used`, `answer_strategy_used`, `created_at`, `updated_at`.
- `answer_events`: `id`, `answer_id` (FK), `event_type` (`created`, `regenerated`, `invalidated`, etc.), `reason`, `actor_type` (`system`, `user`, `judge`, `admin`), `actor_id`, `created_at`.

## 4. HTTP API v0 (sketch)
### 4.1 Entities
- `POST /entities` — create entity (returns `entity_id`, full representation).
- `GET /entities/{entity_id}` — entity details, current context, attached questions (optional).
- `PATCH /entities/{entity_id}` — partial update for `metadata`/`display_name`.

### 4.2 Sources and context
- `POST /entities/{entity_id}/sources` — upload/reference a source; returns `source_id`, `ingestion_status`.
- `POST /entities/{entity_id}/rebuild_context` — trigger async context rebuild; returns `context_id`, `job_id`.
- `GET /contexts/{context_id}` — context state, artifacts, version.

### 4.3 Questions
- `POST /entities/{entity_id}/questions` — create entity-specific question.
- `POST /question_templates` — define generic question by entity type.
- `GET /entities/{entity_id}/questions` — list questions for an entity.

### 4.4 Knowledge queries and answers
- `POST /entities/{entity_id}/query` — body: list of `question_ids` or inline questions; options: `force_regeneration`, `max_latency`, `strategy_profile`; returns snapshot of existing/in-progress answers.
- `GET /entities/{entity_id}/answers` — filter by `status`, `question_type`, `tags`.
- `GET /answers/{answer_id}` — detailed answer with explanations.

### 4.5 Webhooks and events
- Events: `answer.resolved`, `answer.invalidated`, `context.rebuilt`.
- Payload includes `tenant_id`, `entity_id`, `question_id`, `answer_id`, `status`.

### 4.6 Re-evaluation
- Constraints: entities/questions/sources may arrive in any order; auto-eval stops when `min_confidence` met; context changes can mark answers `stale` per `auto_reeval_policy`.
- `POST /answers/{answer_id}/reeval` — re-evaluate existing answer with current context; options: `force_latest_context`, `strategy_override`.
- `POST /entities/{entity_id}/questions/{question_id}/reeval` — targeted re-evaluation.
- Query operations recompute only if missing, `unresolved`/`stale`, or `force_regeneration=true`.

## 5. Ingestion and context pipeline
### 5.1 Generic steps
1) Ingestion: preprocessing (markdownize, OCR/vision, transcription, raw metadata extraction); initial segmentation (pages/slides/chapters/blocks).
2) Encoding: apply chunking strategy (simple/semantic/paragraph/hierarchical); encode into usable representations (tokens/embeddings/features).
3) Augmentation: build BM25 indexes; generate/store embeddings; derive metadata; build/update knowledge graphs.
4) Storage: persist artifacts (indexes/graphs/summaries/metadata) in DB and file system; update `contexts` references.
- Each sub-step is a dedicated class to compose per source, tenant, or ingestion profile.

### 5.2 Ingestion profiles
- `light` for low-volume entities.
- `standard` for typical cases.
- `rich` for content bundles/complex projects.
- Each profile governs artifacts generated, analysis depth, and context rebuild frequency.

## 6. Question resolution engine
### 6.1 Standard cycle
1) Select relevant context (active `context_version`).
2) Retrieval strategy (HyDE, dense+rerrank, grounded gridsearch, etc.) to produce passages with provenance.
3) Answer generation strategy per `question_type` (LLM, rules, or hybrid).
4) Confidence evaluation (similarity, coherence, source diversity, stability).
5) Activation function (update `status` to `resolved` when criteria met).
6) Persistence and events (write `answers`/`answer_events`, emit external events).
- Answers may be re-evaluated; history kept in `answer_events`; stale answers marked on context change.

### 6.2 Resolution strategies
- Retrieval strategies: `hyde_retrieval`, `rerank_retrieval`, `grounded_gridsearch` (consume context artifacts; return passages with provenance).
- Answer strategies: `scalar_answer`, `categorical_answer`, `free_text_answer`, `document_answer`, `scorecard_answer` (consume passages, respect `expected_output_schema`, include citations).
- Higher-level profiles can combine retrieval/answer strategies (e.g., `deterministic_first`, `ia_first`, `graph_heavy`) and can be set per question, entity type, or tenant.

## 7. Governance, security, multi-tenant
- Isolation: each entity belongs to a `tenant_id`; contexts, sources, answers isolated per tenant; no cross-tenant leakage.
- RBAC (minimal): `admin` (schema/strategies/config), `integrator` (entities/sources/API use), `viewer` (read entities/answers/explanations).
- Observability: metrics per tenant and entity type (latency, resolved rate, source volume, context size).

## 8. Extensibility and roadmap
- Add retrieval strategies; native multimodality (image/audio/video) in graphs; external enrichment tools; advanced context versioning with rollback; offline batch recomputation mode.
- This v0 spec is a base to refine per use case (Grants, content catalogs, augmentation engines) and to prioritize incremental delivery.
