# Basic SQL Operations

## 🎯 Learning Objectives
By the end of this module, you will:
- Master fundamental SQL operations (CRUD)
- Write effective WHERE clauses with various conditions
- Use sorting, grouping, and limiting techniques
- Apply built-in functions and operators
- Handle NULL values and data validation

## 📚 Table of Contents
1. [CRUD Operations Overview](#crud-operations-overview)
2. [Creating Data (INSERT)](#creating-data-insert)
3. [Reading Data (SELECT)](#reading-data-select)
4. [Updating Data (UPDATE)](#updating-data-update)
5. [Deleting Data (DELETE)](#deleting-data-delete)
6. [Advanced Filtering](#advanced-filtering)
7. [Functions and Operators](#functions-and-operators)
8. [Hands-On Exercises](#hands-on-exercises)
9. [Best Practices](#best-practices)
10. [Next Steps](#next-steps)

---

## CRUD Operations Overview

CRUD stands for Create, Read, Update, Delete - the four basic operations for persistent storage:

| Operation | SQL Command | Purpose |
|-----------|-------------|---------|
| **Create** | INSERT | Add new data |
| **Read** | SELECT | Retrieve existing data |
| **Update** | UPDATE | Modify existing data |
| **Delete** | DELETE | Remove data |

### Sample Database Setup
Let's create a sample database for our examples:

```sql
-- Create sample tables
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    budget DECIMAL(12,2),
    location VARCHAR(100)
);

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    hire_date DATE DEFAULT CURRENT_DATE,
    salary DECIMAL(10,2),
    department_id INTEGER REFERENCES departments(id),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    start_date DATE,
    end_date DATE,
    budget DECIMAL(12,2),
    status VARCHAR(20) DEFAULT 'planning'
);

CREATE TABLE employee_projects (
    employee_id INTEGER REFERENCES employees(id),
    project_id INTEGER REFERENCES projects(id),
    role VARCHAR(50),
    hours_allocated INTEGER,
    PRIMARY KEY (employee_id, project_id)
);
```

---

## Creating Data (INSERT)

### Basic INSERT Syntax
```sql
INSERT INTO table_name (column1, column2, column3)
VALUES (value1, value2, value3);
```

### Single Row Insert
```sql
-- Insert a department
INSERT INTO departments (name, budget, location)
VALUES ('Engineering', 500000.00, 'San Francisco');

-- Insert an employee
INSERT INTO employees (first_name, last_name, email, salary, department_id)
VALUES ('John', 'Doe', 'john.doe@company.com', 75000.00, 1);
```

### Multiple Row Insert
```sql
-- Insert multiple departments at once
INSERT INTO departments (name, budget, location) VALUES
    ('Marketing', 200000.00, 'New York'),
    ('Sales', 300000.00, 'Chicago'),
    ('HR', 150000.00, 'Austin'),
    ('Finance', 250000.00, 'Boston');

-- Insert multiple employees
INSERT INTO employees (first_name, last_name, email, salary, department_id) VALUES
    ('Jane', 'Smith', 'jane.smith@company.com', 80000.00, 1),
    ('Bob', 'Johnson', 'bob.johnson@company.com', 65000.00, 2),
    ('Alice', 'Wilson', 'alice.wilson@company.com', 70000.00, 1),
    ('Charlie', 'Brown', 'charlie.brown@company.com', 85000.00, 3),
    ('Diana', 'Davis', 'diana.davis@company.com', 90000.00, 4);
```

### INSERT with SELECT (Copy Data)
```sql
-- Create a backup table
CREATE TABLE employees_backup AS SELECT * FROM employees WHERE 1=0; -- Structure only

-- Insert data from another table
INSERT INTO employees_backup 
SELECT * FROM employees WHERE salary > 75000;
```

### INSERT with DEFAULT VALUES
```sql
-- Insert using some default values
INSERT INTO employees (first_name, last_name, email, department_id)
VALUES ('Test', 'User', 'test@company.com', 1);
-- hire_date will use DEFAULT (CURRENT_DATE)
-- is_active will use DEFAULT (TRUE)
```

### Returning Data from INSERT
```sql
-- Return the inserted data
INSERT INTO departments (name, budget, location)
VALUES ('Research', 400000.00, 'Seattle')
RETURNING id, name, budget;

-- Return specific columns
INSERT INTO employees (first_name, last_name, email, salary, department_id)
VALUES ('New', 'Employee', 'new@company.com', 60000.00, 1)
RETURNING id, first_name || ' ' || last_name AS full_name, hire_date;
```

---

## Reading Data (SELECT)

### Basic SELECT Syntax
```sql
SELECT column1, column2, ...
FROM table_name
WHERE condition
ORDER BY column
LIMIT number;
```

### Simple Queries
```sql
-- Select all columns
SELECT * FROM employees;

-- Select specific columns
SELECT first_name, last_name, email FROM employees;

-- Select with calculated column
SELECT first_name, last_name, salary, salary * 12 AS annual_salary
FROM employees;

-- Select distinct values
SELECT DISTINCT department_id FROM employees;
```

### WHERE Clause Conditions

#### Comparison Operators
```sql
-- Equal to
SELECT * FROM employees WHERE department_id = 1;

-- Not equal to
SELECT * FROM employees WHERE department_id != 2;
SELECT * FROM employees WHERE department_id <> 2; -- Alternative syntax

-- Greater than, less than
SELECT * FROM employees WHERE salary > 70000;
SELECT * FROM employees WHERE salary <= 80000;

-- Between range
SELECT * FROM employees WHERE salary BETWEEN 65000 AND 80000;

-- In list
SELECT * FROM employees WHERE department_id IN (1, 2, 3);

-- Not in list
SELECT * FROM employees WHERE department_id NOT IN (4, 5);
```

#### String Matching
```sql
-- Exact match
SELECT * FROM employees WHERE first_name = 'John';

-- Pattern matching with LIKE
SELECT * FROM employees WHERE first_name LIKE 'J%';     -- Starts with 'J'
SELECT * FROM employees WHERE last_name LIKE '%son';    -- Ends with 'son'
SELECT * FROM employees WHERE email LIKE '%@company%';  -- Contains '@company'
SELECT * FROM employees WHERE first_name LIKE 'J_hn';   -- 'J', any char, 'hn'

-- Case-insensitive matching with ILIKE
SELECT * FROM employees WHERE first_name ILIKE 'john';

-- Regular expressions
SELECT * FROM employees WHERE email ~ '^[a-z]+\.';      -- Starts with lowercase letters followed by dot
```

#### NULL Handling
```sql
-- Check for NULL values
SELECT * FROM employees WHERE salary IS NULL;
SELECT * FROM employees WHERE salary IS NOT NULL;

-- COALESCE for NULL replacement
SELECT first_name, last_name, COALESCE(salary, 0) AS salary_or_zero
FROM employees;
```

#### Logical Operators
```sql
-- AND
SELECT * FROM employees 
WHERE department_id = 1 AND salary > 70000;

-- OR
SELECT * FROM employees 
WHERE department_id = 1 OR salary > 85000;

-- NOT
SELECT * FROM employees 
WHERE NOT (department_id = 1);

-- Complex combinations with parentheses
SELECT * FROM employees 
WHERE (department_id = 1 OR department_id = 2) 
  AND salary BETWEEN 70000 AND 90000;
```

### Sorting Results (ORDER BY)
```sql
-- Sort ascending (default)
SELECT * FROM employees ORDER BY last_name;

-- Sort descending
SELECT * FROM employees ORDER BY salary DESC;

-- Multiple column sorting
SELECT * FROM employees 
ORDER BY department_id ASC, salary DESC;

-- Sort by calculated column
SELECT first_name, last_name, salary, salary * 12 AS annual_salary
FROM employees
ORDER BY annual_salary DESC;

-- Sort with NULL handling
SELECT * FROM employees 
ORDER BY salary NULLS LAST;  -- NULLs at the end
```

### Limiting Results (LIMIT and OFFSET)
```sql
-- Get first 5 employees
SELECT * FROM employees LIMIT 5;

-- Skip first 3, get next 5 (pagination)
SELECT * FROM employees OFFSET 3 LIMIT 5;

-- Get top 3 highest paid employees
SELECT first_name, last_name, salary
FROM employees
ORDER BY salary DESC
LIMIT 3;
```

### Aggregation Functions
```sql
-- Count records
SELECT COUNT(*) FROM employees;
SELECT COUNT(salary) FROM employees;  -- Excludes NULLs

-- Sum, Average, Min, Max
SELECT 
    SUM(salary) AS total_payroll,
    AVG(salary) AS average_salary,
    MIN(salary) AS lowest_salary,
    MAX(salary) AS highest_salary
FROM employees;

-- Count distinct values
SELECT COUNT(DISTINCT department_id) AS unique_departments
FROM employees;
```

### GROUP BY and HAVING
```sql
-- Group by department
SELECT department_id, COUNT(*) AS employee_count
FROM employees
GROUP BY department_id;

-- Group with multiple columns
SELECT department_id, is_active, COUNT(*) AS count
FROM employees
GROUP BY department_id, is_active;

-- Group with aggregations
SELECT 
    department_id,
    COUNT(*) AS employee_count,
    AVG(salary) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees
GROUP BY department_id;

-- HAVING clause (filter groups)
SELECT department_id, COUNT(*) AS employee_count
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 1;

-- Combined WHERE and HAVING
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
WHERE is_active = TRUE
GROUP BY department_id
HAVING AVG(salary) > 70000;
```

---

## Updating Data (UPDATE)

### Basic UPDATE Syntax
```sql
UPDATE table_name
SET column1 = value1, column2 = value2, ...
WHERE condition;
```

### Simple Updates
```sql
-- Update single column
UPDATE employees
SET salary = 80000
WHERE id = 1;

-- Update multiple columns
UPDATE employees
SET salary = 85000, department_id = 2
WHERE id = 1;

-- Update with calculation
UPDATE employees
SET salary = salary * 1.10  -- 10% raise
WHERE department_id = 1;
```

### Conditional Updates
```sql
-- Update based on condition
UPDATE employees
SET salary = CASE 
    WHEN salary < 70000 THEN salary * 1.15  -- 15% raise for lower salaries
    WHEN salary < 80000 THEN salary * 1.10  -- 10% raise for medium salaries
    ELSE salary * 1.05                      -- 5% raise for higher salaries
END
WHERE is_active = TRUE;

-- Update with subquery
UPDATE employees
SET department_id = (
    SELECT id FROM departments WHERE name = 'Engineering'
)
WHERE email = 'john.doe@company.com';
```

### UPDATE with JOIN (PostgreSQL style)
```sql
-- Update using data from another table
UPDATE employees
SET salary = employees.salary + (departments.budget * 0.001)
FROM departments
WHERE employees.department_id = departments.id
  AND departments.name = 'Engineering';
```

### UPDATE with RETURNING
```sql
-- Return updated data
UPDATE employees
SET salary = salary * 1.10
WHERE department_id = 1
RETURNING id, first_name, last_name, salary;
```

---

## Deleting Data (DELETE)

### Basic DELETE Syntax
```sql
DELETE FROM table_name
WHERE condition;
```

### Simple Deletes
```sql
-- Delete specific record
DELETE FROM employees WHERE id = 10;

-- Delete based on condition
DELETE FROM employees WHERE is_active = FALSE;

-- Delete with multiple conditions
DELETE FROM employees 
WHERE department_id = 5 AND hire_date < '2020-01-01';
```

### DELETE with Subquery
```sql
-- Delete employees from departments with low budget
DELETE FROM employees
WHERE department_id IN (
    SELECT id FROM departments WHERE budget < 200000
);
```

### DELETE with JOIN
```sql
-- Delete using data from another table
DELETE FROM employees
USING departments
WHERE employees.department_id = departments.id
  AND departments.name = 'Discontinued';
```

### DELETE with RETURNING
```sql
-- Return deleted data
DELETE FROM employees
WHERE is_active = FALSE
RETURNING id, first_name, last_name, hire_date;
```

### TRUNCATE (Fast Delete All)
```sql
-- Remove all data from table (faster than DELETE)
TRUNCATE TABLE table_name;

-- Truncate with cascade (also truncate referencing tables)
TRUNCATE TABLE departments CASCADE;

-- Restart identity columns
TRUNCATE TABLE employees RESTART IDENTITY;
```

---

## Advanced Filtering

### Date and Time Filtering
```sql
-- Date comparisons
SELECT * FROM employees WHERE hire_date = '2023-01-15';
SELECT * FROM employees WHERE hire_date >= '2023-01-01';
SELECT * FROM employees WHERE hire_date BETWEEN '2023-01-01' AND '2023-12-31';

-- Extract parts of dates
SELECT * FROM employees WHERE EXTRACT(YEAR FROM hire_date) = 2023;
SELECT * FROM employees WHERE EXTRACT(MONTH FROM hire_date) = 6;

-- Date functions
SELECT * FROM employees WHERE hire_date > CURRENT_DATE - INTERVAL '1 year';
SELECT * FROM employees WHERE DATE_PART('dow', hire_date) = 1; -- Monday
```

### Advanced String Operations
```sql
-- String functions in WHERE
SELECT * FROM employees WHERE LENGTH(first_name) > 5;
SELECT * FROM employees WHERE UPPER(first_name) = 'JOHN';
SELECT * FROM employees WHERE first_name ILIKE '%john%';

-- String concatenation
SELECT first_name || ' ' || last_name AS full_name FROM employees;

-- String replacement
SELECT REPLACE(email, '@company.com', '@newcompany.com') AS new_email
FROM employees;

-- Substring extraction
SELECT SUBSTRING(email FROM 1 FOR POSITION('@' IN email) - 1) AS username
FROM employees;
```

### Array Operations
```sql
-- Create table with array column
CREATE TABLE employee_skills (
    employee_id INTEGER REFERENCES employees(id),
    skills TEXT[]
);

-- Insert array data
INSERT INTO employee_skills VALUES
    (1, ARRAY['Python', 'SQL', 'JavaScript']),
    (2, ARRAY['Java', 'SQL', 'React']);

-- Query array data
SELECT * FROM employee_skills WHERE 'SQL' = ANY(skills);
SELECT * FROM employee_skills WHERE skills @> ARRAY['Python'];
SELECT * FROM employee_skills WHERE array_length(skills, 1) > 2;
```

---

## Functions and Operators

### Mathematical Functions
```sql
-- Basic math
SELECT 
    salary,
    salary + 5000 AS increased_salary,
    salary * 1.1 AS with_raise,
    salary / 12 AS monthly_salary,
    salary % 1000 AS remainder
FROM employees;

-- Math functions
SELECT 
    ABS(-42) AS absolute_value,
    CEIL(42.3) AS ceiling,
    FLOOR(42.7) AS floor,
    ROUND(42.567, 2) AS rounded,
    POWER(2, 3) AS power,
    SQRT(16) AS square_root;
```

### String Functions
```sql
SELECT 
    first_name,
    UPPER(first_name) AS uppercase,
    LOWER(first_name) AS lowercase,
    INITCAP(first_name) AS title_case,
    LENGTH(first_name) AS name_length,
    REVERSE(first_name) AS reversed,
    LEFT(first_name, 3) AS first_three,
    RIGHT(first_name, 3) AS last_three
FROM employees;

-- String manipulation
SELECT 
    CONCAT(first_name, ' ', last_name) AS full_name,
    TRIM('  spaced  ') AS trimmed,
    LPAD(id::TEXT, 5, '0') AS padded_id,
    SPLIT_PART(email, '@', 1) AS username
FROM employees;
```

### Date Functions
```sql
SELECT 
    hire_date,
    CURRENT_DATE AS today,
    CURRENT_DATE - hire_date AS days_employed,
    AGE(CURRENT_DATE, hire_date) AS employment_duration,
    EXTRACT(YEAR FROM hire_date) AS hire_year,
    DATE_TRUNC('month', hire_date) AS hire_month_start,
    hire_date + INTERVAL '1 year' AS anniversary
FROM employees;
```

### Conditional Functions
```sql
-- CASE statement
SELECT 
    first_name,
    salary,
    CASE 
        WHEN salary < 70000 THEN 'Entry Level'
        WHEN salary < 85000 THEN 'Mid Level'
        ELSE 'Senior Level'
    END AS level
FROM employees;

-- COALESCE (return first non-NULL)
SELECT 
    first_name,
    COALESCE(salary, 0) AS salary_or_zero
FROM employees;

-- NULLIF (return NULL if values are equal)
SELECT 
    first_name,
    NULLIF(salary, 0) AS salary_null_if_zero
FROM employees;
```

---

## Hands-On Exercises

### Exercise 1: Employee Management Queries

Write queries for the following scenarios:

1. Find all employees earning more than $75,000
2. Get employees hired in 2023
3. Find employees whose first name starts with 'J'
4. Calculate the average salary by department
5. Find the top 3 highest-paid employees

```sql
-- Solutions:

-- 1. Employees earning more than $75,000
SELECT first_name, last_name, salary
FROM employees
WHERE salary > 75000
ORDER BY salary DESC;

-- 2. Employees hired in 2023
SELECT first_name, last_name, hire_date
FROM employees
WHERE EXTRACT(YEAR FROM hire_date) = 2023;

-- 3. Employees whose first name starts with 'J'
SELECT first_name, last_name, email
FROM employees
WHERE first_name LIKE 'J%';

-- 4. Average salary by department
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
WHERE salary IS NOT NULL
GROUP BY department_id
ORDER BY avg_salary DESC;

-- 5. Top 3 highest-paid employees
SELECT first_name, last_name, salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 3;
```

### Exercise 2: Data Modification Practice

1. Insert a new department called "Innovation" with a budget of $600,000
2. Add a new employee to the Innovation department
3. Give all Engineering employees a 10% raise
4. Update the location of the Marketing department to "Los Angeles"
5. Remove all inactive employees

```sql
-- Solutions:

-- 1. Insert new department
INSERT INTO departments (name, budget, location)
VALUES ('Innovation', 600000.00, 'San Francisco')
RETURNING id, name;

-- 2. Add new employee (assuming Innovation department id is 6)
INSERT INTO employees (first_name, last_name, email, salary, department_id)
VALUES ('Innovation', 'Leader', 'innovation.leader@company.com', 95000.00, 6);

-- 3. Give Engineering employees 10% raise (assuming dept id 1 is Engineering)
UPDATE employees
SET salary = salary * 1.10
WHERE department_id = 1 AND salary IS NOT NULL;

-- 4. Update Marketing department location
UPDATE departments
SET location = 'Los Angeles'
WHERE name = 'Marketing';

-- 5. Remove inactive employees
DELETE FROM employees
WHERE is_active = FALSE;
```

### Exercise 3: Advanced Queries

1. Find employees who earn more than the average salary
2. Get department names with their employee counts
3. Find employees hired in the last 6 months
4. Calculate years of service for each employee
5. Find departments with more than 2 employees

```sql
-- Solutions:

-- 1. Employees earning more than average
SELECT first_name, last_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees WHERE salary IS NOT NULL);

-- 2. Department names with employee counts
SELECT d.name, COUNT(e.id) AS employee_count
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name
ORDER BY employee_count DESC;

-- 3. Employees hired in last 6 months
SELECT first_name, last_name, hire_date
FROM employees
WHERE hire_date >= CURRENT_DATE - INTERVAL '6 months';

-- 4. Years of service for each employee
SELECT 
    first_name,
    last_name,
    hire_date,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date)) AS years_of_service
FROM employees
ORDER BY years_of_service DESC;

-- 5. Departments with more than 2 employees
SELECT d.name, COUNT(e.id) AS employee_count
FROM departments d
JOIN employees e ON d.id = e.department_id
GROUP BY d.id, d.name
HAVING COUNT(e.id) > 2;
```

---

## Best Practices

### Query Writing
1. **Use specific column names** instead of SELECT *
2. **Always use WHERE clauses** when updating or deleting
3. **Use LIMIT** for large result sets during development
4. **Order results** consistently for predictable output
5. **Use aliases** for calculated columns and complex expressions

### Performance Considerations
```sql
-- Good: Use indexes effectively
SELECT * FROM employees WHERE department_id = 1;

-- Good: Limit results when exploring
SELECT * FROM employees LIMIT 10;

-- Good: Use EXISTS instead of IN with subqueries
SELECT * FROM employees e
WHERE EXISTS (SELECT 1 FROM departments d WHERE d.id = e.department_id);
```

### Safety Practices
```sql
-- Always use WHERE with UPDATE and DELETE
UPDATE employees SET salary = 80000 WHERE id = 1;
DELETE FROM employees WHERE is_active = FALSE;

-- Use transactions for multiple operations
BEGIN;
UPDATE employees SET department_id = 2 WHERE id = 1;
UPDATE departments SET budget = budget - 10000 WHERE id = 1;
UPDATE departments SET budget = budget + 10000 WHERE id = 2;
COMMIT;

-- Test DELETE with SELECT first
SELECT * FROM employees WHERE is_active = FALSE;  -- Check what will be deleted
-- DELETE FROM employees WHERE is_active = FALSE;  -- Then delete
```

---

## Next Steps

### Module Completion Checklist
- [ ] Understand CRUD operations
- [ ] Write complex WHERE clauses
- [ ] Use aggregation functions and GROUP BY
- [ ] Apply sorting and limiting
- [ ] Handle NULL values appropriately
- [ ] Use built-in functions effectively

### Immediate Next Steps
1. **Practice** with the provided exercises
2. **Experiment** with different query patterns
3. **Build** a small project using CRUD operations
4. **Move to Module 4**: [Working with Data](04-working-with-data.md)

### Additional Practice
- Create complex queries combining multiple concepts
- Practice with different data types and operators
- Experiment with conditional logic in queries
- Try importing real data for practice

---

## Module Summary

✅ **What You've Learned:**
- Complete CRUD operations in SQL
- Advanced filtering with WHERE clauses
- Data aggregation and grouping
- Built-in functions and operators
- Query optimization and safety practices

✅ **Skills Acquired:**
- Write efficient SELECT queries
- Safely update and delete data
- Use functions for data manipulation
- Handle NULL values and edge cases
- Apply best practices for query writing

✅ **Ready for Next Module:**
You're now ready to learn data import/export and advanced data handling in [Module 4: Working with Data](04-working-with-data.md).

---

## Quick Reference

### CRUD Operations
```sql
-- CREATE
INSERT INTO table (columns) VALUES (values);

-- READ
SELECT columns FROM table WHERE condition ORDER BY column LIMIT n;

-- UPDATE
UPDATE table SET column = value WHERE condition;

-- DELETE
DELETE FROM table WHERE condition;
```

### Common Functions
```sql
-- Aggregation
COUNT(*), SUM(column), AVG(column), MIN(column), MAX(column)

-- String
UPPER(), LOWER(), LENGTH(), TRIM(), CONCAT(), SUBSTRING()

-- Date
CURRENT_DATE, CURRENT_TIMESTAMP, EXTRACT(), DATE_TRUNC(), AGE()

-- Conditional
CASE WHEN ... THEN ... ELSE ... END, COALESCE(), NULLIF()
```

### WHERE Clause Operators
```sql
=, !=, <>, <, >, <=, >=
BETWEEN, IN, NOT IN
LIKE, ILIKE, ~
IS NULL, IS NOT NULL
AND, OR, NOT
```

---
*Continue your PostgreSQL journey with [Working with Data](04-working-with-data.md) →*
