# Getting Started with Docker

## Overview
Learn what Docker is, why it matters, and how to install it on your system. This module is for absolute beginners.

## What is Docker?
Docker is a platform for building, shipping, and running applications in containers. Containers are lightweight, portable, and consistent across environments.

## Why Use Docker?
- Consistent development and production environments
- Easy dependency management
- Fast, reproducible deployments

## Installation
### Linux
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```
### Windows & Mac
- Download Docker Desktop from [docker.com](https://www.docker.com/products/docker-desktop)
- Follow the installation wizard

## First Steps
```bash
# Verify installation
docker --version
# Run hello-world container
docker run hello-world
```

## Hands-On Exercise
- Install Docker
- Run your first container
- Check Docker version

## Self-Assessment
- Can you explain what a container is?
- Did you install Docker and run hello-world?

---
Next: [02-Docker-Basics.md](02-Docker-Basics.md)
