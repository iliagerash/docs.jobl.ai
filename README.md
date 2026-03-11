# Jobl AI Docs (`docs.jobl.ai`)

Documentation repository for the Jobl AI platform.

## Repositories organization

The project is split into three repositories:

- [`api.jobl.ai`](https://github.com/iliagerash/api.jobl.ai): Python backend (FastAPI + Mini-LLM).
- [`jobl.ai`](https://github.com/iliagerash/jobl.ai): React frontend.
- [`docs.jobl.ai`](https://github.com/iliagerash/docs.jobl.ai): central documentation, architecture, and planning.

## What this repo contains

- [Architecture](./ARCHITECTURE.md): system architecture for ingestion, storage, AI processing, API boundaries, and frontend/mobile integration.
- [Implementation Plan](./IMPLEMENTATION_PLAN.md): roadmap with epics, ticket-level tasks, dependencies, and acceptance criteria.
- [API Repo Docs](./api/README.md): centralized docs moved from `api.jobl.ai`.

## API documentation map

The following docs were moved from `api.jobl.ai` into this repository:

- [api/README.md](./api/README.md)
- [api/libs/common/README.md](./api/libs/common/README.md)
- [api/services/sync/README.md](./api/services/sync/README.md)
- [api/services/inference/README.md](./api/services/inference/README.md)
- [api/services/normalize/README.md](./api/services/normalize/README.md)
- [api/services/training/README.md](./api/services/training/README.md)
- [api/services/training/MODEL_SELECTION.md](./api/services/training/MODEL_SELECTION.md)

## Product direction (summary)

- Existing PHP scrapers remain intact; AI host only adds export marker writes to existing `export` table (`destination='jobl.ai'`).
- AI host pulls new/updated jobs hourly from scraper country DBs, then writes export markers (`export.destination = 'jobl.ai'`) in those DBs and upserts into Postgres (`pgvector` enabled).
- Processing pipeline is hybrid:
  - rules-first normalization
  - LightGBM categorization
  - LLM-assisted semantic structuring and selective fallback tasks
