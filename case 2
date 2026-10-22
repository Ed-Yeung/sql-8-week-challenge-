# Case Study #2: Pizza Runner — SQL Business Analysis

## Project Overview

This project analyses Pizza Runner's order, menu, runner and delivery data using **MySQL**. The objective is to turn raw operational records into useful business insights about product demand, order preparation, delivery performance, menu customisation and revenue.

**Case study:** [8 Week SQL Challenge — Case Study #2: Pizza Runner](https://8weeksqlchallenge.com/case-study-2/)

**Tools:** MySQL, SQL, Common Table Expressions (CTEs), recursive CTEs, joins, aggregate functions, conditional logic, date/time functions and data cleaning.

## Business Questions

The analysis covers four main areas:

1. **Pizza Metrics** — order volumes, pizza mix, customisation and order timing.
2. **Runner and Customer Experience** — runner activity, order-to-pickup time, delivery distance, delivery duration and delivery success.
3. **Ingredient Optimisation** — recipe ingredients, extra toppings and exclusions.
4. **Pricing and Ratings** — estimated pizza revenue, extra-charge scenarios and a proposed runner-rating approach.

## Dataset and Data Preparation

The supplied SQL setup creates tables for runners, customer orders, runner orders, pizza names, pizza recipes and pizza toppings. The raw data contains inconsistent representations of missing values and numeric fields stored as text, so the analysis creates cleaned temporary tables before querying.

Examples of cleaning include:

- Standardising `NULL`, the text value `'null'`, and blank values for exclusions, extras and cancellation fields.
- Removing units such as `km` from distance values.
- Converting delivery duration text such as `32 minutes` into a numeric duration.
- Converting pickup timestamps into a `DATETIME` column.

The cleaned temporary tables must be created in the same MySQL session in which the analysis queries are run, because temporary tables do not persist across sessions.

### Data-quality note

The uploaded setup SQL appears to contain a **year mismatch**: runner registration dates are in January 2021, while order and pickup timestamps are in January 2020. This should be verified against the original case data before interpreting trends by date or comparing runner sign-ups with delivery activity. The insights below report the outputs from the current analysis where noted, and flag findings that need rechecking.

## Executive Summary

| Metric | Result from the current analysis |
|---|---:|
| Pizzas ordered | 14 |
| Unique customer orders | 10 |
| Successful delivery orders | 8 |
| Pizzas in successful deliveries | 12 |
| Meat Lovers pizzas delivered | 9 (75% of delivered pizzas) |
| Vegetarian pizzas delivered | 3 (25% of delivered pizzas) |
| Base pizza revenue at the stated prices | $138 |
| Revenue in the $1-per-extra scenario | $142 |
| Longest delivery duration | 40 minutes |
| Shortest delivery duration | 10 minutes |

The dataset suggests that Meat Lovers is the dominant product in this small sample, order-to-pickup time tends to be longer for larger orders, and there may be opportunities to improve the measurement of delivery operations and customisation. The sample is too small to support strong conclusions about long-term customer behaviour or runner performance.

## Key Insights and Business Recommendations

### 1. Meat Lovers is the leading product in the sample

Of the 12 pizzas included in successful deliveries, 9 were Meat Lovers and 3 were Vegetarian.

| Pizza type | Delivered pizzas | Share |
|---|---:|---:|
| Meat Lovers | 9 | 75% |
| Vegetarian | 3 | 25% |
| **Total** | **12** | **100%** |

**Interpretation:** Meat Lovers accounts for three-quarters of delivered pizza volume in this sample. At the assumed prices of $12 for Meat Lovers and $10 for Vegetarian, it also contributes more base revenue.

**Recommendations:**

- Monitor Meat Lovers ingredient availability to reduce the risk of stockouts.
- Review demand over a longer period before changing inventory levels.
- If Pizza Runner wants to grow Vegetarian sales, test a targeted promotion and measure its effect on orders and contribution margin.
- Compare profitability as well as sales volume; ingredient costs and waste are not included in this dataset.

### 2. Larger orders are associated with longer order-to-pickup times

The current query reports the following average elapsed times between order placement and runner pickup:

| Pizzas in an order | Average order-to-pickup time |
|---:|---:|
| 1 | 12 minutes |
| 2 | 18 minutes |
| 3 | 29 minutes |

**Interpretation:** The observed average rises from 12 minutes for one-pizza orders to 29 minutes for three-pizza orders. Larger orders may require more preparation or coordination.

**Important distinction:** The query measures **order-to-pickup time**, not kitchen preparation time alone. The elapsed time may include queueing, cooking, packing and runner waiting time.

**Recommendations:**

- Track the relationship between order size and order-to-pickup time over a larger sample.
- Capture separate timestamps for order acceptance, cooking start, cooking completion and runner pickup.
- If the pattern continues, investigate kitchen workflow, batching and dispatch timing before making staffing changes.

### 3. Delivery performance varies, but runner rankings would be premature

The current outputs show the following runner metrics:

| Runner | Successful deliveries | Reported success rate | Average order-to-pickup time from the current query |
|---|---:|---:|---:|
| 1 | 4 | 100% | 15.3 minutes |
| 2 | 3 | 75% | 23.4 minutes |
| 3 | 1 | 50% | 10.0 minutes |

**Query limitation:** The current average-time query joins runner orders to the pizza-level `customer_orders_clean` table. Orders containing multiple pizzas therefore appear multiple times, unintentionally weighting the average by the number of pizzas in each order. Calculating one elapsed time per distinct order first gives approximate order-level averages of **14.0 minutes for Runner 1, 19.7 minutes for Runner 2, and 10.0 minutes for Runner 3** using the supplied timestamps and `TIMESTAMPDIFF(MINUTE, ...)`. These corrected figures should still be verified after resolving the dataset's year mismatch.

**Interpretation:** Runner 1 has the highest reported success rate, while Runner 3 has the shortest average order-to-pickup time. Runner 2's order-level average remains the highest in this small sample. These differences should **not** be treated as definitive evidence that one runner is better. The numbers of deliveries are very small, and the metrics may be affected by order size, delivery distance, pickup delays, traffic and the assignment of cancelled orders.

A safer calculation pattern is to aggregate `customer_orders_clean` to one row per order before joining it to the runner table:

```sql
WITH order_times AS (
    SELECT order_id, MIN(order_time) AS order_time
    FROM customer_orders_clean
    GROUP BY order_id
),
successful_order_times AS (
    SELECT
        r.runner_id,
        r.order_id,
        TIMESTAMPDIFF(MINUTE, ot.order_time, r.pickup_time)
            AS order_to_pickup_minutes
    FROM runner_orders_clean AS r
    INNER JOIN order_times AS ot
        ON r.order_id = ot.order_id
    WHERE r.cancellation = ''
      AND r.distance > 0
      AND r.duration > 0
)
SELECT
    runner_id,
    ROUND(AVG(order_to_pickup_minutes), 2) AS avg_order_to_pickup_minutes
FROM successful_order_times
GROUP BY runner_id
ORDER BY runner_id;
```

**Recommendations:**

- Monitor performance over a larger number of orders.
- Compare delivery duration and speed for similar distances and conditions.
- Record cancellation reasons, restaurant waiting time and actual customer feedback.
- Investigate the source of delays before changing runner assignments or using the results to evaluate individual performance.

### 4. Delivery duration and calculated speed should be validated

The current results show a shortest delivery duration of 10 minutes and a longest duration of 40 minutes. The calculated delivery speeds range from approximately 35.1 to 93.6 km/h.

**Interpretation:** The range suggests that delivery experiences vary. However, the highest calculated speed is unusually large for a local delivery service and should be validated before being used as a performance measure.

**Recommendations:**

- Check distance and duration values for data-entry or parsing issues.
- Confirm that all distance values are in kilometres and all duration values are in minutes.
- Compare actual delivery duration with distance bands and time of day.
- Evaluate on-time delivery and customer satisfaction alongside speed; faster is not necessarily better if safety or order accuracy suffers.

### 5. Customisation patterns differ by customer

The current delivered-pizza output reports the following counts:

| Customer | Pizzas with at least one change | Pizzas without changes |
|---|---:|---:|
| 101 | 0 | 2 |
| 102 | 0 | 3 |
| 103 | 3 | 0 |
| 104 | 2 | 1 |
| 105 | 1 | 0 |

**Interpretation:** Customers 101 and 102 had no recorded changes on the delivered pizzas in the output, while every delivered pizza attributed to Customer 103 had a recorded change. This may reflect different preferences, but the dataset is too small to establish stable customer habits.

**Recommendations:**

- Make exclusions and extras easy to record accurately.
- Measure whether customised orders take longer to prepare or have more order errors.
- Consider personalised offers only after repeat behaviour is observed over a longer period.
- Do not assume that customers who customise orders spend more or are more loyal without additional evidence.

### 6. Bacon is the most frequently recorded extra in the current output

The extra-topping query reports bacon four times, with chicken and cheese each appearing once. The query includes all order records and does not restrict the count to successful deliveries.

**Interpretation:** Bacon is a potential add-on opportunity, but its reported count includes cancelled-order records. It should not be used as a delivered-sales or inventory figure without recalculating it against the intended order population.

**Recommendations:**

- Re-run the topping analysis separately for all orders and successful deliveries.
- Test a bacon add-on prompt at checkout and measure attachment rate and contribution margin.
- Track the revenue and ingredient cost of each extra separately.
- Correct the exclusions query before drawing conclusions about the most commonly excluded topping; its current recursive logic references the `extras` column where it should parse `exclusions`.

### 7. Revenue increases modestly under the extra-charge scenario

With the assumed fixed prices of $12 for Meat Lovers and $10 for Vegetarian, the current query reports:

| Pricing scenario | Estimated revenue |
|---|---:|
| Fixed pizza prices; no extra charges | $138 |
| Add $1 per extra topping | $142 |
| **Difference** | **+$4** |

**Interpretation:** Charging $1 for each extra topping increases estimated revenue by $4, or approximately 2.9% of the $138 base figure, under the assumptions used in the query.

**Recommendations:**

- Test an extra-topping charge and monitor conversion, order value and customer response.
- Compare incremental revenue with ingredient costs.
- Deduct runner payments and other operating costs before describing the result as profit.

The $138 and $142 figures are revenue estimates, not profit. The requested runner-payment calculation ($0.30 per kilometre) still needs to be completed to estimate the amount remaining after delivery payments.

## SQL Validation and Improvements

The results are useful for exploration, but the following issues should be fixed or clearly documented before the project is treated as final:

| Area | Issue to address | Recommended action |
|---|---|---|
| Date consistency | The setup SQL has orders in 2020 and runner registrations in 2021 | Verify and correct the year mismatch against the original case data before time-based analysis |
| Runner sign-ups | The current query groups pickup timestamps, which measures delivery activity rather than sign-ups | Count runners by `runners.registration_date` and define each seven-day period from the requested start date |
| Order-to-pickup metric | The query labels elapsed order-to-pickup time as preparation time; its runner-level average also weights multi-pizza orders more heavily | Rename the metric and aggregate to one row per order before averaging; obtain actual kitchen timestamps to measure preparation time directly |
| Exclusions | The recursive query uses `extras` in its remainder calculation and filter | Parse the `exclusions` column throughout the query and validate the counts |
| Extra topping frequency | The current count includes cancelled-order records | Specify whether the analysis covers all orders or successful deliveries and filter consistently |
| Simulated runner ratings | Ratings are inferred from calculated speed rather than collected from customers | Label them as simulated values, or create a table for real customer-submitted ratings |
| Profit calculation | Base revenue and extra-charge revenue do not deduct runner payments or food costs | Calculate runner kilometres and payments, then include available costs before discussing profit |
| Incomplete questions | The current SQL file contains prompts but no completed queries for some ingredient-formatting/ingredient-quantity questions and the final post-runner-payment calculation | Complete these analyses or list them explicitly as future work |

## Suggested Next Steps

1. **Validate the source data** — resolve the year mismatch and confirm the cleaning rules.
2. **Correct the SQL logic** — particularly the signup-week and exclusions queries, and consistently define whether each metric refers to orders placed or successfully delivered.
3. **Complete the financial analysis** — calculate runner payments using the delivery distance and $0.30 per-kilometre assumption, then estimate the amount remaining from revenue.
4. **Expand the dataset** — use more weeks of transactions to validate product demand, customisation patterns and operational trends.
5. **Add better operational data** — capture kitchen preparation stages, cancellation reasons, on-time delivery and genuine customer ratings.

## Conclusion

This case demonstrates how SQL can transform raw order and delivery records into operational metrics and preliminary business recommendations. The initial results point to strong demand for Meat Lovers pizza, a potential association between larger orders and longer order-to-pickup times, opportunities to test add-on pricing, and a need for more reliable delivery-performance measurement.

The most important next step is data and query validation. Because the sample is small and the supplied setup contains a possible date inconsistency, the findings should be treated as exploratory rather than definitive. A stronger analysis would validate the source data, complete the remaining questions, and evaluate revenue alongside runner payments and ingredient costs.

---

*Project completed as part of the [8 Week SQL Challenge](https://8weeksqlchallenge.com/case-study-2/). All business recommendations are based on the current query outputs and should be revalidated after correcting the data-quality and SQL issues identified above.*
