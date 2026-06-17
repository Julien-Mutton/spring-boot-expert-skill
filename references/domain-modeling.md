# Domain Modeling — Value Objects & Rich Entities

Make illegal states unrepresentable. Push validation and behavior into the domain so an invalid object cannot exist and primitives can't be mixed up. Scale this to the architecture you chose — value objects are the heart of Tomato/DDD, optional noise in a CRUD prototype.

## Value objects

Immutable, self-validating, equality-by-value, no identity, rich behavior. Java `record` is the ideal carrier. Validate in the compact constructor; expose an `of()` factory; add domain methods.

| Primitive problem | Value object fix |
|-------------------|------------------|
| `String` orderNumber vs `String` email — interchangeable | Compiler rejects mismatches |
| Validation scattered everywhere | Validation once, in the constructor |
| `BigDecimal` could be price or discount | `Money`, `Percentage` carry intent |
| Easy to pass wrong values | Invalid value objects can't be constructed |

### Typed ids (kill primitive obsession)

```java
public record OrderId(Long id) {
    public OrderId { if (id == null || id < 0) throw new IllegalArgumentException("Invalid order ID"); }
    public static OrderId of(Long id) { return new OrderId(id); }
}
// void process(OrderId o, CustomerId c)  — passing them swapped is now a compile error
```

### Constrained string / email

```java
public record EmailAddress(@JsonValue String value) {
    private static final Pattern P = Pattern.compile("^[A-Za-z0-9+_.-]+@(.+)$");
    @JsonCreator public EmailAddress {
        if (value == null || !P.matcher(value).matches())
            throw new IllegalArgumentException("Invalid email address: " + value);
    }
    public String domain() { return value.substring(value.indexOf('@') + 1); }
}
```

`@JsonValue` + `@JsonCreator` make the value object serialize as its raw scalar over the wire while staying a rich type in code.

### Money (behavior + invariants)

```java
public record Money(@JsonValue BigDecimal amount, Currency currency) {
    public Money {
        if (amount == null || currency == null) throw new IllegalArgumentException("null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) throw new IllegalArgumentException("negative");
    }
    public static Money usd(BigDecimal a) { return new Money(a, Currency.getInstance("USD")); }
    public Money add(Money o) { ensureSameCurrency(o); return new Money(amount.add(o.amount), currency); }
    public Money subtract(Money o) {
        ensureSameCurrency(o);
        var r = amount.subtract(o.amount);
        if (r.compareTo(BigDecimal.ZERO) < 0) throw new IllegalArgumentException("negative money");
        return new Money(r, currency);
    }
    private void ensureSameCurrency(Money o) {
        if (!currency.equals(o.currency)) throw new IllegalArgumentException("currency mismatch");
    }
}
```

### Embedding in JPA

Value objects map as `@Embeddable`/`@Embedded` with `@AttributeOverride`. **Do not use records as JPA entities** — entities need a mutable form and a no-arg constructor; records are immutable. Records are perfect for *embeddables that are read/replaced wholesale*, DTOs, commands, and projections.

## Rich entities & aggregates

Aggregate root with behavior, not getters/setters. Factory methods (not public constructors) for creation; protected no-arg constructor for JPA; `@Version` for optimistic locking; business methods enforce invariants.

```java
@Entity
@Table(name = "products")
class ProductEntity extends BaseEntity {     // BaseEntity = @MappedSuperclass with @CreatedDate/@LastModifiedDate auditing

    @EmbeddedId private ProductId id;
    @Embedded @AttributeOverride(name = "code", column = @Column(name = "sku", unique = true))
    private ProductSKU sku;
    @Embedded private Price price;
    @Embedded private Quantity quantity;
    @Enumerated(EnumType.STRING) private ProductStatus status;
    @Version private int version;

    protected ProductEntity() {}              // JPA only

    public static ProductEntity create(ProductSKU sku, Price price, Quantity qty) {
        return new ProductEntity(ProductId.generate(), sku, price, qty, ProductStatus.DRAFT);
    }

    public void decreaseStock(int amount) {   // invariant lives here, not in a service
        if (quantity.value() < amount) throw new InsufficientStockException("Not enough stock");
        this.quantity = Quantity.of(quantity.value() - amount);
    }
    public void activate() { if (status != ProductStatus.ACTIVE) this.status = ProductStatus.ACTIVE; } // idempotent
    // getters only — no setters
}
```

## Domain events with sealed hierarchies

Model a closed set of events as a sealed interface of records → exhaustive `switch` with no `default`, and the compiler enforces completeness when you add a case.

```java
public sealed interface DomainEvent permits UserCreated, UserUpdated, UserDeleted {
    Instant occurredAt();
}
public record UserCreated(Long userId, String email, Instant occurredAt) implements DomainEvent {}

String describe(DomainEvent e) {
    return switch (e) {
        case UserCreated c -> "created " + c.email();
        case UserUpdated u -> "updated";
        case UserDeleted d -> "deleted " + d.userId();
        // no default — compiler verifies exhaustiveness
    };
}
```

## Boundary error contract

Throw domain exceptions (`DomainException`, `ResourceNotFoundException`) from the domain; translate to RFC 7807 `ProblemDetail` in a `@RestControllerAdvice` extending `ResponseEntityExceptionHandler`. Log details server-side; return generic, non-leaking messages to clients (validation errors as a `422` with an `errors` property, not-found as `404`, unexpected as `500` with a timestamp).
