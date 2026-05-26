# TypeScript Fundamentals

TypeScript extends JavaScript with static type checking. This guide covers essential TypeScript patterns for modern web development.

> **Note**: This guide assumes familiarity with JavaScript fundamentals. See [javascript.md](./javascript.md) for async/await, array methods, fetch API, and other JavaScript patterns.

## Core Concepts

### Type Annotations

Add types to variables, function parameters, and return values.

```typescript
// Variables
const name: string = 'Alice';
const age: number = 30;
const isActive: boolean = true;
const items: string[] = ['apple', 'banana'];

// Function parameters and return types
function greet(name: string): string {
  return `Hello, ${name}`;
}

// Arrow functions
const double = (n: number): number => n * 2;

// Optional parameters
function log(message: string, level?: string): void {
  console.log(level ? `[${level}] ${message}` : message);
}

// Default parameters
function createUser(name: string, role: string = 'user'): void {
  console.log(`${name} is a ${role}`);
}
```

**Key Points**:
- Type annotations come after the variable/parameter name with a colon
- `void` indicates a function returns nothing
- `?` makes parameters optional
- TypeScript infers types when possible (you can omit annotations for initialized variables)

### Interfaces

Define the shape of objects.

```typescript
// Basic interface
interface User {
  id: string;
  name: string;
  email: string;
}

// Optional properties
interface Project {
  id: string;
  name: string;
  summary?: string;  // Optional
}

// Readonly properties
interface Config {
  readonly apiUrl: string;
  readonly version: string;
}

// Nested interfaces
interface ProductBrief {
  id: string;
  project_id: string;
  problem_statement: string;
  stakeholder_goals: string;
  core_requirements: string;
  mvp_scope: string;
  created_at: string;
  updated_at: string;
}

// Function type in interface
interface ApiClient {
  get: (url: string) => Promise<Response>;
  post: (url: string, data: unknown) => Promise<Response>;
}
```

**Key Points**:
- Interfaces define contracts for object shapes
- Use `?` for optional properties
- Use `readonly` for immutable properties
- Interfaces can extend other interfaces with `extends`

### Type Aliases

Create custom type names.

```typescript
// Simple alias
type ID = string;

// Union types
type Status = 'pending' | 'active' | 'completed' | 'archived';
type Result = string | number;

// Object type alias
type Coordinates = {
  x: number;
  y: number;
};

// Function type alias
type Callback = (data: string) => void;

// Generic type alias
type ApiResponse<T> = {
  data?: T;
  error?: string;
};
```

**Key Points**:
- Use `type` for unions, primitives, and simple shapes
- Use `interface` for objects that may be extended
- Both can be used interchangeably for object shapes

### Request/Response Types

Define types for API interactions.

```typescript
// Request types
interface CreateBriefRequest {
  problem_statement: string;
  stakeholder_goals: string;
  core_requirements: string;
  mvp_scope: string;
}

interface UpdateBriefRequest {
  problem_statement: string;
  stakeholder_goals: string;
  core_requirements: string;
  mvp_scope: string;
}

// Response types
interface ApiError {
  error: string;
}

interface ApiResponse<T> {
  data?: T;
  error?: string;
}

// Usage in service functions
async function createBrief(
  projectId: string,
  data: CreateBriefRequest
): Promise<ProductBrief> {
  const response = await fetch(`/projects/${projectId}/brief`, {
    method: 'POST',
    body: JSON.stringify(data),
  });
  return await response.json();
}
```

### Enums

Define named constants.

```typescript
// String enum
enum WorkItemStatus {
  Backlog = 'backlog',
  Todo = 'todo',
  InProgress = 'in_progress',
  InReview = 'in_review',
  Done = 'done',
}

// Numeric enum
enum Priority {
  Low = 1,
  Medium = 2,
  High = 3,
  Critical = 4,
}

// Usage
interface WorkItem {
  id: string;
  title: string;
  status: WorkItemStatus;
  priority: Priority;
}

const item: WorkItem = {
  id: '123',
  title: 'Implement login',
  status: WorkItemStatus.Todo,
  priority: Priority.High,
};

// Check enum value
if (item.status === WorkItemStatus.Done) {
  console.log('Item completed');
}
```

**Key Points**:
- String enums are preferred for readability and JSON serialization
- Enums provide autocomplete and type safety
- Use `const enum` for inline values (smaller bundle)

### Generics

Create reusable, type-safe components.

```typescript
// Generic function
function identity<T>(value: T): T {
  return value;
}

const num = identity<number>(42);      // number
const str = identity<string>('hello'); // string
const auto = identity(true);           // boolean (inferred)

// Generic interface
interface Repository<T> {
  getById(id: string): Promise<T>;
  getAll(): Promise<T[]>;
  create(item: T): Promise<T>;
  update(id: string, item: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}

// Generic constraint
interface HasId {
  id: string;
}

function findById<T extends HasId>(items: T[], id: string): T | undefined {
  return items.find(item => item.id === id);
}

// Generic with multiple parameters
interface KeyValuePair<K, V> {
  key: K;
  value: V;
}
```

**Key Points**:
- `<T>` declares a type parameter
- `extends` constrains what types can be used
- TypeScript often infers generic types automatically

### Utility Types

Built-in types for common transformations.

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
}

// Partial - all properties optional
type UpdateUser = Partial<User>;
// { id?: string; name?: string; email?: string; password?: string; }

// Required - all properties required
type RequiredUser = Required<Partial<User>>;

// Pick - select specific properties
type UserPreview = Pick<User, 'id' | 'name'>;
// { id: string; name: string; }

// Omit - exclude specific properties
type PublicUser = Omit<User, 'password'>;
// { id: string; name: string; email: string; }

// Record - create object type with specific key/value types
type StatusMap = Record<string, boolean>;
// { [key: string]: boolean }

type UserById = Record<string, User>;
// { [userId: string]: User }

// Readonly - all properties readonly
type ImmutableUser = Readonly<User>;

// ReturnType - extract function return type
function getUser() {
  return { id: '1', name: 'Alice' };
}
type UserReturn = ReturnType<typeof getUser>;
// { id: string; name: string; }
```

### Type Guards

Narrow types at runtime.

```typescript
// typeof guard
function process(value: string | number): string {
  if (typeof value === 'string') {
    return value.toUpperCase(); // TypeScript knows it's string
  }
  return value.toFixed(2); // TypeScript knows it's number
}

// in operator guard
interface Dog {
  bark(): void;
}

interface Cat {
  meow(): void;
}

function speak(animal: Dog | Cat): void {
  if ('bark' in animal) {
    animal.bark(); // TypeScript knows it's Dog
  } else {
    animal.meow(); // TypeScript knows it's Cat
  }
}

// Custom type guard
interface ApiSuccess<T> {
  data: T;
  error?: never;
}

interface ApiError {
  data?: never;
  error: string;
}

type ApiResult<T> = ApiSuccess<T> | ApiError;

function isSuccess<T>(result: ApiResult<T>): result is ApiSuccess<T> {
  return result.data !== undefined;
}

// Usage
const result: ApiResult<User> = await fetchUser();
if (isSuccess(result)) {
  console.log(result.data.name); // TypeScript knows data exists
} else {
  console.error(result.error); // TypeScript knows error exists
}
```

## Module System

### Exporting

```typescript
// Named exports
export interface User {
  id: string;
  name: string;
}

export function createUser(name: string): User {
  return { id: crypto.randomUUID(), name };
}

export const DEFAULT_USER: User = { id: '0', name: 'Guest' };

// Default export
export default class UserService {
  getUser(id: string): Promise<User> {
    // ...
  }
}

// Re-export from another module
export { ProductBrief, CreateBriefRequest } from './brief';
export * from './api';
```

### Importing

```typescript
// Named imports
import { User, createUser } from './user';

// Default import
import UserService from './user';

// Combined
import UserService, { User, createUser } from './user';

// Rename import
import { User as UserType } from './user';

// Import all
import * as UserModule from './user';

// Type-only import (removed at compile time)
import type { User } from './user';
import { type User, createUser } from './user';
```

### Barrel Files

Re-export from a single entry point.

```typescript
// src/types/index.ts
export * from './project';
export * from './brief';
export * from './api';

// Usage
import { Project, ProductBrief, ApiResponse } from '@/types';
```

## Declaration Files

### CSS Modules Declaration

```typescript
// src/styles/cssModules.d.ts
declare module '*.module.css' {
  const classes: { [key: string]: string };
  export default classes;
}
```

### Environment Variables

```typescript
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_BUILD_VERSION: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### Ambient Declarations

```typescript
// global.d.ts
declare global {
  interface Window {
    analytics: {
      track: (event: string, data?: Record<string, unknown>) => void;
    };
  }
}

export {}; // Makes this a module
```

## Configuration

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

For Vite configuration file.

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

## Best Practices

1. **Enable strict mode** - Use `"strict": true` in tsconfig
2. **Prefer interfaces for objects** - Use `interface` for extendable shapes
3. **Use type for unions** - `type Status = 'active' | 'inactive'`
4. **Avoid `any`** - Use `unknown` when type is truly unknown
5. **Use type-only imports** - `import type { User }` for types
6. **Leverage inference** - Don't annotate obvious types
7. **Define types near usage** - Co-locate types with related code
8. **Export types from barrel files** - Centralize type exports
9. **Use utility types** - `Partial`, `Pick`, `Omit` for transformations
10. **Prefer string enums** - Better for debugging and serialization
11. **Type function returns** - Explicit return types catch errors early
12. **Use `readonly` for immutability** - Prevent accidental mutations
13. **Document with JSDoc** - Add descriptions to exported types
14. **Consistent naming** - `Interface` for interfaces, `Type` suffix optional
15. **Handle null/undefined explicitly** - Use optional chaining and nullish coalescing
