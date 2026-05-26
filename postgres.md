# PostgreSQL Database Patterns

Patterns for using PostgreSQL with Go applications, including connection management, migrations, and common queries.

## Connection Configuration

### Environment Variables

```bash
POSTGRES_HOST=postgres
POSTGRES_USER=myapp_user
POSTGRES_PASSWORD=myapp_password
POSTGRES_DB=myapp_db
POSTGRES_PORT=5432
```

### pkg/database/database.go

```go
package database

import (
	"database/sql"
	"fmt"
	"os"
	"time"

	"github.com/username/myapp/pkg/errors"
	_ "github.com/lib/pq"
)

// Config holds database configuration
type Config struct {
	Host     string
	Port     string
	User     string
	Password string
	DBName   string
}

// DB represents the database connection
type DB struct {
	*sql.DB
}

// New creates a new database connection
func New(cfg Config) (*DB, error) {
	// Build connection string
	connStr := fmt.Sprintf(
		"host=%s port=%s user=%s password=%s dbname=%s sslmode=disable",
		cfg.Host, cfg.Port, cfg.User, cfg.Password, cfg.DBName,
	)

	// Open connection
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

// NewFromEnv creates a database connection from environment variables
func NewFromEnv() (*DB, error) {
	cfg := Config{
		Host:     os.Getenv("POSTGRES_HOST"),
		Port:     os.Getenv("POSTGRES_PORT"),
		User:     os.Getenv("POSTGRES_USER"),
		Password: os.Getenv("POSTGRES_PASSWORD"),
		DBName:   os.Getenv("POSTGRES_DB"),
	}

	// Set defaults
	if cfg.Port == "" {
		cfg.Port = "5432"
	}

	// Validate required fields
	if cfg.Host == "" || cfg.User == "" || cfg.Password == "" || cfg.DBName == "" {
		return nil, errors.New("missing required database configuration")
	}

	return New(cfg)
}

// Close closes the database connection
func (db *DB) Close() error {
	if db == nil || db.DB == nil {
		return nil
	}
	if err := db.DB.Close(); err != nil {
		return errors.Wrap(err, "failed to close database connection")
	}
	return nil
}
```

**Key Points**:
- Use environment variables for configuration
- Configure connection pool settings
- Retry connection with exponential backoff
- Validate required configuration before connecting
- Blank import `lib/pq` for driver registration

## Connection Pool Settings

```go
// SetMaxOpenConns - Maximum number of open connections
// - Set based on your database's max_connections
// - Leave room for other services
db.SetMaxOpenConns(25)

// SetMaxIdleConns - Maximum idle connections to keep
// - Should be <= MaxOpenConns
// - Higher values = faster response but more memory
db.SetMaxIdleConns(25)

// SetConnMaxLifetime - Maximum time a connection can be reused
// - Helps recycle connections
// - Set lower than database's wait_timeout
db.SetConnMaxLifetime(5 * time.Minute)

// SetConnMaxIdleTime - Maximum time a connection can be idle
// - Closes idle connections to free resources
db.SetConnMaxIdleTime(5 * time.Minute)
```

## Migrations

### Migration Files

Migrations use numbered files for ordering and separate up/down files for reversibility.

```
migrations/
├── 000001_init_schema.up.sql
├── 000001_init_schema.down.sql
├── 000002_create_projects.up.sql
├── 000002_create_projects.down.sql
├── 000003_create_product_briefs.up.sql
└── 000003_create_product_briefs.down.sql
```

### 000001_init_schema.up.sql

Initialize database with extensions and utility functions.

```sql
-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### 000001_init_schema.down.sql

```sql
-- Extensions are typically not dropped
-- DROP EXTENSION IF EXISTS "uuid-ossp";
```

### 000002_create_projects.up.sql

```sql
-- Create projects table
CREATE TABLE IF NOT EXISTS projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    summary TEXT,
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Create index on name for search
CREATE INDEX idx_projects_name ON projects(name);

-- Create index on status for filtering
CREATE INDEX idx_projects_status ON projects(status);

-- Create updated_at trigger function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply trigger to projects table
CREATE TRIGGER update_projects_updated_at
    BEFORE UPDATE ON projects
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### 000002_create_projects.down.sql

```sql
DROP TRIGGER IF EXISTS update_projects_updated_at ON projects;
DROP TABLE IF EXISTS projects;
-- Note: Keep the trigger function for other tables
```

### 000003_create_product_briefs.up.sql

```sql
-- Create product_briefs table with foreign key
CREATE TABLE IF NOT EXISTS product_briefs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    problem_statement TEXT,
    stakeholder_goals TEXT,
    core_requirements TEXT,
    mvp_scope TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id)
);

-- Create index on project_id for joins
CREATE INDEX idx_product_briefs_project_id ON product_briefs(project_id);

-- Apply updated_at trigger
CREATE TRIGGER update_product_briefs_updated_at
    BEFORE UPDATE ON product_briefs
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### 000003_create_product_briefs.down.sql

```sql
DROP TRIGGER IF EXISTS update_product_briefs_updated_at ON product_briefs;
DROP TABLE IF EXISTS product_briefs;
```

**Key Points**:
- Use `uuid-ossp` extension for UUID generation
- `TIMESTAMP WITH TIME ZONE` for proper timezone handling
- `ON DELETE CASCADE` for automatic cleanup
- Indexes on foreign keys and frequently queried columns
- Reusable trigger function for `updated_at`

## Common Query Patterns

### Create with RETURNING

```go
func (r *Repository) Create(ctx context.Context, req *models.CreateRequest) (*models.Entity, error) {
    query := `
        INSERT INTO entities (name, description)
        VALUES ($1, $2)
        RETURNING id, name, description, created_at, updated_at
    `

    var entity models.Entity
    err := r.db.QueryRowContext(ctx, query, req.Name, req.Description).Scan(
        &entity.ID,
        &entity.Name,
        &entity.Description,
        &entity.CreatedAt,
        &entity.UpdatedAt,
    )
    if err != nil {
        return nil, errors.Wrap(err, "failed to create entity")
    }

    return &entity, nil
}
```

### Get Single Row

```go
func (r *Repository) GetByID(ctx context.Context, id string) (*models.Entity, error) {
    query := `
        SELECT id, name, description, created_at, updated_at
        FROM entities
        WHERE id = $1
    `

    var entity models.Entity
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &entity.ID,
        &entity.Name,
        &entity.Description,
        &entity.CreatedAt,
        &entity.UpdatedAt,
    )
    if err == sql.ErrNoRows {
        return nil, models.ErrEntityNotFound
    }
    if err != nil {
        return nil, errors.Wrap(err, "failed to get entity")
    }

    return &entity, nil
}
```

### List Multiple Rows

```go
func (r *Repository) List(ctx context.Context) ([]*models.Entity, error) {
    query := `
        SELECT id, name, description, created_at, updated_at
        FROM entities
        ORDER BY created_at DESC
    `

    rows, err := r.db.QueryContext(ctx, query)
    if err != nil {
        return nil, errors.Wrap(err, "failed to query entities")
    }
    defer rows.Close()

    var entities []*models.Entity
    for rows.Next() {
        var entity models.Entity
        if err := rows.Scan(
            &entity.ID,
            &entity.Name,
            &entity.Description,
            &entity.CreatedAt,
            &entity.UpdatedAt,
        ); err != nil {
            return nil, errors.Wrap(err, "failed to scan entity")
        }
        entities = append(entities, &entity)
    }

    if err := rows.Err(); err != nil {
        return nil, errors.Wrap(err, "error iterating entities")
    }

    return entities, nil
}
```

### Update with RETURNING

```go
func (r *Repository) Update(ctx context.Context, id string, req *models.UpdateRequest) (*models.Entity, error) {
    query := `
        UPDATE entities
        SET name = $1, description = $2
        WHERE id = $3
        RETURNING id, name, description, created_at, updated_at
    `

    var entity models.Entity
    err := r.db.QueryRowContext(ctx, query, req.Name, req.Description, id).Scan(
        &entity.ID,
        &entity.Name,
        &entity.Description,
        &entity.CreatedAt,
        &entity.UpdatedAt,
    )
    if err == sql.ErrNoRows {
        return nil, models.ErrEntityNotFound
    }
    if err != nil {
        return nil, errors.Wrap(err, "failed to update entity")
    }

    return &entity, nil
}
```

### Delete

```go
func (r *Repository) Delete(ctx context.Context, id string) error {
    query := `DELETE FROM entities WHERE id = $1`

    result, err := r.db.ExecContext(ctx, query, id)
    if err != nil {
        return errors.Wrap(err, "failed to delete entity")
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return errors.Wrap(err, "failed to get rows affected")
    }

    if rowsAffected == 0 {
        return models.ErrEntityNotFound
    }

    return nil
}
```

### Query with Foreign Key Join

```go
func (r *Repository) GetBriefByProjectID(ctx context.Context, projectID string) (*models.Brief, error) {
    query := `
        SELECT b.id, b.project_id, b.problem_statement, b.stakeholder_goals,
               b.core_requirements, b.mvp_scope, b.created_at, b.updated_at
        FROM product_briefs b
        WHERE b.project_id = $1
    `

    var brief models.Brief
    err := r.db.QueryRowContext(ctx, query, projectID).Scan(
        &brief.ID,
        &brief.ProjectID,
        &brief.ProblemStatement,
        &brief.StakeholderGoals,
        &brief.CoreRequirements,
        &brief.MVPScope,
        &brief.CreatedAt,
        &brief.UpdatedAt,
    )
    if err == sql.ErrNoRows {
        return nil, models.ErrBriefNotFound
    }
    if err != nil {
        return nil, errors.Wrap(err, "failed to get brief")
    }

    return &brief, nil
}
```

### Handling NULL Values

Use `sql.NullString`, `sql.NullInt64`, etc. for nullable columns.

```go
type Entity struct {
    ID          string
    Name        string
    Description sql.NullString  // Nullable column
}

// Scan handles NULL
err := rows.Scan(&entity.ID, &entity.Name, &entity.Description)

// Check if valid
if entity.Description.Valid {
    fmt.Println(entity.Description.String)
}

// Or use COALESCE in SQL
query := `SELECT id, name, COALESCE(description, '') as description FROM entities`
```

## Transaction Patterns

### Basic Transaction

```go
func (r *Repository) TransferItem(ctx context.Context, itemID, fromProjectID, toProjectID string) error {
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return errors.Wrap(err, "failed to begin transaction")
    }
    defer tx.Rollback() // No-op if committed

    // Remove from source
    _, err = tx.ExecContext(ctx,
        `DELETE FROM project_items WHERE item_id = $1 AND project_id = $2`,
        itemID, fromProjectID)
    if err != nil {
        return errors.Wrap(err, "failed to remove item from source")
    }

    // Add to destination
    _, err = tx.ExecContext(ctx,
        `INSERT INTO project_items (item_id, project_id) VALUES ($1, $2)`,
        itemID, toProjectID)
    if err != nil {
        return errors.Wrap(err, "failed to add item to destination")
    }

    if err := tx.Commit(); err != nil {
        return errors.Wrap(err, "failed to commit transaction")
    }

    return nil
}
```

**Key Points**:
- Always `defer tx.Rollback()` - no-op if already committed
- Use `BeginTx` with context for cancellation support
- Commit explicitly at the end

## Docker Compose for PostgreSQL

### docker-compose.yaml

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

volumes:
  postgres:
    driver: local
```

### .env

```bash
POSTGRES_USER=myapp_user
POSTGRES_PASSWORD=myapp_password
POSTGRES_DB=myapp_db
ALLOWED_ORIGINS=http://localhost:5173,http://localhost:3000
PORT=8000
```

**Key Points**:
- YAML anchors (`&db-variables`) reduce duplication
- `depends_on` ensures postgres starts first
- Shared network for service communication
- Volume mount for data persistence
- `postgres:15.3-alpine` for smaller image

## Running Migrations

### Using golang-migrate

Install the CLI:
```bash
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

Run migrations:
```bash
# Up
migrate -path migrations -database "postgres://user:pass@localhost:5432/dbname?sslmode=disable" up

# Down one step
migrate -path migrations -database "postgres://user:pass@localhost:5432/dbname?sslmode=disable" down 1

# Force version (after failed migration)
migrate -path migrations -database "postgres://user:pass@localhost:5432/dbname?sslmode=disable" force VERSION
```

### Using psql

```bash
# Connect to database
docker exec -it <container> psql -U myapp_user -d myapp_db

# Run migration file
\i /path/to/migration.sql

# Check tables
\dt

# Describe table
\d projects
```

## Best Practices

### Schema Design

1. **Use UUIDs for IDs** - Better for distributed systems
2. **Timestamp with timezone** - Always use `TIMESTAMP WITH TIME ZONE`
3. **Meaningful indexes** - On foreign keys and query predicates
4. **Cascading deletes** - Use `ON DELETE CASCADE` for child tables
5. **Default values** - Set sensible defaults in schema

### Query Patterns

6. **Use RETURNING** - Avoid extra SELECT after INSERT/UPDATE
7. **Parameterized queries** - Always use `$1, $2` placeholders
8. **Context everywhere** - Pass context for cancellation
9. **Close rows** - Always `defer rows.Close()`
10. **Check rows.Err()** - After iterating through results

### Connection Management

11. **Connection pooling** - Configure pool settings appropriately
12. **Retry logic** - Retry connections on startup
13. **Health checks** - Use `db.Ping()` for health endpoints
14. **Graceful shutdown** - Close connections on shutdown

### Migrations

15. **Numbered files** - Use `000001_` prefix for ordering
16. **Separate up/down** - Always write reversible migrations
17. **Atomic changes** - Each migration should be complete
18. **Test migrations** - Test both up and down paths
