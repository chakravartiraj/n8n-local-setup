# Simple Docker Compose Stacks

## Overview
Build a basic web + database stack using Compose.

## Example: Web + Database
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

## Hands-On Exercise
- Deploy a web + database stack
- Access the web service in your browser

## Self-Assessment
- Can you explain how services interact?
- Did you deploy a stack?

---
Next: [04-Environment-Variables.md](04-Environment-Variables.md)
