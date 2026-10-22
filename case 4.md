# 8 Week SQL Challenge — Case Study #4: Data Bank

> **Portfolio project | MySQL 8.0+ | SQL analysis and business recommendations**

## Project Overview

Data Bank is a digital banking business that stores customer data across a network of nodes. This case study explores how customers are allocated to nodes, how they transact with their accounts, and how different balance-based rules could be used to allocate data.

The analysis uses SQL to cleanly aggregate transactions, calculate customer balances over time, measure node reallocation intervals, and model alternative data-allocation approaches.

## Business Questions

This README covers all questions and challenge requirements included in the case study:

- **A. Customer Nodes Exploration** — 5 questions
- **B. Customer Transactions** — 5 questions
- **C. Data Allocation Challenge** — 3 allocation options plus the required running-balance, monthly-balance, customer balance summary, and monthly provisioning analysis

## Dataset

The case uses two tables:

### `customer_nodes`

| Column | Meaning |
|---|---|
| `customer_id` | Unique customer identifier |
| `region_id` | Region to which the node belongs |
| `node_id` | Node assigned to the customer |
| `start_date` | Start date of the node allocation |
| `end_date` | End date of the allocation; `9999-12-31` represents the current allocation |

### `customer_transactions`

| Column | Meaning |
|---|---|
| `customer_id` | Customer who made the transaction |
| `txn_date` | Transaction date |
| `txn_type` | `deposit`, `purchase`, or `withdrawal` |
| `txn_amount` | Transaction amount |

## Important assumptions and SQL notes

1. **Balance sign convention:** deposits increase the balance; purchases and withdrawals decrease it.
2. **Current node allocation:** `end_date = '9999-12-31'` is treated as an active allocation and excluded from completed reallocation-duration calculations.
3. **Monthly balances:** the closing balance is the cumulative sum of all signed transactions through the end of each month. A customer’s first observed month is treated as starting from zero unless an opening balance is provided.
4. **No opening balances are provided.** These are calculated balances from the transaction history, not necessarily the customers’ actual bank balances.
5. **Allocation options are hypotheses, not necessarily the business’s final policy.** A production policy would need explicit rules for negative balances, customers without prior-month data, and the units used for “data allowance.”
6. SQL below is written for **MySQL 8.0+**. Percentile calculations use window functions. Depending on your exact MySQL version, check support for `PERCENT_RANK()` and `CUME_DIST()`.

---

# A. Customer Nodes Exploration

## A1. How many unique nodes are there on the Data Bank system?

```sql
SELECT COUNT(DISTINCT node_id) AS unique_nodes
FROM customer_nodes;
```

**Reported result:** 5 unique nodes.

**Interpretation:** The system uses five distinct nodes. This describes the network’s node count, but does not tell us whether the nodes have equal capacity, transaction load, or operating cost.

**Recommendation:** Track transaction volume, customer count, latency, and capacity by node before deciding whether node resources need to be rebalanced.

## A2. What is the number of nodes per region?

```sql
SELECT
    region_id,
    COUNT(DISTINCT node_id) AS unique_nodes
FROM customer_nodes
GROUP BY region_id
ORDER BY region_id;
```

**Reported result:** Each of the five regions has 5 distinct node IDs.

**Interpretation:** Every region has records associated with all five node IDs in the provided data. This does not necessarily mean every node is physically located in every region; it reflects the recorded customer-to-node allocations.

**Recommendation:** Confirm the meaning of `region_id` and `node_id` with the data model before drawing infrastructure conclusions. If these represent available nodes per region, review whether the distribution meets each region’s demand.

## A3. How many customers are allocated to each region?

```sql
SELECT
    region_id,
    COUNT(DISTINCT customer_id) AS customers
FROM customer_nodes
GROUP BY region_id
ORDER BY region_id;
```

**Reported result:**

| Region | Customers |
|---:|---:|
| 1 | 110 |
| 2 | 105 |
| 3 | 102 |
| 4 | 95 |
| 5 | 88 |

**Interpretation:** Region 1 has the most distinct customers (110), while Region 5 has the fewest (88). The spread is 22 customers.

**Recommendation:** Use customer distribution as an initial workload indicator, but not as a direct measure of capacity needs. Transaction frequency, transaction size, and service-level requirements may vary by customer.

## A4. How many days on average are customers reallocated to a different node?

```sql
SELECT
    ROUND(AVG(DATEDIFF(end_date, start_date)), 2)
        AS average_reallocated_days
FROM customer_nodes
WHERE end_date <> '9999-12-31';
```

**Reported result:** Approximately 14.63 days.

**Interpretation:** Completed allocation records last around two weeks on average. This is the average duration between the recorded allocation start and end dates; it does not, by itself, prove that every end date represents a move to a different node.

**Recommendation:** Monitor the frequency and reason for reallocations. If reallocation is operationally expensive, investigate whether customer placement can be made more stable without compromising performance or resilience.

## A5. What are the median, 80th and 95th percentiles for reallocation days for each region?

The following MySQL 8.0+ query calculates the percentile values using `CUME_DIST()`. It selects the first duration at or above each requested percentile threshold.

```sql
WITH reallocation_days AS (
    SELECT
        region_id,
        DATEDIFF(end_date, start_date) AS reallocated_days
    FROM customer_nodes
    WHERE end_date <> '9999-12-31'
),
ranked AS (
    SELECT
        region_id,
        reallocated_days,
        CUME_DIST() OVER (
            PARTITION BY region_id
            ORDER BY reallocated_days
        ) AS cumulative_distribution
    FROM reallocation_days
)
SELECT
    region_id,
    MIN(CASE WHEN cumulative_distribution >= 0.50
             THEN reallocated_days END) AS median_days,
    MIN(CASE WHEN cumulative_distribution >= 0.80
             THEN reallocated_days END) AS p80_days,
    MIN(CASE WHEN cumulative_distribution >= 0.95
             THEN reallocated_days END) AS p95_days
FROM ranked
GROUP BY region_id
ORDER BY region_id;
```

**Result note:** The uploaded source file contained the question but did not contain a completed query or percentile results. Run the query against your database to populate the values.

**Interpretation guide:**
- **Median:** the typical middle reallocation duration.
- **80th percentile:** 80% of recorded durations are at or below this threshold.
- **95th percentile:** a useful indicator of unusually long allocation periods.

**Recommendation:** Compare regions using both median and high-percentile durations. A high 95th percentile may indicate a subset of customers with unusually long allocation periods or inconsistent reallocation practices. Check sample sizes before comparing regions.

---

# B. Customer Transactions

## B1. What is the unique count and total amount for each transaction type?

The source query calls `COUNT(customer_id)` a unique count, but that counts transaction rows, not unique customers. The query below returns both metrics so the distinction is clear.

```sql
SELECT
    txn_type,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(txn_amount) AS total_amount
FROM customer_transactions
GROUP BY txn_type
ORDER BY txn_type;
```

**Reported transaction-row counts and totals from the source results:**

| Transaction type | Transaction rows | Total amount |
|---|---:|---:|
| Deposit | 2,671 | 1,359,168 |
| Withdrawal | 1,580 | 793,003 |
| Purchase | 1,617 | 806,537 |

**Interpretation:** Deposits have the largest transaction count and total amount. Purchases and withdrawals have similar total amounts in this sample.

**Recommendation:** Monitor deposit inflows alongside purchase and withdrawal outflows to understand customer money movement. These totals describe transaction activity; they do not establish revenue or profit for Data Bank.

## B2. What are the average total historical deposit counts and amounts for all customers?

```sql
WITH customer_deposits AS (
    SELECT
        customer_id,
        COUNT(*) AS deposit_count,
        SUM(txn_amount) AS total_deposit_amount
    FROM customer_transactions
    WHERE txn_type = 'deposit'
    GROUP BY customer_id
)
SELECT
    ROUND(AVG(deposit_count), 4) AS average_deposit_count,
    ROUND(AVG(total_deposit_amount), 4) AS average_total_deposit_amount
FROM customer_deposits;
```

**Reported result:**

| Metric | Average |
|---|---:|
| Historical deposit count per customer | 5.3420 |
| Historical deposit amount per customer | 2,718.3360 |

**Interpretation:** Among customers who have at least one deposit record, the average customer has approximately 5.34 deposits and a total historical deposit amount of about 2,718.34.

**Important caveat:** The query groups customers with deposit activity. Customers with zero deposits are excluded from these averages. If the business wants the average across every customer, including zero-deposit customers, start with a complete customer list and `LEFT JOIN` deposits.

**Recommendation:** Track both deposit frequency and amount. Frequent small deposits may represent a different customer pattern from infrequent large deposits.

## B3. For each month, how many customers make more than one deposit and at least one purchase or withdrawal in the same month?

```sql
WITH customer_monthly_activity AS (
    SELECT
        customer_id,
        DATE_FORMAT(txn_date, '%Y-%m') AS yr_month,
        SUM(txn_type = 'deposit') AS deposit_count,
        SUM(txn_type = 'purchase') AS purchase_count,
        SUM(txn_type = 'withdrawal') AS withdrawal_count
    FROM customer_transactions
    GROUP BY
        customer_id,
        DATE_FORMAT(txn_date, '%Y-%m')
)
SELECT
    yr_month,
    COUNT(DISTINCT customer_id) AS qualifying_customers
FROM customer_monthly_activity
WHERE deposit_count > 1
  AND (purchase_count >= 1 OR withdrawal_count >= 1)
GROUP BY yr_month
ORDER BY yr_month;
```

**Reported result:**

| Month | Customers meeting the condition |
|---|---:|
| 2020-01 | 168 |
| 2020-02 | 181 |
| 2020-03 | 192 |
| 2020-04 | 70 |

**Interpretation:** March has the largest number of customers who deposited more than once and also made a purchase or withdrawal. The lower count in April may reflect a shorter observation period, a change in activity, or incomplete coverage; the data alone cannot determine the cause.

**Recommendation:** Monitor this pattern over additional months and compare it with retention, active-customer rates, and net balance changes. Do not assume that more transactions automatically mean greater profitability.

## B4. What is the closing balance for each customer at the end of each month?

This version first calculates each customer’s net monthly movement, then adds the movements cumulatively to calculate month-end closing balance.

```sql
WITH signed_transactions AS (
    SELECT
        customer_id,
        DATE_FORMAT(txn_date, '%Y-%m') AS yr_month,
        CASE
            WHEN txn_type = 'deposit' THEN txn_amount
            WHEN txn_type IN ('purchase', 'withdrawal') THEN -txn_amount
        END AS signed_amount
    FROM customer_transactions
),
monthly_movement AS (
    SELECT
        customer_id,
        yr_month,
        SUM(signed_amount) AS net_monthly_movement
    FROM signed_transactions
    GROUP BY customer_id, yr_month
)
SELECT
    customer_id,
    yr_month,
    net_monthly_movement,
    SUM(net_monthly_movement) OVER (
        PARTITION BY customer_id
        ORDER BY yr_month
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS closing_balance
FROM monthly_movement
ORDER BY customer_id, yr_month;
```

**Interpretation:** The monthly movement shows the net effect of deposits, purchases, and withdrawals during that month. The closing balance is the cumulative net movement through the end of the month.

**Recommendation:** Identify customers with persistently declining balances or repeated negative balances and examine whether this reflects ordinary spending patterns, irregular deposits, or a need for different account products. Do not infer financial distress from this metric alone.

## B5. What percentage of customers increase their closing balance by more than 5%?

This query compares each customer’s month-end closing balance with their previous observed month. It avoids dividing by zero or a negative prior balance, where percentage growth is not meaningful in the usual sense.

```sql
WITH signed_transactions AS (
    SELECT
        customer_id,
        DATE_FORMAT(txn_date, '%Y-%m') AS yr_month,
        CASE
            WHEN txn_type = 'deposit' THEN txn_amount
            WHEN txn_type IN ('purchase', 'withdrawal') THEN -txn_amount
        END AS signed_amount
    FROM customer_transactions
),
monthly_movement AS (
    SELECT
        customer_id,
        yr_month,
        SUM(signed_amount) AS net_monthly_movement
    FROM signed_transactions
    GROUP BY customer_id, yr_month
),
monthly_closing AS (
    SELECT
        customer_id,
        yr_month,
        SUM(net_monthly_movement) OVER (
            PARTITION BY customer_id
            ORDER BY yr_month
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS closing_balance
    FROM monthly_movement
),
balance_changes AS (
    SELECT
        customer_id,
        yr_month,
        closing_balance,
        LAG(closing_balance) OVER (
            PARTITION BY customer_id
            ORDER BY yr_month
        ) AS previous_closing_balance
    FROM monthly_closing
),
customer_growth AS (
    SELECT
        customer_id,
        MAX(
            CASE
                WHEN previous_closing_balance > 0
                 AND closing_balance > previous_closing_balance * 1.05
                THEN 1 ELSE 0
            END
        ) AS ever_grew_over_5_percent
    FROM balance_changes
    GROUP BY customer_id
)
SELECT
    ROUND(
        100.0 * SUM(ever_grew_over_5_percent) / COUNT(*),
        2
    ) AS pct_customers_ever_grew_over_5_percent
FROM customer_growth;
```

**Reported result from the source query:** 75.8%.

**Important interpretation difference:** The uploaded query counts customer-month records with a greater-than-5% change and divides by the number of distinct customers. That can exceed 100% if a customer qualifies in multiple months. The query above instead answers the customer-level interpretation: the percentage of customers who experienced at least one month with growth greater than 5%. Therefore, its result may differ from the source’s 75.8%; rerun it and report the newly calculated result.

**Recommendation:** Define the metric precisely before reporting it. Useful alternatives include (1) percentage of customers who ever grew by more than 5%, (2) percentage of customer-months with growth above 5%, and (3) average monthly percentage change.

---

# C. Data Allocation Challenge

The business wants to estimate how much data allowance might be allocated to customers under three approaches. The transaction table contains money movements, so the following queries use calculated balances as a **proxy** for allocation. The case does not provide an actual conversion rate between account balance and data units; treat the output as a modelling exercise, not a production-ready policy.

## C1. Option 1 — Allocate data based on the amount of money at the end of the previous month

The original source query uses an average of signed transactions in each month, which is not a month-end balance. The query below uses the correct cumulative closing balance and then applies the previous month’s closing balance as the next month’s allowance.

```sql
WITH signed_transactions AS (
    SELECT
        customer_id,
        DATE_FORMAT(txn_date, '%Y-%m') AS yr_month,
        CASE
            WHEN txn_type = 'deposit' THEN txn_amount
            WHEN txn_type IN ('purchase', 'withdrawal') THEN -txn_amount
        END AS signed_amount
    FROM customer_transactions
),
monthly_movement AS (
    SELECT
        customer_id,
        yr_month,
        SUM(signed_amount) AS net_monthly_movement
    FROM signed_transactions
    GROUP BY customer_id, yr_month
),
monthly_closing AS (
    SELECT
        customer_id,
        yr_month,
        SUM(net_monthly_movement) OVER (
            PARTITION BY customer_id
            ORDER BY yr_month
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS closing_balance
    FROM monthly_movement
),
allocation AS (
    SELECT
        customer_id,
        yr_month,
        closing_balance,
        LAG(closing_balance) OVER (
            PARTITION BY customer_id
            ORDER BY yr_month
        ) AS previous_month_closing_balance
    FROM monthly_closing
)
SELECT
    customer_id,
    yr_month,
    previous_month_closing_balance AS data_allowance_proxy
FROM allocation
WHERE previous_month_closing_balance IS NOT NULL
ORDER BY customer_id, yr_month;
```

**Interpretation:** This option is simple and predictable because the next month’s allowance is based on the previous month-end balance. However, a single balance snapshot may not represent the customer’s usual funds or activity.

**Recommendation:** Consider a minimum and maximum allowance, and define what happens when the previous month’s closing balance is negative or missing. Test how volatile allowances would be from month to month.

## C2. Option 2 — Allocate data based on the average amount of money held in the account in the previous 30 days

A true rolling 30-day average requires a daily balance series. The query below creates daily signed transaction movement, calculates the cumulative balance on transaction dates, and averages the observed daily closing balance over the preceding 30 calendar days. Because the source only has transaction dates, days with no transaction are filled forward using the most recent known balance.

```sql
WITH RECURSIVE
date_bounds AS (
    SELECT
        MIN(txn_date) AS min_date,
        MAX(txn_date) AS max_date
    FROM customer_transactions
),
calendar AS (
    SELECT min_date AS calendar_date
    FROM date_bounds
    UNION ALL
    SELECT DATE_ADD(calendar_date, INTERVAL 1 DAY)
    FROM calendar
    JOIN date_bounds
      ON calendar.calendar_date < date_bounds.max_date
),
customers AS (
    SELECT DISTINCT customer_id
    FROM customer_transactions
),
signed_transactions AS (
    SELECT
        customer_id,
        txn_date,
        CASE
            WHEN txn_type = 'deposit' THEN txn_amount
            WHEN txn_type IN ('purchase', 'withdrawal') THEN -txn_amount
        END AS signed_amount
    FROM customer_transactions
),
daily_movement AS (
    SELECT
        customer_id,
        txn_date,
        SUM(signed_amount) AS net_daily_movement
    FROM signed_transactions
    GROUP BY customer_id, txn_date
),
customer_calendar AS (
    SELECT
        c.customer_id,
        cal.calendar_date
    FROM customers c
    CROSS JOIN calendar cal
),
daily_balances AS (
    SELECT
        cc.customer_id,
        cc.calendar_date,
        SUM(COALESCE(dm.net_daily_movement, 0)) OVER (
            PARTITION BY cc.customer_id
            ORDER BY cc.calendar_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS daily_closing_balance
    FROM customer_calendar cc
    LEFT JOIN daily_movement dm
      ON dm.customer_id = cc.customer_id
     AND dm.txn_date = cc.calendar_date
),
rolling_average AS (
    SELECT
        customer_id,
        calendar_date,
        AVG(daily_closing_balance) OVER (
            PARTITION BY customer_id
            ORDER BY calendar_date
            ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
        ) AS average_balance_previous_30_days
    FROM daily_balances
)
SELECT
    customer_id,
    calendar_date,
    average_balance_previous_30_days AS data_allowance_proxy
FROM rolling_average
ORDER BY customer_id, calendar_date;
```

**MySQL note:** This query uses a recursive date calendar and can generate a large customer-by-day intermediate table. Run it against the sample dataset first. For larger datasets, a permanent calendar table and indexed daily balance table would be more efficient.

**Interpretation:** A rolling average smooths out short-term balance fluctuations and is less sensitive to a single transaction than Option 1. However, it requires more processing and depends on how days without transactions are handled.

**Recommendation:** Compare the stability, computational cost, and fairness of the rolling average with Option 1. Explicitly decide whether new customers use only available history or receive a default allowance until 30 days of history exist.

## C3. Option 3 — Update data allowance in real time

This option recalculates a customer’s balance after every transaction. It requires ordering transactions chronologically. Since the provided data has transaction dates but no transaction timestamps or transaction IDs, transactions occurring on the same date do not have a guaranteed within-day order.

```sql
WITH signed_transactions AS (
    SELECT
        customer_id,
        txn_date,
        txn_type,
        txn_amount,
        CASE
            WHEN txn_type = 'deposit' THEN txn_amount
            WHEN txn_type IN ('purchase', 'withdrawal') THEN -txn_amount
        END AS signed_amount
    FROM customer_transactions
),
running_balance AS (
    SELECT
        customer_id,
        txn_date,
        txn_type,
        txn_amount,
        signed_amount,
        SUM(signed_amount) OVER (
            PARTITION BY customer_id
            ORDER BY txn_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_balance
    FROM signed_transactions
)
SELECT
    customer_id,
    txn_date,
    txn_type,
    txn_amount,
    signed_amount,
    running_balance,
    GREATEST(running_balance, 0) AS nonnegative_allowance_proxy
FROM running_balance
ORDER BY customer_id, txn_date;
```

**Important limitation:** If a customer has multiple transactions on the same date, the exact running balance after each individual transaction may be nondeterministic. For production use, add a timestamp or unique transaction ID and use it in the window `ORDER BY`.

**Interpretation:** Real-time updating responds quickly to customer activity but can make allowances more volatile. It may also require more frequent recalculation and clear controls to prevent negative balances from producing negative data allowances.

**Recommendation:** Consider a real-time system only if the customer experience benefits justify the additional implementation complexity. Use deterministic transaction ordering, minimum allowance rules, audit logs, and monitoring for abrupt changes.

## C4. Required challenge outputs: running balance, month-end balance, and minimum/average/maximum running balance per customer

The case asks for:
- A running customer balance reflecting every transaction
- The customer’s closing balance at the end of each month
- Minimum, average, and maximum running balances for each customer
- A monthly estimate of data required under each allocation option

The following query calculates the first three outputs together. As noted above, same-day transaction ordering cannot be guaranteed without a timestamp or transaction ID.

```sql
WITH signed_transactions AS (
    SELECT
        customer_id,
        txn_date,
        txn_type,
        txn_amount,
        CASE
            WHEN txn_type = 'deposit' THEN txn_amount
            WHEN txn_type IN ('purchase', 'withdrawal') THEN -txn_amount
        END AS signed_amount
    FROM customer_transactions
),
running_balances AS (
    SELECT
        customer_id,
        txn_date,
        txn_type,
        txn_amount,
        signed_amount,
        SUM(signed_amount) OVER (
            PARTITION BY customer_id
            ORDER BY txn_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_balance
    FROM signed_transactions
),
monthly_closing AS (
    SELECT
        customer_id,
        DATE_FORMAT(txn_date, '%Y-%m') AS yr_month,
        MAX(txn_date) AS last_transaction_date
    FROM running_balances
    GROUP BY customer_id, DATE_FORMAT(txn_date, '%Y-%m')
),
monthly_closing_values AS (
    SELECT
        rb.customer_id,
        DATE_FORMAT(rb.txn_date, '%Y-%m') AS yr_month,
        rb.running_balance AS closing_balance,
        ROW_NUMBER() OVER (
            PARTITION BY rb.customer_id, DATE_FORMAT(rb.txn_date, '%Y-%m')
            ORDER BY rb.txn_date DESC
        ) AS rn
    FROM running_balances rb
)
SELECT
    customer_id,
    MIN(running_balance) AS minimum_running_balance,
    ROUND(AVG(running_balance), 2) AS average_running_balance,
    MAX(running_balance) AS maximum_running_balance
FROM running_balances
GROUP BY customer_id
ORDER BY customer_id;
```

To get the month-end balance output from the same CTE pattern, use:

```sql
WITH signed_transactions AS (
    SELECT
        customer_id,
        txn_date,
        CASE
            WHEN txn_type = 'deposit' THEN txn_amount
            WHEN txn_type IN ('purchase', 'withdrawal') THEN -txn_amount
        END AS signed_amount
    FROM customer_transactions
),
running_balances AS (
    SELECT
        customer_id,
        txn_date,
        SUM(signed_amount) OVER (
            PARTITION BY customer_id
            ORDER BY txn_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_balance
    FROM signed_transactions
),
ranked_month_end AS (
    SELECT
        customer_id,
        DATE_FORMAT(txn_date, '%Y-%m') AS yr_month,
        running_balance,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id, DATE_FORMAT(txn_date, '%Y-%m')
            ORDER BY txn_date DESC
        ) AS rn
    FROM running_balances
)
SELECT
    customer_id,
    yr_month,
    running_balance AS closing_balance
FROM ranked_month_end
WHERE rn = 1
ORDER BY customer_id, yr_month;
```

**Interpretation:** The minimum and maximum running balances show the observed range of each customer’s calculated balance, while the average gives a summary of their transaction-level balances. These metrics may help estimate how volatile a balance-based allowance could be.

**Recommendation:** Use these statistics to compare policy options and identify customers whose calculated balances fluctuate significantly. Avoid treating the calculated average of transaction-level balances as a true daily average; a daily balance series is needed for that.

## C5. Using all available data, how much data would have been required for each option on a monthly basis?

There is no explicit currency-to-data conversion rule in the source. The queries below therefore estimate the total of each balance-based **allowance proxy** per month, not actual gigabytes or storage capacity. They should only be interpreted after the business defines how monetary balances translate into data units.

### Option 1 monthly total — previous month-end balance

```sql
WITH signed_transactions AS (
    SELECT
        customer_id,
        DATE_FORMAT(txn_date, '%Y-%m') AS yr_month,
        CASE
            WHEN txn_type = 'deposit' THEN txn_amount
            WHEN txn_type IN ('purchase', 'withdrawal') THEN -txn_amount
        END AS signed_amount
    FROM customer_transactions
),
monthly_movement AS (
    SELECT customer_id, yr_month, SUM(signed_amount) AS net_movement
    FROM signed_transactions
    GROUP BY customer_id, yr_month
),
monthly_closing AS (
    SELECT
        customer_id,
        yr_month,
        SUM(net_movement) OVER (
            PARTITION BY customer_id
            ORDER BY yr_month
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS closing_balance
    FROM monthly_movement
),
monthly_allowance AS (
    SELECT
        customer_id,
        yr_month,
        LAG(closing_balance) OVER (
            PARTITION BY customer_id ORDER BY yr_month
        ) AS allowance_proxy
    FROM monthly_closing
)
SELECT
    yr_month,
    SUM(GREATEST(allowance_proxy, 0)) AS total_allowance_proxy
FROM monthly_allowance
WHERE allowance_proxy IS NOT NULL
GROUP BY yr_month
ORDER BY yr_month;
```

### Option 2 monthly total — previous 30-day average

Use the daily rolling-average result from Option 2, then aggregate it by calendar month:

```sql
-- Wrap the CTE query from Option 2 in a CTE named rolling_average,
-- with columns customer_id, calendar_date, average_balance_previous_30_days.

SELECT
    DATE_FORMAT(calendar_date, '%Y-%m') AS yr_month,
    SUM(GREATEST(average_balance_previous_30_days, 0))
        AS total_allowance_proxy
FROM rolling_average
GROUP BY DATE_FORMAT(calendar_date, '%Y-%m')
ORDER BY yr_month;
```

### Option 3 monthly total — real-time balance

```sql
WITH signed_transactions AS (
    SELECT
        customer_id,
        txn_date,
        CASE
            WHEN txn_type = 'deposit' THEN txn_amount
            WHEN txn_type IN ('purchase', 'withdrawal') THEN -txn_amount
        END AS signed_amount
    FROM customer_transactions
),
running_balance AS (
    SELECT
        customer_id,
        txn_date,
        SUM(signed_amount) OVER (
            PARTITION BY customer_id
            ORDER BY txn_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS balance
    FROM signed_transactions
)
SELECT
    DATE_FORMAT(txn_date, '%Y-%m') AS yr_month,
    SUM(GREATEST(balance, 0)) AS total_allowance_proxy
FROM running_balance
GROUP BY DATE_FORMAT(txn_date, '%Y-%m')
ORDER BY yr_month;
```

**How to compare the options:**

- **Option 1:** simple and predictable, but responds only to the previous month-end snapshot.
- **Option 2:** smoother and more representative of recent balances, but more expensive to calculate.
- **Option 3:** responsive to every transaction, but potentially more volatile and operationally complex.

Do not compare the monthly totals as actual data capacity until a conversion rule is agreed. Also clarify whether “monthly required data” means the sum of all customers’ allowance allocations, actual data consumed, or infrastructure storage; these are different metrics.

---

# Overall Business Insights and Recommendations

## 1. Node distribution should be assessed using workload, not customer count alone

Region 1 has the highest customer count and Region 5 the lowest, but customer counts alone do not show whether a region is overloaded.

**Recommendation:** Add node-level transaction volume, latency, failure rates, and processing costs to evaluate capacity and resilience.

## 2. Deposits are the largest transaction category by total value

The source results show 1,359,168 in deposits, compared with 793,003 in withdrawals and 806,537 in purchases.

**Recommendation:** Monitor net customer cash flow and changes over time. Do not interpret transaction totals as Data Bank revenue without a revenue model.

## 3. Customer activity is not evenly distributed across months

The number of customers meeting the multiple-deposit-and-outflow condition rises from 168 in January to 192 in March, then falls to 70 in April.

**Recommendation:** Check whether April contains a complete month of observations. Compare customer activity with retention, account balances, and transaction frequency before proposing operational changes.

## 4. Allocation policy involves a trade-off between stability and responsiveness

Previous-month balances are easy to explain, rolling averages smooth short-term fluctuations, and real-time balances react quickly.

**Recommendation:** Run a controlled simulation comparing allowance volatility, customer fairness, implementation cost, and total provisioned data under all three approaches. Define minimums, caps, negative-balance handling, and new-customer rules before selecting a policy.

## 5. Data definitions matter

The source analysis includes calculations that should be interpreted carefully:
- `COUNT(customer_id)` counts transaction rows, not unique customers.
- The average of signed transactions is not the month-end closing balance.
- A customer-month percentage change can be undefined when the previous balance is zero or negative.
- The date-only transaction field does not provide deterministic ordering for multiple transactions on the same day.
- The dataset does not define how a money balance maps to a data allowance.

**Recommendation:** Document metric definitions and validate calculations before presenting the figures as business KPIs.

---

# SQL Skills Demonstrated

- Aggregation: `COUNT`, `COUNT(DISTINCT)`, `SUM`, `AVG`, `MIN`, `MAX`
- Conditional aggregation with `CASE` and Boolean expressions
- Date handling: `DATEDIFF`, `DATE_FORMAT`, `DATE_ADD`
- Common table expressions (CTEs)
- Window functions: `LAG`, `SUM() OVER`, `ROW_NUMBER`, `CUME_DIST`
- Running totals and month-end balance calculations
- Percentile analysis
- Rolling 30-day averages
- Business metric interpretation and data-quality checks

# Conclusion

This case study demonstrates how SQL can be used to analyse node allocation, transaction behaviour, customer balance changes, and alternative data-allocation policies. The strongest business takeaway is that allocation rules should be evaluated not only for simplicity, but also for stability, fairness, responsiveness, and operational cost.

The results are descriptive and depend on the supplied transaction history. Before implementing a policy, Data Bank would need a defined conversion from account balance to data units, deterministic transaction timestamps, clear treatment of negative balances and new customers, and a larger-scale operational simulation.
