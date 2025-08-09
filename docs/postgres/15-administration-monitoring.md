# Administration and Monitoring

## 🎯 Learning Objectives
By the end of this module, you will:
- Master PostgreSQL system administration tasks and procedures
- Implement comprehensive monitoring and alerting systems
- Analyze and interpret PostgreSQL logs for troubleshooting
- Perform routine maintenance operations and health checks
- Automate administrative tasks using scripts and tools
- Develop incident response procedures and documentation

## 📚 Table of Contents
1. [System Administration](#system-administration)
2. [Log Analysis](#log-analysis)
3. [Performance Monitoring](#performance-monitoring)
4. [Health Checks](#health-checks)
5. [Maintenance Procedures](#maintenance-procedures)
6. [Automation and Scripting](#automation-and-scripting)
7. [Incident Response](#incident-response)
8. [Monitoring Tools](#monitoring-tools)
9. [Hands-On Exercises](#hands-on-exercises)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)

---

## System Administration

### Daily Administrative Tasks

#### User and Role Management

```sql
-- Monitor user connections and activity
SELECT 
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    state_change,
    query
FROM pg_stat_activity 
WHERE state != 'idle'
ORDER BY query_start;

-- Check user privileges and roles
SELECT 
    r.rolname,
    r.rolsuper,
    r.rolinherit,
    r.rolcreaterole,
    r.rolcreatedb,
    r.rolcanlogin,
    r.rolconnlimit,
    r.rolvaliduntil,
    ARRAY(SELECT b.rolname 
          FROM pg_catalog.pg_auth_members m 
          JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid) 
          WHERE m.member = r.oid) as memberof
FROM pg_catalog.pg_roles r
WHERE r.rolname !~ '^pg_'
ORDER BY 1;

-- Create monitoring user with minimal privileges
CREATE USER monitor_user WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE myapp TO monitor_user;
GRANT USAGE ON SCHEMA public TO monitor_user;
GRANT SELECT ON pg_stat_user_tables TO monitor_user;
GRANT SELECT ON pg_stat_user_indexes TO monitor_user;
```

#### Database Maintenance Commands

```sql
-- Check database sizes
SELECT 
    datname,
    pg_size_pretty(pg_database_size(datname)) as size,
    pg_database_size(datname) as size_bytes
FROM pg_database 
WHERE datname NOT IN ('template0', 'template1', 'postgres')
ORDER BY pg_database_size(datname) DESC;

-- Table and index maintenance
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) as indexes_size,
    n_tup_ins + n_tup_upd + n_tup_del as total_writes,
    n_tup_hot_upd,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables 
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Connection monitoring
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    client_hostname,
    client_port,
    backend_start,
    xact_start,
    query_start,
    state_change,
    wait_event_type,
    wait_event,
    state,
    backend_xid,
    backend_xmin,
    LEFT(query, 100) as query_preview
FROM pg_stat_activity
ORDER BY backend_start;
```

#### System Resource Monitoring

```bash
#!/bin/bash
# postgres_system_check.sh

echo "=== PostgreSQL System Administration Check ==="
echo "Date: $(date)"
echo "Hostname: $(hostname)"
echo ""

# Check PostgreSQL service status
echo "=== Service Status ==="
systemctl status postgresql --no-pager

echo ""
echo "=== Process Information ==="
ps aux | grep postgres | grep -v grep

echo ""
echo "=== Memory Usage ==="
free -h

echo ""
echo "=== Disk Usage ==="
df -h | grep -E "(postgres|pgdata)"

echo ""
echo "=== Network Connections ==="
netstat -tulpn | grep postgres

echo ""
echo "=== System Load ==="
uptime
```

---

## Log Analysis

### PostgreSQL Log Configuration

#### Configure Comprehensive Logging

```sql
-- Configure logging parameters
ALTER SYSTEM SET log_destination = 'stderr,csvlog';
ALTER SYSTEM SET logging_collector = on;
ALTER SYSTEM SET log_directory = 'log';
ALTER SYSTEM SET log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log';
ALTER SYSTEM SET log_rotation_age = '1d';
ALTER SYSTEM SET log_rotation_size = '100MB';
ALTER SYSTEM SET log_truncate_on_rotation = on;

-- Log what to capture
ALTER SYSTEM SET log_min_messages = 'warning';
ALTER SYSTEM SET log_min_error_statement = 'error';
ALTER SYSTEM SET log_min_duration_statement = '1000';  -- Log slow queries > 1s
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_disconnections = on;
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET log_temp_files = 0;
ALTER SYSTEM SET log_autovacuum_min_duration = 0;
ALTER SYSTEM SET log_checkpoints = on;

-- Log statement details
ALTER SYSTEM SET log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h ';
ALTER SYSTEM SET log_statement = 'none';  -- Use log_min_duration_statement instead
ALTER SYSTEM SET log_duration = off;      -- Use log_min_duration_statement instead

-- Reload configuration
SELECT pg_reload_conf();
```

#### Log Analysis Scripts

```bash
#!/bin/bash
# analyze_postgres_logs.sh

LOG_DIR="/var/lib/postgresql/15/main/log"
TODAY=$(date +%Y-%m-%d)

echo "=== PostgreSQL Log Analysis for $TODAY ==="

# Find slow queries
echo "=== Slow Queries (>5 seconds) ==="
grep -n "duration:" $LOG_DIR/postgresql-$TODAY*.log | \
awk -F: '$4 > 5000 {print $0}' | \
head -20

echo ""
echo "=== Connection Errors ==="
grep -i "connection\|authentication\|permission" $LOG_DIR/postgresql-$TODAY*.log | \
head -20

echo ""
echo "=== Lock Waits ==="
grep -i "lock" $LOG_DIR/postgresql-$TODAY*.log | head -10

echo ""
echo "=== Vacuum Activity ==="
grep -i "vacuum" $LOG_DIR/postgresql-$TODAY*.log | tail -10

echo ""
echo "=== Error Summary ==="
grep -i "error\|fatal\|panic" $LOG_DIR/postgresql-$TODAY*.log | \
awk '{print $1,$2,$3}' | sort | uniq -c | sort -nr

echo ""
echo "=== Top Client IPs ==="
grep "connection authorized" $LOG_DIR/postgresql-$TODAY*.log | \
awk '{print $NF}' | sort | uniq -c | sort -nr | head -10
```

#### CSV Log Analysis

```python
#!/usr/bin/env python3
# analyze_csv_logs.py

import pandas as pd
import sys
from datetime import datetime, timedelta

def analyze_csv_logs(log_file):
    """Analyze PostgreSQL CSV format logs"""
    
    # CSV columns for PostgreSQL logs
    columns = [
        'log_time', 'user_name', 'database_name', 'process_id', 'connection_from',
        'session_id', 'session_line_num', 'command_tag', 'session_start_time',
        'virtual_transaction_id', 'transaction_id', 'error_severity',
        'sql_state_code', 'message', 'detail', 'hint', 'internal_query',
        'internal_query_pos', 'context', 'query', 'query_pos', 'location',
        'application_name'
    ]
    
    try:
        df = pd.read_csv(log_file, names=columns, parse_dates=['log_time'])
        
        print(f"=== PostgreSQL CSV Log Analysis ===")
        print(f"Log file: {log_file}")
        print(f"Total entries: {len(df)}")
        print(f"Date range: {df['log_time'].min()} to {df['log_time'].max()}")
        print()
        
        # Error severity distribution
        print("=== Error Severity Distribution ===")
        severity_counts = df['error_severity'].value_counts()
        print(severity_counts)
        print()
        
        # Top databases by activity
        print("=== Top Databases by Activity ===")
        db_activity = df['database_name'].value_counts().head(10)
        print(db_activity)
        print()
        
        # Top users by activity
        print("=== Top Users by Activity ===")
        user_activity = df['user_name'].value_counts().head(10)
        print(user_activity)
        print()
        
        # Recent errors
        print("=== Recent Errors (last 24 hours) ===")
        recent_errors = df[
            (df['log_time'] > datetime.now() - timedelta(days=1)) &
            (df['error_severity'].isin(['ERROR', 'FATAL', 'PANIC']))
        ][['log_time', 'error_severity', 'message']].head(10)
        print(recent_errors.to_string(index=False))
        
    except Exception as e:
        print(f"Error analyzing log file: {e}")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python3 analyze_csv_logs.py <csv_log_file>")
        sys.exit(1)
    
    analyze_csv_logs(sys.argv[1])
```

---

## Performance Monitoring

### Real-Time Performance Monitoring

#### Key Performance Queries

```sql
-- Current database activity overview
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    now() - query_start as duration,
    wait_event_type,
    wait_event,
    LEFT(query, 100) as query_preview
FROM pg_stat_activity 
WHERE state != 'idle' 
    AND pid != pg_backend_pid()
ORDER BY query_start;

-- Lock monitoring
SELECT 
    l.locktype,
    l.database,
    l.relation::regclass,
    l.page,
    l.tuple,
    l.virtualxid,
    l.transactionid,
    l.classid,
    l.objid,
    l.objsubid,
    l.virtualtransaction,
    l.pid,
    l.mode,
    l.granted,
    a.usename,
    a.query,
    a.query_start
FROM pg_locks l
LEFT JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted
ORDER BY l.pid;

-- Table statistics and performance
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
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY seq_scan + idx_scan DESC;

-- Index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Missing indexes (tables with high sequential scans)
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    seq_tup_read / seq_scan as avg_tup_read,
    idx_scan,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_stat_user_tables
WHERE seq_scan > 0
    AND seq_tup_read / seq_scan > 10000
    AND idx_scan < seq_scan
ORDER BY seq_tup_read DESC;
```

#### Performance Monitoring Views

```sql
-- Create monitoring views for easier analysis
CREATE OR REPLACE VIEW monitoring.active_sessions AS
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    now() - query_start as duration,
    wait_event_type,
    wait_event,
    LEFT(query, 200) as query_preview
FROM pg_stat_activity 
WHERE state != 'idle' 
    AND pid != pg_backend_pid();

CREATE OR REPLACE VIEW monitoring.table_sizes AS
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) as indexes_size,
    pg_total_relation_size(schemaname||'.'||tablename) as total_bytes
FROM pg_stat_user_tables 
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

CREATE OR REPLACE VIEW monitoring.replication_status AS
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    sync_priority,
    pg_wal_lsn_diff(sent_lsn, write_lsn) as write_lag_bytes,
    pg_wal_lsn_diff(write_lsn, flush_lsn) as flush_lag_bytes,
    pg_wal_lsn_diff(flush_lsn, replay_lsn) as replay_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

---

## Health Checks

### Automated Health Check Scripts

#### Comprehensive Health Check

```bash
#!/bin/bash
# postgres_health_check.sh

# Configuration
DB_NAME="myapp"
DB_USER="postgres"
THRESHOLD_CONNECTIONS=80
THRESHOLD_REPLICATION_LAG=30
LOG_FILE="/var/log/postgres_health.log"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Logging function
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

# Check function
check_status() {
    local check_name="$1"
    local status="$2"
    local message="$3"
    
    if [ "$status" = "OK" ]; then
        echo -e "${GREEN}✓${NC} $check_name: $message"
        log_message "OK - $check_name: $message"
    elif [ "$status" = "WARNING" ]; then
        echo -e "${YELLOW}⚠${NC} $check_name: $message"
        log_message "WARNING - $check_name: $message"
    else
        echo -e "${RED}✗${NC} $check_name: $message"
        log_message "CRITICAL - $check_name: $message"
    fi
}

echo "=== PostgreSQL Health Check ==="
echo "Date: $(date)"
echo ""

# 1. Service Status
if systemctl is-active --quiet postgresql; then
    check_status "Service Status" "OK" "PostgreSQL service is running"
else
    check_status "Service Status" "CRITICAL" "PostgreSQL service is not running"
    exit 1
fi

# 2. Database Connectivity
if psql -h localhost -U $DB_USER -d $DB_NAME -c "SELECT 1;" >/dev/null 2>&1; then
    check_status "Database Connectivity" "OK" "Can connect to database"
else
    check_status "Database Connectivity" "CRITICAL" "Cannot connect to database"
fi

# 3. Connection Count
CONNECTION_COUNT=$(psql -h localhost -U $DB_USER -d $DB_NAME -t -c "SELECT count(*) FROM pg_stat_activity;" 2>/dev/null | xargs)
MAX_CONNECTIONS=$(psql -h localhost -U $DB_USER -d $DB_NAME -t -c "SHOW max_connections;" 2>/dev/null | xargs)

if [ ! -z "$CONNECTION_COUNT" ] && [ ! -z "$MAX_CONNECTIONS" ]; then
    CONNECTION_PCT=$((CONNECTION_COUNT * 100 / MAX_CONNECTIONS))
    if [ $CONNECTION_PCT -gt $THRESHOLD_CONNECTIONS ]; then
        check_status "Connection Usage" "WARNING" "$CONNECTION_COUNT/$MAX_CONNECTIONS connections (${CONNECTION_PCT}%)"
    else
        check_status "Connection Usage" "OK" "$CONNECTION_COUNT/$MAX_CONNECTIONS connections (${CONNECTION_PCT}%)"
    fi
fi

# 4. Replication Status (if applicable)
REPLICATION_COUNT=$(psql -h localhost -U $DB_USER -d $DB_NAME -t -c "SELECT count(*) FROM pg_stat_replication;" 2>/dev/null | xargs)
if [ "$REPLICATION_COUNT" -gt 0 ]; then
    MAX_LAG=$(psql -h localhost -U $DB_USER -d $DB_NAME -t -c "SELECT COALESCE(EXTRACT(EPOCH FROM MAX(replay_lag)), 0) FROM pg_stat_replication;" 2>/dev/null | xargs)
    if [ ! -z "$MAX_LAG" ]; then
        MAX_LAG_INT=${MAX_LAG%.*}  # Remove decimal part
        if [ $MAX_LAG_INT -gt $THRESHOLD_REPLICATION_LAG ]; then
            check_status "Replication Lag" "WARNING" "Maximum lag: ${MAX_LAG}s"
        else
            check_status "Replication Lag" "OK" "Maximum lag: ${MAX_LAG}s"
        fi
    fi
fi

# 5. Disk Space
DISK_USAGE=$(df /var/lib/postgresql | awk 'NR==2 {print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 90 ]; then
    check_status "Disk Space" "CRITICAL" "Disk usage: ${DISK_USAGE}%"
elif [ $DISK_USAGE -gt 80 ]; then
    check_status "Disk Space" "WARNING" "Disk usage: ${DISK_USAGE}%"
else
    check_status "Disk Space" "OK" "Disk usage: ${DISK_USAGE}%"
fi

# 6. Long Running Queries
LONG_QUERIES=$(psql -h localhost -U $DB_USER -d $DB_NAME -t -c "SELECT count(*) FROM pg_stat_activity WHERE state = 'active' AND now() - query_start > interval '5 minutes';" 2>/dev/null | xargs)
if [ "$LONG_QUERIES" -gt 0 ]; then
    check_status "Long Running Queries" "WARNING" "$LONG_QUERIES queries running >5 minutes"
else
    check_status "Long Running Queries" "OK" "No long running queries"
fi

# 7. Blocked Queries
BLOCKED_QUERIES=$(psql -h localhost -U $DB_USER -d $DB_NAME -t -c "SELECT count(*) FROM pg_locks WHERE NOT granted;" 2>/dev/null | xargs)
if [ "$BLOCKED_QUERIES" -gt 0 ]; then
    check_status "Blocked Queries" "WARNING" "$BLOCKED_QUERIES blocked queries"
else
    check_status "Blocked Queries" "OK" "No blocked queries"
fi

# 8. Archive Status (if archiving enabled)
ARCHIVE_FAILED=$(psql -h localhost -U $DB_USER -d $DB_NAME -t -c "SELECT archived_count, failed_count FROM pg_stat_archiver;" 2>/dev/null)
if [ ! -z "$ARCHIVE_FAILED" ]; then
    FAILED_COUNT=$(echo $ARCHIVE_FAILED | awk '{print $4}')
    if [ "$FAILED_COUNT" -gt 0 ]; then
        check_status "WAL Archiving" "WARNING" "$FAILED_COUNT failed archives"
    else
        check_status "WAL Archiving" "OK" "Archiving working properly"
    fi
fi

echo ""
echo "Health check completed at $(date)"
```

#### Nagios-Compatible Check

```bash
#!/bin/bash
# check_postgres_nagios.sh - Nagios compatible PostgreSQL check

# Exit codes for Nagios
STATE_OK=0
STATE_WARNING=1
STATE_CRITICAL=2
STATE_UNKNOWN=3

# Parse command line arguments
while [[ $# -gt 0 ]]; do
    case $1 in
        -H|--host)
            HOST="$2"
            shift 2
            ;;
        -p|--port)
            PORT="$2"
            shift 2
            ;;
        -d|--database)
            DATABASE="$2"
            shift 2
            ;;
        -U|--username)
            USERNAME="$2"
            shift 2
            ;;
        -w|--warning)
            WARNING="$2"
            shift 2
            ;;
        -c|--critical)
            CRITICAL="$2"
            shift 2
            ;;
        -t|--type)
            CHECK_TYPE="$2"
            shift 2
            ;;
        *)
            echo "Unknown option $1"
            exit $STATE_UNKNOWN
            ;;
    esac
done

# Default values
HOST=${HOST:-localhost}
PORT=${PORT:-5432}
DATABASE=${DATABASE:-postgres}
USERNAME=${USERNAME:-postgres}
WARNING=${WARNING:-80}
CRITICAL=${CRITICAL:-90}
CHECK_TYPE=${CHECK_TYPE:-connections}

# Connection string
PSQL_CMD="psql -h $HOST -p $PORT -U $USERNAME -d $DATABASE -t -A"

case $CHECK_TYPE in
    connections)
        CURRENT=$($PSQL_CMD -c "SELECT count(*) FROM pg_stat_activity;" 2>/dev/null)
        MAX=$($PSQL_CMD -c "SHOW max_connections;" 2>/dev/null)
        
        if [ -z "$CURRENT" ] || [ -z "$MAX" ]; then
            echo "UNKNOWN - Cannot retrieve connection count"
            exit $STATE_UNKNOWN
        fi
        
        PERCENTAGE=$((CURRENT * 100 / MAX))
        PERFDATA="|connections=$CURRENT;$((MAX * WARNING / 100));$((MAX * CRITICAL / 100));0;$MAX"
        
        if [ $PERCENTAGE -ge $CRITICAL ]; then
            echo "CRITICAL - $CURRENT/$MAX connections ($PERCENTAGE%)$PERFDATA"
            exit $STATE_CRITICAL
        elif [ $PERCENTAGE -ge $WARNING ]; then
            echo "WARNING - $CURRENT/$MAX connections ($PERCENTAGE%)$PERFDATA"
            exit $STATE_WARNING
        else
            echo "OK - $CURRENT/$MAX connections ($PERCENTAGE%)$PERFDATA"
            exit $STATE_OK
        fi
        ;;
        
    replication_lag)
        LAG=$($PSQL_CMD -c "SELECT COALESCE(EXTRACT(EPOCH FROM MAX(replay_lag)), 0) FROM pg_stat_replication;" 2>/dev/null)
        
        if [ -z "$LAG" ]; then
            echo "UNKNOWN - Cannot retrieve replication lag"
            exit $STATE_UNKNOWN
        fi
        
        LAG_INT=${LAG%.*}  # Remove decimal part
        PERFDATA="|lag=${LAG}s;$WARNING;$CRITICAL;0;"
        
        if [ $LAG_INT -ge $CRITICAL ]; then
            echo "CRITICAL - Replication lag: ${LAG}s$PERFDATA"
            exit $STATE_CRITICAL
        elif [ $LAG_INT -ge $WARNING ]; then
            echo "WARNING - Replication lag: ${LAG}s$PERFDATA"
            exit $STATE_WARNING
        else
            echo "OK - Replication lag: ${LAG}s$PERFDATA"
            exit $STATE_OK
        fi
        ;;
        
    *)
        echo "UNKNOWN - Invalid check type: $CHECK_TYPE"
        exit $STATE_UNKNOWN
        ;;
esac
```

---

## Maintenance Procedures

### Routine Maintenance Tasks

#### Vacuum and Analyze Automation

```sql
-- Create maintenance schema for procedures
CREATE SCHEMA IF NOT EXISTS maintenance;

-- Automatic vacuum and analyze procedure
CREATE OR REPLACE FUNCTION maintenance.auto_maintenance()
RETURNS TABLE(
    schema_name text,
    table_name text,
    action text,
    duration interval
) AS $$
DECLARE
    rec record;
    start_time timestamp;
    end_time timestamp;
BEGIN
    -- Tables that need vacuum based on dead tuple threshold
    FOR rec IN 
        SELECT schemaname, tablename
        FROM pg_stat_user_tables
        WHERE n_dead_tup > 1000
        AND (n_dead_tup::float / GREATEST(n_live_tup, 1)) > 0.1
        ORDER BY n_dead_tup DESC
    LOOP
        start_time := clock_timestamp();
        EXECUTE format('VACUUM (ANALYZE, VERBOSE) %I.%I', rec.schemaname, rec.tablename);
        end_time := clock_timestamp();
        
        RETURN QUERY SELECT 
            rec.schemaname::text,
            rec.tablename::text,
            'VACUUM ANALYZE'::text,
            end_time - start_time;
    END LOOP;
    
    -- Tables that need analyze based on modification threshold
    FOR rec IN 
        SELECT schemaname, tablename
        FROM pg_stat_user_tables
        WHERE (n_tup_ins + n_tup_upd + n_tup_del) > 1000
        AND (last_analyze IS NULL OR last_analyze < now() - interval '1 day')
        ORDER BY (n_tup_ins + n_tup_upd + n_tup_del) DESC
    LOOP
        start_time := clock_timestamp();
        EXECUTE format('ANALYZE VERBOSE %I.%I', rec.schemaname, rec.tablename);
        end_time := clock_timestamp();
        
        RETURN QUERY SELECT 
            rec.schemaname::text,
            rec.tablename::text,
            'ANALYZE'::text,
            end_time - start_time;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Run maintenance
SELECT * FROM maintenance.auto_maintenance();
```

#### Index Maintenance

```sql
-- Find and rebuild bloated indexes
CREATE OR REPLACE FUNCTION maintenance.rebuild_bloated_indexes(
    bloat_threshold float DEFAULT 2.0
)
RETURNS TABLE(
    schema_name text,
    table_name text,
    index_name text,
    bloat_ratio float,
    action text,
    duration interval
) AS $$
DECLARE
    rec record;
    start_time timestamp;
    end_time timestamp;
    index_bloat float;
BEGIN
    -- Query to find bloated indexes (simplified version)
    FOR rec IN 
        SELECT 
            schemaname,
            tablename,
            indexname,
            pg_relation_size(indexrelid) as index_size
        FROM pg_stat_user_indexes
        WHERE pg_relation_size(indexrelid) > 100 * 1024 * 1024  -- > 100MB
        ORDER BY pg_relation_size(indexrelid) DESC
    LOOP
        -- Simple bloat estimation (actual bloat calculation is complex)
        -- This is a simplified approach
        SELECT INTO index_bloat 
            CASE 
                WHEN idx_scan = 0 THEN 10.0  -- Unused index
                ELSE 1.0 + (pg_relation_size(indexrelid)::float / 
                           GREATEST(idx_tup_read, 1)::float / 1000)
            END
        FROM pg_stat_user_indexes 
        WHERE schemaname = rec.schemaname 
        AND tablename = rec.tablename 
        AND indexname = rec.indexname;
        
        IF index_bloat > bloat_threshold THEN
            start_time := clock_timestamp();
            EXECUTE format('REINDEX INDEX CONCURRENTLY %I.%I', rec.schemaname, rec.indexname);
            end_time := clock_timestamp();
            
            RETURN QUERY SELECT 
                rec.schemaname::text,
                rec.tablename::text,
                rec.indexname::text,
                index_bloat,
                'REINDEX CONCURRENTLY'::text,
                end_time - start_time;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

#### Automated Maintenance Script

```bash
#!/bin/bash
# automated_maintenance.sh

# Configuration
DB_NAME="myapp"
DB_USER="postgres"
LOG_DIR="/var/log/postgresql"
BACKUP_DIR="/var/backups/postgresql"
RETENTION_DAYS=7

# Create log entry
log_entry() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> $LOG_DIR/maintenance.log
}

log_entry "Starting automated maintenance"

# 1. Update table statistics
log_entry "Updating table statistics"
psql -U $DB_USER -d $DB_NAME -c "
SELECT 'Analyzing: ' || schemaname || '.' || tablename as info 
FROM pg_stat_user_tables 
WHERE last_analyze < now() - interval '1 day' OR last_analyze IS NULL;

DO \$\$
DECLARE
    rec record;
BEGIN
    FOR rec IN 
        SELECT schemaname, tablename 
        FROM pg_stat_user_tables 
        WHERE last_analyze < now() - interval '1 day' OR last_analyze IS NULL
    LOOP
        EXECUTE 'ANALYZE ' || quote_ident(rec.schemaname) || '.' || quote_ident(rec.tablename);
        RAISE NOTICE 'Analyzed %.%', rec.schemaname, rec.tablename;
    END LOOP;
END \$\$;
"

# 2. Clean up old WAL files (if archiving is disabled)
log_entry "Checking WAL file cleanup"
psql -U $DB_USER -d $DB_NAME -c "SELECT pg_switch_wal();"

# 3. Monitor long-running transactions
log_entry "Checking for long-running transactions"
psql -U $DB_USER -d $DB_NAME -c "
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    now() - xact_start as xact_duration,
    state,
    LEFT(query, 100) as query_preview
FROM pg_stat_activity 
WHERE xact_start < now() - interval '30 minutes'
    AND state != 'idle';
"

# 4. Check for unused indexes
log_entry "Checking for unused indexes"
psql -U $DB_USER -d $DB_NAME -c "
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes 
WHERE idx_scan < 10
    AND pg_relation_size(indexrelid) > 1024*1024  -- > 1MB
ORDER BY pg_relation_size(indexrelid) DESC;
"

# 5. Backup database (if needed)
if [ ! -f "$BACKUP_DIR/daily_backup_$(date +%Y%m%d).sql.gz" ]; then
    log_entry "Creating daily backup"
    mkdir -p $BACKUP_DIR
    pg_dump -U $DB_USER $DB_NAME | gzip > $BACKUP_DIR/daily_backup_$(date +%Y%m%d).sql.gz
fi

# 6. Clean old backups
log_entry "Cleaning old backups"
find $BACKUP_DIR -name "daily_backup_*.sql.gz" -mtime +$RETENTION_DAYS -delete

# 7. Clean old log files
log_entry "Cleaning old log files"
find $LOG_DIR -name "postgresql-*.log" -mtime +$RETENTION_DAYS -delete

log_entry "Maintenance completed"
```

---

## Automation and Scripting

### Monitoring Scripts

#### Performance Monitoring Script

```python
#!/usr/bin/env python3
# postgres_monitor.py

import psycopg2
import json
import time
import argparse
from datetime import datetime

class PostgreSQLMonitor:
    def __init__(self, host, port, database, username, password):
        self.conn_params = {
            'host': host,
            'port': port,
            'database': database,
            'user': username,
            'password': password
        }
        
    def get_connection(self):
        return psycopg2.connect(**self.conn_params)
    
    def collect_metrics(self):
        """Collect various PostgreSQL metrics"""
        metrics = {
            'timestamp': datetime.now().isoformat(),
            'database_stats': self.get_database_stats(),
            'table_stats': self.get_table_stats(),
            'index_stats': self.get_index_stats(),
            'replication_stats': self.get_replication_stats(),
            'activity_stats': self.get_activity_stats(),
            'system_stats': self.get_system_stats()
        }
        return metrics
    
    def get_database_stats(self):
        """Get database-level statistics"""
        query = """
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
            conflicts,
            temp_files,
            temp_bytes,
            deadlocks,
            pg_database_size(datname) as size_bytes
        FROM pg_stat_database 
        WHERE datname NOT IN ('template0', 'template1');
        """
        
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(query)
                columns = [desc[0] for desc in cur.description]
                return [dict(zip(columns, row)) for row in cur.fetchall()]
    
    def get_table_stats(self):
        """Get table-level statistics"""
        query = """
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
            n_live_tup,
            n_dead_tup,
            n_mod_since_analyze,
            EXTRACT(EPOCH FROM last_vacuum) as last_vacuum_epoch,
            EXTRACT(EPOCH FROM last_autovacuum) as last_autovacuum_epoch,
            EXTRACT(EPOCH FROM last_analyze) as last_analyze_epoch,
            EXTRACT(EPOCH FROM last_autoanalyze) as last_autoanalyze_epoch,
            vacuum_count,
            autovacuum_count,
            analyze_count,
            autoanalyze_count,
            pg_total_relation_size(schemaname||'.'||tablename) as total_size_bytes
        FROM pg_stat_user_tables
        ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
        LIMIT 50;
        """
        
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(query)
                columns = [desc[0] for desc in cur.description]
                return [dict(zip(columns, row)) for row in cur.fetchall()]
    
    def get_index_stats(self):
        """Get index usage statistics"""
        query = """
        SELECT 
            schemaname,
            tablename,
            indexname,
            idx_scan,
            idx_tup_read,
            idx_tup_fetch,
            pg_relation_size(indexrelid) as size_bytes
        FROM pg_stat_user_indexes
        ORDER BY idx_scan DESC
        LIMIT 50;
        """
        
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(query)
                columns = [desc[0] for desc in cur.description]
                return [dict(zip(columns, row)) for row in cur.fetchall()]
    
    def get_replication_stats(self):
        """Get replication statistics"""
        query = """
        SELECT 
            application_name,
            client_addr,
            client_hostname,
            client_port,
            state,
            sync_state,
            sync_priority,
            EXTRACT(EPOCH FROM write_lag) as write_lag_seconds,
            EXTRACT(EPOCH FROM flush_lag) as flush_lag_seconds,
            EXTRACT(EPOCH FROM replay_lag) as replay_lag_seconds
        FROM pg_stat_replication;
        """
        
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(query)
                columns = [desc[0] for desc in cur.description]
                return [dict(zip(columns, row)) for row in cur.fetchall()]
    
    def get_activity_stats(self):
        """Get current activity statistics"""
        query = """
        SELECT 
            state,
            COUNT(*) as count,
            AVG(EXTRACT(EPOCH FROM now() - query_start)) as avg_duration_seconds
        FROM pg_stat_activity 
        WHERE pid != pg_backend_pid()
        GROUP BY state;
        """
        
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(query)
                columns = [desc[0] for desc in cur.description]
                return [dict(zip(columns, row)) for row in cur.fetchall()]
    
    def get_system_stats(self):
        """Get system-level statistics"""
        query = """
        SELECT 
            (SELECT setting FROM pg_settings WHERE name = 'max_connections') as max_connections,
            (SELECT count(*) FROM pg_stat_activity) as current_connections,
            (SELECT pg_database_size(current_database())) as current_db_size,
            (SELECT sum(pg_database_size(datname)) FROM pg_database) as total_db_size;
        """
        
        with self.get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(query)
                columns = [desc[0] for desc in cur.description]
                result = cur.fetchone()
                return dict(zip(columns, result)) if result else {}

def main():
    parser = argparse.ArgumentParser(description='PostgreSQL Monitoring Script')
    parser.add_argument('--host', default='localhost', help='Database host')
    parser.add_argument('--port', default='5432', help='Database port')
    parser.add_argument('--database', required=True, help='Database name')
    parser.add_argument('--username', required=True, help='Database username')
    parser.add_argument('--password', required=True, help='Database password')
    parser.add_argument('--interval', type=int, default=60, help='Collection interval in seconds')
    parser.add_argument('--output', default='stdout', help='Output file (default: stdout)')
    
    args = parser.parse_args()
    
    monitor = PostgreSQLMonitor(
        args.host, args.port, args.database, 
        args.username, args.password
    )
    
    try:
        while True:
            metrics = monitor.collect_metrics()
            
            if args.output == 'stdout':
                print(json.dumps(metrics, indent=2, default=str))
            else:
                with open(args.output, 'a') as f:
                    f.write(json.dumps(metrics, default=str) + '\n')
            
            time.sleep(args.interval)
            
    except KeyboardInterrupt:
        print("\nMonitoring stopped by user")
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    main()
```

---

## Incident Response

### Incident Response Procedures

#### Database Recovery Procedures

```bash
#!/bin/bash
# incident_response.sh

# PostgreSQL Incident Response Script
# Usage: ./incident_response.sh [incident_type]

INCIDENT_TYPE=${1:-"general"}
DB_NAME="myapp"
DB_USER="postgres"
BACKUP_DIR="/var/backups/postgresql"
LOG_DIR="/var/log/postgresql"

# Logging
log_incident() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') INCIDENT: $1" | tee -a $LOG_DIR/incidents.log
}

case $INCIDENT_TYPE in
    "connection_exhaustion")
        log_incident "Handling connection exhaustion incident"
        
        # Kill idle connections
        psql -U $DB_USER -d $DB_NAME -c "
        SELECT pg_terminate_backend(pid) 
        FROM pg_stat_activity 
        WHERE state = 'idle' 
        AND state_change < now() - interval '10 minutes'
        AND pid != pg_backend_pid();
        "
        
        # Kill long-running queries
        psql -U $DB_USER -d $DB_NAME -c "
        SELECT 
            pid,
            usename,
            query_start,
            state,
            LEFT(query, 100) as query
        FROM pg_stat_activity 
        WHERE state = 'active' 
        AND query_start < now() - interval '30 minutes'
        AND pid != pg_backend_pid();
        "
        
        echo "Review the above queries and manually terminate if needed:"
        echo "SELECT pg_terminate_backend([pid]);"
        ;;
        
    "replication_failure")
        log_incident "Handling replication failure incident"
        
        # Check replication status
        psql -U $DB_USER -d $DB_NAME -c "
        SELECT 
            application_name,
            client_addr,
            state,
            sync_state,
            write_lag,
            flush_lag,
            replay_lag
        FROM pg_stat_replication;
        "
        
        # Check for replication slots
        psql -U $DB_USER -d $DB_NAME -c "
        SELECT 
            slot_name,
            plugin,
            slot_type,
            datoid,
            database,
            active,
            active_pid,
            xmin,
            catalog_xmin,
            restart_lsn,
            confirmed_flush_lsn
        FROM pg_replication_slots;
        "
        ;;
        
    "disk_full")
        log_incident "Handling disk full incident"
        
        # Check disk usage
        df -h
        
        # Find large files
        find /var/lib/postgresql -type f -size +100M | head -10
        
        # Clean old WAL files if archiving is off
        echo "Consider cleaning old WAL files if archiving is disabled"
        echo "Check pg_wal directory: $(du -sh /var/lib/postgresql/*/main/pg_wal)"
        
        # Suggest cleanup actions
        echo "Potential cleanup actions:"
        echo "1. Clean old log files: find $LOG_DIR -name '*.log' -mtime +7 -delete"
        echo "2. Clean old backups: find $BACKUP_DIR -name '*.sql.gz' -mtime +30 -delete"
        echo "3. Vacuum large tables to reclaim space"
        ;;
        
    "corruption")
        log_incident "Handling data corruption incident"
        
        # Check for corruption
        psql -U $DB_USER -d $DB_NAME -c "
        DO \$\$
        DECLARE
            rec record;
        BEGIN
            FOR rec IN SELECT tablename FROM pg_tables WHERE schemaname = 'public'
            LOOP
                BEGIN
                    EXECUTE 'SELECT count(*) FROM ' || quote_ident(rec.tablename);
                    RAISE NOTICE 'Table % appears OK', rec.tablename;
                EXCEPTION
                    WHEN others THEN
                        RAISE WARNING 'Potential corruption in table %: %', rec.tablename, SQLERRM;
                END;
            END LOOP;
        END \$\$;
        "
        
        echo "If corruption is confirmed:"
        echo "1. Stop writes to affected tables"
        echo "2. Restore from latest backup"
        echo "3. Consider PITR if available"
        echo "4. Run pg_dump on good tables first"
        ;;
        
    *)
        log_incident "General incident response procedures"
        
        # General health check
        echo "=== System Status ==="
        systemctl status postgresql
        
        echo "=== Current Connections ==="
        psql -U $DB_USER -d $DB_NAME -c "
        SELECT count(*), state FROM pg_stat_activity GROUP BY state;
        "
        
        echo "=== Long Running Queries ==="
        psql -U $DB_USER -d $DB_NAME -c "
        SELECT 
            pid,
            usename,
            application_name,
            client_addr,
            state,
            query_start,
            now() - query_start as duration,
            LEFT(query, 100) as query_preview
        FROM pg_stat_activity 
        WHERE state != 'idle' 
        AND query_start < now() - interval '5 minutes'
        ORDER BY query_start;
        "
        
        echo "=== Lock Information ==="
        psql -U $DB_USER -d $DB_NAME -c "
        SELECT 
            l.locktype,
            l.database,
            l.relation::regclass,
            l.mode,
            l.granted,
            a.usename,
            a.query_start,
            LEFT(a.query, 50) as query
        FROM pg_locks l
        LEFT JOIN pg_stat_activity a ON l.pid = a.pid
        WHERE NOT l.granted;
        "
        
        echo "=== Recent Errors ==="
        tail -50 $LOG_DIR/postgresql-*.log | grep -i "error\|fatal\|panic"
        ;;
esac

log_incident "Incident response completed for: $INCIDENT_TYPE"
```

---

## Monitoring Tools

### Integration with Monitoring Systems

#### Prometheus Exporter Configuration

```yaml
# docker-compose.yml for monitoring stack
version: '3.8'

services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor_user:password@postgres:5432/myapp?sslmode=disable"
    ports:
      - "9187:9187"
    depends_on:
      - postgres

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  prometheus_data:
  grafana_data:
```

---

## Hands-On Exercises

### Exercise 1: Set Up Monitoring

**Objective**: Implement comprehensive PostgreSQL monitoring

**Steps**:
1. Configure PostgreSQL logging
2. Set up health check scripts
3. Create monitoring views
4. Test alerting mechanisms

**Verification**:
```sql
-- Check logging configuration
SHOW log_destination;
SHOW log_min_duration_statement;
SHOW logging_collector;

-- Test monitoring views
SELECT * FROM monitoring.active_sessions;
SELECT * FROM monitoring.table_sizes LIMIT 10;
```

### Exercise 2: Incident Response Simulation

**Objective**: Practice incident response procedures

**Steps**:
1. Simulate connection exhaustion
2. Practice recovery procedures
3. Document response time
4. Review logs for analysis

**Simulation**:
```bash
# Simulate high connection load
for i in {1..100}; do
  psql -h localhost -d myapp -c "SELECT pg_sleep(300);" &
done

# Run incident response
./incident_response.sh connection_exhaustion
```

### Exercise 3: Automated Maintenance

**Objective**: Implement automated maintenance procedures

**Steps**:
1. Set up maintenance scripts
2. Schedule using cron
3. Monitor execution
4. Review results

**Cron Setup**:
```bash
# Add to crontab
0 2 * * * /opt/scripts/automated_maintenance.sh
0 */6 * * * /opt/scripts/postgres_health_check.sh
```

---

## Best Practices

### Administration Best Practices

#### Security Practices
1. **Regular security audits** of users and permissions
2. **Monitor failed login attempts** and suspicious activity
3. **Use SSL/TLS** for all connections
4. **Implement row-level security** where appropriate
5. **Regular password rotation** for service accounts

#### Performance Practices
1. **Monitor query performance** continuously
2. **Regular index maintenance** and optimization
3. **Track growth trends** for capacity planning
4. **Optimize configuration** based on workload
5. **Use connection pooling** to manage resources

#### Operational Practices
1. **Automate routine tasks** to reduce human error
2. **Document all procedures** and keep them updated
3. **Regular disaster recovery testing**
4. **Maintain change logs** for all modifications
5. **Use version control** for scripts and configurations

### Monitoring Best Practices

#### Metrics to Monitor
- Connection count and usage patterns
- Query performance and slow queries
- Lock waits and deadlocks
- Replication lag and status
- Disk space and I/O performance
- Memory usage and cache hit ratios

#### Alerting Guidelines
- Set appropriate thresholds for different environments
- Use escalation procedures for critical alerts
- Avoid alert fatigue with proper filtering
- Test alerting mechanisms regularly
- Document response procedures for each alert type

---

## Next Steps

### Advanced Topics to Explore
1. **Cloud-Native Monitoring** - Kubernetes operators and cloud services
2. **Machine Learning Integration** - Predictive analytics for performance
3. **Advanced Automation** - Infrastructure as Code (IaC) approaches
4. **Multi-Cluster Management** - Managing distributed PostgreSQL environments
5. **Compliance and Auditing** - Meeting regulatory requirements

### Recommended Tools
- **Monitoring**: Prometheus + Grafana, Datadog, New Relic
- **Log Analysis**: ELK Stack, Splunk, Fluentd
- **Automation**: Ansible, Terraform, Kubernetes operators
- **Backup**: Barman, pgBackRest, WAL-G

### Community Resources
- PostgreSQL Administration Documentation
- Database Administration Forums and Communities
- PostgreSQL Conference Presentations
- Open Source Monitoring Solutions

---

## Summary

In this module, you've learned to:
- ✅ Perform comprehensive PostgreSQL system administration
- ✅ Implement effective monitoring and alerting systems
- ✅ Analyze logs and troubleshoot issues systematically
- ✅ Automate routine maintenance tasks and procedures
- ✅ Develop incident response procedures and documentation
- ✅ Integrate with enterprise monitoring solutions

You're now equipped to manage PostgreSQL databases at enterprise scale with proper monitoring, maintenance, and incident response capabilities.

**Continue to**: [Enterprise Patterns](16-enterprise-patterns.md) to learn advanced enterprise-grade PostgreSQL patterns and architectures.
