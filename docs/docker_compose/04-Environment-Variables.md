# Environment Variables in Docker Compose

## Overview
Configure services using environment variables and .env files.

## Using Environment Variables
- Inline in Compose file
- From .env file

## Example
```yaml
version: '3.8'
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
```

## .env File Example
```
DB_PASSWORD=supersecret
```

## Hands-On Exercise
- Create a .env file and reference it in Compose

## Self-Assessment
- Can you use environment variables in Compose?
- Did you configure a service with .env?

---
Next: [05-Scaling-Services.md](05-Scaling-Services.md)
