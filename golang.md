# Go API Development

This guide covers patterns for building REST APIs in Go, focusing on architectural decisions and the reasoning behind common patterns. The examples use gorilla/mux for routing, but the principles apply to any router.

## Why Go for APIs?

### Go's Strengths

1. **Simple concurrency**: Goroutines and channels make concurrent operations straightforward
2. **Fast compilation**: Quick feedback during development
3. **Static binaries**: Deploy a single file with no runtime dependencies
4. **Strong standard library**: `net/http` is production-ready out of the box
5. **Explicit error handling**: Errors are values, not exceptions

### When Go Fits Well

- High-throughput APIs needing concurrent request handling
- Microservices that benefit from small container images
- Teams that value explicit code over magic
- Systems where deployment simplicity matters

## Project Structure Philosophy

### The pkg/ Layout

```
myapi/
├── main.go                 # Entry point - minimal, just wiring
├── go.mod
├── go.sum
└── pkg/
    ├── server/             # HTTP server setup
    ├── router/             # Route definitions
    ├── handlers/           # HTTP request handlers
    ├── middleware/         # HTTP middleware
    ├── models/             # Domain types and DTOs
    ├── repository/         # Data access layer
    ├── services/           # Business logic (optional)
    ├── database/           # Database connection
    └── errors/             # Error utilities
```

**Why this structure?**

1. **Separation of concerns**: Each package has one job
2. **Dependency direction**: Dependencies flow inward (handlers → repository → database)
3. **Testability**: Each layer can be tested independently with mocks
4. **Replaceability**: Swap PostgreSQL for MongoDB by changing only the repository layer

### The Layered Architecture

```
┌─────────────┐
│   Handlers  │  ← HTTP concerns: parse requests, format responses
├─────────────┤
│  Services   │  ← Business logic (optional for simple CRUD)
├─────────────┤
│ Repository  │  ← Data access: queries, transactions
├─────────────┤
│  Database   │  ← Connection management
└─────────────┘
```

**Handlers** translate between HTTP and your domain. They:
- Parse request bodies and URL parameters
- Validate input
- Call services or repositories
- Format responses

**Repositories** abstract data access. They:
- Execute database queries
- Map database rows to domain types
- Handle database-specific errors

**Services** contain business logic. They:
- Orchestrate multiple repository calls
- Enforce business rules
- Handle transactions

For simple CRUD operations, handlers can call repositories directly. Add services when business logic becomes complex.

## Dependency Injection Pattern

### The Problem

Hardcoded dependencies make code hard to test and modify:

```go
// Bad: Hardcoded dependency
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    db, _ := sql.Open("postgres", "...")  // Can't mock this
    // ...
}
```

### Constructor Injection

Pass dependencies through constructors:

```go
// Repository depends on database connection
type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

// Handler depends on repository
type UserHandler struct {
    repo *UserRepository
}

func NewUserHandler(repo *UserRepository) *UserHandler {
    return &UserHandler{repo: repo}
}
```

**Benefits**:
- Dependencies are explicit in the constructor signature
- Easy to mock for testing: pass a mock repository
- Clear initialization order in main.go

### Wiring in main.go

```go
func main() {
    // Initialize dependencies from bottom up
    db, err := database.New(config)
    if err != nil {
        log.Fatalf("database connection failed: %v", err)
    }
    defer db.Close()

    // Build the dependency tree
    userRepo := repository.NewUserRepository(db)
    userHandler := handlers.NewUserHandler(userRepo)

    // Wire up routes
    router := setupRoutes(userHandler)

    // Start server
    server := &http.Server{
        Addr:    ":8000",
        Handler: router,
    }
    log.Fatal(server.ListenAndServe())
}
```

## Handler Patterns

### Anatomy of a Handler

```go
func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    // 1. Parse and validate input
    var input CreateUserInput
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        respondError(w, http.StatusBadRequest, "invalid JSON")
        return
    }

    if input.Email == "" {
        respondError(w, http.StatusBadRequest, "email is required")
        return
    }

    // 2. Call business logic
    user, err := h.repo.Create(r.Context(), input)
    if err != nil {
        // 3. Handle domain errors appropriately
        if errors.Is(err, ErrEmailTaken) {
            respondError(w, http.StatusConflict, "email already registered")
            return
        }
        log.Printf("failed to create user: %v", err)
        respondError(w, http.StatusInternalServerError, "internal error")
        return
    }

    // 4. Format response
    respondJSON(w, http.StatusCreated, user)
}
```

**Key principles**:

1. **Validate early**: Check input before doing work
2. **Use context**: Pass `r.Context()` to enable request cancellation
3. **Distinguish error types**: Client errors (4xx) vs server errors (5xx)
4. **Log internally, respond generically**: Log details for debugging, send safe messages to clients

### Response Helpers

Consistent response formatting:

```go
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}
```

**Why helpers?**
- Consistent Content-Type header
- Consistent error format across all endpoints
- Less repetition in handlers

### HTTP Status Codes

Choose meaningful status codes:

| Status | When to Use |
|--------|-------------|
| 200 OK | Successful GET, PUT that returns data |
| 201 Created | Successful POST that creates a resource |
| 204 No Content | Successful DELETE or PUT that doesn't return data |
| 400 Bad Request | Malformed request (invalid JSON, missing field) |
| 401 Unauthorized | Missing or invalid authentication |
| 403 Forbidden | Authenticated but not authorized |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | Resource state conflict (duplicate email) |
| 500 Internal Server Error | Server-side failure |

## Repository Pattern

### Why Repositories?

Repositories provide an abstraction over data access:

```go
// Without repository - database concerns leak into handlers
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    var user User
    err := h.db.QueryRowContext(r.Context(),
        "SELECT id, name, email FROM users WHERE id = $1",
        id,
    ).Scan(&user.ID, &user.Name, &user.Email)
    // Handler knows SQL, table names, column mapping...
}

// With repository - handler only knows domain operations
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    user, err := h.repo.FindByID(r.Context(), id)
    // Handler doesn't know or care how data is stored
}
```

**Benefits**:
- **Testability**: Mock the repository interface, not the database
- **Flexibility**: Change database without changing handlers
- **Encapsulation**: SQL stays in one place

### Repository Structure

```go
type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) FindByID(ctx context.Context, id string) (*User, error) {
    query := `SELECT id, name, email, created_at FROM users WHERE id = $1`

    var user User
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.CreatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, ErrUserNotFound  // Domain error, not SQL error
    }
    if err != nil {
        return nil, fmt.Errorf("query failed: %w", err)
    }

    return &user, nil
}
```

### Domain vs Database Errors

Translate database errors to domain errors:

```go
// Domain errors - handlers understand these
var (
    ErrUserNotFound = errors.New("user not found")
    ErrEmailTaken   = errors.New("email already taken")
)

func (r *UserRepository) Create(ctx context.Context, input CreateUserInput) (*User, error) {
    // ...
    _, err := r.db.ExecContext(ctx, query, args...)

    if err != nil {
        // Check for unique constraint violation
        if isUniqueViolation(err) {
            return nil, ErrEmailTaken  // Domain error
        }
        return nil, fmt.Errorf("insert failed: %w", err)
    }
    // ...
}
```

**Why domain errors?**
- Handlers don't need to know about SQL error codes
- Error handling is consistent regardless of database
- Tests can check for domain errors without database

## Middleware

### What Middleware Does

Middleware wraps handlers to add cross-cutting concerns:

```
Request → Logger → CORS → Auth → Handler → Response
```

### The Middleware Signature

```go
// Middleware takes a handler and returns a wrapped handler
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Call the next handler
        next.ServeHTTP(w, r)

        // Log after handler completes
        log.Printf("%s %s %s", r.Method, r.URL.Path, time.Since(start))
    })
}
```

### Common Middleware

**Logging** - Record requests and timing:

```go
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Wrap ResponseWriter to capture status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}

        next.ServeHTTP(wrapped, r)

        log.Printf("%s %s %d %s",
            r.Method,
            r.URL.Path,
            wrapped.statusCode,
            time.Since(start),
        )
    })
}
```

**CORS** - Handle cross-origin requests:

```go
func CORS(allowedOrigins []string) func(http.Handler) http.Handler {
    originSet := make(map[string]bool)
    for _, o := range allowedOrigins {
        originSet[o] = true
    }

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")

            if originSet[origin] {
                w.Header().Set("Access-Control-Allow-Origin", origin)
                w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
                w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
            }

            if r.Method == http.MethodOptions {
                w.WriteHeader(http.StatusOK)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

### Applying Middleware

With gorilla/mux:

```go
func setupRouter(h *UserHandler) *mux.Router {
    r := mux.NewRouter()

    // Apply to all routes
    r.Use(Logger)
    r.Use(CORS(allowedOrigins))

    // Routes
    r.HandleFunc("/users", h.List).Methods("GET")
    r.HandleFunc("/users", h.Create).Methods("POST")

    return r
}
```

## Error Handling Philosophy

### Errors Are Values

Go treats errors as values to be handled, not exceptions to be caught:

```go
user, err := repo.FindByID(ctx, id)
if err != nil {
    // Handle the error explicitly
}
```

**Why this approach?**
- Error handling is visible in the code
- Can't accidentally ignore errors (linters catch it)
- Clear control flow - no hidden jumps

### Error Wrapping

Add context as errors bubble up:

```go
// Repository
func (r *UserRepository) FindByID(ctx context.Context, id string) (*User, error) {
    // ...
    if err != nil {
        return nil, fmt.Errorf("finding user %s: %w", id, err)
    }
}

// Handler
user, err := h.repo.FindByID(ctx, id)
if err != nil {
    // Error message: "finding user abc123: connection refused"
    log.Printf("GetUser failed: %v", err)
}
```

The `%w` verb wraps the error, preserving the original for `errors.Is()` checks.

### Error Checking

```go
// Check for specific error
if errors.Is(err, ErrUserNotFound) {
    respondError(w, http.StatusNotFound, "user not found")
    return
}

// Check error type
var validationErr *ValidationError
if errors.As(err, &validationErr) {
    respondError(w, http.StatusBadRequest, validationErr.Message)
    return
}
```

## Graceful Shutdown

### Why It Matters

Without graceful shutdown:
1. Server receives SIGTERM (container stopping)
2. Process exits immediately
3. In-flight requests fail

With graceful shutdown:
1. Server receives SIGTERM
2. Stop accepting new requests
3. Wait for in-flight requests to complete
4. Then exit

### Implementation

```go
func main() {
    server := &http.Server{
        Addr:    ":8000",
        Handler: router,
    }

    // Start server in goroutine
    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    log.Println("shutting down...")

    // Give in-flight requests time to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Printf("shutdown error: %v", err)
    }

    log.Println("server stopped")
}
```

## Best Practices Summary

### Architecture

1. **Layer your code** - Handlers → Services → Repositories → Database
2. **Inject dependencies** - Pass dependencies through constructors
3. **Define domain errors** - Translate database errors to domain errors

### Handlers

4. **Validate early** - Check input before doing work
5. **Use context** - Enable request cancellation
6. **Choose correct status codes** - 4xx for client errors, 5xx for server errors
7. **Log internally, respond safely** - Don't expose internal errors to clients

### Error Handling

8. **Handle every error** - Never ignore returned errors
9. **Wrap with context** - Use `%w` to add context while preserving the original
10. **Use sentinel errors** - Define errors like `ErrNotFound` for checking with `errors.Is()`

### Production Readiness

11. **Graceful shutdown** - Wait for in-flight requests before exiting
12. **Health endpoints** - Provide `/health` for load balancer checks
13. **Structured logging** - Include request IDs and relevant context
14. **Request timeouts** - Set `ReadTimeout` and `WriteTimeout` on the server
