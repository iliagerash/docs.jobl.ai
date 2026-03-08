# Jobl AI Docs (`docs.jobl.ai`)

Documentation repository for the Jobl AI platform.

## Repositories organization

The project is split into three repositories:

- [`api.jobl.ai`](../api.jobl.ai): Python backend (FastAPI + Mini-LLM).
- [`jobl.ai`](../jobl.ai): Astro frontend.
- [`docs.jobl.ai`](.): central documentation, architecture, and planning.

## What this repo contains

- [Architecture](./ARCHITECTURE.md): system architecture for ingestion, storage, AI processing, API boundaries, and frontend/mobile integration.
- [Implementation Plan](./IMPLEMENTATION_PLAN.md): roadmap with epics, ticket-level tasks, dependencies, and acceptance criteria.

## Product direction (summary)

- Existing PHP scrapers remain intact with no AI-related write-path changes.
- AI host pulls new/updated jobs hourly from scraper country DBs via read-only access, then upserts into Postgres (`pgvector` enabled).
- Processing pipeline is hybrid:
  - rules-first normalization
  - LightGBM categorization
  - LLM-assisted semantic structuring and selective fallback tasks
