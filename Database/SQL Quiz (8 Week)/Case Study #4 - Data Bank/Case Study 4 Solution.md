# 8 Week SQL Challenge – Data Bank 🏦



# A. Customer Nodes Exploration

## 1. How many unique nodes are there on the Data Bank system?

```sql
select count(distinct(node_id)) 
from data_bank.customer_nodes;
```

*Output: To be added*

---

## 2. What is the number of nodes per region?

```sql
select r.region_name, count(distinct(c.node_id))
from data_bank.customer_nodes c
join data_bank.regions r 
  on c.region_id = r.region_id

group by r.region_name;
```

*Output: To be added*

---

## 3. How many customers are allocated to each region?

```sql
select r.region_name, count(c.customer_id)
from data_bank.customer_nodes c
join data_bank.regions r 
  on c.region_id = r.region_id

group by r.region_name;
```

*Output: To be added*

---

## 4. How many days on average are customers reallocated to a different node?

```sql
select Round(AVG (end_date - start_date)) as avg_days

from data_bank.customer_nodes

where end_date is not null 
  and end_date < '9999-12-31';
```

*Output: To be added*

---

## 5. Median, 80th and 95th percentile for node reallocation days for each region

```sql
SELECT

    percentile_cont(0.5) 
        WITHIN GROUP (ORDER BY (end_date - start_date)) AS median_days,

    percentile_cont(0.8) 
        WITHIN GROUP (ORDER BY (end_date - start_date)) AS p80_days,

    percentile_cont(0.95) 
        WITHIN GROUP (ORDER BY (end_date - start_date)) AS p95_days

FROM data_bank.customer_nodes

WHERE end_date IS NOT NULL
  AND end_date != '9999-12-31'

GROUP BY region_id;
```

*Output: To be added*

---

# B. Customer Transactions

## 1. Unique count and total amount for each transaction type

```sql
SELECT 
  txn_type, 
  count(customer_id), 
  sum(txn_amount)

from data_bank.customer_transactions

group by txn_type;
```

*Output: To be added*

---

## 2. Average total historical deposit counts and amounts for all customers

```sql
with cte as (

SELECT 
  count(customer_id) as txn_count, 
  sum(txn_amount) as total_amount

from data_bank.customer_transactions

where txn_type='deposit'

group by customer_id

)

select 
  round(avg(txn_count)) as avg_txn, 
  round(avg(total_amount)) as avg_total_amount 

from cte;
```

*Output: To be added*

---

## 3. For each month - customers making more than 1 deposit and either 1 purchase or withdrawal

```sql
WITH monthly_customer_stats AS (

    SELECT
        customer_id,

        DATE_TRUNC('month', txn_date) AS month_start,

        COUNT(CASE WHEN txn_type = 'deposit' THEN 1 END) AS deposit_count,

        COUNT(CASE WHEN txn_type = 'purchase' THEN 1 END) AS purchase_count,

        COUNT(CASE WHEN txn_type = 'withdrawal' THEN 1 END) AS withdrawal_count

    FROM data_bank.customer_transactions

    GROUP BY customer_id, month_start
)

SELECT

    month_start,

    COUNT(customer_id) AS total_customers

FROM monthly_customer_stats

WHERE

    deposit_count > 1

    AND (purchase_count >= 1 OR withdrawal_count >= 1)

GROUP BY month_start

ORDER BY month_start;
```

*Output: To be added*


