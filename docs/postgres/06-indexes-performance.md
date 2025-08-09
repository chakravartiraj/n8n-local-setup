# Indexes and Performance

## 🎯 Learning Objectives
By the end of this module, you will:
- Understand how indexes work and when to use them
- Create and manage different types of indexes
- Analyze query performance with EXPLAIN
- Optimize slow queries systematically
- Monitor database performance effectively

## 📚 Table of Contents
1. [Understanding Indexes](#understanding-indexes)
2. [Index Types](#index-types)
3. [Creating and Managing Indexes](#creating-and-managing-indexes)
4. [Query Optimization](#query-optimization)
5. [Performance Analysis](#performance-analysis)
6. [Best Practices](#best-practices)
7. [Hands-On Exercises](#hands-on-exercises)
8. [Next Steps](#next-steps)

---

## Understanding Indexes

### What Are Indexes?
Indexes are data structures that improve query performance by creating shortcuts to find data quickly, similar to an index in a book.

### How Indexes Work
```sql
-- Without index: Sequential scan through all rows
SELECT * FROM employees WHERE last_name = 'Smith';

-- With index: Direct lookup using index structure
CREATE INDEX idx_employees_last_name ON employees(last_name);
SELECT * FROM employees WHERE last_name = 'Smith';
```

### Index Benefits and Costs

#### Benefits
- **Faster SELECT queries** with WHERE clauses
- **Faster JOIN operations** 
- **Faster ORDER BY** operations
- **Unique constraints** enforcement

#### Costs
- **Additional storage space** (typically 10-20% of table size)
- **Slower INSERT/UPDATE/DELETE** operations
- **Maintenance overhead** during data changes

### When to Use Indexes
```sql
-- Good candidates for indexing:
-- 1. Frequently queried columns in WHERE clauses
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- 2. Foreign key columns
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- 3. Columns used in JOIN conditions
CREATE INDEX idx_products_supplier_id ON products(supplier_id);

-- 4. Columns used in ORDER BY
CREATE INDEX idx_orders_order_date ON orders(order_date);

-- 5. Columns with high selectivity (many unique values)
CREATE INDEX idx_employees_email ON employees(email);
```

---

## Index Types

### B-Tree Indexes (Default)
Most common index type, good for equality and range queries.

```sql
-- Basic B-tree index
CREATE INDEX idx_employees_salary ON employees(salary);

-- Useful for:
SELECT * FROM employees WHERE salary = 75000;
SELECT * FROM employees WHERE salary > 70000;
SELECT * FROM employees WHERE salary BETWEEN 70000 AND 80000;
SELECT * FROM employees ORDER BY salary;
```

### Unique Indexes
Enforce uniqueness and provide fast lookups.

```sql
-- Unique index (automatically created with UNIQUE constraint)
CREATE UNIQUE INDEX idx_employees_email_unique ON employees(email);

-- Composite unique index
CREATE UNIQUE INDEX idx_employees_first_last_dept ON employees(first_name, last_name, department_id);
```

### Composite (Multi-column) Indexes
Index on multiple columns, order matters.

```sql
-- Composite index - order matters!
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Good for:
SELECT * FROM orders WHERE customer_id = 1;  -- Uses index
SELECT * FROM orders WHERE customer_id = 1 AND order_date = '2024-01-15';  -- Uses index fully
SELECT * FROM orders WHERE customer_id = 1 AND order_date > '2024-01-01';  -- Uses index fully

-- Not optimal for:
SELECT * FROM orders WHERE order_date = '2024-01-15';  -- Can't use index effectively
```

### Hash Indexes
Good for equality comparisons only.

```sql
-- Hash index (rarely used, limited functionality)
CREATE INDEX idx_products_sku_hash ON products USING HASH (sku);

-- Only useful for:
SELECT * FROM products WHERE sku = 'ABC123';

-- Not useful for:
SELECT * FROM products WHERE sku > 'ABC123';  -- Can't use hash index
```

### GIN (Generalized Inverted Index)
Excellent for array, JSONB, and full-text search.

```sql
-- Create table with array column
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    tags TEXT[],
    content JSONB
);

-- GIN index for array operations
CREATE INDEX idx_articles_tags ON articles USING GIN (tags);

-- GIN index for JSONB operations
CREATE INDEX idx_articles_content ON articles USING GIN (content);

-- Efficient queries:
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];
SELECT * FROM articles WHERE content @> '{"category": "database"}';
SELECT * FROM articles WHERE content ? 'author';
```

### GiST (Generalized Search Tree)
Good for geometric data and full-text search.

```sql
-- Create table with geometric data
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    coordinates POINT
);

-- GiST index for geometric operations
CREATE INDEX idx_locations_coordinates ON locations USING GIST (coordinates);

-- Efficient geometric queries:
SELECT * FROM locations WHERE coordinates <@ circle '((0,0), 10)';
```

### Partial Indexes
Index only rows that meet certain conditions.

```sql
-- Partial index for active employees only
CREATE INDEX idx_employees_active_salary ON employees(salary) 
WHERE is_active = true;

-- Partial index for recent orders
CREATE INDEX idx_orders_recent ON orders(order_date, customer_id)
WHERE order_date >= '2024-01-01';

-- Benefits: Smaller index size, faster maintenance
```

### Expression Indexes
Index on computed expressions.

```sql
-- Index on computed expression
CREATE INDEX idx_employees_full_name ON employees(lower(first_name || ' ' || last_name));

-- Index on function result
CREATE INDEX idx_orders_year ON orders(EXTRACT(YEAR FROM order_date));

-- Efficient queries:
SELECT * FROM employees WHERE lower(first_name || ' ' || last_name) = 'john doe';
SELECT * FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2024;
```

---

## Creating and Managing Indexes

### Creating Indexes

#### Basic Syntax
```sql
-- Basic index creation
CREATE INDEX index_name ON table_name (column_name);

-- Index with specific method
CREATE INDEX index_name ON table_name USING btree (column_name);

-- Composite index
CREATE INDEX index_name ON table_name (column1, column2, column3);

-- Unique index
CREATE UNIQUE INDEX index_name ON table_name (column_name);

-- Partial index
CREATE INDEX index_name ON table_name (column_name) WHERE condition;
```

#### Real-world Examples
```sql
-- Sample table for examples
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    product_id INTEGER,
    sale_date DATE,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    region VARCHAR(50)
);

-- Insert sample data
INSERT INTO sales (customer_id, product_id, sale_date, amount, status, region)
SELECT 
    (random() * 1000)::INTEGER + 1,
    (random() * 100)::INTEGER + 1,
    '2024-01-01'::DATE + (random() * 365)::INTEGER,
    (random() * 1000 + 10)::DECIMAL(10,2),
    CASE WHEN random() < 0.8 THEN 'completed' ELSE 'pending' END,
    CASE 
        WHEN random() < 0.3 THEN 'North'
        WHEN random() < 0.6 THEN 'South'
        WHEN random() < 0.8 THEN 'East'
        ELSE 'West'
    END
FROM generate_series(1, 100000);

-- Indexes for common query patterns
CREATE INDEX idx_sales_customer_id ON sales(customer_id);
CREATE INDEX idx_sales_product_id ON sales(product_id);
CREATE INDEX idx_sales_date ON sales(sale_date);
CREATE INDEX idx_sales_customer_date ON sales(customer_id, sale_date);
CREATE INDEX idx_sales_completed ON sales(sale_date) WHERE status = 'completed';
CREATE INDEX idx_sales_region_amount ON sales(region, amount);
```

### Managing Indexes

#### Viewing Indexes
```sql
-- List all indexes in current database
SELECT 
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;

-- Get index size information
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS index_size
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexname::regclass) DESC;

-- Check index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_tup_read,
    idx_tup_fetch,
    idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

#### Dropping Indexes
```sql
-- Drop specific index
DROP INDEX idx_sales_product_id;

-- Drop index if exists (safe)
DROP INDEX IF EXISTS idx_sales_product_id;

-- Drop multiple indexes
DROP INDEX idx_sales_customer_id, idx_sales_date;
```

#### Rebuilding Indexes
```sql
-- Rebuild index (useful after heavy data changes)
REINDEX INDEX idx_sales_customer_date;

-- Rebuild all indexes on a table
REINDEX TABLE sales;

-- Rebuild all indexes in database (rarely needed)
REINDEX DATABASE database_name;
```

### Concurrent Index Operations
```sql
-- Create index without blocking writes (PostgreSQL 8.2+)
CREATE INDEX CONCURRENTLY idx_sales_amount ON sales(amount);

-- Drop index without blocking (PostgreSQL 9.2+)
DROP INDEX CONCURRENTLY idx_sales_amount;

-- Note: CONCURRENTLY operations are slower but don't block other operations
```

---

## Query Optimization

### Using EXPLAIN
Understanding query execution plans is crucial for optimization.

#### Basic EXPLAIN
```sql
-- Basic execution plan
EXPLAIN SELECT * FROM sales WHERE customer_id = 500;

-- Detailed execution plan with actual runtime statistics
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM sales WHERE customer_id = 500;

-- Different output formats
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) 
SELECT * FROM sales WHERE customer_id = 500;
```

#### Reading EXPLAIN Output
```sql
-- Example query and execution plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    s.customer_id,
    COUNT(*) as order_count,
    SUM(s.amount) as total_amount
FROM sales s
WHERE s.sale_date >= '2024-06-01'
  AND s.status = 'completed'
GROUP BY s.customer_id
HAVING SUM(s.amount) > 1000
ORDER BY total_amount DESC;

-- Key metrics to understand:
-- 1. Cost: estimated cost (startup_cost..total_cost)
-- 2. Rows: estimated vs actual rows
-- 3. Time: actual time in milliseconds
-- 4. Buffers: pages read from disk vs cache
-- 5. Scan types: Seq Scan, Index Scan, Index Only Scan
```

### Query Optimization Techniques

#### Index Optimization
```sql
-- Before optimization: Sequential scan
EXPLAIN ANALYZE
SELECT * FROM sales WHERE region = 'North' AND amount > 500;

-- Create appropriate index
CREATE INDEX idx_sales_region_amount ON sales(region, amount);

-- After optimization: Index scan
EXPLAIN ANALYZE
SELECT * FROM sales WHERE region = 'North' AND amount > 500;
```

#### JOIN Optimization
```sql
-- Create related tables for JOIN examples
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10,2)
);

-- Insert sample data
INSERT INTO customers (name, email)
SELECT 
    'Customer ' || i,
    'customer' || i || '@example.com'
FROM generate_series(1, 1000) i;

INSERT INTO products (name, category, price)
SELECT 
    'Product ' || i,
    CASE WHEN i % 3 = 0 THEN 'Electronics' 
         WHEN i % 3 = 1 THEN 'Clothing'
         ELSE 'Books' END,
    (random() * 100 + 10)::DECIMAL(10,2)
FROM generate_series(1, 100) i;

-- Optimize JOIN with proper indexes
CREATE INDEX idx_sales_customer_fk ON sales(customer_id);
CREATE INDEX idx_sales_product_fk ON sales(product_id);

-- Optimized JOIN query
EXPLAIN ANALYZE
SELECT 
    c.name,
    p.name,
    s.amount,
    s.sale_date
FROM sales s
INNER JOIN customers c ON s.customer_id = c.id
INNER JOIN products p ON s.product_id = p.id
WHERE s.sale_date >= '2024-06-01'
  AND p.category = 'Electronics';
```

#### Subquery Optimization
```sql
-- Slow subquery approach
EXPLAIN ANALYZE
SELECT *
FROM customers c
WHERE c.id IN (
    SELECT customer_id 
    FROM sales 
    WHERE amount > 500
);

-- Faster EXISTS approach
EXPLAIN ANALYZE
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1 
    FROM sales s 
    WHERE s.customer_id = c.id 
      AND s.amount > 500
);

-- Often fastest: JOIN approach
EXPLAIN ANALYZE
SELECT DISTINCT c.*
FROM customers c
INNER JOIN sales s ON c.id = s.customer_id
WHERE s.amount > 500;
```

### Common Performance Issues

#### Missing Indexes
```sql
-- Identify slow queries without indexes
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
WHERE mean_time > 100  -- queries taking more than 100ms on average
ORDER BY mean_time DESC
LIMIT 10;
```

#### Inefficient WHERE Clauses
```sql
-- Inefficient: function on indexed column
SELECT * FROM sales WHERE EXTRACT(YEAR FROM sale_date) = 2024;

-- Efficient: range query on indexed column
SELECT * FROM sales WHERE sale_date >= '2024-01-01' AND sale_date < '2025-01-01';

-- Inefficient: leading wildcard
SELECT * FROM customers WHERE name LIKE '%john%';

-- Efficient: trailing wildcard with index
SELECT * FROM customers WHERE name LIKE 'john%';
```

#### Index Maintenance
```sql
-- Check for unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND pg_relation_size(indexname::regclass) > 1024*1024  -- larger than 1MB
ORDER BY pg_relation_size(indexname::regclass) DESC;

-- Check for duplicate indexes
WITH index_columns AS (
    SELECT 
        schemaname,
        tablename,
        indexname,
        array_agg(attname ORDER BY attnum) as columns
    FROM pg_index 
    JOIN pg_class ON pg_class.oid = pg_index.indexrelid
    JOIN pg_namespace ON pg_namespace.oid = pg_class.relnamespace
    JOIN pg_attribute ON pg_attribute.attrelid = pg_index.indrelid 
        AND pg_attribute.attnum = ANY(pg_index.indkey)
    GROUP BY schemaname, tablename, indexname
)
SELECT 
    schemaname,
    tablename,
    array_agg(indexname) as duplicate_indexes,
    columns
FROM index_columns
GROUP BY schemaname, tablename, columns
HAVING COUNT(*) > 1;
```

---

## Performance Analysis

### Database Statistics
```sql
-- Enable query statistics collection (if not already enabled)
-- Add to postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.track = all

-- View top slow queries
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    stddev_time,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

### Table and Index Statistics
```sql
-- Table sizes and statistics
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as index_size,
    n_tup_ins as inserts,
    n_tup_upd as updates,
    n_tup_del as deletes,
    n_live_tup as live_rows,
    n_dead_tup as dead_rows
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### Cache Hit Ratios
```sql
-- Buffer cache hit ratio (should be > 95%)
SELECT 
    'Buffer Cache Hit Ratio' as metric,
    ROUND(
        100.0 * sum(blks_hit) / (sum(blks_hit) + sum(blks_read)), 2
    ) as percentage
FROM pg_stat_database;

-- Index cache hit ratio
SELECT 
    'Index Cache Hit Ratio' as metric,
    ROUND(
        100.0 * sum(idx_blks_hit) / nullif(sum(idx_blks_hit) + sum(idx_blks_read), 0), 2
    ) as percentage
FROM pg_statio_user_indexes;
```

### Lock Monitoring
```sql
-- Current locks
SELECT 
    pg_class.relname,
    pg_locks.locktype,
    pg_locks.mode,
    pg_locks.granted,
    pg_stat_activity.query
FROM pg_locks
JOIN pg_class ON pg_locks.relation = pg_class.oid
JOIN pg_stat_activity ON pg_locks.pid = pg_stat_activity.pid
WHERE NOT pg_locks.granted
ORDER BY pg_locks.pid;

-- Blocking queries
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

---

## Best Practices

### Index Design Guidelines

1. **Index columns used in WHERE clauses**
2. **Index foreign key columns**
3. **Consider composite indexes for multi-column queries**
4. **Use partial indexes for filtered queries**
5. **Avoid over-indexing small tables**

### Performance Optimization Checklist

```sql
-- 1. Analyze query patterns
SELECT 
    query,
    calls,
    mean_time,
    total_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;

-- 2. Check for missing indexes
EXPLAIN (ANALYZE, BUFFERS) your_slow_query;

-- 3. Update table statistics
ANALYZE your_table;

-- 4. Consider VACUUM for heavily updated tables
VACUUM ANALYZE your_table;

-- 5. Monitor index usage
SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;
```

### Maintenance Tasks
```sql
-- Regular maintenance script
DO $$
DECLARE
    table_name TEXT;
BEGIN
    -- Update statistics for all tables
    FOR table_name IN 
        SELECT tablename FROM pg_tables WHERE schemaname = 'public'
    LOOP
        EXECUTE 'ANALYZE ' || table_name;
    END LOOP;
    
    -- Report maintenance completion
    RAISE NOTICE 'Statistics updated for all tables';
END $$;

-- Automated VACUUM and ANALYZE (consider pg_cron extension)
-- Or use autovacuum (enabled by default in PostgreSQL)
```

---

## Hands-On Exercises

### Exercise 1: Index Analysis and Creation

1. Create a large table with sample data
2. Analyze query performance without indexes
3. Create appropriate indexes
4. Compare performance before and after

```sql
-- Solution:

-- 1. Create large sample table
CREATE TABLE large_sales (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    product_id INTEGER,
    sale_date DATE,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    region VARCHAR(50),
    salesperson_id INTEGER
);

-- Insert 1 million rows
INSERT INTO large_sales (customer_id, product_id, sale_date, amount, status, region, salesperson_id)
SELECT 
    (random() * 10000)::INTEGER + 1,
    (random() * 1000)::INTEGER + 1,
    '2023-01-01'::DATE + (random() * 730)::INTEGER,
    (random() * 10000 + 10)::DECIMAL(10,2),
    CASE WHEN random() < 0.8 THEN 'completed' ELSE 'pending' END,
    CASE 
        WHEN random() < 0.25 THEN 'North'
        WHEN random() < 0.5 THEN 'South'
        WHEN random() < 0.75 THEN 'East'
        ELSE 'West'
    END,
    (random() * 100)::INTEGER + 1
FROM generate_series(1, 1000000);

-- 2. Test queries without indexes
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_sales WHERE customer_id = 5000;

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, SUM(amount) 
FROM large_sales 
WHERE sale_date >= '2024-01-01' AND region = 'North'
GROUP BY customer_id;

-- 3. Create appropriate indexes
CREATE INDEX idx_large_sales_customer ON large_sales(customer_id);
CREATE INDEX idx_large_sales_date_region ON large_sales(sale_date, region);
CREATE INDEX idx_large_sales_status ON large_sales(status);

-- 4. Test queries with indexes
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_sales WHERE customer_id = 5000;

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, SUM(amount) 
FROM large_sales 
WHERE sale_date >= '2024-01-01' AND region = 'North'
GROUP BY customer_id;
```

### Exercise 2: Query Optimization

1. Write a complex query with performance issues
2. Use EXPLAIN to identify bottlenecks
3. Optimize the query step by step
4. Measure improvements

```sql
-- Solution:

-- 1. Complex query with performance issues
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    c.name AS customer_name,
    p.name AS product_name,
    s.amount,
    s.sale_date,
    sp.name AS salesperson_name
FROM large_sales s
JOIN customers c ON s.customer_id = c.id
JOIN products p ON s.product_id = p.id
JOIN (
    SELECT id, 'Salesperson ' || id AS name 
    FROM generate_series(1, 100) id
) sp ON s.salesperson_id = sp.id
WHERE EXTRACT(YEAR FROM s.sale_date) = 2024
  AND s.amount > (SELECT AVG(amount) FROM large_sales)
  AND s.region IN ('North', 'South')
ORDER BY s.amount DESC, s.sale_date DESC
LIMIT 100;

-- 2. Optimization steps:

-- Step 1: Fix date filter
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    c.name AS customer_name,
    p.name AS product_name,
    s.amount,
    s.sale_date,
    sp.name AS salesperson_name
FROM large_sales s
JOIN customers c ON s.customer_id = c.id
JOIN products p ON s.product_id = p.id
JOIN (
    SELECT id, 'Salesperson ' || id AS name 
    FROM generate_series(1, 100) id
) sp ON s.salesperson_id = sp.id
WHERE s.sale_date >= '2024-01-01' AND s.sale_date < '2025-01-01'
  AND s.amount > (SELECT AVG(amount) FROM large_sales)
  AND s.region IN ('North', 'South')
ORDER BY s.amount DESC, s.sale_date DESC
LIMIT 100;

-- Step 2: Create materialized average
CREATE TEMP TABLE avg_amount AS 
SELECT AVG(amount) AS avg_val FROM large_sales;

-- Step 3: Optimized query
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    c.name AS customer_name,
    p.name AS product_name,
    s.amount,
    s.sale_date,
    'Salesperson ' || s.salesperson_id AS salesperson_name
FROM large_sales s
JOIN customers c ON s.customer_id = c.id
JOIN products p ON s.product_id = p.id
CROSS JOIN avg_amount a
WHERE s.sale_date >= '2024-01-01' AND s.sale_date < '2025-01-01'
  AND s.amount > a.avg_val
  AND s.region IN ('North', 'South')
ORDER BY s.amount DESC, s.sale_date DESC
LIMIT 100;

-- Add supporting indexes
CREATE INDEX idx_large_sales_complex ON large_sales(sale_date, region, amount) 
WHERE sale_date >= '2024-01-01';
```

### Exercise 3: Performance Monitoring Setup

1. Set up performance monitoring queries
2. Create a performance dashboard
3. Identify optimization opportunities
4. Implement monitoring alerts

```sql
-- Solution:

-- 1. Performance monitoring views
CREATE VIEW performance_summary AS
SELECT 
    'Top Slow Queries' AS category,
    query AS description,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
WHERE mean_time > 100
ORDER BY total_time DESC
LIMIT 10;

-- 2. Index usage monitoring
CREATE VIEW index_usage_report AS
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as size,
    CASE 
        WHEN idx_scan = 0 THEN 'UNUSED'
        WHEN idx_scan < 100 THEN 'LOW_USAGE'
        ELSE 'NORMAL'
    END AS usage_status
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- 3. Cache hit ratio monitoring
CREATE VIEW cache_hit_ratios AS
SELECT 
    'Buffer Cache' AS cache_type,
    ROUND(
        100.0 * sum(blks_hit) / (sum(blks_hit) + sum(blks_read)), 2
    ) AS hit_ratio
FROM pg_stat_database
UNION ALL
SELECT 
    'Index Cache' AS cache_type,
    ROUND(
        100.0 * sum(idx_blks_hit) / nullif(sum(idx_blks_hit) + sum(idx_blks_read), 0), 2
    ) AS hit_ratio
FROM pg_statio_user_indexes;

-- 4. Table bloat monitoring
CREATE VIEW table_bloat_report AS
SELECT 
    schemaname,
    tablename,
    n_live_tup,
    n_dead_tup,
    CASE 
        WHEN n_live_tup = 0 THEN 0
        ELSE ROUND(100.0 * n_dead_tup / (n_live_tup + n_dead_tup), 2)
    END AS bloat_percentage,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size
FROM pg_stat_user_tables
WHERE n_live_tup + n_dead_tup > 1000
ORDER BY bloat_percentage DESC;
```

---

## Next Steps

### Module Completion Checklist
- [ ] Understand different index types and when to use them
- [ ] Can read and interpret EXPLAIN output
- [ ] Know how to identify and fix performance bottlenecks
- [ ] Can monitor database performance effectively
- [ ] Understand the trade-offs of indexing strategies

### Immediate Next Steps
1. **Practice** with performance analysis on real datasets
2. **Monitor** your database performance regularly
3. **Experiment** with different index strategies
4. **Move to Module 7**: [Functions and Procedures](07-functions-procedures.md)

---

## Module Summary

✅ **What You've Learned:**
- Index types and their appropriate use cases
- Query optimization techniques and tools
- Performance analysis with EXPLAIN and statistics
- Database monitoring and maintenance strategies
- Best practices for sustainable performance

✅ **Skills Acquired:**
- Create and manage indexes effectively
- Analyze and optimize slow queries
- Monitor database performance metrics
- Identify and resolve performance bottlenecks
- Design efficient database schemas

✅ **Ready for Next Module:**
You're now ready to learn about custom functions and procedures in [Module 7: Functions and Procedures](07-functions-procedures.md).

---

## Quick Reference

### Index Types
```sql
-- B-tree (default)
CREATE INDEX idx_name ON table(column);

-- Unique
CREATE UNIQUE INDEX idx_name ON table(column);

-- Composite
CREATE INDEX idx_name ON table(col1, col2);

-- Partial
CREATE INDEX idx_name ON table(column) WHERE condition;

-- Expression
CREATE INDEX idx_name ON table(function(column));

-- GIN (for arrays, JSONB)
CREATE INDEX idx_name ON table USING GIN(column);
```

### Performance Analysis
```sql
-- Query plan
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- Index usage
SELECT * FROM pg_stat_user_indexes;

-- Slow queries
SELECT * FROM pg_stat_statements ORDER BY total_time DESC;

-- Cache hit ratio
SELECT 100.0 * sum(blks_hit) / (sum(blks_hit) + sum(blks_read)) 
FROM pg_stat_database;
```

---
*Continue your PostgreSQL journey with [Functions and Procedures](07-functions-procedures.md) →*
