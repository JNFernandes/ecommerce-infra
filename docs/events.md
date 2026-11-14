# Event contracts

Every service talks to the others only through Kafka events. This file is the single reference for the topics, who publishes and consumes them, and the JSON each message carries. When a service changes an event, update this file in the same change.

Status: **implemented** means the code exists today. **planned** means the contract is agreed but not built yet.

## Conventions

Every event is a JSON object with these envelope fields:

| Field | Type | Meaning |
|---|---|---|
| `eventId` | UUID | Unique id of this event. Consumers use it for idempotency. A republish of the same event keeps the same `eventId`. |
| `aggregateId` | UUID | Id of the aggregate that raised the event. Also the Kafka message key. |
| `occurredAt` | ISO 8601 timestamp | When it happened. |
| `version` | integer | Schema version. Additive changes (new optional fields) keep the version. |

Rules that apply to every service:

- **Key:** the Kafka message key is the `aggregateId`, so events for one aggregate stay ordered inside a partition.
- **Which id identifies the order:** Orders events use the order id as `aggregateId`. Payments and Inventory events use their own aggregate id (payment id, reservation id) and carry the order id in a separate `orderId` field. Consumers must read `orderId` for those.
- **Publishing:** save to the database first, publish second. A failed publish goes to the service's dead-letter table and a retry worker redelivers it, so delivery is at-least-once.
- **Consuming:** consumers are idempotent by `eventId`, commit the offset only after the database save, and retry with exponential backoff.
- **Startup:** a service must start without Kafka being reachable.
- **Topics:** auto-creation is off. Topics are created by `kafka-init` in `docker-compose.yml` (3 partitions, replication factor 1 locally).

## Topics

| Topic | Published by | Consumed by | Status |
|---|---|---|---|
| `orders.order-placed` | Orders | Payments, Inventory | implemented |
| `orders.order-confirmed` | Orders | Notifications | implemented (published), planned (consumer) |
| `orders.order-cancelled` | Orders | Payments, Inventory, Notifications | implemented (Payments, Inventory), planned (Notifications) |
| `payments.payment-processed` | Payments | Orders | implemented |
| `payments.payment-failed` | Payments | Orders | implemented |
| `payments.payment-refunded` | Payments | Notifications | implemented (published), planned (consumer) |
| `inventory.stock-reserved` | Inventory | Orders | implemented (published), planned (consumer) |
| `inventory.stock-rejected` | Inventory | Orders | implemented (published), planned (consumer) |
| `inventory.stock-released` | Inventory | Notifications (optional) | implemented (published), planned (consumer) |

## Orders events

### `orders.order-placed`

Published after the order is saved. `aggregateId` is the order id.

| Field | Type | Notes |
|---|---|---|
| `customerId` | UUID | The customer who placed the order. |
| `items` | array | Consolidated line items, see below. |
| `totalAmount` | number | Sum of all item subtotals. |

Item: `{ "productId": string, "quantity": integer, "unitPrice": number }`.

### `orders.order-confirmed`

Published after a pending order is confirmed. `aggregateId` is the order id.

| Field | Type | Notes |
|---|---|---|
| `customerId` | UUID | |

### `orders.order-cancelled`

Published after a pending order is cancelled. `aggregateId` is the order id. Only `PENDING` orders can be cancelled.

| Field | Type | Notes |
|---|---|---|
| `customerId` | UUID | |
| `reason` | string, optional | Present only when the cancellation had a reason (today: payment failures). Omitted otherwise. |

This event is the compensation trigger. Payments refunds a processed payment for the order, and Inventory releases a reserved stock. Both ignore the event when there is nothing to undo.

## Payments events

`aggregateId` is the payment id. The order is identified by `orderId`.

### `payments.payment-processed`

| Field | Type | Notes |
|---|---|---|
| `orderId` | UUID | |
| `amount` | decimal | Amount charged. |
| `currency` | string | ISO 4217. |
| `processedAt` | timestamp | |

### `payments.payment-failed`

| Field | Type | Notes |
|---|---|---|
| `orderId` | UUID | |
| `reason` | string | Business reason (for example the amount exceeds the threshold). |
| `failedAt` | timestamp | |

### `payments.payment-refunded`

Published after a Processed payment is refunded because its order was cancelled.

| Field | Type | Notes |
|---|---|---|
| `orderId` | UUID | |
| `amount` | decimal | The full amount of the payment. |
| `currency` | string | ISO 4217. |
| `refundedAt` | timestamp | |

## Inventory events

`aggregateId` is the reservation id. The order is identified by `orderId`.

Inventory consolidates the order's lines first (lines for the same `productId` are summed), so
`quantity` and `requested` are consolidated totals. They are JSON integers that can exceed the
32-bit range; read them as 64-bit integers.

### `inventory.stock-reserved`

All items of the order were reserved atomically.

| Field | Type | Notes |
|---|---|---|
| `orderId` | UUID | |
| `items` | array | `{ "productId": string, "quantity": integer }`, one per product, sorted by `productId` |
| `reservedAt` | timestamp | |

### `inventory.stock-rejected`

At least one item could not be reserved, and nothing was reserved.

| Field | Type | Notes |
|---|---|---|
| `orderId` | UUID | |
| `reason` | string | Exactly one of `insufficient stock`, `unknown product`, `unknown product and insufficient stock` (both kinds of failure in the same order). |
| `items` | array | Only the items that failed, sorted by `productId`: `{ "productId": string, "requested": integer, "available": integer }`. `available` is `0` for an unknown product. |
| `rejectedAt` | timestamp | |

### `inventory.stock-released`

A reservation was released because its order was cancelled. Inventory publishes it only when stock
was actually given back: cancelling a rejected or already released order publishes nothing.

| Field | Type | Notes |
|---|---|---|
| `orderId` | UUID | |
| `items` | array | The consolidated reserved quantities, one per product, sorted by `productId`: `{ "productId": string, "quantity": integer }` |
| `releasedAt` | timestamp | |

## Order saga

```
Orders ── OrderPlaced ──┬─► Payments  ── PaymentProcessed / PaymentFailed ──┐
                        └─► Inventory ── StockReserved / StockRejected ─────┤
                                                                            ▼
                                                           Orders waits for both outcomes
                                                    both succeeded  → Confirmed (OrderConfirmed)
                                                    any failed      → Cancelled (OrderCancelled)
                                                                            │
                                          ┌─────────────────────────────────┘
                                          ▼
                       Payments refunds a Processed payment (PaymentRefunded)
                       Inventory releases a Reserved stock   (StockReleased)
```

Planned behavior: until the Inventory stories land, Orders confirms on `PaymentProcessed` alone.

Cancellation before placement: the two Orders topics are separate, so Inventory can receive
`OrderCancelled` before `OrderPlaced` for the same order. Inventory then records the cancellation and
ignores the later `OrderPlaced` (nothing is reserved and nothing is published for it), so stock is
never held for an order that is already cancelled.
