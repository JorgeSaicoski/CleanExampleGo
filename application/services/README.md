# Application Services (Outbound Ports)

## Purpose

Service interfaces represent **contracts for third-party systems or external microservices** (e.g. payment gateways, loyalty APIs, fraud checks, notification providers). 
They are defined here in the **application layer** (Outbound Ports) and implemented in the **infrastructure layer** (Driven Adapters).

**Key Architectural Rule**: Service contracts use domain entities or domain primitives. They never import infrastructure libraries or delivery DTOs.

---

## Rules

✅ **DO:**
- Include `context.Context` in every method.
- Return domain types or domain errors (e.g. `ErrServiceUnavailable`), never raw HTTP status codes.
- Keep interfaces cohesive and narrow (Interface Segregation Principle).

❌ **DON'T:**
- Import external vendor SDKs (e.g. AWS SDK, Resend, Sendgrid, Stripe) in this package. Vendor dependencies belong exclusively in `infrastructure/`.
- Use HTTP request/response types.

---

## Example: Points Service

```go
// application/services/points_service.go
package services

import "context"

type PointsService interface {
    GetClientPoints(ctx context.Context, clientID string) (int, error)
    AddPoints(ctx context.Context, clientID string, points int) error
    ConsumePoints(ctx context.Context, clientID string, points int) error
}
```

---

## Example: Promotion Service

```go
// application/services/promotion_service.go
package services

import "context"

type PromotionService interface {
    HasActivePromotion(ctx context.Context, clientID string) (bool, error)
    GetClientPromotions(ctx context.Context, clientID string) ([]string, error)
}
```

---

## Testing with Mocks

```go
package services_test

import "context"

type MockPromotionService struct {
    HasPromotionResult bool
    PromotionsList     []string
    Err                error
}

func (m *MockPromotionService) HasActivePromotion(ctx context.Context, clientID string) (bool, error) {
    return m.HasPromotionResult, m.Err
}

func (m *MockPromotionService) GetClientPromotions(ctx context.Context, clientID string) ([]string, error) {
    return m.PromotionsList, m.Err
}
```
