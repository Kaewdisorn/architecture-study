# Clean Architecture + DDD + Ports & Adapters for NestJS

> The full-stack pattern: Uncle Bob's layered dependency rule, DDD tactical modeling (Aggregates, Domain Events, Value Objects), and Hexagonal port symmetry — all combined in one production-grade NestJS architecture.

---

## Table of Contents

- [Why Combine All Three](#why-combine-all-three)
- [How Each Pattern Contributes](#how-each-pattern-contributes)
- [Project Structure](#project-structure)
- [Dependency & Layer Map](#dependency--layer-map)
- [Layer 1 — Domain (DDD Tactical)](#layer-1--domain-ddd-tactical)
  - [Shared Building Blocks](#shared-building-blocks)
  - [Aggregate Root](#aggregate-root)
  - [Child Entities](#child-entities)
  - [Value Objects](#value-objects)
  - [Domain Events](#domain-events)
  - [Domain Services](#domain-services)
- [Layer 2 — Application (Use Cases + Ports)](#layer-2--application-use-cases--ports)
  - [Inbound Ports (Driving)](#inbound-ports-driving)
  - [Outbound Ports (Driven)](#outbound-ports-driven)
  - [Use Cases (Command Handlers)](#use-cases-command-handlers)
  - [Query Handlers](#query-handlers)
  - [Domain Event Handlers](#domain-event-handlers)
- [Layer 3 — Infrastructure (Driven Adapters)](#layer-3--infrastructure-driven-adapters)
  - [Repository Adapter](#repository-adapter)
  - [Payment Adapter](#payment-adapter)
  - [Event Bus Adapter](#event-bus-adapter)
  - [Schema & Mapper](#schema--mapper)
- [Layer 4 — Presentation (Driving Adapters)](#layer-4--presentation-driving-adapters)
- [Cross-Cutting Concerns](#cross-cutting-concerns)
- [Module Wiring](#module-wiring)
- [Error Handling](#error-handling)
- [Testing Strategy](#testing-strategy)
- [Quick Reference](#quick-reference)
- [Pattern Comparison Matrix](#pattern-comparison-matrix)

---

## Why Combine All Three

Each pattern solves a **different problem**:

| Pattern | Solves | Provides |
|---------|--------|----------|
| **Clean Architecture** | "How do I structure layers and dependencies?" | Strict inward dependency rule, layer isolation |
| **DDD** | "How do I model complex business logic?" | Aggregates, value objects, domain events, bounded contexts |
| **Ports & Adapters** | "How do I decouple from external systems?" | Formal driving/driven port contracts, maximum pluggability |

**Combined**, you get:
- Clean Architecture's **concentric layers** defining dependency direction
- DDD's **rich domain model** as the innermost layer
- Hexagonal's **ports** at every layer boundary, making each integration point independently swappable

---

## How Each Pattern Contributes

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  CLEAN ARCHITECTURE layers          DDD concepts         HEXAGONAL       │
│  ─────────────────────────          ────────────         ─────────       │
│                                                                          │
│  Layer 4: Presentation              —                    Driving Adapters│
│     Controllers, Gateways                                (call inbound   │
│     (depends on Layer 2 ports)                            ports)         │
│                                                                          │
│  Layer 3: Infrastructure            —                    Driven Adapters │
│     TypeORM, Stripe, SendGrid                            (implement      │
│     (implements Layer 2 ports)                            outbound ports)│
│                                                                          │
│  Layer 2: Application               Application          Port Definitions│
│     Use Cases, orchestration         Services             Inbound +      │
│     (depends on Layer 1 only)                             Outbound ports │
│                                                                          │
│  Layer 1: Domain                    Aggregates,          —               │
│     Pure business logic              Entities,            (no ports here │
│     (depends on nothing)             Value Objects,        — pure logic) │
│                                      Domain Events,                      │
│                                      Domain Services                     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
src/
├── main.ts
├── app.module.ts
│
├── shared/
│   ├── domain/                              # DDD building blocks
│   │   ├── aggregate-root.base.ts
│   │   ├── entity.base.ts
│   │   ├── value-object.base.ts
│   │   └── domain-event.base.ts
│   ├── ports/                               # Cross-cutting ports (Hexagonal)
│   │   ├── logger.port.ts
│   │   ├── config.port.ts
│   │   └── event-bus.port.ts
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
    └── order/                               # Bounded Context (DDD)
        │
        ├── domain/                          # Layer 1 — DDD Tactical
        │   ├── aggregates/
        │   │   └── order.aggregate.ts       # Aggregate Root
        │   ├── entities/
        │   │   └── order-line-item.entity.ts
        │   ├── value-objects/
        │   │   ├── order-id.vo.ts
        │   │   ├── money.vo.ts
        │   │   └── shipping-address.vo.ts
        │   ├── events/
        │   │   ├── order-created.event.ts
        │   │   ├── order-confirmed.event.ts
        │   │   └── order-cancelled.event.ts
        │   └── services/
        │       └── order-total.service.ts   # Domain Service
        │
        ├── application/                     # Layer 2 — Clean Arch Use Cases + Hexagonal Ports
        │   ├── ports/
        │   │   ├── inbound/                 # Driving ports (Hexagonal)
        │   │   │   ├── create-order.port.ts
        │   │   │   ├── confirm-order.port.ts
        │   │   │   ├── cancel-order.port.ts
        │   │   │   └── get-order.port.ts
        │   │   └── outbound/                # Driven ports (Hexagonal)
        │   │       ├── order.repository.port.ts
        │   │       ├── payment-gateway.port.ts
        │   │       ├── inventory.port.ts
        │   │       └── notification.port.ts
        │   ├── use-cases/                   # Clean Arch Use Cases = DDD App Services
        │   │   ├── create-order.use-case.ts
        │   │   ├── confirm-order.use-case.ts
        │   │   └── cancel-order.use-case.ts
        │   ├── queries/
        │   │   └── get-order.query-handler.ts
        │   ├── event-handlers/
        │   │   ├── on-order-created.handler.ts
        │   │   └── on-order-confirmed.handler.ts
        │   ├── commands/
        │   │   ├── create-order.command.ts
        │   │   ├── confirm-order.command.ts
        │   │   └── cancel-order.command.ts
        │   ├── queries/
        │   │   └── get-order.query.ts
        │   └── dtos/
        │       └── order.dto.ts
        │
        ├── infrastructure/                  # Layer 3 — Driven Adapters (Hexagonal)
        │   ├── repositories/
        │   │   └── order.typeorm.adapter.ts
        │   ├── services/
        │   │   ├── stripe-payment.adapter.ts
        │   │   ├── warehouse-inventory.adapter.ts
        │   │   └── sendgrid-notification.adapter.ts
        │   ├── schemas/
        │   │   ├── order.schema.ts
        │   │   └── order-line-item.schema.ts
        │   └── mappers/
        │       └── order.mapper.ts
        │
        ├── presentation/                    # Layer 4 — Driving Adapters (Hexagonal)
        │   ├── controllers/
        │   │   └── order.controller.ts
        │   └── dtos/
        │       ├── create-order.request.dto.ts
        │       └── order.response.dto.ts
        │
        └── order.module.ts                  # Binds all ports → adapters
```

---

## Dependency & Layer Map

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│  LAYER 4: Presentation           LAYER 2: Application                      │
│  (Driving Adapters)              (Use Cases + Ports)                       │
│                                                                            │
│  ┌──────────┐  Inbound   ┌──────────────────┐  Outbound   ┌────────────┐ │
│  │Controller │──Symbol──▶│ Inbound Port      │             │ Outbound   │ │
│  │(HTTP)     │           │   ▼               │──Symbol──▶ │ Port       │ │
│  └──────────┘           │ Use Case          │             └──────▲─────┘ │
│                          │   │               │                    │       │
│  ┌──────────┐           │   ▼               │  LAYER 3: Infrastructure  │
│  │WebSocket │           │ Aggregate         │  (Driven Adapters)        │
│  │Gateway   │──Symbol──▶│ (LAYER 1: Domain) │             ┌────────────┐ │
│  └──────────┘           └──────────────────┘             │ TypeORM    │ │
│                                                            │ Stripe     │ │
│  ┌──────────┐                                             │ SendGrid   │ │
│  │CLI / Job │──Symbol──▶  ...same ports...                │ Warehouse  │ │
│  └──────────┘                                             └────────────┘ │
│                                                                            │
│  Clean Arch: Layers 4,3 depend on 2, Layer 2 depends on 1               │
│  DDD: Layer 1 contains Aggregates, VOs, Events, Domain Services          │
│  Hexagonal: Every arrow crosses a Port boundary (Symbol-based)           │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Layer 1 — Domain (DDD Tactical)

The innermost layer of Clean Architecture, modeled using DDD tactical patterns. **Zero** framework imports.

### Shared Building Blocks

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

import { randomUUID } from 'crypto';

export abstract class DomainEvent {
  public readonly eventId: string;
  public readonly occurredOn: Date;

  constructor() {
    this.eventId = randomUUID();
    this.occurredOn = new Date();
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

The transactional boundary. All state changes go through it. It raises domain events.

```typescript
// modules/order/domain/aggregates/order.aggregate.ts

import { AggregateRoot } from '../../../../shared/domain/aggregate-root.base';
import { OrderId } from '../value-objects/order-id.vo';
import { Money } from '../value-objects/money.vo';
import { ShippingAddress } from '../value-objects/shipping-address.vo';
import { OrderLineItem } from '../entities/order-line-item.entity';
import { OrderCreatedEvent } from '../events/order-created.event';
import { OrderConfirmedEvent } from '../events/order-confirmed.event';
import { OrderCancelledEvent } from '../events/order-cancelled.event';
import { OrderTotalService } from '../services/order-total.service';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

export enum OrderStatus {
  DRAFT = 'DRAFT',
  CREATED = 'CREATED',
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
    public readonly shippingAddress: ShippingAddress,
    lineItems: OrderLineItem[],
    status: OrderStatus,
    public readonly createdAt: Date,
    public updatedAt: Date,
  ) {
    super(id);
    this._lineItems = lineItems;
    this._status = status;
  }

  // ── Factory ────────────────────────────────────────────────

  static create(
    customerId: string,
    shippingAddress: ShippingAddress,
    lineItems: OrderLineItem[],
  ): Order {
    if (!customerId) {
      throw new DomainError('Customer ID is required');
    }
    if (!lineItems.length) {
      throw new DomainError('Order must have at least one line item');
    }

    const now = new Date();
    const order = new Order(
      OrderId.generate(),
      customerId,
      shippingAddress,
      lineItems,
      OrderStatus.CREATED,
      now,
      now,
    );

    order.addDomainEvent(
      new OrderCreatedEvent(
        order.id.value,
        order.customerId,
        order.totalAmount,
        order.lineItems.length,
      ),
    );

    return order;
  }

  // ── Queries ────────────────────────────────────────────────

  get status(): OrderStatus {
    return this._status;
  }

  get lineItems(): ReadonlyArray<OrderLineItem> {
    return [...this._lineItems];
  }

  get totalAmount(): Money {
    return OrderTotalService.calculate(this._lineItems);
  }

  get discountedTotal(): Money {
    return OrderTotalService.calculateWithDiscount(this._lineItems);
  }

  get itemCount(): number {
    return this._lineItems.reduce((sum, item) => sum + item.quantity, 0);
  }

  // ── Commands ───────────────────────────────────────────────

  confirm(): void {
    if (this._status !== OrderStatus.CREATED) {
      throw new DomainError(
        `Cannot confirm order in status "${this._status}". Only CREATED orders can be confirmed.`,
      );
    }
    this._status = OrderStatus.CONFIRMED;
    this.updatedAt = new Date();

    this.addDomainEvent(
      new OrderConfirmedEvent(this.id.value, this.totalAmount),
    );
  }

  cancel(reason: string): void {
    if (!reason || reason.trim().length < 3) {
      throw new DomainError('Cancellation reason must be at least 3 characters');
    }

    const cancellable = [OrderStatus.CREATED, OrderStatus.CONFIRMED];
    if (!cancellable.includes(this._status)) {
      throw new DomainError(
        `Cannot cancel order in status "${this._status}". Only CREATED or CONFIRMED orders can be cancelled.`,
      );
    }

    this._status = OrderStatus.CANCELLED;
    this.updatedAt = new Date();

    this.addDomainEvent(
      new OrderCancelledEvent(this.id.value, reason.trim()),
    );
  }

  addLineItem(item: OrderLineItem): void {
    this.assertModifiable();
    const existing = this._lineItems.find((i) => i.productId === item.productId);
    if (existing) {
      throw new DomainError(
        `Product ${item.productId} already in order. Change quantity instead.`,
      );
    }
    this._lineItems.push(item);
    this.updatedAt = new Date();
  }

  removeLineItem(itemId: string): void {
    this.assertModifiable();
    const idx = this._lineItems.findIndex((i) => i.id === itemId);
    if (idx === -1) throw new DomainError(`Line item "${itemId}" not found`);
    this._lineItems.splice(idx, 1);
    if (!this._lineItems.length) {
      throw new DomainError('Order must have at least one line item');
    }
    this.updatedAt = new Date();
  }

  changeItemQuantity(itemId: string, newQuantity: number): void {
    this.assertModifiable();
    const item = this._lineItems.find((i) => i.id === itemId);
    if (!item) throw new DomainError(`Line item "${itemId}" not found`);
    item.changeQuantity(newQuantity);
    this.updatedAt = new Date();
  }

  // ── Private ────────────────────────────────────────────────

  private assertModifiable(): void {
    if (this._status !== OrderStatus.CREATED) {
      throw new DomainError(
        `Cannot modify order in status "${this._status}". Only CREATED orders can be modified.`,
      );
    }
  }
}
```

### Child Entities

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
    private _quantity: number,
    public readonly unitPrice: Money,
  ) {
    super(id);
    this.assertValidQuantity(_quantity);
  }

  get quantity(): number {
    return this._quantity;
  }

  get totalPrice(): Money {
    return this.unitPrice.multiply(this._quantity);
  }

  changeQuantity(newQuantity: number): void {
    this.assertValidQuantity(newQuantity);
    this._quantity = newQuantity;
  }

  private assertValidQuantity(qty: number): void {
    if (!Number.isInteger(qty) || qty <= 0) {
      throw new DomainError(`Quantity must be a positive integer, got: ${qty}`);
    }
  }
}
```

### Value Objects

```typescript
// modules/order/domain/value-objects/order-id.vo.ts

import { randomUUID } from 'crypto';

export class OrderId {
  constructor(public readonly value: string) {
    if (!value) throw new Error('OrderId cannot be empty');
  }

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
    if (amount < 0) throw new DomainError('Money amount cannot be negative');
    if (!currency || currency.length !== 3) {
      throw new DomainError('Currency must be a 3-letter ISO code');
    }
    super({ amount, currency: currency.toUpperCase() });
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(
      Math.round((this.amount + other.amount) * 100) / 100,
      this.currency,
    );
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    const result = Math.round((this.amount - other.amount) * 100) / 100;
    if (result < 0) throw new DomainError('Money subtraction would result in negative');
    return new Money(result, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(
      Math.round(this.amount * factor * 100) / 100,
      this.currency,
    );
  }

  isZero(): boolean {
    return this.amount === 0;
  }

  private assertSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new DomainError(`Currency mismatch: ${this.currency} vs ${other.currency}`);
    }
  }
}
```

```typescript
// modules/order/domain/value-objects/shipping-address.vo.ts

import { ValueObject } from '../../../../shared/domain/value-object.base';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

interface AddressProps {
  street: string;
  city: string;
  state: string;
  postalCode: string;
  country: string;
}

export class ShippingAddress extends ValueObject<AddressProps> {
  get street(): string { return this.props.street; }
  get city(): string { return this.props.city; }
  get state(): string { return this.props.state; }
  get postalCode(): string { return this.props.postalCode; }
  get country(): string { return this.props.country; }

  constructor(street: string, city: string, state: string, postalCode: string, country: string) {
    const fields = { street, city, state, postalCode, country };
    const missing = Object.entries(fields)
      .filter(([, v]) => !v || !v.trim())
      .map(([k]) => k);

    if (missing.length) {
      throw new DomainError(`Missing address fields: ${missing.join(', ')}`);
    }

    super({
      street: street.trim(),
      city: city.trim(),
      state: state.trim(),
      postalCode: postalCode.trim(),
      country: country.trim().toUpperCase(),
    });
  }

  toSingleLine(): string {
    return `${this.street}, ${this.city}, ${this.state} ${this.postalCode}, ${this.country}`;
  }
}
```

### Domain Events

```typescript
// modules/order/domain/events/order-created.event.ts

import { DomainEvent } from '../../../../shared/domain/domain-event.base';
import { Money } from '../value-objects/money.vo';

export class OrderCreatedEvent extends DomainEvent {
  constructor(
    public readonly orderId: string,
    public readonly customerId: string,
    public readonly totalAmount: Money,
    public readonly itemCount: number,
  ) {
    super();
  }

  get eventName(): string {
    return 'order.created';
  }
}
```

```typescript
// modules/order/domain/events/order-confirmed.event.ts

import { DomainEvent } from '../../../../shared/domain/domain-event.base';
import { Money } from '../value-objects/money.vo';

export class OrderConfirmedEvent extends DomainEvent {
  constructor(
    public readonly orderId: string,
    public readonly totalAmount: Money,
  ) {
    super();
  }

  get eventName(): string {
    return 'order.confirmed';
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

Stateless business logic that spans multiple entities or doesn't belong in a single aggregate.

```typescript
// modules/order/domain/services/order-total.service.ts

import { Money } from '../value-objects/money.vo';
import { OrderLineItem } from '../entities/order-line-item.entity';

export class OrderTotalService {
  /**
   * Sum all line item totals.
   */
  static calculate(lineItems: ReadonlyArray<OrderLineItem>): Money {
    return lineItems.reduce(
      (sum, item) => sum.add(item.totalPrice),
      new Money(0, lineItems[0].unitPrice.currency),
    );
  }

  /**
   * Domain rule: bulk discount.
   *  - 10-19 items → 5% off
   *  - 20-49 items → 10% off
   *  - 50+ items   → 15% off
   */
  static calculateWithDiscount(lineItems: ReadonlyArray<OrderLineItem>): Money {
    const total = OrderTotalService.calculate(lineItems);
    const totalQty = lineItems.reduce((sum, i) => sum + i.quantity, 0);

    if (totalQty >= 50) return total.multiply(0.85);
    if (totalQty >= 20) return total.multiply(0.90);
    if (totalQty >= 10) return total.multiply(0.95);
    return total;
  }
}
```

> **Layer 1 summary:** Pure TypeScript. No `@Injectable()`, no `@Inject()`, no NestJS, no TypeORM. Aggregates enforce all invariants. Domain events are collected but not dispatched yet.

---

## Layer 2 — Application (Use Cases + Ports)

Clean Architecture's Use Cases layer. Defines Hexagonal ports (inbound + outbound) and implements inbound ports as use cases that orchestrate the DDD domain model.

### Inbound Ports (Driving)

Formal contracts the presentation layer programs against. Each port = one use case.

```typescript
// modules/order/application/ports/inbound/create-order.port.ts

import { CreateOrderCommand } from '../../commands/create-order.command';

export interface CreateOrderResult {
  orderId: string;
  totalAmount: number;
  currency: string;
  itemCount: number;
}

export interface CreateOrderPort {
  execute(command: CreateOrderCommand): Promise<CreateOrderResult>;
}

export const CREATE_ORDER_PORT = Symbol('CreateOrderPort');
```

```typescript
// modules/order/application/ports/inbound/confirm-order.port.ts

import { ConfirmOrderCommand } from '../../commands/confirm-order.command';

export interface ConfirmOrderResult {
  orderId: string;
  status: string;
  chargedAmount: number;
}

export interface ConfirmOrderPort {
  execute(command: ConfirmOrderCommand): Promise<ConfirmOrderResult>;
}

export const CONFIRM_ORDER_PORT = Symbol('ConfirmOrderPort');
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

Contracts for every external dependency. Each port is a Symbol-based interface.

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
// modules/order/application/ports/outbound/payment-gateway.port.ts

export interface ChargeResult {
  transactionId: string;
  status: 'success' | 'failed' | 'pending';
}

export interface PaymentGatewayPort {
  charge(customerId: string, amountCents: number, currency: string): Promise<ChargeResult>;
  refund(transactionId: string): Promise<void>;
}

export const PAYMENT_GATEWAY_PORT = Symbol('PaymentGatewayPort');
```

```typescript
// modules/order/application/ports/outbound/inventory.port.ts

export interface InventoryPort {
  reserve(items: Array<{ productId: string; quantity: number }>): Promise<boolean>;
  release(items: Array<{ productId: string; quantity: number }>): Promise<void>;
}

export const INVENTORY_PORT = Symbol('InventoryPort');
```

```typescript
// modules/order/application/ports/outbound/notification.port.ts

export interface NotificationPort {
  sendOrderCreated(customerId: string, orderId: string): Promise<void>;
  sendOrderConfirmed(customerId: string, orderId: string): Promise<void>;
  sendOrderCancelled(customerId: string, orderId: string, reason: string): Promise<void>;
}

export const NOTIFICATION_PORT = Symbol('NotificationPort');
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
// modules/order/application/commands/create-order.command.ts

export class CreateOrderCommand {
  constructor(
    public readonly customerId: string,
    public readonly shippingAddress: {
      street: string;
      city: string;
      state: string;
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
// modules/order/application/commands/confirm-order.command.ts

export class ConfirmOrderCommand {
  constructor(public readonly orderId: string) {}
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

### Use Cases (Command Handlers)

Each use case **implements an inbound port** and depends on **outbound ports** via `@Inject(SYMBOL)`. The body follows: build domain objects → call aggregate methods → persist → dispatch events.

```typescript
// modules/order/application/use-cases/create-order.use-case.ts

import { Injectable, Inject } from '@nestjs/common';
import { CreateOrderPort, CreateOrderResult } from '../ports/inbound/create-order.port';
import { CreateOrderCommand } from '../commands/create-order.command';
import { ORDER_REPOSITORY_PORT, OrderRepositoryPort } from '../ports/outbound/order.repository.port';
import { INVENTORY_PORT, InventoryPort } from '../ports/outbound/inventory.port';
import { EVENT_BUS_PORT, EventBusPort } from '../../../../shared/ports/event-bus.port';
import { LOGGER_PORT, LoggerPort } from '../../../../shared/ports/logger.port';
import { Order } from '../../domain/aggregates/order.aggregate';
import { OrderLineItem } from '../../domain/entities/order-line-item.entity';
import { ShippingAddress } from '../../domain/value-objects/shipping-address.vo';
import { Money } from '../../domain/value-objects/money.vo';
import { DomainError } from '../../../../shared/exceptions/domain.exception';
import { randomUUID } from 'crypto';

@Injectable()
export class CreateOrderUseCase implements CreateOrderPort {
  constructor(
    @Inject(ORDER_REPOSITORY_PORT)
    private readonly orderRepo: OrderRepositoryPort,

    @Inject(INVENTORY_PORT)
    private readonly inventory: InventoryPort,

    @Inject(EVENT_BUS_PORT)
    private readonly eventBus: EventBusPort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,
  ) {}

  async execute(command: CreateOrderCommand): Promise<CreateOrderResult> {
    this.logger.log('Creating order', CreateOrderUseCase.name);

    // 1. Build domain value objects & entities
    const address = new ShippingAddress(
      command.shippingAddress.street,
      command.shippingAddress.city,
      command.shippingAddress.state,
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

    // 2. Reserve inventory (outbound port)
    const reserved = await this.inventory.reserve(
      command.items.map((i) => ({ productId: i.productId, quantity: i.quantity })),
    );
    if (!reserved) {
      throw new DomainError('Insufficient inventory for one or more items');
    }

    // 3. Create aggregate (domain factory enforces invariants, raises OrderCreatedEvent)
    const order = Order.create(command.customerId, address, lineItems);

    // 4. Persist
    await this.orderRepo.save(order);

    // 5. Dispatch all domain events collected by the aggregate
    await this.eventBus.publishAll(order.domainEvents);
    order.clearDomainEvents();

    this.logger.log(`Order created: ${order.id}`, CreateOrderUseCase.name);

    return {
      orderId: order.id.value,
      totalAmount: order.totalAmount.amount,
      currency: order.totalAmount.currency,
      itemCount: order.itemCount,
    };
  }
}
```

```typescript
// modules/order/application/use-cases/confirm-order.use-case.ts

import { Injectable, Inject } from '@nestjs/common';
import { ConfirmOrderPort, ConfirmOrderResult } from '../ports/inbound/confirm-order.port';
import { ConfirmOrderCommand } from '../commands/confirm-order.command';
import { ORDER_REPOSITORY_PORT, OrderRepositoryPort } from '../ports/outbound/order.repository.port';
import { PAYMENT_GATEWAY_PORT, PaymentGatewayPort } from '../ports/outbound/payment-gateway.port';
import { EVENT_BUS_PORT, EventBusPort } from '../../../../shared/ports/event-bus.port';
import { LOGGER_PORT, LoggerPort } from '../../../../shared/ports/logger.port';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

@Injectable()
export class ConfirmOrderUseCase implements ConfirmOrderPort {
  constructor(
    @Inject(ORDER_REPOSITORY_PORT)
    private readonly orderRepo: OrderRepositoryPort,

    @Inject(PAYMENT_GATEWAY_PORT)
    private readonly paymentGateway: PaymentGatewayPort,

    @Inject(EVENT_BUS_PORT)
    private readonly eventBus: EventBusPort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,
  ) {}

  async execute(command: ConfirmOrderCommand): Promise<ConfirmOrderResult> {
    this.logger.log(`Confirming order ${command.orderId}`, ConfirmOrderUseCase.name);

    // 1. Load aggregate
    const order = await this.orderRepo.findById(command.orderId);
    if (!order) throw new DomainError(`Order "${command.orderId}" not found`);

    // 2. Charge payment (outbound port)
    const amountCents = Math.round(order.discountedTotal.amount * 100);
    const payment = await this.paymentGateway.charge(
      order.customerId,
      amountCents,
      order.totalAmount.currency,
    );

    if (payment.status === 'failed') {
      throw new DomainError('Payment declined');
    }

    // 3. Domain state transition (aggregate enforces invariants, raises OrderConfirmedEvent)
    order.confirm();

    // 4. Persist
    await this.orderRepo.save(order);

    // 5. Dispatch events
    await this.eventBus.publishAll(order.domainEvents);
    order.clearDomainEvents();

    return {
      orderId: order.id.value,
      status: order.status,
      chargedAmount: amountCents / 100,
    };
  }
}
```

```typescript
// modules/order/application/use-cases/cancel-order.use-case.ts

import { Injectable, Inject } from '@nestjs/common';
import { CancelOrderPort } from '../ports/inbound/cancel-order.port';
import { CancelOrderCommand } from '../commands/cancel-order.command';
import { ORDER_REPOSITORY_PORT, OrderRepositoryPort } from '../ports/outbound/order.repository.port';
import { INVENTORY_PORT, InventoryPort } from '../ports/outbound/inventory.port';
import { EVENT_BUS_PORT, EventBusPort } from '../../../../shared/ports/event-bus.port';
import { LOGGER_PORT, LoggerPort } from '../../../../shared/ports/logger.port';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

@Injectable()
export class CancelOrderUseCase implements CancelOrderPort {
  constructor(
    @Inject(ORDER_REPOSITORY_PORT)
    private readonly orderRepo: OrderRepositoryPort,

    @Inject(INVENTORY_PORT)
    private readonly inventory: InventoryPort,

    @Inject(EVENT_BUS_PORT)
    private readonly eventBus: EventBusPort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,
  ) {}

  async execute(command: CancelOrderCommand): Promise<void> {
    this.logger.log(`Cancelling order ${command.orderId}`, CancelOrderUseCase.name);

    // 1. Load aggregate
    const order = await this.orderRepo.findById(command.orderId);
    if (!order) throw new DomainError(`Order "${command.orderId}" not found`);

    // 2. Domain state transition (aggregate validates cancellability)
    order.cancel(command.reason);

    // 3. Release inventory (compensating action)
    await this.inventory.release(
      order.lineItems.map((i) => ({ productId: i.productId, quantity: i.quantity })),
    );

    // 4. Persist
    await this.orderRepo.save(order);

    // 5. Dispatch events
    await this.eventBus.publishAll(order.domainEvents);
    order.clearDomainEvents();
  }
}
```

### Query Handlers

```typescript
// modules/order/application/queries/get-order.query-handler.ts

import { Injectable, Inject } from '@nestjs/common';
import { GetOrderPort } from '../ports/inbound/get-order.port';
import { GetOrderQuery } from './get-order.query';
import { ORDER_REPOSITORY_PORT, OrderRepositoryPort } from '../ports/outbound/order.repository.port';
import { OrderDto } from '../dtos/order.dto';
import { DomainError } from '../../../../shared/exceptions/domain.exception';

@Injectable()
export class GetOrderQueryHandler implements GetOrderPort {
  constructor(
    @Inject(ORDER_REPOSITORY_PORT)
    private readonly orderRepo: OrderRepositoryPort,
  ) {}

  async execute(query: GetOrderQuery): Promise<OrderDto> {
    const order = await this.orderRepo.findById(query.orderId);
    if (!order) throw new DomainError(`Order "${query.orderId}" not found`);

    return {
      id: order.id.value,
      customerId: order.customerId,
      status: order.status,
      shippingAddress: order.shippingAddress.toSingleLine(),
      totalAmount: order.totalAmount.amount,
      discountedTotal: order.discountedTotal.amount,
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
      updatedAt: order.updatedAt,
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
  shippingAddress: string;
  totalAmount: number;
  discountedTotal: number;
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
  updatedAt: Date;
}
```

### Domain Event Handlers

React to domain events dispatched via the event bus. Can call other outbound ports.

```typescript
// modules/order/application/event-handlers/on-order-created.handler.ts

import { Injectable, Inject } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { OrderCreatedEvent } from '../../domain/events/order-created.event';
import { NOTIFICATION_PORT, NotificationPort } from '../ports/outbound/notification.port';
import { LOGGER_PORT, LoggerPort } from '../../../../shared/ports/logger.port';

@Injectable()
export class OnOrderCreatedHandler {
  constructor(
    @Inject(NOTIFICATION_PORT)
    private readonly notification: NotificationPort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,
  ) {}

  @OnEvent('order.created')
  async handle(event: OrderCreatedEvent): Promise<void> {
    this.logger.log(`[Event] order.created: ${event.orderId}`, OnOrderCreatedHandler.name);
    await this.notification.sendOrderCreated(event.customerId, event.orderId);
  }
}
```

```typescript
// modules/order/application/event-handlers/on-order-confirmed.handler.ts

import { Injectable, Inject } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { OrderConfirmedEvent } from '../../domain/events/order-confirmed.event';
import { NOTIFICATION_PORT, NotificationPort } from '../ports/outbound/notification.port';
import { LOGGER_PORT, LoggerPort } from '../../../../shared/ports/logger.port';

@Injectable()
export class OnOrderConfirmedHandler {
  constructor(
    @Inject(NOTIFICATION_PORT)
    private readonly notification: NotificationPort,

    @Inject(LOGGER_PORT)
    private readonly logger: LoggerPort,
  ) {}

  @OnEvent('order.confirmed')
  async handle(event: OrderConfirmedEvent): Promise<void> {
    this.logger.log(`[Event] order.confirmed: ${event.orderId}`, OnOrderConfirmedHandler.name);
    // In a real system this might trigger warehouse fulfillment, invoice generation, etc.
    await this.notification.sendOrderConfirmed('customer', event.orderId);
  }
}
```

---

## Layer 3 — Infrastructure (Driven Adapters)

Implements all outbound ports with concrete framework/library code.

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
import { PaymentGatewayPort, ChargeResult } from '../../application/ports/outbound/payment-gateway.port';

@Injectable()
export class StripePaymentAdapter implements PaymentGatewayPort {
  async charge(
    customerId: string,
    amountCents: number,
    currency: string,
  ): Promise<ChargeResult> {
    // Stripe SDK call
    console.log(`[Stripe] Charging ${amountCents} ${currency} for customer ${customerId}`);
    return { transactionId: `ch_${Date.now()}`, status: 'success' };
  }

  async refund(transactionId: string): Promise<void> {
    console.log(`[Stripe] Refunding ${transactionId}`);
  }
}
```

### Inventory Adapter

```typescript
// modules/order/infrastructure/services/warehouse-inventory.adapter.ts

import { Injectable } from '@nestjs/common';
import { InventoryPort } from '../../application/ports/outbound/inventory.port';

@Injectable()
export class WarehouseInventoryAdapter implements InventoryPort {
  async reserve(items: Array<{ productId: string; quantity: number }>): Promise<boolean> {
    // Call warehouse API
    console.log(`[Warehouse] Reserving ${items.length} products`);
    return true;
  }

  async release(items: Array<{ productId: string; quantity: number }>): Promise<void> {
    console.log(`[Warehouse] Releasing ${items.length} products`);
  }
}
```

### Notification Adapter

```typescript
// modules/order/infrastructure/services/sendgrid-notification.adapter.ts

import { Injectable } from '@nestjs/common';
import { NotificationPort } from '../../application/ports/outbound/notification.port';

@Injectable()
export class SendGridNotificationAdapter implements NotificationPort {
  async sendOrderCreated(customerId: string, orderId: string): Promise<void> {
    console.log(`[SendGrid] Order created notification to ${customerId} for ${orderId}`);
  }

  async sendOrderConfirmed(customerId: string, orderId: string): Promise<void> {
    console.log(`[SendGrid] Order confirmed notification to ${customerId} for ${orderId}`);
  }

  async sendOrderCancelled(customerId: string, orderId: string, reason: string): Promise<void> {
    console.log(`[SendGrid] Order cancelled notification to ${customerId} for ${orderId}: ${reason}`);
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

import { Entity, PrimaryColumn, Column, CreateDateColumn, UpdateDateColumn, OneToMany } from 'typeorm';
import { OrderLineItemSchema } from './order-line-item.schema';

@Entity('orders')
export class OrderSchema {
  @PrimaryColumn('uuid') id: string;
  @Column() customerId: string;
  @Column() status: string;
  @Column() shippingStreet: string;
  @Column() shippingCity: string;
  @Column() shippingState: string;
  @Column() shippingPostalCode: string;
  @Column() shippingCountry: string;

  @OneToMany(() => OrderLineItemSchema, (item) => item.order, { cascade: true, eager: false })
  lineItems: OrderLineItemSchema[];

  @CreateDateColumn() createdAt: Date;
  @UpdateDateColumn() updatedAt: Date;
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
  @Column('decimal', { precision: 10, scale: 2 }) unitPrice: number;
  @Column({ length: 3 }) currency: string;

  @ManyToOne(() => OrderSchema, (order) => order.lineItems)
  order: OrderSchema;
}
```

```typescript
// modules/order/infrastructure/mappers/order.mapper.ts

import { Order, OrderStatus } from '../../domain/aggregates/order.aggregate';
import { OrderId } from '../../domain/value-objects/order-id.vo';
import { ShippingAddress } from '../../domain/value-objects/shipping-address.vo';
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
      new ShippingAddress(
        record.shippingStreet,
        record.shippingCity,
        record.shippingState,
        record.shippingPostalCode,
        record.shippingCountry,
      ),
      lineItems,
      record.status as OrderStatus,
      record.createdAt,
      record.updatedAt,
    );
  }

  static toPersistence(order: Order): Partial<OrderSchema> {
    return {
      id: order.id.value,
      customerId: order.customerId,
      status: order.status,
      shippingStreet: order.shippingAddress.street,
      shippingCity: order.shippingAddress.city,
      shippingState: order.shippingAddress.state,
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
      updatedAt: order.updatedAt,
    };
  }
}
```

---

## Layer 4 — Presentation (Driving Adapters)

Controllers call inbound ports via symbols. They have no knowledge of aggregates, repositories, or any infrastructure.

```typescript
// modules/order/presentation/controllers/order.controller.ts

import {
  Controller, Post, Get, Patch, Param, Body,
  Inject, HttpCode, HttpStatus,
} from '@nestjs/common';
import { CREATE_ORDER_PORT, CreateOrderPort } from '../../application/ports/inbound/create-order.port';
import { CONFIRM_ORDER_PORT, ConfirmOrderPort } from '../../application/ports/inbound/confirm-order.port';
import { CANCEL_ORDER_PORT, CancelOrderPort } from '../../application/ports/inbound/cancel-order.port';
import { GET_ORDER_PORT, GetOrderPort } from '../../application/ports/inbound/get-order.port';
import { CreateOrderCommand } from '../../application/commands/create-order.command';
import { ConfirmOrderCommand } from '../../application/commands/confirm-order.command';
import { CancelOrderCommand } from '../../application/commands/cancel-order.command';
import { GetOrderQuery } from '../../application/queries/get-order.query';
import { CreateOrderRequestDto } from '../dtos/create-order.request.dto';
import { OrderResponseDto } from '../dtos/order.response.dto';

@Controller('orders')
export class OrderController {
  constructor(
    @Inject(CREATE_ORDER_PORT)
    private readonly createOrder: CreateOrderPort,

    @Inject(CONFIRM_ORDER_PORT)
    private readonly confirmOrder: ConfirmOrderPort,

    @Inject(CANCEL_ORDER_PORT)
    private readonly cancelOrder: CancelOrderPort,

    @Inject(GET_ORDER_PORT)
    private readonly getOrder: GetOrderPort,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() dto: CreateOrderRequestDto) {
    return this.createOrder.execute(
      new CreateOrderCommand(dto.customerId, dto.shippingAddress, dto.items),
    );
  }

  @Patch(':id/confirm')
  async confirm(@Param('id') id: string) {
    return this.confirmOrder.execute(new ConfirmOrderCommand(id));
  }

  @Patch(':id/cancel')
  async cancel(@Param('id') id: string, @Body('reason') reason: string) {
    return this.cancelOrder.execute(new CancelOrderCommand(id, reason));
  }

  @Get(':id')
  async findOne(@Param('id') id: string): Promise<OrderResponseDto> {
    const dto = await this.getOrder.execute(new GetOrderQuery(id));
    return {
      ...dto,
      createdAt: dto.createdAt.toISOString(),
      updatedAt: dto.updatedAt.toISOString(),
    };
  }
}
```

### Request / Response DTOs

```typescript
// modules/order/presentation/dtos/create-order.request.dto.ts

import { IsString, IsArray, ValidateNested, IsNumber, Min, MinLength } from 'class-validator';
import { Type } from 'class-transformer';

class AddressDto {
  @IsString() @MinLength(1) street: string;
  @IsString() @MinLength(1) city: string;
  @IsString() @MinLength(1) state: string;
  @IsString() @MinLength(1) postalCode: string;
  @IsString() @MinLength(2) country: string;
}

class LineItemDto {
  @IsString() productId: string;
  @IsString() productName: string;
  @IsNumber() @Min(1) quantity: number;
  @IsNumber() @Min(0) unitPrice: number;
  @IsString() @MinLength(3) currency: string;
}

export class CreateOrderRequestDto {
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
  shippingAddress: string;
  totalAmount: number;
  discountedTotal: number;
  currency: string;
  lineItems: Array<{
    id: string;
    productName: string;
    quantity: number;
    unitPrice: number;
    totalPrice: number;
  }>;
  createdAt: string;
  updatedAt: string;
}
```

---

## Cross-Cutting Concerns

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
    if (value === undefined) throw new Error(`Missing config key: "${key}"`);
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

```typescript
// shared/shared.module.ts

import { Global, Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { EventEmitterModule } from '@nestjs/event-emitter';
import { LOGGER_PORT } from './ports/logger.port';
import { CONFIG_PORT } from './ports/config.port';
import { EVENT_BUS_PORT } from './ports/event-bus.port';
import { WinstonLoggerAdapter } from '../infrastructure/adapters/logger.winston.adapter';
import { NestConfigAdapter } from '../infrastructure/adapters/config.nestjs.adapter';
import { NestJsEventBusAdapter } from '../infrastructure/adapters/event-bus.nestjs.adapter';

@Global()
@Module({
  imports: [
    ConfigModule.forRoot(),
    EventEmitterModule.forRoot(),
  ],
  providers: [
    { provide: LOGGER_PORT,    useClass: WinstonLoggerAdapter },
    { provide: CONFIG_PORT,    useClass: NestConfigAdapter },
    { provide: EVENT_BUS_PORT, useClass: NestJsEventBusAdapter },
  ],
  exports: [LOGGER_PORT, CONFIG_PORT, EVENT_BUS_PORT],
})
export class SharedModule {}
```

---

## Module Wiring

The single place where all three patterns converge — ports are bound to adapters, use cases to inbound ports.

```typescript
// modules/order/order.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

// ── Inbound Ports ──────────────────────────────────────────
import { CREATE_ORDER_PORT } from './application/ports/inbound/create-order.port';
import { CONFIRM_ORDER_PORT } from './application/ports/inbound/confirm-order.port';
import { CANCEL_ORDER_PORT } from './application/ports/inbound/cancel-order.port';
import { GET_ORDER_PORT } from './application/ports/inbound/get-order.port';

// ── Outbound Ports ─────────────────────────────────────────
import { ORDER_REPOSITORY_PORT } from './application/ports/outbound/order.repository.port';
import { PAYMENT_GATEWAY_PORT } from './application/ports/outbound/payment-gateway.port';
import { INVENTORY_PORT } from './application/ports/outbound/inventory.port';
import { NOTIFICATION_PORT } from './application/ports/outbound/notification.port';

// ── Use Cases (implement inbound ports) ────────────────────
import { CreateOrderUseCase } from './application/use-cases/create-order.use-case';
import { ConfirmOrderUseCase } from './application/use-cases/confirm-order.use-case';
import { CancelOrderUseCase } from './application/use-cases/cancel-order.use-case';
import { GetOrderQueryHandler } from './application/queries/get-order.query-handler';

// ── Event Handlers ─────────────────────────────────────────
import { OnOrderCreatedHandler } from './application/event-handlers/on-order-created.handler';
import { OnOrderConfirmedHandler } from './application/event-handlers/on-order-confirmed.handler';

// ── Driven Adapters (implement outbound ports) ─────────────
import { OrderTypeOrmAdapter } from './infrastructure/repositories/order.typeorm.adapter';
import { StripePaymentAdapter } from './infrastructure/services/stripe-payment.adapter';
import { WarehouseInventoryAdapter } from './infrastructure/services/warehouse-inventory.adapter';
import { SendGridNotificationAdapter } from './infrastructure/services/sendgrid-notification.adapter';

// ── Schemas ────────────────────────────────────────────────
import { OrderSchema } from './infrastructure/schemas/order.schema';
import { OrderLineItemSchema } from './infrastructure/schemas/order-line-item.schema';

// ── Presentation ───────────────────────────────────────────
import { OrderController } from './presentation/controllers/order.controller';

@Module({
  imports: [TypeOrmModule.forFeature([OrderSchema, OrderLineItemSchema])],
  controllers: [OrderController],
  providers: [
    // ── Inbound Ports → Use Cases ──────────────────────────
    { provide: CREATE_ORDER_PORT,  useClass: CreateOrderUseCase },
    { provide: CONFIRM_ORDER_PORT, useClass: ConfirmOrderUseCase },
    { provide: CANCEL_ORDER_PORT,  useClass: CancelOrderUseCase },
    { provide: GET_ORDER_PORT,     useClass: GetOrderQueryHandler },

    // ── Outbound Ports → Driven Adapters ───────────────────
    { provide: ORDER_REPOSITORY_PORT, useClass: OrderTypeOrmAdapter },
    { provide: PAYMENT_GATEWAY_PORT,  useClass: StripePaymentAdapter },
    { provide: INVENTORY_PORT,        useClass: WarehouseInventoryAdapter },
    { provide: NOTIFICATION_PORT,     useClass: SendGridNotificationAdapter },

    // ── Event Handlers ─────────────────────────────────────
    OnOrderCreatedHandler,
    OnOrderConfirmedHandler,
  ],
})
export class OrderModule {}
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

### Swapping Any Adapter

```typescript
// Change DB: TypeORM → Prisma
{ provide: ORDER_REPOSITORY_PORT, useClass: OrderPrismaAdapter },

// Change payment: Stripe → PayPal
{ provide: PAYMENT_GATEWAY_PORT, useClass: PayPalPaymentAdapter },

// Change inventory: Warehouse API → In-memory (for testing)
{ provide: INVENTORY_PORT, useClass: InMemoryInventoryAdapter },

// Change event bus: NestJS EventEmitter → RabbitMQ
{ provide: EVENT_BUS_PORT, useClass: RabbitMqEventBusAdapter },

// Change logger: Winston → Pino
{ provide: LOGGER_PORT, useClass: PinoLoggerAdapter },
```

> Zero use case, domain, or controller code changes needed.

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
      const is404 = exception.message.toLowerCase().includes('not found');
      const status = is404 ? 404 : 400;
      response.status(status).json({
        statusCode: status,
        error: is404 ? 'Not Found' : 'Business Rule Violation',
        message: exception.message,
      });
      return;
    }

    this.logger.error(
      exception instanceof Error ? exception.message : String(exception),
      exception instanceof Error ? exception.stack : undefined,
    );
    response.status(500).json({ statusCode: 500, error: 'Internal Server Error' });
  }
}
```

---

## Testing Strategy

### Unit Test — Aggregate (Layer 1, no mocks)

```typescript
// modules/order/domain/aggregates/__tests__/order.aggregate.spec.ts

import { Order, OrderStatus } from '../order.aggregate';
import { OrderLineItem } from '../../entities/order-line-item.entity';
import { ShippingAddress } from '../../value-objects/shipping-address.vo';
import { Money } from '../../value-objects/money.vo';
import { DomainError } from '../../../../../shared/exceptions/domain.exception';

const makeItem = (qty = 2, price = 10) =>
  new OrderLineItem('item-1', 'prod-1', 'Widget', qty, new Money(price, 'USD'));
const makeAddress = () =>
  new ShippingAddress('123 Main', 'Springfield', 'IL', '62701', 'US');

describe('Order Aggregate', () => {
  it('should create with CREATED status and raise event', () => {
    const order = Order.create('cust-1', makeAddress(), [makeItem()]);
    expect(order.status).toBe(OrderStatus.CREATED);
    expect(order.domainEvents).toHaveLength(1);
    expect(order.domainEvents[0].eventName).toBe('order.created');
  });

  it('should calculate total correctly', () => {
    const order = Order.create('cust-1', makeAddress(), [makeItem(3, 15)]);
    expect(order.totalAmount.amount).toBe(45); // 3 × $15
  });

  it('should confirm and raise event', () => {
    const order = Order.create('cust-1', makeAddress(), [makeItem()]);
    order.confirm();
    expect(order.status).toBe(OrderStatus.CONFIRMED);
    expect(order.domainEvents).toHaveLength(2); // created + confirmed
  });

  it('should reject confirming a non-CREATED order', () => {
    const order = Order.create('cust-1', makeAddress(), [makeItem()]);
    order.confirm();
    expect(() => order.confirm()).toThrow(DomainError);
  });

  it('should cancel with reason', () => {
    const order = Order.create('cust-1', makeAddress(), [makeItem()]);
    order.cancel('Changed my mind');
    expect(order.status).toBe(OrderStatus.CANCELLED);
  });

  it('should reject short cancellation reason', () => {
    const order = Order.create('cust-1', makeAddress(), [makeItem()]);
    expect(() => order.cancel('No')).toThrow(DomainError);
  });

  it('should reject empty line items', () => {
    expect(() => Order.create('cust-1', makeAddress(), []))
      .toThrow(DomainError);
  });

  it('should not modify after confirmation', () => {
    const order = Order.create('cust-1', makeAddress(), [makeItem()]);
    order.confirm();
    expect(() => order.addLineItem(makeItem())).toThrow(DomainError);
  });
});
```

### Unit Test — Use Case (Layer 2, mock all outbound ports)

```typescript
// modules/order/application/use-cases/__tests__/create-order.use-case.spec.ts

import { CreateOrderUseCase } from '../create-order.use-case';
import { OrderRepositoryPort } from '../../ports/outbound/order.repository.port';
import { InventoryPort } from '../../ports/outbound/inventory.port';
import { EventBusPort } from '../../../../../shared/ports/event-bus.port';
import { LoggerPort } from '../../../../../shared/ports/logger.port';
import { CreateOrderCommand } from '../../commands/create-order.command';
import { DomainError } from '../../../../../shared/exceptions/domain.exception';

describe('CreateOrderUseCase', () => {
  let useCase: CreateOrderUseCase;
  let mockRepo: jest.Mocked<OrderRepositoryPort>;
  let mockInventory: jest.Mocked<InventoryPort>;
  let mockEventBus: jest.Mocked<EventBusPort>;
  let mockLogger: jest.Mocked<LoggerPort>;

  const validCommand = new CreateOrderCommand(
    'cust-1',
    { street: '123 Main', city: 'Springfield', state: 'IL', postalCode: '62701', country: 'US' },
    [{ productId: 'p1', productName: 'Widget', quantity: 2, unitPrice: 10, currency: 'USD' }],
  );

  beforeEach(() => {
    mockRepo = {
      findById: jest.fn(),
      findByCustomerId: jest.fn(),
      save: jest.fn().mockResolvedValue(undefined),
      delete: jest.fn(),
    };
    mockInventory = {
      reserve: jest.fn().mockResolvedValue(true),
      release: jest.fn(),
    };
    mockEventBus = {
      publish: jest.fn().mockResolvedValue(undefined),
      publishAll: jest.fn().mockResolvedValue(undefined),
    };
    mockLogger = { log: jest.fn(), error: jest.fn(), warn: jest.fn(), debug: jest.fn() };

    useCase = new CreateOrderUseCase(mockRepo, mockInventory, mockEventBus, mockLogger);
  });

  it('should create order, reserve inventory, persist, and dispatch events', async () => {
    const result = await useCase.execute(validCommand);

    expect(result.orderId).toBeDefined();
    expect(result.totalAmount).toBe(20);
    expect(mockInventory.reserve).toHaveBeenCalledWith([{ productId: 'p1', quantity: 2 }]);
    expect(mockRepo.save).toHaveBeenCalledTimes(1);
    expect(mockEventBus.publishAll).toHaveBeenCalledTimes(1);
  });

  it('should fail if inventory cannot be reserved', async () => {
    mockInventory.reserve.mockResolvedValueOnce(false);

    await expect(useCase.execute(validCommand)).rejects.toThrow('Insufficient inventory');
    expect(mockRepo.save).not.toHaveBeenCalled();
  });
});
```

### Unit Test — Value Object (Layer 1, no mocks)

```typescript
// modules/order/domain/value-objects/__tests__/money.vo.spec.ts

import { Money } from '../money.vo';
import { DomainError } from '../../../../../shared/exceptions/domain.exception';

describe('Money', () => {
  it('should reject negative amount', () => {
    expect(() => new Money(-1, 'USD')).toThrow(DomainError);
  });

  it('should reject invalid currency', () => {
    expect(() => new Money(10, 'US')).toThrow(DomainError);
  });

  it('should add same currency', () => {
    expect(new Money(10, 'USD').add(new Money(20, 'USD')).amount).toBe(30);
  });

  it('should reject cross-currency add', () => {
    expect(() => new Money(10, 'USD').add(new Money(10, 'EUR'))).toThrow(DomainError);
  });

  it('should multiply', () => {
    expect(new Money(10.5, 'USD').multiply(3).amount).toBe(31.5);
  });
});
```

### Test Coverage Map

| Layer | What | Mocks Needed | Priority |
|-------|------|-------------|----------|
| **Domain — Aggregate** | State transitions, invariants, events | None | Highest |
| **Domain — Value Object** | Validation, equality, operations | None | High |
| **Domain — Domain Service** | Cross-entity calculations | None | High |
| **Application — Use Case** | Orchestration, port interactions, error paths | All outbound ports | High |
| **Infrastructure — Mapper** | Domain ↔ Persistence lossless round-trip | None | Medium |
| **Infrastructure — Repository** | Query correctness, save/load | Test DB or container | Medium |
| **Presentation — Controller** | HTTP codes, validation, DTO mapping | Inbound ports | Medium |

---

## Quick Reference

| Component | File Location | Pattern |
|-----------|---------------|---------|
| `Order` (Aggregate Root) | `domain/aggregates/` | DDD |
| `OrderLineItem` (Entity) | `domain/entities/` | DDD |
| `Money`, `ShippingAddress`, `OrderId` | `domain/value-objects/` | DDD |
| `OrderCreatedEvent` | `domain/events/` | DDD |
| `OrderTotalService` | `domain/services/` | DDD |
| `CreateOrderPort` | `application/ports/inbound/` | Hexagonal |
| `OrderRepositoryPort` | `application/ports/outbound/` | Hexagonal |
| `PaymentGatewayPort`, `InventoryPort` | `application/ports/outbound/` | Hexagonal |
| `EventBusPort`, `LoggerPort` | `shared/ports/` | Hexagonal |
| `CreateOrderUseCase` | `application/use-cases/` | Clean Arch + Hexagonal |
| `GetOrderQueryHandler` | `application/queries/` | Clean Arch |
| `OnOrderCreatedHandler` | `application/event-handlers/` | DDD |
| `OrderTypeOrmAdapter` | `infrastructure/repositories/` | Hexagonal |
| `StripePaymentAdapter` | `infrastructure/services/` | Hexagonal |
| `OrderMapper` | `infrastructure/mappers/` | Clean Arch |
| `OrderController` | `presentation/controllers/` | Clean Arch + Hexagonal |

---

## Pattern Comparison Matrix

| Aspect | Clean Arch | DDD | Ports & Adapters | **Clean + DDD + Ports** |
|--------|-----------|-----|-----------------|------------------------|
| Layer structure | 4 concentric layers | Tactical patterns, no prescribed layers | Driving/driven hexagon | 4 layers + DDD in domain + ports at boundaries |
| Domain model | Entities | Aggregates, VOs, Events | Not prescribed | Full DDD tactical |
| External dependencies | Via interfaces | Via repository abstractions | Via formal ports (Symbol) | Ports (Symbol) for everything |
| Controller → Use Case | Direct or interface | Direct | Via driving port | Via inbound port (Symbol) |
| Event handling | Not prescribed | Domain events | Not prescribed | Domain events via EventBusPort |
| Transactional boundary | Entity | Aggregate root | Not prescribed | Aggregate root |
| Boilerplate | Low-Medium | Medium | Medium-High | High |
| Pluggability | Good | Medium | Maximum | Maximum |
| Domain richness | Basic | Maximum | Basic | Maximum |
| **Best for** | General purpose | Complex business logic | Integration-heavy systems | **Large, complex, integration-heavy systems** |

---

## When to Use This Pattern

**Use Clean + DDD + Ports when:**
- The domain is genuinely complex (many business rules, state machines, invariants)
- Multiple external integrations that may change (DB, payment, email, inventory, queues)
- Large team where formal contracts (ports) serve as team boundaries
- Long-lived project that must evolve without rewriting core logic

**Don't use when:**
- Simple CRUD application
- Small team, few integrations
- Prototype or MVP phase
- The overhead of ports + aggregates isn't justified by complexity

---

## Further Reading

- [The Clean Architecture — Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Domain-Driven Design — Eric Evans](https://www.domainlanguage.com/ddd/)
- [Hexagonal Architecture — Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- [Implementing DDD — Vaughn Vernon](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/)
- [NestJS Custom Providers](https://docs.nestjs.com/fundamentals/custom-providers)
- [NestJS Event Emitter](https://docs.nestjs.com/techniques/events)
