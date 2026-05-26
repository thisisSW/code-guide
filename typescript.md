# TypeScript Fundamentals

TypeScript adds static type checking to JavaScript, catching errors at compile time rather than runtime. This guide explains when and why to use TypeScript's features, helping you make informed decisions about typing strategies.

> **Note**: This guide assumes familiarity with JavaScript fundamentals. See [javascript.md](./javascript.md) for async/await, array methods, fetch API, and other JavaScript patterns.

## Why TypeScript?

### The Problem with Dynamic Typing

JavaScript's flexibility is both a strength and a weakness. Consider this common scenario:

```javascript
function processUser(user) {
  return user.name.toUpperCase();
}

// This works fine
processUser({ name: 'alice' });

// This crashes at runtime - no name property
processUser({ email: 'alice@example.com' });

// This crashes at runtime - name is undefined
processUser({});
```

These bugs only surface when the code runs, often in production. TypeScript catches them immediately:

```typescript
interface User {
  name: string;
}

function processUser(user: User) {
  return user.name.toUpperCase();
}

// TypeScript error: Property 'name' is missing
processUser({ email: 'alice@example.com' });
```

### When TypeScript Adds Value

TypeScript provides the most benefit when:

1. **Working in teams** - Types serve as documentation and contracts between developers
2. **Building APIs** - Request/response types ensure frontend and backend stay synchronized
3. **Refactoring** - The compiler catches breaking changes across your codebase
4. **Complex data structures** - Types clarify nested objects and relationships
5. **Long-lived projects** - Types help future maintainers understand the code

TypeScript may add unnecessary overhead for:
- Quick prototypes or scripts
- Very small projects with one developer
- Projects where the team lacks TypeScript experience

## Type Annotations

### The Basics

Type annotations tell TypeScript what shape data should have. Place them after variable names with a colon:

```typescript
// Primitives
const name: string = 'Alice';
const age: number = 30;
const isActive: boolean = true;

// Arrays
const tags: string[] = ['typescript', 'javascript'];
const scores: number[] = [95, 87, 92];

// Objects (inline)
const user: { name: string; age: number } = {
  name: 'Alice',
  age: 30,
};
```

### When to Annotate vs. When to Infer

TypeScript can infer types from values. Use inference when types are obvious:

```typescript
// Let TypeScript infer - the type is obvious from the value
const name = 'Alice';        // TypeScript knows this is string
const count = 42;            // TypeScript knows this is number
const items = ['a', 'b'];    // TypeScript knows this is string[]

// Annotate when the type isn't clear from context
let data: User | null = null;  // Will be assigned later
const response: ApiResponse<User[]> = await fetchUsers();
```

**Rule of thumb**: Annotate function parameters and return types. Let TypeScript infer local variables.

### Function Types

Functions benefit most from explicit types because they define contracts:

```typescript
// Parameters and return type annotated
function calculateTotal(price: number, quantity: number): number {
  return price * quantity;
}

// Arrow function with types
const formatCurrency = (amount: number): string => {
  return `$${amount.toFixed(2)}`;
};

// void means the function returns nothing meaningful
function logMessage(message: string): void {
  console.log(message);
}

// Optional parameters use ?
function greet(name: string, greeting?: string): string {
  return `${greeting || 'Hello'}, ${name}`;
}

// Default parameters
function createUser(name: string, role: string = 'member'): User {
  return { name, role };
}
```

## Interfaces vs. Type Aliases

Both define object shapes, but they have different use cases.

### Interfaces: For Object Shapes You'll Extend

Use interfaces when defining objects that represent entities in your domain:

```typescript
// Interface for a domain entity
interface User {
  id: string;
  email: string;
  name: string;
}

// Interfaces can be extended
interface AdminUser extends User {
  permissions: string[];
}

// Interfaces can be merged (declaration merging)
// Useful for extending third-party types
interface Window {
  analytics: AnalyticsClient;
}
```

**Why interfaces for entities?** They clearly signal "this is a thing in our system" and can be extended as requirements evolve.

### Type Aliases: For Unions, Computed Types, and Primitives

Use type aliases for everything else:

```typescript
// Union types - a value that could be one of several types
type Status = 'pending' | 'active' | 'completed' | 'cancelled';
type Result = Success | Failure;
type ID = string | number;

// Computed types
type UserKeys = keyof User;  // 'id' | 'email' | 'name'
type ReadonlyUser = Readonly<User>;

// Function types
type Callback = (error: Error | null, data: string) => void;
type AsyncOperation<T> = () => Promise<T>;
```

**The practical difference**: Interfaces feel like defining "a thing," while types feel like defining "a shape or constraint."

## Modeling Your Domain

### Request and Response Types

When building applications that communicate with APIs, define separate types for:

1. **Domain models** - What the entity looks like in your system
2. **Request types** - What you send to the API
3. **Response types** - What the API returns (if different from domain model)

```typescript
// Domain model - the full entity
interface Article {
  id: string;
  title: string;
  content: string;
  author: User;
  publishedAt: Date;
  createdAt: Date;
  updatedAt: Date;
}

// Request type - only fields the client provides
interface CreateArticleRequest {
  title: string;
  content: string;
}

// Update request - all fields optional
interface UpdateArticleRequest {
  title?: string;
  content?: string;
}
```

**Why separate types?** The API contract is explicit. You can't accidentally send `id` when creating (the server generates it) or omit required fields.

### Enums for Fixed Sets of Values

Use enums when a value must be one of a known set:

```typescript
// String enum - preferred for readability and JSON serialization
enum OrderStatus {
  Pending = 'pending',
  Processing = 'processing',
  Shipped = 'shipped',
  Delivered = 'delivered',
  Cancelled = 'cancelled',
}

// Usage provides autocomplete and type safety
interface Order {
  id: string;
  status: OrderStatus;
}

function canCancel(order: Order): boolean {
  // TypeScript knows all possible values
  return order.status === OrderStatus.Pending
      || order.status === OrderStatus.Processing;
}
```

**Why string enums?** When serialized to JSON (for APIs), you get readable values like `"pending"` instead of `0`. They're also easier to debug.

**Alternative - union types**: For simpler cases, union types work well:

```typescript
type Status = 'pending' | 'active' | 'completed';
```

Use enums when you need to iterate over values or when the set is large and might change.

## Generics: Writing Reusable Code

Generics let you write code that works with multiple types while maintaining type safety.

### The Problem Generics Solve

Without generics, you'd need to write separate functions for each type or lose type information:

```typescript
// Without generics - loses type information
function firstElement(arr: unknown[]): unknown {
  return arr[0];
}

const first = firstElement([1, 2, 3]);
// first is 'unknown' - we lost the fact that it's a number
```

### Generic Functions

Generics preserve type information:

```typescript
// With generics - preserves type
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

const first = firstElement([1, 2, 3]);
// first is 'number | undefined' - TypeScript remembers the type

const firstString = firstElement(['a', 'b', 'c']);
// firstString is 'string | undefined'
```

The `<T>` declares a type parameter. Think of it as a placeholder that gets filled in when the function is called.

### Generic Interfaces

Useful for containers and wrappers:

```typescript
// Generic API response wrapper
interface ApiResponse<T> {
  data: T;
  status: number;
  message?: string;
}

// Usage with different types
type UserResponse = ApiResponse<User>;
type ArticleListResponse = ApiResponse<Article[]>;

// Generic repository interface
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(item: Omit<T, 'id'>): Promise<T>;
  update(id: string, item: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}
```

### Constraining Generics

Sometimes you need to restrict what types can be used:

```typescript
// T must have an 'id' property
interface HasId {
  id: string;
}

function findById<T extends HasId>(items: T[], id: string): T | undefined {
  return items.find(item => item.id === id);
}

// Works - User has id
findById(users, '123');

// TypeScript error - string[] doesn't have id property
findById(['a', 'b'], '123');
```

**When to use generics**: When you find yourself writing the same logic for different types, or when you need to preserve type information through a function.

## Utility Types: Transforming Types

TypeScript provides built-in utility types for common transformations.

### Partial<T> - Make All Properties Optional

Use when you need to update some fields of an entity:

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

// All properties become optional
type UserUpdate = Partial<User>;
// Equivalent to: { id?: string; name?: string; email?: string; }

function updateUser(id: string, changes: Partial<User>): Promise<User> {
  // Can pass any subset of User properties
}

updateUser('123', { name: 'New Name' }); // Valid
updateUser('123', { email: 'new@example.com' }); // Valid
```

### Pick<T, Keys> - Select Specific Properties

Use when you only need certain fields:

```typescript
// Only id and name
type UserPreview = Pick<User, 'id' | 'name'>;
// Equivalent to: { id: string; name: string; }

// Useful for list views where you don't need all data
function renderUserList(users: UserPreview[]): void {
  users.forEach(user => console.log(user.name));
}
```

### Omit<T, Keys> - Exclude Specific Properties

Use when you need everything except certain fields:

```typescript
// Everything except id (server generates it)
type CreateUserInput = Omit<User, 'id'>;
// Equivalent to: { name: string; email: string; }

// Exclude sensitive fields
type PublicUser = Omit<User, 'password' | 'ssn'>;
```

### Record<Keys, Type> - Create Object Types

Use when you need an object with specific keys:

```typescript
// Map of status to count
type StatusCounts = Record<OrderStatus, number>;
// Equivalent to: { pending: number; processing: number; ... }

// Map of ID to entity
type UserMap = Record<string, User>;

// Useful for lookup tables
const statusLabels: Record<OrderStatus, string> = {
  [OrderStatus.Pending]: 'Waiting for processing',
  [OrderStatus.Processing]: 'Being prepared',
  // ... must include all statuses
};
```

## Type Guards: Runtime Type Checking

TypeScript's type system is compile-time only. Sometimes you need runtime checks.

### The typeof Guard

For primitive types:

```typescript
function formatValue(value: string | number): string {
  if (typeof value === 'string') {
    // TypeScript knows value is string here
    return value.toUpperCase();
  }
  // TypeScript knows value is number here
  return value.toFixed(2);
}
```

### The in Operator

For checking object properties:

```typescript
interface Dog {
  bark(): void;
}

interface Cat {
  meow(): void;
}

function makeSound(animal: Dog | Cat): void {
  if ('bark' in animal) {
    animal.bark(); // TypeScript knows it's Dog
  } else {
    animal.meow(); // TypeScript knows it's Cat
  }
}
```

### Custom Type Guards

For complex type checking, write a type predicate:

```typescript
interface SuccessResponse {
  data: unknown;
  error?: never;
}

interface ErrorResponse {
  data?: never;
  error: string;
}

type ApiResult = SuccessResponse | ErrorResponse;

// Type predicate - returns 'result is SuccessResponse'
function isSuccess(result: ApiResult): result is SuccessResponse {
  return 'data' in result && result.data !== undefined;
}

// Usage
const result = await fetchData();
if (isSuccess(result)) {
  // TypeScript knows result.data exists
  processData(result.data);
} else {
  // TypeScript knows result.error exists
  showError(result.error);
}
```

## Module Organization

### Barrel Exports

Group related types in a single import:

```typescript
// src/types/user.ts
export interface User { ... }
export interface CreateUserRequest { ... }
export type UserRole = 'admin' | 'member' | 'guest';

// src/types/index.ts (barrel file)
export * from './user';
export * from './article';
export * from './common';

// Usage - clean single import
import { User, Article, ApiResponse } from '@/types';
```

**Why barrel files?** They provide a stable public API. Internal file structure can change without affecting imports.

### Type-Only Imports

When importing only for type checking:

```typescript
// Removed at compile time - no runtime cost
import type { User } from '@/types';

// Mixed import
import { createUser, type User } from '@/services/user';
```

**Why type-only imports?** They make dependencies clearer and can improve build performance.

## Configuration Philosophy

### Recommended tsconfig.json Settings

```json
{
  "compilerOptions": {
    // Strict mode catches more bugs
    "strict": true,

    // Catch unused code
    "noUnusedLocals": true,
    "noUnusedParameters": true,

    // Ensure all code paths return
    "noImplicitReturns": true,

    // Catch fall-through in switch
    "noFallthroughCasesInSwitch": true
  }
}
```

**Why strict mode?** It enables all strict type checking options. While it requires more upfront effort, it catches significantly more bugs. New projects should always start with strict mode.

## Best Practices Summary

### When Defining Types

1. **Start with interfaces for domain entities** - They're extendable and clearly signal intent
2. **Use type aliases for unions and computed types** - They're more flexible for complex type expressions
3. **Separate request/response types from domain models** - API contracts should be explicit
4. **Prefer string enums over numeric** - Better debugging and JSON serialization

### When Writing Code

5. **Annotate function signatures, infer variables** - Functions are contracts; local variables are implementation details
6. **Use generics when logic repeats across types** - Don't duplicate code for different types
7. **Leverage utility types** - `Partial`, `Pick`, `Omit` solve common problems
8. **Write type guards for runtime checking** - TypeScript can't check types at runtime without help

### When Organizing Code

9. **Use barrel files for public APIs** - Hide internal structure, provide clean imports
10. **Use type-only imports where possible** - Clearer dependencies, better builds
11. **Enable strict mode from the start** - Retrofitting is harder than starting strict
