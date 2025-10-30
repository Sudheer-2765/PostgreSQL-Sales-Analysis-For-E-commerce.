
# End-to-End E-Commerce Sales Analysis in PostgreSQL

This project is a complete, end-to-end sales analysis of a Brazilian e-commerce dataset. The goal was to build a normalized relational database from scratch, perform an ETL process to load the data, and then run complex queries to derive actionable business insights.

---

## 1. Schema Design (schema.sql)

I designed a 5-table normalized relational schema to ensure data integrity and reduce redundancy. Parent tables (`customers`, `products`) were created first, followed by child tables (`orders`) and finally the transaction tables (`order_payments`, `order_items`).

```sql
-- 1. Create the customers table (Parent table)
CREATE TABLE customers (
    customer_id VARCHAR(50) PRIMARY KEY,
    customer_unique_id VARCHAR(50),
    customer_zip_code_prefix INT,
    customer_city VARCHAR(100),
    customer_state VARCHAR(50)
);

-- 2. Create the products table (Parent table)
CREATE TABLE products (
    product_id VARCHAR(50) PRIMARY KEY,
    product_category_name VARCHAR(100),
    product_name_length INT,
    product_description_length INT,
    product_photos_qty INT,
    product_weight_g INT,
    product_length_cm INT,
    product_height_cm INT,
    product_width_cm INT
);

-- 3. Create the orders table (Links to customers)
CREATE TABLE orders (
    order_id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50),
    order_status VARCHAR(50),
    order_purchase_timestamp TIMESTAMP,
    order_approved_at TIMESTAMP,
    order_delivered_carrier_date TIMESTAMP,
    order_delivered_customer_date TIMESTAMP,
    order_estimated_delivery_date TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- 4. Create the order_payments table (Links to orders)
CREATE TABLE order_payments (
    order_id VARCHAR(50),
    payment_sequential INT,
    payment_type VARCHAR(50),
    payment_installments INT,
    payment_value FLOAT,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- 5. Create the order_items table (Links to orders AND products)
CREATE TABLE order_items (
    order_id VARCHAR(50),
    order_item_id INT,
    product_id VARCHAR(50),
    seller_id VARCHAR(50),
    shipping_limit_date TIMESTAMP,
    price FLOAT,
    freight_value FLOAT,
    PRIMARY KEY (order_id, order_item_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);


ETL Process (Extract, Transform, Load)
Extract: Data was provided as 5 raw .csv files (customers, products, orders, payments, items).

Transform: I discovered the .csv files were delimited by a semicolon (;), not a comma. I also had to manage foreign key constraints during loading (e.g., TRUNCATE ... CASCADE) to handle import errors.

Load: I used the pgAdmin GUI "Import/Export" tool, setting the correct (;) delimiter and enabling the "Header" option. I loaded parent tables first to satisfy foreign key dependencies.

Analysis & Insights (analysis.sql)
I authored complex queries using JOINs, CTEs, Aggregate Functions, and GROUP BY to answer key business questions.

Query 1: Top 10 Spending Customers
This query joins three tables (customers, orders, order_payments) to identify the 10 customers with the highest total spending on delivered orders.
SELECT
    c.customer_unique_id,
    SUM(op.payment_value) AS total_spent
FROM
    customers c
JOIN
    orders o ON c.customer_id = o.customer_id
JOIN
    order_payments op ON o.order_id = op.order_id
WHERE
    o.order_status = 'delivered'
GROUP BY
    c.customer_unique_id
ORDER BY
    total_spent DESC
LIMIT 10;

/*What this does: It links customers to their orders, and those orders to their payments. Then, it groups by the unique customer ID, sums up all their payments, and shows you the top 10 biggest spenders.*/
Results:

| "customer_unique_id"               | "total_spent"     |   |
|------------------------------------|-------------------|---|
| "0a0a92112bd4c708ca5fde585afaa872" | 13664.08          |   |
| "da122df9eeddfedc1dc1f5349a1a690c" | 7571.63           |   |
| "763c8b1c9c68a0229c42c9fc6f662b93" | 7274.88           |   |
| "dc4802a71eae9be1dd28f5d788ceb526" | 6929.31           |   |
| "459bef486812aa25204be022145caa62" | 6922.21           |   |
| "ff4159b92c40ebe40454e3e6a7c35ed6" | 6726.66           |   |
| "4007669dec559734d6f53e029e360987" | 6081.54           |   |
| "eebb5dda148d3893cdaf5b5ca3040ccb" | 4764.34           |   |
| "48e1ac109decbb87765a3eade6854098" | 4681.78           |   |
| "c8460e4251689ba205045f3ea17884a1" | 4655.910000000001 |   |
|                                    |                   |   |
|                                    |                   |   |

Query 2: Top 10 Best-Selling Product Categories
This query joins products and order_items to count the number of sales per product category, identifying the most popular categories.
SELECT
    p.product_category_name,
    COUNT(oi.product_id) AS number_of_sales
FROM
    products p
JOIN
    order_items oi ON p.product_id = oi.product_id
WHERE
    p.product_category_name IS NOT NULL
GROUP BY
    p.product_category_name
ORDER BY
    number_of_sales DESC
LIMIT 10;
/*What this does: It links products to the transaction table (order_items). Then, it groups by the product's category name, counts how many times a product from that category was sold, and shows you the top 10 categories.*/
Results:
| "product_category_name"  | "number_of_sales" |   |
|--------------------------|-------------------|---|
| "cama_mesa_banho"        | 11115             |   |
| "beleza_saude"           | 9670              |   |
| "esporte_lazer"          | 8641              |   |
| "moveis_decoracao"       | 8334              |   |
| "informatica_acessorios" | 7827              |   |
| "utilidades_domesticas"  | 6964              |   |
| "relogios_presentes"     | 5991              |   |
| "telefonia"              | 4545              |   |
| "ferramentas_jardim"     | 4347              |   |
| "automotivo"             | 4235              |   |
|                          |                   |   |
|                          |                   |   |

Query 3: Monthly Sales Patterns (using a CTE)
This query uses a Common Table Expression (CTE) to first calculate the total value of each order, then joins with the orders table to analyze sales trends (average order value and number of orders) month-by-month.
WITH order_values AS (
    SELECT
        order_id,
        SUM(payment_value) AS total_order_value
    FROM
        order_payments
    GROUP BY
        order_id
)
-- Now analyze the pattern of these order values over time
SELECT
    EXTRACT(YEAR FROM o.order_purchase_timestamp) AS sale_year,
    EXTRACT(MONTH FROM o.order_purchase_timestamp) AS sale_month,
    AVG(ov.total_order_value) AS average_order_value,
    COUNT(o.order_id) AS number_of_orders
FROM
    orders o
JOIN
    order_values ov ON o.order_id = ov.order_id
WHERE
    o.order_status = 'delivered'
GROUP BY
    sale_year,
    sale_month
ORDER BY
    sale_year,
    sale_month;

/*What this does:

The CTE (the WITH part) first creates a temporary table called order_values that has the total value for each order.

The main query then links this order_values table to the orders table to get the date.

It groups by year and month to show you the average sales value and number of orders for each month.*/

Results:
| "sale_year" | "sale_month" | "average_order_value" | "number_of_orders" |
|-------------|--------------|-----------------------|--------------------|
| 2016        | 10           | 175.72343396226412    | 265                |
| 2016        | 12           | 19.62                 | 1                  |
| 2017        | 1            | 170.06089333333333    | 750                |
| 2017        | 2            | 164.1250151240165     | 1653               |
| 2017        | 3            | 162.7530989787905     | 2546               |
| 2017        | 4            | 169.75778549717765    | 2303               |
| 2017        | 5            | 159.91729554427633    | 3546               |
| 2017        | 6            | 156.37180223285552    | 3135               |
| 2017        | 7            | 146.2820067148769     | 3872               |
| 2017        | 8            | 154.06644645838338    | 4193               |
| 2017        | 9            | 168.95662409638493    | 4150               |
| 2017        | 10           | 167.74012282268885    | 4478               |
| 2017        | 11           | 158.25600905473965    | 7289               |
| 2017        | 12           | 152.9474278976963     | 5513               |
| 2018        | 1            | 152.58266515773087    | 7069               |
| 2018        | 2            | 147.44635850495825    | 6555               |
| 2018        | 3            | 160.02827359702897    | 7003               |
| 2018        | 4            | 166.65695057369717    | 6798               |
| 2018        | 5            | 167.2598444213961     | 6749               |
| 2018        | 6            | 165.94370880472118    | 6099               |
| 2018        | 7            | 166.89460301997104    | 6159               |
| 2018        | 8            | 155.15891670603037    | 6351               |

Done! Thank You.
