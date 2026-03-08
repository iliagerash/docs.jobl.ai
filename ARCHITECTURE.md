# Jobl AI Recruitment Platform — Architecture

## 1) Goals and Constraints

- Keep the existing PHP scraping system intact and high-throughput.
- Keep scraper logic intact; only add AI export marker writes to existing `export` table.
- Provide hourly-or-better freshness to the AI application pipeline.
- Centralize AI-side search, enrichment, and orchestration on Python + Postgres.
- Minimize cross-host coupling and operational risk.

Naming:
- Application name: **Jobl AI**
- Domain name: `jobl.ai`

---

## 2) Final Decision Summary

This architecture combines the strongest choices from prior options:

1. **Minimal scraper-host changes for AI ingestion**:
   - Existing filtered writes to per-country operational MySQL remain unchanged.
   - AI pipeline reads from existing country operational DBs and marks processed jobs in `export` (`destination='jobl.ai'`).

2. **Pull-based sync to AI Host** (hourly):
   - AI Host Python sync worker pulls from each scraper country DB and writes export markers to `export` (`destination='jobl.ai'`).
   - Sync is idempotent and tracks a high-watermark checkpoint to avoid duplicates.

3. **AI source of truth = Postgres + pgvector**:
   - All downstream search, ranking, and agent flows run against Postgres on AI Host.
   - Embeddings generated asynchronously and indexed with HNSW.

4. **FastAPI boundary + webhooks**:
   - Frontend clients (Next.js SPA/PWA now, React Native later) integrate through stable API endpoints and outbound webhook notifications.

5. **Scraper queue remains local**:
   - RabbitMQ stays internal to Scraping Host and is not exposed cross-host.

### Explicit decisions vs. proposal differences

To remove ambiguity from the three candidate proposals, these choices are **final**:

- **No new buffer table on scraper host** is selected to avoid extra write load/cost.
- **Per-country operational MySQL DBs** are the ingestion source for AI sync.
- **AI host does not query scraper MySQL during runtime** for job search/agent reasoning.
  - Runtime reads come from Postgres only.
  - Scraper MySQL is ingestion-only (hourly sync input).
- **High-watermark pull + idempotent upsert** is selected as the only sync contract.
- **One canonical archive table (`jobs_archive`)** on AI host is selected as source of truth.
- **Vector dimension is model-specific** with **768 as current default** (CPU-first embedding stack); schema supports migration/versioned embeddings.

---

## 3) Architecture by Host

### 3.1 Scraping Host (PHP + MySQL + RabbitMQ)

### Responsibilities
- Execute all country/platform scrapers (Indeed, LinkedIn, others).
- Continue filtering logic for operational job board use.

### Components
- **PHP Scrapers (existing)**
  - Keep current retry/proxy/browser logic unchanged.
  - Keep dedupe/exclusion/validation flow for operational output.

- **RabbitMQ (existing, internal only)**
  - Continue coordinating worker tasks.
  - No host-to-host queue dependency.

- **MySQL**
  1. **Operational country DBs** (existing, ingestion source)
     - Filtered jobs.
     - Existing retention policy remains as-is.
     - AI Host reads new/updated rows hourly and writes export markers for processed jobs.

---

### 3.2 AI Host (Python + Postgres/pgvector)

### Responsibilities
- Ingest and archive full history.
- Enrich data (cleaning, extraction, embeddings).
- Normalize titles/descriptions for quality and Google for Jobs compatibility.
- Categorize jobs with a CPU-first classifier pipeline.
- Serve semantic search/ranking.
- Run agentic application workflows.
- Power resume-job matching and AI assistant features.
- Notify frontend/BFF services via webhooks.

### Components

- **Sync Worker (Python, hourly)**
  - Pulls rows from each country operational DB using per-source checkpoints.
  - Normalizes payloads and performs dedupe/upsert into Postgres.
  - Persists source-scoped checkpoints on AI Host (`sync_state` table in Postgres).
  - Uses batch pagination (`ORDER BY id LIMIT N` or timestamp+id cursor) to avoid long transactions.
  - Advances checkpoint only after successful Postgres commit.

- **Embedding Worker (continuous/batch)**
  - Selects rows where `embedding IS NULL`.
  - Uses local model stack (Ollama/vLLM/sentence-transformers).
  - Writes vectors and `embedded_at` timestamp.

- **LLM Enrichment Worker (optional but recommended)**
  - Extracts structured fields (skills, seniority, salary hints, remote policy).
  - Stores outputs in typed columns + JSONB provenance.

- **Normalization Worker (rules-first)**
  - Cleans job titles by removing location/salary/work-mode tokens from title fields.
  - Removes repetitive/low-value boilerplate from descriptions.
  - Extracts explicit application deadlines via regex/date parsing; falls back to LLM only when needed.
  - Produces `title_normalized`, `description_clean`, `description_html`, `expires_at`.

- **Categorization Worker (LightGBM)**
  - Uses `title_normalized + description_clean` as input text.
  - Vectorizes with TF-IDF and classifies into one category (with top-k probabilities).
  - Stores `category_id`, `category_confidence`, and `category_topk`.
  - Runs as a versioned model endpoint with offline retraining.

- **FastAPI Service**
  - `POST /v1/search` semantic + filter retrieval.
  - `GET /v1/jobs/{id}` enriched job detail.
  - `POST /v1/score` candidate-job fit scoring.
  - `POST /v1/match/jobs-for-resume` ranked jobs for a resume.
  - `POST /v1/match/resumes-for-job` ranked resumes for a job.
  - `POST /v1/content/cover-letter` constrained cover-letter generation.
  - `POST /v1/interview/session` start dynamic mock interview session.
  - `POST /v1/webhooks/register` callback registration.
  - `GET /v1/health` health endpoint for monitoring and orchestration.
  - Auth: JWT/OAuth2 for client access + service token for internal workers.
  - Contract: OpenAPI-first, backwards-compatible within major API version.

- **Notification Worker**
  - Evaluates newly embedded/enriched jobs against candidate profiles.
  - Sends webhooks to Next.js BFF/API routes (or dedicated notification service) for high-fit matches.

- **Agent Runtime (Python, e.g., PydanticAI/LangGraph)**
  - Uses Postgres as primary context store.
  - Executes auto-apply workflows (Playwright) with controlled proxy integration.

- **Resume Processing Worker**
  - Parses resume uploads into canonical structured schema.
  - Extracts normalized skills, experience blocks, seniority, and location preferences.
  - Writes resume embeddings for retrieval and scoring.

- **Matching and Ranking Service**
  - Stage 1 retrieval: ANN top-N via pgvector/FAISS over job and resume embeddings.
  - Stage 2 scoring: deterministic weighted score (semantic similarity, skills overlap, category/seniority/location fit).
  - LLM usage is optional for explanation text only, not final ranking decisions.

- **Interview Orchestrator**
  - Runs a stateful interview session per resume/job pair.
  - Selects next question from a controlled pool based on coverage and prior answer quality.
  - Uses LLM for phrasing and feedback generation; stores structured trace and scorecards.

---

### 3.3 Frontend Applications (Next.js + React Native)

### Current (Web)
- **Next.js app (React)** deployed as a web client with PWA capabilities.
- No Astro dependency.
- Uses FastAPI endpoints for search, job details, scoring, and profile flows.
- Optional Next.js BFF layer for session handling, token exchange, and webhook intake.

### Future (Mobile)
- **React Native app** reuses the same versioned FastAPI contracts (`/v1/...`).
- Keep domain logic in shared TypeScript packages where practical (validation/types/client SDK).
- Push notifications should be handled by a notification service, not directly from core workers to device providers.

---

## 4) Canonical AI Host Schema (Postgres)

`jobs_archive` (partitioned by `ingested_at` monthly; optional LIST partition by country at higher scale)

Core fields:
- Identity: `id`, `source_platform`, `country`, `source_job_id`
- Job data: `title`, `title_normalized`, `company`, `location`, `apply_url`, `posted_at`, `description_raw`, `description_clean`, `description_html`
- Metadata: `status`, `raw_json` (JSONB), `content_hash`, `expires_at`
- AI fields: `embedding VECTOR(768)`, `embedded_at`, extraction columns, `category_id`, `category_confidence`, `category_topk` (JSONB)
- Lineage: `scraped_at`, `ingested_at`, `updated_at`

Indexes:
- B-tree: `country`, `posted_at`, `scraped_at`, `status`
- Unique/near-unique key: `(source_platform, country, source_job_id)` with fallback `content_hash`
- Vector: `HNSW (embedding)`

Operational constraints:
- `apply_url` normalized before hashing/deduping.
- `content_hash` generated from stable fields (`title`, `company`, normalized `location`, normalized `apply_url`, `description_raw`).
- Failed enrichment/embedding rows are retried with backoff and a terminal `error_state` after max attempts.

Supporting tables:
- `sync_state` for per-country high-watermark checkpoints and run metadata.
- `webhook_subscriptions` for callback targets, secrets, and status.
- `job_events` (optional) for immutable lifecycle/audit events.
- `categories` for canonical category taxonomy and hierarchy.
- `resumes_archive` for normalized resume records and structured extraction fields.
- `resume_embeddings` for vector search over candidate profiles.
- `match_events` for ranking outputs and offline relevance evaluation.
- `interview_sessions` and `interview_events` for stateful mock interview history.
- `model_registry` for versioned model artifacts and rollout metadata.
- `llm_label_dataset` for teacher-labeled records used in training/distillation.

Retention:
- Default 12–24 months online (partition drop for aging data).

---

## 5) End-to-End Data Flow

1. Scraper fetches job.
2. Scraper writes filtered row to country operational DB.
3. Hourly AI sync pulls new/updated rows from each country DB in batches.
4. AI sync upserts into Postgres archive.
5. AI sync advances per-country checkpoints in AI Host state store.
6. Normalization worker cleans title/description, extracts deadlines, and writes semantic HTML.
7. Categorization worker predicts category and confidence.
8. Embedding/enrichment workers process new archive rows.
9. Search, matching, scoring, and agent automation run on AI Host via FastAPI/services.
10. Notifications sent to frontend-facing service via webhook.

---

## 6) Security and Reliability

- Cross-host access is **DB read + limited write** from AI Host:
  - read jobs from country DBs
  - write processed markers into `export` with `destination='jobl.ai'`
- IP allowlisting + TLS for DB connection.
- Idempotent sync with retries and dead-letter/error logging.
- No cross-host RabbitMQ dependency.
- API security:
  - versioned endpoints (`/v1`)
  - JWT/OAuth2 for user-facing clients
  - per-service tokens + webhook signature verification for server-to-server calls
- Data protection:
  - encrypt credentials/secrets at rest
  - redact PII fields in logs and traces
- Reliability:
  - at-least-once ingestion semantics with idempotent writes
  - backpressure controls for embedding/enrichment queues
- GenAI safety and quality controls:
  - strict JSON/schema-constrained outputs for LLM transforms
  - allowlist of permitted HTML tags for rendered descriptions
  - confidence thresholds and fallback policy for LLM-assisted extraction
- Health metrics:
  - rows scraped/hour
  - rows synced/hour
  - sync lag
  - embedding backlog
  - webhook success rate
  - category accuracy/top-k drift
  - match quality metrics (CTR/apply rate by rank bucket)
  - LLM fallback rate and schema-violation rate

---

## 7) Why this architecture fits Jobl AI

- **Low migration risk**: PHP/RabbitMQ pipeline remains essentially unchanged.
- **Low scraper overhead**: no new schemas required on scraper host; only marker writes to existing `export` table.
- **Operational simplicity**: pull-based sync avoids queue/firewall complexity.
- **AI readiness**: Postgres+pgvector unifies transactional + semantic retrieval.
- **Frontend flexibility**: same API contracts support Next.js web now and React Native later.
- **Scalable evolution**: easy to move from hourly to near-real-time later without redesign.

---

## 8) Implementation Phases

### Phase 1 (Week 1)
- Create AI Host Postgres schema (`jobs_archive`, `sync_state`, webhook tables).
- Implement hourly per-country sync worker with idempotent upsert.

### Phase 2 (Week 2)
- Add embedding worker + HNSW index.
- Ship FastAPI `/v1/search` and `/v1/jobs/{id}`.
- Add auth middleware and OpenAPI contract publishing.

### Phase 3 (Week 3)
- Add scoring + webhook notifications.
- Add agentic auto-apply pipeline (Playwright).
- Add observability dashboards + alerts.

### Phase 4 (Week 4)
- Implement rules-first normalization pipeline (`title_normalized`, `description_clean`, `description_html`, `expires_at`).
- Build initial category taxonomy and train LightGBM categorizer.
- Add resume parsing/normalization and resume embeddings.

### Phase 5 (Week 5)
- Ship `/v1/match/jobs-for-resume` and `/v1/match/resumes-for-job` with deterministic weighted scoring.
- Add `/v1/content/cover-letter` with constrained generation and caching per resume/job pair.
- Add `/v1/interview/session` with interview state machine and evaluation trace.

---

## 9) ML and NLP Pipeline Decisions

### 9.1 Task-to-model mapping (final)
- Deterministic cleanup (title/location/salary/work-mode stripping, boilerplate removal): rules/regex first.
- Deadline extraction: regex/dateparser first, LLM fallback.
- Job categorization: TF-IDF + LightGBM multiclass.
- Semantic formatting to safe HTML sections: LLM with strict schema and post-validation.
- Retrieval and matching: embeddings + deterministic scoring.
- Text generation (cover letters, interview prompts/feedback): LLM.

### 9.2 Labeling strategy for categorization
- Preferred: use existing curated categories if available.
- If labels are sparse/missing:
  1. generate weak labels via rules/keywords,
  2. label a high-quality seed set with GPT-5 models (teacher),
  3. train LightGBM on merged labeled data,
  4. expand with high-confidence self-training and human review queue.

### 9.3 Model persistence and rollout
- Persist artifacts per version:
  - `tfidf_vectorizer.joblib`
  - `job_category_lgbm.joblib`
  - `label_encoder.joblib`
- Register each release in `model_registry` with:
  - version, training window, metrics, checksum, rollout status.
- Runtime loads artifacts on service startup and supports blue/green model switching.

### 9.4 Retraining policy
- Categorization retrain cadence: weekly.
- Training window: rolling 60-90 days for robustness.
- Promotion gates: macro F1 + top-3 accuracy + per-category minimum precision.

### 9.5 Mini-LLM strategy
- Use a hybrid setup:
  - teacher model (GPT-5 family) for dataset generation and low-confidence fallback,
  - fine-tuned mini-LLM for most normalization/structuring calls.
- Keep tasks narrow and schema-bound; avoid open-ended outputs in core ingestion.

---

## 10) Resume Matching and Assistant Features

### 10.1 Resume-job matching (both directions)
- Ingest normalized resumes into `resumes_archive`.
- Create embeddings for jobs and resumes in a shared embedding space.
- Two-stage ranking:
  1. ANN retrieval top-N by cosine similarity.
  2. Deterministic re-ranker with weighted features:
     - semantic similarity
     - skills overlap
     - category fit
     - seniority fit
     - location/work-mode compatibility
- Store ranked results and feature contributions for explainability.

### 10.2 Cover letter composition
- Input is structured signals, not raw full documents.
- Generate with style/tone constraints and no fabricated claims.
- Cache by `(resume_id, job_id, template_version)` hash to control latency and cost.
- Optional premium/fallback pass with teacher model for final polish.

### 10.3 Dynamic mock interview
- Session-oriented state machine:
  - track asked questions, covered skills, answer quality, and weaknesses.
- Controlled question bank + adaptive follow-ups by confidence/coverage rules.
- Store per-turn evaluations and final rubric for candidate feedback.

### 10.4 Product guardrails
- LLMs generate narrative text, but do not own final ranking decisions.
- Ranking and eligibility decisions remain deterministic and auditable.
- All AI outputs include provenance (`model_version`, `prompt_version`, timestamp).

---

## 11) Infrastructure Sizing (CPU-First, GPU-Burst Optional)

### 11.1 Baseline CPU production profile
- Recommended baseline for ~1M jobs + ~1M resumes:
  - 32 vCPU
  - 32 GB RAM
  - 20+ GB NVMe SSD (more if retaining multiple embedding/index snapshots)
- Precompute and cache embeddings/HTML to keep online latency stable.

### 11.2 GPU usage policy
- No always-on GPU is required for core inference.
- Use short-lived rented GPU only for periodic mini-LLM adapter training or refresh.
- Keep CPU inference path as default production mode.
