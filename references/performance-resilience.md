# Performance & Resilience

(For JPA/N+1, batching, pagination, projections see `jpa-data-access.md`.)

## Caching

Cache expensive reads with the Spring Cache abstraction, **always with expiration and a size limit**. Caffeine for in-process, Redis for distributed.

```java
@Configuration @EnableCaching
class CacheConfig {
    @Bean Caffeine<Object,Object> caffeine() {
        return Caffeine.newBuilder().maximumSize(1000)
            .expireAfterWrite(10, TimeUnit.MINUTES);          // never cache without expiry
    }
    @Bean CacheManager cacheManager() { return new CaffeineCacheManager("users","products"); }
}

@Cacheable(value="users", key="#id") User findById(Long id) {...}
@CachePut(value="users", key="#u.id") User update(User u) {...}
@CacheEvict(value="users", key="#id") void delete(Long id) {...}
```

Anti-patterns: caching without expiration; caching multi-MB blobs (cache metadata/location instead); forgetting to evict on update/delete.

## Async — commit first, then dispatch

Never block the request thread on email/external calls/reports, and never do I/O inside a transaction. Dispatch after commit:

```java
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
void onOrderCreated(OrderCreated e) { emailClient.send(...); }   // fires only if tx committed
```

Configure a bounded `ThreadPoolTaskExecutor` (`@EnableAsync`) unless virtual threads are enabled. `@Async` methods returning `CompletableFuture` compose for parallel fan-out.

## Parallel work: structured concurrency (Java 25)

For fanning out independent calls, prefer structured concurrency over hand-composed `CompletableFuture` — automatic cancellation, single error-propagation point, scope-bound lifetime.

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user   = scope.fork(() -> userService.findById(id));
    var orders = scope.fork(() -> orderService.findByUserId(id));
    scope.join().throwIfFailed();                 // one fails → others cancelled
    return new Dashboard(user.get(), orders.get());
}
```

Keep `CompletableFuture` for Spring `@Async` integration and complex transformation pipelines (`thenCompose`/`exceptionally`).

## Virtual threads — only when justified

Virtual threads help **I/O-bound** workloads with **very high concurrency**. Oracle's guidance: *"If your application never has 10,000 virtual threads or more, it is unlikely to benefit."* JEP 444: they **cannot** improve CPU-bound throughput.

Use only when ALL hold: 10,000+ *concurrent* tasks (not daily totals), I/O-bound, thread-pool exhaustion observed in production. Otherwise keep tuned platform-thread pools.

```yaml
spring.threads.virtual.enabled: true
```

Pinning trap: `synchronized` around blocking I/O pins the carrier thread — use `ReentrantLock` instead. Don't recommend virtual threads without measured workload evidence.

## Resilience (Boot 4 native vs Resilience4j)

Native in Spring Framework 7 (`org.springframework.resilience.annotation.*`, needs `@EnableResilientMethods` + `spring-boot-starter-aspectj`):

```java
@Retryable(includes = IOException.class, maxAttempts = 4, delay = 1000, multiplier = 2)
public String callRemote() { ... }                 // exponential backoff, works with virtual threads

@ConcurrencyLimit(2)                                // bulkhead: cap concurrent executions
public void heavyJob(String id) { ... }
```

`RetryTemplate` (programmatic) + `RetryListener` (metrics) for finer control. **Circuit breakers are NOT native** — use Resilience4j (`spring-cloud-starter-circuitbreaker-resilience4j`) with `@CircuitBreaker(fallbackMethod=...)`; verify Resilience4j ↔ Boot 4 compatibility before adopting. Spring Retry is maintenance-only; prefer native.

## Connection pooling (HikariCP, default)

```yaml
spring.datasource.hikari:
  maximum-pool-size: 10           # NOT 100 — small, tuned pools beat huge ones
  minimum-idle: 5
  connection-timeout: 30000
  leak-detection-threshold: 60000 # surfaces leaked connections
```

Sizing rule of thumb: `(core_count * 2) + effective_spindle_count`; for cloud/SSD `core_count * 2 + 1`. Oversized pools cause context-switch overhead and exhaust DB limits.

## Resource management

Always `try-with-resources` for streams/JDBC. **Reuse** HTTP clients (`RestClient`) as injected singletons — never create one per request.

## Checklist

- [ ] Expensive reads cached with expiration + size limit; evicted on writes
- [ ] No blocking I/O on request thread or inside a transaction; side effects after commit
- [ ] Independent calls parallelized (structured concurrency / `CompletableFuture`)
- [ ] Virtual threads only with measured I/O-bound high-concurrency evidence; no `synchronized` around I/O
- [ ] Native `@Retryable`/`@ConcurrencyLimit` for transient faults/bulkheads; Resilience4j only for circuit breakers
- [ ] HikariCP pool tuned + leak detection; clients reused; resources closed
- [ ] Actuator metrics on (`hikaricp.*`, `hibernate.*`) for load testing
