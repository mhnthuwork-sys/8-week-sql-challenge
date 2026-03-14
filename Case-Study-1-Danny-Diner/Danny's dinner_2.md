# 🍜 Case Study #1 - Danny's Diner
## Solution — Q6 to Q10

### Tables Used
- **sales** — customer_id, order_date, product_id
- **menu** — product_id, product_name, price
- **members** — customer_id, join_date

---

## Q6. Which item was purchased first by the customer after they became a member?

```sql
WITH member_sales AS (
    SELECT
        sales.customer_id,
        sales.order_date,
        menu.product_name,
        DENSE_RANK() OVER (
            PARTITION BY sales.customer_id
            ORDER BY sales.order_date
        ) AS rank
    FROM dannys_diner.sales
    INNER JOIN dannys_diner.menu
        ON sales.product_id = menu.product_id
    INNER JOIN dannys_diner.members
        ON sales.customer_id = members.customer_id
    WHERE sales.order_date >= members.join_date
)
SELECT
    customer_id,
    product_name
FROM member_sales
WHERE rank = 1
ORDER BY customer_id ASC;
```

**Steps:**
- Use **JOIN** to merge all 3 tables — `sales`, `menu`, and `members`.
- Filter `WHERE order_date >= join_date` to only keep purchases **after** becoming a member.
- Use **DENSE_RANK()** over `PARTITION BY customer_id ORDER BY order_date` to rank purchases per customer.
- Filter `WHERE rank = 1` to get the first purchase after joining.

**Result:**

| customer_id | product_name |
|-------------|-------------|
| A | curry |
| B | sushi |

> 💡 After joining, Customer A first ordered curry and Customer B ordered sushi.

---

## Q7. Which item was purchased just before the customer became a member?

```sql
WITH before_member AS (
    SELECT
        sales.customer_id,
        sales.order_date,
        menu.product_name,
        DENSE_RANK() OVER (
            PARTITION BY sales.customer_id
            ORDER BY sales.order_date DESC
        ) AS rank
    FROM dannys_diner.sales
    INNER JOIN dannys_diner.menu
        ON sales.product_id = menu.product_id
    INNER JOIN dannys_diner.members
        ON sales.customer_id = members.customer_id
    WHERE sales.order_date < members.join_date
)
SELECT
    customer_id,
    product_name
FROM before_member
WHERE rank = 1
ORDER BY customer_id ASC;
```

**Steps:**
- Filter `WHERE order_date < join_date` to only keep purchases **before** becoming a member.
- Use **DENSE_RANK()** with `ORDER BY order_date DESC` — ranking from newest to oldest so rank 1 = most recent purchase before joining.
- Filter `WHERE rank = 1` to get the last purchase before joining.

**Result:**

| customer_id | product_name |
|-------------|-------------|
| A | sushi |
| A | curry |
| B | sushi |

> 💡 Both customers ordered sushi just before becoming members — sushi might be the item that "converts" customers into members!

---

## Q8. What is the total items and amount spent for each member before they became a member?

```sql
SELECT
    sales.customer_id,
    COUNT(sales.product_id) AS total_items,
    SUM(menu.price) AS total_amount
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
    ON sales.product_id = menu.product_id
INNER JOIN dannys_diner.members
    ON sales.customer_id = members.customer_id
WHERE sales.order_date < members.join_date
GROUP BY sales.customer_id
ORDER BY sales.customer_id ASC;
```

**Steps:**
- Join all 3 tables together.
- Filter `WHERE order_date < join_date` to only count purchases made **before** joining.
- Use **COUNT** for total items and **SUM** for total amount spent.
- Group by `customer_id`.

**Result:**

| customer_id | total_items | total_amount |
|-------------|-------------|-------------|
| A | 2 | $25 |
| B | 3 | $40 |

> 💡 Before joining, Customer B already spent more ($40 across 3 items) vs Customer A ($25 across 2 items).

---

## Q9. If each $1 spent = 10 points and sushi has a 2x points multiplier — how many points would each customer have?

```sql
SELECT
    sales.customer_id,
    SUM(
        CASE
            WHEN menu.product_name = 'sushi' THEN menu.price * 10 * 2
            ELSE menu.price * 10
        END
    ) AS total_points
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
    ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id ASC;
```

**Steps:**
- Use **CASE WHEN** to apply 2x multiplier when `product_name = 'sushi'`, otherwise apply normal 10 points per $1.
- Use **SUM** to total up all points per customer.
- Group by `customer_id`.

**Points Logic:**
```
sushi  → price × 10 × 2  (2x multiplier)
curry  → price × 10
ramen  → price × 10
```

**Result:**

| customer_id | total_points |
|-------------|-------------|
| A | 860 |
| B | 940 |
| C | 360 |

> 💡 Customer B has the most points (940) despite spending less than A — because B ordered sushi more often, earning 2x points.

---

## Q10. In the first week after joining (including join date), customers earn 2x points on ALL items. How many points do A and B have at end of January?

```sql
SELECT
    sales.customer_id,
    SUM(
        CASE
            -- 2x points during first week membership (all items)
            WHEN sales.order_date BETWEEN members.join_date
                AND (members.join_date + INTERVAL '6 days')
                THEN menu.price * 10 * 2
            -- 2x points for sushi anytime
            WHEN menu.product_name = 'sushi'
                THEN menu.price * 10 * 2
            -- normal points otherwise
            ELSE menu.price * 10
        END
    ) AS total_points
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
    ON sales.product_id = menu.product_id
INNER JOIN dannys_diner.members
    ON sales.customer_id = members.customer_id
WHERE sales.order_date <= '2021-01-31'
GROUP BY sales.customer_id
ORDER BY sales.customer_id ASC;
```

**Steps:**
- Join all 3 tables.
- Filter `WHERE order_date <= '2021-01-31'` to only count January purchases.
- Use **CASE WHEN** with 3 conditions:
  - **First week** after joining (`join_date` to `join_date + 6 days`) → 2x points on ALL items
  - **Sushi** ordered outside first week → still 2x points
  - **Everything else** → normal 10 points per $1
- Use `join_date + INTERVAL '6 days'` to calculate the end of the first week.

**Points Logic:**
```
First week (all items) → price × 10 × 2
Sushi (any time)       → price × 10 × 2
Everything else        → price × 10
```

**Result:**

| customer_id | total_points |
|-------------|-------------|
| A | 1,370 |
| B | 820 |

> 💡 Customer A earned significantly more points (1,370) thanks to more purchases during the first week 2x bonus period.

---

## SQL Techniques Used

| Question | Techniques |
|----------|-----------|
| Q6 | CTE, DENSE_RANK, PARTITION BY, 3-table JOIN, WHERE date filter |
| Q7 | CTE, DENSE_RANK, ORDER BY DESC, 3-table JOIN, WHERE date filter |
| Q8 | 3-table JOIN, COUNT, SUM, WHERE date filter |
| Q9 | CASE WHEN, SUM, INNER JOIN |
| Q10 | CASE WHEN with date range, INTERVAL, 3-table JOIN, WHERE date filter |
