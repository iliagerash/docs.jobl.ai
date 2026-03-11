# Model Selection: Phi vs DeepSeek (Normalization)

Date: 2026-03-12

## Scope

This document compares:

- `microsoft/Phi-3-mini-4k-instruct` + LoRA adapter
- `deepseek-ai/DeepSeek-R1-Distill-Qwen-7B` + LoRA adapter

Evaluation setup used:

- Eval command family: `jobl-training-eval-lora*`
- Chunked inference: enabled
- Changed-titles-only filtering: enabled
- Batch size: 8
- Evaluated rows: 100 (limited sample)

Notes on row counts in summary:

- `source_total_rows` is full test source before filtering
- `selected_rows` is post-filter and post-limit
- `skipped_unchanged_title_rows` is computed before `--limit`

## Results

### Phi

- `valid_json_rate`: `1.00`
- `title_exact_rate`: `0.45`
- `title_similarity_avg`: `0.8312`
- `title_similarity_ge_0_8_rate`: `0.67`
- `html_non_empty_rate`: `0.62`
- `html_text_similarity_avg`: `0.3166`
- `html_text_similarity_ge_0_8_rate`: `0.21`

### DeepSeek

- `valid_json_rate`: `1.00`
- `title_exact_rate`: `0.37`
- `title_similarity_avg`: `0.7722`
- `title_similarity_ge_0_8_rate`: `0.58`
- `html_non_empty_rate`: `0.56`
- `html_text_similarity_avg`: `0.3196`
- `html_text_similarity_ge_0_8_rate`: `0.22`

## Decision

Use **Phi-3-mini** as the default normalization model for the next phase.

Reasoning:

- Phi is clearly better on title normalization quality (exact + similarity).
- HTML quality is very close between models; DeepSeek has only a marginal edge on one similarity metric.
- Phi is better balanced for current production goals.

## Next Steps

1. Proceed with CPU-focused inference optimization using Phi as primary.
2. Keep DeepSeek as an experimental branch only.
3. Re-run model comparison on a larger changed-title sample after inference pipeline stabilization.
