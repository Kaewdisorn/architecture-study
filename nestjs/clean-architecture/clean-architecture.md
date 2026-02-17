# Pure Clean Architecture for NestJS

> Uncle Bob's Clean Architecture applied to NestJS — layered separation with the Dependency Rule, without the explicit Ports & Adapters (Hexagonal) pattern.

---

## Table of Contents

- [How This Differs from Hexagonal](#how-this-differs-from-hexagonal)
- [Project Structure](#project-structure)
- [Dependency Flow](#dependency-flow)
- [Layer 1 — Entities (Domain)](#layer-1--entities-domain)
- [Layer 2 — Use Cases (Application)](#layer-2--use-cases-application)
- [Layer 3 — Interface Adapters](#layer-3--interface-adapters)
- [Layer 4 — Frameworks & Drivers](#layer-4--frameworks--drivers)
- [Cross-Cutting Concerns](#cross-cutting-concerns)
- [Module Wiring](#module-wiring)
- [Error Handling](#error-handling)
- [Testing Strategy](#testing-strategy)
- [Quick Reference](#quick-reference)

---

## How This Differs from Hexagonal

| Aspect | Pure Clean Architecture | Clean Arch + Ports & Adapters |
|--------|------------------------|-------------------------------|
| **Port interfaces** | Use cases define their own repository/service interfaces inline or co-located | Explicit `ports/inbound/` and `ports/outbound/` directories |
| **Inbound ports** | Controllers call use cases directly (or via a thin interface) | Controllers depend on driving port interfaces (symbols) |
| **Layer naming** | Entities → Use Cases → Interface Adapters → Frameworks | Domain → Application → Infrastructure → Presentation |
| **Focus** | Concentric layers with Dependency Rule | Symmetry between driving and driven sides |
| **Complexity** | Simpler — fewer abstractions | More flexible — every boundary has a formal contract |

> **Use pure Clean Architecture** when you want clean separation without the overhead of explicit port/adapter symmetry. Use the hexagonal variant when maximum pluggability is needed. See [clean-architecture-with-ports-and-adapters.md](clean-architecture-with-ports-and-adapters.md) for the hexagonal version.

---

## Project Structure

```
src/
├── main.ts                              # Bootstrap (entry point)
├── app.module.ts                        # Root module
│
├── shared/
│   ├── exceptions/
│   │   └── domain.exception.ts
│   ├── filters/
│   │   └── global-exception.filter.ts
│   └── shared.module.ts
│
└── modules/
    └── user/
        │
        ├── domain/                          # Layer 1 — Entities
        │   ├── entities/
        │   │   └── user.entity.ts
        │   ├── value-objects/
        │   │   ├── email.vo.ts
        │   │   └── user-id.vo.ts
        │   └── repositories/
        │       └── user.repository.interface.ts   # Abstract interface
        │
        ├── use-cases/                       # Layer 2 — Use Cases
        │   ├── create-user/
        │   │   ├── create-user.use-case.ts
        │   │   ├── create-user.input.ts
        │   │   └── create-user.output.ts
        │   └── get-user/
        │       ├── get-user.use-case.ts
        │       ├── get-user.input.ts
        │       └── get-user.output.ts
        │
        ├── interface-adapters/              # Layer 3 — Interface Adapters
        │   ├── controllers/
        │   │   └── user.controller.ts
        │   ├── dtos/
        │   │   ├── create-user.request.dto.ts
        │   │   └── user.response.dto.ts
        │   ├── presenters/
        │   │   └── user.presenter.ts
        │   └── gateways/
        │       └── notification.gateway.ts  # e.g. WebSocket adapter
        │
        ├── frameworks/                      # Layer 4 — Frameworks & Drivers
        │   ├── typeorm/
        │   │   ├── user.typeorm.repository.ts
        │   │   ├── user.schema.ts
        │   │   └── user.mapper.ts
        │   └── services/
        │       └── email.service.ts
        │
        └── user.module.ts
```

**Key differences from hexagonal layout:**
- Repository interface lives in `domain/repositories/` (not `application/ports/outbound/`)
- Use cases are grouped by feature, each with their own input/output DTOs
- `interface-adapters/` replaces both `presentation/` and input-side wiring
- `frameworks/` replaces `infrastructure/` — emphasizing these are framework-dependent

---

## Dependency Flow

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│   ┌──────────────────────────────────────────────────────┐     │
│   │                                                      │     │
│   │   ┌──────────────────────────────────────────┐       │     │
│   │   │                                          │       │     │
│   │   │   ┌──────────────────────────┐           │       │     │
│   │   │   │                          │           │       │     │
│   │   │   │     ENTITIES             │           │       │     │
│   │   │   │     (Domain)             │           │       │     │
│   │   │   │                          │           │       │     │
│   │   │   └──────────────────────────┘           │       │     │
│   │   │                                          │       │     │
│   │   │          USE CASES                       │       │     │
│   │   │          (Application Logic)             │       │     │
│   │   │                                          │       │     │
│   │   └──────────────────────────────────────────┘       │     │
│   │                                                      │     │
│   │            INTERFACE ADAPTERS                        │     │
│   │            (Controllers, Presenters, Gateways)       │     │
│   │                                                      │     │
│   └──────────────────────────────────────────────────────┘     │
│                                                                │
│              FRAMEWORKS & DRIVERS                              │
│              (NestJS, TypeORM, Express, SendGrid)              │
│                                                                │
└────────────────────────────────────────────────────────────────┘

  All dependencies point INWARD. The inner circles know nothing about the outer circles.
```

---

## Layer 1 — Entities (Domain)

The innermost layer. Contains enterprise business rules — entities, value objects, and repository contracts.

### Entity

```typescript
// modules/user/domain/entities/user.entity.ts

import { UserId } from '../value-objects/user-id.vo';
import { Email } from '../value-objects/email.vo';
import { DomainError } from '../../../shared/exceptions/domain.exception';

export interface CreateUserProps {
  email: string;
  name: string;
}

export class User {
  constructor(
    public readonly id: UserId,
    public readonly email: Email,
    public name: string,
    public readonly createdAt: Date,
  ) {}

  static create(props: CreateUserProps): User {
    if (!props.email) {
      throw new DomainError('Email is required');
    }
    if (!props.name || props.name.trim().length < 2) {
      throw new DomainError('Name must be at least 2 characters');
    }

    return new User(
      UserId.generate(),
      new Email(props.email),
      props.name.trim(),
      new Date(),
    );
  }

  changeName(name: string): void {
    if (!name || name.trim().length < 2) {
      throw new DomainError('Name must be at least 2 characters');
    }
    this.name = name.trim();
  }
}
```

### Value Objects

```typescript
// modules/user/domain/value-objects/email.vo.ts

import { DomainError } from '../../../shared/exceptions/domain.exception';

export class Email {
  private static readonly PATTERN = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

  public readonly value: string;

  constructor(email: string) {
    if (!Email.PATTERN.test(email)) {
      throw new DomainError(`Invalid email: ${email}`);
    }
    this.value = email.toLowerCase();
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }

  toString(): string {
    return this.value;
  }
}
```

```typescript
// modules/user/domain/value-objects/user-id.vo.ts

import { randomUUID } from 'crypto';

export class UserId {
  constructor(public readonly value: string) {}

  static generate(): UserId {
    return new UserId(randomUUID());
  }

  equals(other: UserId): boolean {
    return this.value === other.value;
  }

  toString(): string {
    return this.value;
  }
}
```

### Repository Interface (Domain-Level Contract)

In pure Clean Architecture, the repository interface lives in the **domain layer** — it's part of the entity's boundary, not a separate "port".

```typescript
// modules/user/domain/repositories/user.repository.interface.ts

import { User } from '../entities/user.entity';

export abstract class UserRepository {
  abstract findById(id: string): Promise<User | null>;
  abstract findByEmail(email: string): Promise<User | null>;
  abstract save(user: User): Promise<void>;
  abstract delete(id: string): Promise<void>;
}
```

> **Why an abstract class instead of an interface?** NestJS DI needs a runtime token. Abstract classes serve as both the contract AND the injection token — no `Symbol()` needed.

---

## Layer 2 — Use Cases (Application)

Each use case is a single class with a single `execute()` method. It orchestrates entities and depends on the repository interface from Layer 1.

### Input / Output Boundaries

Use cases define their own input and output types — these are **not** HTTP DTOs.

```typescript
// modules/user/use-cases/create-user/create-user.input.ts

export class CreateUserInput {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {}
}
```

```typescript
// modules/user/use-cases/create-user/create-user.output.ts

export class CreateUserOutput {
  constructor(
    public readonly id: string,
    public readonly email: string,
    public readonly name: string,
    public readonly createdAt: Date,
  ) {}
}
```

### Use Case Implementation

```typescript
// modules/user/use-cases/create-user/create-user.use-case.ts

import { Injectable } from '@nestjs/common';
import { UserRepository } from '../../domain/repositories/user.repository.interface';
import { User } from '../../domain/entities/user.entity';
import { UserAlreadyExistsError } from '../../../shared/exceptions/domain.exception';
import { CreateUserInput } from './create-user.input';
import { CreateUserOutput } from './create-user.output';

@Injectable()
export class CreateUserUseCase {
  constructor(private readonly userRepo: UserRepository) {}

  async execute(input: CreateUserInput): Promise<CreateUserOutput> {
    // 1. Check business rules
    const existing = await this.userRepo.findByEmail(input.email);
    if (existing) {
      throw new UserAlreadyExistsError(input.email);
    }

    // 2. Create domain entity
    const user = User.create({ email: input.email, name: input.name });

    // 3. Persist via repository abstraction
    await this.userRepo.save(user);

    // 4. Return output boundary
    return new CreateUserOutput(
      user.id.value,
      user.email.toString(),
      user.name,
      user.createdAt,
    );
  }
}
```

```typescript
// modules/user/use-cases/get-user/get-user.input.ts

export class GetUserInput {
  constructor(public readonly id: string) {}
}
```

```typescript
// modules/user/use-cases/get-user/get-user.output.ts

export class GetUserOutput {
  constructor(
    public readonly id: string,
    public readonly email: string,
    public readonly name: string,
    public readonly createdAt: Date,
  ) {}
}
```

```typescript
// modules/user/use-cases/get-user/get-user.use-case.ts

import { Injectable } from '@nestjs/common';
import { UserRepository } from '../../domain/repositories/user.repository.interface';
import { UserNotFoundError } from '../../../shared/exceptions/domain.exception';
import { GetUserInput } from './get-user.input';
import { GetUserOutput } from './get-user.output';

@Injectable()
export class GetUserUseCase {
  constructor(private readonly userRepo: UserRepository) {}

  async execute(input: GetUserInput): Promise<GetUserOutput> {
    const user = await this.userRepo.findById(input.id);
    if (!user) {
      throw new UserNotFoundError(input.id);
    }

    return new GetUserOutput(
      user.id.value,
      user.email.toString(),
      user.name,
      user.createdAt,
    );
  }
}
```

> **Notice:** Use cases inject `UserRepository` directly (the abstract class). No symbols, no port directories. Simpler than hexagonal — but still fully decoupled from the database.

---

## Layer 3 — Interface Adapters

Converts data from the format most convenient for use cases and entities to the format most convenient for external agents (web, DB, etc.). This includes controllers, presenters, and gateways.

### Controller

```typescript
// modules/user/interface-adapters/controllers/user.controller.ts

import {
  Controller, Post, Get, Param, Body,
  HttpCode, HttpStatus,
} from '@nestjs/common';
import { CreateUserUseCase } from '../../use-cases/create-user/create-user.use-case';
import { GetUserUseCase } from '../../use-cases/get-user/get-user.use-case';
import { CreateUserInput } from '../../use-cases/create-user/create-user.input';
import { GetUserInput } from '../../use-cases/get-user/get-user.input';
import { CreateUserRequestDto } from '../dtos/create-user.request.dto';
import { UserResponseDto } from '../dtos/user.response.dto';
import { UserPresenter } from '../presenters/user.presenter';

@Controller('users')
export class UserController {
  constructor(
    private readonly createUserUseCase: CreateUserUseCase,
    private readonly getUserUseCase: GetUserUseCase,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() dto: CreateUserRequestDto): Promise<UserResponseDto> {
    const output = await this.createUserUseCase.execute(
      new CreateUserInput(dto.email, dto.name),
    );
    return UserPresenter.toResponse(output);
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<UserResponseDto> {
    const output = await this.getUserUseCase.execute(new GetUserInput(id));
    return UserPresenter.toResponse(output);
  }
}
```

> **Key difference from hexagonal:** The controller injects the use case class **directly** — no inbound port symbol indirection. The Dependency Rule is still satisfied because the controller (outer layer) depends on the use case (inner layer).

### Presenter

Transforms use case output into HTTP response format. Keeps formatting logic out of both the controller and the use case.

```typescript
// modules/user/interface-adapters/presenters/user.presenter.ts

import { CreateUserOutput } from '../../use-cases/create-user/create-user.output';
import { GetUserOutput } from '../../use-cases/get-user/get-user.output';
import { UserResponseDto } from '../dtos/user.response.dto';

export class UserPresenter {
  static toResponse(output: CreateUserOutput | GetUserOutput): UserResponseDto {
    return {
      id: output.id,
      email: output.email,
      name: output.name,
      createdAt: output.createdAt.toISOString(),
    };
  }
}
```

### DTOs

```typescript
// modules/user/interface-adapters/dtos/create-user.request.dto.ts

import { IsEmail, IsString, MinLength } from 'class-validator';

export class CreateUserRequestDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(2)
  name: string;
}
```

```typescript
// modules/user/interface-adapters/dtos/user.response.dto.ts

export class UserResponseDto {
  id: string;
  email: string;
  name: string;
  createdAt: string;
}
```

---

## Layer 4 — Frameworks & Drivers

The outermost layer. All framework-specific code lives here — TypeORM entities, external service clients, etc.

### TypeORM Repository Implementation

```typescript
// modules/user/frameworks/typeorm/user.typeorm.repository.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UserRepository } from '../../domain/repositories/user.repository.interface';
import { User } from '../../domain/entities/user.entity';
import { UserSchema } from './user.schema';
import { UserMapper } from './user.mapper';

@Injectable()
export class UserTypeOrmRepository extends UserRepository {
  constructor(
    @InjectRepository(UserSchema)
    private readonly repo: Repository<UserSchema>,
  ) {
    super();
  }

  async findById(id: string): Promise<User | null> {
    const record = await this.repo.findOne({ where: { id } });
    return record ? UserMapper.toDomain(record) : null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const record = await this.repo.findOne({ where: { email } });
    return record ? UserMapper.toDomain(record) : null;
  }

  async save(user: User): Promise<void> {
    const schema = UserMapper.toPersistence(user);
    await this.repo.save(schema);
  }

  async delete(id: string): Promise<void> {
    await this.repo.delete({ id });
  }
}
```

> **`extends` instead of `implements`:** Since `UserRepository` is an abstract class, the concrete repository _extends_ it. NestJS can then resolve `UserRepository` to `UserTypeOrmRepository` via standard provider binding.

### Schema

```typescript
// modules/user/frameworks/typeorm/user.schema.ts

import { Entity, PrimaryColumn, Column, CreateDateColumn } from 'typeorm';

@Entity('users')
export class UserSchema {
  @PrimaryColumn('uuid')
  id: string;

  @Column({ unique: true })
  email: string;

  @Column()
  name: string;

  @CreateDateColumn()
  createdAt: Date;
}
```

### Mapper

```typescript
// modules/user/frameworks/typeorm/user.mapper.ts

import { User } from '../../domain/entities/user.entity';
import { UserId } from '../../domain/value-objects/user-id.vo';
import { Email } from '../../domain/value-objects/email.vo';
import { UserSchema } from './user.schema';

export class UserMapper {
  static toDomain(record: UserSchema): User {
    return new User(
      new UserId(record.id),
      new Email(record.email),
      record.name,
      record.createdAt,
    );
  }

  static toPersistence(user: User): Partial<UserSchema> {
    return {
      id: user.id.value,
      email: user.email.toString(),
      name: user.name,
      createdAt: user.createdAt,
    };
  }
}
```

### External Service

```typescript
// modules/user/frameworks/services/email.service.ts

import { Injectable } from '@nestjs/common';

export abstract class EmailService {
  abstract sendWelcome(to: string): Promise<void>;
  abstract sendPasswordReset(to: string, token: string): Promise<void>;
}

@Injectable()
export class SendGridEmailService extends EmailService {
  async sendWelcome(to: string): Promise<void> {
    // SendGrid API call
    console.log(`[SendGrid] Welcome email sent to ${to}`);
  }

  async sendPasswordReset(to: string, token: string): Promise<void> {
    // SendGrid API call
    console.log(`[SendGrid] Password reset email sent to ${to}`);
  }
}
```

---

## Cross-Cutting Concerns

In pure Clean Architecture, cross-cutting concerns are kept simple — no explicit port/adapter split required.

### Logger (Simple Abstract Class)

```typescript
// shared/logger/app-logger.ts

export abstract class AppLogger {
  abstract log(message: string, context?: string): void;
  abstract error(message: string, trace?: string, context?: string): void;
  abstract warn(message: string, context?: string): void;
  abstract debug(message: string, context?: string): void;
}
```

```typescript
// shared/logger/nestjs-logger.ts

import { Injectable, Logger } from '@nestjs/common';
import { AppLogger } from './app-logger';

@Injectable()
export class NestJsLogger extends AppLogger {
  private readonly logger = new Logger();

  log(message: string, context?: string): void {
    this.logger.log(message, context);
  }

  error(message: string, trace?: string, context?: string): void {
    this.logger.error(message, trace, context);
  }

  warn(message: string, context?: string): void {
    this.logger.warn(message, context);
  }

  debug(message: string, context?: string): void {
    this.logger.debug(message, context);
  }
}
```

### Shared Module

```typescript
// shared/shared.module.ts

import { Global, Module } from '@nestjs/common';
import { AppLogger } from './logger/app-logger';
import { NestJsLogger } from './logger/nestjs-logger';

@Global()
@Module({
  providers: [
    { provide: AppLogger, useClass: NestJsLogger },
  ],
  exports: [AppLogger],
})
export class SharedModule {}
```

> **Simpler than hexagonal:** No `Symbol()` tokens. Abstract classes are their own injection tokens. Just `provide: AppLogger, useClass: NestJsLogger`.

---

## Module Wiring

```typescript
// modules/user/user.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

// Domain
import { UserRepository } from './domain/repositories/user.repository.interface';

// Use Cases
import { CreateUserUseCase } from './use-cases/create-user/create-user.use-case';
import { GetUserUseCase } from './use-cases/get-user/get-user.use-case';

// Interface Adapters
import { UserController } from './interface-adapters/controllers/user.controller';

// Frameworks
import { UserTypeOrmRepository } from './frameworks/typeorm/user.typeorm.repository';
import { UserSchema } from './frameworks/typeorm/user.schema';
import { EmailService, SendGridEmailService } from './frameworks/services/email.service';

@Module({
  imports: [TypeOrmModule.forFeature([UserSchema])],
  controllers: [UserController],
  providers: [
    // Use Cases (self-registered)
    CreateUserUseCase,
    GetUserUseCase,

    // Abstract → Concrete bindings
    { provide: UserRepository, useClass: UserTypeOrmRepository },
    { provide: EmailService,   useClass: SendGridEmailService },
  ],
})
export class UserModule {}
```

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { SharedModule } from './shared/shared.module';
import { UserModule } from './modules/user/user.module';

@Module({
  imports: [
    TypeOrmModule.forRoot({ /* ... */ }),
    SharedModule,
    UserModule,
  ],
})
export class AppModule {}
```

### Comparison: Module Wiring

| Pure Clean Architecture | Hexagonal (Ports & Adapters) |
|------------------------|-------------------------------|
| `provide: UserRepository` (abstract class) | `provide: USER_REPOSITORY_PORT` (Symbol) |
| Controller injects `CreateUserUseCase` directly | Controller injects `CREATE_USER_PORT` (Symbol) |
| No symbols needed | Every port boundary requires a `Symbol()` |

---

## Error Handling

### Domain Exceptions

```typescript
// shared/exceptions/domain.exception.ts

export class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class UserAlreadyExistsError extends DomainError {
  constructor(email: string) {
    super(`User with email "${email}" already exists`);
  }
}

export class UserNotFoundError extends DomainError {
  constructor(id: string) {
    super(`User with id "${id}" not found`);
  }
}
```

### Exception Filter

```typescript
// shared/filters/global-exception.filter.ts

import {
  Catch, ExceptionFilter, ArgumentsHost,
  HttpException,
} from '@nestjs/common';
import { Response } from 'express';
import { AppLogger } from '../logger/app-logger';
import { DomainError, UserNotFoundError } from '../exceptions/domain.exception';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(private readonly logger: AppLogger) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse<Response>();

    if (exception instanceof HttpException) {
      response.status(exception.getStatus()).json(exception.getResponse());
      return;
    }

    if (exception instanceof UserNotFoundError) {
      response.status(404).json({
        statusCode: 404,
        error: 'Not Found',
        message: exception.message,
      });
      return;
    }

    if (exception instanceof DomainError) {
      response.status(400).json({
        statusCode: 400,
        error: 'Business Rule Violation',
        message: exception.message,
      });
      return;
    }

    this.logger.error(
      exception instanceof Error ? exception.message : String(exception),
      exception instanceof Error ? exception.stack : undefined,
    );
    response.status(500).json({
      statusCode: 500,
      error: 'Internal Server Error',
    });
  }
}
```

---

## Testing Strategy

### Unit Test — Use Case

```typescript
// modules/user/use-cases/create-user/__tests__/create-user.use-case.spec.ts

import { CreateUserUseCase } from '../create-user.use-case';
import { UserRepository } from '../../../domain/repositories/user.repository.interface';
import { UserAlreadyExistsError } from '../../../../shared/exceptions/domain.exception';
import { CreateUserInput } from '../create-user.input';

describe('CreateUserUseCase', () => {
  let useCase: CreateUserUseCase;
  let mockRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockRepo = {
      findById: jest.fn(),
      findByEmail: jest.fn().mockResolvedValue(null),
      save: jest.fn().mockResolvedValue(undefined),
      delete: jest.fn(),
    } as any;

    useCase = new CreateUserUseCase(mockRepo);
  });

  it('should create a user successfully', async () => {
    const output = await useCase.execute(
      new CreateUserInput('john@example.com', 'John Doe'),
    );

    expect(output.id).toBeDefined();
    expect(output.email).toBe('john@example.com');
    expect(output.name).toBe('John Doe');
    expect(mockRepo.save).toHaveBeenCalledTimes(1);
  });

  it('should reject duplicate email', async () => {
    mockRepo.findByEmail.mockResolvedValueOnce({} as any);

    await expect(
      useCase.execute(new CreateUserInput('taken@example.com', 'Jane')),
    ).rejects.toThrow(UserAlreadyExistsError);

    expect(mockRepo.save).not.toHaveBeenCalled();
  });
});
```

### Unit Test — Entity

```typescript
// modules/user/domain/entities/__tests__/user.entity.spec.ts

import { User } from '../user.entity';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

describe('User Entity', () => {
  it('should create a valid user', () => {
    const user = User.create({ email: 'a@b.com', name: 'Alice' });
    expect(user.email.toString()).toBe('a@b.com');
    expect(user.name).toBe('Alice');
  });

  it('should reject invalid email', () => {
    expect(() => User.create({ email: 'not-an-email', name: 'Bob' }))
      .toThrow(DomainError);
  });

  it('should reject short name', () => {
    expect(() => User.create({ email: 'a@b.com', name: 'A' }))
      .toThrow(DomainError);
  });
});
```

### Testing Simplicity

| Aspect | Pure Clean Arch | Hexagonal |
|--------|----------------|-----------|
| Mocking repositories | Mock the abstract class | Mock the port interface + inject via Symbol |
| Testing controllers | Inject use case directly | Inject via `CREATE_USER_PORT` symbol |
| Setup complexity | Lower — fewer indirections | Higher — more tokens to configure |

---

## Quick Reference

| Component | Location | Layer |
|-----------|----------|-------|
| `User`, `Email`, `UserId` | `domain/entities/`, `domain/value-objects/` | Entities |
| `UserRepository` (abstract) | `domain/repositories/` | Entities |
| `CreateUserUseCase` | `use-cases/create-user/` | Use Cases |
| `CreateUserInput/Output` | `use-cases/create-user/` | Use Cases |
| `UserController` | `interface-adapters/controllers/` | Interface Adapters |
| `UserPresenter` | `interface-adapters/presenters/` | Interface Adapters |
| Request/Response DTOs | `interface-adapters/dtos/` | Interface Adapters |
| `UserTypeOrmRepository` | `frameworks/typeorm/` | Frameworks & Drivers |
| `UserSchema`, `UserMapper` | `frameworks/typeorm/` | Frameworks & Drivers |
| `AppLogger` | `shared/logger/` | Cross-cutting |

---

## When to Choose Which

| Choose **Pure Clean Architecture** when... | Choose **Hexagonal (Ports & Adapters)** when... |
|---|---|
| You want minimal boilerplate | You need maximum pluggability |
| Your team is new to clean architecture | You anticipate frequent adapter swaps |
| The project is small to medium | The project is large with many integration points |
| Direct use case injection is sufficient | You need formal driving/driven port contracts |
| Abstract classes as DI tokens are acceptable | You prefer explicit Symbol-based injection |

---

## Further Reading

- [The Clean Architecture — Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Clean Architecture Book (2017)](https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/)
- [NestJS Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
