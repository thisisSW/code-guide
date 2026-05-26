# React Web Application with Docker

This guide explains how to containerize a React application for both development and production, covering the architectural decisions behind multi-stage builds, development workflows, and deployment strategies.

> **Prerequisites**: See [vite-react.md](./vite-react.md) for React/Vite setup and [docker.md](./docker.md) for Docker fundamentals.

## The Two-Environment Challenge

Web applications need different things in development vs production:

**Development needs**:
- Fast feedback (hot module replacement)
- Source maps for debugging
- Full Node.js environment with dev dependencies
- Volume mounts for live code changes

**Production needs**:
- Minimal image size (faster deploys, smaller attack surface)
- Optimized, minified bundles
- No dev dependencies or source code
- Efficient static file serving

Docker solves this with separate Dockerfiles and multi-stage builds.

## Development Container Strategy

### Goals

1. **Instant feedback**: Code changes appear immediately in the browser
2. **Parity with local dev**: Same experience as running `npm run dev` locally
3. **Isolated dependencies**: Node modules live in the container, not your filesystem

### How Hot Reload Works in Docker

The Vite dev server watches for file changes. In Docker, we mount source directories as volumes:

```yaml
volumes:
  - ./src:/app/src      # Source code
  - ./public:/app/public # Static assets
```

When you edit a file locally:
1. The change is reflected in the container (via volume mount)
2. Vite detects the change
3. Vite sends an HMR update to the browser
4. The browser updates without a full reload

### Development Dockerfile Explained

```dockerfile
FROM node:20-alpine
WORKDIR /app

# Copy package files first (layer caching)
COPY package*.json ./
RUN npm install

# Copy source files
COPY . .

# Expose the dev server port
EXPOSE 5173

# Run dev server, accessible from outside container
CMD ["npm", "run", "dev", "--", "--host"]
```

**Key decisions**:

**`--host` flag**: By default, Vite binds to `localhost`, which is only accessible inside the container. `--host` binds to `0.0.0.0`, making it accessible from your browser.

**Layer ordering**: Package files are copied and installed before source code. Since dependencies change less frequently than source code, Docker can cache the `npm install` layer and skip it on subsequent builds.

**Alpine base**: The `alpine` variant is much smaller (~50MB vs ~350MB for full Node), reducing image size and pull times.

### File Watching in Docker

Some Docker setups (especially on Windows or macOS) don't support native filesystem events. Configure Vite to use polling:

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    watch: {
      usePolling: true,  // Works with any Docker setup
    },
  },
})
```

**Trade-off**: Polling uses more CPU than native watching, but ensures reliability across all platforms.

## Production Build Strategy

### The Multi-Stage Approach

Production builds use two stages:

1. **Build stage**: Full Node.js environment to compile the application
2. **Serve stage**: Minimal web server with only the compiled output

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Why two stages?**

| | Build Stage | Final Image |
|---|---|---|
| Node.js | Yes | No |
| npm/package.json | Yes | No |
| Source code | Yes | No |
| node_modules | Yes | No |
| Built assets | Yes | Yes |
| Image size | ~400MB | ~25MB |

The final image contains only what's needed to serve static files.

### npm ci vs npm install

```dockerfile
RUN npm ci  # Not npm install
```

**`npm ci`** (clean install):
- Removes existing node_modules
- Installs exactly what's in package-lock.json
- Fails if package.json and lock file are out of sync
- Faster and more reliable for CI/CD

**`npm install`**:
- May update package-lock.json
- Resolves versions according to semver ranges
- Can produce different results on different runs

For production builds, always use `npm ci` for reproducibility.

## Serving Strategy: Nginx vs Node.js

### Option 1: Nginx (Recommended for Static SPAs)

Nginx excels at serving static files:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA routing: serve index.html for all routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets aggressively
    location ~* \.(js|css|png|jpg|ico|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Enable compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
}
```

**Why `try_files $uri $uri/ /index.html`?**

Single Page Applications handle routing client-side. When a user navigates to `/users/123`:

1. Browser requests `/users/123` from the server
2. Server doesn't have a file at that path
3. Without SPA handling, server returns 404
4. With `try_files`, server returns `index.html`
5. React Router sees the URL and renders the correct component

**Caching strategy**:

Vite generates filenames with content hashes (e.g., `main.a1b2c3d4.js`). When content changes, the filename changes. This enables aggressive caching:

- Hashed files: Cache for 1 year (`immutable` means "never revalidate")
- `index.html`: Don't cache (it references the current hashed files)

### Option 2: Node.js Server (When You Need Runtime Logic)

Use a Node.js server when you need:
- Server-side runtime configuration
- API proxying
- Server-side rendering
- Custom middleware

```javascript
// server/index.js
import express from 'express';
import path from 'path';

const app = express();

// Serve static files
app.use(express.static(path.resolve('./dist')));

// Runtime configuration endpoint
app.get('/config', (req, res) => {
  res.json({
    apiUrl: process.env.API_URL,
    featureFlags: process.env.FEATURE_FLAGS,
  });
});

// SPA fallback
app.get('*', (req, res) => {
  res.sendFile(path.resolve('./dist/index.html'));
});

app.listen(process.env.PORT || 80);
```

**Trade-off**: Nginx serves static files ~2-3x faster than Node.js, but Node.js provides flexibility for runtime behavior.

## Environment Variables

### The Build-Time vs Runtime Problem

Vite embeds environment variables at build time:

```typescript
// This value is baked into the bundle during build
const apiUrl = import.meta.env.VITE_API_URL;
```

This means the same image can't be used with different API URLs—you'd need to rebuild for each environment.

### Solution 1: Build Per Environment

Build separate images for each environment:

```bash
# Staging
docker build --build-arg VITE_API_URL=https://staging-api.example.com -t myapp:staging .

# Production
docker build --build-arg VITE_API_URL=https://api.example.com -t myapp:production .
```

**Pros**: Simple, no runtime complexity
**Cons**: Multiple images, longer deploy pipelines

### Solution 2: Runtime Configuration

Serve configuration from an endpoint:

```javascript
// In your application entry point
async function loadConfig() {
  const response = await fetch('/config');
  const config = await response.json();
  window.__CONFIG__ = config;
}

await loadConfig();
// Then render your app
```

**Pros**: One image for all environments
**Cons**: Extra HTTP request on startup, more complex server setup

### Which to Choose?

**Build per environment** when:
- You have few environments
- CI/CD can easily build multiple images
- You want simpler runtime behavior

**Runtime configuration** when:
- You deploy to many environments
- You need to change config without rebuilding
- You're using Kubernetes with ConfigMaps

## Docker Compose for Development

### Complete Development Setup

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "5173:5173"
    volumes:
      - ./src:/app/src
      - ./public:/app/public
      - ./index.html:/app/index.html
    environment:
      - VITE_API_URL=http://localhost:8000
```

**Volume mounts explained**:

```yaml
volumes:
  - ./src:/app/src         # Mount source for hot reload
  - ./public:/app/public   # Mount static assets
  - ./index.html:/app/index.html  # Mount HTML template
  # Note: node_modules NOT mounted - stays in container
```

**Why not mount node_modules?**

1. **Platform differences**: Native modules compiled for macOS won't work in Linux container
2. **Performance**: node_modules contains thousands of files; syncing them slows everything down
3. **Isolation**: Each environment has its own dependencies

### Connecting to Backend Services

```yaml
services:
  frontend:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "5173:5173"
    environment:
      - VITE_API_URL=http://api:8000
    depends_on:
      - api
    networks:
      - app-network

  api:
    build: ../backend
    ports:
      - "8000:8000"
    networks:
      - app-network

networks:
  app-network:
```

**Why a shared network?**

Services on the same Docker network can reach each other by service name. `http://api:8000` resolves to the API container.

**Note**: `VITE_API_URL=http://api:8000` works for server-side code, but the browser can't resolve `api`. For browser requests, you might need:
- Use `localhost:8000` (since port is exposed)
- Set up a reverse proxy

## Best Practices Summary

### Development

1. **Use volume mounts for source code** - Enables hot reload without rebuilding
2. **Keep node_modules in the container** - Avoids platform compatibility issues
3. **Configure polling for file watching** - Ensures reliability across platforms
4. **Use `--host` flag** - Makes dev server accessible from outside container

### Production

5. **Multi-stage builds** - Separate build environment from runtime environment
6. **Use `npm ci`** - Reproducible installs from lock file
7. **Choose the right server** - Nginx for static files, Node.js for runtime logic
8. **Aggressive caching** - Leverage content hashing for long cache times

### Environment Variables

9. **Understand build-time vs runtime** - Vite embeds variables at build time
10. **Choose a strategy** - Build per environment or runtime configuration
11. **Never expose secrets** - `VITE_` variables are visible in browser

### Security

12. **Add security headers** - X-Frame-Options, X-Content-Type-Options
13. **Minimize image size** - Less code means smaller attack surface
14. **Keep base images updated** - Regularly update to get security patches
