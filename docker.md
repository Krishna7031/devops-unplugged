# Docker - DevOps Unplugged Newsletter

Complete Docker code examples and resources from the "DevOps Unplugged" newsletter.

## 📚 Content Coverage

This section covers **Week 1, 2, 3, & 4** of the Docker series from the newsletter:
- **Week 1**: Docker fundamentals, what it is, and why it matters
- **Week 2**: Writing Dockerfiles and production-grade best practices
- **Week 3**: Docker Compose and multi-container applications
- **Week 4**: Docker registries, image tagging, and production deployment

For complete context and deep explanations, refer to the full articles on the newsletter.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Docker Basics](#docker-basics)
- [Dockerfile Examples](#dockerfile-examples)
- [Docker Compose](#docker-compose)
- [Docker Registries & Tagging](#docker-registries--tagging)
- [Image Tagging Strategies](#image-tagging-strategies)
- [Pushing & Pulling Images](#pushing--pulling-images)
- [CI/CD Integration](#cicd-integration)
- [Deployment Strategies](#deployment-strategies)
- [Quick Start](#quick-start)
- [Common Commands](#common-commands)
- [Best Practices Checklist](#best-practices-checklist)
- [Directory Structure](#directory-structure)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before getting started, you need:

- Docker installed ([Download here](https://www.docker.com/products/docker-desktop))
- Docker Compose installed (included with Docker Desktop)
- Basic understanding of Linux/command line
- A code editor (VS Code, etc.)

Verify installation:
```bash
docker --version
docker-compose --version
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
| **Compose** | Tool to define and run multi-container applications |
| **Volume** | Persistent storage for containers |
| **Network** | Communication layer between containers |

### The Complete Docker Lifecycle

```
Code → Dockerfile → docker build → Image → docker tag → docker push → Registry
          ↓
      Local Dev      Local Dev     Local Dev   Registry   Registry

Registry → docker pull → docker run → Container → Application
           Production   Production   Production  Running
```

---

## Dockerfile Examples

### Simple Python Application

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

### Node.js with Multi-Stage Build

**Dockerfile:**
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

### Production-Ready Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd -m -u 1000 appuser

# Copy and install dependencies
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY --chown=appuser:appuser . .

USER appuser

EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

ENTRYPOINT ["tini", "--"]
CMD ["python", "-u", "app.py"]
```

---

## Docker Compose

### Basic Multi-Container Setup

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  web:
    build: ./web
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://appuser:apppass@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./web:/app
      - /app/node_modules

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
      POSTGRES_DB: myapp
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:
```

**Run:**
```bash
docker-compose up -d
docker-compose logs -f
docker-compose down
```

---

### Advanced Multi-Stage Setup

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  frontend:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - backend
    networks:
      - frontend
    restart: unless-stopped

  backend:
    build: ./api
    environment:
      DATABASE_URL: postgres://db:5432/app
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
    networks:
      - frontend
      - backend
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    networks:
      - backend
    restart: unless-stopped

volumes:
  db_data:

networks:
  frontend:
  backend:
```

---

## Docker Registries & Tagging

### What is a Docker Registry?

A Docker registry is a server that stores and serves Docker images. It's where you push images for distribution and where production servers pull images for deployment.

### Registry Types

| Registry | Use Case | Examples |
|----------|----------|----------|
| **Public** | Open-source, shareable projects | Docker Hub, Quay.io, GitHub Container Registry |
| **Private** | Proprietary code, company projects | AWS ECR, Google Artifact Registry, Azure ACR |
| **Self-Hosted** | Complete control, on-premise | Docker Distribution, Harbor |

---

## Image Tagging Strategies

### ❌ BAD: Using `latest`

```bash
# DON'T DO THIS IN PRODUCTION
docker build -t myapp:latest .
docker push myapp:latest
docker run myapp:latest
```

**Problem:** `latest` is a moving target. You can't rollback to a specific version.

### ✅ GOOD: Semantic Versioning

**Semantic Version Format:** `MAJOR.MINOR.PATCH`

```bash
# Build with specific version
docker build -t myapp:1.2.3 .

# Tag with multiple versions for flexibility
docker tag myapp:1.2.3 myapp:1.2
docker tag myapp:1.2.3 myapp:1

# Tag for registry
docker tag myapp:1.2.3 registry.example.com/myapp:1.2.3
docker tag myapp:1.2.3 registry.example.com/myapp:1.2
docker tag myapp:1.2.3 registry.example.com/myapp:1
```

**Deployment Strategy:**
- `myapp:1.2.3` → Immutable, specific version
- `myapp:1.2` → Latest patch version
- `myapp:1` → Latest feature version (can include patches)
- `myapp:latest` → Development only (don't use in production)

### Advanced Tagging Examples

```bash
# Environment-specific tags
docker tag myapp:1.0.0 myapp:1.0.0-prod
docker tag myapp:1.0.0 myapp:1.0.0-staging

# Commit hash (unique code identifier)
docker tag myapp:1.0.0 myapp:1.0.0-abc123def

# Build metadata
docker tag myapp:1.0.0 myapp:1.0.0-build-42

# Date-based
docker tag myapp:1.0.0 myapp:1.0.0-2024-12-24

# For Docker Hub: username/repository:tag
docker tag myapp:1.0.0 krishna/myapp:1.0.0
```

---

## Pushing & Pulling Images

### Authenticate with Registry

```bash
# Docker Hub
docker login

# Private registry
docker login registry.example.com

# Using credentials (CI/CD)
docker login -u $USERNAME -p $PASSWORD registry.example.com

# Using access token (more secure)
docker login -u $USERNAME -p $TOKEN registry.example.com
```

Credentials are stored in `~/.docker/config.json`

### Tag Image for Registry

```bash
# For Docker Hub
docker tag myapp:1.0.0 krishna/myapp:1.0.0

# For private registry
docker tag myapp:1.0.0 registry.example.com/team/myapp:1.0.0
```

### Push Image

```bash
# To Docker Hub
docker push krishna/myapp:1.0.0

# To private registry
docker push registry.example.com/team/myapp:1.0.0

# Push all tags
docker push registry.example.com/team/myapp
```

### Pull Image

```bash
# From Docker Hub
docker pull krishna/myapp:1.0.0

# From private registry
docker pull registry.example.com/team/myapp:1.0.0

# Run pulled image
docker run registry.example.com/team/myapp:1.0.0
```

### Docker Compose with Registry

```yaml
version: '3.8'

services:
  app:
    image: registry.example.com/team/myapp:1.0.0
    ports:
      - "3000:3000"
```

---

## CI/CD Integration

### Complete Pipeline Workflow

```
1. Push code to GitHub/GitLab
        ↓
2. CI/CD detects change and builds image
        ↓
3. Run tests on built image
        ↓
4. Push image to registry (if tests pass)
        ↓
5. Deploy to production (pull and run)
```

### GitLab CI/CD Pipeline Example

**.gitlab-ci.yml:**
```yaml
stages:
  - build
  - test
  - push
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  image: docker:25
  services:
    - docker:25-dind
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker tag $DOCKER_IMAGE $CI_REGISTRY_IMAGE:latest

test:
  stage: test
  image: $DOCKER_IMAGE
  script:
    - npm test
    - npm run lint
    - npm run security-check

push:
  stage: push
  image: docker:25
  services:
    - docker:25-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $DOCKER_IMAGE
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

deploy:
  stage: deploy
  image: alpine
  script:
    - apk add --no-cache openssh-client
    - ssh -i $DEPLOY_KEY -o StrictHostKeyChecking=no ubuntu@prod.example.com
      "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD &&
       docker pull $DOCKER_IMAGE &&
       docker-compose up -d"
  only:
    - main
```

**Environment Variables (GitLab CI/CD Settings):**
```
CI_REGISTRY_USER = gitlab-ci-token
CI_REGISTRY_PASSWORD = (automatically set)
CI_REGISTRY = registry.gitlab.com
DOCKER_USERNAME = your_docker_hub_user
DOCKER_PASSWORD = your_docker_hub_token
DEPLOY_KEY = (SSH private key for production server)
```

### GitHub Actions Pipeline Example

**.github/workflows/docker.yml:**
```yaml
name: Docker Build & Deploy

on:
  push:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

---

## Deployment Strategies

### Rolling Deployment

Update containers one at a time without downtime.

**Process:**
1. Start new container with new image
2. Wait for health check to pass
3. Remove old container
4. Repeat for each container

**Pros:** Zero downtime, easy to monitor
**Cons:** Temporary 2x resource usage

**Implementation (Docker Compose):**
```bash
docker-compose up -d --no-deps --build web
```

### Blue-Green Deployment

Two complete environments, switch between them.

**Process:**
1. Blue (current) is live
2. Deploy to Green
3. Test Green thoroughly
4. Switch traffic to Green
5. Blue ready for rollback

**Pros:** Instant switch, instant rollback
**Cons:** 2x resources

**Implementation:**
```bash
# Deploy to green
docker-compose -f docker-compose.green.yml up -d

# Test green
curl http://green.example.com/health

# Switch traffic (DNS, load balancer, or proxy)
# Switch back if needed
docker-compose -f docker-compose.blue.yml up -d
```

### Canary Deployment

Route small percentage of traffic to new version.

**Process:**
1. Deploy new version
2. Route 5% traffic to new version
3. Monitor for issues
4. Gradually increase: 10%, 25%, 50%, 100%

**Pros:** Early issue detection, safe rollback
**Cons:** Slow rollout

**Implementation (with nginx):**
```nginx
upstream app_v1 {
    server old_container:3000 weight=95;
}

upstream app_v2 {
    server new_container:3000 weight=5;
}

server {
    listen 80;
    location / {
        proxy_pass http://app_v1;
    }
}
```

---

## Quick Start

### 1. Build Locally

```bash
docker build -t myapp:1.0.0 .
docker run myapp:1.0.0
```

### 2. Tag for Registry

```bash
docker tag myapp:1.0.0 registry.example.com/myapp:1.0.0
```

### 3. Authenticate

```bash
docker login registry.example.com
```

### 4. Push to Registry

```bash
docker push registry.example.com/myapp:1.0.0
```

### 5. Pull on Production

```bash
docker login registry.example.com
docker pull registry.example.com/myapp:1.0.0
docker run registry.example.com/myapp:1.0.0
```

---

## Common Commands

### Image Commands

```bash
# List images
docker images

# Tag image
docker tag source:tag target:tag

# Remove image
docker rmi image-name:tag

# View image history
docker history image-name:tag

# Inspect image
docker inspect image-name:tag
```

### Registry Commands

```bash
# Login to registry
docker login [registry-url]

# Logout from registry
docker logout [registry-url]

# Push image
docker push registry/image:tag

# Pull image
docker pull registry/image:tag

# Search Docker Hub
docker search image-name
```

### Compose Commands

```bash
# Start services
docker-compose up -d

# Stop services
docker-compose down

# View logs
docker-compose logs -f [service]

# List services
docker-compose ps

# Execute command in service
docker-compose exec [service] command

# Rebuild images
docker-compose up -d --build

# Scale service
docker-compose up -d --scale web=3
```

### Debugging Commands

```bash
# Check image layer details
docker history image-name:tag

# Check image configuration
docker inspect image-name:tag

# Check credentials stored locally
cat ~/.docker/config.json

# Check registry connectivity
docker pull registry/test-image

# Validate Docker files
docker build --dry-run -t test .
```

---

## Best Practices Checklist

### Image Tagging
- [ ] Never use `latest` in production
- [ ] Use semantic versioning (MAJOR.MINOR.PATCH)
- [ ] Use immutable version tags
- [ ] Tag with registry URL for clarity
- [ ] Document version increments

### Registry Management
- [ ] Use private registries for proprietary code
- [ ] Implement image scanning for vulnerabilities
- [ ] Set up image retention policies
- [ ] Use access tokens instead of passwords
- [ ] Audit who can push/pull images

### Deployment
- [ ] Build once, deploy everywhere
- [ ] Never build images on production servers
- [ ] Use health checks during deployment
- [ ] Implement rollback strategy
- [ ] Use immutable infrastructure (don't modify running containers)
- [ ] Tag production deployments with specific versions
- [ ] Document deployment procedures

### Security
- [ ] Use private registries for sensitive code
- [ ] Rotate access tokens regularly
- [ ] Implement image signing
- [ ] Scan images for vulnerabilities
- [ ] Use minimal base images (alpine, distroless)
- [ ] Run as non-root user
- [ ] Use secrets management (not environment variables for sensitive data)

### CI/CD Pipeline
- [ ] Automate build process
- [ ] Run tests in container before push
- [ ] Push only after successful tests
- [ ] Tag images with commit hash
- [ ] Maintain build artifacts for auditing
- [ ] Implement approval gates before production deployment

---

## Directory Structure

```
devops-unplugged/
├── README.md
├── Docker.md (this file)
│
├── 01-basics/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
│
├── 02-nodejs/
│   ├── Dockerfile
│   ├── package.json
│   └── index.js
│
├── 03-production-dockerfile/
│   ├── Dockerfile
│   ├── app/
│   └── .dockerignore
│
├── 04-compose-basic/
│   ├── docker-compose.yml
│   └── app/
│
├── 05-compose-advanced/
│   ├── docker-compose.yml
│   ├── frontend/
│   ├── backend/
│   └── database/
│
├── 06-cicd-gitlab/
│   ├── .gitlab-ci.yml
│   ├── Dockerfile
│   └── app/
│
├── 07-cicd-github/
│   ├── .github/workflows/docker.yml
│   ├── Dockerfile
│   └── app/
│
├── 08-deployment/
│   ├── docker-compose.prod.yml
│   ├── nginx.conf
│   └── scripts/
│
└── docs/
    ├── VERSIONING.md
    ├── DEPLOYMENT.md
    └── TROUBLESHOOTING.md
```

---

## Troubleshooting

### Image Issues

| Problem | Solution |
|---------|----------|
| `authentication required` when pushing | Run `docker login` with correct credentials |
| `image not found` when pulling | Check image name and tag, verify access to registry |
| `permission denied` pushing to registry | Verify credentials, check repository access |
| Image too large | Use multi-stage builds, remove cache, use slim base images |

### Registry Issues

| Problem | Solution |
|---------|----------|
| Can't connect to registry | Check network connectivity, verify registry URL |
| Credentials not working | Regenerate token, check `~/.docker/config.json` |
| Rate limit exceeded | Use private registry or wait, upgrade Docker Hub plan |
| Image push timeout | Split into smaller images, check network speed |

### Deployment Issues

| Problem | Solution |
|---------|----------|
| Container crashes after pull | Check logs, verify health checks, test locally first |
| Can't pull image in CI/CD | Verify registry credentials in CI/CD environment variables |
| Old version still running | Verify deployment updated image tag correctly |
| Rollback failed | Keep multiple version tags, maintain version history |

---

## Resources

- [Docker Official Documentation](https://docs.docker.com/)
- [Docker Registry Documentation](https://docs.docker.com/registry/)
- [Docker Hub](https://hub.docker.com/)
- [Semantic Versioning](https://semver.org/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Container Security](https://docs.docker.com/engine/security/)

---

## Related Articles

Read the full explanations in the DevOps Unplugged newsletter:
- **Week 1**: Docker fundamentals, architecture, and real-world use cases
- **Week 2**: Writing Dockerfiles, layer caching, and production best practices
- **Week 3**: Docker Compose, multi-container applications, and orchestration
- **Week 4**: Docker registries, image tagging, CI/CD integration, and deployment

---

## Contributing

Found an issue or want to improve examples? Contributions welcome!

---

## License

This content is part of the DevOps Unplugged newsletter.

---

**Last Updated:** December 2025
