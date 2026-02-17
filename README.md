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
| Clean Architecture + Ports & Adapters | Layered separation with dependency inversion via ports/adapters |

---

## How to Contribute

1. Create a folder: `<framework>/<architecture-pattern>/`
2. Add a `README.md` with the architecture guide
3. Include project structure, code examples, dependency flow, and testing strategy
