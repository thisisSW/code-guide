# Vite + React + TypeScript Setup

Modern React application setup using Vite as the build tool and TypeScript for type safety.

> **Prerequisites**: See [typescript.md](./typescript.md) for TypeScript fundamentals and [react.md](./react.md) for React patterns.

## Project Structure

```
project-root/
├── index.html              # HTML entry point
├── package.json            # Dependencies and scripts
├── tsconfig.json           # TypeScript config
├── tsconfig.node.json      # TypeScript config for Vite
├── vite.config.ts          # Vite configuration
├── .env                    # Environment variables
├── .env.example            # Environment template
├── public/                 # Static assets (copied as-is)
│   └── favicon.ico
└── src/
    ├── main.tsx            # Application entry
    ├── App/                # Root component
    │   ├── index.tsx
    │   └── App.css
    ├── types/              # Shared TypeScript types
    │   ├── index.ts        # Barrel export
    │   ├── project.ts
    │   └── api.ts
    ├── services/           # API service layer
    │   ├── index.ts
    │   ├── api.ts          # Base fetch utilities
    │   └── projects.ts     # Project-specific API
    ├── styles/             # Global styles
    │   ├── global.css
    │   └── cssModules.d.ts # CSS Modules declaration
    └── [FeatureName]/      # Feature folders
        ├── index.tsx
        └── FeatureName.module.css
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

**Key Points**:
- `"type": "module"` enables ES modules
- `tsc && vite build` runs type check before building
- Separate `type-check` script for CI pipelines

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
    host: true,       // Expose to network (for Docker)
    port: 5173,
    watch: {
      usePolling: true,  // Required for Docker volume mounts
    },
  },
  css: {
    modules: {
      localsConvention: 'camelCase',  // Convert kebab-case to camelCase
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
          // Add other large libraries as separate chunks
        },
      },
    },
  },
})
```

**Key Points**:
- `@` alias maps to `./src` for clean imports
- `host: true` required for Docker container access
- `usePolling: true` required for file watching in Docker
- `manualChunks` splits vendor libraries for better caching

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    /* Path aliases */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### tsconfig.node.json

```json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts"]
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

## Application Entry

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

**Key Points**:
- `!` non-null assertion for `getElementById` result
- `StrictMode` enables additional development checks
- Global CSS imported at entry point

## Type Definitions

### src/types/project.ts

```typescript
export interface Project {
  id: string
  name: string
  summary?: string
  status: ProjectStatus
  created_at: string
  updated_at: string
  archived_at?: string
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

### src/types/api.ts

```typescript
export interface ApiError {
  error: string
}

export interface ApiResponse<T> {
  data?: T
  error?: string
}
```

### src/types/index.ts (Barrel Export)

```typescript
export * from './project'
export * from './api'
```

## CSS Modules

### src/styles/cssModules.d.ts

TypeScript declaration for CSS Modules.

```typescript
declare module '*.module.css' {
  const classes: { [key: string]: string }
  export default classes
}
```

### Component CSS Module

```css
/* src/ProjectCard/ProjectCard.module.css */
.container {
  padding: 1rem;
  border-radius: 8px;
  background: var(--surface);
}

.title {
  font-size: 1.25rem;
  font-weight: 600;
  margin-bottom: 0.5rem;
}

.summary {
  color: var(--text-secondary);
  font-size: 0.875rem;
}
```

### Using CSS Modules

```typescript
import styles from './ProjectCard.module.css'

interface ProjectCardProps {
  project: Project
}

export default function ProjectCard({ project }: ProjectCardProps) {
  return (
    <div className={styles.container}>
      <h3 className={styles.title}>{project.name}</h3>
      <p className={styles.summary}>{project.summary}</p>
    </div>
  )
}
```

## Service Layer

### src/services/api.ts

Base API utilities with error handling.

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
      throw new ApiError(
        response.status,
        `API Error: ${response.statusText}`
      )
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

export async function fetchApiNoResponse(
  endpoint: string,
  options?: RequestInit
): Promise<void> {
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
      throw new ApiError(
        response.status,
        `API Error: ${response.statusText}`
      )
    }
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

Feature-specific service.

```typescript
import { fetchApi, fetchApiNoResponse } from '@/services/api'
import type {
  Project,
  CreateProjectRequest,
  UpdateProjectRequest
} from '@/types'

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
    return fetchApiNoResponse(`/projects/${id}`, {
      method: 'DELETE',
    })
  },
}
```

## React Components with TypeScript

### Component with Props Interface

```typescript
import styles from './ProjectCard.module.css'
import type { Project } from '@/types'

interface ProjectCardProps {
  project: Project
  onClick?: (project: Project) => void
}

export default function ProjectCard({ project, onClick }: ProjectCardProps) {
  return (
    <div
      className={styles.container}
      onClick={() => onClick?.(project)}
    >
      <h3 className={styles.title}>{project.name}</h3>
      <p className={styles.summary}>{project.summary}</p>
    </div>
  )
}
```

### Component with State

```typescript
import { useState } from 'react'
import type { CreateProjectRequest } from '@/types'

interface CreateFormProps {
  onSubmit: (data: CreateProjectRequest) => Promise<void>
  onCancel: () => void
}

export default function CreateForm({ onSubmit, onCancel }: CreateFormProps) {
  const [formData, setFormData] = useState<CreateProjectRequest>({
    name: '',
    summary: '',
  })
  const [isSubmitting, setIsSubmitting] = useState(false)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setIsSubmitting(true)
    try {
      await onSubmit(formData)
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={formData.name}
        onChange={(e) => setFormData(prev => ({ ...prev, name: e.target.value }))}
        placeholder="Project name"
        required
      />
      <textarea
        value={formData.summary || ''}
        onChange={(e) => setFormData(prev => ({ ...prev, summary: e.target.value }))}
        placeholder="Summary"
      />
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create'}
      </button>
      <button type="button" onClick={onCancel}>
        Cancel
      </button>
    </form>
  )
}
```

### Component with Route Parameters

```typescript
import { useParams, useNavigate } from 'react-router-dom'
import { useState, useEffect } from 'react'
import { projectsService } from '@/services/projects'
import type { Project } from '@/types'

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

  if (loading) return <div>Loading...</div>
  if (error) return <div>{error}</div>
  if (!project) return <div>Project not found</div>

  return (
    <div>
      <h1>{project.name}</h1>
      <p>{project.summary}</p>
    </div>
  )
}
```

## React Query Integration

For more complex data fetching, use TanStack Query.

### Setup

```typescript
// src/App/index.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { BrowserRouter, Routes, Route } from 'react-router-dom'

const queryClient = new QueryClient()

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <Routes>
          {/* routes */}
        </Routes>
      </BrowserRouter>
    </QueryClientProvider>
  )
}
```

### Using Queries

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { useParams } from 'react-router-dom'
import { projectsService } from '@/services/projects'
import type { UpdateProjectRequest } from '@/types'

export default function ProjectDetailPage() {
  const { id } = useParams<{ id: string }>()
  const queryClient = useQueryClient()

  // Fetch project
  const { data: project, isLoading, error } = useQuery({
    queryKey: ['projects', id],
    queryFn: () => projectsService.get(id!),
    enabled: !!id,  // Only fetch when id exists
  })

  // Update mutation
  const updateMutation = useMutation({
    mutationFn: (data: UpdateProjectRequest) =>
      projectsService.update(id!, data),
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['projects', id] })
      queryClient.invalidateQueries({ queryKey: ['projects'] })
    },
  })

  if (isLoading) return <div>Loading...</div>
  if (error) return <div>Error loading project</div>
  if (!project) return <div>Project not found</div>

  return (
    <div>
      <h1>{project.name}</h1>
      <button
        onClick={() => updateMutation.mutate({
          name: project.name,
          status: 'archived'
        })}
        disabled={updateMutation.isPending}
      >
        Archive
      </button>
    </div>
  )
}
```

## Environment Variables

### .env

```bash
VITE_API_URL=http://localhost:8000
```

### .env.example

```bash
VITE_API_URL=http://localhost:8000
```

### Usage in Code

```typescript
// Access environment variables
const apiUrl = import.meta.env.VITE_API_URL

// Type-safe environment (add to src/vite-env.d.ts)
interface ImportMetaEnv {
  readonly VITE_API_URL: string
}
```

**Key Points**:
- Only `VITE_` prefixed variables are exposed to client
- Access via `import.meta.env.VITE_*`
- Never commit `.env` with secrets
- Use `.env.example` as template

## Best Practices

### Component Organization

1. **One component per file** - Export as default
2. **Co-locate styles** - CSS Module next to component
3. **Co-locate types** - Component-specific types in same file
4. **Shared types in /types** - Reusable types in dedicated folder

### TypeScript Patterns

1. **Use interfaces for props** - Clear contract for components
2. **Type event handlers** - `React.FormEvent`, `React.MouseEvent`
3. **Use generics sparingly** - Only when truly needed
4. **Prefer type imports** - `import type { X }` for types only

### File Naming

- **Components**: `index.tsx` in named folder (`ProjectCard/index.tsx`)
- **Services**: `featurename.ts` in services folder
- **Types**: `featurename.ts` in types folder
- **CSS Modules**: `ComponentName.module.css`

### Import Order

```typescript
// 1. React and third-party
import { useState, useEffect } from 'react'
import { useParams } from 'react-router-dom'
import { useQuery } from '@tanstack/react-query'

// 2. Internal services/utils
import { projectsService } from '@/services/projects'

// 3. Types
import type { Project } from '@/types'

// 4. Components
import ProjectCard from '../ProjectCard'

// 5. Styles
import styles from './ProjectList.module.css'
```
