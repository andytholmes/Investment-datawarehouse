# Investment Data Warehouse Project

## Comprehensive Vision, Architecture & Roadmap

**Document Version:** 1.0  
**Date:** February 6, 2026  
**Project Type:** Snowflake-based Investment Data Warehouse with Star Schema  
**Region:** Azure UK

-----

## Executive Summary

### Project Overview

This project delivers a modern, cloud-native investment data warehouse on Snowflake Enterprise (Azure UK) that provides portfolio managers and investment teams with timely, accurate position data enriched with security reference information and benchmark analytics.

### Business Value

- **Primary Benefit:** Enable portfolio managers to perform trend analysis and aggregations across portfolio positions
- **Key Capability:** Self-service reporting via custom API-driven pivot table interface with agentic AI query generation
- **Strategic Impact:** Centralized single source of truth for investment data with 1-5 second query performance

### Success Criteria

- Position valuations and benchmark data populating accurately within 1 minute of ingestion
- All positions linked to security and fund reference data
- Data quality checks operational with monitoring dashboard
- Portfolio managers able to query 10 years of historical position data
- Query performance: 1-5 seconds for typical aggregations
- Data accuracy: Position/NAV reconciliation within 0.009% tolerance

### Timeline

**Phase 1 MVP: 2 weeks from kickoff**

- 5 pilot funds with EOD positions and benchmarks
- Core star schema with security and fund dimensions
- Basic data quality framework

**Phase 2 Production: 2 weeks after MVP (Week 5)**

- All 100 funds
- Trade data integration (Charles River)
- Intraday position updates (SimCorp)
- Analytics integration (S&P)
- FX rates (Reuters)

-----

## Table of Contents

1. Business Context
1. Source Systems & Data Landscape
1. Technical Architecture
1. Star Schema Design
1. Data Quality & Validation Framework
1. Real-Time MQ Integration
1. Transformation Strategy (Bronze/Silver/Gold)
1. Infrastructure & DevOps
1. Performance & Optimization
1. Implementation Roadmap
1. Risk Management

-----

## 1. Business Context

### Business Problem

Investment teams currently lack a centralized data warehouse that provides:

- Timely access to portfolio position data with comprehensive security reference
- Historical trending and time-series analysis capabilities
- Integrated benchmark data for performance comparison
- Self-service analytics without dependency on manual data extraction

### Key Decisions Supported

Portfolio managers require granular position-level data with analytics and benchmarks to:

- Track key portfolio metrics (weights, exposures, active positions)
- Adjust investment strategies based on performance trends
- Analyze sector, geographic, and security-type exposures over time
- Compare portfolio positioning against benchmarks

### Primary End Users

1. **Portfolio Managers** - Daily position analysis and trend reporting
1. **Client Reporting & Marketing Teams** - Monthly/quarterly performance reporting
1. **Risk Teams** - Portfolio risk monitoring and active risk analysis
1. **Attribution Teams** - Performance attribution analysis
1. **Regulatory Reporting Teams** - Compliance and regulatory submissions

### Success Metrics

|Metric                     |Target                       |
|---------------------------|-----------------------------|
|Data Freshness             |Within 1 minute of ingestion |
|Query Performance          |1-5 seconds (95th percentile)|
|Position/NAV Reconciliation|Within 0.009%                |
|Benchmark Weight Validation|Sum to 100% ±0.009%          |
|Concurrent Users           |20 users                     |

-----

## 2. Source Systems & Data Landscape

### Source System Inventory

#### SimCorp Dimension

**Purpose:** Primary portfolio accounting system  
**Data:** Positions, valuations, cash, FX spot  
**Volume:** ~50,000 positions across 100 portfolios daily  
**Format:** CSV (SOD T-1 batch), MQ messages (intraday)  
**Update Frequency:**

- SOD batch: Daily T-1 positions
- Intraday: Continuous updates during trading

#### IVP (Integrated Valuation Platform)

**Purpose:** Security master and reference data  
**Data:** Security identifiers, classifications, attributes  
**Volume:** ~100 daily (standard), ~10,000 (month-end)  
**Format:** JSON via MQ  
**Update Frequency:** Real-time (full security record)

#### Rimes

**Purpose:** Benchmark data provider  
**Data:** Benchmark constituents, weights  
**Volume:** ~300,000 positions across 50 benchmarks daily  
**Format:** CSV files  
**Update Frequency:** Daily SOD T-1 batch

#### Charles River (Phase 2)

**Purpose:** Order and execution management  
**Data:** Trade executions, allocations  
**Volume:** ~1,000s daily  
**Format:** JSON via MQ  
**Update Frequency:** Real-time (<1 minute latency)

#### S&P Analytics (Phase 2)

**Purpose:** Security analytics and risk metrics  
**Format:** TBC  
**Update Frequency:** Daily

#### Reuters FX Rates (Phase 2)

**Purpose:** Foreign exchange rates  
**Format:** TBC  
**Update Frequency:** Daily

### Data Volumes & Growth

|Data Type          |Current Daily Volume|10-Year Retention|Projected Growth     |
|-------------------|--------------------|-----------------|---------------------|
|Portfolio Positions|50,000 rows         |183M rows        |10-50% annually      |
|Benchmark Positions|300,000 rows        |1.1B rows        |10-50% annually      |
|Trades             |~1,000s rows        |TBD              |Scalable architecture|

-----

## 3. Technical Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    SOURCE SYSTEMS                        │
│  SimCorp | IVP | Rimes | Charles River                  │
│  CSV/MQ  | MQ  | CSV   | MQ                              │
└────┬──────┬──────┬──────┬───────────────────────────────┘
     │      │      │      │
     │  MQ Consumers (Python) | File Ingestion (Airflow)
     │      │      │      │
     └──────┴──────┴──────┴─────────────────┐
                                             │
┌────────────────────────────────────────────┴─────────────┐
│              SNOWFLAKE (Azure UK Enterprise)             │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  BRONZE LAYER (Raw Data)                                 │
│    - SIMCORP_RAW, IVP_RAW, RIMES_RAW                    │
│    - Immutable source copies                             │
│                                                           │
│  ↓ dbt transformations                                   │
│                                                           │
│  SILVER LAYER (Cleansed & Conformed)                     │
│    - POSITIONS, SECURITIES, BENCHMARKS                   │
│    - Typed, validated, deduplicated                      │
│                                                           │
│  ↓ dbt transformations                                   │
│                                                           │
│  GOLD LAYER (Star Schema)                                │
│    FACTS:                                                │
│      - Daily_Portfolio_Positions                         │
│      - Portfolio_NAV                                     │
│      - Benchmark_Positions                               │
│    DIMENSIONS:                                           │
│      - Dim_Security (SCD Type 2)                        │
│      - Dim_Fund (SCD Type 2)                            │
│      - Dim_Benchmark, Dim_Date, etc.                    │
│                                                           │
│  MATERIALIZED VIEWS:                                     │
│    - Fund daily summaries                                │
│    - Sector exposures                                    │
│                                                           │
└──────────────────┬───────────────────────────────────────┘
                   │
┌──────────────────┴───────────────────────────────────────┐
│             CONSUMPTION LAYER                             │
│  Custom API (Agentic AI Query Generation)                │
│  → Pivot Table Interface                                 │
│  → 20 Concurrent Users                                   │
│  → 1-5 Second Query Performance                          │
└──────────────────────────────────────────────────────────┘
```

### Technology Stack

|Component      |Technology          |Justification                                        |
|---------------|--------------------|-----------------------------------------------------|
|Cloud Platform |Azure UK            |Regional compliance requirement                      |
|Data Warehouse |Snowflake Enterprise|Scalable, auto-scaling, separation of compute/storage|
|Orchestration  |Apache Airflow      |Industry standard, flexible DAGs                     |
|Transformation |dbt                 |SQL-based, version controlled, testing framework     |
|MQ Integration |Python Consumers    |Custom real-time message processing                  |
|Programming    |Python 3.x          |Data engineering, Snowpark                           |
|Version Control|Git                 |Code versioning                                      |

### Environment Strategy

**Single Snowflake Account with Database Separation:**

```
SNOWFLAKE_ACCOUNT (Azure UK)
├── DEV Environment
│   ├── DEV_BRONZE_DB
│   ├── DEV_SILVER_DB
│   └── DEV_GOLD_DB
└── PROD Environment
    ├── PROD_BRONZE_DB
    ├── PROD_SILVER_DB
    └── PROD_GOLD_DB
```

**Database Cloning:** Weekly zero-copy clone from PROD to DEV for testing

### Warehouse Sizing

|Warehouse    |Size   |Auto-Suspend|Purpose                             |
|-------------|-------|------------|------------------------------------|
|INGESTION_WH |X-Small|1 min       |MQ message landing, file ingestion  |
|TRANSFORM_WH |Small  |5 min       |dbt transformations (auto-scale 1-2)|
|QUERY_WH     |X-Small|2 min       |User queries (auto-scale 1-3)       |
|MONITORING_WH|X-Small|5 min       |Data quality checks                 |

**Cost Optimization:** Balanced approach with aggressive auto-suspend settings

-----

## 4. Star Schema Design

### Fact Tables

#### 4.1 Daily_Portfolio_Positions

**Grain:** One row per fund, security, date, leg type, currency

|Column           |Type         |Description                   |
|-----------------|-------------|------------------------------|
|position_key     |NUMBER       |Surrogate key (PK)            |
|fund_key         |NUMBER       |FK to Dim_Fund                |
|security_key     |NUMBER       |FK to Dim_Security            |
|date_key         |NUMBER       |FK to Dim_Date                |
|leg_type_key     |NUMBER       |FK to Dim_Leg_Type            |
|currency_key     |NUMBER       |FK to Dim_Currency            |
|market_value     |NUMBER(20,4) |Position value (base currency)|
|quantity         |NUMBER(20,6) |Position quantity             |
|source_file      |VARCHAR(500) |Source filename               |
|load_timestamp   |TIMESTAMP_NTZ|Load timestamp                |
|data_quality_flag|VARCHAR(50)  |Pass/Fail/Warning             |

**Clustering:** (date_key, fund_key)

**Multi-Leg Handling:**

- Single-leg: One row with leg_type=‘SINGLE’
- Multi-leg (FX fwd, CCS, futures): Two rows (LEG_1, LEG_2)
- Position count: COUNT DISTINCT(fund, security, date) — multi-leg counts as 1
- Valuation: SUM(market_value) — nets automatically

**Incremental Strategy:** Daily full snapshot scoped to (fund, date)

#### 4.2 Portfolio_NAV

**Grain:** One row per fund per date

|Column           |Type         |Description       |
|-----------------|-------------|------------------|
|nav_key          |NUMBER       |Surrogate key (PK)|
|fund_key         |NUMBER       |FK to Dim_Fund    |
|date_key         |NUMBER       |FK to Dim_Date    |
|nav_amount       |NUMBER(20,4) |Net Asset Value   |
|base_currency_key|NUMBER       |FK to Dim_Currency|
|source_file      |VARCHAR(500) |Source filename   |
|load_timestamp   |TIMESTAMP_NTZ|Load timestamp    |

**Clustering:** (date_key, fund_key)

**Validation:** SUM(positions.market_value) / nav_amount = 1.0 ±0.00009

#### 4.3 Benchmark_Positions

**Grain:** One row per benchmark, security, date

|Column                |Type         |Description              |
|----------------------|-------------|-------------------------|
|benchmark_position_key|NUMBER       |Surrogate key (PK)       |
|benchmark_key         |NUMBER       |FK to Dim_Benchmark      |
|security_key          |NUMBER       |FK to Dim_Security       |
|date_key              |NUMBER       |FK to Dim_Date           |
|weight                |NUMBER(10,8) |Benchmark weight (%)     |
|market_value          |NUMBER(20,4) |Position value (optional)|
|source_file           |VARCHAR(500) |Source filename          |
|load_timestamp        |TIMESTAMP_NTZ|Load timestamp           |

**Clustering:** (date_key, benchmark_key)

**Point-in-Time:** Daily snapshots preserve historical constituents

**Validation:** SUM(weight) per benchmark/date = 100.0 ±0.00009

**Incremental:** Daily full snapshot scoped to (benchmark, date), 1-day reprocessing allowed

#### 4.4 Trades (Phase 2)

**Grain:** One row per trade allocation (security, trade date, fund)

|Column             |Type         |Description           |
|-------------------|-------------|----------------------|
|trade_key          |NUMBER       |Surrogate key (PK)    |
|trade_id           |VARCHAR(100) |Source trade ID       |
|fund_key           |NUMBER       |FK to Dim_Fund        |
|security_key       |NUMBER       |FK to Dim_Security    |
|trade_date_key     |NUMBER       |FK to Dim_Date        |
|settlement_date_key|NUMBER       |FK to Dim_Date        |
|counterparty_key   |NUMBER       |FK to Dim_Counterparty|
|broker_key         |NUMBER       |FK to Dim_Broker      |
|quantity           |NUMBER(20,6) |Trade quantity        |
|price              |NUMBER(20,8) |Execution price       |
|trade_amount       |NUMBER(20,4) |Total amount          |
|direction          |VARCHAR(10)  |BUY/SELL              |
|load_timestamp     |TIMESTAMP_NTZ|Message arrival time  |

**Clustering:** (trade_date_key, fund_key)

**Incremental:** Append-only (corrections as new records)

### Dimension Tables

#### 4.5 Dim_Security (SCD Type 2)

|Column                         |Type         |SCD Handling          |
|-------------------------------|-------------|----------------------|
|security_key                   |NUMBER       |Surrogate key (PK)    |
|sec_master_id                  |VARCHAR(100) |Business key (IVP)    |
|accounting_system_security_code|VARCHAR(100) |SimCorp code - Type 1 |
|isin                           |VARCHAR(12)  |Type 1                |
|sedol                          |VARCHAR(10)  |Type 1                |
|bbgid                          |VARCHAR(50)  |Type 1                |
|loanx_id                       |VARCHAR(50)  |Type 1                |
|security_name                  |VARCHAR(500) |Type 1                |
|security_subtype               |VARCHAR(100) |Type 2 (track history)|
|industry                       |VARCHAR(200) |Type 2                |
|sector                         |VARCHAR(200) |Type 2                |
|rating                         |VARCHAR(50)  |Type 2                |
|country_of_risk_key            |NUMBER       |Type 2                |
|country_of_issue_key           |NUMBER       |Type 2                |
|currency_key                   |NUMBER       |Type 1                |
|issue_date                     |DATE         |Type 1                |
|maturity_date                  |DATE         |Type 1                |
|coupon_rate                    |NUMBER(8,5)  |Type 1                |
|effective_from_date            |DATE         |SCD metadata          |
|effective_to_date              |DATE         |SCD metadata          |
|is_current                     |BOOLEAN      |SCD metadata          |
|load_timestamp                 |TIMESTAMP_NTZ|Metadata              |

**SCD Type 2:** Track history for security_subtype, industry, sector, rating, country changes

**Search Optimization:** Enable on sec_master_id, isin, sedol, bbgid

#### 4.6 Dim_Fund (SCD Type 2)

|Column              |Type         |SCD Handling       |
|--------------------|-------------|-------------------|
|fund_key            |NUMBER       |Surrogate key (PK) |
|fund_code           |VARCHAR(50)  |Business key       |
|account_number      |VARCHAR(100) |Type 1             |
|fund_name           |VARCHAR(500) |Type 1 (overwrites)|
|strategy            |VARCHAR(200) |Type 2             |
|base_currency_key   |NUMBER       |Type 1             |
|fund_start_date     |DATE         |Type 1             |
|fund_closed_date    |DATE         |Type 1             |
|linked_benchmark_key|NUMBER       |Type 2             |
|investment_team     |VARCHAR(200) |Type 2             |
|manufacturing_region|VARCHAR(100) |Type 1             |
|effective_from_date |DATE         |SCD metadata       |
|effective_to_date   |DATE         |SCD metadata       |
|is_current          |BOOLEAN      |SCD metadata       |
|load_timestamp      |TIMESTAMP_NTZ|Metadata           |

**SCD Type 2:** Track history for strategy, linked_benchmark, investment_team

#### 4.7 Additional Dimensions

- **Dim_Benchmark:** Benchmark master (Type 1)
- **Dim_Date:** Standard date dimension with business day flags (US, UK, Japan, Canada holidays)
- **Dim_Leg_Type:** SINGLE, LEG_1, LEG_2
- **Dim_Currency:** ISO 4217 currency codes
- **Dim_Country:** ISO 3166 country codes
- **Dim_Counterparty (Phase 2):** Trade counterparties
- **Dim_Broker (Phase 2):** Executing brokers

**Benchmark Vendor Classifications:**  
Benchmark positions link to Dim_Security which contains vendor-specific classifications:

- Barclays (fixed income): sector, rating, maturity
- Moody’s, Fitch, S&P: credit ratings
- Bloomberg: industry groups

Each stored with vendor prefix (e.g., `sp_sector`, `barclays_rating`)

-----

## 5. Data Quality & Validation Framework

### Validation Rules

#### Critical Validations - Alert & Continue

**1. Position/NAV Reconciliation**

- **Rule:** SUM(market_value) / NAV = 1.0 ±0.00009 (0.009%)
- **Frequency:** Daily post-load
- **Action:** Email alert, dashboard flag, continue processing

**2. Benchmark Weight Summation**

- **Rule:** SUM(weight) = 100.0 ±0.00009 per benchmark/date
- **Action:** Alert, dashboard flag, continue

**3. Missing Analytics (Phase 2)**

- **Rule:** All positions should have analytics
- **Action:** Alert, use prior day analytics with flag visible to users

**4. Late-Arriving Benchmark Data**

- **Rule:** Benchmark data should arrive T-1
- **Action:** Flag stale data if >T-2, reprocess 1 day back if late arrival

#### Critical Validations - Quarantine

**5. Orphan Positions**

- **Rule:** All positions must have matching SecMasterId in Dim_Security
- **Action:** Quarantine, block from GOLD layer, immediate alert

### Trade Reconciliation (Phase 2) - Multi-Level

**Level 1: Trade-to-Allocation**

- SUM(allocation quantities) = parent trade quantity
- SUM(allocation amounts) = parent trade amount
- Tolerance: Exact match
- Action: Quarantine on failure

**Level 2: Position Roll-Forward**

- T-1 positions + T-0 trades = T-0 positions
- By fund + security
- Adjust for corporate actions, maturities
- Action: Dashboard alert for review

**Level 3: Cash Settlement**

- Settled trades match cash movements
- Trade date vs settlement date consistency
- Action: Alert for investigation

### Data Quality Dashboard

**Real-Time Monitoring:**

1. **Data Freshness:** Latest position date per fund, stale data alerts
1. **Validation Results:** Position/NAV status, benchmark weights, orphan counts
1. **Volume Metrics:** Position counts with trends
1. **Quality Metrics:** % with analytics, % with complete reference data
1. **Alerts:** Active alerts, email history, resolution tracking

**Alert Configuration:**

- **Recipients:** Data engineering team
- **Triggers:** Variance >0.009%, orphans detected, missing analytics >5%, late data
- **Retries:** File ingestion (3x, 5min intervals), MQ (exponential backoff)

-----

## 6. Real-Time MQ Integration

### MQ Consumer Architecture

**Technology:**

- Language: Python 3.10+
- Snowflake Connector: snowflake-connector-python
- Containerization: Docker
- Orchestration: Kubernetes or Docker Compose

**Consumer Pattern:**

```python
class MQConsumer:
    def process_message(self, message):
        # 1. Parse JSON
        # 2. Validate schema
        # 3. Deduplicate (check message_id)
        # 4. Transform to Snowflake format
        # 5. Insert to staging table
        # 6. Acknowledge message
        # 7. Emit metrics
```

### Source-Specific Consumers

**Charles River Trades (Phase 2):**

- Format: JSON
- Volume: ~1,000s/day (scalable architecture)
- Latency: <1 minute target
- Deduplication: trade_id
- Error Handling: 3 retries → Dead Letter Queue

**IVP Security Reference:**

- Format: JSON (full security record)
- Volume: 100/day standard, 10,000 month-end
- Processing: Real-time dimension updates
- Landing: BRONZE staging → dbt SCD Type 2 merge
- Scalability: 3-5 consumer instances during month-end

**SimCorp Intraday Positions:**

- Format: JSON
- Frequency: Continuous during trading
- Processing: 10-second micro-batches (100 updates → single MERGE)
- Strategy: UPDATE main position fact, replaced by T-1 batch next day
- Scalability: Single consumer with batching

### Error Handling

**Retry Logic:**

- Transient errors: Exponential backoff (1min, 2min, 5min)
- Max retries: 3 attempts
- Post-retry: Dead Letter Queue

**Dead Letter Queue:**

- Manual review and reprocessing
- Dashboard alert when depth >10 messages

**Schema Validation Failures:**

- Log error, send to DLQ, alert team

### Monitoring

**Metrics:**

- Messages consumed per minute
- Processing latency (arrival → Snowflake insert)
- Error rate
- DLQ depth
- Consumer lag

**Health Checks:**

- HTTP endpoint /health
- MQ connection status
- Snowflake connection status
- Last successful message timestamp

-----

## 7. Transformation Strategy (Bronze/Silver/Gold)

### Medallion Architecture

**BRONZE → SILVER → GOLD**

### Bronze Layer (Raw Data Landing)

**Purpose:** Immutable copy of source data exactly as received

**Database:** BRONZE_DB  
**Schemas:** SIMCORP_RAW, IVP_RAW, RIMES_RAW, CHARLESRIVER_RAW

**Characteristics:**

- Store as VARIANT (JSON) or minimal transformation
- Append-only (90-day time travel)
- Lineage: source_file, load_timestamp, message_id

**Example:**

```sql
CREATE TABLE BRONZE_DB.SIMCORP_RAW.positions (
    bronze_id NUMBER AUTOINCREMENT,
    source_file VARCHAR(500),
    raw_data VARIANT,
    load_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### Silver Layer (Cleansed & Conformed)

**Purpose:** Cleansed, typed, validated data ready for business logic

**Database:** SILVER_DB  
**Schemas:** POSITIONS, SECURITIES, BENCHMARKS, TRADES

**Transformations:**

- Parse JSON/VARIANT to typed columns
- Data type conversion and validation
- Deduplication
- Referential integrity checks
- Create natural keys for dimension lookups

**Example dbt model:**

```sql
{{ config(
    materialized='incremental',
    unique_key=['fund_code', 'security_code', 'position_date', 'leg_type'],
    cluster_by=['position_date']
) }}

WITH source AS (
    SELECT 
        raw_data:fund_code::VARCHAR AS fund_code,
        raw_data:security_code::VARCHAR AS security_code,
        raw_data:position_date::DATE AS position_date,
        COALESCE(raw_data:leg_type::VARCHAR, 'SINGLE') AS leg_type,
        raw_data:market_value::NUMBER(20,4) AS market_value,
        source_file,
        load_timestamp
    FROM {{ source('bronze_simcorp', 'positions') }}
    {% if is_incremental() %}
    WHERE load_timestamp > (SELECT MAX(load_timestamp) FROM {{ this }})
    {% endif %}
)
SELECT * FROM source
```

**Incremental Strategy:**

- Positions: Incremental based on load_timestamp, scoped to (fund, date)
- Late arrivals: UPSERT on primary key
- Security dimension: Incremental merge with SCD Type 2

### Gold Layer (Star Schema)

**Purpose:** Final star schema optimized for analytics

**Database:** GOLD_DB  
**Schemas:** FACTS, DIMENSIONS, AGGREGATES

**Transformations:**

- Surrogate key generation
- Dimension lookups (natural → surrogate keys)
- SCD Type 2 implementation
- Business logic and calculations
- Multi-leg position handling

**Example dbt model:**

```sql
WITH dimension_lookups AS (
    SELECT 
        p.*,
        f.fund_key,
        s.security_key,
        d.date_key,
        l.leg_type_key,
        c.currency_key
    FROM {{ ref('daily_portfolio_positions') }} p
    LEFT JOIN {{ ref('dim_fund') }} f 
        ON p.fund_code = f.fund_code AND f.is_current = TRUE
    LEFT JOIN {{ ref('dim_security') }} s 
        ON p.security_code = s.accounting_system_security_code
        AND p.position_date BETWEEN s.effective_from_date AND s.effective_to_date
    -- Additional joins...
)
SELECT * FROM dimension_lookups
```

### SCD Type 2 Options

**Option A: dbt Snapshots**

- Built-in dbt snapshot functionality
- Simple configuration
- Automatic history tracking

**Option B: Custom Merge Macro**

- Full control over logic
- Custom business rules
- Better performance for large volumes

**Option C: Snowflake Streams + Tasks**

- Native Snowflake change data capture
- Near real-time SCD updates
- More complex setup

**Recommendation:** Start with dbt snapshots (Option A) for MVP, evaluate custom macro if performance needed

### dbt Project Structure

```
investment_dw/
├── dbt_project.yml
├── models/
│   ├── bronze/
│   │   └── sources.yml
│   ├── silver/
│   │   ├── positions/
│   │   ├── securities/
│   │   └── benchmarks/
│   ├── gold/
│   │   ├── dimensions/
│   │   ├── facts/
│   │   └── aggregates/
│   └── schema.yml
├── macros/
│   └── scd_type2_merge.sql
├── tests/
│   ├── position_nav_reconciliation.sql
│   └── benchmark_weight_validation.sql
└── snapshots/
```

### Orchestration (Airflow)

**Daily Batch DAG:**

1. Check file arrival (file sensors)
1. Load to Bronze (COPY INTO)
1. Run dbt Silver transformations
1. Run dbt Gold transformations
1. Run dbt tests
1. Data quality validations
1. Update dashboard

**Schedule:** 6 AM daily (after SOD files arrive)

**Late Data Handling:**

- Benchmark reprocessing: 1-day lookback allowed
- Stale data flagging: >T-2 positions flagged in dashboard

### Python/Snowpark Usage

**When to use:**

- Multi-leg position complex netting
- Attribution calculations (Phase 2)
- Custom data quality rules with iterative logic

**Recommendation:** Start with pure SQL/dbt, introduce Snowpark only when SQL becomes unreadable or performance requires procedural logic

-----

## 8. Infrastructure & DevOps

### Snowflake Setup

**Account:** Enterprise Edition, Azure UK South

**Database Structure:**

```sql
-- DEV
CREATE DATABASE DEV_BRONZE_DB;
CREATE DATABASE DEV_SILVER_DB;
CREATE DATABASE DEV_GOLD_DB;

-- PROD
CREATE DATABASE PROD_BRONZE_DB;
CREATE DATABASE PROD_SILVER_DB;
CREATE DATABASE PROD_GOLD_DB;
```

**Warehouses:**

```sql
CREATE WAREHOUSE INGESTION_WH WITH
    WAREHOUSE_SIZE = 'X-SMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE;

CREATE WAREHOUSE TRANSFORM_WH WITH
    WAREHOUSE_SIZE = 'SMALL'
    AUTO_SUSPEND = 300
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 2;

CREATE WAREHOUSE QUERY_WH WITH
    WAREHOUSE_SIZE = 'X-SMALL'
    AUTO_SUSPEND = 120
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 3;
```

**Internal Stages:**

```sql
CREATE STAGE PROD_BRONZE_DB.STAGES.simcorp_stage
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);

CREATE STAGE PROD_BRONZE_DB.STAGES.rimes_stage
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
```

### Airflow Setup

**Deployment:** Containerized on Azure (Kubernetes or Docker Compose)

**Components:**

- Webserver (UI)
- Scheduler (DAG triggers)
- Workers (task execution)
- PostgreSQL (metadata)
- Redis (message broker)

**Connections:**

```
AIRFLOW_CONN_SNOWFLAKE_DEFAULT = 'snowflake://user:pwd@account/database?warehouse=wh&role=role'
```

### MQ Consumer Deployment

**Containerization:**

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY consumers/ ./consumers/
CMD ["python", "consumers/simcorp_consumer.py"]
```

**Kubernetes Deployment:**

- Replicas: 1-3 per consumer type
- Resource limits: 256Mi-512Mi memory, 200m-500m CPU
- Health checks: HTTP /health endpoint
- Secrets: Snowflake credentials via K8s secrets

### CICD Pipeline

**Git Repository Structure:**

```
investment-data-warehouse/
├── airflow/dags/
├── dbt/models/
├── consumers/
├── terraform/ (optional IaC)
├── tests/
├── Dockerfile
└── docker-compose.yml
```

**GitHub Actions Workflow:**

1. **PR to main:** Run dbt compile + dbt test in DEV
1. **Merge to main:** Deploy dbt to PROD, copy Airflow DAGs, build/push container images

**Branching:**

- `main` - Production
- `develop` - Integration
- `feature/*` - Features
- `hotfix/*` - Emergency fixes

**Environment Promotion:**

- DEV: Continuous from `develop`
- PROD: Weekly release from `main`

### Testing Strategy

**Unit Tests:**

- dbt model SQL logic (dbt test framework)
- Python consumer functions (pytest)

**Integration Tests:**

- End-to-end Bronze → Silver → Gold flow
- Sample data ingestion

**Data Quality Tests:**

- dbt tests in schema.yml
- Custom SQL tests

**Example dbt test:**

```yaml
models:
  - name: fact_daily_portfolio_positions
    columns:
      - name: position_key
        tests:
          - unique
          - not_null
      - name: fund_key
        tests:
          - relationships:
              to: ref('dim_fund')
              field: fund_key
```

**Sample Data:** 5 pilot funds, 2-3 benchmarks, 1 week history, multi-leg securities

-----

## 9. Performance & Optimization

### Query Performance Targets

|Query Type                  |Target |Strategy                          |
|----------------------------|-------|----------------------------------|
|Single fund snapshot        |<1 sec |Clustering on (date_key, fund_key)|
|Fund history (1 year)       |1-3 sec|Clustering + result caching       |
|Multi-fund aggregation      |2-4 sec|Materialized views                |
|Benchmark constituent lookup|<1 sec |Search optimization               |
|Sector exposure             |3-5 sec|Materialized view                 |
|Active position calc        |2-5 sec|Pre-joined MV                     |

### Clustering Strategy

```sql
ALTER TABLE GOLD_DB.FACTS.FACT_DAILY_PORTFOLIO_POSITIONS 
CLUSTER BY (date_key, fund_key);

ALTER TABLE GOLD_DB.FACTS.FACT_BENCHMARK_POSITIONS 
CLUSTER BY (date_key, benchmark_key);
```

**Monitoring:**

```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('table_name', '(date_key, fund_key)');
-- Target: Clustering depth <4
```

### Materialized Views

**Fund Daily Summary:**

```sql
CREATE MATERIALIZED VIEW GOLD_DB.AGGREGATES.MV_FUND_DAILY_SUMMARY AS
SELECT 
    f.fund_code,
    d.date,
    COUNT(DISTINCT p.security_key) AS position_count,
    SUM(p.market_value) AS total_market_value,
    n.nav_amount
FROM GOLD_DB.FACTS.FACT_DAILY_PORTFOLIO_POSITIONS p
JOIN GOLD_DB.DIMENSIONS.DIM_FUND f ON p.fund_key = f.fund_key
JOIN GOLD_DB.DIMENSIONS.DIM_DATE d ON p.date_key = d.date_key
JOIN GOLD_DB.FACTS.FACT_PORTFOLIO_NAV n ON p.fund_key = n.fund_key
GROUP BY f.fund_code, d.date, n.nav_amount;
```

**Sector Exposures:**

```sql
CREATE MATERIALIZED VIEW GOLD_DB.AGGREGATES.MV_SECTOR_EXPOSURES AS
SELECT 
    f.fund_code,
    d.date,
    s.sector,
    SUM(p.market_value) AS sector_market_value,
    SUM(p.market_value) / SUM(SUM(p.market_value)) OVER (PARTITION BY f.fund_code, d.date) AS sector_weight
FROM GOLD_DB.FACTS.FACT_DAILY_PORTFOLIO_POSITIONS p
JOIN dimensions...
GROUP BY f.fund_code, d.date, s.sector;
```

**Refresh:** Automatic incremental refresh by Snowflake

### Search Optimization

```sql
ALTER TABLE GOLD_DB.DIMENSIONS.DIM_SECURITY 
ADD SEARCH OPTIMIZATION ON EQUALITY(sec_master_id, isin, sedol, bbgid);

ALTER TABLE GOLD_DB.DIMENSIONS.DIM_FUND 
ADD SEARCH OPTIMIZATION ON EQUALITY(fund_code);
```

### Result Caching

- Snowflake automatically caches query results for 24 hours
- Identical queries from any user hit cache (0 compute cost)
- Encourage common query patterns for maximum cache utilization

### Query Optimization Best Practices

1. **Always filter on clustered columns** (date_key, fund_key)
1. **Use materialized views** for common aggregations
1. **Limit SELECT ***, specify needed columns
1. **Partition large scans** by date ranges
1. **Monitor query profile** for optimization opportunities

-----

## 10. Security & Governance

### Row-Level Security

**Requirement:** Portfolio managers see only their funds

**Implementation:**

```sql
-- Create secure view with RLS
CREATE SECURE VIEW GOLD_DB.SECURE_VIEWS.POSITIONS_RLS AS
SELECT p.*
FROM GOLD_DB.FACTS.FACT_DAILY_PORTFOLIO_POSITIONS p
JOIN GOLD_DB.DIMENSIONS.DIM_FUND f ON p.fund_key = f.fund_key
WHERE f.investment_team = CURRENT_USER()  -- or mapping table
WITH ROW ACCESS POLICY rls_policy;

-- Grant to users
GRANT SELECT ON GOLD_DB.SECURE_VIEWS.POSITIONS_RLS TO ROLE PORTFOLIO_MANAGER;
```

### Role-Based Access Control (RBAC)

**Role Hierarchy:**

```
ACCOUNTADMIN (Snowflake admin)
  └── SYSADMIN (System management)
      ├── DATA_ENGINEER (Build access)
      │   ├── Read/Write: BRONZE, SILVER, GOLD
      │   ├── Manage: Warehouses, stages
      │   └── Execute: dbt, Airflow
      ├── AIRFLOW_SERVICE (Service account)
      │   ├── Read: BRONZE, SILVER
      │   ├── Write: BRONZE, SILVER, GOLD
      │   └── Execute: TRANSFORM_WH
      ├── PORTFOLIO_MANAGER (Read-only)
      │   ├── Read: GOLD (via RLS views)
      │   └── Execute: QUERY_WH
      └── BI_SERVICE (API/reporting)
          ├── Read: GOLD
          └── Execute: QUERY_WH
```

**Role Grants:**

```sql
-- Data Engineer
GRANT ALL ON DATABASE DEV_BRONZE_DB TO ROLE DATA_ENGINEER;
GRANT ALL ON DATABASE DEV_SILVER_DB TO ROLE DATA_ENGINEER;
GRANT ALL ON DATABASE DEV_GOLD_DB TO ROLE DATA_ENGINEER;
GRANT USAGE ON WAREHOUSE TRANSFORM_WH TO ROLE DATA_ENGINEER;

-- Airflow Service
GRANT USAGE, CREATE SCHEMA ON DATABASE PROD_BRONZE_DB TO ROLE AIRFLOW_SERVICE;
GRANT SELECT ON ALL TABLES IN SCHEMA PROD_GOLD_DB.FACTS TO ROLE AIRFLOW_SERVICE;
GRANT USAGE ON WAREHOUSE INGESTION_WH, TRANSFORM_WH TO ROLE AIRFLOW_SERVICE;

-- Portfolio Manager
GRANT USAGE ON DATABASE PROD_GOLD_DB TO ROLE PORTFOLIO_MANAGER;
GRANT SELECT ON GOLD_DB.SECURE_VIEWS.POSITIONS_RLS TO ROLE PORTFOLIO_MANAGER;
GRANT USAGE ON WAREHOUSE QUERY_WH TO ROLE PORTFOLIO_MANAGER;
```

### Data Retention & Compliance

**Time Travel:** 90 days for audit trail  
**Position History:** 10 years  
**Trade History:** Unlimited retention  
**Archival:** No archival to cheaper storage (retain in Snowflake)

**Compliance:**

- UK, European, US regulatory requirements
- 90-day time travel for audit
- Full data lineage tracking
- No column masking required (per current requirements)

### Audit Logging

**Snowflake Query History:**

- Automatic logging of all queries
- Retention: 1 year in SNOWFLAKE.ACCOUNT_USAGE
- Monitor for unusual access patterns

**Custom Audit Table:**

```sql
CREATE TABLE GOLD_DB.AUDIT.DATA_QUALITY_RESULTS (
    check_id NUMBER AUTOINCREMENT,
    check_name VARCHAR(200),
    check_date DATE,
    status VARCHAR(50),
    details VARIANT,
    run_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

-----

## 11. Implementation Roadmap

### Phase 1: MVP (Weeks 1-2)

**Scope:**

- **Sources:** SimCorp (positions, NAV), IVP (security reference), Rimes (benchmarks)
- **Funds:** 5 pilot funds
- **Fact Tables:** Daily_Portfolio_Positions, Portfolio_NAV, Benchmark_Positions
- **Dimensions:** Dim_Security (SCD Type 2), Dim_Fund (SCD Type 2), Dim_Benchmark, Dim_Date, Dim_Leg_Type, Dim_Currency, Dim_Country

**Success Criteria:**

- Position valuations and benchmarks populating accurately
- All positions linked to security and fund reference data
- Data quality checks operational with monitoring dashboard
- Position/NAV reconciliation within 0.009%
- Benchmark weights sum to 100% ±0.009%

**Deliverables:**

1. **Infrastructure:**
- Snowflake databases: DEV/PROD (BRONZE/SILVER/GOLD)
- Airflow containerized setup
- Git repository structure
- Sample data for 5 pilot funds
1. **Data Pipeline:**
- MQ consumer: IVP security reference (real-time)
- File ingestion: SimCorp positions (daily batch)
- File ingestion: Rimes benchmarks (daily batch)
- dbt models: Bronze → Silver → Gold
- Airflow DAG: Daily batch orchestration
1. **Star Schema:**
- 3 fact tables
- 7 dimension tables
- SCD Type 2 for Dim_Security and Dim_Fund
- Clustering on fact tables
- Sample materialized view (fund daily summary)
1. **Data Quality:**
- Position/NAV reconciliation test
- Benchmark weight validation
- Orphan position quarantine
- Data quality dashboard (basic version)
- Email alerts on validation failures
1. **Documentation:**
- dbt model documentation
- Data dictionary
- Operational runbook
- User guide for dashboard

**Week 1 Focus:**

- Infrastructure setup (Snowflake, Airflow, Git)
- Bronze layer: File ingestion + MQ consumer framework
- Silver layer: Basic transformations
- Sample data loading

**Week 2 Focus:**

- Gold layer: Star schema implementation
- SCD Type 2 logic for dimensions
- Data quality validation framework
- Dashboard MVP
- Testing with pilot funds

### Phase 2: Production Rollout (Weeks 3-5)

**Scope:**

- **All 100 funds** with positions and benchmarks
- **Charles River trades** (MQ integration)
- **SimCorp intraday positions** (continuous updates)
- **S&P Analytics** integration
- **Reuters FX rates**

**Deliverables:**

**Week 3:**

1. Scale to 100 funds
1. Performance testing and optimization
1. Materialized views for common queries
1. Search optimization on dimensions
1. User acceptance testing

**Week 4:**

1. Charles River trade integration (MQ consumer)
1. Fact_Trades table implementation
1. Multi-level trade reconciliation
1. Intraday position update logic
1. Testing scalability (1000s trades/day)

**Week 5:**

1. S&P Analytics integration
1. Reuters FX rates integration
1. FX rate dimension table
1. Final performance tuning
1. Production cutover
1. User training
1. Go-live support

### Key Milestones

|Milestone                     |Target Date  |Dependencies        |
|------------------------------|-------------|--------------------|
|Infrastructure setup complete |End of Week 1|None                |
|MVP functional (5 funds)      |End of Week 2|Infrastructure      |
|100 funds in production       |End of Week 3|MVP success         |
|Trade integration complete    |End of Week 4|MQ framework        |
|Full production (all features)|End of Week 5|All prior milestones|

### Post-Production (Ongoing)

**Immediate (Weeks 6-8):**

- Monitor performance and optimize
- Resolve any production issues
- User feedback collection
- Dashboard enhancements
- Additional materialized views as needed

**Short-term (Months 2-3):**

- Custom API development (agentic AI query generation)
- Pivot table interface
- Advanced analytics features
- Attribution analysis integration
- Risk metrics integration

**Long-term (Months 4-6):**

- Machine learning for data quality anomaly detection
- Predictive analytics
- Advanced portfolio optimization features
- Integration with additional data sources

-----

## 12. Risk Management

### Top 3 Project Risks

#### 1. Cost/Performance Balance

**Risk:** Snowflake costs exceed budget or performance doesn’t meet targets

**Mitigation:**

- Start with X-Small warehouses and scale up based on actual usage
- Aggressive auto-suspend settings (1-5 minutes)
- Monitor Snowflake credits daily
- Implement result caching and materialized views
- Use clustering to reduce data scanned
- Regular query optimization reviews

**Contingency:**

- Adjust warehouse sizes based on actual workload
- Implement query result persistence for common patterns
- Consider dedicated warehouse for heavy users

#### 2. Complexity of Multi-Leg Securities & SCD Type 2

**Risk:** Complex logic leads to bugs or performance issues

**Mitigation:**

- Extensive testing with sample multi-leg securities in MVP
- Clear documentation of multi-leg handling rules
- Start with dbt snapshots for SCD (simpler)
- Comprehensive unit tests for transformation logic
- Code review process for critical models

**Contingency:**

- Simplify approach if needed (SCD Type 1 for non-critical attributes)
- External validation by portfolio managers
- Fallback to manual checks during parallel run

#### 3. MQ Integration Scalability

**Risk:** Real-time MQ consumers can’t keep up with peak volumes

**Mitigation:**

- Design for horizontal scaling from day 1 (Kubernetes)
- Micro-batching strategy (100 messages → single MERGE)
- Monitor consumer lag continuously
- Load testing with simulated peak volumes
- Dead letter queue for graceful degradation

**Contingency:**

- Scale to additional consumer replicas (3-5 instances)
- Increase batch size if latency allows
- Temporary queuing if Snowflake under load

### Additional Risks

**Source System Reliability:**

- **Risk:** Source systems unavailable or delayed
- **Mitigation:** File sensors with timeout, retry logic, alerting
- **Contingency:** Manual file uploads, use prior day data with flags

**Team Skills (Using AI Agents):**

- **Risk:** Learning curve for new technologies (dbt, Airflow, Snowflake)
- **Mitigation:** Leverage AI agents for code generation, comprehensive documentation, sample code
- **Contingency:** Phased learning approach, focus on core functionality first

**Data Quality Issues:**

- **Risk:** Poor source data quality impacts warehouse accuracy
- **Mitigation:** Comprehensive validation framework, quarantine bad data, alert immediately
- **Contingency:** Work with source system owners to improve data quality

**No Budget Constraints (Development):**

- **Risk:** Over-engineering or inefficient resource usage
- **Mitigation:** Use minimal setup in DEV (X-Small warehouses), monitor usage
- **Note:** Production should optimize costs while meeting performance targets

-----

## 13. Operational Model

### Daily Operations

**Automated Daily Process (6 AM):**

1. File sensors wait for SimCorp and Rimes files
1. COPY files to Snowflake BRONZE layer
1. dbt transformations: BRONZE → SILVER → GOLD
1. dbt tests execute (data quality)
1. Custom validation checks (Position/NAV, benchmark weights)
1. Dashboard refresh
1. Email alerts if validation failures

**Monitoring:**

- Airflow UI: DAG run status
- Data Quality Dashboard: Validation results, data freshness
- Snowflake Query History: Performance monitoring
- MQ Consumer Health: /health endpoints, consumer lag

**On-Call Rotation:**

- Data engineering team monitors alerts
- Response SLA: 30 minutes for critical alerts
- Escalation: Source system teams if data issues

### Monthly Operations

**Month-End Processing:**

- Increased IVP security updates (100 → 10,000 messages)
- Scale MQ consumers to 3-5 instances
- Extended processing window if needed
- Additional validation for benchmark rebalancing

**Performance Review:**

- Snowflake credit consumption analysis
- Query performance review (95th percentile)
- Warehouse sizing optimization
- Materialized view effectiveness

**Data Quality Metrics:**

- % Position/NAV reconciliation passes
- % Benchmark weight validations passes
- Orphan position trends
- Missing analytics trends

### Quarterly Operations

**Capacity Planning:**

- Review data growth vs. projections
- Warehouse sizing adjustments
- Storage cost projections
- User concurrency analysis

**Enhancements:**

- User feedback incorporation
- New materialized views based on query patterns
- Additional data sources evaluation
- Feature backlog prioritization

### Key Contacts

|Role             |Team/Person                  |Responsibility                  |
|-----------------|-----------------------------|--------------------------------|
|Project Owner    |Data Engineering Team        |Overall delivery                |
|Snowflake Admin  |Data Engineering             |Account management, optimization|
|Airflow Admin    |Data Engineering             |DAG management, scheduling      |
|MQ Infrastructure|Project Team (owned)         |MQ platform, consumers          |
|Source Systems   |SimCorp, IVP, Rimes, CR teams|Data quality, file delivery     |
|End Users        |Portfolio Managers           |Feedback, UAT                   |

-----

## 14. Success Metrics & KPIs

### Technical KPIs

|Metric                     |Target               |Measurement             |
|---------------------------|---------------------|------------------------|
|Data Freshness             |<1 min from ingestion|Monitoring timestamp    |
|Query Performance (p95)    |1-5 seconds          |Snowflake query history |
|Position/NAV Reconciliation|Pass >99.9%          |Daily validation        |
|Benchmark Weight Validation|Pass >99.9%          |Daily validation        |
|Pipeline Success Rate      |>99%                 |Airflow DAG success rate|
|MQ Consumer Lag            |<5 seconds           |Consumer metrics        |

### Business KPIs

|Metric                    |Target                 |Measurement                |
|--------------------------|-----------------------|---------------------------|
|User Adoption             |20 concurrent users    |Snowflake session analytics|
|Portfolio Coverage        |100 funds              |Database record count      |
|Benchmark Coverage        |50 benchmarks          |Database record count      |
|Historical Depth          |10 years               |Date range query           |
|Query Complexity Supported|Multi-dimensional pivot|User feedback              |

### Operational KPIs

|Metric                   |Target                  |Measurement        |
|-------------------------|------------------------|-------------------|
|Incident Response Time   |<30 min                 |Alert response logs|
|Mean Time to Resolution  |<4 hours                |Incident tracking  |
|Data Quality Alert Volume|<5 per week             |Alert history      |
|Snowflake Credit Usage   |TBD (track and optimize)|Billing analysis   |

-----

## 15. Appendices

### Appendix A: Sample Queries

**Fund Position Snapshot:**

```sql
SELECT 
    f.fund_code,
    s.security_name,
    s.isin,
    p.market_value,
    p.quantity
FROM GOLD_DB.FACTS.FACT_DAILY_PORTFOLIO_POSITIONS p
JOIN GOLD_DB.DIMENSIONS.DIM_FUND f ON p.fund_key = f.fund_key
JOIN GOLD_DB.DIMENSIONS.DIM_SECURITY s ON p.security_key = s.security_key
JOIN GOLD_DB.DIMENSIONS.DIM_DATE d ON p.date_key = d.date_key
WHERE f.fund_code = 'FUND001'
  AND d.date = '2026-02-05';
```

**Sector Exposure Trend (1 Year):**

```sql
SELECT 
    d.date,
    s.sector,
    SUM(p.market_value) AS sector_exposure
FROM GOLD_DB.FACTS.FACT_DAILY_PORTFOLIO_POSITIONS p
JOIN GOLD_DB.DIMENSIONS.DIM_SECURITY s ON p.security_key = s.security_key
JOIN GOLD_DB.DIMENSIONS.DIM_DATE d ON p.date_key = d.date_key
JOIN GOLD_DB.DIMENSIONS.DIM_FUND f ON p.fund_key = f.fund_key
WHERE f.fund_code = 'FUND001'
  AND d.date BETWEEN '2025-02-01' AND '2026-02-01'
GROUP BY d.date, s.sector
ORDER BY d.date, sector_exposure DESC;
```

**Active Position (Fund vs Benchmark):**

```sql
WITH fund_positions AS (
    SELECT security_key, market_value / nav AS fund_weight
    FROM fact_daily_portfolio_positions
    WHERE fund_key = ? AND date_key = ?
),
benchmark_positions AS (
    SELECT security_key, weight AS benchmark_weight
    FROM fact_benchmark_positions
    WHERE benchmark_key = ? AND date_key = ?
)
SELECT 
    s.security_name,
    f.fund_weight,
    b.benchmark_weight,
    f.fund_weight - b.benchmark_weight AS active_weight
FROM fund_positions f
FULL OUTER JOIN benchmark_positions b USING (security_key)
JOIN dim_security s USING (security_key);
```

### Appendix B: Data Dictionary

**Available in dbt documentation** (generated via `dbt docs generate`)

Access at: http://dbt-docs-server/#!/overview

Includes:

- All table and column descriptions
- Data lineage diagrams
- Model dependencies
- Test results

### Appendix C: Glossary

- **SOD:** Start of Day
- **EOD:** End of Day
- **T-1:** Previous business day (Trade date minus 1)
- **NAV:** Net Asset Value
- **SCD:** Slowly Changing Dimension
- **MQ:** Message Queue
- **MV:** Materialized View
- **DLQ:** Dead Letter Queue
- **CCS:** Cross-Currency Swap
- **FX Fwd:** Foreign Exchange Forward

### Appendix D: Contact Information

**Project Team:**

- **Data Engineering Lead:** [TBD]
- **Snowflake Administrator:** [TBD]
- **Business Owner:** Portfolio Management Team

**Source System Contacts:**

- **SimCorp Support:** [TBD]
- **IVP Support:** [TBD]
- **Rimes Support:** [TBD]
- **Charles River Support:** [TBD]

-----

## Document Control

**Version History:**

|Version|Date      |Author  |Changes                            |
|-------|----------|--------|-----------------------------------|
|1.0    |2026-02-06|AI Agent|Initial comprehensive specification|

**Approval:**

|Role           |Name |Signature|Date|
|---------------|-----|---------|----|
|Project Sponsor|[TBD]|         |    |
|Technical Lead |[TBD]|         |    |
|Business Owner |[TBD]|         |    |

-----

**END OF DOCUMENT**