# N8N Update Guide

## Overview
This guide covers updating your Docker Compose n8n installation safely and efficiently.

## Current Setup
- **Installation Method**: Docker Compose with PostgreSQL
- **Image**: `n8nio/n8n:latest`
- **Database**: PostgreSQL 15 Alpine
- **Data Persistence**: Docker volumes

## Update Frequency
- **Recommended**: Monthly updates or when critical security patches are released
- **Check**: [N8N Release Notes](https://github.com/n8n-io/n8n/releases) before updating
- **Breaking Changes**: Always review release notes for breaking changes

## Pre-Update Checklist
- [ ] Review latest release notes
- [ ] Ensure you have time for testing
- [ ] Notify users of potential downtime
- [ ] Verify backup space availability

## Update Process

### 1. Create Backup
```bash
# Navigate to project directory
cd /home/kronos/Documents/n8n

# Create timestamped backup directory
BACKUP_DIR="backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# Backup PostgreSQL database
docker exec n8n_postgres pg_dump -U n8n -d n8n > "$BACKUP_DIR/n8n_database_backup.sql"

# Backup n8n data volume
docker run --rm -v n8n_n8n_data:/data -v $(pwd)/$BACKUP_DIR:/backup alpine tar czf /backup/n8n_data_backup.tar.gz -C /data .

echo "Backup created in: $BACKUP_DIR"
```

### 2. Check Current and Latest Versions
```bash
# Check current version
docker exec n8n n8n --version

# Check latest available version
curl -s https://api.github.com/repos/n8n-io/n8n/releases/latest | grep '"tag_name"' | cut -d'"' -f4
```

### 3. Update Images
```bash
# Pull latest images
docker compose pull

# Stop services
docker compose down

# Start with updated images
docker compose up -d
```

### 4. Verify Update
```bash
# Wait for services to start (about 30 seconds)
sleep 30

# Check new version
docker exec n8n n8n --version

# Verify services are healthy
docker compose ps

# Test web interface
curl -f http://localhost:5678/healthz || echo "Health check failed"
```

### 5. Post-Update Testing
- [ ] Access web interface: http://localhost:5678
- [ ] Test existing workflows
- [ ] Verify database connections
- [ ] Check community packages still work
- [ ] Test webhook endpoints

## Rollback Process (If Needed)

### Quick Rollback
```bash
# Stop current services
docker compose down

# Restore from backup
BACKUP_DIR="backups/YYYYMMDD_HHMMSS"  # Replace with your backup directory

# Restore database
docker compose up -d postgres
sleep 10
docker exec -i n8n_postgres psql -U n8n -d n8n < "$BACKUP_DIR/n8n_database_backup.sql"

# Restore data volume
docker run --rm -v n8n_n8n_data:/data -v $(pwd)/$BACKUP_DIR:/backup alpine tar xzf /backup/n8n_data_backup.tar.gz -C /data

# Start services
docker compose up -d
```

## Advanced Update Options

### Pin to Specific Version
If you want to use a specific version instead of `:latest`:

```yaml
# In docker-compose.yml, change:
image: n8nio/n8n:latest
# To:
image: n8nio/n8n:1.105.4  # Replace with desired version
```

### Test in Staging Environment
1. Copy entire project to test directory
2. Update test environment first
3. Validate all functionality
4. Apply to production

## Monitoring After Update

### Health Checks
```bash
# Container health
docker compose ps

# Application health
curl -f http://localhost:5678/healthz

# Database health
docker exec n8n_postgres pg_isready -U n8n -d n8n

# Check logs for errors
docker compose logs n8n --tail=50
docker compose logs postgres --tail=50
```

### Performance Monitoring
- Watch for increased memory usage
- Monitor workflow execution times
- Check database performance
- Verify webhook response times

## Troubleshooting

### Common Issues After Update

#### Database Connection Issues
```bash
# Check database status
docker compose logs postgres

# Reset database connection
docker compose restart n8n
```

#### Permission Problems
```bash
# Fix volume permissions
docker compose down
docker run --rm -v n8n_n8n_data:/data alpine chown -R 1000:1000 /data
docker compose up -d
```

#### Community Packages Broken
```bash
# Reinstall community packages
docker exec n8n npm install -g [package-name]
docker compose restart n8n
```

## Update History

| Date | Version | Notes |
|------|---------|-------|
| $(date +%Y-%m-%d) | 1.105.4 | Updated from 1.104.2 - Check release notes |

## Emergency Contacts
- **N8N Support**: [Community Forum](https://community.n8n.io/)
- **Documentation**: [Official Docs](https://docs.n8n.io/)
- **Release Notes**: [GitHub Releases](https://github.com/n8n-io/n8n/releases)

## Maintenance Schedule
- **Monthly**: Check for updates
- **Quarterly**: Review and update documentation
- **Annually**: Evaluate upgrade strategy and infrastructure

---
*Last updated: $(date +%Y-%m-%d)*
