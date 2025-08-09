# Production Patterns for Docker Compose

## Overview
Secure and optimize Compose stacks for production use.

## Secrets Management
- Use Docker secrets for sensitive data.
- Example:
```yaml
secrets:
  db_password:
    file: ./db_password.txt
```

## Resource Limits
- Set CPU and memory constraints.
- Example:
```yaml
services:
  web:
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
```

## Logging
- Configure centralized logging with drivers.

## Hands-On Exercise
- Add secrets and resource limits to a stack.

## Best Practices
- Regularly update images and monitor logs.

## Self-Assessment
- Can you secure and optimize a Compose stack?
- Did you implement secrets and limits?

---
Next: [10-Troubleshooting.md](10-Troubleshooting.md)
