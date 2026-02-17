# Clean Architecture + Ports & Adapters (Hexagonal) for NestJS

> Production-grade guide to building maintainable, testable, and framework-agnostic NestJS applications.

---

## Table of Contents

- [Core Principles](#core-principles)
- [Project Structure](#project-structure)
- [Dependency Flow](#dependency-flow)
- [Layer 1 — Domain](#layer-1--domain-innermost)
- [Layer 2 — Application](#layer-2--application-use-cases--ports)
- [Layer 3 — Infrastructure](#layer-3--infrastructure-outbound-adapters)
- [Layer 4 — Presentation](#layer-4--presentation-inbound-adapters)
- [Cross-Cutting Concerns](#cross-cutting-concerns)
- [Module Wiring](#module-wiring)
- [Error Handling](#error-handling)
- [Testing Strategy](#testing-strategy)
- [Swapping Implementations](#swapping-implementations)
- [Quick Reference](#quick-reference)

---

## Core Principles

| # | Rule | Description |
|---|------|-------------|
| 1 | **Dependency Rule** | Dependencies always point **inward**. Inner layers never import from outer layers. |
| 2 | **Ports = Contracts** | Interfaces that declare _what_ must be done, never _how_. |
| 3 | **Adapters = Implementations** | Concrete classes in Infrastructure/Presentation that fulfill port contracts. Freely swappable. |
| 4 | **Cross-Cutting → `shared/`** | Logger, Config, and other cross-cutting concerns live in `shared/ports/`, not in any business module. |

---

## Project Structure

```
src/
├── main.ts                                    # Bootstrap (entry point)
├── app.module.ts                              # Root module
│
├── shared/                                    # Cross-cutting concerns
│   ├── ports/
│   │   ├── logger.port.ts                   # LoggerPort interface + symbol
│   │   └── config.port.ts                   # ConfigPort interface + symbol
│   ├── exceptions/
│   │   └── domain.exception.ts              # Base DomainError
│   ├── filters/
│   │   └── global-exception.filter.ts       # Maps domain errors → HTTP responses
│   └── shared.module.ts                     # @Global() — provides LoggerPort, ConfigPort
│
├── infrastructure/                          # Global adapters (cross-cutting implementations)
│   └── adapters/
│       ├── logger.winston.adapter.ts        # Winston → LoggerPort
│       └── config.nestjs.adapter.ts         # NestJS ConfigService → ConfigPort
│
└── modules/
    └── user/                                # Feature module
        ├── domain/                          # Layer 1 — Pure business logic
        │   ├── entities/
        │   │   └── user.entity.ts
        │   └── value-objects/
        │       ├── email.vo.ts
        │       └── user-id.vo.ts
        │
        ├── application/                     # Layer 2 — Orchestration
        │   ├── ports/
        │   │   ├── inbound/                 # Driving ports (controllers call these)
        │   │   │   ├── create-user.port.ts
        │   │   │   └── get-user.port.ts
        │   │   └── outbound/                # Driven ports (use cases depend on these)
        │   │       ├── user.repository.port.ts
        │   │       └── email.service.port.ts
        │   ├── commands/
        │   │   └── create-user.command.ts
        │   ├── queries/
        │   │   └── get-user.query.ts
        │   ├── dtos/
        │   │   └── user.dto.ts
        │   └── use-cases/
        │       ├── create-user.use-case.ts
        │       └── get-user.use-case.ts
        │
        ├── infrastructure/                  # Layer 3 — Outbound adapters
        │   ├── repositories/
        │   │   └── user.typeorm.adapter.ts
        │   ├── services/
        │   │   └── email.sendgrid.adapter.ts
        │   ├── schemas/
        │   │   └── user.schema.ts
        │   └── mappers/
        │       └── user.mapper.ts
        │
        ├── presentation/                    # Layer 4 — Inbound adapters
        │   ├── controllers/
        │   │   └── user.controller.ts
        │   └── dtos/
        │       ├── create-user.request.dto.ts
        │       └── user.response.dto.ts
        │
        └── user.module.ts                   # Binds ports → adapters
```

---

## Dependency Flow

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Presentation        Application          Domain           │
│   ────────────        ───────────          ──────           │
│   Controller ──────▶  Inbound Port ──────▶ Entity           │
│                       Use Case             Value Object     │
│                           │                                 │
│                       Outbound Port                         │
│                           │                                 │
│   Infrastructure      ◀──┘                                  │
│   ──────────────                                            │
│   TypeORM Adapter   implements  UserRepositoryPort          │
│   SendGrid Adapter  implements  EmailServicePort            │
│   Winston Adapter   implements  LoggerPort      (shared)    │
│   NestConfig Adapter implements ConfigPort      (shared)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘

  Arrows always point INWARD → Domain has zero external dependencies.
```

---

## Layer 1 — Domain (Innermost)

**Zero** imports from NestJS, TypeORM, or any external library. Pure TypeScript only.

### Entity

```typescript
// modules/user/domain/entities/user.entity.ts

import { UserId } from '../value-objects/user-id.vo';
import { Email } from '../value-objects/email.vo';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

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

import { DomainError } from '../../../../shared/exceptions/domain.exception';

export class Email {
  private static readonly EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

  public readonly value: string;

  constructor(email: string) {
    if (!Email.EMAIL_REGEX.test(email)) {
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

> **Key:** Domain errors are the only shared dependency. They contain no framework code — just plain `Error` subclasses.

---

## Layer 2 — Application (Use Cases + Ports)

Orchestrates business flow. Depends **only** on the Domain layer. All external dependencies are defined as **port interfaces**.

### Inbound Ports (Driving)

These are the interfaces that controllers program against. They decouple presentation from business logic.

```typescript
// modules/user/application/ports/inbound/create-user.port.ts

import { CreateUserCommand } from '../../commands/create-user.command';

export interface CreateUserResult {
  id: string;
}

export interface CreateUserPort {
  execute(command: CreateUserCommand): Promise<CreateUserResult>;
}

export const CREATE_USER_PORT = Symbol('CreateUserPort');
```

```typescript
// modules/user/application/ports/inbound/get-user.port.ts

import { GetUserQuery } from '../../queries/get-user.query';
import { UserDto } from '../../dtos/user.dto';

export interface GetUserPort {
  execute(query: GetUserQuery): Promise<UserDto>;
}

export const GET_USER_PORT = Symbol('GetUserPort');
```

### Outbound Ports (Driven)

These are the interfaces that infrastructure adapters must implement.

```typescript
// modules/user/application/ports/outbound/user.repository.port.ts

import { User } from '../../../domain/entities/user.entity';

export interface UserRepositoryPort {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}

export const USER_REPOSITORY_PORT = Symbol('UserRepositoryPort');
```

```typescript
// modules/user/application/ports/outbound/email.service.port.ts

export interface EmailServicePort {
  sendWelcome(to: string): Promise<void>;
  sendPasswordReset(to: string, token: string): Promise<void>;
}

export const EMAIL_SERVICE_PORT = Symbol('EmailServicePort');
```

### Commands & Queries

```typescript
// modules/user/application/commands/create-user.command.ts

export class CreateUserCommand {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {}
}
```

```typescript
// modules/user/application/queries/get-user.query.ts

export class GetUserQuery {
  constructor(public readonly id: string) {}
}
```

### Use Case Implementation

```typescript
// modules/user/application/use-cases/create-user.use-case.ts

import { Injectable, Inject } from '@nestjs/common';
import { CreateUserPort, CreateUserResult } from '../ports/inbound/create-user.port';
import { CreateUserCommand } from '../commands/create-user.command';
import { USER_REPOSITORY_PORT, UserRepositoryPort } from '../ports/outbound/user.repository.port';
import { EMAIL_SERVICE_PORT, EmailServicePort } from '../ports/outbound/email.service.port';
import { LOGGER_PORT, LoggerPort } from '../../../../shared/ports/logger.port';
import { User } from '../../domain/entities/user.entity';
import { UserAlreadyExistsError } from '../../../../shared/exceptions/domain.exception';

@Injectable()
export class CreateUserUseCase implements CreateUserPort {
  constructor(
    @Inject(USER_REPOSITORY_PORT)
    private readonly userRepo: UserRepositoryPort,

    @Inject(EMAIL_SERVICE_PORT)
    private readonly emailService: EmailServicePort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,
  ) {}

  async execute(command: CreateUserCommand): Promise<CreateUserResult> {
    this.logger.log('Creating user', CreateUserUseCase.name);

    // 1. Check uniqueness
    const existing = await this.userRepo.findByEmail(command.email);
    if (existing) {
      throw new UserAlreadyExistsError(command.email);
    }

    // 2. Create domain entity
    const user = User.create({ email: command.email, name: command.name });

    // 3. Persist
    await this.userRepo.save(user);

    // 4. Side effects
    await this.emailService.sendWelcome(user.email.toString());

    this.logger.log(`User created: ${user.id}`, CreateUserUseCase.name);

    return { id: user.id.value };
  }
}
```

> **Note:** The use case has no idea about HTTP, databases, or email providers. It only talks through port interfaces.

---

## Layer 3 — Infrastructure (Outbound Adapters)

Implements outbound ports. All database, email, and external service logic is isolated here.

### Repository Adapter

```typescript
// modules/user/infrastructure/repositories/user.typeorm.adapter.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UserRepositoryPort } from '../../application/ports/outbound/user.repository.port';
import { User } from '../../domain/entities/user.entity';
import { UserSchema } from '../schemas/user.schema';
import { UserMapper } from '../mappers/user.mapper';

@Injectable()
export class UserTypeOrmAdapter implements UserRepositoryPort {
  constructor(
    @InjectRepository(UserSchema)
    private readonly repo: Repository<UserSchema>,
  ) {}

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

### Schema (TypeORM Entity)

```typescript
// modules/user/infrastructure/schemas/user.schema.ts

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

Converts between domain entities and persistence schemas. This boundary ensures the domain stays pure.

```typescript
// modules/user/infrastructure/mappers/user.mapper.ts

import { User } from '../../domain/entities/user.entity';
import { UserId } from '../../domain/value-objects/user-id.vo';
import { Email } from '../../domain/value-objects/email.vo';
import { UserSchema } from '../schemas/user.schema';

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

---

## Layer 4 — Presentation (Inbound Adapters)

Transforms HTTP requests into commands/queries and delegates to use cases via **inbound ports**.

### Controller

```typescript
// modules/user/presentation/controllers/user.controller.ts

import {
  Controller, Post, Get, Param, Body,
  Inject, HttpCode, HttpStatus,
} from '@nestjs/common';
import { CREATE_USER_PORT, CreateUserPort } from '../../application/ports/inbound/create-user.port';
import { GET_USER_PORT, GetUserPort } from '../../application/ports/inbound/get-user.port';
import { CreateUserCommand } from '../../application/commands/create-user.command';
import { GetUserQuery } from '../../application/queries/get-user.query';
import { CreateUserRequestDto } from '../dtos/create-user.request.dto';
import { UserResponseDto } from '../dtos/user.response.dto';

@Controller('users')
export class UserController {
  constructor(
    @Inject(CREATE_USER_PORT)
    private readonly createUser: CreateUserPort,

    @Inject(GET_USER_PORT)
    private readonly getUser: GetUserPort,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() dto: CreateUserRequestDto): Promise<UserResponseDto> {
    const result = await this.createUser.execute(
      new CreateUserCommand(dto.email, dto.name),
    );
    return { id: result.id };
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<UserResponseDto> {
    return this.getUser.execute(new GetUserQuery(id));
  }
}
```

### Request / Response DTOs

```typescript
// modules/user/presentation/dtos/create-user.request.dto.ts

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
// modules/user/presentation/dtos/user.response.dto.ts

export class UserResponseDto {
  id: string;
  email?: string;
  name?: string;
}
```

> **Key distinction:** Presentation DTOs handle validation and serialization. Application DTOs/Commands are plain data carriers with no decorators.

---

## Cross-Cutting Concerns

Logger and Config are used across all layers. They live in `shared/ports/` with their adapters in `infrastructure/adapters/`.

### Ports

```typescript
// shared/ports/logger.port.ts

export interface LoggerPort {
  log(message: string, context?: string): void;
  error(message: string, trace?: string, context?: string): void;
  warn(message: string, context?: string): void;
  debug(message: string, context?: string): void;
}

export const LOGGER_PORT = Symbol('LoggerPort');
```

```typescript
// shared/ports/config.port.ts

export interface ConfigPort {
  get<T>(key: string): T;
  getOrThrow<T>(key: string): T;
  isDevelopment(): boolean;
  isProduction(): boolean;
}

export const CONFIG_PORT = Symbol('ConfigPort');
```

### Adapters

```typescript
// infrastructure/adapters/logger.winston.adapter.ts

import { Injectable } from '@nestjs/common';
import { createLogger, transports, Logger } from 'winston';
import { LoggerPort } from '../../shared/ports/logger.port';

@Injectable()
export class WinstonLoggerAdapter implements LoggerPort {
  private readonly logger: Logger = createLogger({
    transports: [new transports.Console()],
  });

  log(message: string, context?: string): void {
    this.logger.info(message, { context });
  }

  error(message: string, trace?: string, context?: string): void {
    this.logger.error(message, { trace, context });
  }

  warn(message: string, context?: string): void {
    this.logger.warn(message, { context });
  }

  debug(message: string, context?: string): void {
    this.logger.debug(message, { context });
  }
}
```

```typescript
// infrastructure/adapters/config.nestjs.adapter.ts

import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { ConfigPort } from '../../shared/ports/config.port';

@Injectable()
export class NestConfigAdapter implements ConfigPort {
  constructor(private readonly configService: ConfigService) {}

  get<T>(key: string): T {
    return this.configService.get<T>(key)!;
  }

  getOrThrow<T>(key: string): T {
    const value = this.configService.get<T>(key);
    if (value === undefined) {
      throw new Error(`Missing config key: "${key}"`);
    }
    return value;
  }

  isDevelopment(): boolean {
    return this.get<string>('NODE_ENV') === 'development';
  }

  isProduction(): boolean {
    return this.get<string>('NODE_ENV') === 'production';
  }
}
```

### Shared Module

```typescript
// shared/shared.module.ts

import { Global, Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { LOGGER_PORT } from './ports/logger.port';
import { CONFIG_PORT } from './ports/config.port';
import { WinstonLoggerAdapter } from '../infrastructure/adapters/logger.winston.adapter';
import { NestConfigAdapter } from '../infrastructure/adapters/config.nestjs.adapter';

@Global()
@Module({
  imports: [ConfigModule.forRoot()],
  providers: [
    { provide: LOGGER_PORT, useClass: WinstonLoggerAdapter },
    { provide: CONFIG_PORT, useClass: NestConfigAdapter },
  ],
  exports: [LOGGER_PORT, CONFIG_PORT],
})
export class SharedModule {}
```

---

## Module Wiring

This is where NestJS DI binds port symbols to concrete adapter classes.

```typescript
// modules/user/user.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

// Ports
import { CREATE_USER_PORT } from './application/ports/inbound/create-user.port';
import { GET_USER_PORT } from './application/ports/inbound/get-user.port';
import { USER_REPOSITORY_PORT } from './application/ports/outbound/user.repository.port';
import { EMAIL_SERVICE_PORT } from './application/ports/outbound/email.service.port';

// Use Cases
import { CreateUserUseCase } from './application/use-cases/create-user.use-case';
import { GetUserUseCase } from './application/use-cases/get-user.use-case';

// Adapters
import { UserTypeOrmAdapter } from './infrastructure/repositories/user.typeorm.adapter';
import { SendGridEmailAdapter } from './infrastructure/services/email.sendgrid.adapter';
import { UserSchema } from './infrastructure/schemas/user.schema';

// Presentation
import { UserController } from './presentation/controllers/user.controller';

@Module({
  imports: [TypeOrmModule.forFeature([UserSchema])],
  controllers: [UserController],
  providers: [
    // Inbound Ports → Use Cases
    { provide: CREATE_USER_PORT, useClass: CreateUserUseCase },
    { provide: GET_USER_PORT,    useClass: GetUserUseCase },

    // Outbound Ports → Adapters
    { provide: USER_REPOSITORY_PORT, useClass: UserTypeOrmAdapter },
    { provide: EMAIL_SERVICE_PORT,   useClass: SendGridEmailAdapter },
  ],
})
export class UserModule {}
```

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { SharedModule } from './shared/shared.module';
import { UserModule } from './modules/user/user.module';

@Module({
  imports: [
    SharedModule,   // @Global() — available everywhere
    UserModule,
  ],
})
export class AppModule {}
```

> **Why symbol-based injection?** NestJS can't inject by interface at runtime (interfaces are erased). Symbols provide unique, collision-free injection tokens.

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

### Global Exception Filter

Maps domain errors to HTTP responses — keeps HTTP concerns out of the domain/application layers.

```typescript
// shared/filters/global-exception.filter.ts

import {
  Catch, ExceptionFilter, ArgumentsHost,
  HttpException, Inject,
} from '@nestjs/common';
import { Response } from 'express';
import { DomainError, UserNotFoundError } from '../exceptions/domain.exception';
import { LOGGER_PORT, LoggerPort } from '../ports/logger.port';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(
    @Inject(LOGGER_PORT) private readonly logger: LoggerPort,
  ) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse<Response>();

    // NestJS built-in exceptions (ValidationPipe, Guards, etc.)
    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      response.status(status).json(exception.getResponse());
      return;
    }

    // Domain errors → 400 or 404
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

    // Unexpected errors → 500
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

Because all dependencies are port interfaces, mocking is trivial and tests run without any infrastructure.

### Unit Test — Use Case

```typescript
// modules/user/application/use-cases/__tests__/create-user.use-case.spec.ts

import { CreateUserUseCase } from '../create-user.use-case';
import { UserRepositoryPort } from '../../ports/outbound/user.repository.port';
import { EmailServicePort } from '../../ports/outbound/email.service.port';
import { LoggerPort } from '../../../../../shared/ports/logger.port';
import { UserAlreadyExistsError } from '../../../../../shared/exceptions/domain.exception';

describe('CreateUserUseCase', () => {
  let useCase: CreateUserUseCase;
  let mockRepo: jest.Mocked<UserRepositoryPort>;
  let mockEmail: jest.Mocked<EmailServicePort>;
  let mockLogger: jest.Mocked<LoggerPort>;

  beforeEach(() => {
    mockRepo = {
      findById: jest.fn(),
      findByEmail: jest.fn().mockResolvedValue(null),
      save: jest.fn().mockResolvedValue(undefined),
      delete: jest.fn(),
    };

    mockEmail = {
      sendWelcome: jest.fn().mockResolvedValue(undefined),
      sendPasswordReset: jest.fn(),
    };

    mockLogger = {
      log: jest.fn(),
      error: jest.fn(),
      warn: jest.fn(),
      debug: jest.fn(),
    };

    useCase = new CreateUserUseCase(mockRepo, mockEmail, mockLogger);
  });

  it('should create a user and send welcome email', async () => {
    const result = await useCase.execute({
      email: 'john@example.com',
      name: 'John Doe',
    });

    expect(result.id).toBeDefined();
    expect(mockRepo.save).toHaveBeenCalledTimes(1);
    expect(mockEmail.sendWelcome).toHaveBeenCalledWith('john@example.com');
  });

  it('should reject duplicate email', async () => {
    mockRepo.findByEmail.mockResolvedValueOnce({} as any);

    await expect(
      useCase.execute({ email: 'taken@example.com', name: 'Jane' }),
    ).rejects.toThrow(UserAlreadyExistsError);

    expect(mockRepo.save).not.toHaveBeenCalled();
  });
});
```

### Unit Test — Domain Entity

```typescript
// modules/user/domain/entities/__tests__/user.entity.spec.ts

import { User } from '../user.entity';
import { DomainError } from '../../../../../shared/exceptions/domain.exception';

describe('User Entity', () => {
  it('should create a valid user', () => {
    const user = User.create({ email: 'test@example.com', name: 'Alice' });

    expect(user.id).toBeDefined();
    expect(user.email.toString()).toBe('test@example.com');
    expect(user.name).toBe('Alice');
  });

  it('should reject empty email', () => {
    expect(() => User.create({ email: '', name: 'Bob' }))
      .toThrow(DomainError);
  });

  it('should reject short name', () => {
    expect(() => User.create({ email: 'a@b.com', name: 'A' }))
      .toThrow(DomainError);
  });

  it('should change name', () => {
    const user = User.create({ email: 'a@b.com', name: 'Alice' });
    user.changeName('Bob');
    expect(user.name).toBe('Bob');
  });
});
```

### What to Test Where

| Layer | What to Test | Mocks Needed |
|-------|-------------|--------------|
| **Domain** | Entity creation, validation, business rules | None |
| **Application** | Use case orchestration, error paths | All outbound ports |
| **Infrastructure** | Repository queries, mapper correctness | Database (test container or in-memory) |
| **Presentation** | Request validation, HTTP status codes | Inbound ports |

---

## Swapping Implementations

One of the key benefits — change the adapter class in the module binding, and nothing else changes.

### Swap Database (TypeORM → Prisma)

```typescript
// user.module.ts — only this line changes:
{ provide: USER_REPOSITORY_PORT, useClass: UserPrismaAdapter },
```

### Swap Logger (Winston → Pino)

```typescript
// shared/shared.module.ts — only this line changes:
{ provide: LOGGER_PORT, useClass: PinoLoggerAdapter },
```

### Swap Email (SendGrid → AWS SES)

```typescript
// user.module.ts — only this line changes:
{ provide: EMAIL_SERVICE_PORT, useClass: AwsSesEmailAdapter },
```

> No use case, domain entity, or controller code changes. Ever.

---

## Quick Reference

| Component | Location | Layer |
|-----------|----------|-------|
| `User`, `Email`, `UserId` | `domain/entities/`, `domain/value-objects/` | Domain |
| `CreateUserPort` | `application/ports/inbound/` | Application (driving) |
| `UserRepositoryPort`, `EmailServicePort` | `application/ports/outbound/` | Application (driven) |
| `CreateUserUseCase` | `application/use-cases/` | Application |
| `UserTypeOrmAdapter` | `infrastructure/repositories/` | Infrastructure |
| `UserMapper`, `UserSchema` | `infrastructure/mappers/`, `infrastructure/schemas/` | Infrastructure |
| `UserController` | `presentation/controllers/` | Presentation |
| `LoggerPort`, `ConfigPort` | `shared/ports/` | Cross-cutting |
| `WinstonLoggerAdapter`, `NestConfigAdapter` | `infrastructure/adapters/` | Cross-cutting |

---

## Further Reading

- [Clean Architecture — Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Hexagonal Architecture — Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- [NestJS Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
