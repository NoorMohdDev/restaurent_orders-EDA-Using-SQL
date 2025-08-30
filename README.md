# restaurent_orders-EDA-Using-SQL

select * from order_table

#-- 📊 Sales & Revenue Analysis

##-- What is the total revenue generated?

select sum(quantity * price) as Total_revenue from order_table

##-- What is the average order value (AOV) per order?

select orderId, avg(quantity*price) over(partition by orderId) as avgOrderValue from order_table

##-- Which food item generates the highest revenue?

select foodItem, sum(quantity * price) as highest_revenue from order_table
group by foodItem
order by highest_revenue desc
limit 1

##-- What is the most profitable category by revenue?

select category, sum(quantity * price) as highest_revenue from order_table
group by category
order by highest_revenue desc
limit 1

##-- How many orders are above ₹500 / ₹1000?

select count(orderId) as no_of_order from order_table
where price * quantity > 500 or price * quantity > 1000

##-- What is the distribution of order values (low, medium, high spenders)?
select price * quantity as order_value, 
case
	when price * quantity<100 then 'low'
	when price * quantity<250 then 'medium'
	else 'high'
end as spenders_distribution
from order_table

#-- 👥 Customer Behavior

##-- Who are the top 10 customers by total spend?

select round(price)*quantity as total_spend, customerName from order_table
order by total_spend desc
limit 10

##-- Which customers have the highest order frequency?

select customerName, count(orderId) as order_freq from order_table
group by customerName
having count(orderId) > 1
order by order_freq desc

##-- What is the average spend per customer?

select customerName, avg(price*quantity) as avg_spend from order_table
group by customerName

##-- Which customers only ordered once (one-time buyers) vs repeat customers?

select customerName, count(orderId) as order_freq,
case
	when count(orderId) > 1 then 'repeat'
	else 'one-time'
end as customer_type
from order_table
group by customerName
order by order_freq desc

##-- What are the preferred categories of each customer?

select customerName,count(category) as cat_count, string_agg(category,', ') as categories from order_table
group by customerName
order by cat_count desc

#-- 🍕 Product Insights

##-- What are the top 5 most ordered food items?

select foodItem, count(orderId) as most_ordered_item from order_table
group by foodItem
order by most_ordered_item desc
limit 5

##-- Which category has the highest average order value?

select category, avg(price*quantity) as avg_order_value from order_table
group by category
order by avg_order_value desc
limit 1

##-- Which food item has the highest average quantity per order?

select foodItem, avg(quantity) as avg_quantity from order_table
group by foodItem
order by avg_quantity desc
limit 1

#-- 💳 Payment Insights

##-- What is the distribution of payment methods (Cash, Card, UPI, Wallets)?

select paymentMethod, count(orderId) as total_orders from order_table
group by paymentMethod
order by total_orders

##-- Is there a difference in average spend by payment method?

select paymentMethod, avg(price*quantity) as avg_spend from order_table
group by paymentMethod
order by avg_spend

##-- Do customers who use online payments spend more than cash users?

select paymentMethod, sum(price*quantity) as avg_spend from order_table
group by paymentMethod
order by avg_spend

#-- ⏰ Time-based Analysis

##-- What are the peak order hours of the day?

with peak_hours as  (
	select extract(day from orderTime) as order_day, extract(hour from orderTime) as order_hours, count(*) as total_orders from order_table
	group by extract(hour from orderTime), extract(day from orderTime)
	order by order_day, total_orders desc
)

select * from peak_hours
where total_orders = (select max(total_orders) from peak_hours)

##-- Which days of the week have the highest number of orders?

with orders_by_day as (
    select 
        extract(week from orderTime) as order_week,
        extract(dow from orderTime) as order_day, -- dow gives day of week (0=Sunday, 6=Saturday)
        count(*) as total_orders
    from order_table
    group by extract(week from orderTime), extract(dow from orderTime)
),
ranked_orders as (
    select 
        order_week,
        order_day,
        total_orders,
        rank() over (partition by order_week order by total_orders desc) as day_rank
    from orders_by_day
)

select 
    order_week,
    order_day,
    total_orders as max_orders
from ranked_orders
where day_rank = 1
order by order_week;

##-- What is the monthly revenue trend?

select extract(month from orderTime) as order_month, sum(price*quantity) as revenue from order_table
group by extract(month from orderTime)
order by order_month

##-- Do weekends have higher order values compared to weekdays?

with cte as( 
	select  to_char(orderTime, 'Day') as day_name, extract(dow from orderTime) as order_weekday, sum(price*quantity) as order_values from order_table
	group by extract(dow from orderTime),to_char(orderTime, 'Day')
	order by order_weekday
)

select string_agg(day_name,', ') as weekday_order_value, sum(order_values) as order_value from cte 
where order_weekday between 1 and 5
union
select string_agg(day_name,', ') as weekday_order_value, sum(order_values) as order_value from cte 
where order_weekday in (0,6)

##-- What are the seasonal trends (if data spans multiple months)?

with cte as (
	select orderTime,
		case when extract(month from orderTime) in (12, 1, 2) then 'winter'
	      when extract(month from orderTime) in (3, 4, 5) then 'spring'
	      when extract(month from orderTime) in (6, 7, 8) then 'summer'
	      when extract(month from orderTime) in (9, 10, 11) then 'autumn'
		  else ''
	 	end as season 
from order_table)

select season, count(*) as total_order from cte
group by season
order by total_order

#-- 📈 Advanced Analysis

##-- What percentage of revenue comes from the top 20% of customers (Pareto analysis)?

with cte as(
	select customerName,sum(price*quantity) as revenue
	from order_table
	group by customerName
)
,
revenue_percent as (select revenue,
percent_rank() over(order by revenue desc ) as topcus
from cte)

select sum(revenue) as revenue, sum(topcus) as top_20_percent from revenue_percent 
where topcus<=.2


##-- Which category has the highest growth rate over time?

with monthly_sales as (
    select 
        category,
        extract(month from ordertime) as order_month,
        sum(price * quantity) as order_value
    from order_table
    group by category, extract(month from ordertime)
),
growth as (
    select 
        category,
        order_month,
        order_value,
        lag(order_value) over (partition by category order by order_month) as prev_value
    from monthly_sales
)
select 
    category,
    order_month,
    order_value,
    round(((order_value - prev_value) * 100.0 / nullif(prev_value,0))) as growth_rate_percent
from growth
order by category, order_month;
