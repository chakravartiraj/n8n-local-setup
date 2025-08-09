# Database Fundamentals

## 🎯 Learning Objectives
By the end of this module, you will:
- Understand core database concepts and terminology
- Master PostgreSQL data types and their usage
- Design effective table structures with constraints
- Implement relationships between tables
- Apply database design principles

## 📚 Table of Contents
1. [Core Database Concepts](#core-database-concepts)
2. [PostgreSQL Data Types](#postgresql-data-types)
3. [Tables and Constraints](#tables-and-constraints)
4. [Relationships and Keys](#relationships-and-keys)
5. [Schema Design Principles](#schema-design-principles)
6. [Hands-On Exercises](#hands-on-exercises)
7. [Best Practices](#best-practices)
8. [Next Steps](#next-steps)

---

## Core Database Concepts

### Database Hierarchy
```
PostgreSQL Cluster
├── Database (e.g., ecommerce)
│   ├── Schema (public, private, etc.)
│   │   ├── Tables
│   │   ├── Views
│   │   ├── Functions
│   │   ├── Triggers
│   │   └── Indexes
│   └── Users and Permissions
```

### Key Terminology

#### Database
A collection of related data organized for efficient access and management.

#### Schema
A logical container for database objects within a database. Think of it as a namespace.

```sql
-- Create custom schema
CREATE SCHEMA inventory;

-- Create table in specific schema
CREATE TABLE inventory.products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

-- Access table in schema
SELECT * FROM inventory.products;
```

#### Table
A collection of related data organized in rows and columns.

#### Row (Record/Tuple)
A single data entry in a table.

#### Column (Field/Attribute)
A single data element in a table, with a specific data type.

#### Primary Key
A unique identifier for each row in a table.

#### Foreign Key
A reference to the primary key of another table, establishing relationships.

---

## PostgreSQL Data Types

### Numeric Types

#### Integer Types
```sql
-- Small integer (-32,768 to 32,767)
SMALLINT

-- Standard integer (-2,147,483,648 to 2,147,483,647)
INTEGER or INT

-- Large integer (-9,223,372,036,854,775,808 to 9,223,372,036,854,775,807)
BIGINT

-- Auto-incrementing integer
SERIAL      -- Creates INTEGER with auto-increment
BIGSERIAL   -- Creates BIGINT with auto-increment
```

#### Decimal Types
```sql
-- Fixed-point number
DECIMAL(precision, scale)
NUMERIC(precision, scale)

-- Example: DECIMAL(10,2) = 8 digits before decimal, 2 after
-- Can store: 12345678.99

-- Floating-point numbers
REAL            -- 4 bytes, 6 decimal digits precision
DOUBLE PRECISION -- 8 bytes, 15 decimal digits precision
```

#### Example Usage
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    quantity SMALLINT,
    price DECIMAL(10,2),
    weight REAL,
    scientific_value DOUBLE PRECISION
);
```

### Character Types

```sql
-- Fixed-length string (padded with spaces)
CHAR(n)
CHARACTER(n)

-- Variable-length string (up to n characters)
VARCHAR(n)
CHARACTER VARYING(n)

-- Unlimited length text
TEXT

-- Examples
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    country_code CHAR(2),        -- 'US', 'CA', etc.
    username VARCHAR(50),        -- Up to 50 characters
    bio TEXT                     -- Unlimited length
);
```

### Date and Time Types

```sql
-- Date only (YYYY-MM-DD)
DATE

-- Time only (HH:MM:SS)
TIME

-- Date and time without timezone
TIMESTAMP

-- Date and time with timezone
TIMESTAMPTZ

-- Time interval
INTERVAL

-- Examples
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_date DATE,
    start_time TIME,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    scheduled_for TIMESTAMPTZ,
    duration INTERVAL
);

-- Insert examples
INSERT INTO events (event_date, start_time, scheduled_for, duration) VALUES
    ('2025-12-25', '09:00:00', '2025-12-25 09:00:00+00', '2 hours 30 minutes');
```

### Boolean Type

```sql
-- Boolean values
BOOLEAN or BOOL

-- Can store: TRUE, FALSE, NULL
-- Also accepts: 't', 'f', 'true', 'false', 'y', 'n', 'yes', 'no', '1', '0'

CREATE TABLE settings (
    id SERIAL PRIMARY KEY,
    is_active BOOLEAN DEFAULT TRUE,
    email_notifications BOOLEAN DEFAULT FALSE
);
```

### Array Types

```sql
-- Array of integers
INTEGER[]

-- Array of text
TEXT[]

-- Multi-dimensional arrays
INTEGER[][]

-- Examples
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    tags TEXT[],
    ratings INTEGER[]
);

-- Insert array data
INSERT INTO articles (title, tags, ratings) VALUES
    ('PostgreSQL Tutorial', ARRAY['database', 'sql', 'tutorial'], ARRAY[5, 4, 5, 4]);

-- Or using array literal syntax
INSERT INTO articles (title, tags, ratings) VALUES
    ('Advanced Queries', '{"postgresql", "advanced", "queries"}', '{5, 5, 4}');
```

### JSON Types

```sql
-- JSON data (stored as text, validated)
JSON

-- Binary JSON (more efficient)
JSONB

CREATE TABLE user_preferences (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    preferences JSONB
);

-- Insert JSON data
INSERT INTO user_preferences (user_id, preferences) VALUES
    (1, '{"theme": "dark", "language": "en", "notifications": {"email": true, "push": false}}');
```

---

## Tables and Constraints

### Creating Tables

#### Basic Syntax
```sql
CREATE TABLE table_name (
    column_name data_type constraints,
    column_name data_type constraints,
    ...
    table_constraints
);
```

#### Comprehensive Example
```sql
CREATE TABLE employees (
    -- Primary key with auto-increment
    id SERIAL PRIMARY KEY,
    
    -- Required text fields
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    
    -- Unique email
    email VARCHAR(100) UNIQUE NOT NULL,
    
    -- Optional phone with pattern check
    phone VARCHAR(15) CHECK (phone ~ '^\+?[1-9]\d{1,14}$'),
    
    -- Salary with range constraint
    salary DECIMAL(10,2) CHECK (salary > 0 AND salary <= 1000000),
    
    -- Department reference (foreign key)
    department_id INTEGER REFERENCES departments(id),
    
    -- Hire date with default
    hire_date DATE DEFAULT CURRENT_DATE,
    
    -- Status with limited options
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'terminated')),
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Column Constraints

#### NOT NULL
```sql
-- Column cannot contain NULL values
first_name VARCHAR(50) NOT NULL
```

#### UNIQUE
```sql
-- Column values must be unique across all rows
email VARCHAR(100) UNIQUE,

-- Composite unique constraint
UNIQUE (first_name, last_name, department_id)
```

#### CHECK
```sql
-- Custom validation rules
age INTEGER CHECK (age >= 18 AND age <= 100),
status VARCHAR(20) CHECK (status IN ('pending', 'approved', 'rejected')),
email VARCHAR(100) CHECK (email LIKE '%@%.%')
```

#### DEFAULT
```sql
-- Default value when none provided
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
is_active BOOLEAN DEFAULT TRUE,
country VARCHAR(2) DEFAULT 'US'
```

### Table Constraints

#### Primary Key
```sql
-- Single column primary key
id SERIAL PRIMARY KEY,

-- Composite primary key
PRIMARY KEY (customer_id, order_id)
```

#### Foreign Key
```sql
-- Reference another table
department_id INTEGER REFERENCES departments(id),

-- With cascading actions
department_id INTEGER REFERENCES departments(id) 
    ON DELETE CASCADE 
    ON UPDATE RESTRICT,

-- Named constraint
CONSTRAINT fk_employee_department 
    FOREIGN KEY (department_id) 
    REFERENCES departments(id)
```

---

## Relationships and Keys

### Types of Relationships

#### One-to-One (1:1)
Each record in table A relates to exactly one record in table B.

```sql
-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL
);

-- User profiles table (1:1 with users)
CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    bio TEXT,
    avatar_url VARCHAR(255)
);
```

#### One-to-Many (1:N)
Each record in table A can relate to multiple records in table B.

```sql
-- Categories table
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT
);

-- Products table (many products per category)
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category_id INTEGER REFERENCES categories(id),
    price DECIMAL(10,2),
    description TEXT
);
```

#### Many-to-Many (M:N)
Records in table A can relate to multiple records in table B and vice versa.

```sql
-- Students table
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE
);

-- Courses table
CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    credits INTEGER
);

-- Junction table for many-to-many relationship
CREATE TABLE student_courses (
    student_id INTEGER REFERENCES students(id) ON DELETE CASCADE,
    course_id INTEGER REFERENCES courses(id) ON DELETE CASCADE,
    enrollment_date DATE DEFAULT CURRENT_DATE,
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

### Advanced Relationship Examples

#### Self-Referencing Table
```sql
-- Employees table with manager relationship
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    manager_id INTEGER REFERENCES employees(id),
    department VARCHAR(50)
);

-- Insert sample data
INSERT INTO employees (name, manager_id, department) VALUES
    ('John CEO', NULL, 'Executive'),
    ('Jane Manager', 1, 'Sales'),
    ('Bob Employee', 2, 'Sales'),
    ('Alice Employee', 2, 'Sales');
```

#### Multiple Foreign Keys
```sql
-- Orders table with multiple relationships
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    salesperson_id INTEGER REFERENCES employees(id),
    shipping_address_id INTEGER REFERENCES addresses(id),
    billing_address_id INTEGER REFERENCES addresses(id),
    order_date DATE DEFAULT CURRENT_DATE,
    total_amount DECIMAL(10,2)
);
```

---

## Schema Design Principles

### Normalization

#### First Normal Form (1NF)
- Each table cell contains a single value
- Each column has a unique name
- Order of rows and columns doesn't matter

```sql
-- ❌ Not 1NF (multiple values in one cell)
CREATE TABLE bad_customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    phones VARCHAR(255)  -- "123-456-7890, 098-765-4321"
);

-- ✅ 1NF compliant
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE customer_phones (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    phone VARCHAR(15),
    phone_type VARCHAR(20) -- 'mobile', 'home', 'work'
);
```

#### Second Normal Form (2NF)
- Must be in 1NF
- No partial dependencies on composite primary keys

```sql
-- ❌ Not 2NF (student_name depends only on student_id, not full key)
CREATE TABLE bad_enrollments (
    student_id INTEGER,
    course_id INTEGER,
    student_name VARCHAR(100),  -- Depends only on student_id
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id)
);

-- ✅ 2NF compliant
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INTEGER REFERENCES students(id),
    course_id INTEGER REFERENCES courses(id),
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

#### Third Normal Form (3NF)
- Must be in 2NF
- No transitive dependencies

```sql
-- ❌ Not 3NF (city and state depend on zip_code, not customer_id)
CREATE TABLE bad_customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    zip_code VARCHAR(10),
    city VARCHAR(50),      -- Depends on zip_code
    state VARCHAR(2)       -- Depends on zip_code
);

-- ✅ 3NF compliant
CREATE TABLE zip_codes (
    zip_code VARCHAR(10) PRIMARY KEY,
    city VARCHAR(50),
    state VARCHAR(2)
);

CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    zip_code VARCHAR(10) REFERENCES zip_codes(zip_code)
);
```

### Design Best Practices

#### Naming Conventions
```sql
-- Use consistent naming patterns
-- Tables: plural nouns (users, orders, products)
-- Columns: singular nouns (name, email, created_at)
-- Foreign keys: table_name_id (user_id, category_id)

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    order_number VARCHAR(50),
    total_amount DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Data Integrity
```sql
-- Use appropriate constraints
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    sku VARCHAR(50) UNIQUE NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock_quantity INTEGER DEFAULT 0 CHECK (stock_quantity >= 0),
    category_id INTEGER NOT NULL REFERENCES categories(id),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Hands-On Exercises

### Exercise 1: E-commerce Database Design

Design a simple e-commerce database with the following requirements:
- Customers can place multiple orders
- Orders contain multiple products
- Products belong to categories
- Track inventory levels

```sql
-- Solution:

-- Categories table
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    sku VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock_quantity INTEGER DEFAULT 0 CHECK (stock_quantity >= 0),
    category_id INTEGER NOT NULL REFERENCES categories(id),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Customers table
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(15),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    order_number VARCHAR(50) UNIQUE NOT NULL,
    order_date DATE DEFAULT CURRENT_DATE,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled')),
    total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Order items table (junction table for orders and products)
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price > 0),
    total_price DECIMAL(10,2) NOT NULL CHECK (total_price >= 0)
);
```

### Exercise 2: Library Management System

Design a library database with:
- Books and authors (many-to-many relationship)
- Members who can borrow books
- Track borrowing history and due dates

```sql
-- Solution:

-- Authors table
CREATE TABLE authors (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    birth_date DATE,
    nationality VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Books table
CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    isbn VARCHAR(13) UNIQUE,
    publication_date DATE,
    genre VARCHAR(50),
    total_copies INTEGER DEFAULT 1 CHECK (total_copies > 0),
    available_copies INTEGER DEFAULT 1 CHECK (available_copies >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Book authors junction table (many-to-many)
CREATE TABLE book_authors (
    book_id INTEGER REFERENCES books(id) ON DELETE CASCADE,
    author_id INTEGER REFERENCES authors(id) ON DELETE CASCADE,
    PRIMARY KEY (book_id, author_id)
);

-- Members table
CREATE TABLE members (
    id SERIAL PRIMARY KEY,
    member_number VARCHAR(20) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(15),
    address TEXT,
    membership_date DATE DEFAULT CURRENT_DATE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Borrowings table
CREATE TABLE borrowings (
    id SERIAL PRIMARY KEY,
    member_id INTEGER NOT NULL REFERENCES members(id),
    book_id INTEGER NOT NULL REFERENCES books(id),
    borrow_date DATE DEFAULT CURRENT_DATE,
    due_date DATE NOT NULL,
    return_date DATE,
    fine_amount DECIMAL(5,2) DEFAULT 0 CHECK (fine_amount >= 0),
    status VARCHAR(20) DEFAULT 'borrowed' CHECK (status IN ('borrowed', 'returned', 'overdue')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Exercise 3: Data Type Practice

Create a table that uses various PostgreSQL data types:

```sql
-- Sample data types table
CREATE TABLE data_type_examples (
    id SERIAL PRIMARY KEY,
    
    -- Numeric types
    small_number SMALLINT,
    regular_number INTEGER,
    big_number BIGINT,
    decimal_number DECIMAL(10,2),
    float_number REAL,
    precise_float DOUBLE PRECISION,
    
    -- Character types
    fixed_char CHAR(5),
    variable_char VARCHAR(100),
    unlimited_text TEXT,
    
    -- Date and time types
    just_date DATE,
    just_time TIME,
    date_and_time TIMESTAMP,
    date_time_with_tz TIMESTAMPTZ,
    time_interval INTERVAL,
    
    -- Boolean
    is_active BOOLEAN,
    
    -- Arrays
    tags TEXT[],
    numbers INTEGER[],
    
    -- JSON
    metadata JSONB,
    
    -- Constraints example
    email VARCHAR(100) CHECK (email LIKE '%@%.%'),
    age INTEGER CHECK (age >= 0 AND age <= 150),
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO data_type_examples (
    small_number, regular_number, big_number, decimal_number, float_number, precise_float,
    fixed_char, variable_char, unlimited_text,
    just_date, just_time, date_and_time, date_time_with_tz, time_interval,
    is_active, tags, numbers, metadata, email, age
) VALUES (
    100, 50000, 9223372036854775807, 999.99, 3.14159, 2.71828182845904523536,
    'ABCDE', 'Variable length string', 'This is unlimited text that can be very long...',
    '2025-08-09', '14:30:00', '2025-08-09 14:30:00', '2025-08-09 14:30:00+00', '2 hours 30 minutes',
    TRUE, ARRAY['tag1', 'tag2', 'tag3'], ARRAY[1, 2, 3, 4, 5],
    '{"name": "John", "preferences": {"theme": "dark", "language": "en"}}',
    'john@example.com', 30
);
```

---

## Best Practices

### Table Design
1. **Use meaningful names**: Choose descriptive table and column names
2. **Be consistent**: Follow naming conventions throughout your database
3. **Normalize appropriately**: Usually aim for 3NF, but denormalize when performance requires it
4. **Use appropriate data types**: Choose the most restrictive type that fits your data
5. **Add constraints**: Use CHECK, UNIQUE, NOT NULL to enforce data integrity

### Column Definitions
```sql
-- Good column definitions
CREATE TABLE users (
    id SERIAL PRIMARY KEY,                    -- Auto-incrementing PK
    username VARCHAR(50) UNIQUE NOT NULL,     -- Constrained length, unique, required
    email VARCHAR(100) UNIQUE NOT NULL        -- Email format could be checked
        CHECK (email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    password_hash CHAR(60) NOT NULL,          -- bcrypt hash is always 60 chars
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Documentation
```sql
-- Add comments to tables and columns
COMMENT ON TABLE users IS 'Application users and their authentication data';
COMMENT ON COLUMN users.password_hash IS 'bcrypt hash of user password';
COMMENT ON COLUMN users.created_at IS 'Timestamp when user account was created';
```

---

## Next Steps

### Module Completion Checklist
- [ ] Understand database hierarchy and terminology
- [ ] Know PostgreSQL data types and when to use them
- [ ] Can create tables with appropriate constraints
- [ ] Understand relationship types and implementation
- [ ] Apply basic normalization principles

### Immediate Next Steps
1. **Practice** creating tables with various data types
2. **Design** a small database for a real-world scenario
3. **Experiment** with different constraint types
4. **Move to Module 3**: [Basic SQL Operations](03-basic-sql-operations.md)

### Additional Resources
- [PostgreSQL Data Types Documentation](https://postgresql.org/docs/current/datatype.html)
- [Database Design Fundamentals](https://en.wikipedia.org/wiki/Database_design)
- [Normalization Rules](https://en.wikipedia.org/wiki/Database_normalization)

---

## Module Summary

✅ **What You've Learned:**
- Core database concepts and PostgreSQL hierarchy
- Comprehensive overview of PostgreSQL data types
- Table creation with constraints and relationships
- Database design principles and normalization
- Best practices for schema design

✅ **Skills Acquired:**
- Design database schemas
- Choose appropriate data types
- Implement table relationships
- Apply constraints for data integrity
- Follow database design best practices

✅ **Ready for Next Module:**
You're now ready to learn SQL operations in [Module 3: Basic SQL Operations](03-basic-sql-operations.md).

---

## Quick Reference

### Common Data Types
```sql
-- Numeric
SERIAL, INTEGER, BIGINT, DECIMAL(p,s), REAL, DOUBLE PRECISION

-- Text
CHAR(n), VARCHAR(n), TEXT

-- Date/Time
DATE, TIME, TIMESTAMP, TIMESTAMPTZ, INTERVAL

-- Other
BOOLEAN, JSON, JSONB, ARRAY
```

### Common Constraints
```sql
-- Column constraints
NOT NULL, UNIQUE, CHECK(condition), DEFAULT value

-- Table constraints
PRIMARY KEY, FOREIGN KEY, UNIQUE(col1, col2), CHECK(condition)
```

### Relationship Patterns
```sql
-- One-to-One: UNIQUE foreign key
user_id INTEGER UNIQUE REFERENCES users(id)

-- One-to-Many: Simple foreign key
category_id INTEGER REFERENCES categories(id)

-- Many-to-Many: Junction table
CREATE TABLE user_roles (
    user_id INTEGER REFERENCES users(id),
    role_id INTEGER REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);
```

---
*Continue your PostgreSQL journey with [Basic SQL Operations](03-basic-sql-operations.md) →*
