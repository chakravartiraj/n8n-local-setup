# Advanced Docker Compose Features

## Overview
Explore multi-file setups, overrides, and profiles for complex projects.

## Multi-File Projects
- Use `-f` to specify multiple Compose files.
- Example:
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

## Overrides and Profiles
- Override settings for dev, test, prod.
- Use profiles to enable/disable services.

## Hands-On Exercise
- Create a base and override Compose file.
- Use profiles to control service startup.

## Best Practices
- Organize files for clarity and maintainability.

## Self-Assessment
- Can you use overrides and profiles in Compose?
- Did you run a multi-file stack?

---
Next: [07-Healthchecks.md](07-Healthchecks.md)
