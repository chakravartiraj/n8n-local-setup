# Docker Production Patterns

## Overview
Deploy Docker containers in production with reliability and scalability.

## Key Patterns
- Blue/Green Deployments
- Canary Releases
- Health Checks
- Resource Limits

## Example: Health Check
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost/health"]
  interval: 30s
  timeout: 10s
  retries: 3
```

## Hands-On Exercise
- Implement a health check in a Compose file
- Set resource limits for a service

## Self-Assessment
- Can you explain blue/green deployment?
- Did you add health checks and limits?

---
Next: [13-Hands-On-Projects.md](13-Hands-On-Projects.md)
