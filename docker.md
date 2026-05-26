# Docker Patterns for Applications

This guide covers Docker containerization patterns, focusing on the decisions and trade-offs involved in containerizing web applications for development and production.

## Why Docker?

### The Problems Docker Solves

**"Works on my machine"**: Different developers have different Node versions, different global packages, different OS configurations. Docker ensures everyone runs the same environment.

**Environment parity**: Development, staging, and production can run identical containers, eliminating environment-specific bugs.

**Dependency isolation**: Applications don't share dependencies or conflict with each other.

**Reproducible builds**: Given the same Dockerfile and source code, you get the same image every time.

### When Docker Adds Overhead

Docker isn't always the answer:
- Simple scripts that run occasionally
- Early prototypes where speed of iteration matters more than consistency
- Teams without Docker expertise (learning curve is real)
- Resource-constrained environments where container overhead matters

## Understanding Images and Containers

### The Relationship

An **image** is a blueprint—a read-only template containing your application and its dependencies.

A **container** is a running instance of an image—an isolated process with its own filesystem, network, and process space.

```
Dockerfile → (build) → Image → (run) → Container
```

You can run multiple containers from the same image, each with its own state.

### Layer Caching

Docker builds images in layers. Each instruction creates a layer:

```dockerfile
FROM node:20-alpine          # Layer 1: Base image
WORKDIR /app                 # Layer 2: Set working directory
COPY package*.json ./        # Layer 3: Copy package files
RUN npm install              # Layer 4: Install dependencies
COPY . .                     # Layer 5: Copy source code
RUN npm run build            # Layer 6: Build application
```

**Why order matters**: Docker caches layers. If a layer hasn't changed, Docker reuses the cached version. This is why we copy `package.json` before the source code—dependencies change less often than code.

**Bad order** (rebuilds npm install on every code change):
```dockerfile
COPY . .                     # Source changes invalidate this layer
RUN npm install              # Must reinstall every time
```

**Good order** (only rebuilds npm install when dependencies change):
```dockerfile
COPY package*.json ./        # Only changes when deps change
RUN npm install              # Cached unless deps changed
COPY . .                     # Source changes only affect this layer
```

## Multi-Stage Builds

### The Problem with Single-Stage

A single-stage build includes everything needed to build the application:

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "server.js"]
```

This image contains:
- Full Node.js installation (~300MB)
- npm and all tooling
- node_modules including dev dependencies
- Source code
- Built output

Result: ~800MB+ image with lots of unnecessary files.

### The Multi-Stage Solution

Separate building from running:

```dockerfile
# Stage 1: Build environment
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production environment
FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/package*.json ./
RUN npm ci --production
CMD ["node", "dist/server.js"]
```

**What happens**:
1. First stage builds the application
2. Second stage starts fresh with a minimal base
3. `COPY --from=build` copies only what's needed
4. First stage is discarded—not part of final image

**Result**: ~150MB image with only production essentials.

### Choosing Base Images

| Base Image | Size | Use Case |
|------------|------|----------|
| `node:20` | ~350MB | Building, full tooling needed |
| `node:20-slim` | ~200MB | Production with some tooling |
| `node:20-alpine` | ~50MB | Production, minimal footprint |
| `alpine` | ~5MB | Static files only (with nginx) |

**Alpine trade-offs**:
- Much smaller
- Uses musl instead of glibc (some native packages may not work)
- Smaller package ecosystem
- Good for most applications

## Development vs Production Patterns

### Development Priorities

1. **Fast feedback**: Code changes should reflect immediately
2. **Easy debugging**: Source maps, full stack traces
3. **Simple workflow**: Minimal commands to start working

### Production Priorities

1. **Small image size**: Faster deploys, less storage
2. **Security**: Minimal attack surface, no dev tools
3. **Performance**: Optimized builds, efficient serving
4. **Reliability**: Graceful handling of signals

### The Two-Dockerfile Approach

**Dockerfile.dev** - Optimized for development:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]
```

**Dockerfile** - Optimized for production:
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

Use Docker Compose to select which to use:
```yaml
services:
  dev:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - ./src:/app/src

  production:
    build:
      dockerfile: Dockerfile
```

## Volume Mounts for Development

### The Hot Reload Pattern

Without volumes, changing code requires rebuilding the image. With volumes, changes sync instantly:

```yaml
services:
  app:
    build:
      dockerfile: Dockerfile.dev
    volumes:
      - ./src:/app/src        # Mount source code
      - ./public:/app/public  # Mount static assets
```

**How it works**:
1. Container starts with code copied during build
2. Volume mount overlays local `./src` onto container's `/app/src`
3. File watcher in container sees local changes
4. Application reloads automatically

### What NOT to Mount

**Don't mount node_modules**:
```yaml
volumes:
  - ./:/app                    # Mounts everything including node_modules
  - /app/node_modules          # "Anonymous volume" to mask the mount
```

**Why?**
- node_modules may have platform-specific binaries (macOS vs Linux)
- Syncing thousands of small files is slow
- Dependencies should be installed inside the container

**Better approach**:
```yaml
volumes:
  - ./src:/app/src             # Only mount what needs hot reload
  - ./public:/app/public
```

## Docker Compose Patterns

### Service Definition

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "8000:8000"           # host:container
    environment:
      - DATABASE_URL=postgres://db:5432/myapp
    depends_on:
      - db
    volumes:
      - ./src:/app/src

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

### Environment Variables

**Three ways to provide them**:

1. **In docker-compose.yaml** (visible in version control):
```yaml
environment:
  - NODE_ENV=development
  - LOG_LEVEL=debug
```

2. **From shell environment**:
```yaml
environment:
  - DATABASE_URL  # Uses value from shell
```

3. **From .env file** (recommended for secrets):
```yaml
# docker-compose.yaml
environment:
  - DATABASE_URL=${DATABASE_URL}

# .env
DATABASE_URL=postgres://user:pass@host:5432/db
```

### YAML Anchors for DRY Configuration

```yaml
x-common-env: &common-env
  NODE_ENV: production
  LOG_LEVEL: info

services:
  api:
    environment:
      <<: *common-env
      API_PORT: 8000

  worker:
    environment:
      <<: *common-env
      WORKER_CONCURRENCY: 4
```

**How it works**:
- `&common-env` defines an anchor (a reusable block)
- `*common-env` references the anchor
- `<<:` merges the anchor's contents

### Networking

**Default behavior**: All services in a compose file can reach each other by service name.

```yaml
services:
  api:
    # Can connect to "db:5432"
  db:
    # Can be reached at "db"
```

**Custom networks** for isolation:
```yaml
services:
  frontend:
    networks:
      - frontend-net

  api:
    networks:
      - frontend-net  # Reachable by frontend
      - backend-net   # Reachable by db

  db:
    networks:
      - backend-net   # NOT reachable by frontend

networks:
  frontend-net:
  backend-net:
```

## .dockerignore

### Why It Matters

Without `.dockerignore`, `COPY . .` copies everything:
- node_modules (hundreds of MB, wrong platform)
- .git (potentially large history)
- Local environment files (.env with secrets)
- Build artifacts (dist/, coverage/)

### Recommended .dockerignore

```
# Dependencies (installed in container)
node_modules
vendor

# Version control
.git
.gitignore

# Environment files (may contain secrets)
.env
.env.*
!.env.example

# Build artifacts
dist
build
coverage
*.log

# IDE
.vscode
.idea
*.swp

# OS
.DS_Store
Thumbs.db
```

## Health Checks

### Container Health

Docker can monitor container health:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

**Parameters**:
- `--interval`: How often to check
- `--timeout`: How long to wait for response
- `--start-period`: Grace period for container startup
- `--retries`: Failures before marking unhealthy

### In Docker Compose

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
```

**Why health checks matter**:
- Load balancers route only to healthy instances
- Orchestrators restart unhealthy containers
- `depends_on` with `condition: service_healthy` waits for actual readiness

## Best Practices Summary

### Image Building

1. **Order instructions for caching** - Least-changing first
2. **Use multi-stage builds** - Separate build from runtime
3. **Choose appropriate base images** - Alpine for size, full images for compatibility
4. **Use .dockerignore** - Don't copy unnecessary files

### Development

5. **Mount source code** - Enable hot reload
6. **Don't mount node_modules** - Install inside container
7. **Use Dockerfile.dev** - Development-specific configuration

### Production

8. **Minimize image size** - Smaller = faster + more secure
9. **Don't run as root** - Use `USER` instruction
10. **Add health checks** - Enable proper orchestration
11. **Handle signals** - Graceful shutdown in your application

### Docker Compose

12. **Use .env files** - Keep secrets out of version control
13. **Named volumes for data** - Persist across container restarts
14. **YAML anchors** - Reduce duplication
15. **Explicit networks** - Control service visibility
