# PostgreSQL Database Implementation

## Purpose

This folder contains the **PostgreSQL database implementation** with proper dependency injection, entities, DTOs, and repository implementations.

## Structure

```
postgresql/
├── entities/                   ← Database-specific entity mappings (internal)
│   ├── client_record.go       ← Database record for clients
│   └── book_record.go         ← Database record for books
├── repositories/              ← Repository implementations
│   ├── client_repository.go   ← Implements application/repositories interfaces
│   └── book_repository.go     ← Implements application/repositories interfaces
├── migrations/                ← Database migrations
│   ├── 001_create_clients.sql
│   ├── 002_create_books.sql
│   └── 003_create_borrowed_books.sql
└── connection.go              ← Database connection management
```

**Note**: DTOs and repository interfaces are defined in the `application/` layer, not here.
The infrastructure adapts to what the application expects.

## Architecture: Application Defines, Infrastructure Adapts

### The Dependency Rule

**Application Layer** (high-level) defines:
- Repository **interfaces** (`application/repositories/`)
- **DTOs** that describe what it expects (`application/dto/`)
- Business use cases

**Infrastructure Layer** (low-level) implements:
- Repository interfaces using PostgreSQL
- **Internal entities** for database mapping (`postgresql/entities/`)
- Conversion from database records to application DTOs

### Why This Matters

```
Application says: "I need GetByID(userID) returning ClientDTO"
Infrastructure says: "I'll query PostgreSQL and convert the result to ClientDTO"
```

This follows the **Dependency Inversion Principle**:
- High-level modules don't depend on low-level modules
- Both depend on abstractions (interfaces)
- The application doesn't know about PostgreSQL, only about its interfaces and DTOs

### Layer Responsibilities

**Application DTOs** (`application/dto/`):
- Define the data contracts
- Used by use cases and handlers
- No database or external service concerns

**Infrastructure Entities** (`postgresql/entities/`):
- Internal to the infrastructure layer
- SQL-specific types (sql.NullString, time.Time, etc.)
- Conversion methods to/from application DTOs

This separation allows:
- Changing database without affecting application layer
- Multiple infrastructure implementations (PostgreSQL, MongoDB, In-Memory)
- Better testability with mocks

## Entities

### Client Record
```go
// entities/client_record.go
package entities

import (
    "time"
    "ExampleProject/application/dto"
)

// ClientRecord is the database representation of a client
// Internal to the infrastructure layer
type ClientRecord struct {
    ID              string
    Name            string
    Email           string
    Points          int
    MaxBooksAllowed int
    CreatedAt       time.Time
    UpdatedAt       time.Time
}

// ToDTO converts database record to application DTO
func (r *ClientRecord) ToDTO() *dto.ClientDTO {
    return &dto.ClientDTO{
        ID:              r.ID,
        Name:            r.Name,
        Email:           r.Email,
        Points:          r.Points,
        MaxBooksAllowed: r.MaxBooksAllowed,
        BorrowedBookIDs: []string{}, // Loaded separately
    }
}

// FromDTO converts application DTO to database record
func FromClientDTO(client *dto.ClientDTO) *ClientRecord {
    return &ClientRecord{
        ID:              client.ID,
        Name:            client.Name,
        Email:           client.Email,
        Points:          client.Points,
        MaxBooksAllowed: client.MaxBooksAllowed,
        UpdatedAt:       time.Now(),
    }
}
```

### Book Record
```go
// entities/book_record.go
package entities

import (
    "database/sql"
    "time"
    "ExampleProject/application/dto"
)

// BookRecord is the database representation of a book
// Internal to the infrastructure layer
type BookRecord struct {
    ID          string
    Title       string
    Author      string
    ISBN        string
    IsAvailable bool
    BorrowedBy  sql.NullString
    DueDate     sql.NullTime
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

// ToDTO converts database record to application DTO
func (r *BookRecord) ToDTO() *dto.BookDTO {
    book := &dto.BookDTO{
        ID:          r.ID,
        Title:       r.Title,
        Author:      r.Author,
        ISBN:        r.ISBN,
        IsAvailable: r.IsAvailable,
    }
    
    if r.BorrowedBy.Valid {
        book.BorrowedBy = r.BorrowedBy.String
    }
    
    if r.DueDate.Valid {
        book.DueDate = &r.DueDate.Time
    }
    
    return book
}

// FromDTO converts application DTO to database record
func FromBookDTO(book *dto.BookDTO) *BookRecord {
    record := &BookRecord{
        ID:          book.ID,
        Title:       book.Title,
        Author:      book.Author,
        ISBN:        book.ISBN,
        IsAvailable: book.IsAvailable,
        UpdatedAt:   time.Now(),
    }
    
    if book.BorrowedBy != "" {
        record.BorrowedBy = sql.NullString{
            String: book.BorrowedBy,
            Valid:  true,
        }
    }
    
    if book.DueDate != nil {
        record.DueDate = sql.NullTime{
            Time:  *book.DueDate,
            Valid: true,
        }
    }
    
    return record
}
```

## Repository Implementation

```go
// repositories/client_repository.go
package repositories

import (
    "database/sql"
    "fmt"
    "ExampleProject/application/dto"
    "ExampleProject/application/repositories"
    pgEntities "ExampleProject/infrastructure/postgresql/entities"
)

type PostgreSQLClientRepository struct {
    db *sql.DB
}

// NewPostgreSQLClientRepository creates repository with dependency injection
func NewPostgreSQLClientRepository(db *sql.DB) repositories.ClientRepository {
    return &PostgreSQLClientRepository{db: db}
}

// GetByID implements the application repository interface
// Returns application DTO, not domain entity
func (r *PostgreSQLClientRepository) GetByID(id string) (*dto.ClientDTO, error) {
    query := `
        SELECT id, name, email, points, max_books_allowed, created_at, updated_at
        FROM clients
        WHERE id = $1
    `
    
    var record pgEntities.ClientRecord
    err := r.db.QueryRow(query, id).Scan(
        &record.ID,
        &record.Name,
        &record.Email,
        &record.Points,
        &record.MaxBooksAllowed,
        &record.CreatedAt,
        &record.UpdatedAt,
    )
    
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("client not found")
        }
        return nil, err
    }
    
    // Convert database record to application DTO
    clientDTO := record.ToDTO()
    
    // Load borrowed book IDs
    clientDTO.BorrowedBookIDs, err = r.getBorrowedBookIDs(id)
    if err != nil {
        return nil, err
    }
    
    return clientDTO, nil
}

func (r *PostgreSQLClientRepository) Save(client *dto.ClientDTO) error {
    // Convert application DTO to database record
    record := pgEntities.FromClientDTO(client)
    
    query := `
        INSERT INTO clients (id, name, email, points, max_books_allowed, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, NOW(), NOW())
    `
    
    _, err := r.db.Exec(
        query,
        record.ID,
        record.Name,
        record.Email,
        record.Points,
        record.MaxBooksAllowed,
    )
    
    return err
}

func (r *PostgreSQLClientRepository) Update(client *dto.ClientDTO) error {
    tx, err := r.db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Convert application DTO to database record
    record := pgEntities.FromClientDTO(client)
    
    // Update client
    query := `
        UPDATE clients
        SET name = $1, email = $2, points = $3, max_books_allowed = $4, updated_at = NOW()
        WHERE id = $5
    `
    
    _, err = tx.Exec(query, record.Name, record.Email, record.Points, record.MaxBooksAllowed, record.ID)
    if err != nil {
        return err
    }
    
    // Update borrowed books
    _, err = tx.Exec("DELETE FROM borrowed_books WHERE client_id = $1", client.ID)
    if err != nil {
        return err
    }
    
    for _, bookID := range client.BorrowedBookIDs {
        _, err = tx.Exec(
            "INSERT INTO borrowed_books (client_id, book_id) VALUES ($1, $2)",
            client.ID,
            bookID,
        )
        if err != nil {
            return err
        }
    }
    
    return tx.Commit()
}

func (r *PostgreSQLClientRepository) getBorrowedBookIDs(clientID string) ([]string, error) {
    query := `SELECT book_id FROM borrowed_books WHERE client_id = $1`
    
    rows, err := r.db.Query(query, clientID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var bookIDs []string
    for rows.Next() {
        var bookID string
        if err := rows.Scan(&bookID); err != nil {
            continue
        }
        bookIDs = append(bookIDs, bookID)
    }
    
    return bookIDs, nil
}
```

## Connection Management

```go
// connection.go
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
        config.Host,
        config.Port,
        config.User,
        config.Password,
        config.Database,
        config.SSLMode,
    )
    
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }
    
    // Configure connection pool
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    db.SetConnMaxIdleTime(10 * time.Minute)
    
    // Test connection
    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }
    
    return db, nil
}
```

## Migrations

```sql
-- migrations/001_create_clients.sql
CREATE TABLE clients (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    points INT DEFAULT 0,
    max_books_allowed INT DEFAULT 3,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clients_email ON clients(email);

-- migrations/002_create_books.sql
CREATE TABLE books (
    id VARCHAR(36) PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    author VARCHAR(255) NOT NULL,
    isbn VARCHAR(20) UNIQUE NOT NULL,
    is_available BOOLEAN DEFAULT true,
    borrowed_by VARCHAR(36),
    due_date TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (borrowed_by) REFERENCES clients(id) ON DELETE SET NULL
);

CREATE INDEX idx_books_isbn ON books(isbn);
CREATE INDEX idx_books_available ON books(is_available);
CREATE INDEX idx_books_borrowed_by ON books(borrowed_by);

-- migrations/003_create_borrowed_books.sql
CREATE TABLE borrowed_books (
    client_id VARCHAR(36),
    book_id VARCHAR(36),
    borrowed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (client_id, book_id),
    FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
    FOREIGN KEY (book_id) REFERENCES books(id) ON DELETE CASCADE
);
```

## Wiring with Dependency Injection

```go
// cmd/api/main.go
package main

import (
    "log"
    "ExampleProject/infrastructure/postgresql"
    pgRepos "ExampleProject/infrastructure/postgresql/repositories"
    "ExampleProject/application/usecases"
)

func main() {
    // Initialize database connection
    dbConfig := postgresql.Config{
        Host:     "localhost",
        Port:     5432,
        User:     "library_user",
        Password: "secret",
        Database: "library_db",
        SSLMode:  "disable",
    }
    
    db, err := postgresql.NewConnection(dbConfig)
    if err != nil {
        log.Fatal("Failed to connect to database:", err)
    }
    defer db.Close()
    
    // Create repository implementations (Dependency Injection)
    clientRepo := pgRepos.NewPostgreSQLClientRepository(db)
    bookRepo := pgRepos.NewPostgreSQLBookRepository(db)
    
    // Inject repositories into use cases
    rentBookUseCase := usecases.NewRentBookUseCase(
        clientRepo,    // Injected dependency
        bookRepo,      // Injected dependency
        promotionSvc,
        pointsService,
    )
    
    // ... rest of application setup
}
```

## Benefits of This Structure

1. **Dependency Injection**: Repositories injected via constructors
2. **Separation**: Database concerns separated from domain
3. **Type Safety**: Explicit conversion between layers
4. **Testability**: Easy to swap with mock implementations
5. **Maintainability**: Changes to database schema isolated
6. **Flexibility**: Can switch from PostgreSQL to MySQL by creating new folder

## Testing

```go
// repositories/client_repository_test.go
func TestClientRepository_GetByID(t *testing.T) {
    db := setupTestDB(t)
    defer db.Close()
    
    repo := NewPostgreSQLClientRepository(db)
    
    // Insert test data
    insertTestClient(db, "test-id", "John Doe", "john@example.com")
    
    // Test
    client, err := repo.GetByID("test-id")
    
    assert.NoError(t, err)
    assert.Equal(t, "John Doe", client.Name)
}
```

---

**Implement your PostgreSQL repository following this pattern with proper dependency injection!**

