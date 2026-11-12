# Automated Compliance Audit & Deterministic Intent Classification Engine

![CI](https://github.com/<your-username>/compliance-audit-engine/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/badge/python-3.11%2B-blue)
![Architecture](https://img.shields.io/badge/architecture-100%25%20deterministic%20(no%20ML)-green)
![Coverage](https://img.shields.io/badge/coverage-95%25-brightgreen)

An end-to-end, rule-based data pipeline that ingests raw UK customer interaction logs, **masks PII under UK GDPR before anything is stored**, classifies intent against a JSON taxonomy, calculates an **auditable Compliance Risk Score**, and loads the results into a SQL warehouse that powers a compliance dashboard and an auditor work queue.

Built for UK regulated sectors (Financial Services, Utilities, Telco, Healthcare) against **FCA Consumer Duty (FG21/1, PS22/9)** and **UK GDPR / DPA 2018 / ICO** requirements.

---

## Table of contents

1. [Why deterministic, not ML](#why-deterministic-not-ml)
2. [Results on the sample dataset](#results-on-the-sample-dataset)
3. [Architecture](#architecture)
4. [Repository structure](#repository-structure)
5. [Quickstart](#quickstart)
6. [Rule engine](#rule-engine)
7. [Database design](#database-design)
8. [SQL query catalogue](#sql-query-catalogue)
9. [Dashboard](#dashboard)
10. [Testing & QA](#testing--qa)
11. [Design justification](#design-justification)
12. [Known limitations](#known-limitations)

---

## Why deterministic, not ML

| Requirement | How this project meets it |
|---|---|
| **Auditability & explainability** | Every score links to named `rule_id`s and the exact trigger phrase in `rule_matches`. |
| **Zero hallucination / drift** | Pure regex + keyword matching. Same input always gives the same output. |
| **Instant governance changes** | Edit `config/rules_config.json` or `config/pii_patterns.json`. No retraining, no redeploy. |

## Results on the sample dataset

Run against the supplied 5,000-row `uk_customer_interactions_raw.csv`:

| Metric | Value |
|---|---|
| Source rows staged | 5,000 |
| Clean interactions loaded | 4,715 |
| Rejected (invalid / missing timestamp) | 139 |
| Duplicates skipped (same `interaction_id`) | 146 |
| Risk tiers (Low / Medium / High) | 3,722 / 828 / 165 |
| Flags routed to audit queues | 993 |
| **PII records found by SQL-11 assurance check** | **0** |
| Re-run loads (idempotency) | 0 new rows |
| Tests / coverage | 70 passing / 95% |

## Architecture

```
CSV / raw logs
     │
     ▼
1. Ingestion        pandas reads source file
     │
     ▼
2. PII Masking Gate src/pii_scrubber.py  (NI, card, phone, email, postcode)
     │               masking happens BEFORE any DB write
     ▼
3. Taxonomy Match   src/rule_classifier.py  ← config/rules_config.json
     │
     ▼
4. Risk Scoring     Base weight + modifiers (+ repeat-escalation post-load pass)
     │
     ▼
5. SQL Warehouse    src/etl_pipeline.py → SQLite / PostgreSQL (SQLAlchemy)
     │
     ▼
6. BI & Auditing    SQL views + sql/01..12 → dashboard/app.py (Streamlit)
```

**Scoring formula**

```
Risk Score = Base Category Weight (highest-weight matched category)
           + SUM(modifiers: Regulator +30, Cancellation +20, Negative sentiment +10)
           + Repeat Escalation (+15 when 3rd+ flagged interaction in 7 days)
```

| Score | Tier | Action |
|---|---|---|
| 0 – 30 | Low | Automated pass |
| 31 – 60 | Medium | Priority 2, 48h SLA, sample audit |
| 61+ | High | Priority 1, < 2h SLA, immediate escalation |

## Repository structure

```
├── config/
│   ├── pii_patterns.json        UK PII regex patterns (editable)
│   └── rules_config.json        Taxonomy, modifiers, tier boundaries
├── data/
│   └── uk_customer_interactions_raw.csv
├── src/
│   ├── pii_scrubber.py          PII masking engine
│   ├── rule_classifier.py       Intent classification + scoring
│   ├── etl_pipeline.py          Ingest → mask → classify → load (entry point)
│   ├── db.py                    Engine factory + schema bootstrap
│   └── run_sql.py               Run any sql/*.sql file and print results
├── sql/
│   ├── 00_schema_ddl.sql        Tables, indexes, views (single script)
│   └── 01_… to 12_…             The 12 analytical queries
├── tests/                       70 PyTest tests
├── dashboard/app.py             Streamlit dashboard
└── .github/workflows/ci.yml     CI: pipeline + PII check + tests
```

## Quickstart

```bash
git clone https://github.com/<your-username>/compliance-audit-engine.git
cd compliance-audit-engine
pip install -r requirements.txt

# 1. Build schema + load data (idempotent: safe to re-run)
python src/etl_pipeline.py

# 2. Run any analytical query
python src/run_sql.py sql/01_daily_compliance_breach_summary.sql
python src/run_sql.py sql/11_pii_assurance_check.sql      # expect: 0 row(s)

# 3. Launch the dashboard
streamlit run dashboard/app.py

# 4. Run the tests
pytest tests/ --cov=src --cov-report=term-missing
```

**Using PostgreSQL 15+ instead of SQLite**

```bash
pip install psycopg2-binary
export DATABASE_URL="postgresql+psycopg2://user:pass@host:5432/dbname"
python src/etl_pipeline.py
```

`00_schema_ddl.sql` runs unmodified on both engines. A few analytical queries use one SQLite date function each; the Postgres equivalent is noted in each file's header.

## Rule engine

### PII masking (`config/pii_patterns.json`)

| Entity | Token |
|---|---|
| National Insurance number | `[REDACTED_NI_NUMBER]` |
| Card number (13–19 digits) | `[REDACTED_CARD]` |
| UK phone | `[REDACTED_PHONE]` |
| Email | `[REDACTED_EMAIL]` |
| UK postcode | `[REDACTED_POSTCODE]` |

### Intent taxonomy (`config/rules_config.json`)

| Category | Mandate | Base weight |
|---|---|---|
| Formal Complaint | FCA DISP / Consumer Duty | 50 |
| Vulnerable / Hardship | FCA Consumer Duty (FG21/1) | 40 |
| DSAR / Data Privacy | UK GDPR / ICO | 35 |
| Billing & Direct Debit | Payment Services Directive | 20 |
| General Enquiry (fallback) | n/a | 0 |

## Database design

```mermaid
erDiagram
    stg_raw_interactions {
        int stg_row_id PK
        string interaction_id
        string masked_text
        bool timestamp_valid
        bool text_blank
    }
    interactions ||--o{ rule_matches : "fired"
    interactions ||--|| risk_scores : "scored"
    interactions ||--o| flag_actions : "flagged"
    rules ||--o{ rule_matches : "defines"
    rules ||--o{ risk_scores : "primary rule"
    interactions {
        string interaction_id PK
        string customer_id
        timestamp interaction_ts
        string channel
        string region
        string agent_id
        text masked_text
    }
    rules {
        string rule_id PK
        string rule_type
        int weight
        bool active
    }
    rule_matches {
        string interaction_id PK,FK
        string rule_id PK,FK
        string trigger_phrase
        int score_contribution
    }
    risk_scores {
        string interaction_id PK,FK
        string primary_category
        int total_score
        string risk_tier
    }
    flag_actions {
        string interaction_id PK,FK
        string priority
        timestamp flagged_ts
        bool reviewed
        timestamp action_ts
    }
```

**Views:** `v_daily_compliance_summary` (SQL-01), `v_auditor_work_queue` (SQL-05), `v_rule_effectiveness` (SQL-06).

## SQL query catalogue

| Ref | File | Business question | Key techniques |
|---|---|---|---|
| SQL-01 | `01_daily_compliance_breach_summary.sql` | Daily breaches by channel and intent | CTE, conditional aggregation, `NULLIF` |
| SQL-02 | `02_channel_region_high_risk_ranking.sql` | Worst channel/region per day | `RANK`, `DENSE_RANK`, `PARTITION BY` |
| SQL-03 | `03_repeat_escalation_detection.sql` | >2 flags in a rolling 7 days | Self-join, `HAVING` |
| SQL-04 | `04_sla_compliance_tracking.sql` | Weekly SLA compliance (P1 2h / P2 48h) | Timestamp arithmetic, `CASE` |
| SQL-05 | `05_auditor_work_queue.sql` | Prioritised unreviewed flags | Multi-key `ORDER BY`, `LIMIT`, EXPLAIN plan |
| SQL-06 | `06_rule_effectiveness_report.sql` | Rule fire counts, points share, never-fired rules | `LEFT JOIN`, window `SUM` |
| SQL-07 | `07_vulnerability_trend.sql` | 7-day rolling vulnerability average | Recursive date spine, `ROWS BETWEEN`, `LAG` |
| SQL-08 | `08_intent_crosstab_tier_region.sql` | Intent × tier × region pivot | Pivot-style aggregation |
| SQL-09 | `09_agent_exposure_report.sql` | Agents above company high-risk average | `HAVING`, subquery |
| SQL-10 | `10_data_quality_report.sql` | Channel variants, blanks, bad timestamps, duplicates | String normalisation, `UNION ALL` |
| SQL-11 | `11_pii_assurance_check.sql` | Any stored text still matching PII (expect 0 rows) | Regex operator, `UNION ALL` |
| SQL-12 | `12_customer_risk_trajectory.sql` | Customers whose risk worsened | `ROW_NUMBER`, `FIRST_VALUE` |

Every script opens with its business question, assumptions and SQL features used. No `SELECT *` in any of them.

## Dashboard

`streamlit run dashboard/app.py` opens four tabs:

- **Executive Summary**: KPI tiles, vulnerability trend (SQL-07), intent mix, daily breach summary (SQL-01 view)
- **Auditor Work Queue**: unreviewed flags (SQL-05 view) and SLA tracking (SQL-04)
- **Rule Effectiveness**: SQL-06 view, never-fired rule warning, agent exposure (SQL-09)
- **Data Quality**: SQL-10 report and live SQL-11 PII assurance banner

Every metric is sourced from a numbered SQL query or a view. Nothing is computed separately in Python.

## Testing & QA

```bash
pytest tests/ --cov=src --cov-report=term-missing
```

| Test file | Covers |
|---|---|
| `test_pii_scrubber.py` | All 5 entity types, multi-entity text, empty/None/non-English/SQL-injection/multi-KB inputs, regression cases |
| `test_rule_classifier.py` | Exact rule IDs and score maths, tier boundaries, brief's worked example (80 pts, High) |
| `test_repeat_escalation.py` | Synthetic proof of the +15 modifier, re-tiering, 7-day window edges, idempotency |
| `test_etl_pipeline.py` | Timestamp parsing (UK format and epoch), 11 channel variants, dedup precedence, idempotent re-run |

CI (`.github/workflows/ci.yml`) runs the pipeline, the SQL-11 PII check, and the tests with `--cov-fail-under=90` on every push.

## Design justification

**Two-layer load.** `stg_raw_interactions` keeps one row per source row (duplicates, blanks, bad timestamps) so SQL-10 has real quality signal. `interactions` is the deduplicated, canonicalised fact table. Both layers store **masked text only**: masking runs in Python before any `INSERT`, so raw PII never touches disk.

**Portable keys.** No `AUTOINCREMENT`/`SERIAL`. Keys are natural (`interaction_id`), composite (`interaction_id, rule_id`) or application-assigned, so one DDL script runs on SQLite and PostgreSQL.

**Primary category = highest weight, not summed.** Overlapping categories (e.g. "cannot afford" + "overcharged") would otherwise stack uncapped. All matches are still logged in `rule_matches`, so audit lineage and SQL-06 lose nothing.

**Two regex defects found in the brief's own patterns, fixed and regression-tested.**
1. The NI pattern excludes the letter `Q`, so it missed the brief's own example `QQ123456C`. The letter class was widened.
2. The card pattern capped at 16 digits with a trailing `\b`, which can never match a longer token. 36 nineteen-digit card numbers in the sample leaked until SQL-11 caught them. The bound is now 19.

Both fixes err towards over-masking, in line with the 0% PII leakage KPI.

**Repeat escalation.** Implemented as a post-load pass with the same rolling-window logic as SQL-03. The sample data spaces each repeat customer's contacts weeks apart, so it never fires on this file (SQL-03 correctly returns 0 rows). The tests prove it works on clustered synthetic data.

**Data cleaning.** 11 channel spellings map to `call` / `email` / `webchat` / `unknown`. Timestamps accepted in `DD/MM/YYYY HH:MM` and 10-digit Unix epoch. Unparseable timestamps are rejected from the fact table but kept in staging. Duplicates prefer the row without the `[DUPLICATE SUBMISSION]` marker, then the earliest timestamp.

**Synthetic review lifecycle.** The brief supplies no review or action timestamps, so `flag_actions` is seeded deterministically per `interaction_id` (about 75% reviewed, of which about 80% meet SLA). This is a stand-in for real operational data and is not a claim about actual performance.

## Known limitations

- SLA and work-queue reports use the synthetic `flag_actions` lifecycle above; a real events table would replace it.
- The PostgreSQL path is written to the same ANSI subset and checked manually; CI runs on SQLite.
- Streamlit was used instead of Power BI (permitted by the brief) so the project runs from the command line with no proprietary tooling.
- Rules are keyword-based by design. Paraphrases that avoid every listed phrase will not be flagged; widening coverage is a config change, not a code change.

## Licence

MIT. See [LICENSE](LICENSE).
