# Service Health Monitoring in Docker Compose

## Overview
Monitor and manage service health with healthchecks.

## Healthcheck Syntax
- Example:
```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## Monitoring Health
- Use `docker compose ps` to view health status.
- Automate recovery with restart policies.

## Hands-On Exercise
- Add healthchecks to a service and observe status changes.

## Best Practices
- Set appropriate intervals and retries.

## Self-Assessment
- Can you configure and monitor healthchecks?
- Did you automate recovery for unhealthy services?

---
Next: [08-Dependencies.md](08-Dependencies.md)
