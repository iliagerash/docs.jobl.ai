# Job Training Service (`services/training`)

Utilities to prepare labeled normalization data and run FLAN-T5 seq2seq fine-tuning for **title normalization only**.

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
- `title` must be non-empty.
- `expected_title_normalized` must be non-empty.
- `language_code` must be one of: `en, fr`.
- Optional: filter by one `batch_tag`.

Arguments:
- `--out`: output JSONL path, default `data/raw/labeled.jsonl`.
- `--limit`: max rows, `0` means all.
- `--batch-tag`: optional batch filter.

Output row fields:
- `id`, `language_code`, `title`, `expected_title_normalized`

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
- Converts split rows into seq2seq JSONL.
- Builds flat `input` and `target` text fields.

Behavior:
- Input format: `normalize job title: <pre_stripped_title>`
- Target format: `<expected_title_normalized>`
- Applies deterministic `pre_strip` cleanup to raw titles before model input creation.

Arguments:
- `--split`: one of `train|val|test`, default `train`.
- `--in`: split JSONL path, default `data/splits/<split>.jsonl`.
- `--out`: seq2seq JSONL path, default `data/sft/<split>.jsonl`.
- `--coverage-check`: enabled by default; on `train` split logs warning for low pattern coverage.

Example:

```bash
jobl-training-build-jsonl
jobl-training-build-jsonl --split=val
jobl-training-build-jsonl --split=test
```

### `jobl-training-train-seq2seq`

Purpose:
- Runs seq2seq fine-tuning on `input`/`target` JSONL.
- Saves best model + tokenizer for local inference/evaluation.

Defaults:
- Base model: `google/flan-t5-large`
- Train/val JSONL: `data/sft/train.jsonl` and `data/sft/val.jsonl`
- Output dir: `artifacts/flan-t5-normalize-v1`

Arguments:
- `--train-jsonl`, `--val-jsonl`: seq2seq datasets.
- `--model`: HF base model id.
- `--out-dir`: training output directory.
- `--epochs`, `--batch-size`, `--lr`: training hyperparameters.
- `--memory-safe`: CPU-safe profile for low-memory hosts.

Outputs:
- Trainer checkpoints under `--out-dir`.
- Best model in `--out-dir/best`.

Example:

```bash
jobl-training-train-seq2seq
jobl-training-train-seq2seq --memory-safe
jobl-training-train-seq2seq \
  --train-jsonl=data/sft/train.jsonl \
  --val-jsonl=data/sft/val.jsonl \
  --model=google/flan-t5-large \
  --out-dir=artifacts/flan-t5-normalize-v1
```

### `jobl-training-eval-seq2seq`

Purpose:
- Evaluates a trained seq2seq model on `data/sft/test.jsonl`.
- Computes title quality metrics and exports mismatches for manual review.

Arguments:
- `--test-jsonl`: test seq2seq JSONL path (default `data/sft/test.jsonl`).
- `--model-dir`: model directory (default `artifacts/flan-t5-normalize-v1/best`).
- `--limit`: optional max rows to evaluate, `0` means all.
- `--batch-size`: inference batch size (default `8`).
- `--changed-titles-only`: default enabled; use `--all-titles` to include unchanged rows.
- `--progress-every`: progress log interval in rows (default `10`).
- `--out-dir`: output artifacts directory.

Metrics:
- `valid_json_rate` (`null` for seq2seq plain-text output; key kept for compatibility)
- `title_exact_rate`
- `title_non_empty_rate`
- `title_similarity_avg`
- `title_similarity_ge_0_8_rate`

Outputs:
- `summary.json`
- `mismatches.jsonl`
- `mismatches.csv` with columns: `title_original,title_expected,title_generated`

Example:

```bash
jobl-training-eval-seq2seq
jobl-training-eval-seq2seq --all-titles
jobl-training-eval-seq2seq --limit=100 --out-dir=artifacts/flan-t5-normalize-v1/eval_smoke
```

## End-to-end quick run

```bash
jobl-training-export --out=data/raw/labeled.jsonl --limit=1000
jobl-training-split --in=data/raw/labeled.jsonl --out-dir=data/splits
jobl-training-build-jsonl --in=data/splits/train.jsonl --out=data/sft/train.jsonl
jobl-training-build-jsonl --in=data/splits/val.jsonl --out=data/sft/val.jsonl
jobl-training-build-jsonl --in=data/splits/test.jsonl --out=data/sft/test.jsonl
jobl-training-train-seq2seq --memory-safe
jobl-training-eval-seq2seq --limit=100
```
