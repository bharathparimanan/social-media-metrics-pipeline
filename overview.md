# Project Overview

## What

`social-media-metrics-pipeline` is a batch data engineering pipeline that transforms **unstructured PDF social media reports** into a **queryable star schema** in Postgres, enabling business analysts to run SQL queries and connect BI tools without ever touching a raw file.

## Why

Social media agencies and marketing teams routinely receive platform reports as PDFs — locked, unstructured, and unqueryable. This pipeline removes the manual effort of copy-pasting metrics into spreadsheets by automating the full extract → clean → model → serve workflow, producing a reliable, versioned data product that analysts can trust.

## How (Conceptual Flow)

```
1. PDFs land in a source directory (local, S3, or shared drive)
2. Ingestion reads each PDF, extracts tables via pdfplumber, and dumps raw rows into Postgres (bronze)
3. Transformation reads bronze, cleans and standardises (silver DataFrame)
4. Modelling inserts silver data into a star schema (gold: fact + dims)
5. Analysts query the gold layer directly via Postgres / BI tool
```

## Design Philosophy

**Bronze → Silver → Gold (Medallion Architecture)**  
The pipeline deliberately separates *storage* from *semantics*. The bronze layer is append-only and schema-on-read — it preserves raw data for replay. The silver layer enforces types and naming conventions. The gold layer is purpose-built for analysts.

**Row-oriented storage (for now)**  
Postgres row storage is appropriate for current volumes (tens of PDFs, thousands of rows). If query patterns become aggregation-heavy across millions of rows, migrate the serving layer to a columnar store (DuckDB, Redshift, BigQuery) — the star schema design is already compatible.

**Batch, not streaming**  
Social media reports arrive monthly or weekly. There is no sub-minute latency requirement. Batch simplifies exactly-once semantics, error recovery, and backfill logic.

## Key Failure Modes to Know

| Failure | Root Cause | Mitigation |
|---|---|---|
| Silent empty extract | PDF layout changed; pdfplumber finds no tables | Assert row count > 0 per file after extraction |
| Duplicate records on re-run | No idempotency guard on raw insert | Add UNIQUE constraint on `(source_file, row_hash)` |
| Wrong report_date | Filename doesn't match `YYYY-MM` pattern | Fallback: extract date from PDF text; raise if unparseable |
| Type coercion silent failure | `pd.to_numeric(errors='coerce')` silently nullifies | Log and count coercion failures; fail if > threshold |
| Star schema FK violation | Platform/metric name changed spelling | Normalise names before dim lookup; use fuzzy match or enum |

## Scope (What This Is Not)

- Not a real-time or near-real-time pipeline
- Not an OCR pipeline (scanned/image PDFs are not supported in the base implementation)
- Not a multi-tenant SaaS product — single-org, single-schema design