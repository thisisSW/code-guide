# Vite + React + TypeScript Setup

This guide explains how to structure and configure a modern React application using Vite as the build tool. Rather than just showing configuration files, it explains the reasoning behind each decision.

> **Prerequisites**: See [typescript.md](./typescript.md) for TypeScript fundamentals and [react.md](./react.md) for React patterns.

## Why Vite?

### The Problem with Traditional Bundlers

Traditional bundlers like Webpack bundle your entire application before the dev server starts. As applications grow, this process slows dramatically—sometimes taking 30+ seconds to start.

Vite takes a different approach:
- **Native ES modules in development**: The browser loads modules directly without bundling
- **On-demand compilation**: Files are compiled only when requested
- **Fast Hot Module Replacement**: Changes appear nearly instantly

### When to Choose Vite

**Vite excels for:**
- New React/Vue/Svelte projects
- Projects prioritizing developer experience
- Teams comfortable with modern tooling

**Consider alternatives when:**
- You need extensive Webpack plugin ecosystem
- Your team has existing Webpack expertise and complex configurations
- You're working with legacy code that requires specific bundler behaviors

## Project Structure Philosophy

### Feature-Based Organization

Organize code by feature (what it does) rather than by type (what it is):

```
src/
├── features/
│   ├── users/              # Everything related to users
│   │   ├── UserList.tsx
│   │   ├── UserDetail.tsx
│   │   ├── UserForm.tsx
│   │   ├── userService.ts
│   │   ├── userTypes.ts
│   │   └── UserList.module.css
│   │
│   └── articles/           # Everything related to articles
│       ├── ArticleList.tsx
│       ├── ArticleEditor.tsx
│       ├── articleService.ts
│       └── articleTypes.ts
│
├── shared/                 # Truly shared code
│   ├── components/         # Reusable UI components
│   │   ├── Button/
│   │   ├── Modal/
│   │   └── Form/
│   ├── hooks/              # Shared custom hooks
│   ├── utils/              # Pure utility functions
│   └── types/              # Shared type definitions
│
├── services/               # API layer
│   ├── api.ts              # Base fetch configuration
│   └── index.ts
│
├── App.tsx                 # Root component
└── main.tsx                # Entry point
```

**Why feature-based?**

1. **Locality of behavior**: Everything related to a feature is in one place. When fixing a user bug, you're not jumping between `components/`, `services/`, `types/`, and `styles/` folders.

2. **Clear boundaries**: Features are self-contained. You can understand a feature by reading one directory.

3. **Easier deletion**: Removing a feature means deleting one folder, not hunting through multiple directories.

4. **Team scaling**: Different developers can work on different features without constant merge conflicts.

### The Shared Folder

Only put code in `shared/` when it's genuinely used by multiple features:

```
shared/
├── components/     # UI primitives used across features
│   ├── Button/     # Generic button component
│   ├── Modal/      # Generic modal wrapper
│   └── Input/      # Form input components
│
├── hooks/          # Hooks used by multiple features
│   ├── useDebounce.ts
│   └── useLocalStorage.ts
│
├── utils/          # Pure functions with no feature logic
│   ├── formatDate.ts
│   └── validateEmail.ts
│
└── types/          # Types used across features
    ├── api.ts      # Generic API types
    └── common.ts   # Shared domain types
```

**Rule of thumb**: Start code in a feature folder. Move to `shared/` only when a second feature needs it.

## Configuration Deep Dive

### vite.config.ts Explained

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

**Path aliases (`@`)**:
```typescript
// Without alias - fragile, changes if file moves
import { Button } from '../../../shared/components/Button'

// With alias - stable, doesn't break when moving files
import { Button } from '@/shared/components/Button'
```

Path aliases make imports cleaner and more maintainable. The `@` convention is widely recognized.

**Server configuration**:
- `host: true` - Exposes the server to the network, required for Docker containers
- `usePolling: true` - Enables file watching in Docker (some filesystems don't support native watching)

**CSS Modules**:
- `localsConvention: 'camelCase'` - Converts `my-class` to `styles.myClass` in JavaScript
- `generateScopedName` - Creates unique class names like `Button__primary___x7h2k` to prevent collisions

**Build optimization**:
- `manualChunks` - Separates vendor libraries into their own bundle. Since React/React DOM change rarely, browsers can cache them separately from your application code.

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",

    "jsx": "react-jsx",

    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

**Key decisions**:

- `moduleResolution: "bundler"` - Uses Vite's module resolution instead of Node.js rules
- `jsx: "react-jsx"` - Uses React 17+ automatic JSX runtime (no need to import React)
- `strict: true` - Enables all strict type checking
- `paths` - Must mirror Vite's aliases for TypeScript to understand imports

### Environment Variables

Vite uses `import.meta.env` for environment variables:

```bash
# .env
VITE_API_URL=http://localhost:8000
VITE_FEATURE_FLAG_NEW_UI=true
```

```typescript
// Access in code
const apiUrl = import.meta.env.VITE_API_URL;
const showNewUI = import.meta.env.VITE_FEATURE_FLAG_NEW_UI === 'true';
```

**Important constraints**:
- Only `VITE_` prefixed variables are exposed to the client
- Variables are embedded at build time, not runtime
- Never put secrets in `VITE_` variables—they're visible in the browser

**Type safety for environment variables**:

```typescript
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_FEATURE_FLAG_NEW_UI: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

## The Service Layer Pattern

### Why Isolate API Calls?

Putting fetch calls directly in components creates problems:

1. **Duplication**: Multiple components fetching the same data differently
2. **Inconsistency**: Each component handles errors differently
3. **Testing difficulty**: Can't mock API calls without mocking fetch globally
4. **Type safety**: No central place to define API contracts

### Service Layer Structure

```typescript
// services/api.ts - Base configuration
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000';

export class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message);
    this.name = 'ApiError';
  }
}

export async function fetchApi<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const response = await fetch(`${API_URL}${endpoint}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  });

  if (!response.ok) {
    throw new ApiError(response.status, `API Error: ${response.statusText}`);
  }

  return response.json();
}
```

```typescript
// features/users/userService.ts - Feature-specific API
import { fetchApi } from '@/services/api';
import type { User, CreateUserInput, UpdateUserInput } from './userTypes';

export const userService = {
  getAll: () => fetchApi<User[]>('/users'),

  getById: (id: string) => fetchApi<User>(`/users/${id}`),

  create: (data: CreateUserInput) =>
    fetchApi<User>('/users', {
      method: 'POST',
      body: JSON.stringify(data),
    }),

  update: (id: string, data: UpdateUserInput) =>
    fetchApi<User>(`/users/${id}`, {
      method: 'PUT',
      body: JSON.stringify(data),
    }),

  delete: (id: string) =>
    fetchApi<void>(`/users/${id}`, { method: 'DELETE' }),
};
```

**Benefits**:
- All API calls are typed
- Error handling is consistent
- Easy to mock for testing
- API changes affect one file, not every component

## CSS Modules

### Why CSS Modules?

Traditional CSS has a global namespace problem:

```css
/* styles.css */
.button { background: blue; }

/* another-styles.css */
.button { background: red; }  /* Conflicts! */
```

CSS Modules solve this by scoping class names to the component:

```css
/* Button.module.css */
.button {
  background: blue;
}
```

```typescript
// Button.tsx
import styles from './Button.module.css';

// styles.button becomes something like "Button_button__x7h2k"
export function Button() {
  return <button className={styles.button}>Click me</button>;
}
```

### Declaration File for TypeScript

TypeScript doesn't understand CSS imports by default:

```typescript
// src/styles/cssModules.d.ts
declare module '*.module.css' {
  const classes: { [key: string]: string };
  export default classes;
}
```

This tells TypeScript that CSS Module imports are objects with string values.

### Organizing Styles

Keep styles next to components:

```
features/users/
├── UserCard.tsx
├── UserCard.module.css    # Styles for UserCard only
├── UserList.tsx
└── UserList.module.css    # Styles for UserList only
```

For global styles (resets, variables, typography):

```
src/
├── styles/
│   ├── global.css         # Reset, base styles
│   ├── variables.css      # CSS custom properties
│   └── cssModules.d.ts    # TypeScript declaration
└── main.tsx               # Import global.css here
```

## Component Patterns

### Typing Component Props

```typescript
// Explicit interface for props
interface UserCardProps {
  user: User;
  onSelect?: (user: User) => void;
  highlighted?: boolean;
}

export function UserCard({ user, onSelect, highlighted = false }: UserCardProps) {
  return (
    <div
      className={highlighted ? styles.highlighted : styles.card}
      onClick={() => onSelect?.(user)}
    >
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}
```

**Why explicit interfaces?**
- Self-documenting: Props interface shows what the component accepts
- Better error messages: TypeScript errors reference the interface
- Refactoring support: IDE can rename props across usages

### Children Props

```typescript
interface CardProps {
  title: string;
  children: React.ReactNode;  // Any valid React child
}

export function Card({ title, children }: CardProps) {
  return (
    <div className={styles.card}>
      <h2>{title}</h2>
      <div className={styles.content}>{children}</div>
    </div>
  );
}
```

### Event Handler Types

```typescript
interface FormProps {
  onSubmit: (data: FormData) => void;
}

export function Form({ onSubmit }: FormProps) {
  const handleSubmit = (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    // ... gather form data
    onSubmit(data);
  };

  const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    // event.target.value is typed as string
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

## Data Fetching Patterns

### Basic Pattern with useState/useEffect

For simple cases:

```typescript
function UserDetail({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;

    async function fetchUser() {
      try {
        setLoading(true);
        setError(null);
        const data = await userService.getById(userId);
        if (!cancelled) {
          setUser(data);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err instanceof Error ? err.message : 'Failed to load');
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }

    fetchUser();

    return () => {
      cancelled = true;  // Prevent state updates if component unmounts
    };
  }, [userId]);

  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage message={error} />;
  if (!user) return <NotFound />;

  return <UserProfile user={user} />;
}
```

### When to Use React Query

For complex data fetching needs, consider TanStack Query (React Query):

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function UserDetail({ userId }: { userId: string }) {
  const queryClient = useQueryClient();

  // Fetch with caching, refetching, error handling
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['users', userId],
    queryFn: () => userService.getById(userId),
  });

  // Mutation with cache invalidation
  const updateMutation = useMutation({
    mutationFn: (data: UpdateUserInput) => userService.update(userId, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users', userId] });
    },
  });

  // ...
}
```

**Choose React Query when you need:**
- Automatic caching and background refetching
- Optimistic updates
- Request deduplication
- Pagination or infinite scroll
- Complex loading states across components

## Best Practices Summary

### Project Organization

1. **Feature-based structure** - Group related code together by what it does
2. **Shared folder is earned** - Code starts in features, moves to shared when genuinely reused
3. **Flat over nested** - Avoid deep nesting; 2-3 levels is usually enough

### Configuration

4. **Path aliases** - Use `@/` for cleaner, more stable imports
5. **Type environment variables** - Define `ImportMetaEnv` for type safety
6. **Separate vendor chunks** - Better caching for libraries that change rarely

### Code Patterns

7. **Service layer** - Isolate API calls from components
8. **CSS Modules** - Scoped styles prevent global conflicts
9. **Explicit prop interfaces** - Document component contracts
10. **Handle all states** - Loading, error, empty, and success states in data fetching
