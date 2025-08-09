# Service Dependencies in Docker Compose

## Overview
Manage service startup order and dependencies.

## depends_on Keyword
- Controls startup order.
- Example:
```yaml
services:
  web:
    depends_on:
      - db
```

## Wait-for-it Scripts
- Use scripts to wait for services to be ready.
- Example:
```yaml
command: ["./wait-for-it.sh", "db:5432", "--", "npm", "start"]
```

## Hands-On Exercise
- Implement `depends_on` and wait-for-it in a stack.

## Best Practices
- Ensure robust startup with healthchecks and wait scripts.

## Self-Assessment
- Can you manage dependencies and startup order?
- Did you use wait-for-it for service readiness?

---
Next: [09-Production-Patterns.md](09-Production-Patterns.md)
