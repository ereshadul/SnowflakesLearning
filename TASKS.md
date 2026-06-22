# SnowflakesLearning — Task List
## GlobalTrader Inc. | Zero to Master | 30-Day Trial Track
**Total Tasks: 95 | Phases: 0–10 | Repo: ereshadul/SnowflakesLearning**

> **Indian-style learning:** Every task builds on the previous. Interview questions are embedded directly into tasks — you answer them in SQL/config before moving on. Hard concepts first, then basics used to confuse you in interviews. No fluff.

---

## Phase 0 — Foundations & Architecture (Tasks T001–T012)
> **Goal:** Understand Snowflake from the inside out. These basics trip up 80% of candidates who learned "on the job."

---

### T001 — Trial Account Setup & Architecture Deep Dive
**Do:**
1. Sign up at trial.snowflake.com — choose AWS us-east-1, Enterprise edition
2. Log in to Snowsight (the web UI)
3. Draw (on paper or Miro) Snowflake's 3-layer architecture: Cloud Services, Compute (Virtual Warehouses), Storage (micro-partitions)
4. Answer in a SQL comment block: *What is a micro-partition? How big is it? Why can't you manually control it?*
5. Answer: *How is Snowflake different from Redshift? From Databricks?*

**Done-when:** You can explain micro-partitions, pruning, and the 3-layer architecture verbally in 2 minutes without notes.

---

### T002 — Warehouse, Database, Schema Setup
**Do:**
```sql
-- Create your compute layer
CREATE WAREHOUSE GLOBALTRADER_WH
  WAREHOUSE_SIZE = 'X-SMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  COMMENT = 'Primary learning warehouse';

-- Create database + schemas (medallion architecture)
CREATE DATABASE GLOBALTRADER_DB;
CREATE SCHEMA GLOBALTRADER_DB.RAW;        -- Bronze: raw ingestion
CREATE SCHEMA GLOBALTRADER_DB.CLEAN;      -- Silver: validated, deduplicated
CREATE SCHEMA GLOBALTRADER_DB.MART;       -- Gold: business-ready models
CREATE SCHEMA GLOBALTRADER_DB.STAGING;    -- External stage objects
CREATE SCHEMA GLOBALTRADER_DB.AUDIT;      -- Governance & audit logs
```
2. Answer in comments: *What is AUTO_SUSPEND? What happens to running queries when warehouse suspends? What is the billing unit?*

**Done-when:** All objects exist, you can explain per-second billing and credit consumption.

---

### T003 — Roles & RBAC (Role-Based Access Control)
**Do:**
```sql
-- Create custom roles
CREATE ROLE GLOBALTRADER_ADMIN;
CREATE ROLE GLOBALTRADER_ENGINEER;
CREATE ROLE GLOBALTRADER_ANALYST;
CREATE ROLE GLOBALTRADER_READONLY;

-- Build role hierarchy
GRANT ROLE GLOBALTRADER_ENGINEER TO ROLE GLOBALTRADER_ADMIN;
GRANT ROLE GLOBALTRADER_ANALYST   TO ROLE GLOBALTRADER_ENGINEER;
GRANT ROLE GLOBALTRADER_READONLY  TO ROLE GLOBALTRADER_ANALYST;

-- Grant privileges
GRANT USAGE ON WAREHOUSE GLOBALTRADER_WH    TO ROLE GLOBALTRADER_ANALYST;
GRANT USAGE ON DATABASE  GLOBALTRADER_DB    TO ROLE GLOBALTRADER_ANALYST;
GRANT USAGE ON SCHEMA    GLOBALTRADER_DB.MART TO ROLE GLOBALTRADER_ANALYST;
GRANT SELECT ON ALL TABLES IN SCHEMA GLOBALTRADER_DB.MART TO ROLE GLOBALTRADER_ANALYST;
```
2. Interview question to answer: *What is the difference between GRANT USAGE and GRANT SELECT? What happens if you grant SELECT but not USAGE on the schema?*

**Done-when:** Role hierarchy exists, you can explain privilege inheritance.

---

### T004 — Data Types & Table Types
**Do:**
1. Create one example of each table type:
```sql
-- Permanent (default, Time Travel + Fail-safe)
CREATE TABLE GLOBALTRADER_DB.RAW.TEST_PERMANENT (id INT, val VARCHAR);

-- Temporary (session-scoped, no Fail-safe)
CREATE TEMPORARY TABLE TEST_TEMP (id INT, val VARCHAR);

-- Transient (no Fail-safe, lower cost)
CREATE TRANSIENT TABLE GLOBALTRADER_DB.STAGING.TEST_TRANSIENT (id INT, val VARCHAR);

-- External (points to S3/Azure, no data stored in Snowflake)
-- (explain only — no actual S3 needed yet)
```
2. Answer: *When would you use TRANSIENT vs PERMANENT? What is Fail-safe and who controls it?*
3. Answer: *What is a VARIANT column? What is the 16MB limit and what do you do when JSON exceeds it?*

**Done-when:** You can explain all 4 table types, Fail-safe, and VARIANT in an interview context.

---

### T005 — Load the Master Data (SQL Generator)
**Do:**
1. Open `data/01_generate_data.sql` from this repo
2. Run it in your Snowflake worksheet (paste section by section)
3. Verify row counts:
```sql
SELECT TABLE_NAME, ROW_COUNT
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'RAW'
ORDER BY TABLE_NAME;
```
4. Expected: ACCOUNTS=500, SALES_REPS=21, OPPORTUNITIES=5000, SALES_TRANSACTIONS=20000, COMPENSATION=~1680, INVOICES=15000, FX_RATES=~3285, HR_HEADCOUNT=2000

**Done-when:** All 8 tables loaded, row counts verified, no errors.

---

### T006 — Internal Stages & CSV Loading
**Do:**
```sql
-- Create internal named stage
CREATE STAGE GLOBALTRADER_DB.STAGING.INTERNAL_STAGE
  FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1 NULL_IF = ('NULL','null',''));

-- Upload CSVs via Snowsight: Data > Add Data > Load files into a stage
-- Upload: 02_marketing_leads.csv, 03_expense_claims.csv, 05_quota_assignments.csv, 06_support_tickets.csv

-- List stage contents
LIST @GLOBALTRADER_DB.STAGING.INTERNAL_STAGE;

-- Inspect a file before loading
SELECT $1,$2,$3,$4,$5 FROM @GLOBALTRADER_DB.STAGING.INTERNAL_STAGE/02_marketing_leads.csv LIMIT 10;
```
2. Load marketing leads:
```sql
CREATE TABLE GLOBALTRADER_DB.RAW.MARKETING_LEADS_RAW LIKE (
  SELECT $1::VARCHAR lead_id, $2::VARCHAR first_name, $3::VARCHAR last_name,
         $4::VARCHAR email, $5::VARCHAR phone, $6::VARCHAR company, $7::VARCHAR title,
         $8::VARCHAR industry, $9::VARCHAR country, $10::VARCHAR lead_source,
         $11::VARCHAR created_date_raw, $12::VARCHAR lead_score_raw,
         $13::VARCHAR annual_budget_raw, $14::VARCHAR interested_product, $15::VARCHAR status
  FROM @GLOBALTRADER_DB.STAGING.INTERNAL_STAGE/02_marketing_leads.csv
);
-- Then COPY INTO
```
3. Answer: *What is the difference between internal named stage, table stage, and user stage?*
4. Answer: *Can you reload a file with COPY INTO? What is FORCE=TRUE and when would you use it?*

**Done-when:** All 4 CSVs loaded to RAW schema, LIST shows files, you can answer stage questions.

---

### T007 — Semi-Structured Data & VARIANT
**Do:**
1. Create a table with a VARIANT column and load the products JSON features:
```sql
CREATE TABLE GLOBALTRADER_DB.RAW.PRODUCTS_RAW (
  product_id    VARCHAR,
  product_name  VARCHAR,
  sku           VARCHAR,
  category      VARCHAR,
  list_price_usd NUMBER,
  features      VARIANT,
  supported_regions VARCHAR
);

-- Load from CSV (features_json column is the VARIANT source)
-- After loading, query the VARIANT:
SELECT
  product_id,
  features:sla::VARCHAR           AS sla_target,
  features:users::INT             AS max_users,
  features:modules[0]::VARCHAR    AS first_module
FROM GLOBALTRADER_DB.RAW.PRODUCTS_RAW;
```
2. Practice FLATTEN:
```sql
SELECT p.product_id, m.value::VARCHAR AS module
FROM GLOBALTRADER_DB.RAW.PRODUCTS_RAW p,
LATERAL FLATTEN(INPUT => p.features:modules) m;
```
3. Answer: *What is the colon (:) notation in Snowflake? What is PARSE_JSON()? What happens when you query a missing key?*

**Done-when:** VARIANT queries return data, FLATTEN works, you understand dot/colon path notation.

---

### T008 — Time Travel & Zero-Copy Cloning
**Do:**
```sql
-- Enable 7-day Time Travel on key tables
ALTER TABLE GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS SET DATA_RETENTION_TIME_IN_DAYS = 7;

-- Simulate an accidental delete
DELETE FROM GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS WHERE STATUS = 'VOIDED';
-- Count: should be ~2,000 rows deleted

-- Recover with Time Travel
CREATE TABLE GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS_RECOVERED AS
SELECT * FROM GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS
AT(OFFSET => -300);  -- 5 minutes ago

-- Verify recovery
SELECT COUNT(*) FROM GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS_RECOVERED WHERE STATUS = 'VOIDED';

-- Zero-Copy Clone for dev/test
CREATE DATABASE GLOBALTRADER_DEV CLONE GLOBALTRADER_DB;
-- This is instant — explain WHY it is instant
```
2. Answer: *What is the difference between Time Travel and Fail-safe? Who controls Fail-safe? Can you query Fail-safe data yourself?*
3. Answer: *How does Zero-Copy Clone work at the storage layer? When does storage cost increase after cloning?*

**Done-when:** Recovery works, clone created instantly, you can explain both concepts mechanically.

---

### T009 — Caching Deep Dive (Most Misunderstood Topic)
**Do:**
1. Run a complex query on SALES_TRANSACTIONS, note the time
2. Run the EXACT same query again — observe near-instant result. This is Result Cache.
3. Run a slightly modified version (add a LIMIT) — Result Cache won't apply. Observe disk cache effect.
```sql
-- Force cache bypass to see real query time
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
SELECT COUNT(*), SUM(TOTAL_AMOUNT_USD), AVG(DISCOUNT_PCT)
FROM GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS
WHERE TRANSACTION_DATE >= '2024-01-01';

-- Re-enable and run again
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
-- same query — should be instant
```
4. Answer in comments: *Name all 3 cache layers. What invalidates each? Who shares result cache — is it per-user or per-warehouse?*

**Done-when:** You can name all 3 cache types, their invalidation conditions, and explain result cache sharing rules.

---

### T010 — Query Profile & EXPLAIN
**Do:**
1. Run a multi-table join query:
```sql
SELECT
  a.ACCOUNT_NAME, a.INDUSTRY, a.TIER,
  COUNT(t.TRANSACTION_ID)      AS TOTAL_TRANSACTIONS,
  SUM(t.TOTAL_AMOUNT_USD)      AS TOTAL_REVENUE,
  AVG(t.DISCOUNT_PCT)          AS AVG_DISCOUNT
FROM GLOBALTRADER_DB.RAW.ACCOUNTS a
JOIN GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS t ON a.ACCOUNT_ID = t.ACCOUNT_ID
JOIN GLOBALTRADER_DB.RAW.INVOICES i           ON t.TRANSACTION_ID = i.TRANSACTION_ID
WHERE a.STATUS = 'ACTIVE'
GROUP BY 1,2,3
ORDER BY TOTAL_REVENUE DESC;
```
2. Click "Query Profile" in Snowsight for this query
3. Identify: TableScan nodes, partitions scanned vs total, any explode nodes
4. Run EXPLAIN:
```sql
EXPLAIN USING TEXT
SELECT ... (same query above)
```
5. Answer: *What does "partitions pruned" mean? What does it tell you about clustering? What is a "spill to disk" warning and why is it bad?*

**Done-when:** You can read a Query Profile, identify bottlenecks, and explain pruning.

---

### T011 — COPY INTO Options & Load Validation
**Do:**
```sql
-- Load expense claims with validation options
COPY INTO GLOBALTRADER_DB.RAW.EXPENSE_CLAIMS
FROM @GLOBALTRADER_DB.STAGING.INTERNAL_STAGE/03_expense_claims.csv
FILE_FORMAT = (TYPE=CSV FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=1 NULL_IF=('','NULL'))
ON_ERROR = 'CONTINUE'     -- don't abort on bad rows
PURGE = FALSE;            -- keep file in stage

-- Check what failed
SELECT * FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME => 'EXPENSE_CLAIMS',
  START_TIME => DATEADD(HOURS,-1,CURRENT_TIMESTAMP())
));
```
2. Answer: *What is the difference between ON_ERROR = CONTINUE vs ABORT_STATEMENT vs SKIP_FILE? When would you use each in production?*
3. Answer: *What does PURGE=TRUE do? Is it reversible?*

**Done-when:** Expense claims loaded, copy history queried, ON_ERROR options understood.

---

### T012 — Micro-Partitions & Clustering Fundamentals
**Do:**
```sql
-- Check partition metadata
SELECT SYSTEM$CLUSTERING_INFORMATION('GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS');
SELECT SYSTEM$CLUSTERING_INFORMATION('GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS', '(TRANSACTION_DATE)');
SELECT SYSTEM$CLUSTERING_INFORMATION('GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS', '(REP_ID)');

-- Compare pruning with vs without clustering
-- Query 1: filter on TRANSACTION_DATE (natural order — good pruning)
SELECT COUNT(*), SUM(TOTAL_AMOUNT_USD)
FROM GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS
WHERE TRANSACTION_DATE BETWEEN '2024-01-01' AND '2024-03-31';

-- Query 2: filter on STATUS (random — poor pruning)
SELECT COUNT(*), SUM(TOTAL_AMOUNT_USD)
FROM GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS
WHERE STATUS = 'POSTED';

-- Check Query Profile for each — compare partitions scanned
```
2. Answer: *When should you define a clustering key? What is the cost of automatic reclustering? What columns make bad clustering keys?*

**Done-when:** You can read CLUSTERING_INFORMATION output and explain the overlap/depth metrics.

---

## Phase 1 — Data Cleaning & Silver Layer (Tasks T013–T020)
> **Goal:** Transform dirty raw data into validated, typed, deduplicated Silver layer tables.

---

### T013 — Profile the Dirty Data
**Do:**
```sql
-- Profile marketing leads for data quality issues
SELECT
  COUNT(*)                                                          AS total_rows,
  COUNT(email)                                                      AS has_email,
  COUNT(CASE WHEN email NOT LIKE '%@%.%' THEN 1 END)               AS invalid_email,
  COUNT(CASE WHEN TRY_CAST(lead_score_raw AS INT) IS NULL THEN 1 END) AS non_numeric_score,
  COUNT(CASE WHEN country IN ('usa','US','U.S.','United States') THEN 1 END) AS country_variants,
  COUNT(CASE WHEN annual_budget_raw::VARCHAR = '-999' THEN 1 END)  AS sentinel_negatives,
  COUNT(DISTINCT country)                                           AS distinct_countries,
  COUNT(status)                                                     AS has_status,
  COUNT(*) - COUNT(status)                                         AS null_status
FROM GLOBALTRADER_DB.RAW.MARKETING_LEADS_RAW;
```
2. Do same profiling for INVOICES (find DISPUTED rows, null PAYMENT_DATE patterns)
3. Document: list every data quality issue found

**Done-when:** You have a written list of all data quality issues per table.

---

### T014 — Build CLEAN.ACCOUNTS
**Do:**
```sql
CREATE TABLE GLOBALTRADER_DB.CLEAN.ACCOUNTS AS
SELECT
  ACCOUNT_ID,
  TRIM(UPPER(ACCOUNT_NAME))                                        AS ACCOUNT_NAME,
  INITCAP(INDUSTRY)                                                 AS INDUSTRY,
  INITCAP(COUNTRY)                                                  AS COUNTRY,
  UPPER(REGION)                                                     AS REGION,
  ANNUAL_REVENUE_USD,
  EMPLOYEE_COUNT,
  UPPER(TIER)                                                       AS TIER,
  UPPER(STATUS)                                                     AS STATUS,
  CREATED_DATE,
  LAST_ACTIVITY_DATE,
  ACCOUNT_OWNER_ID,
  CURRENCY_CODE,
  -- Audit columns
  CURRENT_TIMESTAMP()                                               AS _LOADED_AT,
  'RAW.ACCOUNTS'                                                    AS _SOURCE_TABLE,
  MD5(ACCOUNT_ID || ACCOUNT_NAME || STATUS)                        AS _ROW_HASH
FROM GLOBALTRADER_DB.RAW.ACCOUNTS
WHERE ACCOUNT_ID IS NOT NULL;
```
2. Add a primary key constraint (informational only in Snowflake):
```sql
ALTER TABLE GLOBALTRADER_DB.CLEAN.ACCOUNTS ADD CONSTRAINT PK_ACCOUNTS PRIMARY KEY (ACCOUNT_ID) RELY NOVALIDATE;
```
3. Answer: *What does RELY NOVALIDATE mean? Does Snowflake actually enforce primary keys?*

**Done-when:** CLEAN.ACCOUNTS exists with audit columns, hash column, and constraint.

---

### T015 — Build CLEAN.MARKETING_LEADS (Data Standardization)
**Do:**
```sql
CREATE TABLE GLOBALTRADER_DB.CLEAN.MARKETING_LEADS AS
SELECT
  LEAD_ID,
  TRIM(FIRST_NAME)                                                  AS FIRST_NAME,
  TRIM(LAST_NAME)                                                   AS LAST_NAME,
  LOWER(TRIM(EMAIL))                                                AS EMAIL,
  CASE WHEN EMAIL LIKE '%@%.%' THEN TRUE ELSE FALSE END             AS EMAIL_VALID,
  PHONE,
  TRIM(COMPANY)                                                     AS COMPANY,
  TRIM(TITLE)                                                       AS TITLE,
  INITCAP(INDUSTRY)                                                 AS INDUSTRY,
  -- Standardize country variants
  CASE UPPER(TRIM(COUNTRY))
    WHEN 'US' THEN 'United States'   WHEN 'USA' THEN 'United States'
    WHEN 'U.S.' THEN 'United States' WHEN 'UK'  THEN 'United Kingdom'
    WHEN 'UNKNOWN' THEN NULL         ELSE INITCAP(TRIM(COUNTRY))
  END                                                               AS COUNTRY,
  LEAD_SOURCE,
  -- Handle mixed date formats
  TRY_TO_DATE(CREATED_DATE_RAW, 'YYYY-MM-DD')                      AS CREATED_DATE_ISO,
  TRY_TO_DATE(CREATED_DATE_RAW, 'MM/DD/YYYY')                      AS CREATED_DATE_US,
  COALESCE(TRY_TO_DATE(CREATED_DATE_RAW, 'YYYY-MM-DD'),
           TRY_TO_DATE(CREATED_DATE_RAW, 'MM/DD/YYYY'))             AS CREATED_DATE,
  TRY_CAST(LEAD_SCORE_RAW AS INT)                                   AS LEAD_SCORE,
  -- Replace sentinel values
  CASE WHEN TRY_CAST(ANNUAL_BUDGET_RAW AS NUMBER) < 0 THEN NULL
       ELSE TRY_CAST(ANNUAL_BUDGET_RAW AS NUMBER) END               AS ANNUAL_BUDGET_USD,
  INTERESTED_PRODUCT,
  UPPER(COALESCE(STATUS, 'UNKNOWN'))                                AS STATUS,
  CURRENT_TIMESTAMP()                                               AS _LOADED_AT,
  MD5(LEAD_ID || COALESCE(EMAIL,'') || COALESCE(COMPANY,''))       AS _ROW_HASH
FROM GLOBALTRADER_DB.RAW.MARKETING_LEADS_RAW;
```

**Done-when:** CLEAN.MARKETING_LEADS has standardized country, typed dates, no sentinels, email validation flag.

---

### T016 — Deduplication Logic
**Do:**
```sql
-- Find duplicates in quota assignments (same rep, same quarter, multiple versions)
WITH DUPES AS (
  SELECT
    rep_id, fiscal_quarter, fiscal_year,
    COUNT(*) AS version_count,
    MAX(version) AS latest_version
  FROM GLOBALTRADER_DB.RAW.QUOTA_ASSIGNMENTS_RAW
  GROUP BY 1,2,3
  HAVING COUNT(*) > 1
)
SELECT * FROM DUPES;

-- Keep only latest version using ROW_NUMBER
CREATE TABLE GLOBALTRADER_DB.CLEAN.QUOTA_ASSIGNMENTS AS
WITH RANKED AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY REP_ID, FISCAL_QUARTER, FISCAL_YEAR
      ORDER BY VERSION DESC, EFFECTIVE_DATE DESC
    ) AS RN
  FROM GLOBALTRADER_DB.RAW.QUOTA_ASSIGNMENTS_RAW
)
SELECT * EXCLUDE (RN) FROM RANKED WHERE RN = 1;
```
2. Answer: *What is the difference between ROW_NUMBER, RANK, and DENSE_RANK? Write an example where RANK and DENSE_RANK produce different results.*

**Done-when:** Deduplication logic works, you can explain all three window functions.

---

### T017 — Build CLEAN.SALES_TRANSACTIONS (with derived columns)
**Do:**
```sql
CREATE TABLE GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS AS
SELECT
  t.*,
  -- Convert all amounts to USD using FX_RATES table
  COALESCE(
    t.NET_AMOUNT_USD,
    t.GROSS_AMOUNT_USD * (1 - t.DISCOUNT_PCT/100)
  )                                                                 AS NET_AMOUNT_CALC_USD,
  f.RATE                                                            AS FX_RATE_USED,
  -- Fiscal period derivation
  CASE CEIL(MONTH(t.TRANSACTION_DATE)/3)
    WHEN 1 THEN 'Q1' WHEN 2 THEN 'Q2' WHEN 3 THEN 'Q3' ELSE 'Q4'
  END                                                               AS FISCAL_QUARTER_LABEL,
  YEAR(t.TRANSACTION_DATE)                                          AS FISCAL_YEAR,
  -- Late payment flag
  DATEDIFF(DAY, t.TRANSACTION_DATE, COALESCE(i.PAYMENT_DATE, CURRENT_DATE())) AS DAYS_TO_PAYMENT,
  CASE WHEN i.PAYMENT_DATE > DATEADD(DAY,30,t.TRANSACTION_DATE) THEN TRUE ELSE FALSE END AS IS_LATE_PAYMENT,
  CURRENT_TIMESTAMP()                                               AS _LOADED_AT
FROM GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS t
LEFT JOIN GLOBALTRADER_DB.RAW.FX_RATES f
  ON f.RATE_DATE = t.TRANSACTION_DATE AND f.TARGET_CURRENCY = t.CURRENCY_CODE
LEFT JOIN GLOBALTRADER_DB.RAW.INVOICES i
  ON t.TRANSACTION_ID = i.TRANSACTION_ID
WHERE t.STATUS != 'VOIDED';
```

**Done-when:** CLEAN.SALES_TRANSACTIONS includes FX-converted amounts, fiscal labels, late payment flag.

---

### T018 — Window Functions Mastery (Interview Core)
**Do:** Write ALL of the following queries from scratch — no copy-paste:
```sql
-- 1. Running total revenue by rep, ordered by date
SELECT REP_ID, TRANSACTION_DATE, TOTAL_AMOUNT_USD,
  SUM(TOTAL_AMOUNT_USD) OVER (PARTITION BY REP_ID ORDER BY TRANSACTION_DATE
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RUNNING_TOTAL
FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS;

-- 2. Rank accounts by revenue within each industry
SELECT ACCOUNT_ID, INDUSTRY, TOTAL_REVENUE,
  RANK() OVER (PARTITION BY INDUSTRY ORDER BY TOTAL_REVENUE DESC) AS RANK_IN_INDUSTRY
FROM (...);

-- 3. Previous quarter revenue comparison (LAG)
SELECT REP_ID, FISCAL_QUARTER_LABEL, FISCAL_YEAR, SUM(TOTAL_AMOUNT_USD) AS REVENUE,
  LAG(SUM(TOTAL_AMOUNT_USD)) OVER (PARTITION BY REP_ID ORDER BY FISCAL_YEAR, FISCAL_QUARTER_LABEL) AS PREV_QTR_REVENUE
FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
GROUP BY 1,2,3;

-- 4. 30-day moving average of daily revenue
SELECT TRANSACTION_DATE, SUM(TOTAL_AMOUNT_USD) AS DAILY_REV,
  AVG(SUM(TOTAL_AMOUNT_USD)) OVER (ORDER BY TRANSACTION_DATE
    ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS MOVING_AVG_30D
FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
GROUP BY 1;

-- 5. Percentile of each rep's deal size within their region
SELECT REP_ID, TOTAL_AMOUNT_USD,
  NTILE(4) OVER (PARTITION BY ... ORDER BY TOTAL_AMOUNT_USD) AS QUARTILE,
  PERCENT_RANK() OVER (PARTITION BY ... ORDER BY TOTAL_AMOUNT_USD) AS PCT_RANK
FROM ...;
```

**Done-when:** All 5 queries run correctly. You can write any window function from memory.

---

### T019 — QUALIFY Clause & Advanced Filtering
**Do:**
```sql
-- Find each rep's single highest-value transaction (QUALIFY replaces a subquery)
SELECT REP_ID, TRANSACTION_DATE, TOTAL_AMOUNT_USD, PRODUCT_LINE
FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
QUALIFY ROW_NUMBER() OVER (PARTITION BY REP_ID ORDER BY TOTAL_AMOUNT_USD DESC) = 1;

-- Without QUALIFY (the old way — write this too so you understand the difference):
SELECT * FROM (
  SELECT REP_ID, TRANSACTION_DATE, TOTAL_AMOUNT_USD, PRODUCT_LINE,
    ROW_NUMBER() OVER (PARTITION BY REP_ID ORDER BY TOTAL_AMOUNT_USD DESC) AS RN
  FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
) WHERE RN = 1;

-- QUALIFY with HAVING together (tricky interview question):
SELECT REP_ID, PRODUCT_LINE, SUM(TOTAL_AMOUNT_USD) AS TOTAL
FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
GROUP BY 1,2
HAVING TOTAL > 100000
QUALIFY RANK() OVER (PARTITION BY REP_ID ORDER BY TOTAL DESC) <= 3;
```
2. Answer: *What is QUALIFY? What order of clause evaluation makes it work? Can you use QUALIFY without a window function?*

**Done-when:** QUALIFY queries run, you can explain why it's more efficient than a subquery wrapper.

---

### T020 — MERGE Statement (SCD Type 1 & Full Upsert)
**Do:**
```sql
-- Simulate an incremental update batch arriving
CREATE TEMPORARY TABLE ACCOUNTS_UPDATES AS
SELECT ACCOUNT_ID, ACCOUNT_NAME, 'CHURNED' AS STATUS, CURRENT_DATE() AS LAST_ACTIVITY_DATE
FROM GLOBALTRADER_DB.CLEAN.ACCOUNTS
SAMPLE (50 ROWS);

-- Apply MERGE (upsert)
MERGE INTO GLOBALTRADER_DB.CLEAN.ACCOUNTS AS TARGET
USING ACCOUNTS_UPDATES AS SOURCE
ON TARGET.ACCOUNT_ID = SOURCE.ACCOUNT_ID
WHEN MATCHED AND TARGET._ROW_HASH != MD5(SOURCE.ACCOUNT_ID || SOURCE.ACCOUNT_NAME || SOURCE.STATUS)
  THEN UPDATE SET
    TARGET.STATUS = SOURCE.STATUS,
    TARGET.LAST_ACTIVITY_DATE = SOURCE.LAST_ACTIVITY_DATE,
    TARGET._LOADED_AT = CURRENT_TIMESTAMP(),
    TARGET._ROW_HASH = MD5(SOURCE.ACCOUNT_ID || SOURCE.ACCOUNT_NAME || SOURCE.STATUS)
WHEN NOT MATCHED
  THEN INSERT (ACCOUNT_ID, ACCOUNT_NAME, STATUS, LAST_ACTIVITY_DATE, _LOADED_AT, _ROW_HASH)
  VALUES (SOURCE.ACCOUNT_ID, SOURCE.ACCOUNT_NAME, SOURCE.STATUS, SOURCE.LAST_ACTIVITY_DATE,
          CURRENT_TIMESTAMP(), MD5(SOURCE.ACCOUNT_ID || SOURCE.ACCOUNT_NAME || SOURCE.STATUS));
```
2. Answer: *Why do we check _ROW_HASH before updating? What is the risk of MERGE without a hash check on large tables?*

**Done-when:** MERGE runs, hash-based change detection works, you understand the update-only-on-change pattern.

---

## Phase 2 — Streaming & CDC (Tasks T021–T030)
> **Goal:** Build real-time change capture. Streams + Tasks is the #1 most-asked Snowflake topic in senior interviews.

---

### T021 — Snowflake Streams (Change Data Capture)
**Do:**
```sql
-- Create a stream on the accounts table
CREATE STREAM GLOBALTRADER_DB.RAW.ACCOUNTS_STREAM
  ON TABLE GLOBALTRADER_DB.CLEAN.ACCOUNTS
  APPEND_ONLY = FALSE;  -- capture INSERT, UPDATE, DELETE

-- Check stream is empty initially
SELECT * FROM GLOBALTRADER_DB.RAW.ACCOUNTS_STREAM;

-- Make some changes
UPDATE GLOBALTRADER_DB.CLEAN.ACCOUNTS SET STATUS = 'CHURNED' WHERE TIER = 'SMB' LIMIT 10;
INSERT INTO GLOBALTRADER_DB.CLEAN.ACCOUNTS (ACCOUNT_ID, ACCOUNT_NAME, STATUS)
VALUES ('ACC-99999', 'Test Account', 'PROSPECT');
DELETE FROM GLOBALTRADER_DB.CLEAN.ACCOUNTS WHERE ACCOUNT_ID = 'ACC-99999';

-- Now inspect the stream
SELECT METADATA$ACTION, METADATA$ISUPDATE, METADATA$ROW_ID, *
FROM GLOBALTRADER_DB.RAW.ACCOUNTS_STREAM;
```
2. Answer: *What are METADATA$ACTION values? How does an UPDATE appear in a stream (hint: it's two rows)? What happens to the stream offset when you consume it?*
3. Answer: *What is an APPEND_ONLY stream? When would you use it instead of a standard stream?*

**Done-when:** Stream shows INSERT/UPDATE/DELETE records with metadata columns. You understand offset consumption.

---

### T022 — Snowflake Tasks (Scheduled Automation)
**Do:**
```sql
-- Create a task to process the accounts stream every minute
CREATE TASK GLOBALTRADER_DB.RAW.PROCESS_ACCOUNTS_STREAM
  WAREHOUSE = GLOBALTRADER_WH
  SCHEDULE = '1 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('GLOBALTRADER_DB.RAW.ACCOUNTS_STREAM')
AS
MERGE INTO GLOBALTRADER_DB.CLEAN.ACCOUNTS AS TARGET
USING (
  SELECT * FROM GLOBALTRADER_DB.RAW.ACCOUNTS_STREAM
  WHERE METADATA$ACTION = 'INSERT'
) AS SOURCE
ON TARGET.ACCOUNT_ID = SOURCE.ACCOUNT_ID
WHEN MATCHED THEN UPDATE SET TARGET.STATUS = SOURCE.STATUS, TARGET._LOADED_AT = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN INSERT VALUES (SOURCE.ACCOUNT_ID, SOURCE.ACCOUNT_NAME, SOURCE.STATUS, CURRENT_DATE(), CURRENT_DATE(), NULL, NULL, NULL, CURRENT_TIMESTAMP(), 'STREAM', MD5(SOURCE.ACCOUNT_ID));

-- Enable the task
ALTER TASK GLOBALTRADER_DB.RAW.PROCESS_ACCOUNTS_STREAM RESUME;

-- Check task history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
  TASK_NAME => 'PROCESS_ACCOUNTS_STREAM',
  SCHEDULED_TIME_RANGE_START => DATEADD(HOUR,-1,CURRENT_TIMESTAMP())
));
```
2. Answer: *What does WHEN SYSTEM$STREAM_HAS_DATA() do? What happens if the stream is empty and the condition is omitted? Who pays for the task warehouse?*

**Done-when:** Task runs on schedule, stream consumed, task history shows successful runs.

---

### T023 — Task DAGs (Chained Tasks)
**Do:**
```sql
-- Build a 3-step pipeline as a DAG
-- Step 1: Extract from stream
CREATE TASK GLOBALTRADER_DB.RAW.TASK_EXTRACT
  WAREHOUSE = GLOBALTRADER_WH
  SCHEDULE = '5 MINUTE'
AS
INSERT INTO GLOBALTRADER_DB.STAGING.TRANSACTION_STAGING
SELECT * FROM GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS_STREAM WHERE METADATA$ACTION = 'INSERT';

-- Step 2: Transform (child of Step 1)
CREATE TASK GLOBALTRADER_DB.RAW.TASK_TRANSFORM
  WAREHOUSE = GLOBALTRADER_WH
  AFTER GLOBALTRADER_DB.RAW.TASK_EXTRACT
AS
INSERT INTO GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS_INCREMENTAL
SELECT *, CURRENT_TIMESTAMP() AS _LOADED_AT
FROM GLOBALTRADER_DB.STAGING.TRANSACTION_STAGING;

-- Step 3: Aggregate (child of Step 2)
CREATE TASK GLOBALTRADER_DB.RAW.TASK_AGGREGATE
  WAREHOUSE = GLOBALTRADER_WH
  AFTER GLOBALTRADER_DB.RAW.TASK_TRANSFORM
AS
MERGE INTO GLOBALTRADER_DB.MART.REP_DAILY_SUMMARY ...;

-- Resume ALL tasks in correct order (children first, then root)
ALTER TASK GLOBALTRADER_DB.RAW.TASK_AGGREGATE RESUME;
ALTER TASK GLOBALTRADER_DB.RAW.TASK_TRANSFORM RESUME;
ALTER TASK GLOBALTRADER_DB.RAW.TASK_EXTRACT RESUME;
```
2. Answer: *Why must you resume child tasks before the root task? What happens if a child task fails — does the parent retry?*

**Done-when:** 3-task DAG created and running, task history shows chain execution.

---

### T024 — Snowpipe (Continuous Ingestion)
**Do:**
1. Create an external stage pointing to a public S3 bucket (use Snowflake's sample data bucket)
```sql
CREATE STAGE GLOBALTRADER_DB.STAGING.S3_STAGE
  URL = 's3://snowflake-workshop-lab/citibike-trips/'
  FILE_FORMAT = (TYPE=CSV);

-- Create a pipe
CREATE PIPE GLOBALTRADER_DB.STAGING.CITIBIKE_PIPE
  AUTO_INGEST = FALSE
AS
COPY INTO GLOBALTRADER_DB.RAW.CITIBIKE_TEST
FROM @GLOBALTRADER_DB.STAGING.S3_STAGE;

-- Manual trigger for learning (without SQS notification)
ALTER PIPE GLOBALTRADER_DB.STAGING.CITIBIKE_PIPE REFRESH;

-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('GLOBALTRADER_DB.STAGING.CITIBIKE_PIPE');
SELECT * FROM TABLE(INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(DATE_RANGE_START => DATEADD(DAY,-1,CURRENT_DATE())));
```
2. Answer: *What is the difference between Snowpipe and COPY INTO? When would you NOT use Snowpipe? What is the 7-day file deduplication window in Snowpipe?*
3. Answer: *What happens if you send the same file to Snowpipe twice within 7 days? What about after 7 days?*

**Done-when:** Pipe created, REFRESH works, pipe status understood.

---

### T025 — SCD Type 2 Implementation
**Do:**
```sql
-- Create SCD2 table for account history
CREATE TABLE GLOBALTRADER_DB.CLEAN.ACCOUNTS_SCD2 (
  ACCOUNT_SCD_ID    NUMBER AUTOINCREMENT PRIMARY KEY,
  ACCOUNT_ID        VARCHAR(20),
  ACCOUNT_NAME      VARCHAR(200),
  STATUS            VARCHAR(20),
  TIER              VARCHAR(20),
  ANNUAL_REVENUE_USD NUMBER(18,2),
  ACCOUNT_OWNER_ID  VARCHAR(20),
  EFF_START_DATE    DATE,
  EFF_END_DATE      DATE,     -- NULL = current record
  IS_CURRENT        BOOLEAN,
  _LOADED_AT        TIMESTAMP_NTZ
);

-- Initial load
INSERT INTO GLOBALTRADER_DB.CLEAN.ACCOUNTS_SCD2
SELECT NULL, ACCOUNT_ID, ACCOUNT_NAME, STATUS, TIER, ANNUAL_REVENUE_USD,
       ACCOUNT_OWNER_ID, CREATED_DATE, NULL, TRUE, CURRENT_TIMESTAMP()
FROM GLOBALTRADER_DB.CLEAN.ACCOUNTS;

-- Simulate a change batch arriving
CREATE TEMPORARY TABLE ACCOUNTS_CHANGES AS
SELECT ACCOUNT_ID, 'CHURNED' AS NEW_STATUS, CURRENT_DATE() AS CHANGE_DATE
FROM GLOBALTRADER_DB.CLEAN.ACCOUNTS WHERE TIER = 'ENTERPRISE' LIMIT 5;

-- Apply SCD2 MERGE (expire old + insert new)
MERGE INTO GLOBALTRADER_DB.CLEAN.ACCOUNTS_SCD2 AS TARGET
USING ACCOUNTS_CHANGES AS SOURCE
ON TARGET.ACCOUNT_ID = SOURCE.ACCOUNT_ID AND TARGET.IS_CURRENT = TRUE
WHEN MATCHED AND TARGET.STATUS != SOURCE.NEW_STATUS THEN
  UPDATE SET EFF_END_DATE = SOURCE.CHANGE_DATE - 1, IS_CURRENT = FALSE;

INSERT INTO GLOBALTRADER_DB.CLEAN.ACCOUNTS_SCD2
SELECT NULL, AC.ACCOUNT_ID, A.ACCOUNT_NAME, AC.NEW_STATUS, A.TIER,
       A.ANNUAL_REVENUE_USD, A.ACCOUNT_OWNER_ID, AC.CHANGE_DATE, NULL, TRUE, CURRENT_TIMESTAMP()
FROM ACCOUNTS_CHANGES AC
JOIN GLOBALTRADER_DB.CLEAN.ACCOUNTS A ON AC.ACCOUNT_ID = A.ACCOUNT_ID;

-- Query historical state: what did account ACC-00001 look like on 2024-01-01?
SELECT * FROM GLOBALTRADER_DB.CLEAN.ACCOUNTS_SCD2
WHERE ACCOUNT_ID = 'ACC-00001'
  AND EFF_START_DATE <= '2024-01-01'
  AND (EFF_END_DATE >= '2024-01-01' OR EFF_END_DATE IS NULL);
```
2. Answer: *What is the difference between SCD1, SCD2, and SCD3? Which is most common in production and why?*

**Done-when:** SCD2 table has historical and current records, point-in-time query works.

---

### T026 — Dynamic Tables
**Do:**
```sql
-- Dynamic Table automatically refreshes when upstream data changes
CREATE OR REPLACE DYNAMIC TABLE GLOBALTRADER_DB.CLEAN.ACCOUNTS_SUMMARY
  TARGET_LAG = '10 minutes'
  WAREHOUSE = GLOBALTRADER_WH
AS
SELECT
  a.ACCOUNT_ID, a.ACCOUNT_NAME, a.INDUSTRY, a.TIER, a.STATUS,
  COUNT(t.TRANSACTION_ID)        AS TOTAL_TRANSACTIONS,
  SUM(t.TOTAL_AMOUNT_USD)        AS LIFETIME_VALUE_USD,
  MAX(t.TRANSACTION_DATE)        AS LAST_TRANSACTION_DATE,
  COUNT(CASE WHEN i.STATUS = 'OVERDUE' THEN 1 END) AS OVERDUE_INVOICES
FROM GLOBALTRADER_DB.CLEAN.ACCOUNTS a
LEFT JOIN GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS t ON a.ACCOUNT_ID = t.ACCOUNT_ID
LEFT JOIN GLOBALTRADER_DB.RAW.INVOICES i             ON a.ACCOUNT_ID = i.ACCOUNT_ID
GROUP BY 1,2,3,4,5;

-- Check status
SELECT * FROM INFORMATION_SCHEMA.DYNAMIC_TABLES WHERE NAME = 'ACCOUNTS_SUMMARY';
```
2. Answer: *What is the difference between Dynamic Tables and Streams+Tasks? When would you choose Dynamic Tables? What does TARGET_LAG mean — is it guaranteed?*
3. Answer: *What is the difference between Dynamic Tables and Materialized Views in Snowflake?*

**Done-when:** Dynamic Table created, refreshes automatically, you can compare it to Streams+Tasks.

---

### T027 — Stored Procedures (JavaScript / SQL Scripting)
**Do:**
```sql
-- Build a stored procedure using Snowflake Scripting (SQL, not Python)
CREATE OR REPLACE PROCEDURE GLOBALTRADER_DB.RAW.SP_LOAD_INCREMENTAL(
  P_TABLE_NAME VARCHAR,
  P_BATCH_DATE DATE
)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
  v_rows_inserted INT DEFAULT 0;
  v_rows_updated  INT DEFAULT 0;
  v_log_msg       VARCHAR;
BEGIN
  -- Log start
  INSERT INTO GLOBALTRADER_DB.AUDIT.PROC_LOG (PROC_NAME, BATCH_DATE, STATUS, STARTED_AT)
  VALUES (:P_TABLE_NAME, :P_BATCH_DATE, 'RUNNING', CURRENT_TIMESTAMP());

  -- Simulate work
  LET result RESULTSET := (
    SELECT COUNT(*) AS CNT FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
    WHERE TRANSACTION_DATE = :P_BATCH_DATE
  );
  LET c CURSOR FOR result;
  OPEN c;
  FETCH c INTO v_rows_inserted;
  CLOSE c;

  v_log_msg := 'Loaded ' || v_rows_inserted || ' rows for ' || P_BATCH_DATE::VARCHAR;

  -- Log success
  UPDATE GLOBALTRADER_DB.AUDIT.PROC_LOG
  SET STATUS = 'SUCCESS', ROWS_PROCESSED = :v_rows_inserted,
      COMPLETED_AT = CURRENT_TIMESTAMP(), LOG_MSG = :v_log_msg
  WHERE PROC_NAME = :P_TABLE_NAME AND BATCH_DATE = :P_BATCH_DATE;

  RETURN v_log_msg;
EXCEPTION
  WHEN OTHER THEN
    UPDATE GLOBALTRADER_DB.AUDIT.PROC_LOG
    SET STATUS = 'FAILED', LOG_MSG = SQLERRM
    WHERE PROC_NAME = :P_TABLE_NAME AND BATCH_DATE = :P_BATCH_DATE;
    RAISE;
END;
$$;

-- Create audit table first, then call:
CREATE TABLE GLOBALTRADER_DB.AUDIT.PROC_LOG (
  PROC_NAME VARCHAR, BATCH_DATE DATE, STATUS VARCHAR,
  ROWS_PROCESSED INT, STARTED_AT TIMESTAMP_NTZ, COMPLETED_AT TIMESTAMP_NTZ, LOG_MSG VARCHAR
);

CALL GLOBALTRADER_DB.RAW.SP_LOAD_INCREMENTAL('SALES_TRANSACTIONS', CURRENT_DATE());
```
2. Answer: *What is the difference between a Stored Procedure and a UDF in Snowflake? Can a UDF have side effects? Can a stored procedure return a table?*

**Done-when:** SP runs, audit log records written, exception handling tested by calling with a bad date.

---

### T028 — UDFs (User-Defined Functions)
**Do:**
```sql
-- SQL UDF: classify deal size
CREATE OR REPLACE FUNCTION GLOBALTRADER_DB.RAW.CLASSIFY_DEAL(AMOUNT NUMBER)
RETURNS VARCHAR
AS $$
  CASE
    WHEN AMOUNT >= 500000 THEN 'MEGA'
    WHEN AMOUNT >= 100000 THEN 'LARGE'
    WHEN AMOUNT >= 25000  THEN 'MEDIUM'
    WHEN AMOUNT >= 5000   THEN 'SMALL'
    ELSE 'MICRO'
  END
$$;

-- JavaScript UDF: parse phone number format
CREATE OR REPLACE FUNCTION GLOBALTRADER_DB.RAW.PARSE_PHONE(PHONE_RAW VARCHAR)
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
AS $$
  if (!PHONE_RAW) return null;
  let digits = PHONE_RAW.replace(/\D/g,'');
  if (digits.length === 11 && digits[0] === '1') digits = digits.slice(1);
  if (digits.length !== 10) return 'INVALID';
  return '(' + digits.slice(0,3) + ') ' + digits.slice(3,6) + '-' + digits.slice(6);
$$;

-- Use both:
SELECT
  TRANSACTION_ID,
  TOTAL_AMOUNT_USD,
  GLOBALTRADER_DB.RAW.CLASSIFY_DEAL(TOTAL_AMOUNT_USD) AS DEAL_CATEGORY
FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS LIMIT 100;

SELECT LEAD_ID, PHONE,
  GLOBALTRADER_DB.RAW.PARSE_PHONE(PHONE) AS PHONE_FORMATTED
FROM GLOBALTRADER_DB.CLEAN.MARKETING_LEADS LIMIT 50;
```
2. Answer: *Can you call a stored procedure inside a UDF? Can a UDF run a SELECT statement and return multiple rows? What is a UDTF?*

**Done-when:** Both UDFs work in SELECT queries. You can explain UDF vs UDTF vs SP differences.

---

### T029 — Snowpipe vs COPY INTO Decision Framework
**Do:** Write a SQL comment block answering these scenario questions in your own words:

*Scenario A:* Your ERP dumps 50 files to S3 every night at 2am. Files are 500MB each. Which do you use and why?

*Scenario B:* Your IoT sensors send 1 file every 30 seconds, 24/7. Which do you use?

*Scenario C:* A vendor sent you files from 11 days ago that you missed. Snowpipe or COPY INTO?

*Scenario D:* You need to load 3TB of historical data in 4 hours. Which do you use and how do you optimize?

Then build the answer for Scenario D:
```sql
-- Parallel COPY INTO with multiple warehouses
-- Split files and use COPY INTO with PARALLEL parameter
COPY INTO GLOBALTRADER_DB.RAW.HISTORICAL_LOAD
FROM @GLOBALTRADER_DB.STAGING.S3_STAGE/historical/
PATTERN = '.*2023.*\.csv'
FILE_FORMAT = (TYPE=CSV)
ON_ERROR = CONTINUE;
```

**Done-when:** All 4 scenarios answered with reasoning. COPY INTO with PATTERN works.

---

### T030 — Error Handling & Recovery Patterns
**Do:**
```sql
-- View copy errors in detail
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME => 'EXPENSE_CLAIMS',
  START_TIME => DATEADD(DAYS,-7,CURRENT_TIMESTAMP())
))
WHERE STATUS = 'LOAD_FAILED'
ORDER BY LAST_LOAD_TIME DESC;

-- Create a reject table to capture bad rows
CREATE TABLE GLOBALTRADER_DB.AUDIT.LOAD_REJECTS (
  SOURCE_FILE     VARCHAR,
  REJECT_REASON   VARCHAR,
  RAW_ROW         VARCHAR,
  REJECTED_AT     TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Use VALIDATION_MODE to preview errors before loading
COPY INTO GLOBALTRADER_DB.RAW.MARKETING_LEADS_RAW
FROM @GLOBALTRADER_DB.STAGING.INTERNAL_STAGE/02_marketing_leads.csv
FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1)
VALIDATION_MODE = RETURN_ERRORS;
```
2. Answer: *What is VALIDATION_MODE? Does it load any data? What are the valid values?*

**Done-when:** Error history queried, VALIDATION_MODE used, reject pattern understood.

---

## Phase 3 — Gold Layer / Data Mart (Tasks T031–T040)
> **Goal:** Build business-ready dimensional models. These are what analysts and BI tools consume.

---

### T031 — Dimensional Modeling Design
**Do:** Design on paper (or as SQL comments) a Star Schema for GlobalTrader Inc. with:
- **Fact table:** FACT_SALES_TRANSACTIONS
- **Dimension tables:** DIM_ACCOUNT, DIM_REP, DIM_PRODUCT, DIM_DATE, DIM_GEOGRAPHY

Then answer:
- *What is the difference between a Star Schema and Snowflake Schema?*
- *What is a surrogate key? Why use one instead of the natural key?*
- *What is a slowly changing dimension? Which type applies to account status changes?*

**Done-when:** Schema design documented, all relationships defined, questions answered.

---

### T032 — DIM_DATE (Date Dimension)
**Do:**
```sql
CREATE TABLE GLOBALTRADER_DB.MART.DIM_DATE AS
WITH DATES AS (
  SELECT DATEADD(DAY, SEQ4(), '2020-01-01'::DATE) AS DATE_VAL
  FROM TABLE(GENERATOR(ROWCOUNT => 3650))
)
SELECT
  TO_NUMBER(TO_CHAR(DATE_VAL,'YYYYMMDD'))           AS DATE_KEY,
  DATE_VAL                                           AS FULL_DATE,
  YEAR(DATE_VAL)                                     AS YEAR,
  QUARTER(DATE_VAL)                                  AS QUARTER,
  'Q' || QUARTER(DATE_VAL) || '-' || YEAR(DATE_VAL) AS FISCAL_QUARTER,
  MONTH(DATE_VAL)                                    AS MONTH_NUM,
  MONTHNAME(DATE_VAL)                                AS MONTH_NAME,
  WEEKOFYEAR(DATE_VAL)                               AS WEEK_OF_YEAR,
  DAYOFWEEK(DATE_VAL)                                AS DAY_OF_WEEK,
  DAYNAME(DATE_VAL)                                  AS DAY_NAME,
  DAYOFYEAR(DATE_VAL)                                AS DAY_OF_YEAR,
  CASE WHEN DAYOFWEEK(DATE_VAL) IN (0,6) THEN FALSE ELSE TRUE END AS IS_WEEKDAY,
  CASE WHEN MONTH(DATE_VAL) IN (1,2,3) THEN 'H1'
       WHEN MONTH(DATE_VAL) IN (4,5,6) THEN 'H1'
       ELSE 'H2' END                                 AS HALF_YEAR,
  LAST_DAY(DATE_VAL)                                 AS LAST_DAY_OF_MONTH,
  DATE_VAL = LAST_DAY(DATE_VAL)                      AS IS_MONTH_END
FROM DATES;
```

**Done-when:** DIM_DATE has 3,650 rows (10 years), all columns computed correctly.

---

### T033 — Fact Table Build (FACT_SALES)
**Do:**
```sql
CREATE TABLE GLOBALTRADER_DB.MART.FACT_SALES AS
SELECT
  -- Surrogate keys (join to dims using DATE_KEY pattern)
  TO_NUMBER(TO_CHAR(t.TRANSACTION_DATE,'YYYYMMDD')) AS DATE_KEY,
  t.TRANSACTION_ID,
  t.ACCOUNT_ID,
  t.REP_ID,
  t.PRODUCT_SKU,
  -- Measures (facts are always numeric, additive)
  t.QUANTITY,
  t.UNIT_PRICE_USD,
  t.GROSS_AMOUNT_USD,
  t.DISCOUNT_AMOUNT_USD,
  t.NET_AMOUNT_USD,
  t.TAX_AMOUNT_USD,
  t.TOTAL_AMOUNT_USD,
  t.DISCOUNT_PCT,
  -- Degenerate dimensions (codes with no dim table)
  t.STATUS,
  t.DEAL_TYPE,
  t.CURRENCY_CODE,
  t.PAYMENT_TERMS,
  -- Audit
  CURRENT_TIMESTAMP() AS _LOADED_AT
FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS t
WHERE t.STATUS = 'POSTED';

-- Cluster the fact table on the highest-cardinality filter column
ALTER TABLE GLOBALTRADER_DB.MART.FACT_SALES
CLUSTER BY (DATE_KEY, ACCOUNT_ID);
```
2. Answer: *What is a degenerate dimension? What is an additive vs semi-additive vs non-additive fact? Give an example of each from this data.*

**Done-when:** FACT_SALES built, clustering key set, you can explain additive/non-additive facts.

---

### T034 — Complex Analytical Queries (Gold Layer)
**Do:** Write all 5 queries from scratch:
```sql
-- 1. Revenue Waterfall: Gross → Discount → Net → Tax → Total by quarter
SELECT
  d.FISCAL_QUARTER,
  SUM(f.GROSS_AMOUNT_USD)    AS GROSS_REVENUE,
  SUM(f.DISCOUNT_AMOUNT_USD) AS TOTAL_DISCOUNTS,
  SUM(f.NET_AMOUNT_USD)      AS NET_REVENUE,
  SUM(f.TAX_AMOUNT_USD)      AS TOTAL_TAX,
  SUM(f.TOTAL_AMOUNT_USD)    AS COLLECTED_REVENUE,
  SUM(f.DISCOUNT_AMOUNT_USD)/NULLIF(SUM(f.GROSS_AMOUNT_USD),0)*100 AS DISCOUNT_RATE_PCT
FROM GLOBALTRADER_DB.MART.FACT_SALES f
JOIN GLOBALTRADER_DB.MART.DIM_DATE d ON f.DATE_KEY = d.DATE_KEY
GROUP BY 1 ORDER BY 1;

-- 2. Rep performance vs quota (requires COMPENSATION join)
-- 3. AR Aging analysis: overdue invoices by aging bucket and industry
-- 4. Top 3 accounts by revenue per region (use QUALIFY)
-- 5. Commission effectiveness: total comp paid vs revenue generated per rep
```

**Done-when:** All 5 queries return meaningful results. You can explain each business question.

---

### T035 — Materialized Views
**Do:**
```sql
-- Create a materialized view for heavy aggregation used by dashboards
CREATE MATERIALIZED VIEW GLOBALTRADER_DB.MART.MV_REP_QUARTERLY_SUMMARY AS
SELECT
  r.REP_ID, r.FULL_NAME, r.REGION, r.LEVEL,
  d.FISCAL_QUARTER, d.FISCAL_YEAR,
  COUNT(f.TRANSACTION_ID)      AS TOTAL_DEALS,
  SUM(f.TOTAL_AMOUNT_USD)      AS TOTAL_REVENUE,
  AVG(f.TOTAL_AMOUNT_USD)      AS AVG_DEAL_SIZE,
  SUM(f.DISCOUNT_AMOUNT_USD)   AS TOTAL_DISCOUNTS,
  MAX(f.TOTAL_AMOUNT_USD)      AS LARGEST_DEAL
FROM GLOBALTRADER_DB.MART.FACT_SALES f
JOIN GLOBALTRADER_DB.RAW.SALES_REPS r  ON f.REP_ID = r.REP_ID
JOIN GLOBALTRADER_DB.MART.DIM_DATE d   ON f.DATE_KEY = d.DATE_KEY
GROUP BY 1,2,3,4,5,6;

-- Query it (should be faster than querying FACT_SALES directly)
SELECT * FROM GLOBALTRADER_DB.MART.MV_REP_QUARTERLY_SUMMARY WHERE FISCAL_YEAR = 2024;
```
2. Answer: *What is the difference between a Materialized View and a Dynamic Table? What SQL is NOT supported in a Materialized View? When does a MV become stale?*

**Done-when:** MV created and queryable. You can explain MV limitations (no GROUP BY ROLLUP, no UNION, etc.)

---

### T036 — PIVOT & UNPIVOT
**Do:**
```sql
-- PIVOT: show quarterly revenue per product line as columns
SELECT *
FROM (
  SELECT PRODUCT_LINE, FISCAL_QUARTER_LABEL, TOTAL_AMOUNT_USD
  FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
) AS SOURCE
PIVOT (
  SUM(TOTAL_AMOUNT_USD) FOR FISCAL_QUARTER_LABEL IN ('Q1','Q2','Q3','Q4')
) AS PIVOTED;

-- UNPIVOT: flatten a wide table back to long format
-- (create a sample wide table first, then unpivot)
CREATE TEMPORARY TABLE SALES_WIDE AS
SELECT REP_ID, Q1_REV, Q2_REV, Q3_REV, Q4_REV FROM (...);

SELECT REP_ID, QUARTER_LABEL, REVENUE
FROM SALES_WIDE
UNPIVOT (REVENUE FOR QUARTER_LABEL IN (Q1_REV, Q2_REV, Q3_REV, Q4_REV));
```
2. Answer: *What is the performance implication of PIVOT on large datasets? When would you use PIVOT vs a GROUP BY with CASE WHEN?*

**Done-when:** Both PIVOT and UNPIVOT work. You understand when each is appropriate.

---

### T037 — CTEs vs Subqueries vs Temp Tables (Performance)
**Do:** Write the same complex query 3 ways and compare Query Profile:
```sql
-- VERSION 1: Nested subqueries
SELECT * FROM (SELECT * FROM (SELECT ... FROM ...) WHERE ...) WHERE ...;

-- VERSION 2: CTEs
WITH base AS (...),
enriched AS (SELECT ... FROM base JOIN ...),
final AS (SELECT ... FROM enriched WHERE ...)
SELECT * FROM final;

-- VERSION 3: Temp tables (materialize intermediate results)
CREATE TEMPORARY TABLE T1 AS SELECT ... FROM ...;
CREATE TEMPORARY TABLE T2 AS SELECT ... FROM T1 JOIN ...;
SELECT * FROM T2 WHERE ...;
```
2. Answer: *Are CTEs always better than subqueries in Snowflake? Are they materialized? When should you use a TEMP TABLE instead of a CTE?*

**Done-when:** All 3 versions produce same result, you can compare execution plans.

---

### T038 — ROLLUP, CUBE, GROUPING SETS
**Do:**
```sql
-- ROLLUP: hierarchical subtotals (most useful in reporting)
SELECT
  COALESCE(REGION, 'ALL REGIONS')   AS REGION,
  COALESCE(INDUSTRY, 'ALL INDUSTRIES') AS INDUSTRY,
  COALESCE(TIER, 'ALL TIERS')       AS TIER,
  SUM(f.TOTAL_AMOUNT_USD)            AS REVENUE
FROM GLOBALTRADER_DB.MART.FACT_SALES f
JOIN GLOBALTRADER_DB.CLEAN.ACCOUNTS a ON f.ACCOUNT_ID = a.ACCOUNT_ID
GROUP BY ROLLUP(a.REGION, a.INDUSTRY, a.TIER)
ORDER BY 1,2,3;

-- GROUPING SETS: exactly which combinations you want
SELECT REGION, TIER, SUM(TOTAL_AMOUNT_USD)
FROM ...
GROUP BY GROUPING SETS ((REGION), (TIER), (REGION, TIER), ());

-- CUBE: all combinations
GROUP BY CUBE(REGION, INDUSTRY, TIER);
```
2. Answer: *What is the difference between ROLLUP and CUBE? What does GROUPING() function return and why is it useful?*

**Done-when:** All 3 produce correct results. GROUPING() function used to identify aggregate rows.

---

### T039 — String & Date Functions (Interview Classics)
**Do:** Write queries using each of these — no looking up syntax:
```sql
-- String functions
SPLIT_PART('alex.morgan@globaltrader.com','@',1)  -- get username
REGEXP_SUBSTR(email, '[^@]+')                       -- same with regex
REGEXP_REPLACE(phone, '[^0-9]','')                  -- strip non-digits
EDITDISTANCE('Meridian','Meriedian')                -- fuzzy match
SOUNDEX('Morgan')                                   -- phonetic match
LISTAGG(product_line,',') WITHIN GROUP (ORDER BY product_line) -- string aggregation

-- Date functions  
DATEDIFF('DAY', hire_date, CURRENT_DATE())
DATEADD('MONTH', 3, '2024-01-01')
DATE_TRUNC('MONTH', transaction_date)
LAST_DAY(transaction_date)
DAYOFWEEK(transaction_date)
TO_CHAR(transaction_date, 'Month DD, YYYY')
TRY_TO_DATE('02/30/2024','MM/DD/YYYY')  -- invalid date — what does TRY_ return?
```

**Done-when:** All functions used in real queries against your data. You know TRY_ vs non-TRY_ behavior.

---

### T040 — EXCEPT, INTERSECT, MINUS (Set Operations)
**Do:**
```sql
-- Find accounts that have transactions but NO invoices
SELECT ACCOUNT_ID FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
EXCEPT
SELECT ACCOUNT_ID FROM GLOBALTRADER_DB.RAW.INVOICES;

-- Find reps in BOTH compensation AND transactions table
SELECT REP_ID FROM GLOBALTRADER_DB.RAW.COMPENSATION
INTERSECT
SELECT DISTINCT REP_ID FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS;

-- Answer: What is the difference between EXCEPT and NOT EXISTS? Which is more performant in Snowflake and why?
```

**Done-when:** Set operations return correct results. You understand EXCEPT vs NOT IN vs NOT EXISTS performance.

---

## Phase 4 — Performance & Cost Engineering (Tasks T041–T055)
> **Goal:** This phase is what separates junior from senior. Every interview tests cost + performance.

---

### T041 — Warehouse Sizing Strategy
**Do:**
1. Run the same complex 4-table join at XS, S, and M warehouse sizes
2. Record execution time and credits used for each
3. Calculate cost per query: credits × $2.00/credit (standard rate)
```sql
-- XS Warehouse
ALTER SESSION SET CURRENT_WAREHOUSE = 'XS_TEST_WH';
-- run query, note time from query history

-- Switch to Small
ALTER WAREHOUSE XS_TEST_WH SET WAREHOUSE_SIZE = 'SMALL';
-- run same query

-- Switch to Medium  
ALTER WAREHOUSE XS_TEST_WH SET WAREHOUSE_SIZE = 'MEDIUM';
-- run same query
```
4. Answer: *At what point does increasing warehouse size NOT help? What is spill to disk and how does warehouse size affect it?*
5. Answer: *What is the difference between multi-cluster warehouse (scale-out) and warehouse resizing (scale-up)? Which handles concurrency?*

**Done-when:** You have a table comparing time, credits, cost per size. You understand scale-up vs scale-out.

---

### T042 — Resource Monitors & Cost Control
**Do:**
```sql
-- Create a resource monitor to cap credit spend
CREATE RESOURCE MONITOR GLOBALTRADER_MONITOR
  CREDIT_QUOTA = 20            -- 20 credits max
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 75 PERCENT DO NOTIFY
    ON 90 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND;

-- Attach to your warehouse
ALTER WAREHOUSE GLOBALTRADER_WH SET RESOURCE_MONITOR = GLOBALTRADER_MONITOR;

-- Check credit usage
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE WAREHOUSE_NAME = 'GLOBALTRADER_WH'
  AND START_TIME >= DATEADD(DAY,-7,CURRENT_TIMESTAMP())
ORDER BY START_TIME DESC;
```
2. Answer: *What happens to running queries when a resource monitor suspends a warehouse? What is the difference between SUSPEND and SUSPEND_IMMEDIATE?*

**Done-when:** Resource monitor attached, you can query credit usage history.

---

### T043 — Clustering Keys Deep Dive
**Do:**
```sql
-- Check current clustering depth on FACT_SALES
SELECT SYSTEM$CLUSTERING_INFORMATION('GLOBALTRADER_DB.MART.FACT_SALES');
SELECT SYSTEM$CLUSTERING_INFORMATION('GLOBALTRADER_DB.MART.FACT_SALES','(DATE_KEY)');
SELECT SYSTEM$CLUSTERING_INFORMATION('GLOBALTRADER_DB.MART.FACT_SALES','(DATE_KEY, ACCOUNT_ID)');
SELECT SYSTEM$CLUSTERING_INFORMATION('GLOBALTRADER_DB.MART.FACT_SALES','(ACCOUNT_ID)');

-- Interpret the output:
-- average_depth: lower = better clustering (1.0 = perfect)
-- average_overlaps: number of other partitions each partition overlaps with
-- partition_depth_histogram: distribution of depth across partitions

-- Run queries with different filter columns, compare Query Profile partition counts:
-- Query A: WHERE DATE_KEY = 20240101 (clustered — should prune well)
-- Query B: WHERE REP_ID = 'REP-001' (not clustered — will scan more)
-- Query C: WHERE DATE_KEY = 20240101 AND ACCOUNT_ID = 'ACC-00001' (compound cluster)
```
2. Answer: *When does Snowflake automatically recluster? Who pays for reclustering credits? What columns make BAD clustering keys? (hint: booleans, low cardinality)*
3. Answer: *What is the Search Optimization Service? How is it different from clustering?*

**Done-when:** You can read CLUSTERING_INFORMATION output, explain depth/overlap metrics, and choose appropriate keys.

---

### T044 — Query Optimization Patterns
**Do:** Find and fix these 5 anti-patterns in your own queries:
```sql
-- ANTI-PATTERN 1: SELECT * on a wide table
-- Fix: Select only needed columns (Snowflake is columnar — unused columns cost nothing BUT * hurts network/result cache)

-- ANTI-PATTERN 2: Non-sargable WHERE clause (prevents pruning)
-- Bad:  WHERE YEAR(transaction_date) = 2024
-- Good: WHERE transaction_date BETWEEN '2024-01-01' AND '2024-12-31'

-- ANTI-PATTERN 3: DISTINCT instead of GROUP BY
-- Bad:  SELECT DISTINCT account_id, product_line FROM ...
-- Good: SELECT account_id, product_line FROM ... GROUP BY 1,2

-- ANTI-PATTERN 4: OR conditions instead of UNION ALL
-- Bad:  WHERE status = 'POSTED' OR status = 'PENDING'
-- Good: WHERE status IN ('POSTED','PENDING')  (or UNION ALL if different logic)

-- ANTI-PATTERN 5: Correlated subquery in SELECT
-- Bad:  SELECT t.*, (SELECT SUM(x) FROM table2 WHERE id = t.id) AS sub_total FROM t
-- Good: LEFT JOIN with aggregation CTE
```

**Done-when:** Each anti-pattern written, explained, and fixed. Query Profile compared before/after for at least 2.

---

### T045 — Account Usage & Cost Analysis
**Do:**
```sql
-- Who is running expensive queries?
SELECT
  USER_NAME,
  COUNT(*)                                                           AS QUERY_COUNT,
  SUM(TOTAL_ELAPSED_TIME)/1000/60                                   AS TOTAL_MINUTES,
  AVG(TOTAL_ELAPSED_TIME)/1000                                      AS AVG_SECONDS,
  SUM(BYTES_SCANNED)/1024/1024/1024                                 AS TOTAL_GB_SCANNED,
  SUM(CREDITS_USED_CLOUD_SERVICES)                                  AS TOTAL_CREDITS
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME >= DATEADD(DAY,-7,CURRENT_TIMESTAMP())
  AND WAREHOUSE_NAME = 'GLOBALTRADER_WH'
GROUP BY 1
ORDER BY TOTAL_CREDITS DESC;

-- Most expensive queries
SELECT QUERY_TEXT, TOTAL_ELAPSED_TIME/1000 AS SECONDS,
       BYTES_SCANNED/1024/1024/1024 AS GB_SCANNED
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME >= DATEADD(DAY,-7,CURRENT_TIMESTAMP())
ORDER BY BYTES_SCANNED DESC
LIMIT 20;

-- Storage cost by table
SELECT TABLE_NAME, TABLE_SCHEMA,
       ACTIVE_BYTES/1024/1024/1024 AS ACTIVE_GB,
       TIME_TRAVEL_BYTES/1024/1024/1024 AS TIME_TRAVEL_GB,
       FAILSAFE_BYTES/1024/1024/1024 AS FAILSAFE_GB
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
ORDER BY ACTIVE_BYTES DESC;
```
2. Answer: *What is the latency of ACCOUNT_USAGE views? What is INFORMATION_SCHEMA and how is it different?*

**Done-when:** You can identify top spenders, expensive queries, and storage consumers from ACCOUNT_USAGE.

---

### T046 — Deliberately Expensive Query + Fix It
**Do:**
```sql
-- Run this intentionally bad query (will be slow and expensive)
-- Step 1: Run it, check Query Profile, note: partitions scanned, spill to disk, execution time
SELECT
  a.ACCOUNT_NAME,
  a.INDUSTRY,
  a.ANNUAL_REVENUE_USD,
  t.PRODUCT_LINE,
  SUM(t.TOTAL_AMOUNT_USD) AS REVENUE,
  COUNT(DISTINCT t.TRANSACTION_ID) AS DEAL_COUNT,
  AVG(t.DISCOUNT_PCT) AS AVG_DISCOUNT,
  (SELECT SUM(TOTAL_AMOUNT_USD) FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS) AS GRAND_TOTAL
FROM GLOBALTRADER_DB.RAW.ACCOUNTS a
JOIN GLOBALTRADER_DB.RAW.SALES_TRANSACTIONS t ON UPPER(a.ACCOUNT_ID) = UPPER(t.ACCOUNT_ID)
WHERE YEAR(t.TRANSACTION_DATE) = 2024
GROUP BY 1,2,3,4
ORDER BY REVENUE DESC;

-- Step 2: Rewrite it properly
-- Fix: use CLEAN schema, remove UPPER() on join key, fix non-sargable date filter,
--      move correlated subquery to CTE
-- Compare Query Profile before/after
```

**Done-when:** Fixed query is measurably faster in Query Profile. You document the specific improvements.

---

## Phase 5 — Governance & Security (Tasks T047–T054)
> **Goal:** Enterprise Snowflake is 60% security/governance. These come up in senior interviews constantly.

---

### T047 — Dynamic Data Masking
**Do:**
```sql
-- Create masking policy for PII email addresses
CREATE MASKING POLICY GLOBALTRADER_DB.AUDIT.MASK_EMAIL
AS (VAL VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('GLOBALTRADER_ADMIN') THEN VAL
    WHEN IS_ROLE_IN_SESSION('GLOBALTRADER_ENGINEER') THEN
      REGEXP_REPLACE(VAL,'(.)(.*)(@.*)','\\1****\\3')  -- a****@domain.com
    ELSE '***MASKED***'
  END;

-- Apply to the email column
ALTER TABLE GLOBALTRADER_DB.CLEAN.MARKETING_LEADS
MODIFY COLUMN EMAIL SET MASKING POLICY GLOBALTRADER_DB.AUDIT.MASK_EMAIL;

-- Test: switch roles and see different results
USE ROLE GLOBALTRADER_ANALYST;
SELECT EMAIL FROM GLOBALTRADER_DB.CLEAN.MARKETING_LEADS LIMIT 5;

USE ROLE SYSADMIN;
SELECT EMAIL FROM GLOBALTRADER_DB.CLEAN.MARKETING_LEADS LIMIT 5;
```
2. Answer: *Can you create a masking policy that partially masks based on a lookup table? Can you apply masking to a VARIANT column?*

**Done-when:** Masking behaves differently per role. You can explain policy inheritance.

---

### T048 — Row Access Policies
**Do:**
```sql
-- Create a mapping table: which rep can see which regions
CREATE TABLE GLOBALTRADER_DB.AUDIT.REP_REGION_ACCESS (
  REP_LOGIN VARCHAR,
  ALLOWED_REGION VARCHAR
);
INSERT INTO GLOBALTRADER_DB.AUDIT.REP_REGION_ACCESS VALUES
('ALEX_MORGAN','NORTH_AMERICA'),('EMMA_SCHULZ','EMEA'),
('OMAR_KHALID','MEA'),('PRIYA_NAIR','APAC');

-- Row access policy: reps only see their region's accounts
CREATE ROW ACCESS POLICY GLOBALTRADER_DB.AUDIT.RAP_REGION
AS (ACCOUNT_REGION VARCHAR) RETURNS BOOLEAN ->
  CURRENT_ROLE() = 'GLOBALTRADER_ADMIN'
  OR EXISTS (
    SELECT 1 FROM GLOBALTRADER_DB.AUDIT.REP_REGION_ACCESS
    WHERE REP_LOGIN = CURRENT_USER()
      AND ALLOWED_REGION = ACCOUNT_REGION
  );

-- Apply to accounts table
ALTER TABLE GLOBALTRADER_DB.CLEAN.ACCOUNTS
ADD ROW ACCESS POLICY GLOBALTRADER_DB.AUDIT.RAP_REGION ON (REGION);
```
2. Answer: *Can you apply both a masking policy AND a row access policy to the same table? Can a user see that rows are being hidden from them?*

**Done-when:** Row access policy filters accounts by region per user. Admin sees all rows.

---

### T049 — Data Classification & Sensitive Data
**Do:**
```sql
-- Run Snowflake's automatic sensitive data classification
CALL SNOWFLAKE.DATA_PRIVACY.CLASSIFY_SCHEMA(
  'GLOBALTRADER_DB.CLEAN',
  OBJECT_CONSTRUCT()
);

-- Review classification results
SELECT *
FROM TABLE(SNOWFLAKE.DATA_PRIVACY.CLASSIFICATION_RESULT('GLOBALTRADER_DB.CLEAN.MARKETING_LEADS'));

-- Tag sensitive columns manually
CREATE TAG GLOBALTRADER_DB.AUDIT.PII_TAG ALLOWED_VALUES 'EMAIL','PHONE','NAME','SSN','FINANCIAL';

ALTER TABLE GLOBALTRADER_DB.CLEAN.MARKETING_LEADS
  MODIFY COLUMN EMAIL SET TAG GLOBALTRADER_DB.AUDIT.PII_TAG = 'EMAIL';
ALTER TABLE GLOBALTRADER_DB.CLEAN.MARKETING_LEADS
  MODIFY COLUMN PHONE SET TAG GLOBALTRADER_DB.AUDIT.PII_TAG = 'PHONE';

-- Query all PII-tagged columns across the account
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE TAG_NAME = 'PII_TAG';
```

**Done-when:** Classification run, PII tags applied, TAG_REFERENCES shows tagged columns.

---

### T050 — Audit Logging & Access History
**Do:**
```sql
-- Who accessed sensitive tables?
SELECT
  USER_NAME,
  QUERY_TYPE,
  QUERY_TEXT,
  START_TIME,
  OBJECTS_ACCESSED
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE START_TIME >= DATEADD(DAY,-1,CURRENT_TIMESTAMP())
  AND ARRAY_CONTAINS(
    OBJECT_CONSTRUCT('objectName','GLOBALTRADER_DB.CLEAN.MARKETING_LEADS')::VARIANT,
    OBJECTS_ACCESSED
  )
ORDER BY START_TIME DESC;

-- Login history
SELECT USER_NAME, EVENT_TIMESTAMP, IS_SUCCESS, ERROR_MESSAGE, CLIENT_IP
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE EVENT_TIMESTAMP >= DATEADD(DAY,-7,CURRENT_TIMESTAMP())
ORDER BY EVENT_TIMESTAMP DESC;
```
2. Answer: *What is the difference between QUERY_HISTORY and ACCESS_HISTORY? What is the latency of ACCESS_HISTORY? How long is data retained?*

**Done-when:** You can query who accessed sensitive data and when, from ACCOUNT_USAGE.

---

### T051 — Network Policies & IP Whitelisting
**Do:**
```sql
-- Create a network policy (restrict access to specific IPs)
CREATE NETWORK POLICY GLOBALTRADER_NETWORK_POLICY
  ALLOWED_IP_LIST = ('0.0.0.0/0')  -- allow all for learning; in prod use real IPs
  COMMENT = 'GlobalTrader production network policy';

-- Apply at account level (SYSADMIN can't do this — needs ACCOUNTADMIN)
-- USE ROLE ACCOUNTADMIN;
-- ALTER ACCOUNT SET NETWORK_POLICY = GLOBALTRADER_NETWORK_POLICY;

-- Check existing policies
SHOW NETWORK POLICIES;
```
2. Answer: *At what levels can you apply a network policy? (account, user, session) Which takes precedence? Can you apply different policies to different users?*

**Done-when:** Network policy created, levels of application understood.

---

### T052 — Secure Views & Data Sharing
**Do:**
```sql
-- Create a secure view (prevents query plan inspection by unauthorized users)
CREATE SECURE VIEW GLOBALTRADER_DB.MART.SECURE_REP_PERFORMANCE AS
SELECT
  r.FULL_NAME,
  r.REGION,
  r.LEVEL,
  SUM(f.TOTAL_AMOUNT_USD) AS TOTAL_REVENUE,
  COUNT(f.TRANSACTION_ID) AS DEAL_COUNT
FROM GLOBALTRADER_DB.MART.FACT_SALES f
JOIN GLOBALTRADER_DB.RAW.SALES_REPS r ON f.REP_ID = r.REP_ID
GROUP BY 1,2,3;

-- Test: a non-admin user cannot see the underlying SQL
-- Create a share (for cross-account data sharing)
CREATE SHARE GLOBALTRADER_SHARE;
GRANT USAGE ON DATABASE GLOBALTRADER_DB TO SHARE GLOBALTRADER_SHARE;
GRANT USAGE ON SCHEMA GLOBALTRADER_DB.MART TO SHARE GLOBALTRADER_SHARE;
GRANT SELECT ON GLOBALTRADER_DB.MART.SECURE_REP_PERFORMANCE TO SHARE GLOBALTRADER_SHARE;

SHOW SHARES;
```
2. Answer: *What is the difference between a regular view and a secure view? What does SECURE prevent? Can you share a table directly? Can the consumer modify shared data?*

**Done-when:** Secure view created, share set up, concepts explained.

---

### T053 — Time Travel for Compliance
**Do:**
```sql
-- Compliance scenario: "Show me the state of all invoices as of the end of last month"
SELECT * FROM GLOBALTRADER_DB.RAW.INVOICES
AT(TIMESTAMP => DATEADD(SECOND,-1,DATE_TRUNC('MONTH',CURRENT_DATE()))::TIMESTAMP_NTZ);

-- "Someone updated STATUS incorrectly yesterday — who changed it and when?"
-- Query ACCESS_HISTORY to find the session that ran the update
-- Then use Time Travel to get the before-state

-- "Restore a dropped table" (simulate and recover)
DROP TABLE GLOBALTRADER_DB.CLEAN.MARKETING_LEADS;
-- Restore within Time Travel window:
UNDROP TABLE GLOBALTRADER_DB.CLEAN.MARKETING_LEADS;
```
2. Answer: *Can you UNDROP a table after its Time Travel retention period expires? What is the maximum Time Travel period for Standard vs Enterprise edition? Who determines the retention period for shared objects?*

**Done-when:** Point-in-time query works, UNDROP works, retention limits known.

---

### T054 — Column-Level Security & Object Tagging Strategy
**Do:**
Build a complete governance inventory query:
```sql
-- List all tables, their tags, their masking policies, their row access policies
SELECT
  t.TABLE_CATALOG, t.TABLE_SCHEMA, t.TABLE_NAME, t.ROW_COUNT,
  c.COLUMN_NAME, c.DATA_TYPE,
  tr.TAG_NAME, tr.TAG_VALUE,
  mp.POLICY_NAME AS MASKING_POLICY,
  rp.POLICY_NAME AS ROW_ACCESS_POLICY
FROM INFORMATION_SCHEMA.TABLES t
JOIN INFORMATION_SCHEMA.COLUMNS c
  ON t.TABLE_NAME = c.TABLE_NAME AND t.TABLE_SCHEMA = c.TABLE_SCHEMA
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
  ON tr.OBJECT_NAME = t.TABLE_NAME AND tr.COLUMN_NAME = c.COLUMN_NAME
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES mp
  ON mp.REF_ENTITY_NAME = t.TABLE_NAME AND mp.POLICY_KIND = 'MASKING_POLICY'
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES rp
  ON rp.REF_ENTITY_NAME = t.TABLE_NAME AND rp.POLICY_KIND = 'ROW_ACCESS_POLICY'
WHERE t.TABLE_SCHEMA IN ('CLEAN','MART')
ORDER BY t.TABLE_NAME, c.ORDINAL_POSITION;
```

**Done-when:** Single governance query shows all policies, tags, and column-level security in one place.

---

## Phase 6 — Cortex AI (Tasks T055–T065)
> **Goal:** Cortex AI is your career differentiator. Most Snowflake engineers can't do this yet.

---

### T055 — Cortex Sentiment Analysis on Support Tickets
**Do:**
```sql
-- Sentiment analysis on ticket descriptions
SELECT
  TICKET_ID,
  ACCOUNT_ID,
  PRIORITY,
  CATEGORY,
  DESCRIPTION,
  SNOWFLAKE.CORTEX.SENTIMENT(DESCRIPTION)          AS SENTIMENT_SCORE,
  CASE
    WHEN SNOWFLAKE.CORTEX.SENTIMENT(DESCRIPTION) >= 0.5  THEN 'POSITIVE'
    WHEN SNOWFLAKE.CORTEX.SENTIMENT(DESCRIPTION) <= -0.5 THEN 'NEGATIVE'
    ELSE 'NEUTRAL'
  END                                               AS SENTIMENT_LABEL
FROM GLOBALTRADER_DB.RAW.SUPPORT_TICKETS_RAW
LIMIT 100;

-- Aggregate: which categories get most negative sentiment?
SELECT CATEGORY, PRIORITY,
  AVG(SNOWFLAKE.CORTEX.SENTIMENT(DESCRIPTION)) AS AVG_SENTIMENT,
  COUNT(*) AS TICKET_COUNT
FROM GLOBALTRADER_DB.RAW.SUPPORT_TICKETS_RAW
GROUP BY 1,2
ORDER BY AVG_SENTIMENT ASC;
```
2. Answer: *What does a sentiment score of -1 vs 0 vs +1 mean? Is SENTIMENT() a free function or does it consume Cortex credits?*

**Done-when:** Sentiment scores computed on 100+ tickets, aggregated by category.

---

### T056 — Cortex SUMMARIZE & TRANSLATE
**Do:**
```sql
-- Summarize long resolution notes
SELECT
  TICKET_ID,
  RESOLUTION_NOTES,
  SNOWFLAKE.CORTEX.SUMMARIZE(RESOLUTION_NOTES) AS SUMMARY
FROM GLOBALTRADER_DB.RAW.SUPPORT_TICKETS_RAW
WHERE RESOLUTION_NOTES IS NOT NULL
LIMIT 50;

-- Extract key information with COMPLETE
SELECT
  TICKET_ID,
  DESCRIPTION,
  SNOWFLAKE.CORTEX.COMPLETE(
    'mistral-7b',
    'Extract the root cause in one sentence from this support ticket: ' || DESCRIPTION
  ) AS ROOT_CAUSE_EXTRACTED
FROM GLOBALTRADER_DB.RAW.SUPPORT_TICKETS_RAW
WHERE PRIORITY IN ('P1','P2')
LIMIT 20;
```
2. Answer: *What is the difference between CORTEX.SUMMARIZE() and CORTEX.COMPLETE()? What models are available in Snowflake Cortex as of 2025?*

**Done-when:** Summaries and completions generated on real ticket data.

---

### T057 — Cortex CLASSIFY_TEXT
**Do:**
```sql
-- Auto-classify support tickets into categories
SELECT
  TICKET_ID,
  DESCRIPTION,
  CATEGORY AS HUMAN_CATEGORY,
  SNOWFLAKE.CORTEX.CLASSIFY_TEXT(
    DESCRIPTION,
    ['Infrastructure','Billing','Product Bug','Feature Request','User Error','Security']
  ) AS AI_CATEGORY
FROM GLOBALTRADER_DB.RAW.SUPPORT_TICKETS_RAW
LIMIT 100;

-- How often does AI agree with human categorization?
WITH CLASSIFIED AS (
  SELECT CATEGORY AS HUMAN,
    SNOWFLAKE.CORTEX.CLASSIFY_TEXT(DESCRIPTION,
      ['Infrastructure','Billing','Product Bug','Feature Request','User Error','Security']
    ):label::VARCHAR AS AI
  FROM GLOBALTRADER_DB.RAW.SUPPORT_TICKETS_RAW
  LIMIT 200
)
SELECT
  COUNT(*) AS TOTAL,
  COUNT(CASE WHEN HUMAN = AI THEN 1 END) AS MATCHES,
  COUNT(CASE WHEN HUMAN = AI THEN 1 END)/COUNT(*)*100 AS ACCURACY_PCT
FROM CLASSIFIED;
```

**Done-when:** Classification runs, accuracy metric computed.

---

### T058 — Cortex EXTRACT_ANSWER
**Do:**
```sql
-- Extract specific answers from unstructured ticket text
SELECT
  TICKET_ID,
  DESCRIPTION,
  SNOWFLAKE.CORTEX.EXTRACT_ANSWER(
    DESCRIPTION,
    'What system or component is affected?'
  ) AS AFFECTED_COMPONENT,
  SNOWFLAKE.CORTEX.EXTRACT_ANSWER(
    DESCRIPTION,
    'What is the user trying to do?'
  ) AS USER_INTENT,
  SNOWFLAKE.CORTEX.EXTRACT_ANSWER(
    DESCRIPTION,
    'Is this a performance issue?'
  ) AS IS_PERFORMANCE_ISSUE
FROM GLOBALTRADER_DB.RAW.SUPPORT_TICKETS_RAW
WHERE PRIORITY = 'P1'
LIMIT 30;
```

**Done-when:** EXTRACT_ANSWER runs on P1 tickets with 3 different questions.

---

### T059 — Cortex Search Setup
**Do:**
```sql
-- Create a Cortex Search Service on support tickets
CREATE CORTEX SEARCH SERVICE GLOBALTRADER_DB.RAW.TICKET_SEARCH
  ON DESCRIPTION
  ATTRIBUTES TICKET_ID, ACCOUNT_ID, PRIORITY, CATEGORY, PRODUCT_LINE
  WAREHOUSE = GLOBALTRADER_WH
  TARGET_LAG = '1 hour'
AS
SELECT TICKET_ID, ACCOUNT_ID, PRIORITY, CATEGORY, DESCRIPTION, PRODUCT_LINE
FROM GLOBALTRADER_DB.RAW.SUPPORT_TICKETS_RAW;

-- Query it using REST API (from Snowsight worksheet):
SELECT PARSE_JSON(
  SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
    'GLOBALTRADER_DB.RAW.TICKET_SEARCH',
    '{
      "query": "authentication timeout integration error",
      "columns": ["TICKET_ID","PRIORITY","DESCRIPTION"],
      "limit": 5
    }'
  )
) AS RESULTS;
```
2. Answer: *What is the difference between Cortex Search and a regular SQL LIKE filter? What is Cortex Search built on internally (semantic embeddings)? When would you NOT use Cortex Search?*

**Done-when:** Search service created, semantic search returns relevant tickets.

---

### T060 — Cortex Analyst Setup
**Do:**
1. Create a semantic model YAML for Cortex Analyst:
```yaml
# Save as semantic_model.yaml in Snowsight
name: GlobalTrader Sales Intelligence
tables:
  - name: FACT_SALES
    base_table: {database: GLOBALTRADER_DB, schema: MART, table: FACT_SALES}
    measures:
      - name: total_revenue
        expr: SUM(TOTAL_AMOUNT_USD)
        description: Total net revenue in USD
      - name: deal_count
        expr: COUNT(TRANSACTION_ID)
      - name: avg_discount_pct
        expr: AVG(DISCOUNT_PCT)
    dimensions:
      - name: product_line
        expr: PRODUCT_LINE
      - name: status
        expr: STATUS
    time_dimensions:
      - name: transaction_date
        expr: TRANSACTION_DATE
  - name: DIM_ACCOUNT
    base_table: {database: GLOBALTRADER_DB, schema: CLEAN, table: ACCOUNTS}
    dimensions:
      - name: industry
        expr: INDUSTRY
      - name: tier
        expr: TIER
      - name: region
        expr: REGION
relationships:
  - left_table: FACT_SALES
    right_table: DIM_ACCOUNT
    join_type: left_outer
    relationship_columns:
      - {left_column: ACCOUNT_ID, right_column: ACCOUNT_ID}
```
2. Upload the YAML to a stage and test with natural language questions in Snowsight
3. Ask it: "What was total revenue by region in Q4 2024?" and "Which product line had the highest average discount?"

**Done-when:** Cortex Analyst answers at least 3 natural language business questions correctly.

---

### T061 — Revenue Forecasting with ML_FORECAST
**Do:**
```sql
-- Prepare time series: daily revenue
CREATE TABLE GLOBALTRADER_DB.MART.DAILY_REVENUE AS
SELECT
  TRANSACTION_DATE,
  SUM(TOTAL_AMOUNT_USD) AS DAILY_REVENUE
FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
GROUP BY 1
ORDER BY 1;

-- Train forecast model
CREATE SNOWFLAKE.ML.FORECAST REVENUE_FORECAST (
  INPUT_DATA => SYSTEM$REFERENCE('TABLE','GLOBALTRADER_DB.MART.DAILY_REVENUE'),
  TIMESTAMP_COLNAME => 'TRANSACTION_DATE',
  TARGET_COLNAME => 'DAILY_REVENUE'
);

-- Generate 30-day forecast
CALL REVENUE_FORECAST!FORECAST(
  FORECASTING_PERIODS => 30,
  CONFIG_OBJECT => {'prediction_interval': 0.95}
);

-- Store results
CREATE TABLE GLOBALTRADER_DB.MART.REVENUE_FORECAST_RESULTS AS
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```
2. Answer: *What algorithm does ML_FORECAST use internally? What is a prediction interval? How do you evaluate forecast accuracy on historical data?*

**Done-when:** Forecast model trained, 30-day predictions stored, prediction intervals understood.

---

### T062 — Anomaly Detection with ML_ANOMALY_DETECTION
**Do:**
```sql
-- Detect anomalies in daily transaction volume
CREATE SNOWFLAKE.ML.ANOMALY_DETECTION TXN_ANOMALY_DETECTOR (
  INPUT_DATA => SYSTEM$REFERENCE('TABLE','GLOBALTRADER_DB.MART.DAILY_REVENUE'),
  SERIES_COLNAME => NULL,
  TIMESTAMP_COLNAME => 'TRANSACTION_DATE',
  TARGET_COLNAME => 'DAILY_REVENUE',
  LABEL_COLNAME => NULL
);

-- Run detection on recent data
CALL TXN_ANOMALY_DETECTOR!DETECT_ANOMALIES(
  INPUT_DATA => SYSTEM$REFERENCE('TABLE','GLOBALTRADER_DB.MART.DAILY_REVENUE'),
  TIMESTAMP_COLNAME => 'TRANSACTION_DATE',
  TARGET_COLNAME => 'DAILY_REVENUE',
  CONFIG_OBJECT => {'contamination': 0.05}
);

-- Store and review anomalies
CREATE TABLE GLOBALTRADER_DB.MART.REVENUE_ANOMALIES AS
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

SELECT * FROM GLOBALTRADER_DB.MART.REVENUE_ANOMALIES WHERE IS_ANOMALY = TRUE;
```

**Done-when:** Anomaly detection run, anomalous days identified and stored.

---

### T063 — Cortex Complete for Business Narratives
**Do:**
```sql
-- Generate executive commentary from data
WITH QUARTERLY_DATA AS (
  SELECT
    FISCAL_QUARTER_LABEL,
    FISCAL_YEAR,
    SUM(TOTAL_AMOUNT_USD)   AS REVENUE,
    COUNT(TRANSACTION_ID)   AS DEALS,
    AVG(DISCOUNT_PCT)       AS AVG_DISCOUNT
  FROM GLOBALTRADER_DB.CLEAN.SALES_TRANSACTIONS
  WHERE FISCAL_YEAR = 2024
  GROUP BY 1,2
)
SELECT
  FISCAL_QUARTER_LABEL,
  REVENUE,
  SNOWFLAKE.CORTEX.COMPLETE(
    'mistral-7b',
    'Write a 2-sentence executive summary for a sales quarter with the following metrics: ' ||
    'Revenue: $' || REVENUE::VARCHAR ||
    ', Deals closed: ' || DEALS::VARCHAR ||
    ', Average discount: ' || ROUND(AVG_DISCOUNT,1)::VARCHAR || '%. ' ||
    'Be concise and professional.'
  ) AS EXECUTIVE_SUMMARY
FROM QUARTERLY_DATA
ORDER BY FISCAL_YEAR, FISCAL_QUARTER_LABEL;
```

**Done-when:** AI-generated commentary produced for each quarter of 2024 data.

---

### T064 — Document AI (Unstructured Data)
**Do:**
1. Upload one of your CSV files to a stage and treat it as a "document"
2. Use Cortex Document AI to extract structured fields:
```sql
-- Create a Document AI model (Snowsight: AI & ML > Document AI)
-- Build a model to extract: invoice_number, amount, due_date, status from invoices

-- Once model is built, test:
SELECT GLOBALTRADER_DB.RAW.INVOICE_EXTRACTOR!PREDICT(
  GET_PRESIGNED_URL('@GLOBALTRADER_DB.STAGING.INTERNAL_STAGE','03_expense_claims.csv'),
  1
);
```
2. Answer: *What document types does Cortex Document AI support? What is the difference between Document AI and EXTRACT_ANSWER?*

**Done-when:** Document AI model built and tested. Difference from EXTRACT_ANSWER explained.

---

### T065 — Cortex Cost Analysis
**Do:**
```sql
-- Track Cortex AI credit consumption
SELECT
  START_TIME::DATE AS USAGE_DATE,
  SERVICE_TYPE,
  SERVICE_NAME,
  CREDITS_USED
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY
WHERE SERVICE_TYPE LIKE '%CORTEX%'
  AND START_TIME >= DATEADD(DAY,-7,CURRENT_TIMESTAMP())
ORDER BY CREDITS_USED DESC;
```
2. Build a cost estimate spreadsheet: how many SENTIMENT() calls can you make on $5 of credits?
3. Answer: *Which Cortex functions are cheapest? Which are most expensive? How does model size affect cost?*

**Done-when:** Credit consumption tracked, cost estimate built, you can explain Cortex pricing tiers.

---

## Phase 7 — Snowpark + dbt Native (Tasks T066–T075)
> **Goal:** Modern Snowflake engineers need dbt. Snowpark gives Python-native access. SQL-first approach.

---

### T066 — Snowpark Introduction (Python in Snowflake)
**Do:**
```sql
-- Create a Snowpark stored procedure (Python)
CREATE OR REPLACE PROCEDURE GLOBALTRADER_DB.RAW.SP_PYTHON_CLEAN_LEADS()
RETURNS VARCHAR
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python')
HANDLER = 'run'
AS $$
def run(session):
    from snowflake.snowpark.functions import col, upper, trim, when, lit

    # Load marketing leads
    df = session.table('GLOBALTRADER_DB.CLEAN.MARKETING_LEADS')

    # Apply transformations using Snowpark DataFrame API
    cleaned = df.filter(col('EMAIL_VALID') == True) \
                .filter(col('LEAD_SCORE').isNotNull()) \
                .with_column('COMPANY_UPPER', upper(trim(col('COMPANY')))) \
                .with_column('PRIORITY_TIER',
                    when(col('LEAD_SCORE') >= 80, lit('HOT'))
                    .when(col('LEAD_SCORE') >= 50, lit('WARM'))
                    .otherwise(lit('COLD'))
                )

    row_count = cleaned.count()

    # Write to new table
    cleaned.write.mode('overwrite').save_as_table('GLOBALTRADER_DB.CLEAN.QUALIFIED_LEADS')

    return f'Wrote {row_count} qualified leads'
$$;

CALL GLOBALTRADER_DB.RAW.SP_PYTHON_CLEAN_LEADS();
SELECT COUNT(*) FROM GLOBALTRADER_DB.CLEAN.QUALIFIED_LEADS;
```
2. Answer: *What is Snowpark? Is it running Python on your machine or in Snowflake? What is a Snowpark DataFrame vs a pandas DataFrame?*

**Done-when:** Python SP runs, QUALIFIED_LEADS table created, Snowpark concept explained.

---

### T067 — Snowpark UDTFs (Table Functions)
**Do:**
```sql
-- UDTF: expand comma-separated product regions into rows
CREATE OR REPLACE FUNCTION GLOBALTRADER_DB.RAW.SPLIT_REGIONS(REGIONS VARCHAR)
RETURNS TABLE (REGION VARCHAR)
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python')
HANDLER = 'SplitRegions'
AS $$
class SplitRegions:
    def process(self, regions):
        if regions:
            for r in regions.split(','):
                yield (r.strip(),)
    def end_partition(self):
        pass
$$;

-- Use it:
SELECT p.PRODUCT_ID, p.PRODUCT_NAME, r.REGION
FROM GLOBALTRADER_DB.RAW.PRODUCTS_RAW p,
TABLE(GLOBALTRADER_DB.RAW.SPLIT_REGIONS(p.SUPPORTED_REGIONS)) r;
```
2. Answer: *What is the difference between a UDF and a UDTF? What is the HANDLER class structure requirement?*

**Done-when:** UDTF splits region strings into rows correctly.

---

### T068 — dbt Core Setup on Snowflake
**Do:**
1. Install dbt-core and dbt-snowflake in your local Python environment:
```bash
pip install dbt-core dbt-snowflake
dbt --version
```
2. Initialize a dbt project:
```bash
dbt init globaltrader_dbt
# Configure profiles.yml with your Snowflake trial credentials
```
3. Configure `profiles.yml`:
```yaml
globaltrader_dbt:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: <your_account>
      user: <your_user>
      password: <your_password>
      role: SYSADMIN
      database: GLOBALTRADER_DB
      warehouse: GLOBALTRADER_WH
      schema: DBT_DEV
      threads: 4
```
4. Run `dbt debug` — confirm connection works

**Done-when:** `dbt debug` passes all checks, project structure created.

---

### T069 — dbt Sources & Staging Models
**Do:**
1. Create `models/sources.yml`:
```yaml
version: 2
sources:
  - name: clean
    database: GLOBALTRADER_DB
    schema: CLEAN
    tables:
      - name: accounts
      - name: sales_transactions
      - name: marketing_leads
```
2. Create staging models:
```sql
-- models/staging/stg_accounts.sql
SELECT
  account_id,
  account_name,
  industry,
  region,
  tier,
  status,
  annual_revenue_usd,
  created_date,
  _loaded_at
FROM {{ source('clean', 'accounts') }}
WHERE status != 'CHURNED'
```
3. Run: `dbt run --select stg_accounts`

**Done-when:** Staging model materializes in Snowflake as a view in DBT_DEV schema.

---

### T070 — dbt Intermediate & Mart Models
**Do:**
1. Create an intermediate model:
```sql
-- models/intermediate/int_sales_enriched.sql
SELECT
  t.transaction_id,
  t.account_id,
  t.rep_id,
  t.total_amount_usd,
  t.transaction_date,
  t.fiscal_quarter_label,
  t.fiscal_year,
  a.account_name,
  a.industry,
  a.tier,
  a.region
FROM {{ ref('stg_sales_transactions') }} t
LEFT JOIN {{ ref('stg_accounts') }} a ON t.account_id = a.account_id
```
2. Create a mart model (materialized as TABLE):
```sql
-- models/mart/mart_rep_performance.sql
-- {{ config(materialized='table') }}
SELECT
  rep_id,
  fiscal_quarter_label,
  fiscal_year,
  COUNT(transaction_id)   AS deal_count,
  SUM(total_amount_usd)   AS total_revenue,
  AVG(total_amount_usd)   AS avg_deal_size
FROM {{ ref('int_sales_enriched') }}
GROUP BY 1,2,3
```
3. Run full: `dbt run`
4. Run: `dbt docs generate && dbt docs serve`

**Done-when:** Full dbt DAG runs (staging → intermediate → mart), docs site shows lineage.

---

### T071 — dbt Tests & Data Quality
**Do:**
1. Add tests to `models/staging/schema.yml`:
```yaml
version: 2
models:
  - name: stg_accounts
    columns:
      - name: account_id
        tests:
          - not_null
          - unique
      - name: status
        tests:
          - accepted_values:
              values: ['ACTIVE','CHURNED','PROSPECT']
      - name: annual_revenue_usd
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```
2. Run: `dbt test`
3. Answer: *What happens when a dbt test fails — does the whole run stop? What is the difference between warn and error severity?*

**Done-when:** Tests run, at least one fails (your dirty data ensures this), severity levels understood.

---

### T072 — dbt Incremental Models
**Do:**
```sql
-- models/mart/mart_daily_revenue.sql
-- {{ config(materialized='incremental', unique_key='transaction_date') }}

SELECT
  transaction_date,
  SUM(total_amount_usd) AS daily_revenue,
  COUNT(transaction_id) AS deal_count
FROM {{ ref('stg_sales_transactions') }}

{% if is_incremental() %}
  WHERE transaction_date > (SELECT MAX(transaction_date) FROM {{ this }})
{% endif %}

GROUP BY 1
```
2. Run first time (full load): `dbt run --select mart_daily_revenue`
3. Run second time (incremental): `dbt run --select mart_daily_revenue`
4. Check that only new rows were added on second run
5. Answer: *What are the 4 materialization types in dbt? What does unique_key do in an incremental model? What happens on delete in an incremental model?*

**Done-when:** Incremental model runs twice, second run processes only new data.

---

### T073 — dbt Snapshots (SCD2 in dbt)
**Do:**
```sql
-- snapshots/accounts_snapshot.sql
{% snapshot accounts_snapshot %}
{{
    config(
      target_schema='SNAPSHOTS',
      unique_key='account_id',
      strategy='check',
      check_cols=['status', 'tier', 'annual_revenue_usd', 'account_owner_id']
    )
}}
SELECT account_id, account_name, status, tier, annual_revenue_usd, account_owner_id
FROM {{ source('clean', 'accounts') }}
{% endsnapshot %}
```
2. Run: `dbt snapshot`
3. Change some account statuses in the source table
4. Run `dbt snapshot` again
5. Query the snapshot table — it should have both old and new records with dbt_valid_from/dbt_valid_to

**Done-when:** Snapshot captures historical changes. You can query point-in-time state via dbt_valid_from/to.

---

### T074 — Semantic Views (Snowflake Native Semantic Layer)
**Do:**
```sql
-- Create a Snowflake native Semantic View
CREATE SEMANTIC VIEW GLOBALTRADER_DB.MART.SALES_SEMANTIC_VIEW
  TABLES (
    GLOBALTRADER_DB.MART.FACT_SALES AS FACT_SALES
      PRIMARY KEY (TRANSACTION_ID),
    GLOBALTRADER_DB.CLEAN.ACCOUNTS AS DIM_ACCOUNT
      PRIMARY KEY (ACCOUNT_ID),
    GLOBALTRADER_DB.RAW.SALES_REPS AS DIM_REP
      PRIMARY KEY (REP_ID)
  )
  RELATIONSHIPS (
    FACT_SALES (ACCOUNT_ID) REFERENCES DIM_ACCOUNT (ACCOUNT_ID),
    FACT_SALES (REP_ID) REFERENCES DIM_REP (REP_ID)
  )
  FACTS (
    FACT_SALES.TOTAL_AMOUNT_USD AS total_revenue,
    FACT_SALES.DISCOUNT_AMOUNT_USD AS total_discounts,
    FACT_SALES.QUANTITY AS units_sold
  )
  DIMENSIONS (
    DIM_ACCOUNT.INDUSTRY AS industry,
    DIM_ACCOUNT.TIER AS account_tier,
    DIM_ACCOUNT.REGION AS region,
    DIM_REP.FULL_NAME AS rep_name,
    DIM_REP.LEVEL AS rep_level
  );

-- Query using semantic layer
SELECT industry, SUM(total_revenue) AS revenue
FROM GLOBALTRADER_DB.MART.SALES_SEMANTIC_VIEW
GROUP BY industry;
```
2. Answer: *What is the difference between a Semantic View and a regular view? How does Cortex Analyst use Semantic Views? What problem does a Semantic Layer solve for business users?*

**Done-when:** Semantic View queryable, Cortex Analyst pointed at it for natural language queries.

---

### T075 — Cortex Code Integration
**Do:**
1. In Snowsight, open Cortex Code (AI & ML > Cortex Code)
2. Prompt it: *"Write a Snowflake SQL query to show the top 5 accounts by revenue in 2024, including their industry, tier, and the percentage of total revenue they represent"*
3. Review the generated SQL, run it, fix any errors
4. Prompt it: *"Migrate this Databricks PySpark transformation to Snowflake SQL: [paste a simple PySpark script]"*
5. Prompt it: *"Optimize this Snowflake query for better partition pruning: [paste your bad query from T046]"*
6. Answer: *What is Cortex Code? Is it the same as GitHub Copilot? What data does it have access to when running inside Snowflake?*

**Done-when:** 3 prompts used, generated SQL runs correctly, limitations understood.

---

## Phase 8 — Interview Gauntlet (Tasks T076–T090)
> **Indian-style:** Hard scenarios first. You write the answer in SQL/config BEFORE checking. This is how senior interviewers actually test.

---

### T076 — Scenario: Pipeline Design Question
**Interview Q:** *"Design a near-real-time pipeline that ingests 500 CSV files per hour from S3, validates data quality, applies SCD Type 2 to a customer dimension, and makes data available to a BI tool within 10 minutes of file arrival. What Snowflake features do you use at each step?"*

**Do:** Write a full architecture doc as SQL comments in a script file. Include:
- Ingestion layer (Snowpipe vs COPY INTO and why)
- Staging layer design
- Validation step (Streams + Tasks or Dynamic Tables and why)
- SCD2 implementation
- Gold layer materialization choice
- BI tool connection (Snowflake Partner Connect)
- Monitoring and alerting

**Done-when:** Architecture documented with specific Snowflake feature choices and reasoning.

---

### T077 — Scenario: Performance Investigation
**Interview Q:** *"A query that used to take 30 seconds now takes 8 minutes after a data load last week. Walk me through how you diagnose and fix this."*

**Do:** 
1. Simulate the problem: load 500,000 rows into a table WITHOUT clustering, run a date-filtered query
2. Check QUERY_HISTORY for the slow query — what changed?
3. Check SYSTEM$CLUSTERING_INFORMATION — is the table over-clustered or poorly clustered?
4. Check if a new large join was added to the query
5. Check warehouse credits — did compute tier change?
6. Fix: add clustering key, rebuild statistics, or rewrite the query
7. Document your diagnostic steps in order

**Done-when:** You have a written runbook for Snowflake query performance regression.

---

### T078 — Scenario: Cost Spike Investigation
**Interview Q:** *"Your Snowflake bill doubled this month. Your manager asks you to find the root cause and fix it by end of day. What do you do?"*

**Do:**
```sql
-- Step 1: Find which warehouses consumed the most credits
SELECT WAREHOUSE_NAME, SUM(CREDITS_USED) AS TOTAL_CREDITS
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATE_TRUNC('MONTH',CURRENT_DATE())
GROUP BY 1 ORDER BY 2 DESC;

-- Step 2: Find which users/queries drove the spike
-- Step 3: Check if a warehouse was left running (no AUTO_SUSPEND)
-- Step 4: Check for long-running queries with no result cache
-- Step 5: Check storage growth (Time Travel bloat, transient vs permanent tables)
-- Step 6: Build a resource monitor + alert
```

**Done-when:** Full diagnostic written. Resource monitor in place. You can present this to a manager.

---

### T079 — Write These From Memory (No Hints)
**Do:** Without looking at previous tasks, write SQL from memory for each:
1. SCD2 MERGE statement (insert new record + expire old one in a single MERGE)
2. Stream + Task pipeline (3 steps: stream → staging → mart)
3. Row access policy based on current user's region
4. Window function: running total + previous period comparison + rank within group (all in one query)
5. QUALIFY to find each account's latest transaction
6. COPY INTO with ON_ERROR=CONTINUE + VALIDATION_MODE
7. SYSTEM$CLUSTERING_INFORMATION and how to interpret depth vs overlap

**Done-when:** All 7 written from memory. Verified they run. Fix any errors yourself.

---

### T080 — Basic Questions Interviewers Use to Confuse (AFTER hard questions)
**Interview pattern:** They ask hard architecture questions, you ace them, then they ask these to see if you get overconfident and make silly mistakes.

**Do:** Answer ALL of the following in a single SQL file as comments — write the answer, then verify it:

1. What is the DEFAULT AUTO_SUSPEND setting for a new warehouse? (60 seconds — verify with SHOW WAREHOUSES)
2. Can you query a STREAM twice in the same transaction and get data both times? (No — explain why)
3. What happens to child tasks if you SUSPEND the root task? (They keep running until their own schedule — trick question)
4. What is the maximum size of a VARIANT column? (16MB compressed — what happens if you exceed it?)
5. What are the 3 types of internal stages? (User, Table, Named — what's the difference?)
6. Can a task call a stored procedure? (Yes — show the syntax)
7. What is FAIL-SAFE and can you access it yourself? (No — only Snowflake support can recover Fail-safe data)
8. What does COPY INTO return when VALIDATION_MODE = RETURN_ALL_ERRORS? (A result set — does it load data? No.)
9. What is the difference between TRUNCATE and DELETE in Snowflake regarding Time Travel? (TRUNCATE resets Time Travel for the table — TRICK: actually no, both are DML and both are trackable)
10. How do you share a table with a non-Snowflake user? (Use Snowflake Marketplace or generate a signed URL from a stage)

**Done-when:** All 10 answered with reasoning. Verified against Snowflake docs or tested.

---

### T081 — Live Coding: Complex Compensation Analysis
**Interview Q:** *"Write a query to show each sales rep's quarterly compensation breakdown: base salary, commissions earned, accelerator applied, clawback deducted, SPIFF, and total comp. Include the prior quarter's total comp for comparison, and flag any rep where total comp decreased more than 20% quarter-over-quarter."*

**Do:** Write this query from scratch against your COMPENSATION table. It should use:
- Window functions (LAG for prior quarter)
- CASE WHEN for accelerator tier logic
- Multiple aggregation columns
- A QoQ change calculation
- A flag column for >20% drop

**Done-when:** Query runs and returns one row per rep per quarter with all columns, QoQ flag works.

---

### T082 — Live Coding: AR Aging Report
**Interview Q:** *"Build a complete AR aging report showing outstanding invoices bucketed into: Current, 1-30 days, 31-60 days, 61-90 days, 90+ days. Show totals by industry and a grand total row. Flag any industry where over 20% of AR is in the 90+ bucket."*

**Do:**
```sql
-- Write this from scratch. Use ROLLUP for grand total.
-- Use GROUPING() to label the grand total row.
-- Use a HAVING or QUALIFY to flag high-risk industries.
```

**Done-when:** Full AR aging report with ROLLUP grand total, industry-level risk flag.

---

### T083 — Live Coding: Pipeline Debugging
**Do:**
1. Intentionally break your Stream+Task pipeline (drop the target table)
2. Let the task fail
3. Diagnose the failure:
```sql
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
  TASK_NAME => 'PROCESS_ACCOUNTS_STREAM'
))
WHERE STATE = 'FAILED'
ORDER BY SCHEDULED_TIME DESC LIMIT 5;
```
4. Fix the root cause (recreate the table)
5. Resume the task and verify it processes the stream backlog

**Done-when:** Full break/diagnose/fix cycle completed. You know how to find task errors.

---

### T084 — System Design: Multi-Tenant Architecture
**Interview Q:** *"You're designing a Snowflake architecture for a SaaS company with 200 enterprise customers. Each customer's data must be completely isolated — they must never see each other's data. How do you design this in Snowflake?"*

**Do:** Document 3 possible approaches as SQL comments:
- **Option A:** One database per customer (200 databases)
- **Option B:** One schema per customer (1 database, 200 schemas)  
- **Option C:** Shared tables with row access policies (1 schema, tenant_id column)

For each: write the pros, cons, scaling implications, and how RBAC/row access policies work.
Recommend one and justify.

**Done-when:** All 3 approaches documented with implementation notes, recommendation justified.

---

### T085 — Snowflake vs Other Platforms (Interview Classic)
**Do:** Write a comparison document (as SQL comments or a markdown file) answering:
1. Snowflake vs Databricks: when would you choose each?
2. Snowflake vs Redshift: key architectural differences
3. Snowflake vs BigQuery: pricing model differences
4. When would Snowflake be the WRONG choice?

Include specific Snowflake features that competitors lack and vice versa.

**Done-when:** Comparison written. You can discuss this fluently for 5 minutes in an interview.

---

## Phase 9 — SnowPro Core Prep (Tasks T086–T090)
> **Goal:** Certify your knowledge. SnowPro Core is the gateway credential.

---

### T086 — SnowPro Core: Architecture & Storage Domain
**Do:** Answer all 15 questions below in a SQL file as comments. Then verify each:
1. What is a micro-partition size range? (50-500MB uncompressed)
2. What is the default Time Travel for Standard edition? (1 day)
3. What is the default Time Travel for Enterprise edition? (up to 90 days, default 1 day)
4. What are the 3 Snowflake editions? (Standard, Enterprise, Business Critical)
5. What is Tri-Secret Secure and which edition supports it?
6. What storage format does Snowflake use internally? (columnar, compressed, immutable files)
7. What is a Virtual Private Snowflake (VPS)?
8. What cloud providers does Snowflake support?
9. What is the Snowflake Marketplace?
10. What are the 4 types of Snowflake tables?
11. What is FAIL-SAFE retention period? (7 days, non-configurable)
12. What happens to Time Travel data when retention period expires?
13. What is zero-copy cloning and does it copy the data?
14. What columns does a Stream add to captured rows? (METADATA$ACTION, METADATA$ISUPDATE, METADATA$ROW_ID)
15. Can you create a Stream on a View? (Yes — only certain types)

**Done-when:** All 15 answered correctly from memory.

---

### T087 — SnowPro Core: Compute & Performance Domain
**Do:** Answer all 12:
1. What is a credit? How much does an X-Small warehouse consume per hour?
2. What is the relationship between warehouse size and query parallelism? (Doubles with each size)
3. What triggers auto-resume on a warehouse?
4. What is multi-cluster warehouse and when does scale-out trigger?
5. What are the 3 cache layers and their sizes/lifetimes?
6. What is the result cache sharing scope? (Account-level, 24 hours, exact query match)
7. What invalidates result cache? (Table data change, DDL, time expiry)
8. What is a "spillover to local disk"? What does it indicate?
9. What is Query Profile and where do you find it?
10. What does SYSTEM$CLUSTERING_INFORMATION return? Explain depth and overlap.
11. What is the Search Optimization Service and when is it worth the cost?
12. What is the difference between EXPLAIN USING TEXT and EXPLAIN USING JSON?

**Done-when:** All 12 answered with correct detail.

---

### T088 — SnowPro Core: Security & Governance Domain
**Do:** Answer all 10:
1. What are the system-defined roles? (ORGADMIN, ACCOUNTADMIN, SYSADMIN, SECURITYADMIN, USERADMIN, PUBLIC)
2. What is the role hierarchy and privilege inheritance direction?
3. What is the difference between a Masking Policy and a Row Access Policy?
4. Can you apply both masking and row access policies to the same column? (Yes)
5. What is a Secure View vs a regular view?
6. What is ACCOUNTADMIN role allowed to do that SYSADMIN cannot?
7. What is network policy and at what levels can you apply it?
8. What is Tri-Secret Secure?
9. What is the difference between ACCESS_HISTORY and QUERY_HISTORY in ACCOUNT_USAGE?
10. What is data classification in Snowflake? Is it automatic or manual?

**Done-when:** All 10 answered correctly.

---

### T089 — SnowPro Core: Data Loading & Semi-Structured Domain
**Do:** Write SQL to demonstrate all of these, then answer the questions:
```sql
-- FLATTEN with LATERAL
-- TRY_CAST and TRY_TO_DATE
-- PARSE_JSON and BUILD_OBJECT
-- ARRAY_AGG and ARRAY_CONTAINS
-- OBJECT_CONSTRUCT
-- GET_PATH function
-- TYPEOF function on a VARIANT column
```
Answer:
1. What is the maximum VARIANT column size?
2. What file formats does Snowflake's COPY INTO support natively?
3. What is the difference between COPY INTO ... FROM @stage and INSERT INTO ... SELECT FROM @stage?
4. What does STRIP_OUTER_ARRAY do in a JSON file format?

**Done-when:** All SQL demonstrated, questions answered.

---

### T090 — Mock Exam Simulation
**Do:**
1. Go to https://certificationpractice.com (free practice questions)
2. Take a full 65-question mock exam under timed conditions (90 minutes)
3. Target: 85%+ before attempting real exam
4. For every wrong answer: write the correct answer + explanation in a SQL file as a comment
5. Identify your 3 weakest domains and revisit those task sections

**Done-when:** Mock exam completed, score recorded, weakness areas identified and reviewed.

---

## Phase 10 — Capstone Project (Tasks T091–T095)
> **Goal:** A single end-to-end project that demonstrates everything. This is what you demo in interviews.

---

### T091 — Capstone Design & Architecture
**Capstone:** Build a **GlobalTrader Inc. Enterprise Sales Intelligence Platform** with:
- Multi-source ingestion (CSV + SQL-generated data)
- Full Bronze → Silver → Gold medallion pipeline
- SCD2 on accounts and reps
- Streams + Tasks for CDC
- Dynamic Tables for near-real-time aggregations
- Compensation analysis with clawback detection
- AR aging with overdue risk flags
- Cortex AI: sentiment on tickets + revenue forecast + anomaly detection
- dbt for Gold layer transformation
- Semantic View for Cortex Analyst
- Row access policies + masking for PII
- Resource monitor + cost dashboard
- ACCOUNT_USAGE governance report

**Do:** Create a `CAPSTONE.md` documenting:
- Architecture diagram (ASCII or Mermaid)
- List of all objects created (tables, views, tasks, streams, policies)
- Business questions the platform can answer
- How to run the full pipeline end-to-end

**Done-when:** CAPSTONE.md complete, architecture documented.

---

### T092 — Capstone Build: Pipeline Layer
**Do:** Build and wire up the full pipeline:
1. Ensure all 8 SQL tables and 5 CSVs are loaded (Phase 0)
2. Run all CLEAN layer transformations (Phase 1)
3. Activate Stream + Task pipeline for ACCOUNTS and SALES_TRANSACTIONS (Phase 2)
4. Build complete MART layer including FACT_SALES, DIM_DATE, MV_REP_QUARTERLY_SUMMARY (Phase 3)
5. Verify end-to-end: insert a new account in RAW → confirm it appears in MART within 5 minutes

**Done-when:** Full pipeline verified end-to-end with a new record flowing through all layers.

---

### T093 — Capstone Build: AI & Analytics Layer
**Do:**
1. Run SENTIMENT on all 5,000 support tickets, store in MART.TICKET_SENTIMENT
2. Run ML_FORECAST on daily revenue, store 30-day forecast
3. Run ANOMALY_DETECTION on daily transaction volume, store results
4. Build Cortex Analyst semantic model covering all Gold layer tables
5. Test: ask Cortex Analyst 5 business questions and verify answers

**Done-when:** All 3 AI outputs stored, Cortex Analyst answers 5 questions correctly.

---

### T094 — Capstone Build: Governance Layer
**Do:**
1. All PII columns tagged (email, phone, name, salary)
2. Masking policies applied for email and phone
3. Row access policies applied for region-based account visibility
4. Resource monitor set at 80% notify, 100% suspend
5. ACCOUNT_USAGE governance dashboard query that shows: all tables, their row counts, storage size, masking policies, row access policies, PII tags — in one output

**Done-when:** Governance query produces complete inventory. Masking and RAP verified with role switching.

---

### T095 — Capstone Demo Script & GitHub Cleanup
**Do:**
1. Write a `DEMO_SCRIPT.md` — a 10-minute walkthrough you would give an interviewer:
   - Start with architecture overview (2 min)
   - Show live pipeline: insert a record, show it flowing through Bronze → Silver → Gold (3 min)
   - Show Cortex AI: run sentiment query live, show forecast chart (3 min)
   - Show governance: switch roles, show masking in action (2 min)
2. Clean up your GitHub repo:
   - README.md with architecture, setup instructions, and what each file does
   - All SQL organized into numbered folders by phase
   - TASKS.md with checkboxes
   - CAPSTONE.md
   - DEMO_SCRIPT.md
3. Push everything to `ereshadul/SnowflakesLearning`

**Done-when:** GitHub repo is interview-ready. You can run the 10-minute demo without notes.

---

## Quick Reference: Interview Question Map

| Topic | Tasks | Frequency |
|---|---|---|
| Micro-partitions & pruning | T001, T012, T043 | ⭐⭐⭐⭐⭐ |
| Streams & Tasks | T021, T022, T023, T079 | ⭐⭐⭐⭐⭐ |
| SCD Type 2 | T025, T073, T079 | ⭐⭐⭐⭐⭐ |
| Caching (all 3 layers) | T009, T087 | ⭐⭐⭐⭐⭐ |
| Clustering keys | T012, T033, T043 | ⭐⭐⭐⭐⭐ |
| Window functions | T018, T019, T081 | ⭐⭐⭐⭐⭐ |
| MERGE statement | T020, T025, T079 | ⭐⭐⭐⭐ |
| Snowpipe vs COPY INTO | T024, T029 | ⭐⭐⭐⭐ |
| Masking + Row Access Policies | T047, T048, T088 | ⭐⭐⭐⭐ |
| Cost & resource monitors | T042, T045, T078 | ⭐⭐⭐⭐ |
| Dynamic Tables vs Streams | T026, T087 | ⭐⭐⭐ |
| Cortex AI features | T055–T065 | ⭐⭐⭐ |
| dbt + Snowflake | T068–T073 | ⭐⭐⭐ |
| Snowpark | T066, T067 | ⭐⭐ |
| Time Travel & Fail-safe | T008, T053, T086 | ⭐⭐⭐⭐ |
