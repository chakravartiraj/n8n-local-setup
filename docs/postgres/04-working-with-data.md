# Working with Data

## 🎯 Learning Objectives
By the end of this module, you will:
- Import and export data in various formats
- Handle CSV files and bulk operations efficiently
- Validate and clean data during import
- Perform backup and restore operations
- Use COPY command for high-performance data transfer

## 📚 Table of Contents
1. [Data Import/Export Overview](#data-importexport-overview)
2. [CSV Operations](#csv-operations)
3. [Bulk Data Operations](#bulk-data-operations)
4. [Data Validation and Cleaning](#data-validation-and-cleaning)
5. [Backup and Restore](#backup-and-restore)
6. [Advanced Data Handling](#advanced-data-handling)
7. [Hands-On Exercises](#hands-on-exercises)
8. [Best Practices](#best-practices)
9. [Next Steps](#next-steps)

---

## Data Import/Export Overview

### Common Data Formats
- **CSV**: Comma-Separated Values (most common)
- **JSON**: JavaScript Object Notation
- **XML**: Extensible Markup Language
- **SQL**: Database dump files
- **Binary**: Custom formats

### PostgreSQL Data Transfer Methods

| Method | Use Case | Performance | Complexity |
|--------|----------|-------------|------------|
| **COPY** | Bulk import/export | Fastest | Medium |
| **INSERT** | Small datasets | Slow | Simple |
| **pg_dump/pg_restore** | Full backups | Medium | Simple |
| **ETL Tools** | Complex transformations | Variable | Complex |

---

## CSV Operations

### COPY Command Basics

#### Import CSV to Table
```sql
-- Basic CSV import
COPY table_name FROM '/path/to/file.csv' WITH CSV HEADER;

-- Import specific columns
COPY table_name (column1, column2, column3) 
FROM '/path/to/file.csv' WITH CSV HEADER;

-- Import with custom delimiter
COPY table_name FROM '/path/to/file.csv' 
WITH (FORMAT csv, HEADER true, DELIMITER '|');
```

#### Export Table to CSV
```sql
-- Basic CSV export
COPY table_name TO '/path/to/output.csv' WITH CSV HEADER;

-- Export specific columns
COPY (SELECT column1, column2 FROM table_name WHERE condition) 
TO '/path/to/output.csv' WITH CSV HEADER;

-- Export with custom options
COPY table_name TO '/path/to/output.csv' 
WITH (FORMAT csv, HEADER true, DELIMITER ';', QUOTE '"');
```

### Practical CSV Examples

#### Setup Sample Data
```sql
-- Create employees table for CSV examples
CREATE TABLE employees_import (
    id INTEGER,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    hire_date DATE,
    salary DECIMAL(10,2),
    department VARCHAR(50)
);
```

#### Sample CSV File Content (employees.csv)
```csv
id,first_name,last_name,email,hire_date,salary,department
1,John,Doe,john.doe@company.com,2023-01-15,75000.00,Engineering
2,Jane,Smith,jane.smith@company.com,2023-02-01,80000.00,Engineering
3,Bob,Johnson,bob.johnson@company.com,2023-01-20,65000.00,Marketing
4,Alice,Wilson,alice.wilson@company.com,2023-03-01,70000.00,Sales
5,Charlie,Brown,charlie.brown@company.com,2023-02-15,85000.00,Finance
```

#### Import CSV File
```sql
-- Import the CSV file (adjust path as needed)
COPY employees_import 
FROM '/home/user/employees.csv' 
WITH (FORMAT csv, HEADER true);

-- Verify import
SELECT * FROM employees_import;
```

### Handling CSV Import Issues

#### Missing Files or Permission Issues
```sql
-- Alternative: Use \copy in psql (runs on client side)
\copy employees_import FROM 'employees.csv' WITH (FORMAT csv, HEADER true);
```

#### Data Type Mismatches
```sql
-- Create staging table with all TEXT columns
CREATE TABLE employees_staging (
    id TEXT,
    first_name TEXT,
    last_name TEXT,
    email TEXT,
    hire_date TEXT,
    salary TEXT,
    department TEXT
);

-- Import to staging table
COPY employees_staging FROM '/path/to/employees.csv' WITH CSV HEADER;

-- Transform and insert to final table
INSERT INTO employees_import (id, first_name, last_name, email, hire_date, salary, department)
SELECT 
    id::INTEGER,
    first_name,
    last_name,
    email,
    hire_date::DATE,
    salary::DECIMAL(10,2),
    department
FROM employees_staging
WHERE id ~ '^\d+$'  -- Only numeric IDs
  AND hire_date ~ '^\d{4}-\d{2}-\d{2}$'  -- Valid date format
  AND salary ~ '^\d+\.?\d*$';  -- Valid number format
```

#### Handling NULL Values and Empty Fields
```sql
-- CSV with empty fields
-- id,first_name,last_name,email,hire_date,salary,department
-- 1,John,Doe,john@company.com,2023-01-15,,Engineering
-- 2,Jane,Smith,,2023-02-01,80000,

-- Import with NULL handling
COPY employees_import 
FROM '/path/to/employees_with_nulls.csv' 
WITH (FORMAT csv, HEADER true, NULL '');

-- Or specify custom NULL representation
COPY employees_import 
FROM '/path/to/employees.csv' 
WITH (FORMAT csv, HEADER true, NULL 'NULL');
```

### Advanced CSV Options

#### Custom Delimiters and Quotes
```sql
-- Tab-separated values
COPY employees_import 
FROM '/path/to/employees.tsv' 
WITH (FORMAT csv, HEADER true, DELIMITER E'\t');

-- Custom quote character
COPY employees_import 
FROM '/path/to/employees.csv' 
WITH (FORMAT csv, HEADER true, QUOTE '''');

-- Escape character handling
COPY employees_import 
FROM '/path/to/employees.csv' 
WITH (FORMAT csv, HEADER true, ESCAPE '\');
```

#### Encoding Handling
```sql
-- Specify encoding for international characters
COPY employees_import 
FROM '/path/to/employees_utf8.csv' 
WITH (FORMAT csv, HEADER true, ENCODING 'UTF8');

-- Latin-1 encoding
COPY employees_import 
FROM '/path/to/employees_latin1.csv' 
WITH (FORMAT csv, HEADER true, ENCODING 'LATIN1');
```

---

## Bulk Data Operations

### INSERT with Multiple Values
```sql
-- Insert multiple rows efficiently
INSERT INTO employees_import (first_name, last_name, email, hire_date, salary, department) VALUES
    ('Employee1', 'Test1', 'emp1@company.com', '2023-04-01', 60000, 'IT'),
    ('Employee2', 'Test2', 'emp2@company.com', '2023-04-02', 62000, 'IT'),
    ('Employee3', 'Test3', 'emp3@company.com', '2023-04-03', 64000, 'IT'),
    ('Employee4', 'Test4', 'emp4@company.com', '2023-04-04', 66000, 'IT'),
    ('Employee5', 'Test5', 'emp5@company.com', '2023-04-05', 68000, 'IT');
```

### INSERT with SELECT (Data Transfer)
```sql
-- Copy data between tables
CREATE TABLE employees_backup AS 
SELECT * FROM employees_import WHERE 1=0;  -- Structure only

-- Insert data from another table
INSERT INTO employees_backup 
SELECT * FROM employees_import WHERE department = 'Engineering';

-- Insert with transformation
INSERT INTO employees_backup (id, first_name, last_name, email, hire_date, salary, department)
SELECT 
    id + 1000,  -- Offset IDs
    UPPER(first_name),  -- Uppercase names
    UPPER(last_name),
    LOWER(email),  -- Lowercase emails
    hire_date,
    salary * 1.1,  -- 10% salary increase
    department
FROM employees_import
WHERE salary > 70000;
```

### Batch Processing with Transactions
```sql
-- Process large datasets in batches
DO $$
DECLARE
    batch_size INTEGER := 1000;
    offset_val INTEGER := 0;
    total_rows INTEGER;
BEGIN
    -- Get total row count
    SELECT COUNT(*) INTO total_rows FROM large_source_table;
    
    -- Process in batches
    WHILE offset_val < total_rows LOOP
        INSERT INTO target_table
        SELECT * FROM large_source_table
        ORDER BY id
        LIMIT batch_size OFFSET offset_val;
        
        offset_val := offset_val + batch_size;
        
        -- Commit each batch
        COMMIT;
        
        RAISE NOTICE 'Processed % rows out of %', offset_val, total_rows;
    END LOOP;
END $$;
```

---

## Data Validation and Cleaning

### Pre-Import Validation

#### Create Validation Functions
```sql
-- Function to validate email format
CREATE OR REPLACE FUNCTION is_valid_email(email TEXT)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$';
END;
$$ LANGUAGE plpgsql;

-- Function to validate phone number
CREATE OR REPLACE FUNCTION is_valid_phone(phone TEXT)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN phone ~ '^\+?[1-9]\d{1,14}$';
END;
$$ LANGUAGE plpgsql;

-- Function to validate date range
CREATE OR REPLACE FUNCTION is_valid_hire_date(hire_date DATE)
RETURNS BOOLEAN AS $$
BEGIN
    RETURN hire_date BETWEEN '1950-01-01' AND CURRENT_DATE;
END;
$$ LANGUAGE plpgsql;
```

#### Validation During Import
```sql
-- Create table with validation constraints
CREATE TABLE employees_validated (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL CHECK (LENGTH(TRIM(first_name)) > 0),
    last_name VARCHAR(50) NOT NULL CHECK (LENGTH(TRIM(last_name)) > 0),
    email VARCHAR(100) UNIQUE NOT NULL CHECK (is_valid_email(email)),
    hire_date DATE CHECK (is_valid_hire_date(hire_date)),
    salary DECIMAL(10,2) CHECK (salary > 0 AND salary <= 1000000),
    department VARCHAR(50) CHECK (department IN ('Engineering', 'Marketing', 'Sales', 'Finance', 'HR'))
);

-- Import with automatic validation
INSERT INTO employees_validated (first_name, last_name, email, hire_date, salary, department)
SELECT 
    TRIM(first_name),
    TRIM(last_name),
    LOWER(TRIM(email)),
    hire_date::DATE,
    salary::DECIMAL(10,2),
    department
FROM employees_staging
WHERE is_valid_email(TRIM(email))
  AND hire_date::DATE IS NOT NULL
  AND salary::DECIMAL(10,2) > 0;
```

### Data Cleaning Techniques

#### String Cleaning
```sql
-- Clean and standardize text data
UPDATE employees_import SET
    first_name = INITCAP(TRIM(first_name)),  -- Proper case
    last_name = INITCAP(TRIM(last_name)),
    email = LOWER(TRIM(email)),  -- Lowercase emails
    department = UPPER(TRIM(department));  -- Uppercase departments

-- Remove special characters
UPDATE employees_import SET
    first_name = REGEXP_REPLACE(first_name, '[^A-Za-z\s''-]', '', 'g'),
    last_name = REGEXP_REPLACE(last_name, '[^A-Za-z\s''-]', '', 'g');

-- Standardize phone numbers
UPDATE employees_import SET
    phone = REGEXP_REPLACE(phone, '[^0-9+]', '', 'g')
WHERE phone IS NOT NULL;
```

#### Duplicate Detection and Removal
```sql
-- Find duplicates
SELECT email, COUNT(*) as count
FROM employees_import
GROUP BY email
HAVING COUNT(*) > 1;

-- Remove duplicates keeping the first occurrence
DELETE FROM employees_import a
USING employees_import b
WHERE a.id > b.id 
  AND a.email = b.email;

-- Or create a deduplicated table
CREATE TABLE employees_deduplicated AS
SELECT DISTINCT ON (email) *
FROM employees_import
ORDER BY email, id;
```

#### Missing Data Handling
```sql
-- Fill missing salaries with department average
UPDATE employees_import e1
SET salary = (
    SELECT AVG(salary)
    FROM employees_import e2
    WHERE e2.department = e1.department
      AND e2.salary IS NOT NULL
)
WHERE e1.salary IS NULL;

-- Fill missing departments based on email domain
UPDATE employees_import
SET department = CASE
    WHEN email LIKE '%eng%' THEN 'Engineering'
    WHEN email LIKE '%marketing%' THEN 'Marketing'
    WHEN email LIKE '%sales%' THEN 'Sales'
    ELSE 'General'
END
WHERE department IS NULL;
```

---

## Backup and Restore

### Using pg_dump and pg_restore

#### Database Backup
```bash
# Complete database backup
pg_dump -h localhost -U username -d database_name > backup.sql

# Compressed backup
pg_dump -h localhost -U username -d database_name | gzip > backup.sql.gz

# Custom format backup (recommended)
pg_dump -h localhost -U username -Fc database_name > backup.dump

# Backup specific tables
pg_dump -h localhost -U username -d database_name -t employees -t departments > tables_backup.sql

# Backup with specific options
pg_dump -h localhost -U username -d database_name \
    --no-owner \
    --no-privileges \
    --clean \
    --if-exists > backup.sql
```

#### Database Restore
```bash
# Restore from SQL file
psql -h localhost -U username -d database_name < backup.sql

# Restore from compressed file
gunzip -c backup.sql.gz | psql -h localhost -U username -d database_name

# Restore from custom format
pg_restore -h localhost -U username -d database_name backup.dump

# Restore with options
pg_restore -h localhost -U username -d database_name \
    --clean \
    --if-exists \
    --verbose \
    backup.dump
```

### Table-Level Backup and Restore

#### Export Table Data
```sql
-- Export specific table to CSV
COPY employees_import TO '/backup/employees_backup.csv' WITH CSV HEADER;

-- Export with query
COPY (
    SELECT id, first_name, last_name, email, salary
    FROM employees_import
    WHERE hire_date >= '2023-01-01'
) TO '/backup/recent_employees.csv' WITH CSV HEADER;
```

#### Import Table Data
```sql
-- Import from CSV backup
COPY employees_import FROM '/backup/employees_backup.csv' WITH CSV HEADER;

-- Create table from backup
CREATE TABLE employees_restored (LIKE employees_import);
COPY employees_restored FROM '/backup/employees_backup.csv' WITH CSV HEADER;
```

### Automated Backup Script
```bash
#!/bin/bash
# backup_script.sh

# Configuration
DB_NAME="your_database"
DB_USER="your_username"
BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Database backup
pg_dump -h localhost -U "$DB_USER" -Fc "$DB_NAME" > "$BACKUP_DIR/db_backup_$DATE.dump"

# Compress old backups (older than 7 days)
find "$BACKUP_DIR" -name "*.dump" -mtime +7 -exec gzip {} \;

# Remove very old backups (older than 30 days)
find "$BACKUP_DIR" -name "*.dump.gz" -mtime +30 -delete

echo "Backup completed: db_backup_$DATE.dump"
```

---

## Advanced Data Handling

### JSON Data Import/Export

#### Import JSON Data
```sql
-- Create table for JSON data
CREATE TABLE user_data (
    id SERIAL PRIMARY KEY,
    user_info JSONB
);

-- Import JSON from file (PostgreSQL 12+)
COPY user_data (user_info) 
FROM '/path/to/users.json' 
WITH (FORMAT text);

-- Insert JSON data
INSERT INTO user_data (user_info) VALUES
    ('{"name": "John Doe", "age": 30, "email": "john@example.com", "preferences": {"theme": "dark"}}'),
    ('{"name": "Jane Smith", "age": 25, "email": "jane@example.com", "preferences": {"theme": "light"}}');

-- Query JSON data
SELECT 
    user_info->>'name' AS name,
    user_info->>'email' AS email,
    user_info->'preferences'->>'theme' AS theme
FROM user_data;
```

#### Export JSON Data
```sql
-- Export as JSON
COPY (
    SELECT row_to_json(t) 
    FROM (
        SELECT id, first_name, last_name, email, salary
        FROM employees_import
    ) t
) TO '/path/to/employees.json';

-- Export JSONB column
COPY (SELECT user_info FROM user_data) 
TO '/path/to/user_info.json';
```

### Binary Data Handling

#### Large Object (LO) Operations
```sql
-- Create table with large object reference
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    filename VARCHAR(255),
    content_type VARCHAR(100),
    file_data OID  -- Large object reference
);

-- Import large file (using psql \lo_import)
-- \lo_import '/path/to/large_file.pdf' 12345
INSERT INTO documents (filename, content_type, file_data)
VALUES ('document.pdf', 'application/pdf', 12345);

-- Export large file (using psql \lo_export)
-- \lo_export 12345 '/path/to/exported_file.pdf'
```

### Data Transformation During Import

#### Complex ETL Example
```sql
-- Create staging table for raw data
CREATE TABLE sales_staging (
    raw_data TEXT
);

-- Import raw CSV line by line
COPY sales_staging (raw_data) 
FROM '/path/to/messy_sales.csv' 
WITH (FORMAT text);

-- Transform and load to final table
INSERT INTO sales_clean (date, product, quantity, price, total)
SELECT 
    TO_DATE(split_part(raw_data, ',', 1), 'MM/DD/YYYY') AS date,
    TRIM(split_part(raw_data, ',', 2)) AS product,
    split_part(raw_data, ',', 3)::INTEGER AS quantity,
    REPLACE(split_part(raw_data, ',', 4), '$', '')::DECIMAL(10,2) AS price,
    split_part(raw_data, ',', 3)::INTEGER * 
    REPLACE(split_part(raw_data, ',', 4), '$', '')::DECIMAL(10,2) AS total
FROM sales_staging
WHERE split_part(raw_data, ',', 1) ~ '^\d{2}/\d{2}/\d{4}$'  -- Valid date
  AND split_part(raw_data, ',', 3) ~ '^\d+$'                -- Valid quantity
  AND split_part(raw_data, ',', 4) ~ '^\$\d+\.?\d*$';       -- Valid price
```

---

## Hands-On Exercises

### Exercise 1: CSV Import/Export Practice

1. Create a products table with appropriate columns
2. Create a sample CSV file with product data
3. Import the CSV file
4. Validate and clean the imported data
5. Export specific products to a new CSV file

```sql
-- Solution:

-- 1. Create products table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    sku VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    price DECIMAL(10,2) CHECK (price > 0),
    category VARCHAR(100),
    in_stock BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Sample CSV content (save as products.csv):
-- name,sku,description,price,category,in_stock
-- "Laptop Computer","LAP-001","High-performance laptop",999.99,"Electronics",true
-- "Office Chair","CHR-001","Ergonomic office chair",299.99,"Furniture",true
-- "Coffee Mug","MUG-001","Ceramic coffee mug",12.99,"Kitchen",true

-- 3. Import CSV (adjust path)
\copy products (name, sku, description, price, category, in_stock) FROM 'products.csv' WITH CSV HEADER;

-- 4. Validate and clean
UPDATE products SET
    name = TRIM(name),
    sku = UPPER(TRIM(sku)),
    category = INITCAP(TRIM(category));

-- 5. Export electronics products
COPY (
    SELECT name, sku, price, category
    FROM products
    WHERE category = 'Electronics'
) TO '/tmp/electronics.csv' WITH CSV HEADER;
```

### Exercise 2: Data Validation and Cleaning

1. Create a customer table with validation constraints
2. Import messy customer data
3. Clean and standardize the data
4. Remove duplicates
5. Generate a data quality report

```sql
-- Solution:

-- 1. Customer table with validation
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL CHECK (LENGTH(TRIM(first_name)) > 0),
    last_name VARCHAR(50) NOT NULL CHECK (LENGTH(TRIM(last_name)) > 0),
    email VARCHAR(100) UNIQUE NOT NULL CHECK (email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    phone VARCHAR(15) CHECK (phone ~ '^\+?[1-9]\d{1,14}$'),
    city VARCHAR(100),
    state CHAR(2),
    zip_code VARCHAR(10),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Staging table for messy data
CREATE TABLE customers_staging (
    first_name TEXT,
    last_name TEXT,
    email TEXT,
    phone TEXT,
    city TEXT,
    state TEXT,
    zip_code TEXT
);

-- Sample messy data import (create CSV with inconsistent data)
-- 3. Clean data
UPDATE customers_staging SET
    first_name = INITCAP(TRIM(first_name)),
    last_name = INITCAP(TRIM(last_name)),
    email = LOWER(TRIM(email)),
    phone = REGEXP_REPLACE(phone, '[^0-9+]', '', 'g'),
    city = INITCAP(TRIM(city)),
    state = UPPER(TRIM(state)),
    zip_code = REGEXP_REPLACE(zip_code, '[^0-9-]', '', 'g');

-- 4. Remove duplicates and insert clean data
INSERT INTO customers (first_name, last_name, email, phone, city, state, zip_code)
SELECT DISTINCT ON (email)
    first_name, last_name, email, phone, city, state, zip_code
FROM customers_staging
WHERE email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
  AND LENGTH(TRIM(first_name)) > 0
  AND LENGTH(TRIM(last_name)) > 0
ORDER BY email, first_name;

-- 5. Data quality report
SELECT 
    'Total Records' AS metric,
    COUNT(*) AS count
FROM customers_staging
UNION ALL
SELECT 
    'Valid Email Addresses',
    COUNT(*)
FROM customers_staging
WHERE email ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
UNION ALL
SELECT 
    'Successfully Imported',
    COUNT(*)
FROM customers;
```

### Exercise 3: Backup and Restore Practice

1. Create a complete backup of your database
2. Export specific tables to CSV
3. Simulate data loss by dropping a table
4. Restore the table from your backup
5. Verify data integrity

```bash
# Solution (bash commands):

# 1. Complete database backup
pg_dump -h localhost -U your_username -Fc your_database > full_backup.dump

# 2. Export specific tables
psql -h localhost -U your_username -d your_database -c "\copy employees TO 'employees_backup.csv' WITH CSV HEADER"

# 3. Simulate data loss (be careful!)
psql -h localhost -U your_username -d your_database -c "DROP TABLE IF EXISTS employees_backup;"

# 4. Restore table structure and data
pg_restore -h localhost -U your_username -d your_database -t employees_backup full_backup.dump

# 5. Verify data integrity
psql -h localhost -U your_username -d your_database -c "SELECT COUNT(*) FROM employees_backup;"
```

---

## Best Practices

### Import/Export Best Practices

1. **Always validate data** before importing to production
2. **Use staging tables** for complex transformations
3. **Test with small datasets** first
4. **Use transactions** for data integrity
5. **Monitor disk space** during large operations

### Performance Optimization

```sql
-- Disable indexes during bulk import
DROP INDEX IF EXISTS idx_employees_email;

-- Import data
COPY employees FROM '/large_file.csv' WITH CSV HEADER;

-- Recreate indexes
CREATE INDEX idx_employees_email ON employees(email);

-- Update statistics
ANALYZE employees;
```

### Error Handling

```sql
-- Create error logging table
CREATE TABLE import_errors (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(100),
    error_message TEXT,
    error_data TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Import with error handling
DO $$
DECLARE
    error_record RECORD;
BEGIN
    -- Attempt import
    FOR error_record IN 
        SELECT * FROM messy_data_staging
    LOOP
        BEGIN
            INSERT INTO clean_table (column1, column2)
            VALUES (error_record.column1, error_record.column2);
        EXCEPTION WHEN OTHERS THEN
            INSERT INTO import_errors (table_name, error_message, error_data)
            VALUES ('clean_table', SQLERRM, error_record::TEXT);
        END;
    END LOOP;
END $$;
```

---

## Next Steps

### Module Completion Checklist
- [ ] Import/export CSV data successfully
- [ ] Validate and clean imported data
- [ ] Perform bulk operations efficiently
- [ ] Create and restore backups
- [ ] Handle data transformation scenarios

### Immediate Next Steps
1. **Practice** with real-world datasets
2. **Experiment** with different file formats
3. **Create** backup and restore procedures
4. **Move to Module 5**: [Advanced Queries](05-advanced-queries.md)

### Additional Practice
- Work with JSON data import/export
- Create ETL pipelines for data transformation
- Practice with large datasets (100K+ rows)
- Experiment with different CSV formats and encodings

---

## Module Summary

✅ **What You've Learned:**
- CSV import/export with COPY command
- Data validation and cleaning techniques
- Bulk operations and batch processing
- Backup and restore procedures
- Advanced data handling scenarios

✅ **Skills Acquired:**
- Efficiently transfer data in/out of PostgreSQL
- Validate and clean messy data
- Handle large datasets safely
- Create reliable backup strategies
- Transform data during import

✅ **Ready for Next Module:**
You're now ready to learn advanced SQL queries in [Module 5: Advanced Queries](05-advanced-queries.md).

---

## Quick Reference

### COPY Command
```sql
-- Import CSV
COPY table FROM '/path/file.csv' WITH (FORMAT csv, HEADER true);

-- Export CSV
COPY table TO '/path/file.csv' WITH (FORMAT csv, HEADER true);

-- Custom options
COPY table FROM '/path/file.csv' WITH (
    FORMAT csv, 
    HEADER true, 
    DELIMITER ',', 
    NULL '', 
    ENCODING 'UTF8'
);
```

### Backup Commands
```bash
# Backup database
pg_dump -Fc database > backup.dump

# Restore database
pg_restore -d database backup.dump

# Backup table to CSV
\copy table TO 'file.csv' WITH CSV HEADER
```

### Data Cleaning Functions
```sql
-- String cleaning
TRIM(), UPPER(), LOWER(), INITCAP()
REGEXP_REPLACE(string, pattern, replacement)

-- Validation
LENGTH(), email ~ 'pattern'

-- NULL handling
COALESCE(), NULLIF()
```

---
*Continue your PostgreSQL journey with [Advanced Queries](05-advanced-queries.md) →*
