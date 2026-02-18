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
<img width="293" height="117" alt="image" src="https://github.com/user-attachments/assets/6a373f71-8c8a-4cc0-8a54-8231a566e20e" />

  
  
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
<img width="267" height="118" alt="image" src="https://github.com/user-attachments/assets/125329e5-1abb-4536-9ad2-66607377cda8" />

  
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
<img width="415" height="124" alt="image" src="https://github.com/user-attachments/assets/345a866c-c9b8-455f-868d-f03f2a7c1e96" />

  
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
<img width="254" height="73" alt="image" src="https://github.com/user-attachments/assets/a785c0ee-36f4-4ae4-8424-b31b4a7efc40" />

  
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
<img width="405" height="166" alt="image" src="https://github.com/user-attachments/assets/7a0b7b38-a54d-40f7-a487-ca58fce69a99" />

  
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
<img width="335" height="97" alt="image" src="https://github.com/user-attachments/assets/42d63f8e-bb37-406f-9dd4-126e14af07d5" />

  
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
<img width="330" height="104" alt="image" src="https://github.com/user-attachments/assets/5821b1ff-78b4-4475-8535-41ff78a753a9" />

  
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
<img width="371" height="96" alt="image" src="https://github.com/user-attachments/assets/5f34f025-10e2-4f57-9e7c-cc2f368e78fa" />

  
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
<img width="290" height="119" alt="image" src="https://github.com/user-attachments/assets/67cfa5d3-8eb9-468c-86df-a74eedb4486c" />

  
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
<img width="280" height="92" alt="image" src="https://github.com/user-attachments/assets/11497f40-3951-4802-b943-cd7746079818" />

  
  
