# Docker Basics: Images, Containers, and Commands

## Overview
Understand the core concepts of Docker: images, containers, and essential commands.

## Key Concepts
- **Image:** Blueprint for containers
- **Container:** Running instance of an image
- **Docker Hub:** Public registry for images

## Essential Commands
```bash
# List images
docker images
# Pull image
docker pull ubuntu:latest
# Run container
docker run -it ubuntu:latest bash
# List running containers
docker ps
# Stop container
docker stop <container_id>
# Remove container
docker rm <container_id>
# Remove image
docker rmi <image_id>
```

## Hands-On Exercise
- Pull and run an Ubuntu container
- List, stop, and remove containers

## Self-Assessment
- Can you describe the difference between an image and a container?
- Did you run and manage a container?

---
Next: [03-Dockerfiles.md](03-Dockerfiles.md)
