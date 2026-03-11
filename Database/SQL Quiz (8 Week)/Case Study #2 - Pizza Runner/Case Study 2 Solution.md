# 8 Week SQL Challenge – Pizza Runner 🍕



# Data Cleaning

The following SQL queries were used to clean the dataset.

## Cleaning `customer_orders`

```sql
UPDATE pizza_runner.customer_orders
SET
exclusions = NULLIF(exclusions, ''),
extras = NULLIF(extras,'');

update pizza_runner.customer_orders
SET
exclusions=NULL
Where exclusions='null';

update pizza_runner.customer_orders
SET extras = NULL
where extras = 'null';
```

## Cleaning `runner_orders`

```sql
ALTER table pizza_runner.runner_orders
ALTER COLUMN pickup_time TYPE TIMESTAMP
USING NULLIF(pickup_time, 'null')::timestamp;

ALTER table pizza_runner.runner_orders
ALTER COLUMN distance TYPE Numeric
using NULLIF(
regexp_replace(distance, '[^0-9\\.]','','g'),''
)::numeric;

ALTER table pizza_runner.runner_orders
ALTER COLUMN duration TYPE Numeric
using NULLIF(
regexp_replace(duration, '[^0-9\\.]','','g'),''
)::integer;

UPDATE pizza_runner.runner_orders
SET cancellation = NULL
Where cancellation IN ('','null');
```

---

# A. Pizza Metrics

## 1. How many pizzas were ordered?

```sql
SELECT count(*) from pizza_runner.customer_orders;
```

*Output: To be added*

## 2. How many unique customer orders were made?

```sql
SELECT count(Distinct(order_id)) from pizza_runner.customer_orders;
```

*Output: To be added*

## 3. How many successful orders were delivered by each runner?

```sql
Select runner_id, count(order_id)
from pizza_runner.runner_orders
where cancellation IS NULL
Group by runner_id;
```

*Output: To be added*

## 4. How many of each type of pizza was delivered?

```sql
Select p.pizza_name, count(c.pizza_id)
from pizza_runner.customer_orders c
JOIN pizza_runner.runner_orders r
on c.order_id=r.order_id
Join pizza_runner.pizza_names p on c.pizza_id = p.pizza_id
where r.cancellation is null
Group by p.pizza_name;
```

*Output: To be added*

## 5. How many Vegetarian and Meatlovers were ordered by each customer?

```sql
Select c.customer_id, p.pizza_name, count(c.pizza_id)
from pizza_runner.customer_orders c
join pizza_runner.pizza_names p on c.pizza_id = p.pizza_id

group by c.customer_id, p.pizza_name
order by c.customer_id;
```

*Output: To be added*

## 6. What was the maximum number of pizzas delivered in a single order?

```sql
select c.order_id, count(c.pizza_id) as pizzas_in_order
from pizza_runner.customer_orders c
join pizza_runner.runner_orders r on c.order_id = r.order_id
where r.cancellation IS NULL
group by c.order_id
order by pizzas_in_order DESC LIMIT 1;
```

*Output: To be added*

## 7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

```sql
select c.customer_id,
count(*) filter(where c.exclusions is not null or c.extras is not null) as atleast_1_change,
count(*) filter(where c.exclusions is null and c.extras is null) as no_change
from pizza_runner.customer_orders c

group by c.customer_id;
```

*Output: To be added*

## 8. How many pizzas were delivered that had both exclusions and extras?

```sql
select count(c.pizza_id)
from pizza_runner.customer_orders c
join pizza_runner.runner_orders r on c.order_id = r.order_id
where (exclusions is not null and extras is not null) and r.cancellation is null;
```

*Output: To be added*

## 9. What was the total volume of pizzas ordered for each hour of the day?

```sql
SELECT EXTRACT(HOUR FROM c.order_time) as hours,
count(c.pizza_id ) as pizzas_ordered
from pizza_runner.customer_orders c
Group by hours
order by hours;
```

*Output: To be added*

## 10. What was the volume of orders for each day of the week?

```sql
SELECT to_char(c.order_time AT TIME ZONE 'Asia/Kolkata', 'FMDay') AS weekday,
EXTRACT(DOW FROM c.order_time AT TIME ZONE 'Asia/Kolkata') AS dow,
count(c.pizza_id ) as pizzas_ordered
from pizza_runner.customer_orders c
GROUP BY weekday, dow
ORDER BY pizzas_ordered;
```

*Output: To be added*

---

# B. Runner and Customer Experience

## 1. How many runners signed up for each 1 week period?

```sql
SELECT (registration_date - DATE '2021-01-01') / 7 + 1 AS week_no, count(*) signed_up
FROM pizza_runner.runners r
GROUP BY week_no
ORDER BY week_no;
```

*Output: To be added*

## 2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?

```sql
SELECT
r.runner_id,
ROUND(AVG(EXTRACT(EPOCH FROM (r.pickup_time - c.order_time)) / 60), 2) AS avg_pickup_minutes
FROM pizza_runner.customer_orders c
JOIN pizza_runner.runner_orders r ON c.order_id = r.order_id
WHERE r.pickup_time IS NOT NULL
GROUP BY r.runner_id;
```

*Output: To be added*

## 3. Is there any relationship between the number of pizzas and how long the order takes to prepare?

```sql
WITH cte as (
Select c.order_id, count(c.pizza_id) as number_of_pizza, c.order_time, r.pickup_time
from pizza_runner.customer_orders c
join pizza_runner.runner_orders r on c.order_id = r.order_id
group by c.order_id,c.order_time,r.pickup_time
)

Select order_id, number_of_pizza,
ROUND(AVG(EXTRACT(EPOCH FROM (pickup_time - order_time)) / 60), 2) AS preptime
from cte
group by order_id,number_of_pizza
order by number_of_pizza DESC;
```

*Output: To be added*

## 4. What was the average distance travelled for each customer?

```sql
SELECT c.customer_id,count(c.order_id) as no_of_orders, AVG(r.distance) as avg_distance
from pizza_runner.customer_orders c
join pizza_runner.runner_orders r on c.order_id = r.order_id
group by c.customer_id;
```

*Output: To be added*

## 5. What was the difference between the longest and shortest delivery times for all orders?

```sql
select (MAX(duration)-MIN(duration)) as the_diff from pizza_runner.runner_orders;
```

*Output: To be added*

## 6. What was the average speed for each runner for each delivery and do you notice any trend?

```sql
with cte as (
SELECT r.runner_id, count(c.pizza_id) as number_of_pizza, r.duration, r.distance,(r.distance/(r.duration/60)) as speed
from pizza_runner.customer_orders c
join pizza_runner.runner_orders r on c.order_id=r.order_id
group by c.order_id, r.duration, r.distance, r.runner_id
)

SELECT runner_id, number_of_pizza, Speed
from cte
Group by number_of_pizza, speed, runner_id
Having Speed NOTNUll
order by runner_id;
```

*Output: To be added*

## 7. What is the successful delivery percentage for each runner?

```sql
with cte as
(
SELECT
runner_id,
COUNT(CASE WHEN cancellation IS NULL THEN order_id END) AS successful,
COUNT(CASE WHEN cancellation IS NOT NULL THEN order_id END) AS unsuccessful
FROM pizza_runner.runner_orders
GROUP BY runner_id
)

select runner_id, (successful * 100/ (successful + unsuccessful)) as sucess_perc from cte;
```

*Output: To be added*

---

# C. Ingredient Optimisation

## 1. What are the standard ingredients for each pizza?

```sql
with cte as (
SELECT pizza_id, unnest(string_to_array(toppings, ', ')) AS topping_id
FROM pizza_runner.pizza_recipes
)

select p.pizza_name, STRING_AGG(pt.topping_name,',')
from pizza_runner.pizza_names p
join cte on p.pizza_id = cte.pizza_id
join pizza_runner.pizza_toppings pt on pt.topping_id = CAST(cte.topping_id as int)
Group by p.pizza_name;
```

*Output: To be added*

## 2. What was the most commonly added extra?

```sql
WITH cte AS (
  SELECT unnest(string_to_array(c.extras, ', ')) AS extra
  FROM pizza_runner.customer_orders c
  WHERE c.extras IS NOT NULL
),
ctetwo AS (
  SELECT extra, COUNT(*) AS countt
  FROM cte
  GROUP BY extra
)

SELECT pt.topping_name, c.countt
FROM ctetwo c
JOIN pizza_runner.pizza_toppings pt
  ON pt.topping_id = CAST(c.extra AS INT)
ORDER BY c.countt DESC
LIMIT 1;
```

*Output: To be added*

## 3. What was the most common exclusion?

```sql
WITH cte AS (
  SELECT unnest(string_to_array(c.exclusions, ', ')) AS excl
  FROM pizza_runner.customer_orders c
  WHERE c.exclusions IS NOT NULL
),
ctetwo AS (
  SELECT excl, COUNT(*) AS countt
  FROM cte
  GROUP BY excl
)

SELECT pt.topping_name, c.countt
FROM ctetwo c
JOIN pizza_runner.pizza_toppings pt
  ON pt.topping_id = CAST(c.excl AS INT)
ORDER BY c.countt DESC
LIMIT 1;
```

*Output: To be added*

## 4. Generate formatted order item

```sql
WITH base_orders AS (
  SELECT co.order_id, pn.pizza_name, co.exclusions, co.extras
  FROM pizza_runner.customer_orders co
  JOIN pizza_runner.pizza_names pn
    ON pn.pizza_id = co.pizza_id
),

exclusions AS (
  SELECT bo.order_id,
  STRING_AGG(pt.topping_name, ', ' ORDER BY pt.topping_name) AS excluded_toppings
  FROM base_orders bo
  JOIN pizza_runner.pizza_toppings pt
    ON pt.topping_id = ANY (string_to_array(bo.exclusions, ', ')::INT[])
  WHERE bo.exclusions IS NOT NULL
  GROUP BY bo.order_id
),

extras AS (
  SELECT bo.order_id,
  STRING_AGG(pt.topping_name, ', ' ORDER BY pt.topping_name) AS extra_toppings
  FROM base_orders bo
  JOIN pizza_runner.pizza_toppings pt
    ON pt.topping_id = ANY (string_to_array(bo.extras, ', ')::INT[])
  WHERE bo.extras IS NOT NULL
  GROUP BY bo.order_id
)

SELECT bo.order_id,
bo.pizza_name
|| COALESCE(' - Exclude ' || ex.excluded_toppings, '')
|| COALESCE(' - Extra '   || et.extra_toppings, '') AS order_item

FROM base_orders bo
LEFT JOIN exclusions ex ON bo.order_id = ex.order_id
LEFT JOIN extras et ON bo.order_id = et.order_id
ORDER BY bo.order_id;
```

*Output: To be added*

## 5. Generate an alphabetically ordered comma separated ingredient list

```sql
WITH recipe_ingredients AS (
  SELECT co.order_id, pt.topping_name
  FROM pizza_runner.customer_orders co
  JOIN pizza_runner.pizza_recipes pr
    ON pr.pizza_id = co.pizza_id
  JOIN pizza_runner.pizza_toppings pt
    ON pt.topping_id = ANY (string_to_array(pr.toppings, ', ')::INT[])

  WHERE co.exclusions IS NULL
     OR pt.topping_id <> ALL (string_to_array(co.exclusions, ', ')::INT[])
),

extra_ingredients AS (
  SELECT co.order_id, pt.topping_name
  FROM pizza_runner.customer_orders co
  JOIN pizza_runner.pizza_toppings pt
    ON pt.topping_id = ANY (string_to_array(co.extras, ', ')::INT[])

  WHERE co.extras IS NOT NULL
),

all_ingredients AS (
  SELECT * FROM recipe_ingredients
  UNION ALL
  SELECT * FROM extra_ingredients
),

ingredient_counts AS (
  SELECT order_id, topping_name, COUNT(*) AS qty
  FROM all_ingredients
  GROUP BY order_id, topping_name
)

SELECT order_id,
STRING_AGG(
CASE
WHEN qty > 1 THEN qty || 'x ' || topping_name
ELSE topping_name
END,
', ' ORDER BY topping_name
) AS ingredient_list

FROM ingredient_counts
GROUP BY order_id
ORDER BY order_id;
```

*Output: To be added*

## 6. Total quantity of each ingredient used in all delivered pizzas

```sql
WITH delivered_orders AS (
  SELECT co.order_id, co.pizza_id, co.exclusions, co.extras
  FROM pizza_runner.customer_orders co
  JOIN pizza_runner.runner_orders ro
    ON ro.order_id = co.order_id

  WHERE ro.pickup_time IS NOT NULL
),

recipe_ingredients AS (
  SELECT da.order_id, pt.topping_name
  FROM delivered_orders da
  JOIN pizza_runner.pizza_recipes pr
    ON pr.pizza_id = da.pizza_id
  JOIN pizza_runner.pizza_toppings pt
    ON pt.topping_id = ANY (string_to_array(pr.toppings, ', ')::INT[])

  WHERE da.exclusions IS NULL
     OR pt.topping_id <> ALL (string_to_array(da.exclusions, ', ')::INT[])
),

extra_ingredients AS (
  SELECT da.order_id, pt.topping_name
  FROM delivered_orders da
  JOIN pizza_runner.pizza_toppings pt
    ON pt.topping_id = ANY (string_to_array(da.extras, ', ')::INT[])

  WHERE da.extras IS NOT NULL
),

all_ingredients AS (
  SELECT topping_name FROM recipe_ingredients
  UNION ALL
  SELECT topping_name FROM extra_ingredients
)

SELECT topping_name, COUNT(*) AS total_quantity
FROM all_ingredients
GROUP BY topping_name
ORDER BY total_quantity DESC;
```

*Output: To be added*

---

# D. Pricing and Ratings

## 1. Revenue without delivery fees

```sql
SELECT
  SUM(
    CASE
      WHEN pn.pizza_name = 'Meat Lovers' THEN 12
      WHEN pn.pizza_name = 'Vegetarian' THEN 10
      ELSE 0
    END
  ) AS total_revenue
FROM pizza_runner.customer_orders co
JOIN pizza_runner.runner_orders ro
  ON co.order_id = ro.order_id
JOIN pizza_runner.pizza_names pn
  ON pn.pizza_id = co.pizza_id
WHERE ro.pickup_time IS NOT NULL;
```

*Output: To be added*

## 2. Revenue including $1 per extra topping

```sql
SELECT
  SUM(
    CASE
      WHEN pn.pizza_name = 'Meat Lovers' THEN 12
      WHEN pn.pizza_name = 'Vegetarian' THEN 10
      ELSE 0
    END
    +
    COALESCE(
      array_length(string_to_array(co.extras, ', '), 1),
      0
    )
  ) AS total_revenue
FROM pizza_runner.customer_orders co
JOIN pizza_runner.runner_orders ro
  ON co.order_id = ro.order_id
JOIN pizza_runner.pizza_names pn
  ON pn.pizza_id = co.pizza_id
WHERE ro.pickup_time IS NOT NULL;
```

*Output: To be added*

## 3. Ratings table schema

```sql
CREATE TABLE pizza_runner.runner_ratings (
  rating_id SERIAL PRIMARY KEY,
  order_id INT NOT NULL,
  runner_id INT NOT NULL,
  rating INT CHECK (rating BETWEEN 1 AND 5)
);
```

## 4. Combined delivery analytics table

```sql
WITH clean_runner_orders AS (
  SELECT
    order_id,
    runner_id,
    NULLIF(pickup_time, 'null')::TIMESTAMP AS pickup_time,

    CAST(
      REGEXP_REPLACE(NULLIF(distance, 'null'), '[^0-9.]', '', 'g')
      AS FLOAT
    ) AS distance_km,

    CAST(
      REGEXP_REPLACE(NULLIF(duration, 'null'), '[^0-9]', '', 'g')
      AS INT
    ) AS duration_mins

  FROM pizza_runner.runner_orders
),

successful_deliveries AS (

  SELECT
    co.customer_id,
    co.order_id,
    cro.runner_id,
    rr.rating,
    co.order_time,
    cro.pickup_time,

    EXTRACT(EPOCH FROM (cro.pickup_time - co.order_time)) / 60
      AS time_to_pickup_mins,

    cro.duration_mins,

    cro.distance_km / cro.duration_mins
      AS avg_speed_km_per_min

  FROM pizza_runner.customer_orders co
  JOIN clean_runner_orders cro
    ON co.order_id = cro.order_id

  LEFT JOIN pizza_runner.runner_ratings rr
    ON rr.order_id = co.order_id

  WHERE cro.pickup_time IS NOT NULL
)

SELECT
  customer_id,
  order_id,
  runner_id,
  rating,
  order_time,
  pickup_time,
  time_to_pickup_mins,
  duration_mins AS delivery_duration,
  avg_speed_km_per_min,
  COUNT(*) AS total_pizzas

FROM successful_deliveries

GROUP BY
  customer_id,
  order_id,
  runner_id,
  rating,
  order_time,
  pickup_time,
  time_to_pickup_mins,
  duration_mins,
  avg_speed_km_per_min

ORDER BY order_id;
```

*Output: To be added*

## 5. Profit after paying runners

```sql
WITH delivered_orders AS (
  SELECT
    order_id,
    CAST(
      REGEXP_REPLACE(NULLIF(distance, 'null'), '[^0-9.]', '', 'g')
      AS FLOAT
    ) AS distance_km
  FROM pizza_runner.runner_orders

  WHERE pickup_time IS NOT NULL
    AND pickup_time <> 'null'
),

pizza_revenue AS (

  SELECT
    co.order_id,

    SUM(
      CASE
        WHEN pn.pizza_name = 'Meat Lovers' THEN 12
        WHEN pn.pizza_name = 'Vegetarian' THEN 10
        ELSE 0
      END
    ) AS revenue

  FROM pizza_runner.customer_orders co

  JOIN pizza_runner.pizza_names pn
    ON co.pizza_id = pn.pizza_id

  JOIN delivered_orders d
    ON co.order_id = d.order_id

  GROUP BY co.order_id
),

delivery_cost AS (

  SELECT
    order_id,
    distance_km * 0.30 AS delivery_fee

  FROM delivered_orders
)

SELECT
  SUM(pr.revenue)      AS total_revenue,
  SUM(dc.delivery_fee) AS total_delivery_cost,

  ROUND(
    (SUM(pr.revenue) - SUM(dc.delivery_fee))::NUMERIC,
    2
  ) AS money_left

FROM pizza_revenue pr

JOIN delivery_cost dc
  ON pr.order_id = dc.order_id;
```

*Output: To be added*

