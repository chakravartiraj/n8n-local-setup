# Scaling Services with Docker Compose

## Overview
Learn how to scale services and manage replicas in Docker Compose.

## Scaling Services
- Use the `--scale` flag to run multiple instances of a service.
- Example:
```bash
docker compose up --scale web=3
```

## Load Balancing
- Compose uses round-robin DNS for service discovery.
- For advanced load balancing, integrate with NGINX or Traefik.

## Hands-On Exercise
- Scale a web service and observe multiple containers running.
- Test load balancing by accessing the service repeatedly.

## Best Practices
- Design stateless services for easy scaling.
- Monitor resource usage and set limits.

## Self-Assessment
- Can you scale a service and explain how requests are distributed?
- Did you observe the effect of scaling in your stack?

---
Next: [06-Advanced-Compose.md](06-Advanced-Compose.md)
