# Pre-Analysis Data Validation Checklist

Use this checklist alongside [SOP_simple.md](SOP_simple.md) or [SOP_detailed.md](SOP_detailed.md) before approving any dataset for BI dashboards, reporting, or machine learning.

**Dataset Name:** `_______________________________` *(e.g., Olist Brazilian E-Commerce)*  
**Target Schema / Environment:** `_______________________________` *(e.g., PostgreSQL `olist_audit`)*  
**Validation Date:** `YYYY-MM-DD`  

---

## 1. Structure & Schema Check
- [ ] All expected CSV files / tables arrived and match the manifest count (`9/9` tables for Olist).
- [ ] File encoding verified as `UTF-8` with no corrupted special characters or broken delimiters.
- [ ] Row counts per table match expected baseline volumes (within $\pm 5\%$ for recurring feeds).
- [ ] Column names and data types (`VARCHAR`, `INTEGER`, `NUMERIC`, `TIMESTAMP`) match `schema/create_tables.sql`.
- [ ] Staging tables loaded without truncation or parsing errors.

## 2. Duplicate Checks (`queries/01_duplicates.sql`)
- [ ] **Primary Keys:** Verified `0` duplicate primary keys on core transactional tables (e.g., `orders.order_id`, `customers.customer_id`, `products.product_id`).
- [ ] **Composite Keys:** Verified uniqueness on multi-column keys (e.g., `(order_id, order_item_id)` in `order_items`, `(order_id, payment_sequential)` in `order_payments`).
- [ ] **Shared Entity IDs:** Checked whether child tables reuse identifiers across orders (e.g., `order_reviews.review_id` — `814` duplicate rows flagged and documented).
- [ ] **Reference / Lookup Tables:** Checked lookup tables for exact row duplicates or 1:N fan-out hazards (e.g., `geolocation` — `261,831` duplicate rows deduplicated before joining).

## 3. NULL & Conditional Completeness Checks (`queries/02_nulls.sql`)
- [ ] **Mandatory Keys:** Verified `0` NULL values in primary and foreign key columns (`order_id`, `customer_id`, `product_id`, `seller_id`).
- [ ] **Status-Dependent Timestamps:**
  - [ ] Checked `delivered` orders for missing `order_delivered_customer_date` (`8` rows flagged).
  - [ ] Checked `delivered` orders for missing `order_approved_at` (`14` rows flagged).
  - [ ] Checked `delivered` / `shipped` orders for missing `order_delivered_carrier_date` (`2` rows flagged).
- [ ] **Catalog & Dimension Attributes:** Checked dimension tables for missing categories or labels (`610` products with `product_category_name IS NULL` mapped to `'unclassified'`).

## 4. Range, Format & Chronological Checks (`queries/03_invalid_values.sql`)
- [ ] **Monetary Ranges:**
  - [ ] Verified `order_items.price > 0` (`0` violations) and `freight_value >= 0` (`0` negative values).
  - [ ] Checked `order_payments.payment_value <= 0` (`9` zero-value rows flagged and filtered/documented).
- [ ] **Physical Measurements:** Checked `products.product_weight_g > 0` and dimensions `> 0` (`4` zero-weight products flagged).
- [ ] **Chronological Order of Timestamps:**
  - [ ] Verified `order_delivered_customer_date >= order_delivered_carrier_date` (`23` inverted rows quarantined).
  - [ ] Verified `order_delivered_carrier_date >= order_purchase_timestamp` (`166` inverted rows quarantined).
  - [ ] Verified `order_delivered_carrier_date >= order_approved_at` (`1,359` rows flagged).
- [ ] **Pattern / Format Validation:** Verified zip code prefixes, state codes (`2`-letter ISO UF), and email/ID patterns via `validator.py`.

## 5. Orphan Records & Cross-Table Reconciliation (`queries/04_orphan_records.sql` & `05_cross_table_mismatch.sql`)
- [ ] **Parent-Child Referential Integrity:**
  - [ ] `orders -> customers` (`0` orphans).
  - [ ] `orders -> order_payments` (`1` delivered order missing payment logged).
  - [ ] `orders -> order_items` (`775` unavailable/canceled orders without items logged).
  - [ ] `products -> category_translation` (`13` products in `pc_gamer` and `portateis_cozinha_e_preparadores_de_alimentos` logged).
- [ ] **Financial Reconciliation:** Compared `SUM(price + freight_value)` vs. `SUM(payment_value)` per order (`310` orders with difference `> R$ 0.05` documented).
- [ ] **Lifecycle Contradictions:** Checked `canceled` orders for populated `order_delivered_customer_date` (`6` rows logged).

---

## 6. Issue Logging & Final Sign-Off
- [ ] Exported all flagged records to `output/discrepancy_report.csv`.
- [ ] Logged all non-zero findings in `templates/issue_log_template.xlsx` with Severity, Owner, and Status.
- [ ] Confirmed whether any **High (Blocker)** issues require pipeline remediation before publishing analysis.

### Approval Block

| Role | Name | Decision (`Approved` / `Approved with Caveats` / `Rejected`) | Signature / Handle | Date |
| :--- | :--- | :--- | :--- | :--- |
| **Validator (Data Analyst)** | | | | |
| **Reviewer (Analytics Lead)** | | | | |
