# Functions and Procedures

## 🎯 Learning Objectives
By the end of this module, you will:
- Use built-in PostgreSQL functions effectively
- Create custom functions in SQL and PL/pgSQL
- Understand stored procedures and their use cases
- Implement triggers for automated database actions
- Apply functions for data validation and business logic

## 📚 Table of Contents
1. [Built-in Functions](#built-in-functions)
2. [Custom SQL Functions](#custom-sql-functions)
3. [PL/pgSQL Functions](#plpgsql-functions)
4. [Stored Procedures](#stored-procedures)
5. [Triggers](#triggers)
6. [Function Security](#function-security)
7. [Hands-On Exercises](#hands-on-exercises)
8. [Best Practices](#best-practices)
9. [Next Steps](#next-steps)

---

## Built-in Functions

### String Functions
```sql
-- Text manipulation
SELECT 
    'PostgreSQL' AS original,
    LENGTH('PostgreSQL') AS length,
    UPPER('PostgreSQL') AS uppercase,
    LOWER('PostgreSQL') AS lowercase,
    INITCAP('postgresql database') AS title_case,
    REVERSE('PostgreSQL') AS reversed;

-- String extraction and manipulation
SELECT 
    SUBSTRING('PostgreSQL' FROM 1 FOR 8) AS substring,
    LEFT('PostgreSQL', 8) AS left_chars,
    RIGHT('PostgreSQL', 3) AS right_chars,
    POSITION('SQL' IN 'PostgreSQL') AS position,
    REPLACE('PostgreSQL', 'SQL', 'Database') AS replaced;

-- String concatenation and formatting
SELECT 
    CONCAT('Hello', ' ', 'World') AS concatenated,
    'Hello' || ' ' || 'World' AS pipe_concatenated,
    FORMAT('Hello %s, today is %s', 'John', CURRENT_DATE) AS formatted,
    LPAD('123', 8, '0') AS left_padded,
    RPAD('123', 8, '0') AS right_padded,
    TRIM('  Hello World  ') AS trimmed;

-- Pattern matching
SELECT 
    'john.doe@example.com' ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$' AS valid_email,
    REGEXP_REPLACE('Phone: (555) 123-4567', '\D', '', 'g') AS digits_only,
    STRING_TO_ARRAY('apple,banana,cherry', ',') AS array_from_string;
```

### Numeric Functions
```sql
-- Basic math functions
SELECT 
    ABS(-42.5) AS absolute_value,
    CEIL(42.3) AS ceiling,
    FLOOR(42.7) AS floor,
    ROUND(42.567, 2) AS rounded,
    TRUNC(42.567, 1) AS truncated,
    MOD(17, 5) AS modulo,
    POWER(2, 3) AS power,
    SQRT(16) AS square_root;

-- Statistical functions
WITH sample_data AS (
    SELECT unnest(ARRAY[10, 20, 30, 40, 50]) AS value
)
SELECT 
    AVG(value) AS average,
    STDDEV(value) AS standard_deviation,
    VARIANCE(value) AS variance,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY value) AS median,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY value) AS percentile_95
FROM sample_data;

-- Random and sequence functions
SELECT 
    RANDOM() AS random_0_to_1,
    RANDOM() * 100 AS random_0_to_100,
    FLOOR(RANDOM() * 100) + 1 AS random_1_to_100,
    GENERATE_SERIES(1, 5) AS series;
```

### Date and Time Functions
```sql
-- Current date and time
SELECT 
    NOW() AS current_timestamp,
    CURRENT_DATE AS current_date,
    CURRENT_TIME AS current_time,
    CURRENT_TIMESTAMP AS current_timestamp_std,
    LOCALTIMESTAMP AS local_timestamp,
    EXTRACT(EPOCH FROM NOW()) AS unix_timestamp;

-- Date arithmetic
SELECT 
    CURRENT_DATE + INTERVAL '7 days' AS next_week,
    CURRENT_DATE - INTERVAL '1 month' AS last_month,
    AGE(CURRENT_DATE, '1990-01-01') AS age_since_1990,
    DATE_TRUNC('month', CURRENT_DATE) AS start_of_month,
    DATE_TRUNC('week', CURRENT_DATE) AS start_of_week;

-- Date extraction
SELECT 
    EXTRACT(YEAR FROM CURRENT_DATE) AS year,
    EXTRACT(MONTH FROM CURRENT_DATE) AS month,
    EXTRACT(DAY FROM CURRENT_DATE) AS day,
    EXTRACT(DOW FROM CURRENT_DATE) AS day_of_week,
    EXTRACT(DOY FROM CURRENT_DATE) AS day_of_year,
    TO_CHAR(CURRENT_DATE, 'Day, DD Month YYYY') AS formatted_date;

-- Date validation and conversion
SELECT 
    TO_DATE('2024-12-25', 'YYYY-MM-DD') AS parsed_date,
    TO_TIMESTAMP('2024-12-25 14:30:00', 'YYYY-MM-DD HH24:MI:SS') AS parsed_timestamp,
    ISFINITE(DATE '2024-12-25') AS is_finite_date,
    DATE '2024-12-25' + TIME '14:30:00' AS combined_datetime;
```

### Array Functions
```sql
-- Array creation and manipulation
SELECT 
    ARRAY[1, 2, 3, 4, 5] AS integer_array,
    ARRAY['apple', 'banana', 'cherry'] AS text_array,
    ARRAY_LENGTH(ARRAY[1, 2, 3, 4, 5], 1) AS array_length,
    ARRAY_APPEND(ARRAY[1, 2, 3], 4) AS appended_array,
    ARRAY_PREPEND(0, ARRAY[1, 2, 3]) AS prepended_array;

-- Array operations
SELECT 
    ARRAY[1, 2, 3] || ARRAY[4, 5, 6] AS concatenated_arrays,
    ARRAY[1, 2, 3, 2, 4] && ARRAY[2, 5] AS arrays_overlap,
    ARRAY[1, 2, 3] @> ARRAY[2, 3] AS contains_subarray,
    ARRAY[2, 3] <@ ARRAY[1, 2, 3, 4] AS is_contained_in,
    ARRAY_POSITION(ARRAY['a', 'b', 'c', 'b'], 'b') AS first_position;

-- Array aggregation
WITH sample_data AS (
    SELECT 'Group A' AS group_name, 10 AS value
    UNION ALL SELECT 'Group A', 20
    UNION ALL SELECT 'Group B', 30
    UNION ALL SELECT 'Group B', 40
)
SELECT 
    group_name,
    ARRAY_AGG(value) AS values_array,
    ARRAY_AGG(value ORDER BY value DESC) AS sorted_values
FROM sample_data
GROUP BY group_name;
```

### JSON/JSONB Functions
```sql
-- Create sample JSONB data
CREATE TEMP TABLE user_profiles AS
SELECT * FROM (VALUES
    (1, '{"name": "John", "age": 30, "skills": ["Python", "SQL"], "active": true}'::jsonb),
    (2, '{"name": "Jane", "age": 25, "skills": ["Java", "React"], "active": true}'::jsonb),
    (3, '{"name": "Bob", "age": 35, "skills": ["PHP", "MySQL"], "active": false}'::jsonb)
) AS t(id, profile);

-- JSON extraction and manipulation
SELECT 
    id,
    profile->>'name' AS name,
    profile->>'age' AS age_text,
    (profile->>'age')::INTEGER AS age_integer,
    profile->'skills' AS skills_json,
    profile->>'active' AS active_text,
    (profile->>'active')::BOOLEAN AS active_boolean
FROM user_profiles;

-- JSON path operations
SELECT 
    id,
    profile #> '{skills,0}' AS first_skill,
    profile #>> '{skills,0}' AS first_skill_text,
    JSONB_ARRAY_LENGTH(profile->'skills') AS skills_count,
    profile ? 'email' AS has_email,
    profile ?& ARRAY['name', 'age'] AS has_name_and_age,
    profile ?| ARRAY['email', 'phone'] AS has_email_or_phone
FROM user_profiles;

-- JSON modification
SELECT 
    id,
    profile || '{"email": "user@example.com"}'::jsonb AS with_email,
    JSONB_SET(profile, '{age}', '31'::jsonb) AS updated_age,
    profile - 'active' AS without_active,
    profile #- '{skills,0}' AS without_first_skill
FROM user_profiles;
```

---

## Custom SQL Functions

### Simple SQL Functions
```sql
-- Function to calculate circle area
CREATE OR REPLACE FUNCTION circle_area(radius NUMERIC)
RETURNS NUMERIC AS $$
    SELECT PI() * radius * radius;
$$ LANGUAGE SQL;

-- Usage
SELECT circle_area(5) AS area;

-- Function with multiple parameters
CREATE OR REPLACE FUNCTION full_name(first_name TEXT, last_name TEXT)
RETURNS TEXT AS $$
    SELECT COALESCE(first_name, '') || ' ' || COALESCE(last_name, '');
$$ LANGUAGE SQL;

-- Usage
SELECT full_name('John', 'Doe') AS name;

-- Function returning a table
CREATE OR REPLACE FUNCTION get_recent_orders(days INTEGER DEFAULT 30)
RETURNS TABLE(
    order_id INTEGER,
    customer_name TEXT,
    order_date DATE,
    total_amount NUMERIC
) AS $$
    SELECT 
        o.id,
        c.name,
        o.order_date,
        o.total_amount
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    WHERE o.order_date >= CURRENT_DATE - days;
$$ LANGUAGE SQL;

-- Usage
SELECT * FROM get_recent_orders(7);
```

### Functions with Default Parameters
```sql
-- Function with default parameters
CREATE OR REPLACE FUNCTION calculate_discount(
    original_price NUMERIC,
    discount_percent NUMERIC DEFAULT 10,
    max_discount NUMERIC DEFAULT 100
)
RETURNS NUMERIC AS $$
    SELECT LEAST(original_price * discount_percent / 100, max_discount);
$$ LANGUAGE SQL;

-- Usage examples
SELECT 
    calculate_discount(100) AS default_discount,
    calculate_discount(100, 15) AS custom_discount,
    calculate_discount(100, 20, 15) AS capped_discount;
```

### Variadic Functions
```sql
-- Function that accepts variable number of arguments
CREATE OR REPLACE FUNCTION average_score(VARIADIC scores NUMERIC[])
RETURNS NUMERIC AS $$
    SELECT AVG(score) FROM unnest(scores) AS score;
$$ LANGUAGE SQL;

-- Usage
SELECT 
    average_score(85, 90, 78) AS avg1,
    average_score(92, 88, 90, 95, 87) AS avg2;
```

---

## PL/pgSQL Functions

### Basic PL/pgSQL Structure
```sql
-- Simple PL/pgSQL function
CREATE OR REPLACE FUNCTION calculate_tax(amount NUMERIC, rate NUMERIC DEFAULT 0.08)
RETURNS NUMERIC AS $$
DECLARE
    tax_amount NUMERIC;
BEGIN
    -- Calculate tax
    tax_amount := amount * rate;
    
    -- Return the result
    RETURN tax_amount;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT calculate_tax(100, 0.1) AS tax;
```

### Control Structures
```sql
-- Function with conditional logic
CREATE OR REPLACE FUNCTION get_discount_rate(customer_type TEXT, order_amount NUMERIC)
RETURNS NUMERIC AS $$
DECLARE
    discount_rate NUMERIC := 0;
BEGIN
    -- Determine discount based on customer type and order amount
    IF customer_type = 'VIP' THEN
        discount_rate := 0.15;
    ELSIF customer_type = 'Premium' THEN
        discount_rate := 0.10;
    ELSIF customer_type = 'Regular' THEN
        discount_rate := 0.05;
    ELSE
        discount_rate := 0;
    END IF;
    
    -- Additional discount for large orders
    IF order_amount > 1000 THEN
        discount_rate := discount_rate + 0.05;
    END IF;
    
    -- Cap discount at 25%
    IF discount_rate > 0.25 THEN
        discount_rate := 0.25;
    END IF;
    
    RETURN discount_rate;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT 
    get_discount_rate('VIP', 500) AS vip_small,
    get_discount_rate('VIP', 1500) AS vip_large,
    get_discount_rate('Regular', 2000) AS regular_large;
```

### Loops and Iteration
```sql
-- Function with loops
CREATE OR REPLACE FUNCTION fibonacci(n INTEGER)
RETURNS INTEGER AS $$
DECLARE
    a INTEGER := 0;
    b INTEGER := 1;
    temp INTEGER;
    i INTEGER;
BEGIN
    IF n <= 0 THEN
        RETURN 0;
    ELSIF n = 1 THEN
        RETURN 1;
    END IF;
    
    FOR i IN 2..n LOOP
        temp := a + b;
        a := b;
        b := temp;
    END LOOP;
    
    RETURN b;
END;
$$ LANGUAGE plpgsql;

-- Generate Fibonacci sequence
SELECT i, fibonacci(i) FROM generate_series(0, 10) AS i;
```

### Exception Handling
```sql
-- Function with exception handling
CREATE OR REPLACE FUNCTION safe_divide(numerator NUMERIC, denominator NUMERIC)
RETURNS NUMERIC AS $$
DECLARE
    result NUMERIC;
BEGIN
    -- Attempt division
    BEGIN
        result := numerator / denominator;
        RETURN result;
    EXCEPTION
        WHEN division_by_zero THEN
            RAISE NOTICE 'Division by zero attempted, returning NULL';
            RETURN NULL;
        WHEN OTHERS THEN
            RAISE NOTICE 'Unexpected error: %', SQLERRM;
            RETURN NULL;
    END;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT 
    safe_divide(10, 2) AS normal,
    safe_divide(10, 0) AS division_by_zero;
```

### Working with Records and Arrays
```sql
-- Function returning composite type
CREATE TYPE customer_summary AS (
    customer_id INTEGER,
    customer_name TEXT,
    total_orders INTEGER,
    total_spent NUMERIC,
    last_order_date DATE
);

CREATE OR REPLACE FUNCTION get_customer_summary(cust_id INTEGER)
RETURNS customer_summary AS $$
DECLARE
    result customer_summary;
BEGIN
    SELECT 
        c.id,
        c.name,
        COUNT(o.id),
        COALESCE(SUM(o.total_amount), 0),
        MAX(o.order_date)
    INTO result
    FROM customers c
    LEFT JOIN orders o ON c.id = o.customer_id
    WHERE c.id = cust_id
    GROUP BY c.id, c.name;
    
    RETURN result;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM get_customer_summary(1);
```

### Dynamic SQL
```sql
-- Function with dynamic SQL
CREATE OR REPLACE FUNCTION get_table_stats(table_name TEXT)
RETURNS TABLE(
    total_rows BIGINT,
    table_size TEXT,
    last_updated TIMESTAMP
) AS $$
DECLARE
    sql_query TEXT;
BEGIN
    -- Build dynamic query
    sql_query := FORMAT('
        SELECT 
            COUNT(*) as total_rows,
            pg_size_pretty(pg_total_relation_size(%L)) as table_size,
            GREATEST(
                COALESCE(last_autoanalyze, ''1970-01-01''::timestamp),
                COALESCE(last_analyze, ''1970-01-01''::timestamp)
            ) as last_updated
        FROM %I
        CROSS JOIN pg_stat_user_tables 
        WHERE schemaname = ''public'' AND tablename = %L',
        table_name, table_name, table_name
    );
    
    -- Execute dynamic query
    RETURN QUERY EXECUTE sql_query;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM get_table_stats('customers');
```

---

## Stored Procedures

### Basic Procedures
```sql
-- Simple procedure
CREATE OR REPLACE PROCEDURE update_customer_stats()
LANGUAGE plpgsql AS $$
BEGIN
    -- Update customer statistics
    UPDATE customers 
    SET last_order_date = (
        SELECT MAX(order_date) 
        FROM orders 
        WHERE customer_id = customers.id
    );
    
    -- Log the operation
    INSERT INTO operation_log (operation, timestamp)
    VALUES ('update_customer_stats', NOW());
    
    -- Commit the transaction
    COMMIT;
END;
$$;

-- Call procedure
CALL update_customer_stats();
```

### Procedures with Parameters
```sql
-- Procedure with input and output parameters
CREATE OR REPLACE PROCEDURE process_order(
    IN customer_id INTEGER,
    IN product_ids INTEGER[],
    IN quantities INTEGER[],
    OUT order_id INTEGER,
    OUT total_amount NUMERIC
)
LANGUAGE plpgsql AS $$
DECLARE
    product_id INTEGER;
    quantity INTEGER;
    price NUMERIC;
    i INTEGER;
BEGIN
    -- Create new order
    INSERT INTO orders (customer_id, order_date, status, total_amount)
    VALUES (customer_id, CURRENT_DATE, 'pending', 0)
    RETURNING id INTO order_id;
    
    total_amount := 0;
    
    -- Add order items
    FOR i IN 1..array_length(product_ids, 1) LOOP
        product_id := product_ids[i];
        quantity := quantities[i];
        
        -- Get product price
        SELECT p.price INTO price FROM products p WHERE p.id = product_id;
        
        -- Insert order item
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        VALUES (order_id, product_id, quantity, price);
        
        -- Update total
        total_amount := total_amount + (quantity * price);
    END LOOP;
    
    -- Update order total
    UPDATE orders SET total_amount = total_amount WHERE id = order_id;
    
    COMMIT;
END;
$$;

-- Call procedure with output parameters
DO $$
DECLARE
    new_order_id INTEGER;
    order_total NUMERIC;
BEGIN
    CALL process_order(
        1,                          -- customer_id
        ARRAY[1, 2, 3],            -- product_ids
        ARRAY[2, 1, 3],            -- quantities
        new_order_id,              -- output: order_id
        order_total                -- output: total_amount
    );
    
    RAISE NOTICE 'Created order % with total %', new_order_id, order_total;
END;
$$;
```

---

## Triggers

### Before Triggers
```sql
-- Create audit table
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    operation TEXT,
    old_values JSONB,
    new_values JSONB,
    user_name TEXT,
    timestamp TIMESTAMP DEFAULT NOW()
);

-- Before trigger function for validation
CREATE OR REPLACE FUNCTION validate_employee_data()
RETURNS TRIGGER AS $$
BEGIN
    -- Validate email format
    IF NEW.email IS NOT NULL AND NEW.email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        RAISE EXCEPTION 'Invalid email format: %', NEW.email;
    END IF;
    
    -- Validate salary range
    IF NEW.salary IS NOT NULL AND (NEW.salary < 0 OR NEW.salary > 1000000) THEN
        RAISE EXCEPTION 'Salary must be between 0 and 1,000,000';
    END IF;
    
    -- Auto-generate employee code if not provided
    IF NEW.employee_code IS NULL THEN
        NEW.employee_code := 'EMP' || LPAD(NEW.id::TEXT, 6, '0');
    END IF;
    
    -- Set updated timestamp
    NEW.updated_at := NOW();
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER employee_before_insert_update
    BEFORE INSERT OR UPDATE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION validate_employee_data();
```

### After Triggers
```sql
-- After trigger function for auditing
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, new_values, user_name)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(NEW), user);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_values, new_values, user_name)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD), to_jsonb(NEW), user);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_values, user_name)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD), user);
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Create audit triggers
CREATE TRIGGER employees_audit
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION audit_changes();

CREATE TRIGGER orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION audit_changes();
```

### Statement-Level Triggers
```sql
-- Statement-level trigger for logging
CREATE OR REPLACE FUNCTION log_table_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO operation_log (operation, table_name, timestamp, user_name)
    VALUES (TG_OP, TG_TABLE_NAME, NOW(), user);
    
    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Create statement-level trigger
CREATE TRIGGER products_statement_log
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH STATEMENT
    EXECUTE FUNCTION log_table_changes();
```

### Conditional Triggers
```sql
-- Trigger that only fires for specific conditions
CREATE OR REPLACE FUNCTION notify_large_order()
RETURNS TRIGGER AS $$
BEGIN
    -- Only trigger for orders over $1000
    IF NEW.total_amount > 1000 THEN
        INSERT INTO notifications (message, created_at)
        VALUES (
            FORMAT('Large order created: Order #%s for $%s', NEW.id, NEW.total_amount),
            NOW()
        );
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create conditional trigger
CREATE TRIGGER large_order_notification
    AFTER INSERT OR UPDATE OF total_amount ON orders
    FOR EACH ROW
    WHEN (NEW.total_amount > 1000)
    EXECUTE FUNCTION notify_large_order();
```

---

## Function Security

### Function Privileges
```sql
-- Create function with specific security
CREATE OR REPLACE FUNCTION sensitive_data_access(user_id INTEGER)
RETURNS TABLE(data TEXT)
SECURITY DEFINER  -- Run with creator's privileges
SET search_path = public, pg_temp  -- Set secure search path
AS $$
BEGIN
    -- Check if current user has access
    IF NOT has_table_privilege(current_user, 'sensitive_table', 'SELECT') THEN
        RAISE EXCEPTION 'Access denied';
    END IF;
    
    RETURN QUERY
    SELECT sensitive_column FROM sensitive_table WHERE id = user_id;
END;
$$ LANGUAGE plpgsql;

-- Grant specific access
GRANT EXECUTE ON FUNCTION sensitive_data_access(INTEGER) TO app_user;
```

### Input Validation
```sql
-- Function with comprehensive input validation
CREATE OR REPLACE FUNCTION create_user_safe(
    username TEXT,
    email TEXT,
    age INTEGER
)
RETURNS INTEGER AS $$
DECLARE
    new_user_id INTEGER;
BEGIN
    -- Validate inputs
    IF username IS NULL OR LENGTH(TRIM(username)) = 0 THEN
        RAISE EXCEPTION 'Username cannot be empty';
    END IF;
    
    IF LENGTH(username) > 50 THEN
        RAISE EXCEPTION 'Username too long (max 50 characters)';
    END IF;
    
    IF email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        RAISE EXCEPTION 'Invalid email format';
    END IF;
    
    IF age < 13 OR age > 120 THEN
        RAISE EXCEPTION 'Age must be between 13 and 120';
    END IF;
    
    -- Check for existing username/email
    IF EXISTS (SELECT 1 FROM users WHERE username = create_user_safe.username) THEN
        RAISE EXCEPTION 'Username already exists';
    END IF;
    
    IF EXISTS (SELECT 1 FROM users WHERE email = create_user_safe.email) THEN
        RAISE EXCEPTION 'Email already exists';
    END IF;
    
    -- Create user
    INSERT INTO users (username, email, age, created_at)
    VALUES (username, email, age, NOW())
    RETURNING id INTO new_user_id;
    
    RETURN new_user_id;
END;
$$ LANGUAGE plpgsql;
```

---

## Hands-On Exercises

### Exercise 1: Data Validation Functions

Create a comprehensive validation system for a user registration system.

```sql
-- Solution:

-- 1. Create validation functions
CREATE OR REPLACE FUNCTION validate_email(email TEXT)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$';
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION validate_phone(phone TEXT)
RETURNS BOOLEAN AS $$
BEGIN
    -- Remove all non-digits
    phone := REGEXP_REPLACE(phone, '\D', '', 'g');
    -- Check if 10-15 digits
    RETURN LENGTH(phone) BETWEEN 10 AND 15;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION validate_password_strength(password TEXT)
RETURNS TEXT AS $$
DECLARE
    issues TEXT[] := ARRAY[]::TEXT[];
BEGIN
    IF LENGTH(password) < 8 THEN
        issues := array_append(issues, 'Password must be at least 8 characters');
    END IF;
    
    IF password !~ '[A-Z]' THEN
        issues := array_append(issues, 'Password must contain uppercase letter');
    END IF;
    
    IF password !~ '[a-z]' THEN
        issues := array_append(issues, 'Password must contain lowercase letter');
    END IF;
    
    IF password !~ '[0-9]' THEN
        issues := array_append(issues, 'Password must contain a number');
    END IF;
    
    IF password !~ '[^A-Za-z0-9]' THEN
        issues := array_append(issues, 'Password must contain special character');
    END IF;
    
    IF array_length(issues, 1) = 0 THEN
        RETURN 'VALID';
    ELSE
        RETURN array_to_string(issues, '; ');
    END IF;
END;
$$ LANGUAGE plpgsql;

-- 2. User registration function
CREATE OR REPLACE FUNCTION register_user(
    username TEXT,
    email TEXT,
    phone TEXT,
    password TEXT
)
RETURNS TABLE(success BOOLEAN, message TEXT, user_id INTEGER) AS $$
DECLARE
    new_user_id INTEGER;
    password_validation TEXT;
BEGIN
    -- Validate inputs
    IF NOT validate_email(email) THEN
        RETURN QUERY SELECT FALSE, 'Invalid email format', NULL::INTEGER;
        RETURN;
    END IF;
    
    IF NOT validate_phone(phone) THEN
        RETURN QUERY SELECT FALSE, 'Invalid phone number', NULL::INTEGER;
        RETURN;
    END IF;
    
    password_validation := validate_password_strength(password);
    IF password_validation != 'VALID' THEN
        RETURN QUERY SELECT FALSE, password_validation, NULL::INTEGER;
        RETURN;
    END IF;
    
    -- Check for duplicates
    IF EXISTS (SELECT 1 FROM users WHERE email = register_user.email) THEN
        RETURN QUERY SELECT FALSE, 'Email already registered', NULL::INTEGER;
        RETURN;
    END IF;
    
    -- Create user
    INSERT INTO users (username, email, phone, password_hash, created_at)
    VALUES (username, email, phone, crypt(password, gen_salt('bf')), NOW())
    RETURNING id INTO new_user_id;
    
    RETURN QUERY SELECT TRUE, 'User registered successfully', new_user_id;
END;
$$ LANGUAGE plpgsql;

-- Test the registration function
SELECT * FROM register_user('testuser', 'test@example.com', '555-123-4567', 'SecurePass123!');
```

### Exercise 2: Business Logic with Triggers

Create an inventory management system with automated stock updates and notifications.

```sql
-- Solution:

-- 1. Create inventory tables
CREATE TABLE inventory (
    product_id INTEGER PRIMARY KEY,
    current_stock INTEGER NOT NULL DEFAULT 0,
    min_stock_level INTEGER NOT NULL DEFAULT 10,
    max_stock_level INTEGER NOT NULL DEFAULT 100,
    last_updated TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stock_movements (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id),
    movement_type VARCHAR(20) CHECK (movement_type IN ('IN', 'OUT', 'ADJUSTMENT')),
    quantity INTEGER NOT NULL,
    reference_id INTEGER, -- Could be order_id, receipt_id, etc.
    reason TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stock_alerts (
    id SERIAL PRIMARY KEY,
    product_id INTEGER,
    alert_type VARCHAR(20),
    message TEXT,
    is_resolved BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 2. Stock movement processing function
CREATE OR REPLACE FUNCTION process_stock_movement(
    p_product_id INTEGER,
    p_movement_type TEXT,
    p_quantity INTEGER,
    p_reference_id INTEGER DEFAULT NULL,
    p_reason TEXT DEFAULT NULL
)
RETURNS BOOLEAN AS $$
DECLARE
    current_stock INTEGER;
    new_stock INTEGER;
    min_level INTEGER;
    max_level INTEGER;
BEGIN
    -- Get current stock levels
    SELECT current_stock, min_stock_level, max_stock_level
    INTO current_stock, min_level, max_level
    FROM inventory
    WHERE product_id = p_product_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Product % not found in inventory', p_product_id;
    END IF;
    
    -- Calculate new stock level
    CASE p_movement_type
        WHEN 'IN' THEN
            new_stock := current_stock + p_quantity;
        WHEN 'OUT' THEN
            new_stock := current_stock - p_quantity;
            -- Check if sufficient stock
            IF new_stock < 0 THEN
                RAISE EXCEPTION 'Insufficient stock. Current: %, Required: %', current_stock, p_quantity;
            END IF;
        WHEN 'ADJUSTMENT' THEN
            new_stock := p_quantity; -- Direct adjustment to specific amount
        ELSE
            RAISE EXCEPTION 'Invalid movement type: %', p_movement_type;
    END CASE;
    
    -- Record the movement
    INSERT INTO stock_movements (product_id, movement_type, quantity, reference_id, reason)
    VALUES (p_product_id, p_movement_type, p_quantity, p_reference_id, p_reason);
    
    -- Update inventory
    UPDATE inventory 
    SET current_stock = new_stock, last_updated = NOW()
    WHERE product_id = p_product_id;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- 3. Trigger function for stock alerts
CREATE OR REPLACE FUNCTION check_stock_levels()
RETURNS TRIGGER AS $$
BEGIN
    -- Check for low stock
    IF NEW.current_stock <= (SELECT min_stock_level FROM inventory WHERE product_id = NEW.product_id) THEN
        INSERT INTO stock_alerts (product_id, alert_type, message)
        VALUES (
            NEW.product_id, 
            'LOW_STOCK', 
            FORMAT('Product %s is low on stock: %s units remaining (minimum: %s)', 
                   NEW.product_id, NEW.current_stock, 
                   (SELECT min_stock_level FROM inventory WHERE product_id = NEW.product_id))
        );
    END IF;
    
    -- Check for overstock
    IF NEW.current_stock >= (SELECT max_stock_level FROM inventory WHERE product_id = NEW.product_id) THEN
        INSERT INTO stock_alerts (product_id, alert_type, message)
        VALUES (
            NEW.product_id,
            'OVERSTOCK',
            FORMAT('Product %s is overstocked: %s units (maximum: %s)',
                   NEW.product_id, NEW.current_stock,
                   (SELECT max_stock_level FROM inventory WHERE product_id = NEW.product_id))
        );
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create the trigger
CREATE TRIGGER inventory_alert_trigger
    AFTER UPDATE OF current_stock ON inventory
    FOR EACH ROW
    EXECUTE FUNCTION check_stock_levels();

-- 4. Order processing with automatic stock updates
CREATE OR REPLACE FUNCTION process_order_items()
RETURNS TRIGGER AS $$
BEGIN
    -- Reduce stock for new order items
    PERFORM process_stock_movement(
        NEW.product_id,
        'OUT',
        NEW.quantity,
        NEW.order_id,
        'Order fulfillment'
    );
    
    RETURN NEW;
EXCEPTION
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Stock movement failed: %', SQLERRM;
END;
$$ LANGUAGE plpgsql;

-- Create trigger for order items
CREATE TRIGGER order_item_stock_trigger
    AFTER INSERT ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION process_order_items();
```

### Exercise 3: Reporting and Analytics Functions

Create a comprehensive reporting system with various business metrics.

```sql
-- Solution:

-- 1. Customer analytics function
CREATE OR REPLACE FUNCTION get_customer_analytics(
    start_date DATE DEFAULT CURRENT_DATE - INTERVAL '1 year',
    end_date DATE DEFAULT CURRENT_DATE
)
RETURNS TABLE(
    customer_id INTEGER,
    customer_name TEXT,
    total_orders INTEGER,
    total_spent NUMERIC,
    avg_order_value NUMERIC,
    last_order_date DATE,
    customer_tier TEXT
) AS $$
BEGIN
    RETURN QUERY
    WITH customer_stats AS (
        SELECT 
            c.id,
            c.name,
            COUNT(o.id)::INTEGER AS order_count,
            COALESCE(SUM(o.total_amount), 0) AS total_amount,
            COALESCE(AVG(o.total_amount), 0) AS avg_amount,
            MAX(o.order_date) AS last_order
        FROM customers c
        LEFT JOIN orders o ON c.id = o.customer_id 
            AND o.order_date BETWEEN start_date AND end_date
        GROUP BY c.id, c.name
    )
    SELECT 
        cs.id,
        cs.name,
        cs.order_count,
        cs.total_amount,
        cs.avg_amount,
        cs.last_order,
        CASE 
            WHEN cs.total_amount > 5000 THEN 'VIP'
            WHEN cs.total_amount > 2000 THEN 'Premium'
            WHEN cs.total_amount > 500 THEN 'Regular'
            ELSE 'New'
        END AS tier
    FROM customer_stats cs
    ORDER BY cs.total_amount DESC;
END;
$$ LANGUAGE plpgsql;

-- 2. Sales trend analysis function
CREATE OR REPLACE FUNCTION analyze_sales_trends(
    period_type TEXT DEFAULT 'month', -- 'day', 'week', 'month', 'quarter'
    periods_count INTEGER DEFAULT 12
)
RETURNS TABLE(
    period_start DATE,
    period_end DATE,
    total_sales NUMERIC,
    order_count INTEGER,
    avg_order_value NUMERIC,
    growth_rate NUMERIC
) AS $$
DECLARE
    date_trunc_format TEXT;
    interval_step INTERVAL;
BEGIN
    -- Set format based on period type
    CASE period_type
        WHEN 'day' THEN 
            date_trunc_format := 'day';
            interval_step := '1 day'::INTERVAL;
        WHEN 'week' THEN 
            date_trunc_format := 'week';
            interval_step := '1 week'::INTERVAL;
        WHEN 'month' THEN 
            date_trunc_format := 'month';
            interval_step := '1 month'::INTERVAL;
        WHEN 'quarter' THEN 
            date_trunc_format := 'quarter';
            interval_step := '3 months'::INTERVAL;
        ELSE
            RAISE EXCEPTION 'Invalid period_type. Use: day, week, month, quarter';
    END CASE;
    
    RETURN QUERY
    WITH period_sales AS (
        SELECT 
            DATE_TRUNC(date_trunc_format, o.order_date) AS period_start,
            SUM(o.total_amount) AS sales,
            COUNT(o.id)::INTEGER AS orders,
            AVG(o.total_amount) AS avg_value
        FROM orders o
        WHERE o.order_date >= CURRENT_DATE - (interval_step * periods_count)
        GROUP BY DATE_TRUNC(date_trunc_format, o.order_date)
        ORDER BY period_start
    ),
    period_with_growth AS (
        SELECT 
            ps.period_start,
            ps.period_start + interval_step - INTERVAL '1 day' AS period_end,
            ps.sales,
            ps.orders,
            ps.avg_value,
            LAG(ps.sales) OVER (ORDER BY ps.period_start) AS prev_sales
        FROM period_sales ps
    )
    SELECT 
        pwg.period_start,
        pwg.period_end,
        pwg.sales,
        pwg.orders,
        pwg.avg_value,
        CASE 
            WHEN pwg.prev_sales IS NULL OR pwg.prev_sales = 0 THEN NULL
            ELSE ROUND(((pwg.sales - pwg.prev_sales) / pwg.prev_sales * 100), 2)
        END AS growth_rate
    FROM period_with_growth pwg;
END;
$$ LANGUAGE plpgsql;

-- 3. Product performance analysis
CREATE OR REPLACE FUNCTION analyze_product_performance(
    top_n INTEGER DEFAULT 10,
    analysis_period INTERVAL DEFAULT INTERVAL '3 months'
)
RETURNS TABLE(
    product_id INTEGER,
    product_name TEXT,
    category TEXT,
    units_sold INTEGER,
    total_revenue NUMERIC,
    avg_price NUMERIC,
    profit_margin NUMERIC,
    performance_rank INTEGER
) AS $$
BEGIN
    RETURN QUERY
    WITH product_stats AS (
        SELECT 
            p.id,
            p.name,
            p.category,
            SUM(oi.quantity)::INTEGER AS sold_quantity,
            SUM(oi.quantity * oi.unit_price) AS revenue,
            AVG(oi.unit_price) AS average_price,
            COALESCE(AVG(oi.unit_price) - AVG(p.cost), 0) AS margin
        FROM products p
        LEFT JOIN order_items oi ON p.id = oi.product_id
        LEFT JOIN orders o ON oi.order_id = o.id
        WHERE o.order_date >= CURRENT_DATE - analysis_period
        GROUP BY p.id, p.name, p.category
    )
    SELECT 
        ps.id,
        ps.name,
        ps.category,
        COALESCE(ps.sold_quantity, 0),
        COALESCE(ps.revenue, 0),
        COALESCE(ps.average_price, 0),
        COALESCE(ps.margin, 0),
        ROW_NUMBER() OVER (ORDER BY ps.revenue DESC)::INTEGER
    FROM product_stats ps
    ORDER BY ps.revenue DESC
    LIMIT top_n;
END;
$$ LANGUAGE plpgsql;

-- Usage examples
SELECT * FROM get_customer_analytics();
SELECT * FROM analyze_sales_trends('month', 6);
SELECT * FROM analyze_product_performance(5);
```

---

## Best Practices

### Function Design
1. **Use appropriate return types** - TABLE, SETOF, or scalar
2. **Validate all inputs** - Never trust user input
3. **Handle exceptions gracefully** - Use proper error handling
4. **Keep functions focused** - One function, one responsibility
5. **Use meaningful names** - Clear, descriptive function names

### Performance Considerations
```sql
-- Use STABLE or IMMUTABLE when appropriate
CREATE OR REPLACE FUNCTION calculate_circle_area(radius NUMERIC)
RETURNS NUMERIC
IMMUTABLE  -- Result only depends on input
AS $$
    SELECT PI() * radius * radius;
$$ LANGUAGE SQL;

-- Avoid unnecessary work in loops
CREATE OR REPLACE FUNCTION efficient_function()
RETURNS VOID AS $$
DECLARE
    constant_value NUMERIC;
BEGIN
    -- Calculate once outside loop
    constant_value := expensive_calculation();
    
    FOR i IN 1..1000 LOOP
        -- Use pre-calculated value
        INSERT INTO table VALUES (i, constant_value);
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### Security Guidelines
```sql
-- Use SECURITY DEFINER carefully
CREATE OR REPLACE FUNCTION secure_function()
RETURNS TEXT
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
BEGIN
    -- Always validate permissions
    IF NOT has_table_privilege(current_user, 'sensitive_table', 'SELECT') THEN
        RAISE EXCEPTION 'Access denied';
    END IF;
    
    -- Function logic here
    RETURN 'result';
END;
$$ LANGUAGE plpgsql;
```

---

## Next Steps

### Module Completion Checklist
- [ ] Master built-in PostgreSQL functions
- [ ] Create custom SQL and PL/pgSQL functions
- [ ] Implement stored procedures effectively
- [ ] Use triggers for automated database actions
- [ ] Apply security best practices for functions

### Immediate Next Steps
1. **Practice** creating functions for real-world scenarios
2. **Experiment** with different function types and triggers
3. **Review** performance implications of your functions
4. **Move to Module 8**: [Data Modeling](08-data-modeling.md)

---

## Module Summary

✅ **What You've Learned:**
- Comprehensive coverage of built-in PostgreSQL functions
- Creating custom SQL and PL/pgSQL functions
- Implementing stored procedures and triggers
- Function security and best practices
- Real-world applications and use cases

✅ **Skills Acquired:**
- Write efficient custom functions
- Implement business logic in the database
- Create automated systems with triggers
- Validate and secure function implementations
- Design reusable and maintainable code

✅ **Ready for Next Module:**
You're now ready to learn advanced data modeling techniques in [Module 8: Data Modeling](08-data-modeling.md).

---

## Quick Reference

### Function Types
```sql
-- SQL Function
CREATE FUNCTION name(params) RETURNS type AS $$ SELECT ...; $$ LANGUAGE SQL;

-- PL/pgSQL Function
CREATE FUNCTION name(params) RETURNS type AS $$ DECLARE ... BEGIN ... END; $$ LANGUAGE plpgsql;

-- Table Function
CREATE FUNCTION name(params) RETURNS TABLE(cols) AS $$ ... $$ LANGUAGE plpgsql;

-- Procedure
CREATE PROCEDURE name(params) AS $$ ... $$ LANGUAGE plpgsql;
```

### Trigger Syntax
```sql
-- Trigger Function
CREATE FUNCTION trigger_func() RETURNS TRIGGER AS $$ ... $$ LANGUAGE plpgsql;

-- Trigger
CREATE TRIGGER name BEFORE/AFTER event ON table 
FOR EACH ROW/STATEMENT EXECUTE FUNCTION trigger_func();
```

### Common Patterns
```sql
-- Exception Handling
BEGIN
    -- risky operation
EXCEPTION
    WHEN condition THEN
        -- handle error
END;

-- Dynamic SQL
EXECUTE format('SELECT * FROM %I WHERE %I = %L', table_name, column_name, value);

-- Input Validation
IF input IS NULL OR LENGTH(TRIM(input)) = 0 THEN
    RAISE EXCEPTION 'Invalid input';
END IF;
```

---
*Continue your PostgreSQL journey with [Data Modeling](08-data-modeling.md) →*
