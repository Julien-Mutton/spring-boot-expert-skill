# Migrating Spring Boot 3 → 4

Migrate in steps, testing after each. Scan first for: dependency/starter updates, Jackson 2 assumptions, old test annotations, `TestRestTemplate`, manual `HttpServiceProxyFactory`, `@Retryable`/`@EnableRetry` from Spring Retry, custom API versioning, `org.springframework.lang` null annotations.

## Order of operations

1. **Spring Boot 4 core** — bump parent/BOM to `4.0.0`, Java to 25, swap starters (`-web` → `-webmvc`).
2. **Jackson 3** — the biggest breaking change (below).
3. **Test annotations** — renames (below).
4. **Spring Modulith 2** — event store schema change (below), if used.
5. **Testcontainers 2** — package renames, if used.
6. **Feature modernization** — `RestTestClient`, `@ImportHttpServices`, native API versioning, native resilience, JSpecify.

## Critical breaking changes

### Jackson 3
- Group id `com.fasterxml.jackson` → `tools.jackson` (**exception:** `jackson-annotations` keeps the old group id).
- `Jackson2ObjectMapperBuilderCustomizer` → `JsonMapperBuilderCustomizer`.

### Test annotations
- `@MockBean` → `@MockitoBean`, `@SpyBean` → `@MockitoSpyBean`.
- `@WebMvcTest` package relocated; `@SpringBootTest` may need `@AutoConfigureMockMvc`.

### Spring Modulith 2
- Requires updating the `event_publication` table schema.
- Dedicated `events` schema optional (larger apps): create a Flyway `V1__create_events_schema.sql` and set `spring.modulith.events.jdbc.schema=events`.

### Testcontainers 2
- Package renames — update imports.

## Feature migration map

| Boot 3 | Boot 4 | Action |
|--------|--------|--------|
| `TestRestTemplate` | `RestTestClient` | Rewrite assertions to fluent API; add `@AutoConfigureRestTestClient` |
| Resilience4j `@Retry` / Spring Retry `@Retryable` | native `@Retryable` | Package → `org.springframework.resilience.annotation.*`; `retryFor` → `includes`; `@EnableRetry` → `@EnableResilientMethods`; add `spring-boot-starter-aspectj`; remove `spring-retry` |
| manual concurrency control | `@ConcurrencyLimit` | Native bulkhead |
| Resilience4j `@CircuitBreaker` | **still Resilience4j** | Circuit breaker NOT native — keep the library; verify Boot 4 compatibility |
| manual `HttpServiceProxyFactory` bean | `@ImportHttpServices` | Keep `@HttpExchange` interfaces; move base-url/timeouts to `spring.http.serviceclient.<group>.*` |
| custom API versioning | `spring.mvc.apiversion.*` | Native via properties or beans |
| `org.springframework.lang` nullability | JSpecify `@NullMarked` | Type-use placement; fix arrays/varargs; gradual, package by package |

## Notes

- `TestRestTemplate`, manual `HttpServiceProxyFactory`, and Spring Retry still *work* but are deprecated/maintenance-only — modernize opportunistically, not all at once.
- The native `@Retryable` API differs from Resilience4j's (different package and parameter names) — don't assume a drop-in swap.
- A `scan_migration_issues.py`-style grep pass over the codebase before starting saves rework; categorize hits by the table above.

For a phased upgrade *plan* (vs. a one-shot review), produce: current-state scan → ordered migration scenarios → per-step test gates → rollback notes.
