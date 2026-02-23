# social-media-metrics-pipeline

> End-to-end data engineering pipeline that ingests unstructured PDF social media reports, extracts and validates metrics, models them into a queryable star schema, and serves analyst-ready SQL to BI tools — all on Postgres (SQLite for local dev).

---

## Architecture Overview

```
┌──────────┐    ┌───────────┐    ┌──────────────────┐    ┌────────────────┐    ┌──────────────────┐    ┌──────────────┐
│  SOURCE  │───▶│ INGESTION │───▶│   RAW STORAGE    │───▶│ TRANSFORMATION │───▶│    MODELLING     │───▶│   SERVING    │
│  (PDFs)  │    │ (Extract) │    │ raw_social_metric│    │  (Silver DF)   │    │ fact_social_metr │    │ Analysts/BI  │
│          │    │ pdfplumber│    │  s (Bronze/JSON) │    │  pandas clean  │    │ ics + dim_*      │    │  tools/SQL   │
└──────────┘    └───────────┘    └──────────────────┘    └────────────────┘    └──────────────────┘    └──────────────┘

Delivery guarantee: at-least-once ingestion  |  Batch (not streaming)  |  Row storage (Postgres)
```

**Data flow contract:**

```
PDF Files  →  raw_json (TEXT/JSONB)  →  Silver DataFrame  →  fact_social_metrics  →  SQL queries
(binary)      (schema-on-read)          (typed, validated)    (star schema)           (BI tools)
```

---

## Tech Stack

| Layer | Tool | Why |
|---|---|---|
| PDF Extraction | `pdfplumber` / `PyMuPDF` | Table-aware PDF parsing |
| Data Transformation | `pandas` | In-memory cleaning, type coercion |
| Database | PostgreSQL (prod) / SQLite (dev) | ACID guarantees, analyst-friendly SQL |
| ORM / SQL layer | `SQLAlchemy` | DB-agnostic, connection pooling |
| Orchestration (future) | Airflow / Prefect | Scheduled batch runs |
| Data Quality (future) | Great Expectations | Validation between stages |

---

## Quickstart (Local — SQLite)

```bash
# 1. Clone the repo
git clone https://github.com/your-org/social-media-metrics-pipeline.git
cd social-media-metrics-pipeline

# 2. Create and activate virtual environment
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Copy and edit environment config
cp .env.example .env           # Edit USE_SQLITE=true for local dev

# 5. Prepare sample data directory
mkdir -p data/raw_reports      # Drop your PDF files here

# 6. Run the full pipeline
python src/pipeline/run_pipeline.py

# 7. (Optional) Run the notebook interactively
jupyter lab notebooks/data_engineering_pipeline.ipynb
```

---

## Running Each Stage

Each stage maps to a Python module and can be run independently:

```bash
# Stage 1+2: List sources and ingest PDFs → raw storage
python -m src.ingest.extract --source-dir data/raw_reports

# Stage 3: Validate raw storage contents
python -m src.raw.validate

# Stage 4: Transform raw → silver
python -m src.transform.clean

# Stage 5: Build star schema models
python -m src.model.build_marts

# Stage 6: Run serving-layer queries (smoke test)
python -m src.serving.query_examples
```

---

## Configuration

All runtime config is via environment variables. Copy `.env.example` → `.env`.

| Variable | Default | Description |
|---|---|---|
| `USE_SQLITE` | `true` | Use SQLite (dev) instead of Postgres |
| `POSTGRES_URL` | `postgresql://localhost:5432/social_metrics` | Postgres connection string |
| `SOURCE_DIR` | `data/raw_reports` | Folder containing PDF reports |
| `LOG_LEVEL` | `INFO` | Logging verbosity |

> **Never commit `.env` or any file containing credentials.** Use `.env.example` as the template only.

---

## Data Contracts / Schemas

### Bronze (Raw) — `raw_social_metrics`
```sql
id           INTEGER  PRIMARY KEY
source_file  VARCHAR  -- filename of the originating PDF
ingested_at  TIMESTAMP
raw_json     TEXT/JSONB  -- schema-on-read: full extracted row as JSON
```
*Design intent:* Raw layer is immutable and append-only. Supports replay.

### Silver (Transformed) — in-memory DataFrame
```
platform     VARCHAR  -- standardised: "Instagram", "LinkedIn", etc.
metric_name  VARCHAR  -- standardised: "impressions", "engagement", etc.
value        FLOAT    -- parsed numeric value
report_date  DATE     -- derived from filename (YYYY-MM-01)
source_file  VARCHAR
```

### Gold (Modelled) — `fact_social_metrics`, `dim_platform`, `dim_metric`
```sql
fact_social_metrics: platform_id, metric_id, report_date, value
dim_platform:        platform_id, platform_name
dim_metric:          metric_id,   metric_name
```

---

## Testing

```bash
# Run all tests
pytest tests/ -v

# Run with coverage
pytest tests/ --cov=src --cov-report=term-missing

# Run a specific stage's tests
pytest tests/test_transform.py -v
```

Tests cover: extraction fallback logic, transformation cleaning rules, schema constraints, SQL query correctness.

---

## Logging & Observability

- All modules log via Python `logging` (structured output planned).
- Log format: `%(asctime)s [%(levelname)s] %(name)s — %(message)s`
- Key metrics logged per run: records ingested, records rejected, rows loaded per stage.
- Future: export metrics to Prometheus / Grafana or a simple run log table in Postgres.

---

## Repository Conventions

**Branching:** `main` (protected) → `feature/<stage>-<description>` → PR  
**Commits:** Conventional commits — `feat:`, `fix:`, `docs:`, `refactor:`, `test:`  
**Formatting:** `black` + `isort` enforced via `pre-commit`  
**SQL:** All SQL lives in `/src/sql/` — no inline SQL strings in Python beyond parameterised queries

```bash
# Set up pre-commit hooks
pip install pre-commit
pre-commit install
```

---

## System Design Considerations

> *This section is intentionally kept in the README to surface architectural trade-offs for reviewers.*

**Failure Modes**
- PDF extraction silently returns empty if table layout changes → mitigate with row-count assertions per file
- `INSERT OR IGNORE` on dim tables masks constraint violations → add explicit upsert logic
- SQLite WAL mode not enabled by default → concurrent writes will fail in multi-process runs

**Consistency Guarantee**
- Currently **at-least-once**: re-running ingestion re-inserts records. Add a `source_file` + `report_date` unique constraint on `raw_social_metrics` to move toward exactly-once.

**Latency Profile**
- Batch, not streaming. Suitable for monthly/weekly report cadence.
- End-to-end runtime: O(seconds) for tens of PDFs at current scale.

**Storage Format**
- Raw layer: row-oriented (Postgres/SQLite) — optimised for write throughput during ingestion
- Serving layer: same row store — acceptable at current data volume; switch to columnar (Parquet + DuckDB or Redshift) if querying millions of rows

---

## Future Improvements

- [ ] Add Airflow/Prefect DAG for scheduled ingestion
- [ ] Implement incremental load (skip already-ingested `source_file` values)
- [ ] Add Great Expectations checkpoint between raw and silver layers
- [ ] Replace SQLite serving layer with DuckDB or Redshift for columnar query performance
- [ ] Add dbt models for the gold layer (version-controlled, testable SQL transforms)
- [ ] CI/CD: GitHub Actions for lint + test on every PR
- [ ] Extend extraction to support image-based PDFs via OCR (Tesseract)
- [ ] Add a `report_metadata` table (platform, period covered, file hash) for lineage tracking