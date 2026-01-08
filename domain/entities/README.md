# Domain Entities
Place your entity files here following these patterns!

---

## Purpose

Entities are the core business objects of your application. They encapsulate business data, rules, and behavior. Entities should be independent of frameworks, databases, and external services.

## When to Create an Entity

Create an entity when you have:
- A distinct business concept with identity (e.g., Client, Book, Order)
- Data that changes over time
- Business rules specific to that concept
- Something that needs to be persisted

## Key Principles

1. **Encapsulation**: Keep business rules and logic inside the entity.
2. **Validation**: Entities should validate their own state.
3. **Immutability**: Prefer immutable entities when possible.
4. **Rich Domain Model**: Entities should have behavior, not just data.
5. **No Dependencies**: Entities should not depend on external packages, frameworks, or infrastructure.

## Do's and Don'ts

✅ **DO:**
- Use value objects when appropriate
- Include business logic that belongs to the entity
- Add business validation methods
- Keep entities pure (no external dependencies)

❌ **DON'T:**
- Include framework-specific code
- Access external services
- Import HTTP or database libraries

## Example: Book Entity (Go)

```go
package entities

import (
    "errors"
    "time"
)

type Book struct {
    ID          string
    Title       string
    Author      string
    ISBN        string
    IsAvailable bool
    BorrowedBy  string
    DueDate     *time.Time
}

func (b *Book) Validate() error {
    if b.Title == "" {
        return errors.New("title is required")
    }
    if b.Author == "" {
        return errors.New("author is required")
    }
    if b.ISBN == "" {
        return errors.New("ISBN is required")
    }
    return nil
}

func (b *Book) CanBeBorrowed() bool {
    return b.IsAvailable
}

func (b *Book) Borrow(clientID string, durationDays int) error {
    if !b.CanBeBorrowed() {
        return errors.New("book is not available")
    }
    b.IsAvailable = false
    b.BorrowedBy = clientID
    dueDate := time.Now().AddDate(0, 0, durationDays)
    b.DueDate = &dueDate
    return nil
}

func (b *Book) Return() error {
    if b.IsAvailable {
        return errors.New("book is not currently borrowed")
    }
    b.IsAvailable = true
    b.BorrowedBy = ""
    b.DueDate = nil
    return nil
}

func (b *Book) IsOverdue() bool {
    if b.DueDate == nil {
        return false
    }
    return time.Now().After(*b.DueDate)
}

func (b *Book) DaysOverdue() int {
    if !b.IsOverdue() {
        return 0
    }
    return int(time.Now().Sub(*b.DueDate).Hours() / 24)
}
```

## Testing Entities

- Write unit tests for entity methods and business rules.
- Test validation, business actions, and edge cases.

---

By following these guidelines, you ensure your domain entities are robust, maintainable, and independent of infrastructure concerns.
