# PostgreSQL Database Implementation

## Purpose

This folder contains the **PostgreSQL database adapter** with connection management, internal database records, SQL migrations, and repository implementations.

---

## Structure

```
postgresql/
├── entities/                   ← Database-specific record structures (Internal)
│   ├── client_record.go       ← Database record for clients
│   └── book_record.go         ← Database record for books
├── repositories/              ← Repository implementations (Adapters)
│   ├── client_repository.go   ← Implements application/repositories.ClientRepository
│   └── book_repository.go     ← Implements application/repositories.BookRepository
├── migrations/                ← SQL schema migrations
│   ├── 001_create_clients.sql
│   ├── 002_create_books.sql
│   └── 003_create_borrowed_books.sql
└── connection.go              ← Database connection pooling management
```

---

## Architectural Principle: Infrastructure Adapts to Domain

- **Application Layer** defines the repository contracts (`application/repositories/`).
- **Domain Layer** defines the business entities (`domain/entities/`).
- **PostgreSQL Adapter** queries SQL, scans into internal records (`entities/`), and converts directly to/from `domain/entities/`.

---

## Entities (Internal to Infrastructure)

### Client Record
```go
// infrastructure/postgresql/entities/client_record.go
package entities

import (
    "strings"
    "time"
    "ExampleProject/domain/entities"
)

type ClientRecord struct {
    ID              string    `db:"id"`
    Name            string    `db:"name"`
    Email           string    `db:"email"`
    Points          int       `db:"points"`
    BorrowedBookIDs string    `db:"borrowed_books"`
    MaxBooksAllowed int       `db:"max_books_allowed"`
    CreatedAt       time.Time `db:"created_at"`
    UpdatedAt       time.Time `db:"updated_at"`
}

// ToDomain converts database record to Domain Entity
func (r *ClientRecord) ToDomain() *entities.Client {
    var books []string
    if r.BorrowedBookIDs != "" {
        books = strings.Split(r.BorrowedBookIDs, ",")
    }
    return &entities.Client{
        ID:              r.ID,
        Name:            r.Name,
        Email:           r.Email,
        Points:          r.Points,
        BorrowedBookIDs: books,
        MaxBooksAllowed: r.MaxBooksAllowed,
    }
}

// FromDomainClient converts Domain Entity to database record
func FromDomainClient(c *entities.Client) *ClientRecord {
    return &ClientRecord{
        ID:              c.ID,
        Name:            c.Name,
        Email:           c.Email,
        Points:          c.Points,
        BorrowedBookIDs: strings.Join(c.BorrowedBookIDs, ","),
        MaxBooksAllowed: c.MaxBooksAllowed,
        UpdatedAt:       time.Now(),
    }
}
```

---

## Repository Implementation

```go
// infrastructure/postgresql/repositories/client_repository.go
package repositories

import (
    "context"
    "database/sql"
    "fmt"
    "ExampleProject/application/repositories"
    "ExampleProject/domain/entities"
    pgEntities "ExampleProject/infrastructure/postgresql/entities"
)

type PostgreSQLClientRepository struct {
    db *sql.DB
}

func NewPostgreSQLClientRepository(db *sql.DB) repositories.ClientRepository {
    return &PostgreSQLClientRepository{db: db}
}

// GetByID implements application/repositories.ClientRepository
// Returns Domain Entity
func (r *PostgreSQLClientRepository) GetByID(ctx context.Context, id string) (*entities.Client, error) {
    query := `
        SELECT id, name, email, points, max_books_allowed, borrowed_books, created_at, updated_at
        FROM clients
        WHERE id = $1
    `
    var rec pgEntities.ClientRecord
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &rec.ID,
        &rec.Name,
        &rec.Email,
        &rec.Points,
        &rec.MaxBooksAllowed,
        &rec.BorrowedBookIDs,
        &rec.CreatedAt,
        &rec.UpdatedAt,
    )
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("client not found: %s", id)
        }
        return nil, err
    }

    return rec.ToDomain(), nil
}

func (r *PostgreSQLClientRepository) Save(ctx context.Context, client *entities.Client) error {
    rec := pgEntities.FromDomainClient(client)
    query := `
        INSERT INTO clients (id, name, email, points, max_books_allowed, borrowed_books, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6, NOW(), NOW())
    `
    _, err := r.db.ExecContext(ctx, query, rec.ID, rec.Name, rec.Email, rec.Points, rec.MaxBooksAllowed, rec.BorrowedBookIDs)
    return err
}

func (r *PostgreSQLClientRepository) Update(ctx context.Context, client *entities.Client) error {
    rec := pgEntities.FromDomainClient(client)
    query := `
        UPDATE clients
        SET name = $1, email = $2, points = $3, max_books_allowed = $4, borrowed_books = $5, updated_at = NOW()
        WHERE id = $6
    `
    _, err := r.db.ExecContext(ctx, query, rec.Name, rec.Email, rec.Points, rec.MaxBooksAllowed, rec.BorrowedBookIDs, rec.ID)
    return err
}

func (r *PostgreSQLClientRepository) Delete(ctx context.Context, id string) error {
    query := `DELETE FROM clients WHERE id = $1`
    _, err := r.db.ExecContext(ctx, query, id)
    return err
}

func (r *PostgreSQLClientRepository) GetByEmail(ctx context.Context, email string) (*entities.Client, error) {
    query := `SELECT id, name, email, points, max_books_allowed, borrowed_books FROM clients WHERE email = $1`
    var rec pgEntities.ClientRecord
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &rec.ID, &rec.Name, &rec.Email, &rec.Points, &rec.MaxBooksAllowed, &rec.BorrowedBookIDs,
    )
    if err != nil {
        return nil, err
    }
    return rec.ToDomain(), nil
}
```

---

## Connection Management

```go
// infrastructure/postgresql/connection.go
package postgresql

import (
    "database/sql"
    "fmt"
    "time"
    _ "github.com/lib/pq"
)

type Config struct {
    Host     string
    Port     int
    User     string
    Password string
    Database string
    SSLMode  string
}

func NewConnection(config Config) (*sql.DB, error) {
    dsn := fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        config.Host, config.Port, config.User, config.Password, config.Database, config.SSLMode,
    )
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    return db, db.Ping()
}
```
