# 8 Week SQL Challenge — Case Study #3: Foodie-Fi

## Project overview

Foodie-Fi is a subscription streaming service focused on cooking content. This project analyses the customer subscription lifecycle, trial conversion, plan mix, churn, annual-plan upgrades, and a proposed 2020 payment ledger using **MySQL 8.0+**.

**Official case study:** [Case Study #3 — Foodie-Fi](https://8weeksqlchallenge.com/case-study-3/)

## Business questions and objectives

- Understand the onboarding paths customers take after a free trial.
- Measure trial starts, plan changes, churn, and plan mix at a historical snapshot.
- Quantify how long annual-plan conversion takes and how quickly it happens.
- Construct a payment ledger that follows the stated billing and upgrade rules.
- Translate descriptive SQL results into practical retention, product, and revenue recommendations.

## Dataset and schema

The case uses two source tables:

| Table | Purpose | Key fields |
|---|---|---|
| `plans` | The five subscription states and their prices | `plan_id`, `plan_name`, `price` |
| `subscriptions` | The date each customer changes to a plan | `customer_id`, `plan_id`, `start_date` |

| `plan_id` | `plan_name` | Price |
|---:|---|---:|
| 0 | trial | $0.00 |
| 1 | basic monthly | $9.90/month |
| 2 | pro monthly | $19.90/month |
| 3 | pro annual | $199.00/year |
| 4 | churn | NULL |

The official case explains that `subscriptions.start_date` represents when the plan-state event is recorded. In particular, churn is represented by a churn plan row; the customer may retain access until the end of the billing period. This matters when interpreting a snapshot and modelling payments.

## How to run the project

1. Use **MySQL 8.0 or later**, because the solutions use window functions and recursive common table expressions.
2. Run [`Foodie_Fi_Setup_MySQL.sql`](Foodie_Fi_Setup_MySQL.sql) to create the `foodie_fi` database and load the original `plans` and `subscriptions` data.
3. Run [`Foodie_Fi_Queries_MySQL.sql`](Foodie_Fi_Queries_MySQL.sql) to execute the SQL answers, including the `payments` table challenge.
4. Review the questions, reported outputs, and business interpretations below.

**Dialect note:** the uploaded source setup originally used PostgreSQL's `SET search_path`, while the accompanying analysis uses MySQL-specific functions such as `TIMESTAMPDIFF`. The two downloadable SQL files have been prepared for MySQL 8.0+ so that setup and queries use one dialect consistently.

## Skills demonstrated

- `COUNT(DISTINCT ...)`, grouping, and conditional aggregation
- Date truncation, `DATE_FORMAT`, `DATEDIFF`, and date-range filters
- Joins and plan-name lookup tables
- Window functions: `ROW_NUMBER`, `LAG`, `LEAD`, and window aggregates
- CTEs and recursive CTEs
- Historical plan snapshots and customer lifecycle sequencing
- Billing-event generation, prorated upgrade charges, and payment ordering
- Interpreting product metrics and acknowledging limitations

---

## A. Customer Journey

### Question

Based on the 8 sample customers shown in the `subscriptions` table, write a brief description of each customer's onboarding journey.

### SQL

```sql
SELECT
    s.customer_id,
    GROUP_CONCAT(
        CONCAT(p.plan_name, ' (', DATE_FORMAT(s.start_date, '%Y-%m-%d'), ')')
        ORDER BY s.start_date
        SEPARATOR ' -> '
    ) AS onboarding_journey
FROM subscriptions AS s
JOIN plans AS p
    ON s.plan_id = p.plan_id
WHERE s.customer_id IN (1, 2, 11, 13, 15, 16, 18, 19)
GROUP BY s.customer_id
ORDER BY s.customer_id;
```

### Result and interpretation

| Customer | Onboarding journey |
|---:|---|
| 1 | Trial (2020-08-01) → Basic monthly (2020-08-08) |
| 2 | Trial (2020-09-20) → Pro annual (2020-09-27) |
| 11 | Trial (2020-11-19) → Churn (2020-11-26) |
| 13 | Trial (2020-12-15) → Basic monthly (2020-12-22) → Pro monthly (2021-03-29) |
| 15 | Trial (2020-03-17) → Pro monthly (2020-03-24) → Churn (2020-04-29) |
| 16 | Trial (2020-05-31) → Basic monthly (2020-06-07) → Pro annual (2020-10-21) |
| 18 | Trial (2020-07-06) → Pro monthly (2020-07-13), remaining on Pro monthly in the sample period |
| 19 | Trial (2020-06-22) → Pro monthly (2020-06-29) → Pro annual (2020-08-29) |

The sample shows several conversion paths: trial to basic, trial to pro monthly, direct annual conversion, later upgrades, and early churn. It is illustrative only and should not be treated as a representative sample of all 1,000 customers.

---

## B. Data Analysis Questions

### Question 1. 1. How many customers has Foodie-Fi ever had?

```sql
SELECT COUNT(DISTINCT customer_id) AS total_customers
FROM subscriptions;
```

**Reported result**

| Metric | Result |
|---|---:|
| Distinct customers | 1,000 |

**Interpretation**

Foodie-Fi has 1,000 distinct customers in the supplied dataset. This is the total population in the case data, not a measure of currently active subscribers.

### Question 2. 2. What is the monthly distribution of trial plan start_date values for the dataset? Use the first day of the month as the grouping value.

```sql
SELECT
    CAST(DATE_FORMAT(start_date, '%Y-%m-01') AS DATE) AS month_start,
    COUNT(*) AS trial_starts
FROM subscriptions
WHERE plan_id = 0
GROUP BY CAST(DATE_FORMAT(start_date, '%Y-%m-01') AS DATE)
ORDER BY month_start;
```

**Reported result**

| Month start | Trial starts |
|---|---:|
| 2020-01-01 | 88 |
| 2020-02-01 | 68 |
| 2020-03-01 | 94 |
| 2020-04-01 | 81 |
| 2020-05-01 | 88 |
| 2020-06-01 | 79 |
| 2020-07-01 | 89 |
| 2020-08-01 | 88 |
| 2020-09-01 | 87 |
| 2020-10-01 | 79 |
| 2020-11-01 | 75 |
| 2020-12-01 | 84 |

**Interpretation**

Trial starts range from 68 in February to 94 in March. Volumes are fairly spread across the year, but one year of aggregate counts is not enough to identify seasonality or attribute changes to marketing activity.

### Question 3. 3. What plan start_date values occur after the year 2020? Show the count of events for each plan_name.

```sql
SELECT
    p.plan_id,
    p.plan_name,
    COUNT(*) AS plan_events
FROM subscriptions AS s
JOIN plans AS p
    ON s.plan_id = p.plan_id
WHERE s.start_date >= '2021-01-01'
GROUP BY p.plan_id, p.plan_name
ORDER BY p.plan_id;
```

**Reported result**

| Plan | Events starting from 2021 |
|---|---:|
| Basic monthly | 8 |
| Pro monthly | 60 |
| Pro annual | 63 |
| Churn | 71 |

**Interpretation**

The dataset includes substantial subscription activity after 2020, including 71 churn events. These are plan events, not necessarily unique customers: a customer may have multiple plan changes across the dataset.

### Question 4. 4. What is the customer count and percentage of customers who have churned, rounded to 1 decimal place?

```sql
WITH customer_totals AS (
    SELECT COUNT(DISTINCT customer_id) AS total_customers
    FROM subscriptions
), churn_totals AS (
    SELECT COUNT(DISTINCT customer_id) AS churned_customers
    FROM subscriptions
    WHERE plan_id = 4
)
SELECT
    c.churned_customers,
    t.total_customers,
    ROUND(100.0 * c.churned_customers / t.total_customers, 1) AS churn_pct
FROM churn_totals AS c
CROSS JOIN customer_totals AS t;
```

**Reported result**

| Churned customers | Total customers | Churn percentage |
|---:|---:|---:|
| 307 | 1,000 | 30.7% |

**Interpretation**

About 30.7% of customers have a churn record in the full dataset. This is an overall observed churn share, not a monthly churn rate, and includes events after 2020.

### Question 5. 5. How many customers churned straight after their initial free trial, and what percentage is this rounded to the nearest whole number?

```sql
WITH ranked_plans AS (
    SELECT
        customer_id,
        plan_id,
        start_date,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY start_date, plan_id
        ) AS plan_sequence
    FROM subscriptions
), customer_total AS (
    SELECT COUNT(DISTINCT customer_id) AS total_customers
    FROM subscriptions
), immediate_churn AS (
    SELECT COUNT(DISTINCT customer_id) AS churned_after_trial
    FROM ranked_plans
    WHERE plan_sequence = 2
      AND plan_id = 4
)
SELECT
    c.churned_after_trial,
    t.total_customers,
    ROUND(100.0 * c.churned_after_trial / t.total_customers, 0) AS pct_rounded_whole
FROM immediate_churn AS c
CROSS JOIN customer_total AS t;
```

**Reported result**

| Churned directly after trial | Total customers | Percentage (nearest whole %) |
|---:|---:|---:|
| 92 | 1,000 | 9% |

**Interpretation**

92 customers, or 9.2% of the total customer base (9% rounded to a whole number), have churn as their first plan transition after the trial. This is a key group for reviewing onboarding, content discovery, and trial reminders.

### Question 6. 6. What is the number and percentage of customer plans after their initial free trial?

```sql
WITH ranked_plans AS (
    SELECT
        s.customer_id,
        s.plan_id,
        p.plan_name,
        s.start_date,
        ROW_NUMBER() OVER (
            PARTITION BY s.customer_id
            ORDER BY s.start_date, s.plan_id
        ) AS plan_sequence
    FROM subscriptions AS s
    JOIN plans AS p
        ON s.plan_id = p.plan_id
), first_transition AS (
    SELECT customer_id, plan_id, plan_name
    FROM ranked_plans
    WHERE plan_sequence = 2
), plan_counts AS (
    SELECT plan_name, COUNT(DISTINCT customer_id) AS customers
    FROM first_transition
    GROUP BY plan_name
)
SELECT
    plan_name,
    customers,
    ROUND(100.0 * customers / (SELECT COUNT(*) FROM first_transition), 2) AS pct
FROM plan_counts
ORDER BY customers DESC;
```

**Reported result**

| First plan after trial | Customers | Share |
|---|---:|---:|
| Basic monthly | 546 | 54.60% |
| Pro monthly | 325 | 32.50% |
| Churn | 92 | 9.20% |
| Pro annual | 37 | 3.70% |

**Interpretation**

90.8% have a paid plan as their first transition after trial (54.6% basic monthly, 32.5% pro monthly, and 3.7% pro annual). Basic monthly is the most common initial paid choice. Longer-term retention and plan value should be evaluated separately from initial conversion.

### Question 7. 7. What is the customer count and percentage breakdown of all five plan_name values at 2020-12-31?

```sql
WITH plan_state AS (
    SELECT
        s.customer_id,
        p.plan_name,
        s.start_date,
        ROW_NUMBER() OVER (
            PARTITION BY s.customer_id
            ORDER BY s.start_date DESC, s.plan_id DESC
        ) AS latest_plan
    FROM subscriptions AS s
    JOIN plans AS p
        ON s.plan_id = p.plan_id
    WHERE s.start_date <= '2020-12-31'
), snapshot_counts AS (
    SELECT plan_name, COUNT(*) AS customers
    FROM plan_state
    WHERE latest_plan = 1
    GROUP BY plan_name
)
SELECT
    plan_name,
    customers,
    ROUND(100.0 * customers / SUM(customers) OVER (), 2) AS pct
FROM snapshot_counts
ORDER BY customers DESC;
```

**Reported result**

| Plan state recorded by 2020-12-31 | Customers | Share |
|---|---:|---:|
| Pro monthly | 326 | 32.60% |
| Churn | 236 | 23.60% |
| Basic monthly | 224 | 22.40% |
| Pro annual | 195 | 19.50% |
| Trial | 19 | 1.90% |

**Interpretation**

At the year-end snapshot, pro monthly is the largest recorded plan state (32.6%), followed by churn (23.6%). The snapshot uses the latest subscription event by that date; a churn record represents the cancellation event date in this dataset and does not necessarily mean access ended that same day.

### Question 8. 8. How many customers upgraded to an annual plan in 2020?

```sql
SELECT COUNT(DISTINCT customer_id) AS annual_upgrades_2020
FROM subscriptions
WHERE plan_id = 3
  AND start_date >= '2020-01-01'
  AND start_date < '2021-01-01';
```

**Reported result**

| Metric | Result |
|---|---:|
| Customers starting Pro annual in 2020 | 195 |

**Interpretation**

195 customers started the Pro annual plan during 2020. Annual conversions can be investigated by preceding plan, time since trial, and prior engagement to understand which journeys lead to longer-term commitments.

### Question 9. 9. How many days on average does it take for a customer to move to an annual plan from the day they join Foodie-Fi?

```sql
WITH trial_start AS (
    SELECT customer_id, MIN(start_date) AS trial_date
    FROM subscriptions
    WHERE plan_id = 0
    GROUP BY customer_id
), first_annual AS (
    SELECT customer_id, MIN(start_date) AS annual_date
    FROM subscriptions
    WHERE plan_id = 3
    GROUP BY customer_id
)
SELECT ROUND(AVG(DATEDIFF(a.annual_date, t.trial_date))) AS avg_days_to_annual
FROM trial_start AS t
JOIN first_annual AS a
    ON t.customer_id = a.customer_id
WHERE a.annual_date >= t.trial_date;
```

**Reported result**

| Metric | Result |
|---|---:|
| Average days from trial start to first annual plan | 105 days |

**Interpretation**

Customers who eventually move to an annual plan take about 105 days on average from trial start. This metric excludes customers who never choose annual and should be analysed alongside the distribution and median, not on its own.

### Question 10. 10. Further break down the average time to annual plan into 30-day periods (0–30 days, 31–60 days, etc.).

```sql
WITH trial_start AS (
    SELECT customer_id, MIN(start_date) AS trial_date
    FROM subscriptions
    WHERE plan_id = 0
    GROUP BY customer_id
), first_annual AS (
    SELECT customer_id, MIN(start_date) AS annual_date
    FROM subscriptions
    WHERE plan_id = 3
    GROUP BY customer_id
), conversion_days AS (
    SELECT DATEDIFF(a.annual_date, t.trial_date) AS days_to_annual
    FROM trial_start AS t
    JOIN first_annual AS a
        ON t.customer_id = a.customer_id
    WHERE a.annual_date >= t.trial_date
), bucketed AS (
    SELECT
        CASE
            WHEN days_to_annual <= 30 THEN 1
            WHEN days_to_annual <= 60 THEN 2
            WHEN days_to_annual <= 90 THEN 3
            WHEN days_to_annual <= 120 THEN 4
            WHEN days_to_annual <= 150 THEN 5
            WHEN days_to_annual <= 180 THEN 6
            WHEN days_to_annual <= 210 THEN 7
            WHEN days_to_annual <= 240 THEN 8
            WHEN days_to_annual <= 270 THEN 9
            WHEN days_to_annual <= 300 THEN 10
            WHEN days_to_annual <= 330 THEN 11
            WHEN days_to_annual <= 360 THEN 12
            ELSE 13
        END AS bucket_order,
        CASE
            WHEN days_to_annual <= 30 THEN '0-30 days'
            WHEN days_to_annual <= 60 THEN '31-60 days'
            WHEN days_to_annual <= 90 THEN '61-90 days'
            WHEN days_to_annual <= 120 THEN '91-120 days'
            WHEN days_to_annual <= 150 THEN '121-150 days'
            WHEN days_to_annual <= 180 THEN '151-180 days'
            WHEN days_to_annual <= 210 THEN '181-210 days'
            WHEN days_to_annual <= 240 THEN '211-240 days'
            WHEN days_to_annual <= 270 THEN '241-270 days'
            WHEN days_to_annual <= 300 THEN '271-300 days'
            WHEN days_to_annual <= 330 THEN '301-330 days'
            WHEN days_to_annual <= 360 THEN '331-360 days'
            ELSE '361+ days'
        END AS day_bucket
    FROM conversion_days
)
SELECT day_bucket, COUNT(*) AS customers
FROM bucketed
GROUP BY bucket_order, day_bucket
ORDER BY bucket_order;
```

**Reported result**

| Time to annual plan | Customers |
|---|---:|
| 0–30 days | 49 |
| 31–60 days | 24 |
| 61–90 days | 34 |
| 91–120 days | 35 |
| 121–150 days | 42 |
| 151–180 days | 36 |
| 181–210 days | 26 |
| 211–240 days | 4 |
| 241–270 days | 5 |
| 271–300 days | 1 |
| 301–330 days | 1 |
| 331–360 days | 1 |

**Interpretation**

107 of 258 annual-plan customers (about 41.5%) converted within 90 days of the initial trial date. A timely onboarding and upgrade journey may be worth testing, but these observations do not establish which intervention would cause more upgrades.

### Question 11. 11. How many customers downgraded from Pro monthly to Basic monthly in 2020?

```sql
WITH plan_sequence AS (
    SELECT
        customer_id,
        plan_id,
        start_date,
        LEAD(plan_id) OVER (
            PARTITION BY customer_id
            ORDER BY start_date, plan_id
        ) AS next_plan_id
    FROM subscriptions
)
SELECT COUNT(DISTINCT customer_id) AS pro_to_basic_downgrades
FROM plan_sequence
WHERE plan_id = 2
  AND next_plan_id = 1
  AND start_date >= '2020-01-01'
  AND start_date < '2021-01-01';
```

**Reported result**

| Metric | Result |
|---|---:|
| Pro monthly → Basic monthly downgrades | 0 |

**Interpretation**

No sequence matching a Pro monthly subscription followed by Basic monthly was identified with a Pro monthly start date in 2020. This should not be interpreted as proof that customers never downgrade in later periods or outside the observed data.

---

## C. Challenge Payment Question

### Question

Create a new `payments` table for 2020 that includes the amount paid by each customer. The requirements are:

1. Monthly payments occur on the same day of the month as the start date of the relevant monthly paid plan.
2. Upgrades from Basic to Pro monthly or Pro annual take effect immediately and credit the current $9.90 Basic monthly amount.
3. Pro monthly → Pro annual is charged at the end of the current monthly billing period, as represented by the annual plan's recorded start date.
4. After a churn event, no new payments should be generated.

### SQL solution (MySQL 8.0+)

The query below builds the subscription event sequence, generates recurring monthly payment dates from each original monthly-plan start date, adds annual-plan charges, applies the Basic-plan credit on Basic-to-Pro upgrades, and numbers each customer's payments.

```sql
DROP TABLE IF EXISTS payments;

CREATE TABLE payments AS
WITH RECURSIVE subscription_events AS (
    SELECT
        s.customer_id,
        s.plan_id,
        p.plan_name,
        p.price,
        s.start_date,
        LAG(s.plan_id) OVER (
            PARTITION BY s.customer_id
            ORDER BY s.start_date, s.plan_id
        ) AS previous_plan_id,
        LEAD(s.plan_id) OVER (
            PARTITION BY s.customer_id
            ORDER BY s.start_date, s.plan_id
        ) AS next_plan_id,
        LEAD(s.start_date) OVER (
            PARTITION BY s.customer_id
            ORDER BY s.start_date, s.plan_id
        ) AS next_start_date
    FROM subscriptions AS s
    JOIN plans AS p
        ON s.plan_id = p.plan_id
), monthly_due (
    customer_id,
    plan_id,
    plan_name,
    plan_start_date,
    next_start_date,
    previous_plan_id,
    month_offset,
    payment_date,
    price
) AS (
    -- First monthly payment date for each Basic/Pro monthly subscription.
    SELECT
        customer_id,
        plan_id,
        plan_name,
        start_date AS plan_start_date,
        next_start_date,
        previous_plan_id,
        0 AS month_offset,
        start_date AS payment_date,
        price
    FROM subscription_events
    WHERE plan_id IN (1, 2)
      AND start_date < '2021-01-01'

    UNION ALL

    -- Generate recurring dates from the original start date so day-of-month is preserved.
    SELECT
        customer_id,
        plan_id,
        plan_name,
        plan_start_date,
        next_start_date,
        previous_plan_id,
        month_offset + 1,
        DATE_ADD(plan_start_date, INTERVAL (month_offset + 1) MONTH) AS payment_date,
        price
    FROM monthly_due
    WHERE DATE_ADD(plan_start_date, INTERVAL (month_offset + 1) MONTH)
              < COALESCE(next_start_date, '9999-12-31')
      AND DATE_ADD(plan_start_date, INTERVAL (month_offset + 1) MONTH) < '2021-01-01'
), monthly_payments AS (
    SELECT
        customer_id,
        plan_id,
        plan_name,
        payment_date,
        CAST(
            CASE
                WHEN month_offset = 0
                 AND plan_id = 2
                 AND previous_plan_id = 1
                    THEN price - 9.90
                ELSE price
            END AS DECIMAL(7,2)
        ) AS amount
    FROM monthly_due
    WHERE payment_date >= '2020-01-01'
      AND payment_date < '2021-01-01'
), annual_payments AS (
    SELECT
        customer_id,
        plan_id,
        plan_name,
        start_date AS payment_date,
        CAST(
            CASE
                WHEN previous_plan_id = 1 THEN price - 9.90
                ELSE price
            END AS DECIMAL(7,2)
        ) AS amount
    FROM subscription_events
    WHERE plan_id = 3
      AND start_date >= '2020-01-01'
      AND start_date < '2021-01-01'
), all_payments AS (
    SELECT customer_id, plan_id, plan_name, payment_date, amount
    FROM monthly_payments
    UNION ALL
    SELECT customer_id, plan_id, plan_name, payment_date, amount
    FROM annual_payments
)
SELECT
    customer_id,
    plan_id,
    plan_name,
    payment_date,
    amount,
    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY payment_date, plan_id
    ) AS payment_order
FROM all_payments;
```

### Example validation rows

The official case page provides validation examples such as:

| Customer | Plan | Payment date | Amount | Payment order |
|---:|---|---|---:|---:|
| 1 | Basic monthly | 2020-08-08 | $9.90 | 1 |
| 1 | Basic monthly | 2020-09-08 | $9.90 | 2 |
| 2 | Pro annual | 2020-09-27 | $199.00 | 1 |
| 15 | Pro monthly | 2020-03-24 | $19.90 | 1 |
| 15 | Pro monthly | 2020-04-24 | $19.90 | 2 |
| 16 | Basic monthly | 2020-10-07 | $9.90 | 5 |
| 16 | Pro annual | 2020-10-21 | $189.10 | 6 |
| 19 | Pro annual | 2020-08-29 | $199.00 | 3 |

**Interpretation and assumptions**

The query creates a table containing monthly charges and annual-plan charges for payment dates in 2020. It credits $9.90 when a customer moves from Basic monthly to Pro monthly or Pro annual, and it stops monthly recurring rows before the next plan event. The subscriptions table encodes when plan changes take effect; for Pro monthly → Pro annual, the annual payment is recorded on that change date. Validate edge cases and the final table against the business rules before using it in production.

**Post-build checks**

```sql
-- Check a few customer journeys against the case examples.
SELECT customer_id, plan_id, plan_name, payment_date, amount, payment_order
FROM payments
WHERE customer_id IN (1, 2, 13, 15, 16, 18, 19)
ORDER BY customer_id, payment_order;

-- Check that all payment dates are in 2020 and no trial/churn rows were charged.
SELECT
    MIN(payment_date) AS first_payment_date,
    MAX(payment_date) AS last_payment_date,
    COUNT(*) AS payment_rows,
    SUM(plan_id IN (0, 4)) AS invalid_plan_charges
FROM payments;

-- The following should return no rows if payment_order is unique per customer.
SELECT customer_id, payment_order, COUNT(*) AS row_count
FROM payments
GROUP BY customer_id, payment_order
HAVING COUNT(*) > 1;
```

---

## D. Outside-the-Box Questions

These are open-ended business and technical interview prompts from the official case. The answers below are recommendations rather than unique correct answers.

### 1. How would you calculate the rate of growth for Foodie-Fi?

I would report growth from several angles, because a single number can hide important differences:

- **Paying subscriber growth:** `(paying subscribers at end of period − paying subscribers at start of period) / paying subscribers at start of period`.
- **New paid conversion growth:** compare the number of customers converting from trial to a paid plan against the previous comparable period.
- **Revenue growth:** compare monthly recurring revenue (MRR) or recognised subscription revenue period over period. Annual-plan revenue recognition must be defined consistently before comparing it with monthly-plan revenue.
- **Net growth:** new paid subscribers minus churned subscribers, with upgrades and downgrades shown separately.

I would calculate month-over-month and year-over-year changes, compare cohorts, and avoid claiming that higher trial volume necessarily means healthier growth if churn is increasing.

### 2. What key metrics should management track to assess overall business performance?

| Metric | Why it matters |
|---|---|
| New trials and trial-to-paid conversion | Measures acquisition quality and initial activation |
| Paying subscribers by plan | Tracks customer base composition |
| MRR and annual recurring revenue (ARR) | Tracks recurring revenue; annual-plan treatment must be defined consistently |
| Customer churn and revenue churn | Measures customer loss and the financial impact of cancellations |
| Retention by signup cohort | Shows whether newer customer cohorts stay longer |
| Upgrade, downgrade, and annual conversion rates | Shows plan movement and willingness to commit longer term |
| Average revenue per paying customer | Helps track monetisation by segment and plan |
| Engagement: video starts, watch time, repeat viewing, downloads | Identifies whether customers are receiving ongoing value; these fields are not in the current dataset |
| Customer acquisition cost and customer lifetime value | Evaluates sustainable growth; acquisition-spend data is not provided here |
| Payment failures and customer satisfaction | Helps diagnose involuntary churn and experience problems; those data are not provided here |

### 3. What customer journeys should be analysed further to improve retention?

I would prioritise: (1) trial users who churn at the first plan transition, (2) trial users who convert to Basic but do not upgrade, (3) early Pro monthly customers who churn, (4) customers who upgrade to annual early versus late, and (5) customers whose viewing engagement declines before cancellation. Compare each journey by start month, plan, content engagement, and acquisition source when those fields become available.

### 4. What questions should be included in an exit survey?

Keep the survey short and allow multiple selections plus an optional free-text response:

- What is the main reason you are cancelling? (price, not enough relevant content, low usage, technical problems, switching service, temporary break, other)
- Did Foodie-Fi meet your expectations during the trial or subscription?
- Was there specific content you wanted but could not find?
- How satisfied were you with content discovery, playback quality, and app usability?
- Would a different plan, lower price, pause option, or annual discount change your decision?
- How likely are you to return, and may we contact you with a relevant offer?

Do not make a discount the default response to every cancellation; segment responses and look for recurring product issues.

### 5. What levers could reduce churn, and how would you validate them?

Potential levers include improving the trial onboarding experience, recommending relevant cooking content sooner, sending lifecycle reminders based on engagement, addressing missing content, offering a pause plan, and testing plan or annual-offer presentation. Validate each intervention with a controlled A/B test when feasible. Compare paid conversion, 30/60/90-day retention, churn, engagement, and revenue per user, while checking guardrails such as refunds, support contacts, and discount-driven margin erosion. Run tests long enough to cover the relevant billing and renewal cycles.

---

## Key Findings and Business Recommendations

1. **Trial-to-paid conversion is strong in the observed sequence.** 908 of 1,000 customers moved to a paid plan after trial; 9.2% churned immediately and 90.8% chose a paid plan. Investigate how to improve the first-week experience and monitor later retention in addition to conversion.
2. **Basic monthly is the most common first paid plan.** It accounts for 54.6% of first post-trial transitions. Test plan comparison and upgrade prompts only if they clearly communicate customer value; avoid pushing upgrades that may increase short-term revenue but hurt retention.
3. **Overall churn is material.** 307 customers (30.7%) have a churn record in the full dataset. Review cancellation timing and reasons by prior plan, tenure, and engagement. The 236 churn records in the 2020-12-31 snapshot are a historical state count and are not the same metric as the full-dataset churn total.
4. **Annual upgrades usually occur after the trial period.** The average time to annual plan is 105 days, and 41.5% of annual-plan converters upgraded within 90 days. Analyse engagement and plan history before designing upgrade campaigns; conversion timing alone does not explain why customers upgraded.
5. **Year-end plan mix is concentrated in Pro monthly and churn.** At 2020-12-31, 32.6% of customers were in Pro monthly and 23.6% had a churn record. Retention experiments should focus on value delivery and cancellation reasons, not on plan mix alone.

## Data Limitations and Quality Notes

- The supplied dataset contains subscription events, not actual viewing behaviour, acquisition spend, support contacts, payment failures, or exit-survey responses.
- `plan_id = 4` is a churn event. It should not automatically be interpreted as the exact time account access stops, because the case states that access can continue to the end of the billing period.
- Query B3 counts plan-change events after 2020, not distinct customers.
- The sample journeys are only eight customers and are illustrative.
- The payment table is a reconstructed ledger based on case rules, not evidence of actual payment transactions. Validate the output against the provided examples and explicitly document assumptions about crediting upgrades and billing boundaries.
- This is a small, single-company teaching dataset; the results support descriptive analysis but not causal claims.

## Conclusion

This case demonstrates how SQL can be used to analyse subscription lifecycle events, calculate trial conversion and churn metrics, track plan-state snapshots, measure time to annual conversion, and reconstruct payment events. The business opportunity is to combine these subscription metrics with content engagement, acquisition, and cancellation-reason data to make retention and monetisation decisions with stronger evidence.

## References

- [Official 8 Week SQL Challenge — Case Study #3: Foodie-Fi](https://8weeksqlchallenge.com/case-study-3/)
- [MySQL 8.0 CTE documentation](https://dev.mysql.com/doc/refman/8.0/en/with.html)
