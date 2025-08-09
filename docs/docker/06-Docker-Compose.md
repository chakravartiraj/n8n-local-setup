# Docker Compose: Multi-Container Applications

## Overview
Orchestrate multi-container apps with a single YAML file.

## What is Docker Compose?
Compose lets you define and run multi-container Docker applications.

## Basic Example
```yaml
version: '3.8'
services:
  web:
    image: nginx
    ports:
      - "8080:80"
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: example
```

## Commands
```bash
# Start services
docker-compose up
# Stop services
docker-compose down
# View logs
docker-compose logs
```

## Hands-On Exercise
- Build a web + database stack
- Scale services with `docker-compose up --scale web=3`

## Self-Assessment
- Can you describe the role of docker-compose?
- Did you run a multi-container app?

---
Next: [07-Best-Practices.md](07-Best-Practices.md)
