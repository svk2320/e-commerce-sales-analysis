# Data Dictionary

**Project:** Online Retail Sales Analytics
**Dataset:** Olist Brazilian E-Commerce (2016 – 2018)
**Warehouse:** PostgreSQL
**Pipeline:** Python (load_raw → clean → analytics)
**Refresh Frequency:** Historical dataset — no incremental refresh

---

## Data Flow

```
Raw CSV Files
     ↓
load_raw.py  →  Raw Tables (PostgreSQL)
     ↓
clean.py     →  Clean Tables (PostgreSQL)
     ↓
Power BI Dashboard
```

---

## Source System

**Olist Brazilian E-Commerce Dataset**
Covers orders placed between October 2016 and August 2018.

---

## 1. Raw Tables

Loaded as-is by `load_raw.py`. No transformations applied. Each table maps directly to its source CSV.

| Table | Rows | Source File | Description |
|---|---|---|---|
| `customers` | 99,441 | `olist_customers_dataset.csv` | Customer records with location |
| `orders` | 99,441 | `olist_orders_dataset.csv` | Order lifecycle with timestamps |
| `order_items` | 112,650 | `olist_order_items_dataset.csv` | Line items per order — product, seller, price |
| `order_payments` | 103,886 | `olist_order_payments_dataset.csv` | Payment method and value per order |
| `order_reviews` | 99,224 | `olist_order_reviews_dataset.csv` | Customer review scores and comments |
| `products` | 32,951 | `olist_products_dataset.csv` | Product attributes and dimensions |
| `sellers` | 3,095 | `olist_sellers_dataset.csv` | Seller location |
| `geolocation` | 1,000,163 | `olist_geolocation_dataset.csv` | ZIP code to lat/lng mapping |
| `category_translation` | 71 | `product_category_name_translation.csv` | Portuguese → English category names |

---

### customers

**Grain:** One row per customer account (customer_id).
**Primary Key:** `customer_id`

| Column | Type | Description |
|---|---|---|
| `customer_id` | varchar | Unique customer identifier — maps to `orders.customer_id` |
| `customer_unique_id` | varchar | Persistent customer identity across multiple orders |
| `customer_zip_code_prefix` | int | First 5 digits of customer ZIP code |
| `customer_city` | varchar | Customer city name |
| `customer_state` | varchar | Two-letter Brazilian state code (e.g. `SP`, `RJ`) |

> **Note:** `customer_id` is order-scoped — the same physical person can have multiple `customer_id` values across orders. `customer_unique_id` is the stable identifier.

---

### orders

**Grain:** One row per order.
**Primary Key:** `order_id`
**Foreign Keys:** `customer_id → customers.customer_id`

| Column | Type | Description |
|---|---|---|
| `order_id` | varchar | Unique order identifier |
| `customer_id` | varchar | Links to the customer who placed the order |
| `order_status` | varchar | Order lifecycle status — see values below |
| `order_purchase_timestamp` | varchar | When the customer placed the order (raw: string) |
| `order_approved_at` | varchar | When payment was approved; null if not yet approved |
| `order_delivered_carrier_date` | varchar | When the order was handed to the carrier |
| `order_delivered_customer_date` | varchar | When the customer received the order |
| `order_estimated_delivery_date` | varchar | Original estimated delivery date |

**Order status values:**

| Value | Meaning |
|---|---|
| `delivered` | Customer received the order |
| `shipped` | Order dispatched to carrier |
| `processing` | Payment confirmed, preparing to ship |
| `canceled` | Order was cancelled |
| `invoiced` | Invoice issued, awaiting shipment |
| `approved` | Payment approved |
| `unavailable` | Products unavailable |
| `created` | Order created, payment not yet confirmed |

---

### order_items

**Grain:** One row per item within an order (order_id + order_item_id).
**Primary Key:** `order_id` + `order_item_id` (composite)
**Foreign Keys:**
- `order_id → orders.order_id`
- `product_id → products.product_id`
- `seller_id → sellers.seller_id`

| Column | Type | Description |
|---|---|---|
| `order_id` | varchar | Links to the parent order |
| `order_item_id` | int | Sequential item number within the order (1, 2, 3…) |
| `product_id` | varchar | Links to the product |
| `seller_id` | varchar | Links to the seller fulfilling this item |
| `shipping_limit_date` | varchar | Seller's deadline to hand the item to the carrier |
| `price` | float | Item sale price (BRL) |
| `freight_value` | float | Freight cost for this item (BRL) |

---

### order_payments

**Grain:** One row per payment installment per order (order_id + payment_sequential).
**Primary Key:** `order_id` + `payment_sequential` (composite)
**Foreign Keys:** `order_id → orders.order_id`

| Column | Type | Description |
|---|---|---|
| `order_id` | varchar | Links to the parent order |
| `payment_sequential` | int | Payment sequence number — orders can have multiple payment methods |
| `payment_type` | varchar | Payment method — see values below |
| `payment_installments` | int | Number of installments chosen by the customer |
| `payment_value` | float | Amount paid in this payment entry (BRL) |

**Payment type values:**

| Value | Meaning |
|---|---|
| `credit_card` | Credit card payment |
| `boleto` | Brazilian bank slip (cash payment method) |
| `voucher` | Gift voucher or promotional credit |
| `debit_card` | Debit card payment |
| `not_defined` | Payment method not recorded |

---

### order_reviews

**Grain:** One row per review submission.
**Primary Key:** `review_id`
**Foreign Keys:** `order_id → orders.order_id`

| Column | Type | Description |
|---|---|---|
| `review_id` | varchar | Unique review identifier |
| `order_id` | varchar | Links to the reviewed order |
| `review_score` | int | Customer rating: 1 (worst) to 5 (best) |
| `review_comment_title` | varchar | Optional review title; frequently empty |
| `review_comment_message` | varchar | Optional review body text; frequently empty |
| `review_creation_date` | varchar | When the review form was sent to the customer (raw: string) |
| `review_answer_timestamp` | varchar | When the customer submitted the review (raw: string) |

---

### products

**Grain:** One row per product.
**Primary Key:** `product_id`

| Column | Type | Description |
|---|---|---|
| `product_id` | varchar | Unique product identifier |
| `product_category_name` | varchar | Category name in Portuguese; null for some products |
| `product_name_lenght` | int | Character count of the product name (sic — source typo) |
| `product_description_lenght` | int | Character count of the product description (sic) |
| `product_photos_qty` | int | Number of product photos listed |
| `product_weight_g` | float | Product weight in grams |
| `product_length_cm` | float | Product length in centimetres |
| `product_height_cm` | float | Product height in centimetres |
| `product_width_cm` | float | Product width in centimetres |

---

### sellers

**Grain:** One row per seller.
**Primary Key:** `seller_id`

| Column | Type | Description |
|---|---|---|
| `seller_id` | varchar | Unique seller identifier |
| `seller_zip_code_prefix` | int | First 5 digits of seller ZIP code |
| `seller_city` | varchar | Seller city name |
| `seller_state` | varchar | Two-letter Brazilian state code |

---

### geolocation

**Grain:** One row per ZIP code prefix occurrence (contains duplicates — multiple lat/lng per ZIP).
**Primary Key:** None in raw; deduplicated to one row per ZIP in clean layer.

| Column | Type | Description |
|---|---|---|
| `geolocation_zip_code_prefix` | int | 5-digit Brazilian ZIP code prefix |
| `geolocation_lat` | float | Latitude coordinate |
| `geolocation_lng` | float | Longitude coordinate |
| `geolocation_city` | varchar | City name |
| `geolocation_state` | varchar | Two-letter Brazilian state code |

---

### category_translation

**Grain:** One row per product category.
**Primary Key:** `product_category_name`

| Column | Type | Description |
|---|---|---|
| `product_category_name` | varchar | Category name in Portuguese |
| `product_category_name_english` | varchar | Translated English category name |

---

## 2. Clean Tables

Produced by `clean.py`. Applied on top of raw tables — type corrections, null handling, deduplication, and enrichment. These are the tables consumed by the Power BI dashboard.

### orders_clean

**Source:** `orders`
**Grain:** One row per order.
**Primary Key:** `order_id`

Key transformations:
- All five timestamp columns converted from string to `datetime`
- `order_approved_at` nulls backfilled with `order_purchase_timestamp`

---

### products_clean

**Source:** `products` + `category_translation`
**Grain:** One row per product.
**Primary Key:** `product_id`

Key transformations:
- English category name joined from `category_translation` on `product_category_name`
- Products with no matching category assigned `"unknown"` in `product_category_name_english`

---

### customers_clean

**Source:** `customers`
**Grain:** One row per customer_id.
**Primary Key:** `customer_id`

Key transformations:
- Duplicate `customer_id` rows dropped (keeping first occurrence)

---

### geolocation_clean

**Source:** `geolocation`
**Grain:** One row per ZIP code prefix.
**Primary Key:** `geolocation_zip_code_prefix`

Key transformations:
- Deduplicated to one row per `geolocation_zip_code_prefix` (raw has multiple lat/lng entries per ZIP)

---

### reviews_clean

**Source:** `order_reviews`
**Grain:** One row per review.
**Primary Key:** `review_id`

Key transformations:
- `review_comment_title` nulls filled with empty string `""`
- `review_comment_message` nulls filled with empty string `""`
- `review_creation_date` converted from string to `datetime`
- `review_answer_timestamp` converted from string to `datetime`

---

## 3. Field Glossary

| Term | Definition |
|---|---|
| `customer_id` | Order-scoped customer identifier. One physical customer can have multiple `customer_id` values. |
| `customer_unique_id` | Stable customer identity. Use this to count distinct buyers across orders. |
| `order_item_id` | Sequential item number within a single order. An order with 3 products has `order_item_id` values 1, 2, 3. |
| `payment_sequential` | Sequential index of payment entries per order. An order paid partly by voucher and partly by credit card has two rows. |
| `review_score` | Integer rating 1–5 submitted by the customer post-delivery. 5 = best. |
| `freight_value` | Shipping cost per line item in Brazilian Real (BRL). Summing across `order_item_id` gives total freight for the order. |
| `price` | Sale price of the product in Brazilian Real (BRL). Does not include freight. |
| `geolocation_zip_code_prefix` | First 5 digits of a Brazilian CEP (postal code). Used to join customers and sellers to geographic coordinates. |
| `product_category_name_english` | English translation of the Portuguese product category. Assigned `"unknown"` where no translation exists. |
| `boleto` | A Brazilian payment slip (printed or digital) redeemable at banks, post offices, or ATMs. |
| `order_status` | Current lifecycle state of an order. Most analytical work filters to `delivered` orders only. |

---

## 4. Dashboard Consumption

The Power BI dashboard consumes the following clean tables:

| Table | Role |
|---|---|
| `orders_clean` | Order-level facts — revenue, delivery timelines, status |
| `products_clean` | Product dimension — category (English), attributes |
| `customers_clean` | Customer dimension — location, unique identity |
| `geolocation_clean` | Geographic dimension — ZIP to lat/lng for map visuals |
| `reviews_clean` | Review facts — scores and comment text |

Metrics are aggregated dynamically in Power BI. No pre-aggregated tables are written to PostgreSQL.
