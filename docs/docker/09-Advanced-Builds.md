# Advanced Docker Builds

## Overview
Master multi-stage builds, BuildKit, and advanced image creation.

## Multi-Stage Builds
```dockerfile
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

## BuildKit Features
- Faster builds
- Better caching
- Secrets management

## Hands-On Exercise
- Create a multi-stage Dockerfile
- Enable BuildKit and build an image

## Self-Assessment
- Can you explain multi-stage builds?
- Did you use BuildKit features?

---
Next: [10-Orchestration.md](10-Orchestration.md)
