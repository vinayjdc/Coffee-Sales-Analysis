# Monday Coffee Expansion SQL Project

![Company Logo](https://github.com/najirh/Monday-Coffee-Expansion-Project-P8/blob/main/1.png)

## Objective
The goal of this project is to analyze the sales data of Monday Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

Schema- Coffee Sales Analysis

<img width="768" alt="Schemas Diagram" src="https://github.com/user-attachments/assets/9f3a9fa9-9d72-4ed2-bcf2-c4350464aa82">

## Key Questions
1. **Coffee Consumers Count**  
   How many people in each city are estimated to consume coffee, given that 25% of the population does?

```sql
select city_name, population*0.25 as estimated_people, city_rank
from city
order by 2 desc;
```

2. **Total Revenue from Coffee Sales**  
   What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?

```sql
select sum(total) as total_revenue from sales where sale_date between '2023-10-01' and '2023-12-31';

    OR

select sum(total) as total_revenue 
from sales 
where extract(year from sale_date) = 2023 
and extract(quarter from sale_date)= 4; 
```

3. **Sales Count for Each Product**  
   How many units of each coffee product have been sold?

```sql
   select p.product_name, count(s.sale_id) as coffee_sold from sales s
   join products p
   on p.product_id = s.product_id
   group by p.product_name
   order by 2 desc;
```

4. **Average Sales Amount per City**  
   What is the average sales amount per customer in each city?
   
```sql
   select ci.city_name,sum(s.total) as total_revenue, count(distinct cu.customer_id) as total_customer, 
   round(sum(s.total)::numeric/count(distinct cu.customer_id)::numeric) as avg_sale_per_customer
   from city ci
   join customers cu
   on ci.city_id = cu.city_id
   join sales s
   on s.customer_id = cu.customer_id
   group by 1
   order by 2 desc
```

5. **City Population and Coffee Consumers**  
   Provide a list of cities along with their populations and estimated coffee consumers.

```sql
  SELECT 
    ci.city_name, 
    round(ci.population * 0.25/1000000,2) AS coffee_consumer_in_millions, 
    COUNT(distinct cu.customer_id) AS unique_customer
FROM 
    city ci 
LEFT JOIN 
    customers cu ON ci.city_id = cu.city_id 
GROUP BY 
    ci.city_name, ci.population
ORDER BY 
    unique_customer desc;

         OR

WITH city_table as 
(
	SELECT 
		city_name,
		ROUND((population * 0.25)/1000000, 2) as coffee_consumers
	FROM city
),
customers_table
AS
(
	SELECT 
		ci.city_name,
		COUNT(DISTINCT c.customer_id) as unique_cx
	FROM sales as s
	JOIN customers as c
	ON c.customer_id = s.customer_id
	JOIN city as ci
	ON ci.city_id = c.city_id
	GROUP BY 1
)
SELECT 
	customers_table.city_name,
	city_table.coffee_consumers as coffee_consumer_in_millions,
	customers_table.unique_cx
FROM city_table
JOIN 
customers_table
ON city_table.city_name = customers_table.city_name
```

6. **Top Selling Products by City**  
   What are the top 3 selling products in each city based on sales volume?

```sql
     with product_sales_per_city as(
   select  ci.city_name, p.product_name,sum(s.total) as total_sales,
   rank() over(partition by  ci.city_name order by sum(s.total) desc) as rank
   from city ci
   join customers cu
   on cu.city_id = ci.city_id
   join sales s
   on cu.customer_id = s.customer_id
   join products p
   on p.product_id = s.product_id
   group by ci.city_name, p.product_name)
   
   select city_name, product_name, total_sales, rank from product_sales_per_city
   where rank <=3;

      OR

   SELECT * 
   FROM -- table
   (
   	SELECT 
   		ci.city_name,
   		p.product_name,
   		COUNT(s.sale_id) as total_orders,
   		DENSE_RANK() OVER(PARTITION BY ci.city_name ORDER BY COUNT(s.sale_id) DESC) as rank
   	FROM sales as s
   	JOIN products as p
   	ON s.product_id = p.product_id
   	JOIN customers as c
   	ON c.customer_id = s.customer_id
   	JOIN city as ci
   	ON ci.city_id = c.city_id
   	GROUP BY 1, 2
   	-- ORDER BY 1, 3 DESC
   ) as t1
   WHERE rank <= 3
```

7. **Customer Segmentation by City**  
   How many unique customers are there in each city who have purchased coffee products?

```sql
      select ci.city_name,count(distinct cu.customer_id) as no_of_customer
   FROM sales as s
   	JOIN products as p
   	ON s.product_id = p.product_id
   	JOIN customers as cu
   	ON cu.customer_id = s.customer_id
   	JOIN city as ci
   	ON ci.city_id = cu.city_id
   	group by 1
   	order by no_of_customer desc
```

8. **Average Sale vs Rent**  
   Find each city and their average sale per customer and avg rent per customer

```sql
   WITH city_sales AS (
    SELECT 
        ci.city_name,
        SUM(s.total) AS total_revenue,
        COUNT(DISTINCT cu.customer_id) AS total_customers,
        ROUND(SUM(s.total)::numeric / COUNT(DISTINCT cu.customer_id)::numeric, 2) AS avg_sale_per_customer
    FROM city ci
    JOIN customers cu ON ci.city_id = cu.city_id
    JOIN sales s ON s.customer_id = cu.customer_id
    GROUP BY ci.city_name
),
city_rent AS (
    SELECT 
        city_name, 
        estimated_rent
    FROM city
)
SELECT 
    cs.city_name,
    cs.total_revenue,
    cs.total_customers,
    cs.avg_sale_per_customer,
    cr.estimated_rent,
    ROUND(cr.estimated_rent::numeric / cs.total_customers::numeric, 2) AS avg_rent_per_customer
FROM city_sales AS cs
JOIN city_rent AS cr ON cs.city_name = cr.city_name
ORDER BY cs.total_revenue DESC;
```

9. **Monthly Sales Growth**  
   Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly).

```sql
   with monthly_sales as
(select ci.city_name,
	extract(month from sale_date) as month,
	extract(year from sale_date) as year,
	sum(s.total) as total_sale
	from city ci
    JOIN customers cu ON ci.city_id = cu.city_id
    JOIN sales s ON s.customer_id = cu.customer_id
	group by ci.city_name, month, year
	order by 1, year,month
),
growth_rate as (
select  city_name,month,year, total_sale as cr_month_sale,
lag(total_sale,1) over(partition by city_name order by year,month) as last_month_sale
from monthly_sales
)

select city_name,month,year,cr_month_sale,last_month_sale,
round(
(cr_month_sale-last_month_sale)::numeric/last_month_sale::numeric*100,2
) as growth_rate
from growth_rate
where last_month_sale is not null
```

10. **Market Potential Analysis**  
    Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated  coffee consumer

```sql
   WITH city_table AS (
    SELECT 
        ci.city_name,
        SUM(s.total) AS total_revenue,
        COUNT(DISTINCT cu.customer_id) AS total_customers,
        ROUND(SUM(s.total)::numeric / COUNT(DISTINCT cu.customer_id)::numeric, 2) AS avg_sale_per_customer
    FROM city ci
    JOIN customers cu ON ci.city_id = cu.city_id
    JOIN sales s ON s.customer_id = cu.customer_id
    GROUP BY ci.city_name
),
city_rent AS (
    SELECT 
        city_name, 
        estimated_rent,
		ROUND((population * 0.25)/1000000, 3) as estimated_coffee_consumer_in_millions
    FROM city
)
SELECT 
	cr.city_name,
	total_revenue,
	cr.estimated_rent as total_rent,
	ct.total_customers,
	estimated_coffee_consumer_in_millions,
	ct.avg_sale_per_customer,
	ROUND(
		cr.estimated_rent::numeric/
									ct.total_customers::numeric
		, 2) as avg_rent_per_cx
FROM city_rent as cr
JOIN city_table as ct
ON cr.city_name = ct.city_name
ORDER BY 2 DESC
```
    

## Recommendations
After analyzing the data, the recommended top three cities for new store openings are:

**City 1: Pune**  
1. Average rent per customer is very low.  
2. Highest total revenue.  
3. Average sales per customer is also high.

**City 2: Delhi**  
1. Highest estimated coffee consumers at 7.7 million.  
2. Highest total number of customers, which is 68.  
3. Average rent per customer is 330 (still under 500).

**City 3: Jaipur**  
1. Highest number of customers, which is 69.  
2. Average rent per customer is very low at 156.  
3. Average sales per customer is better at 11.6k.

---
