# Data Modeling

## 🎯 Learning Objectives
By the end of this module, you will:
- Apply advanced normalization techniques and understand when to denormalize
- Design complex relationships and handle inheritance patterns
- Implement effective schema design patterns for various use cases
- Optimize data models for performance and maintainability
- Handle temporal data and versioning strategies

## 📚 Table of Contents
1. [Advanced Normalization](#advanced-normalization)
2. [Relationship Design Patterns](#relationship-design-patterns)
3. [Schema Design Strategies](#schema-design-strategies)
4. [Inheritance and Polymorphism](#inheritance-and-polymorphism)
5. [Temporal Data Modeling](#temporal-data-modeling)
6. [Performance-Oriented Design](#performance-oriented-design)
7. [Hands-On Exercises](#hands-on-exercises)
8. [Best Practices](#best-practices)
9. [Next Steps](#next-steps)

---

## Advanced Normalization

### Beyond Third Normal Form

#### Boyce-Codd Normal Form (BCNF)
Stricter version of 3NF - every determinant must be a candidate key.

```sql
-- Violation of BCNF example
CREATE TABLE course_instructor_violation (
    student_id INTEGER,
    course_id INTEGER,
    instructor_id INTEGER,
    instructor_office VARCHAR(10),
    PRIMARY KEY (student_id, course_id)
);

-- Problem: instructor_office depends on instructor_id, not the full primary key
-- Solution: Separate tables

CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    credits INTEGER,
    department_id INTEGER
);

CREATE TABLE instructors (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    office VARCHAR(10),
    department_id INTEGER
);

CREATE TABLE course_sections (
    id SERIAL PRIMARY KEY,
    course_id INTEGER REFERENCES courses(id),
    instructor_id INTEGER REFERENCES instructors(id),
    section_number VARCHAR(10),
    semester VARCHAR(20),
    year INTEGER
);

CREATE TABLE enrollments (
    student_id INTEGER,
    course_section_id INTEGER REFERENCES course_sections(id),
    enrollment_date DATE DEFAULT CURRENT_DATE,
    grade CHAR(2),
    PRIMARY KEY (student_id, course_section_id)
);
```

#### Fourth Normal Form (4NF)
Eliminates multi-valued dependencies.

```sql
-- Violation of 4NF
CREATE TABLE student_skills_languages_violation (
    student_id INTEGER,
    skill VARCHAR(50),
    language VARCHAR(50),
    PRIMARY KEY (student_id, skill, language)
);

-- Problem: Skills and languages are independent multi-valued attributes
-- Solution: Separate tables

CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE
);

CREATE TABLE student_skills (
    student_id INTEGER REFERENCES students(id),
    skill VARCHAR(50),
    proficiency_level INTEGER CHECK (proficiency_level BETWEEN 1 AND 5),
    PRIMARY KEY (student_id, skill)
);

CREATE TABLE student_languages (
    student_id INTEGER REFERENCES students(id),
    language VARCHAR(50),
    fluency_level VARCHAR(20) CHECK (fluency_level IN ('Basic', 'Intermediate', 'Advanced', 'Native')),
    PRIMARY KEY (student_id, language)
);
```

### Strategic Denormalization

#### When to Denormalize
1. **Read-heavy workloads** with expensive JOINs
2. **Reporting and analytics** requirements
3. **Performance-critical** applications
4. **Calculated fields** used frequently

```sql
-- Normalized design (pure 3NF)
CREATE TABLE orders_normalized (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date DATE,
    status VARCHAR(20)
);

CREATE TABLE order_items_normalized (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders_normalized(id),
    product_id INTEGER,
    quantity INTEGER,
    unit_price DECIMAL(10,2)
);

-- Denormalized design for performance
CREATE TABLE orders_denormalized (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    customer_name VARCHAR(100),      -- Denormalized
    customer_email VARCHAR(100),     -- Denormalized
    order_date DATE,
    status VARCHAR(20),
    total_amount DECIMAL(12,2),      -- Calculated field
    item_count INTEGER,              -- Calculated field
    last_updated TIMESTAMP DEFAULT NOW()
);

-- Trigger to maintain denormalized data
CREATE OR REPLACE FUNCTION maintain_order_denormalization()
RETURNS TRIGGER AS $$
BEGIN
    -- Update order totals when order items change
    UPDATE orders_denormalized
    SET 
        total_amount = (
            SELECT COALESCE(SUM(quantity * unit_price), 0)
            FROM order_items_normalized
            WHERE order_id = NEW.order_id
        ),
        item_count = (
            SELECT COUNT(*)
            FROM order_items_normalized
            WHERE order_id = NEW.order_id
        ),
        last_updated = NOW()
    WHERE id = NEW.order_id;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER maintain_order_totals
    AFTER INSERT OR UPDATE OR DELETE ON order_items_normalized
    FOR EACH ROW
    EXECUTE FUNCTION maintain_order_denormalization();
```

---

## Relationship Design Patterns

### Complex Many-to-Many Relationships

#### Association Tables with Attributes
```sql
-- Enhanced many-to-many with relationship attributes
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    start_date DATE,
    end_date DATE,
    budget DECIMAL(12,2)
);

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    department VARCHAR(50),
    hourly_rate DECIMAL(8,2)
);

-- Association table with rich attributes
CREATE TABLE project_assignments (
    project_id INTEGER REFERENCES projects(id),
    employee_id INTEGER REFERENCES employees(id),
    role VARCHAR(50) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE,
    hours_allocated INTEGER,
    actual_hours INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'completed', 'suspended')),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (project_id, employee_id, start_date)
);

-- Views for easier querying
CREATE VIEW active_assignments AS
SELECT 
    p.name AS project_name,
    e.name AS employee_name,
    pa.role,
    pa.start_date,
    pa.hours_allocated,
    pa.actual_hours,
    pa.status
FROM project_assignments pa
JOIN projects p ON pa.project_id = p.id
JOIN employees e ON pa.employee_id = e.id
WHERE pa.status = 'active'
  AND (pa.end_date IS NULL OR pa.end_date >= CURRENT_DATE);
```

### Hierarchical Relationships

#### Adjacency List Pattern
```sql
-- Traditional adjacency list
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INTEGER REFERENCES categories(id),
    level INTEGER, -- Denormalized for performance
    path TEXT,     -- Materialized path for easier queries
    is_active BOOLEAN DEFAULT TRUE
);

-- Trigger to maintain path and level
CREATE OR REPLACE FUNCTION maintain_category_hierarchy()
RETURNS TRIGGER AS $$
DECLARE
    parent_path TEXT := '';
    parent_level INTEGER := 0;
BEGIN
    IF NEW.parent_id IS NOT NULL THEN
        SELECT path, level 
        INTO parent_path, parent_level
        FROM categories 
        WHERE id = NEW.parent_id;
        
        NEW.path := parent_path || '/' || NEW.id::TEXT;
        NEW.level := parent_level + 1;
    ELSE
        NEW.path := '/' || NEW.id::TEXT;
        NEW.level := 1;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER category_hierarchy_trigger
    BEFORE INSERT OR UPDATE ON categories
    FOR EACH ROW
    EXECUTE FUNCTION maintain_category_hierarchy();
```

#### Nested Set Model
```sql
-- Nested set model for read-heavy hierarchies
CREATE TABLE nested_categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    left_boundary INTEGER NOT NULL,
    right_boundary INTEGER NOT NULL,
    level INTEGER NOT NULL,
    UNIQUE(left_boundary, right_boundary)
);

-- Function to get all descendants
CREATE OR REPLACE FUNCTION get_category_descendants(category_id INTEGER)
RETURNS TABLE(id INTEGER, name TEXT, level INTEGER) AS $$
BEGIN
    RETURN QUERY
    SELECT nc2.id, nc2.name, nc2.level
    FROM nested_categories nc1
    JOIN nested_categories nc2 ON nc2.left_boundary > nc1.left_boundary 
                               AND nc2.right_boundary < nc1.right_boundary
    WHERE nc1.id = category_id
    ORDER BY nc2.left_boundary;
END;
$$ LANGUAGE plpgsql;

-- Function to get path to root
CREATE OR REPLACE FUNCTION get_category_path(category_id INTEGER)
RETURNS TABLE(id INTEGER, name TEXT, level INTEGER) AS $$
BEGIN
    RETURN QUERY
    SELECT nc2.id, nc2.name, nc2.level
    FROM nested_categories nc1
    JOIN nested_categories nc2 ON nc1.left_boundary >= nc2.left_boundary 
                               AND nc1.right_boundary <= nc2.right_boundary
    WHERE nc1.id = category_id
    ORDER BY nc2.level;
END;
$$ LANGUAGE plpgsql;
```

### Flexible Relationships

#### Entity-Attribute-Value (EAV) Pattern
```sql
-- EAV pattern for flexible attributes
CREATE TABLE entities (
    id SERIAL PRIMARY KEY,
    entity_type VARCHAR(50) NOT NULL,
    name VARCHAR(200) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE attribute_definitions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    data_type VARCHAR(20) NOT NULL CHECK (data_type IN ('text', 'number', 'date', 'boolean')),
    entity_type VARCHAR(50) NOT NULL,
    is_required BOOLEAN DEFAULT FALSE,
    validation_rule TEXT
);

CREATE TABLE entity_attributes (
    entity_id INTEGER REFERENCES entities(id),
    attribute_id INTEGER REFERENCES attribute_definitions(id),
    value_text TEXT,
    value_number DECIMAL(15,4),
    value_date DATE,
    value_boolean BOOLEAN,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (entity_id, attribute_id)
);

-- View to make EAV queries easier
CREATE OR REPLACE VIEW entity_attribute_values AS
SELECT 
    e.id AS entity_id,
    e.entity_type,
    e.name AS entity_name,
    ad.name AS attribute_name,
    ad.data_type,
    CASE ad.data_type
        WHEN 'text' THEN ea.value_text
        WHEN 'number' THEN ea.value_number::TEXT
        WHEN 'date' THEN ea.value_date::TEXT
        WHEN 'boolean' THEN ea.value_boolean::TEXT
    END AS attribute_value
FROM entities e
JOIN entity_attributes ea ON e.id = ea.entity_id
JOIN attribute_definitions ad ON ea.attribute_id = ad.id;
```

---

## Schema Design Strategies

### Multi-Tenant Architectures

#### Shared Schema with Tenant ID
```sql
-- Shared schema pattern
CREATE TABLE tenants (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    subdomain VARCHAR(50) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE tenant_users (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER REFERENCES tenants(id),
    email VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255),
    role VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE tenant_data (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER REFERENCES tenants(id),
    data_type VARCHAR(50),
    content JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Row Level Security for tenant isolation
ALTER TABLE tenant_users ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_data ENABLE ROW LEVEL SECURITY;

-- Policies to ensure tenant isolation
CREATE POLICY tenant_users_policy ON tenant_users
    FOR ALL TO PUBLIC
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);

CREATE POLICY tenant_data_policy ON tenant_data
    FOR ALL TO PUBLIC
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);
```

#### Schema-per-Tenant Pattern
```sql
-- Function to create tenant schema
CREATE OR REPLACE FUNCTION create_tenant_schema(tenant_name TEXT)
RETURNS VOID AS $$
DECLARE
    schema_name TEXT;
BEGIN
    schema_name := 'tenant_' || lower(regexp_replace(tenant_name, '[^a-zA-Z0-9]', '_', 'g'));
    
    -- Create schema
    EXECUTE format('CREATE SCHEMA %I', schema_name);
    
    -- Create tables in tenant schema
    EXECUTE format('
        CREATE TABLE %I.users (
            id SERIAL PRIMARY KEY,
            email VARCHAR(100) UNIQUE,
            password_hash VARCHAR(255),
            role VARCHAR(50),
            created_at TIMESTAMP DEFAULT NOW()
        )', schema_name);
    
    EXECUTE format('
        CREATE TABLE %I.user_data (
            id SERIAL PRIMARY KEY,
            user_id INTEGER REFERENCES %I.users(id),
            data_type VARCHAR(50),
            content JSONB,
            created_at TIMESTAMP DEFAULT NOW()
        )', schema_name, schema_name);
    
    -- Grant permissions
    EXECUTE format('GRANT USAGE ON SCHEMA %I TO tenant_user', schema_name);
    EXECUTE format('GRANT ALL ON ALL TABLES IN SCHEMA %I TO tenant_user', schema_name);
END;
$$ LANGUAGE plpgsql;
```

### Event Sourcing Pattern
```sql
-- Event sourcing implementation
CREATE TABLE event_streams (
    id SERIAL PRIMARY KEY,
    stream_id UUID NOT NULL,
    stream_type VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(stream_id)
);

CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    stream_id UUID REFERENCES event_streams(stream_id),
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    event_version INTEGER NOT NULL,
    occurred_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(stream_id, event_version)
);

CREATE TABLE snapshots (
    id SERIAL PRIMARY KEY,
    stream_id UUID REFERENCES event_streams(stream_id),
    snapshot_data JSONB NOT NULL,
    snapshot_version INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(stream_id, snapshot_version)
);

-- Function to apply events and create projections
CREATE OR REPLACE FUNCTION get_current_state(p_stream_id UUID)
RETURNS JSONB AS $$
DECLARE
    current_state JSONB := '{}';
    event_record RECORD;
    latest_snapshot JSONB;
    start_version INTEGER := 1;
BEGIN
    -- Get latest snapshot
    SELECT snapshot_data, snapshot_version
    INTO latest_snapshot, start_version
    FROM snapshots
    WHERE stream_id = p_stream_id
    ORDER BY snapshot_version DESC
    LIMIT 1;
    
    IF latest_snapshot IS NOT NULL THEN
        current_state := latest_snapshot;
        start_version := start_version + 1;
    END IF;
    
    -- Apply events since snapshot
    FOR event_record IN
        SELECT event_type, event_data
        FROM events
        WHERE stream_id = p_stream_id
          AND event_version >= start_version
        ORDER BY event_version
    LOOP
        -- Apply event to state (simplified example)
        current_state := current_state || event_record.event_data;
    END LOOP;
    
    RETURN current_state;
END;
$$ LANGUAGE plpgsql;
```

---

## Inheritance and Polymorphism

### Table Inheritance Patterns

#### Single Table Inheritance
```sql
-- Single table for all entity types
CREATE TABLE contacts (
    id SERIAL PRIMARY KEY,
    contact_type VARCHAR(20) NOT NULL CHECK (contact_type IN ('person', 'company')),
    
    -- Common fields
    name VARCHAR(200) NOT NULL,
    email VARCHAR(100),
    phone VARCHAR(20),
    address TEXT,
    
    -- Person-specific fields
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    birth_date DATE,
    
    -- Company-specific fields
    company_name VARCHAR(200),
    industry VARCHAR(100),
    employee_count INTEGER,
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    -- Constraints to ensure data integrity
    CONSTRAINT person_fields_check CHECK (
        (contact_type = 'person' AND first_name IS NOT NULL AND last_name IS NOT NULL)
        OR contact_type != 'person'
    ),
    CONSTRAINT company_fields_check CHECK (
        (contact_type = 'company' AND company_name IS NOT NULL)
        OR contact_type != 'company'
    )
);

-- Views for type-specific access
CREATE VIEW persons AS
SELECT 
    id,
    first_name,
    last_name,
    email,
    phone,
    address,
    birth_date,
    created_at
FROM contacts
WHERE contact_type = 'person';

CREATE VIEW companies AS
SELECT 
    id,
    company_name AS name,
    email,
    phone,
    address,
    industry,
    employee_count,
    created_at
FROM contacts
WHERE contact_type = 'company';
```

#### Class Table Inheritance
```sql
-- Base table for common attributes
CREATE TABLE base_contacts (
    id SERIAL PRIMARY KEY,
    email VARCHAR(100),
    phone VARCHAR(20),
    address TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Specific tables for each type
CREATE TABLE persons (
    id INTEGER PRIMARY KEY REFERENCES base_contacts(id) ON DELETE CASCADE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    birth_date DATE
);

CREATE TABLE companies (
    id INTEGER PRIMARY KEY REFERENCES base_contacts(id) ON DELETE CASCADE,
    company_name VARCHAR(200) NOT NULL,
    industry VARCHAR(100),
    employee_count INTEGER
);

-- View for unified access
CREATE VIEW all_contacts AS
SELECT 
    bc.id,
    'person' AS contact_type,
    p.first_name || ' ' || p.last_name AS name,
    bc.email,
    bc.phone,
    bc.address,
    bc.created_at
FROM base_contacts bc
JOIN persons p ON bc.id = p.id
UNION ALL
SELECT 
    bc.id,
    'company' AS contact_type,
    c.company_name AS name,
    bc.email,
    bc.phone,
    bc.address,
    bc.created_at
FROM base_contacts bc
JOIN companies c ON bc.id = c.id;
```

### Polymorphic Associations
```sql
-- Polymorphic comments system
CREATE TABLE commentable_types (
    id SERIAL PRIMARY KEY,
    type_name VARCHAR(50) UNIQUE NOT NULL
);

INSERT INTO commentable_types (type_name) VALUES ('article'), ('product'), ('user');

CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    commentable_type_id INTEGER REFERENCES commentable_types(id),
    commentable_id INTEGER NOT NULL,
    author_id INTEGER,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    
    -- Index for polymorphic lookups
    INDEX (commentable_type_id, commentable_id)
);

-- Function to get comments for any entity
CREATE OR REPLACE FUNCTION get_comments(entity_type TEXT, entity_id INTEGER)
RETURNS TABLE(
    comment_id INTEGER,
    content TEXT,
    author_id INTEGER,
    created_at TIMESTAMP
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        c.id,
        c.content,
        c.author_id,
        c.created_at
    FROM comments c
    JOIN commentable_types ct ON c.commentable_type_id = ct.id
    WHERE ct.type_name = entity_type
      AND c.commentable_id = entity_id
    ORDER BY c.created_at DESC;
END;
$$ LANGUAGE plpgsql;
```

---

## Temporal Data Modeling

### Slowly Changing Dimensions

#### Type 2 SCD - Historical Tracking
```sql
-- Customer dimension with history tracking
CREATE TABLE customer_history (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(100),
    address TEXT,
    city VARCHAR(100),
    state VARCHAR(50),
    country VARCHAR(50),
    
    -- Temporal fields
    valid_from DATE NOT NULL,
    valid_to DATE, -- NULL means current record
    is_current BOOLEAN DEFAULT TRUE,
    
    -- Metadata
    created_at TIMESTAMP DEFAULT NOW(),
    created_by VARCHAR(100),
    
    -- Ensure only one current record per customer
    UNIQUE(customer_id, valid_from),
    EXCLUDE USING gist (
        customer_id WITH =, 
        daterange(valid_from, valid_to, '[)') WITH &&
    ) WHERE (is_current = true)
);

-- Function to update customer (creates new version)
CREATE OR REPLACE FUNCTION update_customer_scd2(
    p_customer_id INTEGER,
    p_name VARCHAR(200),
    p_email VARCHAR(100),
    p_address TEXT,
    p_city VARCHAR(100),
    p_state VARCHAR(50),
    p_country VARCHAR(50),
    p_effective_date DATE DEFAULT CURRENT_DATE
)
RETURNS VOID AS $$
BEGIN
    -- End current record
    UPDATE customer_history
    SET 
        valid_to = p_effective_date - 1,
        is_current = FALSE
    WHERE customer_id = p_customer_id 
      AND is_current = TRUE;
    
    -- Insert new record
    INSERT INTO customer_history (
        customer_id, name, email, address, city, state, country,
        valid_from, is_current, created_by
    ) VALUES (
        p_customer_id, p_name, p_email, p_address, p_city, p_state, p_country,
        p_effective_date, TRUE, current_user
    );
END;
$$ LANGUAGE plpgsql;

-- View for current customers
CREATE VIEW current_customers AS
SELECT 
    customer_id,
    name,
    email,
    address,
    city,
    state,
    country,
    valid_from
FROM customer_history
WHERE is_current = TRUE;
```

### Bitemporal Data
```sql
-- Bitemporal table with valid time and transaction time
CREATE TABLE bitemporal_prices (
    id SERIAL PRIMARY KEY,
    product_id INTEGER NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    
    -- Valid time (when the fact was true in reality)
    valid_from DATE NOT NULL,
    valid_to DATE NOT NULL DEFAULT '9999-12-31',
    
    -- Transaction time (when the fact was recorded in database)
    transaction_from TIMESTAMP NOT NULL DEFAULT NOW(),
    transaction_to TIMESTAMP NOT NULL DEFAULT 'infinity',
    
    -- Metadata
    entered_by VARCHAR(100),
    reason VARCHAR(200),
    
    -- Constraints
    CHECK (valid_from < valid_to),
    CHECK (transaction_from < transaction_to)
);

-- Function to get price as of specific valid time and transaction time
CREATE OR REPLACE FUNCTION get_price_as_of(
    p_product_id INTEGER,
    p_valid_date DATE,
    p_transaction_time TIMESTAMP DEFAULT NOW()
)
RETURNS DECIMAL(10,2) AS $$
DECLARE
    result_price DECIMAL(10,2);
BEGIN
    SELECT price
    INTO result_price
    FROM bitemporal_prices
    WHERE product_id = p_product_id
      AND p_valid_date >= valid_from
      AND p_valid_date < valid_to
      AND p_transaction_time >= transaction_from
      AND p_transaction_time < transaction_to
    ORDER BY transaction_from DESC
    LIMIT 1;
    
    RETURN result_price;
END;
$$ LANGUAGE plpgsql;
```

---

## Performance-Oriented Design

### Partitioning Strategies
```sql
-- Range partitioning by date
CREATE TABLE sales_data (
    id SERIAL,
    sale_date DATE NOT NULL,
    customer_id INTEGER,
    product_id INTEGER,
    amount DECIMAL(10,2),
    region VARCHAR(50)
) PARTITION BY RANGE (sale_date);

-- Create partitions
CREATE TABLE sales_2024_q1 PARTITION OF sales_data
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE sales_2024_q2 PARTITION OF sales_data
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

CREATE TABLE sales_2024_q3 PARTITION OF sales_data
    FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');

CREATE TABLE sales_2024_q4 PARTITION OF sales_data
    FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

-- Hash partitioning for even distribution
CREATE TABLE user_activities (
    id SERIAL,
    user_id INTEGER NOT NULL,
    activity_type VARCHAR(50),
    activity_data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
) PARTITION BY HASH (user_id);

-- Create hash partitions
CREATE TABLE user_activities_0 PARTITION OF user_activities
    FOR VALUES WITH (modulus 4, remainder 0);

CREATE TABLE user_activities_1 PARTITION OF user_activities
    FOR VALUES WITH (modulus 4, remainder 1);

CREATE TABLE user_activities_2 PARTITION OF user_activities
    FOR VALUES WITH (modulus 4, remainder 2);

CREATE TABLE user_activities_3 PARTITION OF user_activities
    FOR VALUES WITH (modulus 4, remainder 3);
```

### Materialized Views for Performance
```sql
-- Complex aggregation materialized view
CREATE MATERIALIZED VIEW customer_metrics AS
SELECT 
    c.id AS customer_id,
    c.name,
    c.email,
    COUNT(o.id) AS total_orders,
    SUM(o.total_amount) AS total_spent,
    AVG(o.total_amount) AS avg_order_value,
    MAX(o.order_date) AS last_order_date,
    EXTRACT(DAYS FROM CURRENT_DATE - MAX(o.order_date)) AS days_since_last_order,
    CASE 
        WHEN SUM(o.total_amount) > 10000 THEN 'VIP'
        WHEN SUM(o.total_amount) > 5000 THEN 'Premium'
        WHEN SUM(o.total_amount) > 1000 THEN 'Regular'
        ELSE 'New'
    END AS customer_tier
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.email;

-- Create indexes on materialized view
CREATE INDEX idx_customer_metrics_tier ON customer_metrics(customer_tier);
CREATE INDEX idx_customer_metrics_spent ON customer_metrics(total_spent);

-- Refresh function with concurrency
CREATE OR REPLACE FUNCTION refresh_customer_metrics()
RETURNS VOID AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY customer_metrics;
END;
$$ LANGUAGE plpgsql;

-- Schedule regular refresh (using pg_cron if available)
-- SELECT cron.schedule('refresh-customer-metrics', '0 2 * * *', 'SELECT refresh_customer_metrics();');
```

---

## Hands-On Exercises

### Exercise 1: E-commerce Platform Data Model

Design a comprehensive e-commerce platform with complex relationships and requirements.

```sql
-- Solution: Complete e-commerce data model

-- 1. Core entities
CREATE TABLE vendors (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(20),
    address JSONB,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended')),
    commission_rate DECIMAL(5,4) DEFAULT 0.0300, -- 3%
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE,
    parent_id INTEGER REFERENCES categories(id),
    description TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE brands (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    logo_url VARCHAR(500),
    website_url VARCHAR(500),
    is_active BOOLEAN DEFAULT TRUE
);

-- 2. Product catalog with variants
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(300) NOT NULL,
    slug VARCHAR(300) UNIQUE,
    description TEXT,
    short_description VARCHAR(500),
    category_id INTEGER REFERENCES categories(id),
    brand_id INTEGER REFERENCES brands(id),
    vendor_id INTEGER REFERENCES vendors(id),
    base_price DECIMAL(10,2),
    cost_price DECIMAL(10,2),
    weight DECIMAL(8,3),
    dimensions JSONB, -- {length, width, height}
    status VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'discontinued')),
    is_digital BOOLEAN DEFAULT FALSE,
    requires_shipping BOOLEAN DEFAULT TRUE,
    seo_title VARCHAR(200),
    seo_description VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Product variants (different sizes, colors, etc.)
CREATE TABLE product_variants (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id) ON DELETE CASCADE,
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(200),
    price DECIMAL(10,2),
    cost_price DECIMAL(10,2),
    weight DECIMAL(8,3),
    variant_attributes JSONB, -- {color: "red", size: "large"}
    inventory_quantity INTEGER DEFAULT 0,
    inventory_policy VARCHAR(20) DEFAULT 'deny' CHECK (inventory_policy IN ('deny', 'continue')),
    requires_shipping BOOLEAN DEFAULT TRUE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 3. Customer management
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(20),
    birth_date DATE,
    gender VARCHAR(10) CHECK (gender IN ('M', 'F', 'Other')),
    customer_group VARCHAR(50) DEFAULT 'general',
    accepts_marketing BOOLEAN DEFAULT FALSE,
    email_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE customer_addresses (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id) ON DELETE CASCADE,
    type VARCHAR(20) DEFAULT 'shipping' CHECK (type IN ('shipping', 'billing')),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company VARCHAR(100),
    address_line_1 VARCHAR(200) NOT NULL,
    address_line_2 VARCHAR(200),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(2) NOT NULL,
    phone VARCHAR(20),
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 4. Order management
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id INTEGER REFERENCES customers(id),
    email VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded')),
    financial_status VARCHAR(20) DEFAULT 'pending' CHECK (financial_status IN ('pending', 'paid', 'partially_paid', 'refunded', 'cancelled')),
    
    -- Pricing
    subtotal DECIMAL(12,2) NOT NULL,
    tax_amount DECIMAL(12,2) DEFAULT 0,
    shipping_amount DECIMAL(12,2) DEFAULT 0,
    discount_amount DECIMAL(12,2) DEFAULT 0,
    total_amount DECIMAL(12,2) NOT NULL,
    
    -- Addresses (denormalized for historical accuracy)
    billing_address JSONB,
    shipping_address JSONB,
    
    -- Timestamps
    order_date TIMESTAMP DEFAULT NOW(),
    shipped_date TIMESTAMP,
    delivered_date TIMESTAMP,
    cancelled_date TIMESTAMP,
    
    -- Metadata
    notes TEXT,
    currency VARCHAR(3) DEFAULT 'USD',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE,
    product_variant_id INTEGER REFERENCES product_variants(id),
    
    -- Product info snapshot (for historical accuracy)
    product_name VARCHAR(300) NOT NULL,
    variant_name VARCHAR(200),
    sku VARCHAR(100),
    
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    
    created_at TIMESTAMP DEFAULT NOW()
);

-- 5. Inventory tracking
CREATE TABLE inventory_movements (
    id SERIAL PRIMARY KEY,
    product_variant_id INTEGER REFERENCES product_variants(id),
    movement_type VARCHAR(20) NOT NULL CHECK (movement_type IN ('purchase', 'sale', 'adjustment', 'return', 'damage')),
    quantity_change INTEGER NOT NULL, -- Positive for increases, negative for decreases
    reference_type VARCHAR(50), -- 'order', 'return', 'adjustment', etc.
    reference_id INTEGER,
    reason TEXT,
    cost_per_unit DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW(),
    created_by INTEGER -- user_id
);

-- 6. Reviews and ratings
CREATE TABLE product_reviews (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id),
    customer_id INTEGER REFERENCES customers(id),
    rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title VARCHAR(200),
    content TEXT,
    is_verified_purchase BOOLEAN DEFAULT FALSE,
    is_approved BOOLEAN DEFAULT FALSE,
    helpful_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(product_id, customer_id)
);

-- 7. Promotions and discounts
CREATE TABLE discount_codes (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL,
    description VARCHAR(200),
    discount_type VARCHAR(20) NOT NULL CHECK (discount_type IN ('percentage', 'fixed_amount')),
    discount_value DECIMAL(10,2) NOT NULL,
    minimum_order_amount DECIMAL(10,2),
    usage_limit INTEGER,
    usage_count INTEGER DEFAULT 0,
    start_date TIMESTAMP,
    end_date TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Views and functions for common operations
CREATE VIEW product_catalog AS
SELECT 
    p.id,
    p.name,
    p.slug,
    p.description,
    c.name AS category_name,
    b.name AS brand_name,
    v.name AS vendor_name,
    p.base_price,
    COUNT(pv.id) AS variant_count,
    SUM(pv.inventory_quantity) AS total_inventory,
    AVG(pr.rating) AS avg_rating,
    COUNT(pr.id) AS review_count
FROM products p
LEFT JOIN categories c ON p.category_id = c.id
LEFT JOIN brands b ON p.brand_id = b.id
LEFT JOIN vendors v ON p.vendor_id = v.id
LEFT JOIN product_variants pv ON p.id = pv.product_id AND pv.is_active = TRUE
LEFT JOIN product_reviews pr ON p.id = pr.product_id AND pr.is_approved = TRUE
WHERE p.status = 'active'
GROUP BY p.id, p.name, p.slug, p.description, c.name, b.name, v.name, p.base_price;
```

### Exercise 2: Multi-Tenant SaaS Platform

Design a multi-tenant SaaS platform with proper isolation and shared resources.

```sql
-- Solution: Multi-tenant SaaS data model

-- 1. Tenant management
CREATE TABLE organizations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    subdomain VARCHAR(50) UNIQUE,
    custom_domain VARCHAR(100),
    plan_type VARCHAR(50) DEFAULT 'free' CHECK (plan_type IN ('free', 'starter', 'professional', 'enterprise')),
    settings JSONB DEFAULT '{}',
    billing_email VARCHAR(100),
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'cancelled')),
    trial_ends_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- 2. User management with RBAC
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    avatar_url VARCHAR(500),
    is_system_admin BOOLEAN DEFAULT FALSE,
    email_verified_at TIMESTAMP,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE organization_memberships (
    id SERIAL PRIMARY KEY,
    organization_id INTEGER REFERENCES organizations(id) ON DELETE CASCADE,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'pending')),
    invited_by INTEGER REFERENCES users(id),
    invited_at TIMESTAMP,
    joined_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(organization_id, user_id)
);

-- 3. Permission system
CREATE TABLE permissions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    resource VARCHAR(50) NOT NULL,
    action VARCHAR(50) NOT NULL,
    UNIQUE(resource, action)
);

CREATE TABLE role_permissions (
    role VARCHAR(50) NOT NULL,
    permission_id INTEGER REFERENCES permissions(id),
    PRIMARY KEY (role, permission_id)
);

-- 4. Core application entities with tenant isolation
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    organization_id INTEGER REFERENCES organizations(id) ON DELETE CASCADE,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'archived', 'deleted')),
    settings JSONB DEFAULT '{}',
    owner_id INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    organization_id INTEGER REFERENCES organizations(id) ON DELETE CASCADE,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    title VARCHAR(300) NOT NULL,
    description TEXT,
    status VARCHAR(20) DEFAULT 'todo' CHECK (status IN ('todo', 'in_progress', 'review', 'done', 'cancelled')),
    priority VARCHAR(10) DEFAULT 'medium' CHECK (priority IN ('low', 'medium', 'high', 'urgent')),
    assignee_id INTEGER REFERENCES users(id),
    reporter_id INTEGER REFERENCES users(id),
    due_date DATE,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    -- Ensure project belongs to same organization
    CONSTRAINT fk_task_project_org CHECK (
        (SELECT organization_id FROM projects WHERE id = project_id) = organization_id
    )
);

-- 5. Activity tracking
CREATE TABLE activities (
    id SERIAL PRIMARY KEY,
    organization_id INTEGER REFERENCES organizations(id) ON DELETE CASCADE,
    user_id INTEGER REFERENCES users(id),
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id INTEGER NOT NULL,
    metadata JSONB DEFAULT '{}',
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 6. Row Level Security implementation
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE activities ENABLE ROW LEVEL SECURITY;

-- Policies for tenant isolation
CREATE POLICY org_projects_policy ON projects
    FOR ALL TO PUBLIC
    USING (organization_id = current_setting('app.current_organization_id')::INTEGER);

CREATE POLICY org_tasks_policy ON tasks
    FOR ALL TO PUBLIC
    USING (organization_id = current_setting('app.current_organization_id')::INTEGER);

CREATE POLICY org_activities_policy ON activities
    FOR ALL TO PUBLIC
    USING (organization_id = current_setting('app.current_organization_id')::INTEGER);

-- 7. Usage tracking and billing
CREATE TABLE usage_metrics (
    id SERIAL PRIMARY KEY,
    organization_id INTEGER REFERENCES organizations(id) ON DELETE CASCADE,
    metric_type VARCHAR(50) NOT NULL,
    metric_value DECIMAL(15,4) NOT NULL,
    period_start TIMESTAMP NOT NULL,
    period_end TIMESTAMP NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(organization_id, metric_type, period_start)
);

-- Function to set tenant context
CREATE OR REPLACE FUNCTION set_organization_context(org_id INTEGER)
RETURNS VOID AS $$
BEGIN
    PERFORM set_config('app.current_organization_id', org_id::TEXT, true);
END;
$$ LANGUAGE plpgsql;

-- Function to check user permissions
CREATE OR REPLACE FUNCTION user_has_permission(
    p_user_id INTEGER,
    p_organization_id INTEGER,
    p_resource VARCHAR(50),
    p_action VARCHAR(50)
)
RETURNS BOOLEAN AS $$
DECLARE
    user_role VARCHAR(50);
    has_permission BOOLEAN := FALSE;
BEGIN
    -- Get user role in organization
    SELECT role INTO user_role
    FROM organization_memberships
    WHERE user_id = p_user_id 
      AND organization_id = p_organization_id
      AND status = 'active';
    
    IF user_role IS NULL THEN
        RETURN FALSE;
    END IF;
    
    -- Check if role has the required permission
    SELECT EXISTS(
        SELECT 1
        FROM role_permissions rp
        JOIN permissions p ON rp.permission_id = p.id
        WHERE rp.role = user_role
          AND p.resource = p_resource
          AND p.action = p_action
    ) INTO has_permission;
    
    RETURN has_permission;
END;
$$ LANGUAGE plpgsql;
```

---

## Best Practices

### Schema Evolution
```sql
-- Migration pattern for schema changes
CREATE TABLE schema_migrations (
    version VARCHAR(50) PRIMARY KEY,
    description TEXT,
    applied_at TIMESTAMP DEFAULT NOW()
);

-- Example migration function
CREATE OR REPLACE FUNCTION migrate_add_customer_tier()
RETURNS VOID AS $$
BEGIN
    -- Check if migration already applied
    IF EXISTS (SELECT 1 FROM schema_migrations WHERE version = '20241201_001') THEN
        RETURN;
    END IF;
    
    -- Add new column
    ALTER TABLE customers ADD COLUMN tier VARCHAR(20) DEFAULT 'bronze';
    
    -- Populate based on existing data
    UPDATE customers SET tier = CASE
        WHEN total_orders > 50 THEN 'gold'
        WHEN total_orders > 20 THEN 'silver'
        ELSE 'bronze'
    END;
    
    -- Add constraint
    ALTER TABLE customers ADD CONSTRAINT check_tier 
        CHECK (tier IN ('bronze', 'silver', 'gold', 'platinum'));
    
    -- Record migration
    INSERT INTO schema_migrations (version, description)
    VALUES ('20241201_001', 'Add customer tier classification');
END;
$$ LANGUAGE plpgsql;
```

### Data Quality Enforcement
```sql
-- Comprehensive data quality checks
CREATE OR REPLACE FUNCTION validate_data_quality()
RETURNS TABLE(
    table_name TEXT,
    issue_type TEXT,
    issue_count BIGINT,
    sample_ids TEXT
) AS $$
BEGIN
    -- Check for orphaned records
    RETURN QUERY
    SELECT 
        'order_items'::TEXT,
        'orphaned_records'::TEXT,
        COUNT(*),
        string_agg(id::TEXT, ', ' ORDER BY id LIMIT 10)
    FROM order_items oi
    WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.id = oi.order_id)
    HAVING COUNT(*) > 0;
    
    -- Check for invalid email formats
    RETURN QUERY
    SELECT 
        'customers'::TEXT,
        'invalid_email'::TEXT,
        COUNT(*),
        string_agg(id::TEXT, ', ' ORDER BY id LIMIT 10)
    FROM customers
    WHERE email IS NOT NULL 
      AND email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
    HAVING COUNT(*) > 0;
    
    -- Check for negative quantities
    RETURN QUERY
    SELECT 
        'order_items'::TEXT,
        'negative_quantity'::TEXT,
        COUNT(*),
        string_agg(id::TEXT, ', ' ORDER BY id LIMIT 10)
    FROM order_items
    WHERE quantity <= 0
    HAVING COUNT(*) > 0;
    
    -- Check for future dates where inappropriate
    RETURN QUERY
    SELECT 
        'orders'::TEXT,
        'future_order_date'::TEXT,
        COUNT(*),
        string_agg(id::TEXT, ', ' ORDER BY id LIMIT 10)
    FROM orders
    WHERE order_date > CURRENT_TIMESTAMP
    HAVING COUNT(*) > 0;
END;
$$ LANGUAGE plpgsql;
```

---

## Next Steps

### Module Completion Checklist
- [ ] Understand advanced normalization and when to denormalize
- [ ] Design complex relationships and inheritance patterns
- [ ] Implement multi-tenant architectures
- [ ] Handle temporal data and versioning
- [ ] Apply performance-oriented design principles

### Immediate Next Steps
1. **Practice** designing schemas for real-world applications
2. **Experiment** with different modeling approaches
3. **Consider** performance implications of design decisions
4. **Move to Module 9**: [Advanced Features](09-advanced-features.md)

---

## Module Summary

✅ **What You've Learned:**
- Advanced normalization techniques and strategic denormalization
- Complex relationship patterns and inheritance models
- Multi-tenant architecture design
- Temporal data modeling and historical tracking
- Performance-oriented schema design

✅ **Skills Acquired:**
- Design sophisticated data models
- Balance normalization with performance needs
- Implement complex business requirements
- Handle multi-tenant and temporal scenarios
- Apply industry best practices

✅ **Ready for Next Module:**
You're now ready to explore advanced PostgreSQL features in [Module 9: Advanced Features](09-advanced-features.md).

---

## Quick Reference

### Normalization Levels
```sql
-- 1NF: Atomic values, unique column names
-- 2NF: 1NF + no partial dependencies
-- 3NF: 2NF + no transitive dependencies  
-- BCNF: 3NF + every determinant is a candidate key
-- 4NF: BCNF + no multi-valued dependencies
```

### Inheritance Patterns
```sql
-- Single Table Inheritance
CREATE TABLE entities (
    id SERIAL PRIMARY KEY,
    entity_type VARCHAR(20),
    -- all possible columns
);

-- Class Table Inheritance  
CREATE TABLE base_entities (id SERIAL PRIMARY KEY, common_cols);
CREATE TABLE sub_entities (id INTEGER REFERENCES base_entities(id), specific_cols);
```

### Temporal Patterns
```sql
-- SCD Type 2
CREATE TABLE dimensions (
    id SERIAL PRIMARY KEY,
    business_key INTEGER,
    valid_from DATE,
    valid_to DATE,
    is_current BOOLEAN
);

-- Bitemporal
CREATE TABLE facts (
    valid_from DATE,
    valid_to DATE, 
    transaction_from TIMESTAMP,
    transaction_to TIMESTAMP
);
```

---
*Continue your PostgreSQL journey with [Advanced Features](09-advanced-features.md) →*
