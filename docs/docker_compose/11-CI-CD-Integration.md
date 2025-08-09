# CI/CD Integration with Docker Compose

## Overview
Automate builds, tests, and deployments using Compose in CI/CD pipelines.

## Compose in DevOps
- Use Compose in GitHub Actions, GitLab CI, Jenkins.

## Example: GitHub Actions
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Compose Up
        run: docker compose up -d
      - name: Run Tests
        run: docker compose exec web npm test
```

## Hands-On Exercise
- Set up a CI pipeline with Compose.

## Best Practices
- Automate testing and cleanup.

## Self-Assessment
- Can you use Compose in CI/CD?
- Did you automate a deployment?

---
Next: [12-Expert-Patterns.md](12-Expert-Patterns.md)
