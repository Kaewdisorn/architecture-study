# Architecture Study

A collection of architecture patterns and best practices, organized by **framework** and then by **architecture pattern**.

---

## Folder Structure

```
<framework>/
  └── <architecture-pattern>/
        └── README.md
```

### Example

```
nestjs/
  ├── clean-architecture/
  │     └── README.md
  ├── cqrs/
  │     └── README.md
  └── microservices/
        └── README.md

spring-boot/
  ├── clean-architecture/
  │     └── README.md
  └── hexagonal/
        └── README.md
```

---

## Frameworks

| Framework | Status |
|-----------|--------|
| NestJS | In Progress |

## Architecture Patterns

| Pattern | Description |
|---------|-------------|
| Clean Architecture | Uncle Bob's concentric layers with Dependency Rule |
| Clean Architecture + Ports & Adapters | Clean Architecture combined with Hexagonal (driving/driven port symmetry) |
| Domain-Driven Design (DDD) | Aggregates, bounded contexts, domain events, and tactical patterns |
| DDD + Ports & Adapters | DDD tactical patterns combined with Hexagonal port symmetry |

---

## How to Contribute

1. Create a folder: `<framework>/<architecture-pattern>/`
2. Add a `README.md` with the architecture guide
3. Include project structure, code examples, dependency flow, and testing strategy
