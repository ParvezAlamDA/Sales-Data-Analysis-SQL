select * from sales

-- Sales over time

Select	
	Year(order_date) as Order_Year,
	Month(order_date) as Order_Month,
	sum(sales_amount),
	count(distinct customer_key) as total_customers,
	sum(quantity) as Total_Quantity
from sales
where order_date is not null
group by Year(order_date), Month(order_date)
order by Year(order_date), Month(order_date)






--Calculate the total sales per month and he running total of sales over time

select
	Order_Date,
	Total_Sales,
	sum(Total_Sales) over(partition by order_date order by Order_Date) as Sales_Running_Toal,
	Avg(Avg_Price) over(partition by order_date order by Order_Date) as Sales_Moving_Avg
From
(
select 
	datetrunc(month, order_date) as Order_Date,
	sum(sales_amount) as Total_Sales,
	avg(price) as Avg_Price
from sales
where order_date is not null
Group by datetrunc(month, order_date)
)subquery


--PERFORMANCE ANALYSIS (YoY)
--Comparing the Yearly sales to average sales and previous year sales.
with Yearly_Sales as
(
select 
	year(s.order_date) as Order_Year,
	p.product_name,
	sum(s.sales_amount) as Total_Sales
from sales s
left join products p 
on s.product_key = p.product_key
where s.order_date is not null
group by 
	year(s.order_date),
	p.product_name
)
Select 
Order_Year,
product_name,
Total_Sales,
AVG(Total_Sales) over (partition by product_name) as Avg_Sales,
Total_Sales - AVG(Total_Sales) over (partition by product_name) as Diff_Avg,
Case when Total_Sales - AVG(Total_Sales) over (partition by product_name)>0 then 'Above Avg'
	when Total_Sales - AVG(Total_Sales) over (partition by product_name)<0 then 'Below Avg'
	else 'Avg'
End as Avg_Change,
LAG(Total_Sales) over (partition by product_name order by Order_Year) as Prev_Year_Sales,
Total_Sales - LAG(Total_Sales) over (partition by product_name order by Order_Year) as Prv_Year,
Case when Total_Sales - LAG(Total_Sales) over (partition by product_name order by Order_Year)>0 then 'Increase'
	when Total_Sales - LAG(Total_Sales) over (partition by product_name order by Order_Year)<0 then 'Decrease'
	else 'No Change'
End as Net_Change
from Yearly_Sales
order by product_name

--Proportional Analysis
--Which categories contribute the most to overall sales

with category_sales as 
(
select 
	category,
	sum(sales_amount) as Total_Sales
from sales s
left join products p on
p.product_key = s.product_key
group by category
)
select 
	category,
	Total_Sales,
	sum(Total_Sales) over() overall_Sales,
	round(cast(Total_Sales as float) /sum(Total_Sales) over(),4)*100 as Percentage
from category_sales


--Data Segmentation
--Segmen the products into cost ranges and count number of products in each category.
with product_segment as (
select 
	product_key,
	product_name,
	cost,
Case when cost <100 then 'Below 100'
	when cost between 100 and 500 then '100-500'
	when cost between 500 and 1000 then '500-1000'
	else 'Above 1000'
end as cost_range
from products)
Select 
	cost_range,
	count(product_key) as total_products
from product_segment
group by cost_range
order by total_products desc

--Group customers based on their spending behaviour
With Customer_spending as (
select
	c.customer_key,
	sum(s.sales_amount) as Total_Spending,
	min(s.order_date) as first_order,
	max(s.order_date) as last_order,
	datediff(month,min(order_date), max(order_date)) as lifespan
from sales s
left join customers c
on c.customer_key = s.customer_key
group by c.customer_key
)
select 
	customer_segment,
	count(customer_key) as total_Customer
from
(
	Select
		customer_key,
	case when lifespan>= 12 and total_spending >5000 then 'VIP'
		 when lifespan>= 12 and total_spending <=5000 then 'Regular'
		 else 'new'
	end customer_segment
	from Customer_spending
) t
group by customer_segment
order by total_Customer desc

--Customer Report
with base_query as (
select 
	s.order_number,
	s.product_key,
	s.order_date,
	s.sales_amount,
	s.quantity,
	c.customer_key,
	c.customer_number,
	concat(c.first_name, ' ',c.last_name) as customer_name,
	datediff(year,c.birthdate,getdate()) as age
from sales s
left join customers c
on c.customer_key = s.customer_key
where order_date is not null )
Select 
	customer_key,
	customer_number,
	customer_name,
	age,
	count(distinct order_number) as total_order,
	sum(sales_amount) as total_sales,
	sum(quantity) as total_quantity,
	count(distinct product_key) as total_products,
	max(order_date) as last_order_date,
	datediff(month,min(order_date), max(order_date)) as lifespan
from base_query
group by 
customer_key,customer_number,customer_name,age
