# Danny's Diner SQL Case Study

**Tools**
- PostgreSQL
- PGAdmin 4


# Introduction

Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favourite foods: sushi, curry and ramen.

Danny’s Diner is in need of your assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

## Problem Statement

Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

Danny has provided you with a sample of his overall customer data due to privacy issues - but he hopes that these examples are enough for you to write fully functioning SQL queries to help him answer his questions!

Danny has shared with you 3 key datasets for this case study:

- sales
- menu
- members
 






## Schema SQL
```sql
CREATE SCHEMA dannys_diner;
SET search_path = dannys_diner;

CREATE TABLE sales (
  "customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_id" INTEGER
);

INSERT INTO sales
  ("customer_id", "order_date", "product_id")
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  "product_id" INTEGER,
  "product_name" VARCHAR(5),
  "price" INTEGER
);

INSERT INTO menu
  ("product_id", "product_name", "price")
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');
```


# Case Study Questions / Answers

- 1. What is the total amount each customer spent at the restaurant?
```sql
SELECT 
    customer_id, 
    SUM(price) AS total_spent,
    CASE 
        WHEN SUM(price) > 50 THEN 'Regular Client'
        ELSE 'Walk-in Client'
    END AS CFLAG
FROM sales s
JOIN menu m ON s.product_id = m.product_id
GROUP BY customer_id
ORDER BY total_spent DESC;
```


- 2. How many days has each customer visited the restaurant?
```sql
SELECT
 	customer_id,
	COUNT(DISTINCT order_date) AS Visit_days
FROM sales
GROUP BY 1
ORDER BY 2 DESC
```


- 3. What was the first & last item from the menu purchased by each customer?
```sql
WITH First_item AS (
SELECT
	s.customer_id,
	s.order_date,
	m.product_name,
	ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) as rn
FROM sales s
JOIN menu m ON s.product_id = m.product_id
),

Last_item AS (
SELECT
	s.customer_id,
	s.order_date,
	m.product_name,
	ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date DESC) as rn
FROM sales s
JOIN menu m ON s.product_id = m.product_id
)

SELECT
	customer_id,
	order_date,
	Product_name,
	'First Item Purchased' AS Status
FROM
	First_item
WHERE
	rn = 1
	
UNION

SELECT
	customer_id,
	order_date,
	Product_name,
	'Last Item Purchased' AS Status
FROM
	Last_item
WHERE
	rn = 1
```


- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT 
    m.product_name, 
    COUNT(s.product_id) AS total_purchases
FROM sales s
JOIN menu m ON s.product_id = m.product_id
GROUP BY m.product_name
ORDER BY total_purchases DESC
LIMIT 1;
```


- 5. Which item was the most popular for each customer?

```sql
With First_step AS (
SELECT 
	customer_id, 
	m.product_name, 
	COUNT(s.product_id) AS Purchase_count
FROM sales s
JOIN menu m ON s.product_id = m.product_id
GROUP BY 1,2
ORDER BY 1,3 DESC
),

Second_step AS (
SELECT customer_id, product_name, Purchase_count,
	ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY purchase_count DESC) AS r
	FROM First_step
)

SELECT customer_id, Product_name, Purchase_count
FROM Second_step
WHERE r = 1
```


- 6. Which item was purchased first by the customer after they became a member?
```sql
WITH member_purchases AS (
    SELECT 
        s.customer_id,
        s.order_date,
        m.product_name,
        mem.join_date
    FROM sales s
    JOIN menu m ON s.product_id = m.product_id
    JOIN members mem ON s.customer_id = mem.customer_id
    WHERE s.order_date >= mem.join_date
),
ranked_purchases AS (
    SELECT 
        customer_id,
        product_name,
        order_date,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date ASC) AS r
    FROM member_purchases
)
SELECT 
    customer_id,
    product_name AS first_item_after_membership,
    order_date AS purchase_date
FROM ranked_purchases
WHERE r = 1;
```

- 7. Which item was purchased just before the customer became a member?
```sql
WITH member_purchases AS (
    SELECT 
        s.customer_id,
        s.order_date,
        m.product_name,
        mem.join_date
    FROM sales s
    JOIN menu m ON s.product_id = m.product_id
    JOIN members mem ON s.customer_id = mem.customer_id
    WHERE s.order_date < mem.join_date
),
ranked_purchases AS (
    SELECT 
        customer_id,
        product_name,
        order_date,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS r
    FROM member_purchases
)
SELECT 
    customer_id,
    product_name AS first_item_after_membership,
    order_date AS purchase_date
FROM ranked_purchases
WHERE r = 1;
```

- 8. What is the total items and amount spent for each member before they became a member?
```sql
SELECT 
    s.customer_id,
    COUNT(s.product_id) AS total_items,
    SUM(m.price) AS total_amount_spent
FROM sales s
JOIN menu m ON s.product_id = m.product_id
JOIN members mem ON s.customer_id = mem.customer_id
WHERE s.order_date < mem.join_date
GROUP BY s.customer_id
ORDER BY s.customer_id;
```


- 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier how many points would each customer have?
 ```sql
SELECT 
    s.customer_id,
    SUM(
        CASE 
            WHEN m.product_name = 'sushi' THEN m.price * 10 * 2  -- 2x points for sushi
            ELSE m.price * 10                                    -- Regular points for other items
        END
    ) AS total_points
FROM sales s
JOIN menu m ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id;
```

- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi how many points do customer A and B have at the end of January? how many points do customer A and B have at the end of January?
```sql
WITH purchases_within_january AS (
    SELECT 
        s.customer_id,
        s.order_date,
        m.product_name,
        m.price,
        mem.join_date,
        CASE 
            WHEN s.order_date BETWEEN mem.join_date AND mem.join_date + INTERVAL '6 days' THEN 2 -- 2x points for the first week
            WHEN m.product_name = 'sushi' THEN 2                                               -- 2x points for sushi outside the first week
            ELSE 1                                                                            -- Regular points for other items
        END AS multiplier
    FROM sales s
    JOIN menu m ON s.product_id = m.product_id
    JOIN members mem ON s.customer_id = mem.customer_id
    WHERE s.order_date <= '2021-01-31' -- Filter for January purchases
),
points_calculation AS (
    SELECT 
        customer_id,
        SUM(price * 10 * multiplier) AS total_points
    FROM purchases_within_january
    GROUP BY customer_id
)
SELECT 
    customer_id,
    total_points
FROM points_calculation
WHERE customer_id IN ('A', 'B'); -- Only include customers A and B
```







































































































