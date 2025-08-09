# Docker Compose Basic Concepts

## Overview
Understand services, networks, and volumes in Compose.

## Key Concepts
- **Service:** Defines a container
- **Network:** Connects containers
- **Volume:** Persists data

## Example Compose File
```yaml
version: '3.8'
services:
  app:
    image: node:18
    volumes:
      - appdata:/usr/src/app
    networks:
      - appnet
volumes:
  appdata:
networks:
  appnet:
```

## Hands-On Exercise
- Create a Compose file with a service, network, and volume

## Self-Assessment
- Can you describe the role of services, networks, and volumes?
- Did you create a basic Compose file?

---
Next: [03-Simple-Stacks.md](03-Simple-Stacks.md)
