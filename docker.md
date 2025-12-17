# Docker - DevOps Unplugged Newsletter

Complete Docker code examples and resources from the "DevOps Unplugged" newsletter.

## 📚 Content Coverage

This section covers **Week 1, 2, & 3** of the Docker series from the newsletter:
- **Week 1**: Docker fundamentals, what it is, and why it matters
- **Week 2**: Writing Dockerfiles and production-grade best practices
- **Week 3**: Docker Compose and multi-container applications

For complete context and deep explanations, refer to the full articles on the newsletter.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Docker Basics](#docker-basics)
- [Dockerfile Examples](#dockerfile-examples)
  - [Simple Python Application](#simple-python-application)
  - [Node.js Application with Multi-Stage Build](#nodejs-application-with-multi-stage-build)
  - [Production-Ready Best Practices](#production-ready-best-practices)
- [Docker Compose](#docker-compose)
  - [Basic Multi-Container Setup](#basic-multi-container-setup)
  - [Web Application with Database](#web-application-with-database)
  - [Multi-Stage with Networking](#multi-stage-with-networking)
  - [Production-Grade Configuration](#production-grade-configuration)
- [Volumes & Storage](#volumes--storage)
- [Networking](#networking)
- [Environment Variables & Secrets](#environment-variables--secrets)
- [Quick Start](#quick-start)
- [Common Commands](#common-commands)
- [Best Practices Checklist](#best-practices-checklist)
- [Directory Structure](#directory-structure)

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

### Image vs Container vs Compose

```
Dockerfile → docker build → Image → docker run → Container
(Recipe)    (Compile)      (Template) (Execute)  (Running instance)

docker-compose.yml → docker-compose up → Multiple Containers + Networking + Volumes
(Application spec)  (Orchestrate)       (Complete system)
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
ENTRYPOINT ["tini", "--"]

# Start application
CMD ["python", "-u", "app.py"]
```

**Key Best Practices:**

1. ✅ **Specific versions**: `python:3.11-slim` (not `latest`)
2. ✅ **Non-root user**: Created `appuser` for security
3. ✅ **Minimal layers**: Chained RUN commands with `&&`
4. ✅ **Layer caching**: Dependencies before code
5. ✅ **Clean up**: Removed apt cache
6. ✅ **Health checks**: Added HEALTHCHECK instruction
7. ✅ **Signal handling**: Using tini as init process

---

## Docker Compose

### What is Docker Compose?

Docker Compose lets you define multi-container applications in a YAML file. One command brings your entire application stack to life.

### Basic Multi-Container Setup

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  web:
    image: node:20-alpine
    ports:
      - "3000:3000"
    command: npm start

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  db_data:
```

**Run:**
```bash
docker-compose up -d
```

---

### Web Application with Database

**Project Structure:**
```
app-stack/
├── docker-compose.yml
├── web/
│   ├── Dockerfile
│   ├── package.json
│   └── index.js
└── .env
```

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
      NODE_ENV: development
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./web:/app
      - /app/node_modules
    command: npm run dev

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

networks:
  default:
    name: app-network
```

**Run:**
```bash
docker-compose up -d
docker-compose logs -f web
```

---

### Multi-Stage with Networking

**docker-compose.yml with Frontend, Backend, Database, Cache:**
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
      DATABASE_URL: postgres://appuser:apppass@db:5432/app
      REDIS_URL: redis://cache:6379
      NODE_ENV: production
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - frontend
      - backend
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
      POSTGRES_DB: app
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
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
    driver: bridge
  backend:
    driver: bridge
```

**Run with scaling:**
```bash
docker-compose up -d
docker-compose up -d --scale backend=3
```

---

### Production-Grade Configuration

**Base configuration (docker-compose.yml):**
```yaml
version: '3.8'

services:
  app:
    image: myapp:1.0.0
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    environment:
      NODE_ENV: production
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  db_data:
```

**.env file:**
```
DB_USER=appuser
DB_PASSWORD=secure_password_here
DB_NAME=production_db
```

**docker-compose.prod.yml (overrides):**
```yaml
version: '3.8'

services:
  app:
    image: myapp:1.0.0
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      LOG_LEVEL: warn
```

**Run production:**
```bash
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Volumes & Storage

### Three Types of Mounts

#### 1. Named Volumes (Recommended for Databases)

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

**Commands:**
```bash
docker volume ls
docker volume inspect postgres_data
docker-compose down -v  # Remove volume
```

#### 2. Bind Mounts (For Development)

**Short syntax:**
```yaml
services:
  web:
    image: node:20-alpine
    volumes:
      - ./app:/app              # Read-write
      - ./config:/etc/config:ro # Read-only
```

**Long syntax:**
```yaml
services:
  web:
    image: node:20-alpine
    volumes:
      - type: bind
        source: ./app
        target: /app
        read_only: false
```

#### 3. Tmpfs (Temporary, In-Memory)

```yaml
services:
  cache:
    image: redis:7-alpine
    tmpfs:
      - /tmp
```

### Mount Options

```yaml
volumes:
  # Multiple mounts
  - ./src:/app
  - ./cache:/tmp/cache
  - ./config:/etc/configs
  
  # Read-only
  - ./config:/etc/config:ro
  
  # Named volume
  - my_volume:/data
```

---

## Networking

### Automatic Discovery

Services discover each other by name automatically:

```yaml
services:
  web:
    environment:
      DATABASE_URL: postgres://db:5432/app  # 'db' resolves automatically
      REDIS_URL: redis://cache:6379

  db:
    image: postgres:16

  cache:
    image: redis:7-alpine
```

### Custom Networks

```yaml
version: '3.8'

services:
  web:
    image: node:20-alpine
    networks:
      - frontend
    ports:
      - "3000:3000"

  db:
    image: postgres:16
    networks:
      - backend

  cache:
    image: redis:7-alpine
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

**Network types:**
- `bridge`: Isolated network (default)
- `host`: Use host's network directly
- `overlay`: For Docker Swarm

---

## Environment Variables & Secrets

### Method 1: Inline Environment Variables

```yaml
services:
  web:
    image: node:20-alpine
    environment:
      NODE_ENV: production
      LOG_LEVEL: info
      DATABASE_HOST: db
```

### Method 2: .env File

**.env:**
```
DB_USER=appuser
DB_PASSWORD=secure_pass
REDIS_HOST=cache
```

**docker-compose.yml:**
```yaml
services:
  web:
    image: node:20-alpine
    env_file: .env
```

### Method 3: Multiple Environment Files

```yaml
services:
  web:
    image: node:20-alpine
    env_file:
      - .env
      - .env.local
```

### Method 4: Docker Secrets (Production)

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**Create secret file:**
```bash
mkdir -p secrets
echo "your_secure_password" > secrets/db_password.txt
chmod 600 secrets/db_password.txt
```

---

## Quick Start

### 1. Clone Examples

```bash
git clone <repository-url>
cd devops-unplugged
```

### 2. Create a Simple Compose File

```bash
mkdir my-app && cd my-app
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
EOF
```

### 3. Start Services

```bash
docker-compose up -d
```

### 4. View Status

```bash
docker-compose ps
docker-compose logs -f
```

### 5. Stop Services

```bash
docker-compose down
```

---

## Common Commands

### Essential Commands

```bash
# Start services in background
docker-compose up -d

# Start services in foreground (see logs)
docker-compose up

# Rebuild images before starting
docker-compose up -d --build

# Stop services (containers saved)
docker-compose stop

# Start stopped services
docker-compose start

# Stop and remove containers, networks (volumes persist)
docker-compose down

# Stop and remove everything including volumes
docker-compose down -v

# Restart services
docker-compose restart
```

### Viewing Information

```bash
# List running services
docker-compose ps

# View logs
docker-compose logs

# Follow logs (real-time)
docker-compose logs -f

# View logs for specific service
docker-compose logs -f web

# Last 50 lines
docker-compose logs --tail=50 web

# View specific time period
docker-compose logs --since 10m web
```

### Container Operations

```bash
# Execute command in running container
docker-compose exec db psql -U postgres

# Interactive bash shell
docker-compose exec web bash

# Run one-off command
docker-compose run web npm test

# View running processes
docker-compose top web
```

### Building & Images

```bash
# Build services
docker-compose build

# Build specific service
docker-compose build web

# Build without cache
docker-compose build --no-cache

# Build with build arguments
docker-compose build --build-arg ENV=production
```

### Scaling & Management

```bash
# Scale a service
docker-compose up -d --scale web=3

# Remove stopped containers
docker-compose rm

# Remove unused images
docker-compose rmi

# Remove unused networks/volumes
docker-compose prune

# Validate compose file
docker-compose config

# Show image layers
docker-compose images
```

### Debugging

```bash
# Inspect service configuration
docker-compose config

# Show environment variables
docker-compose exec web env

# Check network connectivity
docker-compose exec web ping db

# View disk usage
docker system df

# View resource usage
docker stats
```

---

## Best Practices Checklist

### Dockerfile Best Practices
- [ ] Use specific version, not `latest`
- [ ] Use lightweight variants (`-slim`, `-alpine`)
- [ ] Order instructions from least to most frequently changed
- [ ] Copy dependencies before code
- [ ] Combine RUN commands with `&&`
- [ ] Create and use non-root user
- [ ] Never bake secrets into image
- [ ] Remove package manager cache
- [ ] Use multi-stage builds when applicable
- [ ] Include health checks
- [ ] Handle SIGTERM for graceful shutdown

### Docker Compose Best Practices
- [ ] Use specific image versions
- [ ] Define health checks for services
- [ ] Use `depends_on` with health conditions
- [ ] Mount volumes for persistent data
- [ ] Use named volumes for databases
- [ ] Use bind mounts for development only
- [ ] Use custom networks for isolation
- [ ] Set resource limits (CPU, memory)
- [ ] Use `.env` files for configuration
- [ ] Use Docker Secrets for production
- [ ] Add restart policies (`unless-stopped`)
- [ ] Organize services logically
- [ ] Use `.dockerignore` to exclude files
- [ ] Use override files for environments
- [ ] Document all services

### Production Checklist
- [ ] Resource limits set (CPU, memory)
- [ ] Health checks configured
- [ ] Restart policies set to `unless-stopped`
- [ ] Logging to stdout/stderr
- [ ] Environment-specific override files
- [ ] Secrets stored securely (not in code)
- [ ] Networks configured for isolation
- [ ] Volumes for persistent data
- [ ] Non-root users in images
- [ ] Monitoring/observability planned

---

## Directory Structure

Recommended repository structure for Docker projects:

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
│   ├── index.js
│   └── .dockerignore
│
├── 03-multi-stage/
│   ├── Dockerfile
│   ├── src/
│   └── .dockerignore
│
├── 04-production-ready/
│   ├── Dockerfile
│   ├── app/
│   └── .dockerignore
│
├── 05-compose-basic/
│   ├── docker-compose.yml
│   ├── app/
│   └── .env
│
├── 06-compose-webdb/
│   ├── docker-compose.yml
│   ├── docker-compose.override.yml
│   ├── web/
│   ├── db/
│   ├── .env
│   └── .env.example
│
├── 07-compose-production/
│   ├── docker-compose.yml
│   ├── docker-compose.prod.yml
│   ├── docker-compose.dev.yml
│   ├── backend/
│   ├── database/
│   ├── secrets/
│   └── .env.production
│
└── scripts/
    ├── build.sh
    ├── up.sh
    ├── down.sh
    └── logs.sh
```

---

## Layer Caching Principle

**Order matters for build speed:**

```dockerfile
# ❌ BAD: Invalidates cache frequently
FROM python:3.11-slim
COPY . .                          # Code changes often
RUN pip install -r requirements.txt  # Rebuilt every time

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

## Troubleshooting

### Image Problems

| Problem | Solution |
|---------|----------|
| Image too large | Use multi-stage builds, remove dependencies, use `-alpine` variant |
| Slow builds | Check layer caching, avoid unnecessary RUN commands |
| Permission denied | Check USER instruction, verify file ownership with chown |
| Dependencies missing | Verify all RUN install commands, check .dockerignore |

### Container Problems

| Problem | Solution |
|---------|----------|
| Container crashes | Check logs: `docker logs container-name` |
| Can't connect to service | Verify DNS names, check network connectivity |
| Port already in use | Change port mapping or stop conflicting container |
| Data not persisting | Use named volumes, not bind mounts for databases |

### Compose Problems

| Problem | Solution |
|---------|----------|
| Services can't communicate | Check network, verify service names, check DNS |
| Port conflicts | Check `docker-compose ps`, verify port mappings |
| Permissions issues | Check volume mounts, verify user permissions |
| Environment variables not set | Verify .env file, check env_file path, validate YAML |

---

## Resources

- [Docker Official Documentation](https://docs.docker.com/)
- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Hub Official Images](https://hub.docker.com/search?type=image&image_filter=official)

---

## Related Articles

Read the full explanations in the DevOps Unplugged newsletter:
- **Week 1**: Docker fundamentals, architecture, and real-world use cases
- **Week 2**: Writing Dockerfiles, layer caching, and production best practices
- **Week 3**: Docker Compose, multi-container applications, and orchestration
- **Week 4**: Docker registries and CI/CD integration (coming soon)

---

## Contributing

Found an issue or want to improve examples? Contributions welcome!

---

## License

This content is part of the DevOps Unplugged newsletter.

---

**Last Updated:** December 2025
