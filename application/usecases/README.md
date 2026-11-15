# Application Use Cases

## Purpose

Use cases represent **application-specific business workflows**. They orchestrate domain entities, repository contracts, and external services to fulfill a specific user or system action.

**Key Architectural Rule**: Use cases operate directly on **Domain Entities** (`domain/entities`) and invoke domain methods to enforce invariants.

---

## Rules

✅ **DO:**
- Implement one use case per file/struct.
- Accept dependencies via constructor injection (Dependency Injection).
- Accept primitive parameters or command structs from driving adapters.
- Call repository and service interfaces defined in `application/`.
- Delegate business rule enforcement to domain entities (e.g. `client.CanBorrow()`, `book.Rent()`).
- Return domain entities, primitive results, or domain errors.

❌ **DON'T:**
- Access databases or external HTTP clients directly.
- Include HTTP, CLI, or framework logic.
- Duplicate domain invariant checks procedurally when the entity can encapsulate them.
- Depend on infrastructure packages.

---

## Example: Rent Book Use Case

```go
// application/usecases/rent_book.go
package usecases

import (
    "context"
    "errors"
    "time"
    "ExampleProject/application/repositories"
    "ExampleProject/application/services"
)

type RentBookUseCase struct {
    clientRepo    repositories.ClientRepository
    bookRepo      repositories.BookRepository
    promotionSvc  services.PromotionService
    pointsService services.PointsService
}

func NewRentBookUseCase(
    clientRepo repositories.ClientRepository,
    bookRepo repositories.BookRepository,
    promotionSvc services.PromotionService,
    pointsService services.PointsService,
) *RentBookUseCase {
    return &RentBookUseCase{
        clientRepo:    clientRepo,
        bookRepo:      bookRepo,
        promotionSvc:  promotionSvc,
        pointsService: pointsService,
    }
}

func (uc *RentBookUseCase) Execute(ctx context.Context, clientID, bookID string, days int) error {
    if days <= 0 || days > 30 {
        return errors.New("duration must be between 1 and 30 days")
    }

    // 1. Fetch domain entities
    client, err := uc.clientRepo.GetByID(ctx, clientID)
    if err != nil {
        return err
    }

    book, err := uc.bookRepo.GetByID(ctx, bookID)
    if err != nil {
        return err
    }

    // 2. Enforce domain business rules on entities
    if !client.CanBorrow() {
        return errors.New("client borrowing limit reached")
    }

    if !book.IsAvailable {
        return errors.New("book is currently rented")
    }

    // 3. Mutate entity state through entity methods
    if err := client.BorrowBook(bookID); err != nil {
        return err
    }
    book.Rent(clientID, time.Duration(days)*24*time.Hour)

    // 4. Orchestrate optional external services
    hasPromo, _ := uc.promotionSvc.HasActivePromotion(ctx, clientID)
    if hasPromo {
        _ = uc.pointsService.AddPoints(ctx, clientID, 10)
    }

    // 5. Persist updated domain entities
    if err := uc.clientRepo.Update(ctx, client); err != nil {
        return err
    }
    return uc.bookRepo.Update(ctx, book)
}
```

---

## Testing Use Cases

Use cases are unit-tested by providing mock implementations of the repository and service interfaces:

```go
// application/usecases/rent_book_test.go
package usecases_test

import (
    "context"
    "testing"
    "ExampleProject/application/usecases"
    "ExampleProject/domain/entities"
)

func TestRentBook_Success(t *testing.T) {
    client := &entities.Client{
        ID:              "client-1",
        MaxBooksAllowed: 3,
        Points:          10,
    }
    book := &entities.Book{
        ID:          "book-1",
        IsAvailable: true,
    }

    mockClientRepo := &MockClientRepository{client: client}
    mockBookRepo := &MockBookRepository{book: book}
    mockPromo := &MockPromotionService{hasPromo: false}
    mockPoints := &MockPointsService{}

    uc := usecases.NewRentBookUseCase(mockClientRepo, mockBookRepo, mockPromo, mockPoints)

    err := uc.Execute(context.Background(), "client-1", "book-1", 7)
    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }

    if book.IsAvailable {
        t.Error("expected book to be marked unavailable")
    }
    if len(client.BorrowedBookIDs) != 1 {
        t.Errorf("expected client to have 1 borrowed book, got %d", len(client.BorrowedBookIDs))
    }
}
```
