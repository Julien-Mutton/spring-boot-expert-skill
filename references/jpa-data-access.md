# Spring Data JPA — Data Access at Scale

## Repository boundaries

- Create repositories **only for aggregate roots**, not every entity. Internal child entities are reached through their aggregate.
- Keep transaction boundaries in the **service layer** unless the architecture intentionally does otherwise.
- Understand persist vs merge before calling `save()` blindly when entity-state transitions matter.

| Need | Use |
|------|-----|
| Basic CRUD + 1-2 simple lookups | Derived query methods on a `JpaRepository` |
| Multiple filters, joins, sorting | `@Query` (JPQL, text blocks) |
| Read-only, performance-critical responses | DTO / interface projection |
| Criteria API, bulk ops, `EntityManager` logic | Custom repository fragment |
| Read model differs from write model, reporting | CQRS query service (often `JdbcTemplate`) |

## The N+1 problem (the most common JPA performance bug)

```java
List<Order> orders = orderRepository.findAll();      // 1 query
for (Order o : orders) o.getCustomer().getName();    // N more queries → 1 + N
```

Fixes:

```java
// 1. JOIN FETCH (best for loading entities you'll navigate)
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findWithCustomer(@Param("status") OrderStatus status);

// 2. @EntityGraph (declarative, reuses derived queries)
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findByStatus(OrderStatus status);

// 3. DTO projection (best for reads — fetches only needed columns)
@Query("SELECT new com.example.OrderSummary(o.id, o.orderNumber, c.name, o.total) " +
       "FROM Order o JOIN o.customer c WHERE o.status = :status")
List<OrderSummary> findSummaries(@Param("status") OrderStatus status);
```

Detect it: enable `hibernate.generate_statistics`, SQL logging (`org.hibernate.SQL: DEBUG`), or p6spy in dev. **Never** access a lazy association outside its transaction.

## Projections — don't load whole entities for reads

```java
// Interface projection
interface UserSummary { Long getId(); String getName(); }
List<UserSummary> findAllProjectedBy();

// DTO/record projection
record UserDTO(Long id, String name) {}
@Query("SELECT new com.example.UserDTO(u.id, u.name) FROM User u")
List<UserDTO> findAllDTOs();
```

## CQRS read model with JdbcTemplate

For read-heavy / reporting paths in Tomato/DDD, separate a public `*QueryService` (read, returns view models) from a package-private write repository (returns entities). `JdbcTemplate` is faster than JPA for reads, has no lazy-loading traps, and never leaks entities.

```java
@Service
@Transactional(readOnly = true)
public class ProductQueryService {
    private final JdbcTemplate jdbc;
    public ProductQueryService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    public List<ProductVM> findAllActive() {
        return jdbc.query("""
            SELECT id, sku, name, price, stock FROM products
            WHERE status = 'ACTIVE' ORDER BY created_at DESC
            """,
            (rs, n) -> new ProductVM(rs.getLong("id"), rs.getString("sku"),
                                     rs.getString("name"), rs.getBigDecimal("price"), rs.getInt("stock")));
    }
}
```

For dynamic search, build SQL with bound `?` parameters in a list — **never** string-concatenate user input (see `security.md`).

## Relationships

- Prefer `@ManyToOne` over `@OneToMany` when you can; map the owning side.
- Use **all associations `FetchType.LAZY`**, then `JOIN FETCH`/`@EntityGraph` where you actually need them.
- Prefer storing **ids** over entity references when loose coupling matters more than navigation (and across module boundaries).
- Treat `@ManyToMany` as a warning sign — model an explicit **join entity** instead so you can add attributes and control fetching.

## Batching, pagination, streaming

```yaml
spring.jpa.properties.hibernate:
  jdbc.batch_size: 25
  order_inserts: true
  order_updates: true
```

```java
repository.saveAll(products);                                   // batched insert
// large batches: flush() + clear() the EntityManager every batch_size rows
Page<Product> page = repo.findByCategory("Electronics",
        PageRequest.of(0, 20, Sort.by("createdAt").descending()));   // always paginate growable result sets

@QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "50"))
@Query("SELECT p FROM Product p WHERE p.status = 'ACTIVE'")
Stream<Product> streamAllActive();                              // stream huge sets in a try-with-resources
```

## Concurrency: atomic updates for hot counters

When many requests mutate one hot row (seat capacity, account balance, stock), pick the cheapest correct tool:

| Situation | Tool |
|-----------|------|
| Hot single-row counter under peak load | **Set-based atomic `UPDATE ... WHERE`** — best |
| General update of an entity that's rarely contended | `@Version` optimistic lock + bounded retry |
| Heavy contention where you must load-then-decide | `@Lock(PESSIMISTIC_WRITE)` on the read |

```java
@Modifying
@Query("UPDATE Event e SET e.seatsTaken = e.seatsTaken + 1 " +
       "WHERE e.id = :id AND e.seatsTaken < e.capacity")
int takeSeat(@Param("id") UUID id);   // rows-affected: 1 = got a seat, 0 = sold out
```

The single statement takes a row lock and serializes concurrent writers — no read-then-write race, no lost update, and no retry storm (unlike optimistic locking on a row everyone touches). Back it with a `CHECK (seats_taken <= capacity)` constraint as the cross-instance backstop. Optimistic `@Version` is still the right default for ordinary entity updates; pessimistic locking fits when you must read, branch on the value, then write within one transaction.

## Transactions & read-only

```java
@Transactional(readOnly = true)   // no dirty checking, can route to read replicas
public List<User> findActive() { ... }
```

Class-level `readOnly = true` on a query service, override with `@Transactional` on the rare write method.

## Indexes & schema

Add indexes for every foreign key and frequently-filtered/sorted column; compose them to match query predicates:

```sql
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);
```

Enforce invariants the application can't guarantee alone with DB `UNIQUE`/`CHECK` constraints, shipped via Flyway/Liquibase migrations.

## Checklist

- [ ] Repositories only at aggregate roots
- [ ] All associations LAZY; `JOIN FETCH`/`@EntityGraph` where needed
- [ ] No lazy access outside a transaction; no N+1 in loops
- [ ] DTO/interface projections for read-only responses
- [ ] Pagination on growable result sets; batch size configured for bulk writes
- [ ] `@Transactional(readOnly = true)` on query paths
- [ ] Indexes on FKs and filtered columns; `@ManyToMany` replaced by a join entity
- [ ] Query logging on in dev to catch N+1 early
