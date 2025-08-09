# Backup and Recovery

## 🎯 Learning Objectives
By the end of this module, you will:
- Implement comprehensive backup strategies for PostgreSQL
- Perform point-in-time recovery operations
- Set up continuous archiving and streaming replication
- Handle disaster recovery scenarios
- Monitor backup integrity and performance
- Automate backup and recovery processes

## 📚 Table of Contents
1. [Backup Strategies](#backup-strategies)
2. [Physical Backups](#physical-backups)
3. [Logical Backups](#logical-backups)
4. [Point-in-Time Recovery](#point-in-time-recovery)
5. [Continuous Archiving](#continuous-archiving)
6. [Streaming Replication](#streaming-replication)
7. [Backup Monitoring](#backup-monitoring)
8. [Disaster Recovery](#disaster-recovery)
9. [Hands-On Exercises](#hands-on-exercises)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)

---

## Backup Strategies

### Backup Types Overview

#### Understanding Backup Methods
```sql
-- Check current database size and activity
SELECT 
    pg_database.datname AS database_name,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size,
    numbackends AS active_connections,
    xact_commit AS transactions_committed,
    xact_rollback AS transactions_rolled_back,
    blks_read AS blocks_read,
    blks_hit AS blocks_hit,
    tup_returned AS tuples_returned,
    tup_fetched AS tuples_fetched,
    tup_inserted AS tuples_inserted,
    tup_updated AS tuples_updated,
    tup_deleted AS tuples_deleted
FROM pg_database
LEFT JOIN pg_stat_database ON pg_database.datname = pg_stat_database.datname
WHERE pg_database.datistemplate = false
ORDER BY pg_database_size(pg_database.datname) DESC;

-- Check WAL generation rate
SELECT 
    NOW() - pg_postmaster_start_time() AS uptime,
    pg_current_wal_lsn() AS current_wal_lsn,
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 AS wal_mb_generated,
    ROUND(
        pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 / 
        EXTRACT(EPOCH FROM (NOW() - pg_postmaster_start_time())) * 3600
    , 2) AS wal_mb_per_hour;
```

#### Backup Strategy Decision Matrix
```sql
-- Function to recommend backup strategy based on database characteristics
CREATE OR REPLACE FUNCTION recommend_backup_strategy()
RETURNS TABLE(
    database_name TEXT,
    size_category TEXT,
    activity_level TEXT,
    recommended_strategy TEXT,
    backup_frequency TEXT,
    retention_period TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        pd.datname::TEXT,
        CASE 
            WHEN pg_database_size(pd.datname) < 1024*1024*1024 THEN 'Small (<1GB)'
            WHEN pg_database_size(pd.datname) < 10*1024*1024*1024 THEN 'Medium (1-10GB)'
            WHEN pg_database_size(pd.datname) < 100*1024*1024*1024 THEN 'Large (10-100GB)'
            ELSE 'Very Large (>100GB)'
        END::TEXT,
        CASE 
            WHEN COALESCE(psd.tup_inserted + psd.tup_updated + psd.tup_deleted, 0) < 1000 THEN 'Low'
            WHEN COALESCE(psd.tup_inserted + psd.tup_updated + psd.tup_deleted, 0) < 100000 THEN 'Medium'
            ELSE 'High'
        END::TEXT,
        CASE 
            WHEN pg_database_size(pd.datname) < 1024*1024*1024 AND 
                 COALESCE(psd.tup_inserted + psd.tup_updated + psd.tup_deleted, 0) < 1000 
            THEN 'pg_dump + WAL archiving'
            WHEN pg_database_size(pd.datname) < 10*1024*1024*1024 
            THEN 'pg_basebackup + WAL archiving'
            ELSE 'Streaming replication + pg_basebackup'
        END::TEXT,
        CASE 
            WHEN COALESCE(psd.tup_inserted + psd.tup_updated + psd.tup_deleted, 0) < 1000 THEN 'Daily'
            WHEN COALESCE(psd.tup_inserted + psd.tup_updated + psd.tup_deleted, 0) < 100000 THEN 'Every 6 hours'
            ELSE 'Every 2 hours'
        END::TEXT,
        CASE 
            WHEN pg_database_size(pd.datname) < 1024*1024*1024 THEN '30 days'
            WHEN pg_database_size(pd.datname) < 10*1024*1024*1024 THEN '14 days'
            ELSE '7 days'
        END::TEXT
    FROM pg_database pd
    LEFT JOIN pg_stat_database psd ON pd.datname = psd.datname
    WHERE pd.datistemplate = false;
END;
$$ LANGUAGE plpgsql;

-- Get backup strategy recommendations
SELECT * FROM recommend_backup_strategy();
```

---

## Physical Backups

### pg_basebackup

#### Basic pg_basebackup Usage
```bash
# Basic backup command
pg_basebackup -D /backup/base_backup_$(date +%Y%m%d_%H%M%S) \
    -Ft -z -P -v \
    -h localhost -p 5432 -U postgres

# Compressed tar format backup
pg_basebackup -D /backup/compressed_backup_$(date +%Y%m%d_%H%M%S) \
    -Ft -z -Z 9 -P -v \
    -h localhost -p 5432 -U postgres

# Backup with WAL files included
pg_basebackup -D /backup/backup_with_wal_$(date +%Y%m%d_%H%M%S) \
    -Ft -z -X stream -P -v \
    -h localhost -p 5432 -U postgres
```

#### Advanced pg_basebackup Script
```bash
#!/bin/bash
# advanced_backup.sh - Comprehensive PostgreSQL backup script

# Configuration
BACKUP_BASE_DIR="/backup/postgresql"
POSTGRES_HOST="localhost"
POSTGRES_PORT="5432"
POSTGRES_USER="postgres"
RETENTION_DAYS=7
COMPRESSION_LEVEL=6
NOTIFICATION_EMAIL="admin@example.com"

# Create timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_BASE_DIR}/base_backup_${TIMESTAMP}"
LOG_FILE="${BACKUP_BASE_DIR}/backup_${TIMESTAMP}.log"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Function to send notification
send_notification() {
    local subject="$1"
    local message="$2"
    echo "$message" | mail -s "$subject" "$NOTIFICATION_EMAIL"
}

# Start backup
log_message "Starting PostgreSQL backup to $BACKUP_DIR"

# Create backup directory
mkdir -p "$BACKUP_DIR"
mkdir -p "$(dirname "$LOG_FILE")"

# Perform backup
log_message "Running pg_basebackup..."
if pg_basebackup \
    -D "$BACKUP_DIR" \
    -Ft -z -Z "$COMPRESSION_LEVEL" \
    -X stream -P -v \
    -h "$POSTGRES_HOST" -p "$POSTGRES_PORT" -U "$POSTGRES_USER" \
    >> "$LOG_FILE" 2>&1; then
    
    log_message "Backup completed successfully"
    
    # Get backup size
    BACKUP_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
    log_message "Backup size: $BACKUP_SIZE"
    
    # Verify backup integrity
    log_message "Verifying backup integrity..."
    if tar -tzf "$BACKUP_DIR/base.tar.gz" > /dev/null 2>&1; then
        log_message "Backup integrity check passed"
    else
        log_message "ERROR: Backup integrity check failed"
        send_notification "PostgreSQL Backup Failed" "Backup integrity check failed for $BACKUP_DIR"
        exit 1
    fi
    
    # Clean old backups
    log_message "Cleaning old backups (older than $RETENTION_DAYS days)..."
    find "$BACKUP_BASE_DIR" -type d -name "base_backup_*" -mtime +$RETENTION_DAYS -exec rm -rf {} \;
    find "$BACKUP_BASE_DIR" -type f -name "backup_*.log" -mtime +$RETENTION_DAYS -delete
    
    log_message "Backup process completed successfully"
    send_notification "PostgreSQL Backup Successful" "Backup completed: $BACKUP_DIR (Size: $BACKUP_SIZE)"
    
else
    log_message "ERROR: Backup failed"
    send_notification "PostgreSQL Backup Failed" "Backup failed. Check log: $LOG_FILE"
    exit 1
fi
```

### File System Level Backups

#### Using rsync for Hot Backups
```bash
#!/bin/bash
# filesystem_backup.sh - File system level backup with proper PostgreSQL handling

PGDATA="/var/lib/postgresql/15/main"
BACKUP_DIR="/backup/filesystem/$(date +%Y%m%d_%H%M%S)"
POSTGRES_USER="postgres"

# Function to perform backup label operations
create_backup_label() {
    sudo -u "$POSTGRES_USER" psql -c "SELECT pg_start_backup('filesystem_backup', false, false);"
}

stop_backup_label() {
    sudo -u "$POSTGRES_USER" psql -c "SELECT pg_stop_backup(false, true);"
}

# Create backup directory
mkdir -p "$BACKUP_DIR"

echo "Starting backup label..."
BACKUP_LABEL=$(create_backup_label)

echo "Performing file system backup..."
# Exclude temporary files and logs that shouldn't be backed up
rsync -av --exclude 'postmaster.pid' \
          --exclude 'pg_wal/archive_status/*' \
          --exclude 'pg_replslot/*' \
          --exclude 'pg_stat_tmp/*' \
          --exclude 'pg_subtrans/*' \
          "$PGDATA/" "$BACKUP_DIR/"

echo "Stopping backup label..."
BACKUP_INFO=$(stop_backup_label)

echo "Backup completed: $BACKUP_DIR"
echo "Backup info: $BACKUP_INFO"
```

---

## Logical Backups

### pg_dump and pg_dumpall

#### Comprehensive pg_dump Usage
```bash
# Full database dump with all objects
pg_dump -h localhost -p 5432 -U postgres \
    --verbose --format=custom --compress=6 \
    --file=/backup/logical/$(date +%Y%m%d_%H%M%S)_full_backup.dump \
    mydb

# Schema-only dump
pg_dump -h localhost -p 5432 -U postgres \
    --schema-only --verbose --format=custom \
    --file=/backup/logical/$(date +%Y%m%d_%H%M%S)_schema_only.dump \
    mydb

# Data-only dump
pg_dump -h localhost -p 5432 -U postgres \
    --data-only --verbose --format=custom \
    --file=/backup/logical/$(date +%Y%m%d_%H%M%S)_data_only.dump \
    mydb

# Specific tables dump
pg_dump -h localhost -p 5432 -U postgres \
    --table=sales.customers --table=sales.orders \
    --verbose --format=custom \
    --file=/backup/logical/$(date +%Y%m%d_%H%M%S)_tables_backup.dump \
    mydb

# Parallel dump for large databases
pg_dump -h localhost -p 5432 -U postgres \
    --format=directory --jobs=4 --compress=6 \
    --file=/backup/logical/$(date +%Y%m%d_%H%M%S)_parallel_backup \
    mydb
```

#### Advanced Logical Backup Script
```bash
#!/bin/bash
# logical_backup.sh - Advanced logical backup with error handling

# Configuration
CONFIG_FILE="/etc/postgresql/backup.conf"
DEFAULT_BACKUP_DIR="/backup/logical"
DEFAULT_RETENTION_DAYS=30
DEFAULT_COMPRESSION=6
DEFAULT_PARALLEL_JOBS=2

# Load configuration if exists
if [ -f "$CONFIG_FILE" ]; then
    source "$CONFIG_FILE"
fi

# Set defaults
BACKUP_DIR="${BACKUP_DIR:-$DEFAULT_BACKUP_DIR}"
RETENTION_DAYS="${RETENTION_DAYS:-$DEFAULT_RETENTION_DAYS}"
COMPRESSION="${COMPRESSION:-$DEFAULT_COMPRESSION}"
PARALLEL_JOBS="${PARALLEL_JOBS:-$DEFAULT_PARALLEL_JOBS}"

# Database configuration
DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
DB_USER="${DB_USER:-postgres}"

# Function to backup a single database
backup_database() {
    local db_name="$1"
    local timestamp="$2"
    local backup_file="${BACKUP_DIR}/${timestamp}_${db_name}.dump"
    local log_file="${BACKUP_DIR}/${timestamp}_${db_name}.log"
    
    echo "Backing up database: $db_name"
    
    # Get database size before backup
    local db_size=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$db_name" -t -c "SELECT pg_size_pretty(pg_database_size('$db_name'));" | xargs)
    
    # Perform backup
    if pg_dump -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" \
        --verbose --format=custom --compress="$COMPRESSION" \
        --file="$backup_file" \
        "$db_name" > "$log_file" 2>&1; then
        
        # Get backup file size
        local backup_size=$(du -sh "$backup_file" | cut -f1)
        
        # Verify backup
        if pg_restore --list "$backup_file" > /dev/null 2>&1; then
            echo "✓ $db_name backed up successfully (DB: $db_size, Backup: $backup_size)"
            
            # Create metadata file
            cat > "${backup_file}.meta" << EOF
database_name=$db_name
backup_timestamp=$timestamp
database_size=$db_size
backup_size=$backup_size
backup_format=custom
compression_level=$COMPRESSION
backup_type=full
status=success
EOF
        else
            echo "✗ $db_name backup verification failed"
            rm -f "$backup_file"
            return 1
        fi
    else
        echo "✗ $db_name backup failed"
        return 1
    fi
}

# Function to backup all databases
backup_all_databases() {
    local timestamp="$1"
    local success_count=0
    local total_count=0
    
    # Get list of databases to backup (exclude templates and postgres)
    local databases=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -t -c "SELECT datname FROM pg_database WHERE datistemplate = false AND datname != 'postgres';" | xargs)
    
    for db in $databases; do
        ((total_count++))
        if backup_database "$db" "$timestamp"; then
            ((success_count++))
        fi
    done
    
    echo "Backup summary: $success_count/$total_count databases backed up successfully"
    
    # Backup global objects (roles, tablespaces, etc.)
    echo "Backing up global objects..."
    local globals_file="${BACKUP_DIR}/${timestamp}_globals.sql"
    if pg_dumpall -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" \
        --globals-only --verbose \
        --file="$globals_file"; then
        echo "✓ Global objects backed up successfully"
    else
        echo "✗ Global objects backup failed"
    fi
}

# Function to cleanup old backups
cleanup_old_backups() {
    echo "Cleaning up backups older than $RETENTION_DAYS days..."
    find "$BACKUP_DIR" -name "*.dump" -mtime +$RETENTION_DAYS -delete
    find "$BACKUP_DIR" -name "*.sql" -mtime +$RETENTION_DAYS -delete
    find "$BACKUP_DIR" -name "*.log" -mtime +$RETENTION_DAYS -delete
    find "$BACKUP_DIR" -name "*.meta" -mtime +$RETENTION_DAYS -delete
}

# Main execution
main() {
    local timestamp=$(date +%Y%m%d_%H%M%S)
    
    echo "Starting PostgreSQL logical backup at $(date)"
    echo "Backup directory: $BACKUP_DIR"
    echo "Retention period: $RETENTION_DAYS days"
    echo "Compression level: $COMPRESSION"
    
    # Create backup directory
    mkdir -p "$BACKUP_DIR"
    
    # Perform backups
    backup_all_databases "$timestamp"
    
    # Cleanup old backups
    cleanup_old_backups
    
    echo "Backup process completed at $(date)"
}

# Run main function
main "$@"
```

---

## Point-in-Time Recovery

### WAL Configuration

#### Setting up WAL Archiving
```sql
-- Check current WAL settings
SELECT name, setting, unit, context, pending_restart
FROM pg_settings 
WHERE name IN (
    'wal_level', 'archive_mode', 'archive_command', 
    'max_wal_senders', 'wal_keep_size', 'checkpoint_timeout'
)
ORDER BY name;

-- Enable WAL archiving (requires restart)
-- In postgresql.conf:
-- wal_level = replica
-- archive_mode = on
-- archive_command = 'cp %p /archive/wal/%f'
-- max_wal_senders = 3
-- wal_keep_size = 2GB

-- Check WAL generation and archiving status
SELECT 
    name,
    setting,
    CASE name 
        WHEN 'archive_mode' THEN 
            CASE WHEN setting = 'on' THEN '✓ Enabled' ELSE '✗ Disabled' END
        WHEN 'wal_level' THEN 
            CASE WHEN setting IN ('replica', 'logical') THEN '✓ ' || setting ELSE '⚠ ' || setting END
        ELSE setting
    END AS status
FROM pg_settings 
WHERE name IN ('wal_level', 'archive_mode', 'archive_command');

-- Monitor WAL archiving
SELECT 
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time,
    stats_reset
FROM pg_stat_archiver;
```

#### WAL Archive Script
```bash
#!/bin/bash
# wal_archive.sh - WAL archiving script with error handling and compression

WAL_FILE="$1"
WAL_PATH="$2"
ARCHIVE_DIR="/archive/wal"
COMPRESSION="gzip"
LOG_FILE="/var/log/postgresql/wal_archive.log"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Validate inputs
if [ -z "$WAL_FILE" ] || [ -z "$WAL_PATH" ]; then
    log_message "ERROR: WAL file or path not provided"
    exit 1
fi

# Check if source file exists
if [ ! -f "$WAL_PATH" ]; then
    log_message "ERROR: Source WAL file does not exist: $WAL_PATH"
    exit 1
fi

# Create archive directory if it doesn't exist
mkdir -p "$ARCHIVE_DIR"

# Determine destination path
DEST_PATH="$ARCHIVE_DIR/$WAL_FILE"

# Compress and copy the WAL file
if [ "$COMPRESSION" = "gzip" ]; then
    DEST_PATH="$DEST_PATH.gz"
    if gzip -c "$WAL_PATH" > "$DEST_PATH.tmp" && mv "$DEST_PATH.tmp" "$DEST_PATH"; then
        log_message "SUCCESS: Archived $WAL_FILE (compressed)"
        exit 0
    else
        log_message "ERROR: Failed to archive $WAL_FILE (compression failed)"
        rm -f "$DEST_PATH.tmp"
        exit 1
    fi
else
    # Simple copy
    if cp "$WAL_PATH" "$DEST_PATH.tmp" && mv "$DEST_PATH.tmp" "$DEST_PATH"; then
        log_message "SUCCESS: Archived $WAL_FILE"
        exit 0
    else
        log_message "ERROR: Failed to archive $WAL_FILE (copy failed)"
        rm -f "$DEST_PATH.tmp"
        exit 1
    fi
fi
```

### PITR Recovery Process

#### Recovery Configuration
```sql
-- Create recovery script
-- This would be used during recovery process
```

```bash
#!/bin/bash
# pitr_recovery.sh - Point-in-Time Recovery script

# Configuration
BACKUP_DIR="/backup/base_backup_20240109_140000"
ARCHIVE_DIR="/archive/wal"
RECOVERY_TARGET_TIME="2024-01-09 14:30:00"
POSTGRES_DATA_DIR="/var/lib/postgresql/15/main"
POSTGRES_USER="postgres"

# Function to setup recovery
setup_recovery() {
    echo "Setting up Point-in-Time Recovery..."
    
    # Stop PostgreSQL if running
    systemctl stop postgresql
    
    # Backup current data directory
    if [ -d "$POSTGRES_DATA_DIR" ]; then
        mv "$POSTGRES_DATA_DIR" "${POSTGRES_DATA_DIR}.backup.$(date +%Y%m%d_%H%M%S)"
    fi
    
    # Restore base backup
    echo "Restoring base backup from $BACKUP_DIR..."
    mkdir -p "$POSTGRES_DATA_DIR"
    
    # Extract base backup
    if [ -f "$BACKUP_DIR/base.tar.gz" ]; then
        tar -xzf "$BACKUP_DIR/base.tar.gz" -C "$POSTGRES_DATA_DIR"
    else
        echo "ERROR: Base backup not found at $BACKUP_DIR/base.tar.gz"
        exit 1
    fi
    
    # Set ownership
    chown -R "$POSTGRES_USER:$POSTGRES_USER" "$POSTGRES_DATA_DIR"
    
    # Create recovery signal file
    touch "$POSTGRES_DATA_DIR/recovery.signal"
    
    # Create recovery configuration
    cat > "$POSTGRES_DATA_DIR/postgresql.auto.conf" << EOF
# Recovery configuration
restore_command = 'gunzip -c $ARCHIVE_DIR/%f.gz > %p 2>/dev/null || cp $ARCHIVE_DIR/%f %p'
recovery_target_time = '$RECOVERY_TARGET_TIME'
recovery_target_action = 'promote'
EOF

    echo "Recovery setup complete. Starting PostgreSQL..."
    
    # Start PostgreSQL
    systemctl start postgresql
    
    # Monitor recovery
    echo "Monitoring recovery progress..."
    while [ -f "$POSTGRES_DATA_DIR/recovery.signal" ]; do
        echo "Recovery in progress..."
        sleep 5
    done
    
    echo "Recovery completed!"
    
    # Verify recovery
    sudo -u "$POSTGRES_USER" psql -c "SELECT now(), pg_is_in_recovery();"
}

# Execute recovery
setup_recovery
```

---

## Continuous Archiving

### Automated WAL Management

#### WAL Archive Management Script
```bash
#!/bin/bash
# wal_manager.sh - Comprehensive WAL archive management

ARCHIVE_DIR="/archive/wal"
BACKUP_DIR="/backup"
RETENTION_DAYS=30
COMPRESSION_AGE_HOURS=24
LOG_FILE="/var/log/postgresql/wal_manager.log"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Function to compress old WAL files
compress_old_wal() {
    log_message "Compressing WAL files older than $COMPRESSION_AGE_HOURS hours..."
    
    find "$ARCHIVE_DIR" -name "*.wal" -type f -mtime +0 -mmin +$((COMPRESSION_AGE_HOURS * 60)) | while read wal_file; do
        if gzip "$wal_file"; then
            log_message "Compressed: $(basename "$wal_file")"
        else
            log_message "ERROR: Failed to compress $(basename "$wal_file")"
        fi
    done
}

# Function to clean old WAL files
cleanup_old_wal() {
    log_message "Cleaning WAL files older than $RETENTION_DAYS days..."
    
    # Get the oldest base backup date to ensure we don't delete needed WAL files
    local oldest_backup_date=$(find "$BACKUP_DIR" -name "base_backup_*" -type d | sort | head -1 | sed 's/.*base_backup_\([0-9]\{8\}\).*/\1/')
    
    if [ -n "$oldest_backup_date" ]; then
        log_message "Oldest backup: $oldest_backup_date"
        
        # Convert to days ago
        local oldest_backup_days=$(( ($(date +%s) - $(date -d "$oldest_backup_date" +%s)) / 86400 ))
        local safe_retention_days=$((oldest_backup_days + 7)) # Add 7 days buffer
        
        if [ $safe_retention_days -lt $RETENTION_DAYS ]; then
            safe_retention_days=$RETENTION_DAYS
        fi
        
        log_message "Using safe retention period: $safe_retention_days days"
        
        # Delete old WAL files
        find "$ARCHIVE_DIR" -name "*.gz" -type f -mtime +$safe_retention_days -delete
        find "$ARCHIVE_DIR" -name "*.wal" -type f -mtime +$safe_retention_days -delete
        
        local deleted_count=$(find "$ARCHIVE_DIR" -name "*.gz" -o -name "*.wal" | wc -l)
        log_message "Cleanup completed. Remaining WAL files: $deleted_count"
    else
        log_message "WARNING: No base backups found, skipping WAL cleanup"
    fi
}

# Function to verify WAL archive integrity
verify_wal_integrity() {
    log_message "Verifying WAL archive integrity..."
    
    local corrupt_files=0
    find "$ARCHIVE_DIR" -name "*.gz" -type f | while read wal_file; do
        if ! gzip -t "$wal_file" 2>/dev/null; then
            log_message "ERROR: Corrupt WAL file: $(basename "$wal_file")"
            ((corrupt_files++))
        fi
    done
    
    if [ $corrupt_files -eq 0 ]; then
        log_message "WAL integrity check passed"
    else
        log_message "WARNING: Found $corrupt_files corrupt WAL files"
    fi
}

# Function to generate WAL archive report
generate_report() {
    log_message "Generating WAL archive report..."
    
    local total_files=$(find "$ARCHIVE_DIR" -name "*.gz" -o -name "*.wal" | wc -l)
    local total_size=$(du -sh "$ARCHIVE_DIR" | cut -f1)
    local compressed_files=$(find "$ARCHIVE_DIR" -name "*.gz" | wc -l)
    local uncompressed_files=$(find "$ARCHIVE_DIR" -name "*.wal" | wc -l)
    
    cat << EOF | tee -a "$LOG_FILE"

WAL Archive Report ($(date)):
================================
Total WAL files: $total_files
Total archive size: $total_size
Compressed files: $compressed_files
Uncompressed files: $uncompressed_files
Archive directory: $ARCHIVE_DIR

EOF
}

# Main execution
main() {
    log_message "Starting WAL archive management"
    
    # Create directories if they don't exist
    mkdir -p "$ARCHIVE_DIR"
    mkdir -p "$(dirname "$LOG_FILE")"
    
    # Perform operations
    verify_wal_integrity
    compress_old_wal
    cleanup_old_wal
    generate_report
    
    log_message "WAL archive management completed"
}

# Run with different operations based on argument
case "${1:-all}" in
    compress)
        compress_old_wal
        ;;
    cleanup)
        cleanup_old_wal
        ;;
    verify)
        verify_wal_integrity
        ;;
    report)
        generate_report
        ;;
    all)
        main
        ;;
    *)
        echo "Usage: $0 {compress|cleanup|verify|report|all}"
        exit 1
        ;;
esac
```

---

## Streaming Replication

### Primary Server Setup

#### Configuring the Primary Server
```sql
-- Check replication settings
SELECT name, setting, unit, context
FROM pg_settings 
WHERE name LIKE '%replication%' OR name LIKE '%wal%sender%'
ORDER BY name;

-- Create replication user
CREATE USER replicator WITH REPLICATION PASSWORD 'replication_password_2024!';

-- Grant necessary permissions
GRANT CONNECT ON DATABASE postgres TO replicator;

-- Check replication slots
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

-- Create replication slot for standby
SELECT pg_create_physical_replication_slot('standby_slot');

-- Monitor replication status
SELECT 
    client_addr,
    client_hostname,
    client_port,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    sync_priority,
    sync_state
FROM pg_stat_replication;
```

### Standby Server Setup

#### Setting up Streaming Replication
```bash
#!/bin/bash
# setup_standby.sh - Setup streaming replication standby

# Configuration
PRIMARY_HOST="192.168.1.100"
PRIMARY_PORT="5432"
REPLICATION_USER="replicator"
REPLICATION_PASSWORD="replication_password_2024!"
STANDBY_DATA_DIR="/var/lib/postgresql/15/standby"
REPLICATION_SLOT="standby_slot"

# Stop PostgreSQL if running
systemctl stop postgresql

# Backup existing data directory
if [ -d "$STANDBY_DATA_DIR" ]; then
    mv "$STANDBY_DATA_DIR" "${STANDBY_DATA_DIR}.backup.$(date +%Y%m%d_%H%M%S)"
fi

# Create base backup from primary
echo "Creating base backup from primary server..."
PGPASSWORD="$REPLICATION_PASSWORD" pg_basebackup \
    -h "$PRIMARY_HOST" -p "$PRIMARY_PORT" -U "$REPLICATION_USER" \
    -D "$STANDBY_DATA_DIR" \
    -Ft -z -P -v -R \
    -S "$REPLICATION_SLOT"

# Set ownership
chown -R postgres:postgres "$STANDBY_DATA_DIR"

# Create standby signal
touch "$STANDBY_DATA_DIR/standby.signal"

# Configure standby settings
cat >> "$STANDBY_DATA_DIR/postgresql.auto.conf" << EOF
# Standby configuration
primary_conninfo = 'host=$PRIMARY_HOST port=$PRIMARY_PORT user=$REPLICATION_USER password=$REPLICATION_PASSWORD application_name=standby1'
primary_slot_name = '$REPLICATION_SLOT'
hot_standby = on
hot_standby_feedback = on
max_standby_streaming_delay = 30s
wal_receiver_timeout = 60s
EOF

echo "Standby setup complete. Starting PostgreSQL..."
systemctl start postgresql

# Verify replication
echo "Verifying replication status..."
sudo -u postgres psql -c "SELECT pg_is_in_recovery(), pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();"
```

#### Standby Monitoring Script
```bash
#!/bin/bash
# monitor_standby.sh - Monitor standby server health

LOG_FILE="/var/log/postgresql/standby_monitor.log"
ALERT_THRESHOLD_SECONDS=300  # Alert if lag > 5 minutes
NOTIFICATION_EMAIL="admin@example.com"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Function to send alert
send_alert() {
    local subject="$1"
    local message="$2"
    echo "$message" | mail -s "$subject" "$NOTIFICATION_EMAIL"
    log_message "ALERT: $subject"
}

# Function to check replication status
check_replication() {
    local status=$(sudo -u postgres psql -t -c "SELECT pg_is_in_recovery();" | xargs)
    
    if [ "$status" != "t" ]; then
        send_alert "Standby Server Alert" "Server is not in recovery mode - replication may be broken"
        return 1
    fi
    
    # Get replication lag information
    local lag_info=$(sudo -u postgres psql -t -c "
        SELECT 
            EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::int as lag_seconds,
            pg_last_wal_receive_lsn(),
            pg_last_wal_replay_lsn(),
            pg_last_xact_replay_timestamp()
    " | xargs)
    
    if [ -n "$lag_info" ]; then
        local lag_seconds=$(echo "$lag_info" | cut -d' ' -f1)
        local receive_lsn=$(echo "$lag_info" | cut -d' ' -f2)
        local replay_lsn=$(echo "$lag_info" | cut -d' ' -f3)
        local last_replay=$(echo "$lag_info" | cut -d' ' -f4-)
        
        log_message "Replication lag: ${lag_seconds}s, Receive LSN: $receive_lsn, Replay LSN: $replay_lsn"
        
        if [ "$lag_seconds" -gt "$ALERT_THRESHOLD_SECONDS" ]; then
            send_alert "High Replication Lag" "Replication lag is ${lag_seconds} seconds (threshold: ${ALERT_THRESHOLD_SECONDS}s)"
        fi
    else
        send_alert "Standby Monitoring Error" "Unable to retrieve replication status"
    fi
}

# Function to check connection to primary
check_primary_connection() {
    local primary_info=$(sudo -u postgres psql -t -c "
        SELECT 
            conninfo,
            status,
            receive_start_lsn,
            receive_start_tli
        FROM pg_stat_wal_receiver;
    ")
    
    if [ -z "$primary_info" ]; then
        send_alert "Primary Connection Lost" "WAL receiver is not running - connection to primary may be lost"
        return 1
    else
        log_message "Primary connection: OK"
        return 0
    fi
}

# Main monitoring function
main() {
    log_message "Starting standby monitoring check"
    
    local errors=0
    
    # Check if server is in recovery mode
    if ! check_replication; then
        ((errors++))
    fi
    
    # Check connection to primary
    if ! check_primary_connection; then
        ((errors++))
    fi
    
    if [ $errors -eq 0 ]; then
        log_message "All checks passed"
    else
        log_message "Monitoring completed with $errors errors"
    fi
}

# Run monitoring
main "$@"
```

---

## Backup Monitoring

### Backup Validation

#### Backup Testing Script
```bash
#!/bin/bash
# test_backup.sh - Automated backup testing

BACKUP_DIR="/backup"
TEST_DIR="/tmp/postgres_backup_test"
TEST_DB_NAME="backup_test_$(date +%Y%m%d_%H%M%S)"
LOG_FILE="/var/log/postgresql/backup_test.log"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Function to test logical backup
test_logical_backup() {
    local backup_file="$1"
    
    log_message "Testing logical backup: $(basename "$backup_file")"
    
    # Create test database
    createdb "$TEST_DB_NAME"
    
    # Restore backup
    if pg_restore -d "$TEST_DB_NAME" "$backup_file" > /dev/null 2>&1; then
        log_message "✓ Logical backup restore successful"
        
        # Basic integrity check
        local table_count=$(psql -d "$TEST_DB_NAME" -t -c "SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';" | xargs)
        log_message "Restored tables: $table_count"
        
        # Cleanup
        dropdb "$TEST_DB_NAME"
        return 0
    else
        log_message "✗ Logical backup restore failed"
        dropdb "$TEST_DB_NAME" 2>/dev/null
        return 1
    fi
}

# Function to test physical backup
test_physical_backup() {
    local backup_dir="$1"
    
    log_message "Testing physical backup: $(basename "$backup_dir")"
    
    # Create test directory
    mkdir -p "$TEST_DIR"
    
    # Extract backup
    if [ -f "$backup_dir/base.tar.gz" ]; then
        tar -xzf "$backup_dir/base.tar.gz" -C "$TEST_DIR"
    else
        log_message "✗ Base backup file not found"
        return 1
    fi
    
    # Basic file structure check
    if [ -f "$TEST_DIR/PG_VERSION" ] && [ -f "$TEST_DIR/postgresql.conf" ]; then
        log_message "✓ Physical backup structure valid"
        
        # Check PG_VERSION
        local pg_version=$(cat "$TEST_DIR/PG_VERSION")
        log_message "PostgreSQL version in backup: $pg_version"
        
        # Cleanup
        rm -rf "$TEST_DIR"
        return 0
    else
        log_message "✗ Physical backup structure invalid"
        rm -rf "$TEST_DIR"
        return 1
    fi
}

# Function to test all recent backups
test_recent_backups() {
    local test_count=0
    local success_count=0
    
    log_message "Starting backup testing..."
    
    # Test logical backups (last 3 days)
    find "$BACKUP_DIR/logical" -name "*.dump" -mtime -3 | head -5 | while read backup_file; do
        ((test_count++))
        if test_logical_backup "$backup_file"; then
            ((success_count++))
        fi
    done
    
    # Test physical backups (last 3 days)
    find "$BACKUP_DIR" -name "base_backup_*" -type d -mtime -3 | head -3 | while read backup_dir; do
        ((test_count++))
        if test_physical_backup "$backup_dir"; then
            ((success_count++))
        fi
    done
    
    log_message "Backup testing completed: $success_count/$test_count backups passed"
}

# Function to generate backup report
generate_backup_report() {
    log_message "Generating backup report..."
    
    local logical_backups=$(find "$BACKUP_DIR/logical" -name "*.dump" -mtime -7 | wc -l)
    local physical_backups=$(find "$BACKUP_DIR" -name "base_backup_*" -type d -mtime -7 | wc -l)
    local total_backup_size=$(du -sh "$BACKUP_DIR" | cut -f1)
    
    cat << EOF | tee -a "$LOG_FILE"

Backup Report ($(date)):
========================
Logical backups (last 7 days): $logical_backups
Physical backups (last 7 days): $physical_backups
Total backup storage used: $total_backup_size

Latest backups:
$(find "$BACKUP_DIR" -name "*.dump" -o -name "base_backup_*" | sort -r | head -10)

EOF
}

# Main execution
case "${1:-test}" in
    test)
        test_recent_backups
        ;;
    report)
        generate_backup_report
        ;;
    all)
        test_recent_backups
        generate_backup_report
        ;;
    *)
        echo "Usage: $0 {test|report|all}"
        exit 1
        ;;
esac
```

---

## Disaster Recovery

### DR Planning and Procedures

#### Disaster Recovery Runbook
```bash
#!/bin/bash
# disaster_recovery.sh - Comprehensive disaster recovery script

# Configuration
DR_CONFIG_FILE="/etc/postgresql/dr_config.conf"
PRIMARY_BACKUP_LOCATION="/backup"
REMOTE_BACKUP_LOCATION="s3://my-postgres-backups"
RECOVERY_TARGET_DIR="/var/lib/postgresql/15/main"
LOG_FILE="/var/log/postgresql/disaster_recovery.log"

# Load DR configuration
if [ -f "$DR_CONFIG_FILE" ]; then
    source "$DR_CONFIG_FILE"
fi

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Function to show DR status
show_dr_status() {
    log_message "Disaster Recovery Status Check"
    echo "=================================="
    
    # Check PostgreSQL status
    if systemctl is-active --quiet postgresql; then
        echo "✓ PostgreSQL is running"
        
        # Check if it's primary or standby
        local recovery_status=$(sudo -u postgres psql -t -c "SELECT pg_is_in_recovery();" 2>/dev/null | xargs)
        if [ "$recovery_status" = "f" ]; then
            echo "✓ Running as PRIMARY server"
        elif [ "$recovery_status" = "t" ]; then
            echo "✓ Running as STANDBY server"
        else
            echo "⚠ Unable to determine server role"
        fi
    else
        echo "✗ PostgreSQL is not running"
    fi
    
    # Check backup availability
    local latest_backup=$(find "$PRIMARY_BACKUP_LOCATION" -name "base_backup_*" -type d | sort -r | head -1)
    if [ -n "$latest_backup" ]; then
        echo "✓ Latest backup available: $(basename "$latest_backup")"
    else
        echo "✗ No local backups found"
    fi
    
    # Check WAL archive
    local wal_count=$(find /archive/wal -name "*.gz" -o -name "*.wal" 2>/dev/null | wc -l)
    echo "✓ WAL archive files: $wal_count"
    
    # Check disk space
    local disk_usage=$(df -h "$RECOVERY_TARGET_DIR" 2>/dev/null | tail -1 | awk '{print $5}' | sed 's/%//')
    if [ "$disk_usage" -lt 80 ]; then
        echo "✓ Disk space: ${disk_usage}% used"
    else
        echo "⚠ Disk space: ${disk_usage}% used (HIGH)"
    fi
}

# Function to perform emergency backup
emergency_backup() {
    log_message "Performing emergency backup..."
    
    local emergency_backup_dir="/backup/emergency_$(date +%Y%m%d_%H%M%S)"
    
    if pg_basebackup -D "$emergency_backup_dir" -Ft -z -X stream -P -v; then
        log_message "✓ Emergency backup completed: $emergency_backup_dir"
        return 0
    else
        log_message "✗ Emergency backup failed"
        return 1
    fi
}

# Function to initiate failover
initiate_failover() {
    log_message "Initiating failover procedure..."
    
    # Check if this is a standby server
    local recovery_status=$(sudo -u postgres psql -t -c "SELECT pg_is_in_recovery();" 2>/dev/null | xargs)
    
    if [ "$recovery_status" = "t" ]; then
        log_message "Promoting standby server to primary..."
        
        # Promote standby to primary
        if sudo -u postgres pg_ctl promote -D "$RECOVERY_TARGET_DIR"; then
            log_message "✓ Standby promoted to primary successfully"
            
            # Remove standby signal file
            rm -f "$RECOVERY_TARGET_DIR/standby.signal"
            
            # Verify promotion
            sleep 5
            local new_status=$(sudo -u postgres psql -t -c "SELECT pg_is_in_recovery();" | xargs)
            if [ "$new_status" = "f" ]; then
                log_message "✓ Failover completed - server is now primary"
                return 0
            else
                log_message "✗ Failover verification failed"
                return 1
            fi
        else
            log_message "✗ Failover failed during promotion"
            return 1
        fi
    else
        log_message "✗ This server is not a standby - failover not applicable"
        return 1
    fi
}

# Function to restore from backup
restore_from_backup() {
    local backup_location="$1"
    local target_time="$2"
    
    log_message "Initiating restore from backup: $backup_location"
    
    # Stop PostgreSQL
    systemctl stop postgresql
    
    # Backup current data directory
    if [ -d "$RECOVERY_TARGET_DIR" ]; then
        mv "$RECOVERY_TARGET_DIR" "${RECOVERY_TARGET_DIR}.disaster_backup.$(date +%Y%m%d_%H%M%S)"
    fi
    
    # Restore from backup
    mkdir -p "$RECOVERY_TARGET_DIR"
    
    if [ -f "$backup_location/base.tar.gz" ]; then
        tar -xzf "$backup_location/base.tar.gz" -C "$RECOVERY_TARGET_DIR"
    else
        log_message "✗ Backup file not found: $backup_location/base.tar.gz"
        return 1
    fi
    
    # Set ownership
    chown -R postgres:postgres "$RECOVERY_TARGET_DIR"
    
    # Create recovery configuration
    touch "$RECOVERY_TARGET_DIR/recovery.signal"
    
    cat > "$RECOVERY_TARGET_DIR/postgresql.auto.conf" << EOF
# Disaster Recovery Configuration
restore_command = 'gunzip -c /archive/wal/%f.gz > %p 2>/dev/null || cp /archive/wal/%f %p'
recovery_target_action = 'promote'
EOF

    # Add recovery target time if specified
    if [ -n "$target_time" ]; then
        echo "recovery_target_time = '$target_time'" >> "$RECOVERY_TARGET_DIR/postgresql.auto.conf"
    fi
    
    # Start PostgreSQL
    systemctl start postgresql
    
    # Monitor recovery
    log_message "Monitoring recovery progress..."
    while [ -f "$RECOVERY_TARGET_DIR/recovery.signal" ]; do
        log_message "Recovery in progress..."
        sleep 10
    done
    
    log_message "✓ Disaster recovery completed"
}

# Function to sync from remote backup
sync_remote_backup() {
    if command -v aws >/dev/null 2>&1; then
        log_message "Syncing backups from remote location..."
        aws s3 sync "$REMOTE_BACKUP_LOCATION" "$PRIMARY_BACKUP_LOCATION/" --exclude "*" --include "base_backup_*"
    else
        log_message "⚠ AWS CLI not available for remote backup sync"
    fi
}

# Function to run disaster recovery drill
run_dr_drill() {
    log_message "Starting disaster recovery drill..."
    
    local drill_dir="/tmp/dr_drill_$(date +%Y%m%d_%H%M%S)"
    local latest_backup=$(find "$PRIMARY_BACKUP_LOCATION" -name "base_backup_*" -type d | sort -r | head -1)
    
    if [ -z "$latest_backup" ]; then
        log_message "✗ No backup available for DR drill"
        return 1
    fi
    
    # Create drill environment
    mkdir -p "$drill_dir"
    
    # Extract backup to drill directory
    if [ -f "$latest_backup/base.tar.gz" ]; then
        tar -xzf "$latest_backup/base.tar.gz" -C "$drill_dir"
    else
        log_message "✗ Backup file not found for drill"
        return 1
    fi
    
    # Test basic file integrity
    if [ -f "$drill_dir/PG_VERSION" ]; then
        local pg_version=$(cat "$drill_dir/PG_VERSION")
        log_message "✓ DR drill successful - PostgreSQL version $pg_version"
        
        # Cleanup
        rm -rf "$drill_dir"
        return 0
    else
        log_message "✗ DR drill failed - backup integrity issue"
        rm -rf "$drill_dir"
        return 1
    fi
}

# Main menu
show_menu() {
    echo "PostgreSQL Disaster Recovery Tool"
    echo "=================================="
    echo "1. Show DR Status"
    echo "2. Emergency Backup"
    echo "3. Initiate Failover"
    echo "4. Restore from Backup"
    echo "5. Sync Remote Backups"
    echo "6. Run DR Drill"
    echo "7. Exit"
    echo
}

# Interactive mode
interactive_mode() {
    while true; do
        show_menu
        read -p "Select an option (1-7): " choice
        
        case $choice in
            1)
                show_dr_status
                ;;
            2)
                emergency_backup
                ;;
            3)
                read -p "Are you sure you want to initiate failover? (yes/no): " confirm
                if [ "$confirm" = "yes" ]; then
                    initiate_failover
                fi
                ;;
            4)
                read -p "Enter backup location: " backup_loc
                read -p "Enter target time (optional, YYYY-MM-DD HH:MM:SS): " target_time
                restore_from_backup "$backup_loc" "$target_time"
                ;;
            5)
                sync_remote_backup
                ;;
            6)
                run_dr_drill
                ;;
            7)
                echo "Exiting..."
                exit 0
                ;;
            *)
                echo "Invalid option. Please try again."
                ;;
        esac
        
        echo
        read -p "Press Enter to continue..."
        echo
    done
}

# Command line execution
case "${1:-interactive}" in
    status)
        show_dr_status
        ;;
    backup)
        emergency_backup
        ;;
    failover)
        initiate_failover
        ;;
    restore)
        restore_from_backup "$2" "$3"
        ;;
    sync)
        sync_remote_backup
        ;;
    drill)
        run_dr_drill
        ;;
    interactive)
        interactive_mode
        ;;
    *)
        echo "Usage: $0 {status|backup|failover|restore|sync|drill|interactive}"
        echo
        echo "Commands:"
        echo "  status    - Show disaster recovery status"
        echo "  backup    - Perform emergency backup"
        echo "  failover  - Initiate failover to standby"
        echo "  restore   - Restore from backup"
        echo "  sync      - Sync remote backups"
        echo "  drill     - Run disaster recovery drill"
        echo "  interactive - Interactive mode (default)"
        ;;
esac
```

---

## Hands-On Exercises

### Exercise 1: Complete Backup Strategy Implementation

Design and implement a comprehensive backup strategy for a production environment.

```sql
-- Create exercise database with sample data
CREATE DATABASE backup_exercise;
\c backup_exercise

-- Create sample tables with data
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    order_date DATE DEFAULT CURRENT_DATE,
    total_amount DECIMAL(12,2),
    status VARCHAR(20) DEFAULT 'pending'
);

-- Insert sample data
INSERT INTO customers (name, email) 
SELECT 
    'Customer ' || i,
    'customer' || i || '@example.com'
FROM generate_series(1, 10000) i;

INSERT INTO orders (customer_id, total_amount, status)
SELECT 
    (random() * 10000)::int + 1,
    (random() * 1000 + 10)::numeric(12,2),
    CASE (random() * 3)::int
        WHEN 0 THEN 'pending'
        WHEN 1 THEN 'completed'
        ELSE 'cancelled'
    END
FROM generate_series(1, 50000);

-- Test backup and recovery procedures
SELECT 
    pg_size_pretty(pg_database_size('backup_exercise')) AS database_size,
    (SELECT count(*) FROM customers) AS customer_count,
    (SELECT count(*) FROM orders) AS order_count;
```

---

## Best Practices

### Backup Strategy Guidelines

#### 3-2-1 Backup Rule Implementation
```sql
-- Function to verify 3-2-1 backup compliance
CREATE OR REPLACE FUNCTION check_backup_compliance()
RETURNS TABLE(
    compliance_check TEXT,
    status TEXT,
    details TEXT
) AS $$
BEGIN
    -- Check for multiple backup copies (3 copies)
    RETURN QUERY
    SELECT 
        '3 Copies'::TEXT,
        CASE WHEN backup_count >= 3 THEN 'PASS' ELSE 'FAIL' END,
        'Found ' || backup_count || ' backup copies'::TEXT
    FROM (
        SELECT COUNT(*) as backup_count
        FROM (
            SELECT 'local_physical' as backup_type, count(*) as cnt
            FROM pg_ls_dir('/backup') AS f 
            WHERE f ~ '^base_backup_'
            
            UNION ALL
            
            SELECT 'local_logical' as backup_type, count(*) as cnt
            FROM pg_ls_dir('/backup/logical') AS f 
            WHERE f ~ '\.dump$'
        ) backup_counts
    ) t;
    
    -- Add more compliance checks...
END;
$$ LANGUAGE plpgsql;
```

### Backup Security
```sql
-- Function to encrypt sensitive backup data
CREATE OR REPLACE FUNCTION create_encrypted_backup(
    database_name TEXT,
    encryption_key TEXT
) RETURNS TEXT AS $$
DECLARE
    backup_file TEXT;
    encrypted_file TEXT;
BEGIN
    backup_file := '/tmp/' || database_name || '_' || 
                   to_char(now(), 'YYYYMMDD_HH24MISS') || '.dump';
    encrypted_file := backup_file || '.enc';
    
    -- Create backup
    PERFORM pg_dump(database_name, backup_file);
    
    -- Encrypt backup (pseudocode - would use actual encryption tool)
    -- PERFORM encrypt_file(backup_file, encrypted_file, encryption_key);
    
    -- Remove unencrypted backup
    -- PERFORM delete_file(backup_file);
    
    RETURN encrypted_file;
END;
$$ LANGUAGE plpgsql;
```

---

## Next Steps

### Module Completion Checklist
- [ ] Understand different backup strategies and their use cases
- [ ] Implement automated backup procedures
- [ ] Configure point-in-time recovery
- [ ] Set up streaming replication
- [ ] Create disaster recovery procedures
- [ ] Test backup and recovery processes regularly

### Immediate Next Steps
1. **Implement** appropriate backup strategy for your environment
2. **Test** recovery procedures regularly
3. **Monitor** backup success and integrity
4. **Move to Module 13**: [Performance Tuning](13-performance-tuning.md)

---

## Module Summary

✅ **What You've Learned:**
- Comprehensive backup strategies (physical and logical)
- Point-in-time recovery implementation
- Continuous archiving and WAL management
- Streaming replication setup and monitoring
- Disaster recovery planning and procedures
- Backup monitoring and validation

✅ **Skills Acquired:**
- Design backup strategies for different scenarios
- Implement automated backup and recovery procedures
- Configure and monitor streaming replication
- Handle disaster recovery situations
- Validate backup integrity and performance

✅ **Ready for Next Module:**
You're now ready to optimize PostgreSQL performance in [Module 13: Performance Tuning](13-performance-tuning.md).

---

## Quick Reference

### Backup Commands
```bash
# Physical backup
pg_basebackup -D /backup/base -Ft -z -X stream -P -v

# Logical backup
pg_dump -Fc -v -f backup.dump database_name

# Restore logical backup
pg_restore -d database_name backup.dump

# Point-in-time recovery
# Create recovery.signal and configure postgresql.auto.conf
```

### Replication Commands
```sql
-- Create replication slot
SELECT pg_create_physical_replication_slot('slot_name');

-- Monitor replication
SELECT * FROM pg_stat_replication;

-- Check standby status
SELECT pg_is_in_recovery(), pg_last_wal_receive_lsn();
```

---
*Continue your PostgreSQL journey with [Performance Tuning](13-performance-tuning.md) →*
