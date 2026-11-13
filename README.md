# ecommerce-infra

Shared local infrastructure for the ecommerce platform, defined in `docker-compose.yml`:

- **Kafka** (KRaft mode, no Zookeeper), plus a one-shot `kafka-init` container that creates every topic
- **PostgreSQL**, one database per service (Orders, Inventory, Payments)
- **MongoDB** for the Orders read model and the Notifications log
- **Kafka UI** (optional)

## Quickstart

```bash
docker compose up -d
```

To also start Kafka UI on http://localhost:8080:

```bash
docker compose --profile tools up -d
```

Stop with `docker compose down` (add `-v` to delete the data volumes).

## Connection details

Services running on the host connect through `localhost`. Containers on the compose network use the internal names (for example `kafka:29092`, `postgres-orders:5432`).

| Service | Host port | Database | User | Password |
|---|---|---|---|---|
| Kafka | `9092` (internal `kafka:29092`) | | | |
| Postgres (Orders) | `5432` | `orders` | `orders_user` | `orders_pass` |
| Postgres (Inventory) | `5433` | `inventory` | `inventory_user` | `inventory_pass` |
| Postgres (Payments) | `5434` | `payments` | `payments_user` | `payments_pass` |
| MongoDB | `27017` | | `mongo_user` | `mongo_pass` (auth database `admin`) |
| Kafka UI | `8080` (`tools` profile) | | | |

These credentials are for local development only.

## Kafka topics

Topic auto-creation is off. `kafka-init` creates these with 3 partitions and replication factor 1:

- `orders.order-placed`
- `orders.order-confirmed`
- `orders.order-cancelled`
- `payments.payment-processed`
- `payments.payment-failed`
- `payments.payment-refunded`
- `inventory.stock-reserved`
- `inventory.stock-rejected`
- `inventory.stock-released`

Publishers, consumers and payload schemas are in [docs/events.md](docs/events.md).
