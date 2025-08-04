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
- [Base Images](#base-images)

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

### Core Concepts

**Docker Compose**: A tool for defining and running multi-container Docker applications using YAML files to configure application services.

**Supported file names** (in order of priority):

- `docker-compose.yml`
- `docker-compose.yaml`
- `compose.yml`
- `compose.yaml`

**File versions**: Latest is Compose Specification, older numbered versions (2.x, 3.x) are still supported.

### File Structure

Basic structure of a `docker-compose.yml` file:

```yaml
version: "3.9"  # Optional in latest versions

services:
  service1:
    # service configuration
  service2:
    # service configuration

networks:
  # network configuration

volumes:
  # volume configuration

configs:
  # configs configuration

secrets:
  # secrets configuration
```

### Command Reference

|Command                                  |Description                         |
|-----------------------------------------|------------------------------------|
|`docker compose up`                      |Create and start containers         |
|`docker compose up -d`                   |Start in detached mode              |
|`docker compose down`                    |Stop and remove containers, networks|
|`docker compose down -v`                 |Also remove volumes                 |
|`docker compose ps`                      |List running containers             |
|`docker compose logs`                    |View output from containers         |
|`docker compose logs -f`                 |Follow log output                   |
|`docker compose exec <service> <command>`|Execute command in a container      |
|`docker compose build`                   |Build or rebuild services           |
|`docker compose pull`                    |Pull service images                 |
|`docker compose restart`                 |Restart services                    |
|`docker compose stop`                    |Stop services                       |
|`docker compose start`                   |Start services                      |
|`docker compose rm`                      |Remove stopped containers           |
|`docker compose config`                  |Validate and view compose file      |
|`docker compose top`                     |Display running processes           |

## Components & Configuration

### Services

Services define the containers that will be run:

```yaml
services:
  webapp:
    image: nginx:latest       # Use existing image
    build: ./app              # Or build from Dockerfile
    build:                    # Advanced build options
      context: ./app
      dockerfile: Dockerfile.dev
      args:
        ENV: development
    ports:
      - "80:80"               # HOST:CONTAINER
      - "443:443"
    expose:
      - "8000"                # Only to other containers
    volumes:
      - ./app:/usr/share/nginx/html
    environment:
      - NODE_ENV=production
    env_file: .env
    restart: always           # none, on-failure, unless-stopped, always
    depends_on:
      - db
    networks:
      - frontend
      - backend
    deploy:                   # Settings for the deployed containers
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 50M
    user: "1000:1000"         # Run as specific user:group
    working_dir: /app         # Set working directory
    entrypoint: ./entrypoint.sh
    command: npm start        # Override default command
```

### Networks

Define custom networks for communication between containers:

```yaml
networks:
  frontend:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1
  backend:
    driver: bridge
  database:
    external: true          # Use pre-existing network
    name: actual-network-name
```

### Volumes

Define persistent storage for your containers:

```yaml
volumes:
  db-data:
    driver: local           # Default local storage
  cached-data:
    driver_opts:
      type: tmpfs
      device: tmpfs
  logs:
    external: true          # Use pre-existing volume
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.1,rw
      device: ":/path/to/export"
```

### Configs & Secrets

For storing configuration and sensitive data:

```yaml
configs:
  app_config:
    file: ./config/app.json
  http_config:
    content: |
      server {
        listen 80;
        server_name example.com;
      }

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
    name: production_api_key
```

Using configs and secrets in services:

```yaml
services:
  webapp:
    image: nginx
    configs:
      - source: http_config
        target: /etc/nginx/conf.d/site.conf
    secrets:
      - source: api_key
        target: /run/secrets/api_key
        mode: 0400
```

### Advanced Techniques

### Environment Variables

**In docker-compose.yml**:

```yaml
services:
  webapp:
    environment:
      - DEBUG=1
      - DATABASE_URL=postgres://user:pass@db:5432/dbname
```

**Using .env file**:

```yaml
services:
  webapp:
    env_file:
      - .env.common
      - .env.production
```

**Variable substitution**:

```yaml
services:
  webapp:
    image: ${REGISTRY:-localhost}/webapp:${TAG:-latest}
    environment:
      - NODE_ENV=${ENVIRONMENT:-development}
```

### Extending Configurations

**Using extension fields**:

```yaml
# Common configuration to reuse
x-common-service: &common-service
  restart: always
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "3"

services:
  web:
    <<: *common-service  # Include common configuration
    image: nginx
```

**Multiple compose files**:

```bash
# Override default configuration
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

### Dependencies & Health Checks

**Service dependencies**:

```yaml
services:
  webapp:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
```

**Health checks**:

```yaml
services:
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

### Resource Management

```yaml
services:
  webapp:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 50M
        reservations:
          cpus: '0.25'
          memory: 20M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```

### Scaling Services

```bash
# Start multiple replicas of a service
docker compose up --scale webapp=3 --scale worker=2
```

### Examples

### Basic Web Application

```yaml
services:
  web:
    build: ./app
    ports:
      - "8000:8000"
    volumes:
      - ./app:/app
    depends_on:
      - db
  
  db:
    image: postgres:14
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_USER=user
      - POSTGRES_DB=myapp

volumes:
  postgres_data:
```

### Microservices Architecture

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "80:80"
    networks:
      - web
    depends_on:
      - api

  api:
    build: ./api
    ports:
      - "8080:8080"
    networks:
      - web
      - internal
    depends_on:
      - db
      - redis
    environment:
      - DB_HOST=db
      - REDIS_HOST=redis

  db:
    image: postgres:14
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - internal
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password

  redis:
    image: redis:alpine
    networks:
      - internal

networks:
  web:
  internal:

volumes:
  db_data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Development Environment

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - CHOKIDAR_USEPOLLING=true
    stdin_open: true # for interactive sessions
    tty: true        # for terminal access
```

### Best Practices

### File Organization

1. **Multiple environments**:
- Base file: `docker-compose.yml`
- Override files: `docker-compose.dev.yml`, `docker-compose.prod.yml`
1. **Project structure**:
   
   ```
   ├── docker-compose.yml
   ├── docker-compose.override.yml
   ├── .env
   ├── service1/
   │   ├── Dockerfile
   │   └── ...
   ├── service2/
   │   ├── Dockerfile
   │   └── ...
   ```
1. **Using profiles** for grouped services:
   
   ```yaml
   services:
     app:
       profiles: ["app", "dev"]
     db:
       profiles: ["app", "dev"]
     selenium:
       profiles: ["test"]
   ```
   
   ```bash
   # Start only services in the "dev" profile
   docker compose --profile dev up
   ```

### Security

1. **Never hardcode secrets** in Compose files.
   
   ```yaml
   # Bad
   environment:
     - DB_PASSWORD=secretpassword
   
   # Good
   environment:
     - DB_PASSWORD_FILE=/run/secrets/db_password
   secrets:
     - db_password
   ```
1. **Use non-root users** in containers:
   
   ```yaml
   services:
     app:
       user: "1000:1000"
   ```
1. **Set memory and CPU limits**:
   
   ```yaml
   services:
     app:
       deploy:
         resources:
           limits:
             cpus: '0.5'
             memory: 256M
   ```
1. **Restrict network access**:
   
   ```yaml
   services:
     db:
       networks:
         - backend
       # Not exposing ports to host
   ```

### Performance

1. **Use bind mounts sparingly** in production:
   
   ```yaml
   # Development
   volumes:
     - ./src:/app/src
   
   # Production
   volumes:
     - app_data:/app/data
   ```
1. **Optimize build context**:
- Use `.dockerignore` files
- Keep build context small
1. **Use multi-stage builds** for smaller images:
   
   ```dockerfile
   FROM node:16 AS builder
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci
   COPY . .
   RUN npm run build
   
   FROM nginx:alpine
   COPY --from=builder /app/build /usr/share/nginx/html
   ```

### Maintainability

1. **Use extension fields** for common configurations:
   
   ```yaml
   x-logging: &default-logging
     logging:
       driver: "json-file"
       options:
         max-size: "10m"
         max-file: "3"
   
   services:
     web:
       <<: *default-logging
     db:
       <<: *default-logging
   ```
1. **Version control** your Compose files.
1. **Document** with comments:
   
   ```yaml
   services:
     web:
       # This service handles all incoming HTTP requests
       # and serves the frontend application
       image: myapp/frontend:latest
   ```
1. **Use meaningful names** for services, networks, and volumes.

### Troubleshooting

Common issues and solutions:

1. **Connection refused between services**
- Check network configuration
- Ensure service names are used as hostnames
1. **Volume permissions**
- Set appropriate user in service
- Check host directory permissions
1. **Service startup order**
- Use `depends_on` with condition
- Implement retry logic in application
1. **Debugging containers**
   
   ```bash
   # Check logs
   docker compose logs service_name
   
   # Execute into container
   docker compose exec service_name sh
   
   # Check configuration
   docker compose config
   ```
1. **Resource constraints**
- Check container resource usage
- Adjust memory/CPU limits

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

# Base Images

## Operating System Base Images

### Ubuntu

Most popular Linux distribution for containers due to familiarity and extensive package availability.

**Variants:**

- `ubuntu:24.04` (Noble Numbat) - Latest LTS
- `ubuntu:22.04` (Jammy Jellyfish) - Previous LTS
- `ubuntu:20.04` (Focal Fossa) - Older LTS
- `ubuntu:latest` - Points to latest LTS

**Characteristics:**

- Size: ~77MB (24.04)
- Package manager: apt
- Shell: bash
- Best for: General purpose, development, CI/CD

### Alpine Linux

Minimal, security-oriented Linux distribution designed for containers.

**Variants:**

- `alpine:3.19` - Latest stable
- `alpine:3.18` - Previous stable
- `alpine:edge` - Rolling release
- `alpine:latest` - Points to latest stable

**Characteristics:**

- Size: ~7MB
- Package manager: apk
- Shell: ash (not bash)
- C library: musl (not glibc)
- Best for: Production, minimal footprint, security-focused

### Debian

Stable, well-maintained distribution, basis for Ubuntu.

**Variants:**

- `debian:12` (Bookworm) - Current stable
- `debian:11` (Bullseye) - Previous stable
- `debian:12-slim` - Minimal variant (~80MB vs ~124MB)
- `debian:bullseye-slim`

**Characteristics:**

- Size: ~124MB (full), ~80MB (slim)
- Package manager: apt
- Shell: bash
- Best for: Stability, compatibility, production

### CentOS/Rocky Linux/AlmaLinux

Enterprise Linux alternatives after CentOS Stream changes.

**Variants:**

- `rockylinux:9` - RHEL 9 compatible
- `rockylinux:8` - RHEL 8 compatible
- `almalinux:9` - RHEL 9 compatible
- `centos:stream9` - Rolling release

**Characteristics:**

- Size: ~200-230MB
- Package manager: yum/dnf
- Shell: bash
- Best for: Enterprise environments, RHEL compatibility

### Fedora

Cutting-edge Linux distribution from Red Hat.

**Variants:**

- `fedora:39` - Latest stable
- `fedora:38` - Previous stable
- `fedora:rawhide` - Development branch

**Characteristics:**

- Size: ~190MB
- Package manager: dnf
- Shell: bash
- Best for: Latest features, development

## Programming Language Images

### Node.js

Official Node.js runtime images.

**Variants:**

- `node:20` - Current LTS
- `node:18` - Previous LTS
- `node:20-alpine` - Alpine-based (~40MB vs ~400MB)
- `node:20-slim` - Debian slim-based (~200MB)
- `node:20-bullseye` - Debian Bullseye-based

**Characteristics:**

- Includes: Node.js, npm, yarn
- Best for: JavaScript/TypeScript applications

### Python

Official Python interpreter images.

**Variants:**

- `python:3.12` - Latest stable
- `python:3.11` - Previous stable
- `python:3.12-slim` - Minimal Debian-based (~45MB vs ~130MB)
- `python:3.12-alpine` - Alpine-based (~25MB)
- `python:3.12-bullseye` - Full Debian Bullseye

**Characteristics:**

- Includes: Python, pip
- Best for: Python applications, data science, ML

### OpenJDK/Java

Official Java Development Kit images.

**Variants:**

- `openjdk:21` - Latest LTS
- `openjdk:17` - Previous LTS
- `openjdk:11` - Older LTS
- `openjdk:21-alpine` - Alpine-based
- `openjdk:21-slim` - Debian slim-based

**Alternative Java Images:**

- `amazoncorretto:21` - Amazon’s OpenJDK distribution
- `eclipse-temurin:21` - Eclipse Foundation’s OpenJDK
- `adoptopenjdk:21-jre-hotspot` - (deprecated, use eclipse-temurin)

### Go (Golang)

Official Go programming language images.

**Variants:**

- `golang:1.21` - Latest stable
- `golang:1.20` - Previous stable
- `golang:1.21-alpine` - Alpine-based (~110MB vs ~380MB)
- `golang:1.21-bullseye` - Debian-based

**Characteristics:**

- Includes: Go compiler, tools
- Best for: Go applications, microservices

### .NET

Microsoft’s .NET runtime and SDK images.

**Variants:**

- `mcr.microsoft.com/dotnet/runtime:8.0` - Runtime only
- `mcr.microsoft.com/dotnet/sdk:8.0` - Full SDK
- `mcr.microsoft.com/dotnet/aspnet:8.0` - ASP.NET runtime
- `mcr.microsoft.com/dotnet/runtime:8.0-alpine` - Alpine-based

### PHP

Official PHP interpreter images.

**Variants:**

- `php:8.3-fpm` - PHP-FPM for Nginx
- `php:8.3-apache` - PHP with Apache
- `php:8.3-cli` - Command line only
- `php:8.3-alpine` - Alpine-based variants

### Ruby

Official Ruby interpreter images.

**Variants:**

- `ruby:3.3` - Latest stable
- `ruby:3.2` - Previous stable
- `ruby:3.3-alpine` - Alpine-based
- `ruby:3.3-slim` - Debian slim-based

### Rust

Official Rust programming language images.

**Variants:**

- `rust:1.75` - Latest stable
- `rust:1.75-alpine` - Alpine-based
- `rust:1.75-slim` - Debian slim-based

## Web Servers & Reverse Proxies

### Nginx

High-performance web server and reverse proxy.

**Variants:**

- `nginx:1.25` - Latest stable
- `nginx:1.25-alpine` - Alpine-based (~40MB vs ~140MB)
- `nginx:1.25-perl` - With Perl module
- `nginx:mainline` - Development version

**Characteristics:**

- Config location: `/etc/nginx/nginx.conf`
- Document root: `/usr/share/nginx/html`
- Logs: `/var/log/nginx/`

### Apache HTTP Server

Traditional web server with extensive module support.

**Variants:**

- `httpd:2.4` - Latest stable
- `httpd:2.4-alpine` - Alpine-based

**Characteristics:**

- Config location: `/usr/local/apache2/conf/httpd.conf`
- Document root: `/usr/local/apache2/htdocs/`

### Traefik

Modern reverse proxy and load balancer.

**Variants:**

- `traefik:v3.0` - Latest major version
- `traefik:v2.10` - Previous stable
- `traefik:latest`

## Databases

### MySQL

Popular relational database.

**Variants:**

- `mysql:8.0` - Latest stable
- `mysql:5.7` - Older stable (deprecated)
- `mysql:8.0-debian` - Debian-based

**Environment Variables:**

- `MYSQL_ROOT_PASSWORD`
- `MYSQL_DATABASE`
- `MYSQL_USER`/`MYSQL_PASSWORD`

### PostgreSQL

Advanced relational database.

**Variants:**

- `postgres:16` - Latest stable
- `postgres:15` - Previous stable
- `postgres:16-alpine` - Alpine-based

**Environment Variables:**

- `POSTGRES_PASSWORD`
- `POSTGRES_DB`
- `POSTGRES_USER`

### MongoDB

NoSQL document database.

**Variants:**

- `mongo:7.0` - Latest stable
- `mongo:6.0` - Previous stable

### Redis

In-memory data structure store.

**Variants:**

- `redis:7.2` - Latest stable
- `redis:7.2-alpine` - Alpine-based (~30MB vs ~110MB)

### MariaDB

MySQL-compatible database server.

**Variants:**

- `mariadb:11.2` - Latest stable
- `mariadb:10.11` - LTS version

## Container Runtimes & Tools

### Distroless

Google’s minimal container images containing only application and runtime dependencies.

**Variants:**

- `gcr.io/distroless/java:17` - Java runtime
- `gcr.io/distroless/python3` - Python runtime
- `gcr.io/distroless/nodejs` - Node.js runtime
- `gcr.io/distroless/static` - Static binaries only
- `gcr.io/distroless/base` - Minimal base with glibc

**Characteristics:**

- No shell, package manager, or OS utilities
- Extremely secure (minimal attack surface)
- Very small size
- Best for: Production, security-critical applications

### Scratch

Empty base image for static binaries.

**Usage:**

```dockerfile
FROM scratch
COPY myapp /
CMD ["/myapp"]
```

**Characteristics:**

- Size: 0MB
- No OS, shell, or utilities
- Best for: Static Go binaries, minimal containers

### BusyBox

Minimal Unix utilities in a single executable.

**Variants:**

- `busybox:1.36` - Latest stable
- `busybox:uclibc` - uClibc-based
- `busybox:glibc` - glibc-based

**Characteristics:**

- Size: ~2MB
- Includes: Essential Unix utilities
- Shell: ash

## Cloud Provider Images

### Amazon Linux

AWS’s Linux distribution.

**Variants:**

- `amazonlinux:2023` - Latest generation
- `amazonlinux:2` - Previous generation

**Characteristics:**

- Optimized for AWS
- Package manager: yum/dnf
- Size: ~160MB

### Google Cloud

Google’s container-optimized images.

**Variants:**

- `gcr.io/google.com/cloudsdktool/cloud-sdk` - Google Cloud SDK
- `gcr.io/google.com/cloudsdktool/cloud-sdk:alpine`

### Microsoft

Microsoft’s container images.

**Variants:**

- `mcr.microsoft.com/windows/servercore` - Windows Server Core
- `mcr.microsoft.com/windows/nanoserver` - Windows Nano Server

## Specialized Images

### Jenkins

CI/CD automation server.

**Variants:**

- `jenkins/jenkins:lts` - Long Term Support
- `jenkins/jenkins:latest` - Weekly releases

### GitLab

DevOps platform images.

**Variants:**

- `gitlab/gitlab-ce` - Community Edition
- `gitlab/gitlab-ee` - Enterprise Edition
- `gitlab/gitlab-runner` - CI/CD runner

### Elasticsearch

Search and analytics engine.

**Variants:**

- `elasticsearch:8.11.0`
- `elasticsearch:7.17.16` - Previous major version

### Kibana

Data visualization for Elasticsearch.

**Variants:**

- `kibana:8.11.0`
- `kibana:7.17.16`

### Logstash

Data processing pipeline.

**Variants:**

- `logstash:8.11.0`
- `logstash:7.17.16`

### RabbitMQ

Message broker.

**Variants:**

- `rabbitmq:3.12` - Latest stable
- `rabbitmq:3.12-management` - With web UI

### Consul

Service discovery and configuration.

**Variants:**

- `consul:1.16`
- `consul:1.15`

## Image Selection Guidelines

### Size Considerations

1. **Minimal (< 20MB)**: Alpine, Distroless, Scratch
1. **Small (20-100MB)**: Debian slim, Ubuntu core
1. **Medium (100-200MB)**: Standard Debian, Ubuntu
1. **Large (> 200MB)**: Full-featured distributions

### Security Considerations

1. **Most Secure**: Distroless, Scratch
1. **High Security**: Alpine (regular updates)
1. **Standard Security**: Debian, Ubuntu LTS
1. **Development Only**: Latest/edge versions

### Compatibility Considerations

1. **glibc vs musl**: Alpine uses musl, which can cause issues with some binaries
1. **Package availability**: Ubuntu/Debian have more packages
1. **Shell differences**: Alpine uses ash, not bash

### Production Recommendations

**Web Applications:**

- Node.js: `node:18-alpine` or `node:18-slim`
- Python: `python:3.11-slim` or `python:3.11-alpine`
- Java: `eclipse-temurin:17-alpine` or `openjdk:17-slim`

**Databases:**

- PostgreSQL: `postgres:15-alpine`
- MySQL: `mysql:8.0`
- Redis: `redis:7-alpine`

**Web Servers:**

- Nginx: `nginx:alpine`
- Apache: `httpd:2.4-alpine`

**Minimal/Security-focused:**

- Static binaries: `scratch` or `gcr.io/distroless/static`
- Dynamic applications: `gcr.io/distroless/java` or similar

## Best Practices for Base Image Selection

1. **Use specific versions** instead of `latest`
1. **Prefer official images** from Docker Hub
1. **Choose Alpine for production** when compatibility allows
1. **Use slim variants** to reduce size while maintaining compatibility
1. **Consider Distroless** for maximum security
1. **Match your development environment** to production base images
1. **Regularly update** base images for security patches
1. **Test thoroughly** when switching between glibc and musl (Alpine)
