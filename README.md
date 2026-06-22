# SnowflakesLearning — GlobalTrader Inc.
### Zero to Master | 95 Tasks | 10 Phases | 30-Day Trial Track

**Learner:** Ereshadul Islam | **Stack:** Snowflake + Cortex AI + dbt | **Target:** SnowPro Core + Senior DE Roles

---

## What This Is

A structured, interview-driven Snowflake learning project built around a fictional company — **GlobalTrader Inc.** — a B2B SaaS sales organization with real sales, finance, compensation, and HR data.

This is **not a tutorial**. Every task produces a real artifact (table, view, policy, pipeline, AI model) that you can demo in an interview. Indian-style: hard questions first, theory embedded in practice, no fluff.

---

## Company: GlobalTrader Inc.

| Detail | Value |
|---|---|
| Business | B2B SaaS — sells data platform licenses |
| Sales Reps | Alex Morgan (NA), Emma Schulz (EMEA), Omar Khalid (MEA), Priya Nair (APAC) + 14 others |
| Anchor Accounts | Meridian Industrial Group, Vantix Aerospace Systems |
| Regions | NORTH_AMERICA, EMEA, APAC, MEA, LATAM |
| Data Size | ~46,000+ SQL-generated rows + 5 CSV files |

---

## Data Files

| File | Description | Rows |
|---|---|---|
| `data/01_generate_data.sql` | Run in Snowflake — creates 8 tables (Accounts, Reps, Opps, Transactions, Compensation, Invoices, FX Rates, HR) | 46,000+ |
| `data/02_marketing_leads.csv` | Dirty data — nulls, bad emails, inconsistent countries, sentinel values | 1,000 |
| `data/03_expense_claims.csv` | Multi-currency expense claims across 9 currencies | 3,000 |
| `data/04_products.csv` | Product catalog with JSON features column | 20 |
| `data/05_quota_assignments.csv` | Sales quota history with multiple versions per rep/quarter (SCD2 practice) | ~700 |
| `data/06_support_tickets.csv` | Unstructured text — used for Cortex AI sentiment/classification | 5,000 |

---

## Architecture

```
SOURCE DATA
    │
    ▼
┌─────────────────────────────┐
│  RAW (Bronze)               │  ← SQL generator + CSV COPY INTO
│  8 tables, 5 CSVs           │
└─────────────┬───────────────┘
              │ Streams + Tasks / MERGE
              ▼
┌─────────────────────────────┐
│  CLEAN (Silver)             │  ← Validated, deduplicated, typed
│  SCD2 history, row hashes   │    + dbt staging models
└─────────────┬───────────────┘
              │ dbt + Dynamic Tables
              ▼
┌─────────────────────────────┐
│  MART (Gold)                │  ← Star schema, aggregations
│  FACT_SALES + DIM_*         │    Materialized Views, Semantic Views
└─────────────┬───────────────┘
              │
    ┌─────────┴─────────┐
    ▼                   ▼
Cortex AI           Governance
Sentiment           Masking Policies
Forecast            Row Access Policies
Anomaly             Tags + Classification
Analyst             ACCOUNT_USAGE Reports
```

---

## Phases

| Phase | Focus | Tasks | Key Concepts |
|---|---|---|---|
| 0 | Foundations | T001–T012 | Architecture, RBAC, Caching, Time Travel, Stages, COPY INTO, Micro-partitions |
| 1 | Silver Layer | T013–T020 | Data profiling, Cleaning, Dedup, MERGE, Window functions, QUALIFY |
| 2 | Streaming & CDC | T021–T030 | Streams, Tasks, DAGs, Snowpipe, SCD2, Dynamic Tables, Stored Procedures, UDFs |
| 3 | Gold Layer | T031–T040 | Dimensional modeling, Fact/Dim tables, MViews, PIVOT, ROLLUP, Set operations |
| 4 | Performance & Cost | T041–T046 | Warehouse sizing, Resource monitors, Clustering, Query optimization, ACCOUNT_USAGE |
| 5 | Governance | T047–T054 | Masking policies, Row access, Tags, Audit logs, Secure views, Data sharing, Time Travel compliance |
| 6 | Cortex AI | T055–T065 | Sentiment, Summarize, Classify, Extract, Search, Analyst, Forecast, Anomaly Detection, Document AI |
| 7 | Snowpark + dbt | T066–T075 | Snowpark DataFrames, Python SPs, UDTFs, dbt sources/staging/mart/tests/incremental/snapshots, Semantic Views, Cortex Code |
| 8 | Interview Gauntlet | T076–T085 | Scenario questions, live coding, system design, platform comparisons |
| 9 | SnowPro Core Prep | T086–T090 | Certification domains, mock exam, weak area review |
| 10 | Capstone | T091–T095 | End-to-end platform, demo script, GitHub cleanup |

---

## Setup (Day 1)

```
1. Sign up: trial.snowflake.com → AWS us-east-1 → Enterprise
2. Complete T001 (architecture overview)
3. Complete T002 (create warehouse + databases)
4. Run data/01_generate_data.sql in Snowsight worksheet
5. Upload CSVs via Snowsight: Data > Add Data > Load files into a stage
6. Complete T006 (load CSVs)
7. You're ready — work through tasks in order
```

**Trial clock:** You have 30 days and $400 in credits. X-Small warehouse = 1 credit/hour = ~$2.
Keep warehouse AUTO_SUSPEND = 60. Don't leave it running overnight.

---

## Interview Quick Reference

The most-tested topics in senior Snowflake interviews:

1. **Micro-partitions** — what they are, how pruning works, why you can't manually control them
2. **Streams + Tasks** — offset consumption, WHEN clause, DAG ordering, failure behavior
3. **SCD Type 2** — MERGE pattern for expire + insert
4. **Caching** — all 3 layers, what invalidates each, result cache sharing scope
5. **Clustering keys** — depth vs overlap, good vs bad columns, reclustering cost
6. **MERGE** — hash-based change detection, WHEN MATCHED + NOT MATCHED
7. **Snowpipe vs COPY INTO** — 7-day dedup window, when NOT to use Snowpipe
8. **Masking + Row Access Policies** — role-based masking, can you combine both?
9. **Time Travel vs Fail-safe** — who controls each, can you access Fail-safe?
10. **Dynamic Tables vs Streams+Tasks** — TARGET_LAG, when to use each

---

## Capstone Deliverables

When complete, your repo will contain:
- All phase SQL organized in folders
- `TASKS.md` — task checklist
- `CAPSTONE.md` — architecture + business questions answered
- `DEMO_SCRIPT.md` — 10-minute interview demo guide
- Working dbt project in `globaltrader_dbt/` folder

---

*Built by Ereshadul Islam | Cacacumo Inc. | Oklahoma City, OK*
