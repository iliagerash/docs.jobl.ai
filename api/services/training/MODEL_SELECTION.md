# Model Selection: Phi (Normalization)

Date: 2026-03-12

## Decision

Use `microsoft/Phi-3-mini-4k-instruct` as the base model for Jobl normalization training.

## Scope

- Normalization target: `title_normalized` only.
- Evaluation command: `jobl-training-eval-lora`.
- Primary metrics: `title_exact_rate`, `title_similarity_avg`, `valid_json_rate`.

## Notes

- Legacy non-Phi scripts and defaults have been removed from the training service.
- Chunked description generation/evaluation paths have been removed.
