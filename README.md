# Orders Data Cleaning & Quality Project

## Problem Summary
A raw orders export (`raw_orders.xlsx`, 932 rows) contained inconsistent text, mixed date formats, missing values, invalid discounts, duplicate records, and sales/profit figures that did not reconcile. The goal was to clean and standardize the data into a reliable analytical dataset, document every decision, flag (not silently drop) problem records, and produce quality and pivot summary reports.

## Dataset Description
- **Grain:** one row per order line.
- **Rows:** 932 raw → 912 cleaned.
- **Key fields:** `order_id`, `order_date`, `ship_date`, customer attributes (`customer_id`, `customer_name`, `segment`), geography (`region`, `state`, `city`), product attributes (`category`, `sub_category`, `product_name`), `ship_mode`, transaction values (`quantity`, `unit_price`, `discount`, `sales`, `cost`, `profit`), and statuses (`payment_status`, `order_status`).
- A second sheet, `business_rules`, defined the required handling rules.

## Tools Used
- **Python** (pandas, numpy, re) for parsing, cleaning, and flagging logic.
-  Excel workbooks with live formulas.

## Cleaning Steps Performed
1. Preserved the raw file; performed all work in `cleaned_orders.xlsx`.
2. Standardized 10 text fields (trim, collapse spaces, strip stray characters, Title Case).
3. Parsed 4 date formats into one ISO date type; derived `shipping_delay_days`.
4. Removed 20 exact duplicate rows; flagged 24 conflicting `order_id` rows for review.
5. Standardized discounts (percent-text → decimal; missing → 0; out-of-range flagged).
6. Applied business rules for missing region/ship_mode/discount and order/payment statuses.
7. Added calculated columns and a per-record `data_quality_flag`.

## Business Rules Applied
- Missing `region` / `ship_mode` → `Unknown` + flagged.
- Missing `discount` → 0 (other sales fields valid) + flagged.
- Negative or >50% discounts → flagged invalid.
- Cancelled / failed-payment / returned orders excluded from completed-sales totals.
- Refunded orders summarized separately.
- Ship-before-order records flagged as invalid shipping.
- *Completed sale* = `order_status = Completed` AND `payment_status = Paid`.

## Summary of Data Quality Issues Found
| Category | Count |
|---|---|
| Exact duplicate rows removed | 20 |
| Conflicting duplicate IDs (flagged) | 12 IDs / 24 rows |
| Missing region / ship_mode / discount | 25 / 21 / 18 |
| Negative discounts | 16 |
| Above-range discounts | 14 |
| Ship date before order date | 21 |
| Sales calculation mismatches | 59 |
| Profit calculation mismatches | 40 |
| **Final:** clean / warning / invalid | **756 / 105 / 51** |

## Summary of Final Pivot Reports (`outputs/pivot_summary.xlsx`)
1. **Sales & profit by region** — sorted by sales.
2. **Sales & profit by category & sub-category** — sorted within category.
3. **Order count by ship mode** — sorted by count.
4. **Profit margin by customer segment.**
5. **Refunded / cancelled / failed orders by region** — filtered to problem records.
6. **Monthly sales trend** — chronological.

## Key Business Insights
- **Completed sales:** 602 orders, **≈ ₹5.93M revenue**, **≈ ₹1.69M profit** (**28.5% overall margin**).
- **Technology** is the top category by completed sales (≈ ₹2.16M).
- Region performance is tight: **South** leads on sales while **East** has the strongest margin (~30%).
- **216 non-completed records** (71 refunded, 76 cancelled, 69 failed payments); refunded orders represent ≈ ₹677k that did not convert to revenue. North region carries the most problem orders.
- Average shipping delay for valid records is **~4 days**.

## Assumptions and Limitations
- Slash dates read as US `MM/DD/YYYY`, dash dates as `DD-MM-YYYY` (data-supported).
- Valid discount range assumed 0–50% (documented business choice).
- Conflicting duplicates are flagged for human review, not auto-resolved.
- Raw sales/profit mismatches are flagged; cleaned recomputed values are authoritative.
- Geography fields standardized for format only, not validated against a reference.
- Full detail in `outputs/cleaning_log.md`.

