# Architecture — E-Commerce Sales Analysis

> **Olist Brazilian E-Commerce · End-to-End Analytics Pipeline**
> Stack: Python · PostgreSQL · Power BI · Pandas · SQLAlchemy

---

## Architecture Highlights

- 9 raw Olist datasets ingested from CSV
- PostgreSQL as the central analytical warehouse
- Pandas-based cleaning and preprocessing pipeline
- Business rule validation with structured JSON logging
- RFM customer segmentation
- Power BI dashboard with multi-page sales, customer, and regional analysis
- Fully reproducible via Conda environment

---

## Architecture Principles

- **Single source of truth** — all raw data loaded into PostgreSQL before any transformation
- **Separation of concerns** — ingest, preprocess, validate, and analyze are distinct pipeline stages
- **Re-executable by design** — each run rebuilds tables deterministically using overwrite semantics (`if_exists="replace"`)
- **Local-first** — the entire stack runs on a local machine without cloud dependencies
- **Validated before analysis** — business rule tests run before notebooks consume data
- **Reproducible environment** — Conda `environment.yml` pins all dependencies

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Source Layer        Compute Layer       Storage Layer       │
│                                                             │
│  CSV Files (data/)                                          │
│      │                                                      │
│      ▼                                                      │
│  Python ETL ──────────────────────> PostgreSQL              │
│  (pandas + SQLAlchemy)              (raw + clean tables)    │
│      │                                   │                  │
│      ▼                                   ▼                  │
│  Validation Layer               Jupyter Notebooks           │
│  (assumption_test.py)               (analysis)              │
│      │                                   │                  │
│      ▼                                   ▼                  │
│  JSON Logs                        Power BI Dashboard        │
│  (logs/assumption_tests/)         (.pbix)                   │
└─────────────────────────────────────────────────────────────┘
```

This view separates compute (Python ETL, notebooks) from storage (PostgreSQL) from visualisation (Power BI), with the validation layer producing an independent audit trail.

---

## High-Level Data Flow

```
Raw CSVs (data/)
      │
      ▼
[Ingest: load_raw.py]
      │  pandas.read_csv → df.to_sql
      ▼
PostgreSQL — raw tables
(customers, orders, order_items,
 order_payments, order_reviews,
 products, sellers, geolocation,
 category_translation)
      │
      ▼
[Preprocess: clean.py]
      │  date parsing, deduplication,
      │  null fills, category joins
      ▼
PostgreSQL — clean tables
(orders_clean, products_clean,
 customers_clean, geolocation_clean,
 reviews_clean)
      │
      ▼
[Validate: assumption_test.py]
      │  business rule assertions
      │  → logs/assumption_tests/run_*.json
      ▼
[Analyze: notebooks/]
      │  revenue, time trends, customers,
      │  cancellations, regional, RFM
      ▼
[Visualize: Power BI]
      e-commerce-sales-analysis.pbix
```

---

## Analytical Grain Strategy

Different analytical questions operate at different grains. All analysis is anchored to one of three levels:

| Grain | Key | Used For |
|---|---|---|
| Order | `order_id` | Order lifecycle, delivery performance, cancellation analysis |
| Item | `order_id` + `order_item_id` | Revenue breakdown, category and seller analysis |
| Buyer | `customer_unique_id` | Distinct customer counts, repeat purchase behaviour, RFM segmentation |

> **Default grain is order-level** unless the question explicitly involves per-item revenue or per-buyer behaviour. Mixing grains without explicit aggregation is the most common source of double-counting in this dataset — particularly when joining `order_items` (112,650 rows) back up to `orders_clean` (99,441 rows).

---

## Data Modeling Strategy

The warehouse follows a two-layer approach:

**Raw Layer** → `raw_*` tables loaded directly from CSV with no transformation. Serves as the source of truth and enables full re-processing at any time.

**Clean Layer** → `*_clean` tables produced by `clean.py`. These apply date parsing, deduplication, null imputation, and category translation joins. All analytical notebooks and Power BI consume this layer.

**Persistent tables:** 5 clean tables in PostgreSQL (see [Clean Table Reference](#clean-table-reference)).  
**Non-persistent outputs:** 7 analytical datasets produced in-memory during notebook runs (see [Analytical Datasets](#analytical-datasets)). These are not written back to the database.

---

## Database Model

The entity relationships across all tables are as follows:

```
customers_clean
    │
    └──< orders_clean
              │
              ├──< order_items >──── products_clean
              │         │
              │         └──────────> sellers
              │
              ├──< order_payments
              │
              └──< reviews_clean

customers_clean
    │
    └──> geolocation_clean   (via customer_zip_code_prefix)

sellers
    │
    └──> geolocation_clean   (via seller_zip_code_prefix)
```

**Relationship summary:**

| Parent Table | Child Table | Join Key | Cardinality |
|---|---|---|---|
| `customers_clean` (`customer_unique_id`) | `orders_clean` | `customer_unique_id` | 1 → many |
| `customers_clean` (`customer_id`) | `orders_clean` | `customer_id` | 1 → 1 *(order-scoped mapping — customer_id is a surrogate generated per order, not a reusable buyer identifier)* |
| `orders_clean` | `order_items` | `order_id` | 1 → many |
| `orders_clean` | `order_payments` | `order_id` | 1 → many |
| `orders_clean` | `reviews_clean` | `order_id` | 1 → 1 |
| `products_clean` | `order_items` | `product_id` | 1 → many |
| `sellers` | `order_items` | `seller_id` | 1 → many |
| `customers_clean` | `geolocation_clean` | `customer_zip_code_prefix` | many → 1 |
| `sellers` | `geolocation_clean` | `seller_zip_code_prefix` | many → 1 |

> All foreign key relationships were verified as fully intact in the raw layer — 0 orphaned rows across all joins. See `data_quality_report.md` for referential integrity results.

---

## Customer Identity Model

Olist's data model uses two separate customer identifiers. Understanding the difference is essential for any buyer-level analysis.

| Identifier | Scope | Use For |
|---|---|---|
| `customer_id` | Order-scoped — a new value is assigned per order | Joining `customers` to `orders`; not suitable for buyer counts |
| `customer_unique_id` | Buyer-scoped — stable across all orders by the same person | Counting distinct buyers; identifying repeat purchasers; RFM segmentation |

**Implication for cardinality:**

- `customer_id` is an **order-level surrogate key** (1:1 with orders) — each value is minted per order, not per buyer. It is used only to join `customers_clean` to `orders_clean`; it is not a buyer identifier and must not be used for customer counts or repeat-purchase analysis.
- `customer_unique_id → orders_clean` is **1:many** — one physical buyer can have multiple orders, each with a different `customer_id`. Always aggregate at `customer_unique_id` level when the question involves buyer behaviour.

**Key numbers:**

| Metric | Value |
|---|---|
| Total `customer_id` records | 99,441 |
| Distinct `customer_unique_id` values | 96,096 |
| Repeat buyers | 3,345 |

---

## Data Sources

| Source | Format | Rows (approx.) | Purpose |
|---|---|---|---|
| olist_customers_dataset.csv | CSV | 99,441 | Customer profiles and zip codes |
| olist_orders_dataset.csv | CSV | 99,441 | Order lifecycle and timestamps |
| olist_order_items_dataset.csv | CSV | 112,650 | Line items, prices, freight |
| olist_order_payments_dataset.csv | CSV | 103,886 | Payment type and installments |
| olist_order_reviews_dataset.csv | CSV | 100,000 | Review scores and comments |
| olist_products_dataset.csv | CSV | 32,951 | Product dimensions and category |
| olist_sellers_dataset.csv | CSV | 3,095 | Seller profiles and locations |
| olist_geolocation_dataset.csv | CSV | 1,000,163 | ZIP-to-lat/lng mapping |
| product_category_name_translation.csv | CSV | 71 | Portuguese → English category names |

---

## Raw Table Grain Reference

| Table | Grain | Primary Key |
|---|---|---|
| `customers` | One row per customer account | `customer_id` |
| `orders` | One row per order | `order_id` |
| `order_items` | One row per item within an order | `order_id` + `order_item_id` |
| `order_payments` | One row per payment installment per order | `order_id` + `payment_sequential` |
| `order_reviews` | One row per review submission | `review_id` |
| `products` | One row per product | `product_id` |
| `sellers` | One row per seller | `seller_id` |
| `geolocation` | One row per ZIP coordinate entry (multiple per ZIP in raw) | None |
| `category_translation` | One row per product category | `product_category_name` |

> See [Customer Identity Model](#customer-identity-model) for why `customer_id` is not a buyer identifier.

---

## Scale

| Metric | Value |
|---|---|
| Raw datasets | 9 |
| Total raw rows (approx.) | 1.6M+ |
| Orders | 99,441 |
| Order line items | 112,650 |
| Customers | 99,441 |
| Review records | 100,000 |
| Geolocation records | 1,000,163 |
| Product categories | 71 |
| Sellers | 3,095 |
| Date range | 2016 – 2018 |
| Clean tables produced (persisted to PostgreSQL) | 5 |
| Analytical outputs (in-memory, not persisted) | 7 |
| Validation tests | 11 |
| Analysis notebooks | 8 |

---

## Repository Structure

```
e-commerce-sales-analysis/
│
├── data/                          # Raw CSV files (not version controlled)
├── dashboard/                     # Power BI files (.pbix, .pbit, .pdf)
│
├── docs/
│   ├── architecture_report.md            # This document
│   ├── business_insights_report.md      # Analytical goals per notebook
│   ├── data_dictionary_report.md         # Column-level field definitions
│   └── data_quality_report.md            # Data quality findings and decisions
│
├── notebooks/
│   ├── 00_explore.ipynb           # Initial data exploration
│   ├── 01_profile.ipynb           # Statistical profiling
│   ├── 02_cleaning.ipynb          # Cleaning audit and verification
│   ├── 03_revenue_analysis.ipynb  # Revenue trends and breakdown
│   ├── 04_time_trends.ipynb       # Monthly and seasonal patterns
│   ├── 05_customer_analysis.ipynb # RFM segmentation and LTV
│   ├── 06_cancellation_analysis.ipynb  # Order cancellation drivers
│   └── 07_regional_analysis.ipynb # State-level performance
│
├── src/
│   ├── config.py                  # DB connection and engine factory
│   ├── ingest/
│   │   └── load_raw.py            # CSV → PostgreSQL (raw tables)
│   ├── preprocess/
│   │   └── clean.py               # Raw → clean tables
│   ├── validation/
│   │   └── assumption_test.py     # Business rule assertions
│   └── tests/
│       └── test_connection.py     # DB connectivity check
│
├── logs/
│   └── assumption_tests/          # JSON test results per run
│
├── scripts/
│   └── convert_dashboard_pdf.py   # Converts dashboard pdf in individual image files
│
├── annotations/
│   └── RFM.txt                    # RFM segmentation notes
│
├── .env                           # DB credentials (not version controlled)
├── environment.yml                # Conda environment definition
├── requirements.txt               # pip dependencies
└── Makefile                       # Pipeline task runner
```

---

## Pipeline Stages

### Stage 1 — Ingest (`src/ingest/load_raw.py`)

Reads all 9 CSV files from `data/` using `pandas.read_csv` and loads them into PostgreSQL via `df.to_sql(..., if_exists="replace")`. Creates the target database if it does not already exist.

```
data/*.csv  →  pandas DataFrame  →  PostgreSQL raw tables
```

### Stage 2 — Preprocess (`src/preprocess/clean.py`)

Reads raw tables from PostgreSQL and applies the following transformations:

| Table | Transformations Applied |
|---|---|
| orders_clean | Parse 5 timestamp columns to datetime; fill null approval dates with purchase date |
| products_clean | Join English category names from translation table; fill unknown categories |
| customers_clean | Drop duplicate customer_ids |
| geolocation_clean | Deduplicate by zip_code_prefix (keeps first occurrence) |
| reviews_clean | Fill null comment fields with empty string; parse 2 date columns |

### Stage 3 — Validate (`src/validation/assumption_test.py`)

Runs 11 business rule assertions across orders, customers, order items, payments, and products. Results are printed to terminal and saved as a timestamped JSON file under `logs/assumption_tests/`.

### Stage 4 — Analyze (`notebooks/`)

8 Jupyter notebooks consume the clean PostgreSQL tables and produce exploratory analysis, visualizations, and segmentation outputs.

### Stage 5 — Visualize (`dashboard/`)

Power BI connects to PostgreSQL clean tables to produce the final dashboard covering sales performance, time trends, customer segments, regional breakdown, and cancellation analysis.

---

## Key Design Decisions

### Why PostgreSQL as the warehouse

PostgreSQL acts as both the staging warehouse and the serving layer for Power BI. Using a proper relational database (instead of keeping data as CSVs) enables SQL-based transformations across notebooks, reliable Power BI connectivity via the standard PostgreSQL connector, and reproducible data access across the team.

### Why Pandas for preprocessing

The dataset fits comfortably in memory (~1.6M rows). Pandas provides expressive, readable transformation code and integrates directly with SQLAlchemy for read/write against PostgreSQL. A heavier tool (dbt, Spark) would introduce unnecessary infrastructure for a local portfolio project of this scale.

### Why separate raw and clean tables

Keeping raw tables unchanged means any preprocessing bug can be fixed and re-run without re-ingesting from CSV. It also makes the lineage explicit: raw tables are never modified after ingestion.

### Why overwrite semantics (`if_exists="replace"`) throughout

The pipeline is fully re-executable: each run rebuilds all tables deterministically from source. This makes iterating on transformations safe during development without requiring manual cleanup between runs. This is a deliberate trade-off — the pipeline prioritises simplicity and reproducibility over incrementality. It is not designed for partial re-runs or failure recovery mid-stage; a full re-run is always the recovery path.

### Why `.env` for credentials

Database credentials are stored in `.env` and loaded via `python-dotenv`. This keeps secrets out of source code while keeping local development simple.

### Why Power BI over other BI tools

Power BI provides stable, native PostgreSQL connectivity, a desktop-first workflow that matches the local-first architecture, and the `.pbit` template format for sharing a credential-free dashboard structure.

---

## Data Quality Results

| Check | Table | Result |
|---|---|---|
| No null order_ids | orders | PASS |
| No duplicate order_ids | orders | PASS |
| No null customer_ids | orders | PASS |
| No null customer_ids | customers | PASS |
| No duplicate customer_ids | customers | PASS |
| No negative prices | order_items | PASS |
| No null product_ids | order_items | PASS |
| All order_items have matching order | order_items | PASS |
| No negative payment values | order_payments | PASS |
| No null payment values | order_payments | PASS |
| No null product_ids | products | PASS |

**Summary: PASS=11 FAIL=0 WARN=0**

**Referential Integrity Checks: PASS=4 FAIL=0**

| Relationship | Orphaned Rows |
|---|---|
| `order_items.order_id → orders.order_id` | 0 |
| `order_payments.order_id → orders.order_id` | 0 |
| `order_reviews.order_id → orders.order_id` | 0 |
| `order_items.product_id → products.product_id` | 0 |

Test results are persisted at `logs/assumption_tests/run_<YYYY_MM_DD_HHMM>.json` on every run.

---

## Clean Table Reference

### orders_clean
**Grain:** one row per order

| Column | Type | Notes |
|---|---|---|
| order_id | varchar | Primary key |
| customer_id | varchar | FK to customers_clean |
| order_status | varchar | delivered, shipped, cancelled, etc. |
| order_purchase_timestamp | timestamp | Parsed from string |
| order_approved_at | timestamp | Null-filled with purchase timestamp |
| order_delivered_carrier_date | timestamp | Parsed from string |
| order_delivered_customer_date | timestamp | Parsed from string |
| order_estimated_delivery_date | timestamp | Parsed from string |

**Business use:** order lifecycle, delivery performance, cancellation analysis

---

### products_clean
**Grain:** one row per product

| Column | Type | Notes |
|---|---|---|
| product_id | varchar | Primary key |
| product_category_name | varchar | Original Portuguese name |
| product_category_name_english | varchar | Joined from translation table |
| product_weight_g | float | Used for freight analysis |
| product_length_cm | float | |
| product_height_cm | float | |
| product_width_cm | float | |

**Business use:** category revenue breakdown, product-level analysis

---

### customers_clean
**Grain:** one row per customer record (`customer_id`)

| Column | Type | Notes |
|---|---|---|
| customer_id | varchar | Primary key — order-scoped (see Customer Identity Model) |
| customer_unique_id | varchar | Persistent buyer identity — use for distinct buyer counts |
| customer_zip_code_prefix | varchar | For geolocation joins |
| customer_city | varchar | |
| customer_state | varchar | Used in regional analysis |

**Business use:** customer segmentation, RFM, regional performance

---

### geolocation_clean
**Grain:** one row per zip code prefix (deduplicated)

| Column | Type | Notes |
|---|---|---|
| geolocation_zip_code_prefix | varchar | Primary key after dedup |
| geolocation_lat | float | |
| geolocation_lng | float | |
| geolocation_city | varchar | |
| geolocation_state | varchar | |

**Business use:** map-based regional visualizations in Power BI

---

### reviews_clean
**Grain:** one row per review

| Column | Type | Notes |
|---|---|---|
| review_id | varchar | Primary key |
| order_id | varchar | FK to orders_clean |
| review_score | int | 1–5 |
| review_comment_title | varchar | Empty string if null |
| review_comment_message | varchar | Empty string if null |
| review_creation_date | timestamp | Parsed from string |
| review_answer_timestamp | timestamp | Parsed from string |

**Business use:** customer satisfaction analysis, score distribution

---

## Non-Functional Characteristics

- **Reproducible** — `environment.yml` pins all Python and library versions; re-running the pipeline from scratch produces identical output
- **Re-executable** — every stage can be re-run safely; overwrite semantics mean no manual cleanup is required between runs
- **Modular** — ingest, preprocess, validate, and analyze are independently executable stages
- **Testable** — business rules are explicit assertions with pass/fail tracking per run
- **Local-first** — no cloud services required; runs entirely on a local machine
- **Auditable** — validation results are timestamped and persisted as JSON logs

---

## Technology Stack

| Tool | Version | Role |
|---|---|---|
| Python | 3.11 | Primary language |
| PostgreSQL | 15.x | Analytical warehouse and serving layer |
| pandas | 2.x | Data loading and transformation |
| SQLAlchemy | 2.x | Database connection abstraction |
| psycopg2-binary | 2.9.x | PostgreSQL driver |
| JupyterLab | 4.x | Interactive analysis notebooks |
| Power BI Desktop | Latest stable | Dashboard and visualization |
| Conda | Latest stable | Environment and dependency management |
| python-dotenv | 1.x | Credential management via `.env` |

---

## Environment Setup

```bash
# 1. Clone the repository
git clone <repo-url>
cd e-commerce-sales-analysis

# 2. Create Conda environment
conda env create -f environment.yml
conda activate ecommerce-analysis

# 3. Configure database credentials
cp .env.example .env
# Edit .env: DB_USER, DB_PASSWORD, DB_HOST, DB_PORT, DB_NAME

# 4. Run the full pipeline (single command)
make all
```

`make all` runs the complete pipeline in order:

```
make ingest → make preprocess → make validate
```

Individual stages can also be run independently:

```bash
make ingest       # Load raw CSVs into PostgreSQL
make preprocess   # Produce clean tables
make validate     # Run business rule assertions
```

---

## Pipeline Runtime (Approximate)

| Stage | Runtime |
|---|---|
| Ingest (load_raw.py) | ~2 min |
| Preprocess (clean.py) | ~1 min |
| Validate (assumption_test.py) | ~30 sec |
| Notebook analysis (all 8) | ~10 min |
| **Total pipeline** | **~14 min** |

*Runtimes measured on a mid-range laptop with PostgreSQL running locally.*

---

## Analytical Datasets

The following in-memory or notebook-scoped outputs are produced during analysis. They are not persisted to PostgreSQL but represent meaningful analytical modeling and are the basis for Power BI dashboard metrics.

| Output | Produced In | Description |
|---|---|---|
| `rfm_scores` | `05_customer_analysis.ipynb` | Per-customer Recency, Frequency, Monetary scores and segment labels (Champions, At Risk, Lost, etc.) |
| `monthly_revenue_summary` | `03_revenue_analysis.ipynb`, `04_time_trends.ipynb` | Revenue and order counts aggregated by month |
| `category_revenue_summary` | `03_revenue_analysis.ipynb` | Total revenue, order count, and average order value per product category |
| `state_revenue_summary` | `07_regional_analysis.ipynb` | Revenue, orders, and customer counts by Brazilian state |
| `cancellation_summary` | `06_cancellation_analysis.ipynb` | Cancellation rate by month, category, and seller |
| `seller_performance` | `03_revenue_analysis.ipynb` | Top sellers by revenue and volume |
| `delivery_performance` | `04_time_trends.ipynb` | Average delivery days vs estimated; on-time rate by state |

> These datasets are kept in-memory intentionally: persisting RFM scores or aggregates to PostgreSQL would introduce denormalized outputs into the warehouse, couple the notebook logic to a specific schema, and reduce reproducibility. Re-running any notebook from clean tables always produces the same result. Power BI aggregates directly against the clean PostgreSQL tables via DAX measures — no pre-aggregated tables are written to the database.

---

## Data Lineage

Shows which source tables flow into each analytical output:

```
customers_clean ──────────────────────────────> RFM Analysis (05)
    │                                            customer segmentation
    └──> orders_clean ──> order_items ────────> Revenue Analysis (03)
              │               │                  category, seller, total revenue
              │               └──> products_clean
              │
              └──> order_payments ────────────> Revenue Analysis (03)
              │                                  payment method breakdown
              │
              └──> reviews_clean ─────────────> Customer Analysis (05)
              │                                  satisfaction, review scores
              │
              └─────────────────────────────────> Cancellation Analysis (06)
                                                  cancellation rate, status trends

orders_clean ─────────────────────────────────> Time Trends (04)
                                                 monthly orders, revenue trends

orders_clean ──────────────────────────────────> Delivery Analysis (04)
customers_clean                                  actual vs estimated delivery
geolocation_clean

orders_clean ──────────────────────────────────> Regional Analysis (07)
customers_clean                                  state-level revenue, orders
geolocation_clean
```

---

## Performance Metrics

| Metric | Value |
|---|---|
| PostgreSQL database size | ~180 MB (all tables) |
| Largest table | `geolocation` — 1,000,163 rows |
| Largest business table | `orders` / `customers` — 99,441 rows each |
| Geolocation after deduplication | 19,015 rows (98.1% reduction) |
| Peak memory during preprocessing | ~500 MB (geolocation load dominates) |
| Delivered orders (analytical scope) | 96,478 of 99,441 (97.0%) |
| Distinct buyers | 96,096 (`customer_unique_id`) |
| Repeat buyers | 3,345 |
| Average review score | 4.09 / 5 |

---

## Key Business Insights

Selected findings from the analysis notebooks:

- **São Paulo dominates revenue** — SP state accounts for the largest share of orders and revenue, followed by RJ and MG. Regional concentration is significant, with the top 3 states representing the majority of sales volume.
- **Credit card is the dominant payment method** — 73.9% of transactions use credit card; boleto (Brazilian bank slip) accounts for 19%, reflecting the local payment landscape.
- **Seasonal demand spike in late 2017** — order volumes peak in November 2017 (Black Friday / holiday season), with a visible month-on-month acceleration through Q4 2017 before normalising in 2018.
- **Delivery delays strongly correlate with low review scores** — orders rated 1-star show significantly longer actual delivery times vs estimated. On-time delivery is the strongest predictor of customer satisfaction in this dataset.
- **RFM segmentation reveals a small high-value core** — Champions and Loyal Customers represent a disproportionate share of total revenue despite being a small proportion of the buyer base.
- **Cancellation rate is low but category-concentrated** — overall cancellation rate is under 1%, but specific categories and sellers show elevated cancellation patterns, suggesting supplier-side fulfilment issues rather than demand-side behaviour.
- **Average review score is 4.09** — the distribution is bimodal: heavily weighted toward 5-star with a secondary spike at 1-star. Mid-range scores (2–4) are underrepresented, consistent with polarised satisfaction patterns common in e-commerce.

---

## Business Questions Addressed

| Notebook | Business Question |
|---|---|
| 03_revenue_analysis | What are total revenues, top categories, and top sellers? |
| 04_time_trends | How do orders and revenue trend month-over-month and seasonally? |
| 05_customer_analysis | Who are our highest-value customers? (RFM segmentation) |
| 06_cancellation_analysis | What drives order cancellations and when do they peak? |
| 07_regional_analysis | Which Brazilian states generate the most revenue and orders? |
