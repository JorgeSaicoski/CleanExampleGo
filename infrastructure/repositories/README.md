# Infrastructure Repositories (Implementations)

## Purpose

This folder contains **implementations of repository interfaces** defined in the domain layer. Here you write the actual database access code.

## Rules

✅ **DO:**
- Implement repository interfaces from domain layer
- Use database libraries (sql, gorm, mongo, etc.)
- Handle database-specific errors
- Implement connection pooling
- Add logging and monitoring

❌ **DON'T:**
- Define new interfaces here (define in domain)
- Include business logic (that goes in entities/use cases)
- Return database-specific types (return domain entities)
- Expose implementation details to other layers

## Example: Client Repository Implementation (SQL)

```go
// client_repository_impl.go
package repositories

import (
    "database/sql"
    "ExampleProject/domain/entities"
    "ExampleProject/domain/repositories"
    "errors"
)

type ClientRepositoryImpl struct {
    db *sql.DB
}

// NewClientRepository creates a new instance of ClientRepository
func NewClientRepository(db *sql.DB) repositories.ClientRepository {
    return &ClientRepositoryImpl{db: db}
}

func (r *ClientRepositoryImpl) GetByID(id string) (*entities.Client, error) {
    query := `
        SELECT id, name, email, points, max_books_allowed 
        FROM clients 
        WHERE id = ?
    `
    
    var client entities.Client
    err := r.db.QueryRow(query, id).Scan(
        &client.ID,
        &client.Name,
        &client.Email,
        &client.Points,
        &client.MaxBooksAllowed,
    )
    
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, errors.New("client not found")
        }
        return nil, err
    }
    
    // Load borrowed books
    client.BorrowedBooks, err = r.getBorrowedBooks(id)
    if err != nil {
        return nil, err
    }
    
    return &client, nil
}

func (r *ClientRepositoryImpl) GetAll() ([]*entities.Client, error) {
    query := `
        SELECT id, name, email, points, max_books_allowed 
        FROM clients
    `
    
    rows, err := r.db.Query(query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var clients []*entities.Client
    for rows.Next() {
        var client entities.Client
        err := rows.Scan(
            &client.ID,
            &client.Name,
            &client.Email,
            &client.Points,
            &client.MaxBooksAllowed,
        )
        if err != nil {
            return nil, err
        }
        
        // Load borrowed books for each client
        client.BorrowedBooks, _ = r.getBorrowedBooks(client.ID)
        clients = append(clients, &client)
    }
    
    return clients, nil
}

func (r *ClientRepositoryImpl) Save(client *entities.Client) error {
    query := `
        INSERT INTO clients (id, name, email, points, max_books_allowed)
        VALUES (?, ?, ?, ?, ?)
    `
    
    _, err := r.db.Exec(
        query,
        client.ID,
        client.Name,
        client.Email,
        client.Points,
        client.MaxBooksAllowed,
    )
    
    return err
}

func (r *ClientRepositoryImpl) Update(client *entities.Client) error {
    // Start transaction
    tx, err := r.db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Update client data
    query := `
        UPDATE clients 
        SET name = ?, email = ?, points = ?, max_books_allowed = ?
        WHERE id = ?
    `
    
    _, err = tx.Exec(
        query,
        client.Name,
        client.Email,
        client.Points,
        client.MaxBooksAllowed,
        client.ID,
    )
    if err != nil {
        return err
    }
    
    // Update borrowed books
    // First, delete existing entries
    _, err = tx.Exec("DELETE FROM borrowed_books WHERE client_id = ?", client.ID)
    if err != nil {
        return err
    }
    
    // Then insert current ones
    for _, bookID := range client.BorrowedBooks {
        _, err = tx.Exec(
            "INSERT INTO borrowed_books (client_id, book_id) VALUES (?, ?)",
            client.ID,
            bookID,
        )
        if err != nil {
            return err
        }
    }
    
    return tx.Commit()
}

func (r *ClientRepositoryImpl) Delete(id string) error {
    query := `DELETE FROM clients WHERE id = ?`
    _, err := r.db.Exec(query, id)
    return err
}

func (r *ClientRepositoryImpl) GetByEmail(email string) (*entities.Client, error) {
    query := `
        SELECT id, name, email, points, max_books_allowed 
        FROM clients 
        WHERE email = ?
    `
    
    var client entities.Client
    err := r.db.QueryRow(query, email).Scan(
        &client.ID,
        &client.Name,
        &client.Email,
        &client.Points,
        &client.MaxBooksAllowed,
    )
    
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, errors.New("client not found")
        }
        return nil, err
    }
    
    client.BorrowedBooks, _ = r.getBorrowedBooks(client.ID)
    return &client, nil
}

func (r *ClientRepositoryImpl) GetClientsWithOverdueBooks() ([]*entities.Client, error) {
    query := `
        SELECT DISTINCT c.id, c.name, c.email, c.points, c.max_books_allowed
        FROM clients c
        INNER JOIN borrowed_books bb ON c.id = bb.client_id
        INNER JOIN books b ON bb.book_id = b.id
        WHERE b.due_date < NOW()
    `
    
    rows, err := r.db.Query(query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var clients []*entities.Client
    for rows.Next() {
        var client entities.Client
        err := rows.Scan(
            &client.ID,
            &client.Name,
            &client.Email,
            &client.Points,
            &client.MaxBooksAllowed,
        )
        if err != nil {
            continue
        }
        
        client.BorrowedBooks, _ = r.getBorrowedBooks(client.ID)
        clients = append(clients, &client)
    }
    
    return clients, nil
}

func (r *ClientRepositoryImpl) GetTopClientsByPoints(limit int) ([]*entities.Client, error) {
    query := `
        SELECT id, name, email, points, max_books_allowed
        FROM clients
        ORDER BY points DESC
        LIMIT ?
    `
    
    rows, err := r.db.Query(query, limit)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var clients []*entities.Client
    for rows.Next() {
        var client entities.Client
        err := rows.Scan(
            &client.ID,
            &client.Name,
            &client.Email,
            &client.Points,
            &client.MaxBooksAllowed,
        )
        if err != nil {
            continue
        }
        
        client.BorrowedBooks, _ = r.getBorrowedBooks(client.ID)
        clients = append(clients, &client)
    }
    
    return clients, nil
}

// Helper method to get borrowed books
func (r *ClientRepositoryImpl) getBorrowedBooks(clientID string) ([]string, error) {
    query := `SELECT book_id FROM borrowed_books WHERE client_id = ?`
    
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

## Example: Book Repository Implementation

```go
// book_repository_impl.go
package repositories

import (
    "database/sql"
    "ExampleProject/domain/entities"
    "ExampleProject/domain/repositories"
    "errors"
)

type BookRepositoryImpl struct {
    db *sql.DB
}

func NewBookRepository(db *sql.DB) repositories.BookRepository {
    return &BookRepositoryImpl{db: db}
}

func (r *BookRepositoryImpl) GetByID(id string) (*entities.Book, error) {
    query := `
        SELECT id, title, author, isbn, is_available, borrowed_by, due_date
        FROM books
        WHERE id = ?
    `
    
    var book entities.Book
    var borrowedBy sql.NullString
    var dueDate sql.NullTime
    
    err := r.db.QueryRow(query, id).Scan(
        &book.ID,
        &book.Title,
        &book.Author,
        &book.ISBN,
        &book.IsAvailable,
        &borrowedBy,
        &dueDate,
    )
    
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, errors.New("book not found")
        }
        return nil, err
    }
    
    if borrowedBy.Valid {
        book.BorrowedBy = borrowedBy.String
    }
    if dueDate.Valid {
        book.DueDate = &dueDate.Time
    }
    
    return &book, nil
}

func (r *BookRepositoryImpl) Save(book *entities.Book) error {
    query := `
        INSERT INTO books (id, title, author, isbn, is_available, borrowed_by, due_date)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    `
    
    _, err := r.db.Exec(
        query,
        book.ID,
        book.Title,
        book.Author,
        book.ISBN,
        book.IsAvailable,
        book.BorrowedBy,
        book.DueDate,
    )
    
    return err
}

func (r *BookRepositoryImpl) Update(book *entities.Book) error {
    query := `
        UPDATE books
        SET title = ?, author = ?, isbn = ?, is_available = ?, 
            borrowed_by = ?, due_date = ?
        WHERE id = ?
    `
    
    _, err := r.db.Exec(
        query,
        book.Title,
        book.Author,
        book.ISBN,
        book.IsAvailable,
        book.BorrowedBy,
        book.DueDate,
        book.ID,
    )
    
    return err
}

func (r *BookRepositoryImpl) FindAvailableBooks() ([]*entities.Book, error) {
    query := `
        SELECT id, title, author, isbn, is_available, borrowed_by, due_date
        FROM books
        WHERE is_available = true
        ORDER BY title
    `
    
    return r.queryBooks(query)
}

func (r *BookRepositoryImpl) FindByBorrower(clientID string) ([]*entities.Book, error) {
    query := `
        SELECT id, title, author, isbn, is_available, borrowed_by, due_date
        FROM books
        WHERE borrowed_by = ?
    `
    
    return r.queryBooks(query, clientID)
}

func (r *BookRepositoryImpl) FindOverdueBooks() ([]*entities.Book, error) {
    query := `
        SELECT id, title, author, isbn, is_available, borrowed_by, due_date
        FROM books
        WHERE is_available = false AND due_date < NOW()
    `
    
    return r.queryBooks(query)
}

func (r *BookRepositoryImpl) SearchByTitle(searchQuery string) ([]*entities.Book, error) {
    query := `
        SELECT id, title, author, isbn, is_available, borrowed_by, due_date
        FROM books
        WHERE title LIKE ?
        ORDER BY title
    `
    
    return r.queryBooks(query, "%"+searchQuery+"%")
}

// Helper method
func (r *BookRepositoryImpl) queryBooks(query string, args ...interface{}) ([]*entities.Book, error) {
    rows, err := r.db.Query(query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var books []*entities.Book
    for rows.Next() {
        var book entities.Book
        var borrowedBy sql.NullString
        var dueDate sql.NullTime
        
        err := rows.Scan(
            &book.ID,
            &book.Title,
            &book.Author,
            &book.ISBN,
            &book.IsAvailable,
            &borrowedBy,
            &dueDate,
        )
        
        if err != nil {
            continue
        }
        
        if borrowedBy.Valid {
            book.BorrowedBy = borrowedBy.String
        }
        if dueDate.Valid {
            book.DueDate = &dueDate.Time
        }
        
        books = append(books, &book)
    }
    
    return books, nil
}
```

## Database Schema Example

```sql
-- clients table
CREATE TABLE clients (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    points INT DEFAULT 0,
    max_books_allowed INT DEFAULT 3,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- books table
CREATE TABLE books (
    id VARCHAR(36) PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    author VARCHAR(255) NOT NULL,
    isbn VARCHAR(20) UNIQUE NOT NULL,
    is_available BOOLEAN DEFAULT true,
    borrowed_by VARCHAR(36),
    due_date TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (borrowed_by) REFERENCES clients(id) ON DELETE SET NULL
);

-- borrowed_books junction table
CREATE TABLE borrowed_books (
    client_id VARCHAR(36),
    book_id VARCHAR(36),
    borrowed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (client_id, book_id),
    FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
    FOREIGN KEY (book_id) REFERENCES books(id) ON DELETE CASCADE
);

-- indexes
CREATE INDEX idx_clients_email ON clients(email);
CREATE INDEX idx_books_isbn ON books(isbn);
CREATE INDEX idx_books_available ON books(is_available);
CREATE INDEX idx_books_borrowed_by ON books(borrowed_by);
```

## Best Practices

1. **Connection Pooling**: Reuse database connections
2. **Transactions**: Use transactions for multi-step operations
3. **Error Handling**: Convert database errors to domain errors
4. **SQL Injection**: Always use parameterized queries
5. **NULL Handling**: Use sql.Null* types for nullable columns
6. **Logging**: Log queries for debugging (but not in production)
7. **Caching**: Consider caching frequently accessed data

---

**Implement your repository interfaces here!**

