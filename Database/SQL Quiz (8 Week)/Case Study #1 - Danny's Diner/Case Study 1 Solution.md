#  Case Study #1 – Danny’s Diner
  
## Question 1
  
### What is the total amount each customer spent at the restaurant?
  
```sql
SELECT s.customer_id, SUM(m.price) AS amount_spent
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY amount_spent DESC;
```
  
###  Screenshot
![alt text](image.png )
  
  
---
  
## Question 2
  
### How many days has each customer visited the restaurant?
  
```sql
SELECT s.customer_id,
       COUNT(DISTINCT order_date) AS visit_days
FROM dannys_diner.sales s
GROUP BY customer_id;
```
  
###  Screenshot
![alt text](image-2.png )
  
---
  
## Question 3
  
### What was the first item from the menu purchased by each customer?
  
```sql
SELECT DISTINCT s.customer_id,
       s.order_date,
       m.product_name
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
ON s.product_id = m.product_id
ORDER BY s.order_date
LIMIT 3;
```
  
###  Screenshot
![alt text](image-3.png )
  
---
  
##  Question 4
  
### What is the most purchased item on the menu and how many times was it purchased by all customers?
  
```sql
SELECT COUNT(s.product_id) AS countt,
       m.product_name
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
ON s.product_id = m.product_id
GROUP BY m.product_name
ORDER BY countt DESC
LIMIT 1;
```
  
### Screenshot
![alt text](image-4.png )
  
---
  
##  Question 5
  
### Which item was the most popular for each customer?
  
```sql
WITH cte AS (
    SELECT s.customer_id,
           m.product_name,
           COUNT(*) AS o_count,
           RANK() OVER (
               PARTITION BY s.customer_id
               ORDER BY COUNT(*) DESC
           ) AS rankk
    FROM dannys_diner.sales s
    JOIN dannys_diner.menu m
    ON s.product_id = m.product_id
    GROUP BY s.customer_id, m.product_name
)
SELECT customer_id, product_name, o_count
FROM cte
WHERE rankk = 1
ORDER BY customer_id;
```
  
### Screenshot
![alt text](image-5.png )
  
---
  
##  Question 6
  
### Which item was purchased first by the customer after they became a member?
  
```sql
WITH cte AS (
    SELECT s.customer_id,
           s.product_id,
           ROW_NUMBER() OVER (
               PARTITION BY s.customer_id
               ORDER BY s.order_date
           ) AS row_num
    FROM dannys_diner.members mb
    JOIN dannys_diner.sales s
    ON mb.customer_id = s.customer_id
    AND s.order_date > mb.join_date
)
SELECT cte.customer_id, m.product_name
FROM cte
JOIN dannys_diner.menu m
ON cte.product_id = m.product_id
WHERE row_num = 1;
```
  
### Screenshot
![alt text](image-6.png )
  
---
  
##  Question 7
  
### Which item was purchased just before the customer became a member?
  
```sql
WITH cte AS (
    SELECT s.customer_id,
           s.product_id,
           ROW_NUMBER() OVER (
               PARTITION BY s.customer_id
               ORDER BY s.order_date DESC
           ) AS row_num
    FROM dannys_diner.members mb
    JOIN dannys_diner.sales s
    ON mb.customer_id = s.customer_id
    AND s.order_date < mb.join_date
)
SELECT cte.customer_id, m.product_name
FROM cte
JOIN dannys_diner.menu m
ON cte.product_id = m.product_id
WHERE row_num = 1;
```
  
###  Screenshot
![alt text](image-7.png )
  
---
  
##  Question 8
  
### What is the total items and amount spent for each member before they became a member?
  
```sql
WITH cte AS (
    SELECT s.customer_id,
           s.product_id
    FROM dannys_diner.members mb
    JOIN dannys_diner.sales s
    ON mb.customer_id = s.customer_id
    AND s.order_date < mb.join_date
)
SELECT cte.customer_id,
       COUNT(*) AS total_items,
       SUM(m.price) AS total_spent
FROM cte
JOIN dannys_diner.menu m
ON cte.product_id = m.product_id
GROUP BY cte.customer_id;
```
  
### Screenshot
![alt text](image-9.png )
  
---
  
## Question 9
  
### If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
  
```sql
WITH cte AS (
    SELECT product_id,
           CASE
               WHEN product_id = 1 THEN price * 20
               ELSE price * 10
           END AS points
    FROM dannys_diner.menu
)
SELECT s.customer_id,
       SUM(cte.points) AS points_earned
FROM dannys_diner.sales s
JOIN cte
ON s.product_id = cte.product_id
GROUP BY s.customer_id;
```
  
### Screenshot
![alt text](image-10.png )
  
---
  
## Question 10
  
### In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
  
```sql
WITH customer_points AS (
    SELECT
        m.customer_id,
        s.order_date,
        m.join_date,
        menu.price,
        CASE
            WHEN s.order_date BETWEEN m.join_date AND (m.join_date + INTERVAL '6 DAY')
                THEN 2 * 10 * menu.price
            WHEN menu.product_name = 'sushi'
                THEN 2 * 10 * menu.price
            ELSE 10 * menu.price
        END AS points
    FROM dannys_diner.sales s
    JOIN dannys_diner.members m
        ON s.customer_id = m.customer_id
    JOIN dannys_diner.menu menu
        ON s.product_id = menu.product_id
    WHERE s.order_date <= '2021-01-31'
)
SELECT customer_id,
       SUM(points) AS total_points
FROM customer_points
GROUP BY customer_id
ORDER BY customer_id;
```
  
###  Screenshot
![alt text](image-11.png )
  
  