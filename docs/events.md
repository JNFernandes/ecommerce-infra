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
| `orders.order-placed` | Orders | Payments, Inventory, Notifications | implemented |
| `orders.order-confirmed` | Orders | Notifications | implemented |
| `orders.order-cancelled` | Orders | Payments, Inventory, Notifications | implemented |
| `payments.payment-processed` | Payments | Orders | implemented |
| `payments.payment-failed` | Payments | Orders | implemented |
| `payments.payment-refunded` | Payments | Notifications | implemented |
| `inventory.stock-reserved` | Inventory | Orders | implemented |
| `inventory.stock-rejected` | Inventory | Orders | implemented |
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

Notifications uses `customerId` and `totalAmount` (read as an exact decimal) to record the "order
received" notification, and remembers both per order so it can address the later refund
notification, whose event has no `customerId`. It does not read `items`.

### `orders.order-confirmed`

Published after a pending order is confirmed: when both its payment is processed and its stock is reserved (see [Order saga](#order-saga)), or when an operator confirms it manually. `aggregateId` is the order id.

| Field | Type | Notes |
|---|---|---|
| `customerId` | UUID | |

### `orders.order-cancelled`

Published after a pending order is cancelled. `aggregateId` is the order id. Only `PENDING` orders can be cancelled.

| Field | Type | Notes |
|---|---|---|
| `customerId` | UUID | |
| `reason` | string, optional | Present when the cancellation came from an outcome: `payment failed: <reason>` or `stock rejected: <reason>` (just `payment failed` / `stock rejected` when the source reason is empty). Omitted for a cancellation requested through the Orders API. |

This event is the compensation trigger. Payments refunds a processed payment for the order, and Inventory releases a reserved stock. Both ignore the event when there is nothing to undo.

Notifications records an "order cancelled" notification for `customerId` and shows `reason` in the
message when present (trimmed, on one line, at most 500 characters); without a reason the message
is generic. A `reason` that is not a string or `null` makes the message invalid. Notifications
also records an "order confirmed" notification for every `orders.order-confirmed` event. Neither
event needs `orders.order-placed` to have been received first.

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

Notifications records a "payment refunded" notification with the amount and currency. The event
has no `customerId`, so Notifications finds the customer through the order context it recorded
from `orders.order-placed` (by `orderId`, never `aggregateId`). If that context is missing, the
refund is retried and then dead-lettered; it is never sent without a customer.

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

Notifications listens alongside the saga and never takes part in it (it publishes nothing):
  OrderPlaced     → ORDER_RECEIVED      OrderCancelled  → ORDER_CANCELLED
  OrderConfirmed  → ORDER_CONFIRMED     PaymentRefunded → PAYMENT_REFUNDED
```

Notifications (implemented) consumes `orders.order-placed`, `orders.order-confirmed`,
`orders.order-cancelled` and `payments.payment-refunded` in the consumer group
`notifications-service` and records one notification per event (idempotent by `eventId`). A
confirmed order therefore produces `ORDER_RECEIVED` and `ORDER_CONFIRMED`; a cancelled order
produces `ORDER_RECEIVED` and `ORDER_CANCELLED`, plus `PAYMENT_REFUNDED` when its payment had been
processed and is refunded. It does not consume `inventory.stock-released`.

How Orders joins the two outcomes (Orders US-09):

- The outcomes can arrive in either order. The first positive one is recorded and the order stays
  pending; the order is confirmed when the second positive one arrives.
- The first negative outcome cancels the order at once, without waiting for the other one.
- An outcome for an order that is no longer pending is ignored (and logged). A `PaymentProcessed`
  for an order that is already cancelled is logged as a warning, because that payment needs the
  refund that `OrderCancelled` triggers.
- A second outcome on the same side that contradicts the first (a failure after a processed
  payment, a rejection after a reserved stock) is ignored with a warning: the first one wins.

Cancellation before placement: the two Orders topics are separate, so Inventory can receive
`OrderCancelled` before `OrderPlaced` for the same order. Inventory then records the cancellation and
ignores the later `OrderPlaced` (nothing is reserved and nothing is published for it), so stock is
never held for an order that is already cancelled.

## Notifications read API

Notifications publishes no events. What it recorded is read over a read-only HTTP API (port 8000;
no authentication, local and demo use only; other methods answer 405). OpenAPI at
`/openapi.json`, interactive docs at `/docs`. Source of truth:
[`ecommerce-notifications`](../../ecommerce-notifications/README.md).

| Endpoint | Purpose | Status |
|---|---|---|
| `GET /notifications` | List notifications, newest first, filtered and paged. | implemented |
| `GET /notifications/{id}` | One notification; 404 when no notification has this id. | implemented |
| `GET /health/live`, `GET /health/ready` | Liveness; readiness (200 when MongoDB answers, 503 otherwise; reports the Kafka consumer state without failing on it). | implemented |

### `GET /notifications`

| Query parameter | Type | Default | Rule |
|---|---|---|---|
| `customerId` | UUID | | optional filter |
| `orderId` | UUID | | optional filter |
| `type` | notification type (see below) | | optional filter |
| `limit` | integer | 20 | 1 to 100 |
| `offset` | integer | 0 | 0 or more |

Filters combine with AND. Items are ordered newest first: `createdAt` descending, then `id`
descending, so paging an unchanged data set never skips or repeats one. Response:

```json
{
  "items": [ { "id": "…", "customerId": "…", "orderId": "…", "type": "ORDER_CONFIRMED", "…": "…" } ],
  "total": 1,
  "limit": 20,
  "offset": 0
}
```

`total` is the exact number of matching notifications across all pages; `limit` and `offset` are
the values applied.

### `GET /notifications/{id}`

Returns one notification (the same shape as an `items` element). An unknown id answers 404 with
`{"detail": "notification <id> not found"}`.

Errors use FastAPI's standard body: 422 for an unknown `type`, an out-of-range or non-integer
`limit` / `offset`, or a malformed UUID; 503 `{"detail": "database unavailable"}` when MongoDB is
unreachable (never reported as an empty result).

### Notification fields

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Notification id. |
| `customerId` | UUID | The customer it is for. |
| `orderId` | UUID | The order it is about. |
| `type` | string | One of the types below. |
| `channel` | string | Always `email` (sending is simulated: stored and logged, no real provider). |
| `message` | string | Text for the customer. |
| `status` | string | Always `SENT`. |
| `createdAt` | ISO 8601 timestamp, UTC | When it was recorded. |
| `sourceEventId` | UUID | `eventId` of the event that produced it; unique, so a redelivered event adds nothing. |

### Notification types

| Type | Produced by | Message | Status |
|---|---|---|---|
| `ORDER_RECEIVED` | `orders.order-placed` | `Your order <orderId> for <totalAmount> was received.` | implemented |
| `ORDER_CONFIRMED` | `orders.order-confirmed` | `Your order <orderId> was confirmed.` | implemented |
| `ORDER_CANCELLED` | `orders.order-cancelled` | `Your order <orderId> was cancelled: <reason>.`, or `Your order <orderId> was cancelled.` without a reason | implemented |
| `PAYMENT_REFUNDED` | `payments.payment-refunded` | `Your payment of <amount> <currency> for order <orderId> was refunded.` (customer from the order's `orders.order-placed`) | implemented |
