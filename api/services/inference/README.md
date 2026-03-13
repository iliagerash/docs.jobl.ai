# jobl-inference

`jobl-inference` is a CPU-oriented FastAPI service that loads a fine-tuned `google/flan-t5-large` seq2seq model and exposes HTTP endpoints to normalize raw job titles into canonical titles for downstream systems.

## Environment Variables

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `MODEL_DIR` | Yes | None | Path to the fine-tuned model directory to load at startup. |
| `NUM_BEAMS` | No | `4` | Beam width used during generation. |
| `MAX_NEW_TOKENS` | No | `32` | Maximum generated token count per prediction. |
| `BATCH_SIZE_LIMIT` | No | `64` | Maximum number of items accepted by `POST /normalize/batch`. |
| `WORKERS` | No | `1` | Number of uvicorn worker processes used by `jobl-inference`. |
| `LOG_LEVEL` | No | `INFO` | Service log level (for app + uvicorn startup). |

## Install and Run

```bash
pip install -e .
MODEL_DIR=./artifacts/flan-t5-normalize-v1/best jobl-inference
MODEL_DIR=./artifacts/flan-t5-normalize-v1/best jobl-inference --workers=4
MODEL_DIR=./artifacts/flan-t5-normalize-v1/best WORKERS=4 jobl-inference
```

## Example Requests

Single title normalization:

```bash
curl -sS http://localhost:8000/normalize \
  -H "Content-Type: application/json" \
  -d '{"title_raw":"Senior SWE - Full Time"}'
```

Batch title normalization:

```bash
curl -sS http://localhost:8000/normalize/batch \
  -H "Content-Type: application/json" \
  -d '{"items":[{"title_raw":"RN Permanent Reliever"},{"title_raw":"Lab Technician (PERM)"}]}'
```

`MODEL_DIR` is required. If it is missing or invalid, the service will fail during startup and refuse to run.
