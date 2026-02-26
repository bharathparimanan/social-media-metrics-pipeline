# Local Development Guide

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Python | >= 3.10 | Type hint syntax used throughout |
| pip | >= 23.0 | Dependency installation |
| git | any | Version control |
| PostgreSQL | >= 14 (optional) | Production-equivalent local DB |
| SQLite | (stdlib) | Default local dev DB — no install needed |

---

## Setup

### 1. Clone and enter the repo
```bash
git clone https://github.com/your-org/social-media-metrics-pipeline.git
cd social-media-metrics-pipeline
```

### 2. Create a virtual environment
```bash
python -m venv .venv
source .venv/bin/activate       # macOS/Linux
.venv\Scripts\activate          # Windows
```

### 3. Install dependencies
```bash
pip install --upgrade pip
pip install -r requirements.txt

# For development tools (tests, linting):
pip install pytest pytest-cov black isort pre-commit python-dotenv
```

### 4. Configure environment
```bash
cp .env.example .env
```

Edit `.env`:
```
USE_SQLITE=true                            # Keep true for local dev
POSTGRES_URL=postgresql://localhost:5432/social_metrics   # Ignored when USE_SQLITE=true
SOURCE_DIR=data/raw_reports
LOG_LEVEL=DEBUG
```

> Load `.env` automatically in your shell or use `python-dotenv` in scripts via `load_dotenv()`.

### 5. Create the data directories
```bash
mkdir -p data/raw_reports data/unprocessable
```

Drop any sample PDF files into `data/raw_reports/`. Filenames should follow `YYYY-MM_<description>.pdf` (e.g. `2024-01_instagram_report.pdf`) — the pipeline derives `report_date` from this pattern.

### 6. Run the pipeline
```bash
python src/pipeline/run_pipeline.py
```

Expected output:
```
[INFO] Source: 3 PDF file(s) identified
[INFO] Ingestion complete: 12 raw records extracted
[INFO] Loaded 12 records into raw storage
[INFO] Transformation complete: 11 cleaned records  (1 dropped: value=0)
[INFO] Modelling complete: fact_social_metrics populated
[INFO] Serving: query returned 2 platforms
```

---

## Running Tests

```bash
# All tests
pytest tests/ -v

# With coverage report
pytest tests/ --cov=src --cov-report=term-missing

# One stage only
pytest tests/test_transform.py -v
```

Tests do **not** require a running Postgres — they use an in-memory SQLite engine by default.

---

## Running the Notebook

```bash
jupyter lab notebooks/data_engineering_pipeline.ipynb
```

The notebook is the **reference implementation** — it explains every design decision inline. The `src/` modules are the production-ready refactor of the same logic.

---

## Switching to Postgres (Optional)

1. Ensure Postgres is running and you have a database created:
```bash
createdb social_metrics
```

2. Update `.env`:
```
USE_SQLITE=false
POSTGRES_URL=postgresql://your_user:your_password@localhost:5432/social_metrics
```

3. Install the Postgres driver if not already installed:
```bash
pip install psycopg2-binary
```

4. Re-run the pipeline — the schema will be auto-created on first run.

---

## Pre-commit Hooks

```bash
pre-commit install
```

Hooks run automatically on `git commit`. To run manually:
```bash
pre-commit run --all-files
```

Hooks configured: `black` (formatting), `isort` (import sorting), `trailing-whitespace`, `end-of-file-fixer`.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: pdfplumber` | Not installed | `pip install pdfplumber` |
| `0 records extracted` from PDF | PDF layout changed or image-based PDF | Check PDF manually; try `pymupdf` alternative |
| `OperationalError: no such table` | Schema not created | Ensure `create_raw_storage_schema()` runs before `load_into_raw_storage()` |
| `AUTOINCREMENT` error on SQLite | SQLite syntax vs Postgres | Use the SQLite-specific `CREATE TABLE` block — both are in `raw/load.py` with dialect detection |
| Date not parsed from filename | Filename doesn't match `YYYY-MM` | Rename file or pass `report_date` override flag |
| `value` column all zeros | `to_numeric` coercion failure | Check raw JSON for unexpected number formats (commas in thousands: `1,500`) — add comma-strip in transform |