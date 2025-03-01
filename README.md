# Danny's Diner
A case study on Danny's Diner using MySQL.
The following are the business questions and how I solved them.
```sql

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

  SELECT * FROM members;
  SELECT * FROM menu;
  SELECT * FROM sales;

-- 1.What is the total amount each customer spent at the restaurant?
SELECT s.customer_id, SUM(m.price) AS amount_spent
FROM sales AS s
INNER JOIN menu AS m
ON s.product_id = m.product_id
GROUP BY s.customer_id;

-- 2.How many days has each customer visited the restaurant?
SELECT customer_id, COUNT(DISTINCT order_date) AS days_visited
FROM sales
GROUP BY customer_id;

-- 3.What was the first item from the menu purchased by each customer?
WITH order_rank AS(
SELECT customer_id, 
       product_id, 
       ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY order_date) AS row_num
FROM sales)
SELECT o.customer_id, m.product_name AS first_product
FROM order_rank AS o
INNER JOIN menu AS m
ON o.product_id = m.product_id
WHERE o.row_num = 1;


-- 4.What is the most purchased item on the menu and how many times was it purchased by all customers?
SELECT TOP 1 m.product_name, COUNT(s.product_id) AS count_purchase
FROM menu m
INNER JOIN sales s
ON m.product_id = s.product_id
GROUP BY product_name
ORDER BY count_purchase DESC;

-- 5.Which item was the most popular for each customer?
WITH item_rank AS(
	SELECT s.customer_id AS customer,
		   m.product_name AS popular_product,
		  COUNT(*) AS order_count,
		  DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY COUNT(s.product_id) DESC) AS rownum
	FROM sales s
	INNER JOIN menu m
	ON s.product_id = m.product_id
	GROUP BY s.customer_id, 
		   m.product_name)
SELECT customer, popular_product
FROM item_rank
WHERE rownum = 1;

-- 6.Which item was purchased first by the customer after they became a member?
WITH purchases AS(
SELECT s.customer_id,
      m.product_name,
	  mb.join_date,
	  s.order_date,
	  DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS row_number
FROM members AS mb
INNER JOIN sales AS s
ON mb.customer_id = s.customer_id
INNER JOIN menu AS m
ON s.product_id = m.product_id
WHERE s.order_date > mb.join_date
)
SELECT customer_id, product_name
FROM purchases
WHERE row_number = 1;


-- 7.Which item was purchased just before the customer became a member?
WITH purchases AS(
SELECT s.customer_id,
      m.product_name,
	  mb.join_date,
	  s.order_date,
	  DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS row_number
FROM members AS mb
INNER JOIN sales AS s
ON mb.customer_id = s.customer_id
INNER JOIN menu AS m
ON s.product_id = m.product_id
WHERE s.order_date < mb.join_date
)
SELECT customer_id, product_name
FROM purchases
WHERE row_number = 1;
-- 8.What is the total items and amount spent for each member before they became a member?
WITH purchases AS(
SELECT s.customer_id,
      m.product_name,
	  mb.join_date,
	  s.order_date,
	  m.price,
	  DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS row_number
FROM members AS mb
INNER JOIN sales AS s
ON mb.customer_id = s.customer_id
INNER JOIN menu AS m
ON s.product_id = m.product_id
WHERE s.order_date < mb.join_date
)
SELECT customer_id, COUNT(product_name) AS items_bought, SUM(price) AS amount_spent
FROM purchases
GROUP BY customer_id;
-- 9.If each $1 spent equates to 10 points and sushi has a 2x points multiplier - 
-- how many points would each customer have?
USE dannys_diner;

WITH ranks AS(
SELECT s.customer_id, 
       m.product_name, 
	   SUM(m.price) AS amount_spent,
	   ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY SUM(m.price)) AS row_num
FROM sales s
INNER JOIN menu m
ON s.product_id = m.product_id
GROUP BY s.customer_id, m.product_name),
points AS(
SELECT customer_id,
       CASE WHEN product_name = 'sushi' THEN amount_spent * 20 ELSE amount_spent * 10 END AS points
FROM ranks)
SELECT customer_id, SUM(points) AS total_points
FROM points
GROUP BY customer_id;
-- 10.In the first week after a customer joins the program (including their join date) 
-- they earn 2x points on all items, not just sushi - 
-- how many points do customer A and B have at the end of January?
WITH points AS(
SELECT s.customer_id,
       CASE 
	       WHEN s.order_date BETWEEN mb.join_date AND DATEADD(day,7,mb.join_date) THEN m.price*10*2
		   WHEN m.product_name = 'sushi' THEN m.price*10*2
		   ELSE m.price*10 END AS points
FROM sales s
INNER JOIN members mb
ON s.customer_id = mb.customer_id
INNER JOIN menu m
ON s.product_id = m.product_id
WHERE s.order_date < '2021-02-01')
SELECT customer_id, SUM(points) AS total_points
FROM points
GROUP BY customer_id;

-- bonus question 11: determine the name and price of the product ordered by each customer on all order dates
-- and find out whether the customer was a member on the date or not

SELECT s.customer_id, 
       m.product_name,
	   m.price,
	   s.order_date,
	   CASE WHEN s.order_date >= mb.join_date THEN 'Y'
	        ELSE 'N' END AS member
FROM sales s
LEFT JOIN members mb
ON s.customer_id = mb.customer_id
INNER JOIN menu m
ON s.product_id = m.product_id;
      

-- bonus question 12: rank the previous output from Q11 based on the order date from each customer. 
-- display null if customer was not a member when dish was ordered. 
WITH customers AS(
SELECT s.customer_id, 
       m.product_name,
	   m.price,
	   s.order_date,
	   CASE WHEN s.order_date >= mb.join_date THEN 'Y'
	        ELSE 'N' END AS member
FROM sales s
LEFT JOIN members mb
ON s.customer_id = mb.customer_id
INNER JOIN menu m
ON s.product_id = m.product_id
)
SELECT *,
    CASE WHEN member = 'Y' THEN RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date) 
	     ELSE NULL END AS ranking
FROM customers;
```
