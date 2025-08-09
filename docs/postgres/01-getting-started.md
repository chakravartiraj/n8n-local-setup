# Getting Started with PostgreSQL

## 🎯 Learning Objectives
By the end of this module, you will:
- Understand what PostgreSQL is and its advantages
- Install PostgreSQL on your system
- Connect to PostgreSQL using different tools
- Navigate the basic PostgreSQL environment
- Create your first database and table

## 📚 Table of Contents
1. [What is PostgreSQL?](#what-is-postgresql)
2. [Installation](#installation)
3. [First Connection](#first-connection)
4. [Basic Navigation](#basic-navigation)
5. [Your First Database](#your-first-database)
6. [Hands-On Exercises](#hands-on-exercises)
7. [Common Issues](#common-issues)
8. [Next Steps](#next-steps)

---

## What is PostgreSQL?

### Overview
PostgreSQL (often called "Postgres") is a powerful, open-source relational database management system (RDBMS) that has been in active development for over 30 years. It's known for its reliability, feature robustness, and performance.

### Key Features
- **ACID Compliance**: Ensures data integrity
- **Extensible**: Custom functions, data types, and extensions
- **Standards Compliant**: Follows SQL standards closely
- **Multi-Platform**: Runs on all major operating systems
- **Open Source**: Free to use and modify

### Why Choose PostgreSQL?
- **Reliability**: Battle-tested in production environments
- **Performance**: Handles complex queries efficiently
- **Scalability**: From small apps to enterprise systems
- **Advanced Features**: JSON, full-text search, geospatial data
- **Active Community**: Excellent documentation and support

### PostgreSQL vs Other Databases

| Feature | PostgreSQL | MySQL | SQLite | Oracle |
|---------|------------|-------|--------|--------|
| **Open Source** | ✅ | ✅ | ✅ | ❌ |
| **ACID Compliance** | ✅ | ✅ | ✅ | ✅ |
| **JSON Support** | ✅ | ✅ | ✅ | ✅ |
| **Extensibility** | ✅ | ⚠️ | ❌ | ✅ |
| **Learning Curve** | Medium | Easy | Easy | Hard |
| **Enterprise Features** | ✅ | ⚠️ | ❌ | ✅ |

---

## Installation

### Option 1: Native Installation

#### Windows
1. **Download**: Visit [postgresql.org/download/windows](https://postgresql.org/download/windows/)
2. **Run Installer**: Choose PostgreSQL 15+ version
3. **Setup Wizard**:
   - Choose installation directory
   - Select components (PostgreSQL Server, pgAdmin, Command Line Tools)
   - Set superuser password (remember this!)
   - Choose port (default: 5432)
   - Set locale (default is fine)
4. **Complete Installation**: Launch Stack Builder if needed

#### macOS
```bash
# Option A: Using Homebrew (recommended)
brew install postgresql
brew services start postgresql

# Option B: Using Postgres.app
# Download from https://postgresapp.com/

# Option C: Using official installer
# Download from postgresql.org
```

#### Linux (Ubuntu/Debian)
```bash
# Update package list
sudo apt update

# Install PostgreSQL
sudo apt install postgresql postgresql-contrib

# Start PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Switch to postgres user
sudo -u postgres psql
```

#### Linux (CentOS/RHEL)
```bash
# Install PostgreSQL
sudo yum install postgresql postgresql-server

# Initialize database
sudo postgresql-setup initdb

# Start and enable service
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Option 2: Docker Installation (Recommended for Learning)

```bash
# Pull PostgreSQL image
docker pull postgres:15-alpine

# Run PostgreSQL container
docker run --name learning-postgres \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=learning_db \
  -p 5432:5432 \
  -d postgres:15-alpine

# Connect to container
docker exec -it learning-postgres psql -U postgres -d learning_db
```

### Option 3: Using Docker Compose

Create `docker-compose.yml`:
```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    container_name: learning-postgres
    environment:
      POSTGRES_DB: learning_db
      POSTGRES_USER: student
      POSTGRES_PASSWORD: studentpass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

```bash
# Start PostgreSQL
docker-compose up -d

# Connect
docker-compose exec postgres psql -U student -d learning_db
```

---

## First Connection

### Using Command Line (psql)

#### Basic Connection
```bash
# Connect as default user
psql

# Connect with specific parameters
psql -h localhost -p 5432 -U postgres -d postgres

# Connect using connection string
psql "postgresql://username:password@localhost:5432/database_name"
```

#### Common psql Options
```bash
-h, --host          # Database server host
-p, --port          # Database server port
-U, --username      # Database user name
-d, --dbname        # Database name
-W, --password      # Force password prompt
-f, --file          # Execute commands from file
-c, --command       # Run single command
-l, --list          # List available databases
```

### Using pgAdmin (GUI Tool)

1. **Install pgAdmin**: Download from [pgadmin.org](https://pgadmin.org/)
2. **Launch pgAdmin**: Open in web browser
3. **Add Server**:
   - Name: Local PostgreSQL
   - Host: localhost
   - Port: 5432
   - Username: postgres
   - Password: [your password]
4. **Connect**: Expand server tree

### Using DBeaver (Universal Tool)

1. **Install DBeaver**: Download from [dbeaver.io](https://dbeaver.io/)
2. **New Connection**: Database → New Database Connection
3. **Select PostgreSQL**: Choose PostgreSQL driver
4. **Connection Settings**:
   - Server Host: localhost
   - Port: 5432
   - Database: postgres
   - Username: postgres
   - Password: [your password]
5. **Test Connection**: Click "Test Connection"

---

## Basic Navigation

### Essential psql Commands

#### Meta-Commands (start with backslash)
```sql
-- List databases
\l

-- Connect to database
\c database_name

-- List tables in current database
\dt

-- List all tables, views, sequences
\d

-- Describe table structure
\d table_name

-- List users/roles
\du

-- List schemas
\dn

-- Show current connection info
\conninfo

-- Get help
\h          -- SQL command help
\?          -- psql command help

-- Quit
\q
```

#### SQL Information Queries
```sql
-- Current database
SELECT current_database();

-- Current user
SELECT current_user;

-- PostgreSQL version
SELECT version();

-- Current date and time
SELECT now();

-- List all databases
SELECT datname FROM pg_database;

-- List all tables in current database
SELECT tablename FROM pg_tables WHERE schemaname = 'public';
```

### Understanding PostgreSQL Structure

```
PostgreSQL Instance (Cluster)
├── Database 1
│   ├── Schema (public)
│   │   ├── Tables
│   │   ├── Views
│   │   ├── Functions
│   │   └── Indexes
│   └── Schema (custom)
├── Database 2
└── Database 3
```

---

## Your First Database

### Step 1: Create a Database
```sql
-- Create a new database
CREATE DATABASE my_first_db;

-- Connect to the new database
\c my_first_db
```

### Step 2: Create Your First Table
```sql
-- Create a simple users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Step 3: Insert Data
```sql
-- Insert some sample data
INSERT INTO users (username, email) VALUES 
    ('john_doe', 'john@example.com'),
    ('jane_smith', 'jane@example.com'),
    ('bob_wilson', 'bob@example.com');
```

### Step 4: Query Data
```sql
-- Select all users
SELECT * FROM users;

-- Select specific columns
SELECT username, email FROM users;

-- Select with condition
SELECT * FROM users WHERE username = 'john_doe';

-- Count total users
SELECT COUNT(*) FROM users;
```

### Step 5: Update and Delete
```sql
-- Update a user's email
UPDATE users 
SET email = 'john.doe@example.com' 
WHERE username = 'john_doe';

-- Delete a user
DELETE FROM users WHERE username = 'bob_wilson';

-- Verify changes
SELECT * FROM users;
```

---

## Hands-On Exercises

### Exercise 1: Database Creation
1. Create a database named `bookstore`
2. Connect to the bookstore database
3. List all databases to confirm creation

```sql
-- Your solution here
CREATE DATABASE bookstore;
\c bookstore
\l
```

### Exercise 2: Table Creation
Create a `books` table with the following structure:
- `id`: Auto-incrementing primary key
- `title`: Book title (up to 200 characters)
- `author`: Author name (up to 100 characters)
- `isbn`: ISBN number (13 characters)
- `price`: Book price (decimal)
- `publication_date`: Date of publication

```sql
-- Your solution here
CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    author VARCHAR(100) NOT NULL,
    isbn CHAR(13) UNIQUE,
    price DECIMAL(10,2),
    publication_date DATE
);
```

### Exercise 3: Data Manipulation
1. Insert 5 books into the books table
2. Query all books
3. Find books by a specific author
4. Update the price of one book
5. Delete one book

```sql
-- Insert sample data
INSERT INTO books (title, author, isbn, price, publication_date) VALUES
    ('The Great Gatsby', 'F. Scott Fitzgerald', '9780743273565', 12.99, '1925-04-10'),
    ('To Kill a Mockingbird', 'Harper Lee', '9780446310789', 14.99, '1960-07-11'),
    ('1984', 'George Orwell', '9780451524935', 13.99, '1949-06-08'),
    ('Pride and Prejudice', 'Jane Austen', '9780141439518', 11.99, '1813-01-28'),
    ('The Catcher in the Rye', 'J.D. Salinger', '9780316769174', 15.99, '1951-07-16');

-- Query all books
SELECT * FROM books;

-- Find books by George Orwell
SELECT * FROM books WHERE author = 'George Orwell';

-- Update price of The Great Gatsby
UPDATE books SET price = 10.99 WHERE title = 'The Great Gatsby';

-- Delete Pride and Prejudice
DELETE FROM books WHERE title = 'Pride and Prejudice';
```

---

## Common Issues

### Connection Problems

#### Issue: "psql: FATAL: role 'username' does not exist"
```sql
-- Solution: Create the user or connect as correct user
sudo -u postgres createuser --interactive

-- Or connect as postgres user
psql -U postgres
```

#### Issue: "psql: FATAL: database 'username' does not exist"
```bash
# Solution: Specify database explicitly
psql -U postgres -d postgres

# Or create the database first
createdb my_database
```

#### Issue: "psql: FATAL: password authentication failed"
```bash
# Solution: Reset password or check pg_hba.conf
sudo -u postgres psql
ALTER USER postgres PASSWORD 'newpassword';
```

### Installation Issues

#### Issue: PostgreSQL service not starting
```bash
# Check service status
sudo systemctl status postgresql

# Check logs
sudo journalctl -u postgresql

# Restart service
sudo systemctl restart postgresql
```

#### Issue: Port 5432 already in use
```bash
# Find process using port
sudo netstat -tlnp | grep 5432

# Kill process or use different port
sudo kill -9 <process_id>
```

### Permission Issues

#### Issue: "permission denied for table"
```sql
-- Solution: Grant proper permissions
GRANT ALL PRIVILEGES ON TABLE table_name TO username;

-- Or grant all privileges on database
GRANT ALL PRIVILEGES ON DATABASE database_name TO username;
```

---

## Next Steps

### Immediate Next Steps
1. **Complete exercises** in this module
2. **Explore pgAdmin** or your preferred GUI tool
3. **Practice basic SQL** commands daily
4. **Move to Module 2**: [Database Fundamentals](02-database-fundamentals.md)

### Additional Practice
- Create multiple databases and tables
- Experiment with different data types
- Try importing data from CSV files
- Explore PostgreSQL documentation

### Recommended Reading
- [PostgreSQL Official Tutorial](https://postgresql.org/docs/current/tutorial.html)
- [PostgreSQL Documentation](https://postgresql.org/docs/)
- [psql Documentation](https://postgresql.org/docs/current/app-psql.html)

### Tools to Explore
- **pgAdmin**: Web-based administration tool
- **DBeaver**: Universal database tool
- **DataGrip**: JetBrains database IDE
- **Postico**: macOS PostgreSQL client

---

## Module Summary

✅ **What You've Learned:**
- PostgreSQL basics and advantages
- Installation on multiple platforms
- Connection methods and tools
- Basic psql commands
- Creating databases and tables
- Basic CRUD operations

✅ **Skills Acquired:**
- PostgreSQL installation and setup
- Database connection
- Basic SQL commands
- Table creation and data manipulation
- Navigation using psql

✅ **Ready for Next Module:**
You're now ready to dive deeper into database fundamentals in [Module 2: Database Fundamentals](02-database-fundamentals.md).

---

## Quick Reference

### Essential Commands
```sql
-- Connect to database
\c database_name

-- List databases
\l

-- List tables
\dt

-- Describe table
\d table_name

-- Create database
CREATE DATABASE name;

-- Create table
CREATE TABLE name (column type constraints);

-- Insert data
INSERT INTO table (columns) VALUES (values);

-- Select data
SELECT columns FROM table WHERE condition;

-- Update data
UPDATE table SET column = value WHERE condition;

-- Delete data
DELETE FROM table WHERE condition;
```

### Connection String Format
```
postgresql://[user[:password]@][netloc][:port][/dbname][?param1=value1&...]
```

---
*Continue your PostgreSQL journey with [Database Fundamentals](02-database-fundamentals.md) →*
