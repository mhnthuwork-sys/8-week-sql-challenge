# 🥑 Case Study #3 - Foodie-Fi

## A. Customer Journey

```sql
SELECT
    s.customer_id,
    p.plan_name,
    s.start_date
FROM foodie_fi.subscriptions s
INNER JOIN foodie_fi.plans p
    ON s.plan_id = p.plan_id
WHERE s.customer_id IN (1, 2, 11, 13, 15, 16, 18, 19)
ORDER BY s.customer_id, s.start_date;
```

**Customer Journeys:**

| customer_id | Journey |
|-------------|---------|
| 1 | Started trial Aug 1 → downgraded to basic monthly Aug 8 |
| 2 | Started trial Sep 20 → upgraded straight to pro annual Sep 27 |
| 11 | Started trial Nov 19 → churned Nov 26 (never converted) |
| 13 | Started trial Dec 15 → basic monthly Dec 22 → pro monthly Mar 29 2021 |
| 15 | Started trial Mar 17 → pro monthly Mar 24 → churned Apr 29 |
| 16 | Started trial May 31 → basic monthly Jun 7 → pro annual Oct 21 |
| 18 | Started trial Jul 6 → pro monthly Jul 13 |
| 19 | Started trial Jun 22 → pro monthly Jun 29 → pro annual Aug 29 |

> 💡 Most customers convert within 7 days (end of trial). Customer 11 is the only one who churned immediately after trial.

---

## B. Data Analysis Questions

### B1. How many customers has Foodie-Fi ever had?

```sql
SELECT COUNT(DISTINCT customer_id) AS total_customers
FROM foodie_fi.subscriptions;
```

**Result:**

| total_customers |
|----------------|
| 1000 |

---

### B2. What is the monthly distribution of trial plan start_date values?

```sql
SELECT
    DATE_TRUNC('month', start_date) AS month_start,
    TO_CHAR(start_date, 'Month')    AS month_name,
    COUNT(*)                        AS trial_count
FROM foodie_fi.subscriptions
WHERE plan_id = 0
GROUP BY DATE_TRUNC('month', start_date), TO_CHAR(start_date, 'Month')
ORDER BY month_start;
```

**Result:**

| month_name | trial_count |
|------------|-------------|
| January | 88 |
| February | 68 |
| March | 94 |
| April | 81 |
| May | 88 |
| June | 79 |
| July | 89 |
| August | 88 |
| September | 87 |
| October | 79 |
| November | 75 |
| December | 84 |

> 💡 March has the highest trial signups (94). February is lowest (68) — likely due to shorter month.

---

### B3. What plan start_date values occur after 2020? Show count by plan name.

```sql
SELECT
    p.plan_name,
    COUNT(*) AS event_count
FROM foodie_fi.subscriptions s
INNER JOIN foodie_fi.plans p
    ON s.plan_id = p.plan_id
WHERE s.start_date >= '2021-01-01'
GROUP BY p.plan_name
ORDER BY event_count DESC;
```

**Result:**

| plan_name | event_count |
|-----------|-------------|
| churn | 71 |
| pro annual | 63 |
| pro monthly | 60 |
| basic monthly | 8 |

> 💡 In 2021, churn is the most common event — worth investigating customer retention strategies. No trial events after 2020 (all trials are short-lived).

---

### B4. What is the customer count and percentage of customers who have churned?

```sql
SELECT
    COUNT(*)                                                AS churned_customers,
    ROUND(COUNT(*) * 100.0 / (
        SELECT COUNT(DISTINCT customer_id)
        FROM foodie_fi.subscriptions
    ), 1)                                                   AS churn_pct
FROM foodie_fi.subscriptions
WHERE plan_id = 4;
```

**Result:**

| churned_customers | churn_pct |
|------------------|-----------|
| 307 | 30.7% |

> 💡 Nearly 1 in 3 customers have churned — a significant retention problem that needs addressing.

---

### B5. How many customers churned straight after their initial free trial?

```sql
WITH ranked_plans AS (
    SELECT
        customer_id,
        plan_id,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY start_date
        ) AS rn
    FROM foodie_fi.subscriptions
)
SELECT
    COUNT(*)                                                    AS churned_after_trial,
    ROUND(COUNT(*) * 100.0 / (
        SELECT COUNT(DISTINCT customer_id)
        FROM foodie_fi.subscriptions
    ), 0)                                                       AS pct_of_total
FROM ranked_plans
WHERE plan_id = 4 AND rn = 2;
```

**Steps:**
- Use `ROW_NUMBER()` to rank each customer's plans in chronological order.
- Filter where `plan_id = 4` (churn) AND `rn = 2` — meaning churn was their second event (straight after trial).

**Result:**

| churned_after_trial | pct_of_total |
|--------------------|-------------|
| 92 | 9% |

> 💡 9% of all customers churned immediately after their free trial — these customers never saw enough value to convert. Exit survey data would help understand why.

---

### B6. What is the number and percentage of customer plans after their initial free trial?

```sql
WITH ranked_plans AS (
    SELECT
        customer_id,
        plan_id,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY start_date
        ) AS rn
    FROM foodie_fi.subscriptions
)
SELECT
    p.plan_name,
    COUNT(*)                                                    AS customer_count,
    ROUND(COUNT(*) * 100.0 / (
        SELECT COUNT(DISTINCT customer_id)
        FROM foodie_fi.subscriptions
    ), 1)                                                       AS pct
FROM ranked_plans rp
INNER JOIN foodie_fi.plans p ON rp.plan_id = p.plan_id
WHERE rn = 2
GROUP BY p.plan_name
ORDER BY customer_count DESC;
```

**Result:**

| plan_name | customer_count | pct |
|-----------|---------------|-----|
| basic monthly | 546 | 54.6% |
| pro monthly | 325 | 32.5% |
| churn | 92 | 9.2% |
| pro annual | 37 | 3.7% |

> 💡 Over half of customers default to basic monthly after trial. Only 3.7% go straight to pro annual — these are the most committed customers.

---

### B7. What is the customer count and percentage breakdown of all 5 plans at 2020-12-31?

```sql
WITH latest_plan AS (
    SELECT
        customer_id,
        plan_id,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY start_date DESC
        ) AS rn
    FROM foodie_fi.subscriptions
    WHERE start_date <= '2020-12-31'
)
SELECT
    p.plan_name,
    COUNT(*)                                                    AS customer_count,
    ROUND(COUNT(*) * 100.0 / (
        SELECT COUNT(DISTINCT customer_id)
        FROM foodie_fi.subscriptions
    ), 1)                                                       AS pct
FROM latest_plan lp
INNER JOIN foodie_fi.plans p ON lp.plan_id = p.plan_id
WHERE rn = 1
GROUP BY p.plan_name
ORDER BY customer_count DESC;
```

**Steps:**
- Use `ROW_NUMBER()` with `ORDER BY start_date DESC` to get each customer's most recent plan as of 2020-12-31.
- Filter `WHERE rn = 1` to get only the latest plan per customer.

**Result:**

| plan_name | customer_count | pct |
|-----------|---------------|-----|
| pro monthly | 326 | 32.6% |
| churn | 236 | 23.6% |
| basic monthly | 224 | 22.4% |
| pro annual | 195 | 19.5% |
| trial | 19 | 1.9% |

> 💡 At end of 2020, pro monthly is the most common active plan. 23.6% have already churned — nearly 1 in 4 customers lost by year end.

---

### B8. How many customers have upgraded to an annual plan in 2020?

```sql
SELECT COUNT(DISTINCT customer_id) AS annual_upgrades_2020
FROM foodie_fi.subscriptions
WHERE plan_id = 3
  AND start_date BETWEEN '2020-01-01' AND '2020-12-31';
```

**Result:**

| annual_upgrades_2020 |
|---------------------|
| 195 |

> 💡 195 customers committed to pro annual in 2020 — these represent the highest LTV (lifetime value) customers at $199 each.

---

### B9. How many days on average does it take for a customer to upgrade to an annual plan?

```sql
WITH trial_start AS (
    SELECT customer_id, start_date AS trial_date
    FROM foodie_fi.subscriptions
    WHERE plan_id = 0
),
annual_start AS (
    SELECT customer_id, start_date AS annual_date
    FROM foodie_fi.subscriptions
    WHERE plan_id = 3
)
SELECT
    ROUND(AVG(annual_date - trial_date), 0) AS avg_days_to_annual
FROM trial_start t
INNER JOIN annual_start a ON t.customer_id = a.customer_id;
```

**Result:**

| avg_days_to_annual |
|-------------------|
| 105 |

> 💡 On average, customers take ~105 days (about 3.5 months) to upgrade to an annual plan. This suggests a 3-4 month window is critical for annual plan conversion campaigns.

---

### B10. Break down average days to annual plan into 30-day periods

```sql
WITH trial_start AS (
    SELECT customer_id, start_date AS trial_date
    FROM foodie_fi.subscriptions
    WHERE plan_id = 0
),
annual_start AS (
    SELECT customer_id, start_date AS annual_date
    FROM foodie_fi.subscriptions
    WHERE plan_id = 3
),
days_diff AS (
    SELECT
        t.customer_id,
        (a.annual_date - t.trial_date) AS days_to_annual
    FROM trial_start t
    INNER JOIN annual_start a ON t.customer_id = a.customer_id
)
SELECT
    CONCAT(
        (FLOOR(days_to_annual / 30) * 30), '–',
        (FLOOR(days_to_annual / 30) * 30 + 30), ' days'
    )                           AS period,
    COUNT(*)                    AS customer_count,
    ROUND(AVG(days_to_annual))  AS avg_days
FROM days_diff
GROUP BY FLOOR(days_to_annual / 30)
ORDER BY FLOOR(days_to_annual / 30);
```

**Result:**

| period | customer_count | avg_days |
|--------|---------------|----------|
| 0–30 days | 49 | 10 |
| 31–60 days | 24 | 42 |
| 61–90 days | 34 | 71 |
| 91–120 days | 35 | 101 |
| 121–150 days | 42 | 133 |
| 151–180 days | 36 | 162 |
| 181–210 days | 26 | 191 |
| 211–240 days | 4 | 224 |
| 241–270 days | 5 | 257 |
| 271–300 days | 1 | 285 |
| 301–330 days | 1 | 327 |
| 331–360 days | 1 | 346 |

> 💡 49 customers converted within the first 30 days — likely highly motivated users. The biggest cluster is in the 121–150 day window (42 customers), suggesting a targeted campaign around month 4 could boost annual conversions.

---

### B11. How many customers downgraded from pro monthly to basic monthly in 2020?

```sql
WITH ranked AS (
    SELECT
        customer_id,
        plan_id,
        start_date,
        LEAD(plan_id) OVER (
            PARTITION BY customer_id
            ORDER BY start_date
        ) AS next_plan_id
    FROM foodie_fi.subscriptions
    WHERE start_date <= '2020-12-31'
)
SELECT COUNT(*) AS downgrades
FROM ranked
WHERE plan_id = 2         -- pro monthly
  AND next_plan_id = 1;   -- basic monthly
```

**Steps:**
- Use `LEAD()` window function to look at the next plan for each customer.
- Filter where current plan is `pro monthly (2)` and next plan is `basic monthly (1)`.

**Result:**

| downgrades |
|-----------|
| 0 |

> 💡 Zero customers downgraded from pro monthly to basic in 2020 — customers either stayed on pro or churned entirely. This is a positive signal for plan stickiness.

---

## C. Challenge Payment Question

Create a `payments` table for 2020 showing all payments made by customers.

```sql
CREATE TABLE payments AS
WITH RECURSIVE payment_dates AS (
    -- Base case: get all paid plan subscriptions in 2020
    SELECT
        s.customer_id,
        s.plan_id,
        p.plan_name,
        s.start_date                                    AS payment_date,
        p.price                                         AS amount,
        LEAD(s.start_date) OVER (
            PARTITION BY s.customer_id
            ORDER BY s.start_date
        )                                               AS next_plan_date
    FROM foodie_fi.subscriptions s
    INNER JOIN foodie_fi.plans p ON s.plan_id = p.plan_id
    WHERE p.price IS NOT NULL          -- exclude churn (null price)
      AND s.plan_id != 0               -- exclude free trial
      AND s.start_date <= '2020-12-31'

    UNION ALL

    -- Recursive case: generate monthly payments
    SELECT
        customer_id,
        plan_id,
        plan_name,
        (payment_date + INTERVAL '1 month')::DATE       AS payment_date,
        amount,
        next_plan_date
    FROM payment_dates
    WHERE plan_id IN (1, 2)            -- only monthly plans recurse
      AND (payment_date + INTERVAL '1 month') <= '2020-12-31'
      AND (next_plan_date IS NULL
           OR (payment_date + INTERVAL '1 month') < next_plan_date)
)
SELECT
    customer_id,
    plan_id,
    plan_name,
    payment_date,
    amount,
    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY payment_date
    ) AS payment_order
FROM payment_dates
ORDER BY customer_id, payment_date;
```

**Sample Output:**

| customer_id | plan_id | plan_name | payment_date | amount | payment_order |
|-------------|---------|-----------|-------------|--------|---------------|
| 1 | 1 | basic monthly | 2020-08-08 | 9.90 | 1 |
| 1 | 1 | basic monthly | 2020-09-08 | 9.90 | 2 |
| 1 | 1 | basic monthly | 2020-10-08 | 9.90 | 3 |
| 2 | 3 | pro annual | 2020-09-27 | 199.00 | 1 |
| 16 | 3 | pro annual | 2020-10-21 | 189.10 | 6 |

> 💡 Note: Customer 16 paid $189.10 for pro annual instead of $199 because they upgraded from basic monthly mid-month — the $9.90 already paid that month is deducted.

---

## D. Outside The Box Questions

### D1. How would you calculate the rate of growth for Foodie-Fi?

```
Key growth metrics to track month-over-month:
- New trial signups
- Trial → paid conversion rate
- MRR (Monthly Recurring Revenue) = SUM of active paid subscriptions
- Net Revenue Retention = (MRR end of month - MRR start + expansions - churned) / MRR start
```

```sql
-- Monthly new trials (growth proxy)
SELECT
    TO_CHAR(start_date, 'YYYY-MM') AS month,
    COUNT(*)                        AS new_trials
FROM foodie_fi.subscriptions
WHERE plan_id = 0
GROUP BY TO_CHAR(start_date, 'YYYY-MM')
ORDER BY month;
```

---

### D2. What key metrics would you recommend Foodie-Fi to track?

| Metric | Why It Matters |
|--------|---------------|
| Monthly churn rate | Core health indicator — target < 5% monthly |
| Trial conversion rate | % of trials → any paid plan |
| Average Revenue Per User (ARPU) | Revenue efficiency |
| Customer Lifetime Value (CLV) | Long-term profitability |
| Plan upgrade rate | Basic → Pro shows product value |
| Days to annual upgrade | Conversion window for targeted campaigns |

---

### D3. Key customer journeys to analyse for retention

```
1. Trial churners      → What content did they view? Did they complete onboarding?
2. Basic → Churn path  → Did they hit watch limits? Price sensitivity?
3. Pro → Churn path    → Long-term users who left — what triggered it?
4. Basic → Pro upgrade → What moment drove the decision to upgrade?
```

---

### D4. Exit survey questions for churning customers

```
1. What was the primary reason for cancelling?
   □ Too expensive   □ Not enough content   □ Technical issues
   □ Found a better service   □ No longer need it

2. How satisfied were you with the content variety? (1–5)

3. Was there a specific feature you felt was missing?

4. Would a lower-priced plan have kept you subscribed? (Yes / No)

5. How likely are you to recommend Foodie-Fi to a friend? (NPS 0–10)
```

---

### D5. Business levers to reduce churn rate

| Lever | Action | Validation Method |
|-------|--------|-----------------|
| Price sensitivity | Offer pause subscription instead of cancel | A/B test cancel flow |
| Content gap | Survey churners on missing content categories | Track reactivation rate |
| Engagement drop | Email campaign when usage drops 50% | Compare churn rate pre/post campaign |
| Annual incentive | Offer 2 months free on annual upgrade | Track annual conversion rate lift |
| Onboarding | Add personalised content recommendations in first 7 days | Compare trial→paid rate by cohort |

---

## SQL Techniques Used

| Section | Techniques |
|---------|-----------|
| A | JOIN, ORDER BY, WHERE IN |
| B1–B3 | COUNT DISTINCT, DATE_TRUNC, TO_CHAR, GROUP BY |
| B4 | Subquery, COUNT, ROUND |
| B5–B6 | CTE, ROW_NUMBER, PARTITION BY |
| B7 | CTE, ROW_NUMBER DESC (latest plan snapshot) |
| B8 | BETWEEN date filter |
| B9 | CTE, date arithmetic, AVG |
| B10 | CTE, FLOOR, CONCAT for bucketing |
| B11 | LEAD window function |
| C | Recursive CTE, INTERVAL, ROW_NUMBER |
