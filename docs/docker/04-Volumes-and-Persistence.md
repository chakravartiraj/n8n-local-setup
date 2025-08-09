# Volumes and Persistence

## Overview
Learn how to manage data in Docker containers using volumes and bind mounts.

## Why Volumes?
- Persist data beyond container lifecycle
- Share data between containers
- Backup and restore data easily

## Types of Storage
- **Volumes:** Managed by Docker
- **Bind Mounts:** Link host directory to container

## Commands
```bash
# Create a volume
docker volume create mydata
# List volumes
docker volume ls
# Run container with volume
docker run -v mydata:/data ubuntu:latest
# Bind mount example
docker run -v /host/path:/container/path ubuntu:latest
```

## Hands-On Exercise
- Create a volume and use it in a container
- Try a bind mount

## Self-Assessment
- Can you explain the difference between volumes and bind mounts?
- Did you persist data using a volume?

---
Next: [05-Networking.md](05-Networking.md)
