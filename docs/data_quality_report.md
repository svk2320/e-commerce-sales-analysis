# Data Quality Documentation

**Project:** Online Retail Sales Analytics
**Dataset:** Olist Brazilian E-Commerce (2016 – 2018)
**Warehouse:** PostgreSQL
**Pipeline:** Python (`load_raw.py` → `clean.py` → Power BI)
**Last Updated:** 2026-06-16

---

## Executive Summary

| Metric | Value |
|---|---|
| Tables Profiled | 9 |
| Validation Checks | 11 |
| Checks Passed | 11 |
| Checks Failed | 0 |
| Referential Integrity Coverage | 100% |
| Duplicate PK Violations | 0 |
| Business Table Retention | 100% |

- Geolocation deduplicated from 1,000,163 rows to 19,015 distinct ZIP prefixes (by design — no analytical data lost).
- No critical data quality issues identified after cleaning and validation.

**Source notebooks:** `00_explore.ipynb`, `01_profile.ipynb`
**Validation scripts:** `src/validation/assumption_test.py`
**Cleaning pipeline:** `src/preprocess/clean.py`

---

## Data Quality Checks — Summary

Results from `src/validation/assumption_test.py` run on 2026-06-16.

| Check | Table | Status |
|---|---|---|
| No null order_ids | `orders` | ✅ PASS |
| No duplicate order_ids | `orders` | ✅ PASS |
| No null customer_ids | `orders` | ✅ PASS |
| No null customer_ids | `customers` | ✅ PASS |
| No duplicate customer_ids | `customers` | ✅ PASS |
| No negative prices | `order_items` | ✅ PASS |
| No null product_ids | `order_items` | ✅ PASS |
| All order_items have matching order | `order_items → orders` | ✅ PASS |
| No negative payment values | `order_payments` | ✅ PASS |
| No null payment values | `order_payments` | ✅ PASS |
| No null product_ids | `products` | ✅ PASS |

**Total: 11 checks — PASS: 11 | FAIL: 0 | WARN: 0**

---

## Data Quality Rules

The following constraints must hold true across all tables. These rules are encoded in `assumption_test.py` and validated on every pipeline run.

- `order_id` must be non-null and unique in `orders`
- `customer_id` must be non-null in `orders` and `customers`
- `payment_value` must be ≥ 0
- `price` must be ≥ 0 in `order_items`
- `order_items.order_id` must exist in `orders`
- `order_items.product_id` must exist in `products`
- `order_items.seller_id` must exist in `sellers`

---

## 1. Profiling Summary

### 1.1 Row Counts — All Raw Tables

| Table | Rows |
|---|---|
| `customers` | 99,441 |
| `orders` | 99,441 |
| `order_items` | 112,650 |
| `order_payments` | 103,886 |
| `order_reviews` | 99,224 |
| `products` | 32,951 |
| `sellers` | 3,095 |
| `geolocation` | 1,000,163 |
| `category_translation` | 71 |

**Date range:** Orders span October 2016 – August 2018, confirmed via `MIN` / `MAX` of `order_purchase_timestamp`.

---

### 1.2 Null Analysis

#### orders

| Column | Nulls | % Null | Explanation |
|---|---|---|---|
| `order_approved_at` | 160 | 0.16% | Orders where payment approval has not yet been recorded |
| `order_delivered_carrier_date` | 1,783 | 1.79% | Orders not yet handed to carrier (non-delivered statuses) |
| `order_delivered_customer_date` | 2,965 | 2.98% | Orders not yet received by customer |

All other `orders` columns: **0 nulls**.

> **Decision:** `order_approved_at` nulls are backfilled with `order_purchase_timestamp` in the clean layer. The remaining delivery timestamp nulls are expected — non-delivered orders have no delivery date by definition. No imputation applied.

---

#### order_items

All columns: **0 nulls**. No negative prices. No negative freight values.

---

#### order_payments

All columns: **0 nulls**. No negative payment values.

---

#### order_reviews

| Column | Nulls | % Null | Explanation |
|---|---|---|---|
| `review_comment_title` | 87,656 | 88.3% | Title field is optional — most reviewers skip it |
| `review_comment_message` | 58,247 | 58.7% | Message field is optional |

> **Decision:** Both comment fields filled with empty string `""` in `reviews_clean`. This makes downstream string handling predictable without imputing false content.

---

#### products

| Column | Nulls | % Null | Explanation |
|---|---|---|---|
| `product_category_name` | 610 | 1.85% | Products with no category assigned in source data |
| `product_name_lenght` | 610 | 1.85% | Same 610 products missing name length |
| `product_description_lenght` | 610 | 1.85% | Same 610 products missing description length |
| `product_photos_qty` | 610 | 1.85% | Same 610 products missing photo count |
| `product_weight_g` | 2 | 0.01% | Two products with no weight recorded |
| `product_length_cm` | 2 | 0.01% | Same two products |
| `product_height_cm` | 2 | 0.01% | Same two products |
| `product_width_cm` | 2 | 0.01% | Same two products |

> **Decision:** Products with null `product_category_name` are assigned `"unknown"` in `product_category_name_english` after the category translation join. The two products with null dimensions are kept in the clean layer — they are valid products with incomplete physical attributes, and dropping them would silently remove revenue from any join.

---

### 1.3 Referential Integrity

Validated in `01_profile.ipynb` (Section 8).

| Relationship | Orphaned Rows | Coverage |
|---|---|---|
| `order_items.order_id → orders.order_id` | 0 | 100% |
| `order_payments.order_id → orders.order_id` | 0 | 100% |
| `order_reviews.order_id → orders.order_id` | 0 | 100% |
| `order_items.product_id → products.product_id` | 0 | 100% |
| `order_items.seller_id → sellers.seller_id` | 0 | 100% |

All foreign key relationships are fully intact in the raw layer.

---

### 1.4 Duplicate Check — Primary Keys

| Table | Key | Duplicates |
|---|---|---|
| `orders` | `order_id` | 0 |
| `customers` | `customer_id` | 0 |
| `order_reviews` | `review_id` | 0 |
| `products` | `product_id` | 0 |
| `sellers` | `seller_id` | 0 |
| `order_items` | `order_id + order_item_id` | 0 |
| `order_payments` | `order_id + payment_sequential` | 0 |

All primary keys are unique across all tables.

---

### 1.5 Order Status Distribution

| Status | Count | % of Orders |
|---|---|---|
| `delivered` | 96,478 | 96.97% |
| `shipped` | 1,107 | 1.11% |
| `canceled` | 625 | 0.63% |
| `unavailable` | 609 | 0.61% |
| `invoiced` | 314 | 0.32% |
| `processing` | 301 | 0.30% |
| `created` | 5 | 0.01% |
| `approved` | 2 | <0.01% |

> **Note:** Most analytical work in the Power BI dashboard filters to `delivered` orders only (96,478 rows). Revenue and delivery metrics are not meaningful for orders in any other status.

---

### 1.6 Price & Freight Distribution (order_items)

| Metric | Price (BRL) | Freight (BRL) |
|---|---|---|
| Min | 0.85 | 0.00 |
| P25 | 39.90 | 13.08 |
| P50 (median) | 74.90 | 19.62 |
| P75 | 134.90 | 33.07 |
| P99 | 579.99 | 96.00 |
| Max | 6,735.00 | 409.68 |
| Mean | 120.65 | 19.99 |
| Negative values | 0 | 0 |
| Zero price items | 2 | — |

The price distribution is right-skewed, driven by a small number of high-value products. The two zero-price items are verified as present in the raw source data and are kept; they may represent promotional or bundled items.

---

### 1.7 Payment Distribution

| Payment Type | Count | % of Payments |
|---|---|---|
| `credit_card` | 76,795 | 73.9% |
| `boleto` | 19,784 | 19.0% |
| `voucher` | 5,775 | 5.6% |
| `debit_card` | 1,529 | 1.5% |
| `not_defined` | 3 | <0.01% |

| Metric | Payment Value (BRL) | Installments |
|---|---|---|
| Min | 0.00 | 0 |
| Max | 13,664.08 | 24 |
| Mean | 154.10 | 2.85 |
| Zero values | present (voucher partial payments) | — |
| Installments > 24 | 0 | — |

---

### 1.8 Review Score Distribution

| Score | Count | % |
|---|---|---|
| 5 | 57,420 | 57.8% |
| 4 | 19,142 | 19.3% |
| 3 | 8,179 | 8.2% |
| 2 | 3,151 | 3.2% |
| 1 | 11,332 | 11.4% |

Average review score: **4.09**

The distribution is bimodal: heavily weighted toward 5-star reviews with a secondary spike at 1-star. This is a common pattern in e-commerce review data.

---

### 1.9 Geolocation Duplicates

| Metric | Value |
|---|---|
| Total raw rows | 1,000,163 |
| Distinct ZIP prefixes | 19,015 |
| Duplicate ZIP entries | 981,148 |
| Max entries per single ZIP | — |

The raw table averages roughly 53 lat/lng entries per ZIP prefix. One representative coordinate per ZIP prefix is retained in `geolocation_clean` to prevent join fan-out in downstream analysis.

---

### 1.10 Customer Identity Check

| Metric | Value |
|---|---|
| Total `customer_id` rows | 99,441 |
| Distinct `customer_unique_id` | 96,096 |
| Repeat buyers | 3,345 |

`customer_id` is order-scoped — the same physical customer can appear multiple times with different `customer_id` values. Use `customer_unique_id` for buyer-count metrics.

---

## 2. Assumptions & Business Rules

### 2.1 Data Cleaning & Transformation Rules

| Table | Rule | Raw Issue | Clean Result | Decision |
|---|---|---|---|---|
| `orders` | Cast 5 timestamp columns to `datetime` | `varchar` dtype | `datetime64` | Type cast |
| `orders` | Backfill `order_approved_at` nulls | 160 nulls | 0 nulls | Fill with `order_purchase_timestamp` |
| `products` | Join English category name | No English column | `product_category_name_english` added | Left join on `category_translation` |
| `products` | Fill unmatched category | 610 nulls post-join | `"unknown"` string | `fillna("unknown")` |
| `customers` | Remove duplicate `customer_id` rows | 0 duplicates found | 0 duplicates | `drop_duplicates` (no rows removed) |
| `geolocation` | Deduplicate to one row per ZIP | 981,148 duplicate entries | 1 row per ZIP | `drop_duplicates` on ZIP prefix |
| `order_reviews` | Fill null comment title & message | 87,656 / 58,247 nulls | Empty string `""` | `fillna("")` |
| `order_reviews` | Cast date columns to `datetime` | `varchar` dtype | `datetime64` | Type cast |

---

### 2.2 Cleaning Impact — Row Counts

| Raw Table | Raw Rows | Clean Rows | Rows Removed | Retention |
|---|---|---|---|---|
| `orders` | 99,441 | 99,441 | 0 | 100.0000% |
| `products` | 32,951 | 32,951 | 0 | 100.0000% |
| `customers` | 99,441 | 99,441 | 0 | 100.0000% |
| `geolocation` | 1,000,163 | 19,015 | 981,148 | 1.9012% |
| `order_reviews` | 99,224 | 99,224 | 0 | 100.0000% |

> `geolocation` is intentionally collapsed from 1,000,163 raw entries to 19,015 distinct ZIP prefixes. No analytical data is lost — the raw table contains repeated lat/lng coordinates for the same ZIP codes.
>
> **Total rows removed from business tables: 0. Total rows removed from geolocation: 981,148.**

---

### 2.3 Analytical Filter Convention

Most Power BI dashboard metrics apply an implicit filter: `order_status = 'delivered'`.

| Scope | Row Count |
|---|---|
| All orders | 99,441 |
| Delivered orders only | 96,478 |
| Non-delivered (excluded from most metrics) | 2,963 |

---

### 2.4 Derived Fields

| Field | Logic | Location |
|---|---|---|
| `product_category_name_english` | Left join `products → category_translation`; unmatched → `"unknown"` | `products_clean` |
| Delivery delay flag | `order_delivered_customer_date > order_estimated_delivery_date` | Computed in Power BI |
| Actual delivery days | `order_delivered_customer_date − order_purchase_timestamp` | Computed in Power BI |

---

## 3. Known Issues, Assumptions & Limitations

### ⚠️ `order_approved_at` nulls in raw orders

- **What:** 160 orders (0.16%) have no `order_approved_at` timestamp.
- **Why:** Orders where payment approval has not been recorded — likely `canceled` or `created` status orders that never reached approval.
- **Impact:** Without backfill, any pipeline step requiring a non-null approval timestamp would fail or silently drop these rows.
- **Decision:** Backfilled with `order_purchase_timestamp` in `orders_clean` so that all rows have a non-null approval timestamp. Downstream delivery metrics are unaffected since these orders are non-delivered and filtered out of revenue analysis.

---

### ⚠️ Products with null physical dimensions

- **What:** 2 products have null `product_weight_g`, `product_length_cm`, `product_height_cm`, and `product_width_cm`.
- **Why:** Incomplete records in the source dataset.
- **Impact:** These products cannot be used in any weight- or dimension-based analysis (e.g. freight cost modelling). They are valid products with sales data and are not dropped.
- **Decision:** Kept in `products_clean`. Any analysis involving product dimensions should filter or handle these two rows explicitly.

---

### ⚠️ `payment_type = 'not_defined'` entries

- **What:** 3 payment rows have `payment_type = 'not_defined'`.
- **Why:** Payment method was not recorded for these transactions.
- **Impact:** Negligible — 3 rows out of 103,886 (< 0.01%).
- **Decision:** Kept as-is. Excluded from payment method breakdowns in the dashboard.

---

### ⚠️ Zero-price order items

- **What:** 2 order items have `price = 0.00`.
- **Why:** Likely promotional, bundled, or sample items included at no charge.
- **Impact:** Minimal effect on revenue totals. Including them in average price calculations slightly deflates the mean.
- **Decision:** Kept. These are valid records present in the source data.

---

### ℹ️ `customer_id` is order-scoped — not a buyer identifier

- **What:** The same physical customer can have multiple `customer_id` values across orders.
- **Why:** Olist's data model assigns a new `customer_id` per order, not per person.
- **Impact:** Using `customer_id` to count distinct buyers will over-count. The dataset has 99,441 `customer_id` values but only 96,096 distinct buyers (`customer_unique_id`).
- **Decision:** The Power BI dashboard uses `customer_unique_id` for all distinct buyer counts.

---

### ℹ️ Geolocation has multiple coordinates per ZIP code

- **What:** The raw `geolocation` table contains 1,000,163 rows for 19,015 distinct ZIP prefixes — an average of ~53 entries per ZIP.
- **Why:** The source maps each ZIP to multiple lat/lng points rather than a single centroid.
- **Impact:** Joining customers or sellers directly to raw geolocation would produce a fan-out (one customer → many map points).
- **Decision:** `geolocation_clean` retains one representative coordinate per ZIP prefix to prevent join fan-out in downstream analysis. This is the table used for all map visuals in Power BI.

---

## Conclusion

The dataset passed all automated validation checks and retained 100% of business records after cleaning. Identified data quality issues were either expected by design (optional review comments, non-delivered order timestamps) or addressed through documented cleaning rules. The resulting clean layer is suitable for downstream analytics and Power BI reporting.

*Last updated from profiling, cleaning, and validation workflows.*
