# Case Study #1: Danny's Diner --- Customer & Loyalty Analytics

![SQL](https://img.shields.io/badge/SQL-MySQL-blue)
![Project](https://img.shields.io/badge/Project-Customer%20Analytics-purple)
![Status](https://img.shields.io/badge/Status-Completed%20Analysis-success)

## 1. Project Overview

Danny's Diner is the first case study in the [8 Week SQL
Challenge](https://8weeksqlchallenge.com/case-study-1/). This project
uses SQL to explore customer spending, menu-item popularity, and
purchasing behaviour around loyalty-programme enrolment.

The aim is not only to retrieve answers with SQL, but to translate the
results into business observations and practical recommendations.

### Business context

Danny owns a restaurant serving sushi, curry, and ramen. He has
transaction records and a loyalty programme with membership join dates.
He wants to better understand his customers and how the membership
programme relates to their purchasing behaviour.

### Business objectives

-   Measure customer spending and visit frequency.
-   Identify the most purchased menu items and each customer's most
    frequently purchased item.
-   Explore purchases before and after customers join the loyalty
    programme.
-   Calculate points under the programme's standard and introductory
    bonus rules.
-   Produce transaction-level data that can support future reporting.

### Tools and SQL skills

-   MySQL
-   `JOIN` and `LEFT JOIN`
-   Aggregations: `SUM()`, `COUNT()` and `GROUP BY`
-   Common Table Expressions (CTEs)
-   Window functions: `DENSE_RANK()` and `RANK()`
-   Conditional logic with `CASE`
-   Date filtering and `DATEDIFF()`
-   Translating query results into business insights

------------------------------------------------------------------------

## 2. Dataset and Data Model

The case uses three tables.

  -----------------------------------------------------------------------
  Table                   Relevant columns        Description
  ----------------------- ----------------------- -----------------------
  `sales`                 `customer_id`,          Each recorded product
                          `order_date`,           purchase
                          `product_id`            

  `menu`                  `product_id`,           Menu products and their
                          `product_name`, `price` prices

  `members`               `customer_id`,          Customers who joined
                          `join_date`             the loyalty programme
  -----------------------------------------------------------------------

### Relationship between tables

``` mermaid
erDiagram
    MENU ||--o{ SALES : "product_id"
    MEMBERS ||--o{ SALES : "customer_id"
    MENU {
        int product_id PK
        string product_name
        int price
    }
    SALES {
        string customer_id
        date order_date
        int product_id FK
    }
    MEMBERS {
        string customer_id PK
        date join_date
    }
```

The `sales` table connects to `menu` through `product_id`. It connects
to `members` through `customer_id`. A `LEFT JOIN` to `members` is useful
when all sales must remain in the result, including purchases by
customers who never joined.

### Important analytical assumptions

-   The supplied dataset records dates, not transaction timestamps. We
    can identify the earliest or latest *recorded date*, but cannot
    determine the order of multiple items purchased on the same date.
-   "Visit days" means distinct dates with recorded purchases, not
    necessarily distinct physical visits.
-   "Most popular" means most frequently purchased in this dataset. It
    does not automatically mean highest revenue or profit.
-   Customer purchasing patterns are descriptive. They do not prove that
    loyalty membership caused a change in behaviour.
-   The sample is small, so recommendations should be treated as
    hypotheses to test rather than broad conclusions about all
    restaurant customers.

------------------------------------------------------------------------

## 3. Executive Summary

### Key findings

-   **Customer A spent the most:** \$76, compared with \$74 for B and
    \$36 for C.
-   **Customer B visited on the most distinct dates:** six, compared
    with four for A and two for C.
-   **Ramen was the best-selling item:** eight purchases.
-   **Ramen was also the most frequently purchased item for A and C.**
    Customer B had a three-way tie between curry, ramen, and sushi.
-   **Before membership**, A purchased two items for \$25 and B
    purchased three items for \$40.
-   Under the standard points rules, **B earned 940 points**, more than
    A's 860, despite spending slightly less.
-   Applying the introductory membership bonus only to eligible
    purchases from the join date through the end of January gives
    **1,020 points for A and 320 for B**.

### Recommendations

1.  Monitor both visit frequency and spending instead of ranking
    customers on revenue alone.
2.  Track ramen demand and consider whether stock, menu placement, or a
    carefully measured promotion could support sales.
3.  Test relevant offers based on observed purchasing patterns without
    assuming that a small sample represents a stable preference.
4.  Compare pre- and post-membership activity over comparable periods to
    investigate changes in customer behaviour.
5.  Track points awarded and reward-redemption costs alongside revenue
    when evaluating loyalty-programme sustainability.

------------------------------------------------------------------------

## 4. Case Study Questions and Solutions

## Question 1 --- What is the total amount each customer spent?

### Business question

How much revenue did each customer generate across the recorded
transactions?

### SQL solution

``` sql
SELECT
    s.customer_id,
    SUM(m.price) AS total_spent
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
    ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id;
```

### How the SQL works

1.  `JOIN` matches each sale to its menu item using `product_id`, so
    each transaction receives the correct price.
2.  `SUM(m.price)` adds the prices of the customer's purchases.
3.  `GROUP BY s.customer_id` returns one aggregate row per customer.
4.  `ORDER BY` sorts the output for easier reading.

### Result

  Customer      Total spent
  ----------- -------------
  A                    \$76
  B                    \$74
  C                    \$36
  **Total**       **\$186**

### Business insight

A generated the highest recorded spend at \$76, narrowly ahead of B at
\$74. Together, A and B contributed \$150, or approximately 80.6% of the
\$186 total.

### Recommendation

Investigate ways to retain A and B, while exploring opportunities to
increase C's engagement. Revenue alone does not tell us which customers
are most profitable because product costs and customer-level costs are
not provided.

------------------------------------------------------------------------

## Question 2 --- How many days has each customer visited?

### Business question

How frequently did each customer make purchases on distinct dates?

### SQL solution

``` sql
SELECT
    customer_id,
    COUNT(DISTINCT order_date) AS visit_days
FROM dannys_diner.sales
GROUP BY customer_id
ORDER BY customer_id;
```

### How the SQL works

-   `COUNT(DISTINCT order_date)` counts each date once for a customer,
    even if they purchased multiple items on that date.
-   `GROUP BY customer_id` calculates the count separately for each
    customer.

### Result

  Customer     Distinct visit days
  ---------- ---------------------
  A                              4
  B                              6
  C                              2

### Business insight

B has the most distinct purchase dates, with six. A has four, and C has
two. Combined with Question 1, A spent more in total despite appearing
on fewer purchase dates than B.

The average spend per distinct recorded purchase date is:

  Customer     Total spent   Visit days   Spend per recorded date
  ---------- ------------- ------------ -------------------------
  A                   \$76            4                   \$19.00
  B                   \$74            6                   \$12.33
  C                   \$36            2                   \$18.00

### Recommendation

Use both frequency and spending to understand customer behaviour. B may
be worth studying for repeat-visit engagement, while A's higher average
spend per recorded date may merit further investigation. These averages
are per date, not necessarily per physical visit.

------------------------------------------------------------------------

## Question 3 --- What was the first item purchased by each customer?

### Business question

Which menu items appeared on each customer's earliest recorded purchase
date?

### SQL solution

``` sql
WITH RankedSales AS (
    SELECT
        s.customer_id,
        m.product_name,
        s.order_date,
        DENSE_RANK() OVER (
            PARTITION BY s.customer_id
            ORDER BY s.order_date ASC
        ) AS purchase_rank
    FROM dannys_diner.sales AS s
    JOIN dannys_diner.menu AS m
        ON s.product_id = m.product_id
)
SELECT DISTINCT
    customer_id,
    product_name,
    order_date
FROM RankedSales
WHERE purchase_rank = 1
ORDER BY customer_id, product_name;
```

### How the SQL works

-   The CTE, `RankedSales`, prepares a dataset with customer, product,
    date, and rank.
-   `PARTITION BY customer_id` restarts the ranking for each customer.
-   `ORDER BY order_date ASC` puts the earliest date first.
-   `DENSE_RANK()` assigns rank 1 to every item on that earliest date.
-   `DISTINCT` removes duplicate output rows without discarding
    different products purchased on the same date.

### Result

  Customer   First-date item   Date
  ---------- ----------------- ------------
  A          Curry             2021-01-01
  A          Sushi             2021-01-01
  B          Curry             2021-01-01
  C          Ramen             2021-01-01

### Business insight

A's earliest recorded date includes both curry and sushi. B's includes
curry, while C's includes ramen.

### Recommendation and limitation

Danny could investigate which products appear in first recorded
purchases. The data does not establish each customer's first-ever visit
to the restaurant, and it cannot sequence items bought on the same date
because timestamps are unavailable.

------------------------------------------------------------------------

## Question 4 --- What is the most purchased item overall?

### Business question

Which menu item has the highest number of recorded purchases?

### SQL solution

``` sql
WITH ProductSales AS (
    SELECT
        m.product_name,
        COUNT(*) AS total_purchased
    FROM dannys_diner.sales AS s
    JOIN dannys_diner.menu AS m
        ON s.product_id = m.product_id
    GROUP BY m.product_name
),
RankedProducts AS (
    SELECT
        product_name,
        total_purchased,
        DENSE_RANK() OVER (
            ORDER BY total_purchased DESC
        ) AS purchase_rank
    FROM ProductSales
)
SELECT
    product_name,
    total_purchased
FROM RankedProducts
WHERE purchase_rank = 1;
```

### How the SQL works

1.  Group transactions by product and count purchases.
2.  Rank products by purchase count in descending order.
3.  Keep rank 1, preserving any ties for the highest count.

### Result

  Product     Total purchases
  --------- -----------------
  Ramen                     8

### Business insight

Ramen is the most purchased menu item in the dataset, with eight
purchases.

### Recommendation

Danny could monitor ramen demand and investigate whether stock planning
or menu placement could support sales. Purchase count is not the same as
revenue or profit, so price and ingredient cost would also matter when
making commercial decisions.

------------------------------------------------------------------------

## Question 5 --- Which item was most popular for each customer?

### Business question

Which menu item or items did each customer purchase most frequently?

### SQL solution

``` sql
WITH CustomerProductCounts AS (
    SELECT
        s.customer_id,
        m.product_name,
        COUNT(*) AS total_purchased
    FROM dannys_diner.sales AS s
    JOIN dannys_diner.menu AS m
        ON s.product_id = m.product_id
    GROUP BY s.customer_id, m.product_name
),
RankedPreferences AS (
    SELECT
        customer_id,
        product_name,
        total_purchased,
        DENSE_RANK() OVER (
            PARTITION BY customer_id
            ORDER BY total_purchased DESC
        ) AS preference_rank
    FROM CustomerProductCounts
)
SELECT
    customer_id,
    product_name,
    total_purchased
FROM RankedPreferences
WHERE preference_rank = 1
ORDER BY customer_id, product_name;
```

### How the SQL works

-   The first CTE counts purchases for each customer-product
    combination.
-   The window function ranks products within each customer, rather than
    across all customers.
-   `DENSE_RANK()` retains all products tied for first place.

### Result

  Customer   Most purchased item(s)     Count
  ---------- ------------------------ -------
  A          Ramen                          3
  B          Curry                          2
  B          Ramen                          2
  B          Sushi                          2
  C          Ramen                          3

### Business insight

Ramen is the most frequently purchased item for A and C. B has no single
leading product: all three menu items tie at two purchases each.

### Recommendation

Danny could test offers based on observed purchase patterns. B's tie
suggests that a varied offer may be more appropriate than assuming one
dominant preference. More data would be needed to determine whether
these patterns persist over time.

------------------------------------------------------------------------

## Question 6 --- Which item was purchased first after each customer became a member?

### Business question

What was each member's first recorded purchase after their membership
join date?

### SQL solution

``` sql
WITH MemberPurchases AS (
    SELECT
        s.customer_id,
        s.order_date,
        m.product_name,
        DENSE_RANK() OVER (
            PARTITION BY s.customer_id
            ORDER BY s.order_date ASC
        ) AS purchase_rank
    FROM dannys_diner.sales AS s
    JOIN dannys_diner.members AS mb
        ON s.customer_id = mb.customer_id
    JOIN dannys_diner.menu AS m
        ON s.product_id = m.product_id
    WHERE s.order_date >= mb.join_date
)
SELECT DISTINCT
    customer_id,
    product_name
FROM MemberPurchases
WHERE purchase_rank = 1
ORDER BY customer_id, product_name;
```

### How the SQL works

-   Joining `members` restricts the analysis to customers with
    membership records.
-   `s.order_date >= mb.join_date` includes purchases made on the join
    date as well as later dates.
-   `DENSE_RANK()` identifies the earliest qualifying date for each
    member.
-   `DISTINCT` avoids repeating the same product in the output.

### Result

  Customer   First qualifying post-joining item
  ---------- ------------------------------------
  A          Ramen
  B          Sushi

### Business insight

A's first qualifying post-joining purchase is ramen, while B's is sushi.

### Recommendation and correction

This query uses an inclusive join-date rule. The original uploaded SQL
used `mb.join_date < s.order_date`, which excludes purchases made on the
join date. Use `>=` when following the usual inclusive interpretation of
the case.

The result describes what members purchased; it does not prove that
membership caused them to select a particular product.

------------------------------------------------------------------------

## Question 7 --- Which item was purchased just before each customer became a member?

### Business question

What did each customer purchase on their last recorded purchase date
before joining the programme?

### SQL solution

``` sql
WITH PreMembershipPurchases AS (
    SELECT
        s.customer_id,
        s.order_date,
        m.product_name,
        DENSE_RANK() OVER (
            PARTITION BY s.customer_id
            ORDER BY s.order_date DESC
        ) AS purchase_rank
    FROM dannys_diner.sales AS s
    JOIN dannys_diner.members AS mb
        ON s.customer_id = mb.customer_id
    JOIN dannys_diner.menu AS m
        ON s.product_id = m.product_id
    WHERE s.order_date < mb.join_date
)
SELECT DISTINCT
    customer_id,
    product_name
FROM PreMembershipPurchases
WHERE purchase_rank = 1
ORDER BY customer_id, product_name;
```

### How the SQL works

The query filters to dates strictly before enrolment, then sorts dates
in descending order. The most recent qualifying date receives rank 1. If
multiple items were bought on that date, each can be returned.

### Result

  Customer   Last pre-membership item(s)
  ---------- -----------------------------
  A          Curry
  A          Sushi
  B          Sushi

### Business insight

A purchased curry and sushi on the last recorded date before joining. B
purchased sushi on that date.

### Recommendation and limitation

This result can provide a baseline for subsequent purchasing analysis.
It cannot tell us what motivated either customer to enrol, and it does
not prove that any particular product influenced the decision.

------------------------------------------------------------------------

## Question 8 --- How many items and how much money did members spend before joining?

### Business question

How much had each future member spent before their loyalty membership
began?

### SQL solution

``` sql
SELECT
    s.customer_id,
    COUNT(*) AS total_items,
    SUM(m.price) AS total_spent
FROM dannys_diner.sales AS s
JOIN dannys_diner.members AS mb
    ON s.customer_id = mb.customer_id
   AND s.order_date < mb.join_date
JOIN dannys_diner.menu AS m
    ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id;
```

### How the SQL works

-   The join condition selects only purchases made before the join date.
-   `COUNT(*)` counts recorded purchase rows, which represent purchased
    items in this dataset.
-   `SUM(m.price)` calculates the total value of those purchases.
-   `GROUP BY` produces one row per qualifying customer.

### Result

  Customer     Items purchased   Total spent
  ---------- ----------------- -------------
  A                          2          \$25
  B                          3          \$40

### Business insight

Before enrolment, B had purchased three items for \$40, compared with
A's two items for \$25. The customers had different spending histories
before joining.

### Recommendation

Compare pre- and post-membership activity over comparable time periods.
A fair assessment should account for differences in observation windows
and visit frequency; the figures above alone do not measure programme
impact.

------------------------------------------------------------------------

## Question 9 --- How many loyalty points would each customer earn under the standard rules?

### Business question

If every \$1 spent earns 10 points and sushi earns double points, what
is each customer's points total?

### SQL solution

``` sql
SELECT
    s.customer_id,
    SUM(m.price) AS total_spent,
    SUM(
        m.price *
        CASE
            WHEN m.product_name = 'sushi' THEN 20
            ELSE 10
        END
    ) AS total_points
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
    ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id;
```

### How the SQL works

-   `CASE` assigns 20 points per dollar to sushi and 10 points per
    dollar to other items.
-   Multiplying the price by the applicable rate calculates points per
    purchased item.
-   `SUM()` adds those points for each customer.

### Result

  Customer     Total spent   Total points
  ---------- ------------- --------------
  A                   \$76            860
  B                   \$74            940
  C                   \$36            360

### Business insight

B spent \$2 less than A but earned 80 more points. The difference
reflects the product mix and the sushi double-points rule. C had both
the lowest recorded spending and the lowest points total.

### Recommendation

Track points alongside spending, redemptions, and programme costs. More
points awarded do not automatically mean higher profitability.

### Alternative implementation

The original SQL file creates a `point_dict` table with a multiplier per
`product_id`. That is a valid way to store reward rules separately from
the query. The `CASE` version above is simpler for this small case; a
separate rule table can be preferable when the programme becomes more
complex.

------------------------------------------------------------------------

## Question 10 --- How many points did members A and B earn by the end of January under the introductory bonus?

### Business question

During the first seven days of membership, including the join date, all
products earn double points. Sushi also earns double points under the
standard rule. Calculate eligible points through 31 January 2021.

### SQL solution

``` sql
SELECT
    s.customer_id,
    SUM(
        m.price *
        CASE
            WHEN m.product_name = 'sushi' THEN 20
            WHEN s.order_date BETWEEN mb.join_date
                 AND DATE_ADD(mb.join_date, INTERVAL 6 DAY)
                THEN 20
            ELSE 10
        END
    ) AS total_points
FROM dannys_diner.sales AS s
JOIN dannys_diner.members AS mb
    ON s.customer_id = mb.customer_id
   AND s.order_date >= mb.join_date
   AND s.order_date <= '2021-01-31'
JOIN dannys_diner.menu AS m
    ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id;
```

### How the SQL works

1.  The join to `members` identifies the join date for each customer.
2.  `s.order_date >= mb.join_date` excludes purchases before enrolment.
3.  `s.order_date <= '2021-01-31'` restricts the analysis to January.
4.  `DATE_ADD(mb.join_date, INTERVAL 6 DAY)` calculates the end of the
    seven-day introductory period.
5.  `CASE` applies 20 points per dollar to sushi or to any product
    bought within the introductory period, and 10 points per dollar
    otherwise.
6.  `SUM()` aggregates the points by customer.

### Result

  Customer     Total points through 31 January
  ---------- ---------------------------------
  A                                      1,020
  B                                        320

### Correction to the uploaded query

The uploaded query filters transactions to 31 January but does not
exclude transactions before the membership join date. This means it
awards membership-related points to some pre-membership purchases.

The corrected query filters purchases to the membership period before
calculating points. Under the stated rules, A earns 1,020 points and B
earns 320 points.

### Business insight

The introductory rule changes the points awarded for eligible purchases.
This calculation helps Danny estimate reward accrual, but it does not
establish whether the promotion caused customers to spend more or visit
more frequently.

### Recommendation

Compare subsequent engagement and spending with a suitable baseline, and
monitor reward-redemption costs to evaluate the promotion's business
value.

------------------------------------------------------------------------

## 5. Bonus Question 1 --- Join All the Things

### Business question

Create a transaction-level report with customer, order date, product,
price, and whether the purchase occurred during the customer's
membership period.

### SQL solution

``` sql
SELECT
    s.customer_id,
    s.order_date,
    m.product_name,
    m.price,
    CASE
        WHEN mb.join_date IS NOT NULL
         AND s.order_date >= mb.join_date
        THEN 'Y'
        ELSE 'N'
    END AS member_status
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
    ON s.product_id = m.product_id
LEFT JOIN dannys_diner.members AS mb
    ON s.customer_id = mb.customer_id
ORDER BY
    s.customer_id,
    s.order_date,
    m.product_name;
```

### How the SQL works

-   `LEFT JOIN` keeps transactions for customers without a membership
    record, including C's purchases.
-   `CASE` labels a transaction `Y` only when the customer has a join
    date and the transaction date is on or after it.
-   The final `ORDER BY` makes the output easier to inspect.

### Business value

This creates a reusable transaction-level dataset for reporting sales by
product, customer, and membership status.

### Correction to the uploaded query

The original query uses date subtraction to calculate `days_after_join`.
Direct date comparison is clearer for a simple membership flag and
avoids relying on arithmetic with date values.

------------------------------------------------------------------------

## 6. Bonus Question 2 --- Rank All the Things

### Business question

Rank each customer's transactions during their membership period, while
leaving non-member purchases unranked.

### SQL solution

``` sql
WITH SalesMenu AS (
    SELECT
        s.customer_id,
        s.order_date,
        m.product_name,
        m.price,
        mb.join_date
    FROM dannys_diner.sales AS s
    JOIN dannys_diner.menu AS m
        ON s.product_id = m.product_id
    LEFT JOIN dannys_diner.members AS mb
        ON s.customer_id = mb.customer_id
),
MembershipLabels AS (
    SELECT
        customer_id,
        order_date,
        product_name,
        price,
        CASE
            WHEN join_date IS NOT NULL
             AND order_date >= join_date
            THEN 'Y'
            ELSE 'N'
        END AS member_status
    FROM SalesMenu
)
SELECT
    customer_id,
    order_date,
    product_name,
    price,
    member_status,
    CASE
        WHEN member_status = 'Y' THEN
            RANK() OVER (
                PARTITION BY customer_id, member_status
                ORDER BY order_date
            )
        ELSE NULL
    END AS purchase_rank
FROM MembershipLabels
ORDER BY
    customer_id,
    order_date,
    product_name;
```

### How the SQL works

-   The first CTE joins transaction, menu, and membership data.
-   The second CTE labels each transaction according to membership
    status on its date.
-   `RANK()` ranks member purchases within each customer.
-   Non-member transactions receive `NULL` for the ranking.

### Business value

This dataset can support chronological analysis of member transactions
and the creation of member activity reports.

### Important limitation

`RANK()` assigns the same rank to transactions on the same date and may
leave gaps in subsequent ranks. That is appropriate when the date is the
only ordering field available. If every item needs a unique sequential
rank, a transaction ID or timestamp is required; inventing an order for
same-day purchases would be misleading.

------------------------------------------------------------------------

## 7. Quality Checks and Corrections

The following points should be checked before treating this analysis as
final:

  -----------------------------------------------------------------------
  Area                    Review point            Treatment in this
                                                  project
  ----------------------- ----------------------- -----------------------
  Membership date         Does a purchase on the  Questions 6 and 10 use
                          join date qualify?      an inclusive join-date
                                                  rule where applicable.
                                                  Question 7 strictly
                                                  excludes the join date.

  Introductory points     Are purchases before    No. Question 10 filters
                          enrolment included?     to transactions on or
                                                  after joining.

  Date arithmetic         Is date subtraction     Use direct date
                          needed?                 comparisons for
                                                  membership status and
                                                  `DATEDIFF()` when a
                                                  numeric day difference
                                                  is required.

  Ties                    Can more than one item  Yes. `DENSE_RANK()`
                          be first or most        preserves tied results.
                          popular?                

  Visit frequency         Does one date equal one Not necessarily; this
                          physical visit?         project reports
                                                  distinct purchase
                                                  dates.

  Profitability           Does higher spending    Not necessarily; costs
                          mean higher profit?     are not included in the
                                                  dataset.

  Causation               Does membership cause   The dataset alone
                          higher spending?        cannot establish
                                                  causation.
  -----------------------------------------------------------------------

## 8. Conclusion

This case study demonstrates how SQL can turn transactional records into
useful business analysis. Aggregations quantify customer value and
purchase frequency, joins combine transactions with product and
membership information, and window functions identify first purchases
and tied product preferences.

The main business lesson is to combine metrics rather than relying on
one number. Customer A generated the most spending, Customer B visited
on the most distinct dates and earned the most points under the standard
rules, and ramen led product purchase frequency. Each finding answers a
different business question.

The analysis also illustrates the importance of validating business
rules. In particular, loyalty points must be calculated only for
purchases eligible under the membership rules. A technically valid query
can still produce misleading results if its date filters do not match
the business definition.

### Potential next steps

-   Extend the analysis to a larger transaction dataset.
-   Compare customer spending and visit frequency before and after
    enrolment over comparable periods.
-   Add product cost data to analyse gross margin rather than purchase
    count alone.
-   Build a dashboard showing customer value, menu popularity, and
    loyalty activity.
-   Evaluate promotions using a defined comparison method rather than
    relying on descriptive results alone.

------------------------------------------------------------------------

## 9. Project Structure

``` text
case-study-01-dannys-diner/
├── README.md
├── sql/
│   ├── 01_customer_spending.sql
│   ├── 02_visit_frequency.sql
│   ├── 03_first_purchase.sql
│   ├── 04_most_popular_item.sql
│   ├── 05_customer_favourites.sql
│   ├── 06_first_purchase_after_joining.sql
│   ├── 07_last_purchase_before_joining.sql
│   ├── 08_pre_membership_spending.sql
│   ├── 09_loyalty_points.sql
│   ├── 10_january_points.sql
│   └── bonus_questions.sql
└── images/
    └── query-results/
```

*Note: the structure above is the recommended repository layout. Add
screenshots of actual MySQL results only after running and verifying the
queries in your own environment.*
