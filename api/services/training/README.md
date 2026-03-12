# Job Training Service (`services/training`)

Utilities to prepare labeled normalization data and run LoRA fine-tuning for **title normalization only**.

## Install

```bash
cd /home/<user>/Jobl/api.jobl.ai/services/training
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -e .
cp .env.example .env
```

For model training dependencies:

```bash
pip install -e ".[train]"
```

## Scripts reference

### `jobl-training-export`

Purpose:
- Reads labeled rows from Postgres `normalization_samples`.
- Produces the raw supervised dataset in JSONL.

Selection logic:
- `title_raw` must be non-empty.
- `expected_title_normalized` must be non-empty.
- `language_code` must be one of: `en, es, de, fr, pt, it, gr, uk, nl, da`.
- Optional: filter by one `batch_tag`.

Arguments:
- `--out`: output JSONL path, default `data/raw/labeled.jsonl`.
- `--limit`: max rows, `0` means all.
- `--batch-tag`: optional batch filter.

Output row fields:
- `id`, `language_code`, `country_code`, `country_name`, `region_title`, `city_title`
- `title_raw`
- `expected_title_normalized`

Example:

```bash
jobl-training-export --out=data/raw/labeled.jsonl --limit=1000
jobl-training-export --batch-tag=train_v1_0001 --out=data/raw/labeled.jsonl
```

### `jobl-training-split`

Purpose:
- Splits exported JSONL into `train/val/test`.
- Preserves language distribution by stratifying on `language_code`.

Arguments:
- `--in`: input JSONL path.
- `--out-dir`: output directory.
- `--train`, `--val`, `--test`: ratios; must sum to `1.0`.
- `--seed`: deterministic shuffle seed.

Outputs:
- `train.jsonl`, `val.jsonl`, `test.jsonl`
- `manifest.json` with total counts and language distribution per split.

### `jobl-training-build-jsonl`

Purpose:
- Converts split rows into instruction-format JSONL for SFT.
- Builds `messages` in `system/user/assistant` format.

Behavior:
- `system`: title normalization prompt.
- `user`: JSON payload with raw title and location context.
- `assistant`: JSON payload with expected `title_normalized`.

Arguments:
- `--split`: one of `train|val|test`, default `train`.
- `--in`: split JSONL path, default `data/splits/<split>.jsonl`.
- `--out`: instruction JSONL path, default `data/sft/<split>.jsonl`.
- `--prompt-version`: stored in user payload, default `v1`.

Example:

```bash
jobl-training-build-jsonl
jobl-training-build-jsonl --split=val
jobl-training-build-jsonl --split=test
```

### `jobl-training-train-lora`

Purpose:
- Runs LoRA SFT training on instruction JSONL.
- Saves adapter checkpoint for local inference/evaluation.

Defaults:
- Base model: `microsoft/Phi-3-mini-4k-instruct`
- Train/val JSONL: `data/sft/train.jsonl` and `data/sft/val.jsonl`

Arguments:
- `--train-jsonl`, `--val-jsonl`: instruction datasets.
- `--model`: HF base model id.
- `--out-dir`: training output directory.
- `--epochs`, `--batch-size`, `--grad-accum`, `--lr`, `--max-seq-len`: training hyperparameters.
- `--memory-safe`: CPU-safe profile for low-memory hosts.

Outputs:
- Trainer checkpoints under `--out-dir`.
- Final adapter in `--out-dir/adapter`.

Example:

```bash
jobl-training-train-lora
jobl-training-train-lora --memory-safe
jobl-training-train-lora \
  --train-jsonl=data/sft/train.jsonl \
  --val-jsonl=data/sft/val.jsonl \
  --model=microsoft/Phi-3-mini-4k-instruct \
  --out-dir=artifacts/lora-normalize-v1
```

### `jobl-training-eval-lora`

Purpose:
- Evaluates a trained LoRA adapter on `data/sft/test.jsonl`.
- Computes title quality metrics and exports mismatches for manual review.

Arguments:
- `--test-jsonl`: test instruction JSONL path (default `data/sft/test.jsonl`).
- `--model`: optional base model id override.
- `--adapter-dir`: adapter directory (default `artifacts/lora-normalize-v1/adapter`).
- `--limit`: optional max rows to evaluate, `0` means all.
- `--batch-size`: inference batch size (default `8`).
- `--max-new-tokens`, `--temperature`: inference params.
- `--changed-titles-only`: default enabled; use `--all-titles` to include unchanged rows.
- `--progress-every`: progress log interval in rows (default `10`).
- `--out-dir`: output artifacts directory.

Metrics:
- `valid_json_rate`
- `title_exact_rate`
- `title_non_empty_rate`
- `title_similarity_avg`
- `title_similarity_ge_0_8_rate`

Outputs:
- `summary.json`
- `mismatches.jsonl`
- `mismatches.csv`

Example:

```bash
jobl-training-eval-lora
jobl-training-eval-lora --all-titles
jobl-training-eval-lora --limit=100 --out-dir=artifacts/lora-normalize-v1/eval_smoke
```

## End-to-end quick run

```bash
jobl-training-export --out=data/raw/labeled.jsonl --limit=1000
jobl-training-split --in=data/raw/labeled.jsonl --out-dir=data/splits
jobl-training-build-jsonl --in=data/splits/train.jsonl --out=data/sft/train.jsonl
jobl-training-build-jsonl --in=data/splits/val.jsonl --out=data/sft/val.jsonl
jobl-training-build-jsonl --in=data/splits/test.jsonl --out=data/sft/test.jsonl
jobl-training-train-lora --memory-safe
jobl-training-eval-lora --limit=100
```
