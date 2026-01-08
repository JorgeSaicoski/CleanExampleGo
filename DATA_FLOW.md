# Data Flow: Application DTOs and Infrastructure Entities

## The Complete Picture

This document shows how data flows through the architecture, clarifying when to use application DTOs vs infrastructure entities.

## Key Concept

**Infrastructure needs internal entities to handle external system formats, then converts them to application DTOs.**

---

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                         │
│                                                              │
│  Use Case: "Get client balance"                             │
│     ↓ calls                                                  │
│  PointsService.GetBalance(clientID) → *dto.PointsDTO       │
│                                           ↑                  │
│  Application DTO (what app expects):      │                 │
│  ┌──────────────────────────┐             │                 │
│  │ type PointsDTO struct {  │             │                 │
│  │   ClientID   string      │             │                 │
│  │   Points     int         │             │ returns         │
│  │   Tier       string      │             │                 │
│  │   HasDiscount bool       │             │                 │
│  │   DiscountPercent int    │             │                 │
│  │ }                        │             │                 │
│  └──────────────────────────┘             │                 │
└────────────────────────────────────────────┼─────────────────┘
                                             │
                                    ┌────────┴────────┐
                                    │  Conversion     │
                                    │  .ToDTO()       │
                                    └────────┬────────┘
                                             │
┌────────────────────────────────────────────┼─────────────────┐
│              INFRASTRUCTURE LAYER          │                 │
│                                            │                 │
│  Service Implementation:                   │                 │
│    func GetBalance(clientID) (*dto.PointsDTO, error) {      │
│      // 1. Call external API              │                 │
│      var response entities.PointsBalanceResponse            │
│      httpClient.Get(url, &response)       │                 │
│                                            │                 │
│      // 2. Convert to application DTO     │                 │
│      return response.ToDTO(), nil         │                 │
│    }                                       │                 │
│                                            ↑                 │
│  Internal Entity (external API format):    │                 │
│  ┌────────────────────────────────────────┐                 │
│  │ type PointsBalanceResponse struct {    │                 │
│  │   ClientID       string `json:"client_id"`               │
│  │   TotalPoints    int    `json:"total_points"`            │
│  │   CurrentBalance int    `json:"current_balance"`         │
│  │   TierLevel      string `json:"tier_level"`              │
│  │   HasActivePromo bool   `json:"has_active_promo"`        │
│  │   PromoDiscount  int    `json:"promo_discount"`          │
│  │   ExpirationDate string `json:"expiration_date"`         │
│  │   // ... many other external API fields                  │
│  │ }                                      │                 │
│  └────────────────────────────────────────┘                 │
│                          ↑                                   │
│                          │ unmarshals JSON                   │
└──────────────────────────┼───────────────────────────────────┘
                           │
┌──────────────────────────┼───────────────────────────────────┐
│          EXTERNAL LOYALTY API                                │
│                          │                                   │
│  GET /api/loyalty/balance/{clientId}                         │
│                          │                                   │
│  Returns JSON:           │                                   │
│  {                       │                                   │
│    "client_id": "123",                                       │
│    "total_points": 5000,                                     │
│    "current_balance": 3500,                                  │
│    "tier_level": "gold",                                     │
│    "has_active_promo": true,                                 │
│    "promo_discount": 15,                                     │
│    "expiration_date": "2026-12-31",                          │
│    "next_tier_threshold": 7000,                              │
│    "member_since": "2020-01-15",                             │
│    ... many other fields                                     │
│  }                                                           │
└──────────────────────────────────────────────────────────────┘
```

---

## Real Example: PostgreSQL Repository

### Application Layer Says:
```go
// application/repositories/client_repository.go
type ClientRepository interface {
    GetByID(id string) (*dto.ClientDTO, error)
}

// application/dto/client_dto.go
type ClientDTO struct {
    ID              string
    Name            string
    Email           string
    Points          int
    BorrowedBookIDs []string
}
```

### Infrastructure Responds:

```go
// infrastructure/postgresql/entities/client_record.go (INTERNAL)
type ClientRecord struct {
    ID              string
    Name            string
    Email           string
    Points          int
    MaxBooksAllowed int
    CreatedAt       time.Time    // Database specific
    UpdatedAt       time.Time    // Database specific
    DeletedAt       *time.Time   // Database specific (soft delete)
}

func (r *ClientRecord) ToDTO() *dto.ClientDTO {
    return &dto.ClientDTO{
        ID:     r.ID,
        Name:   r.Name,
        Email:  r.Email,
        Points: r.Points,
        // BorrowedBookIDs loaded separately
    }
}

// infrastructure/postgresql/repositories/client_repository.go
func (r *PostgreSQLClientRepository) GetByID(id string) (*dto.ClientDTO, error) {
    // 1. Query database using internal entity
    var record entities.ClientRecord
    err := r.db.QueryRow(`
        SELECT id, name, email, points, max_books_allowed, 
               created_at, updated_at, deleted_at
        FROM clients 
        WHERE id = $1 AND deleted_at IS NULL
    `, id).Scan(
        &record.ID,
        &record.Name,
        &record.Email,
        &record.Points,
        &record.MaxBooksAllowed,
        &record.CreatedAt,
        &record.UpdatedAt,
        &record.DeletedAt,
    )
    if err != nil {
        return nil, err
    }
    
    // 2. Convert database record to application DTO
    clientDTO := record.ToDTO()
    
    // 3. Load borrowed books
    clientDTO.BorrowedBookIDs, err = r.getBorrowedBookIDs(id)
    if err != nil {
        return nil, err
    }
    
    return clientDTO, nil
}
```

---

## Real Example: VTEX Service

### Application Layer Says:
```go
// application/services/order_service.go
type OrderService interface {
    GetOrder(orderID string) (*dto.OrderDTO, error)
}

// application/dto/order_dto.go
type OrderDTO struct {
    ID            string
    CustomerEmail string
    CustomerName  string
    Items         []OrderItemDTO
    TotalAmount   float64
}

type OrderItemDTO struct {
    ProductID string
    Quantity  int
    Price     float64
}
```

### Infrastructure Responds:

```go
// infrastructure/vtex/entities/order_response.go (INTERNAL)
// This matches VTEX's complex nested JSON structure
type VTEXOrderResponse struct {
    OrderId       string `json:"orderId"`
    OrderFormId   string `json:"orderFormId"`
    Sequence      string `json:"sequence"`
    MarketplaceOrderId string `json:"marketplaceOrderId"`
    
    ClientProfileData struct {
        Email     string `json:"email"`
        FirstName string `json:"firstName"`
        LastName  string `json:"lastName"`
        Document  string `json:"document"`
        Phone     string `json:"phone"`
    } `json:"clientProfileData"`
    
    Items []struct {
        UniqueId           string  `json:"uniqueId"`
        Id                 string  `json:"id"`
        ProductId          string  `json:"productId"`
        RefId              string  `json:"refId"`
        Name               string  `json:"name"`
        Quantity           int     `json:"quantity"`
        SellingPrice       int     `json:"sellingPrice"`  // in cents
        Price              int     `json:"price"`
        ListPrice          int     `json:"listPrice"`
        IsGift             bool    `json:"isGift"`
        // ... 50+ more fields VTEX returns
    } `json:"items"`
    
    Totals []struct {
        Id    string `json:"id"`
        Name  string `json:"name"`
        Value int    `json:"value"`
    } `json:"totals"`
    
    // ... many more VTEX-specific fields
}

// Convert VTEX format to application DTO
func (v *VTEXOrderResponse) ToDTO() *dto.OrderDTO {
    order := &dto.OrderDTO{
        ID:            v.OrderId,
        CustomerEmail: v.ClientProfileData.Email,
        CustomerName:  v.ClientProfileData.FirstName + " " + v.ClientProfileData.LastName,
        Items:         make([]dto.OrderItemDTO, len(v.Items)),
    }
    
    var total int
    for i, item := range v.Items {
        order.Items[i] = dto.OrderItemDTO{
            ProductID: item.ProductId,
            Quantity:  item.Quantity,
            Price:     float64(item.SellingPrice) / 100, // Convert cents to dollars
        }
        total += item.SellingPrice * item.Quantity
    }
    order.TotalAmount = float64(total) / 100
    
    return order
}

// infrastructure/vtex/service_impl.go
func (s *VTEXOrderService) GetOrder(orderID string) (*dto.OrderDTO, error) {
    // 1. Call VTEX API
    var vtexResponse entities.VTEXOrderResponse
    err := s.client.Get(
        s.baseURL + "/api/oms/pvt/orders/" + orderID,
        &vtexResponse,
    )
    if err != nil {
        return nil, err
    }
    
    // 2. Convert VTEX's complex format to simple application DTO
    return vtexResponse.ToDTO(), nil
}
```

---

## Why This Pattern is Necessary

### External Systems Have Their Own Formats

1. **VTEX API**: Returns 100+ fields in nested JSON
   - Application only needs 10 fields
   - Field names are camelCase (`orderId`, `clientProfileData`)
   - Prices in cents, dates in ISO format

2. **Loyalty API**: Returns their domain model
   - Field names: `tier_level`, `current_balance`
   - Extra metadata we don't need
   - Different structure than our app needs

3. **PostgreSQL**: Has database-specific fields
   - `created_at`, `updated_at`, `deleted_at`
   - `sql.NullString`, `sql.NullTime`
   - Join tables for relationships

### Application Wants Clean, Simple DTOs

```go
// Application doesn't want to deal with VTEX's complexity
type OrderDTO struct {
    ID            string
    CustomerEmail string
    CustomerName  string
    Items         []OrderItemDTO
    TotalAmount   float64
}
```

---

## Folder Structure with Examples

```
application/
  dto/                           ← What application expects
    client_dto.go                 ClientDTO
    book_dto.go                   BookDTO
    order_dto.go                  OrderDTO
    points_dto.go                 PointsDTO
  
  repositories/                  ← Interfaces using application DTOs
    client_repository.go          GetByID(id) → *ClientDTO
  
  services/                      ← Interfaces using application DTOs
    order_service.go              GetOrder(id) → *OrderDTO
    points_service.go             GetBalance(id) → *PointsDTO

infrastructure/
  postgresql/
    entities/                    ← Database records (INTERNAL)
      client_record.go            ClientRecord + ToDTO()
      book_record.go              BookRecord + ToDTO()
    repositories/                ← Implements app interfaces
      client_repository.go        Returns *ClientDTO
  
  vtex/
    entities/                    ← VTEX API responses (INTERNAL)
      order_response.go           VTEXOrderResponse + ToDTO()
      product_response.go         VTEXProductResponse + ToDTO()
    service_impl.go              ← Returns *OrderDTO
  
  loyalty_points/
    entities/                    ← Loyalty API responses (INTERNAL)
      points_response.go          PointsBalanceResponse + ToDTO()
      transaction_response.go     TransactionResponse + ToDTO()
    service_impl.go              ← Returns *PointsDTO
```

---

## Summary

### ✅ What Infrastructure DOES Need

**Internal entities** to handle external system formats:
- PostgreSQL records with database-specific fields
- API response structs with JSON tags matching external APIs
- Conversion methods (`ToDTO()`) to map to application DTOs

### ✅ What Infrastructure DOES

1. Define internal entities matching external formats
2. Query/call external systems using these entities
3. Convert internal entities to application DTOs
4. Return application DTOs to application layer

### ✅ What Application Layer Defines

- **DTOs**: Clean data contracts
- **Interfaces**: What operations it needs
- **Use Cases**: Business logic using DTOs

### ❌ What Application Layer NEVER Does

- Import infrastructure entities
- Know about PostgreSQL, VTEX, or any external system
- Deal with JSON tags, sql.NullString, or external formats

---

## The Key Rule

**"Infrastructure entities are internal adapters. Application DTOs are the contract."**

- External system → Infrastructure Entity → `.ToDTO()` → Application DTO
- Application only sees and uses Application DTOs
- Infrastructure handles all format conversions

This keeps the application clean and independent of external system changes!

