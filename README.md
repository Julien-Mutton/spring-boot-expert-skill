# spring-boot-expert

A Claude skill for architecting and building **production-grade Spring Boot 4 / Java 25** platforms — scalable REST APIs and concurrent backend systems that stay correct when distributed.

The skill is opinionated about the failures that pass local tests and detonate under load: check-then-act races, N+1 queries, mutable singleton state, unverified JWTs, and blocking I/O inside transactions. It pairs a sharp core with deep, just-in-time reference material.

## Structure

```
skill.md                 # core: principles, architecture decision matrix, canonical example
references/
  architecture-decisions.md      # Layered → DDD+Hex, package layouts, naming, upgrade paths
  domain-modeling.md             # value objects, rich entities/aggregates, sealed events
  jpa-data-access.md             # repositories, N+1, projections, CQRS, hot-counter concurrency
  security.md                    # zero-trust, OWASP Top 10, Spring Security, JWT, validation
  performance-resilience.md      # caching, async, virtual threads, structured concurrency, Hikari
  spring-boot-4-and-java-25.md   # RestTestClient, @ImportHttpServices, API versioning, records, JSpecify
  migration-to-boot-4.md         # Jackson 3, test annotations, Modulith 2, native resilience
```

The core (`skill.md`) loads first; each reference is self-contained and read only when the task touches that area.

## Core principles

1. **Verify topology and boundaries before building** — never assume a single instance or in-memory state.
2. **Start simple** — pick the lightest architecture the domain demands now; know the upgrade path.
3. **Never expose entities; validate at the boundary** — thin controllers, record DTOs, `ProblemDetail`.
4. **Concurrency guaranteed at the database** — `@Version`, atomic counter updates, DB constraints.
5. **Zero-trust security** — deny by default, full JWT validation, per-resource ownership checks.
6. **Model the domain** — value objects and rich entities over primitive obsession and anemic services.
7. **Prefer native and modern** — Boot 4 / Java 25 features over external libraries and legacy patterns.

## Usage

Install as a personal Claude skill:

```bash
mkdir -p ~/.claude/skills/spring-boot-expert
cp -r skill.md references ~/.claude/skills/spring-boot-expert/
```

Some tooling expects the entry file to be named `SKILL.md` — rename `skill.md` if your setup requires it.

## Credits

Synthesized from a hand-written Spring Boot expert prompt and patterns drawn from
[a-pavithraa/springboot-skills-marketplace](https://github.com/a-pavithraa/springboot-skills-marketplace).
