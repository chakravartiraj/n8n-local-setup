# Docker in CI/CD Pipelines

## Overview
Integrate Docker into automated build, test, and deployment workflows.

## Common Tools
- GitHub Actions
- GitLab CI
- Jenkins

## Example: GitHub Actions
```yaml
name: Build and Push Docker Image
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Push image
        run: docker push myapp:${{ github.sha }}
```

## Hands-On Exercise
- Set up a CI pipeline to build and push a Docker image

## Self-Assessment
- Can you describe how Docker fits into CI/CD?
- Did you automate a Docker build?

---
Next: [12-Production-Patterns.md](12-Production-Patterns.md)
