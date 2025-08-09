# Performance Tuning

## 🎯 Learning Objectives
By the end of this module, you will:
- Analyze and optimize PostgreSQL performance bottlenecks
- Configure memory and storage parameters for optimal performance
- Implement query optimization techniques and index strategies
- Monitor and tune database performance using PostgreSQL tools
- Optimize connection management and resource utilization
- Implement performance testing and benchmarking procedures

## 📚 Table of Contents
1. [Performance Analysis](#performance-analysis)
2. [Memory Configuration](#memory-configuration)
3. [Storage Optimization](#storage-optimization)
4. [Query Optimization](#query-optimization)
5. [Index Optimization](#index-optimization)
6. [Connection Tuning](#connection-tuning)
7. [Monitoring and Metrics](#monitoring-and-metrics)
8. [Performance Testing](#performance-testing)
9. [Hands-On Exercises](#hands-on-exercises)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)

---

## Performance Analysis

### System Resource Analysis

#### CPU and Memory Monitoring
```sql
-- Monitor current database activity and resource usage
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    state_change,
    wait_event_type,
    wait_event,
    LEFT(query, 100) as query_preview
FROM pg_stat_activity 
WHERE state != 'idle'
ORDER BY query_start;

-- Check for long-running queries
SELECT 
    pid,
    usename,
    LEFT(query, 150) as query,
    state,
    EXTRACT(EPOCH FROM (now() - query_start)) as duration_seconds,
    EXTRACT(EPOCH FROM (now() - state_change)) as state_duration_seconds
FROM pg_stat_activity 
WHERE state != 'idle' 
    AND query_start < now() - interval '30 seconds'
ORDER BY duration_seconds DESC;

-- System resource usage view
CREATE OR REPLACE VIEW system_performance AS
SELECT 
    'CPU Usage' as metric,
    ROUND(
        (SELECT sum(total_time) FROM pg_stat_statements) / 
        (SELECT EXTRACT(EPOCH FROM (now() - stats_reset)) FROM pg_stat_database WHERE datname = current_database()) * 100,
        2
    ) as value,
    '%' as unit
UNION ALL
SELECT 
    'Shared Buffers Hit Ratio',
    ROUND(
        100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0),
        2
    ),
    '%'
FROM pg_stat_database
UNION ALL
SELECT 
    'Database Connections',
    (SELECT count(*) FROM pg_stat_activity WHERE state != 'idle')::numeric,
    'connections'
UNION ALL
SELECT 
    'Database Size',
    ROUND(pg_database_size(current_database()) / 1024.0 / 1024.0 / 1024.0, 2),
    'GB';

-- View system performance
SELECT * FROM system_performance;
```

#### Lock Analysis
```sql
-- Monitor locks and blocking queries
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS current_statement_in_blocking_process,
    blocked_activity.application_name AS blocked_application,
    blocking_activity.application_name AS blocking_application
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity 
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity 
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.GRANTED;

-- Detailed lock information
SELECT 
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats 
WHERE schemaname = 'public'
ORDER BY tablename, attname;
```

### Database Statistics Analysis

#### Comprehensive Performance Dashboard
```sql
-- Create comprehensive performance monitoring function
CREATE OR REPLACE FUNCTION get_performance_dashboard()
RETURNS TABLE(
    category TEXT,
    metric TEXT,
    value NUMERIC,
    unit TEXT,
    status TEXT,
    recommendation TEXT
) AS $$
BEGIN
    -- Buffer Cache Hit Ratio
    RETURN QUERY
    SELECT 
        'Memory'::TEXT,
        'Buffer Cache Hit Ratio'::TEXT,
        ROUND(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0), 2),
        '%'::TEXT,
        CASE 
            WHEN 100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0) > 95 THEN 'GOOD'
            WHEN 100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0) > 90 THEN 'OK'
            ELSE 'POOR'
        END::TEXT,
        CASE 
            WHEN 100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0) <= 90 
            THEN 'Consider increasing shared_buffers'
            ELSE 'Buffer cache performing well'
        END::TEXT
    FROM pg_stat_database;
    
    -- Connection count
    RETURN QUERY
    SELECT 
        'Connections'::TEXT,
        'Active Connections'::TEXT,
        (SELECT count(*) FROM pg_stat_activity WHERE state != 'idle')::NUMERIC,
        'connections'::TEXT,
        CASE 
            WHEN (SELECT count(*) FROM pg_stat_activity WHERE state != 'idle') < 
                 (SELECT setting::int * 0.7 FROM pg_settings WHERE name = 'max_connections') THEN 'GOOD'
            WHEN (SELECT count(*) FROM pg_stat_activity WHERE state != 'idle') < 
                 (SELECT setting::int * 0.9 FROM pg_settings WHERE name = 'max_connections') THEN 'OK'
            ELSE 'HIGH'
        END::TEXT,
        CASE 
            WHEN (SELECT count(*) FROM pg_stat_activity WHERE state != 'idle') >= 
                 (SELECT setting::int * 0.9 FROM pg_settings WHERE name = 'max_connections') 
            THEN 'Consider connection pooling'
            ELSE 'Connection usage normal'
        END::TEXT;
    
    -- Table scan ratio
    RETURN QUERY
    SELECT 
        'I/O'::TEXT,
        'Sequential Scan Ratio'::TEXT,
        ROUND(100.0 * sum(seq_scan) / NULLIF(sum(seq_scan) + sum(idx_scan), 0), 2),
        '%'::TEXT,
        CASE 
            WHEN 100.0 * sum(seq_scan) / NULLIF(sum(seq_scan) + sum(idx_scan), 0) < 10 THEN 'GOOD'
            WHEN 100.0 * sum(seq_scan) / NULLIF(sum(seq_scan) + sum(idx_scan), 0) < 25 THEN 'OK'
            ELSE 'HIGH'
        END::TEXT,
        CASE 
            WHEN 100.0 * sum(seq_scan) / NULLIF(sum(seq_scan) + sum(idx_scan), 0) >= 25 
            THEN 'Review queries and add indexes'
            ELSE 'Index usage is good'
        END::TEXT
    FROM pg_stat_user_tables;
    
    -- Database size growth
    RETURN QUERY
    SELECT 
        'Storage'::TEXT,
        'Database Size'::TEXT,
        ROUND(pg_database_size(current_database()) / 1024.0 / 1024.0 / 1024.0, 2),
        'GB'::TEXT,
        'INFO'::TEXT,
        'Monitor growth trends'::TEXT;
    
    -- WAL generation rate
    RETURN QUERY
    SELECT 
        'WAL'::TEXT,
        'WAL Generation Rate'::TEXT,
        ROUND(
            pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024.0 / 1024.0 / 
            EXTRACT(EPOCH FROM (now() - pg_postmaster_start_time())) * 3600,
            2
        ),
        'MB/hour'::TEXT,
        'INFO'::TEXT,
        'Monitor for backup planning'::TEXT;
END;
$$ LANGUAGE plpgsql;

-- Get performance dashboard
SELECT * FROM get_performance_dashboard() ORDER BY category, metric;
```

---

## Memory Configuration

### Shared Memory Settings

#### Optimal Memory Configuration
```sql
-- Check current memory settings
SELECT 
    name,
    setting,
    unit,
    context,
    short_desc
FROM pg_settings 
WHERE name IN (
    'shared_buffers',
    'effective_cache_size',
    'work_mem',
    'maintenance_work_mem',
    'temp_buffers',
    'max_connections'
)
ORDER BY name;

-- Memory recommendations based on system memory
CREATE OR REPLACE FUNCTION recommend_memory_settings()
RETURNS TABLE(
    parameter TEXT,
    current_value TEXT,
    recommended_value TEXT,
    description TEXT
) AS $$
DECLARE
    total_memory_gb NUMERIC;
    shared_buffers_mb NUMERIC;
    effective_cache_mb NUMERIC;
    work_mem_mb NUMERIC;
    maintenance_work_mem_mb NUMERIC;
BEGIN
    -- Estimate total system memory (simplified)
    -- In production, you'd get this from system information
    total_memory_gb := 8; -- Assume 8GB for example
    
    -- Calculate recommendations
    shared_buffers_mb := total_memory_gb * 1024 * 0.25; -- 25% of total memory
    effective_cache_mb := total_memory_gb * 1024 * 0.75; -- 75% of total memory
    work_mem_mb := GREATEST(4, shared_buffers_mb / 32); -- Based on connections
    maintenance_work_mem_mb := shared_buffers_mb / 4; -- 25% of shared_buffers
    
    RETURN QUERY
    SELECT 
        'shared_buffers'::TEXT,
        (SELECT setting || ' ' || COALESCE(unit, '') FROM pg_settings WHERE name = 'shared_buffers'),
        shared_buffers_mb::int || 'MB',
        'Main buffer cache for data pages'::TEXT
    UNION ALL
    SELECT 
        'effective_cache_size'::TEXT,
        (SELECT setting || ' ' || COALESCE(unit, '') FROM pg_settings WHERE name = 'effective_cache_size'),
        effective_cache_mb::int || 'MB',
        'Estimate of total system cache'::TEXT
    UNION ALL
    SELECT 
        'work_mem'::TEXT,
        (SELECT setting || ' ' || COALESCE(unit, '') FROM pg_settings WHERE name = 'work_mem'),
        work_mem_mb::int || 'MB',
        'Memory for sort and hash operations'::TEXT
    UNION ALL
    SELECT 
        'maintenance_work_mem'::TEXT,
        (SELECT setting || ' ' || COALESCE(unit, '') FROM pg_settings WHERE name = 'maintenance_work_mem'),
        maintenance_work_mem_mb::int || 'MB',
        'Memory for maintenance operations'::TEXT;
END;
$$ LANGUAGE plpgsql;

-- Get memory recommendations
SELECT * FROM recommend_memory_settings();
```

#### Memory Usage Monitoring
```sql
-- Monitor memory usage by query type
WITH memory_stats AS (
    SELECT 
        query,
        calls,
        total_time,
        mean_time,
        rows,
        100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS hit_percent,
        shared_blks_read,
        shared_blks_written,
        temp_blks_read,
        temp_blks_written
    FROM pg_stat_statements
    WHERE calls > 10
)
SELECT 
    LEFT(query, 100) AS query_snippet,
    calls,
    ROUND(total_time::numeric, 2) AS total_time_ms,
    ROUND(mean_time::numeric, 2) AS avg_time_ms,
    ROUND(hit_percent, 2) AS buffer_hit_ratio,
    shared_blks_read AS disk_reads,
    temp_blks_read AS temp_reads,
    CASE 
        WHEN temp_blks_read > 0 THEN 'Uses temp files'
        WHEN hit_percent < 95 THEN 'High disk I/O'
        ELSE 'Good memory usage'
    END AS memory_assessment
FROM memory_stats
ORDER BY total_time DESC
LIMIT 20;
```

---

## Storage Optimization

### Disk I/O Optimization

#### Storage Configuration Analysis
```sql
-- Analyze storage performance metrics
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_tup_hot_upd,
    ROUND(100.0 * idx_scan / NULLIF(seq_scan + idx_scan, 0), 2) AS index_usage_pct,
    ROUND(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) AS hot_update_pct
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 0
ORDER BY seq_scan + idx_scan DESC;

-- Identify tables needing indexes
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    seq_tup_read / NULLIF(seq_scan, 0) AS avg_tuples_per_scan,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_stat_user_tables
WHERE seq_scan > 1000
    AND seq_tup_read / NULLIF(seq_scan, 0) > 1000
ORDER BY seq_scan DESC;

-- Monitor I/O performance
SELECT 
    tablename,
    heap_blks_read,
    heap_blks_hit,
    ROUND(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) AS heap_hit_ratio,
    idx_blks_read,
    idx_blks_hit,
    ROUND(100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0), 2) AS idx_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY heap_blks_read + idx_blks_read DESC;
```

#### Tablespace and Storage Management
```sql
-- Create performance-optimized tablespace setup
-- (This would be done at the system level)

-- Monitor tablespace usage
SELECT 
    spcname AS tablespace_name,
    pg_size_pretty(pg_tablespace_size(spcname)) AS size,
    spclocation AS location
FROM pg_tablespace;

-- Analyze table storage characteristics
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) AS index_size,
    ROUND(
        100.0 * pg_indexes_size(schemaname||'.'||tablename) / 
        NULLIF(pg_total_relation_size(schemaname||'.'||tablename), 0),
        2
    ) AS index_ratio_pct,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_tuple_pct
FROM pg_stat_user_tables t
JOIN pg_stat_user_tables s USING (schemaname, tablename)
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### VACUUM and ANALYZE Optimization

#### Automated Maintenance Strategy
```sql
-- Monitor autovacuum performance
SELECT 
    schemaname,
    tablename,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count,
    analyze_count,
    autoanalyze_count,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY dead_pct DESC NULLS LAST;

-- Create custom vacuum strategy
CREATE OR REPLACE FUNCTION intelligent_vacuum_analyze()
RETURNS TEXT AS $$
DECLARE
    table_record RECORD;
    vacuum_cmd TEXT;
    result_text TEXT := '';
BEGIN
    FOR table_record IN
        SELECT 
            schemaname,
            tablename,
            n_dead_tup,
            n_live_tup,
            ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
            last_autovacuum,
            last_autoanalyze
        FROM pg_stat_user_tables
        WHERE n_dead_tup > 1000
        ORDER BY n_dead_tup DESC
    LOOP
        -- Determine vacuum strategy based on dead tuple percentage
        IF table_record.dead_pct > 20 THEN
            vacuum_cmd := format('VACUUM (VERBOSE, ANALYZE) %I.%I;', 
                                table_record.schemaname, 
                                table_record.tablename);
            result_text := result_text || format('High dead tuples (%s%%): %s.%s - Running VACUUM ANALYZE\n',
                                                table_record.dead_pct,
                                                table_record.schemaname,
                                                table_record.tablename);
        ELSIF table_record.dead_pct > 10 THEN
            vacuum_cmd := format('VACUUM %I.%I;', 
                                table_record.schemaname, 
                                table_record.tablename);
            result_text := result_text || format('Moderate dead tuples (%s%%): %s.%s - Running VACUUM\n',
                                                table_record.dead_pct,
                                                table_record.schemaname,
                                                table_record.tablename);
        ELSE
            -- Just analyze if stats are old
            IF table_record.last_autoanalyze < NOW() - INTERVAL '1 day' THEN
                vacuum_cmd := format('ANALYZE %I.%I;', 
                                    table_record.schemaname, 
                                    table_record.tablename);
                result_text := result_text || format('Old statistics: %s.%s - Running ANALYZE\n',
                                                    table_record.schemaname,
                                                    table_record.tablename);
            ELSE
                CONTINUE;
            END IF;
        END IF;
        
        -- Execute the command (uncomment in production)
        -- EXECUTE vacuum_cmd;
    END LOOP;
    
    RETURN COALESCE(result_text, 'No maintenance needed');
END;
$$ LANGUAGE plpgsql;

-- Check what maintenance is recommended
SELECT intelligent_vacuum_analyze();
```

---

## Query Optimization

### Query Analysis and Tuning

#### Slow Query Identification
```sql
-- Top slowest queries by total time
SELECT 
    query,
    calls,
    ROUND(total_time::numeric, 2) AS total_time_ms,
    ROUND(mean_time::numeric, 2) AS avg_time_ms,
    ROUND(100.0 * total_time / sum(total_time) OVER(), 2) AS pct_total_time,
    rows,
    100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
WHERE calls > 5
ORDER BY total_time DESC
LIMIT 15;

-- Queries with highest I/O
SELECT 
    LEFT(query, 100) AS query_snippet,
    calls,
    shared_blks_read + shared_blks_written AS total_io,
    ROUND((shared_blks_read + shared_blks_written)::numeric / calls, 2) AS avg_io_per_call,
    temp_blks_read + temp_blks_written AS temp_io,
    ROUND(total_time::numeric / calls, 2) AS avg_time_ms
FROM pg_stat_statements
WHERE calls > 5
    AND (shared_blks_read + shared_blks_written) > 0
ORDER BY total_io DESC
LIMIT 15;

-- Queries using temporary files
SELECT 
    LEFT(query, 150) AS query_snippet,
    calls,
    temp_blks_read,
    temp_blks_written,
    ROUND(temp_blks_written * 8.0 / 1024, 2) AS temp_mb_written,
    ROUND(mean_time::numeric, 2) AS avg_time_ms
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 10;
```

#### Query Plan Analysis
```sql
-- Function to analyze query plans
CREATE OR REPLACE FUNCTION analyze_query_performance(
    query_text TEXT
) RETURNS TABLE(
    plan_line TEXT,
    analysis TEXT
) AS $$
DECLARE
    plan_record RECORD;
    explain_text TEXT;
BEGIN
    -- Get the execution plan
    FOR plan_record IN
        EXECUTE format('EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) %s', query_text)
    LOOP
        RETURN QUERY SELECT 
            plan_record.plan_line,
            CASE 
                WHEN plan_record.plan_line ~ 'Seq Scan' THEN 'Consider adding index'
                WHEN plan_record.plan_line ~ 'Nested Loop.*rows=\d{4,}' THEN 'Large nested loop - consider hash join'
                WHEN plan_record.plan_line ~ 'Sort.*external merge' THEN 'Sort spilling to disk - increase work_mem'
                WHEN plan_record.plan_line ~ 'shared hit=(\d+).*read=(\d+)' THEN 'Monitor buffer cache efficiency'
                ELSE 'OK'
            END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Example usage (would be used with actual slow queries)
-- SELECT * FROM analyze_query_performance('SELECT * FROM large_table WHERE some_column = 123');
```

### Advanced Query Optimization

#### Query Rewriting Techniques
```sql
-- Example: Optimizing subqueries to joins
-- Before (slow subquery):
-- SELECT * FROM customers c 
-- WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.total > 1000);

-- After (optimized join):
-- SELECT DISTINCT c.* FROM customers c 
-- JOIN orders o ON c.id = o.customer_id 
-- WHERE o.total > 1000;

-- Function to suggest query optimizations
CREATE OR REPLACE FUNCTION suggest_query_optimizations()
RETURNS TABLE(
    query_snippet TEXT,
    issue TEXT,
    suggestion TEXT,
    impact TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        LEFT(query, 80)::TEXT,
        'High sequential scan ratio'::TEXT,
        'Add appropriate indexes'::TEXT,
        'HIGH'::TEXT
    FROM pg_stat_statements
    WHERE calls > 100
        AND shared_blks_read > shared_blks_hit
        AND query NOT LIKE '%pg_stat%'
        AND query NOT LIKE '%information_schema%'
    
    UNION ALL
    
    SELECT 
        LEFT(query, 80)::TEXT,
        'Using temporary files'::TEXT,
        'Increase work_mem or optimize query'::TEXT,
        'MEDIUM'::TEXT
    FROM pg_stat_statements
    WHERE temp_blks_written > 1000
    
    UNION ALL
    
    SELECT 
        LEFT(query, 80)::TEXT,
        'Long average execution time'::TEXT,
        'Review query plan and indexes'::TEXT,
        'HIGH'::TEXT
    FROM pg_stat_statements
    WHERE mean_time > 1000
        AND calls > 10;
END;
$$ LANGUAGE plpgsql;

-- Get optimization suggestions
SELECT * FROM suggest_query_optimizations() ORDER BY impact DESC;
```

---

## Index Optimization

### Index Analysis and Strategy

#### Comprehensive Index Analysis
```sql
-- Unused indexes detection
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    CASE 
        WHEN idx_scan = 0 THEN 'Never used'
        WHEN idx_scan < 10 THEN 'Rarely used'
        ELSE 'Actively used'
    END AS usage_status
FROM pg_stat_user_indexes
ORDER BY idx_scan, pg_relation_size(indexrelid) DESC;

-- Index efficiency analysis
WITH index_stats AS (
    SELECT 
        schemaname,
        tablename,
        indexname,
        idx_scan,
        idx_tup_read,
        idx_tup_fetch,
        pg_relation_size(indexrelid) AS index_size,
        CASE 
            WHEN idx_scan > 0 THEN idx_tup_read::numeric / idx_scan
            ELSE 0
        END AS avg_tuples_per_scan
    FROM pg_stat_user_indexes
)
SELECT 
    *,
    pg_size_pretty(index_size) AS formatted_size,
    CASE 
        WHEN idx_scan = 0 THEN 'Consider dropping'
        WHEN avg_tuples_per_scan > 1000 THEN 'Low selectivity - review'
        WHEN avg_tuples_per_scan < 10 THEN 'High selectivity - good'
        ELSE 'Moderate selectivity'
    END AS selectivity_assessment
FROM index_stats
WHERE index_size > 1048576  -- Indexes larger than 1MB
ORDER BY index_size DESC;

-- Missing index suggestions
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    seq_tup_read / NULLIF(seq_scan, 0) AS avg_seq_tup_read,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS table_size,
    'Consider adding indexes' AS recommendation
FROM pg_stat_user_tables
WHERE seq_scan > 1000
    AND seq_tup_read / NULLIF(seq_scan, 0) > 100
    AND pg_total_relation_size(schemaname||'.'||tablename) > 10485760  -- Tables larger than 10MB
ORDER BY seq_scan * seq_tup_read DESC;
```

#### Advanced Index Strategies
```sql
-- Composite index analysis for multi-column queries
CREATE OR REPLACE FUNCTION analyze_composite_index_opportunities()
RETURNS TABLE(
    table_name TEXT,
    columns_used TEXT,
    query_count BIGINT,
    suggestion TEXT
) AS $$
BEGIN
    -- This is a simplified example
    -- In practice, you'd analyze actual query patterns from pg_stat_statements
    
    RETURN QUERY
    SELECT 
        'Example: orders'::TEXT,
        'customer_id, order_date'::TEXT,
        100::BIGINT,
        'CREATE INDEX idx_orders_customer_date ON orders (customer_id, order_date);'::TEXT
    WHERE NOT EXISTS (
        SELECT 1 FROM pg_indexes 
        WHERE tablename = 'orders' 
        AND indexdef LIKE '%customer_id%order_date%'
    );
    
    -- Add more analysis based on your specific query patterns
END;
$$ LANGUAGE plpgsql;

-- Partial index opportunities
CREATE OR REPLACE FUNCTION suggest_partial_indexes()
RETURNS TABLE(
    table_name TEXT,
    condition_analysis TEXT,
    suggested_index TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        'orders'::TEXT,
        'Many queries filter by status = ''active'''::TEXT,
        'CREATE INDEX idx_orders_active ON orders (customer_id, order_date) WHERE status = ''active'';'::TEXT
    
    UNION ALL
    
    SELECT 
        'users'::TEXT,
        'Queries often filter non-deleted records'::TEXT,
        'CREATE INDEX idx_users_active ON users (email, created_at) WHERE deleted_at IS NULL;'::TEXT;
END;
$$ LANGUAGE plpgsql;

-- Get index suggestions
SELECT * FROM analyze_composite_index_opportunities();
SELECT * FROM suggest_partial_indexes();
```

---

## Connection Tuning

### Connection Pool Optimization

#### Connection Analysis
```sql
-- Monitor connection patterns
SELECT 
    usename,
    application_name,
    client_addr,
    COUNT(*) AS connection_count,
    COUNT(CASE WHEN state = 'active' THEN 1 END) AS active_connections,
    COUNT(CASE WHEN state = 'idle' THEN 1 END) AS idle_connections,
    COUNT(CASE WHEN state = 'idle in transaction' THEN 1 END) AS idle_in_transaction,
    MAX(backend_start) AS latest_connection,
    MIN(backend_start) AS earliest_connection
FROM pg_stat_activity
GROUP BY usename, application_name, client_addr
ORDER BY connection_count DESC;

-- Long idle connections
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    state_change,
    EXTRACT(EPOCH FROM (now() - state_change)) AS idle_seconds,
    backend_start
FROM pg_stat_activity
WHERE state = 'idle'
    AND state_change < now() - interval '10 minutes'
ORDER BY state_change;

-- Connection pool recommendations
CREATE OR REPLACE FUNCTION recommend_connection_settings()
RETURNS TABLE(
    parameter TEXT,
    current_value TEXT,
    recommended_value TEXT,
    reasoning TEXT
) AS $$
DECLARE
    max_conn INTEGER;
    active_conn INTEGER;
    available_conn INTEGER;
BEGIN
    SELECT setting::int INTO max_conn FROM pg_settings WHERE name = 'max_connections';
    SELECT count(*) INTO active_conn FROM pg_stat_activity WHERE state != 'idle';
    available_conn := max_conn - active_conn;
    
    RETURN QUERY
    SELECT 
        'max_connections'::TEXT,
        max_conn::TEXT,
        CASE 
            WHEN active_conn::float / max_conn > 0.8 THEN (max_conn * 1.5)::TEXT
            WHEN active_conn::float / max_conn < 0.3 THEN (max_conn * 0.7)::TEXT
            ELSE max_conn::TEXT
        END,
        CASE 
            WHEN active_conn::float / max_conn > 0.8 THEN 'High utilization - consider increasing'
            WHEN active_conn::float / max_conn < 0.3 THEN 'Low utilization - consider decreasing'
            ELSE 'Current setting appropriate'
        END;
    
    RETURN QUERY
    SELECT 
        'shared_buffers'::TEXT,
        (SELECT setting FROM pg_settings WHERE name = 'shared_buffers'),
        'Consider adjusting based on connection count'::TEXT,
        'shared_buffers should account for connection memory overhead'::TEXT;
END;
$$ LANGUAGE plpgsql;

-- Get connection recommendations
SELECT * FROM recommend_connection_settings();
```

#### Connection Pool Configuration Script
```bash
#!/bin/bash
# connection_pool_optimizer.sh - Optimize connection pool settings

DB_HOST="localhost"
DB_PORT="5432"
DB_USER="postgres"
DB_NAME="postgres"

# Function to get current connection stats
get_connection_stats() {
    psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -t -c "
        SELECT 
            (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections,
            (SELECT count(*) FROM pg_stat_activity) AS total_connections,
            (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') AS active_connections,
            (SELECT count(*) FROM pg_stat_activity WHERE state = 'idle') AS idle_connections,
            (SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction') AS idle_in_transaction
    " | xargs
}

# Function to recommend pgbouncer settings
recommend_pgbouncer_config() {
    echo "# PgBouncer configuration recommendations"
    echo "[databases]"
    echo "mydb = host=$DB_HOST port=$DB_PORT dbname=$DB_NAME"
    echo ""
    echo "[pgbouncer]"
    echo "pool_mode = transaction"
    echo "max_client_conn = 200"
    echo "default_pool_size = 20"
    echo "min_pool_size = 5"
    echo "reserve_pool_size = 3"
    echo "server_lifetime = 3600"
    echo "server_idle_timeout = 600"
    echo "log_connections = 1"
    echo "log_disconnections = 1"
    echo "stats_period = 60"
}

# Main execution
echo "PostgreSQL Connection Pool Optimizer"
echo "===================================="

echo "Current connection statistics:"
get_connection_stats

echo ""
echo "PgBouncer configuration recommendations:"
recommend_pgbouncer_config
```

---

## Monitoring and Metrics

### Performance Monitoring Dashboard

#### Real-time Performance Metrics
```sql
-- Create comprehensive monitoring view
CREATE OR REPLACE VIEW performance_monitor AS
WITH current_activity AS (
    SELECT 
        COUNT(*) AS total_connections,
        COUNT(CASE WHEN state = 'active' THEN 1 END) AS active_connections,
        COUNT(CASE WHEN state = 'idle' THEN 1 END) AS idle_connections,
        COUNT(CASE WHEN wait_event IS NOT NULL THEN 1 END) AS waiting_connections
    FROM pg_stat_activity
),
database_stats AS (
    SELECT 
        datname,
        numbackends,
        xact_commit,
        xact_rollback,
        blks_read,
        blks_hit,
        tup_returned,
        tup_fetched,
        tup_inserted,
        tup_updated,
        tup_deleted,
        deadlocks,
        temp_files,
        temp_bytes
    FROM pg_stat_database 
    WHERE datname = current_database()
),
buffer_stats AS (
    SELECT 
        ROUND(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0), 2) AS buffer_hit_ratio
    FROM pg_stat_database
)
SELECT 
    now() AS timestamp,
    ca.total_connections,
    ca.active_connections,
    ca.idle_connections,
    ca.waiting_connections,
    ds.xact_commit,
    ds.xact_rollback,
    ROUND(100.0 * ds.xact_commit / NULLIF(ds.xact_commit + ds.xact_rollback, 0), 2) AS commit_ratio,
    bs.buffer_hit_ratio,
    ds.deadlocks,
    ds.temp_files,
    pg_size_pretty(ds.temp_bytes) AS temp_bytes_formatted,
    pg_size_pretty(pg_database_size(current_database())) AS database_size
FROM current_activity ca
CROSS JOIN database_stats ds
CROSS JOIN buffer_stats bs;

-- View current performance
SELECT * FROM performance_monitor;

-- Historical performance tracking
CREATE TABLE IF NOT EXISTS performance_history (
    recorded_at TIMESTAMP DEFAULT now(),
    total_connections INTEGER,
    active_connections INTEGER,
    buffer_hit_ratio NUMERIC(5,2),
    commit_ratio NUMERIC(5,2),
    database_size_bytes BIGINT,
    temp_files BIGINT,
    temp_bytes BIGINT
);

-- Function to record performance metrics
CREATE OR REPLACE FUNCTION record_performance_metrics()
RETURNS VOID AS $$
BEGIN
    INSERT INTO performance_history (
        total_connections,
        active_connections,
        buffer_hit_ratio,
        commit_ratio,
        database_size_bytes,
        temp_files,
        temp_bytes
    )
    SELECT 
        total_connections,
        active_connections,
        buffer_hit_ratio,
        commit_ratio,
        pg_database_size(current_database()),
        (SELECT temp_files FROM pg_stat_database WHERE datname = current_database()),
        (SELECT temp_bytes FROM pg_stat_database WHERE datname = current_database())
    FROM performance_monitor;
END;
$$ LANGUAGE plpgsql;

-- Schedule this to run periodically (e.g., every 5 minutes)
-- SELECT record_performance_metrics();
```

#### Alert System
```sql
-- Create performance alert system
CREATE OR REPLACE FUNCTION check_performance_alerts()
RETURNS TABLE(
    alert_type TEXT,
    severity TEXT,
    message TEXT,
    current_value NUMERIC,
    threshold NUMERIC
) AS $$
BEGIN
    -- Buffer hit ratio alert
    RETURN QUERY
    SELECT 
        'Buffer Hit Ratio'::TEXT,
        CASE WHEN buffer_hit_ratio < 95 THEN 'HIGH' ELSE 'INFO' END::TEXT,
        'Buffer hit ratio is below optimal threshold'::TEXT,
        buffer_hit_ratio,
        95.0
    FROM performance_monitor
    WHERE buffer_hit_ratio < 98;
    
    -- Connection limit alert
    RETURN QUERY
    SELECT 
        'Connection Usage'::TEXT,
        CASE 
            WHEN total_connections::float / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') > 0.9 THEN 'HIGH'
            WHEN total_connections::float / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') > 0.7 THEN 'MEDIUM'
            ELSE 'LOW'
        END::TEXT,
        'Connection usage is high'::TEXT,
        total_connections,
        (SELECT setting::int * 0.8 FROM pg_settings WHERE name = 'max_connections')::NUMERIC
    FROM performance_monitor
    WHERE total_connections::float / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') > 0.7;
    
    -- Long running query alert
    RETURN QUERY
    SELECT 
        'Long Running Query'::TEXT,
        'MEDIUM'::TEXT,
        'Query running longer than 5 minutes: ' || LEFT(query, 100),
        EXTRACT(EPOCH FROM (now() - query_start))::NUMERIC,
        300.0
    FROM pg_stat_activity
    WHERE state = 'active'
        AND query_start < now() - interval '5 minutes'
        AND query NOT LIKE '%pg_stat_activity%';
    
    -- Deadlock alert
    RETURN QUERY
    SELECT 
        'Deadlocks'::TEXT,
        'HIGH'::TEXT,
        'Deadlocks detected in database'::TEXT,
        deadlocks::NUMERIC,
        0.0
    FROM pg_stat_database
    WHERE datname = current_database()
        AND deadlocks > 0;
END;
$$ LANGUAGE plpgsql;

-- Check for alerts
SELECT * FROM check_performance_alerts() ORDER BY severity DESC;
```

---

## Performance Testing

### Benchmarking Tools and Techniques

#### Database Benchmarking Script
```bash
#!/bin/bash
# postgres_benchmark.sh - Comprehensive PostgreSQL benchmarking

DB_HOST="localhost"
DB_PORT="5432"
DB_USER="postgres"
DB_NAME="benchmark_test"
BENCHMARK_DURATION="300"  # 5 minutes
CLIENTS="10"
THREADS="4"

# Function to setup benchmark database
setup_benchmark_db() {
    echo "Setting up benchmark database..."
    
    # Create benchmark database
    createdb -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" "$DB_NAME"
    
    # Initialize pgbench
    pgbench -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -i -s 100 "$DB_NAME"
    
    echo "Benchmark database setup complete"
}

# Function to run standard TPC-B benchmark
run_tpcb_benchmark() {
    echo "Running TPC-B benchmark..."
    
    pgbench -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" \
        -c "$CLIENTS" -j "$THREADS" -T "$BENCHMARK_DURATION" \
        -P 30 -r "$DB_NAME"
}

# Function to run read-only benchmark
run_readonly_benchmark() {
    echo "Running read-only benchmark..."
    
    pgbench -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" \
        -c "$CLIENTS" -j "$THREADS" -T "$BENCHMARK_DURATION" \
        -S -P 30 -r "$DB_NAME"
}

# Function to run custom benchmark with complex queries
run_custom_benchmark() {
    echo "Running custom benchmark..."
    
    # Create custom benchmark script
    cat > /tmp/custom_benchmark.sql << 'EOF'
\set aid random(1, 100000 * :scale)
\set bid random(1, 1 * :scale)
\set tid random(1, 10 * :scale)
\set delta random(-5000, 5000)
\set value random(1, 1000)

-- Complex analytical query
SELECT 
    a.aid,
    SUM(h.delta) as total_delta,
    COUNT(*) as transaction_count,
    AVG(h.delta) as avg_delta
FROM pgbench_accounts a
JOIN pgbench_history h ON a.aid = h.aid
WHERE a.aid BETWEEN :aid AND :aid + 100
GROUP BY a.aid
HAVING COUNT(*) > 5
ORDER BY total_delta DESC
LIMIT 10;
EOF

    pgbench -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" \
        -c "$CLIENTS" -j "$THREADS" -T "$BENCHMARK_DURATION" \
        -f /tmp/custom_benchmark.sql -P 30 -r "$DB_NAME"
}

# Function to cleanup
cleanup_benchmark() {
    echo "Cleaning up benchmark database..."
    dropdb -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" "$DB_NAME"
    rm -f /tmp/custom_benchmark.sql
}

# Function to generate performance report
generate_performance_report() {
    echo "Generating performance report..."
    
    cat << EOF
PostgreSQL Performance Benchmark Report
======================================
Date: $(date)
Database: $DB_NAME
Host: $DB_HOST:$DB_PORT
Clients: $CLIENTS
Threads: $THREADS
Duration: $BENCHMARK_DURATION seconds

System Information:
- CPU: $(nproc) cores
- Memory: $(free -h | grep Mem | awk '{print $2}')
- Disk: $(df -h / | tail -1 | awk '{print $4}') available

Database Configuration:
- shared_buffers: $(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -t -c "SHOW shared_buffers;" | xargs)
- work_mem: $(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -t -c "SHOW work_mem;" | xargs)
- max_connections: $(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -t -c "SHOW max_connections;" | xargs)

Benchmark Results:
- TPC-B results: See output above
- Read-only results: See output above
- Custom query results: See output above

EOF
}

# Main execution
case "${1:-all}" in
    setup)
        setup_benchmark_db
        ;;
    tpcb)
        run_tpcb_benchmark
        ;;
    readonly)
        run_readonly_benchmark
        ;;
    custom)
        run_custom_benchmark
        ;;
    cleanup)
        cleanup_benchmark
        ;;
    all)
        setup_benchmark_db
        echo "Starting benchmarks..."
        run_tpcb_benchmark
        run_readonly_benchmark
        run_custom_benchmark
        generate_performance_report
        cleanup_benchmark
        ;;
    *)
        echo "Usage: $0 {setup|tpcb|readonly|custom|cleanup|all}"
        exit 1
        ;;
esac
```

#### Performance Testing Framework
```sql
-- Create performance testing framework
CREATE SCHEMA IF NOT EXISTS performance_testing;

-- Performance test results table
CREATE TABLE performance_testing.test_results (
    test_id SERIAL PRIMARY KEY,
    test_name VARCHAR(200) NOT NULL,
    test_category VARCHAR(100) NOT NULL,
    start_time TIMESTAMP DEFAULT now(),
    end_time TIMESTAMP,
    duration_ms NUMERIC,
    rows_processed BIGINT,
    rows_per_second NUMERIC,
    cpu_usage_percent NUMERIC,
    memory_usage_mb NUMERIC,
    io_read_mb NUMERIC,
    io_write_mb NUMERIC,
    test_parameters JSONB,
    test_results JSONB,
    notes TEXT
);

-- Function to run performance tests
CREATE OR REPLACE FUNCTION performance_testing.run_test(
    p_test_name TEXT,
    p_test_category TEXT,
    p_sql_query TEXT,
    p_iterations INTEGER DEFAULT 1,
    p_parameters JSONB DEFAULT '{}'::jsonb
) RETURNS INTEGER AS $$
DECLARE
    test_id INTEGER;
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    duration_ms NUMERIC;
    rows_processed BIGINT := 0;
    i INTEGER;
BEGIN
    -- Insert test record
    INSERT INTO performance_testing.test_results (test_name, test_category, test_parameters)
    VALUES (p_test_name, p_test_category, p_parameters)
    RETURNING test_results.test_id INTO test_id;
    
    -- Record start time
    start_time := clock_timestamp();
    
    -- Run test iterations
    FOR i IN 1..p_iterations LOOP
        EXECUTE p_sql_query;
        GET DIAGNOSTICS rows_processed = ROW_COUNT;
    END LOOP;
    
    -- Record end time
    end_time := clock_timestamp();
    duration_ms := EXTRACT(EPOCH FROM (end_time - start_time)) * 1000;
    
    -- Update test results
    UPDATE performance_testing.test_results 
    SET 
        end_time = end_time,
        duration_ms = duration_ms,
        rows_processed = rows_processed * p_iterations,
        rows_per_second = (rows_processed * p_iterations) / NULLIF(duration_ms / 1000.0, 0)
    WHERE performance_testing.test_results.test_id = run_test.test_id;
    
    RETURN test_id;
END;
$$ LANGUAGE plpgsql;

-- Example performance tests
SELECT performance_testing.run_test(
    'Select All Customers',
    'Read Performance',
    'SELECT * FROM customers',
    10
);

-- Performance test report
CREATE OR REPLACE FUNCTION performance_testing.generate_report(
    p_test_category TEXT DEFAULT NULL
) RETURNS TABLE(
    test_name TEXT,
    category TEXT,
    avg_duration_ms NUMERIC,
    avg_rows_per_second NUMERIC,
    test_count BIGINT,
    latest_test TIMESTAMP
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        tr.test_name::TEXT,
        tr.test_category::TEXT,
        ROUND(AVG(tr.duration_ms), 2) AS avg_duration_ms,
        ROUND(AVG(tr.rows_per_second), 2) AS avg_rows_per_second,
        COUNT(*) AS test_count,
        MAX(tr.start_time) AS latest_test
    FROM performance_testing.test_results tr
    WHERE p_test_category IS NULL OR tr.test_category = p_test_category
    GROUP BY tr.test_name, tr.test_category
    ORDER BY avg_duration_ms DESC;
END;
$$ LANGUAGE plpgsql;

-- Generate performance report
SELECT * FROM performance_testing.generate_report();
```

---

## Hands-On Exercises

### Exercise 1: Complete Performance Audit

Perform a comprehensive performance audit of your PostgreSQL instance.

```sql
-- Create performance audit procedure
CREATE OR REPLACE FUNCTION perform_performance_audit()
RETURNS TABLE(
    audit_category TEXT,
    finding TEXT,
    severity TEXT,
    recommendation TEXT,
    current_value TEXT
) AS $$
BEGIN
    -- Memory configuration audit
    RETURN QUERY
    SELECT 
        'Memory Configuration'::TEXT,
        'shared_buffers setting'::TEXT,
        CASE 
            WHEN setting::bigint < pg_size_bytes('256MB') THEN 'HIGH'
            WHEN setting::bigint < pg_size_bytes('1GB') THEN 'MEDIUM'
            ELSE 'LOW'
        END::TEXT,
        'Consider increasing shared_buffers to 25% of system RAM'::TEXT,
        setting || ' ' || COALESCE(unit, '')
    FROM pg_settings 
    WHERE name = 'shared_buffers';
    
    -- Add more audit checks...
    -- Connection usage audit
    RETURN QUERY
    SELECT 
        'Connection Management'::TEXT,
        'Connection utilization'::TEXT,
        CASE 
            WHEN (SELECT count(*) FROM pg_stat_activity WHERE state != 'idle')::float / 
                 (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') > 0.8 THEN 'HIGH'
            ELSE 'LOW'
        END::TEXT,
        'Monitor connection pooling effectiveness'::TEXT,
        (SELECT count(*) FROM pg_stat_activity WHERE state != 'idle')::TEXT || ' active connections';
END;
$$ LANGUAGE plpgsql;

-- Run performance audit
SELECT * FROM perform_performance_audit() ORDER BY severity DESC;
```

---

## Best Practices

### Performance Optimization Checklist

#### Configuration Best Practices
```sql
-- Function to validate configuration best practices
CREATE OR REPLACE FUNCTION validate_performance_config()
RETURNS TABLE(
    config_area TEXT,
    parameter TEXT,
    current_value TEXT,
    recommended_range TEXT,
    status TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        'Memory'::TEXT,
        'shared_buffers'::TEXT,
        setting,
        '25% of system RAM'::TEXT,
        CASE 
            WHEN setting::bigint >= pg_size_bytes('1GB') THEN 'GOOD'
            WHEN setting::bigint >= pg_size_bytes('128MB') THEN 'OK'
            ELSE 'POOR'
        END
    FROM pg_settings WHERE name = 'shared_buffers'
    
    UNION ALL
    
    SELECT 
        'Memory'::TEXT,
        'work_mem'::TEXT,
        setting,
        '4-16MB per connection'::TEXT,
        CASE 
            WHEN setting::bigint BETWEEN pg_size_bytes('4MB') AND pg_size_bytes('16MB') THEN 'GOOD'
            ELSE 'REVIEW'
        END
    FROM pg_settings WHERE name = 'work_mem'
    
    UNION ALL
    
    SELECT 
        'Checkpoints'::TEXT,
        'checkpoint_timeout'::TEXT,
        setting,
        '5-15 minutes'::TEXT,
        CASE 
            WHEN setting::int BETWEEN 300 AND 900 THEN 'GOOD'
            ELSE 'REVIEW'
        END
    FROM pg_settings WHERE name = 'checkpoint_timeout';
END;
$$ LANGUAGE plpgsql;

-- Check configuration
SELECT * FROM validate_performance_config();
```

### Monitoring Best Practices
```sql
-- Create monitoring checklist
CREATE OR REPLACE VIEW monitoring_checklist AS
SELECT 
    'pg_stat_statements enabled' AS check_item,
    CASE 
        WHEN EXISTS (SELECT 1 FROM pg_extension WHERE extname = 'pg_stat_statements') 
        THEN 'PASS' 
        ELSE 'FAIL' 
    END AS status,
    'Essential for query performance monitoring' AS importance
UNION ALL
SELECT 
    'Regular VACUUM/ANALYZE scheduled',
    CASE 
        WHEN (SELECT setting FROM pg_settings WHERE name = 'autovacuum') = 'on' 
        THEN 'PASS' 
        ELSE 'FAIL' 
    END,
    'Prevents table bloat and maintains statistics'
UNION ALL
SELECT 
    'Sufficient maintenance_work_mem',
    CASE 
        WHEN (SELECT setting::bigint FROM pg_settings WHERE name = 'maintenance_work_mem') >= pg_size_bytes('256MB')
        THEN 'PASS' 
        ELSE 'REVIEW' 
    END,
    'Improves VACUUM and index creation performance';

-- View monitoring checklist
SELECT * FROM monitoring_checklist;
```

---

## Next Steps

### Module Completion Checklist
- [ ] Analyze current system performance bottlenecks
- [ ] Optimize memory and storage configuration
- [ ] Implement query optimization strategies
- [ ] Review and optimize index usage
- [ ] Configure connection pooling appropriately
- [ ] Set up performance monitoring and alerting

### Immediate Next Steps
1. **Implement** performance monitoring dashboard
2. **Optimize** identified performance bottlenecks
3. **Configure** appropriate memory settings
4. **Move to Module 14**: [High Availability](14-high-availability.md)

---

## Module Summary

✅ **What You've Learned:**
- Comprehensive performance analysis techniques
- Memory and storage optimization strategies
- Query and index optimization methods
- Connection management and tuning
- Performance monitoring and alerting systems
- Benchmarking and testing procedures

✅ **Skills Acquired:**
- Identify and resolve performance bottlenecks
- Configure PostgreSQL for optimal performance
- Implement effective monitoring systems
- Optimize queries and indexes
- Conduct performance testing and benchmarking

✅ **Ready for Next Module:**
You're now ready to implement high availability solutions in [Module 14: High Availability](14-high-availability.md).

---

## Quick Reference

### Key Performance Queries
```sql
-- Top slow queries
SELECT query, total_time, calls, mean_time 
FROM pg_stat_statements 
ORDER BY total_time DESC LIMIT 10;

-- Buffer hit ratio
SELECT ROUND(100.0 * sum(blks_hit) / sum(blks_hit + blks_read), 2) 
FROM pg_stat_database;

-- Index usage
SELECT schemaname, tablename, indexname, idx_scan 
FROM pg_stat_user_indexes 
ORDER BY idx_scan;
```

### Configuration Parameters
```
# Memory settings
shared_buffers = 256MB          # 25% of RAM
work_mem = 4MB                  # Per connection
maintenance_work_mem = 64MB     # For maintenance ops

# Checkpoint settings
checkpoint_timeout = 5min       # Checkpoint frequency
checkpoint_completion_target = 0.7
```

---
*Continue your PostgreSQL journey with [High Availability](14-high-availability.md) →*
