# PostgreSQL Database Patterns

This guide covers PostgreSQL patterns for application development, focusing on schema design decisions, connection management, and query patterns. The principles apply whether you're using Go, Node.js, or another language.

## Connection Management

### Why Connection Pooling Matters

Opening a database connection is expensive—it involves TCP handshake, authentication, and session setup. Without pooling:

```
Request 1: Open connection → Query → Close connection
Request 2: Open connection → Query → Close connection
Request 3: Open connection → Query → Close connection
```

Each request pays the full connection cost. With pooling:

```
Startup: Open 25 connections (pool)

Request 1: Borrow connection → Query → Return to pool
Request 2: Borrow connection → Query → Return to pool
Request 3: Borrow connection → Query → Return to pool
```

Connections are reused, dramatically reducing overhead.

### Pool Configuration

Key settings to configure:

```go
// Maximum open connections
// Set based on: database max_connections / number of app instances
db.SetMaxOpenConns(25)

// Maximum idle connections
// Keep some connections ready for quick response
db.SetMaxIdleConns(10)

// Maximum connection lifetime
// Recycle connections periodically to prevent stale connections
db.SetConnMaxLifetime(5 * time.Minute)

// Maximum idle time
// Close connections that sit unused too long
db.SetConnMaxIdleTime(5 * time.Minute)
```

**Tuning guidelines**:

- **MaxOpenConns**: Start with 25, monitor connection wait times
- **MaxIdleConns**: Usually 25-50% of MaxOpenConns
- **ConnMaxLifetime**: Shorter than database's `wait_timeout`
- **ConnMaxIdleTime**: Prevents idle connection accumulation

### Connection Retry Pattern

Database connections can fail at startup (database not ready) or during operation (network issues). Implement retry logic:

```go
func connectWithRetry(dsn string, maxRetries int) (*sql.DB, error) {
    var db *sql.DB
    var err error

    for i := 0; i < maxRetries; i++ {
        db, err = sql.Open("postgres", dsn)
        if err != nil {
            continue
        }

        if err = db.Ping(); err == nil {
            return db, nil
        }

        log.Printf("database connection attempt %d failed: %v", i+1, err)
        time.Sleep(time.Duration(i+1) * time.Second)  // Backoff
    }

    return nil, fmt.Errorf("failed to connect after %d attempts: %w", maxRetries, err)
}
```

**Why retry?** In containerized environments, the database might start slower than your application. Retries handle this gracefully.

## Schema Design Principles

### Use UUIDs for Primary Keys

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    -- ...
);
```

**Why UUIDs over auto-increment?**

1. **No sequence contention**: Auto-increment IDs require a lock to ensure uniqueness; UUIDs don't
2. **Merge-friendly**: Combining data from multiple databases doesn't cause ID collisions
3. **URL-safe**: UUIDs don't reveal information about your data volume or creation order
4. **Distributed systems**: Can generate IDs without database coordination

**Trade-offs**:
- UUIDs use more storage (16 bytes vs 4-8 bytes for integers)
- Random UUIDs fragment B-tree indexes (consider UUIDv7 for ordered IDs)
- Less human-readable for debugging

### Timestamps with Time Zones

```sql
CREATE TABLE articles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

**Why `TIMESTAMP WITH TIME ZONE`?**

`TIMESTAMP WITHOUT TIME ZONE` stores the literal value—if you insert `2024-01-15 10:00:00`, that's what's stored, with no timezone context.

`TIMESTAMP WITH TIME ZONE` converts to UTC for storage and converts back to the session timezone for display. This means:
- Data is unambiguous regardless of server location
- Comparisons work correctly across timezones
- Daylight saving time transitions are handled properly

**Rule**: Always use `TIMESTAMP WITH TIME ZONE` for wall-clock times.

### Automatic updated_at Trigger

Manually updating `updated_at` is error-prone. Use a trigger:

```sql
-- Create a reusable function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply to each table
CREATE TRIGGER update_articles_updated_at
    BEFORE UPDATE ON articles
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

**Why triggers?**
- Can't forget to update the timestamp
- Works regardless of how data is modified (application, migration, admin query)
- Reusable function applied to multiple tables

### Indexing Strategy

Indexes speed up reads but slow down writes. Create them deliberately:

```sql
-- Primary key is automatically indexed

-- Foreign keys should be indexed (PostgreSQL doesn't do this automatically)
CREATE INDEX idx_articles_author_id ON articles(author_id);

-- Columns used in WHERE clauses
CREATE INDEX idx_articles_status ON articles(status);

-- Columns used in ORDER BY (if queried frequently)
CREATE INDEX idx_articles_created_at ON articles(created_at DESC);

-- Composite index for common query patterns
CREATE INDEX idx_articles_author_status ON articles(author_id, status);
```

**Index guidelines**:

1. **Always index foreign keys** - Join performance depends on it
2. **Index columns in WHERE clauses** - If you filter by it, index it
3. **Consider composite indexes** - Column order matters; put equality conditions first
4. **Monitor before adding** - Use `EXPLAIN ANALYZE` to verify indexes help

### Cascade Deletes

When a parent record is deleted, what happens to children?

```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    article_id UUID NOT NULL REFERENCES articles(id) ON DELETE CASCADE,
    content TEXT NOT NULL
);
```

**Options**:

- `ON DELETE CASCADE` - Delete children automatically
- `ON DELETE SET NULL` - Set foreign key to NULL (column must be nullable)
- `ON DELETE RESTRICT` - Prevent deletion if children exist (default)

**When to use CASCADE**:
- Comments on an article (comments are meaningless without the article)
- Line items in an order (if the order is deleted, items should go too)

**When to use RESTRICT**:
- Users with orders (don't accidentally delete order history)
- Categories with products (require explicit reassignment)

## Migration Patterns

### File Naming Convention

```
migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_create_articles.up.sql
├── 000002_create_articles.down.sql
├── 000003_add_articles_published_at.up.sql
└── 000003_add_articles_published_at.down.sql
```

**Why this structure?**

1. **Numbered prefix** ensures migrations run in order
2. **Descriptive name** explains what the migration does
3. **up/down pairs** enable rollback

### Writing Reversible Migrations

Every `up` migration should have a corresponding `down`:

```sql
-- 000003_add_articles_published_at.up.sql
ALTER TABLE articles
ADD COLUMN published_at TIMESTAMP WITH TIME ZONE;

CREATE INDEX idx_articles_published_at ON articles(published_at);
```

```sql
-- 000003_add_articles_published_at.down.sql
DROP INDEX IF EXISTS idx_articles_published_at;

ALTER TABLE articles
DROP COLUMN IF EXISTS published_at;
```

**Why reversible?**
- Roll back failed deployments
- Remove features cleanly
- Test migrations in both directions

### Safe Schema Changes

Some changes lock tables and block queries. For high-traffic tables:

**Adding a column** (safe):
```sql
ALTER TABLE articles ADD COLUMN view_count INTEGER DEFAULT 0;
```

**Adding an index** (can block on large tables):
```sql
-- CONCURRENTLY prevents locking, but takes longer
CREATE INDEX CONCURRENTLY idx_articles_view_count ON articles(view_count);
```

**Renaming a column** (requires careful migration):
1. Add new column
2. Deploy code that writes to both columns
3. Backfill old data
4. Deploy code that reads from new column
5. Drop old column

## Query Patterns

### Use RETURNING to Avoid Extra Queries

Instead of INSERT then SELECT:

```sql
-- Two queries
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
SELECT * FROM users WHERE email = 'alice@example.com';

-- One query with RETURNING
INSERT INTO users (name, email)
VALUES ('Alice', 'alice@example.com')
RETURNING id, name, email, created_at, updated_at;
```

Same for UPDATE:

```sql
UPDATE users
SET name = 'Alicia'
WHERE id = '123'
RETURNING id, name, email, created_at, updated_at;
```

**Why RETURNING?**
- One round-trip instead of two
- Atomic—no race condition between INSERT and SELECT
- Returns the actual stored values (after defaults and triggers)

### Parameterized Queries

Never concatenate user input into SQL:

```go
// DANGEROUS - SQL injection vulnerability
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", userInput)

// SAFE - parameterized query
query := "SELECT * FROM users WHERE email = $1"
rows, err := db.QueryContext(ctx, query, userInput)
```

**Why parameterized queries?**
- Input is treated as data, not SQL
- Database can cache and reuse query plans
- Prevents SQL injection attacks

### Handle NULL Values

NULL in SQL requires special handling:

```go
// Using sql.NullString for nullable columns
type User struct {
    ID        string
    Name      string
    Bio       sql.NullString  // Nullable in database
}

// Scanning nullable column
err := row.Scan(&user.ID, &user.Name, &user.Bio)

// Checking if value is present
if user.Bio.Valid {
    fmt.Println(user.Bio.String)
}
```

**Alternative—COALESCE in SQL**:

```sql
SELECT id, name, COALESCE(bio, '') as bio FROM users
```

This returns empty string instead of NULL, simplifying application code.

### Transaction Patterns

Use transactions when multiple operations must succeed or fail together:

```go
func transferFunds(ctx context.Context, db *sql.DB, from, to string, amount int) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback()  // No-op if already committed

    // Debit source account
    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        amount, from)
    if err != nil {
        return fmt.Errorf("debit failed: %w", err)
    }

    // Credit destination account
    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, to)
    if err != nil {
        return fmt.Errorf("credit failed: %w", err)
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit failed: %w", err)
    }

    return nil
}
```

**Key patterns**:
- `defer tx.Rollback()` - Ensures rollback if anything fails
- All operations use `tx`, not `db`
- Explicit `Commit()` at the end

### Checking Affected Rows

For UPDATE and DELETE, verify something actually changed:

```go
result, err := db.ExecContext(ctx,
    "DELETE FROM users WHERE id = $1",
    userID)
if err != nil {
    return fmt.Errorf("delete failed: %w", err)
}

rowsAffected, err := result.RowsAffected()
if err != nil {
    return fmt.Errorf("rows affected: %w", err)
}

if rowsAffected == 0 {
    return ErrUserNotFound
}
```

**Why check affected rows?**
- Distinguish "not found" from "deleted successfully"
- Detect concurrent modification issues
- Provide accurate feedback to users

## Connection String Configuration

### Environment-Based Configuration

```bash
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=myapp
POSTGRES_PASSWORD=secret
POSTGRES_DB=myapp_development
```

Build the connection string:

```go
dsn := fmt.Sprintf(
    "host=%s port=%s user=%s password=%s dbname=%s sslmode=disable",
    os.Getenv("POSTGRES_HOST"),
    os.Getenv("POSTGRES_PORT"),
    os.Getenv("POSTGRES_USER"),
    os.Getenv("POSTGRES_PASSWORD"),
    os.Getenv("POSTGRES_DB"),
)
```

**SSL Mode options**:
- `disable` - No SSL (development only)
- `require` - Use SSL but don't verify certificate
- `verify-full` - Use SSL and verify certificate (production)

## Best Practices Summary

### Schema Design

1. **Use UUIDs for IDs** - Better for distributed systems and don't leak information
2. **TIMESTAMP WITH TIME ZONE** - Always store timestamps with timezone awareness
3. **Automatic updated_at** - Use triggers to ensure consistency
4. **Index foreign keys** - PostgreSQL doesn't do this automatically
5. **Explicit cascade behavior** - Choose CASCADE or RESTRICT deliberately

### Queries

6. **Use RETURNING** - Get inserted/updated data in one query
7. **Parameterized queries always** - Never concatenate user input into SQL
8. **Handle NULL explicitly** - Use `sql.NullString` or `COALESCE`
9. **Check affected rows** - Verify UPDATE/DELETE actually changed something

### Connections

10. **Configure connection pool** - Don't use defaults in production
11. **Retry on startup** - Database might not be ready immediately
12. **Use context** - Enable query cancellation and timeouts

### Migrations

13. **Number migrations** - Ensure consistent ordering
14. **Write reversible migrations** - Always include `down` scripts
15. **Test both directions** - Verify `up` and `down` work correctly
