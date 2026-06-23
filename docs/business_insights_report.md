# Business Questions & Analytical Findings

**Project:** Brazilian E-Commerce Analytics (Olist)  
**Dataset:** Olist public e-commerce dataset (orders, payments, customers, products, sellers, reviews)  
**Notebooks:** 03 Revenue Analysis · 04 Time Trends · 05 Customer Analysis · 06 Cancellation Analysis · 07 Regional Analysis

---

## Analytical Methodology

The analysis follows a descriptive analytics workflow:

1. Clean and validate raw Olist data
2. Build analytical tables in PostgreSQL (`orders_clean`, `customers_clean`, `products_clean`)
3. Perform SQL-based aggregations with Python (pandas, SQLAlchemy)
4. Visualize trends in Python (matplotlib, seaborn) and Power BI
5. Interpret findings from a business perspective

Unless otherwise stated:
- Revenue analysis uses **delivered orders only** (`order_status = 'delivered'`)
- Customer-level analysis uses `customer_unique_id` (persistent across orders) not `customer_id` (order-scoped)
- Category analyses exclude rows where `product_category_name_english = 'unknown'`
- State-level analyses apply a minimum threshold (>100 orders) to ensure statistical reliability

---

## Core Insights Summary

These cross-cutting patterns emerge across the full analysis:

- The platform processed 96,478 delivered orders generating BRL 16.0M in revenue, with a true average order value of BRL 160.99.
- Revenue and order volume are geographically concentrated: São Paulo alone accounts for 40% of total orders.
- A small number of product categories (bed & bath, health & beauty, computers & accessories) generate the majority of revenue.
- Most customers make exactly one purchase — the platform operates in a low-repeat, high-acquisition mode.
- Credit card is dominant, representing 73% of orders and 78% of total revenue.
- Revenue and order volume trended strongly upward from late 2016 through mid-2018, with a clear November spike (Black Friday effect).
- Aggregate cancellation rate is low (0.63%), but varies meaningfully by product category.
- Delivery performance varies by a factor of 3x across Brazilian states, from 8.8 days (SP) to 26.4 days (AM).

---

## 1. What Is the Overall Revenue and Order Scale?

*(Notebook 03 — § 1)*

**Finding:** The platform processed **96,478 delivered orders** with total payment value of **BRL 16,008,872.12** and a true average order value of **BRL 160.99**.

**Evidence:** The notebook query uses `AVG(payment_value)` over `order_payments` rows (103,886 rows), which returns BRL 154.10 — an average over payment records, not orders. Since one order can have multiple payment rows (installments, split payment types), this understates true AOV. The correct order-level AOV is computed as:
```sql
WITH order_totals AS (
    SELECT order_id, SUM(payment_value) AS order_value
    FROM order_payments
    GROUP BY order_id
)
SELECT
    COUNT(*)          AS total_orders,
    SUM(order_value)  AS total_revenue,
    AVG(order_value)  AS avg_order_value
FROM order_totals;
```
This yields AVG = BRL 160.99 (16,008,872.12 / 99,440 unique orders in the payments table). The 96,478 figure from `orders_clean WHERE order_status = 'delivered'` is the authoritative delivered order count used throughout the analysis.

**Interpretation:** A true AOV of BRL 161 positions Olist in the everyday consumer goods segment — above impulse purchases but below considered electronics or high-ticket categories. This is consistent with the top revenue categories (bed & bath, health & beauty) which are repeat-need, moderate-value purchases.

---

## 2. Which Product Categories Drive the Most Revenue?

*(Notebook 03 — § 2)*

**Finding:** Top 10 categories by total revenue (delivered orders):

| Rank | Category | Revenue (BRL) |
|------|----------|---------------|
| 1 | bed_bath_table | 1,712,553.67 |
| 2 | health_beauty | 1,657,373.12 |
| 3 | computers_accessories | 1,585,330.45 |
| 4 | furniture_decor | 1,430,176.39 |
| 5 | watches_gifts | 1,429,216.68 |
| 6 | sports_leisure | 1,392,127.56 |
| 7 | housewares | 1,094,758.13 |
| 8 | auto | 852,294.33 |
| 9 | garden_tools | 838,280.75 |
| 10 | cool_stuff | 779,698.00 |

**Evidence:** SQL joins `order_payments → order_items → products_clean`, filters delivered orders and excludes 'unknown' categories, aggregates by category.

**Interpretation:** The top 3 categories (bed & bath, health & beauty, computers & accessories) together represent a disproportionate share of platform revenue. Computers & accessories has high unit value but likely lower order count than bed & bath; a count-vs-revenue decomposition would reveal which categories are high-frequency vs high-ticket. Revenue concentration in these categories creates both operational efficiency and category risk.

**Limitation:** Revenue allocation at the category level requires joining order-level payments to item-level products. Orders containing multiple categories may require proportional allocation of payment value across items. Depending on aggregation logic, category revenue may not exactly match order-level revenue totals.

---

## 3. Which Payment Types Are Most Popular?

*(Notebook 03 — § 3)*

**Finding:**

| Payment Type | Orders | Total Revenue (BRL) | Avg Order Value (BRL) |
|---|---|---|---|
| credit_card | 76,795 (73.1%) | 12,542,084.19 (78.3%) | 163.32 |
| boleto | 19,784 (18.8%) | 2,869,361.27 (17.9%) | 145.03 |
| voucher | 5,775 (5.5%) | 379,436.87 (2.4%) | 65.70 |
| debit_card | 1,529 (1.5%) | 217,989.79 (1.4%) | 142.57 |

**Evidence:** SQL aggregates `count`, `sum`, and `avg` of `payment_value` grouped by `payment_type`.

**Interpretation:** Credit card's higher AOV (BRL 163 vs BRL 145 for boleto) reflects Brazilian installment payment behavior (parcelamento) — customers using credit cards for larger purchases spread repayment across multiple months. Boleto, being a one-time cash instrument, is used for lower-value or budget-constrained purchases. Voucher's low AOV (BRL 65) suggests it is primarily used for discounted or subsidized purchases rather than full-price transactions.

**Limitation:** `order_payments` can have multiple rows per `order_id` (one per payment method, for split payments). The aggregation here is at the `payment_type` level, not the `order_id` level. An order using both a voucher and a credit card contributes to both rows. Total order count across payment types will therefore exceed 99,440.

---

## 4. How Has Revenue and Order Volume Trended Over Time?

*(Notebook 04 — § 1)*

**Finding:** Monthly revenue grew from BRL 46,567 (October 2016) to a peak of BRL 1,153,528 in November 2017, remaining in the BRL 965,000–1,133,000 range through mid-2018. Order volume followed the same trajectory, growing from 265 orders/month (Oct 2016) to 7,289 orders/month (Nov 2017) and stabilizing around 6,000–7,000/month through 2018.

**Evidence:** Line chart of monthly revenue and order count from `orders_clean JOIN order_payments`, deduplicated on `order_id` before aggregation.

**Interpretation:** The November 2017 spike (BRL 1.15M, 7,289 orders) represents a Black Friday/Cyber Monday effect — a pattern expected in Brazilian e-commerce. Revenue and orders appear to have plateaued in 2018 after the growth phase, though the dataset ends in August 2018 and may not capture the full-year picture. The 2016 data (only Q4) should be treated as partial and excluded from trend slope calculations.

**Limitation:** October 2016 shows a large BRL value (BRL 46,567) from only 265 orders — the dataset start is sparse and not representative of steady-state operations. December 2016 shows only 1 order (BRL 19.62), which is almost certainly a data artifact. Trend analysis should begin from January 2017.

---

## 5. Which Days of the Week Drive the Most Orders?

*(Notebook 04 — § 2)*

**Finding:**

| Day | Orders | Revenue (BRL) |
|---|---|---|
| Monday | 15,701 | 2,497,696.56 |
| Tuesday | 15,503 | 2,429,522.11 |
| Wednesday | 15,076 | 2,351,500.53 |
| Thursday | 14,322 | 2,248,486.24 |
| Friday | 13,685 | 2,184,855.57 |
| Sunday | 11,635 | 1,779,465.66 |
| Saturday | 10,555 | 1,677,485.76 |

**Evidence:** Aggregated from `order_purchase_timestamp` using `dt.dayofweek` and `dt.day_name()`, after deduplicating on `order_id`.

**Interpretation:** Weekday order volume (especially Mon–Wed) is substantially higher than weekends. This is consistent with Brazilian workplace browsing behavior — purchase intent is higher when consumers are more likely to browse and purchase online during workdays. From an operational standpoint, this means order intake peaks mid-week, concentrating fulfillment pressure on Wednesday–Friday processing windows.

---

## 6. Are There Seasonal Revenue Patterns?

*(Notebook 04 — § 3)*

**Finding:** Revenue by calendar month (aggregated across all years in the dataset):

| Month | Revenue (BRL) | Orders |
|---|---|---|
| January | 1,186,079 | 7,819 |
| February | 1,220,878 | 8,208 |
| March | 1,511,174 | 9,549 |
| April | 1,498,384 | 9,101 |
| May | 1,668,769 | 10,295 |
| June | 1,472,758 | 9,234 |
| July | 1,567,607 | 10,031 |
| August | 1,606,875 | 10,544 |
| September | 687,423 | 4,150 |
| October | 783,276 | 4,743 |
| November | 1,136,309 | 7,289 |
| December | 829,480 | 5,514 |

**Evidence:** Aggregated by `dt.month` and `dt.strftime("%B")` from purchase timestamps.

**Interpretation:** September and October show notably lower revenue and order volume — approximately half of peak months. This is not a seasonal demand trough in the traditional sense; rather, it reflects the fact that the dataset only has one year of Sep/Oct data (2017) compared to 1–2 years of data for other months. May and August show high volumes partly because the dataset covers two complete instances of those months (2017 and 2018). Direct month comparisons should account for this representation imbalance before concluding on seasonality.

**Limitation:** Aggregating across years inflates months with more dataset coverage. A rigorous seasonality analysis would normalize by dividing each month's revenue by the number of years it appears in the dataset, or apply seasonal decomposition on the monthly time series directly.

---

## 7. How Do Customers Segment by RFM?

*(Notebook 05 — § 1–2)*

**Finding:** RFM segmentation across 93,357 unique customers:

| Segment | Customers | Avg Spend (BRL) | Avg Recency (days) |
|---|---|---|---|
| Potential Loyal | 23,356 | 160.98 | 111 |
| At Risk | 23,187 | 166.41 | 363 |
| Loyal | 17,553 | 167.86 | 130 |
| Needs Attention | 17,423 | 160.50 | 335 |
| Champion | 5,938 | 185.92 | 56 |
| Lost | 5,900 | 162.22 | 451 |

**Evidence:** SQL aggregates `last_purchase`, `frequency`, and `monetary` per `customer_unique_id`; quartile scoring (R: lower recency = higher score; F, M: higher = better); segment labels applied via R and F threshold rules.

**Interpretation:** The combined At Risk + Lost population (29,087 customers, 31% of the base) represents the largest retention challenge — these customers purchased previously but have not returned. Their average spend (BRL 162–166) is not meaningfully different from active segments, which means the issue is churn rather than low-value customers leaving. Champions (5,938 customers) have the highest spend (BRL 185.92) and lowest recency (56 days), confirming they are both the most valuable and the most recently active segment.

**Limitation:** Frequency is structurally constrained in this dataset — the majority of customers have exactly 1 order, which compresses the F-score quartile distribution. Quartile boundaries in the F dimension are therefore very close together, meaning small differences in order count (1 vs 2 orders) can produce meaningfully different F scores. Segment labels should be interpreted relative to this dataset's behavioral range, not as absolute descriptors.

---

## 8. Which RFM Segments Have the Highest Average Spend?

*(Notebook 05 — § 3)*

**Finding:** Champions lead on average spend at BRL 185.92. The remaining segments cluster tightly between BRL 160–168, with no large separation between active and churned segments on monetary value alone.

**Evidence:** `segment_summary` aggregation of `avg(monetary)` per segment, from the RFM dataframe.

**Interpretation:** The narrow monetary spread across segments (BRL 160–186) indicates that customers who churned were not systematically low-value buyers — their spend was similar to active customers. This shifts the retention framing: the At Risk and Lost segments are not to be discounted as low-value; they are full-value customers who stopped purchasing. Re-engagement campaigns targeting these segments have a reasonable expected revenue per recovery.

---

## 9. What Is the Overall Cancellation Rate?

*(Notebook 06 — § 1)*

**Finding:** Order status distribution across all 99,441 orders:

| Status | Count | % of Total |
|---|---|---|
| delivered | 96,478 | 97.02% |
| shipped | 1,107 | 1.11% |
| canceled | 625 | 0.63% |
| unavailable | 609 | 0.61% |
| invoiced | 314 | 0.32% |
| processing | 301 | 0.30% |

**Evidence:** Window-function SQL computing `COUNT(*) * 100.0 / SUM(COUNT(*)) OVER()` per `order_status`.

**Interpretation:** A 0.63% cancellation rate is low in absolute terms and healthy for an e-commerce marketplace. The `unavailable` status (0.61%) is nearly as large as `canceled` and may reflect seller stock-outs or product de-listing after order placement — a distinct fulfillment failure mode worth tracking separately from customer-initiated cancellations. The ~3% non-delivered orders (shipped + unavailable + processing + invoiced) represent orders still in-flight or stuck, which may affect customer experience even if they do not register as cancellations.

---

## 10. How Have Cancellations Trended Over Time?

*(Notebook 06 — § 2)*

**Finding:** Monthly cancellation counts follow the overall order volume trend — rising through 2017 and 2018 as the platform scaled.

**Evidence:** SQL filters `order_status = 'canceled'` and aggregates by `DATE_TRUNC('month', order_purchase_timestamp)`.

**Interpretation:** Because total order volume grew significantly over the observation period, the raw cancellation count increase does not necessarily indicate a worsening cancellation problem. The meaningful metric is the cancellation rate (cancellations / total orders per month). If the rate is stable while counts rise, the platform is scaling normally. A rate-adjusted time series would be the appropriate follow-up analysis to confirm this.

---

## 11. Which Categories Have the Most Cancellations (Absolute)?

*(Notebook 06 — § 3)*

**Finding:** The top 10 categories by absolute cancelled order count skew toward the same high-volume categories that generate the most orders overall (bed & bath, health & beauty, etc.).

**Evidence:** SQL joins `orders_clean → order_items → products_clean`, filters `order_status = 'canceled'`, groups by category.

**Interpretation:** Absolute cancellation count is a biased metric — it favors high-volume categories by construction. A category appearing in this ranking primarily because it generates large order volume is not evidence of a cancellation problem specific to that category. The rate-adjusted view (Section 12) is the correct diagnostic lens.

---

## 12. Which Categories Have the Highest Cancellation Rate?

*(Notebook 06 — § 4)*

**Finding:** When controlling for order volume (minimum 100 orders per category), the highest cancellation-rate categories differ from the highest absolute-count categories. High-rate categories tend to be lower-volume niches rather than platform-wide top sellers.

**Evidence:** SQL computes `SUM(CASE WHEN order_status = 'canceled' THEN 1 ELSE 0 END) * 100.0 / COUNT(DISTINCT order_id)` per category, filtered to `HAVING COUNT > 100`.

**Interpretation:** Elevated cancellation rates in specific categories suggest product-level issues: longer fulfillment lead times generating buyer impatience, higher price points increasing post-purchase doubt, or product descriptions that systematically mismatch delivered goods. These categories are the appropriate targets for seller quality review and SLA improvement, not categories that simply have large order counts.

---

## 13. Which States Generate the Most Revenue and Orders?

*(Notebook 07 — § 1)*

**Finding:** Top 10 states by revenue (delivered orders):

| State | Orders | Revenue (BRL) |
|---|---|---|
| SP | 40,500 | 5,770,266.19 |
| RJ | 12,350 | 2,055,690.45 |
| MG | 11,354 | 1,819,277.61 |
| RS | 5,345 | 861,802.40 |
| PR | 4,923 | 781,919.55 |
| SC | 3,546 | 595,208.40 |
| BA | 3,256 | 591,270.60 |
| DF | 2,080 | 346,146.17 |
| GO | 1,957 | 334,294.22 |
| ES | 1,995 | 317,682.65 |

**Evidence:** SQL joins `customers_clean → orders_clean → order_payments`, filters to delivered orders, groups by `customer_state`.

**Interpretation:** São Paulo alone generates 40.7% of total orders and 36.0% of total revenue. The top 3 states (SP, RJ, MG) together account for 64.2% of orders. This concentration reflects Brazil's population and income distribution — the Southeast is both the most populous and the highest-income region. Operationally, logistics performance in SP directly determines the platform's aggregate delivery metrics.

---

## 14. Which States Have the Highest Average Order Value?

*(Notebook 07 — § 2)*

**Finding:** Paraíba (PB) leads on average order value at **BRL 250.15**, despite ranking 16th by order volume (517 orders). High-volume São Paulo has a lower AOV by comparison.

**Evidence:** SQL computes `AVG(payment_value)` per `customer_state`, filtered to `HAVING COUNT(DISTINCT order_id) > 100`.

**Interpretation:** Higher AOV in lower-volume states may reflect: a different product mix (e.g., higher proportion of electronics or high-value goods purchased online in regions with fewer physical retail options), or wealthier demographics within those states making larger online purchases. Without joining to product category at the state level, these explanations are not distinguishable from the current analysis. The `HAVING > 100` filter ensures the average is computed over a statistically stable base.

---

## 15. Which States Have the Slowest and Fastest Delivery Times?

*(Notebook 07 — § 3)*

**Finding:**
- **Fastest:** São Paulo (SP) — **8.8 days** average
- **Slowest:** Amazonas (AM) — **26.4 days** average

**Evidence:** SQL computes `AVG(EXTRACT(EPOCH FROM (order_delivered_customer_date - order_purchase_timestamp)) / 86400)` per `customer_state`, filtered to delivered orders with non-null delivery timestamps and `HAVING COUNT(*) > 100`.

**Interpretation:** The 3x delivery time gap between SP and AM reflects Brazil's logistics infrastructure reality — northern and remote states are served by longer, less-frequent carrier routes. SP's advantage comes from proximity to Olist's seller base and dense carrier coverage. This delivery time disparity is likely a meaningful driver of customer satisfaction and repeat purchase rates in those regions, independent of product quality or seller behavior.

**Limitation:** This metric captures total elapsed time from purchase timestamp to customer delivery, which includes: seller processing time, carrier pickup time, in-transit time, and last-mile time. The current analysis does not decompose these stages. The bottleneck — whether it is slow seller fulfillment or slow carrier transit — cannot be identified without sequentially comparing `order_approved_at`, `order_delivered_carrier_date`, and `order_delivered_customer_date`.

---

## Assumptions, Limitations & Caveats

- **Payment table fan-out:** `order_payments` has multiple rows per `order_id` (installments, split payment types). Aggregations that do not deduplicate on `order_id` will over-count revenue. This analysis uses `nunique` or single-query aggregations where needed.
- **Customer identity:** `customer_unique_id` (persists across orders) is used for customer-level analysis. Using `customer_id` instead would incorrectly treat returning customers as new buyers and deflate apparent repeat rates.
- **Sparse dataset edges:** October–December 2016 and parts of August 2018 have partial data. Trend and seasonal analyses that include these periods should flag them as edge-incomplete months.
- **Delivery time null exclusion:** Orders with null `order_delivered_customer_date` are excluded from delivery time analysis. These may be systematically non-random (unfulfilled orders, disputes), introducing survivorship bias into delivery time estimates.
- **Category label completeness:** Product category labels come from the English translation mapping. Rows where translation is missing or 'unknown' are excluded from all category-level analyses. Categories with incomplete coverage may be underrepresented.
- **RFM quartile compression:** Most customers have frequency = 1, compressing the F-score quartile boundaries. RFM segment labels describe relative behavior within this dataset, not absolute behavioral archetypes.
- **Revenue vs. GMV:** `SUM(payment_value)` from `order_payments` approximates revenue but may differ from seller-reported GMV due to installment recording, voucher subsidies, and partial payment structures.

---

## Executive Summary

Key findings from the Brazilian E-Commerce Analytics (Olist) project:

- The platform processed **96,478 delivered orders** (99,440 total across all statuses) generating **BRL 16.0 million** in total revenue, with a true average order value of **BRL 160.99**.
- Revenue is concentrated in three product categories — bed & bath, health & beauty, and computers & accessories — which together account for a disproportionate share of platform GMV.
- **São Paulo dominates** both order volume (40.7% of total orders) and revenue (36.0% of total revenue). The top 3 states (SP, RJ, MG) generate 64% of all orders.
- **Credit card is the dominant payment method**, representing 73% of orders and 78% of revenue, reflecting Brazilian installment payment behavior. Boleto accounts for most of the remainder.
- Revenue and order volume trended strongly upward from early 2017 through mid-2018, with a clear November 2017 peak consistent with Black Friday promotional activity.
- **Customer repeat purchasing is low** — most customers purchased exactly once, with 31% of the base classified as At Risk or Lost. These customers have comparable average spend to active segments, making re-engagement viable.
- The aggregate cancellation rate is low at **0.63%**, but varies by product category. Rate-adjusted category analysis identifies niche categories with disproportionate cancellation risk.
- Delivery performance ranges from **8.8 days (SP) to 26.4 days (AM)** — a 3x gap driven by logistics infrastructure differences across Brazilian regions.

---

*Generated from: `03_revenue_analysis.ipynb`, `04_time_trends.ipynb`, `05_customer_analysis.ipynb`, `06_cancellation_analysis.ipynb`, `07_regional_analysis.ipynb`*
