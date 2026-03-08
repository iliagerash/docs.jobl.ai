# Jobl AI Implementation Plan

## 1) Scope

This plan operationalizes the architecture in `ARCHITECTURE.md` into actionable epics and tickets for backend, ML/NLP, and product-facing AI features.

Goals:
- Deliver reliable ingestion and normalization of job data.
- Ship production categorization with LightGBM.
- Enable resume-job matching, cover letters, and mock interviews.
- Keep rollout safe with observability, model versioning, and fallback controls.

---

## 2) Delivery Phases

## Phase 1 (Foundation)
- No-change scraper integration (read-only access to country DBs)
- AI sync pipeline + canonical Postgres schema
- Core API skeleton + auth baseline

## Phase 2 (Core AI Data Plane)
- Normalization pipeline (rules + guarded LLM)
- Embeddings + vector index
- LightGBM categorization v1

## Phase 3 (Matching + User Features)
- Resume ingestion/normalization
- Two-stage matching/ranking
- Cover-letter generation
- Mock interview orchestration

## Phase 4 (Hardening + Optimization)
- Evaluation loops, drift monitoring, retraining
- Cost/latency optimization (caching, batching, fallback tuning)
- Rollout playbooks and incident readiness

---

## 3) Epics and Tickets

## Epic A: Ingestion and Storage

### A1. Configure country DB read access
- Description: provision read-only access from AI host to each scraper country DB.
- Dependencies: none.
- Acceptance criteria:
  - AI host can read required tables for each country DB.
  - Access scope is read-only and audited.
  - Country-to-source mapping is documented and tested.

### A2. Provision AI Postgres schema
- Description: create `jobs_archive` and supporting tables (`sync_state`, `webhook_subscriptions`, `model_registry`, etc.).
- Dependencies: A1.
- Acceptance criteria:
  - Schema migrations are idempotent and reversible.
  - Partitioning and indexes are created.
  - Row-level constraints for required fields are enforced.

### A3. Build sync worker (high-watermark)
- Description: implement batch pull from country operational DBs with source-scoped checkpoints.
- Dependencies: A1, A2.
- Acceptance criteria:
  - Upsert is idempotent.
  - Checkpoint advances only on successful commit.
  - Checkpoints are tracked per country/source.
  - Retries/backoff and error logging are implemented.
  - End-to-end lag metric is exposed.

### A4. Data retention and lifecycle jobs
- Description: implement retention policies for AI archive partitions and checkpoint metadata.
- Dependencies: A2, A3.
- Acceptance criteria:
  - Retention windows are configurable.
  - Expired partitions/rows are dropped safely.
  - Lifecycle jobs emit audit logs.

---

## Epic B: Normalization and Enrichment

### B1. Rules-first title normalization
- Description: strip location/salary/work-mode from titles.
- Dependencies: A3.
- Acceptance criteria:
  - `title_normalized` populated for >99% records.
  - Rules are unit-tested with a representative US title corpus.
  - Changes are deterministic and reproducible.

### B2. Description cleanup pipeline
- Description: remove repetitive boilerplate/low-value sections.
- Dependencies: A3.
- Acceptance criteria:
  - `description_clean` stored with provenance flags.
  - Major legal boilerplate patterns are removed safely.
  - No HTML/script artifacts survive cleanup.

### B3. Deadline extraction
- Description: regex/dateparser first, LLM fallback.
- Dependencies: B2.
- Acceptance criteria:
  - `expires_at` extracted when explicitly present.
  - Fallback invoked only when rule confidence is low.
  - Extraction confidence and method are stored.

### B4. Semantic HTML formatting
- Description: convert cleaned descriptions into safe semantic HTML.
- Dependencies: B2.
- Acceptance criteria:
  - Only allowlisted tags are present.
  - Invalid outputs are rejected/retried/fallbacked.
  - `description_html` generated and cached.

### B5. Enrichment worker baseline
- Description: extract skills/seniority/remote policy into structured fields.
- Dependencies: B2, B4.
- Acceptance criteria:
  - Structured columns populated for configured minimum coverage.
  - Enrichment failures are retried with capped attempts.
  - Error state is visible in ops dashboard.

---

## Epic C: Categorization (LightGBM)

### C1. Category taxonomy and mapping
- Description: define canonical categories and hierarchy.
- Dependencies: A2.
- Acceptance criteria:
  - `categories` table populated with stable IDs.
  - Mapping and display labels are documented.
  - Deprecated categories support migration mapping.

### C2. Label dataset assembly
- Description: merge existing labels + weak labels + teacher-labeled seed set.
- Dependencies: C1, B1, B2.
- Acceptance criteria:
  - Dataset snapshot stored with version and checksum.
  - Label quality sample audit completed.
  - Leakage checks pass.

### C3. Train TF-IDF + LightGBM v1
- Description: train multiclass model and compute top-k metrics.
- Dependencies: C2.
- Acceptance criteria:
  - Artifacts saved: vectorizer/model/label encoder.
  - Metrics: macro F1, top-1, top-3, per-class precision/recall.
  - Training report persisted in `model_registry`.

### C4. Categorization inference service
- Description: load artifacts on startup and score new jobs.
- Dependencies: C3.
- Acceptance criteria:
  - `category_id`, `category_confidence`, `category_topk` stored.
  - P95 inference latency target is met.
  - Blue/green model switching supported.

### C5. Weekly retraining pipeline
- Description: scheduled retraining on rolling 60-90 day window.
- Dependencies: C4.
- Acceptance criteria:
  - Automated train/eval/register flow.
  - Promotion gates enforced before production rollout.
  - Rollback to prior model version is one command/playbook step.

---

## Epic D: Embeddings and Retrieval

### D1. Embedding model integration (CPU-first)
- Description: integrate baseline embedding model and batch inference worker.
- Dependencies: A3.
- Acceptance criteria:
  - `embedding VECTOR(768)` generated for jobs and resumes.
  - Throughput and backlog metrics are captured.
  - Model version recorded per vector write.

### D2. Vector index and ANN retrieval
- Description: configure pgvector HNSW (or FAISS where needed) for top-N retrieval.
- Dependencies: D1.
- Acceptance criteria:
  - Query latency targets met at expected scale.
  - Recall@K benchmark recorded.
  - Index rebuild procedure documented.

### D3. Hybrid retrieval (optional enhancement)
- Description: blend lexical + vector signals for better recall.
- Dependencies: D2.
- Acceptance criteria:
  - A/B benchmark shows measurable lift over vector-only baseline.
  - Feature flags allow safe rollout.

---

## Epic E: Resume Processing and Matching

### E1. Resume ingestion and canonical parser
- Description: parse upload into structured resume schema.
- Dependencies: A2.
- Acceptance criteria:
  - `resumes_archive` populated with normalized sections.
  - Parsing errors categorized and observable.
  - PII handling policy is enforced.

### E2. Resume embeddings
- Description: generate and index resume vectors.
- Dependencies: E1, D1.
- Acceptance criteria:
  - Resume vectors available for retrieval flow.
  - Re-embedding supports model-version migration.

### E3. Matching retrieval endpoints
- Description: ship `/v1/match/jobs-for-resume` and `/v1/match/resumes-for-job`.
- Dependencies: D2, E2, C4.
- Acceptance criteria:
  - Endpoints return top-N with component scores.
  - Access controls applied to resume data.
  - Contract tests and load tests pass.

### E4. Deterministic reranker
- Description: implement weighted scoring (semantic, skills, category, seniority, location/work-mode).
- Dependencies: E3.
- Acceptance criteria:
  - Final score is auditable and explainable.
  - Weights are config-driven and versioned.
  - Offline relevance evaluation is tracked in `match_events`.

---

## Epic F: Generative Features

### F1. Cover-letter generation endpoint
- Description: implement `/v1/content/cover-letter` using structured inputs.
- Dependencies: E4.
- Acceptance criteria:
  - Output follows safety/style constraints.
  - Cache by `(resume_id, job_id, template_version)` is active.
  - Hallucination guardrails and rejection rules implemented.

### F2. Mock interview session service
- Description: implement `/v1/interview/session` with stateful flow and adaptive follow-ups.
- Dependencies: E4.
- Acceptance criteria:
  - Session persists turn-by-turn state and evaluations.
  - Coverage-based question selection works deterministically.
  - Rubric-based summary is produced at session end.

### F3. Teacher/fallback policy
- Description: integrate premium fallback model for low-confidence cases.
- Dependencies: F1, F2.
- Acceptance criteria:
  - Confidence thresholds configurable per task.
  - Fallback invocation and costs are tracked.
  - Degraded-mode behavior defined when fallback unavailable.

---

## Epic G: API, Security, and Frontend Integration

### G1. API contracts and versioning
- Description: finalize OpenAPI contracts for all `/v1` endpoints.
- Dependencies: A2.
- Acceptance criteria:
  - Schemas validated in CI.
  - Breaking changes blocked without version bump.
  - SDK/client generation pipeline defined.

### G2. AuthN/AuthZ and secrets management
- Description: JWT/OAuth2 for clients, service tokens for internal traffic.
- Dependencies: G1.
- Acceptance criteria:
  - Endpoint-level permissions enforced.
  - Secret rotation playbook documented.
  - Webhook signatures verified.

### G3. Next.js + React Native integration
- Description: stabilize BFF/API consumption patterns for web and future mobile.
- Dependencies: G1, G2.
- Acceptance criteria:
  - Shared typed client package available.
  - Error handling and retry patterns standardized.
  - Mobile compatibility constraints documented.

---

## Epic H: Observability, QA, and Operations

### H1. Metrics and dashboards
- Description: build dashboards for ingestion, enrichment, categorization, matching, and LLM quality.
- Dependencies: A3, B5, C4, E4, F3.
- Acceptance criteria:
  - SLO dashboards visible per service.
  - Alert thresholds configured for lag/error/backlog/drift.
  - Runbooks linked from alert channels.

### H2. Evaluation suite and regression gates
- Description: create offline benchmark suites for category and matching quality.
- Dependencies: C3, E4.
- Acceptance criteria:
  - Eval data snapshots versioned.
  - Release blocked if quality gates fail.
  - Historical trend reports available.

### H3. Incident response and rollback drills
- Description: define incident classes and rehearse rollback for models/services.
- Dependencies: C4, F3.
- Acceptance criteria:
  - Rollback tested for model and API deployments.
  - On-call playbooks completed.
  - Mean time to recovery target documented.

---

## 4) Cross-Cutting Non-Functional Requirements

- Availability target: define per-service SLO (API, sync worker, categorization worker).
- Latency targets:
  - Search/matching endpoints: P95 within product-defined budget.
  - Categorization inference: P95 within ingestion budget.
  - Generative endpoints: P95 with cache hit/miss breakdown.
- Data quality:
  - Schema-conformant LLM output rate target.
  - Category drift monitoring by class.
- Compliance/privacy:
  - PII minimization and redaction in logs.
  - Access controls for resumes and generated artifacts.

---

## 5) Suggested Execution Order (Critical Path)

1. A1 -> A2 -> A3 -> G1 -> G2  
2. B1 -> B2 -> B3 -> B4 -> B5  
3. C1 -> C2 -> C3 -> C4 -> C5  
4. D1 -> D2 -> E1 -> E2 -> E3 -> E4  
5. F1 -> F2 -> F3  
6. H1 -> H2 -> H3 (continuous from first production slice)

---

## 6) Definition of Done (Program Level)

- Production ingestion is stable and idempotent.
- Normalized job records are available with validated semantic HTML and extracted deadlines.
- LightGBM categorization is versioned, monitored, and retrainable.
- Resume-job matching is live with explainable deterministic reranking.
- Cover letters and mock interviews run behind guardrails with measurable quality.
- Dashboards, alerts, rollback procedures, and evaluation gates are active.
