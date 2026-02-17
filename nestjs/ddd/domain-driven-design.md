# Domain-Driven Design (DDD) for NestJS

> Production-grade guide to implementing DDD tactical patterns in NestJS — Aggregates, Entities, Value Objects, Domain Events, Repositories, and Application Services.

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [Project Structure](#project-structure)
- [Bounded Context Map](#bounded-context-map)
- [Domain Layer](#domain-layer)
  - [Value Objects](#value-objects)
  - [Entities](#entities)
  - [Aggregate Root](#aggregate-root)
  - [Domain Events](#domain-events)
  - [Domain Services](#domain-services)
  - [Repository Interface](#repository-interface)
- [Application Layer](#application-layer)
  - [Application Services (Command Handlers)](#application-services)
  - [Query Services](#query-services)
  - [DTOs](#dtos)
- [Infrastructure Layer](#infrastructure-layer)
  - [Repository Implementation](#repository-implementation)
  - [Event Bus](#event-bus)
  - [Schema & Mapper](#schema--mapper)
- [Presentation Layer](#presentation-layer)
- [Module Wiring](#module-wiring)
- [Domain Events Flow](#domain-events-flow)
- [Error Handling](#error-handling)
- [Testing Strategy](#testing-strategy)
- [Quick Reference](#quick-reference)
- [DDD vs Clean Architecture](#ddd-vs-clean-architecture)

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Bounded Context** | A distinct area of the business with its own ubiquitous language and models. Each NestJS module = one bounded context. |
| **Aggregate** | A cluster of entities and value objects treated as a single unit for data changes. Has a root entity. |
| **Entity** | An object with a unique identity that persists over time. |
| **Value Object** | An immutable object defined by its attributes, not identity. Two VOs with the same values are equal. |
| **Domain Event** | Something that happened in the domain that other parts of the system care about. |
| **Repository** | Abstracts persistence. Operates on aggregates, not raw data. |
| **Domain Service** | Business logic that doesn't naturally belong to any single entity or value object. |
| **Application Service** | Orchestrates use cases. Thin layer that delegates to the domain. |
| **Ubiquitous Language** | The shared vocabulary between developers and domain experts, reflected in code. |

---

## Project Structure

```
src/
├── main.ts                                  # Bootstrap (entry point)
├── app.module.ts                            # Root module
│
├── shared/
│   ├── domain/
│   │   ├── aggregate-root.base.ts           # Base class for aggregate roots
│   │   ├── entity.base.ts                   # Base class for entities
│   │   ├── value-object.base.ts             # Base class for value objects
│   │   ├── domain-event.base.ts             # Base class for domain events
│   │   └── identifier.ts                    # Typed ID value object
│   ├── application/
│   │   └── event-bus.interface.ts           # Domain event bus contract
│   ├── infrastructure/
│   │   └── nestjs-event-bus.ts              # NestJS EventEmitter2 adapter
│   ├── exceptions/
│   │   └── domain.exception.ts
│   ├── filters/
│   │   └── global-exception.filter.ts
│   └── shared.module.ts
│
└── modules/
    │
    ├── order/                               # Bounded Context: Order
    │   ├── domain/
    │   │   ├── aggregates/
    │   │   │   └── order.aggregate.ts
    │   │   ├── entities/
    │   │   │   └── order-line-item.entity.ts
    │   │   ├── value-objects/
    │   │   │   ├── order-id.vo.ts
    │   │   │   ├── money.vo.ts
    │   │   │   └── address.vo.ts
    │   │   ├── events/
    │   │   │   ├── order-placed.event.ts
    │   │   │   └── order-cancelled.event.ts
    │   │   ├── services/
    │   │   │   └── order-pricing.service.ts
    │   │   └── repositories/
    │   │       └── order.repository.ts      # Abstract interface
    │   ├── application/
    │   │   ├── commands/
    │   │   │   ├── place-order/
    │   │   │   │   ├── place-order.command.ts
    │   │   │   │   └── place-order.handler.ts
    │   │   │   └── cancel-order/
    │   │   │       ├── cancel-order.command.ts
    │   │   │       └── cancel-order.handler.ts
    │   │   ├── queries/
    │   │   │   └── get-order/
    │   │   │       ├── get-order.query.ts
    │   │   │       └── get-order.handler.ts
    │   │   ├── event-handlers/
    │   │   │   └── on-order-placed.handler.ts
    │   │   └── dtos/
    │   │       └── order.dto.ts
    │   ├── infrastructure/
    │   │   ├── repositories/
    │   │   │   └── order.typeorm.repository.ts
    │   │   ├── schemas/
    │   │   │   ├── order.schema.ts
    │   │   │   └── order-line-item.schema.ts
    │   │   └── mappers/
    │   │       └── order.mapper.ts
    │   ├── presentation/
    │   │   ├── controllers/
    │   │   │   └── order.controller.ts
    │   │   └── dtos/
    │   │       ├── place-order.request.dto.ts
    │   │       └── order.response.dto.ts
    │   └── order.module.ts
    │
    └── user/                                # Bounded Context: User
        ├── domain/
        │   ├── aggregates/
        │   │   └── user.aggregate.ts
        │   ├── value-objects/
        │   │   ├── user-id.vo.ts
        │   │   └── email.vo.ts
        │   ├── events/
        │   │   └── user-registered.event.ts
        │   └── repositories/
        │       └── user.repository.ts
        ├── application/
        │   └── ...
        ├── infrastructure/
        │   └── ...
        ├── presentation/
        │   └── ...
        └── user.module.ts
```

> **Key:** Each module is a **Bounded Context** with its own domain model, events, and ubiquitous language. Modules communicate via domain events, not direct imports.

---

## Bounded Context Map

```
┌─────────────────────────────────────────────────────┐
│                    Application                       │
│                                                     │
│   ┌─────────────┐    Domain Events    ┌──────────┐  │
│   │             │ ──────────────────▶ │          │  │
│   │    User     │                     │  Order   │  │
│   │  (Context)  │ ◀────────────────── │ (Context)│  │
│   │             │    Domain Events    │          │  │
│   └─────────────┘                     └──────────┘  │
│                                                     │
│   Rules:                                            │
│   • No direct imports between bounded contexts      │
│   • Communication via domain events only            │
│   • Each context owns its own data/schema           │
│   • Shared concepts get their own value objects      │
│     per context (e.g., UserId in Order context       │
│     is a VO, not imported from User context)         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Domain Layer

The heart of DDD. All business rules live here. Zero framework dependencies.

### Shared Base Classes

#### Value Object Base

```typescript
// shared/domain/value-object.base.ts

export abstract class ValueObject<T> {
  protected readonly props: T;

  constructor(props: T) {
    this.props = Object.freeze(props);
  }

  equals(other: ValueObject<T>): boolean {
    if (!other) return false;
    return JSON.stringify(this.props) === JSON.stringify(other.props);
  }
}
```

#### Entity Base

```typescript
// shared/domain/entity.base.ts

export abstract class Entity<ID> {
  constructor(public readonly id: ID) {}

  equals(other: Entity<ID>): boolean {
    if (!other) return false;
    return JSON.stringify(this.id) === JSON.stringify(other.id);
  }
}
```

#### Domain Event Base

```typescript
// shared/domain/domain-event.base.ts

export abstract class DomainEvent {
  public readonly occurredOn: Date;

  constructor() {
    this.occurredOn = new Date();
  }

  abstract get eventName(): string;
}
```

#### Aggregate Root Base

```typescript
// shared/domain/aggregate-root.base.ts

import { Entity } from './entity.base';
import { DomainEvent } from './domain-event.base';

export abstract class AggregateRoot<ID> extends Entity<ID> {
  private _domainEvents: DomainEvent[] = [];

  get domainEvents(): ReadonlyArray<DomainEvent> {
    return [...this._domainEvents];
  }

  protected addDomainEvent(event: DomainEvent): void {
    this._domainEvents.push(event);
  }

  clearDomainEvents(): void {
    this._domainEvents = [];
  }
}
```

> **Aggregate Root** extends Entity and holds a list of domain events. Events are collected during business operations and dispatched after persistence.

---

### Value Objects

Immutable. No identity. Equality by value.

```typescript
// modules/order/domain/value-objects/order-id.vo.ts

import { randomUUID } from 'crypto';

export class OrderId {
  constructor(public readonly value: string) {}

  static generate(): OrderId {
    return new OrderId(randomUUID());
  }

  equals(other: OrderId): boolean {
    return this.value === other.value;
  }

  toString(): string {
    return this.value;
  }
}
```

```typescript
// modules/order/domain/value-objects/money.vo.ts

import { ValueObject } from '../../../../shared/domain/value-object.base';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

interface MoneyProps {
  amount: number;
  currency: string;
}

export class Money extends ValueObject<MoneyProps> {
  get amount(): number {
    return this.props.amount;
  }

  get currency(): string {
    return this.props.currency;
  }

  constructor(amount: number, currency: string) {
    if (amount < 0) throw new DomainError('Amount cannot be negative');
    if (!currency) throw new DomainError('Currency is required');
    super({ amount, currency });
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(Math.round(this.amount * factor * 100) / 100, this.currency);
  }

  private assertSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new DomainError(
        `Cannot operate on different currencies: ${this.currency} and ${other.currency}`,
      );
    }
  }
}
```

```typescript
// modules/order/domain/value-objects/address.vo.ts

import { ValueObject } from '../../../../shared/domain/value-object.base';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

interface AddressProps {
  street: string;
  city: string;
  postalCode: string;
  country: string;
}

export class Address extends ValueObject<AddressProps> {
  get street(): string { return this.props.street; }
  get city(): string { return this.props.city; }
  get postalCode(): string { return this.props.postalCode; }
  get country(): string { return this.props.country; }

  constructor(street: string, city: string, postalCode: string, country: string) {
    if (!street || !city || !postalCode || !country) {
      throw new DomainError('All address fields are required');
    }
    super({ street, city, postalCode, country });
  }
}
```

---

### Entities

Have identity. Mutable (within aggregate rules). Not directly persisted — only through the aggregate root.

```typescript
// modules/order/domain/entities/order-line-item.entity.ts

import { Entity } from '../../../../shared/domain/entity.base';
import { Money } from '../value-objects/money.vo';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

export class OrderLineItem extends Entity<string> {
  constructor(
    id: string,
    public readonly productId: string,
    public readonly productName: string,
    public quantity: number,
    public readonly unitPrice: Money,
  ) {
    super(id);
    if (quantity <= 0) {
      throw new DomainError('Quantity must be positive');
    }
  }

  get totalPrice(): Money {
    return this.unitPrice.multiply(this.quantity);
  }

  changeQuantity(newQuantity: number): void {
    if (newQuantity <= 0) {
      throw new DomainError('Quantity must be positive');
    }
    this.quantity = newQuantity;
  }
}
```

---

### Aggregate Root

The transactional boundary. All modifications go through the aggregate root. External code never reaches into child entities directly.

```typescript
// modules/order/domain/aggregates/order.aggregate.ts

import { AggregateRoot } from '../../../../shared/domain/aggregate-root.base';
import { OrderId } from '../value-objects/order-id.vo';
import { Money } from '../value-objects/money.vo';
import { Address } from '../value-objects/address.vo';
import { OrderLineItem } from '../entities/order-line-item.entity';
import { OrderPlacedEvent } from '../events/order-placed.event';
import { OrderCancelledEvent } from '../events/order-cancelled.event';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

export enum OrderStatus {
  DRAFT = 'DRAFT',
  PLACED = 'PLACED',
  CONFIRMED = 'CONFIRMED',
  SHIPPED = 'SHIPPED',
  DELIVERED = 'DELIVERED',
  CANCELLED = 'CANCELLED',
}

export class Order extends AggregateRoot<OrderId> {
  private _lineItems: OrderLineItem[];
  private _status: OrderStatus;

  constructor(
    id: OrderId,
    public readonly customerId: string,
    public readonly shippingAddress: Address,
    lineItems: OrderLineItem[],
    status: OrderStatus,
    public readonly createdAt: Date,
  ) {
    super(id);
    this._lineItems = lineItems;
    this._status = status;
  }

  // ── Factory ──────────────────────────────────────────────

  static create(
    customerId: string,
    shippingAddress: Address,
    lineItems: OrderLineItem[],
  ): Order {
    if (!lineItems.length) {
      throw new DomainError('Order must have at least one line item');
    }

    const order = new Order(
      OrderId.generate(),
      customerId,
      shippingAddress,
      lineItems,
      OrderStatus.DRAFT,
      new Date(),
    );

    return order;
  }

  // ── Queries ──────────────────────────────────────────────

  get status(): OrderStatus {
    return this._status;
  }

  get lineItems(): ReadonlyArray<OrderLineItem> {
    return [...this._lineItems];
  }

  get totalAmount(): Money {
    return this._lineItems.reduce(
      (sum, item) => sum.add(item.totalPrice),
      new Money(0, this._lineItems[0].unitPrice.currency),
    );
  }

  // ── Commands (business operations) ───────────────────────

  place(): void {
    if (this._status !== OrderStatus.DRAFT) {
      throw new DomainError(`Cannot place order in status: ${this._status}`);
    }
    this._status = OrderStatus.PLACED;

    // Raise domain event
    this.addDomainEvent(
      new OrderPlacedEvent(this.id.value, this.customerId, this.totalAmount),
    );
  }

  cancel(reason: string): void {
    const cancellable = [OrderStatus.DRAFT, OrderStatus.PLACED, OrderStatus.CONFIRMED];
    if (!cancellable.includes(this._status)) {
      throw new DomainError(`Cannot cancel order in status: ${this._status}`);
    }
    this._status = OrderStatus.CANCELLED;

    this.addDomainEvent(
      new OrderCancelledEvent(this.id.value, reason),
    );
  }

  addLineItem(item: OrderLineItem): void {
    if (this._status !== OrderStatus.DRAFT) {
      throw new DomainError('Can only modify draft orders');
    }
    this._lineItems.push(item);
  }

  removeLineItem(itemId: string): void {
    if (this._status !== OrderStatus.DRAFT) {
      throw new DomainError('Can only modify draft orders');
    }
    const index = this._lineItems.findIndex((i) => i.id === itemId);
    if (index === -1) {
      throw new DomainError(`Line item ${itemId} not found`);
    }
    this._lineItems.splice(index, 1);

    if (this._lineItems.length === 0) {
      throw new DomainError('Order must have at least one line item');
    }
  }
}
```

> **Aggregate invariants:** The Order enforces that it always has line items, only draft orders can be modified, and state transitions follow valid paths. These rules are **in the domain**, not scattered across services.

---

### Domain Events

Events that signal something meaningful happened. Collected by the aggregate root, dispatched after persistence.

```typescript
// modules/order/domain/events/order-placed.event.ts

import { DomainEvent } from '../../../../shared/domain/domain-event.base';
import { Money } from '../value-objects/money.vo';

export class OrderPlacedEvent extends DomainEvent {
  constructor(
    public readonly orderId: string,
    public readonly customerId: string,
    public readonly totalAmount: Money,
  ) {
    super();
  }

  get eventName(): string {
    return 'order.placed';
  }
}
```

```typescript
// modules/order/domain/events/order-cancelled.event.ts

import { DomainEvent } from '../../../../shared/domain/domain-event.base';

export class OrderCancelledEvent extends DomainEvent {
  constructor(
    public readonly orderId: string,
    public readonly reason: string,
  ) {
    super();
  }

  get eventName(): string {
    return 'order.cancelled';
  }
}
```

---

### Domain Services

Business logic that spans multiple aggregates or doesn't fit in a single entity.

```typescript
// modules/order/domain/services/order-pricing.service.ts

import { Money } from '../value-objects/money.vo';
import { OrderLineItem } from '../entities/order-line-item.entity';

export class OrderPricingService {
  /**
   * Calculates discount based on the number of items.
   * Domain rule: 10+ items get 10% off, 20+ items get 15% off.
   */
  static calculateDiscount(lineItems: ReadonlyArray<OrderLineItem>): Money {
    const total = lineItems.reduce(
      (sum, item) => sum.add(item.totalPrice),
      new Money(0, lineItems[0].unitPrice.currency),
    );

    const totalQuantity = lineItems.reduce((sum, item) => sum + item.quantity, 0);

    if (totalQuantity >= 20) {
      return total.multiply(0.15);
    }
    if (totalQuantity >= 10) {
      return total.multiply(0.10);
    }

    return new Money(0, total.currency);
  }
}
```

> **Domain services are stateless.** They contain pure business logic. They do NOT depend on repositories, HTTP, or any infrastructure.

---

### Repository Interface

Lives in the domain layer. Defined per aggregate root (not per entity).

```typescript
// modules/order/domain/repositories/order.repository.ts

import { Order } from '../aggregates/order.aggregate';

export abstract class OrderRepository {
  abstract findById(id: string): Promise<Order | null>;
  abstract findByCustomerId(customerId: string): Promise<Order[]>;
  abstract save(order: Order): Promise<void>;
  abstract delete(id: string): Promise<void>;
}
```

> **One repository per aggregate.** You never have a repository for `OrderLineItem` — it's only accessible through the `Order` aggregate root.

---

## Application Layer

Thin orchestration layer. Coordinates use cases by:
1. Loading aggregates from repositories
2. Calling domain methods
3. Persisting changes
4. Dispatching domain events

### Event Bus Interface

```typescript
// shared/application/event-bus.interface.ts

import { DomainEvent } from '../domain/domain-event.base';

export abstract class EventBus {
  abstract publish(event: DomainEvent): Promise<void>;
  abstract publishAll(events: DomainEvent[]): Promise<void>;
}
```

### Application Services

```typescript
// modules/order/application/commands/place-order/place-order.command.ts

export class PlaceOrderCommand {
  constructor(
    public readonly customerId: string,
    public readonly shippingAddress: {
      street: string;
      city: string;
      postalCode: string;
      country: string;
    },
    public readonly items: Array<{
      productId: string;
      productName: string;
      quantity: number;
      unitPrice: number;
      currency: string;
    }>,
  ) {}
}
```

```typescript
// modules/order/application/commands/place-order/place-order.handler.ts

import { Injectable } from '@nestjs/common';
import { OrderRepository } from '../../../domain/repositories/order.repository';
import { EventBus } from '../../../../../shared/application/event-bus.interface';
import { Order } from '../../../domain/aggregates/order.aggregate';
import { OrderLineItem } from '../../../domain/entities/order-line-item.entity';
import { Address } from '../../../domain/value-objects/address.vo';
import { Money } from '../../../domain/value-objects/money.vo';
import { PlaceOrderCommand } from './place-order.command';
import { randomUUID } from 'crypto';

@Injectable()
export class PlaceOrderHandler {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly eventBus: EventBus,
  ) {}

  async execute(command: PlaceOrderCommand): Promise<{ orderId: string }> {
    // 1. Build value objects and entities from command data
    const address = new Address(
      command.shippingAddress.street,
      command.shippingAddress.city,
      command.shippingAddress.postalCode,
      command.shippingAddress.country,
    );

    const lineItems = command.items.map(
      (item) =>
        new OrderLineItem(
          randomUUID(),
          item.productId,
          item.productName,
          item.quantity,
          new Money(item.unitPrice, item.currency),
        ),
    );

    // 2. Create aggregate (domain factory applies business rules)
    const order = Order.create(command.customerId, address, lineItems);

    // 3. Invoke domain behavior
    order.place();

    // 4. Persist
    await this.orderRepo.save(order);

    // 5. Dispatch domain events
    await this.eventBus.publishAll(order.domainEvents);
    order.clearDomainEvents();

    return { orderId: order.id.value };
  }
}
```

```typescript
// modules/order/application/commands/cancel-order/cancel-order.command.ts

export class CancelOrderCommand {
  constructor(
    public readonly orderId: string,
    public readonly reason: string,
  ) {}
}
```

```typescript
// modules/order/application/commands/cancel-order/cancel-order.handler.ts

import { Injectable } from '@nestjs/common';
import { OrderRepository } from '../../../domain/repositories/order.repository';
import { EventBus } from '../../../../../shared/application/event-bus.interface';
import { CancelOrderCommand } from './cancel-order.command';
import { DomainError } from '../../../../../shared/exceptions/domain.exception';

@Injectable()
export class CancelOrderHandler {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly eventBus: EventBus,
  ) {}

  async execute(command: CancelOrderCommand): Promise<void> {
    const order = await this.orderRepo.findById(command.orderId);
    if (!order) {
      throw new DomainError(`Order ${command.orderId} not found`);
    }

    // Domain logic — the aggregate decides if cancellation is allowed
    order.cancel(command.reason);

    await this.orderRepo.save(order);

    await this.eventBus.publishAll(order.domainEvents);
    order.clearDomainEvents();
  }
}
```

> **Notice the pattern:** Load → call domain method → persist → dispatch events. Application services are thin — all business rules live in the aggregate.

### Query Services

```typescript
// modules/order/application/queries/get-order/get-order.query.ts

export class GetOrderQuery {
  constructor(public readonly orderId: string) {}
}
```

```typescript
// modules/order/application/queries/get-order/get-order.handler.ts

import { Injectable } from '@nestjs/common';
import { OrderRepository } from '../../../domain/repositories/order.repository';
import { GetOrderQuery } from './get-order.query';
import { OrderDto } from '../../dtos/order.dto';
import { DomainError } from '../../../../../shared/exceptions/domain.exception';

@Injectable()
export class GetOrderHandler {
  constructor(private readonly orderRepo: OrderRepository) {}

  async execute(query: GetOrderQuery): Promise<OrderDto> {
    const order = await this.orderRepo.findById(query.orderId);
    if (!order) {
      throw new DomainError(`Order ${query.orderId} not found`);
    }

    return {
      id: order.id.value,
      customerId: order.customerId,
      status: order.status,
      totalAmount: order.totalAmount.amount,
      currency: order.totalAmount.currency,
      lineItems: order.lineItems.map((item) => ({
        id: item.id,
        productId: item.productId,
        productName: item.productName,
        quantity: item.quantity,
        unitPrice: item.unitPrice.amount,
        totalPrice: item.totalPrice.amount,
      })),
      createdAt: order.createdAt,
    };
  }
}
```

### Application DTOs

```typescript
// modules/order/application/dtos/order.dto.ts

export interface OrderDto {
  id: string;
  customerId: string;
  status: string;
  totalAmount: number;
  currency: string;
  lineItems: Array<{
    id: string;
    productId: string;
    productName: string;
    quantity: number;
    unitPrice: number;
    totalPrice: number;
  }>;
  createdAt: Date;
}
```

### Event Handlers (React to Domain Events)

```typescript
// modules/order/application/event-handlers/on-order-placed.handler.ts

import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { OrderPlacedEvent } from '../../domain/events/order-placed.event';

@Injectable()
export class OnOrderPlacedHandler {
  @OnEvent('order.placed')
  async handle(event: OrderPlacedEvent): Promise<void> {
    // React to the event:
    // - Send confirmation email
    // - Update inventory
    // - Notify warehouse
    console.log(
      `[Event] Order ${event.orderId} placed by customer ${event.customerId}` +
      ` for ${event.totalAmount.amount} ${event.totalAmount.currency}`,
    );
  }
}
```

---

## Infrastructure Layer

Implements repository and event bus abstractions with concrete frameworks.

### Repository Implementation

```typescript
// modules/order/infrastructure/repositories/order.typeorm.repository.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { OrderRepository } from '../../domain/repositories/order.repository';
import { Order } from '../../domain/aggregates/order.aggregate';
import { OrderSchema } from '../schemas/order.schema';
import { OrderMapper } from '../mappers/order.mapper';

@Injectable()
export class OrderTypeOrmRepository extends OrderRepository {
  constructor(
    @InjectRepository(OrderSchema)
    private readonly repo: Repository<OrderSchema>,
  ) {
    super();
  }

  async findById(id: string): Promise<Order | null> {
    const record = await this.repo.findOne({
      where: { id },
      relations: ['lineItems'],
    });
    return record ? OrderMapper.toDomain(record) : null;
  }

  async findByCustomerId(customerId: string): Promise<Order[]> {
    const records = await this.repo.find({
      where: { customerId },
      relations: ['lineItems'],
    });
    return records.map(OrderMapper.toDomain);
  }

  async save(order: Order): Promise<void> {
    const schema = OrderMapper.toPersistence(order);
    await this.repo.save(schema);
  }

  async delete(id: string): Promise<void> {
    await this.repo.delete({ id });
  }
}
```

### Event Bus Implementation

```typescript
// shared/infrastructure/nestjs-event-bus.ts

import { Injectable } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { EventBus } from '../application/event-bus.interface';
import { DomainEvent } from '../domain/domain-event.base';

@Injectable()
export class NestJsEventBus extends EventBus {
  constructor(private readonly emitter: EventEmitter2) {
    super();
  }

  async publish(event: DomainEvent): Promise<void> {
    this.emitter.emit(event.eventName, event);
  }

  async publishAll(events: DomainEvent[]): Promise<void> {
    for (const event of events) {
      await this.publish(event);
    }
  }
}
```

### Schema & Mapper

```typescript
// modules/order/infrastructure/schemas/order.schema.ts

import {
  Entity, PrimaryColumn, Column, CreateDateColumn, OneToMany,
} from 'typeorm';
import { OrderLineItemSchema } from './order-line-item.schema';

@Entity('orders')
export class OrderSchema {
  @PrimaryColumn('uuid')
  id: string;

  @Column()
  customerId: string;

  @Column()
  status: string;

  @Column()
  shippingStreet: string;

  @Column()
  shippingCity: string;

  @Column()
  shippingPostalCode: string;

  @Column()
  shippingCountry: string;

  @OneToMany(() => OrderLineItemSchema, (item) => item.order, { cascade: true })
  lineItems: OrderLineItemSchema[];

  @CreateDateColumn()
  createdAt: Date;
}
```

```typescript
// modules/order/infrastructure/schemas/order-line-item.schema.ts

import { Entity, PrimaryColumn, Column, ManyToOne } from 'typeorm';
import { OrderSchema } from './order.schema';

@Entity('order_line_items')
export class OrderLineItemSchema {
  @PrimaryColumn('uuid')
  id: string;

  @Column()
  productId: string;

  @Column()
  productName: string;

  @Column()
  quantity: number;

  @Column('decimal')
  unitPrice: number;

  @Column()
  currency: string;

  @ManyToOne(() => OrderSchema, (order) => order.lineItems)
  order: OrderSchema;
}
```

```typescript
// modules/order/infrastructure/mappers/order.mapper.ts

import { Order, OrderStatus } from '../../domain/aggregates/order.aggregate';
import { OrderId } from '../../domain/value-objects/order-id.vo';
import { Address } from '../../domain/value-objects/address.vo';
import { Money } from '../../domain/value-objects/money.vo';
import { OrderLineItem } from '../../domain/entities/order-line-item.entity';
import { OrderSchema } from '../schemas/order.schema';
import { OrderLineItemSchema } from '../schemas/order-line-item.schema';

export class OrderMapper {
  static toDomain(record: OrderSchema): Order {
    const lineItems = record.lineItems.map(
      (item) =>
        new OrderLineItem(
          item.id,
          item.productId,
          item.productName,
          item.quantity,
          new Money(Number(item.unitPrice), item.currency),
        ),
    );

    return new Order(
      new OrderId(record.id),
      record.customerId,
      new Address(
        record.shippingStreet,
        record.shippingCity,
        record.shippingPostalCode,
        record.shippingCountry,
      ),
      lineItems,
      record.status as OrderStatus,
      record.createdAt,
    );
  }

  static toPersistence(order: Order): Partial<OrderSchema> {
    return {
      id: order.id.value,
      customerId: order.customerId,
      status: order.status,
      shippingStreet: order.shippingAddress.street,
      shippingCity: order.shippingAddress.city,
      shippingPostalCode: order.shippingAddress.postalCode,
      shippingCountry: order.shippingAddress.country,
      lineItems: order.lineItems.map((item) => ({
        id: item.id,
        productId: item.productId,
        productName: item.productName,
        quantity: item.quantity,
        unitPrice: item.unitPrice.amount,
        currency: item.unitPrice.currency,
      })) as OrderLineItemSchema[],
      createdAt: order.createdAt,
    };
  }
}
```

> **Mapper rehydrates the full aggregate** — including all child entities and value objects. The aggregate is always loaded as a whole.

---

## Presentation Layer

```typescript
// modules/order/presentation/dtos/place-order.request.dto.ts

import { IsString, IsArray, ValidateNested, IsNumber, Min, MinLength } from 'class-validator';
import { Type } from 'class-transformer';

class AddressDto {
  @IsString() @MinLength(1) street: string;
  @IsString() @MinLength(1) city: string;
  @IsString() @MinLength(1) postalCode: string;
  @IsString() @MinLength(1) country: string;
}

class LineItemDto {
  @IsString() productId: string;
  @IsString() productName: string;
  @IsNumber() @Min(1) quantity: number;
  @IsNumber() @Min(0) unitPrice: number;
  @IsString() currency: string;
}

export class PlaceOrderRequestDto {
  @IsString()
  customerId: string;

  @ValidateNested()
  @Type(() => AddressDto)
  shippingAddress: AddressDto;

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => LineItemDto)
  items: LineItemDto[];
}
```

```typescript
// modules/order/presentation/dtos/order.response.dto.ts

export class OrderResponseDto {
  id: string;
  customerId: string;
  status: string;
  totalAmount: number;
  currency: string;
  lineItems: Array<{
    id: string;
    productName: string;
    quantity: number;
    unitPrice: number;
    totalPrice: number;
  }>;
  createdAt: string;
}
```

```typescript
// modules/order/presentation/controllers/order.controller.ts

import {
  Controller, Post, Get, Patch, Param, Body,
  HttpCode, HttpStatus,
} from '@nestjs/common';
import { PlaceOrderHandler } from '../../application/commands/place-order/place-order.handler';
import { CancelOrderHandler } from '../../application/commands/cancel-order/cancel-order.handler';
import { GetOrderHandler } from '../../application/queries/get-order/get-order.handler';
import { PlaceOrderCommand } from '../../application/commands/place-order/place-order.command';
import { CancelOrderCommand } from '../../application/commands/cancel-order/cancel-order.command';
import { GetOrderQuery } from '../../application/queries/get-order/get-order.query';
import { PlaceOrderRequestDto } from '../dtos/place-order.request.dto';
import { OrderResponseDto } from '../dtos/order.response.dto';

@Controller('orders')
export class OrderController {
  constructor(
    private readonly placeOrderHandler: PlaceOrderHandler,
    private readonly cancelOrderHandler: CancelOrderHandler,
    private readonly getOrderHandler: GetOrderHandler,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async place(@Body() dto: PlaceOrderRequestDto): Promise<{ orderId: string }> {
    return this.placeOrderHandler.execute(
      new PlaceOrderCommand(dto.customerId, dto.shippingAddress, dto.items),
    );
  }

  @Patch(':id/cancel')
  @HttpCode(HttpStatus.OK)
  async cancel(
    @Param('id') id: string,
    @Body('reason') reason: string,
  ): Promise<void> {
    return this.cancelOrderHandler.execute(new CancelOrderCommand(id, reason));
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<OrderResponseDto> {
    const dto = await this.getOrderHandler.execute(new GetOrderQuery(id));
    return {
      ...dto,
      createdAt: dto.createdAt.toISOString(),
    };
  }
}
```

---

## Module Wiring

```typescript
// modules/order/order.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

// Domain
import { OrderRepository } from './domain/repositories/order.repository';

// Application
import { PlaceOrderHandler } from './application/commands/place-order/place-order.handler';
import { CancelOrderHandler } from './application/commands/cancel-order/cancel-order.handler';
import { GetOrderHandler } from './application/queries/get-order/get-order.handler';
import { OnOrderPlacedHandler } from './application/event-handlers/on-order-placed.handler';

// Infrastructure
import { OrderTypeOrmRepository } from './infrastructure/repositories/order.typeorm.repository';
import { OrderSchema } from './infrastructure/schemas/order.schema';
import { OrderLineItemSchema } from './infrastructure/schemas/order-line-item.schema';

// Presentation
import { OrderController } from './presentation/controllers/order.controller';

@Module({
  imports: [TypeOrmModule.forFeature([OrderSchema, OrderLineItemSchema])],
  controllers: [OrderController],
  providers: [
    // Command & Query Handlers
    PlaceOrderHandler,
    CancelOrderHandler,
    GetOrderHandler,

    // Event Handlers
    OnOrderPlacedHandler,

    // Repository binding
    { provide: OrderRepository, useClass: OrderTypeOrmRepository },
  ],
})
export class OrderModule {}
```

```typescript
// shared/shared.module.ts

import { Global, Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';
import { EventBus } from './application/event-bus.interface';
import { NestJsEventBus } from './infrastructure/nestjs-event-bus';

@Global()
@Module({
  imports: [EventEmitterModule.forRoot()],
  providers: [
    { provide: EventBus, useClass: NestJsEventBus },
  ],
  exports: [EventBus],
})
export class SharedModule {}
```

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { SharedModule } from './shared/shared.module';
import { OrderModule } from './modules/order/order.module';
import { UserModule } from './modules/user/user.module';

@Module({
  imports: [
    TypeOrmModule.forRoot({ /* ... */ }),
    SharedModule,
    OrderModule,
    UserModule,
  ],
})
export class AppModule {}
```

---

## Domain Events Flow

```
┌───────────┐     ┌──────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│Controller │────▶│ Handler  │────▶│ Aggregate │────▶│   Repo    │────▶│    DB     │
│ (HTTP)    │     │(App Svc) │     │ (Domain)  │     │  .save()  │     │           │
└───────────┘     └────┬─────┘     └───────────┘     └───────────┘     └───────────┘
                       │
                       │ after persist
                       ▼
                  ┌──────────┐
                  │ EventBus │
                  │.publishAll│
                  └────┬─────┘
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
         ┌────────┐ ┌───────┐ ┌──────────┐
         │ Send   │ │Update │ │ Notify   │
         │ Email  │ │Stock  │ │Warehouse │
         └────────┘ └───────┘ └──────────┘

  Events are collected DURING domain operations,
  dispatched AFTER successful persistence.
```

---

## Error Handling

```typescript
// shared/exceptions/domain.exception.ts

export class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
  }
}
```

```typescript
// shared/filters/global-exception.filter.ts

import {
  Catch, ExceptionFilter, ArgumentsHost, HttpException,
} from '@nestjs/common';
import { Response } from 'express';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse<Response>();

    if (exception instanceof HttpException) {
      response.status(exception.getStatus()).json(exception.getResponse());
      return;
    }

    if (exception instanceof DomainError) {
      const status = exception.message.includes('not found') ? 404 : 400;
      response.status(status).json({
        statusCode: status,
        error: status === 404 ? 'Not Found' : 'Business Rule Violation',
        message: exception.message,
      });
      return;
    }

    console.error(exception);
    response.status(500).json({
      statusCode: 500,
      error: 'Internal Server Error',
    });
  }
}
```

---

## Testing Strategy

### Unit Test — Aggregate

The most important tests in DDD. Test the aggregate's business rules in isolation.

```typescript
// modules/order/domain/aggregates/__tests__/order.aggregate.spec.ts

import { Order, OrderStatus } from '../order.aggregate';
import { OrderLineItem } from '../../entities/order-line-item.entity';
import { Address } from '../../value-objects/address.vo';
import { Money } from '../../value-objects/money.vo';
import { DomainError } from '../../../../../shared/exceptions/domain.exception';

const makeLineItem = (overrides?: Partial<{ quantity: number; price: number }>) =>
  new OrderLineItem(
    'item-1',
    'prod-1',
    'Widget',
    overrides?.quantity ?? 2,
    new Money(overrides?.price ?? 10, 'USD'),
  );

const makeAddress = () =>
  new Address('123 Main St', 'Springfield', '62701', 'US');

describe('Order Aggregate', () => {
  it('should create a draft order', () => {
    const order = Order.create('cust-1', makeAddress(), [makeLineItem()]);

    expect(order.status).toBe(OrderStatus.DRAFT);
    expect(order.lineItems).toHaveLength(1);
    expect(order.totalAmount.amount).toBe(20); // 2 × $10
  });

  it('should reject order with no items', () => {
    expect(() => Order.create('cust-1', makeAddress(), []))
      .toThrow(DomainError);
  });

  it('should place an order and emit event', () => {
    const order = Order.create('cust-1', makeAddress(), [makeLineItem()]);
    order.place();

    expect(order.status).toBe(OrderStatus.PLACED);
    expect(order.domainEvents).toHaveLength(1);
    expect(order.domainEvents[0].eventName).toBe('order.placed');
  });

  it('should not place an already-placed order', () => {
    const order = Order.create('cust-1', makeAddress(), [makeLineItem()]);
    order.place();

    expect(() => order.place()).toThrow(DomainError);
  });

  it('should cancel a placed order', () => {
    const order = Order.create('cust-1', makeAddress(), [makeLineItem()]);
    order.place();
    order.cancel('Changed my mind');

    expect(order.status).toBe(OrderStatus.CANCELLED);
    expect(order.domainEvents).toHaveLength(2); // placed + cancelled
  });

  it('should not allow modification after placing', () => {
    const order = Order.create('cust-1', makeAddress(), [makeLineItem()]);
    order.place();

    expect(() => order.addLineItem(makeLineItem())).toThrow(DomainError);
  });
});
```

### Unit Test — Value Object

```typescript
// modules/order/domain/value-objects/__tests__/money.vo.spec.ts

import { Money } from '../money.vo';
import { DomainError } from '../../../../../shared/exceptions/domain.exception';

describe('Money Value Object', () => {
  it('should create valid money', () => {
    const money = new Money(100, 'USD');
    expect(money.amount).toBe(100);
    expect(money.currency).toBe('USD');
  });

  it('should reject negative amount', () => {
    expect(() => new Money(-5, 'USD')).toThrow(DomainError);
  });

  it('should add same currency', () => {
    const a = new Money(10, 'USD');
    const b = new Money(20, 'USD');
    expect(a.add(b).amount).toBe(30);
  });

  it('should reject adding different currencies', () => {
    const usd = new Money(10, 'USD');
    const eur = new Money(10, 'EUR');
    expect(() => usd.add(eur)).toThrow(DomainError);
  });
});
```

### Unit Test — Application Handler

```typescript
// modules/order/application/commands/place-order/__tests__/place-order.handler.spec.ts

import { PlaceOrderHandler } from '../place-order.handler';
import { OrderRepository } from '../../../../domain/repositories/order.repository';
import { EventBus } from '../../../../../../shared/application/event-bus.interface';
import { PlaceOrderCommand } from '../place-order.command';

describe('PlaceOrderHandler', () => {
  let handler: PlaceOrderHandler;
  let mockRepo: jest.Mocked<OrderRepository>;
  let mockEventBus: jest.Mocked<EventBus>;

  beforeEach(() => {
    mockRepo = {
      findById: jest.fn(),
      findByCustomerId: jest.fn(),
      save: jest.fn().mockResolvedValue(undefined),
      delete: jest.fn(),
    } as any;

    mockEventBus = {
      publish: jest.fn().mockResolvedValue(undefined),
      publishAll: jest.fn().mockResolvedValue(undefined),
    } as any;

    handler = new PlaceOrderHandler(mockRepo, mockEventBus);
  });

  it('should place an order and dispatch events', async () => {
    const command = new PlaceOrderCommand(
      'cust-1',
      { street: '123 Main', city: 'Springfield', postalCode: '62701', country: 'US' },
      [{ productId: 'p1', productName: 'Widget', quantity: 2, unitPrice: 10, currency: 'USD' }],
    );

    const result = await handler.execute(command);

    expect(result.orderId).toBeDefined();
    expect(mockRepo.save).toHaveBeenCalledTimes(1);
    expect(mockEventBus.publishAll).toHaveBeenCalledTimes(1);
  });
});
```

### What to Test Where

| Layer | What to Test | Key Assertion |
|-------|-------------|---------------|
| **Domain (Aggregate)** | State transitions, invariants, event generation | expect(order.status).toBe(...) |
| **Domain (Value Object)** | Validation, equality, immutability | expect(() => new Money(-1, 'USD')).toThrow() |
| **Domain (Domain Service)** | Cross-aggregate business logic | expect(discount.amount).toBe(...) |
| **Application (Handler)** | Orchestration, repo called, events dispatched | expect(mockRepo.save).toHaveBeenCalled() |
| **Infrastructure (Mapper)** | Domain ↔ Persistence round-trip | expect(toDomain(toPersistence(order))).toEqual(order) |

---

## Quick Reference

| DDD Concept | Location | Purpose |
|-------------|----------|---------|
| **Aggregate Root** | `domain/aggregates/` | Transactional boundary, owns child entities |
| **Entity** | `domain/entities/` | Has identity, mutable within aggregate |
| **Value Object** | `domain/value-objects/` | Immutable, compared by value |
| **Domain Event** | `domain/events/` | Signals something happened |
| **Domain Service** | `domain/services/` | Stateless cross-entity business logic |
| **Repository (abstract)** | `domain/repositories/` | Persistence contract per aggregate |
| **Command Handler** | `application/commands/` | Orchestrates write operations |
| **Query Handler** | `application/queries/` | Orchestrates read operations |
| **Event Handler** | `application/event-handlers/` | Reacts to domain events |
| **Repository (concrete)** | `infrastructure/repositories/` | TypeORM/Prisma implementation |
| **Schema** | `infrastructure/schemas/` | ORM entity (persistence model) |
| **Mapper** | `infrastructure/mappers/` | Domain ↔ Persistence conversion |
| **Controller** | `presentation/controllers/` | HTTP → Command/Query |
| **Event Bus** | `shared/infrastructure/` | Dispatches domain events |

---

## DDD vs Clean Architecture

| Aspect | DDD | Clean Architecture |
|--------|-----|-------------------|
| **Primary focus** | Modeling the business domain | Layer separation & dependency direction |
| **Central concept** | Aggregates & bounded contexts | Use cases & entities |
| **Persistence boundary** | Aggregate root (whole loaded/saved) | Entity (individually loaded/saved) |
| **Communication** | Domain events between contexts | Direct injection between layers |
| **Business logic location** | Inside aggregates & domain services | Inside use cases & entities |
| **Complexity management** | Bounded contexts isolate subdomains | Layers isolate concerns |
| **Best combined with** | Clean Architecture or Hexagonal | DDD tactical patterns |

> **In practice**, DDD and Clean Architecture complement each other. DDD provides the modeling approach (how to structure domain logic), while Clean Architecture provides the layering approach (how to structure dependencies). Many production systems use both.

---

## Further Reading

- [Domain-Driven Design — Eric Evans (2003)](https://www.domainlanguage.com/ddd/)
- [Implementing Domain-Driven Design — Vaughn Vernon (2013)](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/)
- [NestJS Event Emitter](https://docs.nestjs.com/techniques/events)
- [Aggregate Design — Vaughn Vernon](https://www.dddcommunity.org/library/vernon_2011/)
