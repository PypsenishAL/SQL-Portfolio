# Danny's Diner Project:

## Problem Statement

Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers. 

Full description: [Case Study #1 - Danny's Diner](https://8weeksqlchallenge.com/case-study-1/)

## Case Study Questions

Each of the following case study questions can be answered using a single SQL statement. I'll mostly use two queries for convenience purposes.

#### 1. What is the total amount each customer spent at the restaurant?

````sql
select m.customer_id, sum(price) as total_spent
from members m left outer join sales s on m.customer_id = s.customer_id
left outer join menu on menu.product_id  = s.product_id
group by m.customer_id
  ````


|customer_id|total_amount|
|-----------|------------|
|A          |          76|
|B          |          74|

---
#### 2. How many days has each customer visited the restaurant?

````sql
SET
  search_path = dannys_diner;
SELECT
  customer_id,
  COUNT(DISTINCT order_date) AS days_of_visiting
FROM
  sales
GROUP BY
  1
ORDER BY
  1
  ````
  
| customer_id | days_of_visiting |
| ----------- | ---------------- |
| A           | 4                |
| B           | 6                |

---

#### 3. What was the first item from the menu purchased by each customer?

In order to solve this problem I've used first_value window-function.

````sql
select distinct members.customer_id, 
first_value(product_name) over (partition by sales.customer_id order by order_date) as first_product
from members left outer join sales on members.customer_id = sales.customer_id
left outer join menu on menu.product_id = sales.product_id
order by members.customer_id
````  

| customer_id | product_name | order_date |
| ----------- | ------------ | ---------- |
| A           | sushi        | 2021-01-01 |
| B           | curry        | 2021-01-01 |

  
***The first purchase for customer A was :sushi:***

***The first purchase for customer B was :curry:***

#### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

````sql
SET
  search_path = dannys_diner;

select 
	product_name, 
	count(*) as total_purchased
from menu inner join sales 
	on menu.product_id  = sales.product_id 
	
group by product_name
order by count(*) desc
limit 1
  ````

 | product_name | total_purchase_quantity |
 | ------------ | ----------------------- |
 | ramen        | 8                       |


---
 
***The most purchased item on the menu was :ramen:, it was purchased 8 times in total.***

#### 5. Which item was the most popular for each customer?

We can select the most popular products for each customer using `rank` window function:

````sql
SET
  search_path = dannys_diner;
with cte as (
	select
  members.customer_id,
  menu.product_name, count(product_name) as total_purchased
	from
	members left outer join sales
	on members.customer_id = sales.customer_id 
	left outer join menu on sales.product_id = menu.product_id 
	group by
  members.customer_id, menu.product_name 
),  ranker_cte as (select 
                   customer_id,
                   product_name,
                   total_purchased, 
			dense_rank() over (
                      partition by
                      customer_id
                      order by
                      total_purchased desc) as ranker
		from cte
)
select customer_id, product_name, total_purchased
from ranker_cte
where ranker = 1
 ```` 
 
| customer_id | product_name | total_purchase_quantity |
| ----------- | ------------ | ----------------------- |
| A           | ramen        | 3                       |
| B           | ramen        | 2                       |
| B           | curry        | 2                       |
| B           | sushi        | 2                       |

---
 
***The most popular item for customer A was :ramen:, they purchased it 3 times.***

***The most popular item for customer B was :curry:, :ramen: and :sushi:, they purchased each dish 2 times.***


#### 6. Which item was purchased first by the customer after they became a member?

Problem was solved using correlated subquery for each customer_id

````sql
SET
  search_path = dannys_diner;
select
	customer_id,
	(select 
		first_value(product_name) over ()
		from sales inner join menu on sales.product_id = menu.product_id
	where 
		(sales.order_date >= members.join_date) 
		and (sales.customer_id = members.customer_id)
	limit 1) as first_purchased_product
from members
  ````

| customer_id | join_date  | order_date | product_name |
| ----------- | ---------- | ---------- | ------------ |
| A           | 2021-01-07 | 2021-01-07 | curry        |
| B           | 2021-01-09 | 2021-01-11 | sushi        |

---

#### 7. Which item was purchased just before the customer became a member?

````sql
SET
  search_path = dannys_diner;
with cte as (
	select distinct members.customer_id, 
	join_date, 
	order_date,
	product_name,
	dense_rank() over (
		partition by members.customer_id order by order_date desc
	) as ranker
from 
	members left outer join sales on members.customer_id = sales.customer_id 
	left outer join menu on menu.product_id = sales.product_id
where
	join_date > order_date
order by customer_id
)
select customer_id, 
	   join_date, 
	   order_date,
	   product_name
from cte
where 
ranker = 1
  ````

| customer_id | join_date  | order_date | product_name |
| ----------- | ---------- | ---------- | ------------ |
| A           | 2021-01-07 | 2021-01-01 | sushi        |
| A           | 2021-01-07 | 2021-01-01 | curry        |
| B           | 2021-01-09 | 2021-01-04 | sushi        |

---

***Customer A purchased :curry: and :sushi: on 2021-01-01***

***Customer B purchased :sushi: on 2021-01-04***

#### 8. What is the total items and amount spent for each member before they became a member?

Let's consider that if the purchase date matches the membership date, then the purchase made on this date, was the first customer's purchase as a member. 

````sql
SET
  search_path = dannys_diner;
select
	members.customer_id, count(product_name) as total_items, sum(price) as total_spent
from 
	members left outer join sales on members.customer_id = sales.customer_id
	left outer join menu on menu.product_id = sales.product_id
where
	order_date < join_date
group by 
	members.customer_id
order by 
	members.customer_id
  ````

| customer_id | total_number_of_items | total_purchase_amount |
| ----------- | --------------------- | --------------------- |
| A           | 2                     | 25                    |
| B           | 3                     | 40                    |

---

#### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

````sql
SET
  search_path = dannys_diner;
select 
	members.customer_id, 
	sum(case 
			when product_name = 'sushi' then price * 10 * 2
			else price * 10
		end) as total_points
	from members left outer join sales on members.customer_id = sales.customer_id 
	left outer join menu on menu.product_id = sales.product_id 
group by 
	members.customer_id 
  ````

| customer_id | points |
| ----------- | ------ |
| A           | 860    |
| B           | 940    |

---

#### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

First we need to count points as usual: 10 points for each dollar spent on :curry: and :ramen: and 20 points for each dollar spent on :sushi:. 
We add this calculation to the `CTE` using `WITH` statement.

````sql
SET
  search_path = dannys_diner;
with cte as (
select 
	members.customer_id,
	order_date,
    join_date,
    product_name,
	sum(case 
			when product_name = 'sushi' then price * 10 * 2
			else price * 10
		end) as total_points
	from members left outer join sales on members.customer_id = sales.customer_id 
	left outer join menu on menu.product_id = sales.product_id 
group by 
	members.customer_id,
	order_date,
    join_date,
    product_name
)
select 
	customer_id,
	sum(case
		when order_date >= join_date
		and order_date < join_date + interval '1 week'
		and product_name != 'sushi' then total_points * 2
		else total_points
	end
	) as adjusted_total_points
from cte
where
	extract('month' from order_date) = 1
group by customer_id
order by customer_id
  ````

| customer_id | new_points |
| ----------- | ---------- |
| A           | 1370       |
| B           | 820        |

---

***Customer A at the end of January would have 1370 points***

***Customer B at the end of January would have 820 points*** and 0 benefits from their first week membership
