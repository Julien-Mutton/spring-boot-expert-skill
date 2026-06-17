# Spring Boot 4 & Java 25 Features

Target stack: Java 25 + Spring Boot 4.0.x. Prefer native/modern over external libraries and legacy patterns. (Resilience, virtual threads, structured concurrency live in `performance-resilience.md`.)

## Spring Boot 4 highlights

### Modular starters

`spring-boot-starter-webmvc` (replaces `-web`), `-restclient` / `-webclient`, `-aspectj` (AOP for `@Retryable`/`@ConcurrencyLimit`). Add only what you need.

### RestTestClient (replaces TestRestTemplate)

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class ProductControllerTest {
    @Autowired WebApplicationContext ctx;
    RestTestClient client;
    @BeforeEach void setup() { client = RestTestClient.bindToApplicationContext(ctx).build(); }

    @Test void getsProduct() {
        client.get().uri("/api/products/{id}", "PRD-001")
            .exchange().expectStatus().isOk()
            .expectBody(ProductVM.class);
    }
}
```

Fluent, type-safe, built-in API-version header support (`.apiVersion("2.0")`).

### HTTP Service Clients — `@ImportHttpServices`

Declarative interface clients; auto-configuration replaces manual `HttpServiceProxyFactory` wiring (`@HttpExchange` itself exists since Framework 6).

```java
@HttpExchange
public interface ProductClient {
    @GetExchange("/api/products/{id}") Optional<ProductDTO> findById(@PathVariable Long id);
    @PostExchange("/api/products")     ProductDTO create(CreateProductRequest req);
}

@Configuration
@ImportHttpServices(group = "product", types = ProductClient.class)
class ClientConfig {}                  // RestClient is the default client type
```

```yaml
spring.http.serviceclient.product:    # group name matches @ImportHttpServices
  base-url: http://localhost:8080
  read-timeout: 10s
```

### Native API versioning

```yaml
spring.mvc.apiversion:
  enabled: true
  strategy: header            # or path | query-parameter | media-type
  default-version: "1.0"
  header-name: "API-Version"
```

```java
@GetMapping(value = "/search", version = "1.0") List<ProductVM> v1(...) {...}
@GetMapping(value = "/search", version = "2.0") List<ProductEnrichedVM> v2(...) {...}
```

Header strategy recommended. Works with HTTP Service Clients (`@GetExchange(url=..., version="2.0")`).

### Spring Data AOT / native image

Add the `process-aot` execution to `spring-boot-maven-plugin` for GraalVM native images — fast startup, low memory for serverless/containers. Cost: longer builds; reflection/proxies may need hints.

## Java 25 language features

| Feature | Use for | Note |
|---------|---------|------|
| **Records** | DTOs, value objects, projections, commands, events | Not as JPA entities (immutable) |
| **Sealed classes** | Closed type hierarchies, state machines, ADTs | Enables exhaustive `switch` (no `default`) |
| **Pattern matching** (`instanceof`, `switch`, record deconstruction) | Type-driven dispatch | Replaces instanceof+cast ladders |
| **Text blocks** | Multi-line JPQL/SQL in `@Query` | Readability |
| **Switch expressions** | Value-producing branching | Fewer bugs than fall-through statements |
| **Sequenced collections** | `getFirst()`/`getLast()`/`reversed()` | Safer than manual index math |
| **Virtual threads** (stable, 21+) | High-concurrency I/O | See `performance-resilience.md` |

```java
// pattern matching + sealed → exhaustive, no default
String describe(Shape s) {
    return switch (s) {
        case Circle(double r)        -> "circle " + r;
        case Rectangle(double w, double h) -> "rect " + w + "x" + h;
    };
}

@Query("""
    SELECT u FROM User u LEFT JOIN FETCH u.roles
    WHERE u.active = true AND u.createdAt > :since
    """)
List<User> findActiveSince(@Param("since") LocalDateTime since);
```

**Not available:** String Templates (`STR."..."`) were withdrawn — use `String.format`, concatenation, or `"...".formatted(...)`.

**Preview (use deliberately):** primitive type patterns (JEP 507), module import declarations, flexible constructor bodies (validate before `super()`), `ScopedValue` (request context for virtual threads, better than `ThreadLocal`), `StableValue` (lazy-once init), Key Derivation Function API.

Migration priority: records, text blocks, switch expressions, `instanceof` patterns, sequenced collections first (easy wins); sealed classes + switch patterns where hierarchies are fixed; virtual threads only with evidence.

## JSpecify null-safety (optional but recommended for new Boot 4 code)

Spring Framework 7 / Boot 4 expose null-safe APIs via JSpecify; deprecate `org.springframework.lang` annotations.

```java
// package-info.java — everything non-null by default
@NullMarked
package com.example.products;
import org.jspecify.annotations.NullMarked;
```

JSpecify annotations are **type-use** — place them next to the type, and update arrays/varargs carefully:

```java
private @Nullable String description;                 // field
public @Nullable String build(@Nullable String in) {} // param + return
@Nullable Object[] a;   // nullable elements, non-null array
Object @Nullable [] b;  // non-null elements, nullable array
```

Rules: default non-null via `@NullMarked`; `@Nullable` sparingly (prefer `Optional<T>` for return types); **copy nullability annotations onto overrides** (not inherited). For build-time enforcement use NullAway (`OnlyNullMarked=true`, then `JSpecifyMode=true`). It's a transition, not a switch — don't flag every legacy Spring annotation in a codebase mid-migration.
