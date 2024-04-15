# Case Study #1: Danny's Diner
## Introduction
Danny’s Diner is in need of your assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite.

***

## Entity Relationship Diagram

![image](LINK)

***

## Questions and Solutions

Note that all queries were executed with MySQL

**1. What is the total amount each customer spent at the restaurant?

#### Steps:
- We will use **JOIN** to merge `dannys_diner.sales` and `dannys_diner.menu` tables together. This will give us data on which items were sold and the price per item.
- **SUM** is used to total the price of each sale.
- Lastly, we use ``GROUP BY s.customer_id`` to group the aggregated results by customer.

````sql
SELECT
s.customer_id,
SUM(m.price) total_spending
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY total_spending DESC;
````

#### Results:

| customer_id | total_spending |
| ----------- | -------------- |
| A           | 76             |
| B           | 74             |
| C           | 36             |

****

**2. How many days has each customer visited the restaurant?

#### Steps:
- Use **COUNT** to count each day an order was placed.
- **DISTINCT*** is used in ``COUNT(DISTINCT order_date)`` to avoid counting days where multiple purchases were made (duplicate days).
- Again, using **GROUP BY** to group the aggregated results.

````sql
SELECT
customer_id,
COUNT(DISTINCT order_date) visit_count
FROM dannys_diner.sales
GROUP BY customer_id
ORDER BY visit_count;
````

#### Results:
| customer_id | visit_count |
| ----------- | ----------- |
| B           | 6           |
| A           | 4           |
| C           | 2           |
****

**3. What was the first item from the menu purchased by each customer?

*Because we only have the date of each order, and not the time, we will show any order made on the earliest date as the first order.*
#### Steps:
- Create a Common Table Expression (CTE) using **DENSE_RANK** to create a new column that assigns a rank to each order, by date. Our new column ``order_rank`` will output a 1 for each order a customer made on their earliest ``order_date``.
- Now we will create an outer query where we **JOIN** ``dannys_diner.menu`` to ``dannys_diner.sales``. This allows us to display both the ``customer_id`` and ``product_name``. 
- Finally we use ``WHERE s.order_rank = 1`` so that we only show orders from each customers earliest ``order_date`` as explained in the first step.

````sql
WITH sales_order AS (
SELECT
customer_id,
product_id,
order_date,
DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date ASC) order_rank
FROM dannys_diner.sales
)

SELECT
s.customer_id,
m.product_name
FROM sales_order s
LEFT JOIN dannys_diner.menu m
ON s.product_id = m.product_id
WHERE s.order_rank = 1
GROUP BY s.customer_id, m.product_name;
````

#### Results:
| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| A           | curry        |
| B           | curry        |
| C           | ramen        |
#### Note:
- We can use **GROUP_CONCAT** to display each customer's first order on a single row.

````sql
WITH sales_order AS (
SELECT
customer_id,
product_id,
order_date,
DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date ASC) order_rank
FROM dannys_diner.sales
)

SELECT
s.customer_id,
GROUP_CONCAT(DISTINCT m.product_name ORDER BY m.product_name SEPARATOR ', ') first_order
FROM sales_order s
LEFT JOIN dannys_diner.menu m
ON s.product_id = m.product_id
WHERE s.order_rank = 1
GROUP BY s.customer_id;
````

#### Results:
| customer_id | first_order  |
| ----------- | ------------ |
| A           | curry, sushi |
| B           | curry        |
| C           | ramen        |

****

**4. What is the most purchased item on the menu and how many times was it purchased by all customers?
#### Steps:
- Using ``COUNT(s.product_id)`` and ``GROUP BY m.product_name`` we can find how many times each item was purchased.
- At the end of our code we use ``ORDER BY purchase_count DESC`` to sort by the highest count first. Then ``LIMIT 1`` to only show the first row.

````sql
SELECT
m.product_name,
COUNT(s.product_id) purchase_count
FROM dannys_diner.sales s
LEFT JOIN dannys_diner.menu m
ON s.product_id = m.product_id
GROUP BY m.product_name
ORDER BY purchase_count DESC
LIMIT 1;
````

#### Results:
| product_name | purchase_count |
| ------------ | -------------- |
| ramen        | 8              |

#### Note:
- The above code does not account for ties for the highest ``purchase_count``. The below code can be used to show any ties
- **DENSE_RANK** is utilized to rank our count of purchases. This will return any ties in count as 1.
- We then use ``WHERE s.to_ranking = 1`` to filter out any results without the highest count.

````sql
SELECT
m.product_name,
s.purchase_count
FROM (
SELECT
product_id,
COUNT(*) purchase_count,
DENSE_RANK() OVER(ORDER BY COUNT(*) DESC) to_ranking
FROM dannys_diner.sales
GROUP BY product_id
) s
LEFT JOIN dannys_diner.menu m
ON s.product_id = m.product_id
WHERE s.to_ranking = 1;
````

#### Results:
| product_name | purchase_count |
| ------------ | -------------- |
| ramen        | 8              |

****

**5. Which item was the most popular for each customer?
#### Steps:
- We create a CTE named `orders`. Within the CTE, **COUNT** will find the amount of times each item was purchased by each customer.
- **DENSE_RANK** will create a column ``to_ranking`` that ranks based off how many times each product was purchased by each customer.
- Outside of our subquery, we can use `WHERE o.to_ranking = 1` to only show the product(s) purchased the most by each customer. 

````sql
WITH orders AS (
SELECT
customer_id,
product_id,
COUNT(product_id) times_purchased,
DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY COUNT(product_id) DESC) to_ranking
FROM dannys_diner.sales
GROUP BY customer_id, product_id
)

SELECT
o.customer_id,
m.product_name,
o.times_purchased
FROM orders o
LEFT JOIN dannys_diner.menu m
ON o.product_id = m.product_id
WHERE o.to_ranking = 1;
````

#### Results:
| customer_id | product_name | times_purchased |
| ----------- | ------------ | --------------- |
| A           | ramen        | 3               |
| B           | curry        | 2               |
| B           | sushi        | 2               |
| B           | ramen        | 2               |
| C           | ramen        | 3               |
#### Note:
- **GROUP_CONCAT** can be used to return the results so that each customer only has one row, similar to Question 3.

****

**6. Which item was purchased first by the customer after they became a member?

*We assume that any purchases made on a customers join date were made after they became a member.*
#### Steps:
- A CTE named `sales_order` is created to rank each member's order, by date, after they became a member.
- We ``JOIN dannys_diner.members`` to find when each member joined. By joining on `s.order_date >= mem.join_date` we only join orders that are made on, or after, a customer became a member.
- **DENSE_RANK** is used to create a ranking `order_rank`, by date, for each member's order. Because of our **ON** clause, there will not be any orders, made before a member joined, to rank.
- Outside our subquery, we **SELECT** the information we want to show and use **WHERE** to only show orders ranked as 1 with our **DENSE_RANK**.

````sql
WITH sales_order AS (
SELECT
s.customer_id,
s.product_id,
s.order_date,
mem.join_date,
DENSE_RANK() OVER(PARTITION BY mem.customer_id ORDER BY s.order_date ASC) order_rank
FROM dannys_diner.sales s
JOIN dannys_diner.members mem
ON s.customer_id = mem.customer_id
AND s.order_date >= mem.join_date
)

SELECT
s.customer_id,
m.product_name
FROM sales_order s
LEFT JOIN dannys_diner.menu m
ON s.product_id = m.product_id
WHERE s.order_rank = 1
ORDER BY s.customer_id;
````

#### Results:
| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| B           | sushi        |

****

**7. Which item was purchased just before the customer became a member?

*We assume that any purchases made on a customers join date were made after they became a member.*
#### Steps:
- Since this question is similar to Question #6, we can start with the same query and make a few changes.
- First we want to change the **ON** clause in our CTE `sales_order` to join on order dates made before the customer's `join_date`. This is seen in `AND s.order_date < mem.join_date`. 
- In our **DENSE_RANK** we change our `ORDER BY` to `DESC` so that we will rank by our latest possible dates before a customer became a member.

````sql
WITH sales_order AS (
SELECT
s.customer_id,
s.product_id,
s.order_date,
mem.join_date,
DENSE_RANK() OVER(PARTITION BY mem.customer_id ORDER BY s.order_date DESC) order_rank
FROM dannys_diner.sales s
LEFT JOIN dannys_diner.members mem
ON s.customer_id = mem.customer_id
AND s.order_date < mem.join_date
)

SELECT
s.customer_id,
m.product_name
FROM sales_order s
LEFT JOIN dannys_diner.menu m
ON s.product_id = m.product_id
WHERE s.order_rank = 1
ORDER BY s.customer_id;
````

#### Results:
| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| A           | curry        |
| B           | ramen        |
| B           | sushi        |

**8. What is the total items and amount spent for each member before they became a member?

*We assume that any purchases made on a customers join date were made after they became a member.*
#### Steps:
 - We will **JOIN** the `dannys_diner.members` and `dannys_diner.menu` tables. To filter out any orders made after a customer became a member, we will `JOIN dannys_diner.members` `ON s.order_date < mem.join_date`. 
 - By using `GROUP BY s.customer_id` and `SUM(m.price)` we will add the price of all orders made by each customer.
 - A dollar sign is added in front of our **SUM** by using **CONCAT**.

````sql
SELECT
s.customer_id,
CONCAT('$', SUM(m.price)) total_spent
FROM dannys_diner.sales s
JOIN dannys_diner.members mem
ON s.customer_id = mem.customer_id
JOIN dannys_diner.menu m
ON s.product_id = m.product_id
WHERE s.order_date < mem.join_date
GROUP BY s.customer_id
ORDER BY total_spent;
````

#### Results:
| customer_id | total_spent |
| ----------- | ----------- |
| B           | $40         |
| A           | $25         |

****

**9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

*We assume that only members can earn points and they start earning once they joined. We also assume that any purchases made on a customers join date were made after they became a member.*
#### Steps:
- To begin we join `dannys_diner.menu` for our price value and `dannys_diner.members` so that we only have orders of members after they joined. 
- To find the amount of points each ordered item is worth we need to take:
	- The price of sushi x20
	- And the price of non-sushi x10
	-We can use a conditional **CASE** statement to perform the calculation.
- Now we perform a **SUM** to calculate the total points.

````sql
SELECT
s.customer_id,
SUM(CASE
WHEN m.product_name LIKE 'sushi'
THEN m.price * 20
ELSE m.price * 10
END) total_points
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
ON s.product_id = m.product_id
JOIN dannys_diner.members mem
ON s.customer_id = mem.customer_id
AND s.order_date >= mem.join_date
GROUP BY s.customer_id
ORDER BY total_points DESC;
````

#### Results:
| customer_id | total_points |
| ----------- | ------------ |
| A           | 510          |
| B           | 440          |

****

**10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
#### Steps:
- Let's start with the query used in Question #9.
- Adding a condition to our **CASE** statement let's us apply double points (x20) to any order date within 7 days of when the member joined. `s.order_date BETWEEN mem.join_date AND (mem.join_date +7)`
- Since we want to see the amount of points customer A and B have by the end of January, we will add the following **WHERE** clause:
	- `s.order_date < '2021-02-01`
	- `s.customer_id IN ('A','B')`

````sql
SELECT
s.customer_id,
SUM(CASE
WHEN s.order_date BETWEEN mem.join_date AND (mem.join_date +7)
THEN m.price * 20
ELSE m.price * 10
END) total_points
FROM dannys_diner.members mem
LEFT JOIN dannys_diner.sales s
ON s.customer_id = mem.customer_id
JOIN dannys_diner.menu m
ON s.product_id = m.product_id
WHERE s.order_date >= mem.join_date
AND s.order_date < '2021-02-01'
AND s.customer_id IN ('A','B')
GROUP BY s.customer_id
ORDER BY total_points DESC;
````

#### Results:

| customer_id | total_points |
| ----------- | ------------ |
| A           | 1020         |
| B           | 440          |
