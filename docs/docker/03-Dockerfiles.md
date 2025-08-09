# Dockerfiles: Building Custom Images

## Overview
Learn how to create Dockerfiles to build your own images.

## What is a Dockerfile?
A Dockerfile is a text file with instructions to build a Docker image.

## Basic Dockerfile Example
```dockerfile
# Use official Python image
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

## Building an Image
```bash
docker build -t my-python-app .
```

## Hands-On Exercise
- Write a Dockerfile for a simple Python app
- Build and run your custom image

## Self-Assessment
- Can you explain each instruction in the Dockerfile?
- Did you build and run your own image?

---
Next: [04-Volumes-and-Persistence.md](04-Volumes-and-Persistence.md)
