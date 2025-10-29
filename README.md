
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
