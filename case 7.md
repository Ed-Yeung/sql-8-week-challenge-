# 8 Week SQL Challenge — Case Study #7: Balanced Tree Clothing Co.

## Project overview

This case study analyses sales transactions for Balanced Tree Clothing Co. using SQL. The goal is to turn transaction-level sales records into a repeatable set of business metrics that the team can use for sales monitoring, merchandising decisions, customer-membership analysis, and product-bundling ideas.

**SQL dialect:** MySQL 8.0+ (the queries use common table expressions and window functions).

**Important:** This README is deliberately self-contained: the SQL code appears inside the relevant question sections, together with the outputs supplied with the original queries and an explanation of what the results mean. The result tables below are transcribed from the supplied material; the queries have not been independently executed against a database while preparing this README.

## Business questions and analysis

# 1. High-Level Sales Analysis

### Business goal

Establish a high-level view of sales volume, gross sales value before discounts, and the total discount recorded. These are useful starting points for a monthly commercial performance report.

### Question 1.1 — What was the total quantity sold for all products?

### Question 1.2 — What was the total generated revenue for all products before discounts?

### Question 1.3 — What was the total discount amount for all products?

The three measures are calculated together in one query.

```sql
SELECT
    SUM(qty) AS total_quantity,
    SUM(qty * price) AS revenue_b4_discount,
    SUM(discount) AS tot_discountAmount
FROM balanced_tree.sales;
```

**Supplied result**

| Total quantity | Revenue before discount | Total discount amount |
|---:|---:|---:|
| 135,648 | 3,868,359 | 548,100 |

### Analysis and business insight

- **Sales volume:** 135,648 product units were sold across the dataset.
- **Gross sales value:** `SUM(qty * price)` gives 3,868,359 before subtracting discounts.
- **Discounts:** the sum of the `discount` field is 548,100. Based on these figures, discounts are approximately **14.17% of gross sales value** (548,100 / 3,868,359). This is a derived ratio, not a separately supplied query result.
- **Derived net sales value:** gross sales less the supplied discount total is **3,320,259**, assuming the `discount` field represents an amount that can be subtracted directly from each line's gross sales.

**Business use:** Track quantity, gross sales, discount value, and net sales together each reporting period. A discount total should not be interpreted in isolation: it is most useful alongside the sales value it was intended to generate and the net sales remaining after discounts.

---

# 2. Transaction Analysis

### Business goal

Understand transaction size and value, the spread of transaction revenue, discounting by transaction, and differences between member and non-member transactions. The supplied query aggregates the sales rows into one row per transaction before calculating the overall measures.

### Question 2.1 — How many unique transactions were there?

### Question 2.2 — What is the average number of unique products purchased in each transaction?

### Question 2.3 — What are the 25th, 50th, and 75th percentile values for revenue per transaction?

### Question 2.4 — What is the average discount value per transaction?

### Question 2.5 — What is the percentage split of all transactions for members versus non-members?

### Question 2.6 — What is the average revenue for member transactions and non-member transactions?

The submitted query answers these six questions in one CTE-based query.

```sql
WITH tb1 AS (
    SELECT
        txn_id,
        SUM(qty * price) - SUM(discount) AS revenue,
        COUNT(DISTINCT prod_id) AS unique_products,
        SUM(discount) AS tot_discount,
        ROUND(AVG(member), 0) AS mb
    FROM balanced_tree.sales
    GROUP BY txn_id
),
tb2 AS (
    SELECT
        txn_id,
        revenue,
        unique_products,
        tot_discount,
        mb,
        DENSE_RANK() OVER (ORDER BY revenue) AS rk
    FROM tb1
),
tb3 AS (
    SELECT
        txn_id,
        revenue,
        unique_products,
        rk,
        tot_discount,
        mb,
        MAX(rk) OVER () AS tot_rk
    FROM tb2
),
tb4 AS (
    SELECT
        MAX(CASE WHEN mb = 0 THEN avg_rev END) AS avg_rev_non_memeber,
        MAX(CASE WHEN mb = 1 THEN avg_rev END) AS avg_rev_memeber
    FROM (
        SELECT
            mb,
            AVG(revenue) AS avg_rev
        FROM tb3
        GROUP BY mb
    ) AS t
)
SELECT
    COUNT(DISTINCT txn_id) AS unique_tran,
    ROUND(AVG(unique_products)) AS avg_unique_perTran,
    ROUND(AVG(tot_discount)) AS avg_tot_discount,
    MAX(CASE WHEN rk = CEIL(tot_rk * 0.25) THEN revenue END) AS p25_rev,
    MAX(CASE WHEN rk = CEIL(tot_rk * 0.50) THEN revenue END) AS p50_rev,
    MAX(CASE WHEN rk = CEIL(tot_rk * 0.75) THEN revenue END) AS p75_rev,
    SUM(mb) / COUNT(DISTINCT txn_id) AS member_pert,
    1 - (SUM(mb) / COUNT(DISTINCT txn_id)) AS non_member_pert,
    (SELECT avg_rev_non_memeber FROM tb4) AS avg_rev_non_memeber,
    (SELECT avg_rev_memeber FROM tb4) AS avg_rev_memeber
FROM tb3;
```

**Supplied result**

| Metric | Supplied result |
|---|---:|
| Unique transactions | 2,500 |
| Average unique products per transaction (`ROUND` to whole number) | 6 |
| Average discount per transaction (`ROUND` to whole number) | 219 |
| P25 transaction revenue (using the submitted rank logic) | 798 |
| P50 transaction revenue (using the submitted rank logic) | 1,368 |
| P75 transaction revenue (using the submitted rank logic) | 1,953 |
| Member transaction share | 60.20% |
| Non-member transaction share | 39.80% |
| Average non-member transaction revenue | 1,323.1899 |
| Average member transaction revenue | 1,331.3522 |

### How the SQL works

1. **`tb1` — transaction-level aggregation:** Groups the sales lines by `txn_id`; calculates transaction revenue, distinct products, discount total, and a transaction-level member flag.
2. **`tb2` — rank revenue:** Uses `DENSE_RANK()` to rank transaction revenue from low to high.
3. **`tb3` — adds the maximum revenue rank:** Adds the largest rank value to each row for the percentile calculation.
4. **`tb4` — member vs. non-member average:** Calculates average transaction revenue for the two membership groups and pivots those group averages into columns.
5. **Final `SELECT`:** Produces the count, rounded averages, rank-based revenue cut-offs, member transaction proportions, and two average revenue measures.

### Analysis and business insight

- **Membership mix:** 60.20% of transactions are classified as member transactions and 39.80% as non-member transactions. Membership transactions make up the majority of recorded transactions.
- **Average transaction value:** Member transactions average 1,331.35 compared with 1,323.19 for non-member transactions. The supplied figures differ by about **8.16**, or roughly **0.62%** of the non-member average. This is a small descriptive difference; it does not establish that membership causes greater spending.
- **Transaction size:** The average count of distinct products rounds to six. Because the SQL rounds to a whole number, it does not show the precise average.
- **Discounts:** The average transaction discount rounds to 219. A more detailed report could retain decimal places to make comparisons between months easier.
- **Revenue distribution:** The submitted query reports P25, P50, and P75 values of 798, 1,368, and 1,953. These should be treated as values produced by this specific ranking method, not as verified standard percentiles (see the validation note below).

### SQL validation note — percentile calculation

The submitted code sets `tot_rk` to `MAX(rk)`, where `rk` is a `DENSE_RANK()` over transaction revenue. Therefore, `tot_rk` is the number of **distinct revenue ranking values**, not necessarily the number of transactions. The expression `CEIL(tot_rk * 0.25)` can consequently select different cut-offs from a percentile method based on all 2,500 transactions, especially when transaction revenues tie. For a production report, define whether a discrete or continuous percentile is wanted and use a method that applies that definition to the complete transaction-level dataset.

The transaction-level flag `ROUND(AVG(member), 0)` also assumes that the member value is consistent across every line in a transaction. That assumption should be checked.

---

# 3. Product Analysis

### Business goal

Identify products that contribute the most revenue, understand which products lead within a merchandise segment or category, assess product-level revenue concentration, measure product penetration across transactions, and identify common product combinations.

**Revenue definition used in the supplied product queries:** most queries calculate `SUM(qty * price - discount)`, which subtracts the discount. That is net revenue under the assumed meaning of the `discount` field, not revenue before discount. The original queries are preserved below and this difference is called out rather than silently changed.

## Question 3.1 — What are the top 3 products by total revenue before discount?

### Submitted SQL

```sql
SELECT rk, product_name, rev
FROM (
    SELECT
        DENSE_RANK() OVER (
            ORDER BY SUM(s.qty * s.price - s.discount) DESC
        ) AS rk,
        prod_id,
        product_name,
        SUM(s.qty) AS tot_q,
        MAX(s.price),
        SUM(s.discount) AS tot_dis,
        SUM(s.qty * s.price - s.discount) AS rev
    FROM balanced_tree.sales AS s
    LEFT JOIN balanced_tree.product_details AS d
        ON prod_id = product_id
    GROUP BY prod_id, product_name
    ORDER BY rev DESC
) AS t
WHERE rk <= 3
ORDER BY rk;
```

**Supplied result**

| Rank | Product | Revenue using submitted expression |
|---:|---|---:|
| 1 | Blue Polo Shirt - Mens | 1,212,780 |
| 2 | Grey Fashion Jacket - Womens | 1,162,824 |
| 3 | White Tee Shirt - Mens | 819,078 |

### Analysis and business insight

Blue Polo Shirt - Mens ranks first under the query's revenue expression, followed by Grey Fashion Jacket - Womens and White Tee Shirt - Mens. These products are useful candidates for closer merchandising review because they have the highest aggregated revenue under the calculation used.

**Important interpretation issue:** the question says “before discount”, but the SQL uses `qty * price - discount`. The results above therefore correspond to the submitted **post-discount/net revenue expression**, assuming `discount` is an amount subtracted from line revenue. To answer gross revenue before discount, use `SUM(s.qty * s.price)` consistently in the rank and output calculations.

---

## Question 3.2 — What are the total quantity, revenue, and discount for each segment? What is the top-selling product for each segment?

### Submitted SQL

```sql
SELECT rk, segment_name, product_name, tot_q, tot_dis, rev
FROM (
    SELECT
        DENSE_RANK() OVER (
            PARTITION BY segment_name
            ORDER BY SUM(s.qty * s.price - s.discount) DESC
        ) AS rk,
        segment_name,
        product_name,
        SUM(s.qty) AS tot_q,
        SUM(s.discount) AS tot_dis,
        SUM(s.qty * s.price - s.discount) AS rev
    FROM balanced_tree.sales AS s
    LEFT JOIN balanced_tree.product_details AS d
        ON prod_id = product_id
    GROUP BY segment_name, product_name
) AS t
WHERE rk = 1;
```

**Supplied result**

| Segment | Top product | Quantity of top product | Discount for top product | Revenue using submitted expression |
|---|---|---:|---:|---:|
| Jacket | Grey Fashion Jacket - Womens | 23,256 | 93,000 | 1,162,824 |
| Jeans | Black Straight Jeans - Womens | 22,716 | 91,542 | 635,370 |
| Shirt | Blue Polo Shirt - Mens | 22,914 | 93,318 | 1,212,780 |
| Socks | Navy Solid Socks - Mens | 22,752 | 93,876 | 725,196 |

### Analysis and business insight

- **Jacket:** Grey Fashion Jacket - Womens is the leading product within the Jacket segment.
- **Jeans:** Black Straight Jeans - Womens leads the Jeans segment.
- **Shirt:** Blue Polo Shirt - Mens leads the Shirt segment and has the highest net-revenue figure among the four segment leaders shown.
- **Socks:** Navy Solid Socks - Mens leads the Socks segment.

The query correctly ranks products within each segment by the expression it uses. However, because it filters to `rk = 1`, the output only shows the leading product per segment; it does **not** report the overall total quantity, total revenue, and total discount for every segment. The additional query below calculates segment-level totals separately.

### Additional SQL — total metrics by segment

```sql
SELECT
    d.segment_name,
    SUM(s.qty) AS total_quantity,
    SUM(s.qty * s.price) AS gross_revenue_before_discount,
    SUM(s.discount) AS total_discount,
    SUM(s.qty * s.price - s.discount) AS net_revenue
FROM balanced_tree.sales AS s
LEFT JOIN balanced_tree.product_details AS d
    ON s.prod_id = d.product_id
GROUP BY d.segment_name
ORDER BY net_revenue DESC;
```

This is an additional query to fully answer the first part of the business question. No output was supplied for this added query, so no numeric results are claimed here.

---

## Question 3.3 — What are the total quantity, revenue, and discount for each category? What is the top-selling product for each category?

### Submitted SQL

```sql
SELECT rk, category_name, product_name, rev
FROM (
    SELECT
        DENSE_RANK() OVER (
            PARTITION BY category_name
            ORDER BY SUM(s.qty * s.price - s.discount) DESC
        ) AS rk,
        category_name,
        product_name,
        SUM(s.qty) AS tot_q,
        SUM(s.discount) AS tot_dis,
        SUM(s.qty * s.price - s.discount) AS rev
    FROM balanced_tree.sales AS s
    LEFT JOIN balanced_tree.product_details AS d
        ON prod_id = product_id
    GROUP BY category_name, product_name
) AS T
WHERE rk = 1;
```

**Supplied result**

| Category | Top product | Revenue using submitted expression |
|---|---|---:|
| Mens | Blue Polo Shirt - Mens | 1,212,780 |
| Womens | Grey Fashion Jacket - Womens | 1,162,824 |

### Analysis and business insight

The top product in the Mens category is Blue Polo Shirt - Mens, while Grey Fashion Jacket - Womens leads the Womens category. This provides a simple headline product for each category under the submitted revenue calculation.

As in the segment query, the submitted query filters to rank 1. It therefore does not show total quantity, revenue, and discount for the full category. The additional query below supplies those category totals.

### Additional SQL — total metrics by category

```sql
SELECT
    d.category_name,
    SUM(s.qty) AS total_quantity,
    SUM(s.qty * s.price) AS gross_revenue_before_discount,
    SUM(s.discount) AS total_discount,
    SUM(s.qty * s.price - s.discount) AS net_revenue
FROM balanced_tree.sales AS s
LEFT JOIN balanced_tree.product_details AS d
    ON s.prod_id = d.product_id
GROUP BY d.category_name
ORDER BY net_revenue DESC;
```

This is an additional query to fully answer the category-level totals question. Its output was not included in the supplied results.

---

## Question 3.4 — What is the percentage split of revenue by product for each segment?

### Submitted SQL

```sql
SELECT
    rk,
    segment_name,
    product_name,
    tot_q,
    tot_dis,
    rev,
    (rev / SUM(rev) OVER (PARTITION BY segment_name)) * 100 AS percentage
FROM (
    SELECT
        DENSE_RANK() OVER (
            PARTITION BY segment_name
            ORDER BY SUM(s.qty * s.price - s.discount) DESC
        ) AS rk,
        segment_name,
        product_name,
        SUM(s.qty) AS tot_q,
        SUM(s.discount) AS tot_dis,
        SUM(s.qty * s.price - s.discount) AS rev
    FROM balanced_tree.sales AS s
    LEFT JOIN balanced_tree.product_details AS d
        ON prod_id = product_id
    GROUP BY segment_name, product_name
) AS t;
```

**Supplied result**

| Segment | Product | Quantity | Discount | Revenue using submitted expression | Share of segment revenue |
|---|---|---:|---:|---:|---:|
| Jacket | Grey Fashion Jacket - Womens | 23,256 | 93,000 | 1,162,824 | 60.2754% |
| Jacket | Khaki Suit Jacket - Womens | 22,512 | 88,014 | 429,762 | 22.2769% |
| Jacket | Indigo Rain Jacket - Womens | 22,542 | 91,698 | 336,600 | 17.4478% |
| Jeans | Black Straight Jeans - Womens | 22,716 | 91,542 | 635,370 | 65.1221% |
| Jeans | Navy Oversized Jeans - Womens | 23,136 | 92,508 | 208,260 | 21.3456% |
| Jeans | Cream Relaxed Jeans - Womens | 22,242 | 90,390 | 132,030 | 13.5324% |
| Shirt | Blue Polo Shirt - Mens | 22,914 | 93,318 | 1,212,780 | 56.1316% |
| Shirt | White Tee Shirt - Mens | 22,800 | 92,922 | 819,078 | 37.9097% |
| Shirt | Teal Button Up Shirt - Mens | 21,876 | 90,018 | 128,742 | 5.9586% |
| Socks | Navy Solid Socks - Mens | 22,752 | 93,876 | 725,196 | 46.0421% |
| Socks | Pink Fluro Polkadot Socks - Mens | 22,620 | 89,676 | 566,304 | 35.9542% |
| Socks | White Striped Socks - Mens | 21,930 | 89,238 | 283,572 | 18.0037% |

### Analysis and business insight

- **Jacket segment:** Grey Fashion Jacket - Womens contributes about 60.28% of the segment's revenue, making the segment relatively concentrated in one product.
- **Jeans segment:** Black Straight Jeans - Womens contributes about 65.12% of Jeans revenue, the highest leading-product share among the four segments shown.
- **Shirt segment:** Blue Polo Shirt - Mens and White Tee Shirt - Mens together contribute about 94.04% of Shirt revenue (56.1316% + 37.9097%). Teal Button Up Shirt - Mens accounts for about 5.96%.
- **Socks segment:** Navy Solid Socks - Mens is the largest contributor at about 46.04%, followed by Pink Fluro Polkadot Socks - Mens at about 35.95%.

**Business use:** Product revenue share can help merchandising teams prioritise inventory planning and identify category concentration. It should be considered together with margin, inventory availability, returns, and product lifecycle information, none of which is included in this query.

---

## Question 3.5 — What is the percentage split of revenue by segment for each category?

### Submitted SQL (preserved as supplied)

```sql
SELECT
    rk,
    category_name,
    product_name,
    tot_q,
    tot_dis,
    rev,
    (rev / SUM(rev) OVER (PARTITION BY category_name)) * 100 AS percentage
FROM (
    SELECT
        DENSE_RANK() OVER (
            PARTITION BY category_name
            ORDER BY SUM(s.qty * s.price - s.discount) DESC
        ) AS rk,
        category_name,
        product_name,
        SUM(s.qty) AS tot_q,
        SUM(s.discount) AS tot_dis,
        SUM(s.qty * s.price - s.discount) AS rev
    FROM balanced_tree.sales AS s
    LEFT JOIN balanced_tree.product_details AS d
        ON prod_id = product_id
    GROUP BY category_name, product_name
) AS t;
```

**Supplied result — this is product share within each category, because the query groups by `product_name`.**

| Category | Product | Revenue using submitted expression | Share of category revenue |
|---|---|---:|---:|
| Mens | Blue Polo Shirt - Mens | 1,212,780 | 32.4648% |
| Mens | White Tee Shirt - Mens | 819,078 | 21.9259% |
| Mens | Navy Solid Socks - Mens | 725,196 | 19.4127% |
| Mens | Pink Fluro Polkadot Socks - Mens | 566,304 | 15.1594% |
| Mens | White Striped Socks - Mens | 283,572 | 7.5909% |
| Mens | Teal Button Up Shirt - Mens | 128,742 | 3.4463% |
| Womens | Grey Fashion Jacket - Womens | 1,162,824 | 40.0305% |
| Womens | Black Straight Jeans - Womens | 635,370 | 21.8728% |
| Womens | Khaki Suit Jacket - Womens | 429,762 | 14.7947% |
| Womens | Indigo Rain Jacket - Womens | 336,600 | 11.5875% |
| Womens | Navy Oversized Jeans - Womens | 208,260 | 7.1694% |
| Womens | Cream Relaxed Jeans - Womens | 132,030 | 4.5452% |

### Analysis and query-wording mismatch

The supplied query divides each **product's** revenue by total revenue within its category. Its results answer “What percentage of category revenue does each product contribute?” That is not the same as the stated question about **segment** share within each category.

### Recommended SQL to answer the stated segment-by-category question

The following is a separate recommended correction; it was not part of the submitted query, and no output was supplied for it.

```sql
WITH segment_revenue AS (
    SELECT
        d.category_name,
        d.segment_name,
        SUM(s.qty * s.price - s.discount) AS net_revenue
    FROM balanced_tree.sales AS s
    LEFT JOIN balanced_tree.product_details AS d
        ON s.prod_id = d.product_id
    GROUP BY d.category_name, d.segment_name
)
SELECT
    category_name,
    segment_name,
    net_revenue,
    ROUND(
        100.0 * net_revenue
        / SUM(net_revenue) OVER (PARTITION BY category_name),
        2
    ) AS category_revenue_share_pct
FROM segment_revenue
ORDER BY category_name, segment_name;
```

### Business use

Product-by-category shares reveal which individual products make up each category's sales. Segment-by-category shares answer a different decision question: how much of a category's net revenue comes from each merchandise segment. It is important to group by the dimension named in the business question.

---

## Question 3.6 — What is the total transaction penetration for each product?

**Definition supplied in the question:** penetration is the number of transactions containing at least one unit of a product divided by the total number of transactions.

### Submitted SQL

```sql
WITH tb1 AS (
    SELECT
        product_name,
        COUNT(DISTINCT txn_id) AS ex,
        (SELECT COUNT(DISTINCT txn_id) FROM balanced_tree.sales AS s) AS tot_tran
    FROM balanced_tree.sales AS s
    LEFT JOIN balanced_tree.product_details AS d
        ON prod_id = product_id
    GROUP BY product_name
)
SELECT
    product_name,
    (ex / tot_tran) AS pert
FROM tb1;
```

**Supplied result, converted to percentages for readability**

| Product | Penetration |
|---|---:|
| Black Straight Jeans - Womens | 49.84% |
| Blue Polo Shirt - Mens | 50.72% |
| Cream Relaxed Jeans - Womens | 49.72% |
| Grey Fashion Jacket - Womens | 51.00% |
| Indigo Rain Jacket - Womens | 50.00% |
| Khaki Suit Jacket - Womens | 49.88% |
| Navy Oversized Jeans - Womens | 50.96% |
| Navy Solid Socks - Mens | 51.24% |
| Pink Fluro Polkadot Socks - Mens | 50.32% |
| Teal Button Up Shirt - Mens | 49.68% |
| White Striped Socks - Mens | 49.72% |
| White Tee Shirt - Mens | 50.72% |

### Analysis and business insight

Penetration is tightly grouped, ranging from 49.68% for Teal Button Up Shirt - Mens to 51.24% for Navy Solid Socks - Mens. Within this dataset, each of the 12 products appears in roughly half of all transactions. The relatively narrow spread suggests no single product dominates transaction reach by a wide margin, although penetration alone does not tell us whether products are profitable or typically purchased in high quantities.

**Business use:** Penetration can help identify products frequently included in baskets and form a starting point for cross-sell or product-bundle analysis. It should not be confused with units sold or revenue share.

---

## Question 3.7 — What is the most common combination of at least one unit of any three products in a single transaction?

### Submitted SQL

```sql
WITH products AS (
    SELECT
        txn_id,
        product_name
    FROM balanced_tree.sales AS s
    LEFT JOIN balanced_tree.product_details AS d
        ON prod_id = product_id
    GROUP BY product_name, txn_id
)
SELECT
    p1.product_name AS product_1,
    p2.product_name AS product_2,
    p3.product_name AS product_3,
    COUNT(*) AS combo_count
FROM products AS p1
JOIN products AS p2
    ON p1.txn_id = p2.txn_id
    AND p1.product_name < p2.product_name
JOIN products AS p3
    ON p1.txn_id = p3.txn_id
    AND p2.product_name < p3.product_name
GROUP BY
    p1.product_name,
    p2.product_name,
    p3.product_name
ORDER BY combo_count DESC
LIMIT 1;
```

**Supplied result**

| Product 1 | Product 2 | Product 3 | Transactions containing the combination |
|---|---|---|---:|
| Grey Fashion Jacket - Womens | Teal Button Up Shirt - Mens | White Tee Shirt - Mens | 352 |

### How the SQL works

1. The `products` CTE reduces the sales data to one row per transaction and product, so multiple units of the same product do not duplicate it in the combination analysis.
2. The query self-joins this product list three times on the same transaction ID to enumerate triples of distinct products.
3. Alphabetical comparisons (`p1.product_name < p2.product_name`, then `p2.product_name < p3.product_name`) stop the same combination from appearing in different orders.
4. `COUNT(*)` counts the transactions in which each trio appears; sorting descending and applying `LIMIT 1` returns the most common trio.

### Analysis and business insight

The most common three-product combination reported is Grey Fashion Jacket - Womens, Teal Button Up Shirt - Mens, and White Tee Shirt - Mens, appearing together in **352 transactions**. Relative to the 2,500 unique transactions reported in the supplied transaction query, this is 14.08% of transactions (352 / 2,500), assuming both query outputs cover the same complete dataset.

**Business use:** This trio could be investigated as a potential cross-sell or merchandising bundle. The count alone does not prove that a formal bundle would increase sales or profit; margin, discount cost, inventory, seasonality, and an experiment would be needed to validate such a decision.

---

# 4. Summary of Findings

Based on the results supplied with the submitted SQL (and not an independent execution of the queries):

1. **Overall scale:** 135,648 units were sold, gross sales before discounts totalled 3,868,359, and recorded discounts totalled 548,100.
2. **Transactions:** the supplied query reports 2,500 transactions; 60.20% are member transactions and 39.80% are non-member transactions.
3. **Member spending:** average member transaction revenue is about 1,331.35, versus 1,323.19 for non-member transactions, a difference of about 8.16 (approximately 0.62%). This is descriptive and is not evidence that membership causes greater spend.
4. **Leading products:** Blue Polo Shirt - Mens leads under the supplied net-revenue expression, followed by Grey Fashion Jacket - Womens and White Tee Shirt - Mens.
5. **Segment leaders:** Grey Fashion Jacket - Womens leads Jackets, Black Straight Jeans - Womens leads Jeans, Blue Polo Shirt - Mens leads Shirts, and Navy Solid Socks - Mens leads Socks.
6. **Revenue concentration:** Black Straight Jeans - Womens contributes 65.12% of Jeans segment revenue, while Grey Fashion Jacket - Womens contributes 60.28% of Jacket segment revenue under the supplied calculation.
7. **Transaction reach:** product penetration ranges from about 49.68% to 51.24%, a narrow range across the products shown.
8. **Product combinations:** the Grey Fashion Jacket + Teal Button Up Shirt + White Tee Shirt combination appears in 352 transactions.

## 5. Data and SQL Quality Notes

These issues matter if the project is used as a professional portfolio or adapted to a monthly business report.

- **Before-discount vs. after-discount revenue:** the top-three query is described as revenue before discount, but the ranking expression subtracts `discount`. Use `SUM(qty * price)` for gross revenue before discount, or describe the current expression as net revenue.
- **Revenue/discount definition:** the calculations assume the value in `discount` can be subtracted directly from `qty * price`. Confirm the source schema's definition and granularity before using the numbers in a financial report.
- **Percentiles:** `DENSE_RANK()` creates ranks over distinct revenue values. Using the maximum rank as the percentile denominator does not necessarily produce standard transaction revenue percentiles when there are ties.
- **Category share query:** the submitted query says segment share by category but groups by `product_name`. The README distinguishes its actual product-level result from the recommended segment-level correction.
- **Segment/category totals:** the supplied top-product queries filter to `rk = 1`, which returns the leading product's metrics, not the overall segment or category totals. Additional SQL is included where needed to report aggregate totals.
- **Membership flag:** `ROUND(AVG(member), 0)` assumes each sales row for a transaction has a consistent membership status. Validate this before production use.
- **Validation status:** output tables in this README are from the result sets supplied alongside the queries. The additional corrected queries have not been run here, so their numeric outputs are not represented as known findings.

## 6. SQL Skills Demonstrated

- Aggregation with `SUM`, `AVG`, `COUNT`, and `MAX`
- CTEs (`WITH`) for staged transaction-level calculations
- Joins between transaction sales and product dimensions
- Window functions: `DENSE_RANK()` and `SUM(...) OVER (PARTITION BY ...)`
- Conditional aggregation using `CASE WHEN`
- Transaction-level member/non-member comparisons
- Share-of-total and transaction penetration calculations
- Self-joins to identify frequently co-occurring products
- Query review: checking that SQL logic matches the business question



## 8. Business Takeaway

This case demonstrates how SQL can transform product transaction records into sales KPIs, customer-membership comparisons, product performance rankings, contribution analysis, and potential cross-sell insights. The strongest analyst contribution is not only writing the query: it is checking that each metric actually answers the business question, clearly communicating what the figures mean, and calling out assumptions or calculation limitations before stakeholders make decisions.
