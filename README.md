# Домашнее задание 3. Системы хранения и обработки данных

<!-- ## Headers -->
## Пролог
Работу выполнил: Игошин Дмитрий Владимирович. Все картинки находятся тут, в README.md. Код тоже

<!-- # This is a Heading h1 -->
## Описание
<!-- ###### This is a Heading h6 -->
Перед началом работы с данными, их нужно загрузить. Предварительно я создал структуру базы данных в DBeaver.

![This is an alt text.](/image/db_schema.png "This is a sample image.")

Макет базы данных создавал с помощью кода:
```
CREATE TABLE "customer" (
  "customer_id" integer PRIMARY KEY,
  "first_name" varchar NOT NULL,
  "last_name" varchar,
  "gender" varchar NOT NULL,
  "DOB" date,
  "job_title" varchar,
  "job_industry_category" varchar NOT NULL,
  "wealth_segment" varchar NOT NULL,
  "deceased_indicator" varchar NOT NULL,
  "owns_car" varchar NOT NULL,
  "address" varchar NOT NULL,
  "postcode" smallint NOT NULL,
  "state" varchar NOT NULL,
  "country" varchar NOT NULL,
  "property_valuation" smallint NOT NULL
);

CREATE TABLE "product" (
  "product_id" integer PRIMARY KEY,
  "brand" varchar,
  "product_line" varchar,
  "product_class" varchar,
  "product_size" varchar,
  "list_price" float4 NOT NULL,
  "standard_cost" float4
);

CREATE TABLE "orders" (
  "order_id" integer PRIMARY KEY,
  "customer_id" integer NOT NULL,
  "order_date" date NOT NULL,
  "online_order" bool,
  "order_status" varchar NOT NULL
);

CREATE TABLE "order_items" (
  "order_item_id" integer PRIMARY KEY,
  "order_id" integer NOT NULL,
  "product_id" integer NOT NULL,
  "quantity" float4 NOT NULL,
  "item_list_price_at_sale" float4 NOT NULL,
  "item_standard_cost_at_sale" float4
);
```
Значения ключей между таблицами не связывал.

Так как табличка product содержала неоднозначные соответсвия между product_id и всем остальным, было решено использовать немного модифицированные данные (не без помощи преподавателя). С помощью SQL запроса были взяты только те строки, у которых впервые упоминается product_id. Результат записал в таблицу prooduct_cor
```
create table product_cor as
 select *
 from (
  select *
   ,row_number() over(partition by product_id order by list_price desc) as rn
  from product)
 where rn = 1
```
Скриншоты успешного создания баз данных:

![This is an alt text.](/image/success_bd_create_1.png "This is a sample image.")
![This is an alt text.](/image/success_bd_create_2.png "This is a sample image.")
![This is an alt text.](/image/success_bd_create_3.png "This is a sample image.")
![This is an alt text.](/image/success_bd_create_4.png "This is a sample image.")
![This is an alt text.](/image/success_bd_create_5.png "This is a sample image.")

Для форматирования кода использовал комбинацию ```CTRL + Shift + F``` в Dbeaver

## Задачи
### Задача 1. 

```
select
	job_industry_category,
	count(customer_id)
from
	customer
where job_industry_category <> 'n/a'	
group by
	job_industry_category
order by
	count(customer_id) desc
```
Значения n/a я не учитывал. Тем не менее, в таблице они были заполнены. Для их учета нужно просто удалить where


![This is an alt text.](/image/1.png "This is a sample image.")

### Задача 2. 

```
select
	to_char(o.order_date, 'YYYY') "year",
	extract(month from o.order_date) "month",
	c.job_industry_category "job industry category",
	sum(oi.item_list_price_at_sale * oi.quantity) "sum of orders"
from
	(order_items oi
left join orders o on
	oi.order_item_id = o.order_id)
natural join customer c
group by
	to_char(o.order_date, 'YYYY'),
	extract(month from o.order_date),
	c.job_industry_category
order by
	to_char(o.order_date, 'YYYY'),
	extract(month from o.order_date),
	c.job_industry_category
```

![This is an alt text.](/image/2.png "This is a sample image.")

### Задача 3. 

```
(
select
	distinct pc.brand ,
	count(o.online_order)
from
	(orders o
join order_items oi on
	o.order_id = oi.order_item_id)
natural join product_cor pc
natural join customer c
where
	c.job_industry_category = 'IT'
	and o.order_status = 'Approved'
group by
	pc.brand)
union	
	(
select
		distinct pc.brand,
		0 "count"
from
		product_cor pc
natural join order_items oi
join orders o on
		o.order_id = oi.order_item_id
natural join customer c
group by
		pc.brand,
		c.job_industry_category
having
		(c.job_industry_category = 'IT'
	and count(o.online_order) = 0))				
```

Брендов, у которых нет онлайн-заказов от IT-клиентов, с количеством 0, я не обнаружил. Да и left join в критериях - не единственный вариант решения этой задачи

![This is an alt text.](/image/3.png "This is a sample image.")


### Задача 4. 

```
select 
	    o.customer_id,
	sum(oi.item_list_price_at_sale * oi.quantity) total_revenue,
	max(oi.item_list_price_at_sale * oi.quantity) max_order_sum,
	min(oi.item_list_price_at_sale * oi.quantity) min_order_sum,
	count(distinct o.order_id) number_of_orders,
	avg(order_total.order_sum) avg_order_sum
from
	orders o
join order_items oi on
	o.order_id = oi.order_item_id
join (
	select
		order_id,
		sum(item_list_price_at_sale * quantity) order_sum
	from
		order_items
	group by
		order_id) order_total on
	o.order_id = order_total.order_id
group by
	o.customer_id
order by
	total_revenue desc,
	number_of_orders desc
```


![This is an alt text.](/image/4.png "This is a sample image.")

### Задача 5. 

```
(
select
	first_name,
	last_name
from
	customer c
natural join orders o
join order_items oi on
	oi.order_item_id = o.order_id
group by
	1,
	2
order by
	sum(oi.item_list_price_at_sale * oi.quantity) desc
limit 3
)
union
(
select
first_name,
last_name
from
customer c
natural join orders o
join order_items oi on
oi.order_item_id = o.order_id
group by
1,
2
order by
sum(oi.item_list_price_at_sale * oi.quantity)
limit 3
)

select
	first_name,
	last_name
from
	customer c
natural join orders o
join order_items oi on
	oi.order_item_id = o.order_id
group by 1,2
having count(oi.quantity) = 0
```

![This is an alt text.](/image/5.png "This is a sample image.")

### Задача 6. 

```
with tab_res as(select
	customer_id,
	order_date,
	order_id,
	online_order,
	order_status,
	row_number() over (
            partition by customer_id
order by
	order_date
        ) as rn 
from orders   
group by 1, 2, 3
order by 1, 2, 3)
select
	customer_id,
	order_date,
	order_id,
	online_order,
	order_status
from
	tab_res
where
	rn = 2
```



![This is an alt text.](/image/6.png "This is a sample image.")

### Задача 7. 

```
select
	distinct t.first_name,
	t.last_name,
	t.job_title,
	max(t.time_diff) "max time diff"
from
	(
	select
		c.first_name,
		c.last_name,
		c.job_title,
		order_date,
		(o.order_date - lag(o.order_date, 1, o.order_date) over (partition by c.first_name,
		c.last_name,
		c.job_title
	order by
		order_date)) time_diff
	from
		customer c
	natural join orders o
	order by
		1,
		2,
		order_date) t
group by
	1,
	2,
	3
having
	max(t.time_diff) <> 0
order by
	1,
	2,
	3
```

Клиенты, у которых меньше 1 заказа, были отфильтрованы через условие "разность между заказами" = 0. Это следует из заданной lag функции, у которой в случае NaN значения, происходит замена на order_date. В таком случае разность получается равна 0

![This is an alt text.](/image/7.png "This is a sample image.")

### Задача 8. 

```
with customer_revenue as(select
	first_name,
	last_name,
	wealth_segment,
	sum(oi.item_list_price_at_sale * oi.quantity) total_revenue
from
	order_items oi
join orders o on
	o.order_id = oi.order_item_id
natural join customer c
group by
	1,
	2,
	3
order by
	sum(oi.item_list_price_at_sale * oi.quantity) desc),
ranked_customers as(select
	first_name,
	last_name,
	wealth_segment,
	total_revenue,
	dense_rank() over (partition by wealth_segment
order by
	total_revenue desc) segment_rank
from
	customer_revenue
)
select * from ranked_customers 
where segment_rank <= 5
order by wealth_segment, total_revenue desc
```

![This is an alt text.](/image/8.png "This is a sample image.")
