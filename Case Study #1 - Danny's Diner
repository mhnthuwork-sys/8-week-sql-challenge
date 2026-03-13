# 🍜 Case Study #1 - Danny's Diner
## Solution

### Tables Used
- **sales** — customer_id, order_date, product_id
- **menu** — product_id, product_name, price
- **members** — customer_id, join_date

---

## Q1. What is the total amount each customer spent at the restaurant?

```sql
SELECT
    sales.customer_id,
    SUM(menu.price) AS total_amount
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
    ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id ASC;
```

**Steps:**
- Use **JOIN** to merge `dannys_diner.sales` and `dannys_diner.menu` tables as `sales.customer_id` and `menu.price` are from both tables.
- Use **SUM** to calculate the total amount spent by each customer.
- Group the aggregated results by `sales.customer_id`.

**Result:**

| customer_id | total_amount |
|-------------|-------------|
| A | $76 |
| B | $74 |
| C | $36 |

> 💡 Customer A spent the most ($76), Customer C spent the least ($36).

---

## Q2. How many days has each customer visited the restaurant?

```sql
SELECT
    sales.customer_id,
    COUNT(DISTINCT sales.order_date) AS total_days_visited
FROM dannys_diner.sales
GROUP BY sales.customer_id
ORDER BY sales.customer_id ASC;
```

**Steps:**
- Use **COUNT(DISTINCT order_date)** to count unique visit days per customer.
- Using `COUNT` without `DISTINCT` would count duplicate days if a customer ordered multiple items on the same day.

**Result:**

| customer_id | total_days_visited |
|-------------|-------------------|
| A | 4 |
| B | 6 |
| C | 2 |

> 💡 Customer B visited the most days (6) despite spending less than A — more loyal but lower spend per visit.

---

## Q3. What was the first item from the menu purchased by each customer?

```sql
WITH ordered_sales AS (
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
)
SELECT DISTINCT
    customer_id,
    product_name
FROM ordered_sales
WHERE rank = 1
ORDER BY customer_id ASC;
```

**Steps:**
- Use a **CTE** to create a temporary table with rankings.
- Use **DENSE_RANK()** over `PARTITION BY customer_id ORDER BY order_date` to rank each purchase per customer by date.
- `DENSE_RANK` is used instead of `ROW_NUMBER` because if a customer ordered 2 items on the same day, both should be ranked 1st.
- Filter `WHERE rank = 1` to get only the first purchase(s).
- Use `DISTINCT` to avoid duplicates if the same item was ordered twice on the first day.

**Result:**

| customer_id | product_name |
|-------------|-------------|
| A | curry |
| A | sushi |
| B | curry |
| C | ramen |

> 💡 Customer A ordered 2 items on their first visit. Curry was the first item for both A and B.

---

## Q4. What is the most purchased item on the menu and how many times was it purchased by all customers?

```sql
SELECT
    menu.product_name,
    COUNT(sales.product_id) AS total_purchased
FROM dannys_diner.sales
INNER JOIN dannys_diner.menu
    ON sales.product_id = menu.product_id
GROUP BY menu.product_name
ORDER BY total_purchased DESC
LIMIT 1;
```

**Steps:**
- Use **COUNT** to count the number of times each item was purchased.
- Use **JOIN** to get `product_name` from the menu table.
- Use **ORDER BY DESC** + **LIMIT 1** to return only the most purchased item.

**Result:**

| product_name | total_purchased |
|-------------|----------------|
| ramen | 8 |

> 💡 Ramen is the most purchased item with 8 orders — should be prioritized in promotions.

---

## Q5. Which item was the most popular for each customer?

```sql
WITH most_popular AS (
    SELECT
        sales.customer_id,
        menu.product_name,
        COUNT(sales.product_id) AS order_count,
        DENSE_RANK() OVER (
            PARTITION BY sales.customer_id
            ORDER BY COUNT(sales.product_id) DESC
        ) AS rank
    FROM dannys_diner.sales
    INNER JOIN dannys_diner.menu
        ON sales.product_id = menu.product_id
    GROUP BY sales.customer_id, menu.product_name
)
SELECT
    customer_id,
    product_name,
    order_count
FROM most_popular
WHERE rank = 1
ORDER BY customer_id ASC;
```

**Steps:**
- Use a **CTE** to calculate order count per customer per item.
- Use **DENSE_RANK()** over `PARTITION BY customer_id ORDER BY COUNT DESC` to rank items by popularity per customer.
- `DENSE_RANK` is used so that if a customer ordered 2 items equally, both are returned.
- Filter `WHERE rank = 1` to get the most popular item(s) per customer.

**Result:**

| customer_id | product_name | order_count |
|-------------|-------------|-------------|
| A | ramen | 3 |
| B | ramen | 2 |
| B | curry | 2 |
| B | sushi | 2 |
| C | ramen | 3 |

> 💡 Ramen is the favourite for A and C. Customer B ordered all 3 items equally — an adventurous eater!

---

## SQL Techniques Used

| Question | Techniques |
|----------|-----------|
| Q1 | INNER JOIN, SUM, GROUP BY |
| Q2 | COUNT(DISTINCT) |
| Q3 | CTE, DENSE_RANK, PARTITION BY, INNER JOIN |
| Q4 | COUNT, ORDER BY DESC, LIMIT |
| Q5 | CTE, DENSE_RANK, PARTITION BY, GROUP BY |
