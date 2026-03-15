# 🍕 Case Study #2 - Pizza Runner
---

## ⚠️ Data Cleaning 

The `customer_orders` and `runner_orders` tables have messy null values — clean them before any query.

### Clean customer_orders

```sql
CREATE TEMP TABLE customer_orders_cleaned AS
SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE WHEN exclusions IN ('null', 'NaN', '')  THEN NULL ELSE exclusions END AS exclusions,
    CASE WHEN extras    IN ('null', 'NaN', '')  THEN NULL ELSE extras    END AS extras,
    order_time
FROM pizza_runner.customer_orders;
```

### Clean runner_orders

```sql
CREATE TEMP TABLE runner_orders_cleaned AS
SELECT
    order_id,
    runner_id,
    CASE WHEN pickup_time IN ('null', 'NaN', '') THEN NULL ELSE pickup_time END AS pickup_time,
    CASE WHEN distance    IN ('null', 'NaN', '') THEN NULL
         ELSE REGEXP_REPLACE(distance, '[^0-9.]', '', 'g') END::NUMERIC          AS distance,
    CASE WHEN duration    IN ('null', 'NaN', '') THEN NULL
         ELSE REGEXP_REPLACE(duration, '[^0-9.]', '', 'g') END::NUMERIC          AS duration,
    CASE WHEN cancellation IN ('null', 'NaN', '') THEN NULL ELSE cancellation END AS cancellation
FROM pizza_runner.runner_orders;
```

> 💡 `REGEXP_REPLACE` strips non-numeric characters from `distance` (e.g. "20km" → 20) and `duration` (e.g. "32 minutes" → 32).

---

## A. Pizza Metrics

### A1. How many pizzas were ordered?

```sql
SELECT COUNT(*) AS total_pizzas_ordered
FROM customer_orders_cleaned;
```

**Result:**

| total_pizzas_ordered |
|----------------------|
| 14 |

> 💡 14 pizzas were ordered in total across all orders.

---

### A2. How many unique customer orders were made?

```sql
SELECT COUNT(DISTINCT order_id) AS unique_orders
FROM customer_orders_cleaned;
```

**Result:**

| unique_orders |
|---------------|
| 10 |

> 💡 10 unique orders were placed (some orders contained multiple pizzas).

---

### A3. How many successful orders were delivered by each runner?

```sql
SELECT
    runner_id,
    COUNT(*) AS successful_deliveries
FROM runner_orders_cleaned
WHERE cancellation IS NULL
GROUP BY runner_id
ORDER BY runner_id;
```

**Steps:**
- Filter `WHERE cancellation IS NULL` to exclude cancelled orders.
- `GROUP BY runner_id` to count per runner.

**Result:**

| runner_id | successful_deliveries |
|-----------|----------------------|
| 1 | 4 |
| 2 | 3 |
| 3 | 1 |

> 💡 Runner 1 completed the most deliveries (4). Runner 3 only completed 1.

---

### A4. How many of each type of pizza was delivered?

```sql
SELECT
    pn.pizza_name,
    COUNT(*) AS total_delivered
FROM customer_orders_cleaned co
INNER JOIN runner_orders_cleaned ro
    ON co.order_id = ro.order_id
INNER JOIN pizza_runner.pizza_names pn
    ON co.pizza_id = pn.pizza_id
WHERE ro.cancellation IS NULL
GROUP BY pn.pizza_name
ORDER BY total_delivered DESC;
```

**Result:**

| pizza_name | total_delivered |
|------------|----------------|
| Meatlovers | 9 |
| Vegetarian | 3 |

> 💡 Meatlovers is 3x more popular than Vegetarian among delivered orders.

---

### A5. How many Vegetarian and Meatlovers were ordered by each customer?

```sql
SELECT
    co.customer_id,
    pn.pizza_name,
    COUNT(*) AS total_ordered
FROM customer_orders_cleaned co
INNER JOIN pizza_runner.pizza_names pn
    ON co.pizza_id = pn.pizza_id
GROUP BY co.customer_id, pn.pizza_name
ORDER BY co.customer_id, pn.pizza_name;
```

**Result:**

| customer_id | pizza_name | total_ordered |
|-------------|------------|---------------|
| 101 | Meatlovers | 2 |
| 101 | Vegetarian | 1 |
| 102 | Meatlovers | 2 |
| 102 | Vegetarian | 1 |
| 103 | Meatlovers | 3 |
| 103 | Vegetarian | 1 |
| 104 | Meatlovers | 3 |
| 105 | Vegetarian | 1 |

> 💡 Customer 104 ordered the most Meatlovers (3). No customer ordered more Vegetarian than Meatlovers.

---

### A6. What was the maximum number of pizzas delivered in a single order?

```sql
SELECT MAX(pizza_count) AS max_pizzas_in_single_order
FROM (
    SELECT
        co.order_id,
        COUNT(*) AS pizza_count
    FROM customer_orders_cleaned co
    INNER JOIN runner_orders_cleaned ro
        ON co.order_id = ro.order_id
    WHERE ro.cancellation IS NULL
    GROUP BY co.order_id
) AS order_counts;
```

**Result:**

| max_pizzas_in_single_order |
|---------------------------|
| 3 |

> 💡 The most pizzas delivered in a single order was 3.

---

### A7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

```sql
SELECT
    co.customer_id,
    SUM(CASE WHEN co.exclusions IS NOT NULL
              OR  co.extras     IS NOT NULL THEN 1 ELSE 0 END) AS with_changes,
    SUM(CASE WHEN co.exclusions IS NULL
             AND  co.extras     IS NULL     THEN 1 ELSE 0 END) AS no_changes
FROM customer_orders_cleaned co
INNER JOIN runner_orders_cleaned ro
    ON co.order_id = ro.order_id
WHERE ro.cancellation IS NULL
GROUP BY co.customer_id
ORDER BY co.customer_id;
```

**Steps:**
- Use `CASE WHEN` to flag pizzas with at least one exclusion or extra as "with changes".
- Use `SUM` to count both groups per customer.

**Result:**

| customer_id | with_changes | no_changes |
|-------------|-------------|------------|
| 101 | 0 | 2 |
| 102 | 0 | 3 |
| 103 | 3 | 0 |
| 104 | 2 | 1 |
| 105 | 1 | 0 |

> 💡 Customers 101 and 102 never customized their orders. Customer 103 always customized.

---

### A8. How many pizzas were delivered that had both exclusions AND extras?

```sql
SELECT COUNT(*) AS pizzas_with_both
FROM customer_orders_cleaned co
INNER JOIN runner_orders_cleaned ro
    ON co.order_id = ro.order_id
WHERE ro.cancellation IS NULL
  AND co.exclusions IS NOT NULL
  AND co.extras     IS NOT NULL;
```

**Result:**

| pizzas_with_both |
|-----------------|
| 1 |

> 💡 Only 1 pizza was delivered with both exclusions and extras — customers rarely make complex customizations.

---

### A9. What was the total volume of pizzas ordered for each hour of the day?

```sql
SELECT
    EXTRACT(HOUR FROM order_time) AS hour_of_day,
    COUNT(*)                      AS total_pizzas
FROM customer_orders_cleaned
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

**Result:**

| hour_of_day | total_pizzas |
|-------------|-------------|
| 11 | 1 |
| 13 | 3 |
| 18 | 3 |
| 19 | 1 |
| 21 | 3 |
| 23 | 3 |

> 💡 Peak ordering hours are 13:00, 18:00, 21:00, and 23:00 — lunch and dinner times.

---

### A10. What was the volume of orders for each day of the week?

```sql
SELECT
    TO_CHAR(order_time, 'Day') AS day_of_week,
    COUNT(*)                   AS total_pizzas
FROM customer_orders_cleaned
GROUP BY day_of_week
ORDER BY total_pizzas DESC;
```

**Result:**

| day_of_week | total_pizzas |
|-------------|-------------|
| Wednesday   | 5 |
| Saturday    | 5 |
| Thursday    | 3 |
| Friday      | 1 |

> 💡 Wednesday and Saturday are the busiest days — worth scheduling more runners on these days.

---

## B. Runner and Customer Experience

### B1. How many runners signed up for each 1 week period?

```sql
SELECT
    FLOOR(EXTRACT(DOY FROM registration_date) / 7) AS week_number,
    COUNT(*)                                        AS runners_signed_up
FROM pizza_runner.runners
GROUP BY week_number
ORDER BY week_number;
```

**Result:**

| week_number | runners_signed_up |
|-------------|------------------|
| 0 | 2 |
| 1 | 1 |
| 2 | 1 |

> 💡 2 runners signed up in week 1, then 1 per week after that.

---

### B2. What was the average time for each runner to arrive at HQ to pick up the order?

```sql
SELECT
    ro.runner_id,
    ROUND(AVG(
        EXTRACT(EPOCH FROM (ro.pickup_time::TIMESTAMP - co.order_time)) / 60
    ), 2) AS avg_pickup_minutes
FROM runner_orders_cleaned ro
INNER JOIN customer_orders_cleaned co
    ON ro.order_id = co.order_id
WHERE ro.cancellation IS NULL
GROUP BY ro.runner_id
ORDER BY ro.runner_id;
```

**Result:**

| runner_id | avg_pickup_minutes |
|-----------|--------------------|
| 1 | 14.33 |
| 2 | 20.01 |
| 3 | 10.47 |

> 💡 Runner 3 is fastest to arrive at HQ (10.5 min avg). Runner 2 is slowest (20 min avg).

---

### B3. Is there any relationship between the number of pizzas and how long the order takes to prepare?

```sql
SELECT
    co.order_id,
    COUNT(co.pizza_id)                                                      AS pizza_count,
    ROUND(AVG(
        EXTRACT(EPOCH FROM (ro.pickup_time::TIMESTAMP - co.order_time)) / 60
    ), 2)                                                                    AS avg_prep_minutes
FROM customer_orders_cleaned co
INNER JOIN runner_orders_cleaned ro
    ON co.order_id = ro.order_id
WHERE ro.cancellation IS NULL
GROUP BY co.order_id
ORDER BY pizza_count;
```

**Result:**

| order_id | pizza_count | avg_prep_minutes |
|----------|-------------|-----------------|
| 1 | 1 | 10.53 |
| 2 | 1 | 10.03 |
| 5 | 1 | 10.47 |
| 7 | 1 | 10.47 |
| 8 | 1 | 20.48 |
| 3 | 2 | 21.23 |
| 10 | 2 | 15.52 |
| 4 | 3 | 29.28 |

> 💡 Yes — more pizzas = longer prep time. 1 pizza ~10 min, 2 pizzas ~18 min, 3 pizzas ~29 min.

---

### B4. What was the average distance travelled for each customer?

```sql
SELECT
    co.customer_id,
    ROUND(AVG(ro.distance), 2) AS avg_distance_km
FROM customer_orders_cleaned co
INNER JOIN runner_orders_cleaned ro
    ON co.order_id = ro.order_id
WHERE ro.cancellation IS NULL
GROUP BY co.customer_id
ORDER BY co.customer_id;
```

**Result:**

| customer_id | avg_distance_km |
|-------------|----------------|
| 101 | 20.00 |
| 102 | 16.73 |
| 103 | 23.40 |
| 104 | 10.00 |
| 105 | 25.00 |

> 💡 Customer 105 lives farthest (25km avg). Customer 104 is closest (10km).

---

### B5. What was the difference between the longest and shortest delivery times?

```sql
SELECT
    MAX(duration) - MIN(duration) AS delivery_time_difference_minutes
FROM runner_orders_cleaned
WHERE cancellation IS NULL;
```

**Result:**

| delivery_time_difference_minutes |
|---------------------------------|
| 30 |

> 💡 30 minute gap between the fastest and slowest deliveries.

---

### B6. What was the average speed for each runner per delivery?

```sql
SELECT
    runner_id,
    order_id,
    distance,
    duration,
    ROUND(distance / (duration / 60.0), 2) AS avg_speed_kmh
FROM runner_orders_cleaned
WHERE cancellation IS NULL
ORDER BY runner_id, order_id;
```

**Result:**

| runner_id | order_id | distance | duration | avg_speed_kmh |
|-----------|----------|----------|----------|--------------|
| 1 | 1 | 20 | 32 | 37.50 |
| 1 | 2 | 20 | 27 | 44.44 |
| 1 | 3 | 13.4 | 20 | 40.20 |
| 1 | 10 | 10 | 10 | 60.00 |
| 2 | 4 | 23.4 | 40 | 35.10 |
| 2 | 7 | 25 | 25 | 60.00 |
| 2 | 8 | 23.4 | 15 | 93.60 |
| 3 | 5 | 10 | 15 | 40.00 |

> 💡 Runner 2's speed varies wildly from 35 to 93.6 km/h — worth investigating if data is correct or if there's a safety concern.

---

### B7. What is the successful delivery percentage for each runner?

```sql
SELECT
    runner_id,
    COUNT(*)                                                    AS total_orders,
    COUNT(CASE WHEN cancellation IS NULL THEN 1 END)            AS successful,
    ROUND(
        COUNT(CASE WHEN cancellation IS NULL THEN 1 END) * 100.0
        / COUNT(*), 0
    )                                                           AS success_pct
FROM runner_orders_cleaned
GROUP BY runner_id
ORDER BY runner_id;
```

**Result:**

| runner_id | total_orders | successful | success_pct |
|-----------|-------------|------------|------------|
| 1 | 4 | 4 | 100% |
| 2 | 4 | 3 | 75% |
| 3 | 2 | 1 | 50% |

> 💡 Runner 1 has a perfect 100% delivery rate. Runner 3 has only 50% — may need additional support or training.

---

## C. Ingredient Optimisation

### C1. What are the standard ingredients for each pizza?

```sql
SELECT
    pn.pizza_name,
    STRING_AGG(pt.topping_name, ', ' ORDER BY pt.topping_name) AS ingredients
FROM pizza_runner.pizza_recipes pr
CROSS JOIN LATERAL unnest(string_to_array(pr.toppings::TEXT, ', ')) AS t(topping_id)
INNER JOIN pizza_runner.pizza_toppings pt
    ON pt.topping_id = t.topping_id::INT
INNER JOIN pizza_runner.pizza_names pn
    ON pn.pizza_id = pr.pizza_id
GROUP BY pn.pizza_name
ORDER BY pn.pizza_name;
```

**Result:**

| pizza_name | ingredients |
|------------|-------------|
| Meatlovers | Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |
| Vegetarian | Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce |

---

### C2. What was the most commonly added extra?

```sql
SELECT
    pt.topping_name,
    COUNT(*) AS times_added
FROM customer_orders_cleaned co
CROSS JOIN LATERAL unnest(string_to_array(co.extras, ', ')) AS e(topping_id)
INNER JOIN pizza_runner.pizza_toppings pt
    ON pt.topping_id = e.topping_id::INT
WHERE co.extras IS NOT NULL
GROUP BY pt.topping_name
ORDER BY times_added DESC
LIMIT 1;
```

**Result:**

| topping_name | times_added |
|-------------|------------|
| Bacon | 4 |

> 💡 Bacon is by far the most popular extra — could be featured in a marketing promotion.

---

### C3. What was the most common exclusion?

```sql
SELECT
    pt.topping_name,
    COUNT(*) AS times_excluded
FROM customer_orders_cleaned co
CROSS JOIN LATERAL unnest(string_to_array(co.exclusions, ', ')) AS e(topping_id)
INNER JOIN pizza_runner.pizza_toppings pt
    ON pt.topping_id = e.topping_id::INT
WHERE co.exclusions IS NOT NULL
GROUP BY pt.topping_name
ORDER BY times_excluded DESC
LIMIT 1;
```

**Result:**

| topping_name | times_excluded |
|-------------|---------------|
| Cheese | 4 |

> 💡 Cheese is the most excluded ingredient — consider offering a "no cheese" option prominently in the app.

---

## D. Pricing and Ratings

### D1. How much money has Pizza Runner made so far (no delivery fees)?

```sql
SELECT
    CONCAT('$', SUM(
        CASE WHEN pn.pizza_name = 'Meatlovers' THEN 12
             ELSE 10
        END
    )) AS total_revenue
FROM customer_orders_cleaned co
INNER JOIN runner_orders_cleaned ro
    ON co.order_id = ro.order_id
INNER JOIN pizza_runner.pizza_names pn
    ON co.pizza_id = pn.pizza_id
WHERE ro.cancellation IS NULL;
```

**Result:**

| total_revenue |
|--------------|
| $138 |

---

### D2. What if there was an additional $1 charge for each extra?

```sql
SELECT
    CONCAT('$', SUM(
        CASE WHEN pn.pizza_name = 'Meatlovers' THEN 12 ELSE 10 END
        + COALESCE(array_length(string_to_array(co.extras, ', '), 1), 0)
    )) AS total_revenue_with_extras
FROM customer_orders_cleaned co
INNER JOIN runner_orders_cleaned ro
    ON co.order_id = ro.order_id
INNER JOIN pizza_runner.pizza_names pn
    ON co.pizza_id = pn.pizza_id
WHERE ro.cancellation IS NULL;
```

**Result:**

| total_revenue_with_extras |
|--------------------------|
| $142 |

> 💡 Charging $1 per extra adds $4 to total revenue.

---

### D3. Design a ratings table schema

```sql
CREATE TABLE pizza_runner.ratings (
    rating_id    SERIAL PRIMARY KEY,
    order_id     INTEGER     NOT NULL,
    runner_id    INTEGER     NOT NULL,
    customer_id  INTEGER     NOT NULL,
    rating       INTEGER     CHECK (rating BETWEEN 1 AND 5),
    rated_at     TIMESTAMP   DEFAULT NOW(),
    comments     TEXT
);

-- Insert sample ratings for successful deliveries
INSERT INTO pizza_runner.ratings
    (order_id, runner_id, customer_id, rating, comments)
VALUES
    (1,  1, 101, 5, 'Super fast delivery!'),
    (2,  1, 101, 4, 'Pizza was still hot'),
    (3,  1, 102, 4, NULL),
    (4,  2, 103, 3, 'Took a while but pizza was good'),
    (5,  3, 104, 5, 'Great service'),
    (7,  2, 105, 4, NULL),
    (8,  2, 102, 4, 'Quick delivery'),
    (10, 1, 104, 5, 'Perfect order!');
```

---

### D4. Full successful delivery summary table

```sql
SELECT
    co.customer_id,
    ro.order_id,
    ro.runner_id,
    r.rating,
    co.order_time,
    ro.pickup_time::TIMESTAMP                                           AS pickup_time,
    ROUND(EXTRACT(EPOCH FROM (
        ro.pickup_time::TIMESTAMP - co.order_time)) / 60, 2)           AS mins_to_pickup,
    ro.duration                                                         AS delivery_duration_mins,
    ROUND(ro.distance / (ro.duration / 60.0), 2)                       AS avg_speed_kmh,
    COUNT(co.pizza_id)                                                  AS total_pizzas
FROM customer_orders_cleaned co
INNER JOIN runner_orders_cleaned ro
    ON co.order_id = ro.order_id
INNER JOIN pizza_runner.ratings r
    ON ro.order_id = r.order_id
WHERE ro.cancellation IS NULL
GROUP BY
    co.customer_id, ro.order_id, ro.runner_id, r.rating,
    co.order_time, ro.pickup_time, ro.duration, ro.distance
ORDER BY ro.order_id;
```

---

### D5. How much money is left after paying runners $0.30/km?

```sql
SELECT
    CONCAT('$', ROUND(
        SUM(CASE WHEN pn.pizza_name = 'Meatlovers' THEN 12 ELSE 10 END)
        - SUM(ro.distance * 0.30)
    , 2)) AS profit_after_runner_fees
FROM customer_orders_cleaned co
INNER JOIN runner_orders_cleaned ro
    ON co.order_id = ro.order_id
INNER JOIN pizza_runner.pizza_names pn
    ON co.pizza_id = pn.pizza_id
WHERE ro.cancellation IS NULL;
```

**Result:**

| profit_after_runner_fees |
|-------------------------|
| $94.44 |

> 💡 After paying runners, Pizza Runner keeps $94.44 — about 68% margin.

---

## E. Bonus — Add a new Supreme Pizza

```sql
-- Add to pizza_names
INSERT INTO pizza_runner.pizza_names (pizza_id, pizza_name)
VALUES (3, 'Supreme');

-- Add to pizza_recipes with all toppings (1–12)
INSERT INTO pizza_runner.pizza_recipes (pizza_id, toppings)
VALUES (3, '1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12');
```

> 💡 Adding a Supreme pizza only requires 2 INSERT statements — the existing schema handles it cleanly with no structural changes needed.

---

## SQL Techniques Used

| Section | Techniques |
|---------|-----------|
| Data Cleaning | CASE WHEN, REGEXP_REPLACE, CREATE TEMP TABLE |
| A - Pizza Metrics | COUNT, DISTINCT, GROUP BY, EXTRACT, TO_CHAR, CASE WHEN |
| B - Runner Experience | AVG, EXTRACT(EPOCH), ROUND, arithmetic on timestamps |
| C - Ingredients | STRING_AGG, unnest, string_to_array, CROSS JOIN LATERAL |
| D - Pricing | CASE WHEN pricing, COALESCE, array_length, JOIN ratings table |
| E - Bonus | INSERT INTO, schema design |
