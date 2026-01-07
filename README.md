# N8N Local Development Setup

This setup allows you to run n8n locally using Docker Compose v2 with data persistence and environment variables.

## Features

- ✅ PostgreSQL database with persistence
- ✅ Data persistence using Docker volumes
- ✅ Environment variables configuration
- ✅ Health checks
- ✅ Basic authentication
- ✅ Community packages support
- ✅ Proper networking
- ✅ Auto-restart on failure

## Quick Start

1. **Start n8n:**
   ```bash
   docker compose up -d
   ```

2. **Access n8n:**
   - URL: http://localhost:5678
   - Username: `admin`
   - Password: `password`

3. **Stop n8n:**
   ```bash
   docker compose down
   ```

4. **View logs:**
   ```bash
   docker compose logs -f n8n
   ```

## Configuration

### Environment Variables

Edit the `.env` file to customize your n8n instance:

- `N8N_BASIC_AUTH_USER`: Username for basic auth
- `N8N_BASIC_AUTH_PASSWORD`: Password for basic auth
- `N8N_HOST`: Hostname (default: localhost)
- `N8N_PORT`: Port (default: 5678)
- `GENERIC_TIMEZONE`: Timezone setting
- `N8N_LOG_LEVEL`: Log verbosity (debug, info, warn, error)

### Database Configuration

- `DB_POSTGRESDB_DATABASE`: PostgreSQL database name
- `DB_POSTGRESDB_USER`: PostgreSQL username  
- `DB_POSTGRESDB_PASSWORD`: PostgreSQL password
- `POSTGRES_DB`: PostgreSQL database name (for container)
- `POSTGRES_USER`: PostgreSQL user (for container)
- `POSTGRES_PASSWORD`: PostgreSQL password (for container)

### Data Persistence

- **n8n data**: All n8n data (workflows, credentials, executions) is stored in the `n8n_data` Docker volume
- **PostgreSQL data**: Database data is stored in the `postgres_data` Docker volume
- Both volumes ensure your work persists between container restarts

## Useful Commands

```bash
# Start in foreground (see logs in real-time)
docker compose up

# Start in background
docker compose up -d

# Stop services
docker compose down

# Stop and remove volumes (WARNING: This will delete all data!)
docker compose down -v

# View logs
docker compose logs -f n8n

# Restart n8n
docker compose restart n8n

# Update to latest n8n version
docker compose pull && docker compose up -d

# Access PostgreSQL database
docker compose exec postgres psql -U n8n -d n8n

# View PostgreSQL logs
docker compose logs postgres

# Backup PostgreSQL database
docker compose exec postgres pg_dump -U n8n n8n > n8n_backup.sql

# Restore PostgreSQL database (stop n8n first)
docker compose stop n8n
docker compose exec -T postgres psql -U n8n -d n8n < n8n_backup.sql
docker compose start n8n
```

## Troubleshooting

### Check container status:
```bash
docker compose ps
```

### View detailed logs:
```bash
docker compose logs n8n
```

### Access container shell:
```bash
docker compose exec n8n sh
```

### Reset everything (WARNING: Deletes all data):
```bash
docker compose down -v
docker compose up -d
```

### If Docker is installed and running! Yet can't run docker commands in terminal

Likely, the docker command is not yet in your system PATH. This is likely because the symlinks failed to create automatically.

Please run these commands to fix it:
```bash
sudo ln -s /Applications/Docker.app/Contents/Resources/bin/docker /usr/local/bin/docker
sudo ln -s /Applications/Docker.app/Contents/Resources/bin/docker-compose /usr/local/bin/docker-compose
sudo ln -s /Applications/Docker.app/Contents/Resources/bin/docker-credential-desktop /usr/local/bin/docker-credential-desktop
sudo ln -s /Applications/Docker.app/Contents/Resources/bin/hub-tool /usr/local/bin/hub-tool
sudo ln -s /Applications/Docker.app/Contents/Resources/bin/com.docker.cli /usr/local/bin/com.docker.cli
```
After running these, verifying with docker --version should work. Let me know when you've done this so I can verify and tidy up!