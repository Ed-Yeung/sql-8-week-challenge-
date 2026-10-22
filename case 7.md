# Case Study #7 — Balanced Tree Clothing Co.

**Portfolio focus:** SQL sales analysis, transaction-level metrics, product merchandising, customer membership analysis, and a reusable monthly reporting script.

> This is the Balanced Tree Clothing Co. case study (officially Case Study #7 in the 8 Week SQL Challenge). It is separate from Case Study #5, Data Mart.

## 1. Business problem

Balanced Tree Clothing Co. sells clothing and lifestyle products across product categories and segments. The merchandising team needs to understand product performance and share a repeatable financial report with the wider business. The CEO wants a set of sales and transaction metrics that can be produced each month without rewriting every query.

The analysis therefore has two goals:

1. **Understand performance:** quantify sales volume, gross revenue, discount value, customer baskets, membership differences, and product/category contribution.
2. **Operationalise reporting:** provide one SQL script that can be re-run for a selected reporting month by changing only the start and end dates.

Source: [Official Balanced Tree Clothing Co. case study](https://8weeksqlchallenge.com/case-study-7/).

## 2. Dataset and metric definitions

The regular questions use two tables:

- `balanced_tree.sales`: product-level transaction lines, including product ID, quantity, unit price, discount percentage, member flag, transaction ID, and transaction timestamp.
- `balanced_tree.product_details`: product name, category, segment, and product hierarchy fields.

The sales table is joined to product details using `sales.prod_id = product_details.product_id`.

### Financial definitions used in this project

The `discount` field represents a **percentage**, not a currency amount. The report therefore calculates:

```sql
-- Gross sales value, before discount
qty * price

-- Monetary discount value
qty * price * discount / 100.0

-- Net sales value, after discount
qty * price * (1 - discount / 100.0)
```

This distinction matters. `SUM(discount)` measures the sum of stored percentage values and is **not** the total discount amount in currency. Similarly, subtracting `discount` directly from `qty * price` does not calculate net revenue correctly.

## 3. Reporting approach

The reusable script is in [`monthly_report.sql`](./monthly_report.sql). It uses MySQL 8.0+ syntax and returns separate result sets for the report sections.

The reporting window is controlled by two variables:

```sql
SET @report_start = '2021-01-01 00:00:00';
SET @report_end   = '2021-02-01 00:00:00';
```

The start is inclusive and the end is exclusive. This avoids accidentally omitting timestamps later on the final calendar day. To run February, change the dates to `2021-02-01` and `2021-03-01`.

The script creates temporary reporting tables for the selected month, transaction-level metrics, and product-level metrics. Reusing these intermediate tables keeps the calculations consistent across the report and reduces repeated logic.

## 4. Business questions, SQL approach, and interpretation

### A. High-level sales analysis

**Questions:** How many units were sold? What was revenue before discount? What was the monetary value of discounts?

**SQL approach:** Aggregate quantity, gross revenue, discount amount, and net revenue across all selected sales lines.

**Business use:** These metrics form the top-line monthly trading summary. Gross revenue shows sales value before promotions; discount amount quantifies the cost of promotions; net revenue estimates sales value after discount.

**Interpretation caution:** Do not compare discount totals calculated as `SUM(discount)` with monetary revenue. The percentage must be applied to line value first.

### B. Transaction analysis

**Questions:** How many unique transactions occurred? How many distinct products were purchased per transaction on average? What are the 25th, 50th, and 75th percentiles of transaction revenue? What is the average discount per transaction? What is the member/non-member transaction split? How does average revenue differ by member status?

**SQL approach:** First aggregate sales lines to one row per `txn_id`. From that transaction table, calculate basket size, gross revenue, discount amount, net revenue, and a numeric member flag. Apply ranking to transaction-level gross revenue to calculate percentile estimates.

**Business use:**

- The transaction count and average number of distinct products describe customer activity and basket breadth.
- Revenue percentiles show a typical transaction and the spread between smaller and larger baskets; the median is less affected by very large purchases than the mean.
- Average discount per transaction quantifies promotional value at the basket level.
- Member/non-member comparisons help the business decide whether member transactions differ in value and whether membership should be investigated further.

**Method note:** The supplied draft uses `DENSE_RANK()` to derive percentile positions. `DENSE_RANK()` counts distinct revenue values rather than all transactions, so it can select incorrect percentile cut-offs when revenue values repeat. The report script uses `ROW_NUMBER()` across transaction rows and a discrete nearest-rank approach for P25/P75; for P50 it averages the middle value(s). This percentile convention should be documented and kept consistent across reporting periods.

### C. Product analysis

#### Top three products by revenue before discount

**Question:** Which products generate the most gross revenue?

**SQL approach:** Sum `qty * price` by product and sort descending. The report also displays discount value and net revenue for context.

**Business use:** Identify products with the largest contribution to sales value and prioritise merchandising review. Revenue leadership should not automatically be interpreted as the strongest unit demand or highest profit, because cost and margin data are not supplied.

#### Quantity, revenue, and discounts by segment and category

**Question:** How do product segments and broad categories contribute to sales?

**SQL approach:** Aggregate quantities, gross revenue, discount value, and net revenue at segment and category levels.

**Business use:** Compare product lines and categories to identify where sales volume and sales value are concentrated. A segment that sells many units may not necessarily generate the highest revenue if its unit prices are lower.

#### Top-selling product within each segment/category

**Question:** Which product sells the most units within each product group?

**SQL approach:** Rank product-level quantity using `DENSE_RANK()` within each segment or category; the query preserves ties.

**Important distinction:** The supplied draft ranks some “top-selling” products by revenue. That answers which product generates the most revenue, not which product sells the most units. This README uses **quantity sold** for “top-selling” and keeps revenue-ranked analysis separate.

#### Revenue contribution by product, segment, and category

**Questions:** What percentage of each segment's revenue comes from each product? What percentage of a category's revenue comes from each segment? What is each category's share of total revenue?

**SQL approach:** Aggregate at the level asked by the question first, then divide each group's revenue by its parent total using window functions.

**Business use:** Contribution percentages reveal whether a segment relies heavily on one product and whether total sales are concentrated in a particular category. High concentration can help focus merchandising review, although it does not by itself prove risk or indicate profitability.

**Aggregation note:** A query grouped by `category_name, product_name` calculates product contribution within a category; it does not calculate segment contribution within a category. The latter requires aggregation by `category_name, segment_name` before the percentage is calculated.

#### Product transaction penetration

**Question:** What share of transactions included each product at least once?

**SQL approach:** For each product, count distinct transactions containing it and divide by the total number of transactions in the reporting month.

**Business use:** Penetration measures how broadly a product appears across customer baskets. A product can have high revenue but lower penetration if it tends to appear in fewer, higher-value transactions.

#### Most common combination of three products

**Question:** Which combination of three distinct products most frequently appears in the same transaction?

**SQL approach:** Deduplicate each transaction/product pair, self-join the product list three times, and use a consistent product-ID ordering to avoid counting the same combination in different orders.

**Business use:** Frequent co-purchase combinations may help merchandising teams consider cross-selling, bundles, or product placement. Co-occurrence is descriptive, not proof that one product causes another to sell.

## 5. Review of the supplied SQL draft

The SQL and outputs supplied for this case are useful working material, but several formulas need correcting before their numbers are presented as final portfolio findings:

1. **Discount amount:** `SUM(discount)` is not the monetary discount when `discount` is a percentage. Use `SUM(qty * price * discount / 100.0)`.
2. **Revenue before discount:** use `SUM(qty * price)` without subtracting anything.
3. **Revenue after discount:** use `SUM(qty * price * (1 - discount / 100.0))` or subtract the calculated monetary discount from gross revenue.
4. **Member flag:** do not rely on `AVG(member)` if the source stores values such as `t` and `f`. Convert membership explicitly to 1/0 at transaction level.
5. **Percentiles:** dense ranks can distort percentile selection when transactions share the same revenue; rank transaction rows instead.
6. **Top-selling product:** the ranking metric should match the business question. Use quantity for best seller by units and revenue for top product by sales value.
7. **Revenue contribution:** aggregate at the intended level (product within segment, segment within category, or category within all sales) before calculating the denominator.
8. **Monthly consistency:** filter every report output using the same transaction date window so that numerator and denominator cover the same month.

The figures in the supplied query output should be treated as **draft values pending a rerun with the corrected definitions and a data-quality check**. This README intentionally does not present those draft figures as validated business results.

## 6. SQL skills demonstrated

- Aggregation with `SUM`, `AVG`, `COUNT`, and `COUNT(DISTINCT ...)`
- Multi-table joins
- Common table expressions (CTEs)
- Window functions: `ROW_NUMBER`, `DENSE_RANK`, and windowed totals
- Conditional aggregation for member/non-member comparisons
- Percentile estimation at transaction level
- Product basket combinations and transaction penetration
- Reusable monthly filtering and temporary reporting tables
- Translating business questions into metric definitions

## 7. Repository files

- `README.md` — business context, analytical method, interpretation, and data-quality notes.
- `monthly_report.sql` — reusable MySQL 8.0+ monthly reporting script.

## 8. Limitations and next steps

- The supplied schema does not include product cost, margin, inventory, or marketing campaign details; revenue metrics cannot establish profit or campaign effectiveness.
- Membership comparisons are observational and do not establish that membership causes higher spending.
- Product combinations show co-occurrence, not causation.
- Before publishing numeric findings, run the corrected script against the intended monthly source data and validate transaction counts, product counts, discount rates, and reporting-period coverage.

## Reference

- [8 Week SQL Challenge — Case Study #7: Balanced Tree Clothing Co.](https://8weeksqlchallenge.com/case-study-7/)
