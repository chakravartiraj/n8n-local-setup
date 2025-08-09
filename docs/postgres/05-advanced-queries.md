# Advanced Queries

## 🎯 Learning Objectives
By the end of this module, you will:
- Master complex JOIN operations across multiple tables
- Write sophisticated subqueries and Common Table Expressions (CTEs)
- Use window functions for analytical queries
- Apply advanced aggregation and grouping techniques
- Optimize query performance with proper techniques

## 📚 Table of Contents
1. [JOIN Operations](#join-operations)
2. [Subqueries](#subqueries)
3. [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
4. [Window Functions](#window-functions)
5. [Advanced Aggregation](#advanced-aggregation)
6. [Set Operations](#set-operations)
7. [Hands-On Exercises](#hands-on-exercises)
8. [Performance Considerations](#performance-considerations)
9. [Next Steps](#next-steps)

---

## JOIN Operations

### Sample Database Setup
```sql
-- Create comprehensive sample database
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    city VARCHAR(50),
    registration_date DATE DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    order_date DATE DEFAULT CURRENT_DATE,
    total_amount DECIMAL(10,2),
    status VARCHAR(20) DEFAULT 'pending'
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category VARCHAR(50),
    price DECIMAL(10,2),
    supplier_id INTEGER
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER,
    unit_price DECIMAL(10,2)
);

CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    country VARCHAR(50),
    contact_email VARCHAR(100)
);

-- Insert sample data
INSERT INTO customers (name, email, city) VALUES
    ('John Doe', 'john@email.com', 'New York'),
    ('Jane Smith', 'jane@email.com', 'London'),
    ('Bob Johnson', 'bob@email.com', 'Tokyo'),
    ('Alice Wilson', 'alice@email.com', 'Paris'),
    ('Charlie Brown', 'charlie@email.com', 'Sydney');

INSERT INTO suppliers (name, country, contact_email) VALUES
    ('Tech Corp', 'USA', 'contact@techcorp.com'),
    ('Global Systems', 'UK', 'info@globalsys.com'),
    ('Asian Manufacturing', 'Japan', 'sales@asianmfg.com');

INSERT INTO products (name, category, price, supplier_id) VALUES
    ('Laptop Pro', 'Electronics', 1299.99, 1),
    ('Wireless Mouse', 'Electronics', 29.99, 1),
    ('Office Chair', 'Furniture', 199.99, 2),
    ('Standing Desk', 'Furniture', 399.99, 2),
    ('Coffee Maker', 'Appliances', 89.99, 3);

INSERT INTO orders (customer_id, order_date, total_amount, status) VALUES
    (1, '2024-01-15', 1329.98, 'completed'),
    (2, '2024-01-20', 199.99, 'completed'),
    (1, '2024-02-01', 89.99, 'pending'),
    (3, '2024-02-05', 429.98, 'shipped'),
    (4, '2024-02-10', 29.99, 'completed');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 1299.99), (1, 2, 1, 29.99),
    (2, 3, 1, 199.99),
    (3, 5, 1, 89.99),
    (4, 4, 1, 399.99), (4, 2, 1, 29.99),
    (5, 2, 1, 29.99);
```

### INNER JOIN
Returns only rows that have matching values in both tables.

```sql
-- Basic INNER JOIN
SELECT 
    c.name AS customer_name,
    o.order_date,
    o.total_amount
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

-- Multiple table INNER JOIN
SELECT 
    c.name AS customer_name,
    o.order_date,
    p.name AS product_name,
    oi.quantity,
    oi.unit_price
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id;
```

### LEFT JOIN (LEFT OUTER JOIN)
Returns all rows from the left table and matching rows from the right table.

```sql
-- Show all customers and their orders (including customers with no orders)
SELECT 
    c.name AS customer_name,
    c.email,
    o.order_date,
    o.total_amount,
    COALESCE(o.status, 'No orders') AS order_status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
ORDER BY c.name, o.order_date;

-- Count orders per customer (including customers with 0 orders)
SELECT 
    c.name AS customer_name,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total_amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
```

### RIGHT JOIN (RIGHT OUTER JOIN)
Returns all rows from the right table and matching rows from the left table.

```sql
-- Show all orders and customer info (including orders without customer data)
SELECT 
    o.id AS order_id,
    o.order_date,
    o.total_amount,
    c.name AS customer_name,
    c.email
FROM customers c
RIGHT JOIN orders o ON c.id = o.customer_id;
```

### FULL OUTER JOIN
Returns all rows from both tables, with NULLs where there's no match.

```sql
-- Show all customers and all orders, whether they match or not
SELECT 
    c.name AS customer_name,
    c.email,
    o.order_date,
    o.total_amount
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id
ORDER BY c.name, o.order_date;
```

### CROSS JOIN
Returns the Cartesian product of both tables (every row from first table with every row from second table).

```sql
-- Generate all possible customer-product combinations
SELECT 
    c.name AS customer_name,
    p.name AS product_name,
    p.price
FROM customers c
CROSS JOIN products p
WHERE p.category = 'Electronics'
ORDER BY c.name, p.price;
```

### Self JOIN
Join a table with itself.

```sql
-- Create employee hierarchy table for self-join example
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER REFERENCES employees(id),
    department VARCHAR(50)
);

INSERT INTO employees (name, manager_id, department) VALUES
    ('CEO Boss', NULL, 'Executive'),
    ('Sales Manager', 1, 'Sales'),
    ('Dev Manager', 1, 'Engineering'),
    ('Sales Rep 1', 2, 'Sales'),
    ('Sales Rep 2', 2, 'Sales'),
    ('Developer 1', 3, 'Engineering'),
    ('Developer 2', 3, 'Engineering');

-- Find employees and their managers
SELECT 
    e.name AS employee_name,
    e.department,
    m.name AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
ORDER BY e.department, e.name;

-- Find managers and count their direct reports
SELECT 
    m.name AS manager_name,
    COUNT(e.id) AS direct_reports
FROM employees m
LEFT JOIN employees e ON m.id = e.manager_id
WHERE m.id IS NOT NULL
GROUP BY m.id, m.name
HAVING COUNT(e.id) > 0
ORDER BY direct_reports DESC;
```

### Advanced JOIN Techniques

#### Multiple JOIN Conditions
```sql
-- Join with multiple conditions
SELECT 
    c.name AS customer_name,
    o.order_date,
    o.total_amount
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id 
    AND o.order_date >= '2024-02-01'
    AND o.status = 'completed';
```

#### JOIN with CASE Statements
```sql
-- Complex JOIN with conditional logic
SELECT 
    c.name AS customer_name,
    COUNT(o.id) AS order_count,
    SUM(o.total_amount) AS total_spent,
    CASE 
        WHEN SUM(o.total_amount) > 1000 THEN 'VIP'
        WHEN SUM(o.total_amount) > 500 THEN 'Premium'
        WHEN SUM(o.total_amount) > 0 THEN 'Regular'
        ELSE 'New'
    END AS customer_tier
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
```

---

## Subqueries

### Scalar Subqueries
Return a single value.

```sql
-- Find customers who spent more than the average
SELECT 
    name,
    email,
    (SELECT SUM(total_amount) 
     FROM orders 
     WHERE customer_id = customers.id) AS total_spent
FROM customers
WHERE (SELECT SUM(total_amount) 
       FROM orders 
       WHERE customer_id = customers.id) > 
      (SELECT AVG(total_amount) 
       FROM orders);

-- Add average order value to each order
SELECT 
    o.id,
    o.order_date,
    o.total_amount,
    (SELECT AVG(total_amount) FROM orders) AS avg_order_value,
    o.total_amount - (SELECT AVG(total_amount) FROM orders) AS difference_from_avg
FROM orders o
ORDER BY difference_from_avg DESC;
```

### Correlated Subqueries
Reference columns from the outer query.

```sql
-- Find customers with above-average order values
SELECT 
    c.name,
    c.email
FROM customers c
WHERE (SELECT AVG(o.total_amount) 
       FROM orders o 
       WHERE o.customer_id = c.id) > 
      (SELECT AVG(total_amount) FROM orders);

-- Find products that have been ordered
SELECT 
    p.name,
    p.category,
    p.price
FROM products p
WHERE EXISTS (
    SELECT 1 
    FROM order_items oi 
    WHERE oi.product_id = p.id
);

-- Find products that have never been ordered
SELECT 
    p.name,
    p.category,
    p.price
FROM products p
WHERE NOT EXISTS (
    SELECT 1 
    FROM order_items oi 
    WHERE oi.product_id = p.id
);
```

### Subqueries with IN/NOT IN
```sql
-- Find customers who have placed orders
SELECT name, email
FROM customers
WHERE id IN (SELECT DISTINCT customer_id FROM orders);

-- Find customers who haven't placed any orders
SELECT name, email
FROM customers
WHERE id NOT IN (
    SELECT customer_id 
    FROM orders 
    WHERE customer_id IS NOT NULL
);

-- Find products in categories that have been ordered
SELECT name, category, price
FROM products
WHERE category IN (
    SELECT DISTINCT p.category
    FROM products p
    INNER JOIN order_items oi ON p.id = oi.product_id
);
```

### Subqueries with ANY/ALL
```sql
-- Find products more expensive than ANY Electronics product
SELECT name, category, price
FROM products
WHERE price > ANY (
    SELECT price 
    FROM products 
    WHERE category = 'Electronics'
);

-- Find products more expensive than ALL Furniture products
SELECT name, category, price
FROM products
WHERE price > ALL (
    SELECT price 
    FROM products 
    WHERE category = 'Furniture'
);
```

---

## Common Table Expressions (CTEs)

### Basic CTE
```sql
-- Simple CTE example
WITH customer_totals AS (
    SELECT 
        customer_id,
        COUNT(*) AS order_count,
        SUM(total_amount) AS total_spent
    FROM orders
    GROUP BY customer_id
)
SELECT 
    c.name,
    c.email,
    ct.order_count,
    ct.total_spent
FROM customers c
INNER JOIN customer_totals ct ON c.id = ct.customer_id
ORDER BY ct.total_spent DESC;
```

### Multiple CTEs
```sql
-- Multiple CTEs with different calculations
WITH customer_stats AS (
    SELECT 
        customer_id,
        COUNT(*) AS order_count,
        SUM(total_amount) AS total_spent,
        AVG(total_amount) AS avg_order_value
    FROM orders
    GROUP BY customer_id
),
product_stats AS (
    SELECT 
        p.id,
        p.name,
        p.category,
        COUNT(oi.id) AS times_ordered,
        SUM(oi.quantity) AS total_quantity_sold
    FROM products p
    LEFT JOIN order_items oi ON p.id = oi.product_id
    GROUP BY p.id, p.name, p.category
)
SELECT 
    c.name AS customer_name,
    cs.order_count,
    cs.total_spent,
    cs.avg_order_value,
    ps.name AS most_expensive_product,
    ps.times_ordered
FROM customers c
INNER JOIN customer_stats cs ON c.id = cs.customer_id
CROSS JOIN LATERAL (
    SELECT ps.name, ps.times_ordered
    FROM product_stats ps
    ORDER BY ps.times_ordered DESC
    LIMIT 1
) ps;
```

### Recursive CTEs
```sql
-- Recursive CTE for employee hierarchy
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: top-level managers (no manager)
    SELECT 
        id,
        name,
        manager_id,
        department,
        0 AS level,
        name AS hierarchy_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees with managers
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        e.department,
        eh.level + 1,
        eh.hierarchy_path || ' -> ' || e.name
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT 
    level,
    REPEAT('  ', level) || name AS indented_name,
    department,
    hierarchy_path
FROM employee_hierarchy
ORDER BY hierarchy_path;

-- Recursive CTE for number series
WITH RECURSIVE number_series AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1
    FROM number_series
    WHERE n < 10
)
SELECT * FROM number_series;
```

### CTE for Data Transformation
```sql
-- Complex data transformation using CTE
WITH monthly_sales AS (
    SELECT 
        EXTRACT(YEAR FROM order_date) AS year,
        EXTRACT(MONTH FROM order_date) AS month,
        SUM(total_amount) AS monthly_total,
        COUNT(*) AS order_count
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)
),
sales_with_growth AS (
    SELECT 
        year,
        month,
        monthly_total,
        order_count,
        LAG(monthly_total) OVER (ORDER BY year, month) AS previous_month_total,
        monthly_total - LAG(monthly_total) OVER (ORDER BY year, month) AS growth_amount
    FROM monthly_sales
)
SELECT 
    year,
    month,
    monthly_total,
    order_count,
    previous_month_total,
    growth_amount,
    CASE 
        WHEN previous_month_total IS NULL THEN NULL
        WHEN previous_month_total = 0 THEN NULL
        ELSE ROUND((growth_amount / previous_month_total) * 100, 2)
    END AS growth_percentage
FROM sales_with_growth
ORDER BY year, month;
```

---

## Window Functions

### Basic Window Functions

#### ROW_NUMBER, RANK, DENSE_RANK
```sql
-- Ranking customers by total spending
SELECT 
    c.name,
    SUM(o.total_amount) AS total_spent,
    ROW_NUMBER() OVER (ORDER BY SUM(o.total_amount) DESC) AS row_num,
    RANK() OVER (ORDER BY SUM(o.total_amount) DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY SUM(o.total_amount) DESC) AS dense_rank
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;

-- Ranking products within each category
SELECT 
    name,
    category,
    price,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS rank_in_category
FROM products
ORDER BY category, rank_in_category;
```

#### LAG and LEAD
```sql
-- Compare each order with previous and next order for the same customer
SELECT 
    c.name AS customer_name,
    o.order_date,
    o.total_amount,
    LAG(o.total_amount) OVER (PARTITION BY c.id ORDER BY o.order_date) AS previous_order,
    LEAD(o.total_amount) OVER (PARTITION BY c.id ORDER BY o.order_date) AS next_order,
    o.total_amount - LAG(o.total_amount) OVER (PARTITION BY c.id ORDER BY o.order_date) AS change_from_previous
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
ORDER BY c.name, o.order_date;
```

#### FIRST_VALUE and LAST_VALUE
```sql
-- Show first and last order for each customer
SELECT 
    c.name AS customer_name,
    o.order_date,
    o.total_amount,
    FIRST_VALUE(o.total_amount) OVER (
        PARTITION BY c.id 
        ORDER BY o.order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_order_amount,
    LAST_VALUE(o.total_amount) OVER (
        PARTITION BY c.id 
        ORDER BY o.order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_order_amount
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
ORDER BY c.name, o.order_date;
```

### Aggregate Window Functions
```sql
-- Running totals and moving averages
SELECT 
    order_date,
    total_amount,
    SUM(total_amount) OVER (ORDER BY order_date) AS running_total,
    AVG(total_amount) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3,
    COUNT(*) OVER (ORDER BY order_date) AS cumulative_order_count,
    total_amount / SUM(total_amount) OVER () * 100 AS percentage_of_total
FROM orders
ORDER BY order_date;
```

### Advanced Window Function Examples
```sql
-- Customer segmentation using window functions
WITH customer_metrics AS (
    SELECT 
        c.id,
        c.name,
        c.registration_date,
        COUNT(o.id) AS order_count,
        COALESCE(SUM(o.total_amount), 0) AS total_spent,
        COALESCE(AVG(o.total_amount), 0) AS avg_order_value
    FROM customers c
    LEFT JOIN orders o ON c.id = o.customer_id
    GROUP BY c.id, c.name, c.registration_date
),
customer_segments AS (
    SELECT 
        *,
        NTILE(4) OVER (ORDER BY total_spent) AS spending_quartile,
        NTILE(3) OVER (ORDER BY order_count) AS frequency_tertile,
        PERCENT_RANK() OVER (ORDER BY total_spent) AS spending_percentile
    FROM customer_metrics
)
SELECT 
    name,
    total_spent,
    order_count,
    spending_quartile,
    frequency_tertile,
    ROUND(spending_percentile * 100, 1) AS spending_percentile,
    CASE 
        WHEN spending_quartile = 4 AND frequency_tertile >= 2 THEN 'VIP'
        WHEN spending_quartile >= 3 THEN 'High Value'
        WHEN frequency_tertile >= 2 THEN 'Frequent'
        ELSE 'Regular'
    END AS customer_segment
FROM customer_segments
ORDER BY total_spent DESC;
```

---

## Advanced Aggregation

### GROUPING SETS
```sql
-- Multiple grouping levels in one query
SELECT 
    COALESCE(p.category, 'ALL CATEGORIES') AS category,
    COALESCE(DATE_TRUNC('month', o.order_date)::TEXT, 'ALL MONTHS') AS month,
    COUNT(*) AS order_count,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM orders o
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id
GROUP BY GROUPING SETS (
    (p.category, DATE_TRUNC('month', o.order_date)),
    (p.category),
    (DATE_TRUNC('month', o.order_date)),
    ()
)
ORDER BY category, month;
```

### ROLLUP and CUBE
```sql
-- ROLLUP for hierarchical aggregation
SELECT 
    COALESCE(p.category, 'TOTAL') AS category,
    COALESCE(p.name, 'SUBTOTAL') AS product_name,
    SUM(oi.quantity) AS total_quantity,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM products p
INNER JOIN order_items oi ON p.id = oi.product_id
GROUP BY ROLLUP(p.category, p.name)
ORDER BY p.category, p.name;

-- CUBE for all possible combinations
SELECT 
    COALESCE(p.category, 'ALL') AS category,
    COALESCE(o.status, 'ALL') AS status,
    COUNT(*) AS order_count,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM orders o
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id
GROUP BY CUBE(p.category, o.status)
ORDER BY category, status;
```

### FILTER Clause
```sql
-- Conditional aggregation
SELECT 
    c.name,
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE o.status = 'completed') AS completed_orders,
    COUNT(*) FILTER (WHERE o.status = 'pending') AS pending_orders,
    SUM(o.total_amount) AS total_spent,
    SUM(o.total_amount) FILTER (WHERE o.order_date >= '2024-02-01') AS recent_spending
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
```

---

## Set Operations

### UNION and UNION ALL
```sql
-- Combine customer emails and supplier emails
SELECT email, 'customer' AS type FROM customers WHERE email IS NOT NULL
UNION
SELECT contact_email, 'supplier' AS type FROM suppliers WHERE contact_email IS NOT NULL
ORDER BY type, email;

-- UNION ALL includes duplicates
SELECT 'High Value' AS category, name FROM customers WHERE id IN (1, 2)
UNION ALL
SELECT 'Frequent Buyer' AS category, name FROM customers WHERE id IN (2, 3)
ORDER BY category, name;
```

### INTERSECT and EXCEPT
```sql
-- Find customers who are also suppliers (by email domain)
SELECT name FROM customers
WHERE SPLIT_PART(email, '@', 2) IN (
    SELECT SPLIT_PART(contact_email, '@', 2) 
    FROM suppliers
);

-- Products that exist but haven't been ordered
SELECT p.name
FROM products p
EXCEPT
SELECT p.name
FROM products p
INNER JOIN order_items oi ON p.id = oi.product_id;
```

---

## Hands-On Exercises

### Exercise 1: Complex JOIN Practice
Write queries for the following business scenarios:

1. Find all customers with their total orders and spending
2. List products that have never been ordered
3. Show monthly sales by category
4. Find the most popular product in each category

```sql
-- Solutions:

-- 1. Customer summary with total orders and spending
SELECT 
    c.name,
    c.email,
    c.city,
    COUNT(o.id) AS total_orders,
    COALESCE(SUM(o.total_amount), 0) AS total_spent,
    COALESCE(AVG(o.total_amount), 0) AS avg_order_value
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.email, c.city
ORDER BY total_spent DESC;

-- 2. Products never ordered
SELECT 
    p.name,
    p.category,
    p.price
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE oi.product_id IS NULL;

-- 3. Monthly sales by category
SELECT 
    DATE_TRUNC('month', o.order_date) AS month,
    p.category,
    COUNT(DISTINCT o.id) AS orders,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.quantity * oi.unit_price) AS total_sales
FROM orders o
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id
GROUP BY DATE_TRUNC('month', o.order_date), p.category
ORDER BY month, category;

-- 4. Most popular product in each category
WITH product_popularity AS (
    SELECT 
        p.id,
        p.name,
        p.category,
        COUNT(oi.id) AS times_ordered,
        SUM(oi.quantity) AS total_quantity,
        ROW_NUMBER() OVER (PARTITION BY p.category ORDER BY COUNT(oi.id) DESC) AS popularity_rank
    FROM products p
    LEFT JOIN order_items oi ON p.id = oi.product_id
    GROUP BY p.id, p.name, p.category
)
SELECT 
    category,
    name AS most_popular_product,
    times_ordered,
    total_quantity
FROM product_popularity
WHERE popularity_rank = 1
ORDER BY category;
```

### Exercise 2: Window Functions and Analytics
Create analytical queries using window functions:

1. Customer purchase behavior analysis
2. Product sales trend analysis
3. Running totals and growth rates
4. Customer segmentation

```sql
-- Solutions:

-- 1. Customer purchase behavior analysis
WITH customer_orders AS (
    SELECT 
        c.name AS customer_name,
        o.order_date,
        o.total_amount,
        LAG(o.order_date) OVER (PARTITION BY c.id ORDER BY o.order_date) AS previous_order_date,
        LAG(o.total_amount) OVER (PARTITION BY c.id ORDER BY o.order_date) AS previous_order_amount
    FROM customers c
    INNER JOIN orders o ON c.id = o.customer_id
)
SELECT 
    customer_name,
    order_date,
    total_amount,
    previous_order_date,
    CASE 
        WHEN previous_order_date IS NOT NULL 
        THEN order_date - previous_order_date 
        ELSE NULL 
    END AS days_between_orders,
    total_amount - COALESCE(previous_order_amount, 0) AS amount_change
FROM customer_orders
ORDER BY customer_name, order_date;

-- 2. Product sales trend analysis
SELECT 
    p.name AS product_name,
    DATE_TRUNC('month', o.order_date) AS month,
    SUM(oi.quantity) AS monthly_quantity,
    LAG(SUM(oi.quantity)) OVER (PARTITION BY p.id ORDER BY DATE_TRUNC('month', o.order_date)) AS previous_month_quantity,
    SUM(oi.quantity) - LAG(SUM(oi.quantity)) OVER (PARTITION BY p.id ORDER BY DATE_TRUNC('month', o.order_date)) AS quantity_change
FROM products p
INNER JOIN order_items oi ON p.id = oi.product_id
INNER JOIN orders o ON oi.order_id = o.id
GROUP BY p.id, p.name, DATE_TRUNC('month', o.order_date)
ORDER BY p.name, month;

-- 3. Running totals and growth rates
SELECT 
    order_date,
    total_amount,
    SUM(total_amount) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS running_total,
    AVG(total_amount) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3_days,
    (total_amount / LAG(total_amount) OVER (ORDER BY order_date) - 1) * 100 AS daily_growth_rate
FROM orders
ORDER BY order_date;

-- 4. Customer segmentation using RFM analysis
WITH customer_rfm AS (
    SELECT 
        c.id,
        c.name,
        EXTRACT(DAYS FROM CURRENT_DATE - MAX(o.order_date)) AS recency_days,
        COUNT(o.id) AS frequency,
        SUM(o.total_amount) AS monetary_value
    FROM customers c
    LEFT JOIN orders o ON c.id = o.customer_id
    GROUP BY c.id, c.name
),
customer_scores AS (
    SELECT 
        *,
        NTILE(5) OVER (ORDER BY recency_days DESC) AS recency_score,
        NTILE(5) OVER (ORDER BY frequency) AS frequency_score,
        NTILE(5) OVER (ORDER BY monetary_value) AS monetary_score
    FROM customer_rfm
)
SELECT 
    name,
    recency_days,
    frequency,
    monetary_value,
    recency_score,
    frequency_score,
    monetary_score,
    CASE 
        WHEN recency_score >= 4 AND frequency_score >= 4 AND monetary_score >= 4 THEN 'Champions'
        WHEN recency_score >= 3 AND frequency_score >= 3 AND monetary_score >= 3 THEN 'Loyal Customers'
        WHEN recency_score >= 3 AND frequency_score <= 2 THEN 'Potential Loyalists'
        WHEN recency_score <= 2 AND frequency_score >= 3 THEN 'At Risk'
        WHEN recency_score <= 2 AND frequency_score <= 2 THEN 'Lost Customers'
        ELSE 'Regular'
    END AS customer_segment
FROM customer_scores
ORDER BY monetary_value DESC;
```

---

## Performance Considerations

### Query Optimization Tips
```sql
-- Use EXPLAIN to analyze query performance
EXPLAIN (ANALYZE, BUFFERS) 
SELECT c.name, COUNT(o.id) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- Use EXISTS instead of IN for better performance
SELECT c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- Use appropriate indexes
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_order_items_order_product ON order_items(order_id, product_id);
```

### Common Performance Pitfalls
```sql
-- Avoid functions in WHERE clauses on large tables
-- Bad:
SELECT * FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2024;

-- Good:
SELECT * FROM orders WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';

-- Use LIMIT for exploratory queries
SELECT * FROM large_table LIMIT 100;

-- Use appropriate data types for JOINs
-- Ensure JOIN columns have same data types
```

---

## Next Steps

### Module Completion Checklist
- [ ] Master all JOIN types and their use cases
- [ ] Write complex subqueries and correlated subqueries
- [ ] Use CTEs for query organization and readability
- [ ] Apply window functions for analytical queries
- [ ] Understand performance implications of complex queries

### Immediate Next Steps
1. **Practice** with increasingly complex scenarios
2. **Analyze** query performance using EXPLAIN
3. **Experiment** with different query approaches
4. **Move to Module 6**: [Indexes and Performance](06-indexes-performance.md)

---

## Module Summary

✅ **What You've Learned:**
- Advanced JOIN operations and techniques
- Subqueries and Common Table Expressions
- Window functions for analytical queries
- Advanced aggregation with GROUPING SETS, ROLLUP, CUBE
- Set operations and query optimization basics

✅ **Skills Acquired:**
- Write complex multi-table queries
- Perform sophisticated data analysis
- Optimize query performance
- Use window functions for business analytics
- Structure complex queries for readability

✅ **Ready for Next Module:**
You're now ready to learn about database performance optimization in [Module 6: Indexes and Performance](06-indexes-performance.md).

---

## Quick Reference

### JOIN Types
```sql
INNER JOIN    -- Only matching rows
LEFT JOIN     -- All left + matching right
RIGHT JOIN    -- All right + matching left
FULL JOIN     -- All rows from both tables
CROSS JOIN    -- Cartesian product
```

### Window Functions
```sql
ROW_NUMBER() OVER (ORDER BY column)
RANK() OVER (PARTITION BY col1 ORDER BY col2)
LAG(column, 1) OVER (ORDER BY date)
SUM(column) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
```

### CTEs
```sql
WITH cte_name AS (
    SELECT ...
)
SELECT ... FROM cte_name;

-- Recursive CTE
WITH RECURSIVE cte AS (
    SELECT ... -- Base case
    UNION ALL
    SELECT ... FROM cte -- Recursive case
)
```

---
*Continue your PostgreSQL journey with [Indexes and Performance](06-indexes-performance.md) →*
