# 8 Week SQL Challenge – Foodie-Fi 🍜



# A. Customer Journey

The following query shows the onboarding journey for the sample customers by joining the subscriptions table with the plans table.

```sql
select s.customer_id, p.plan_name, s.start_date  
from foodie_fi.subscriptions s join  
Foodie_fi.plans p on s.plan_id = p.plan_id
order by s.customer_id, s.start_date
```

*Output: To be added*

---

# B. Data Analysis Questions

## 1. How many customers has Foodie-Fi ever had?

```sql
SELECT count(Distinct(subscriptions.customer_id))
from foodie_fi.subscriptions
```

*Output: To be added*

---

## 2. Monthly distribution of trial plan start_date

```sql
select extract(month from s.start_date) as monthh,
count(s.customer_id)
from foodie_fi.subscriptions s
join foodie_fi.plans p on s.plan_id = p.plan_id

group by monthh
order by monthh
```

*Output: To be added*

---

## 3. Plan start_date values after 2020 with breakdown by plan

```sql
select p.plan_name, count(s.customer_id) as countt

from foodie_fi.subscriptions s
join foodie_fi.plans p on s.plan_id = p.plan_id

where s.start_date > '2021-01-01'

group by p.plan_name
order by countt
```

*Output: To be added*

---

## 4. Customer count and percentage who have churned

```sql
SELECT
  COUNT(DISTINCT s.customer_id) AS churned,
  ROUND(
    COUNT(DISTINCT s.customer_id) * 100.0
    / (SELECT COUNT(DISTINCT customer_id) FROM foodie_fi.subscriptions),
    1
  ) AS churn_percentage

FROM foodie_fi.subscriptions s
JOIN foodie_fi.plans p
  ON s.plan_id = p.plan_id

WHERE p.plan_name = 'churn';
```

*Output: To be added*

---

## 5. Customers who churned straight after free trial

```sql
WITH ranked_cte AS (
  SELECT
    sub.customer_id,
    plans.plan_name,
    LEAD(plans.plan_name) OVER (
      PARTITION BY sub.customer_id
      ORDER BY sub.start_date
    ) AS next_plan

  FROM foodie_fi.subscriptions AS sub
  JOIN foodie_fi.plans
    ON sub.plan_id = plans.plan_id
)

SELECT
  COUNT(customer_id) AS churned_customers,

  ROUND(100.0 *
    COUNT(customer_id)
    / (SELECT COUNT(DISTINCT customer_id)
      FROM foodie_fi.subscriptions)
  ) AS churn_percentage

FROM ranked_cte

WHERE plan_name = 'trial'
  AND next_plan = 'churn';
```

*Output: To be added*

---

## 6. Number and percentage of customer plans after free trial

```sql
WITH next_plans AS (

  SELECT
    sub.customer_id,
    p.plan_name,

    LEAD(plan_name) OVER(
      PARTITION BY customer_id
      ORDER BY start_date
    ) as next_plan

  FROM foodie_fi.subscriptions sub

  join foodie_fi.plans p
    on sub.plan_id = p.plan_id
)

SELECT
  next_plan AS plan,
  COUNT(customer_id) AS converted_customers

FROM next_plans

WHERE next_plan IS NOT NULL
  AND plan_name = 'trial'

GROUP BY next_plan
ORDER BY converted_customers;
```

*Output: To be added*

---

## 7. Customer count and percentage breakdown of all plans at 2020-12-31

```sql
WITH cte as (

SELECT
customer_id,
plan_id,
start_date,

LEAD(start_date) OVER (
PARTITION by customer_id
order by start_date
) as nextplan

from foodie_fi.subscriptions s

where s.start_date < '2021-01-01'
)

Select p.plan_name , count(distinct(customer_id))

from cte

join foodie_fi.plans p
  on cte.plan_id = p.plan_id

where nextplan is null

group by p.plan_name
```

*Output: To be added*

---

## 8. Customers who upgraded to annual plan in 2020

```sql
WITH cte as (

SELECT
customer_id,
plan_id,
start_date,

LEAD(plan_id) OVER (
PARTITION by customer_id
order by start_date
) as nextplan

from foodie_fi.subscriptions s

where s.start_date < '2021-01-01'
)

Select count(distinct(customer_id))

from cte

join foodie_fi.plans p
  on cte.plan_id = p.plan_id

where nextplan =3
```

*Output: To be added*

---

## 9. Average number of days to upgrade to annual plan

```sql
with cte as(

select customer_id, start_date

from foodie_fi.subscriptions

where plan_id=0

),

ctetwo as (

select customer_id,start_date

from foodie_fi.subscriptions

where plan_id=3
)

select round(avg(ctetwo.start_date - cte.start_date),2)

from cte

join ctetwo

on cte.customer_id = ctetwo.customer_id
```

*Output: To be added*

---

## 10. Breakdown of upgrade time into 30-day buckets

```sql
WITH trial_plan AS (

  SELECT
    customer_id,
    start_date AS trial_date

  FROM foodie_fi.subscriptions

  WHERE plan_id = 0
),

annual_plan AS (

  SELECT
    customer_id,
    start_date AS annual_date

  FROM foodie_fi.subscriptions

  WHERE plan_id = 3
),

bins AS (

  SELECT

    WIDTH_BUCKET(annual.annual_date - trial.trial_date, 0, 360, 12)

      AS avg_days_to_upgrade

  FROM trial_plan AS trial

  JOIN annual_plan AS annual

    ON trial.customer_id = annual.customer_id
)

SELECT

((avg_days_to_upgrade - 1) * 30 || ' - ' || avg_days_to_upgrade * 30 || ' days') AS bucket,

COUNT(*) AS num_of_customers

FROM bins

GROUP BY avg_days_to_upgrade

ORDER BY avg_days_to_upgrade;
```

*Output: To be added*

---

## 11. Customers who downgraded from pro monthly to basic monthly in 2020

```sql
WITH next_plans AS (

  SELECT

    sub.customer_id,

    p.plan_name,

    LEAD(plan_name) OVER(

      PARTITION BY customer_id

      ORDER BY start_date) as next_plan

  FROM foodie_fi.subscriptions sub

  join foodie_fi.plans p

  on sub.plan_id = p.plan_id

  where sub.start_date < '2021-01-01'
)

SELECT

  COUNT(customer_id) AS downgraded_customers

FROM next_plans

WHERE next_plan = 'basic monthly'

  AND plan_name =  'pro monthly'

ORDER BY downgraded_customers;
```

*Output: To be added*

---

# C. Challenge Payment Question

The following query generates a **payments table for the year 2020** based on subscription upgrades, billing cycles and pricing rules.

```sql
WITH plan_history AS (

  SELECT
    s.customer_id,
    s.start_date,
    p.plan_id,
    p.plan_name,
    p.price,

    LEAD(s.start_date) OVER (
      PARTITION BY s.customer_id
      ORDER BY s.start_date
    ) AS next_start_date

  FROM foodie_fi.subscriptions s

  JOIN foodie_fi.plans p
    ON s.plan_id = p.plan_id
),

billing_anchor AS (

  SELECT

    customer_id,

    MIN(start_date) AS billing_anchor

  FROM plan_history

  WHERE plan_id IN (1, 2, 3)

  GROUP BY customer_id
),

billing_cycles AS (

  SELECT

    b.customer_id,

    generate_series(

      b.billing_anchor,

      '2020-12-31',

      INTERVAL '1 month'

    )::date AS billing_date

  FROM billing_anchor b
),

monthly_base_charge AS (

  SELECT

    bc.customer_id,

    bc.billing_date,

    ph.price

  FROM billing_cycles bc

  JOIN plan_history ph

    ON ph.customer_id = bc.customer_id

   AND bc.billing_date >= ph.start_date

   AND (bc.billing_date < ph.next_start_date OR ph.next_start_date IS NULL)

  WHERE ph.plan_id IN (1, 2)
),

upgrade_difference AS (

  SELECT

    ph.customer_id,

    ph.next_start_date::date AS payment_date,

    p2.price - ph.price AS amount

  FROM plan_history ph

  JOIN foodie_fi.plans p2

    ON p2.plan_id = 2

  WHERE ph.plan_id = 1

    AND ph.next_start_date IS NOT NULL
)

SELECT customer_id, billing_date AS payment_date, price AS amount

FROM monthly_base_charge

UNION ALL

SELECT customer_id, payment_date, amount

FROM upgrade_difference

ORDER BY customer_id, payment_date;
```

*Output: To be added*

