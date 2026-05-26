# Go API with PostgreSQL and Docker

This guide explains how to build and deploy a Go REST API with PostgreSQL, covering architectural decisions from project structure to production deployment.

> **Prerequisites**: See [golang.md](./golang.md) for Go patterns and [postgres.md](./postgres.md) for database patterns.

## Architecture Overview

### The Full Stack

```
┌─────────────────────────────────────────────────────────────┐
│                        Docker Compose                        │
│  ┌─────────────────┐           ┌─────────────────────────┐  │
│  │   PostgreSQL    │◄─────────►│         Go API          │  │
│  │                 │           │                         │  │
│  │  - Data storage │           │  - HTTP handlers        │  │
│  │  - Transactions │           │  - Business logic       │  │
│  │  - Migrations   │           │  - Data access          │  │
│  └─────────────────┘           └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Why these components together?**

- **Go**: Fast, compiles to a single binary, excellent concurrency
- **PostgreSQL**: Mature, reliable, rich feature set
- **Docker Compose**: Reproducible environments, easy local development

## Project Structure Decisions

### Organizing Go Code

```
myapi/
├── main.go                 # Minimal - just wiring
├── go.mod
└── pkg/
    ├── server/             # HTTP server lifecycle
    ├── router/             # Route definitions
    ├── handlers/           # Request/response handling
    ├── middleware/         # Cross-cutting concerns
    ├── models/             # Domain types
    ├── repository/         # Data access
    ├── database/           # Connection management
    └── errors/             # Error utilities
```

### Why pkg/?

Go has two conventions for non-main code:
- `internal/` - Cannot be imported by other projects
- `pkg/` - Can be imported by other projects

Use `pkg/` when your code might be useful as a library. Use `internal/` for truly internal code. For most APIs, either works; the structure matters more than the folder name.

### The Dependency Flow

```
main.go
    ↓
server (starts HTTP server)
    ↓
router (defines routes, wires handlers)
    ↓
handlers (HTTP in/out)
    ↓
repository (data access)
    ↓
database (connection pool)
```

**The rule**: Dependencies flow downward. A handler can use a repository, but a repository should never import a handler.

**Why this matters**:
- Clear mental model of what depends on what
- Easy to test each layer in isolation
- Changes in one layer don't ripple unexpectedly

## The Database Connection

### Connection Pool Configuration

Go's `database/sql` maintains a connection pool. Configure it for your workload:

```go
func NewDatabase(dsn string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }

    // Tune for your workload
    db.SetMaxOpenConns(25)         // Max connections to database
    db.SetMaxIdleConns(10)         // Connections to keep ready
    db.SetConnMaxLifetime(5 * time.Minute)  // Recycle connections

    // Verify connection works
    if err := db.Ping(); err != nil {
        return nil, err
    }

    return db, nil
}
```

**Sizing the pool**:

- **MaxOpenConns**: Start with 25, monitor for connection wait times
- **MaxIdleConns**: Usually 25-50% of MaxOpenConns
- **ConnMaxLifetime**: Shorter than PostgreSQL's `idle_in_transaction_session_timeout`

### Handling Container Startup

In Docker Compose, the API container might start before PostgreSQL is ready. Handle this with retries:

```go
func ConnectWithRetry(dsn string, maxAttempts int) (*sql.DB, error) {
    var db *sql.DB
    var err error

    for i := 0; i < maxAttempts; i++ {
        db, err = NewDatabase(dsn)
        if err == nil {
            return db, nil
        }

        log.Printf("Database connection attempt %d failed: %v", i+1, err)
        time.Sleep(time.Duration(i+1) * time.Second)
    }

    return nil, fmt.Errorf("failed after %d attempts: %w", maxAttempts, err)
}
```

**Why not use Docker's depends_on?**

`depends_on` waits for the container to start, not for the service to be ready. PostgreSQL might take several seconds after container start to accept connections.

## Docker Configuration

### Development: Hot Reload

For development, we want code changes to take effect immediately without rebuilding:

```dockerfile
# Dockerfile.dev
FROM golang:1.21-alpine
WORKDIR /app

# Install air for hot reload
RUN go install github.com/air-verse/air@latest

# Copy go.mod first for layer caching
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Air watches for changes and rebuilds
ENTRYPOINT ["air"]
```

**How Air works**:

1. Watches `.go` files for changes
2. Rebuilds the binary when changes detected
3. Restarts the running process

**Volume mounts enable the workflow**:

```yaml
volumes:
  - ./main.go:/app/main.go
  - ./pkg:/app/pkg
```

When you edit code locally, the change appears in the container, Air detects it, and your server restarts with the new code.

### Production: Multi-Stage Build

Production images should be minimal:

```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS build
RUN apk add --no-cache gcc musl-dev  # For CGO if needed
WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /api main.go

# Stage 2: Runtime
FROM alpine:latest
RUN apk --no-cache add ca-certificates  # For HTTPS
WORKDIR /app

COPY --from=build /api .

ENTRYPOINT ["./api"]
```

**Why two stages?**

| | Build Stage | Final Image |
|---|---|---|
| Go compiler | Yes | No |
| Source code | Yes | No |
| go.mod/sum | Yes | No |
| Test files | Yes | No |
| Binary | Generated | Copied |
| Image size | ~800MB | ~15MB |

**CGO considerations**:

- `CGO_ENABLED=0` builds a pure Go binary (no C dependencies)
- Some packages require CGO (e.g., some database drivers)
- If you need CGO, use `golang:alpine` with `apk add gcc musl-dev`

## Docker Compose Setup

### Service Configuration

```yaml
x-db-env: &db-env
  POSTGRES_HOST: postgres
  POSTGRES_PORT: 5432
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  POSTGRES_DB: ${POSTGRES_DB}

services:
  postgres:
    image: postgres:15-alpine
    environment:
      <<: *db-env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend

  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    environment:
      <<: *db-env
      ALLOWED_ORIGINS: ${ALLOWED_ORIGINS}
    depends_on:
      - postgres
    ports:
      - "8000:8000"
    volumes:
      - ./main.go:/app/main.go
      - ./pkg:/app/pkg
    networks:
      - backend

networks:
  backend:

volumes:
  postgres_data:
```

### YAML Anchors Explained

```yaml
x-db-env: &db-env    # Define anchor
  POSTGRES_HOST: postgres

environment:
  <<: *db-env        # Insert all values from anchor
  EXTRA_VAR: value   # Add more values
```

**Why anchors?**
- Define database credentials once
- Use them in both postgres and api services
- Change in one place updates everywhere

### Networks and Service Discovery

Services on the same Docker network can reach each other by name:

```yaml
networks:
  backend:   # Creates an isolated network

services:
  postgres:
    networks:
      - backend   # Connected to backend network

  api:
    networks:
      - backend   # Also connected to backend network
```

The API can connect to `postgres:5432` because Docker's internal DNS resolves `postgres` to the container's IP.

### Data Persistence

```yaml
volumes:
  postgres_data:   # Named volume for database files

services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

**Why named volumes?**

Without volumes, database data lives inside the container and is lost when the container is removed.

Named volumes:
- Persist across container restarts
- Are managed by Docker
- Can be backed up and migrated

## Environment Configuration

### Separation of Concerns

```bash
# .env file
POSTGRES_USER=myapp
POSTGRES_PASSWORD=secretpassword
POSTGRES_DB=myapp_development
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

**Never commit .env with real credentials.** Use `.env.example` as a template:

```bash
# .env.example
POSTGRES_USER=myapp
POSTGRES_PASSWORD=changeme
POSTGRES_DB=myapp_development
ALLOWED_ORIGINS=http://localhost:3000
```

### Environment in the Application

Read environment variables in Go:

```go
type Config struct {
    DatabaseURL    string
    AllowedOrigins []string
    Port           string
}

func LoadConfig() (*Config, error) {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8000"  // Default
    }

    origins := strings.Split(os.Getenv("ALLOWED_ORIGINS"), ",")

    return &Config{
        DatabaseURL:    buildDSN(),
        AllowedOrigins: origins,
        Port:           port,
    }, nil
}
```

**Pattern**: Provide sensible defaults, fail fast on missing required values.

## Database Migrations

### When to Run Migrations

**Option 1: Manually**
```bash
# Using golang-migrate CLI
migrate -path ./migrations -database "$DATABASE_URL" up
```

**Option 2: On application startup**

```go
func main() {
    db, _ := database.Connect()

    if err := runMigrations(db); err != nil {
        log.Fatalf("migration failed: %v", err)
    }

    // Continue with server setup
}
```

**Option 3: Separate init container (Kubernetes)**

```yaml
initContainers:
  - name: migrate
    image: myapp:latest
    command: ["./api", "migrate"]
```

**Recommendation**: Run migrations manually or via CI/CD for production. Automatic migration on startup can cause issues with multiple replicas racing to migrate.

### Migration File Organization

```
migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_create_articles.up.sql
├── 000002_create_articles.down.sql
└── ...
```

**Naming convention**:
- Sequential numbers ensure order
- Descriptive names explain purpose
- Matching up/down pairs enable rollback

## Health Checks

### Why Health Endpoints Matter

In containerized deployments:
- Load balancers check if instances are healthy
- Orchestrators restart unhealthy containers
- Deployment tools wait for health before routing traffic

### Implementation

```go
func HealthCheck(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Check database connectivity
        if err := db.PingContext(r.Context()); err != nil {
            w.WriteHeader(http.StatusServiceUnavailable)
            json.NewEncoder(w).Encode(map[string]string{
                "status": "unhealthy",
                "error":  "database unavailable",
            })
            return
        }

        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{
            "status": "healthy",
        })
    }
}
```

**What to check**:
- Database connection (`db.Ping()`)
- External service dependencies (if critical)
- Internal state (if applicable)

**What not to check**:
- Expensive operations that slow down health checks
- Non-critical external services

## Graceful Shutdown

### The Problem

When a container receives SIGTERM:
1. Without handling: Process exits immediately, dropping active requests
2. With graceful shutdown: Stop accepting new requests, finish active ones, then exit

### Implementation

```go
func main() {
    server := &http.Server{
        Addr:    ":8000",
        Handler: router,
    }

    // Start server in background
    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Wait for shutdown signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down...")

    // Give active requests time to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Printf("Forced shutdown: %v", err)
    }

    // Close database connection
    db.Close()

    log.Println("Server stopped")
}
```

### Shutdown Order

1. Stop accepting new connections
2. Wait for active requests to complete (with timeout)
3. Close database connections
4. Exit

## Local Development Workflow

### Starting the Stack

```bash
# Start all services
docker-compose up

# Or in detached mode
docker-compose up -d

# View logs
docker-compose logs -f api
```

### Common Operations

```bash
# Rebuild after changing Dockerfile
docker-compose build api

# Run database migrations
docker-compose exec api ./api migrate up

# Connect to PostgreSQL
docker-compose exec postgres psql -U myapp -d myapp_development

# Reset everything
docker-compose down -v  # -v removes volumes
```

### Debugging

```bash
# Shell into container
docker-compose exec api sh

# Check environment variables
docker-compose exec api env

# Test database connection
docker-compose exec api nc -zv postgres 5432
```

## Best Practices Summary

### Architecture

1. **Clear dependency direction** - Handlers → Repositories → Database
2. **Minimal main.go** - Just wiring, no business logic
3. **Pass dependencies explicitly** - No global state

### Docker

4. **Multi-stage builds** - Small production images
5. **Volume mounts for development** - Enable hot reload
6. **Named volumes for data** - Persist database across restarts
7. **Use networks** - Isolated service communication

### Database

8. **Retry connections on startup** - Handle container startup order
9. **Configure connection pool** - Don't use defaults in production
10. **Run migrations deliberately** - Not automatically in production

### Operations

11. **Health endpoints** - Enable load balancer and orchestrator integration
12. **Graceful shutdown** - Don't drop active requests
13. **Environment configuration** - Use .env files, never commit secrets
14. **Structured logging** - Include request IDs and context
