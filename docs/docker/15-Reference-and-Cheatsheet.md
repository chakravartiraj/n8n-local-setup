# Docker Reference & Cheatsheet

## Overview
Quick reference for Docker commands, options, and troubleshooting.

## Common Commands
```bash
# Images
docker images
docker pull <image>
docker rmi <image>

# Containers
docker ps
docker run <image>
docker stop <container>
docker rm <container>

# Volumes
docker volume create <name>
docker volume ls
docker volume rm <name>

# Networks
docker network create <name>
docker network ls
docker network rm <name>

# Compose
docker-compose up
docker-compose down
docker-compose logs
```

## Troubleshooting
- Check logs: `docker logs <container>`
- Inspect: `docker inspect <container>`
- Prune unused resources: `docker system prune`

## Resources
- [Official Docs](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Play with Docker](https://labs.play-with-docker.com/)

---
*End of Docker Learning Path*
