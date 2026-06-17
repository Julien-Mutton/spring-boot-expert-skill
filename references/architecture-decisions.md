# Architecture Decisions

Pick the simplest pattern that fits the domain *now*, and know the upgrade path. Measure complexity by team size, lifespan, and domain complexity — not by ambition.

## Decision matrix

| Criteria | Layered | Package-by-Module | Modulith | Tomato | DDD+Hex |
|----------|---------|-------------------|----------|--------|---------|
| Team size | 1-3 | 3-10 | 5-15 | 5-15 | 10+ |
| Lifespan | Months | 1-2 yrs | 2-5 yrs | 3-5 yrs | 5+ yrs |
| Type safety | Low | Low | Low | High | High |
| Learning curve | ⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |

## Package structures

```
# Layered — simple CRUD, prototypes, MVPs
com.example.app/{controller, service, repository, domain}

# Package-by-Module — 3-5 features, feature isolation
com.example.app/
  products/{domain, rest}
  orders/{domain, rest}
  shared/

# Modular Monolith (Spring Modulith) — enforced boundaries, extraction-ready
com.example.app/
  products/{internal, domain, ProductsAPI.java}   # internal = package-private impls
  orders/{internal, domain, OrdersAPI.java}
  shared/

# Tomato — rich domain, value objects, type safety
com.example.app/
  products/
    domain/{vo/{ProductSKU,Price,Quantity}, ProductEntity, ProductService, ProductQueryService}
    rest/{ProductController, converters/StringToProductSKUConverter}
  shared/vo/{Money, Email}

# DDD + Hexagonal — complex domains, CQRS, infra independence
com.example.app/
  products/
    application/{command/{CreateProductHandler}, query/{ProductQueryService}}
    domain/{model/{ProductAggregate,ProductFactory}, vo/, ports/{ProductRepository (interface)}}
    infra/persistence/{JpaProductRepository (impl)}
    interfaces/rest/{ProductController}
  shared/domain/
```

## Naming conventions (Tomato / DDD)

| Type | Pattern | Example | Package |
|------|---------|---------|---------|
| Entity | `*Entity` | `ProductEntity` | `domain/model/` |
| Aggregate root | `*Aggregate` | `OrderAggregate` | `domain/model/` |
| Value object | domain name | `ProductSKU`, `Price`, `Email` | `domain/vo/` |
| Command | `*Cmd` | `CreateProductCmd` | `application/command/` |
| Command handler | `*Handler` | `CreateProductHandler` | `application/command/` |
| View model | `*VM` | `ProductVM` | `interfaces/rest/dto/` |
| Write service | `*Service` | `ProductService` | `domain/` |
| Read service | `*QueryService` | `ProductQueryService` | `application/query/` |
| Repository interface | `*Repository` | `ProductRepository` | `domain/ports/` (DDD) or `domain/` |
| Repository impl | `Jpa*Repository` | `JpaProductRepository` | `infra/persistence/` |
| Module API | `*API` | `ProductsAPI` | package root |
| Converter | `StringTo*Converter` | `StringToProductSKUConverter` | `rest/converters/` |

Value objects are immutable, use an `of()` factory; commands and view models are immutable records; aggregates encapsulate invariants.

## Anti-patterns

| Don't | Do | Why |
|-------|-----|-----|
| Jump to implementation | Assess complexity first | Prevents over/under-engineering |
| DDD for simple CRUD | Layered / Package-by-Module | DDD adds cost without benefit |
| Layered for complex domain | Tomato / DDD+Hex | Layered → anemic domain models |
| Skip infrastructure | Always Flyway + Testcontainers + Docker Compose | Production readiness |
| Primitive obsession | Value objects for domain concepts | Type safety, explicit intent |
| Anemic domain model | Business logic in entities/aggregates | Core DDD principle |
| God services | Split command/query (CQRS) | Maintainability |
| Shared mutable state | Immutable value objects, stateless beans | Thread safety |
| Controller → repository | Controller → service → repository | Keep boundaries; logic out of controllers |
| JPA entities in API | Map to record DTOs/VMs | Decouples contract from schema |

## Upgrade triggers & migration steps

| From → To | Trigger | Effort |
|-----------|---------|--------|
| Layered → Package-by-Module | 3+ features, team grows to 3+ | Low |
| Package-by-Module → Modulith | Need enforced boundaries / independent tests | Medium |
| Modulith → Tomato | Type-confusion bugs, validation scattered | Medium |
| Tomato → DDD+Hex | Need infra independence, CQRS, complex rules | High |

- **→ Package-by-Module:** create module packages, move related controllers/services/repos in, extract `shared/`, fix cross-module deps.
- **→ Modulith:** add Spring Modulith, create `*API.java` per module, make internals package-private, add `ApplicationModules.of(App.class).verify()` test (fails build on cycles or internal-package access across modules).
- **→ Tomato:** identify primitives that are domain concepts, create value objects, replace primitives in entities, add Spring `Converter`s for `@PathVariable` binding, update tests.
- **→ DDD+Hex:** separate commands from queries, add application layer with handlers, define domain ports (repo interfaces in `domain/`), move impls to `infra/`, extract aggregates, add ArchUnit tests for layer dependencies.

## Signs you've outgrown the current pattern

- **Layered:** features span unrelated domains; services > 500 lines; changes ripple across unrelated features.
- **Package-by-Module:** circular module deps; hard to test modules in isolation; unclear boundaries.
- **Modulith:** type-confusion bugs (mixing ids/codes/values); validation scattered; financial/healthcare type-safety pressure.
- **Tomato:** complex rules across many aggregates; need to swap infrastructure; multiple teams on one codebase.

## Module boundaries (always)

Clear single responsibility per module; minimize inter-module deps; use **domain events** for cross-module communication (publish, then `@TransactionalEventListener(AFTER_COMMIT)` consume); test modules independently; never reach into another module's `internal` package.
