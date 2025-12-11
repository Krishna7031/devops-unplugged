# Docker - DevOps Unplugged Newsletter

Complete Docker code examples and resources from the "DevOps Unplugged" newsletter.

## 📚 Content Coverage

This section covers **Week 1 & 2** of the Docker series from the newsletter:
- **Week 1**: Docker fundamentals, what it is, and why it matters
- **Week 2**: Writing Dockerfiles and production-grade best practices

For complete context and deep explanations, refer to the full articles on the newsletter.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Docker Basics](#docker-basics)
- [Dockerfile Examples](#dockerfile-examples)
  - [Simple Python Application](#simple-python-application)
  - [Node.js Application with Multi-Stage Build](#nodejs-application-with-multi-stage-build)
  - [Production-Ready Best Practices](#production-ready-best-practices)
- [Directory Structure](#directory-structure)
- [Quick Start](#quick-start)
- [Common Commands](#common-commands)
- [Best Practices Checklist](#best-practices-checklist)

---

## Prerequisites

Before getting started, you need:

- Docker installed ([Download here](https://www.docker.com/products/docker-desktop))
- Basic understanding of Linux/command line
- A code editor (VS Code, etc.)

Verify installation:
```bash
docker --version
docker run hello-world
```

---

## Docker Basics

### What is Docker?

Docker is a containerization platform that packages your application with all dependencies into a standardized unit called a container.

### Key Concepts

| Concept | Definition |
|---------|-----------|
| **Image** | Blueprint/template containing OS, dependencies, and code |
| **Container** | Running instance of an image |
| **Layer** | Snapshot of the filesystem created by each Dockerfile instruction |
| **Registry** | Centralized repository to store and share images (Docker Hub, ECR, etc.) |

### Image vs Container

```
Dockerfile → docker build → Image → docker run → Container
(Recipe)    (Compile)      (Template) (Execute)  (Running instance)
```

---

## Dockerfile Examples

### Simple Python Application

**Project Structure:**
```
python-app/
├── app.py
├── requirements.txt
└── Dockerfile
```

**app.py:**
```python
print("Hello from Docker!")
print("Application is running...")
```

**requirements.txt:**
```
Flask==2.3.0
requests==2.31.0
```

**Dockerfile:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Build & Run:**
```bash
docker build -t my-python-app:1.0 .
docker run my-python-app:1.0
```

---

### Node.js Application with Multi-Stage Build

**Project Structure:**
```
nodejs-app/
├── package.json
├── index.js
├── .dockerignore
└── Dockerfile
```

**package.json:**
```json
{
  "name": "nodejs-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

**index.js:**
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello from Docker!');
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

**.dockerignore:**
```
node_modules
.git
.gitignore
npm-debug.log
README.md
.DS_Store
.env
```

**Dockerfile (Multi-Stage):**
```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Runtime
FROM node:20-alpine
WORKDIR /app
RUN apk add --no-cache tini
RUN addgroup -g 1000 appgroup && \
    adduser -D -u 1000 -G appgroup appuser

COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --chown=appuser:appgroup . .

USER appuser
EXPOSE 3000

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "index.js"]
```

**Build & Run:**
```bash
docker build -t my-nodejs-app:1.0 .
docker run -p 3000:3000 my-nodejs-app:1.0
```

---

### Production-Ready Best Practices

**Complete Dockerfile with all best practices:**

```dockerfile
# Use specific base image version (not 'latest')
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd -m -u 1000 appuser

# Copy only requirements first (for better caching)
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Expose port (for documentation)
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Use tini as init process for proper signal handling
RUN pip install tini
ENTRYPOINT ["tini", "--"]

# Start application
CMD ["python", "-u", "app.py"]
```

**Key Best Practices in This Dockerfile:**

1. ✅ **Specific versions**: `python:3.11-slim` (not `latest`)
2. ✅ **Non-root user**: Created `appuser` for security
3. ✅ **Minimal layers**: Chained RUN commands with `&&`
4. ✅ **Layer caching**: Dependencies before code
5. ✅ **Clean up**: Removed apt cache with `rm -rf /var/lib/apt/lists/*`
6. ✅ **Health checks**: Added HEALTHCHECK instruction
7. ✅ **Signal handling**: Using tini as init process
8. ✅ **Unbuffered output**: `python -u` for immediate log flushing

---

## Directory Structure

Recommended GitHub repository structure for Docker projects:

```
devops-unplugged/
├── README.md
├── Docker.md (this file)
├── 01-basics/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── 02-nodejs/
│   ├── Dockerfile
│   ├── package.json
│   ├── index.js
│   └── .dockerignore
├── 03-multi-stage/
│   ├── Dockerfile
│   ├── src/
│   └── .dockerignore
├── 04-production-ready/
│   ├── Dockerfile
│   ├── app/
│   └── .dockerignore
└── scripts/
    ├── build.sh
    └── run.sh
```

---

## Quick Start

### 1. Clone or Download Examples

```bash
git clone <repository-url>
cd devops-unplugged/01-basics
```

### 2. Build Docker Image

```bash
docker build -t my-app:1.0 .
```

**Flags:**
- `-t`: Tag (name) for the image
- `.`: Use Dockerfile in current directory

### 3. Run Container

```bash
docker run my-app:1.0
```

### 4. Run with Port Mapping

```bash
docker run -p 3000:3000 my-app:1.0
```

- `-p <host-port>:<container-port>`: Map ports

### 5. Run in Background

```bash
docker run -d --name my-container my-app:1.0
```

- `-d`: Detached mode (run in background)
- `--name`: Assign a name to the container

---

## Common Commands

### Image Commands

```bash
# List all images
docker images

# Remove image
docker rmi image-name:tag

# View image history and layers
docker history image-name:tag

# Inspect image details
docker inspect image-name:tag

# Tag image
docker tag old-name:tag new-name:tag
```

### Container Commands

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# View container logs
docker logs container-name

# Follow logs (real-time)
docker logs -f container-name

# Execute command in running container
docker exec -it container-name bash

# Stop container
docker stop container-name

# Start stopped container
docker start container-name

# Remove container
docker rm container-name

# View resource usage
docker stats container-name
```

### Build Commands

```bash
# Build image
docker build -t image-name:tag .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t image-name:tag .

# Build without using cache
docker build --no-cache -t image-name:tag .

# View build details
docker build --progress=plain -t image-name:tag .
```

---

## Best Practices Checklist

Use this checklist when writing Dockerfiles:

### Base Image
- [ ] Use specific version, not `latest`
- [ ] Use lightweight variants (`-slim`, `-alpine`) when possible
- [ ] Use official images from Docker Hub

### Layers & Caching
- [ ] Order instructions from least to most frequently changed
- [ ] Copy `requirements.txt`/`package.json` before copying entire codebase
- [ ] Combine related RUN commands with `&&`
- [ ] Remove package manager cache (`rm -rf /var/lib/apt/lists/*`)

### Security
- [ ] Create and use non-root user
- [ ] Never bake secrets into image
- [ ] Minimize dependencies (smaller attack surface)
- [ ] Pin package versions

### Size & Performance
- [ ] Use multi-stage builds when applicable
- [ ] Use `.dockerignore` to exclude unnecessary files
- [ ] Keep images lean and focused
- [ ] Remove build tools not needed at runtime

### Observability
- [ ] Include HEALTHCHECK instruction
- [ ] Use proper logging (stdout/stderr)
- [ ] Expose necessary ports with EXPOSE
- [ ] Use meaningful CMD/ENTRYPOINT

### Signal Handling
- [ ] Handle SIGTERM for graceful shutdown
- [ ] Use init system (tini) if needed
- [ ] Use unbuffered output for immediate logging

---

## Layer Caching Principle

**Order matters for build speed:**

```dockerfile
# ❌ BAD: Invalidates cache frequently
FROM python:3.11-slim
COPY . .                          # Code changes often
RUN pip install -r requirements.txt  # Rebuilt every time code changes

# ✅ GOOD: Leverages cache effectively
FROM python:3.11-slim
COPY requirements.txt .           # Dependencies change rarely
RUN pip install -r requirements.txt  # Cached most of the time
COPY . .                          # Code changes often
```

**Result:**
- Bad approach: 2+ minute rebuild when code changes
- Good approach: 5-10 second rebuild when code changes

---

## Understanding Image Layers

View how Docker builds layers:

```bash
docker build -t my-app:1.0 .
docker history my-app:1.0
```

Output shows each instruction and its size. Use this to identify where optimization is needed.

---

## Troubleshooting

### Image too large?
- Use multi-stage builds
- Remove unnecessary dependencies
- Use `.dockerignore`
- Switch to `-alpine` base image

### Slow build times?
- Check layer caching (order instructions properly)
- Avoid unnecessary RUN commands
- Use `--no-cache` only when needed

### Container crashes on startup?
- Check logs: `docker logs container-name`
- Verify CMD/ENTRYPOINT syntax
- Ensure all dependencies are installed
- Check health: `docker inspect container-name`

### Permission issues?
- Verify USER instruction creates correct permissions
- Check file ownership with chown
- Ensure non-root user has access to required directories

---

## Resources

- [Docker Official Documentation](https://docs.docker.com/)
- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Hub Official Images](https://hub.docker.com/search?type=image&image_filter=official)

---

## Related Articles

Read the full explanations in the DevOps Unplugged newsletter:
- **Week 1**: Docker fundamentals, architecture, and real-world use cases
- **Week 2**: Writing Dockerfiles, layer caching, and production best practices
- **Week 3**: Docker Compose (coming soon)
- **Week 4**: Docker registries and CI/CD integration (coming soon)

---

## Contributing

Found an issue or want to improve examples? Contributions welcome!

---

## License

This content is part of the DevOps Unplugged newsletter.

---

**Last Updated:** December 2025
