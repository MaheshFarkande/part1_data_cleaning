# Cleaning Log — Orders Dataset

**Source file:** `data/raw_orders.xlsx` (sheet `raw_orders`, 932 rows, 21 columns)
**Output file:** `data/cleaned_orders.xlsx` (912 rows, 31 columns)
**Raw file preserved:** `data/raw_orders.xlsx` was copied and never modified.

---

## 1. Issues Found

| # | Issue | Where | Scale |
|---|-------|-------|-------|
| 1 | Inconsistent casing & stray/leading/trailing/internal spaces in text fields | `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status` | e.g. 27 raw `customer_name` variants → 15 real customers; `segment` had 16 raw variants → 4 |
| 2 | Mixed date formats | `order_date`, `ship_date` | 4 distinct formats (MM/DD/YYYY, DD-MM-YYYY, DD Mon YYYY, YYYY-MM-DD) |
| 3 | Ship date earlier than order date | dates | 21 records |
| 4 | Missing values | `region` (25), `ship_mode` (21), `discount` (18) | 64 cells |
| 5 | Negative discounts | `discount` | 16 records |
| 6 | Discounts above allowed range | `discount` | 14 records (0.55, 0.65, plus 8 written as `70%`/`85%`) |
| 7 | Discounts stored as percent text (`70%`, `85%`) | `discount` | 8 records |
| 8 | Exact duplicate rows | whole row | 20 rows |
| 9 | Duplicate `order_id` with conflicting data | `order_id` | 12 IDs / 24 rows |
| 10 | Raw `sales` ≠ qty × price × (1 − discount) | `sales` | 59 records |
| 11 | Raw `profit` ≠ `sales` − `cost` | `profit` | 40 records |

---

## 2. Cleaning Actions Performed

**Text fields** — trimmed leading/trailing spaces, collapsed internal multi-spaces to single spaces, removed stray special characters, standardized to Title Case. This unified variants such as `SMALL BUSINESS`, `small  business`, `Small Business ` into `Small Business`.

**Dates** — each value parsed against its detected format and converted to a single ISO date type (`yyyy-mm-dd`). Format detection was unambiguous: slash dates use month-first (max first part = 12), dash dates use day-first (max first part = 31), so no date was guessed. A `shipping_delay_days` column was derived as `ship_date − order_date`.

**Discounts** — percent-text values divided by 100 (`70%` → 0.70); missing values set to 0; the standardized value (`cleaned_discount`) was bounded to the valid range and the original out-of-range condition recorded separately for flagging.

**Duplicates** — exact full-row duplicates removed (kept first occurrence). Conflicting same-`order_id` rows were **flagged, not deleted**.

**Calculated columns added** to `cleaned_orders.xlsx` (live Excel formulas): `calculated_sales`, `calculated_profit`, `profit_margin`, `shipping_delay_days`, `order_month`, `order_year`, plus `cleaned_discount`, `is_completed_sale`, `record_category`, `data_quality_flag`, `quality_notes`.

---

## 3. Business Rules Applied

| Rule | Decision implemented |
|------|----------------------|
| Missing `region` | Filled as `Unknown`, flagged (warning) — 25 records |
| Missing `ship_mode` | Filled as `Unknown`, flagged (warning) — 21 records |
| Missing `discount` | Set to `0` (all other sales fields were valid), flagged — 18 records |
| Negative `discount` | Flagged **invalid**; bounded to 0 for calculation — 16 records |
| Discount above allowed range | Allowed range set to **0–50%**; values above flagged **invalid**, bounded to 0.50 — 14 records |
| Cancelled orders | Excluded from completed-sales summary |
| Failed payments | Excluded from completed-sales summary |
| Refunded orders | Excluded from completed sales; summarized separately (71 records, ≈ ₹677k) |
| Ship date before order date | Flagged **invalid shipping record** — 21 records |

**Completed sale** is defined as `order_status = Completed` AND `payment_status = Paid` (602 records).

---

## 4. Assumptions Made

1. **Date interpretation:** slash format = `MM/DD/YYYY` (US), dash format = `DD-MM-YYYY`. Supported by the data — no first-part value exceeded 12 for slash dates, and dash dates required day-first.
2. **Discount allowed range = 0% to 50%.** The standard discount tiers in the data are 0–25%; 55%–85% values are treated as out-of-range outliers. This threshold is a documented business choice and can be adjusted.
3. **Percent-text discounts** (`70%`) represent 0.70, not 70.0.
4. **Missing discount → 0** is applied because every record with a missing discount had valid `quantity`, `unit_price`, `sales` and `cost`.
5. **`calculated_sales` / `calculated_profit`** (recomputed from quantity, unit price, cleaned discount, and cost) are treated as the authoritative figures; the original `sales`/`profit` columns are retained for reference and their mismatches flagged.
6. **Returned orders** are treated as non-finalized and excluded from completed sales.

---

## 5. Records Removed

- **20 exact duplicate rows** removed (byte-for-byte identical, first kept). No other rows were deleted.

## 6. Records Flagged (not removed)

- **24 conflicting duplicate rows** (12 order IDs) flagged for manual review.
- **30 invalid-discount** records (16 negative, 14 above range).
- **21 ship-before-order** records.
- **64 missing-value** records auto-filled and flagged (`region`/`ship_mode`/`discount`).
- **59 sales** and **40 profit** raw-calculation mismatches flagged as warnings.
- Overall: **756 clean**, **105 warning**, **51 invalid** records.

---

## 7. Limitations

1. **Conflicting duplicates are not auto-resolved.** Without a source of truth, the correct version among conflicting `order_id` rows cannot be chosen automatically; these are left for human review.
2. **Discount range threshold is a judgment call.** The 50% ceiling is reasonable for this data but not derived from an official policy document.
3. **Raw sales/profit mismatches are flagged, not corrected in place.** The cleaned file provides recomputed values, but it cannot determine which original figure (if any) was intended.
4. **No external validation** of `state`/`city`/`region` consistency against a geographic reference; only formatting was standardized.
5. **Dates assumed Gregorian and within 2024–2025**; values were not cross-checked against an external calendar source.
