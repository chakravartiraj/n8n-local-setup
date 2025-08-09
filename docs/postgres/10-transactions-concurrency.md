# Transactions and Concurrency

## 🎯 Learning Objectives
By the end of this module, you will:
- Understand ACID properties and transaction isolation levels
- Master transaction control and error handling
- Handle concurrent access and locking mechanisms
- Implement optimistic and pessimistic concurrency control
- Prevent and resolve deadlocks
- Optimize for high-concurrency scenarios

## 📚 Table of Contents
1. [ACID Properties](#acid-properties)
2. [Transaction Basics](#transaction-basics)
3. [Isolation Levels](#isolation-levels)
4. [Locking Mechanisms](#locking-mechanisms)
5. [Concurrency Control](#concurrency-control)
6. [Deadlock Prevention](#deadlock-prevention)
7. [Performance Optimization](#performance-optimization)
8. [Hands-On Exercises](#hands-on-exercises)
9. [Best Practices](#best-practices)
10. [Next Steps](#next-steps)

---

## ACID Properties

### Understanding ACID

#### Atomicity
Transactions are all-or-nothing operations.

```sql
-- Example: Bank transfer demonstrating atomicity
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE NOT NULL,
    balance DECIMAL(15,2) NOT NULL CHECK (balance >= 0),
    account_type VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO accounts (account_number, balance, account_type) VALUES
('ACC001', 5000.00, 'checking'),
('ACC002', 2500.00, 'savings');

-- Function demonstrating atomic transaction
CREATE OR REPLACE FUNCTION transfer_funds(
    from_account VARCHAR(20),
    to_account VARCHAR(20),
    amount DECIMAL(15,2)
) RETURNS BOOLEAN AS $$
DECLARE
    from_balance DECIMAL(15,2);
BEGIN
    -- Start transaction is implicit in function
    
    -- Check and lock source account
    SELECT balance INTO from_balance
    FROM accounts 
    WHERE account_number = from_account
    FOR UPDATE;
    
    -- Verify sufficient funds
    IF from_balance < amount THEN
        RAISE EXCEPTION 'Insufficient funds. Available: %, Requested: %', from_balance, amount;
    END IF;
    
    -- Debit source account
    UPDATE accounts 
    SET balance = balance - amount
    WHERE account_number = from_account;
    
    -- Credit destination account
    UPDATE accounts 
    SET balance = balance + amount
    WHERE account_number = to_account;
    
    -- Log the transaction
    INSERT INTO transaction_log (from_account, to_account, amount, transaction_date)
    VALUES (from_account, to_account, amount, NOW());
    
    RETURN TRUE;
    
EXCEPTION
    WHEN OTHERS THEN
        -- Transaction automatically rolled back on exception
        RAISE NOTICE 'Transfer failed: %', SQLERRM;
        RETURN FALSE;
END;
$$ LANGUAGE plpgsql;

-- Create transaction log table
CREATE TABLE transaction_log (
    id SERIAL PRIMARY KEY,
    from_account VARCHAR(20),
    to_account VARCHAR(20),
    amount DECIMAL(15,2),
    transaction_date TIMESTAMP,
    status VARCHAR(20) DEFAULT 'completed'
);

-- Test atomic transaction
SELECT transfer_funds('ACC001', 'ACC002', 1000.00);
SELECT * FROM accounts;
```

#### Consistency
Database remains in a valid state before and after transaction.

```sql
-- Consistency through constraints and triggers
CREATE TABLE inventory (
    id SERIAL PRIMARY KEY,
    product_id INTEGER NOT NULL,
    warehouse_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity >= 0),
    reserved_quantity INTEGER DEFAULT 0 CHECK (reserved_quantity >= 0),
    CONSTRAINT chk_reserved_not_exceed_quantity 
        CHECK (reserved_quantity <= quantity)
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    status VARCHAR(20) DEFAULT 'pending',
    order_date TIMESTAMP DEFAULT NOW()
);

-- Trigger to maintain inventory consistency
CREATE OR REPLACE FUNCTION maintain_inventory_consistency()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' AND NEW.status = 'confirmed' THEN
        -- Reserve inventory for new confirmed order
        UPDATE inventory 
        SET reserved_quantity = reserved_quantity + NEW.quantity
        WHERE product_id = NEW.product_id;
        
        -- Check if we have sufficient inventory
        IF NOT FOUND OR 
           (SELECT quantity - reserved_quantity FROM inventory WHERE product_id = NEW.product_id) < 0 THEN
            RAISE EXCEPTION 'Insufficient inventory for product %', NEW.product_id;
        END IF;
        
    ELSIF TG_OP = 'UPDATE' THEN
        -- Handle status changes
        IF OLD.status = 'pending' AND NEW.status = 'confirmed' THEN
            UPDATE inventory 
            SET reserved_quantity = reserved_quantity + NEW.quantity
            WHERE product_id = NEW.product_id;
            
        ELSIF OLD.status = 'confirmed' AND NEW.status = 'shipped' THEN
            UPDATE inventory 
            SET quantity = quantity - NEW.quantity,
                reserved_quantity = reserved_quantity - NEW.quantity
            WHERE product_id = NEW.product_id;
            
        ELSIF OLD.status = 'confirmed' AND NEW.status = 'cancelled' THEN
            UPDATE inventory 
            SET reserved_quantity = reserved_quantity - NEW.quantity
            WHERE product_id = NEW.product_id;
        END IF;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER inventory_consistency_trigger
    AFTER INSERT OR UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION maintain_inventory_consistency();
```

#### Isolation
Concurrent transactions don't interfere with each other.

```sql
-- Demonstration of isolation levels (run in separate sessions)

-- Session 1:
BEGIN ISOLATION LEVEL READ COMMITTED;
UPDATE accounts SET balance = balance + 100 WHERE account_number = 'ACC001';
-- Don't commit yet

-- Session 2 (different connection):
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM accounts WHERE account_number = 'ACC001';
-- Will see old value until Session 1 commits

-- Session 1:
COMMIT;

-- Session 2:
SELECT balance FROM accounts WHERE account_number = 'ACC001';
-- Now sees updated value
COMMIT;
```

#### Durability
Committed changes persist even after system failure.

```sql
-- PostgreSQL ensures durability through:
-- 1. Write-Ahead Logging (WAL)
-- 2. Checkpoints
-- 3. Synchronous commits

-- Check WAL settings
SHOW wal_level;
SHOW synchronous_commit;
SHOW fsync;

-- Force WAL flush (for testing)
SELECT pg_current_wal_lsn(), pg_current_wal_insert_lsn();
```

---

## Transaction Basics

### Manual Transaction Control

#### Basic Transaction Commands
```sql
-- Explicit transaction control
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE account_number = 'ACC001';
    UPDATE accounts SET balance = balance + 500 WHERE account_number = 'ACC002';
    
    -- Check balances before committing
    SELECT account_number, balance FROM accounts WHERE account_number IN ('ACC001', 'ACC002');
COMMIT;

-- Transaction with rollback
BEGIN;
    UPDATE accounts SET balance = balance - 10000 WHERE account_number = 'ACC001';
    
    -- Realize this is too much, roll back
ROLLBACK;

-- Transaction with savepoints
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE account_number = 'ACC001';
    SAVEPOINT sp1;
    
    UPDATE accounts SET balance = balance - 200 WHERE account_number = 'ACC001';
    SAVEPOINT sp2;
    
    UPDATE accounts SET balance = balance - 300 WHERE account_number = 'ACC001';
    
    -- Rollback to savepoint
    ROLLBACK TO sp2;
    
    -- Continue with transaction
    UPDATE accounts SET balance = balance + 50 WHERE account_number = 'ACC002';
COMMIT;
```

#### Transaction in Functions
```sql
-- Function with transaction control
CREATE OR REPLACE FUNCTION process_batch_orders(order_ids INTEGER[])
RETURNS TABLE(order_id INTEGER, status TEXT, message TEXT) AS $$
DECLARE
    current_order_id INTEGER;
    order_rec RECORD;
BEGIN
    FOREACH current_order_id IN ARRAY order_ids LOOP
        BEGIN
            -- Start a nested transaction for each order
            
            SELECT * INTO order_rec FROM orders WHERE id = current_order_id;
            
            IF NOT FOUND THEN
                RETURN QUERY SELECT current_order_id, 'failed'::TEXT, 'Order not found'::TEXT;
                CONTINUE;
            END IF;
            
            -- Process the order
            UPDATE orders 
            SET status = 'processing', 
                updated_at = NOW()
            WHERE id = current_order_id;
            
            -- Simulate some processing
            PERFORM pg_sleep(0.1);
            
            UPDATE orders 
            SET status = 'completed',
                updated_at = NOW()
            WHERE id = current_order_id;
            
            RETURN QUERY SELECT current_order_id, 'success'::TEXT, 'Order processed'::TEXT;
            
        EXCEPTION
            WHEN OTHERS THEN
                -- Individual order failure doesn't affect others
                RETURN QUERY SELECT current_order_id, 'failed'::TEXT, SQLERRM::TEXT;
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

---

## Isolation Levels

### PostgreSQL Isolation Levels

#### Read Uncommitted
```sql
-- Read Uncommitted (allows dirty reads)
-- Session 1:
BEGIN ISOLATION LEVEL READ UNCOMMITTED;
UPDATE accounts SET balance = balance + 1000 WHERE account_number = 'ACC001';
-- Don't commit yet

-- Session 2:
BEGIN ISOLATION LEVEL READ UNCOMMITTED;
SELECT balance FROM accounts WHERE account_number = 'ACC001';
-- May see uncommitted change (dirty read)
COMMIT;

-- Session 1:
ROLLBACK; -- The change Session 2 saw is now gone
```

#### Read Committed (Default)
```sql
-- Read Committed (prevents dirty reads)
-- Session 1:
BEGIN ISOLATION LEVEL READ COMMITTED;
UPDATE accounts SET balance = 1000 WHERE account_number = 'ACC001';
-- Don't commit yet

-- Session 2:
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM accounts WHERE account_number = 'ACC001';
-- Sees old value (no dirty read)

-- Session 1:
COMMIT;

-- Session 2:
SELECT balance FROM accounts WHERE account_number = 'ACC001';
-- Now sees new value (non-repeatable read)
COMMIT;
```

#### Repeatable Read
```sql
-- Repeatable Read (prevents dirty and non-repeatable reads)
-- Session 1:
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE account_number = 'ACC001';

-- Session 2:
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE accounts SET balance = balance + 500 WHERE account_number = 'ACC001';
COMMIT;

-- Session 1:
SELECT balance FROM accounts WHERE account_number = 'ACC001';
-- Still sees original value (repeatable read)

-- Try to update
UPDATE accounts SET balance = balance + 100 WHERE account_number = 'ACC001';
-- ERROR: could not serialize access due to concurrent update
ROLLBACK;
```

#### Serializable
```sql
-- Serializable (strongest isolation)
-- Prevents all phenomena including phantom reads

-- Create test scenario for phantom reads
CREATE TABLE products_temp AS SELECT * FROM products WHERE false; -- Empty copy

-- Session 1:
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*) FROM products_temp WHERE category = 'electronics';

-- Session 2:
BEGIN ISOLATION LEVEL SERIALIZABLE;
INSERT INTO products_temp (name, category, price) VALUES ('New Phone', 'electronics', 999.99);
COMMIT;

-- Session 1:
SELECT COUNT(*) FROM products_temp WHERE category = 'electronics';
-- Still sees same count (no phantom read)

INSERT INTO products_temp (name, category, price) VALUES ('Another Phone', 'electronics', 799.99);
-- ERROR: could not serialize access due to read/write dependencies
ROLLBACK;
```

### Choosing Isolation Levels

#### Isolation Level Selection Guidelines
```sql
-- Function to demonstrate isolation level selection
CREATE OR REPLACE FUNCTION select_isolation_level(operation_type TEXT)
RETURNS TEXT AS $$
BEGIN
    CASE operation_type
        WHEN 'financial_transfer' THEN
            -- Use SERIALIZABLE for critical financial operations
            SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
            RETURN 'SERIALIZABLE - Maximum data integrity for financial operations';
            
        WHEN 'inventory_update' THEN
            -- Use REPEATABLE READ for inventory management
            SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
            RETURN 'REPEATABLE READ - Consistent inventory view';
            
        WHEN 'reporting' THEN
            -- Use READ COMMITTED for most reporting (default)
            SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
            RETURN 'READ COMMITTED - Good for reporting with acceptable read phenomena';
            
        WHEN 'logging' THEN
            -- Use READ UNCOMMITTED for non-critical logging (rarely used)
            SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
            RETURN 'READ UNCOMMITTED - Fastest but allows dirty reads';
            
        ELSE
            RETURN 'READ COMMITTED - Default isolation level';
    END CASE;
END;
$$ LANGUAGE plpgsql;
```

---

## Locking Mechanisms

### Row-Level Locking

#### FOR UPDATE and Variants
```sql
-- Demonstrate different lock types

-- FOR UPDATE - Exclusive lock
-- Session 1:
BEGIN;
SELECT * FROM accounts WHERE account_number = 'ACC001' FOR UPDATE;

-- Session 2:
BEGIN;
SELECT * FROM accounts WHERE account_number = 'ACC001' FOR UPDATE;
-- Blocks until Session 1 commits/rollbacks

-- FOR SHARE - Shared lock
-- Session 1:
BEGIN;
SELECT * FROM accounts WHERE account_number = 'ACC001' FOR SHARE;

-- Session 2:
BEGIN;
SELECT * FROM accounts WHERE account_number = 'ACC001' FOR SHARE;
-- Both succeed (shared locks compatible)

UPDATE accounts SET balance = balance + 100 WHERE account_number = 'ACC001';
-- Blocks because of shared lock from Session 1

-- FOR UPDATE NOWAIT - Don't wait for locks
BEGIN;
SELECT * FROM accounts WHERE account_number = 'ACC001' FOR UPDATE NOWAIT;
-- ERROR if row is locked

-- FOR UPDATE SKIP LOCKED - Skip locked rows
SELECT * FROM accounts WHERE id BETWEEN 1 AND 10 FOR UPDATE SKIP LOCKED;
-- Returns only unlocked rows
```

#### Implementing a Job Queue
```sql
-- Job queue with proper locking
CREATE TABLE job_queue (
    id SERIAL PRIMARY KEY,
    job_type VARCHAR(50) NOT NULL,
    job_data JSONB,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    priority INTEGER DEFAULT 5,
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    worker_id VARCHAR(50),
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3
);

-- Function to claim next job
CREATE OR REPLACE FUNCTION claim_next_job(worker_id_param VARCHAR(50))
RETURNS TABLE(
    job_id INTEGER,
    job_type VARCHAR(50),
    job_data JSONB,
    priority INTEGER
) AS $$
DECLARE
    claimed_job RECORD;
BEGIN
    -- Find and lock the highest priority pending job
    SELECT id, job_type, job_data, priority
    INTO claimed_job
    FROM job_queue
    WHERE status = 'pending'
      AND retry_count < max_retries
    ORDER BY priority DESC, created_at ASC
    FOR UPDATE SKIP LOCKED
    LIMIT 1;
    
    IF FOUND THEN
        -- Mark job as processing
        UPDATE job_queue
        SET status = 'processing',
            started_at = NOW(),
            worker_id = worker_id_param
        WHERE id = claimed_job.id;
        
        -- Return job details
        RETURN QUERY SELECT claimed_job.id, claimed_job.job_type, 
                           claimed_job.job_data, claimed_job.priority;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Function to complete job
CREATE OR REPLACE FUNCTION complete_job(job_id_param INTEGER, success BOOLEAN)
RETURNS VOID AS $$
BEGIN
    IF success THEN
        UPDATE job_queue
        SET status = 'completed',
            completed_at = NOW()
        WHERE id = job_id_param;
    ELSE
        UPDATE job_queue
        SET status = CASE 
                        WHEN retry_count + 1 >= max_retries THEN 'failed'
                        ELSE 'pending'
                    END,
            retry_count = retry_count + 1,
            started_at = NULL,
            worker_id = NULL
        WHERE id = job_id_param;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### Table-Level Locking

#### Advisory Locks
```sql
-- Advisory locks for application-level coordination
CREATE OR REPLACE FUNCTION process_critical_section(lock_id BIGINT)
RETURNS TEXT AS $$
DECLARE
    lock_acquired BOOLEAN;
BEGIN
    -- Try to acquire advisory lock
    SELECT pg_try_advisory_lock(lock_id) INTO lock_acquired;
    
    IF NOT lock_acquired THEN
        RETURN 'Could not acquire lock - another process is running';
    END IF;
    
    -- Critical section - only one process can execute this
    PERFORM pg_sleep(2); -- Simulate work
    
    -- Release the lock
    PERFORM pg_advisory_unlock(lock_id);
    
    RETURN 'Critical section completed successfully';
END;
$$ LANGUAGE plpgsql;

-- Session-level advisory locks
SELECT pg_advisory_lock(12345); -- Blocks until acquired
SELECT pg_advisory_unlock(12345); -- Release lock

-- Transaction-level advisory locks  
BEGIN;
SELECT pg_advisory_xact_lock(12345); -- Released automatically on commit/rollback
-- ... do work ...
COMMIT; -- Lock automatically released
```

---

## Concurrency Control

### Optimistic Concurrency Control

#### Version-Based Optimistic Locking
```sql
-- Table with version column for optimistic locking
CREATE TABLE products_versioned (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    description TEXT,
    version INTEGER DEFAULT 1,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Function for optimistic update
CREATE OR REPLACE FUNCTION update_product_optimistic(
    product_id INTEGER,
    new_name VARCHAR(200),
    new_price DECIMAL(10,2),
    expected_version INTEGER
) RETURNS BOOLEAN AS $$
DECLARE
    updated_rows INTEGER;
BEGIN
    UPDATE products_versioned
    SET name = new_name,
        price = new_price,
        version = version + 1,
        updated_at = NOW()
    WHERE id = product_id 
      AND version = expected_version;
    
    GET DIAGNOSTICS updated_rows = ROW_COUNT;
    
    IF updated_rows = 0 THEN
        -- Either product doesn't exist or version mismatch
        IF EXISTS (SELECT 1 FROM products_versioned WHERE id = product_id) THEN
            RAISE EXCEPTION 'Version conflict: Product was modified by another user';
        ELSE
            RAISE EXCEPTION 'Product not found';
        END IF;
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Timestamp-based optimistic locking
CREATE TABLE documents_timestamp (
    id SERIAL PRIMARY KEY,
    title VARCHAR(300),
    content TEXT,
    last_modified TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION update_document_timestamp(
    doc_id INTEGER,
    new_title VARCHAR(300),
    new_content TEXT,
    expected_timestamp TIMESTAMP
) RETURNS BOOLEAN AS $$
DECLARE
    current_timestamp TIMESTAMP;
BEGIN
    SELECT last_modified INTO current_timestamp
    FROM documents_timestamp
    WHERE id = doc_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Document not found';
    END IF;
    
    IF current_timestamp != expected_timestamp THEN
        RAISE EXCEPTION 'Document was modified by another user at %', current_timestamp;
    END IF;
    
    UPDATE documents_timestamp
    SET title = new_title,
        content = new_content,
        last_modified = NOW()
    WHERE id = doc_id;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

### Pessimistic Concurrency Control

#### Explicit Locking Strategies
```sql
-- Pessimistic locking with timeout
CREATE OR REPLACE FUNCTION update_account_pessimistic(
    account_num VARCHAR(20),
    amount DECIMAL(15,2)
) RETURNS BOOLEAN AS $$
DECLARE
    current_balance DECIMAL(15,2);
    lock_timeout CONSTANT INTEGER := 10; -- seconds
BEGIN
    -- Set lock timeout
    EXECUTE format('SET LOCAL lock_timeout = %s', lock_timeout * 1000);
    
    -- Acquire exclusive lock on account
    SELECT balance INTO current_balance
    FROM accounts
    WHERE account_number = account_num
    FOR UPDATE;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Account not found: %', account_num;
    END IF;
    
    -- Verify business rules
    IF current_balance + amount < 0 THEN
        RAISE EXCEPTION 'Insufficient funds';
    END IF;
    
    -- Update account
    UPDATE accounts
    SET balance = balance + amount,
        updated_at = NOW()
    WHERE account_number = account_num;
    
    RETURN TRUE;
    
EXCEPTION
    WHEN lock_not_available THEN
        RAISE EXCEPTION 'Could not acquire lock on account within % seconds', lock_timeout;
END;
$$ LANGUAGE plpgsql;
```

---

## Deadlock Prevention

### Understanding Deadlocks

#### Deadlock Detection and Resolution
```sql
-- Create scenario that can cause deadlocks
CREATE TABLE table_a (id INTEGER PRIMARY KEY, value TEXT);
CREATE TABLE table_b (id INTEGER PRIMARY KEY, value TEXT);

INSERT INTO table_a VALUES (1, 'A1'), (2, 'A2');
INSERT INTO table_b VALUES (1, 'B1'), (2, 'B2');

-- Potential deadlock scenario:
-- Session 1:
BEGIN;
UPDATE table_a SET value = 'A1-updated' WHERE id = 1;
-- Now try to update table_b
UPDATE table_b SET value = 'B1-updated' WHERE id = 1;

-- Session 2 (at the same time):
BEGIN;
UPDATE table_b SET value = 'B1-session2' WHERE id = 1;
-- Now try to update table_a
UPDATE table_a SET value = 'A1-session2' WHERE id = 1; -- DEADLOCK!

-- PostgreSQL will detect and resolve by aborting one transaction
```

#### Deadlock Prevention Strategies
```sql
-- Strategy 1: Consistent ordering of locks
CREATE OR REPLACE FUNCTION transfer_with_ordered_locks(
    from_account VARCHAR(20),
    to_account VARCHAR(20),
    amount DECIMAL(15,2)
) RETURNS BOOLEAN AS $$
DECLARE
    first_account VARCHAR(20);
    second_account VARCHAR(20);
    from_balance DECIMAL(15,2);
BEGIN
    -- Always lock accounts in alphabetical order to prevent deadlocks
    IF from_account < to_account THEN
        first_account := from_account;
        second_account := to_account;
    ELSE
        first_account := to_account;
        second_account := from_account;
    END IF;
    
    -- Lock accounts in consistent order
    SELECT balance INTO from_balance
    FROM accounts 
    WHERE account_number = first_account
    FOR UPDATE;
    
    SELECT balance INTO STRICT from_balance
    FROM accounts 
    WHERE account_number = second_account
    FOR UPDATE;
    
    -- Now perform the actual transfer
    UPDATE accounts SET balance = balance - amount WHERE account_number = from_account;
    UPDATE accounts SET balance = balance + amount WHERE account_number = to_account;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Strategy 2: Lock timeout to break potential deadlocks
CREATE OR REPLACE FUNCTION safe_multi_table_update()
RETURNS VOID AS $$
BEGIN
    SET LOCAL lock_timeout = '5s';
    
    -- Perform updates that might conflict
    UPDATE table_a SET value = 'safe update' WHERE id = 1;
    UPDATE table_b SET value = 'safe update' WHERE id = 1;
    
EXCEPTION
    WHEN lock_not_available THEN
        RAISE NOTICE 'Lock timeout - operation aborted to prevent deadlock';
        RAISE;
END;
$$ LANGUAGE plpgsql;

-- Strategy 3: Retry logic for deadlock handling
CREATE OR REPLACE FUNCTION retry_on_deadlock(
    max_retries INTEGER DEFAULT 3
) RETURNS BOOLEAN AS $$
DECLARE
    retry_count INTEGER := 0;
    success BOOLEAN := FALSE;
BEGIN
    WHILE retry_count < max_retries AND NOT success LOOP
        BEGIN
            -- Perform operation that might deadlock
            UPDATE accounts SET balance = balance + 10 WHERE account_number = 'ACC001';
            UPDATE accounts SET balance = balance - 10 WHERE account_number = 'ACC002';
            
            success := TRUE;
            
        EXCEPTION
            WHEN deadlock_detected THEN
                retry_count := retry_count + 1;
                RAISE NOTICE 'Deadlock detected, retry % of %', retry_count, max_retries;
                
                IF retry_count >= max_retries THEN
                    RAISE EXCEPTION 'Max retries exceeded due to deadlocks';
                END IF;
                
                -- Wait a random amount before retry
                PERFORM pg_sleep(random() * 0.1);
        END;
    END LOOP;
    
    RETURN success;
END;
$$ LANGUAGE plpgsql;
```

### Monitoring Deadlocks

#### Deadlock Logging and Analysis
```sql
-- Enable deadlock logging (requires superuser)
-- ALTER SYSTEM SET log_lock_waits = on;
-- ALTER SYSTEM SET deadlock_timeout = '1s';
-- SELECT pg_reload_conf();

-- View current locks
SELECT 
    pid,
    usename,
    application_name,
    state,
    query,
    locktype,
    mode,
    granted
FROM pg_stat_activity psa
JOIN pg_locks pl ON psa.pid = pl.pid
WHERE NOT granted
ORDER BY pid;

-- Function to check for potential deadlocks
CREATE OR REPLACE FUNCTION check_lock_conflicts()
RETURNS TABLE(
    blocking_pid INTEGER,
    blocked_pid INTEGER,
    blocking_query TEXT,
    blocked_query TEXT,
    lock_type TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        kl.pid AS blocking_pid,
        bl.pid AS blocked_pid,
        ka.query AS blocking_query,
        ba.query AS blocked_query,
        bl.locktype AS lock_type
    FROM pg_locks bl
    JOIN pg_stat_activity ba ON bl.pid = ba.pid
    JOIN pg_locks kl ON bl.locktype = kl.locktype
        AND bl.database IS NOT DISTINCT FROM kl.database
        AND bl.relation IS NOT DISTINCT FROM kl.relation
        AND bl.page IS NOT DISTINCT FROM kl.page
        AND bl.tuple IS NOT DISTINCT FROM kl.tuple
        AND bl.virtualxid IS NOT DISTINCT FROM kl.virtualxid
        AND bl.transactionid IS NOT DISTINCT FROM kl.transactionid
        AND bl.classid IS NOT DISTINCT FROM kl.classid
        AND bl.objid IS NOT DISTINCT FROM kl.objid
        AND bl.objsubid IS NOT DISTINCT FROM kl.objsubid
        AND bl.pid != kl.pid
    JOIN pg_stat_activity ka ON kl.pid = ka.pid
    WHERE NOT bl.granted
      AND kl.granted;
END;
$$ LANGUAGE plpgsql;
```

---

## Performance Optimization

### Connection Pooling

#### Connection Pool Configuration
```sql
-- Monitor connection usage
SELECT 
    COUNT(*) AS total_connections,
    COUNT(*) FILTER (WHERE state = 'active') AS active_connections,
    COUNT(*) FILTER (WHERE state = 'idle') AS idle_connections,
    COUNT(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_transaction
FROM pg_stat_activity
WHERE pid != pg_backend_pid();

-- Identify long-running transactions
SELECT 
    pid,
    usename,
    application_name,
    state,
    xact_start,
    EXTRACT(EPOCH FROM (NOW() - xact_start)) AS transaction_duration_seconds,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND state != 'idle'
ORDER BY xact_start;

-- Function to terminate long-running transactions
CREATE OR REPLACE FUNCTION terminate_long_transactions(
    max_duration_seconds INTEGER DEFAULT 3600
) RETURNS TABLE(
    terminated_pid INTEGER,
    duration_seconds NUMERIC,
    query_text TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        psa.pid,
        EXTRACT(EPOCH FROM (NOW() - psa.xact_start)),
        psa.query
    FROM pg_stat_activity psa
    WHERE psa.xact_start IS NOT NULL
      AND EXTRACT(EPOCH FROM (NOW() - psa.xact_start)) > max_duration_seconds
      AND psa.pid != pg_backend_pid()
      AND pg_terminate_backend(psa.pid);
END;
$$ LANGUAGE plpgsql;
```

### Transaction Optimization

#### Batch Processing Techniques
```sql
-- Efficient batch processing
CREATE OR REPLACE FUNCTION process_batch_efficiently(
    batch_size INTEGER DEFAULT 1000
) RETURNS TABLE(
    batch_number INTEGER,
    processed_count INTEGER,
    processing_time INTERVAL
) AS $$
DECLARE
    start_time TIMESTAMP;
    batch_count INTEGER := 0;
    total_processed INTEGER := 0;
    current_batch_size INTEGER;
BEGIN
    LOOP
        start_time := clock_timestamp();
        
        -- Process one batch
        WITH batch_data AS (
            SELECT id 
            FROM pending_records 
            WHERE status = 'pending'
            ORDER BY created_at
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        )
        UPDATE pending_records 
        SET status = 'processed',
            processed_at = NOW()
        FROM batch_data
        WHERE pending_records.id = batch_data.id;
        
        GET DIAGNOSTICS current_batch_size = ROW_COUNT;
        
        IF current_batch_size = 0 THEN
            EXIT; -- No more records to process
        END IF;
        
        batch_count := batch_count + 1;
        total_processed := total_processed + current_batch_size;
        
        RETURN QUERY SELECT 
            batch_count,
            current_batch_size,
            clock_timestamp() - start_time;
        
        -- Commit transaction to release locks
        COMMIT;
        
        -- Small pause between batches
        PERFORM pg_sleep(0.01);
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

---

## Hands-On Exercises

### Exercise 1: Banking System with Concurrency

Create a robust banking system that handles concurrent operations safely.

```sql
-- Solution: Complete banking system

-- Enhanced accounts table
CREATE TABLE bank_accounts (
    id SERIAL PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE NOT NULL,
    customer_name VARCHAR(100) NOT NULL,
    balance DECIMAL(15,2) NOT NULL DEFAULT 0 CHECK (balance >= 0),
    account_type VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'frozen', 'closed')),
    overdraft_limit DECIMAL(15,2) DEFAULT 0,
    version INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Transaction history
CREATE TABLE bank_transactions (
    id SERIAL PRIMARY KEY,
    transaction_id UUID DEFAULT gen_random_uuid(),
    from_account VARCHAR(20),
    to_account VARCHAR(20),
    transaction_type VARCHAR(20) NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    balance_after DECIMAL(15,2),
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    processed_by VARCHAR(100)
);

-- Comprehensive transfer function
CREATE OR REPLACE FUNCTION bank_transfer(
    from_account_num VARCHAR(20),
    to_account_num VARCHAR(20),
    transfer_amount DECIMAL(15,2),
    description TEXT DEFAULT NULL,
    operator_id VARCHAR(100) DEFAULT USER
) RETURNS UUID AS $$
DECLARE
    transaction_uuid UUID;
    from_account_record RECORD;
    to_account_record RECORD;
    available_balance DECIMAL(15,2);
BEGIN
    -- Input validation
    IF transfer_amount <= 0 THEN
        RAISE EXCEPTION 'Transfer amount must be positive';
    END IF;
    
    IF from_account_num = to_account_num THEN
        RAISE EXCEPTION 'Cannot transfer to the same account';
    END IF;
    
    -- Generate transaction ID
    transaction_uuid := gen_random_uuid();
    
    -- Lock accounts in consistent order (alphabetical)
    IF from_account_num < to_account_num THEN
        SELECT * INTO from_account_record 
        FROM bank_accounts 
        WHERE account_number = from_account_num AND status = 'active'
        FOR UPDATE;
        
        SELECT * INTO to_account_record
        FROM bank_accounts 
        WHERE account_number = to_account_num AND status = 'active'
        FOR UPDATE;
    ELSE
        SELECT * INTO to_account_record
        FROM bank_accounts 
        WHERE account_number = to_account_num AND status = 'active'
        FOR UPDATE;
        
        SELECT * INTO from_account_record 
        FROM bank_accounts 
        WHERE account_number = from_account_num AND status = 'active'
        FOR UPDATE;
    END IF;
    
    -- Verify accounts exist
    IF from_account_record IS NULL THEN
        RAISE EXCEPTION 'Source account not found or inactive: %', from_account_num;
    END IF;
    
    IF to_account_record IS NULL THEN
        RAISE EXCEPTION 'Destination account not found or inactive: %', to_account_num;
    END IF;
    
    -- Calculate available balance including overdraft
    available_balance := from_account_record.balance + from_account_record.overdraft_limit;
    
    -- Check sufficient funds
    IF available_balance < transfer_amount THEN
        RAISE EXCEPTION 'Insufficient funds. Available: %, Requested: %', 
            available_balance, transfer_amount;
    END IF;
    
    -- Perform the transfer
    UPDATE bank_accounts 
    SET balance = balance - transfer_amount,
        version = version + 1,
        updated_at = NOW()
    WHERE account_number = from_account_num;
    
    UPDATE bank_accounts 
    SET balance = balance + transfer_amount,
        version = version + 1,
        updated_at = NOW()
    WHERE account_number = to_account_num;
    
    -- Record debit transaction
    INSERT INTO bank_transactions (
        transaction_id, from_account, to_account, transaction_type,
        amount, balance_after, description, processed_by
    ) VALUES (
        transaction_uuid, from_account_num, to_account_num, 'debit',
        transfer_amount, from_account_record.balance - transfer_amount,
        description, operator_id
    );
    
    -- Record credit transaction
    INSERT INTO bank_transactions (
        transaction_id, from_account, to_account, transaction_type,
        amount, balance_after, description, processed_by
    ) VALUES (
        transaction_uuid, from_account_num, to_account_num, 'credit',
        transfer_amount, to_account_record.balance + transfer_amount,
        description, operator_id
    );
    
    RETURN transaction_uuid;
    
EXCEPTION
    WHEN OTHERS THEN
        -- Log the error
        INSERT INTO bank_transactions (
            transaction_id, from_account, to_account, transaction_type,
            amount, description, processed_by
        ) VALUES (
            transaction_uuid, from_account_num, to_account_num, 'failed',
            transfer_amount, 'ERROR: ' || SQLERRM, operator_id
        );
        
        RAISE;
END;
$$ LANGUAGE plpgsql;

-- Test concurrent transfers
INSERT INTO bank_accounts (account_number, customer_name, balance, account_type, overdraft_limit) VALUES
('ACC001', 'Alice Johnson', 5000.00, 'checking', 1000.00),
('ACC002', 'Bob Smith', 3000.00, 'savings', 0.00),
('ACC003', 'Charlie Brown', 10000.00, 'checking', 2000.00);

-- Simulate concurrent transfers (run in multiple sessions)
SELECT bank_transfer('ACC001', 'ACC002', 500.00, 'Test transfer 1');
SELECT bank_transfer('ACC002', 'ACC003', 200.00, 'Test transfer 2');
SELECT bank_transfer('ACC003', 'ACC001', 1000.00, 'Test transfer 3');
```

### Exercise 2: High-Concurrency Inventory System

Build an inventory system that can handle many concurrent operations.

```sql
-- Solution: Concurrent inventory management

-- Products and inventory
CREATE TABLE products_inventory (
    id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    current_stock INTEGER NOT NULL DEFAULT 0 CHECK (current_stock >= 0),
    reserved_stock INTEGER NOT NULL DEFAULT 0 CHECK (reserved_stock >= 0),
    reorder_point INTEGER DEFAULT 10,
    max_stock INTEGER DEFAULT 1000,
    version INTEGER DEFAULT 1,
    CONSTRAINT stock_consistency CHECK (reserved_stock <= current_stock)
);

-- Stock movements tracking
CREATE TABLE stock_movements (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products_inventory(id),
    movement_type VARCHAR(20) NOT NULL, -- 'in', 'out', 'reserve', 'release'
    quantity INTEGER NOT NULL,
    reference_type VARCHAR(50), -- 'purchase', 'sale', 'adjustment', 'return'
    reference_id VARCHAR(50),
    reason TEXT,
    performed_by VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Reservations for pending orders
CREATE TABLE stock_reservations (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products_inventory(id),
    quantity INTEGER NOT NULL,
    reservation_type VARCHAR(50) DEFAULT 'order',
    reference_id VARCHAR(50) NOT NULL,
    expires_at TIMESTAMP DEFAULT (NOW() + INTERVAL '1 hour'),
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'fulfilled', 'expired', 'cancelled')),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Function to reserve stock
CREATE OR REPLACE FUNCTION reserve_stock(
    product_sku VARCHAR(50),
    quantity_needed INTEGER,
    reference_order_id VARCHAR(50),
    reservation_duration INTERVAL DEFAULT '1 hour'
) RETURNS INTEGER AS $$
DECLARE
    product_record RECORD;
    available_stock INTEGER;
    reservation_id INTEGER;
BEGIN
    -- Validate inputs
    IF quantity_needed <= 0 THEN
        RAISE EXCEPTION 'Quantity must be positive';
    END IF;
    
    -- Lock product for update
    SELECT * INTO product_record
    FROM products_inventory
    WHERE sku = product_sku
    FOR UPDATE;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Product not found: %', product_sku;
    END IF;
    
    -- Calculate available stock
    available_stock := product_record.current_stock - product_record.reserved_stock;
    
    IF available_stock < quantity_needed THEN
        RAISE EXCEPTION 'Insufficient stock. Available: %, Requested: %', 
            available_stock, quantity_needed;
    END IF;
    
    -- Create reservation
    INSERT INTO stock_reservations (
        product_id, quantity, reference_id, expires_at
    ) VALUES (
        product_record.id, quantity_needed, reference_order_id,
        NOW() + reservation_duration
    ) RETURNING id INTO reservation_id;
    
    -- Update reserved stock
    UPDATE products_inventory
    SET reserved_stock = reserved_stock + quantity_needed,
        version = version + 1
    WHERE id = product_record.id;
    
    -- Log movement
    INSERT INTO stock_movements (
        product_id, movement_type, quantity, reference_type, 
        reference_id, reason, performed_by
    ) VALUES (
        product_record.id, 'reserve', quantity_needed, 'order',
        reference_order_id, 'Stock reserved for order', USER
    );
    
    RETURN reservation_id;
END;
$$ LANGUAGE plpgsql;

-- Function to fulfill reservation
CREATE OR REPLACE FUNCTION fulfill_reservation(
    reservation_id INTEGER
) RETURNS BOOLEAN AS $$
DECLARE
    reservation_record RECORD;
    product_record RECORD;
BEGIN
    -- Get reservation details
    SELECT * INTO reservation_record
    FROM stock_reservations
    WHERE id = reservation_id AND status = 'active'
    FOR UPDATE;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Active reservation not found: %', reservation_id;
    END IF;
    
    -- Lock product
    SELECT * INTO product_record
    FROM products_inventory
    WHERE id = reservation_record.product_id
    FOR UPDATE;
    
    -- Update stock levels
    UPDATE products_inventory
    SET current_stock = current_stock - reservation_record.quantity,
        reserved_stock = reserved_stock - reservation_record.quantity,
        version = version + 1
    WHERE id = product_record.id;
    
    -- Mark reservation as fulfilled
    UPDATE stock_reservations
    SET status = 'fulfilled'
    WHERE id = reservation_id;
    
    -- Log movement
    INSERT INTO stock_movements (
        product_id, movement_type, quantity, reference_type,
        reference_id, reason, performed_by
    ) VALUES (
        product_record.id, 'out', reservation_record.quantity, 'sale',
        reservation_record.reference_id, 'Reservation fulfilled', USER
    );
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Function to clean up expired reservations
CREATE OR REPLACE FUNCTION cleanup_expired_reservations()
RETURNS TABLE(
    cleaned_product_id INTEGER,
    released_quantity INTEGER
) AS $$
DECLARE
    expired_reservation RECORD;
BEGIN
    -- Find and process expired reservations
    FOR expired_reservation IN
        SELECT * FROM stock_reservations
        WHERE status = 'active' AND expires_at < NOW()
        FOR UPDATE
    LOOP
        -- Release reserved stock
        UPDATE products_inventory
        SET reserved_stock = reserved_stock - expired_reservation.quantity,
            version = version + 1
        WHERE id = expired_reservation.product_id;
        
        -- Mark reservation as expired
        UPDATE stock_reservations
        SET status = 'expired'
        WHERE id = expired_reservation.id;
        
        -- Log the release
        INSERT INTO stock_movements (
            product_id, movement_type, quantity, reference_type,
            reference_id, reason, performed_by
        ) VALUES (
            expired_reservation.product_id, 'release', expired_reservation.quantity,
            'expiration', expired_reservation.reference_id,
            'Reservation expired', 'system'
        );
        
        RETURN QUERY SELECT expired_reservation.product_id, expired_reservation.quantity;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Test data and concurrent operations
INSERT INTO products_inventory (sku, name, current_stock, reorder_point) VALUES
('LAPTOP001', 'Gaming Laptop Pro', 50, 5),
('MOUSE001', 'Wireless Mouse', 200, 20),
('KEYBOARD001', 'Mechanical Keyboard', 75, 10);

-- Test concurrent reservations
SELECT reserve_stock('LAPTOP001', 5, 'ORDER001');
SELECT reserve_stock('LAPTOP001', 3, 'ORDER002');
SELECT reserve_stock('LAPTOP001', 2, 'ORDER003');
```

---

## Best Practices

### Transaction Design Principles

#### Keep Transactions Short
```sql
-- BAD: Long-running transaction
BEGIN;
    SELECT * FROM large_table; -- Expensive query
    PERFORM pg_sleep(10);       -- Long processing
    UPDATE accounts SET balance = balance + 1;
COMMIT;

-- GOOD: Short transaction
-- Do expensive work outside transaction
SELECT * FROM large_table; -- Outside transaction
-- Process data in application

BEGIN;
    UPDATE accounts SET balance = balance + 1; -- Quick update only
COMMIT;
```

#### Avoid User Interaction in Transactions
```sql
-- BAD: Transaction waits for user input
BEGIN;
    SELECT * FROM accounts FOR UPDATE;
    -- Application waits for user confirmation...
    UPDATE accounts SET balance = 0;
COMMIT;

-- GOOD: Prepare data, then transact
-- Get user confirmation first
-- Then execute quick transaction
BEGIN;
    UPDATE accounts SET balance = 0 WHERE confirmed_by_user = true;
COMMIT;
```

### Monitoring and Alerting

#### Transaction Monitoring
```sql
-- Create monitoring view
CREATE VIEW transaction_monitoring AS
SELECT 
    pid,
    usename,
    application_name,
    state,
    xact_start,
    query_start,
    EXTRACT(EPOCH FROM (NOW() - xact_start)) AS transaction_age_seconds,
    EXTRACT(EPOCH FROM (NOW() - query_start)) AS query_age_seconds,
    waiting,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND state != 'idle';

-- Alert function for long transactions
CREATE OR REPLACE FUNCTION check_long_transactions()
RETURNS TABLE(
    alert_level TEXT,
    message TEXT,
    pid INTEGER,
    duration_seconds NUMERIC
) AS $$
BEGIN
    -- Critical: Transactions over 10 minutes
    RETURN QUERY
    SELECT 
        'CRITICAL'::TEXT,
        'Long transaction detected'::TEXT,
        tm.pid,
        tm.transaction_age_seconds
    FROM transaction_monitoring tm
    WHERE tm.transaction_age_seconds > 600;
    
    -- Warning: Transactions over 2 minutes
    RETURN QUERY
    SELECT 
        'WARNING'::TEXT,
        'Medium-duration transaction'::TEXT,
        tm.pid,
        tm.transaction_age_seconds
    FROM transaction_monitoring tm
    WHERE tm.transaction_age_seconds > 120 AND tm.transaction_age_seconds <= 600;
END;
$$ LANGUAGE plpgsql;
```

---

## Next Steps

### Module Completion Checklist
- [ ] Understand ACID properties and their implementation
- [ ] Master transaction control and isolation levels
- [ ] Handle concurrent access safely
- [ ] Implement proper locking strategies
- [ ] Prevent and resolve deadlocks
- [ ] Optimize for high-concurrency scenarios

### Immediate Next Steps
1. **Practice** with concurrent scenarios
2. **Monitor** transaction performance in production
3. **Implement** proper error handling and retry logic
4. **Move to Module 11**: [Security and Authentication](11-security-authentication.md)

---

## Module Summary

✅ **What You've Learned:**
- ACID properties and transaction fundamentals
- Isolation levels and their trade-offs
- Locking mechanisms and concurrency control
- Deadlock prevention and resolution strategies
- Performance optimization for concurrent systems

✅ **Skills Acquired:**
- Design safe concurrent operations
- Choose appropriate isolation levels
- Implement optimistic and pessimistic locking
- Handle deadlocks gracefully
- Monitor and optimize transaction performance

✅ **Ready for Next Module:**
You're now ready to explore database security in [Module 11: Security and Authentication](11-security-authentication.md).

---

## Quick Reference

### Isolation Levels
```sql
READ UNCOMMITTED  -- Allows dirty reads
READ COMMITTED    -- Default, prevents dirty reads
REPEATABLE READ   -- Prevents dirty and non-repeatable reads
SERIALIZABLE      -- Strongest isolation, prevents all phenomena
```

### Lock Types
```sql
FOR SHARE         -- Shared lock, allows other shared locks
FOR UPDATE        -- Exclusive lock, blocks all other locks
FOR UPDATE NOWAIT -- Don't wait for locks
FOR UPDATE SKIP LOCKED -- Skip locked rows
```

### Advisory Locks
```sql
pg_advisory_lock(key)      -- Session-level lock
pg_advisory_xact_lock(key) -- Transaction-level lock
pg_try_advisory_lock(key)  -- Non-blocking attempt
pg_advisory_unlock(key)    -- Release session lock
```

---
*Continue your PostgreSQL journey with [Security and Authentication](11-security-authentication.md) →*
