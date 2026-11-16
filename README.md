# ecommerce-infra

Local runtime for the ecommerce platform: a set of microservices that place an order, charge it
and reserve its stock, coordinated only through Kafka events (a choreographed saga). This
repository holds the shared infrastructure and a `docker-compose.yml` that can also build and run
the services from their sibling repositories.

| Service | Repository | Stack | Role |
|---|---|---|---|
| Orders | `../ecommerce-orders` | NestJS, TypeORM | REST API to place, cancel and confirm orders; GraphQL read model; joins the payment and stock outcomes |
| Payments | `../ecommerce-payments` | .NET worker, EF Core | Charges each placed order (fails above 10 000.00), refunds cancelled orders |
| Inventory | `../ecommerce-inventory` | Spring Boot, Flyway | Reserves stock for each placed order, releases it when the order is cancelled |

## Architecture

```
                 REST :3000 / GraphQL :3000/graphql
                                │
                          ┌─────▼─────┐   writes     ┌──────────────────┐
                          │  Orders   ├─────────────►│ postgres-orders  │
                          │           │   read model │ mongo (orders_read)
                          └──┬─────▲──┘─────────────►└──────────────────┘
          orders.order-placed│     │payments.payment-processed / -failed
       orders.order-cancelled│     │inventory.stock-reserved / -rejected
       orders.order-confirmed│     │
                          ┌──▼─────┴──┐
                          │   Kafka   │  KRaft, topics created by kafka-init
                          └──┬─────▲──┘
              ┌──────────────┘     └──────────────┐
        ┌─────▼─────┐                       ┌─────┴─────┐
        │ Payments  │ payments.payment-     │ Inventory │ inventory.stock-released
        │           │ refunded              │           │
        └─────┬─────┘                       └─────┬─────┘
     postgres-payments                     postgres-inventory
```

An order starts `PENDING`. It becomes `CONFIRMED` when the payment is processed **and** the stock
is reserved, and `CANCELLED` as soon as either fails. A cancellation makes Payments refund a
processed payment and Inventory release a reserved stock. Topics, payloads and the saga are
specified in [docs/events.md](docs/events.md).

## Prerequisites

- Docker with Docker Compose v2 (`docker compose`).
- For the `apps` profile: the service repositories cloned next to this one:

  ```
  ecommerce-project/
  ├── ecommerce-infra/        (this repository)
  ├── ecommerce-orders/
  ├── ecommerce-payments/
  └── ecommerce-inventory/
  ```

- Free host ports: 3000, 5432, 5433, 5434, 8080 (only with `tools`), 8081, 9092, 27017.

## Run

Infrastructure only (Kafka, the three Postgres databases, MongoDB), for running the services from
your IDE against `localhost`:

```bash
docker compose up -d
```

Everything, with the services built from the sibling repositories:

```bash
docker compose --profile apps up -d --build
```

Add `--profile tools` to either command to also start Kafka UI on http://localhost:8080.

`docker compose --profile apps ps` should show `orders` and `inventory` as `healthy`,
`payments` as `Up` (it has no HTTP endpoint). Add `-a` to also see `kafka-init` and `orders-seed`,
which should both be `Exited (0)`.

What happens on start:

1. Kafka, Postgres and MongoDB start and report healthy.
2. `kafka-init` creates the topics (idempotent) and exits.
3. The services start once their database and Kafka are healthy and `kafka-init` has finished.
   They restart on failure, so a service that loses Kafka at startup recovers.
4. Each service migrates its own schema on startup: Orders with TypeORM, Payments with EF Core,
   Inventory with Flyway.
5. Demo data is loaded, idempotently (existing rows are never changed):
   - Orders: customer `3fa85f64-5717-4562-b3fc-2c963f66afa6`, by the one-shot `orders-seed`
     container (the `seed:customers` script), which runs once Orders is healthy and then exits.
   - Inventory: the sample stock below (Spring profile `local`, `db/local/R__seed_local_stock.sql`).

| Product | Available |
|---|---|
| `SKU-001` | 10 |
| `SKU-002` | 25 |
| `SKU-003` | 100 |
| `SKU-004` | 5 |
| `SKU-005` | 0 |

## Ports and credentials

Services on the host use `localhost` and the host port. Containers use the compose service name
and the internal port.

| Component | Host | Internal | Database | User / password |
|---|---|---|---|---|
| Orders (REST, GraphQL, Swagger at `/api`) | `localhost:3000` | `orders:3000` | | |
| Inventory actuator health | `localhost:8081` | `inventory:8080` | | |
| Payments | not published | | | |
| Kafka | `localhost:9092` | `kafka:29092` | | |
| Kafka UI (`tools` profile) | `localhost:8080` | | | |
| Postgres Orders | `localhost:5432` | `postgres-orders:5432` | `orders` | `orders_user` / `orders_pass` |
| Postgres Inventory | `localhost:5433` | `postgres-inventory:5432` | `inventory` | `inventory_user` / `inventory_pass` |
| Postgres Payments | `localhost:5434` | `postgres-payments:5432` | `payments` | `payments_user` / `payments_pass` |
| MongoDB | `localhost:27017` | `mongo:27017` | `orders_read` (Orders read model) | `mongo_user` / `mongo_pass`, auth database `admin` |

These credentials are for local development only.

### Kafka topics

Created by `kafka-init` with 3 partitions and replication factor 1 (auto-creation is off):
`orders.order-placed`, `orders.order-confirmed`, `orders.order-cancelled`,
`payments.payment-processed`, `payments.payment-failed`, `payments.payment-refunded`,
`inventory.stock-reserved`, `inventory.stock-rejected`, `inventory.stock-released`.

## Demo scenarios

Start everything with `--profile apps`. Every scenario places an order for the seeded customer,
then reads it back through GraphQL. The saga takes a second or two, so a query right after the
`POST` may still show `PENDING`; query again.

You can also use Postman, or Swagger at http://localhost:3000/api to place orders.

### bash (Linux, macOS, Git Bash, WSL)

Place an order:

```bash
curl -s -X POST localhost:3000/orders -H 'content-type: application/json' -d '{
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "items": [{ "productId": "SKU-003", "quantity": 2, "unitPrice": 19.99 }]
}'
# → 201 {"orderId":"<id>"}
```

Read it back (replace `<id>`):

```bash
curl -s localhost:3000/graphql -H 'content-type: application/json' -d '{
  "query": "{ order(orderId: \"<id>\") { orderId status totalAmount cancellationReason items { productId quantity } } }"
}'
```

### PowerShell (Windows)

In Windows PowerShell, `curl` is an alias for `Invoke-WebRequest`, so use `Invoke-RestMethod`:

```powershell
# Place an order
$body = @{
  customerId = '3fa85f64-5717-4562-b3fc-2c963f66afa6'
  items      = @(@{ productId = 'SKU-003'; quantity = 2; unitPrice = 19.99 })
} | ConvertTo-Json -Depth 5
$r = Invoke-RestMethod -Method Post -Uri http://localhost:3000/orders -ContentType 'application/json' -Body $body
$r.orderId

# Read it back
$q = @{ query = "{ order(orderId: `"$($r.orderId)`") { orderId status totalAmount cancellationReason items { productId quantity } } }" } | ConvertTo-Json
(Invoke-RestMethod -Method Post -Uri http://localhost:3000/graphql -ContentType 'application/json' -Body $q).data.order | Format-List
```

For the other scenarios, change `items` as in the table below, for example
`items = @(@{ productId = 'SKU-999'; quantity = 1; unitPrice = 9.99 })`.

### Postman

`POST http://localhost:3000/orders` with a raw JSON body as above, then
`POST http://localhost:3000/graphql` with body type GraphQL and the same query.

### Expected results

| Scenario | Items in the request | Expected `status` and `cancellationReason` | Side effects |
|---|---|---|---|
| Happy path | `SKU-003` × 2 at 19.99 | `CONFIRMED`, `null` | Payment `Processed`; `SKU-003` stock 100 → 98 |
| Payment failure | `SKU-001` × 1 at 15000 | `CANCELLED`, `payment failed: Amount 15000 USD exceeds the maximum allowed threshold of 10000.00 USD.` | Payment `Failed`; the stock reserved for it is released, so `SKU-001` stays at 10 |
| Stock rejection | `SKU-005` × 1 at 9.99 (or more units than available, e.g. `SKU-004` × 6) | `CANCELLED`, `stock rejected: insufficient stock` | Payment `Processed`, then `Refunded` |
| Unknown product | `SKU-999` × 1 at 9.99 | `CANCELLED`, `stock rejected: unknown product` | Payment `Processed`, then `Refunded` |

To watch the events, start with `--profile tools` and open Kafka UI → Topics. To check the
databases directly:

```bash
docker compose exec postgres-inventory psql -U inventory_user -d inventory -c "select * from stock_items order by 1"
docker compose exec postgres-payments psql -U payments_user -d payments -c 'select "OrderId","Status","Amount" from payments'
```

## Reset

```bash
docker compose --profile apps --profile tools down -v
```

`-v` deletes every volume (Kafka log, all databases, MongoDB). The next `up` recreates the topics,
the schemas and the demo data from scratch. Without `-v`, data is kept and the seeds leave existing
rows alone.

## Troubleshooting

- **A service keeps restarting**: `docker compose logs -f <service>`. Payments exits with code 1
  when its startup migration fails; the Critical log line says why.
- **`port is already allocated`**: something else on the host uses that port (often a service
  you started from your IDE, or a local Postgres on 5432). Stop it or change the host side of the
  port mapping.
- **`build context ../ecommerce-… not found`**: the service repositories must sit next to this
  one (see [Prerequisites](#prerequisites)).
- **`orders-seed` fails with `relation "customers" does not exist`**: your `ecommerce-orders`
  checkout is older than the version that migrates on startup. Update it and rebuild with
  `--build`.
- **Code changes not picked up**: add `--build` to `up`, so the images are rebuilt.
- **Orders stays `PENDING`**: check that `payments` and `inventory` are running and that
  `kafka-init` exited with code 0 (`docker compose --profile apps ps -a`). Any message that fails
  is retried and then dead-lettered; see each service's logs.
- **Payments log line `libgssapi_krb5.so.2: cannot open shared object file`**: harmless. The
  Postgres driver checks for Kerberos support, which the image does not include, and falls back
  to password auth.
- **Topic missing or `UNKNOWN_TOPIC_OR_PARTITION`**: re-run `docker compose up kafka-init`; it only
  creates what is missing.
- **Stale schema or data after switching branches in a service repo**: reset with `down -v`.
