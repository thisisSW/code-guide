# Go API Development

Patterns for building REST APIs in Go using gorilla/mux router, repository pattern, and standard library conventions.

## Project Structure

```
project-root/
├── main.go                 # Application entry point
├── go.mod                  # Go module definition
├── go.sum                  # Dependency checksums
├── .env                    # Environment variables
├── migrations/             # Database migrations
│   ├── 000001_init_schema.up.sql
│   └── 000001_init_schema.down.sql
└── pkg/
    ├── database/           # Database connection
    │   └── database.go
    ├── errors/             # Error utilities
    │   └── errors.go
    ├── handlers/           # HTTP handlers
    │   ├── health.go
    │   ├── projects.go
    │   └── response.go     # Response helpers
    ├── middleware/         # HTTP middleware
    │   ├── logger.go
    │   └── cors.go
    ├── models/             # Data models
    │   ├── project.go
    │   └── errors.go       # Domain errors
    ├── repository/         # Data access layer
    │   └── project.go
    ├── router/             # Route configuration
    │   └── router.go
    ├── server/             # HTTP server setup
    │   └── server.go
    └── services/           # Business logic (optional)
        └── workflow.go
```

## Module Setup

### go.mod

```go
module github.com/username/myapp

go 1.21

require (
	github.com/gorilla/mux v1.8.1
	github.com/lib/pq v1.10.9
)
```

**Key Points**:
- Module path should match your repository
- Use Go 1.21+ for latest features
- `gorilla/mux` for HTTP routing
- `lib/pq` for PostgreSQL driver

## Application Entry

### main.go

```go
package main

import (
	"log"
	"os"

	"github.com/username/myapp/pkg/database"
	"github.com/username/myapp/pkg/server"
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

**Key Points**:
- Environment variables for configuration
- Defer database close
- Structured error handling with `log.Fatalf`
- Clean separation of concerns

## Server Setup

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

	"github.com/username/myapp/pkg/errors"
	"github.com/username/myapp/pkg/router"
)

// Server represents the HTTP server
type Server struct {
	httpServer *http.Server
	port       string
	db         *sql.DB
}

// New creates a new server instance
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

// Start starts the HTTP server with graceful shutdown
func (s *Server) Start() error {
	// Channel to listen for errors from the server
	serverErrors := make(chan error, 1)

	// Start the server
	go func() {
		serverErrors <- s.httpServer.ListenAndServe()
	}()

	// Channel to listen for interrupt or terminate signals
	shutdown := make(chan os.Signal, 1)
	signal.Notify(shutdown, os.Interrupt, syscall.SIGTERM)

	// Block until we receive a signal or an error
	select {
	case err := <-serverErrors:
		return errors.Wrap(err, "server error")

	case sig := <-shutdown:
		fmt.Printf("\nReceived signal: %v. Starting graceful shutdown...\n", sig)

		// Give outstanding requests 20 seconds to complete
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

**Key Points**:
- Configurable timeouts prevent resource exhaustion
- Graceful shutdown allows in-flight requests to complete
- Signal handling for Docker/Kubernetes deployments

## Router Configuration

### pkg/router/router.go

```go
package router

import (
	"database/sql"
	"net/http"

	"github.com/gorilla/mux"
	"github.com/username/myapp/pkg/handlers"
	"github.com/username/myapp/pkg/middleware"
	"github.com/username/myapp/pkg/repository"
)

// New creates and configures a new router
func New(db *sql.DB) *mux.Router {
	r := mux.NewRouter()

	// Global middleware
	r.Use(middleware.Logger)
	r.Use(middleware.CORS)

	// Health check
	r.HandleFunc("/health", handlers.HealthCheck).Methods(http.MethodGet)

	// Initialize repositories
	projectRepo := repository.NewProjectRepository(db)

	// Initialize handlers
	projectHandler := handlers.NewProjectHandler(projectRepo)

	// Project routes
	// UUID regex pattern for path parameters
	uuidPattern := "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"

	r.HandleFunc("/projects", projectHandler.Create).Methods(http.MethodPost, http.MethodOptions)
	r.HandleFunc("/projects", projectHandler.List).Methods(http.MethodGet, http.MethodOptions)
	r.HandleFunc("/projects/{id:"+uuidPattern+"}", projectHandler.Get).Methods(http.MethodGet, http.MethodOptions)
	r.HandleFunc("/projects/{id:"+uuidPattern+"}", projectHandler.Update).Methods(http.MethodPut, http.MethodOptions)
	r.HandleFunc("/projects/{id:"+uuidPattern+"}", projectHandler.Delete).Methods(http.MethodDelete, http.MethodOptions)

	return r
}
```

**Key Points**:
- Middleware applied with `r.Use()`
- UUID validation with regex in route
- `http.MethodOptions` for CORS preflight
- Dependency injection through constructors

## Middleware

### pkg/middleware/logger.go

```go
package middleware

import (
	"log"
	"net/http"
	"time"
)

// responseWriter wraps http.ResponseWriter to capture status code
type responseWriter struct {
	http.ResponseWriter
	statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

// Logger logs HTTP requests
func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		rw := &responseWriter{
			ResponseWriter: w,
			statusCode:     http.StatusOK,
		}

		next.ServeHTTP(rw, r)

		log.Printf(
			"%s %s %d %s",
			r.Method,
			r.RequestURI,
			rw.statusCode,
			time.Since(start),
		)
	})
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

// CORS handles Cross-Origin Resource Sharing
func CORS(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Get allowed origins from environment
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

		// Handle preflight
		if r.Method == http.MethodOptions {
			w.WriteHeader(http.StatusOK)
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

## Models

### pkg/models/project.go

```go
package models

import "time"

// Project represents a project entity
type Project struct {
	ID        string    `json:"id" db:"id"`
	Name      string    `json:"name" db:"name"`
	Summary   string    `json:"summary" db:"summary"`
	Status    string    `json:"status" db:"status"`
	CreatedAt time.Time `json:"created_at" db:"created_at"`
	UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// CreateProjectRequest represents the request to create a project
type CreateProjectRequest struct {
	Name    string `json:"name"`
	Summary string `json:"summary"`
}

// UpdateProjectRequest represents the request to update a project
type UpdateProjectRequest struct {
	Name    string `json:"name"`
	Summary string `json:"summary"`
	Status  string `json:"status"`
}
```

**Key Points**:
- Struct tags for JSON serialization
- Separate request/response types from domain models
- `db` tags for database mapping (if using sqlx)

### pkg/models/errors.go

```go
package models

import "errors"

// Domain errors
var (
	ErrProjectNotFound = errors.New("project not found")
	ErrBriefNotFound   = errors.New("brief not found")
)
```

## Repository Pattern

### pkg/repository/project.go

```go
package repository

import (
	"context"
	"database/sql"

	"github.com/username/myapp/pkg/errors"
	"github.com/username/myapp/pkg/models"
)

// ProjectRepository handles project data operations
type ProjectRepository struct {
	db *sql.DB
}

// NewProjectRepository creates a new project repository
func NewProjectRepository(db *sql.DB) *ProjectRepository {
	return &ProjectRepository{db: db}
}

// Create creates a new project
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

// GetByID retrieves a project by ID
func (r *ProjectRepository) GetByID(ctx context.Context, id string) (*models.Project, error) {
	query := `
		SELECT id, name, summary, status, created_at, updated_at
		FROM projects
		WHERE id = $1
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

// List retrieves all projects
func (r *ProjectRepository) List(ctx context.Context) ([]*models.Project, error) {
	query := `
		SELECT id, name, summary, status, created_at, updated_at
		FROM projects
		ORDER BY created_at DESC
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

// Update updates a project
func (r *ProjectRepository) Update(ctx context.Context, id string, req *models.UpdateProjectRequest) (*models.Project, error) {
	query := `
		UPDATE projects
		SET name = $1, summary = $2, status = $3
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

// Delete deletes a project
func (r *ProjectRepository) Delete(ctx context.Context, id string) error {
	query := `DELETE FROM projects WHERE id = $1`

	result, err := r.db.ExecContext(ctx, query, id)
	if err != nil {
		return errors.Wrap(err, "failed to delete project")
	}

	rowsAffected, err := result.RowsAffected()
	if err != nil {
		return errors.Wrap(err, "failed to get rows affected")
	}

	if rowsAffected == 0 {
		return models.ErrProjectNotFound
	}

	return nil
}
```

**Key Points**:
- Accept `context.Context` for cancellation/timeouts
- Use `RETURNING` to get updated data
- Handle `sql.ErrNoRows` for not found cases
- Always `defer rows.Close()` for queries
- Check `rows.Err()` after iteration

## Handlers

### pkg/handlers/response.go

```go
package handlers

import (
	"encoding/json"
	"net/http"
)

// respondJSON writes a JSON response
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

// respondError writes an error response
func respondError(w http.ResponseWriter, status int, message string) {
	respondJSON(w, status, map[string]string{"error": message})
}
```

### pkg/handlers/health.go

```go
package handlers

import (
	"encoding/json"
	"net/http"
)

// HealthCheck returns the health status of the API
func HealthCheck(w http.ResponseWriter, r *http.Request) {
	response := map[string]string{
		"status": "healthy",
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(response)
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
	"github.com/username/myapp/pkg/errors"
	"github.com/username/myapp/pkg/models"
	"github.com/username/myapp/pkg/repository"
)

// ProjectHandler handles project requests
type ProjectHandler struct {
	repo *repository.ProjectRepository
}

// NewProjectHandler creates a new project handler
func NewProjectHandler(repo *repository.ProjectRepository) *ProjectHandler {
	return &ProjectHandler{repo: repo}
}

// Create handles POST /projects
func (h *ProjectHandler) Create(w http.ResponseWriter, r *http.Request) {
	var req models.CreateProjectRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		log.Printf("Create: failed to decode request body: %v", err)
		respondError(w, http.StatusBadRequest, "invalid request body")
		return
	}

	// Validate required fields
	if req.Name == "" {
		respondError(w, http.StatusBadRequest, "name is required")
		return
	}

	project, err := h.repo.Create(r.Context(), &req)
	if err != nil {
		log.Printf("Create: failed to create project: %v", err)
		respondError(w, http.StatusInternalServerError, "failed to create project")
		return
	}

	respondJSON(w, http.StatusCreated, project)
}

// Get handles GET /projects/{id}
func (h *ProjectHandler) Get(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id := vars["id"]

	project, err := h.repo.GetByID(r.Context(), id)
	if err != nil {
		if errors.Is(err, models.ErrProjectNotFound) {
			respondError(w, http.StatusNotFound, "project not found")
			return
		}
		log.Printf("Get: failed to get project %s: %v", id, err)
		respondError(w, http.StatusInternalServerError, "failed to get project")
		return
	}

	respondJSON(w, http.StatusOK, project)
}

// List handles GET /projects
func (h *ProjectHandler) List(w http.ResponseWriter, r *http.Request) {
	projects, err := h.repo.List(r.Context())
	if err != nil {
		log.Printf("List: failed to list projects: %v", err)
		respondError(w, http.StatusInternalServerError, "failed to list projects")
		return
	}

	respondJSON(w, http.StatusOK, projects)
}

// Update handles PUT /projects/{id}
func (h *ProjectHandler) Update(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id := vars["id"]

	var req models.UpdateProjectRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		log.Printf("Update: failed to decode request body: %v", err)
		respondError(w, http.StatusBadRequest, "invalid request body")
		return
	}

	project, err := h.repo.Update(r.Context(), id, &req)
	if err != nil {
		if errors.Is(err, models.ErrProjectNotFound) {
			respondError(w, http.StatusNotFound, "project not found")
			return
		}
		log.Printf("Update: failed to update project %s: %v", id, err)
		respondError(w, http.StatusInternalServerError, "failed to update project")
		return
	}

	respondJSON(w, http.StatusOK, project)
}

// Delete handles DELETE /projects/{id}
func (h *ProjectHandler) Delete(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id := vars["id"]

	if err := h.repo.Delete(r.Context(), id); err != nil {
		if errors.Is(err, models.ErrProjectNotFound) {
			respondError(w, http.StatusNotFound, "project not found")
			return
		}
		log.Printf("Delete: failed to delete project %s: %v", id, err)
		respondError(w, http.StatusInternalServerError, "failed to delete project")
		return
	}

	w.WriteHeader(http.StatusNoContent)
}
```

**Key Points**:
- Use `mux.Vars(r)` to get path parameters
- Pass `r.Context()` to repository methods
- Return appropriate HTTP status codes
- Log errors server-side, return generic messages to clients
- Check for domain errors like `ErrProjectNotFound`

## Error Utilities

### pkg/errors/errors.go

```go
package errors

import (
	"errors"
	"fmt"
)

// New creates a new error with a message
func New(message string) error {
	return errors.New(message)
}

// Wrap wraps an error with additional context
func Wrap(err error, message string) error {
	if err == nil {
		return nil
	}
	return fmt.Errorf("%s: %w", message, err)
}

// Is reports whether any error in err's chain matches target
func Is(err, target error) bool {
	return errors.Is(err, target)
}
```

**Key Points**:
- `%w` verb wraps errors (Go 1.13+)
- `errors.Is` checks error chain
- Wrap errors with context as they bubble up

## Best Practices

### Code Organization

1. **One package per concern** - database, handlers, models, etc.
2. **Constructor functions** - `NewXxx()` for initializing structs
3. **Interface receivers** - Use pointer receivers for methods
4. **Package-level errors** - Define domain errors in models

### Error Handling

5. **Always check errors** - Never ignore returned errors
6. **Wrap with context** - Add context as errors propagate
7. **Log and respond** - Log details server-side, return safe messages
8. **Use sentinel errors** - Define typed errors for specific cases

### HTTP Handlers

9. **Accept context** - Pass `r.Context()` to downstream calls
10. **Validate input** - Check required fields before processing
11. **Use appropriate status codes** - 201 Created, 204 No Content, etc.
12. **Consistent response format** - Standard JSON structure

### Database

13. **Use context** - Accept context for cancellation
14. **Close rows** - Always `defer rows.Close()`
15. **Check rows.Err()** - After iterating through rows
16. **Use RETURNING** - Get updated data in single query
17. **Parameterized queries** - Use `$1, $2` placeholders (never concatenate)

### Testing

18. **Table-driven tests** - Use `[]struct{}` for test cases
19. **Test handlers** - Use `httptest.NewRecorder()`
20. **Mock repositories** - Use interfaces for dependency injection
