# Clean Architecture + Ports & Adapters for NestJS (Production Grade)

---

## Overview

Clean Architecture combined with Hexagonal Architecture (Ports & Adapters) separates code into distinct layers where **inner layers never depend on outer layers**. This makes your codebase testable, maintainable, and resilient to change.

---

## Full Project Structure

```
src/
├── shared/
│   ├── ports/
│   │   ├── logger.port.ts
│   │   └── config.port.ts
│   └── shared.module.ts              # @Global()
│
├── infrastructure/
│   └── adapters/
│       ├── logger.winston.adapter.ts
│       └── config.nestjs.adapter.ts
│
└── modules/
    └── user/
        ├── domain/
        │   ├── entities/user.entity.ts
        │   └── value-objects/email.vo.ts
        ├── application/
        │   ├── ports/
        │   │   ├── inbound/
        │   │   │   └── create-user.port.ts
        │   │   └── outbound/
        │   │       ├── user.repository.port.ts
        │   │       └── email.service.port.ts
        │   └── use-cases/
        │       └── create-user.use-case.ts
        ├── infrastructure/
        │   ├── repositories/user.typeorm.adapter.ts
        │   ├── schemas/user.schema.ts
        │   └── mappers/user.mapper.ts
        ├── presentation/
        │   └── controllers/user.controller.ts
        └── user.module.ts
```

---

## Dependency Flow

```
Presentation           Application            Domain
────────────           ───────────            ──────
Controller      →      Inbound Port    →      (knows no one)
                        Use Case
                            │
                        Outbound Port
                            │
Infrastructure         ─────┘
──────────────
TypeORM Adapter   implements   UserRepositoryPort
SendGrid Adapter  implements   EmailServicePort
Winston Adapter   implements   LoggerPort          ← from shared/
NestConfig Adapter implements  ConfigPort          ← from shared/
```

---

## Layer 1: Domain (Innermost)

No external dependencies — zero imports from NestJS, TypeORM, or any library.

```typescript
// modules/user/domain/entities/user.entity.ts
export class User {
  constructor(
    public readonly id: UserId,
    public readonly email: Email,
    public name: string,
    public readonly createdAt: Date,
  ) {}

  static create(props: CreateUserProps): User {
    if (!props.email) throw new DomainError('Email is required');
    return new User(
      UserId.generate(),
      new Email(props.email),
      props.name,
      new Date(),
    );
  }

  changeName(name: string): void {
    if (!name || name.length < 2)
      throw new DomainError('Name must be at least 2 characters');
    this.name = name;
  }
}
```

```typescript
// modules/user/domain/value-objects/email.vo.ts
export class Email {
  private readonly value: string;

  constructor(email: string) {
    if (!this.isValid(email)) throw new DomainError(`Invalid email: ${email}`);
    this.value = email.toLowerCase();
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  toString(): string {
    return this.value;
  }
}
```

---

## Layer 2: Application (Use Cases + Ports)

Orchestrates business flow. Knows only the Domain layer. Defines all interfaces (Ports) for external dependencies.

### Inbound Ports (Driving)

Interfaces that Use Cases must implement — decouples Controllers from concrete Use Cases.

```typescript
// modules/user/application/ports/inbound/create-user.port.ts
export interface CreateUserPort {
  execute(command: CreateUserCommand): Promise<CreateUserResult>;
}

export const CREATE_USER_PORT = Symbol('CreateUserPort');
```

```typescript
// modules/user/application/ports/inbound/get-user.port.ts
export interface GetUserPort {
  execute(query: GetUserQuery): Promise<UserDto>;
}

export const GET_USER_PORT = Symbol('GetUserPort');
```

### Outbound Ports (Driven)

Interfaces for everything the application needs from the outside world.

```typescript
// modules/user/application/ports/outbound/user.repository.port.ts
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

### Use Case

```typescript
// modules/user/application/use-cases/create-user.use-case.ts
@Injectable()
export class CreateUserUseCase implements CreateUserPort {
  constructor(
    @Inject(USER_REPOSITORY_PORT)
    private readonly userRepo: UserRepositoryPort,

    @Inject(EMAIL_SERVICE_PORT)
    private readonly emailService: EmailServicePort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,

    @Inject(CONFIG_PORT)
    private readonly config: ConfigPort,
  ) {}

  async execute(command: CreateUserCommand): Promise<CreateUserResult> {
    this.logger.log('Creating user', CreateUserUseCase.name);

    const existing = await this.userRepo.findByEmail(command.email);
    if (existing) throw new UserAlreadyExistsError(command.email);

    const maxUsers = this.config.get<number>('MAX_USERS_PER_DAY');

    const user = User.create(command);
    await this.userRepo.save(user);
    await this.emailService.sendWelcome(user.email.toString());

    this.logger.log(`User created: ${user.id.value}`, CreateUserUseCase.name);
    return { id: user.id.value };
  }
}
```

---

## Layer 3: Infrastructure (Adapters — Outbound)

Implements Outbound Ports. All database, email, and external service logic lives here.

```typescript
// modules/user/infrastructure/repositories/user.typeorm.adapter.ts
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

  async save(user: User): Promise<void> {
    await this.repo.save(UserMapper.toPersistence(user));
  }

  async findByEmail(email: string): Promise<User | null> {
    const record = await this.repo.findOne({ where: { email } });
    return record ? UserMapper.toDomain(record) : null;
  }

  async delete(id: string): Promise<void> {
    await this.repo.delete({ id });
  }
}
```

```typescript
// modules/user/infrastructure/mappers/user.mapper.ts
export class UserMapper {
  static toDomain(record: UserSchema): User {
    return new User(
      new UserId(record.id),
      new Email(record.email),
      record.name,
      record.createdAt,
    );
  }

  static toPersistence(user: User): UserSchema {
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

## Layer 4: Presentation (Adapters — Inbound)

Transforms HTTP requests into Commands and delegates to Use Cases via Inbound Ports.

```typescript
// modules/user/presentation/controllers/user.controller.ts
@Controller('users')
@UseGuards(JwtAuthGuard)
export class UserController {
  constructor(
    @Inject(CREATE_USER_PORT)
    private readonly createUser: CreateUserPort,   // ← knows only the interface

    @Inject(GET_USER_PORT)
    private readonly getUser: GetUserPort,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() dto: CreateUserDto): Promise<UserResponseDto> {
    const result = await this.createUser.execute(new CreateUserCommand(dto));
    return UserResponseDto.from(result);
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<UserResponseDto> {
    const result = await this.getUser.execute(new GetUserQuery(id));
    return UserResponseDto.from(result);
  }
}
```

---

## Shared: Cross-Cutting Ports

Logger and Config are used across all layers — they belong in `shared/ports/`, not inside any single module.

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

### Cross-Cutting Adapters

```typescript
// infrastructure/adapters/logger.winston.adapter.ts
@Injectable()
export class WinstonLoggerAdapter implements LoggerPort {
  private readonly logger = createLogger({
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
@Injectable()
export class NestConfigAdapter implements ConfigPort {
  constructor(private readonly configService: ConfigService) {}

  get<T>(key: string): T {
    return this.configService.get<T>(key);
  }

  getOrThrow<T>(key: string): T {
    const value = this.configService.get<T>(key);
    if (value === undefined) throw new Error(`Config key "${key}" not found`);
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

## Module Binding

```typescript
// modules/user/user.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([UserSchema])],
  controllers: [UserController],
  providers: [
    // Use Cases
    CreateUserUseCase,
    GetUserUseCase,

    // Bind Inbound Ports → Use Cases
    { provide: CREATE_USER_PORT, useClass: CreateUserUseCase },
    { provide: GET_USER_PORT,    useClass: GetUserUseCase },

    // Bind Outbound Ports → Adapters
    { provide: USER_REPOSITORY_PORT, useClass: UserTypeOrmAdapter },
    { provide: EMAIL_SERVICE_PORT,   useClass: SendGridEmailAdapter },
  ],
})
export class UserModule {}
```

```typescript
// app.module.ts
@Module({
  imports: [
    SharedModule,   // ← import once, available everywhere via @Global()
    UserModule,
    OrderModule,
  ],
})
export class AppModule {}
```

---

## Error Handling

```typescript
// shared/exceptions/domain.exception.ts
export class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'DomainError';
  }
}

export class UserAlreadyExistsError extends DomainError {
  constructor(email: string) {
    super(`User with email ${email} already exists`);
  }
}
```

```typescript
// shared/filters/global-exception.filter.ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(@Inject(LOGGER_PORT) private readonly logger: LoggerPort) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    if (exception instanceof DomainError) {
      return response.status(400).json({
        statusCode: 400,
        error: 'Business Rule Violation',
        message: exception.message,
      });
    }

    this.logger.error(String(exception));
    return response.status(500).json({
      statusCode: 500,
      error: 'Internal Server Error',
    });
  }
}
```

---

## Unit Testing

Because everything is wired through interfaces, mocking is trivial.

```typescript
// create-user.use-case.spec.ts
describe('CreateUserUseCase', () => {
  let useCase: CreateUserUseCase;

  const mockRepo: UserRepositoryPort = {
    findById: jest.fn(),
    findByEmail: jest.fn().mockResolvedValue(null),
    save: jest.fn(),
    delete: jest.fn(),
  };

  const mockEmail: EmailServicePort = {
    sendWelcome: jest.fn(),
    sendPasswordReset: jest.fn(),
  };

  const mockLogger: LoggerPort = {
    log: jest.fn(),
    error: jest.fn(),
    warn: jest.fn(),
    debug: jest.fn(),
  };

  const mockConfig: ConfigPort = {
    get: jest.fn().mockReturnValue(100),
    getOrThrow: jest.fn(),
    isDevelopment: jest.fn().mockReturnValue(false),
    isProduction: jest.fn().mockReturnValue(true),
  };

  beforeEach(() => {
    useCase = new CreateUserUseCase(mockRepo, mockEmail, mockLogger, mockConfig);
  });

  it('should create a user successfully', async () => {
    const result = await useCase.execute({
      email: 'test@example.com',
      name: 'John Doe',
    });

    expect(result.id).toBeDefined();
    expect(mockRepo.save).toHaveBeenCalledTimes(1);
    expect(mockEmail.sendWelcome).toHaveBeenCalledWith('test@example.com');
  });

  it('should throw if user already exists', async () => {
    (mockRepo.findByEmail as jest.Mock).mockResolvedValueOnce(new User(...));
    await expect(useCase.execute({ email: 'exists@example.com', name: 'Jane' }))
      .rejects.toThrow(UserAlreadyExistsError);
  });
});
```

---

## Quick Reference Table

| What | Where | Type |
|---|---|---|
| `User`, `Email` (Value Object) | `domain/entities/` | Domain |
| `UserRepositoryPort` | `application/ports/outbound/` | Business-specific Port |
| `CreateUserPort` | `application/ports/inbound/` | Business-specific Port |
| `CreateUserUseCase` | `application/use-cases/` | Use Case |
| `UserTypeOrmAdapter` | `infrastructure/repositories/` | Outbound Adapter |
| `UserController` | `presentation/controllers/` | Inbound Adapter |
| `LoggerPort`, `ConfigPort` | `shared/ports/` | Cross-Cutting Port |
| `WinstonAdapter`, `NestConfigAdapter` | `infrastructure/adapters/` | Cross-Cutting Adapter |

---

## The 4 Golden Rules

1. **Dependency Rule** — Arrows always point inward toward the Domain. Inner layers never import from outer layers.

2. **Ports = Contracts** — An interface that declares *what* must be done, never *how* it is done.

3. **Adapters = Implementations** — Live in Infrastructure and Presentation. Swap them freely without touching business logic.

4. **Cross-Cutting → `shared/ports/`** — Logger and Config are used everywhere; they do not belong to any single business module.

---

## Key Benefits

**Swap Database** — Change `useClass: UserTypeOrmAdapter` to `useClass: UserPrismaAdapter`. Use Cases untouched.

**Swap Logger** — Winston → Pino? Create a new adapter implementing `LoggerPort`. Done.

**True Unit Tests** — Mock only the Port interfaces. No real DB, no HTTP, no side effects.

**Team Scalability** — Each team owns a layer. Interfaces act as contracts between teams.