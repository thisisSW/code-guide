# React Web Application with Docker

Complete guide for building a React frontend application with TypeScript, Vite, and Docker multi-stage builds.

> **Prerequisites**: See [typescript.md](./typescript.md), [vite-react.md](./vite-react.md), and [docker.md](./docker.md) for individual technology details.

## Project Structure

```
project-root/
├── Dockerfile              # Production multi-stage build
├── Dockerfile.dev          # Development with hot reload
├── docker-compose.yaml     # Service orchestration
├── nginx.conf              # Nginx configuration
├── .dockerignore           # Files to exclude from build
├── .env                    # Environment variables
├── .env.example            # Environment template
├── index.html              # HTML entry point
├── package.json            # Dependencies and scripts
├── tsconfig.json           # TypeScript config
├── tsconfig.node.json      # TypeScript config for Vite
├── vite.config.ts          # Vite configuration
├── public/                 # Static assets
│   └── favicon.ico
└── src/
    ├── main.tsx            # Application entry
    ├── App/                # Root component
    │   ├── index.tsx
    │   └── App.css
    ├── types/              # TypeScript types
    │   ├── index.ts
    │   ├── project.ts
    │   └── api.ts
    ├── services/           # API layer
    │   ├── index.ts
    │   ├── api.ts
    │   └── projects.ts
    ├── styles/             # Global styles
    │   ├── global.css
    │   └── cssModules.d.ts
    └── [Feature]/          # Feature components
        ├── index.tsx
        └── Feature.module.css
```

## Configuration Files

### package.json

```json
{
  "name": "my-app",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "preview": "vite preview",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^7.1.3"
  },
  "devDependencies": {
    "@types/node": "^22.10.5",
    "@types/react": "^18.3.18",
    "@types/react-dom": "^18.3.5",
    "@typescript-eslint/eslint-plugin": "^8.20.0",
    "@typescript-eslint/parser": "^8.20.0",
    "@vitejs/plugin-react": "^4.3.4",
    "eslint": "^9.18.0",
    "eslint-plugin-react-hooks": "^5.1.0",
    "eslint-plugin-react-refresh": "^0.4.16",
    "typescript": "^5.7.3",
    "vite": "^6.0.11"
  }
}
```

### vite.config.ts

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    host: true,
    port: 5173,
    watch: {
      usePolling: true,
    },
  },
  css: {
    modules: {
      localsConvention: 'camelCase',
      generateScopedName: '[name]__[local]___[hash:base64:5]',
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
        },
      },
    },
  },
})
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### index.html

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

## Docker Configuration

### Dockerfile (Production)

Multi-stage build for optimized production image with Nginx.

```dockerfile
FROM node:alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
# Transpiles TypeScript to JavaScript
RUN npm run build

FROM nginx:alpine as release
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Key Points**:
- **Build stage**: Full Node.js with build tools
- **Release stage**: Minimal Nginx for serving static files
- `npm run build` runs TypeScript compiler then Vite build
- Copies only built artifacts to final image

### Dockerfile.dev (Development)

Development container with hot reload.

```dockerfile
FROM node:24-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
RUN chmod -R +x node_modules/.bin
COPY . .
EXPOSE 5173
# Runs Vite dev server with TS compiler (no transpilation)
CMD ["npm", "run", "dev", "--", "--host"]
```

**Key Points**:
- Single stage for simplicity
- `--host` exposes dev server to Docker network
- Volume mounts enable hot reload (see docker-compose)

### nginx.conf

```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 10240;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json;
}
```

**Key Points**:
- `try_files $uri $uri/ /index.html` enables SPA routing
- Aggressive caching for static assets
- Security headers for protection
- Gzip compression for performance

### docker-compose.yaml

```yaml
services:
  dev:
    build:
      context: .
      dockerfile: ./Dockerfile.dev
    environment:
      VITE_API_URL: ${VITE_API_URL:-http://localhost:8000}
    ports:
      - 5173:5173
    volumes:
      - ./src:/app/src
      - ./public:/app/public

  release:
    build:
      context: .
      dockerfile: ./Dockerfile
    environment:
      VITE_API_URL: ${VITE_API_URL:-http://localhost:8080}
    ports:
      - 3000:80
```

**Key Points**:
- Volume mounts for hot reload in development
- Environment variables with defaults
- Different port mappings for dev/prod

### .dockerignore

```
node_modules
dist
.git
.gitignore
*.md
.env
.env.*
.DS_Store
npm-debug.log*
coverage
.vscode
.idea
```

### .env.example

```bash
VITE_API_URL=http://localhost:8000
```

## Application Code

### src/main.tsx

```typescript
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './styles/global.css'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

### src/App/index.tsx

```typescript
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom'
import Navigation from '../Navigation'
import ProjectListPage from '../ProjectListPage'
import ProjectDetailPage from '../ProjectDetailPage'
import './App.css'

export default function App() {
  return (
    <BrowserRouter>
      <div className="app">
        <Navigation />
        <div className="content">
          <Routes>
            <Route path="/" element={<Navigate to="/projects" replace />} />
            <Route path="/projects" element={<ProjectListPage />} />
            <Route path="/projects/:id" element={<ProjectDetailPage />} />
          </Routes>
        </div>
      </div>
    </BrowserRouter>
  )
}
```

### src/types/project.ts

```typescript
export interface Project {
  id: string
  name: string
  summary?: string
  status: ProjectStatus
  created_at: string
  updated_at: string
}

export enum ProjectStatus {
  Active = 'active',
  OnHold = 'on_hold',
  Archived = 'archived',
}

export interface CreateProjectRequest {
  name: string
  summary?: string
}

export interface UpdateProjectRequest {
  name: string
  summary?: string
  status?: ProjectStatus
}
```

### src/types/index.ts

```typescript
export * from './project'
export * from './api'
```

### src/services/api.ts

```typescript
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000'

export class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message)
    this.name = 'ApiError'
  }
}

export async function fetchApi<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const url = `${API_URL}${endpoint}`

  try {
    const response = await fetch(url, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers,
      },
      credentials: 'include',
    })

    if (!response.ok) {
      throw new ApiError(response.status, `API Error: ${response.statusText}`)
    }

    return await response.json()
  } catch (error) {
    if (error instanceof ApiError) {
      throw error
    }
    console.error('API Error:', error)
    throw new Error('Network error occurred')
  }
}
```

### src/services/projects.ts

```typescript
import { fetchApi } from '@/services/api'
import type { Project, CreateProjectRequest, UpdateProjectRequest } from '@/types'

export const projectsService = {
  list: async (): Promise<Project[]> => {
    return fetchApi<Project[]>('/projects')
  },

  get: async (id: string): Promise<Project> => {
    return fetchApi<Project>(`/projects/${id}`)
  },

  create: async (data: CreateProjectRequest): Promise<Project> => {
    return fetchApi<Project>('/projects', {
      method: 'POST',
      body: JSON.stringify(data),
    })
  },

  update: async (id: string, data: UpdateProjectRequest): Promise<Project> => {
    return fetchApi<Project>(`/projects/${id}`, {
      method: 'PUT',
      body: JSON.stringify(data),
    })
  },

  delete: async (id: string): Promise<void> => {
    await fetchApi(`/projects/${id}`, { method: 'DELETE' })
  },
}
```

### src/styles/cssModules.d.ts

```typescript
declare module '*.module.css' {
  const classes: { [key: string]: string }
  export default classes
}
```

## Example Component

### src/ProjectDetailPage/index.tsx

```typescript
import { useState, useEffect } from 'react'
import { useParams, useNavigate } from 'react-router-dom'
import { projectsService } from '@/services/projects'
import type { Project } from '@/types'
import styles from './ProjectDetailPage.module.css'

export default function ProjectDetailPage() {
  const { id } = useParams<{ id: string }>()
  const navigate = useNavigate()
  const [project, setProject] = useState<Project | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    async function fetchProject() {
      if (!id) return

      try {
        setLoading(true)
        setError(null)
        const data = await projectsService.get(id)
        setProject(data)
      } catch (err) {
        setError('Failed to load project')
        console.error(err)
      } finally {
        setLoading(false)
      }
    }

    fetchProject()
  }, [id])

  const handleDelete = async () => {
    if (!id || !window.confirm('Delete this project?')) return

    try {
      await projectsService.delete(id)
      navigate('/projects')
    } catch (err) {
      setError('Failed to delete project')
    }
  }

  if (loading) return <div className={styles.loading}>Loading...</div>
  if (error) return <div className={styles.error}>{error}</div>
  if (!project) return <div className={styles.error}>Project not found</div>

  return (
    <div className={styles.container}>
      <h1 className={styles.title}>{project.name}</h1>
      <p className={styles.summary}>{project.summary}</p>
      <div className={styles.actions}>
        <button onClick={handleDelete} className={styles.deleteBtn}>
          Delete
        </button>
      </div>
    </div>
  )
}
```

### src/ProjectDetailPage/ProjectDetailPage.module.css

```css
.container {
  max-width: 800px;
  margin: 2rem auto;
  padding: 2rem;
}

.title {
  font-size: 2rem;
  margin-bottom: 1rem;
}

.summary {
  color: var(--text-secondary);
  margin-bottom: 2rem;
}

.loading,
.error {
  text-align: center;
  padding: 2rem;
}

.error {
  color: #d32f2f;
}

.actions {
  display: flex;
  gap: 1rem;
}

.deleteBtn {
  background-color: #d32f2f;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

.deleteBtn:hover {
  background-color: #b71c1c;
}
```

## Usage Commands

### Development

```bash
# Start development container with hot reload
docker-compose up dev

# Or with environment file
docker-compose --env-file .env.dev up dev

# View logs
docker-compose logs -f dev

# Stop
docker-compose down
```

### Production

```bash
# Build production image
docker build -f Dockerfile -t myapp:latest .

# Run production container
docker run -p 80:80 myapp:latest

# Or use docker-compose
docker-compose up release
```

### Local Development (without Docker)

```bash
# Install dependencies
npm install

# Start dev server
npm run dev

# Type check
npm run type-check

# Build for production
npm run build

# Preview production build
npm run preview
```

## Best Practices

### Docker

1. **Multi-stage builds** - Separate build and runtime stages
2. **Alpine base images** - Smaller image sizes
3. **.dockerignore** - Exclude unnecessary files
4. **Volume mounts** - Enable hot reload in development
5. **Environment defaults** - Use `${VAR:-default}` syntax

### TypeScript

6. **Strict mode** - Enable all strict checks
7. **Path aliases** - Use `@/` for clean imports
8. **Type exports** - Export types from barrel files
9. **Interface props** - Type all component props

### React

10. **Feature folders** - Organize by feature, not file type
11. **CSS Modules** - Scoped styles per component
12. **Service layer** - Isolate API calls
13. **Error states** - Handle loading, error, and empty states

### Vite

14. **Manual chunks** - Split vendor libraries
15. **Source maps** - Enable for debugging
16. **Polling** - Use `usePolling` in Docker
