# DDD + Ports & Adapters (Hexagonal) for NestJS

> Production-grade guide combining DDD tactical patterns (Aggregates, Domain Events, Value Objects) with Hexagonal Architecture (explicit driving & driven ports). Maximum modularity and pluggability.

---

## Table of Contents

- [Why Combine DDD + Hexagonal](#why-combine-ddd--hexagonal)
- [Project Structure](#project-structure)
- [Dependency & Port Map](#dependency--port-map)
- [Domain Layer](#domain-layer)
  - [Aggregate Root](#aggregate-root)
  - [Entities](#entities)
  - [Value Objects](#value-objects)
  - [Domain Events](#domain-events)
  - [Domain Services](#domain-services)
- [Application Layer](#application-layer)
  - [Inbound Ports (Driving)](#inbound-ports-driving)
  - [Outbound Ports (Driven)](#outbound-ports-driven)
  - [Command Handlers (Use Cases)](#command-handlers-use-cases)
  - [Query Handlers](#query-handlers)
  - [Domain Event Handlers](#domain-event-handlers)
- [Infrastructure Layer](#infrastructure-layer)
  - [Driven Adapters](#driven-adapters)
  - [Event Bus Adapter](#event-bus-adapter)
  - [Schema & Mapper](#schema--mapper)
- [Presentation Layer](#presentation-layer)
  - [Driving Adapters (Controllers)](#driving-adapters-controllers)
- [Module Wiring](#module-wiring)
- [Error Handling](#error-handling)
- [Testing Strategy](#testing-strategy)
- [Quick Reference](#quick-reference)
- [Comparison: DDD vs DDD+Hexagonal vs Clean Arch](#comparison)

---

## Why Combine DDD + Hexagonal

| DDD Alone | DDD + Hexagonal |
|-----------|-----------------|
| Rich domain model with aggregates, events, VOs | Same rich domain model |
| Repository abstract class as DI token | Explicit `Symbol()`-based outbound ports for every external dependency |
| Controllers inject handlers directly | Controllers depend on **inbound port interfaces** (symbols) |
| Application handlers are concrete classes | Handlers implement formal inbound port contracts |
| Single event dispatching mechanism | Event bus defined as an outbound port, adapter swappable |

**When to use this pattern:**
- Large teams where driving/driven boundaries are formal API contracts
- Systems where every integration point (DB, email, queue, cache) must be independently swappable
- When you want both DDD's domain modeling power AND hexagonal's port symmetry

---

## Project Structure

```
src/
├── main.ts
├── app.module.ts
│
├── shared/
│   ├── domain/
│   │   ├── aggregate-root.base.ts
│   │   ├── entity.base.ts
│   │   ├── value-object.base.ts
│   │   └── domain-event.base.ts
│   ├── ports/
│   │   ├── logger.port.ts                   # LoggerPort + Symbol
│   │   ├── config.port.ts                   # ConfigPort + Symbol
│   │   └── event-bus.port.ts                # EventBusPort + Symbol
│   ├── exceptions/
│   │   └── domain.exception.ts
│   ├── filters/
│   │   └── global-exception.filter.ts
│   └── shared.module.ts                     # @Global()
│
├── infrastructure/                          # Global driven adapters
│   └── adapters/
│       ├── logger.winston.adapter.ts
│       ├── config.nestjs.adapter.ts
│       └── event-bus.nestjs.adapter.ts
│
└── modules/
    └── order/                               # Bounded Context
        ├── domain/
        │   ├── aggregates/
        │   │   └── order.aggregate.ts
        │   ├── entities/
        │   │   └── order-line-item.entity.ts
        │   ├── value-objects/
        │   │   ├── order-id.vo.ts
        │   │   ├── money.vo.ts
        │   │   └── address.vo.ts
        │   ├── events/
        │   │   ├── order-placed.event.ts
        │   │   └── order-cancelled.event.ts
        │   └── services/
        │       └── order-pricing.service.ts
        │
        ├── application/
        │   ├── ports/
        │   │   ├── inbound/                 # Driving ports
        │   │   │   ├── place-order.port.ts
        │   │   │   ├── cancel-order.port.ts
        │   │   │   └── get-order.port.ts
        │   │   └── outbound/                # Driven ports
        │   │       ├── order.repository.port.ts
        │   │       ├── payment.service.port.ts
        │   │       └── notification.service.port.ts
        │   ├── commands/
        │   │   ├── place-order.command.ts
        │   │   └── cancel-order.command.ts
        │   ├── queries/
        │   │   └── get-order.query.ts
        │   ├── handlers/
        │   │   ├── place-order.handler.ts
        │   │   ├── cancel-order.handler.ts
        │   │   └── get-order.handler.ts
        │   ├── event-handlers/
        │   │   └── on-order-placed.handler.ts
        │   └── dtos/
        │       └── order.dto.ts
        │
        ├── infrastructure/                  # Driven adapters
        │   ├── repositories/
        │   │   └── order.typeorm.adapter.ts
        │   ├── services/
        │   │   ├── stripe-payment.adapter.ts
        │   │   └── sendgrid-notification.adapter.ts
        │   ├── schemas/
        │   │   ├── order.schema.ts
        │   │   └── order-line-item.schema.ts
        │   └── mappers/
        │       └── order.mapper.ts
        │
        ├── presentation/                    # Driving adapters
        │   ├── controllers/
        │   │   └── order.controller.ts
        │   └── dtos/
        │       ├── place-order.request.dto.ts
        │       └── order.response.dto.ts
        │
        └── order.module.ts
```

---

## Dependency & Port Map

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  DRIVING SIDE                         DRIVEN SIDE                    │
│  (who calls us)                       (who we call)                  │
│                                                                      │
│  ┌──────────┐   ┌───────────────┐     ┌───────────────┐  ┌────────┐│
│  │Controller │──▶│ Inbound Port  │     │ Outbound Port │──▶│TypeORM ││
│  │(HTTP)     │   │ (Symbol)      │     │ (Symbol)      │  │Adapter ││
│  └──────────┘   └──────┬────────┘     └──────▲────────┘  └────────┘│
│                        │                      │                      │
│  ┌──────────┐   ┌──────▼────────┐     ┌──────┴────────┐  ┌────────┐│
│  │WebSocket │   │   Handler     │────▶│  Aggregate     │  │Stripe  ││
│  │Gateway   │   │ (implements   │     │  (Domain)      │  │Adapter ││
│  └──────────┘   │  inbound port)│     └───────────────┘  └────────┘│
│                  └──────┬────────┘                                   │
│  ┌──────────┐          │              ┌───────────────┐  ┌────────┐│
│  │CLI       │          └─────────────▶│ EventBusPort  │──▶│NestJS  ││
│  │Command   │                         │ (Symbol)      │  │Emitter ││
│  └──────────┘                         └───────────────┘  └────────┘│
│                                                                      │
│  Driving adapters                     Driven adapters                │
│  call inbound ports                   implement outbound ports       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Domain Layer

Pure business logic. Zero framework dependencies. Identical to the pure DDD version — the domain doesn't know about ports.

### Shared Base Classes

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

```typescript
// shared/domain/domain-event.base.ts

export abstract class DomainEvent {
  public readonly occurredOn: Date;
  public readonly eventId: string;

  constructor() {
    this.occurredOn = new Date();
    this.eventId = crypto.randomUUID();
  }

  abstract get eventName(): string;
}
```

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

### Aggregate Root

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
  PAID = 'PAID',
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

    return new Order(
      OrderId.generate(),
      customerId,
      shippingAddress,
      lineItems,
      OrderStatus.DRAFT,
      new Date(),
    );
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

  // ── Commands ─────────────────────────────────────────────

  place(): void {
    if (this._status !== OrderStatus.DRAFT) {
      throw new DomainError(`Cannot place order in status: ${this._status}`);
    }
    this._status = OrderStatus.PLACED;

    this.addDomainEvent(
      new OrderPlacedEvent(this.id.value, this.customerId, this.totalAmount),
    );
  }

  markAsPaid(): void {
    if (this._status !== OrderStatus.PLACED) {
      throw new DomainError(`Cannot mark as paid in status: ${this._status}`);
    }
    this._status = OrderStatus.PAID;
  }

  cancel(reason: string): void {
    const cancellable = [OrderStatus.DRAFT, OrderStatus.PLACED];
    if (!cancellable.includes(this._status)) {
      throw new DomainError(`Cannot cancel order in status: ${this._status}`);
    }
    this._status = OrderStatus.CANCELLED;

    this.addDomainEvent(new OrderCancelledEvent(this.id.value, reason));
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
    const idx = this._lineItems.findIndex((i) => i.id === itemId);
    if (idx === -1) throw new DomainError(`Line item ${itemId} not found`);
    this._lineItems.splice(idx, 1);
    if (!this._lineItems.length) {
      throw new DomainError('Order must have at least one line item');
    }
  }
}
```

### Entities

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
    if (quantity <= 0) throw new DomainError('Quantity must be positive');
  }

  get totalPrice(): Money {
    return this.unitPrice.multiply(this.quantity);
  }

  changeQuantity(newQuantity: number): void {
    if (newQuantity <= 0) throw new DomainError('Quantity must be positive');
    this.quantity = newQuantity;
  }
}
```

### Value Objects

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
  get amount(): number { return this.props.amount; }
  get currency(): string { return this.props.currency; }

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
      throw new DomainError(`Currency mismatch: ${this.currency} vs ${other.currency}`);
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

### Domain Events

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

### Domain Services

```typescript
// modules/order/domain/services/order-pricing.service.ts

import { Money } from '../value-objects/money.vo';
import { OrderLineItem } from '../entities/order-line-item.entity';

export class OrderPricingService {
  static calculateDiscount(lineItems: ReadonlyArray<OrderLineItem>): Money {
    const total = lineItems.reduce(
      (sum, item) => sum.add(item.totalPrice),
      new Money(0, lineItems[0].unitPrice.currency),
    );
    const qty = lineItems.reduce((sum, i) => sum + i.quantity, 0);

    if (qty >= 20) return total.multiply(0.15);
    if (qty >= 10) return total.multiply(0.10);
    return new Money(0, total.currency);
  }
}
```

---

## Application Layer

Defines **all port interfaces** (inbound + outbound) and implements them through handlers.

### Inbound Ports (Driving)

Formal contracts that the presentation layer programs against. Each port = one use case.

```typescript
// modules/order/application/ports/inbound/place-order.port.ts

import { PlaceOrderCommand } from '../../commands/place-order.command';

export interface PlaceOrderResult {
  orderId: string;
}

export interface PlaceOrderPort {
  execute(command: PlaceOrderCommand): Promise<PlaceOrderResult>;
}

export const PLACE_ORDER_PORT = Symbol('PlaceOrderPort');
```

```typescript
// modules/order/application/ports/inbound/cancel-order.port.ts

import { CancelOrderCommand } from '../../commands/cancel-order.command';

export interface CancelOrderPort {
  execute(command: CancelOrderCommand): Promise<void>;
}

export const CANCEL_ORDER_PORT = Symbol('CancelOrderPort');
```

```typescript
// modules/order/application/ports/inbound/get-order.port.ts

import { GetOrderQuery } from '../../queries/get-order.query';
import { OrderDto } from '../../dtos/order.dto';

export interface GetOrderPort {
  execute(query: GetOrderQuery): Promise<OrderDto>;
}

export const GET_ORDER_PORT = Symbol('GetOrderPort');
```

### Outbound Ports (Driven)

Contracts for every external dependency — DB, payment, notification, event bus.

```typescript
// modules/order/application/ports/outbound/order.repository.port.ts

import { Order } from '../../../domain/aggregates/order.aggregate';

export interface OrderRepositoryPort {
  findById(id: string): Promise<Order | null>;
  findByCustomerId(customerId: string): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: string): Promise<void>;
}

export const ORDER_REPOSITORY_PORT = Symbol('OrderRepositoryPort');
```

```typescript
// modules/order/application/ports/outbound/payment.service.port.ts

export interface PaymentResult {
  transactionId: string;
  status: 'success' | 'failed';
}

export interface PaymentServicePort {
  charge(customerId: string, amountCents: number, currency: string): Promise<PaymentResult>;
  refund(transactionId: string): Promise<void>;
}

export const PAYMENT_SERVICE_PORT = Symbol('PaymentServicePort');
```

```typescript
// modules/order/application/ports/outbound/notification.service.port.ts

export interface NotificationServicePort {
  sendOrderConfirmation(to: string, orderId: string): Promise<void>;
  sendOrderCancellation(to: string, orderId: string, reason: string): Promise<void>;
}

export const NOTIFICATION_SERVICE_PORT = Symbol('NotificationServicePort');
```

```typescript
// shared/ports/event-bus.port.ts

import { DomainEvent } from '../domain/domain-event.base';

export interface EventBusPort {
  publish(event: DomainEvent): Promise<void>;
  publishAll(events: DomainEvent[]): Promise<void>;
}

export const EVENT_BUS_PORT = Symbol('EventBusPort');
```

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

### Commands & Queries

```typescript
// modules/order/application/commands/place-order.command.ts

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
// modules/order/application/commands/cancel-order.command.ts

export class CancelOrderCommand {
  constructor(
    public readonly orderId: string,
    public readonly reason: string,
  ) {}
}
```

```typescript
// modules/order/application/queries/get-order.query.ts

export class GetOrderQuery {
  constructor(public readonly orderId: string) {}
}
```

### Command Handlers (Use Cases)

Each handler implements an inbound port and depends only on outbound ports.

```typescript
// modules/order/application/handlers/place-order.handler.ts

import { Injectable, Inject } from '@nestjs/common';
import { PlaceOrderPort, PlaceOrderResult } from '../ports/inbound/place-order.port';
import { PlaceOrderCommand } from '../commands/place-order.command';
import { ORDER_REPOSITORY_PORT, OrderRepositoryPort } from '../ports/outbound/order.repository.port';
import { PAYMENT_SERVICE_PORT, PaymentServicePort } from '../ports/outbound/payment.service.port';
import { EVENT_BUS_PORT, EventBusPort } from '../../../../shared/ports/event-bus.port';
import { LOGGER_PORT, LoggerPort } from '../../../../shared/ports/logger.port';
import { Order } from '../../domain/aggregates/order.aggregate';
import { OrderLineItem } from '../../domain/entities/order-line-item.entity';
import { Address } from '../../domain/value-objects/address.vo';
import { Money } from '../../domain/value-objects/money.vo';
import { DomainError } from '../../../../shared/exceptions/domain.exception';
import { randomUUID } from 'crypto';

@Injectable()
export class PlaceOrderHandler implements PlaceOrderPort {
  constructor(
    @Inject(ORDER_REPOSITORY_PORT)
    private readonly orderRepo: OrderRepositoryPort,

    @Inject(PAYMENT_SERVICE_PORT)
    private readonly paymentService: PaymentServicePort,

    @Inject(EVENT_BUS_PORT)
    private readonly eventBus: EventBusPort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,
  ) {}

  async execute(command: PlaceOrderCommand): Promise<PlaceOrderResult> {
    this.logger.log('Placing order', PlaceOrderHandler.name);

    // 1. Build domain objects
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

    // 2. Create & place (domain logic)
    const order = Order.create(command.customerId, address, lineItems);
    order.place();

    // 3. Charge payment (outbound port)
    const payment = await this.paymentService.charge(
      command.customerId,
      Math.round(order.totalAmount.amount * 100),
      order.totalAmount.currency,
    );

    if (payment.status === 'failed') {
      throw new DomainError('Payment failed');
    }

    order.markAsPaid();

    // 4. Persist aggregate
    await this.orderRepo.save(order);

    // 5. Dispatch domain events
    await this.eventBus.publishAll(order.domainEvents);
    order.clearDomainEvents();

    this.logger.log(`Order placed: ${order.id}`, PlaceOrderHandler.name);
    return { orderId: order.id.value };
  }
}
```

```typescript
// modules/order/application/handlers/cancel-order.handler.ts

import { Injectable, Inject } from '@nestjs/common';
import { CancelOrderPort } from '../ports/inbound/cancel-order.port';
import { CancelOrderCommand } from '../commands/cancel-order.command';
import { ORDER_REPOSITORY_PORT, OrderRepositoryPort } from '../ports/outbound/order.repository.port';
import { EVENT_BUS_PORT, EventBusPort } from '../../../../shared/ports/event-bus.port';
import { LOGGER_PORT, LoggerPort } from '../../../../shared/ports/logger.port';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

@Injectable()
export class CancelOrderHandler implements CancelOrderPort {
  constructor(
    @Inject(ORDER_REPOSITORY_PORT)
    private readonly orderRepo: OrderRepositoryPort,

    @Inject(EVENT_BUS_PORT)
    private readonly eventBus: EventBusPort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,
  ) {}

  async execute(command: CancelOrderCommand): Promise<void> {
    this.logger.log(`Cancelling order ${command.orderId}`, CancelOrderHandler.name);

    const order = await this.orderRepo.findById(command.orderId);
    if (!order) throw new DomainError(`Order ${command.orderId} not found`);

    // Domain logic decides if cancellation is valid
    order.cancel(command.reason);

    await this.orderRepo.save(order);

    await this.eventBus.publishAll(order.domainEvents);
    order.clearDomainEvents();
  }
}
```

```typescript
// modules/order/application/handlers/get-order.handler.ts

import { Injectable, Inject } from '@nestjs/common';
import { GetOrderPort } from '../ports/inbound/get-order.port';
import { GetOrderQuery } from '../queries/get-order.query';
import { ORDER_REPOSITORY_PORT, OrderRepositoryPort } from '../ports/outbound/order.repository.port';
import { OrderDto } from '../dtos/order.dto';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

@Injectable()
export class GetOrderHandler implements GetOrderPort {
  constructor(
    @Inject(ORDER_REPOSITORY_PORT)
    private readonly orderRepo: OrderRepositoryPort,
  ) {}

  async execute(query: GetOrderQuery): Promise<OrderDto> {
    const order = await this.orderRepo.findById(query.orderId);
    if (!order) throw new DomainError(`Order ${query.orderId} not found`);

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

### Domain Event Handlers

```typescript
// modules/order/application/event-handlers/on-order-placed.handler.ts

import { Injectable, Inject } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { OrderPlacedEvent } from '../../domain/events/order-placed.event';
import {
  NOTIFICATION_SERVICE_PORT,
  NotificationServicePort,
} from '../ports/outbound/notification.service.port';
import { LOGGER_PORT, LoggerPort } from '../../../../shared/ports/logger.port';

@Injectable()
export class OnOrderPlacedHandler {
  constructor(
    @Inject(NOTIFICATION_SERVICE_PORT)
    private readonly notificationService: NotificationServicePort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,
  ) {}

  @OnEvent('order.placed')
  async handle(event: OrderPlacedEvent): Promise<void> {
    this.logger.log(
      `Handling order.placed for ${event.orderId}`,
      OnOrderPlacedHandler.name,
    );

    await this.notificationService.sendOrderConfirmation(
      event.customerId,
      event.orderId,
    );
  }
}
```

> **Key difference from pure DDD:** The event handler depends on `NOTIFICATION_SERVICE_PORT` (symbol), not a concrete class. The notification provider is fully swappable.

---

## Infrastructure Layer

Driven adapters that implement outbound ports.

### Repository Adapter

```typescript
// modules/order/infrastructure/repositories/order.typeorm.adapter.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { OrderRepositoryPort } from '../../application/ports/outbound/order.repository.port';
import { Order } from '../../domain/aggregates/order.aggregate';
import { OrderSchema } from '../schemas/order.schema';
import { OrderMapper } from '../mappers/order.mapper';

@Injectable()
export class OrderTypeOrmAdapter implements OrderRepositoryPort {
  constructor(
    @InjectRepository(OrderSchema)
    private readonly repo: Repository<OrderSchema>,
  ) {}

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
    await this.repo.save(OrderMapper.toPersistence(order));
  }

  async delete(id: string): Promise<void> {
    await this.repo.delete({ id });
  }
}
```

### Payment Adapter

```typescript
// modules/order/infrastructure/services/stripe-payment.adapter.ts

import { Injectable } from '@nestjs/common';
import { PaymentServicePort, PaymentResult } from '../../application/ports/outbound/payment.service.port';

@Injectable()
export class StripePaymentAdapter implements PaymentServicePort {
  async charge(
    customerId: string,
    amountCents: number,
    currency: string,
  ): Promise<PaymentResult> {
    // Stripe API call
    console.log(`[Stripe] Charging ${amountCents} ${currency} for ${customerId}`);
    return { transactionId: `txn_${Date.now()}`, status: 'success' };
  }

  async refund(transactionId: string): Promise<void> {
    console.log(`[Stripe] Refunding ${transactionId}`);
  }
}
```

### Notification Adapter

```typescript
// modules/order/infrastructure/services/sendgrid-notification.adapter.ts

import { Injectable } from '@nestjs/common';
import { NotificationServicePort } from '../../application/ports/outbound/notification.service.port';

@Injectable()
export class SendGridNotificationAdapter implements NotificationServicePort {
  async sendOrderConfirmation(to: string, orderId: string): Promise<void> {
    console.log(`[SendGrid] Order confirmation to ${to} for order ${orderId}`);
  }

  async sendOrderCancellation(to: string, orderId: string, reason: string): Promise<void> {
    console.log(`[SendGrid] Order cancellation to ${to} for order ${orderId}: ${reason}`);
  }
}
```

### Event Bus Adapter

```typescript
// infrastructure/adapters/event-bus.nestjs.adapter.ts

import { Injectable } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { EventBusPort } from '../../shared/ports/event-bus.port';
import { DomainEvent } from '../../shared/domain/domain-event.base';

@Injectable()
export class NestJsEventBusAdapter implements EventBusPort {
  constructor(private readonly emitter: EventEmitter2) {}

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

### Logger Adapter

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

### Schema & Mapper

```typescript
// modules/order/infrastructure/schemas/order.schema.ts

import { Entity, PrimaryColumn, Column, CreateDateColumn, OneToMany } from 'typeorm';
import { OrderLineItemSchema } from './order-line-item.schema';

@Entity('orders')
export class OrderSchema {
  @PrimaryColumn('uuid') id: string;
  @Column() customerId: string;
  @Column() status: string;
  @Column() shippingStreet: string;
  @Column() shippingCity: string;
  @Column() shippingPostalCode: string;
  @Column() shippingCountry: string;

  @OneToMany(() => OrderLineItemSchema, (item) => item.order, { cascade: true })
  lineItems: OrderLineItemSchema[];

  @CreateDateColumn() createdAt: Date;
}
```

```typescript
// modules/order/infrastructure/schemas/order-line-item.schema.ts

import { Entity, PrimaryColumn, Column, ManyToOne } from 'typeorm';
import { OrderSchema } from './order.schema';

@Entity('order_line_items')
export class OrderLineItemSchema {
  @PrimaryColumn('uuid') id: string;
  @Column() productId: string;
  @Column() productName: string;
  @Column() quantity: number;
  @Column('decimal') unitPrice: number;
  @Column() currency: string;

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

---

## Presentation Layer

Driving adapters that call inbound ports via symbols.

### Controller

```typescript
// modules/order/presentation/controllers/order.controller.ts

import {
  Controller, Post, Get, Patch, Param, Body,
  Inject, HttpCode, HttpStatus,
} from '@nestjs/common';
import { PLACE_ORDER_PORT, PlaceOrderPort } from '../../application/ports/inbound/place-order.port';
import { CANCEL_ORDER_PORT, CancelOrderPort } from '../../application/ports/inbound/cancel-order.port';
import { GET_ORDER_PORT, GetOrderPort } from '../../application/ports/inbound/get-order.port';
import { PlaceOrderCommand } from '../../application/commands/place-order.command';
import { CancelOrderCommand } from '../../application/commands/cancel-order.command';
import { GetOrderQuery } from '../../application/queries/get-order.query';
import { PlaceOrderRequestDto } from '../dtos/place-order.request.dto';
import { OrderResponseDto } from '../dtos/order.response.dto';

@Controller('orders')
export class OrderController {
  constructor(
    @Inject(PLACE_ORDER_PORT)
    private readonly placeOrder: PlaceOrderPort,

    @Inject(CANCEL_ORDER_PORT)
    private readonly cancelOrder: CancelOrderPort,

    @Inject(GET_ORDER_PORT)
    private readonly getOrder: GetOrderPort,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async place(@Body() dto: PlaceOrderRequestDto): Promise<{ orderId: string }> {
    return this.placeOrder.execute(
      new PlaceOrderCommand(dto.customerId, dto.shippingAddress, dto.items),
    );
  }

  @Patch(':id/cancel')
  async cancel(
    @Param('id') id: string,
    @Body('reason') reason: string,
  ): Promise<void> {
    return this.cancelOrder.execute(new CancelOrderCommand(id, reason));
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<OrderResponseDto> {
    const dto = await this.getOrder.execute(new GetOrderQuery(id));
    return { ...dto, createdAt: dto.createdAt.toISOString() };
  }
}
```

> **Controller depends on symbols, not classes.** It has no idea whether `PLACE_ORDER_PORT` is backed by `PlaceOrderHandler` or a mock. This is the hexagonal driving side.

### Request / Response DTOs

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
  @IsString() customerId: string;

  @ValidateNested() @Type(() => AddressDto)
  shippingAddress: AddressDto;

  @IsArray() @ValidateNested({ each: true }) @Type(() => LineItemDto)
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

---

## Module Wiring

All port-to-adapter bindings happen here — the single place to swap any implementation.

```typescript
// modules/order/order.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

// Inbound Ports
import { PLACE_ORDER_PORT } from './application/ports/inbound/place-order.port';
import { CANCEL_ORDER_PORT } from './application/ports/inbound/cancel-order.port';
import { GET_ORDER_PORT } from './application/ports/inbound/get-order.port';

// Outbound Ports
import { ORDER_REPOSITORY_PORT } from './application/ports/outbound/order.repository.port';
import { PAYMENT_SERVICE_PORT } from './application/ports/outbound/payment.service.port';
import { NOTIFICATION_SERVICE_PORT } from './application/ports/outbound/notification.service.port';

// Handlers (implement inbound ports)
import { PlaceOrderHandler } from './application/handlers/place-order.handler';
import { CancelOrderHandler } from './application/handlers/cancel-order.handler';
import { GetOrderHandler } from './application/handlers/get-order.handler';
import { OnOrderPlacedHandler } from './application/event-handlers/on-order-placed.handler';

// Adapters (implement outbound ports)
import { OrderTypeOrmAdapter } from './infrastructure/repositories/order.typeorm.adapter';
import { StripePaymentAdapter } from './infrastructure/services/stripe-payment.adapter';
import { SendGridNotificationAdapter } from './infrastructure/services/sendgrid-notification.adapter';

// Schemas
import { OrderSchema } from './infrastructure/schemas/order.schema';
import { OrderLineItemSchema } from './infrastructure/schemas/order-line-item.schema';

// Presentation
import { OrderController } from './presentation/controllers/order.controller';

@Module({
  imports: [TypeOrmModule.forFeature([OrderSchema, OrderLineItemSchema])],
  controllers: [OrderController],
  providers: [
    // ── Inbound Ports → Handlers ──────────────────────────
    { provide: PLACE_ORDER_PORT,  useClass: PlaceOrderHandler },
    { provide: CANCEL_ORDER_PORT, useClass: CancelOrderHandler },
    { provide: GET_ORDER_PORT,    useClass: GetOrderHandler },

    // ── Outbound Ports → Adapters ─────────────────────────
    { provide: ORDER_REPOSITORY_PORT,   useClass: OrderTypeOrmAdapter },
    { provide: PAYMENT_SERVICE_PORT,    useClass: StripePaymentAdapter },
    { provide: NOTIFICATION_SERVICE_PORT, useClass: SendGridNotificationAdapter },

    // ── Event Handlers ────────────────────────────────────
    OnOrderPlacedHandler,
  ],
})
export class OrderModule {}
```

```typescript
// shared/shared.module.ts

import { Global, Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';
import { LOGGER_PORT } from './ports/logger.port';
import { EVENT_BUS_PORT } from './ports/event-bus.port';
import { WinstonLoggerAdapter } from '../infrastructure/adapters/logger.winston.adapter';
import { NestJsEventBusAdapter } from '../infrastructure/adapters/event-bus.nestjs.adapter';

@Global()
@Module({
  imports: [EventEmitterModule.forRoot()],
  providers: [
    { provide: LOGGER_PORT,    useClass: WinstonLoggerAdapter },
    { provide: EVENT_BUS_PORT, useClass: NestJsEventBusAdapter },
  ],
  exports: [LOGGER_PORT, EVENT_BUS_PORT],
})
export class SharedModule {}
```

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { SharedModule } from './shared/shared.module';
import { OrderModule } from './modules/order/order.module';

@Module({
  imports: [
    TypeOrmModule.forRoot({ /* ... */ }),
    SharedModule,
    OrderModule,
  ],
})
export class AppModule {}
```

### Swapping Is Trivial

```typescript
// Change payment provider — only the module changes:
{ provide: PAYMENT_SERVICE_PORT, useClass: PayPalPaymentAdapter },

// Change event bus to RabbitMQ:
{ provide: EVENT_BUS_PORT, useClass: RabbitMqEventBusAdapter },

// Change notification to Twilio SMS:
{ provide: NOTIFICATION_SERVICE_PORT, useClass: TwilioNotificationAdapter },
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
  Catch, ExceptionFilter, ArgumentsHost,
  HttpException, Inject,
} from '@nestjs/common';
import { Response } from 'express';
import { DomainError } from '../exceptions/domain.exception';
import { LOGGER_PORT, LoggerPort } from '../ports/logger.port';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(
    @Inject(LOGGER_PORT) private readonly logger: LoggerPort,
  ) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse<Response>();

    if (exception instanceof HttpException) {
      response.status(exception.getStatus()).json(exception.getResponse());
      return;
    }

    if (exception instanceof DomainError) {
      const status = exception.message.toLowerCase().includes('not found') ? 404 : 400;
      response.status(status).json({
        statusCode: status,
        error: status === 404 ? 'Not Found' : 'Business Rule Violation',
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

### Unit Test — Aggregate (Same as Pure DDD)

```typescript
// modules/order/domain/aggregates/__tests__/order.aggregate.spec.ts

import { Order, OrderStatus } from '../order.aggregate';
import { OrderLineItem } from '../../entities/order-line-item.entity';
import { Address } from '../../value-objects/address.vo';
import { Money } from '../../value-objects/money.vo';
import { DomainError } from '../../../../../shared/exceptions/domain.exception';

const makeItem = () =>
  new OrderLineItem('item-1', 'prod-1', 'Widget', 2, new Money(10, 'USD'));
const makeAddress = () =>
  new Address('123 Main', 'Springfield', '62701', 'US');

describe('Order Aggregate', () => {
  it('should create, place, and emit event', () => {
    const order = Order.create('cust-1', makeAddress(), [makeItem()]);
    order.place();

    expect(order.status).toBe(OrderStatus.PLACED);
    expect(order.domainEvents).toHaveLength(1);
    expect(order.domainEvents[0].eventName).toBe('order.placed');
  });

  it('should reject double placement', () => {
    const order = Order.create('cust-1', makeAddress(), [makeItem()]);
    order.place();
    expect(() => order.place()).toThrow(DomainError);
  });
});
```

### Unit Test — Handler (Port-Based Mocking)

```typescript
// modules/order/application/handlers/__tests__/place-order.handler.spec.ts

import { PlaceOrderHandler } from '../place-order.handler';
import { OrderRepositoryPort } from '../../ports/outbound/order.repository.port';
import { PaymentServicePort } from '../../ports/outbound/payment.service.port';
import { EventBusPort } from '../../../../../shared/ports/event-bus.port';
import { LoggerPort } from '../../../../../shared/ports/logger.port';
import { PlaceOrderCommand } from '../../commands/place-order.command';

describe('PlaceOrderHandler', () => {
  let handler: PlaceOrderHandler;
  let mockRepo: jest.Mocked<OrderRepositoryPort>;
  let mockPayment: jest.Mocked<PaymentServicePort>;
  let mockEventBus: jest.Mocked<EventBusPort>;
  let mockLogger: jest.Mocked<LoggerPort>;

  beforeEach(() => {
    mockRepo = {
      findById: jest.fn(),
      findByCustomerId: jest.fn(),
      save: jest.fn().mockResolvedValue(undefined),
      delete: jest.fn(),
    };

    mockPayment = {
      charge: jest.fn().mockResolvedValue({ transactionId: 'txn_1', status: 'success' }),
      refund: jest.fn(),
    };

    mockEventBus = {
      publish: jest.fn().mockResolvedValue(undefined),
      publishAll: jest.fn().mockResolvedValue(undefined),
    };

    mockLogger = {
      log: jest.fn(),
      error: jest.fn(),
      warn: jest.fn(),
      debug: jest.fn(),
    };

    handler = new PlaceOrderHandler(mockRepo, mockPayment, mockEventBus, mockLogger);
  });

  it('should place order, charge payment, persist, and dispatch events', async () => {
    const command = new PlaceOrderCommand(
      'cust-1',
      { street: '123 Main', city: 'Springfield', postalCode: '62701', country: 'US' },
      [{ productId: 'p1', productName: 'Widget', quantity: 2, unitPrice: 10, currency: 'USD' }],
    );

    const result = await handler.execute(command);

    expect(result.orderId).toBeDefined();
    expect(mockPayment.charge).toHaveBeenCalledWith('cust-1', 2000, 'USD');
    expect(mockRepo.save).toHaveBeenCalledTimes(1);
    expect(mockEventBus.publishAll).toHaveBeenCalledTimes(1);
  });

  it('should fail if payment is declined', async () => {
    mockPayment.charge.mockResolvedValueOnce({ transactionId: '', status: 'failed' });

    const command = new PlaceOrderCommand(
      'cust-1',
      { street: '123', city: 'City', postalCode: '00000', country: 'US' },
      [{ productId: 'p1', productName: 'X', quantity: 1, unitPrice: 50, currency: 'USD' }],
    );

    await expect(handler.execute(command)).rejects.toThrow('Payment failed');
    expect(mockRepo.save).not.toHaveBeenCalled();
  });
});
```

### What to Test Where

| Layer | What | Mock |
|-------|------|------|
| **Domain (Aggregate)** | State transitions, invariants, event generation | Nothing |
| **Domain (Value Object)** | Validation, equality, immutability | Nothing |
| **Application (Handler)** | Orchestration flow, port interactions | All outbound port interfaces |
| **Infrastructure (Adapter)** | DB queries, API calls, mapper round-trips | Database (test container) |
| **Presentation (Controller)** | HTTP validation, status codes | Inbound port interfaces |

---

## Quick Reference

| Component | Location | DDD Concept | Hexagonal Role |
|-----------|----------|-------------|----------------|
| `Order` | `domain/aggregates/` | Aggregate Root | — |
| `OrderLineItem` | `domain/entities/` | Entity | — |
| `Money`, `Address`, `OrderId` | `domain/value-objects/` | Value Object | — |
| `OrderPlacedEvent` | `domain/events/` | Domain Event | — |
| `OrderPricingService` | `domain/services/` | Domain Service | — |
| `PlaceOrderPort` | `application/ports/inbound/` | — | Driving Port |
| `OrderRepositoryPort` | `application/ports/outbound/` | Repository Contract | Driven Port |
| `PaymentServicePort` | `application/ports/outbound/` | — | Driven Port |
| `NotificationServicePort` | `application/ports/outbound/` | — | Driven Port |
| `EventBusPort` | `shared/ports/` | — | Driven Port |
| `PlaceOrderHandler` | `application/handlers/` | Application Service | Inbound Port Impl |
| `OrderTypeOrmAdapter` | `infrastructure/repositories/` | Repository Impl | Driven Adapter |
| `StripePaymentAdapter` | `infrastructure/services/` | — | Driven Adapter |
| `OrderController` | `presentation/controllers/` | — | Driving Adapter |

---

## Comparison

| Aspect | Pure DDD | DDD + Hexagonal | Clean Architecture |
|--------|----------|-----------------|-------------------|
| **Domain model** | Aggregates, VOs, Events | Aggregates, VOs, Events | Entities, VOs |
| **Repository contract** | Abstract class in domain | Symbol-based port in application | Interface/abstract in domain or application |
| **Controller → Use Case** | Direct class injection | Symbol-based inbound port | Direct or symbol-based |
| **External services** | Abstract class or direct | Symbol-based outbound port | Symbol-based port |
| **Event dispatching** | Abstract EventBus class | `EVENT_BUS_PORT` symbol | Not prescribed |
| **Boilerplate** | Medium | High | Low to Medium |
| **Pluggability** | Good | Maximum | Good |
| **Best for** | DDD-focused teams | Large systems with many integrations | General purpose |

---

## Further Reading

- [Domain-Driven Design — Eric Evans](https://www.domainlanguage.com/ddd/)
- [Hexagonal Architecture — Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- [Implementing DDD — Vaughn Vernon](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/)
- [NestJS Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
- [NestJS Event Emitter](https://docs.nestjs.com/techniques/events)
