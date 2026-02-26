# Data Governance

> This document covers data lineage, PII considerations, retention policy, and access control for the pipeline. Some sections are placeholders — they define *where* governance controls should live, even if not yet implemented.

---

## Data Lineage

Every record in the pipeline traces back to a specific PDF file via `source_file`. The lineage chain is:

```
PDF File (source_file)
  └─▶ raw_social_metrics.raw_json   (bronze — ingested_at timestamp)
        └─▶ silver DataFrame         (transformed — report_date derived)
              └─▶ fact_social_metrics (gold — platform_id, metric_id FKs)
```

**What lineage tells you:**
- If a metric value looks wrong, query `raw_social_metrics WHERE source_file = 'YYYY-MM_report.pdf'` to see the original extracted value
- If a PDF is corrected and re-issued, truncate the associated raw rows and re-run from bronze

**Planned improvement:** Add a `pipeline_run_id` column to `raw_social_metrics` and `fact_social_metrics` to track which pipeline execution produced each row. This enables rollback of a specific bad run without affecting others.

---

## PII Considerations

**Current data:** Social media performance metrics (impressions, engagement rates, follower counts). These are **aggregate, non-personal metrics** — they describe platform-level performance, not individual users.

**PII risk assessment:** LOW for the current schema.

**Where PII could enter:**
- If future PDFs include named author/influencer performance data → those names are PII
- If PDFs contain email addresses of report recipients in headers/footers → pdfplumber may extract these

**Mitigation for future PII:**
- Add a `pii_scan()` step in the ingestion stage that checks extracted text for email regex patterns, personal names, etc.
- Store PII fields in a separate, access-controlled table; never in `raw_json` in plaintext
- Document which fields are PII in the schema registry

---

## Data Retention

| Layer | Retention | Rationale |
|---|---|---|
| PDF source files | Indefinitely (or per client contract) | Legal record of original reports |
| `raw_social_metrics` (bronze) | 3 years | Enables full pipeline replay; audit trail |
| Silver (in-memory) | No persistence | Ephemeral — recreated each run |
| `fact_social_metrics` (gold) | 3 years | Matches bronze; truncate and rebuild on schema change |

**Not yet implemented:** Automated retention enforcement. Add a scheduled job or Postgres `pg_partman` partition drop policy to enforce the above.

---

## Access Control

**Principle of least privilege:**

| Role | Tables | Access |
|---|---|---|
| `pipeline_svc` (service account) | All | `INSERT`, `SELECT` |
| `analyst_role` | `fact_social_metrics`, `dim_*`, `v_social_metrics` | `SELECT` only |
| `dba_role` | All | Full DDL + DML |

**Postgres setup (example):**
```sql
-- Create read-only analyst role
CREATE ROLE analyst_role;
GRANT SELECT ON fact_social_metrics, dim_platform, dim_metric TO analyst_role;
GRANT SELECT ON v_social_metrics TO analyst_role;

-- Revoke direct access to raw bronze layer
REVOKE ALL ON raw_social_metrics FROM analyst_role;
```

**Not yet implemented:** Role creation scripts. Add these to `scripts/db_setup.sql`.

---

## Schema Registry / Data Contracts

A data contract defines what downstream consumers can depend on. The gold layer contract for `fact_social_metrics`:

```yaml
# contracts/fact_social_metrics.yaml  (placeholder — not yet implemented)
table: fact_social_metrics
owner: data-engineering-team
updated: 2024-01-01
columns:
  - name: platform_id
    type: INTEGER
    nullable: false
    references: dim_platform.platform_id
  - name: metric_id
    type: INTEGER
    nullable: false
    references: dim_metric.metric_id
  - name: report_date
    type: DATE
    nullable: false
    description: First day of the reporting month (YYYY-MM-01)
  - name: value
    type: REAL
    nullable: false
    constraints: value > 0
sla:
  freshness: monthly
  completeness: no NULL values in any column
  accuracy: value matches source PDF within 0.1%
```

**Breaking vs non-breaking changes:**
- Non-breaking: adding a new column, adding a new platform to `dim_platform`
- Breaking: renaming a column, changing a column's type, removing a column

Breaking changes require: a migration script, communication to downstream consumers, and a versioned view (`v2_social_metrics`) during the transition period.

---

## Audit Log (Placeholder)

A future `pipeline_run_log` table should capture:

```sql
CREATE TABLE pipeline_run_log (
    run_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    started_at   TIMESTAMP,
    finished_at  TIMESTAMP,
    stage        VARCHAR(50),
    status       VARCHAR(20),   -- success, failed, partial
    records_in   INTEGER,
    records_out  INTEGER,
    error_msg    TEXT
);
```

This enables: run duration tracking, anomaly detection (sudden drop in record count), and debugging without log file archaeology.