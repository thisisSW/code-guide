# Go API with PostgreSQL and Docker

Complete guide for building a REST API in Go using gorilla/mux, PostgreSQL database, and Docker containerization.

> **Prerequisites**: See [golang.md](./golang.md), [postgres.md](./postgres.md), and [docker.md](./docker.md) for individual technology details.

## Project Structure

```
project-root/
├── main.go                 # Application entry point
├── go.mod                  # Go module definition
├── go.sum                  # Dependency checksums
├── Dockerfile              # Production build
├── Dockerfile.dev          # Development with hot reload
├── docker-compose.yaml     # Service orchestration
├── .air.toml               # Air hot reload config
├── .env                    # Environment variables
├── .gitignore
├── migrations/             # Database migrations
│   ├── 000001_init_schema.up.sql
│   ├── 000001_init_schema.down.sql
│   ├── 000002_create_projects.up.sql
│   └── 000002_create_projects.down.sql
└── pkg/
    ├── database/           # Database connection
    │   └── database.go
    ├── errors/             # Error utilities
    │   └── errors.go
    ├── handlers/           # HTTP handlers
    │   ├── health.go
    │   ├── projects.go
    │   └── response.go
    ├── middleware/         # HTTP middleware
    │   ├── logger.go
    │   └── cors.go
    ├── models/             # Data models
    │   ├── project.go
    │   └── errors.go
    ├── repository/         # Data access layer
    │   └── project.go
    ├── router/             # Route configuration
    │   └── router.go
    └── server/             # HTTP server setup
        └── server.go
```

## Module Setup

### go.mod

```go
module github.com/username/myapi

go 1.21

require (
	github.com/gorilla/mux v1.8.1
	github.com/lib/pq v1.10.9
)
```

## Docker Configuration

### Dockerfile (Production)

Multi-stage build for minimal production image.

```dockerfile
FROM golang:alpine as build
RUN apk add --no-cache gcc musl-dev
WORKDIR /app
COPY main.go main.go
COPY pkg/ pkg/
COPY go.mod go.mod
COPY go.sum go.sum
RUN go mod download
RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=1 go build -o bin/api main.go

FROM alpine:latest as release
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=build /app/bin ./bin
ENTRYPOINT ./bin/api
```

**Key Points**:
- **Build stage**: Full Go toolchain with CGO support
- **Release stage**: Minimal Alpine with only binary
- `CGO_ENABLED=1` for PostgreSQL driver compatibility
- `ca-certificates` for HTTPS connections

### Dockerfile.dev (Development)

Development container with hot reload using Air.

```dockerfile
FROM golang:alpine
WORKDIR /app
ENV GO111MODULE=on
# Install Air for hot-reload
RUN go install github.com/air-verse/air@latest
COPY main.go .
COPY pkg/ pkg/
COPY go.sum go.sum
COPY go.mod go.mod
RUN go mod download
ENTRYPOINT air
```

### .air.toml

Air configuration for hot reload.

```toml
root = "."
testdata_dir = "testdata"
tmp_dir = "tmp"

[build]
  args_bin = []
  bin = "./tmp/main"
  cmd = "go build -o ./tmp/main ."
  delay = 1000
  exclude_dir = ["assets", "tmp", "vendor", "testdata"]
  exclude_file = []
  exclude_regex = ["_test.go"]
  exclude_unchanged = false
  follow_symlink = false
  full_bin = ""
  include_dir = []
  include_ext = ["go", "tpl", "tmpl", "html"]
  include_file = []
  kill_delay = "0s"
  log = "build-errors.log"
  poll = false
  poll_interval = 0
  rerun = false
  rerun_delay = 500
  send_interrupt = false
  stop_on_error = false

[color]
  app = ""
  build = "yellow"
  main = "magenta"
  runner = "green"
  watcher = "cyan"

[log]
  main_only = false
  time = false

[misc]
  clean_on_exit = false

[screen]
  clear_on_rebuild = false
  keep_scroll = true
```

### docker-compose.yaml

Complete orchestration with PostgreSQL.

```yaml
x-db-variables: &db-variables
  POSTGRES_HOST: postgres
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  POSTGRES_DB: ${POSTGRES_DB}
  POSTGRES_PORT: 5432

x-api-variables: &api-variables
  <<: *db-variables
  PORT: 8000
  ALLOWED_ORIGINS: ${ALLOWED_ORIGINS}

networks:
  local:

services:
  postgres:
    image: postgres:15.3-alpine
    restart: always
    environment:
      <<: *db-variables
    hostname: postgres
    networks:
      local:
        aliases:
          - postgres
    ports:
      - 5432:5432
    volumes:
      - ./postgres:/var/lib/postgresql/data

  dev:
    build:
      context: .
      dockerfile: ./Dockerfile.dev
    environment:
      <<: *api-variables
    depends_on:
      - postgres
    networks:
      local:
    ports:
      - 8000:8000
    volumes:
      - ./pkg:/app/pkg
      - ./main.go:/app/main.go

  release:
    build:
      context: .
      dockerfile: ./Dockerfile
    environment:
      <<: *api-variables
      PORT: 80
    depends_on:
      - postgres
    networks:
      local:
    ports:
      - 8080:80

volumes:
  postgres:
    driver: local
```

**Key Points**:
- YAML anchors reduce environment variable duplication
- `depends_on` ensures postgres starts before API
- Shared `local` network for inter-service communication
- Volume mounts for hot reload in development
- Persistent volume for PostgreSQL data

### .env

```bash
POSTGRES_USER=myapi_user
POSTGRES_PASSWORD=myapi_password
POSTGRES_DB=myapi_db
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:3000
PORT=8000
```

### .gitignore

```
# Binaries
bin/
tmp/

# Dependencies
vendor/

# IDE
.idea/
.vscode/

# Environment
.env
.env.local

# Database
postgres/

# Logs
*.log
```

## Application Code

### main.go

```go
package main

import (
	"log"
	"os"

	"github.com/username/myapi/pkg/database"
	"github.com/username/myapi/pkg/server"
)

func main() {
	// Get port from environment
	port := os.Getenv("PORT")
	if port == "" {
		port = "8000"
	}

	// Connect to database
	db, err := database.NewFromEnv()
	if err != nil {
		log.Fatalf("Failed to connect to database: %v", err)
	}
	defer db.Close()

	log.Println("Connected to database")

	// Create and start server
	srv, err := server.New(port, db.DB)
	if err != nil {
		log.Fatalf("Failed to create server: %v", err)
	}

	log.Printf("Starting server on port %s", port)
	if err := srv.Start(); err != nil {
		log.Fatalf("Server error: %v", err)
	}
}
```

### pkg/database/database.go

```go
package database

import (
	"database/sql"
	"fmt"
	"os"
	"time"

	"github.com/username/myapi/pkg/errors"
	_ "github.com/lib/pq"
)

type Config struct {
	Host     string
	Port     string
	User     string
	Password string
	DBName   string
}

type DB struct {
	*sql.DB
}

func New(cfg Config) (*DB, error) {
	connStr := fmt.Sprintf(
		"host=%s port=%s user=%s password=%s dbname=%s sslmode=disable",
		cfg.Host, cfg.Port, cfg.User, cfg.Password, cfg.DBName,
	)

	db, err := sql.Open("postgres", connStr)
	if err != nil {
		return nil, errors.Wrap(err, "failed to open database connection")
	}

	// Configure connection pool
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(25)
	db.SetConnMaxLifetime(5 * time.Minute)

	// Verify connection with retries
	maxRetries := 5
	var lastErr error
	for i := 0; i < maxRetries; i++ {
		if err := db.Ping(); err == nil {
			return &DB{db}, nil
		} else {
			lastErr = err
		}
		if i < maxRetries-1 {
			time.Sleep(2 * time.Second)
		}
	}

	return nil, errors.Wrap(lastErr, "failed to ping database after retries")
}

func NewFromEnv() (*DB, error) {
	cfg := Config{
		Host:     os.Getenv("POSTGRES_HOST"),
		Port:     os.Getenv("POSTGRES_PORT"),
		User:     os.Getenv("POSTGRES_USER"),
		Password: os.Getenv("POSTGRES_PASSWORD"),
		DBName:   os.Getenv("POSTGRES_DB"),
	}

	if cfg.Port == "" {
		cfg.Port = "5432"
	}

	if cfg.Host == "" || cfg.User == "" || cfg.Password == "" || cfg.DBName == "" {
		return nil, errors.New("missing required database configuration")
	}

	return New(cfg)
}

func (db *DB) Close() error {
	if db == nil || db.DB == nil {
		return nil
	}
	return db.DB.Close()
}
```

### pkg/server/server.go

```go
package server

import (
	"context"
	"database/sql"
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/username/myapi/pkg/errors"
	"github.com/username/myapi/pkg/router"
)

type Server struct {
	httpServer *http.Server
	port       string
	db         *sql.DB
}

func New(port string, db *sql.DB) (*Server, error) {
	if port == "" {
		return nil, errors.New("port is required")
	}
	if db == nil {
		return nil, errors.New("database connection is required")
	}

	r := router.New(db)

	srv := &Server{
		httpServer: &http.Server{
			Addr:         fmt.Sprintf(":%s", port),
			Handler:      r,
			ReadTimeout:  15 * time.Second,
			WriteTimeout: 15 * time.Second,
			IdleTimeout:  60 * time.Second,
		},
		port: port,
		db:   db,
	}

	return srv, nil
}

func (s *Server) Start() error {
	serverErrors := make(chan error, 1)

	go func() {
		serverErrors <- s.httpServer.ListenAndServe()
	}()

	shutdown := make(chan os.Signal, 1)
	signal.Notify(shutdown, os.Interrupt, syscall.SIGTERM)

	select {
	case err := <-serverErrors:
		return errors.Wrap(err, "server error")

	case sig := <-shutdown:
		fmt.Printf("\nReceived signal: %v. Starting graceful shutdown...\n", sig)

		ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
		defer cancel()

		if err := s.httpServer.Shutdown(ctx); err != nil {
			s.httpServer.Close()
			return errors.Wrap(err, "failed to gracefully shutdown server")
		}
	}

	return nil
}
```

### pkg/router/router.go

```go
package router

import (
	"database/sql"
	"net/http"

	"github.com/gorilla/mux"
	"github.com/username/myapi/pkg/handlers"
	"github.com/username/myapi/pkg/middleware"
	"github.com/username/myapi/pkg/repository"
)

func New(db *sql.DB) *mux.Router {
	r := mux.NewRouter()

	// Middleware
	r.Use(middleware.Logger)
	r.Use(middleware.CORS)

	// Health check
	r.HandleFunc("/health", handlers.HealthCheck).Methods(http.MethodGet)

	// Repositories
	projectRepo := repository.NewProjectRepository(db)

	// Handlers
	projectHandler := handlers.NewProjectHandler(projectRepo)

	// UUID pattern for path parameters
	uuid := "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"

	// Project routes
	r.HandleFunc("/projects", projectHandler.Create).Methods(http.MethodPost, http.MethodOptions)
	r.HandleFunc("/projects", projectHandler.List).Methods(http.MethodGet, http.MethodOptions)
	r.HandleFunc("/projects/{id:"+uuid+"}", projectHandler.Get).Methods(http.MethodGet, http.MethodOptions)
	r.HandleFunc("/projects/{id:"+uuid+"}", projectHandler.Update).Methods(http.MethodPut, http.MethodOptions)
	r.HandleFunc("/projects/{id:"+uuid+"}", projectHandler.Delete).Methods(http.MethodDelete, http.MethodOptions)

	return r
}
```

### pkg/middleware/cors.go

```go
package middleware

import (
	"net/http"
	"os"
	"strings"
)

func CORS(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		allowedOrigins := os.Getenv("ALLOWED_ORIGINS")
		origins := strings.Split(allowedOrigins, ",")

		origin := r.Header.Get("Origin")
		for _, allowed := range origins {
			if origin == strings.TrimSpace(allowed) {
				w.Header().Set("Access-Control-Allow-Origin", origin)
				break
			}
		}

		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
		w.Header().Set("Access-Control-Allow-Credentials", "true")

		if r.Method == http.MethodOptions {
			w.WriteHeader(http.StatusOK)
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

### pkg/middleware/logger.go

```go
package middleware

import (
	"log"
	"net/http"
	"time"
)

type responseWriter struct {
	http.ResponseWriter
	statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		rw := &responseWriter{
			ResponseWriter: w,
			statusCode:     http.StatusOK,
		}

		next.ServeHTTP(rw, r)

		log.Printf("%s %s %d %s", r.Method, r.RequestURI, rw.statusCode, time.Since(start))
	})
}
```

### pkg/errors/errors.go

```go
package errors

import (
	"errors"
	"fmt"
)

func New(message string) error {
	return errors.New(message)
}

func Wrap(err error, message string) error {
	if err == nil {
		return nil
	}
	return fmt.Errorf("%s: %w", message, err)
}

func Is(err, target error) bool {
	return errors.Is(err, target)
}
```

### pkg/models/project.go

```go
package models

import "time"

type Project struct {
	ID        string    `json:"id"`
	Name      string    `json:"name"`
	Summary   string    `json:"summary"`
	Status    string    `json:"status"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

type CreateProjectRequest struct {
	Name    string `json:"name"`
	Summary string `json:"summary"`
}

type UpdateProjectRequest struct {
	Name    string `json:"name"`
	Summary string `json:"summary"`
	Status  string `json:"status"`
}
```

### pkg/models/errors.go

```go
package models

import "errors"

var (
	ErrProjectNotFound = errors.New("project not found")
)
```

### pkg/repository/project.go

```go
package repository

import (
	"context"
	"database/sql"

	"github.com/username/myapi/pkg/errors"
	"github.com/username/myapi/pkg/models"
)

type ProjectRepository struct {
	db *sql.DB
}

func NewProjectRepository(db *sql.DB) *ProjectRepository {
	return &ProjectRepository{db: db}
}

func (r *ProjectRepository) Create(ctx context.Context, req *models.CreateProjectRequest) (*models.Project, error) {
	query := `
		INSERT INTO projects (name, summary)
		VALUES ($1, $2)
		RETURNING id, name, summary, status, created_at, updated_at
	`

	var project models.Project
	err := r.db.QueryRowContext(ctx, query, req.Name, req.Summary).Scan(
		&project.ID,
		&project.Name,
		&project.Summary,
		&project.Status,
		&project.CreatedAt,
		&project.UpdatedAt,
	)
	if err != nil {
		return nil, errors.Wrap(err, "failed to create project")
	}

	return &project, nil
}

func (r *ProjectRepository) GetByID(ctx context.Context, id string) (*models.Project, error) {
	query := `
		SELECT id, name, summary, status, created_at, updated_at
		FROM projects WHERE id = $1
	`

	var project models.Project
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&project.ID,
		&project.Name,
		&project.Summary,
		&project.Status,
		&project.CreatedAt,
		&project.UpdatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, models.ErrProjectNotFound
	}
	if err != nil {
		return nil, errors.Wrap(err, "failed to get project")
	}

	return &project, nil
}

func (r *ProjectRepository) List(ctx context.Context) ([]*models.Project, error) {
	query := `
		SELECT id, name, summary, status, created_at, updated_at
		FROM projects ORDER BY created_at DESC
	`

	rows, err := r.db.QueryContext(ctx, query)
	if err != nil {
		return nil, errors.Wrap(err, "failed to list projects")
	}
	defer rows.Close()

	var projects []*models.Project
	for rows.Next() {
		var project models.Project
		if err := rows.Scan(
			&project.ID,
			&project.Name,
			&project.Summary,
			&project.Status,
			&project.CreatedAt,
			&project.UpdatedAt,
		); err != nil {
			return nil, errors.Wrap(err, "failed to scan project")
		}
		projects = append(projects, &project)
	}

	if err := rows.Err(); err != nil {
		return nil, errors.Wrap(err, "error iterating projects")
	}

	return projects, nil
}

func (r *ProjectRepository) Update(ctx context.Context, id string, req *models.UpdateProjectRequest) (*models.Project, error) {
	query := `
		UPDATE projects SET name = $1, summary = $2, status = $3
		WHERE id = $4
		RETURNING id, name, summary, status, created_at, updated_at
	`

	var project models.Project
	err := r.db.QueryRowContext(ctx, query, req.Name, req.Summary, req.Status, id).Scan(
		&project.ID,
		&project.Name,
		&project.Summary,
		&project.Status,
		&project.CreatedAt,
		&project.UpdatedAt,
	)
	if err == sql.ErrNoRows {
		return nil, models.ErrProjectNotFound
	}
	if err != nil {
		return nil, errors.Wrap(err, "failed to update project")
	}

	return &project, nil
}

func (r *ProjectRepository) Delete(ctx context.Context, id string) error {
	query := `DELETE FROM projects WHERE id = $1`

	result, err := r.db.ExecContext(ctx, query, id)
	if err != nil {
		return errors.Wrap(err, "failed to delete project")
	}

	rowsAffected, _ := result.RowsAffected()
	if rowsAffected == 0 {
		return models.ErrProjectNotFound
	}

	return nil
}
```

### pkg/handlers/response.go

```go
package handlers

import (
	"encoding/json"
	"net/http"
)

func respondJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
	respondJSON(w, status, map[string]string{"error": message})
}
```

### pkg/handlers/health.go

```go
package handlers

import "net/http"

func HealthCheck(w http.ResponseWriter, r *http.Request) {
	respondJSON(w, http.StatusOK, map[string]string{"status": "healthy"})
}
```

### pkg/handlers/projects.go

```go
package handlers

import (
	"encoding/json"
	"log"
	"net/http"

	"github.com/gorilla/mux"
	"github.com/username/myapi/pkg/errors"
	"github.com/username/myapi/pkg/models"
	"github.com/username/myapi/pkg/repository"
)

type ProjectHandler struct {
	repo *repository.ProjectRepository
}

func NewProjectHandler(repo *repository.ProjectRepository) *ProjectHandler {
	return &ProjectHandler{repo: repo}
}

func (h *ProjectHandler) Create(w http.ResponseWriter, r *http.Request) {
	var req models.CreateProjectRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		respondError(w, http.StatusBadRequest, "invalid request body")
		return
	}

	if req.Name == "" {
		respondError(w, http.StatusBadRequest, "name is required")
		return
	}

	project, err := h.repo.Create(r.Context(), &req)
	if err != nil {
		log.Printf("Create: %v", err)
		respondError(w, http.StatusInternalServerError, "failed to create project")
		return
	}

	respondJSON(w, http.StatusCreated, project)
}

func (h *ProjectHandler) Get(w http.ResponseWriter, r *http.Request) {
	id := mux.Vars(r)["id"]

	project, err := h.repo.GetByID(r.Context(), id)
	if err != nil {
		if errors.Is(err, models.ErrProjectNotFound) {
			respondError(w, http.StatusNotFound, "project not found")
			return
		}
		log.Printf("Get: %v", err)
		respondError(w, http.StatusInternalServerError, "failed to get project")
		return
	}

	respondJSON(w, http.StatusOK, project)
}

func (h *ProjectHandler) List(w http.ResponseWriter, r *http.Request) {
	projects, err := h.repo.List(r.Context())
	if err != nil {
		log.Printf("List: %v", err)
		respondError(w, http.StatusInternalServerError, "failed to list projects")
		return
	}

	respondJSON(w, http.StatusOK, projects)
}

func (h *ProjectHandler) Update(w http.ResponseWriter, r *http.Request) {
	id := mux.Vars(r)["id"]

	var req models.UpdateProjectRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		respondError(w, http.StatusBadRequest, "invalid request body")
		return
	}

	project, err := h.repo.Update(r.Context(), id, &req)
	if err != nil {
		if errors.Is(err, models.ErrProjectNotFound) {
			respondError(w, http.StatusNotFound, "project not found")
			return
		}
		log.Printf("Update: %v", err)
		respondError(w, http.StatusInternalServerError, "failed to update project")
		return
	}

	respondJSON(w, http.StatusOK, project)
}

func (h *ProjectHandler) Delete(w http.ResponseWriter, r *http.Request) {
	id := mux.Vars(r)["id"]

	if err := h.repo.Delete(r.Context(), id); err != nil {
		if errors.Is(err, models.ErrProjectNotFound) {
			respondError(w, http.StatusNotFound, "project not found")
			return
		}
		log.Printf("Delete: %v", err)
		respondError(w, http.StatusInternalServerError, "failed to delete project")
		return
	}

	w.WriteHeader(http.StatusNoContent)
}
```

## Database Migrations

### migrations/000001_init_schema.up.sql

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### migrations/000001_init_schema.down.sql

```sql
-- Extensions typically not dropped
```

### migrations/000002_create_projects.up.sql

```sql
CREATE TABLE IF NOT EXISTS projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    summary TEXT,
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_projects_name ON projects(name);
CREATE INDEX idx_projects_status ON projects(status);

CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_projects_updated_at
    BEFORE UPDATE ON projects
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### migrations/000002_create_projects.down.sql

```sql
DROP TRIGGER IF EXISTS update_projects_updated_at ON projects;
DROP TABLE IF EXISTS projects;
```

## Usage Commands

### Development

```bash
# Start all services (postgres + api with hot reload)
docker-compose up dev

# View logs
docker-compose logs -f dev

# Rebuild after go.mod changes
docker-compose build dev && docker-compose up dev

# Stop all services
docker-compose down
```

### Production

```bash
# Build production image
docker build -f Dockerfile -t myapi:latest .

# Run with docker-compose
docker-compose up release

# Run standalone
docker run -p 8080:80 \
  -e POSTGRES_HOST=host.docker.internal \
  -e POSTGRES_USER=user \
  -e POSTGRES_PASSWORD=pass \
  -e POSTGRES_DB=mydb \
  -e ALLOWED_ORIGINS=https://myapp.com \
  myapi:latest
```

### Database Operations

```bash
# Connect to postgres
docker exec -it <container> psql -U myapi_user -d myapi_db

# Run migrations (using golang-migrate)
migrate -path migrations -database "postgres://user:pass@localhost:5432/mydb?sslmode=disable" up

# Check current version
migrate -path migrations -database "..." version

# Rollback one migration
migrate -path migrations -database "..." down 1
```

### Local Development (without Docker)

```bash
# Install dependencies
go mod download

# Run with Air (hot reload)
air

# Run directly
go run main.go

# Build binary
go build -o bin/api main.go
./bin/api
```

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /health | Health check |
| GET | /projects | List all projects |
| POST | /projects | Create project |
| GET | /projects/{id} | Get project by ID |
| PUT | /projects/{id} | Update project |
| DELETE | /projects/{id} | Delete project |

### Example Requests

```bash
# Health check
curl http://localhost:8000/health

# Create project
curl -X POST http://localhost:8000/projects \
  -H "Content-Type: application/json" \
  -d '{"name": "My Project", "summary": "Description"}'

# List projects
curl http://localhost:8000/projects

# Get project
curl http://localhost:8000/projects/{uuid}

# Update project
curl -X PUT http://localhost:8000/projects/{uuid} \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated", "summary": "New desc", "status": "active"}'

# Delete project
curl -X DELETE http://localhost:8000/projects/{uuid}
```

## Best Practices Summary

### Architecture

1. **Layered structure** - handlers → repository → database
2. **Dependency injection** - Pass dependencies through constructors
3. **Domain errors** - Define sentinel errors in models
4. **Graceful shutdown** - Handle signals properly

### Database

5. **Connection pooling** - Configure pool settings
6. **Retry logic** - Retry connections on startup
7. **Context propagation** - Pass context for cancellation
8. **RETURNING clause** - Get updated data in single query

### Docker

9. **Multi-stage builds** - Separate build and runtime
10. **Alpine images** - Smaller image sizes
11. **Volume mounts** - Enable hot reload in dev
12. **YAML anchors** - Reduce configuration duplication

### Error Handling

13. **Wrap errors** - Add context as errors propagate
14. **Log internally** - Log details, return generic messages
15. **Use sentinel errors** - Check with `errors.Is()`
16. **Validate input** - Check required fields in handlers
