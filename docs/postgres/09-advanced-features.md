# Advanced Features

## 🎯 Learning Objectives
By the end of this module, you will:
- Master advanced PostgreSQL features for complex applications
- Implement window functions and CTEs for sophisticated analytics
- Use JSON/JSONB effectively for semi-structured data
- Create and optimize custom data types and operators
- Implement full-text search and spatial data handling
- Understand and apply advanced indexing strategies

## 📚 Table of Contents
1. [Window Functions](#window-functions)
2. [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
3. [JSON and JSONB](#json-and-jsonb)
4. [Custom Data Types](#custom-data-types)
5. [Full-Text Search](#full-text-search)
6. [Spatial Data (PostGIS)](#spatial-data-postgis)
7. [Advanced Indexing](#advanced-indexing)
8. [Extensions and Plugins](#extensions-and-plugins)
9. [Hands-On Exercises](#hands-on-exercises)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)

---

## Window Functions

### Basic Window Functions

#### Row Number and Ranking
```sql
-- Sample data for demonstrations
CREATE TABLE sales_data (
    id SERIAL PRIMARY KEY,
    salesperson VARCHAR(100),
    region VARCHAR(50),
    product VARCHAR(100),
    sale_amount DECIMAL(10,2),
    sale_date DATE
);

INSERT INTO sales_data (salesperson, region, product, sale_amount, sale_date) VALUES
('John Smith', 'North', 'Laptop', 1200.00, '2024-01-15'),
('Jane Doe', 'South', 'Mouse', 25.00, '2024-01-16'),
('Bob Johnson', 'North', 'Keyboard', 75.00, '2024-01-17'),
('Alice Brown', 'East', 'Monitor', 300.00, '2024-01-18'),
('John Smith', 'North', 'Tablet', 450.00, '2024-01-19'),
('Jane Doe', 'South', 'Laptop', 1100.00, '2024-01-20');

-- Row numbering and ranking
SELECT 
    salesperson,
    region,
    sale_amount,
    -- Simple row number
    ROW_NUMBER() OVER (ORDER BY sale_amount DESC) AS row_num,
    
    -- Ranking with ties
    RANK() OVER (ORDER BY sale_amount DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY sale_amount DESC) AS dense_rank,
    
    -- Percentile rank
    PERCENT_RANK() OVER (ORDER BY sale_amount) AS percent_rank,
    
    -- Row number within partition
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY sale_amount DESC) AS region_rank
FROM sales_data;
```

#### Aggregate Window Functions
```sql
-- Running totals and cumulative statistics
SELECT 
    salesperson,
    sale_date,
    sale_amount,
    
    -- Running total
    SUM(sale_amount) OVER (ORDER BY sale_date) AS running_total,
    
    -- Running average
    AVG(sale_amount) OVER (ORDER BY sale_date) AS running_avg,
    
    -- Running count
    COUNT(*) OVER (ORDER BY sale_date) AS running_count,
    
    -- Cumulative max/min
    MAX(sale_amount) OVER (ORDER BY sale_date) AS cumulative_max,
    MIN(sale_amount) OVER (ORDER BY sale_date) AS cumulative_min,
    
    -- Moving average (3-row window)
    AVG(sale_amount) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3,
    
    -- Percentage of total
    sale_amount / SUM(sale_amount) OVER () * 100 AS pct_of_total
FROM sales_data
ORDER BY sale_date;
```

### Advanced Window Functions

#### Lead and Lag Functions
```sql
-- Comparing current row with previous/next rows
SELECT 
    salesperson,
    sale_date,
    sale_amount,
    
    -- Previous and next values
    LAG(sale_amount) OVER (ORDER BY sale_date) AS prev_sale,
    LEAD(sale_amount) OVER (ORDER BY sale_date) AS next_sale,
    
    -- Multiple rows back/forward
    LAG(sale_amount, 2) OVER (ORDER BY sale_date) AS sale_2_back,
    
    -- With default values
    LAG(sale_amount, 1, 0) OVER (ORDER BY sale_date) AS prev_sale_default,
    
    -- Calculate differences
    sale_amount - LAG(sale_amount) OVER (ORDER BY sale_date) AS change_from_prev,
    
    -- Percentage change
    CASE 
        WHEN LAG(sale_amount) OVER (ORDER BY sale_date) > 0 THEN
            (sale_amount - LAG(sale_amount) OVER (ORDER BY sale_date)) / 
            LAG(sale_amount) OVER (ORDER BY sale_date) * 100
    END AS pct_change
FROM sales_data
ORDER BY sale_date;
```

#### First and Last Value Functions
```sql
-- Getting first and last values in window
SELECT 
    salesperson,
    region,
    sale_amount,
    sale_date,
    
    -- First and last in entire dataset
    FIRST_VALUE(sale_amount) OVER (ORDER BY sale_date) AS first_sale_ever,
    LAST_VALUE(sale_amount) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_sale_ever,
    
    -- First and last within partition
    FIRST_VALUE(sale_amount) OVER (
        PARTITION BY region 
        ORDER BY sale_date
    ) AS first_sale_in_region,
    
    LAST_VALUE(sale_amount) OVER (
        PARTITION BY region 
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_sale_in_region,
    
    -- Nth value
    NTH_VALUE(sale_amount, 2) OVER (
        PARTITION BY region 
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_sale_in_region
FROM sales_data
ORDER BY region, sale_date;
```

### Complex Window Scenarios

#### Time-Series Analysis
```sql
-- Time-series specific analysis
SELECT 
    DATE_TRUNC('month', sale_date) AS month,
    region,
    SUM(sale_amount) AS monthly_sales,
    
    -- Year-over-year comparison
    LAG(SUM(sale_amount), 12) OVER (
        PARTITION BY region 
        ORDER BY DATE_TRUNC('month', sale_date)
    ) AS same_month_last_year,
    
    -- Monthly growth rate
    (SUM(sale_amount) - LAG(SUM(sale_amount)) OVER (
        PARTITION BY region 
        ORDER BY DATE_TRUNC('month', sale_date)
    )) / LAG(SUM(sale_amount)) OVER (
        PARTITION BY region 
        ORDER BY DATE_TRUNC('month', sale_date)
    ) * 100 AS monthly_growth_rate,
    
    -- 3-month moving average
    AVG(SUM(sale_amount)) OVER (
        PARTITION BY region 
        ORDER BY DATE_TRUNC('month', sale_date)
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3months,
    
    -- Rank months by performance within region
    RANK() OVER (
        PARTITION BY region 
        ORDER BY SUM(sale_amount) DESC
    ) AS performance_rank
FROM sales_data
GROUP BY DATE_TRUNC('month', sale_date), region
ORDER BY region, month;
```

---

## Common Table Expressions (CTEs)

### Basic CTEs

#### Simple CTE Usage
```sql
-- Basic CTE for readability
WITH high_value_sales AS (
    SELECT 
        salesperson,
        region,
        sale_amount,
        sale_date
    FROM sales_data
    WHERE sale_amount > 500
)
SELECT 
    region,
    COUNT(*) AS high_value_count,
    AVG(sale_amount) AS avg_high_value_amount,
    SUM(sale_amount) AS total_high_value
FROM high_value_sales
GROUP BY region
ORDER BY total_high_value DESC;
```

#### Multiple CTEs
```sql
-- Multiple CTEs in single query
WITH regional_stats AS (
    SELECT 
        region,
        COUNT(*) AS total_sales,
        SUM(sale_amount) AS total_amount,
        AVG(sale_amount) AS avg_amount
    FROM sales_data
    GROUP BY region
),
salesperson_stats AS (
    SELECT 
        salesperson,
        region,
        COUNT(*) AS sales_count,
        SUM(sale_amount) AS total_amount
    FROM sales_data
    GROUP BY salesperson, region
),
top_performers AS (
    SELECT 
        salesperson,
        region,
        total_amount,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY total_amount DESC) AS rank
    FROM salesperson_stats
)
SELECT 
    r.region,
    r.total_sales AS region_total_sales,
    r.total_amount AS region_total_amount,
    tp.salesperson AS top_performer,
    tp.total_amount AS top_performer_amount,
    tp.total_amount / r.total_amount * 100 AS pct_of_region_sales
FROM regional_stats r
JOIN top_performers tp ON r.region = tp.region
WHERE tp.rank = 1
ORDER BY r.total_amount DESC;
```

### Recursive CTEs

#### Hierarchical Data Traversal
```sql
-- Create hierarchical organization structure
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    manager_id INTEGER REFERENCES employees(id),
    department VARCHAR(50),
    salary DECIMAL(10,2),
    hire_date DATE
);

INSERT INTO employees (name, manager_id, department, salary, hire_date) VALUES
('CEO John', NULL, 'Executive', 200000, '2020-01-01'),
('VP Sales Mary', 1, 'Sales', 150000, '2020-02-01'),
('VP Tech Bob', 1, 'Technology', 160000, '2020-02-01'),
('Sales Mgr Alice', 2, 'Sales', 100000, '2020-03-01'),
('Dev Mgr Charlie', 3, 'Technology', 120000, '2020-03-01'),
('Sales Rep Dave', 4, 'Sales', 70000, '2020-04-01'),
('Developer Eve', 5, 'Technology', 90000, '2020-04-01'),
('Developer Frank', 5, 'Technology', 85000, '2020-05-01');

-- Recursive CTE to build organizational hierarchy
WITH RECURSIVE org_hierarchy AS (
    -- Base case: start with CEO (no manager)
    SELECT 
        id,
        name,
        manager_id,
        department,
        salary,
        0 AS level,
        ARRAY[name] AS path,
        name AS hierarchy_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: find direct reports
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        e.department,
        e.salary,
        oh.level + 1,
        oh.path || e.name,
        oh.hierarchy_path || ' > ' || e.name
    FROM employees e
    JOIN org_hierarchy oh ON e.manager_id = oh.id
)
SELECT 
    REPEAT('  ', level) || name AS indented_name,
    hierarchy_path,
    level,
    department,
    salary,
    -- Calculate team size (including self)
    (SELECT COUNT(*) FROM org_hierarchy oh2 
     WHERE oh2.path[1:array_length(oh.path, 1)] = oh.path) AS team_size
FROM org_hierarchy
ORDER BY path;
```

#### Series Generation
```sql
-- Generate date series using recursive CTE
WITH RECURSIVE date_series AS (
    SELECT '2024-01-01'::DATE AS date_value
    
    UNION ALL
    
    SELECT date_value + INTERVAL '1 day'
    FROM date_series
    WHERE date_value < '2024-12-31'::DATE
),
monthly_targets AS (
    SELECT 
        DATE_TRUNC('month', date_value) AS month,
        CASE EXTRACT(MONTH FROM date_value)
            WHEN 1 THEN 50000
            WHEN 2 THEN 60000
            WHEN 3 THEN 75000
            ELSE 80000
        END AS target_amount
    FROM date_series
    GROUP BY DATE_TRUNC('month', date_value)
)
SELECT 
    mt.month,
    mt.target_amount,
    COALESCE(SUM(sd.sale_amount), 0) AS actual_amount,
    mt.target_amount - COALESCE(SUM(sd.sale_amount), 0) AS variance
FROM monthly_targets mt
LEFT JOIN sales_data sd ON DATE_TRUNC('month', sd.sale_date) = mt.month
GROUP BY mt.month, mt.target_amount
ORDER BY mt.month;
```

---

## JSON and JSONB

### JSON vs JSONB

#### Basic JSON Operations
```sql
-- Create table with JSON columns
CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    profile_data JSON,
    preferences JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert sample data
INSERT INTO user_profiles (username, profile_data, preferences) VALUES
('john_doe', 
 '{"name": "John Doe", "age": 30, "email": "john@example.com", "skills": ["PostgreSQL", "Python", "JavaScript"]}',
 '{"theme": "dark", "notifications": {"email": true, "push": false}, "language": "en"}'
),
('jane_smith',
 '{"name": "Jane Smith", "age": 25, "email": "jane@example.com", "skills": ["React", "Node.js", "MongoDB"]}',
 '{"theme": "light", "notifications": {"email": false, "push": true}, "language": "es"}'
);

-- Basic JSON querying
SELECT 
    username,
    profile_data->>'name' AS full_name,
    profile_data->>'age' AS age,
    profile_data->'skills' AS skills_array,
    profile_data->>'email' AS email,
    
    -- JSONB operations (more efficient)
    preferences->>'theme' AS theme,
    preferences->'notifications'->>'email' AS email_notifications
FROM user_profiles;
```

#### Advanced JSON Querying
```sql
-- Complex JSON queries
SELECT 
    username,
    profile_data->>'name' AS name,
    
    -- Extract array elements
    jsonb_array_elements_text(profile_data->'skills') AS skill
FROM user_profiles
WHERE jsonb_typeof(profile_data->'skills') = 'array';

-- Aggregating JSON data
SELECT 
    skill,
    COUNT(*) AS user_count
FROM (
    SELECT jsonb_array_elements_text(profile_data->'skills') AS skill
    FROM user_profiles
) skills_breakdown
GROUP BY skill
ORDER BY user_count DESC;

-- Path-based queries
SELECT 
    username,
    preferences #> '{notifications, email}' AS email_pref,
    preferences #>> '{notifications, push}' AS push_pref_text
FROM user_profiles
WHERE preferences #>> '{notifications, email}' = 'true';
```

### JSON Manipulation

#### Updating JSON Data
```sql
-- Update entire JSON object
UPDATE user_profiles 
SET preferences = '{"theme": "auto", "notifications": {"email": true, "push": true}, "language": "fr"}'
WHERE username = 'john_doe';

-- Update specific JSON keys
UPDATE user_profiles 
SET preferences = jsonb_set(preferences, '{theme}', '"light"')
WHERE username = 'john_doe';

-- Update nested JSON
UPDATE user_profiles 
SET preferences = jsonb_set(
    preferences, 
    '{notifications, push}', 
    'true'
)
WHERE username = 'jane_smith';

-- Add new key to JSON
UPDATE user_profiles 
SET preferences = preferences || '{"timezone": "UTC"}'::jsonb
WHERE username = 'john_doe';

-- Remove keys from JSON
UPDATE user_profiles 
SET preferences = preferences - 'timezone'
WHERE username = 'john_doe';

-- Update array elements
UPDATE user_profiles 
SET profile_data = jsonb_set(
    profile_data,
    '{skills}',
    (profile_data->'skills')::jsonb || '["Docker"]'::jsonb
)
WHERE username = 'john_doe';
```

#### JSON Aggregation
```sql
-- Aggregate JSON data
SELECT 
    jsonb_object_agg(username, profile_data->>'name') AS user_names,
    jsonb_agg(preferences->>'theme') AS themes,
    jsonb_agg(DISTINCT preferences->>'language') AS languages
FROM user_profiles;

-- Build complex JSON objects
SELECT 
    jsonb_build_object(
        'total_users', COUNT(*),
        'themes', jsonb_object_agg(
            preferences->>'theme', 
            COUNT(*)
        ),
        'average_age', AVG((profile_data->>'age')::INTEGER),
        'skills_summary', (
            SELECT jsonb_object_agg(skill, skill_count)
            FROM (
                SELECT 
                    skill,
                    COUNT(*) AS skill_count
                FROM (
                    SELECT jsonb_array_elements_text(profile_data->'skills') AS skill
                    FROM user_profiles
                ) t
                GROUP BY skill
            ) skill_stats
        )
    ) AS user_analytics
FROM user_profiles;
```

### JSON Indexing and Performance

#### Creating JSON Indexes
```sql
-- GIN index on entire JSONB column
CREATE INDEX idx_preferences_gin ON user_profiles USING GIN (preferences);

-- Specific path index
CREATE INDEX idx_theme ON user_profiles USING BTREE ((preferences->>'theme'));

-- Expression index for complex queries
CREATE INDEX idx_email_notifications ON user_profiles 
USING BTREE ((preferences #>> '{notifications, email}'));

-- Functional index for JSON array containment
CREATE INDEX idx_skills_gin ON user_profiles 
USING GIN ((profile_data->'skills'));

-- Test query performance
EXPLAIN (ANALYZE, BUFFERS) 
SELECT username 
FROM user_profiles 
WHERE preferences->>'theme' = 'dark';

EXPLAIN (ANALYZE, BUFFERS) 
SELECT username 
FROM user_profiles 
WHERE profile_data->'skills' ? 'PostgreSQL';
```

---

## Custom Data Types

### Enumerated Types

#### Creating and Using Enums
```sql
-- Create custom enum types
CREATE TYPE order_status AS ENUM (
    'pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled'
);

CREATE TYPE priority_level AS ENUM ('low', 'medium', 'high', 'urgent');

-- Create table using custom types
CREATE TABLE support_tickets (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    status order_status DEFAULT 'pending',
    priority priority_level DEFAULT 'medium',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Insert data using enum values
INSERT INTO support_tickets (title, description, status, priority) VALUES
('Login Issue', 'Cannot access account', 'pending', 'high'),
('Feature Request', 'Add dark mode', 'confirmed', 'low'),
('Bug Report', 'Page not loading', 'processing', 'urgent');

-- Query using enum values
SELECT 
    title,
    status,
    priority,
    CASE priority
        WHEN 'urgent' THEN 1
        WHEN 'high' THEN 2
        WHEN 'medium' THEN 3
        WHEN 'low' THEN 4
    END AS priority_rank
FROM support_tickets
ORDER BY priority_rank, created_at;

-- Enum comparison and ordering
SELECT * FROM support_tickets 
WHERE priority >= 'medium' AND status != 'cancelled'
ORDER BY priority DESC, created_at;
```

#### Modifying Enums
```sql
-- Add new values to enum
ALTER TYPE order_status ADD VALUE 'on_hold' AFTER 'confirmed';
ALTER TYPE priority_level ADD VALUE 'critical' BEFORE 'urgent';

-- Cannot remove enum values directly, need to recreate
-- or use a different approach for flexible status systems
```

### Composite Types

#### Creating Composite Types
```sql
-- Define composite types
CREATE TYPE address_type AS (
    street VARCHAR(200),
    city VARCHAR(100),
    state VARCHAR(50),
    postal_code VARCHAR(20),
    country VARCHAR(50)
);

CREATE TYPE contact_info AS (
    email VARCHAR(100),
    phone VARCHAR(20),
    website VARCHAR(200)
);

-- Table using composite types
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    address address_type,
    contact contact_info,
    founded_date DATE,
    employee_count INTEGER
);

-- Insert data using composite types
INSERT INTO companies (name, address, contact, founded_date, employee_count) VALUES
('Tech Corp', 
 ROW('123 Tech St', 'San Francisco', 'CA', '94105', 'USA')::address_type,
 ROW('info@techcorp.com', '+1-555-0123', 'https://techcorp.com')::contact_info,
 '2010-05-15',
 150
),
('Data Solutions',
 ROW('456 Data Ave', 'New York', 'NY', '10001', 'USA')::address_type,
 ROW('contact@datasolutions.com', '+1-555-0456', 'https://datasolutions.com')::contact_info,
 '2015-08-20',
 75
);

-- Query composite type fields
SELECT 
    name,
    (address).city AS city,
    (address).state AS state,
    (contact).email AS email,
    (contact).website AS website
FROM companies;

-- Filter using composite type fields
SELECT name, (address).city, (contact).email
FROM companies
WHERE (address).state = 'CA';
```

### Domain Types

#### Creating Domain Types
```sql
-- Create domain types with constraints
CREATE DOMAIN email_address AS VARCHAR(320)
CHECK (VALUE ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

CREATE DOMAIN phone_number AS VARCHAR(20)
CHECK (VALUE ~ '^\+?[1-9]\d{1,14}$');

CREATE DOMAIN positive_decimal AS DECIMAL(15,2)
CHECK (VALUE >= 0);

CREATE DOMAIN year_value AS INTEGER
CHECK (VALUE BETWEEN 1900 AND 2100);

-- Use domain types in table
CREATE TABLE customers_with_domains (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email email_address,
    phone phone_number,
    credit_limit positive_decimal DEFAULT 0,
    birth_year year_value
);

-- Domain constraints are automatically enforced
INSERT INTO customers_with_domains (name, email, phone, credit_limit, birth_year) VALUES
('John Doe', 'john.doe@example.com', '+1-555-0123', 5000.00, 1985);

-- This would fail due to domain constraints:
-- INSERT INTO customers_with_domains (name, email) VALUES ('Invalid', 'not-an-email');
```

---

## Full-Text Search

### Basic Text Search

#### Setting Up Text Search
```sql
-- Create table for text search demonstration
CREATE TABLE blog_posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    content TEXT NOT NULL,
    author VARCHAR(100),
    tags TEXT[],
    published_date DATE,
    tsvector_col TSVECTOR -- Pre-computed search vector
);

-- Insert sample blog posts
INSERT INTO blog_posts (title, content, author, tags, published_date) VALUES
('Introduction to PostgreSQL', 
 'PostgreSQL is a powerful open-source relational database management system. It supports advanced features like JSON, full-text search, and spatial data.',
 'John Smith',
 ARRAY['postgresql', 'database', 'sql'],
 '2024-01-15'
),
('Advanced SQL Techniques',
 'Learn about window functions, CTEs, and complex queries in PostgreSQL. These techniques will help you write more efficient and readable SQL code.',
 'Jane Doe',
 ARRAY['sql', 'postgresql', 'advanced'],
 '2024-01-20'
),
('JSON in PostgreSQL',
 'PostgreSQL provides excellent support for JSON and JSONB data types. You can store, query, and manipulate JSON data efficiently.',
 'Bob Johnson',
 ARRAY['json', 'postgresql', 'nosql'],
 '2024-01-25'
);

-- Basic text search queries
SELECT 
    title,
    author,
    ts_rank(to_tsvector('english', title || ' ' || content), 
            plainto_tsquery('english', 'PostgreSQL database')) AS relevance
FROM blog_posts
WHERE to_tsvector('english', title || ' ' || content) @@ 
      plainto_tsquery('english', 'PostgreSQL database')
ORDER BY relevance DESC;
```

#### Advanced Text Search
```sql
-- Update tsvector column for better performance
UPDATE blog_posts 
SET tsvector_col = to_tsvector('english', title || ' ' || content);

-- Create index on tsvector column
CREATE INDEX idx_blog_posts_fts ON blog_posts USING GIN(tsvector_col);

-- Trigger to maintain tsvector automatically
CREATE OR REPLACE FUNCTION update_blog_tsvector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.tsvector_col := to_tsvector('english', NEW.title || ' ' || NEW.content);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tsvector_update_trigger
    BEFORE INSERT OR UPDATE ON blog_posts
    FOR EACH ROW
    EXECUTE FUNCTION update_blog_tsvector();

-- Complex search queries
SELECT 
    title,
    author,
    published_date,
    ts_rank_cd(tsvector_col, query) AS relevance,
    ts_headline('english', content, query, 'MaxWords=50') AS headline
FROM blog_posts, 
     websearch_to_tsquery('english', 'PostgreSQL AND (JSON OR database)') AS query
WHERE tsvector_col @@ query
ORDER BY relevance DESC;

-- Search with phrase queries
SELECT 
    title,
    author,
    ts_rank(tsvector_col, phraseto_tsquery('english', 'full text search')) AS relevance
FROM blog_posts
WHERE tsvector_col @@ phraseto_tsquery('english', 'full text search')
ORDER BY relevance DESC;

-- Search with prefix matching
SELECT 
    title,
    author,
    ts_rank(tsvector_col, to_tsquery('english', 'postgr:*')) AS relevance
FROM blog_posts
WHERE tsvector_col @@ to_tsquery('english', 'postgr:*')
ORDER BY relevance DESC;
```

### Custom Text Search Configurations

#### Creating Custom Dictionaries
```sql
-- Create custom text search configuration
CREATE TEXT SEARCH CONFIGURATION custom_english (COPY = english);

-- Create synonym dictionary
CREATE TEXT SEARCH DICTIONARY synonym_dict (
    TEMPLATE = synonym,
    SYNONYMS = 'synonyms'
);

-- Example synonyms file content (would be in data directory):
-- database db
-- postgresql postgres pgsql
-- javascript js

-- Add custom dictionary to configuration
ALTER TEXT SEARCH CONFIGURATION custom_english
    ALTER MAPPING FOR asciiword WITH synonym_dict, english_stem;

-- Use custom configuration
SELECT 
    title,
    to_tsvector('custom_english', content) AS custom_vector
FROM blog_posts;
```

---

## Spatial Data (PostGIS)

### Setting Up PostGIS

#### Installing and Enabling PostGIS
```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;

-- Verify installation
SELECT postgis_version();
```

#### Basic Spatial Operations
```sql
-- Create table with spatial data
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    location GEOMETRY(POINT, 4326), -- WGS84 coordinate system
    area GEOMETRY(POLYGON, 4326),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert spatial data
INSERT INTO locations (name, location) VALUES
('New York City', ST_GeomFromText('POINT(-74.0059 40.7128)', 4326)),
('Los Angeles', ST_GeomFromText('POINT(-118.2437 34.0522)', 4326)),
('Chicago', ST_GeomFromText('POINT(-87.6298 41.8781)', 4326)),
('Houston', ST_GeomFromText('POINT(-95.3698 29.7604)', 4326));

-- Basic spatial queries
SELECT 
    name,
    ST_AsText(location) AS coordinates,
    ST_X(location) AS longitude,
    ST_Y(location) AS latitude
FROM locations;

-- Distance calculations
SELECT 
    l1.name AS from_city,
    l2.name AS to_city,
    ST_Distance(
        ST_Transform(l1.location, 3857),  -- Transform to meters
        ST_Transform(l2.location, 3857)
    ) / 1000 AS distance_km
FROM locations l1
CROSS JOIN locations l2
WHERE l1.id != l2.id
ORDER BY distance_km;

-- Find locations within radius
SELECT 
    name,
    ST_Distance(
        ST_Transform(location, 3857),
        ST_Transform(ST_GeomFromText('POINT(-74.0059 40.7128)', 4326), 3857)
    ) / 1000 AS distance_from_nyc_km
FROM locations
WHERE ST_DWithin(
    ST_Transform(location, 3857),
    ST_Transform(ST_GeomFromText('POINT(-74.0059 40.7128)', 4326), 3857),
    500000  -- 500km in meters
);
```

### Advanced Spatial Analysis
```sql
-- Create more complex spatial data
CREATE TABLE regions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    boundary GEOMETRY(POLYGON, 4326)
);

-- Insert polygon data (simplified example)
INSERT INTO regions (name, boundary) VALUES
('Northeast', ST_GeomFromText('POLYGON((-80 40, -70 40, -70 45, -80 45, -80 40))', 4326)),
('Southwest', ST_GeomFromText('POLYGON((-125 30, -100 30, -100 40, -125 40, -125 30))', 4326));

-- Spatial joins and containment
SELECT 
    l.name AS location_name,
    r.name AS region_name,
    ST_Contains(r.boundary, l.location) AS is_contained
FROM locations l
CROSS JOIN regions r
WHERE ST_Contains(r.boundary, l.location);

-- Buffer operations
SELECT 
    name,
    ST_AsText(ST_Buffer(ST_Transform(location, 3857), 50000)) AS buffer_50km
FROM locations
WHERE name = 'New York City';

-- Spatial indexes for performance
CREATE INDEX idx_locations_geom ON locations USING GIST(location);
CREATE INDEX idx_regions_boundary ON regions USING GIST(boundary);
```

---

## Advanced Indexing

### Specialized Index Types

#### Partial Indexes
```sql
-- Create sample table for indexing examples
CREATE TABLE orders_advanced (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    total_amount DECIMAL(12,2),
    status VARCHAR(20),
    order_date TIMESTAMP DEFAULT NOW(),
    shipped_date TIMESTAMP,
    is_priority BOOLEAN DEFAULT FALSE
);

-- Partial index for active orders only
CREATE INDEX idx_orders_active_status ON orders_advanced (status)
WHERE status IN ('pending', 'processing', 'confirmed');

-- Partial index for recent orders
CREATE INDEX idx_orders_recent ON orders_advanced (order_date, customer_id)
WHERE order_date >= '2024-01-01';

-- Partial index for high-value orders
CREATE INDEX idx_orders_high_value ON orders_advanced (total_amount, customer_id)
WHERE total_amount > 1000;
```

#### Expression Indexes
```sql
-- Index on computed expressions
CREATE INDEX idx_orders_year_month ON orders_advanced 
(EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date));

-- Index on lower-case text for case-insensitive searches
CREATE INDEX idx_customers_email_lower ON customers (LOWER(email));

-- Index on JSON expression
CREATE INDEX idx_user_preferences_theme ON user_profiles 
((preferences->>'theme'));

-- Functional index for complex calculations
CREATE INDEX idx_orders_profit_margin ON orders_advanced 
((total_amount - (total_amount * 0.3)) / total_amount)
WHERE total_amount > 0;
```

#### Multi-Column and Covering Indexes
```sql
-- Multi-column index with column order considerations
CREATE INDEX idx_orders_customer_date ON orders_advanced (customer_id, order_date);
CREATE INDEX idx_orders_status_priority ON orders_advanced (status, is_priority, order_date);

-- Covering index (includes additional columns for index-only scans)
CREATE INDEX idx_orders_covering ON orders_advanced (customer_id, status) 
INCLUDE (total_amount, order_date);

-- Analyze index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan AS index_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;
```

### Index Monitoring and Optimization

#### Index Usage Analysis
```sql
-- Find unused indexes
SELECT 
    s.schemaname,
    s.relname AS tablename,
    s.indexrelname AS indexname,
    s.idx_scan,
    pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size
FROM pg_stat_user_indexes s
JOIN pg_index i ON s.indexrelid = i.indexrelid
WHERE s.idx_scan = 0 
  AND NOT i.indisunique
  AND NOT i.indisprimary
ORDER BY pg_relation_size(s.indexrelid) DESC;

-- Find duplicate indexes
WITH index_definitions AS (
    SELECT 
        schemaname,
        tablename,
        indexname,
        pg_get_indexdef(indexrelid) AS index_def,
        pg_relation_size(indexrelid) AS index_size
    FROM pg_stat_user_indexes
)
SELECT 
    id1.schemaname,
    id1.tablename,
    id1.indexname AS index1,
    id2.indexname AS index2,
    pg_size_pretty(id1.index_size) AS size1,
    pg_size_pretty(id2.index_size) AS size2
FROM index_definitions id1
JOIN index_definitions id2 ON id1.index_def = id2.index_def
    AND id1.indexname < id2.indexname;

-- Index bloat estimation
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    CASE 
        WHEN pg_relation_size(indexrelid) > 0 THEN
            ROUND(100.0 * pg_stat_get_tuples_inserted(indexrelid) / 
                  GREATEST(pg_stat_get_tuples_inserted(indexrelid) + 
                          pg_stat_get_tuples_updated(indexrelid) + 
                          pg_stat_get_tuples_deleted(indexrelid), 1), 2)
        ELSE 0
    END AS estimated_bloat_pct
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## Extensions and Plugins

### Popular PostgreSQL Extensions

#### Installing and Using Extensions
```sql
-- List available extensions
SELECT name, installed_version, comment 
FROM pg_available_extensions 
ORDER BY name;

-- Install useful extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "btree_gin";
CREATE EXTENSION IF NOT EXISTS "hstore";

-- Use UUID extension
SELECT uuid_generate_v4() AS new_uuid;

-- Create table with UUID primary key
CREATE TABLE uuid_example (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### Trigram Similarity Extension
```sql
-- Use pg_trgm for fuzzy text matching
SELECT 
    name,
    similarity(name, 'Johnn') AS similarity_score
FROM customers
WHERE similarity(name, 'Johnn') > 0.3
ORDER BY similarity_score DESC;

-- Create trigram index for fast similarity searches
CREATE INDEX idx_customers_name_trigram ON customers 
USING GIN (name gin_trgm_ops);

-- Advanced pattern matching
SELECT name 
FROM customers 
WHERE name % 'Jon';  -- Similar to 'Jon'
```

#### HStore Extension
```sql
-- Use hstore for key-value data
CREATE TABLE products_hstore (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    attributes HSTORE
);

INSERT INTO products_hstore (name, attributes) VALUES
('Laptop', 'color=>black, weight=>2.5kg, warranty=>2years'),
('Phone', 'color=>white, storage=>128GB, os=>Android');

-- Query hstore data
SELECT 
    name,
    attributes->'color' AS color,
    attributes->'weight' AS weight,
    (attributes ? 'warranty') AS has_warranty
FROM products_hstore;

-- Hstore operators
SELECT name 
FROM products_hstore 
WHERE attributes @> 'color=>black';

SELECT name 
FROM products_hstore 
WHERE attributes ? 'warranty';
```

---

## Hands-On Exercises

### Exercise 1: Advanced Analytics Dashboard

Create a comprehensive analytics system using window functions and CTEs.

```sql
-- Solution: Analytics dashboard for e-commerce

-- Create comprehensive sales data
CREATE TABLE daily_sales (
    id SERIAL PRIMARY KEY,
    sale_date DATE NOT NULL,
    product_category VARCHAR(50),
    region VARCHAR(50),
    salesperson VARCHAR(100),
    revenue DECIMAL(12,2),
    units_sold INTEGER,
    cost DECIMAL(12,2)
);

-- Insert sample data (simplified)
INSERT INTO daily_sales (sale_date, product_category, region, salesperson, revenue, units_sold, cost) VALUES
('2024-01-01', 'Electronics', 'North', 'John Smith', 15000, 50, 9000),
('2024-01-01', 'Clothing', 'South', 'Jane Doe', 8000, 80, 4800),
('2024-01-02', 'Electronics', 'North', 'John Smith', 18000, 60, 10800),
('2024-01-02', 'Books', 'East', 'Bob Johnson', 3000, 100, 1800);

-- Comprehensive analytics query
WITH daily_metrics AS (
    SELECT 
        sale_date,
        product_category,
        region,
        salesperson,
        revenue,
        cost,
        revenue - cost AS profit,
        units_sold,
        
        -- Running totals
        SUM(revenue) OVER (
            PARTITION BY product_category 
            ORDER BY sale_date
        ) AS category_running_revenue,
        
        -- Moving averages
        AVG(revenue) OVER (
            PARTITION BY region 
            ORDER BY sale_date 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS region_7day_avg,
        
        -- Ranking within time periods
        RANK() OVER (
            PARTITION BY DATE_TRUNC('month', sale_date), product_category
            ORDER BY revenue DESC
        ) AS monthly_category_rank,
        
        -- Percentage of total
        revenue / SUM(revenue) OVER (PARTITION BY sale_date) * 100 AS pct_of_daily_total,
        
        -- Previous period comparison
        LAG(revenue) OVER (
            PARTITION BY product_category, region
            ORDER BY sale_date
        ) AS prev_day_revenue
    FROM daily_sales
),
performance_metrics AS (
    SELECT 
        *,
        CASE 
            WHEN prev_day_revenue IS NOT NULL THEN
                (revenue - prev_day_revenue) / prev_day_revenue * 100
        END AS day_over_day_growth,
        
        -- Profit margin
        profit / revenue * 100 AS profit_margin,
        
        -- Performance tier
        CASE 
            WHEN monthly_category_rank <= 3 THEN 'Top Performer'
            WHEN monthly_category_rank <= 10 THEN 'Good Performer'
            ELSE 'Needs Improvement'
        END AS performance_tier
    FROM daily_metrics
)
SELECT 
    sale_date,
    product_category,
    region,
    salesperson,
    revenue,
    profit,
    profit_margin,
    performance_tier,
    pct_of_daily_total,
    day_over_day_growth,
    region_7day_avg
FROM performance_metrics
ORDER BY sale_date DESC, revenue DESC;
```

### Exercise 2: Document Search System

Build a sophisticated search system using full-text search and JSON.

```sql
-- Solution: Document management with search

CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    content TEXT NOT NULL,
    metadata JSONB,
    tags TEXT[],
    document_type VARCHAR(50),
    author_id INTEGER,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    search_vector TSVECTOR
);

-- Insert sample documents
INSERT INTO documents (title, content, metadata, tags, document_type, author_id) VALUES
('PostgreSQL Performance Guide',
 'This comprehensive guide covers PostgreSQL performance optimization techniques including indexing strategies, query optimization, and configuration tuning.',
 '{"difficulty": "advanced", "pages": 45, "format": "pdf", "language": "en"}',
 ARRAY['postgresql', 'performance', 'database', 'optimization'],
 'guide',
 1
),
('JSON Data Processing',
 'Learn how to effectively work with JSON data in PostgreSQL. This tutorial covers JSONB operations, indexing, and performance considerations.',
 '{"difficulty": "intermediate", "pages": 28, "format": "markdown", "language": "en"}',
 ARRAY['json', 'postgresql', 'tutorial', 'data'],
 'tutorial',
 2
);

-- Update search vectors
UPDATE documents 
SET search_vector = to_tsvector('english', 
    title || ' ' || content || ' ' || array_to_string(tags, ' ')
);

-- Create indexes
CREATE INDEX idx_documents_search ON documents USING GIN(search_vector);
CREATE INDEX idx_documents_metadata ON documents USING GIN(metadata);
CREATE INDEX idx_documents_tags ON documents USING GIN(tags);

-- Advanced search function
CREATE OR REPLACE FUNCTION search_documents(
    search_text TEXT,
    document_types TEXT[] DEFAULT NULL,
    min_difficulty TEXT DEFAULT NULL,
    languages TEXT[] DEFAULT NULL
) RETURNS TABLE(
    doc_id INTEGER,
    title VARCHAR(300),
    document_type VARCHAR(50),
    relevance REAL,
    headline TEXT,
    metadata JSONB
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        d.id,
        d.title,
        d.document_type,
        ts_rank_cd(d.search_vector, websearch_to_tsquery('english', search_text)) AS relevance,
        ts_headline('english', d.content, websearch_to_tsquery('english', search_text), 
                   'MaxWords=30, MinWords=10') AS headline,
        d.metadata
    FROM documents d
    WHERE d.search_vector @@ websearch_to_tsquery('english', search_text)
      AND (document_types IS NULL OR d.document_type = ANY(document_types))
      AND (min_difficulty IS NULL OR 
           (d.metadata->>'difficulty' IN ('beginner', 'intermediate', 'advanced') AND
            CASE d.metadata->>'difficulty'
                WHEN 'beginner' THEN 1
                WHEN 'intermediate' THEN 2
                WHEN 'advanced' THEN 3
            END >= CASE min_difficulty
                WHEN 'beginner' THEN 1
                WHEN 'intermediate' THEN 2
                WHEN 'advanced' THEN 3
            END))
      AND (languages IS NULL OR d.metadata->>'language' = ANY(languages))
    ORDER BY relevance DESC;
END;
$$ LANGUAGE plpgsql;

-- Use the search function
SELECT * FROM search_documents('PostgreSQL performance optimization');
SELECT * FROM search_documents('JSON', ARRAY['tutorial'], 'intermediate');
```

---

## Best Practices

### Performance Best Practices
```sql
-- Monitor query performance
CREATE OR REPLACE FUNCTION analyze_slow_queries()
RETURNS TABLE(
    query TEXT,
    calls BIGINT,
    total_time DOUBLE PRECISION,
    avg_time DOUBLE PRECISION,
    rows BIGINT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        pg_stat_statements.query,
        pg_stat_statements.calls,
        pg_stat_statements.total_exec_time,
        pg_stat_statements.mean_exec_time,
        pg_stat_statements.rows
    FROM pg_stat_statements
    WHERE pg_stat_statements.mean_exec_time > 100 -- queries taking more than 100ms
    ORDER BY pg_stat_statements.mean_exec_time DESC
    LIMIT 20;
END;
$$ LANGUAGE plpgsql;
```

### JSON Best Practices
```sql
-- Efficient JSON querying patterns
-- Use JSONB for better performance
-- Index frequently queried JSON paths
-- Avoid deep nesting when possible

-- Good: Specific path index
CREATE INDEX idx_user_theme ON user_profiles ((preferences->>'theme'));

-- Good: GIN index for containment queries
CREATE INDEX idx_user_preferences_gin ON user_profiles USING GIN (preferences);

-- Avoid: Scanning entire JSON documents without indexes
-- SELECT * FROM user_profiles WHERE preferences::text LIKE '%dark%';

-- Better: Use proper JSON operators
-- SELECT * FROM user_profiles WHERE preferences @> '{"theme": "dark"}';
```

---

## Next Steps

### Module Completion Checklist
- [ ] Master window functions for analytics
- [ ] Use CTEs for complex queries
- [ ] Handle JSON/JSONB data effectively
- [ ] Create custom data types
- [ ] Implement full-text search
- [ ] Understand spatial data basics
- [ ] Optimize with advanced indexing

### Immediate Next Steps
1. **Practice** with real-world datasets
2. **Experiment** with different PostgreSQL extensions
3. **Optimize** existing queries using advanced features
4. **Move to Module 10**: [Transactions and Concurrency](10-transactions-concurrency.md)

---

## Module Summary

✅ **What You've Learned:**
- Advanced SQL features like window functions and CTEs
- JSON/JSONB handling and optimization
- Custom data types and domain creation
- Full-text search implementation
- Spatial data basics with PostGIS
- Advanced indexing strategies and monitoring

✅ **Skills Acquired:**
- Write sophisticated analytical queries
- Design flexible data models with JSON
- Implement search functionality
- Create custom types for specific needs
- Optimize database performance with advanced indexes
- Use PostgreSQL extensions effectively

✅ **Ready for Next Module:**
You're now ready to dive into database transactions and concurrency in [Module 10: Transactions and Concurrency](10-transactions-concurrency.md).

---

## Quick Reference

### Window Function Syntax
```sql
function() OVER (
    [PARTITION BY column] 
    [ORDER BY column] 
    [ROWS/RANGE BETWEEN ... AND ...]
)
```

### JSON Operators
```sql
-> : Get JSON object field
->> : Get JSON object field as text
#> : Get JSON object at path
#>> : Get JSON object at path as text
@> : Contains
? : Key exists
?& : All keys exist
?| : Any key exists
```

### Common Extensions
```sql
CREATE EXTENSION uuid-ossp;    -- UUID generation
CREATE EXTENSION pg_trgm;      -- Trigram similarity
CREATE EXTENSION hstore;       -- Key-value pairs
CREATE EXTENSION postgis;      -- Spatial data
CREATE EXTENSION btree_gin;    -- GIN indexes on btree types
```

---
*Continue your PostgreSQL journey with [Transactions and Concurrency](10-transactions-concurrency.md) →*
