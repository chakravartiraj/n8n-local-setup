# Docker Networking

## Overview
Learn how containers communicate, set up custom networks, and expose services.

## Networking Concepts
- **Bridge Network:** Default, containers on same host
- **Host Network:** Shares host’s network stack
- **Overlay Network:** Multi-host, for Swarm/Kubernetes

## Commands
```bash
# List networks
docker network ls
# Create network
docker network create mynet
# Run container on network
docker run --network mynet nginx
# Inspect network
docker network inspect mynet
```

## Exposing Ports
```bash
# Map host port to container port
docker run -p 8080:80 nginx
```

## Hands-On Exercise
- Create a custom network
- Run containers and test connectivity

## Self-Assessment
- Can you explain bridge vs host networks?
- Did you expose a service and connect containers?

---
Next: [06-Docker-Compose.md](06-Docker-Compose.md)
