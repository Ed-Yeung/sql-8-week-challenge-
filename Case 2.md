# 8 Week SQL Challenge — Case Study #2: Pizza Runner

**SQL dialect:** MySQL 8.0.16+  
**Project type:** Data cleaning, exploratory analysis, operations analytics, menu analysis, and revenue estimation  
**Source case study:** [8 Week SQL Challenge — Pizza Runner](https://8weeksqlchallenge.com/case-study-2/)  
**Dataset / original work:** The questions and the initial SQL outputs below are based on my uploaded `Case 2.session.sql` and `case 2.docx`. I have retained the reported results where useful and identified corrections separately.

---

## 1. Project overview

Pizza Runner is a pizza delivery business that records customer orders, pizza types, customisations, runner assignments, pickups, delivery distance, and delivery duration. The goal of this project is to clean inconsistent source data, use SQL to answer the case-study questions, and translate the results into reasonable business insights.

### Business questions

- How much pizza is being ordered and delivered, and which products are most popular?
- When does demand occur, and how does order size relate to the time an order is ready for pickup?
- How do delivery distance, duration, speed, and delivery success vary?
- Which toppings are most often added or excluded, and how can ingredient usage be measured?
- What revenue is generated under different pricing assumptions, and how much remains after runner delivery payments?

### SQL skills demonstrated

`SELECT`, filtering, aggregation, `GROUP BY`, `CASE`, joins, CTEs, recursive CTEs, date/time functions, string cleaning and splitting, window functions, `GROUP_CONCAT`, subqueries, table design, and `INSERT` statements.

---

## 2. Important data-quality note

The uploaded SQL file inserts `customer_orders.order_time` values in **January 2020**, while the case-study page presents those customer order timestamps in **January 2021** and the `runners.registration_date` values are in January 2021. The uploaded outputs for the weekday question therefore reflect 2020 calendar weekdays. The runner pickup dates for some rows also need to be checked against the corresponding customer-order timestamps.

**Do not silently change dates just to match an expected result.** Inspect the actual loaded tables, decide whether the year difference is a transcription error, correct the source data if appropriate, and rerun the queries. This matters for weekday analysis, signup-week analysis, and elapsed-time calculations.

The queries below use MySQL 8.0+ and the cleaned tables described next. All metrics should be treated as descriptive because the dataset is very small.

---

## 3. Data preparation

Run the source schema and `INSERT` statements first (the supplied `Case 2.session.sql` contains the raw tables and sample data). Then run the following cleaning steps in the **same MySQL session**, because temporary tables only exist for that connection.

### 3.1 Clean customer orders

The source data represents missing customisations using a mixture of SQL `NULL`, the string `'null'`, `'NaN'`, and empty values. Convert those values to empty strings and remove spaces between topping IDs.

```sql
USE pizza_runner;

DROP TEMPORARY TABLE IF EXISTS customer_orders_clean;

CREATE TEMPORARY TABLE customer_orders_clean AS
SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
        WHEN exclusions IS NULL
          OR LOWER(TRIM(exclusions)) IN ('', 'null', 'nan')
        THEN ''
        ELSE REPLACE(TRIM(exclusions), ' ', '')
    END AS exclusions,
    CASE
        WHEN extras IS NULL
          OR LOWER(TRIM(extras)) IN ('', 'null', 'nan')
        THEN ''
        ELSE REPLACE(TRIM(extras), ' ', '')
    END AS extras,
    order_time
FROM customer_orders;
```

### 3.2 Clean runner orders

The source `distance` and `duration` columns contain values such as `'20km'`, `'23.4 km'`, `'32 minutes'`, and `'15 minute'`. The query below extracts numeric values and normalises missing data. `duration` is retained in minutes and `distance` in kilometres.

```sql
DROP TEMPORARY TABLE IF EXISTS runner_orders_clean;

CREATE TEMPORARY TABLE runner_orders_clean AS
SELECT
    order_id,
    runner_id,
    CASE
        WHEN pickup_time IS NULL
          OR LOWER(TRIM(pickup_time)) IN ('', 'null', 'nan')
        THEN NULL
        ELSE STR_TO_DATE(TRIM(pickup_time), '%Y-%m-%d %H:%i:%s')
    END AS pickup_time,
    CASE
        WHEN distance IS NULL
          OR LOWER(TRIM(distance)) IN ('', 'null', 'nan')
        THEN 0
        ELSE CAST(
            REPLACE(REPLACE(LOWER(TRIM(distance)), 'km', ''), ' ', '')
            AS DECIMAL(8, 2)
        )
    END AS distance,
    CASE
        WHEN duration IS NULL
          OR LOWER(TRIM(duration)) IN ('', 'null', 'nan')
        THEN 0
        ELSE CAST(REGEXP_SUBSTR(duration, '[0-9]+') AS UNSIGNED)
    END AS duration,
    CASE
        WHEN cancellation IS NULL
          OR LOWER(TRIM(cancellation)) IN ('', 'null', 'nan')
        THEN ''
        ELSE TRIM(cancellation)
    END AS cancellation
FROM runner_orders;
```

### 3.3 Create a stable row identifier for each pizza line

An order can contain multiple pizzas and can contain repeated-looking pizza rows. A row identifier helps connect each pizza line to its exclusions and extras in the recursive parsing queries. Exact duplicate rows are interchangeable in this sample; the generated ID is for analysis, not a real source-system key.

```sql
DROP TEMPORARY TABLE IF EXISTS customer_order_items_clean;

CREATE TEMPORARY TABLE customer_order_items_clean AS
SELECT
    ROW_NUMBER() OVER (
        ORDER BY order_id, customer_id, pizza_id, order_time, exclusions, extras
    ) AS item_id,
    order_id,
    customer_id,
    pizza_id,
    exclusions,
    extras,
    order_time
FROM customer_orders_clean;
```

### 3.4 Split comma-separated recipe, exclusion, and extra IDs

The topping IDs are stored as comma-separated text rather than one row per topping. These helper tables convert those strings into rows. The later ingredient queries reuse these helpers.

```sql
DROP TEMPORARY TABLE IF EXISTS pizza_recipe_toppings_clean;

CREATE TEMPORARY TABLE pizza_recipe_toppings_clean AS
WITH RECURSIVE recipe_split AS (
    SELECT
        pizza_id,
        TRIM(SUBSTRING_INDEX(toppings, ',', 1)) AS topping_id_text,
        CASE
            WHEN INSTR(toppings, ',') > 0
            THEN TRIM(SUBSTRING(toppings, INSTR(toppings, ',') + 1))
            ELSE ''
        END AS remaining
    FROM pizza_recipes

    UNION ALL

    SELECT
        pizza_id,
        TRIM(SUBSTRING_INDEX(remaining, ',', 1)),
        CASE
            WHEN INSTR(remaining, ',') > 0
            THEN TRIM(SUBSTRING(remaining, INSTR(remaining, ',') + 1))
            ELSE ''
        END
    FROM recipe_split
    WHERE remaining <> ''
)
SELECT
    pizza_id,
    CAST(topping_id_text AS UNSIGNED) AS topping_id
FROM recipe_split
WHERE topping_id_text <> '';

DROP TEMPORARY TABLE IF EXISTS customer_order_exclusions_clean;

CREATE TEMPORARY TABLE customer_order_exclusions_clean AS
WITH RECURSIVE exclusion_split AS (
    SELECT
        item_id,
        TRIM(SUBSTRING_INDEX(exclusions, ',', 1)) AS topping_id_text,
        CASE
            WHEN INSTR(exclusions, ',') > 0
            THEN TRIM(SUBSTRING(exclusions, INSTR(exclusions, ',') + 1))
            ELSE ''
        END AS remaining
    FROM customer_order_items_clean
    WHERE exclusions <> ''

    UNION ALL

    SELECT
        item_id,
        TRIM(SUBSTRING_INDEX(remaining, ',', 1)),
        CASE
            WHEN INSTR(remaining, ',') > 0
            THEN TRIM(SUBSTRING(remaining, INSTR(remaining, ',') + 1))
            ELSE ''
        END
    FROM exclusion_split
    WHERE remaining <> ''
)
SELECT
    item_id,
    CAST(topping_id_text AS UNSIGNED) AS topping_id
FROM exclusion_split
WHERE topping_id_text <> '';

DROP TEMPORARY TABLE IF EXISTS customer_order_extras_clean;

CREATE TEMPORARY TABLE customer_order_extras_clean AS
WITH RECURSIVE extra_split AS (
    SELECT
        item_id,
        TRIM(SUBSTRING_INDEX(extras, ',', 1)) AS topping_id_text,
        CASE
            WHEN INSTR(extras, ',') > 0
            THEN TRIM(SUBSTRING(extras, INSTR(extras, ',') + 1))
            ELSE ''
        END AS remaining
    FROM customer_order_items_clean
    WHERE extras <> ''

    UNION ALL

    SELECT
        item_id,
        TRIM(SUBSTRING_INDEX(remaining, ',', 1)),
        CASE
            WHEN INSTR(remaining, ',') > 0
            THEN TRIM(SUBSTRING(remaining, INSTR(remaining, ',') + 1))
            ELSE ''
        END
    FROM extra_split
    WHERE remaining <> ''
)
SELECT
    item_id,
    CAST(topping_id_text AS UNSIGNED) AS topping_id
FROM extra_split
WHERE topping_id_text <> '';
```

### 3.5 Define a successful delivery consistently

For this project, a successful delivery has an empty cancellation field and positive distance and duration. The helper expression is repeated in the queries so that cancelled or undelivered orders are not accidentally counted as deliveries.

```sql
cancellation = '' AND distance > 0 AND duration > 0
```

---

# 4. Case Study Questions and SQL Solutions

The official case study has **29 questions**: 10 in Pizza Metrics, 7 in Runner and Customer Experience, 6 in Ingredient Optimisation, 5 in Pricing and Ratings, and 1 bonus question. Every question is included below.

# A. Pizza Metrics

## A1. How many pizzas were ordered?

```sql
SELECT COUNT(*) AS pizzas_ordered
FROM customer_orders_clean;
```

**Reported result:** 14 pizza lines.

**Interpretation:** Fourteen individual pizzas were recorded across all customer orders. This counts each pizza row, not unique orders.

## A2. How many unique customer orders were made?

```sql
SELECT COUNT(DISTINCT order_id) AS unique_customer_orders
FROM customer_orders_clean;
```

**Reported result:** 10 unique orders.

**Interpretation:** The 14 pizzas were grouped into 10 order IDs, so at least some customers ordered more than one pizza within a single order.

## A3. How many successful orders were delivered by each runner?

```sql
SELECT
    runner_id,
    COUNT(*) AS successful_orders
FROM runner_orders_clean
WHERE cancellation = ''
  AND distance > 0
  AND duration > 0
GROUP BY runner_id
ORDER BY runner_id;
```

**Reported result:** Runner 1 delivered 4 orders, Runner 2 delivered 3, and Runner 3 delivered 1. There were 8 successful deliveries in total.

**Interpretation:** Runner 1 completed the most deliveries in this sample. This is a workload count, not proof that Runner 1 is the best-performing runner; runners may have received different assignments or delivery conditions.

## A4. How many of each type of pizza was delivered?

```sql
SELECT
    pn.pizza_name,
    COUNT(*) AS pizzas_delivered
FROM customer_orders_clean AS c
JOIN runner_orders_clean AS r
    ON c.order_id = r.order_id
JOIN pizza_names AS pn
    ON c.pizza_id = pn.pizza_id
WHERE r.cancellation = ''
  AND r.distance > 0
  AND r.duration > 0
GROUP BY pn.pizza_id, pn.pizza_name
ORDER BY pizzas_delivered DESC;
```

**Reported result:** Meat Lovers = 9; Vegetarian = 3; total = 12 pizzas delivered.

**Interpretation:** Meat Lovers accounts for 75% of delivered pizza volume. The business should ensure availability of its ingredients while still tracking stock waste and product-level margins.

## A5. How many Vegetarian and Meat Lovers pizzas were ordered by each customer?

```sql
SELECT
    c.customer_id,
    pn.pizza_name,
    COUNT(*) AS pizzas_delivered
FROM customer_orders_clean AS c
JOIN runner_orders_clean AS r
    ON c.order_id = r.order_id
JOIN pizza_names AS pn
    ON c.pizza_id = pn.pizza_id
WHERE r.cancellation = ''
  AND r.distance > 0
  AND r.duration > 0
GROUP BY c.customer_id, pn.pizza_id, pn.pizza_name
ORDER BY c.customer_id, pn.pizza_name;
```

**Correction to the uploaded query:** the original query joins to runner orders but does not filter cancellations or failed deliveries. The query above answers the wording more precisely by counting successfully delivered pizzas.

**Corrected result from the supplied rows:**

| Customer | Meat Lovers | Vegetarian |
|---:|---:|---:|
| 101 | 2 | 0 |
| 102 | 2 | 1 |
| 103 | 2 | 1 |
| 104 | 3 | 0 |
| 105 | 0 | 1 |

**Interpretation:** Meat Lovers is the dominant product across these customers. Customer 103's third Meat Lovers pizza in the uploaded result comes from an order that was cancelled, which is why a successful-delivery filter changes that count.

## A6. What was the maximum number of pizzas delivered in a single order?

```sql
WITH pizzas_per_order AS (
    SELECT
        c.order_id,
        COUNT(*) AS pizza_count
    FROM customer_orders_clean AS c
    JOIN runner_orders_clean AS r
        ON c.order_id = r.order_id
    WHERE r.cancellation = ''
      AND r.distance > 0
      AND r.duration > 0
    GROUP BY c.order_id
)
SELECT MAX(pizza_count) AS maximum_pizzas_in_order
FROM pizzas_per_order;
```

**Reported result:** 3 pizzas.

**Interpretation:** The largest successful order contains three pizzas. Larger orders may need more preparation capacity, but this sample alone cannot show how frequently such orders occur in normal operations.

## A7. For each customer, how many delivered pizzas had at least one change and how many had no changes?

```sql
SELECT
    c.customer_id,
    SUM(CASE
        WHEN c.exclusions <> '' OR c.extras <> '' THEN 1
        ELSE 0
    END) AS pizzas_with_changes,
    SUM(CASE
        WHEN c.exclusions = '' AND c.extras = '' THEN 1
        ELSE 0
    END) AS pizzas_without_changes
FROM customer_orders_clean AS c
JOIN runner_orders_clean AS r
    ON c.order_id = r.order_id
WHERE r.cancellation = ''
  AND r.distance > 0
  AND r.duration > 0
GROUP BY c.customer_id
ORDER BY c.customer_id;
```

**Reported result:** Customer 101: 0 with changes, 2 without; Customer 102: 0 and 3; Customer 103: 3 and 0; Customer 104: 2 and 1; Customer 105: 1 and 0.

**Interpretation:** Customisation behaviour varies. Customer 103 customised every successfully delivered pizza in this small sample, whereas Customers 101 and 102 did not customise any of their delivered pizzas. This is not enough evidence to infer long-term customer preferences.

## A8. How many pizzas were delivered that had both exclusions and extras?

```sql
SELECT COUNT(*) AS pizzas_with_exclusions_and_extras
FROM customer_orders_clean AS c
JOIN runner_orders_clean AS r
    ON c.order_id = r.order_id
WHERE r.cancellation = ''
  AND r.distance > 0
  AND r.duration > 0
  AND c.exclusions <> ''
  AND c.extras <> '';
```

**Reported result:** 1 pizza.

**Interpretation:** Only one successfully delivered pizza line in this sample contains both exclusions and extras. A reliable count over a longer period could help estimate how frequently complex orders occur.

## A9. What was the total volume of pizzas ordered for each hour of the day?

```sql
SELECT
    HOUR(order_time) AS order_hour,
    COUNT(*) AS pizzas_ordered
FROM customer_orders_clean
GROUP BY HOUR(order_time)
ORDER BY order_hour;
```

**Reported result from the uploaded data:**

| Hour | Pizzas ordered |
|---:|---:|
| 11 | 1 |
| 13 | 3 |
| 18 | 3 |
| 19 | 1 |
| 21 | 3 |
| 23 | 3 |

**Interpretation:** The recorded orders appear around lunch and evening hours. Several hours tie at three pizzas, so there is no single dominant hour in this small sample. Use more weeks of data before changing staffing schedules.

## A10. What was the volume of orders for each day of the week?

The original query counted pizza rows, not unique order IDs. The query below returns both metrics, which makes the distinction explicit.

```sql
SELECT
    DAYNAME(order_time) AS weekday,
    COUNT(DISTINCT order_id) AS unique_orders,
    COUNT(*) AS pizzas_ordered
FROM customer_orders_clean
GROUP BY DAYOFWEEK(order_time), DAYNAME(order_time)
ORDER BY DAYOFWEEK(order_time);
```

**Reported output from the uploaded file:** Wednesday = 5, Thursday = 3, Friday = 1, Saturday = 5 when counting pizza rows. These weekday labels reflect the 2020 dates in the uploaded SQL, not the 2021 order dates shown in the case-study prompt.

**If the order timestamps are corrected to the 2021 dates shown in the case-study tables:** the pizza-row volume is Monday = 5, Friday = 5, Saturday = 3, Sunday = 1. Unique order counts for those days are Monday = 2, Friday = 5, Saturday = 2, Sunday = 1. Rerun the query after confirming the year in your underlying data.

**Interpretation:** Weekday performance cannot be trusted until the year values are consistent. Also, count unique orders for order volume; count rows for pizza-item volume.

---

# B. Runner and Customer Experience

## B1. How many runners signed up for each one-week period, where the week starts on 2021-01-01?

The uploaded query groups runner deliveries by pickup week. The question asks when runners signed up, so use `runners.registration_date` instead.

```sql
SELECT
    FLOOR(DATEDIFF(registration_date, '2021-01-01') / 7) AS signup_week,
    COUNT(*) AS runners_signed_up
FROM runners
WHERE registration_date >= '2021-01-01'
  AND registration_date < '2021-01-22'
GROUP BY FLOOR(DATEDIFF(registration_date, '2021-01-01') / 7)
ORDER BY signup_week;
```

**Result from the registration dates in the supplied data:** Week 0 = 2 runners (Jan 1 and Jan 3); Week 1 = 1 runner (Jan 8); Week 2 = 1 runner (Jan 15).

**Interpretation:** Two runners joined in the first seven-day period, followed by one runner in each of the next two periods. This measures recruitment, not the number of runners actively delivering in a week.

## B2. What was the average time in minutes for each runner to arrive at Pizza Runner HQ and pick up an order?

The original query joins each order to each pizza line, which repeats the pickup time for multi-pizza orders. The query below first reduces customer order data to one row per order.

```sql
WITH order_level AS (
    SELECT
        order_id,
        MIN(order_time) AS order_time
    FROM customer_orders_clean
    GROUP BY order_id
)
SELECT
    r.runner_id,
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, o.order_time, r.pickup_time)), 2)
        AS avg_order_to_pickup_minutes
FROM runner_orders_clean AS r
JOIN order_level AS o
    ON r.order_id = o.order_id
WHERE r.cancellation = ''
  AND r.distance > 0
  AND r.duration > 0
GROUP BY r.runner_id
ORDER BY r.runner_id;
```

**Corrected result using the timestamps as loaded in the uploaded SQL:** Runner 1 = 14.00 minutes; Runner 2 = 19.67 minutes; Runner 3 = 10.00 minutes.

**Why this differs from the uploaded output:** the original join weights orders with multiple pizza rows more heavily. The case wording also calls this the time until pickup, which includes kitchen preparation and waiting; it is not a kitchen-only preparation-time measure.

**Interpretation:** Runner 2's successful orders have the longest average order-to-pickup interval in this sample. Investigate order mix, restaurant wait time, dispatch timing, and the date mismatch before attributing this to the runner.

## B3. Is there a relationship between the number of pizzas and how long the order takes to prepare?

Because actual kitchen start/finish timestamps are not available, this query measures **order-to-pickup time** as a proxy.

```sql
WITH order_level AS (
    SELECT
        c.order_id,
        COUNT(*) AS pizza_count,
        MIN(c.order_time) AS order_time,
        MAX(r.pickup_time) AS pickup_time
    FROM customer_orders_clean AS c
    JOIN runner_orders_clean AS r
        ON c.order_id = r.order_id
    WHERE r.cancellation = ''
      AND r.distance > 0
      AND r.duration > 0
    GROUP BY c.order_id
)
SELECT
    pizza_count,
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, order_time, pickup_time)), 2)
        AS avg_order_to_pickup_minutes
FROM order_level
GROUP BY pizza_count
ORDER BY pizza_count;
```

**Reported result:** 1 pizza = 12 minutes; 2 pizzas = 18 minutes; 3 pizzas = 29 minutes.

**Interpretation:** Larger orders are associated with longer order-to-pickup times in this sample. A three-pizza order takes 17 minutes longer on average than a single-pizza order. Further timestamps are needed to separate kitchen preparation from queueing and runner waiting.

## B4. What was the average distance travelled for each customer?

Calculate the average across unique deliveries, rather than allowing one delivery to be counted once per pizza in the order.

```sql
WITH delivery_per_order AS (
    SELECT
        c.customer_id,
        c.order_id,
        MAX(r.distance) AS distance_km
    FROM customer_orders_clean AS c
    JOIN runner_orders_clean AS r
        ON c.order_id = r.order_id
    WHERE r.cancellation = ''
      AND r.distance > 0
      AND r.duration > 0
    GROUP BY c.customer_id, c.order_id
)
SELECT
    customer_id,
    ROUND(AVG(distance_km), 2) AS avg_delivery_distance_km
FROM delivery_per_order
GROUP BY customer_id
ORDER BY customer_id;
```

**Corrected result from the supplied rows:** Customer 101 = 20.00 km; 102 = 18.40 km; 103 = 23.40 km; 104 = 10.00 km; 105 = 25.00 km.

**Why this differs from the uploaded output:** Customer 102's 13.4 km delivery contains two pizza lines, so the original join counts that distance twice. Averaging unique deliveries gives 18.4 km instead of 16.73 km.

**Interpretation:** Customer 105 has the longest average delivery distance in this sample, but has only one successful delivery. Distance comparisons require more observations per customer.

## B5. What was the difference between the longest and shortest delivery times for all orders?

```sql
SELECT
    MAX(duration) AS longest_delivery_minutes,
    MIN(duration) AS shortest_delivery_minutes,
    MAX(duration) - MIN(duration) AS delivery_time_range_minutes
FROM runner_orders_clean
WHERE cancellation = ''
  AND distance > 0
  AND duration > 0;
```

**Reported result:** Longest = 40 minutes; shortest = 10 minutes; range = 30 minutes.

**Interpretation:** Delivery duration varies by 30 minutes. Distance, route conditions, time of day, and data-quality issues should be examined before concluding why the variation occurs.

## B6. What was the average speed for each runner for each delivery, and is there a trend?

```sql
SELECT
    runner_id,
    order_id,
    distance,
    duration,
    ROUND(distance / (duration / 60.0), 2) AS average_speed_kmh
FROM runner_orders_clean
WHERE cancellation = ''
  AND distance > 0
  AND duration > 0
ORDER BY runner_id, order_id;
```

**Reported speed values (approximately):**

| Runner | Order | Speed (km/h) |
|---:|---:|---:|
| 1 | 1 | 37.50 |
| 1 | 2 | 44.44 |
| 1 | 3 | 40.20 |
| 1 | 10 | 60.00 |
| 2 | 4 | 35.10 |
| 2 | 7 | 60.00 |
| 2 | 8 | 93.60 |
| 3 | 5 | 40.00 |

**Interpretation:** The calculated speeds vary widely. The 93.6 km/h observation is unusually high for a delivery runner and should be validated against recorded distance, duration, and units. Speed alone is not an appropriate measure of delivery quality or safety.

## B7. What is the successful delivery percentage for each runner?

```sql
SELECT
    runner_id,
    COUNT(*) AS assigned_orders,
    SUM(CASE
        WHEN cancellation = '' AND distance > 0 AND duration > 0 THEN 1
        ELSE 0
    END) AS successful_orders,
    ROUND(
        100.0 * SUM(CASE
            WHEN cancellation = '' AND distance > 0 AND duration > 0 THEN 1
            ELSE 0
        END) / COUNT(*),
        2
    ) AS success_rate_pct
FROM runner_orders_clean
GROUP BY runner_id
ORDER BY runner_id;
```

**Reported result:** Runner 1 = 100%; Runner 2 = 75%; Runner 3 = 50%.

**Interpretation:** Runner 1 has the highest observed success rate, but the sample sizes are only 4, 4, and 2 assigned orders, respectively. Do not use these small denominators to make strong decisions about performance.

---

# C. Ingredient Optimisation

## C1. What are the standard ingredients for each pizza?

```sql
SELECT
    pn.pizza_name,
    GROUP_CONCAT(
        pt.topping_name
        ORDER BY pt.topping_name
        SEPARATOR ', '
    ) AS standard_ingredients
FROM pizza_recipe_toppings_clean AS pr
JOIN pizza_names AS pn
    ON pr.pizza_id = pn.pizza_id
JOIN pizza_toppings AS pt
    ON pr.topping_id = pt.topping_id
GROUP BY pn.pizza_id, pn.pizza_name
ORDER BY pn.pizza_id;
```

**Standard ingredients from the recipe data:**

- **Meat Lovers:** Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami.
- **Vegetarian:** Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce.

**Interpretation:** The recipe table provides a baseline for production planning. Actual usage differs when customers exclude ingredients or add extras, so purchasing estimates should use delivered pizzas and their customisations rather than recipes alone.

## C2. What was the most commonly added extra?

This question examines all order records, including cancelled orders, because the original question refers to the customer orders table. Filter to successful deliveries only if the business question is specifically about delivered pizzas.

```sql
SELECT
    pt.topping_name,
    COUNT(*) AS times_added
FROM customer_order_extras_clean AS e
JOIN pizza_toppings AS pt
    ON e.topping_id = pt.topping_id
GROUP BY pt.topping_id, pt.topping_name
ORDER BY times_added DESC, pt.topping_name;
```

**Reported result:** Bacon = 4; Chicken = 1; Cheese = 1.

**Interpretation:** Bacon is the most frequently recorded extra. Pizza Runner could test an add-on prompt for bacon, then measure conversion and contribution margin before making it a permanent promotion.

## C3. What was the most common exclusion?

```sql
SELECT
    pt.topping_name,
    COUNT(*) AS times_excluded
FROM customer_order_exclusions_clean AS e
JOIN pizza_toppings AS pt
    ON e.topping_id = pt.topping_id
GROUP BY pt.topping_id, pt.topping_name
ORDER BY times_excluded DESC, pt.topping_name;
```

**Corrected result from the supplied order rows:** Cheese = 4; BBQ Sauce = 1; Mushrooms = 1.

**Correction to uploaded SQL:** the original recursive query starts from the `extras` column and also uses it for the remaining string. It should parse the `exclusions` column consistently. The corrected query above uses the pre-cleaned exclusion tokens.

**Interpretation:** Cheese is the most commonly excluded ingredient in the recorded orders. The result may inform customer research or menu labelling, but four occurrences in a small sample do not establish a general preference.

## C4. Generate a descriptive order item for each record in `customer_orders`

Examples include `Meat Lovers`, `Meat Lovers - Exclude Beef`, `Meat Lovers - Extra Bacon`, or a line describing both exclusions and extras.

```sql
WITH exclusion_names AS (
    SELECT
        e.item_id,
        GROUP_CONCAT(pt.topping_name ORDER BY pt.topping_name SEPARATOR ', ')
            AS excluded_ingredients
    FROM customer_order_exclusions_clean AS e
    JOIN pizza_toppings AS pt
        ON e.topping_id = pt.topping_id
    GROUP BY e.item_id
),
extra_names AS (
    SELECT
        e.item_id,
        GROUP_CONCAT(pt.topping_name ORDER BY pt.topping_name SEPARATOR ', ')
            AS extra_ingredients
    FROM customer_order_extras_clean AS e
    JOIN pizza_toppings AS pt
        ON e.topping_id = pt.topping_id
    GROUP BY e.item_id
)
SELECT
    oi.item_id,
    oi.order_id,
    oi.customer_id,
    CONCAT(
        pn.pizza_name,
        CASE
            WHEN en.excluded_ingredients IS NOT NULL
            THEN CONCAT(' - Exclude ', en.excluded_ingredients)
            ELSE ''
        END,
        CASE
            WHEN xn.extra_ingredients IS NOT NULL
            THEN CONCAT(' - Extra ', xn.extra_ingredients)
            ELSE ''
        END
    ) AS order_item_description
FROM customer_order_items_clean AS oi
JOIN pizza_names AS pn
    ON oi.pizza_id = pn.pizza_id
LEFT JOIN exclusion_names AS en
    ON oi.item_id = en.item_id
LEFT JOIN extra_names AS xn
    ON oi.item_id = xn.item_id
ORDER BY oi.order_id, oi.item_id;
```

**Interpretation:** This creates a readable order description that can be used for order confirmation, kitchen tickets, and quality checks. The `item_id` is an analysis key created for this query; in a production system, each pizza line should have a persistent `order_item_id`.

## C5. Generate an alphabetically ordered ingredient list for each pizza order, marking doubled ingredients with `2x`

This query starts with standard recipe ingredients, removes exclusions, adds extras, then aggregates the final ingredient quantities per pizza line.

```sql
WITH ingredient_rows AS (
    -- Keep standard recipe ingredients unless they were excluded.
    SELECT
        oi.item_id,
        pr.topping_id
    FROM customer_order_items_clean AS oi
    JOIN pizza_recipe_toppings_clean AS pr
        ON oi.pizza_id = pr.pizza_id
    WHERE NOT EXISTS (
        SELECT 1
        FROM customer_order_exclusions_clean AS ex
        WHERE ex.item_id = oi.item_id
          AND ex.topping_id = pr.topping_id
    )

    UNION ALL

    -- Add each requested extra as another ingredient occurrence.
    SELECT
        e.item_id,
        e.topping_id
    FROM customer_order_extras_clean AS e
),
ingredient_counts AS (
    SELECT
        item_id,
        topping_id,
        COUNT(*) AS ingredient_quantity
    FROM ingredient_rows
    GROUP BY item_id, topping_id
)
SELECT
    oi.order_id,
    oi.customer_id,
    CONCAT(
        pn.pizza_name,
        ': ',
        GROUP_CONCAT(
            CASE
                WHEN ic.ingredient_quantity > 1
                THEN CONCAT(ic.ingredient_quantity, 'x', pt.topping_name)
                ELSE pt.topping_name
            END
            ORDER BY pt.topping_name
            SEPARATOR ', '
        )
    ) AS final_ingredient_list
FROM ingredient_counts AS ic
JOIN customer_order_items_clean AS oi
    ON ic.item_id = oi.item_id
JOIN pizza_names AS pn
    ON oi.pizza_id = pn.pizza_id
JOIN pizza_toppings AS pt
    ON ic.topping_id = pt.topping_id
GROUP BY oi.item_id, oi.order_id, oi.customer_id, pn.pizza_name
ORDER BY oi.order_id, oi.item_id;
```

**Interpretation:** The result is a kitchen-friendly list of final ingredients, with duplicates represented as quantities such as `2xBacon`. This is more reliable for preparation than storing customisation instructions as unparsed text.

## C6. What is the total quantity of each ingredient used in all delivered pizzas, sorted by most frequent first?

```sql
WITH delivered_items AS (
    SELECT oi.*
    FROM customer_order_items_clean AS oi
    WHERE EXISTS (
        SELECT 1
        FROM runner_orders_clean AS r
        WHERE r.order_id = oi.order_id
          AND r.cancellation = ''
          AND r.distance > 0
          AND r.duration > 0
    )
),
ingredient_rows AS (
    SELECT
        oi.item_id,
        pr.topping_id
    FROM delivered_items AS oi
    JOIN pizza_recipe_toppings_clean AS pr
        ON oi.pizza_id = pr.pizza_id
    WHERE NOT EXISTS (
        SELECT 1
        FROM customer_order_exclusions_clean AS ex
        WHERE ex.item_id = oi.item_id
          AND ex.topping_id = pr.topping_id
    )

    UNION ALL

    SELECT
        e.item_id,
        e.topping_id
    FROM customer_order_extras_clean AS e
    JOIN delivered_items AS oi
        ON e.item_id = oi.item_id
)
SELECT
    pt.topping_name,
    COUNT(*) AS total_quantity_used
FROM ingredient_rows AS ir
JOIN pizza_toppings AS pt
    ON ir.topping_id = pt.topping_id
GROUP BY pt.topping_id, pt.topping_name
ORDER BY total_quantity_used DESC, pt.topping_name;
```

**Calculated from the supplied order rows, applying successful-delivery filters and customer modifications:** Bacon = 12; Mushrooms = 11; Cheese = 10; Beef = 9; Chicken = 9; Pepperoni = 9; Salami = 9; BBQ Sauce = 8; Onions = 3; Peppers = 3; Tomatoes = 3; Tomato Sauce = 3.

**Interpretation:** Bacon and mushrooms have the largest calculated usage quantities. This can support inventory planning, but actual purchasing should also account for stock on hand, wastage, supplier lead times, and future demand. Validate the ingredient calculation after resolving the date/source-data issues.

---

# D. Pricing and Ratings

## D1. If Meat Lovers costs $12 and Vegetarian costs $10, with no charge for changes and no delivery fees, how much revenue has Pizza Runner made?

```sql
SELECT
    SUM(CASE
        WHEN c.pizza_id = 1 THEN 12
        WHEN c.pizza_id = 2 THEN 10
        ELSE 0
    END) AS base_pizza_revenue
FROM customer_orders_clean AS c
JOIN runner_orders_clean AS r
    ON c.order_id = r.order_id
WHERE r.cancellation = ''
  AND r.distance > 0
  AND r.duration > 0;
```

**Reported result:** $138, calculated as 9 Meat Lovers × $12 + 3 Vegetarian × $10.

**Interpretation:** $138 is estimated gross pizza revenue under the given prices. It is not profit because ingredient costs, labour, packaging, overhead, and runner payments have not been deducted.

## D2. What if each extra topping costs an additional $1, including cheese?

```sql
WITH delivered_base_revenue AS (
    SELECT
        SUM(CASE
            WHEN oi.pizza_id = 1 THEN 12
            WHEN oi.pizza_id = 2 THEN 10
            ELSE 0
        END) AS base_revenue
    FROM customer_order_items_clean AS oi
    WHERE EXISTS (
        SELECT 1
        FROM runner_orders_clean AS r
        WHERE r.order_id = oi.order_id
          AND r.cancellation = ''
          AND r.distance > 0
          AND r.duration > 0
    )
),
delivered_extra_revenue AS (
    SELECT COUNT(*) AS extra_charge_revenue
    FROM customer_order_extras_clean AS e
    JOIN customer_order_items_clean AS oi
        ON e.item_id = oi.item_id
    WHERE EXISTS (
        SELECT 1
        FROM runner_orders_clean AS r
        WHERE r.order_id = oi.order_id
          AND r.cancellation = ''
          AND r.distance > 0
          AND r.duration > 0
    )
)
SELECT
    b.base_revenue,
    e.extra_charge_revenue,
    b.base_revenue + e.extra_charge_revenue AS revenue_with_extra_charges
FROM delivered_base_revenue AS b
CROSS JOIN delivered_extra_revenue AS e;
```

**Reported result:** $142 total revenue, an increase of $4 from the $138 base. Four extra topping occurrences are recorded on successful deliveries in the supplied sample.

**Interpretation:** Charging $1 per extra raises estimated revenue by about 2.9% in this sample. A business should test the effect on conversion and customer satisfaction and deduct the cost of the additional ingredients before judging whether the policy improves profit.

## D3. Design a table that allows customers to rate their runner from 1 to 5, then insert example ratings for each successful order

The ratings below are **simulated examples for demonstrating SQL table design**. They are not actual customer feedback and should not be presented as measured satisfaction scores.

```sql
DROP TABLE IF EXISTS runner_ratings;

CREATE TABLE runner_ratings (
    order_id INT PRIMARY KEY,
    rating TINYINT NOT NULL,
    feedback VARCHAR(255),
    rated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_runner_rating CHECK (rating BETWEEN 1 AND 5)
);

-- Example / simulated ratings for the 8 successful orders in this dataset.
INSERT INTO runner_ratings (order_id, rating, feedback) VALUES
    (1, 5, 'Simulated sample rating'),
    (2, 4, 'Simulated sample rating'),
    (3, 4, 'Simulated sample rating'),
    (4, 3, 'Simulated sample rating'),
    (5, 5, 'Simulated sample rating'),
    (7, 4, 'Simulated sample rating'),
    (8, 2, 'Simulated sample rating'),
    (10, 5, 'Simulated sample rating');
```

**Design note:** `order_id` is the primary key because the case has one runner assignment per order and one rating per completed order in this simplified design. A production system would normally also capture the customer, runner, submission timestamp, and possibly a review status. A foreign key can be added once the source `runner_orders.order_id` has a declared unique or primary-key constraint.

## D4. Join ratings with successful-delivery data

Required fields: customer ID, order ID, runner ID, rating, order time, pickup time, time between order and pickup, delivery duration, average speed, and total pizzas.

```sql
WITH order_summary AS (
    SELECT
        order_id,
        MIN(customer_id) AS customer_id,
        MIN(order_time) AS order_time,
        COUNT(*) AS total_pizzas
    FROM customer_order_items_clean
    GROUP BY order_id
)
SELECT
    os.customer_id,
    r.order_id,
    r.runner_id,
    rr.rating,
    os.order_time,
    r.pickup_time,
    TIMESTAMPDIFF(MINUTE, os.order_time, r.pickup_time)
        AS order_to_pickup_minutes,
    r.duration AS delivery_duration_minutes,
    ROUND(r.distance / NULLIF(r.duration / 60.0, 0), 2)
        AS average_speed_kmh,
    os.total_pizzas
FROM runner_orders_clean AS r
JOIN order_summary AS os
    ON r.order_id = os.order_id
JOIN runner_ratings AS rr
    ON r.order_id = rr.order_id
WHERE r.cancellation = ''
  AND r.distance > 0
  AND r.duration > 0
ORDER BY r.order_id;
```

**Interpretation:** This query creates a delivery-level reporting table that combines order size, operational timings, delivery speed, and the simulated rating. Once real customer ratings are collected, the same structure can support comparisons between customer experience and operational metrics. The sample simulated ratings cannot be used to infer a real customer satisfaction relationship.

## D5. If Meat Lovers costs $12, Vegetarian costs $10, extras are free, and runners receive $0.30 per kilometre, how much money remains after runner payments?

```sql
WITH base_revenue AS (
    SELECT
        SUM(CASE
            WHEN oi.pizza_id = 1 THEN 12
            WHEN oi.pizza_id = 2 THEN 10
            ELSE 0
        END) AS revenue
    FROM customer_order_items_clean AS oi
    WHERE EXISTS (
        SELECT 1
        FROM runner_orders_clean AS r
        WHERE r.order_id = oi.order_id
          AND r.cancellation = ''
          AND r.distance > 0
          AND r.duration > 0
    )
),
delivery_cost AS (
    SELECT SUM(distance * 0.30) AS runner_payment
    FROM runner_orders_clean
    WHERE cancellation = ''
      AND distance > 0
      AND duration > 0
)
SELECT
    b.revenue AS gross_pizza_revenue,
    ROUND(d.runner_payment, 2) AS runner_payments,
    ROUND(b.revenue - d.runner_payment, 2) AS revenue_after_runner_payments
FROM base_revenue AS b
CROSS JOIN delivery_cost AS d;
```

**Calculated result from the supplied values:**

- Gross pizza revenue: **$138.00**
- Total successful-delivery distance: **145.2 km**
- Runner payments at $0.30/km: **$43.56**
- Revenue remaining after runner payments: **$94.44**

**Interpretation:** $94.44 remains after deducting the assumed runner payments from pizza revenue. This is not net profit because ingredient cost, kitchen labour, packaging, rent, technology, refunds, and other business costs are not included.

---

# E. Bonus Question — Adding a Supreme Pizza

## E1. How would the data design change if Danny added a Supreme pizza with all toppings?

For the current schema, add a row to `pizza_names` and a corresponding comma-separated recipe to `pizza_recipes`. The existing `pizza_toppings` table already contains topping IDs 1 to 12.

```sql
INSERT INTO pizza_names (pizza_id, pizza_name)
VALUES (3, 'Supreme');

INSERT INTO pizza_recipes (pizza_id, toppings)
VALUES (3, '1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12');
```

**Interpretation and design recommendation:** The inserts extend the current design with a new pizza and recipe. However, storing multiple topping IDs in a comma-separated string makes exclusions, extras, validation, and ingredient aggregation harder. A more scalable relational design would use a `pizza_recipe_toppings` bridge table with one row per `(pizza_id, topping_id)` pair.

---

# 5. Consolidated Business Insights and Recommendations

## 5.1 Product demand

**Insight:** Meat Lovers accounts for 9 of 12 successfully delivered pizzas (75%).  
**Recommendation:** Ensure its ingredients are available, but base stocking levels on longer-term demand, spoilage, supplier lead times, and margin rather than this small sample alone.

## 5.2 Order preparation and pickup

**Insight:** Average order-to-pickup time increases from 12 minutes for one pizza to 29 minutes for three pizzas.  
**Recommendation:** Investigate whether larger orders cause a kitchen or dispatch bottleneck. Add timestamps for preparation start, preparation completion, and runner arrival to separate the stages.

## 5.3 Runner operations

**Insight:** The reported success rates range from 50% to 100%, while delivery time and calculated speed also vary.  
**Recommendation:** Track performance over a larger sample and compare deliveries of similar distance and order size. Record cancellation reasons and customer ratings; do not rank runners solely by the tiny sample or estimated speed.

## 5.4 Menu customisation

**Insight:** Bacon is the most frequently added extra in the supplied orders, and cheese is the most frequently recorded exclusion after correcting the exclusions query.  
**Recommendation:** Test add-on prompts and investigate exclusion patterns, but evaluate conversion, ingredient costs, waste, and customer response before making menu changes.

## 5.5 Revenue and cost

**Insight:** Base revenue is $138 under the stated prices, and the $1-per-extra scenario produces $142. Deducting $43.56 of runner payments leaves $94.44 before all other operating costs.  
**Recommendation:** Build a contribution-margin view that includes ingredient, packaging, labour, and delivery costs. Treat gross revenue and revenue after runner payments as intermediate calculations—not final profit.

---

# 6. Query Corrections and Limitations

| Area | Issue in the uploaded work | Correction or limitation |
|---|---|---|
| Data dates | Customer order dates in the uploaded SQL are in 2020, while the case page shows 2021 | Verify and correct the source dates consistently before using weekday or elapsed-time results |
| A5 | The original query counts records linked to cancelled deliveries | Add the successful-delivery filter when the question asks for delivered pizzas |
| A10 | The original query counts pizza lines but labels the metric volume of orders | Report both unique order IDs and pizza rows, and validate the years used for weekday names |
| B1 | The original query groups pickup dates rather than runner registration dates | Use `runners.registration_date` and fixed seven-day periods from 2021-01-01 |
| B2 | Joining to every pizza line gives multiple-pizza orders extra weight | Calculate elapsed time once per unique order before averaging by runner |
| B3 | Order-to-pickup time is described as preparation time | Label it as a proxy unless actual kitchen preparation timestamps are available |
| B4 | Orders with multiple pizzas repeat the same delivery distance | Average distance once per delivery/order |
| C3 | The original recursive query parses `extras` while attempting to calculate exclusions | Parse `exclusions` consistently; the provided helper table does this |
| C4–C6 | The uploaded SQL includes the questions but does not provide completed solutions | Complete recursive-token-based solutions are included above |
| D3 | The original approach derives ratings from speed | Customer ratings should be collected from customers; any example data must be labelled simulated |
| D5 | The uploaded SQL lists the question but not a final profit calculation | The solution calculates pizza revenue less the stated runner payment only |
| Ingredient schema | Toppings and customisations are stored as comma-separated strings | A normalised bridge table per topping occurrence would be more maintainable |

## 7. Recommended next steps

1. Correct and validate the source date values, then rerun all queries affected by date and time.
2. Save the setup and cleaning statements in `sql/00_setup_and_cleaning.sql`, and the question solutions in clearly labelled SQL files.
3. Add screenshots or small charts for product mix, order-to-pickup time by pizza count, delivery success rate, and ingredient usage.
4. Add more weeks of data and actual cost/customer feedback fields before drawing stronger commercial conclusions.

## Conclusion

This project demonstrates how SQL can turn operational records into useful measures of product demand, order handling, delivery performance, ingredient usage, and revenue. The strongest analysis combines correct aggregation grain, consistent data cleaning, transparent assumptions, and practical recommendations. The results here are exploratory because the dataset is small and some timestamps in the supplied SQL require validation.

## Reference

- [Danny Ma — 8 Week SQL Challenge, Case Study #2: Pizza Runner](https://8weeksqlchallenge.com/case-study-2/)
