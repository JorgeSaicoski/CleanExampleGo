# Application DTOs (Design Note)

> [!NOTE]
> **Architectural Decision: Deprecated in Favor of Domain Entities**
> 
> In this architecture, dedicated `application/dto` structures are **not used**. 
> Use cases and repository contracts operate directly with **Domain Entities** (`domain/entities`).

---

## Why Application DTOs Were Removed

1. **Avoid Quadruple Model Duplication**: 
   When using Application DTOs, every concept requires 4 representations:
   - Database record: `infrastructure/postgresql/entities/ClientRecord`
   - Application DTO: `application/dto/ClientDTO`
   - Domain entity: `domain/entities/Client`
   - API DTO: `interfaces/dto/ClientResponse`
   
   Eliminating `application/dto` cuts out redundant mapping layers without loss of encapsulation.

2. **Prevent Anemic Domain Models**:
   When use cases work with flat, behaviorless DTOs, domain validation logic tends to leak into use cases procedurally. Using Domain Entities directly guarantees that entity rules (e.g. `client.CanBorrow()`) remain encapsulated inside the domain entity.

3. **Two Explicit Boundaries**:
   - **Outer Boundary**: `interfaces/dto` (API request / response contracts).
   - **Persistence Boundary**: `infrastructure/postgresql/entities` (Database schema representations).
   - **Internal Core**: `domain/entities` (Used across use cases and repository contracts).
