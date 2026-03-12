# Job Normalize Service (`services/normalize`)

Normalization service now focuses on sample extraction and sample processing.

## Commands

- `jobl-normalize-extract-samples`
  - Extracts rows from `jobs` into `normalization_samples`.
  - Current extractor rules:
    - countries: `US, CA, GB, AU, NZ, SG`
    - languages: `en, fr`
    - random selection (`ORDER BY RANDOM()`)
    - per-country balancing
    - around `95%` rows with `@` in description and `5%` without
  - CLI:
    - `--limit`

- `jobl-normalize-process-emails`
  - Rule-based email extraction and masking from `normalization_samples.description`.
  - Handles plain emails and `mailto:` variants (including HTML anchor forms).
  - Replaces every detected email occurrence with `***email_hidden***`.
  - Stores extracted emails in `normalization_samples.email`.
  - Does not update `jobs`; this step is scoped to `normalization_samples` only.

- `jobl-normalize-process-titles`
  - LLM title normalization over `normalization_samples` using OpenAI Flex processing.
  - Sends both `title` and `description` as context for title decisions.
  - Sends `company_name` too, so company names can be stripped from normalized titles.
  - Strips HTML tags from description before sending to OpenAI to reduce prompt size.
  - Writes title labels to `normalization_samples.expected_title_normalized`.

## Schema Notes

- `jobs`
  - uses `title` (raw source title)
  - uses `title_clean` (cleaned title)
  - has new `email` column
  - no `description_html`

- `normalization_samples`
  - has `company_name` (used as title normalization context)
  - has `email` (processed/extracted emails)
  - uses `title` (renamed from `title_raw`)
  - uses `description` (renamed from `description_raw`)
  - removed description expected/generated/match columns

## Run locally

```bash
cd services/normalize
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -e .
cp .env.example .env
```

## Extract samples

```bash
jobl-normalize-extract-samples --limit=5000
```

## Process emails (rule-based)

```bash
jobl-normalize-process-emails
jobl-normalize-process-emails --limit=5000
jobl-normalize-process-emails --batch-tag=extract_samples_YYYYMMDD_HHMMSS
jobl-normalize-process-emails --random
```

## Process titles (LLM)

Configure `.env`:

```bash
OPENAI_API_KEY="..."
OPENAI_MODEL="gpt-5-nano"
OPENAI_TIMEOUT_SECONDS=60
OPENAI_MAX_RETRIES=3
OPENAI_BATCH_COMPLETION_WINDOW="24h"
OPENAI_BATCH_POLL_SECONDS=15
LLM_PROMPT_VERSION="v1"
```

Run title processing:

```bash
jobl-normalize-process-titles --batch-tag=train_v1_0001 --limit=1000
jobl-normalize-process-titles --batch-tag=train_v1_0001 --limit=1000 --service-tier=flex
jobl-normalize-process-titles --batch-tag=train_v1_debug --limit=20 --debug
```
