# Infrastructure Layer (Driven Adapters)

## Purpose

The infrastructure layer contains **concrete implementations (adapters)** of the outbound interfaces (ports) defined in the **application layer**. 

This layer interacts directly with external systems:
- Databases (PostgreSQL, MySQL, Redis)
- Third-party REST / gRPC APIs (Loyalty, Serasa, Clearing, Resend)
- Message brokers (RabbitMQ, Kafka)
- File systems and object stores

---

## Key Architectural Principle: Infrastructure Adapts to Domain

- **Application defines the contracts** (`application/repositories/`, `application/services/`).
- **Infrastructure implements the contracts** using vendor libraries, SQL drivers, and HTTP clients.
- **Infrastructure translates directly to/from Domain Entities**: DB record structs and API response schemas map directly to `domain/entities`.

---

## Directory Structure

```
infrastructure/
├── postgresql/              ← PostgreSQL implementation
│   ├── entities/           ← Internal DB record structures (ClientRecord, BookRecord)
│   ├── repositories/       ← Implements application/repositories
│   └── connection.go       ← DB connection pooling
│
├── loyalty_points/         ← Loyalty points external API adapter
│   ├── entities/           ← External JSON payloads (Internal to infrastructure)
│   ├── client.go           ← Raw HTTP client
│   └── service_impl.go     ← Implements application/services.PointsService
│
└── http/                   ← Shared HTTP transport utilities (retries, timeouts)
```

---

## Example: PostgreSQL Adapter Mapping to Domain Entities

### 1. Database-Specific Record (Internal)
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
    BorrowedBookIDs string    `db:"borrowed_books"` // CSV stored in DB
    MaxBooksAllowed int       `db:"max_books"`
    CreatedAt       time.Time `db:"created_at"`
    UpdatedAt       time.Time `db:"updated_at"`
}

// Maps DB record directly to Domain Entity
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

// Maps Domain Entity directly to DB record
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

### 2. Repository Implementation
```go
// infrastructure/postgresql/repositories/client_repository.go
package repositories

import (
    "context"
    "database/sql"
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

func (r *PostgreSQLClientRepository) GetByID(ctx context.Context, id string) (*entities.Client, error) {
    query := `SELECT id, name, email, points, borrowed_books, max_books FROM clients WHERE id = $1`
    var rec pgEntities.ClientRecord
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &rec.ID, &rec.Name, &rec.Email, &rec.Points, &rec.BorrowedBookIDs, &rec.MaxBooksAllowed,
    )
    if err != nil {
        return nil, err
    }
    return rec.ToDomain(), nil
}

func (r *PostgreSQLClientRepository) Update(ctx context.Context, client *entities.Client) error {
    rec := pgEntities.FromDomainClient(client)
    query := `UPDATE clients SET points = $1, borrowed_books = $2, updated_at = $3 WHERE id = $4`
    _, err := r.db.ExecContext(ctx, query, rec.Points, rec.BorrowedBookIDs, rec.UpdatedAt, rec.ID)
    return err
}

func (r *PostgreSQLClientRepository) Save(ctx context.Context, client *entities.Client) error {
    rec := pgEntities.FromDomainClient(client)
    query := `INSERT INTO clients (id, name, email, points, borrowed_books, max_books, created_at, updated_at)
              VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`
    _, err := r.db.ExecContext(ctx, query, rec.ID, rec.Name, rec.Email, rec.Points, rec.BorrowedBookIDs, rec.MaxBooksAllowed, rec.CreatedAt, rec.UpdatedAt)
    return err
}
```
