# Domain Repositories (Interfaces)

## ⚠️ IMPORTANT: This folder is for reference only

**Repository interfaces should be defined in `application/repositories/`, not here.**

## Why Repository Interfaces Belong in the Application Layer

Repository interfaces are **application concerns**, not domain concerns:

### Application Layer Owns Data Access Contracts
- **Use cases** (in application layer) need to retrieve and persist data
- **Repository interfaces** define what data operations the application needs
- **Application DTOs** flow through repositories as data contracts
- **Infrastructure** implements these interfaces using specific technologies

### Domain Layer is for Business Logic
- **Domain entities** contain business rules and behavior
- **Value objects** represent domain concepts
- **Domain services** contain complex business logic
- Domain should be **pure** - no dependencies on data access concerns

### Following the Dependency Rule

```
Application Layer (Use Cases)
    ↓ depends on
Repository Interfaces (Application Layer) ← Defined here for use cases
    ↑ implements
Infrastructure Layer (PostgreSQL, MongoDB, etc.)
```

The application defines **what it needs**, infrastructure provides **how to get it**.

## Architecture Principle

**"Application defines, infrastructure adapts"**

- Application layer says: "I need GetByID(id) returning ClientDTO"
- Infrastructure layer says: "I'll implement that using PostgreSQL/MongoDB/etc."

This keeps the application independent of infrastructure details.

## For Reference: Example Repository Pattern

Repository interfaces define **contracts for data access** without specifying how the data is stored or retrieved. They are defined in the **application layer** and implemented in the infrastructure layer.

### Current Architecture
- **Interfaces**: `application/repositories/` - Defines contracts using application DTOs
- **Implementations**: `infrastructure/postgresql/repositories/` - Implements using PostgreSQL
- **Domain**: `domain/entities/` - Contains only business entities (no repository interfaces)

## Example Patterns (For Reference Only)

**Note:** These examples show the repository pattern concept, but remember that interfaces should be defined in `application/repositories/` using application DTOs, not domain entities.

## Rules (When Defining in Application Layer)

✅ **DO:**
- Define interfaces in `application/repositories/`
- Use **application DTOs** as parameters and return types
- Keep interfaces focused (Interface Segregation Principle)
- Use meaningful method names
- Document expected behavior

❌ **DON'T:**
- Define repository interfaces in domain layer
- Implement data access logic in interface definitions
- Use database-specific types (like sql.Rows)
- Include HTTP or framework code
- Use domain entities in repository signatures (use DTOs instead)

## Example: Client Repository Interface (Reference Pattern)

**Note:** This is a reference example. In actual implementation, define this interface in `application/repositories/` using `*dto.ClientDTO` instead of `*entities.Client`.

```go
// Reference pattern - DO NOT implement this in domain/repositories/
// Actual implementation should be in: application/repositories/client_repository.go
package repositories

import "ExampleProject/application/dto"

// ClientRepository defines the contract for client data access
type ClientRepository interface {
    // Basic CRUD operations
    GetByID(id string) (*dto.ClientDTO, error)
    GetAll() ([]*dto.ClientDTO, error)
    Save(client *dto.ClientDTO) error
    Update(client *dto.ClientDTO) error
    Delete(id string) error
    
    // Custom query methods
    GetByEmail(email string) (*dto.ClientDTO, error)
    GetClientsWithOverdueBooks() ([]*dto.ClientDTO, error)
    GetTopClientsByPoints(limit int) ([]*dto.ClientDTO, error)
}
```

## Example: Book Repository Interface (Reference Pattern)

**Note:** This is a reference example. In actual implementation, define this interface in `application/repositories/` using `*dto.BookDTO`.

```go
// Reference pattern - DO NOT implement this in domain/repositories/
// Actual implementation should be in: application/repositories/book_repository.go
package repositories

import "ExampleProject/application/dto"

// BookRepository defines the contract for book data access
type BookRepository interface {
    // Basic CRUD operations
    GetByID(id string) (*dto.BookDTO, error)
    GetAll() ([]*dto.BookDTO, error)
    Save(book *dto.BookDTO) error
    Update(book *dto.BookDTO) error
    Delete(id string) error
    
    // Custom query methods
    FindByISBN(isbn string) (*dto.BookDTO, error)
    FindAvailableBooks() ([]*dto.BookDTO, error)
    FindByBorrower(clientID string) ([]*dto.BookDTO, error)
    FindOverdueBooks() ([]*dto.BookDTO, error)
    SearchByTitle(query string) ([]*dto.BookDTO, error)
    SearchByAuthor(author string) ([]*dto.BookDTO, error)
}
```

## Example: Promotion Repository Interface (Reference Pattern)

**Note:** This is a reference example. In actual implementation, define this interface in `application/repositories/` using `*dto.PromotionDTO`.

```go
// Reference pattern - DO NOT implement this in domain/repositories/
// Actual implementation should be in: application/repositories/promotion_repository.go
package repositories

import (
    "ExampleProject/application/dto"
    "time"
)

// PromotionRepository defines the contract for promotion data access
type PromotionRepository interface {
    GetByID(id string) (*dto.PromotionDTO, error)
    GetActivePromotions(currentTime time.Time) ([]*dto.PromotionDTO, error)
    GetPromotionsByClient(clientID string) ([]*dto.PromotionDTO, error)
    Save(promotion *dto.PromotionDTO) error
    Update(promotion *dto.PromotionDTO) error
    Delete(id string) error
}
```

## Why Repository Interfaces in Application Layer?

**Dependency Inversion Principle**: 
- High-level modules (use cases) should not depend on low-level modules (infrastructure)
- Both should depend on abstractions (interfaces)
- By defining interfaces in the **application layer**, the application defines its needs
- Infrastructure adapts to fulfill those needs

```
Use Case (Application) ──depends on──> Repository Interface (Application)
                                                    ▲
                                                    │ implements
                                Infrastructure Repository
```

This is better than putting interfaces in domain because:
- **Use cases are in application layer** - they need repository interfaces
- **DTOs are in application layer** - repositories use DTOs
- **Domain entities** are optional - not all apps need rich domain models
- **Application defines contracts** - infrastructure adapts to them

## Method Naming Conventions

### Read Operations
- `Get{Entity}ByID` - Get single entity by ID
- `Get{Entity}By{Field}` - Get single entity by field
- `GetAll{Entities}` - Get all entities
- `Find{Entities}By{Criteria}` - Find multiple entities
- `Search{Entities}` - Search with query
- `Count{Entities}` - Count entities

### Write Operations
- `Save{Entity}` - Create new entity
- `Update{Entity}` - Update existing entity
- `Delete{Entity}` - Delete entity
- `DeleteBy{Field}` - Delete by criteria

### Example
```go
// Reference - Actual implementation in application/repositories/
package repositories

import "ExampleProject/application/dto"

type UserRepository interface {
    GetUserByID(id string) (*dto.UserDTO, error)
    GetUserByEmail(email string) (*dto.UserDTO, error)
    GetAllUsers() ([]*dto.UserDTO, error)
    FindActiveUsers() ([]*dto.UserDTO, error)
    SearchUsersByName(query string) ([]*dto.UserDTO, error)
    CountUsers() (int, error)
    SaveUser(user *dto.UserDTO) error
    UpdateUser(user *dto.UserDTO) error
    DeleteUser(id string) error
}
```

## Return Types

### Single Entity
```go
GetByID(id string) (*dto.ClientDTO, error)
```
- Return pointer to DTO or nil if not found
- Return error for actual errors (DB connection, etc.)

### Multiple Entities
```go
GetAll() ([]*dto.ClientDTO, error)
```
- Return slice of DTO pointers
- Return empty slice if no results (not error)
- Return error only for actual errors

### Boolean Results
```go
Exists(id string) (bool, error)
```
- Return true/false for existence
- Return error for actual errors

## Context Support

For production code, include context:

```go
import "ExampleProject/application/dto"

type ClientRepository interface {
    GetByID(ctx context.Context, id string) (*dto.ClientDTO, error)
    Save(ctx context.Context, client *dto.ClientDTO) error
    // ... other methods
}
```

This allows:
- Request cancellation
- Timeout handling
- Tracing
- Request-scoped values

## Testing with Repositories

Repositories make testing easy - create mocks:

```go
// mock_client_repository.go (in tests)
import "ExampleProject/application/dto"

type MockClientRepository struct {
    Clients map[string]*dto.ClientDTO
    Err     error
}

func (m *MockClientRepository) GetByID(id string) (*dto.ClientDTO, error) {
    if m.Err != nil {
        return nil, m.Err
    }
    return m.Clients[id], nil
}

func (m *MockClientRepository) Save(client *dto.ClientDTO) error {
    if m.Err != nil {
        return m.Err
    }
    m.Clients[client.ID] = client
    return nil
}

// ... implement other methods
```

## When to Create a Repository

Create a repository when:
- You have an entity that needs persistence
- You need to query/search the entity
- You want to abstract data access from business logic
- You need multiple storage implementations (SQL, NoSQL, cache)

## Common Repository Patterns

### Generic Repository (Not Recommended)
```go
// Too generic - loses type safety
type Repository[T any] interface {
    GetByID(id string) (T, error)
    GetAll() ([]T, error)
}
```

### Specific Repository (Recommended)
```go
// Type-safe and application-specific
import "ExampleProject/application/dto"

type ClientRepository interface {
    GetByID(id string) (*dto.ClientDTO, error)
    GetClientsWithOverdueBooks() ([]*dto.ClientDTO, error) // Application-specific
}
```

---

**Remember**: Repository interfaces belong in `application/repositories/`, not here!
Use application DTOs in repository method signatures.

