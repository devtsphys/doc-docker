# doc-docker

# Docker Reference Card & Cheat Sheet

## Table of Contents
- [Dockerfile Best Practices](#dockerfile-best-practices)
- [Docker CLI Commands](#docker-cli-commands)
- [Docker Compose](#docker-compose)
- [Docker Networking](#docker-networking)
- [Docker Volumes](#docker-volumes)
- [Docker Security](#docker-security)
- [Troubleshooting](#troubleshooting)

## Dockerfile Best Practices

### Base Image Selection
```dockerfile
# Good: Use specific version tags
FROM node:18.15-alpine

# Avoid: Using latest tag
# FROM node:latest
```

### Layer Optimization
```dockerfile
# Good: Combine commands to reduce layers
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Avoid: Multiple RUN commands for related operations
# RUN apt-get update
# RUN apt-get install -y python3
# RUN apt-get install -y python3-pip
# RUN rm -rf /var/lib/apt/lists/*
```

### Multi-stage Builds
```dockerfile
# Build stage
FROM golang:1.20 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Runtime stage
FROM alpine:3.17
WORKDIR /app
COPY --from=builder /app/myapp .
CMD ["./myapp"]
```

### Leverage Build Cache
```dockerfile
# Good: Copy package files first, then install dependencies
COPY package.json package-lock.json ./
RUN npm install
# Other files change more frequently
COPY . .

# Avoid: Copying all files before installing dependencies
# COPY . .
# RUN npm install
```

### Non-root User
```dockerfile
# Create a user and switch to it
RUN useradd -r -u 1001 -g appgroup appuser
USER appuser

# For Alpine
RUN addgroup -S appgroup && adduser -S -G appgroup appuser
USER appuser
```

### WORKDIR Usage
```dockerfile
# Good: Use WORKDIR instead of RUN mkdir + CD
WORKDIR /app

# Avoid:
# RUN mkdir -p /app
# RUN cd /app
```

### .dockerignore Usage
Create a `.dockerignore` file:
```
node_modules
npm-debug.log
.git
.gitignore
*.md
.env
```

### Environment Variables
```dockerfile
# Set default value that can be overridden at runtime
ENV NODE_ENV=production

# For multiple related variables
ENV APP_HOME=/app \
    APP_PORT=3000 \
    APP_VERSION=1.0.0
```

### LABEL for Metadata
```dockerfile
LABEL maintainer="name@example.com" \
      version="1.0" \
      description="My application container"
```

### Proper CMD and ENTRYPOINT
```dockerfile
# For applications with default commands
CMD ["node", "app.js"]

# For executable wrappers
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["--help"]

# Shell form vs Exec form
# Prefer exec form:
CMD ["nginx", "-g", "daemon off;"]
# Avoid shell form:
# CMD nginx -g "daemon off;"
```

## Docker CLI Commands

### Basic Commands
```bash
# Build an image from a Dockerfile
docker build -t myapp:1.0 .

# Run a container
docker run -d -p 8080:80 --name mycontainer myapp:1.0

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# List images
docker images

# Pull an image from registry
docker pull nginx:alpine

# Push an image to registry
docker push myregistry/myapp:1.0

# Remove a container
docker rm mycontainer

# Remove an image
docker rmi myapp:1.0

# Stop a container
docker stop mycontainer

# Start a stopped container
docker start mycontainer

# Restart a container
docker restart mycontainer

# Display container logs
docker logs mycontainer

# Follow logs in real time
docker logs -f mycontainer
```

The port publishing follows the logic
```
HOST_PORT:CONTAINER_PORT
```
with
* HOST_PORT: The port number on your host machine where you want to receive traffic
* CONTAINER_PORT: The port number within the container that's listening for connections

### Container Management
```bash
# Execute a command in a running container
docker exec -it mycontainer bash

# Copy files to/from a container
docker cp mycontainer:/app/log.txt ./local/path/
docker cp ./local/file.txt mycontainer:/app/

# Inspect container details
docker inspect mycontainer

# View container resource usage
docker stats mycontainer

# Pause/unpause container processes
docker pause mycontainer
docker unpause mycontainer

# Rename a container
docker rename mycontainer newname

# Show container port mappings
docker port mycontainer

# Update container resources
docker update --memory 512m --cpus 0.5 mycontainer
```

### Image Management
```bash
# Tag an image
docker tag myapp:1.0 myapp:latest

# Build with no cache
docker build --no-cache -t myapp:1.0 .

# Save an image to a tar archive
docker save -o myapp.tar myapp:1.0

# Load an image from a tar archive
docker load -i myapp.tar

# Display image history
docker history myapp:1.0

# Create an image from a container
docker commit mycontainer myapp:custom

# Prune unused images
docker image prune

# Prune all unused objects
docker system prune -a
```

### Advanced Run Options
```bash
# Run with environment variables
docker run -e VARIABLE=value myapp:1.0

# Run with volume mount
docker run -v /host/path:/container/path myapp:1.0

# Run with bind mount (more explicit)
docker run --mount type=bind,source=/host/path,target=/container/path myapp:1.0

# Run with port publishing
docker run -p 8080:80 -p 443:443 myapp:1.0

# Run with network
docker run --network my-network myapp:1.0

# Run with resource limits
docker run --memory=512m --cpus=0.5 myapp:1.0

# Run in the background (detached)
docker run -d myapp:1.0

# Run interactively with a terminal
docker run -it myapp:1.0 bash

# Run with a custom hostname
docker run --hostname myhost myapp:1.0

# Run with a specific user
docker run --user 1000:1000 myapp:1.0

# Run with specific restart policy
docker run --restart=always myapp:1.0

# Run with healthcheck
docker run --health-cmd="curl -f http://localhost || exit 1" myapp:1.0
```

## Docker Compose

### Basic docker-compose.yml
```yaml
version: '3.8'

services:
  web:
    build: ./web
    ports:
      - "8080:80"
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/mydb
    volumes:
      - ./web:/app
    restart: unless-stopped
    
  db:
    image: postgres:13
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    ports:
      - "5432:5432"

volumes:
  postgres_data:
```

### Common Compose Commands
```bash
# Start services (detached mode)
docker-compose up -d

# Stop services
docker-compose down

# Stop services and remove volumes
docker-compose down -v

# View logs from all services
docker-compose logs

# View logs from a specific service
docker-compose logs web

# Follow logs in real time
docker-compose logs -f

# Rebuild services
docker-compose build

# Rebuild and start services
docker-compose up -d --build

# Show running services
docker-compose ps

# Execute command in a service
docker-compose exec web bash

# Scale a service
docker-compose up -d --scale web=3

# View compose config
docker-compose config
```

## Docker Networking

### Network Types
- **bridge**: Default network for containers on a host
- **host**: Container uses the host's network
- **none**: Container has no network access
- **overlay**: Multi-host networking
- **macvlan**: Assign MAC address to container

### Network Commands
```bash
# List networks
docker network ls

# Create a network
docker network create my-network

# Create an overlay network for swarm
docker network create --driver overlay swarm-network

# Inspect a network
docker network inspect my-network

# Connect a container to a network
docker network connect my-network mycontainer

# Disconnect a container from a network
docker network disconnect my-network mycontainer

# Remove a network
docker network rm my-network

# Prune unused networks
docker network prune
```

### Network in docker-compose.yml
```yaml
version: '3.8'

networks:
  frontend:
  backend:
    internal: true  # No outbound connectivity
  custom:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

services:
  web:
    image: nginx
    networks:
      - frontend
      - backend
      
  db:
    image: postgres
    networks:
      - backend
```

## Docker Volumes

### Volume Types
- **Named volumes**: Managed by Docker
- **Bind mounts**: Host directory mounted in container
- **tmpfs mounts**: Stored in host memory only

### Volume Commands
```bash
# List volumes
docker volume ls

# Create a volume
docker volume create my-volume

# Inspect a volume
docker volume inspect my-volume

# Remove a volume
docker volume rm my-volume

# Prune unused volumes
docker volume prune

# Run container with a named volume
docker run -v my-volume:/app/data myapp:1.0

# Run container with a bind mount
docker run -v $(pwd):/app myapp:1.0

# Run container with a tmpfs mount
docker run --tmpfs /app/temp myapp:1.0
```

### Volumes in docker-compose.yml
```yaml
version: '3.8'

volumes:
  db_data:
    external: false  # Managed by this compose file
  logs:
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '/var/log/myapp'

services:
  app:
    image: myapp:1.0
    volumes:
      - ./config:/app/config:ro  # Bind mount (read-only)
      - db_data:/app/data        # Named volume
      - logs:/app/logs           # Named volume with custom options
```

## Docker Security

### Security Best Practices
1. **Use official images** from trusted sources
2. **Scan images** for vulnerabilities with `docker scan`
3. **Run containers as non-root** user
4. **Limit capabilities** using `--cap-drop` and `--cap-add`
5. **Set resource limits** to prevent DoS
6. **Use read-only file systems** when possible
7. **Enable content trust** with `DOCKER_CONTENT_TRUST=1`
8. **Implement network segmentation**
9. **Regular security updates**
10. **Restrict container communication**

### Security-focused Commands
```bash
# Run with dropped capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp:1.0

# Run with read-only filesystem
docker run --read-only myapp:1.0

# Run with seccomp profile
docker run --security-opt seccomp=/path/to/seccomp.json myapp:1.0

# Run with no new privileges
docker run --security-opt no-new-privileges myapp:1.0

# Run with limited memory and CPU
docker run --memory=512m --cpus=0.5 myapp:1.0

# Enable content trust
export DOCKER_CONTENT_TRUST=1
docker pull nginx:alpine
```

## Troubleshooting

### Common Issues & Solutions

#### Container won't start
```bash
# Check container logs
docker logs mycontainer

# Check container exit code
docker inspect mycontainer --format='{{.State.ExitCode}}'

# Check container configuration
docker inspect mycontainer
```

#### Image build fails
```bash
# Build with verbose output
docker build --progress=plain -t myapp:1.0 .

# Check available disk space
docker system df

# Clean up to free space
docker system prune -a
```

#### Network connectivity issues
```bash
# Inspect the container's network settings
docker inspect --format='{{json .NetworkSettings.Networks}}' mycontainer

# Test network from within container
docker exec mycontainer ping -c 3 google.com

# Check if container port is exposed correctly
docker port mycontainer
```

#### Performance issues
```bash
# Check resource usage
docker stats mycontainer

# Check running processes in container
docker top mycontainer

# Get system-wide information
docker info
```

#### Volume mount issues
```bash
# Verify volume exists
docker volume inspect myvolume

# Check container mounts
docker inspect --format='{{json .Mounts}}' mycontainer

# Verify file permissions on host
ls -la /path/to/host/directory
```

### Debug Flags
```bash
# Enable debug mode on Docker daemon
dockerd -D

# View Docker daemon logs
journalctl -u docker.service

# Run container with debug options
docker run --log-driver=json-file --log-opt debug myapp:1.0
```
