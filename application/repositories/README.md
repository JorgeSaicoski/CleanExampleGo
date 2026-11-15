# Application Repositories (Outbound Ports)

## Purpose

Repository interfaces define the **contracts for data access** that the application layer requires. 
They are defined here in the **application layer** (Outbound Ports) and implemented in the **infrastructure layer** (Driven Adapters).

**Key Architectural Rule**: Repository interfaces accept and return **Domain Entities** (`domain/entities`), never infrastructure records or delivery DTOs.

---

## Rules

✅ **DO:**
- Define interfaces in terms of `domain/entities`.
- Include `context.Context` in every method for production resilience (timeouts, cancellation, tracing).
- Keep interfaces focused (Interface Segregation Principle).
- Return domain entities or `nil` with an error when not found.

❌ **DON'T:**
- Use database-specific types (e.g. `*sql.DB`, `sql.NullString`, BSON types).
- Return infrastructure-specific records (e.g. `ClientRecord`).
- Introduce redundant Application DTOs when Domain Entities already model the concept.
- Implement any data access or SQL logic here.

---

## Architectural Flow

```
Use Case (application/usecases)
    ↓ depends on
Repository Interface (application/repositories)  [Contract]
    ↑ implements
PostgreSQL Repository (infrastructure/postgresql/repositories)  [Adapter]
```

---

## Example: Client Repository Interface

```go
// application/repositories/client_repository.go
package repositories

import (
    "context"
    "ExampleProject/domain/entities"
)

// ClientRepository defines the contract for client persistence
type ClientRepository interface {
    GetByID(ctx context.Context, id string) (*entities.Client, error)
    GetByEmail(ctx context.Context, email string) (*entities.Client, error)
    Save(ctx context.Context, client *entities.Client) error
    Update(ctx context.Context, client *entities.Client) error
    Delete(ctx context.Context, id string) error
}
```

## Example: Book Repository Interface

```go
// application/repositories/book_repository.go
package repositories

import (
    "context"
    "ExampleProject/domain/entities"
)

// BookRepository defines the contract for book persistence
type BookRepository interface {
    GetByID(ctx context.Context, id string) (*entities.Book, error)
    FindByISBN(ctx context.Context, isbn string) (*entities.Book, error)
    FindAvailableBooks(ctx context.Context) ([]*entities.Book, error)
    Save(ctx context.Context, book *entities.Book) error
    Update(ctx context.Context, book *entities.Book) error
    Delete(ctx context.Context, id string) error
}
```

---

## Why Domain Entities in Repository Interfaces?

### ✅ Correct: Domain Entities
```go
type ClientRepository interface {
    GetByID(ctx context.Context, id string) (*entities.Client, error)
}
```
- Preserves business rules and entity invariants on the retrieved object.
- Avoids extra boilerplate mapping between DTOs and entities inside use cases.

### ❌ Anti-pattern: Infrastructure Records
```go
type ClientRepository interface {
    GetByID(ctx context.Context, id string) (*ClientRecord, error) // LEAKS DB CONCERNS
}
```

---

## Testing with Mocks

Because repository interfaces are defined in the application layer, use cases are trivially unit-tested with mocks:

```go
// application/usecases/rent_book_test.go
package usecases_test

import (
    "context"
    "ExampleProject/domain/entities"
)

type MockClientRepository struct {
    client *entities.Client
    err    error
}

func (m *MockClientRepository) GetByID(ctx context.Context, id string) (*entities.Client, error) {
    return m.client, m.err
}

func (m *MockClientRepository) Update(ctx context.Context, client *entities.Client) error {
    m.client = client
    return m.err
}

func (m *MockClientRepository) Save(ctx context.Context, client *entities.Client) error {
    m.client = client
    return m.err
}

func (m *MockClientRepository) Delete(ctx context.Context, id string) error {
    return m.err
}

func (m *MockClientRepository) GetByEmail(ctx context.Context, email string) (*entities.Client, error) {
    return m.client, m.err
}
```
