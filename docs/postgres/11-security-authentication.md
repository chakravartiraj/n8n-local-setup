# Security and Authentication

## 🎯 Learning Objectives
By the end of this module, you will:
- Implement robust authentication and authorization systems
- Configure role-based access control (RBAC)
- Apply Row Level Security (RLS) for fine-grained access control
- Secure database connections and communications
- Handle sensitive data with encryption and masking
- Monitor and audit database security

## 📚 Table of Contents
1. [Authentication Methods](#authentication-methods)
2. [Role-Based Access Control](#role-based-access-control)
3. [Row Level Security](#row-level-security)
4. [Connection Security](#connection-security)
5. [Data Encryption](#data-encryption)
6. [Auditing and Monitoring](#auditing-and-monitoring)
7. [Security Best Practices](#security-best-practices)
8. [Hands-On Exercises](#hands-on-exercises)
9. [Next Steps](#next-steps)

---

## Authentication Methods

### Basic Authentication Setup

#### User Management
```sql
-- Create users with different authentication methods
CREATE USER app_user WITH PASSWORD 'secure_password_123!';
CREATE USER read_only_user WITH PASSWORD 'readonly_pass_456!';
CREATE USER api_service WITH PASSWORD 'api_service_789!';

-- Create roles for group management
CREATE ROLE developers;
CREATE ROLE analysts;
CREATE ROLE administrators;

-- Grant roles to users
GRANT developers TO app_user;
GRANT analysts TO read_only_user;
GRANT administrators TO app_user; -- app_user can be both developer and admin

-- Set password policies
ALTER USER app_user VALID UNTIL '2025-12-31';
ALTER USER api_service CONNECTION LIMIT 10; -- Limit concurrent connections

-- Create user with specific privileges
CREATE USER backup_user WITH 
    PASSWORD 'backup_secure_pass!'
    NOSUPERUSER
    NOCREATEDB
    NOCREATEROLE
    CONNECTION LIMIT 5;

-- View user information
SELECT 
    usename AS username,
    usesuper AS is_superuser,
    usecreatedb AS can_create_db,
    usecreaterole AS can_create_role,
    useconnlimit AS connection_limit,
    valuntil AS password_expires
FROM pg_user
ORDER BY usename;
```

#### Authentication Configuration (pg_hba.conf)
```sql
-- View current authentication settings
SELECT * FROM pg_hba_file_rules ORDER BY line_number;

-- Common pg_hba.conf configurations:
-- # TYPE  DATABASE        USER            ADDRESS                 METHOD
-- local   all             postgres                                peer
-- local   all             all                                     md5
-- host    all             all             127.0.0.1/32            md5
-- host    all             all             ::1/128                 md5
-- host    mydb            app_user        192.168.1.0/24          md5
-- host    mydb            +developers     10.0.0.0/8              scram-sha-256

-- Reload configuration after changes
SELECT pg_reload_conf();
```

### Advanced Authentication

#### SCRAM-SHA-256 Authentication
```sql
-- Enable SCRAM-SHA-256 (more secure than MD5)
-- In postgresql.conf: password_encryption = scram-sha-256

-- Create user with SCRAM password
CREATE USER secure_user WITH PASSWORD 'very_secure_password_2024!';

-- Convert existing user to SCRAM
\password app_user  -- Enter new password when prompted

-- Verify password encryption method
SELECT 
    usename,
    CASE 
        WHEN passwd ~ '^SCRAM-SHA-256' THEN 'SCRAM-SHA-256'
        WHEN passwd ~ '^md5' THEN 'MD5'
        ELSE 'Unknown'
    END AS password_method
FROM pg_shadow
WHERE usename IN ('app_user', 'secure_user');
```

#### Certificate-Based Authentication
```sql
-- Create user for certificate authentication
CREATE USER cert_user;

-- Configure in pg_hba.conf:
-- hostssl mydb cert_user 192.168.1.0/24 cert clientcert=verify-full

-- User must present valid client certificate matching username
```

---

## Role-Based Access Control

### Hierarchical Role System

#### Creating Role Hierarchy
```sql
-- Create base roles
CREATE ROLE base_user NOLOGIN;
CREATE ROLE power_user NOLOGIN;
CREATE ROLE admin_user NOLOGIN;

-- Create specific functional roles
CREATE ROLE sales_team NOLOGIN;
CREATE ROLE finance_team NOLOGIN;
CREATE ROLE hr_team NOLOGIN;
CREATE ROLE report_viewers NOLOGIN;
CREATE ROLE data_analysts NOLOGIN;

-- Create application-specific roles
CREATE ROLE web_app NOLOGIN;
CREATE ROLE api_service NOLOGIN;
CREATE ROLE etl_process NOLOGIN;

-- Set up role inheritance
GRANT base_user TO power_user;
GRANT power_user TO admin_user;

-- Grant functional roles to users
GRANT sales_team TO base_user;
GRANT finance_team TO power_user;
GRANT report_viewers TO base_user;
GRANT data_analysts TO power_user;

-- Create actual login users and assign roles
CREATE USER alice_sales WITH PASSWORD 'alice_pass!' IN ROLE sales_team;
CREATE USER bob_finance WITH PASSWORD 'bob_pass!' IN ROLE finance_team, data_analysts;
CREATE USER charlie_admin WITH PASSWORD 'charlie_pass!' IN ROLE admin_user;

-- View role memberships
SELECT 
    r.rolname AS role_name,
    m.rolname AS member_name,
    grantor.rolname AS granted_by,
    admin_option
FROM pg_roles r
JOIN pg_auth_members am ON r.oid = am.roleid
JOIN pg_roles m ON am.member = m.oid
JOIN pg_roles grantor ON am.grantor = grantor.oid
ORDER BY r.rolname, m.rolname;
```

### Permission Management

#### Database and Schema Permissions
```sql
-- Create application schemas
CREATE SCHEMA sales;
CREATE SCHEMA finance;
CREATE SCHEMA hr;
CREATE SCHEMA reporting;

-- Grant schema-level permissions
GRANT USAGE ON SCHEMA sales TO sales_team;
GRANT USAGE ON SCHEMA finance TO finance_team;
GRANT USAGE ON SCHEMA hr TO hr_team;
GRANT USAGE ON SCHEMA reporting TO report_viewers;

-- Grant creation privileges
GRANT CREATE ON SCHEMA sales TO sales_team;
GRANT CREATE ON SCHEMA finance TO finance_team;

-- Revoke public schema access
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT USAGE ON SCHEMA public TO base_user;

-- Set default privileges for future objects
ALTER DEFAULT PRIVILEGES IN SCHEMA sales 
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO sales_team;

ALTER DEFAULT PRIVILEGES IN SCHEMA finance 
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO finance_team;

-- Read-only access to reporting
ALTER DEFAULT PRIVILEGES IN SCHEMA reporting 
    GRANT SELECT ON TABLES TO report_viewers;
```

#### Table-Level Permissions
```sql
-- Create sample tables
CREATE TABLE sales.customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(100),
    phone VARCHAR(20),
    address TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE sales.orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES sales.customers(id),
    order_date DATE DEFAULT CURRENT_DATE,
    total_amount DECIMAL(12,2),
    status VARCHAR(20) DEFAULT 'pending',
    created_by VARCHAR(100) DEFAULT CURRENT_USER
);

CREATE TABLE finance.invoices (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES sales.orders(id),
    invoice_number VARCHAR(50) UNIQUE,
    amount DECIMAL(12,2),
    due_date DATE,
    paid_date DATE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Grant specific table permissions
GRANT SELECT, INSERT, UPDATE ON sales.customers TO sales_team;
GRANT SELECT, INSERT, UPDATE, DELETE ON sales.orders TO sales_team;

-- Finance team gets read access to sales data
GRANT SELECT ON sales.customers TO finance_team;
GRANT SELECT ON sales.orders TO finance_team;
GRANT ALL ON finance.invoices TO finance_team;

-- Reporting team gets read-only access to all
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO report_viewers;
GRANT SELECT ON ALL TABLES IN SCHEMA finance TO report_viewers;

-- Column-level permissions
GRANT SELECT (id, name, email, created_at) ON sales.customers TO report_viewers;
-- Exclude sensitive columns like phone, address from general reporting
```

#### Function and Procedure Security
```sql
-- Create security definer function (runs with creator's privileges)
CREATE OR REPLACE FUNCTION sales.get_customer_summary(customer_id INTEGER)
RETURNS TABLE(
    customer_name VARCHAR(200),
    total_orders BIGINT,
    total_spent DECIMAL(12,2)
) 
SECURITY DEFINER  -- Runs with creator's permissions
LANGUAGE SQL
AS $$
    SELECT 
        c.name,
        COUNT(o.id),
        COALESCE(SUM(o.total_amount), 0)
    FROM sales.customers c
    LEFT JOIN sales.orders o ON c.id = o.customer_id
    WHERE c.id = customer_id
    GROUP BY c.name;
$$;

-- Grant execute permission
GRANT EXECUTE ON FUNCTION sales.get_customer_summary(INTEGER) TO report_viewers;

-- Create security invoker function (runs with caller's privileges)
CREATE OR REPLACE FUNCTION sales.add_customer(
    customer_name VARCHAR(200),
    customer_email VARCHAR(100)
)
RETURNS INTEGER
SECURITY INVOKER  -- Runs with caller's permissions
LANGUAGE plpgsql
AS $$
DECLARE
    new_customer_id INTEGER;
BEGIN
    INSERT INTO sales.customers (name, email)
    VALUES (customer_name, customer_email)
    RETURNING id INTO new_customer_id;
    
    RETURN new_customer_id;
END;
$$;

-- Only users with INSERT privilege on customers table can use this function
GRANT EXECUTE ON FUNCTION sales.add_customer(VARCHAR, VARCHAR) TO sales_team;
```

---

## Row Level Security

### Basic RLS Implementation

#### Enabling RLS
```sql
-- Enable RLS on tables
ALTER TABLE sales.customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE sales.orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE finance.invoices ENABLE ROW LEVEL SECURITY;

-- Add required columns for RLS
ALTER TABLE sales.customers ADD COLUMN owner_id VARCHAR(100) DEFAULT CURRENT_USER;
ALTER TABLE sales.orders ADD COLUMN owner_id VARCHAR(100) DEFAULT CURRENT_USER;
ALTER TABLE finance.invoices ADD COLUMN department VARCHAR(50) DEFAULT 'finance';
```

#### Basic RLS Policies
```sql
-- Policy: Users can only see their own customer records
CREATE POLICY customer_owner_policy ON sales.customers
    FOR ALL
    TO base_user
    USING (owner_id = CURRENT_USER);

-- Policy: Sales team can see all customers
CREATE POLICY customer_sales_policy ON sales.customers
    FOR ALL
    TO sales_team
    USING (true);

-- Policy: Users can only see their own orders
CREATE POLICY order_owner_policy ON sales.orders
    FOR ALL
    TO base_user
    USING (owner_id = CURRENT_USER);

-- Policy: Finance team can see all orders
CREATE POLICY order_finance_policy ON sales.orders
    FOR ALL
    TO finance_team
    USING (true);

-- Policy: Department-based access to invoices
CREATE POLICY invoice_department_policy ON finance.invoices
    FOR ALL
    TO base_user
    USING (department = CURRENT_USER);
```

### Advanced RLS Patterns

#### Multi-Tenant RLS
```sql
-- Add tenant isolation
CREATE TABLE tenants (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    subdomain VARCHAR(50) UNIQUE,
    status VARCHAR(20) DEFAULT 'active'
);

CREATE TABLE user_tenants (
    user_id VARCHAR(100) NOT NULL,
    tenant_id INTEGER REFERENCES tenants(id),
    role VARCHAR(50) NOT NULL,
    PRIMARY KEY (user_id, tenant_id)
);

-- Add tenant_id to existing tables
ALTER TABLE sales.customers ADD COLUMN tenant_id INTEGER;
ALTER TABLE sales.orders ADD COLUMN tenant_id INTEGER;

-- Update existing data with tenant information
UPDATE sales.customers SET tenant_id = 1; -- Default tenant
UPDATE sales.orders SET tenant_id = 1;

-- Create RLS policies for multi-tenancy
CREATE POLICY customer_tenant_policy ON sales.customers
    FOR ALL
    USING (tenant_id IN (
        SELECT ut.tenant_id 
        FROM user_tenants ut 
        WHERE ut.user_id = CURRENT_USER
    ));

CREATE POLICY order_tenant_policy ON sales.orders
    FOR ALL
    USING (tenant_id IN (
        SELECT ut.tenant_id 
        FROM user_tenants ut 
        WHERE ut.user_id = CURRENT_USER
    ));

-- Function to set current tenant context
CREATE OR REPLACE FUNCTION set_current_tenant(tenant_id INTEGER)
RETURNS VOID AS $$
BEGIN
    PERFORM set_config('app.current_tenant_id', tenant_id::TEXT, true);
END;
$$ LANGUAGE plpgsql;

-- Enhanced policy using session variable
CREATE POLICY customer_session_tenant_policy ON sales.customers
    FOR ALL
    USING (
        tenant_id = COALESCE(
            current_setting('app.current_tenant_id', true)::INTEGER,
            (SELECT ut.tenant_id FROM user_tenants ut WHERE ut.user_id = CURRENT_USER LIMIT 1)
        )
    );
```

#### Time-Based RLS
```sql
-- Add temporal columns
ALTER TABLE sales.orders ADD COLUMN valid_from DATE DEFAULT CURRENT_DATE;
ALTER TABLE sales.orders ADD COLUMN valid_to DATE;

-- Policy: Only show current valid records
CREATE POLICY order_temporal_policy ON sales.orders
    FOR SELECT
    USING (
        valid_from <= CURRENT_DATE 
        AND (valid_to IS NULL OR valid_to >= CURRENT_DATE)
    );

-- Policy: Historical data access for analysts
CREATE POLICY order_historical_policy ON sales.orders
    FOR SELECT
    TO data_analysts
    USING (true); -- Analysts can see all historical data
```

#### Hierarchical RLS
```sql
-- Create organizational hierarchy
CREATE TABLE org_hierarchy (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(100) NOT NULL,
    manager_id VARCHAR(100),
    department VARCHAR(50),
    level INTEGER DEFAULT 1
);

-- Recursive function to get subordinates
CREATE OR REPLACE FUNCTION get_subordinates(manager_user_id VARCHAR(100))
RETURNS TABLE(subordinate_id VARCHAR(100)) AS $$
WITH RECURSIVE subordinates AS (
    -- Direct reports
    SELECT user_id FROM org_hierarchy WHERE manager_id = manager_user_id
    
    UNION ALL
    
    -- Indirect reports
    SELECT oh.user_id
    FROM org_hierarchy oh
    JOIN subordinates s ON oh.manager_id = s.subordinate_id
)
SELECT subordinate_id FROM subordinates;
$$ LANGUAGE sql;

-- Policy: Managers can see their subordinates' data
CREATE POLICY customer_hierarchy_policy ON sales.customers
    FOR ALL
    USING (
        owner_id = CURRENT_USER 
        OR owner_id IN (SELECT subordinate_id FROM get_subordinates(CURRENT_USER))
        OR pg_has_role(CURRENT_USER, 'admin_user', 'MEMBER')
    );
```

---

## Connection Security

### SSL/TLS Configuration

#### SSL Setup
```sql
-- Check SSL status
SELECT 
    datname,
    usename,
    ssl,
    client_addr,
    backend_start
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE pid != pg_backend_pid();

-- Force SSL connections in pg_hba.conf:
-- hostssl all all 0.0.0.0/0 md5

-- Connection parameters for clients:
-- host=localhost port=5432 dbname=mydb user=myuser sslmode=require
-- sslmode options: disable, allow, prefer, require, verify-ca, verify-full
```

#### Connection Limits and Monitoring
```sql
-- Set connection limits
ALTER SYSTEM SET max_connections = 200;
ALTER SYSTEM SET superuser_reserved_connections = 3;

-- Per-database connection limits
ALTER DATABASE mydb CONNECTION LIMIT 100;

-- Monitor connections
CREATE VIEW connection_monitoring AS
SELECT 
    datname AS database,
    usename AS username,
    client_addr,
    client_port,
    application_name,
    state,
    state_change,
    query_start,
    NOW() - backend_start AS connection_age,
    NOW() - query_start AS query_age
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
ORDER BY backend_start;

-- Function to kill idle connections
CREATE OR REPLACE FUNCTION kill_idle_connections(
    max_idle_minutes INTEGER DEFAULT 30
) RETURNS TABLE(
    killed_pid INTEGER,
    username TEXT,
    idle_duration INTERVAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        psa.pid,
        psa.usename::TEXT,
        NOW() - psa.state_change
    FROM pg_stat_activity psa
    WHERE psa.state = 'idle'
      AND psa.pid != pg_backend_pid()
      AND NOW() - psa.state_change > (max_idle_minutes * INTERVAL '1 minute')
      AND pg_terminate_backend(psa.pid);
END;
$$ LANGUAGE plpgsql;
```

---

## Data Encryption

### Column-Level Encryption

#### Using pgcrypto Extension
```sql
-- Enable encryption extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Create table with encrypted columns
CREATE TABLE sensitive_data (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    phone_encrypted BYTEA,  -- Encrypted phone number
    ssn_encrypted BYTEA,    -- Encrypted SSN
    notes_encrypted BYTEA,  -- Encrypted notes
    created_at TIMESTAMP DEFAULT NOW()
);

-- Functions for encryption/decryption
CREATE OR REPLACE FUNCTION encrypt_sensitive_data(
    plaintext TEXT,
    encryption_key TEXT DEFAULT 'my_secret_key_2024!'
) RETURNS BYTEA AS $$
BEGIN
    RETURN pgp_sym_encrypt(plaintext, encryption_key);
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION decrypt_sensitive_data(
    ciphertext BYTEA,
    encryption_key TEXT DEFAULT 'my_secret_key_2024!'
) RETURNS TEXT AS $$
BEGIN
    RETURN pgp_sym_decrypt(ciphertext, encryption_key);
EXCEPTION
    WHEN OTHERS THEN
        RETURN '[DECRYPTION_ERROR]';
END;
$$ LANGUAGE plpgsql;

-- Insert encrypted data
INSERT INTO sensitive_data (username, email, phone_encrypted, ssn_encrypted, notes_encrypted)
VALUES (
    'john_doe',
    'john@example.com',
    encrypt_sensitive_data('555-123-4567'),
    encrypt_sensitive_data('123-45-6789'),
    encrypt_sensitive_data('VIP customer - handle with care')
);

-- Query with decryption
SELECT 
    id,
    username,
    email,
    decrypt_sensitive_data(phone_encrypted) AS phone,
    decrypt_sensitive_data(notes_encrypted) AS notes,
    '[REDACTED]' AS ssn_display  -- Don't show SSN in regular queries
FROM sensitive_data;

-- Secure view that only shows decrypted data to authorized users
CREATE VIEW sensitive_data_secure AS
SELECT 
    id,
    username,
    email,
    CASE 
        WHEN pg_has_role(CURRENT_USER, 'admin_user', 'MEMBER') OR
             pg_has_role(CURRENT_USER, 'hr_team', 'MEMBER') THEN
            decrypt_sensitive_data(phone_encrypted)
        ELSE '[REDACTED]'
    END AS phone,
    CASE 
        WHEN pg_has_role(CURRENT_USER, 'admin_user', 'MEMBER') THEN
            decrypt_sensitive_data(ssn_encrypted)
        ELSE '[REDACTED]'
    END AS ssn,
    CASE 
        WHEN pg_has_role(CURRENT_USER, 'admin_user', 'MEMBER') OR
             pg_has_role(CURRENT_USER, 'sales_team', 'MEMBER') THEN
            decrypt_sensitive_data(notes_encrypted)
        ELSE '[REDACTED]'
    END AS notes,
    created_at
FROM sensitive_data;

-- Grant access to secure view
GRANT SELECT ON sensitive_data_secure TO base_user;
```

### Data Masking

#### Dynamic Data Masking
```sql
-- Create masking functions
CREATE OR REPLACE FUNCTION mask_email(email TEXT)
RETURNS TEXT AS $$
DECLARE
    at_position INTEGER;
    username_part TEXT;
    domain_part TEXT;
BEGIN
    IF email IS NULL OR email = '' THEN
        RETURN email;
    END IF;
    
    at_position := POSITION('@' IN email);
    IF at_position = 0 THEN
        RETURN email; -- Invalid email, return as-is
    END IF;
    
    username_part := SUBSTRING(email FROM 1 FOR at_position - 1);
    domain_part := SUBSTRING(email FROM at_position);
    
    IF LENGTH(username_part) <= 2 THEN
        RETURN REPEAT('*', LENGTH(username_part)) || domain_part;
    ELSE
        RETURN LEFT(username_part, 1) || REPEAT('*', LENGTH(username_part) - 2) || 
               RIGHT(username_part, 1) || domain_part;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION mask_phone(phone TEXT)
RETURNS TEXT AS $$
BEGIN
    IF phone IS NULL OR LENGTH(phone) < 4 THEN
        RETURN phone;
    END IF;
    
    RETURN '***-***-' || RIGHT(phone, 4);
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION mask_credit_card(cc TEXT)
RETURNS TEXT AS $$
BEGIN
    IF cc IS NULL OR LENGTH(cc) < 4 THEN
        RETURN cc;
    END IF;
    
    RETURN '**** **** **** ' || RIGHT(cc, 4);
END;
$$ LANGUAGE plpgsql;

-- Create masked view
CREATE VIEW customers_masked AS
SELECT 
    id,
    name,
    CASE 
        WHEN pg_has_role(CURRENT_USER, 'admin_user', 'MEMBER') OR
             pg_has_role(CURRENT_USER, 'sales_team', 'MEMBER') THEN email
        ELSE mask_email(email)
    END AS email,
    CASE 
        WHEN pg_has_role(CURRENT_USER, 'admin_user', 'MEMBER') THEN phone
        ELSE mask_phone(phone)
    END AS phone,
    created_at
FROM sales.customers;

-- Test masking
SELECT * FROM customers_masked;
```

---

## Auditing and Monitoring

### Audit Logging

#### Built-in Logging
```sql
-- Configure logging (in postgresql.conf)
-- log_statement = 'all'  -- Log all statements
-- log_connections = on
-- log_disconnections = on
-- log_duration = on
-- log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

-- Check current log settings
SELECT name, setting, unit, context 
FROM pg_settings 
WHERE name LIKE 'log_%' 
ORDER BY name;
```

#### Custom Audit Trail
```sql
-- Create audit log table
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    operation VARCHAR(10) NOT NULL, -- INSERT, UPDATE, DELETE
    user_name VARCHAR(100) NOT NULL,
    timestamp TIMESTAMP DEFAULT NOW(),
    old_values JSONB,
    new_values JSONB,
    changed_columns TEXT[],
    client_ip INET,
    application_name TEXT
);

-- Generic audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
DECLARE
    old_values JSONB := NULL;
    new_values JSONB := NULL;
    changed_cols TEXT[] := ARRAY[]::TEXT[];
    col_name TEXT;
BEGIN
    -- Get client IP and application name
    PERFORM set_config('audit.client_ip', 
        COALESCE(inet_client_addr()::TEXT, 'localhost'), true);
    PERFORM set_config('audit.app_name', 
        COALESCE(current_setting('application_name', true), 'unknown'), true);
    
    IF TG_OP = 'DELETE' THEN
        old_values := row_to_json(OLD);
        INSERT INTO audit_log (
            table_name, operation, user_name, old_values,
            client_ip, application_name
        ) VALUES (
            TG_TABLE_NAME, TG_OP, USER,
            old_values,
            current_setting('audit.client_ip', true)::INET,
            current_setting('audit.app_name', true)
        );
        RETURN OLD;
        
    ELSIF TG_OP = 'INSERT' THEN
        new_values := row_to_json(NEW);
        INSERT INTO audit_log (
            table_name, operation, user_name, new_values,
            client_ip, application_name
        ) VALUES (
            TG_TABLE_NAME, TG_OP, USER,
            new_values,
            current_setting('audit.client_ip', true)::INET,
            current_setting('audit.app_name', true)
        );
        RETURN NEW;
        
    ELSIF TG_OP = 'UPDATE' THEN
        old_values := row_to_json(OLD);
        new_values := row_to_json(NEW);
        
        -- Find changed columns
        FOR col_name IN 
            SELECT column_name 
            FROM information_schema.columns 
            WHERE table_name = TG_TABLE_NAME 
              AND table_schema = TG_TABLE_SCHEMA
        LOOP
            IF (old_values ->> col_name) IS DISTINCT FROM (new_values ->> col_name) THEN
                changed_cols := array_append(changed_cols, col_name);
            END IF;
        END LOOP;
        
        -- Only log if there are actual changes
        IF array_length(changed_cols, 1) > 0 THEN
            INSERT INTO audit_log (
                table_name, operation, user_name, old_values, new_values,
                changed_columns, client_ip, application_name
            ) VALUES (
                TG_TABLE_NAME, TG_OP, USER,
                old_values, new_values, changed_cols,
                current_setting('audit.client_ip', true)::INET,
                current_setting('audit.app_name', true)
            );
        END IF;
        RETURN NEW;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Apply audit trigger to tables
CREATE TRIGGER customers_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE ON sales.customers
    FOR EACH ROW EXECUTE FUNCTION audit_trigger();

CREATE TRIGGER orders_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE ON sales.orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger();

-- Test audit logging
INSERT INTO sales.customers (name, email, tenant_id) 
VALUES ('Test User', 'test@example.com', 1);

UPDATE sales.customers 
SET email = 'test.updated@example.com' 
WHERE name = 'Test User';

-- View audit log
SELECT 
    timestamp,
    table_name,
    operation,
    user_name,
    changed_columns,
    client_ip,
    application_name
FROM audit_log
ORDER BY timestamp DESC
LIMIT 10;
```

### Security Monitoring

#### Failed Login Attempts
```sql
-- Create failed login tracking table
CREATE TABLE failed_logins (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100),
    client_ip INET,
    attempted_at TIMESTAMP DEFAULT NOW(),
    failure_reason TEXT
);

-- Function to log failed logins (called from application)
CREATE OR REPLACE FUNCTION log_failed_login(
    attempted_username VARCHAR(100),
    client_address INET,
    reason TEXT
) RETURNS VOID AS $$
BEGIN
    INSERT INTO failed_logins (username, client_ip, failure_reason)
    VALUES (attempted_username, client_address, reason);
    
    -- Check for brute force attempts
    IF (SELECT COUNT(*) 
        FROM failed_logins 
        WHERE client_ip = client_address 
          AND attempted_at > NOW() - INTERVAL '15 minutes') > 5 THEN
        
        RAISE WARNING 'Potential brute force attack from %', client_address;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Monitor suspicious activity
CREATE VIEW security_alerts AS
SELECT 
    'Multiple failed logins' AS alert_type,
    client_ip::TEXT AS source,
    COUNT(*) AS event_count,
    MIN(attempted_at) AS first_attempt,
    MAX(attempted_at) AS last_attempt
FROM failed_logins
WHERE attempted_at > NOW() - INTERVAL '1 hour'
GROUP BY client_ip
HAVING COUNT(*) > 3

UNION ALL

SELECT 
    'Privilege escalation attempt' AS alert_type,
    user_name AS source,
    COUNT(*) AS event_count,
    MIN(timestamp) AS first_attempt,
    MAX(timestamp) AS last_attempt
FROM audit_log
WHERE timestamp > NOW() - INTERVAL '1 hour'
  AND (old_values ? 'role' OR new_values ? 'role')
GROUP BY user_name
HAVING COUNT(*) > 5;
```

---

## Security Best Practices

### Password Policies
```sql
-- Function to validate password strength
CREATE OR REPLACE FUNCTION validate_password(password TEXT)
RETURNS BOOLEAN AS $$
BEGIN
    -- Minimum length check
    IF LENGTH(password) < 12 THEN
        RAISE EXCEPTION 'Password must be at least 12 characters long';
    END IF;
    
    -- Complexity checks
    IF NOT (password ~ '[a-z]' AND 
            password ~ '[A-Z]' AND 
            password ~ '[0-9]' AND 
            password ~ '[^a-zA-Z0-9]') THEN
        RAISE EXCEPTION 'Password must contain lowercase, uppercase, number, and special character';
    END IF;
    
    -- Common password check (simplified)
    IF LOWER(password) = ANY(ARRAY[
        'password123!', 'admin123!', 'qwerty123!', 'welcome123!'
    ]) THEN
        RAISE EXCEPTION 'Password is too common';
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Function to create secure user
CREATE OR REPLACE FUNCTION create_secure_user(
    username VARCHAR(100),
    password TEXT,
    user_roles TEXT[] DEFAULT ARRAY['base_user']
) RETURNS VOID AS $$
DECLARE
    role_name TEXT;
BEGIN
    -- Validate password
    PERFORM validate_password(password);
    
    -- Create user
    EXECUTE format('CREATE USER %I WITH PASSWORD %L', username, password);
    
    -- Grant roles
    FOREACH role_name IN ARRAY user_roles LOOP
        EXECUTE format('GRANT %I TO %I', role_name, username);
    END LOOP;
    
    -- Set expiration
    EXECUTE format('ALTER USER %I VALID UNTIL %L', 
                   username, 
                   (CURRENT_DATE + INTERVAL '90 days')::DATE);
END;
$$ LANGUAGE plpgsql;
```

### Secure Configuration Checklist
```sql
-- Function to check security configuration
CREATE OR REPLACE FUNCTION security_configuration_check()
RETURNS TABLE(
    check_name TEXT,
    status TEXT,
    recommendation TEXT
) AS $$
BEGIN
    -- Check SSL enforcement
    RETURN QUERY
    SELECT 
        'SSL Configuration'::TEXT,
        CASE WHEN setting = 'on' THEN 'PASS' ELSE 'FAIL' END,
        'Ensure ssl = on in postgresql.conf'::TEXT
    FROM pg_settings WHERE name = 'ssl';
    
    -- Check password encryption
    RETURN QUERY
    SELECT 
        'Password Encryption'::TEXT,
        CASE WHEN setting = 'scram-sha-256' THEN 'PASS' ELSE 'WARN' END,
        'Use scram-sha-256 instead of md5'::TEXT
    FROM pg_settings WHERE name = 'password_encryption';
    
    -- Check log settings
    RETURN QUERY
    SELECT 
        'Connection Logging'::TEXT,
        CASE WHEN setting = 'on' THEN 'PASS' ELSE 'WARN' END,
        'Enable log_connections for security monitoring'::TEXT
    FROM pg_settings WHERE name = 'log_connections';
    
    -- Check for default passwords (simplified check)
    RETURN QUERY
    SELECT 
        'Default Passwords'::TEXT,
        CASE WHEN EXISTS(
            SELECT 1 FROM pg_shadow 
            WHERE usename = 'postgres' AND passwd IS NULL
        ) THEN 'FAIL' ELSE 'PASS' END,
        'Ensure postgres user has a strong password'::TEXT;
    
    -- Check for excessive permissions
    RETURN QUERY
    SELECT 
        'Public Schema Permissions'::TEXT,
        CASE WHEN has_schema_privilege('public', 'public', 'CREATE') 
             THEN 'WARN' ELSE 'PASS' END,
        'Revoke CREATE privilege from public schema'::TEXT;
END;
$$ LANGUAGE plpgsql;

-- Run security check
SELECT * FROM security_configuration_check();
```

---

## Hands-On Exercises

### Exercise 1: Multi-Tenant SaaS Security

Implement a complete security model for a multi-tenant SaaS application.

```sql
-- Solution: Complete multi-tenant security system

-- 1. Tenant and user management
CREATE TABLE security_tenants (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    subdomain VARCHAR(50) UNIQUE NOT NULL,
    encryption_key VARCHAR(255) NOT NULL, -- Tenant-specific encryption
    status VARCHAR(20) DEFAULT 'active',
    max_users INTEGER DEFAULT 100,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE security_users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    salt VARCHAR(100) NOT NULL,
    mfa_secret VARCHAR(100), -- For 2FA
    mfa_enabled BOOLEAN DEFAULT FALSE,
    last_login TIMESTAMP,
    failed_login_attempts INTEGER DEFAULT 0,
    locked_until TIMESTAMP,
    password_changed_at TIMESTAMP DEFAULT NOW(),
    must_change_password BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE tenant_memberships (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER REFERENCES security_tenants(id),
    user_id INTEGER REFERENCES security_users(id),
    role VARCHAR(50) NOT NULL,
    permissions JSONB DEFAULT '{}',
    is_active BOOLEAN DEFAULT TRUE,
    joined_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(tenant_id, user_id)
);

-- 2. Application data with tenant isolation
CREATE TABLE tenant_projects (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER REFERENCES security_tenants(id),
    name VARCHAR(200) NOT NULL,
    description TEXT,
    owner_id INTEGER REFERENCES security_users(id),
    is_public BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE project_data (
    id SERIAL PRIMARY KEY,
    project_id INTEGER REFERENCES tenant_projects(id),
    data_type VARCHAR(50),
    encrypted_content BYTEA, -- Encrypted with tenant key
    metadata JSONB,
    created_by INTEGER REFERENCES security_users(id),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE tenant_projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE project_data ENABLE ROW LEVEL SECURITY;

-- 3. Tenant-aware encryption functions
CREATE OR REPLACE FUNCTION get_tenant_key(tenant_id INTEGER)
RETURNS TEXT AS $$
DECLARE
    encryption_key TEXT;
BEGIN
    SELECT t.encryption_key INTO encryption_key
    FROM security_tenants t
    WHERE t.id = tenant_id;
    
    IF encryption_key IS NULL THEN
        RAISE EXCEPTION 'Tenant not found or invalid';
    END IF;
    
    RETURN encryption_key;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE OR REPLACE FUNCTION encrypt_tenant_data(
    plaintext TEXT,
    tenant_id INTEGER
) RETURNS BYTEA AS $$
BEGIN
    RETURN pgp_sym_encrypt(plaintext, get_tenant_key(tenant_id));
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION decrypt_tenant_data(
    ciphertext BYTEA,
    tenant_id INTEGER
) RETURNS TEXT AS $$
BEGIN
    RETURN pgp_sym_decrypt(ciphertext, get_tenant_key(tenant_id));
EXCEPTION
    WHEN OTHERS THEN
        RETURN '[DECRYPTION_ERROR]';
END;
$$ LANGUAGE plpgsql;

-- 4. Context functions
CREATE OR REPLACE FUNCTION get_current_user_id()
RETURNS INTEGER AS $$
BEGIN
    RETURN COALESCE(
        current_setting('app.current_user_id', true)::INTEGER,
        0
    );
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION get_current_tenant_id()
RETURNS INTEGER AS $$
BEGIN
    RETURN COALESCE(
        current_setting('app.current_tenant_id', true)::INTEGER,
        (SELECT tm.tenant_id 
         FROM tenant_memberships tm 
         WHERE tm.user_id = get_current_user_id() 
           AND tm.is_active = TRUE 
         LIMIT 1)
    );
END;
$$ LANGUAGE plpgsql;

-- 5. RLS Policies
CREATE POLICY project_tenant_isolation ON tenant_projects
    FOR ALL
    USING (
        tenant_id = get_current_tenant_id()
        AND (
            is_public = TRUE
            OR owner_id = get_current_user_id()
            OR EXISTS (
                SELECT 1 FROM tenant_memberships tm
                WHERE tm.tenant_id = tenant_projects.tenant_id
                  AND tm.user_id = get_current_user_id()
                  AND tm.is_active = TRUE
            )
        )
    );

CREATE POLICY project_data_access ON project_data
    FOR ALL
    USING (
        project_id IN (
            SELECT tp.id FROM tenant_projects tp
            WHERE tp.tenant_id = get_current_tenant_id()
        )
    );

-- 6. Authentication functions
CREATE OR REPLACE FUNCTION authenticate_user(
    username_param VARCHAR(100),
    password_param TEXT,
    client_ip INET DEFAULT NULL
) RETURNS TABLE(
    user_id INTEGER,
    success BOOLEAN,
    message TEXT,
    must_change_password BOOLEAN
) AS $$
DECLARE
    user_record RECORD;
    password_valid BOOLEAN := FALSE;
BEGIN
    -- Get user record
    SELECT * INTO user_record
    FROM security_users
    WHERE username = username_param;
    
    IF NOT FOUND THEN
        PERFORM log_failed_login(username_param, client_ip, 'User not found');
        RETURN QUERY SELECT 0, FALSE, 'Invalid credentials'::TEXT, FALSE;
        RETURN;
    END IF;
    
    -- Check if account is locked
    IF user_record.locked_until IS NOT NULL AND user_record.locked_until > NOW() THEN
        RETURN QUERY SELECT user_record.id, FALSE, 
            'Account locked until ' || user_record.locked_until::TEXT, FALSE;
        RETURN;
    END IF;
    
    -- Validate password (simplified - in production use proper hashing)
    password_valid := (user_record.password_hash = crypt(password_param, user_record.salt));
    
    IF password_valid THEN
        -- Reset failed attempts
        UPDATE security_users
        SET failed_login_attempts = 0,
            last_login = NOW(),
            locked_until = NULL
        WHERE id = user_record.id;
        
        RETURN QUERY SELECT 
            user_record.id, 
            TRUE, 
            'Login successful'::TEXT,
            user_record.must_change_password;
    ELSE
        -- Increment failed attempts
        UPDATE security_users
        SET failed_login_attempts = failed_login_attempts + 1,
            locked_until = CASE 
                WHEN failed_login_attempts + 1 >= 5 THEN NOW() + INTERVAL '15 minutes'
                ELSE NULL
            END
        WHERE id = user_record.id;
        
        PERFORM log_failed_login(username_param, client_ip, 'Invalid password');
        RETURN QUERY SELECT user_record.id, FALSE, 'Invalid credentials'::TEXT, FALSE;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- 7. Permission checking
CREATE OR REPLACE FUNCTION user_has_permission(
    user_id_param INTEGER,
    tenant_id_param INTEGER,
    permission_name TEXT
) RETURNS BOOLEAN AS $$
DECLARE
    user_permissions JSONB;
    has_perm BOOLEAN := FALSE;
BEGIN
    SELECT tm.permissions INTO user_permissions
    FROM tenant_memberships tm
    WHERE tm.user_id = user_id_param
      AND tm.tenant_id = tenant_id_param
      AND tm.is_active = TRUE;
    
    IF user_permissions IS NULL THEN
        RETURN FALSE;
    END IF;
    
    -- Check if user has the specific permission
    has_perm := COALESCE((user_permissions ->> permission_name)::BOOLEAN, FALSE);
    
    -- Check if user has admin role (has all permissions)
    IF NOT has_perm THEN
        has_perm := COALESCE((user_permissions ->> 'admin')::BOOLEAN, FALSE);
    END IF;
    
    RETURN has_perm;
END;
$$ LANGUAGE plpgsql;

-- Test the security system
INSERT INTO security_tenants (name, subdomain, encryption_key) VALUES
('Acme Corp', 'acme', 'acme_secret_key_2024!'),
('Beta Industries', 'beta', 'beta_secret_key_2024!');

INSERT INTO security_users (username, email, password_hash, salt) VALUES
('alice', 'alice@acme.com', crypt('password123!', gen_salt('bf')), gen_salt('bf')),
('bob', 'bob@beta.com', crypt('password456!', gen_salt('bf')), gen_salt('bf'));

INSERT INTO tenant_memberships (tenant_id, user_id, role, permissions) VALUES
(1, 1, 'admin', '{"admin": true, "read": true, "write": true}'),
(2, 2, 'user', '{"read": true, "write": false}');
```

---

## Next Steps

### Module Completion Checklist
- [ ] Implement authentication and authorization
- [ ] Configure role-based access control
- [ ] Set up Row Level Security
- [ ] Secure database connections
- [ ] Implement data encryption and masking
- [ ] Set up auditing and monitoring

### Immediate Next Steps
1. **Review** your current security posture
2. **Implement** appropriate security measures for your use case
3. **Monitor** and audit security events regularly
4. **Move to Module 12**: [Backup and Recovery](12-backup-recovery.md)

---

## Module Summary

✅ **What You've Learned:**
- Authentication methods and user management
- Role-based access control (RBAC) implementation
- Row Level Security (RLS) for fine-grained access
- Connection security and SSL/TLS configuration
- Data encryption and masking techniques
- Comprehensive auditing and security monitoring

✅ **Skills Acquired:**
- Design secure multi-tenant applications
- Implement proper authentication and authorization
- Protect sensitive data with encryption
- Monitor and audit database security
- Configure secure database connections
- Apply security best practices

✅ **Ready for Next Module:**
You're now ready to learn about backup and recovery in [Module 12: Backup and Recovery](12-backup-recovery.md).

---

## Quick Reference

### User Management
```sql
CREATE USER username WITH PASSWORD 'password';
CREATE ROLE rolename;
GRANT rolename TO username;
ALTER USER username VALID UNTIL 'date';
```

### RLS Commands
```sql
ALTER TABLE tablename ENABLE ROW LEVEL SECURITY;
CREATE POLICY policyname ON tablename FOR operation TO role USING (condition);
```

### Encryption Functions
```sql
pgp_sym_encrypt(data, key)  -- Encrypt data
pgp_sym_decrypt(data, key)  -- Decrypt data
crypt(password, salt)       -- Hash password
gen_salt('bf')              -- Generate salt
```

---
*Continue your PostgreSQL journey with [Backup and Recovery](12-backup-recovery.md) →*
