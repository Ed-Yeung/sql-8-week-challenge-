# 8-Week SQL Challenge — Data Analytics Portfolio

This repository documents my solutions to selected case studies from the **[8 Week SQL Challenge](https://8weeksqlchallenge.com/)** created by **[Data With Danny](https://www.linkedin.com/company/datawithdanny/)**.

The goal of this portfolio is to demonstrate how SQL can be used to answer business questions, investigate customer and operational behaviour, and translate query results into practical insights. Rather than presenting SQL code alone, each case study aims to explain the business problem, the analytical approach, the SQL solution, and what the results may mean for decision-making.

> **Approach:** Business context → analytical questions → SQL queries → results → business insights and recommendations → assumptions and limitations.

## Case Studies

The table below links to the case-study write-ups currently in this repository. Case 6 and Case 8 have not yet been documented here.

| Case | Business / Dataset | Main analytical focus | Solution |
|---|---|---|---|
| **1** | Danny's Diner | Customer spending and visit frequency, popular menu items, loyalty-programme behaviour, and points calculations | [Read Case Study #1]([Case%201.md](https://github.com/Ed-Yeung/sql-8-week-challenge-/blob/959ea09cabb49c3b052fdf8d5bd6ea78db4e92f8/Case%201.md)) |
| **2** | Pizza Runner | Order and delivery operations, data cleaning, pizza and topping trends, delivery performance, and revenue scenarios | [Read Case Study #2](Case%202.md) |
| **3** | Foodie-Fi | Subscription lifecycle, trial conversion, plan changes, churn, annual-plan upgrades, and payment-ledger analysis | [Read Case Study #3](case%203.md) |
| **4** | Data Bank | Customer transactions, node allocations, balance calculations, and data-allocation scenarios | [Read Case Study #4](case%204.md) |
| **5** | Data Mart | Weekly sales performance, data preparation, sales segmentation, and before-and-after analysis of a sustainable-packaging rollout | [Read Case Study #5](case%205%20.md) |
| **6** | Clique Bait | Planned: digital-funnel and customer journey analysis | *unable to process as the data file is too large* |
| **7** | Balanced Tree Clothing Co. | Sales and discount metrics, transaction behaviour, product rankings, revenue mix, product penetration, and product combinations | [Read Case Study #7](case%207.md) |
| **8** | Fresh Segments | Planned: customer-segment and interest-affinity analysis | *Not yet documented* |

## SQL Skills Demonstrated

Across the available case studies, I practise and apply:

- **Querying and aggregation:** `SELECT`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `COUNT`, `SUM`, `AVG`, and distinct counts.
- **Joining datasets:** `INNER JOIN`, `LEFT JOIN`, and combining transaction, customer, product, and lookup data.
- **Multi-step analysis:** Common Table Expressions (CTEs), subqueries, conditional aggregation, and `CASE` expressions.
- **Window functions:** Ranking, running totals, period comparisons, and calculations with `PARTITION BY`.
- **Date and time analysis:** Date conversion, period filtering, time differences, weekly and monthly summaries, and lifecycle sequencing.
- **Data preparation:** Handling inconsistent values, cleaning strings, transforming fields, and checking data quality before analysis.
- **Business analysis:** Customer behaviour, loyalty and subscriptions, operational performance, sales trends, revenue mix, and product performance.

Most case-study solutions use **MySQL 8.0+**; please check the individual case page for its stated SQL dialect and version requirements.

## How Each Case Is Presented

Where applicable, a case-study page includes:

1. **Business context and objectives** — what the business wants to understand or improve.
2. **Dataset and schema** — key tables, fields, and relationships.
3. **Questions and SQL solutions** — the query used for each question, kept alongside its explanation.
4. **Results and interpretation** — what the output shows, with a distinction between observed results and derived metrics where relevant.
5. **Business insights and recommendations** — potential actions suggested by the analysis.
6. **Assumptions and limitations** — data-quality concerns, methodological choices, and what the data cannot prove.

## Notes on Interpretation and Validation

The case studies are analytical exercises using the datasets and assumptions provided by the challenge. A result can describe an observed pattern, but it does not automatically establish its cause. For example, a before-and-after sales comparison can show that performance changed around an event, but other factors may also have contributed.

I aim to make the SQL and reasoning transparent. Where a case page uses previously recorded query outputs or has not been independently re-executed against a fresh database, the page should make that limitation clear. Derived calculations and business recommendations should be interpreted in light of the definitions and limitations described in the relevant case.


The reference repository is used for learning and structural inspiration. The case write-ups in this repository are intended to communicate my own SQL approach and business interpretation rather than reproduce another author's explanation verbatim.

