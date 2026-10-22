# 8 Week SQL Challenge — Case Study #5: Data Mart

> **Project type:** Sales performance analysis · Data cleaning · Before-and-after analysis  
> **SQL dialect:** MySQL 8.0  
> **Source data:** `data_mart.weekly_sales` from the supplied `Case 5.sql` file

## 1. Business Problem

Data Mart is an international online supermarket specialising in fresh produce. In June 2020, it introduced sustainable packaging across its supply chain, from farms to customers. The business wants to understand whether sales changed after the rollout and which business areas were most affected.

This is not only a reporting task. The analysis should turn weekly sales data into evidence that helps the business decide how to roll out future sustainability initiatives while managing commercial risk.

### Key business questions

1. How did sales change after the sustainable-packaging rollout?
2. Were the changes concentrated in particular regions, platforms, age bands, demographics, or customer types?
3. What should Data Mart monitor or change before introducing similar initiatives in the future?

**Important interpretation limit:** a before-and-after comparison can identify a change that occurred around the rollout, but it cannot prove that packaging alone caused the change. Seasonality, market conditions, promotions, customer behaviour, or other 2020 events may also have contributed.

## 2. Analysis Goals

- Prepare reliable date, week, month, year, segment, and transaction-size fields.
- Validate the time coverage before comparing periods.
- Establish a sales and transaction baseline by year, region, channel, and customer group.
- Compare four-week and twelve-week periods before and after 15 June 2020.
- Identify where the largest absolute and percentage sales changes occurred.
- Translate SQL outputs into practical business recommendations without overstating causation.

## 3. Dataset and Data Dictionary

The supplied SQL file contains **17,116 weekly aggregated records** in `weekly_sales`. The data is already aggregated by combinations of week, region, platform, segment, and customer type; each row is not an individual customer or order.

| Column | Meaning |
|---|---|
| `week_date` | Date label for the weekly sales record. In the supplied SQL, it is stored as a string in `day/month/two-digit-year` format. |
| `region` | Business region, including Africa, Asia, Canada, Europe, Oceania, South America, and USA. |
| `platform` | Sales channel: `Retail` or `Shopify`. |
| `segment` | Coded customer segment such as `C1` or `F3`; the literal string `'null'` represents an unclassified segment in the supplied data. |
| `customer_type` | `New`, `Existing`, or `Guest`. |
| `transactions` | Number of transactions represented by that aggregated row. |
| `sales` | Sales amount represented by that aggregated row, in the units stored in the data. |

### Derived fields

| Field | Definition |
|---|---|
| `week_number` | ISO week number, calculated with `WEEK(week_date, 3)`. This makes 15 June 2020 week 25, consistent with the before-and-after period definition used below. |
| `month_number` | Calendar month number from 1 to 12. |
| `calendar_year` | Calendar year extracted from the correctly parsed date. |
| `age_band` | Segment suffix `1` → Young Adults; `2` → Middle Aged; `3` or `4` → Retirees; unclassified segment → unknown. |
| `demographic` | Segment prefix `C` → Couples; `F` → Families; unclassified segment → unknown. |
| `avg_transaction` | `sales / transactions` for an individual aggregated row, rounded to two decimal places. It is **not** by itself the correct value to average across rows to obtain an overall transaction size. |

## 4. Data Cleaning and Validation

### Why the original date conversion needs attention

The supplied SQL changes `week_date` from `VARCHAR` directly to `DATE`. Because the strings are stored as `day/month/year`, an implicit conversion can interpret the date incorrectly. The accompanying Word output contains values such as `2031-08-20` for an input string such as `31/8/20`; that is a conversion error, not a valid business date.

The following version explicitly parses the date first and then derives all fields. It also normalises the literal string `'null'` to `'unknown'` and guards against division by zero.

```sql
USE data_mart;

DROP TABLE IF EXISTS clean_weekly_sales;

CREATE TABLE clean_weekly_sales AS
SELECT
    STR_TO_DATE(week_date, '%e/%c/%y') AS week_date,
    region,
    platform,
    CASE
        WHEN LOWER(segment) = 'null' THEN 'unknown'
        ELSE segment
    END AS segment,
    customer_type,
    transactions,
    sales
FROM weekly_sales;

ALTER TABLE clean_weekly_sales
    ADD COLUMN week_number INT AFTER week_date,
    ADD COLUMN month_number INT AFTER week_number,
    ADD COLUMN calendar_year INT AFTER month_number,
    ADD COLUMN age_band VARCHAR(20) AFTER segment,
    ADD COLUMN demographic VARCHAR(20) AFTER age_band,
    ADD COLUMN avg_transaction DECIMAL(14,2) AFTER sales;

UPDATE clean_weekly_sales
SET
    week_number = WEEK(week_date, 3),
    month_number = MONTH(week_date),
    calendar_year = YEAR(week_date),
    age_band = CASE
        WHEN segment = 'unknown' THEN 'unknown'
        WHEN SUBSTRING(segment, 2, 1) = '1' THEN 'Young Adults'
        WHEN SUBSTRING(segment, 2, 1) = '2' THEN 'Middle Aged'
        WHEN SUBSTRING(segment, 2, 1) IN ('3', '4') THEN 'Retirees'
        ELSE 'unknown'
    END,
    demographic = CASE
        WHEN segment = 'unknown' THEN 'unknown'
        WHEN SUBSTRING(segment, 1, 1) = 'C' THEN 'Couples'
        WHEN SUBSTRING(segment, 1, 1) = 'F' THEN 'Families'
        ELSE 'unknown'
    END,
    avg_transaction = ROUND(sales / NULLIF(transactions, 0), 2);
```

### Validation checks

```sql
-- Confirm date parsing and the time span of the supplied file.
SELECT
    MIN(week_date) AS first_week,
    MAX(week_date) AS last_week,
    COUNT(DISTINCT week_date) AS distinct_week_dates,
    COUNT(*) AS row_count
FROM clean_weekly_sales;

-- Inspect year-level weekly coverage.
SELECT
    calendar_year,
    MIN(week_number) AS first_week_number,
    MAX(week_number) AS last_week_number,
    COUNT(DISTINCT week_date) AS observed_weeks
FROM clean_weekly_sales
GROUP BY calendar_year
ORDER BY calendar_year;

-- Check for unparsed dates and zero/null transaction counts.
SELECT
    SUM(week_date IS NULL) AS null_dates,
    SUM(transactions IS NULL) AS null_transactions,
    SUM(transactions = 0) AS zero_transaction_rows
FROM clean_weekly_sales;
```

**Coverage note:** after explicit parsing, the supplied file contains 24 weekly dates per year, with ISO week numbers 13–36. There are no missing weekly dates within that supplied window, but the file is not a complete 52/53-week calendar year. Weekday output from the supplied dates is Monday. This differs from examples on the public challenge page, so the results below are deliberately described as calculations from the **uploaded SQL dataset**, rather than assumed to match every public copy of the data.

## 5. Data Exploration

### Q1. What day of the week is used for each `week_date` value?

**Business goal:** confirm the grain and timing of the weekly records before using date-based comparisons.

```sql
SELECT DISTINCT DAYNAME(week_date) AS week_day
FROM clean_weekly_sales;
```

**Result from the supplied SQL file:** Monday.

**Business interpretation:** the date should be treated consistently as the weekly date label for this supplied extract. Explicit date parsing is essential before deriving weekday or week number; otherwise the time analysis can be shifted or corrupted.

### Q2. What range of week numbers is missing from the dataset?

**Business goal:** distinguish genuine gaps in the observed time series from weeks that simply fall outside the extract's coverage.

```sql
SELECT
    calendar_year,
    MIN(week_number) AS first_week_number,
    MAX(week_number) AS last_week_number,
    COUNT(DISTINCT week_number) AS observed_weeks
FROM clean_weekly_sales
GROUP BY calendar_year
ORDER BY calendar_year;
```

**Result:** each year in the supplied file has 24 observed weekly records, covering ISO weeks 13–36. There are no gaps inside that range. Weeks outside 13–36 are not present in this extract; they should not automatically be described as missing records unless full-year coverage was expected.

**Why this matters:** a trend analysis should compare periods that actually exist in the data. Treating the extract as a full calendar year would lead to an incorrect completeness assumption.

### Q3. How many total transactions were there for each year?

**Business goal:** measure yearly transaction volume alongside sales.

```sql
SELECT
    calendar_year,
    SUM(transactions) AS total_transactions,
    SUM(sales) AS total_sales
FROM clean_weekly_sales
GROUP BY calendar_year
ORDER BY calendar_year;
```

| Year | Transactions | Sales amount |
|---:|---:|---:|
| 2018 | 346,187,944 | 12,885,297,352 |
| 2019 | 365,639,285 | 13,746,032,500 |
| 2020 | 375,813,651 | 14,100,220,900 |

**SQL lesson:** `COUNT(*)` counts aggregated records, not the number of transactions represented by those records. The correct transaction metric is `SUM(transactions)`.

**Business insight:** within the 24-week windows supplied for each year, sales increased by approximately **9.43%** from 2018 to 2020, while transaction volume increased by approximately **8.56%**. This provides useful context for assessing the 2020 before-and-after comparison.

### Q4. What is the total sales for each region for each month?

**Business goal:** identify the regions contributing the most sales and support regional planning.

```sql
SELECT
    calendar_year,
    month_number,
    region,
    SUM(sales) AS total_sales
FROM clean_weekly_sales
GROUP BY calendar_year, month_number, region
ORDER BY calendar_year, month_number, total_sales DESC;
```

Including `calendar_year` prevents March 2018, March 2019, and March 2020 from being silently combined. If a pooled month-of-year view is specifically required, remove `calendar_year` from the selection and grouping.

**Overall result across the supplied data:** Oceania has the largest cumulative sales amount (13,338,831,126), followed by Africa (9,928,409,594) and Asia (9,165,169,960). Europe has the smallest total (672,561,457).

**Business insight:** regional performance is not evenly distributed. Oceania, Africa, and Asia warrant particular attention in overall planning, while Europe's smaller sales base should be considered when comparing absolute changes with larger regions. Absolute sales alone do not explain profitability or sales potential; margin, operating cost, and region size would be useful additional fields.

### Q5. What is the total count of transactions for each platform?

**Business goal:** compare the scale of the Retail and Shopify channels.

```sql
SELECT
    platform,
    SUM(transactions) AS total_transactions,
    SUM(sales) AS total_sales
FROM clean_weekly_sales
GROUP BY platform
ORDER BY total_transactions DESC;
```

| Platform | Transactions | Sales amount |
|---|---:|---:|
| Retail | 1,081,715,711 | 39,642,532,962 |
| Shopify | 5,925,169 | 1,089,017,790 |

**Business insight:** Retail accounts for the overwhelming majority of recorded transactions and sales in this extract. Shopify is much smaller in transaction volume, but the average sales amount per transaction is substantially higher (see Q9). Management should therefore avoid evaluating the two platforms using transaction count alone.

### Q6. What is the percentage of sales for Retail vs Shopify for each month?

**Business goal:** understand the sales mix between in-store and online channels.

```sql
WITH monthly_platform_sales AS (
    SELECT
        month_number,
        platform,
        SUM(sales) AS platform_sales
    FROM clean_weekly_sales
    GROUP BY month_number, platform
),
monthly_totals AS (
    SELECT
        month_number,
        SUM(platform_sales) AS total_month_sales
    FROM monthly_platform_sales
    GROUP BY month_number
)
SELECT
    p.month_number,
    p.platform,
    p.platform_sales,
    t.total_month_sales,
    ROUND(100.0 * p.platform_sales / NULLIF(t.total_month_sales, 0), 2) AS sales_share_pct
FROM monthly_platform_sales AS p
JOIN monthly_totals AS t
  ON p.month_number = t.month_number
ORDER BY p.month_number, p.platform;
```

**Result summary:** this pooled month-of-year view combines the same month across all three yearly windows. Retail contributes approximately 97.08%–97.59% of monthly sales and Shopify approximately 2.41%–2.92%.

Because pooling years can hide changes in channel mix over time, a useful follow-up is the annual channel mix:

| Year | Retail sales share | Shopify sales share |
|---:|---:|---:|
| 2018 | 97.78% | 2.22% |
| 2019 | 97.47% | 2.53% |
| 2020 | 96.78% | 3.22% |

**Business insight:** Shopify's sales share increased across the three annual windows, from 2.22% to 3.22%. This is evidence of a growing contribution in this dataset, although additional data would be needed to explain whether the change came from more customers, higher spending, or a shift between channels.

### Q7. What is the percentage of sales by demographic for each year?

**Business goal:** compare the contribution of Couples, Families, and unclassified customers over time.

```sql
WITH demographic_sales AS (
    SELECT
        calendar_year,
        demographic,
        SUM(sales) AS demographic_sales
    FROM clean_weekly_sales
    GROUP BY calendar_year, demographic
)
SELECT
    calendar_year,
    demographic,
    demographic_sales,
    ROUND(
        100.0 * demographic_sales
        / NULLIF(SUM(demographic_sales) OVER (PARTITION BY calendar_year), 0),
        2
    ) AS sales_share_pct
FROM demographic_sales
ORDER BY calendar_year, demographic;
```

| Year | Couples | Families | Unknown |
|---:|---:|---:|---:|
| 2018 | 26.31% | 32.02% | 41.67% |
| 2019 | 27.28% | 32.47% | 40.25% |
| 2020 | 28.72% | 32.73% | 38.55% |

**Business insight:** Families are the largest **classified** demographic group, contributing about 32%–33% of sales. Couples' share rises across the observed years. However, the unknown group represents a large share of sales (38.55%–41.67%), so demographic-level decisions should be made cautiously until Data Mart improves customer-segment completeness.

### Q8. Which `age_band` and `demographic` values contribute the most to Retail sales?

**Business goal:** identify the strongest classified customer groups within the Retail channel.

```sql
SELECT
    age_band,
    demographic,
    SUM(sales) AS retail_sales
FROM clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY age_band, demographic
ORDER BY retail_sales DESC;
```

**Result summary from the supplied data:**

| Age band | Demographic | Retail sales amount |
|---|---|---:|
| Unknown | Unknown | 16,067,285,533 |
| Retirees | Families | 6,634,686,916 |
| Retirees | Couples | 6,358,496,539 |
| Middle Aged | Families | 4,354,091,554 |
| Young Adults | Couples | 2,602,922,797 |

**Business insight:** the largest raw category is unclassified, not a meaningful targetable demographic. Among classified groups, Retirees–Families contributes the most Retail sales, followed by Retirees–Couples. Marketing teams could examine these groups further, but should validate margin, retention, and promotion response before changing investment.

### Q9. Can `avg_transaction` be used to find average transaction size by year for Retail vs Shopify?

**Business goal:** compare how much is spent per transaction on each platform.

The row-level `avg_transaction = sales / transactions` is useful for inspecting an individual aggregated row. However, taking a simple average of those row averages gives each aggregated row equal weight, regardless of how many transactions it represents. That is not the overall average transaction size.

Use the weighted calculation instead:

```sql
SELECT
    calendar_year,
    platform,
    SUM(sales) AS total_sales,
    SUM(transactions) AS total_transactions,
    ROUND(SUM(sales) / NULLIF(SUM(transactions), 0), 2) AS average_transaction_size
FROM clean_weekly_sales
GROUP BY calendar_year, platform
ORDER BY calendar_year, platform;
```

| Year | Platform | Average transaction size |
|---:|---|---:|
| 2018 | Retail | 36.55 |
| 2018 | Shopify | 192.48 |
| 2019 | Retail | 36.83 |
| 2019 | Shopify | 183.36 |
| 2020 | Retail | 36.56 |
| 2020 | Shopify | 179.03 |

**Business insight:** the average Shopify transaction is much larger than the average Retail transaction in the supplied extract. Shopify's average transaction size declines over the period while its sales share increases, suggesting that channel sales growth should be examined alongside transaction count and spend per transaction rather than using revenue share alone.

## 6. Before-and-After Analysis: Sustainable Packaging

The analysis event date is **15 June 2020**. In the period definition used here, 15 June is the first date in the *after* period. Before-and-after windows contain four or twelve observed weekly dates on each side.

### Q1. What is total sales for the four weeks before and after 15 June 2020?

**Business goal:** quantify the short-term change in sales around the rollout.

```sql
WITH weekly_sales AS (
    SELECT week_date, SUM(sales) AS total_sales
    FROM clean_weekly_sales
    WHERE calendar_year = 2020
      AND week_date >= DATE_SUB('2020-06-15', INTERVAL 4 WEEK)
      AND week_date <  DATE_ADD('2020-06-15', INTERVAL 4 WEEK)
    GROUP BY week_date
), comparison AS (
    SELECT
        SUM(CASE WHEN week_date < '2020-06-15' THEN total_sales ELSE 0 END) AS sales_before,
        SUM(CASE WHEN week_date >= '2020-06-15' THEN total_sales ELSE 0 END) AS sales_after
    FROM weekly_sales
)
SELECT
    sales_before,
    sales_after,
    sales_after - sales_before AS sales_variance,
    ROUND(100.0 * (sales_after - sales_before) / NULLIF(sales_before, 0), 2) AS variance_pct
FROM comparison;
```

| Period | Sales amount |
|---|---:|
| Four weeks before | 2,345,878,357 |
| Four weeks after | 2,318,994,169 |
| Change | -26,884,188 (-1.15%) |

**Business insight:** sales declined by 26.88 million, or 1.15%, in the four-week period after the rollout date compared with the preceding four weeks.

### Q2. What about the entire twelve weeks before and after?

**Business goal:** test whether the short-term movement remains visible across a longer window.

```sql
WITH weekly_sales AS (
    SELECT week_date, SUM(sales) AS total_sales
    FROM clean_weekly_sales
    WHERE calendar_year = 2020
      AND week_date >= DATE_SUB('2020-06-15', INTERVAL 12 WEEK)
      AND week_date <  DATE_ADD('2020-06-15', INTERVAL 12 WEEK)
    GROUP BY week_date
), comparison AS (
    SELECT
        SUM(CASE WHEN week_date < '2020-06-15' THEN total_sales ELSE 0 END) AS sales_before,
        SUM(CASE WHEN week_date >= '2020-06-15' THEN total_sales ELSE 0 END) AS sales_after
    FROM weekly_sales
)
SELECT
    sales_before,
    sales_after,
    sales_after - sales_before AS sales_variance,
    ROUND(100.0 * (sales_after - sales_before) / NULLIF(sales_before, 0), 2) AS variance_pct
FROM comparison;
```

| Period | Sales amount |
|---|---:|
| Twelve weeks before | 7,126,273,147 |
| Twelve weeks after | 6,973,947,753 |
| Change | -152,325,394 (-2.14%) |

**Business insight:** the twelve-week comparison shows a larger decline in both absolute and percentage terms than the four-week comparison. This warrants investigation, but the design does not prove the packaging change caused the decline.

### Q3. How do the four-week and twelve-week changes compare with 2018 and 2019?

**Business goal:** put the 2020 movement into historical context instead of assuming every change after the rollout is attributable to the initiative.

The query below uses ISO week numbers consistently (`WEEK(date, 3)`). It calculates both periods for each year present in the uploaded data.

```sql
WITH weekly_sales AS (
    SELECT
        calendar_year,
        week_number,
        SUM(sales) AS total_sales
    FROM clean_weekly_sales
    WHERE week_number BETWEEN 13 AND 36
    GROUP BY calendar_year, week_number
), comparison AS (
    SELECT
        calendar_year,
        SUM(CASE WHEN week_number BETWEEN 21 AND 24 THEN total_sales ELSE 0 END) AS before_4_weeks,
        SUM(CASE WHEN week_number BETWEEN 25 AND 28 THEN total_sales ELSE 0 END) AS after_4_weeks,
        SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales ELSE 0 END) AS before_12_weeks,
        SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN total_sales ELSE 0 END) AS after_12_weeks
    FROM weekly_sales
    GROUP BY calendar_year
)
SELECT
    calendar_year,
    after_4_weeks - before_4_weeks AS variance_4_weeks,
    ROUND(100.0 * (after_4_weeks - before_4_weeks) / NULLIF(before_4_weeks, 0), 2) AS variance_pct_4_weeks,
    after_12_weeks - before_12_weeks AS variance_12_weeks,
    ROUND(100.0 * (after_12_weeks - before_12_weeks) / NULLIF(before_12_weeks, 0), 2) AS variance_pct_12_weeks
FROM comparison
ORDER BY calendar_year;
```

| Year | Four-week change | Four-week change % | Twelve-week change | Twelve-week change % |
|---:|---:|---:|---:|---:|
| 2018 | +4,102,105 | +0.19% | +116,339,668 | +1.82% |
| 2019 | +2,336,594 | +0.10% | -20,740,294 | -0.30% |
| 2020 | -26,884,188 | -1.15% | -152,325,394 | -2.14% |

**Business insight:** the 2020 declines are more negative than those observed in the corresponding windows for the two earlier years. This is a useful signal for further investigation, not causal proof. The values above were recalculated from the supplied SQL records after correcting date parsing; different public copies of the case may not be identical.

## 7. Bonus: Which Business Areas Had the Largest Negative Impact in 2020?

**Business goal:** break the 2020 twelve-week change into actionable dimensions. The results below compare ISO weeks 13–24 with weeks 25–36.

### SQL approach for the bonus analysis

The following query applies the same twelve-week before-and-after definition to each business dimension, aggregating each category before calculating the variance. This avoids averaging window-function totals that have been repeated across detailed rows.

```sql
WITH dimension_sales AS (
    SELECT
        'region' AS dimension_name,
        region AS dimension_value,
        SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN sales ELSE 0 END) AS sales_before,
        SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN sales ELSE 0 END) AS sales_after
    FROM clean_weekly_sales
    WHERE calendar_year = 2020 AND week_number BETWEEN 13 AND 36
    GROUP BY region

    UNION ALL

    SELECT
        'platform', platform,
        SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN sales ELSE 0 END),
        SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN sales ELSE 0 END)
    FROM clean_weekly_sales
    WHERE calendar_year = 2020 AND week_number BETWEEN 13 AND 36
    GROUP BY platform

    UNION ALL

    SELECT
        'age_band', age_band,
        SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN sales ELSE 0 END),
        SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN sales ELSE 0 END)
    FROM clean_weekly_sales
    WHERE calendar_year = 2020 AND week_number BETWEEN 13 AND 36
    GROUP BY age_band

    UNION ALL

    SELECT
        'demographic', demographic,
        SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN sales ELSE 0 END),
        SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN sales ELSE 0 END)
    FROM clean_weekly_sales
    WHERE calendar_year = 2020 AND week_number BETWEEN 13 AND 36
    GROUP BY demographic

    UNION ALL

    SELECT
        'customer_type', customer_type,
        SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN sales ELSE 0 END),
        SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN sales ELSE 0 END)
    FROM clean_weekly_sales
    WHERE calendar_year = 2020 AND week_number BETWEEN 13 AND 36
    GROUP BY customer_type
)
SELECT
    dimension_name,
    dimension_value,
    sales_before,
    sales_after,
    sales_after - sales_before AS sales_change,
    ROUND(100.0 * (sales_after - sales_before) / NULLIF(sales_before, 0), 2) AS change_pct
FROM dimension_sales
ORDER BY dimension_name, sales_change;
```

### Regional impact

| Region | Sales change | Change % |
|---|---:|---:|
| Oceania | -71,321,100 | -3.03% |
| Asia | -53,436,845 | -3.26% |
| USA | -10,814,843 | -1.60% |
| Africa | -9,146,811 | -0.54% |
| Canada | -8,174,013 | -1.92% |
| South America | -4,584,174 | -2.15% |
| Europe | +5,152,392 | +4.73% |

Oceania had the largest absolute decline, followed by Asia. Asia's percentage decline was slightly larger than Oceania's, which is why both absolute and relative changes matter. Europe moved in the opposite direction in this comparison.

### Platform impact

| Platform | Sales change | Change % |
|---|---:|---:|
| Retail | -168,083,834 | -2.43% |
| Shopify | +15,758,440 | +7.18% |

The total decline is driven by Retail. Shopify increased over the same comparison windows, although its starting sales base is much smaller.

### Age-band impact

| Age band | Sales change | Change % |
|---|---:|---:|
| Unknown | -92,393,021 | -3.34% |
| Retirees | -29,549,521 | -1.23% |
| Middle Aged | -22,994,292 | -1.97% |
| Young Adults | -7,388,560 | -0.92% |

The unknown group accounts for the largest decline by far. It includes sales with no usable segment assignment, so the result is not evidence that a specific age group drove the decline.

### Demographic impact

| Demographic | Sales change | Change % |
|---|---:|---:|
| Unknown | -92,393,021 | -3.34% |
| Families | -42,320,015 | -1.82% |
| Couples | -17,612,358 | -0.87% |

Among classified demographics, Families experienced the larger absolute and relative decline. This could guide further segmentation work, but a targeted intervention should be based on more granular customer behaviour and margin evidence.

### Customer-type impact

| Customer type | Sales change | Change % |
|---|---:|---:|
| Existing | -83,872,973 | -2.27% |
| Guest | -77,202,666 | -3.00% |
| New | +8,750,245 | +1.01% |

Existing and Guest customer sales account for most of the decrease, while New customer sales increased slightly. This suggests that retention, repeat purchasing, and the guest checkout experience are sensible areas for follow-up analysis.

> **Calculation note:** dimensional changes above are calculated by aggregating sales separately before and after and then comparing the resulting totals for each dimension. Do not average duplicated window totals across detailed rows; window-function totals repeated across row-level records can inflate or distort results if aggregated again.

## 8. Recommendations to the Business

1. **Investigate Retail first.** It explains the net decline in the twelve-week comparison, while Shopify grew. Break Retail down further by product, store, basket size, units sold, and margin to look for a specific mechanism.
2. **Prioritise Oceania and Asia for diagnosis.** They have the largest absolute reductions. Compare the change with inventory availability, product mix, promotions, region-specific customer feedback, and historical seasonality before deciding on an intervention.
3. **Improve segment completeness.** The unknown segment/customer group contributes materially to sales and accounts for the largest decline. Improve data collection or mapping before making demographic targeting decisions.
4. **Monitor customer behaviour, not only revenue.** Track transactions, spend per transaction, repeat purchase, customer retention, and product-level conversion alongside sales.
5. **Use a stronger evaluation design next time.** Where possible, use a phased rollout, a comparable control group, or an A/B test. Document rollout dates and monitor leading indicators so that a sales movement can be separated more credibly from other market effects.

## 9. SQL Skills Demonstrated

- Date parsing and date-derived dimensions
- `CASE WHEN` for data standardisation and category mapping
- Aggregations with `GROUP BY` and `SUM`
- Common Table Expressions (CTEs)
- Window functions for percentage-of-total calculations
- Period-over-period variance and percentage change
- Weighted averages and denominator selection
- Data validation, scope checks, and interpretation of limitations

## 10. Conclusion

In the supplied extract, sales fell by **1.15%** in the four weeks and **2.14%** in the twelve weeks after the sustainable-packaging baseline date. Retail, Oceania, Asia, and unclassified customer groups explain much of the negative movement, while Shopify and New customer sales increased over the twelve-week comparison.

The right business response is not to conclude immediately that sustainable packaging caused the decline. Data Mart should investigate channel, region, basket, and customer-behaviour patterns, improve customer-segment data quality, and use a more robust rollout evaluation for future sustainability changes.

---
