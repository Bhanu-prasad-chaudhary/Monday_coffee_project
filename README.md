# Monday Coffee Expansion SQL Project

![Company Logo](https://github.com/Bhanu-prasad-chaudhary/Monday_coffee_project/blob/main/1.png)

## Objective
The goal of this project is to analyze the sales data of Monday Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

## Key Questions
1. **Coffee Consumers Count**  
   How many people in each city are estimated to consume coffee, given that 25% of the population does?
```sql
SELECT
		city_name,
		ROUND((0.25*population)/1000000,2) AS coffee_consumers_in_millions,
		city_rank
FROM city
ORDER BY 2 DESC
```
2. **Total Revenue from Coffee Sales**  
   What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?
```sql
SELECT 
   c.city_name,
   SUM(s.total) as total_revenue
FROM sales as s
JOIN customers as cx
ON s.customer_id = cx.customer_id
JOIN city as c
ON c.city_id = cx.city_id
WHERE 
    EXTRACT(YEAR FROM s.sale_date) = 2023
	AND
	EXTRACT( quarter FROM s.sale_date) = 4
GROUP BY 1	
ORDER BY 2 DESC
```
3. **Sales Count for Each Product**  
   How many units of each coffee product have been sold?
```sql
    SELECT 
    p.product_name,
	COUNT(s.sale_id) as sales_count
FROM products as p
 JOIN sales as s
ON
   p.product_id = s.product_id
GROUP BY 1   
ORDER BY 2 DESC
```

4. **Average Sales Amount per City**  
   What is the average sales amount per customer in each city?
```sql
SELECT 
     c.city_name,
	 SUM(s.total) as total_revune,
	 COUNT(DISTINCT s.customer_id),
	 
	ROUND(
			SUM(s.total)::numeric/
				COUNT(DISTINCT s.customer_id)::numeric
			,2) as avg_sale_pr_cx
FROM sales as s 
JOIN customers as cx
ON s.customer_id = cx.customer_id
JOIN city as c
ON c.city_id = cx.city_id
GROUP BY 1
ORDER BY 2 DESC
```
5. **City Population and Coffee Consumers**  
   Provide a list of cities along with their populations and estimated coffee consumers.
```sql
SELECT cx.city_name,
      ROUND((0.25*population)/1000000,2) AS coffee_consumers_in_millions,
	  COUNT(DISTINCT s.customer_id) as unique_cx
FROM customers as c

JOIN sales as s
ON c.customer_id = s.customer_id

JOIN products as p
ON s.product_id = p.product_id

JOIN city as cx
ON c.city_id = cx.city_id
GROUP BY 1,2
ORDER BY 3 DESC
```
6. **Top Selling Products by City**  
   What are the top 3 selling products in each city based on sales volume?
```sql
WITH ranking_table
AS
(SELECT c.city_name as citys,
      p.product_name as products,
	  COUNT(sale_id) as sale_volume,
	  DENSE_RANK() OVER( PARTITION BY  c.city_name  ORDER BY COUNT(sale_id) DESC) AS rank
FROM sales as s
JOIN customers as cx
ON s.customer_id = cx.customer_id
JOIN city as c
ON c.city_id = cx.city_id
JOIN products as p
ON p.product_id = s.product_id
GROUP BY 1,2)
SELECT 
      citys,
	  products,
      sale_volume,
	  rank
FROM  ranking_table
WHERE rank <=3
```
7. **Customer Segmentation by City**  
   How many unique customers are there in each city who have purchased coffee products?
```sql
SELECT 

    cx.city_name  ,
	  COUNT(DISTINCT s.customer_id) as coffee_consumers
    
FROM customers as c

JOIN sales as s
ON c.customer_id = s.customer_id

JOIN city as cx
ON c.city_id = cx.city_id

WHERE 
s.product_id BETWWEN 1 AND 14 --- WE KNOW THE 1 TO 14 ARE ONLY COFFEE
GROUP BY 1
ORDER BY 2 DESC
```
8. **Average Sale vs Rent**  
   Find each city and their average sale per customer and avg rent per customer
```sql
WITH table_sale
AS
(SELECT
     c.city_name,
	 SUM(s.total) as total_revenue,
	 COUNT( DISTINCT cx.customer_id) as total_customer,
	 ROUND((SUM(s.total)::numeric/COUNT( DISTINCT cx.customer_id)),2) AS avg_sale_pr_cx

	FROM sales as s
	JOIN customers as cx
	ON s.customer_id = cx.customer_id
	JOIN city as c
	ON c.city_id = cx.city_id
    GROUP BY 1
	ORDER BY 2 DESC
),
table_rent
AS
( SELECT city_name,
        estimated_rent
FROM city
)
SELECT
    ts.city_name,
	tr.estimated_rent,
	ts.total_customer,
	ts.avg_sale_pr_cx,
	ROUND(tr.estimated_rent::numeric/ts.total_customer::numeric,2) as avg_rent_pr_cx
	
FROM table_sale AS ts
JOIN table_rent AS tr
ON ts.city_name = tr.city_name
```
9. **Monthly Sales Growth**  
   Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly).
```sql
WITH monthly_sales  
	As
	(
	SELECT 
		ci.city_name,
		EXTRACT(MONTH FROM sale_date) as month,
		EXTRACT(YEAR FROM sale_date) as YEAR,
		SUM(s.total) as total_sale
	FROM sales as s
	JOIN customers as c
	ON c.customer_id = s.customer_id
	JOIN city as ci
	ON ci.city_id = c.city_id
	GROUP BY 1, 2, 3
	ORDER BY 1, 3, 2
	),
	growth_ratio
	As(
        SELECT
		      city_name,
			  month,
			  year,
			  total_sale as cur_monthly_sales,
			  LAG(total_sale,1) OVER(PARTITION BY city_name ORDER BY year , month) as last_monthly_sales 
		FROM monthly_sales
	)
	SELECT
	       city_name,
			  month,
			  year,
	         cur_monthly_sales,
			 last_monthly_sales ,
			 ROUND(
		(cur_monthly_sales-last_monthly_sales)::numeric/last_monthly_sales::numeric * 100
		, 2
		) as growth_ratio

	FROM growth_ratio
	WHERE last_monthly_sales IS NOT NULL
```
10. **Market Potential Analysis**  
    Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated  coffee consumer
  ```sql

    WITH city_table
AS
(
	SELECT 
		ci.city_name,
		SUM(s.total) as total_revenue,
		COUNT(DISTINCT s.customer_id) as total_cx,
		ROUND(
				SUM(s.total)::numeric/
					COUNT(DISTINCT s.customer_id)::numeric
				,2) as avg_sale_pr_cx
		
	FROM sales as s
	JOIN customers as c
	ON s.customer_id = c.customer_id
	JOIN city as ci
	ON ci.city_id = c.city_id
	GROUP BY 1
	ORDER BY 2 DESC
),
city_rent
AS
(
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
	ct.total_cx,
	estimated_coffee_consumer_in_millions,
	ct.avg_sale_pr_cx,
	ROUND(
		cr.estimated_rent::numeric/
									ct.total_cx::numeric
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
